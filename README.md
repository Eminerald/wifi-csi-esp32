# ESP32-S3 CSI 7-Pose Classification 


WiFi CSI(Channel State Information) 기반 사용자 자세(pose) 인식 프로젝트 중,
**ESP32-S3 플랫폼(2번)** 데이터에 대한 모델 학습 결과 정리.

1번 플랫폼(ipTIME AX7800M-6E AP)은 별도 팀원이 담당했으며, 이 문서는 ESP32-S3
CSI 데이터의 전처리 파이프라인 구축부터 모델 학습, 그리고 학습 중 발견한
데이터 일반화/품질 이슈를 정리한다.

## 요약 (TL;DR)

- ESP32(단일 안테나) CSI로 7가지 자세(empty, sitting_still, sitting_pick_up,
  standing_still, standing_look_around, walking, walking_in_place)를
  분류하는 CNN을 학습했다.
- **같은 사람 데이터가 학습에 섞여 있으면 89~94% 정확도**가 나오지만,
  **완전히 새로운 사람으로만 평가하면 48~58%로 떨어진다.** 안테나 2개를 쓰는
  1번(AP) 플랫폼은 동일한 평가 방식으로도 90.2%를 유지한다 — 안테나 1개인
  ESP32가 사람 간 일반화에 더 취약하다는 신호.
- `sitting_still` 클래스는 원본 100개(사람1+사람2) 중 47개(47%)가 유효 CSI
  레코드 부족으로 전처리 단계에서 제외됐다. 원인은 **캡처 당시 ESP32 기기
  발열(온도 과열)**로 확인됨 (데이터 추출 담당자 확인).

## 데이터

- 원본: `esp32s3_cap/{1,2}/{7 pose classes}/*.pcap` (700개, 사람 2명 × 7클래스 × 50회)
- 폴더 구조는 "사람구분 → 행동구분 → 캡처데이터 순서" (세션이 아니라 **피험자 구분**)
- pcap 페이로드 포맷: ESP-IDF 펌웨어(`csi_recv_s3`)의 `tools/csi_pcap_saver.py`
  기준으로 확정 — `b"CSI:" + meta(">IIbBBbHH", 16B) + raw_csi(int8 I/Q pairs)`
- subcarrier 192개 (AP의 256개와 다름, ESP32 고유 스펙)

## 전처리

`preprocess_esp32_csi.py`

- 각 pcap을 파싱해 (I, Q) 시계열 추출 → timestamp 기준 선형보간으로 고정
  길이 T=128 bin으로 리샘플
- 출력 배열: `iq_raw`/`iq_z` = `(2, 192, 128)`, `amplitude`/`amplitude_z` = `(1, 192, 128)`
- z-normalization은 train split 통계로만 계산해 val/test에 동일 적용
- 유효 CSI 레코드 4개 미만인 파일은 스킵: **700개 중 58개 스킵, 642개 사용**

## 모델 / 학습 레시피

1번(AP) 팀이 검증한 레시피를 그대로 재사용 ("돌리는 방식·정제방식을 플랫폼 간 다르게
하지 않는다"는 팀 규칙에 따름):

- **아키텍처**: `CSI2DCNN` — ConvBlock×4 (Conv2d→BatchNorm→ReLU ×2→MaxPool,
  채널 32→64→128→192) + AdaptiveAvgPool2d + FC(192→128→7)
- **입력 특징**: IQ + amplitude (3채널) — amplitude 추가 시 소폭 개선되는
  경향이 AP와 동일하게 재현됨
- **augmentation 없음** (AP 실험에서 augmentation이 성능을 크게 무너뜨린 것 확인 → 동일 적용)
- **BatchNorm** (GroupNorm보다 우수했던 AP 결론 적용)
- AdamW + ReduceLROnPlateau + early stopping(patience=15)

## 실험 결과

| 실험 | 분할 방식 | val 최고 | test |
|---|---|---:|---:|
| 피험자 홀드아웃 (사람1→train/val, 사람2→test) | subject holdout | 90.9~92.4% | 48.1~52.3% |
| 역방향 피험자 홀드아웃 (사람2→train/val, 사람1→test) | subject holdout | 95.2% | 58.4% |
| Capture-holdout (두 사람 섞어 무작위 분할) | 두 사람 혼합, 클래스별 stratified 70/15/15 | 93.9% | 89.4% |
| *(참고) AP(1번, RX0+RX1 6채널)* | 피험자 홀드아웃 (jun→train, sin→test) | 94.9% | **90.2%** |

같은 설정을 두 번 재현했을 때(터미널 재실행) val 90.9%→92.4%, test 52.3%→48.1%로
약간의 변동은 있었지만 **"피험자 홀드아웃에서 val-test 격차가 크게 벌어진다"는
핵심 패턴은 재현됨.**

### 클래스별 test 정확도 (피험자 홀드아웃, 사람1→train/사람2→test 기준)

| 클래스 | 정확도 |
|---|---:|
| empty | 90~98% |
| sitting_pick_up | 46~90% |
| sitting_still | 33~93% (표본 15개뿐, 변동 큼) |
| standing_look_around | **0%** (양방향 홀드아웃 모두 0%) |
| standing_still | 30~70% |
| walking | 20~36% |
| walking_in_place | 60~98% |

## 핵심 발견

### 1. 사람 간 일반화 문제 (안테나 개수 가설)

같은 CNN 구조를 두 방향(사람1↔사람2)으로 홀드아웃해도, capture-holdout(89.4%) 대비
피험자 홀드아웃(48~58%)에서 공통적으로 큰 낙폭을 보였다. 반면 AP(안테나 2개)는
동일한 피험자 홀드아웃 방식으로도 90.2%를 유지했다.

**가설**: 안테나가 1개뿐이라 사람마다 달라지는 반사/다중경로 패턴을 상쇄할 공간
정보가 부족해, 모델이 "포즈"보다 "그 사람 특유의 CSI 패턴"을 더 많이 학습했을
가능성. 현재 피험자가 2명뿐이라 통계적으로 확정할 수는 없고, **피험자 수를 늘린
재검증이 필요하다.**

`standing_look_around`는 두 홀드아웃 방향 모두 test 정확도 0%로, 사람에 따라 이
클래스의 CSI 패턴이 특히 크게 달라지거나 다른 클래스(주로 walking)와 지속적으로
혼동되는 것으로 보인다.

### 2. `sitting_still` 데이터 품질 이슈 (하드웨어 원인)

| | 사람1 사용가능 | 사람2 사용가능 |
|---|---:|---:|
| sitting_still | 38/50 (24% 스킵) | 15/50 (**70% 스킵**) |
| 나머지 6개 클래스 평균 | 48~50/50 (0~6% 스킵) | 47~50/50 (0~4% 스킵) |

**원인 확인 (데이터 추출 담당자): 캡처 당시 ESP32 기기 온도 과열.** 전처리
스크립트 버그나 라벨링 오류가 아니라 하드웨어 발열로 인한 캡처 품질 저하로
확인됨. 재캡처 시 온도 모니터링 또는 클래스 간 쿨링 시간 확보가 필요하다.

## 재현 방법

```bash
# 1. 전처리 (700개 pcap -> npz)
python3 preprocess_esp32_csi.py \
  --root esp32s3_cap \
  --out-dir esp32_windows_20260921

# 2. 학습 (피험자 홀드아웃, 사람1->train/val, 사람2->test)
python3 train_csi_cnn_esp32.py \
  --data-root esp32_windows_20260921 \
  --manifest esp32_windows_20260921/manifest.csv \
  --class-names esp32_windows_20260921/class_names.json \
  --output-dir runs/esp32_iq_amp_baseline \
  --feature iq_amp --epochs 80 --batch-size 16 --patience 15 --norm batch --device cpu

# 3. 진단 실험 (역방향 홀드아웃 / capture-holdout)
python3 reverse_holdout.py
python3 capture_holdout_diagnostic.py
```

## 알려진 한계 / 다음 단계

1. **피험자가 2명뿐** — 3번째 피험자 데이터를 확보해 최소 3-fold 교차검증으로
   "안테나 개수 vs 사람 간 일반화" 가설을 재검증해야 한다.
2. **`sitting_still` 표본 부족** — 발열 방지 대책(쿨링, 촬영 순서 조정) 후
   재캡처 검토.
3. **`standing_look_around` 0% 원인** — confusion matrix
   (`runs/*/test_confusion_matrix.csv`)를 통한 추가 분석 필요.
4. 현재 결과를 "ESP32 최종 성능"으로 보고할 때는 반드시 분할 방식(피험자
   홀드아웃 vs capture-holdout)을 명시할 것 — 두 수치의 의미가 다르다.

## 산출물 위치

```
wifi-csi/
├── esp32_windows_20260921/         # 전처리된 npz 데이터, manifest.csv
├── preprocess_esp32_csi.py         # pcap -> npz 전처리 스크립트
├── train_csi_cnn_esp32.py          # 학습 스크립트 (피험자 홀드아웃)
├── reverse_holdout.py              # 역방향 피험자 홀드아웃 진단 스크립트
├── capture_holdout_diagnostic.py   # capture-holdout 진단 스크립트
├── visualize_sitting_still_issue.py # sitting_still 스킵 이슈 시각화
└── runs/
    ├── esp32_iq_amp_baseline_20260921/
    ├── esp32_iq_only_baseline_20260921/
    ├── esp32_reverse_holdout_iq_amp_20260921/
    └── esp32_capture_holdout_iq_amp_20260921/
```

---
*작성: 2026-09-21
