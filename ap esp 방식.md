# AP(1번) ↔ ESP32(2번) 학습 구현 방식 비교 — 2026-09-28 (정정판)

## 1. AP(1번, AX7800M-6E) 학습 방식

**데이터 파싱**: MediaTek `mt7915` 드라이버의 커널 CSI 이벤트를 OpenWrt에서 UDP로 덤프받아 pcap 저장. RX0/RX1 이벤트를 timestamp 기준으로 pair 매칭하는 전용 파서 사용.

**전처리**:
- 패킷 수가 파일마다 달라 timestamp 기준 선형보간으로 고정 길이 T=512로 리샘플
- subcarrier 256개, 배열 shape `(2, 256, 512)` [I, Q]
- amplitude(`sqrt(I²+Q²)`) 파생, z-normalization은 train split 통계로만 계산해 val/test에 적용

**모델**: `CSI2DCNN` — ConvBlock×4(32→64→128→192채널) + AdaptiveAvgPool2d + FC(192→128→7)

**학습 설정 (run_config.json 실측값)**:

| 항목 | 값 |
|---|---|
| 입력 특징 | IQ + amplitude (6채널, RX0+RX1) |
| Augmentation | 없음 |
| Norm | BatchNorm |
| Optimizer | AdamW |
| Learning rate | 3e-4 |
| Weight decay | 1e-4 |
| Epochs | 40 |
| Batch size | 16 |
| Scheduler | ReduceLROnPlateau (factor=0.5, patience=3) |
| Early stopping patience | **8** |
| 학습 장비 | GPU (NVIDIA GB10, CUDA) — DGX 서버 |

**결과**: val 94.9%, test **90.2%** (피험자 홀드아웃, jun→train, sin→test)

**사용 스크립트**: `train_csi_cnn.py`, `preprocess_csi_rx01_windows.py`

---

## 2. ESP32(2번) 학습 방식

**데이터 파싱**: ESP-IDF 펌웨어(`csi_recv_s3`)의 자체 pcap payload 포맷(`b"CSI:" + meta(">IIbBBbHH", 16바이트) + raw CSI(int8 I/Q)`)을 scapy로 직접 파싱.

**전처리**:
- 패킷 수가 파일마다 달라(약 20~90개) timestamp 기준 선형보간으로 고정 길이 T=128로 리샘플
- subcarrier 192개, 배열 shape `(2, 192, 128)` [I, Q]
- amplitude 파생, z-normalization은 train split 통계로만 계산해 val/test에 적용
- 유효 레코드 4개 미만 파일은 스킵 (700개 중 58개)

**모델**: `CSI2DCNN` — AP와 동일한 클래스 구조, 입력 채널/subcarrier/T 상수만 변경

**학습 설정 (스크립트 실측값)**:

| 항목 | 값 |
|---|---|
| 입력 특징 | IQ + amplitude (3채널, RX0 단일) |
| Augmentation | 없음 |
| Norm | BatchNorm |
| Optimizer | AdamW |
| Learning rate | 1e-3 |
| Weight decay | 1e-4 |
| Epochs | 80 |
| Batch size | 16 |
| Scheduler | ReduceLROnPlateau (factor=0.5, patience=5) |
| Early stopping patience | **15** |
| 학습 장비 | CPU (클라우드 샌드박스, GPU 없음) |

**결과**: val 90.9%, test **52.3%** (피험자 홀드아웃, 사람1→train, 사람2→test)

**사용 스크립트**: `train_csi_cnn_esp32.py`, `preprocess_esp32_csi.py`

---

## 3. 같은 스크립트를 썼는가?

**아니요, 스크립트 파일 자체는 서로 다릅니다.** AP는 `train_csi_cnn.py`/`preprocess_csi_rx01_windows.py`, ESP32는 `train_csi_cnn_esp32.py`/`preprocess_esp32_csi.py`로 **별도 파일**입니다. ESP32용 스크립트는 AP용 스크립트를 베이스로 "파생(derive)"해서 만든 것이며, 파일을 그대로 재사용한 게 아니라 **핵심 설계 원칙(모델 구조, augmentation 미사용, BatchNorm, optimizer 종류)만 그대로 복제하고, 나머지(하이퍼파라미터 구체값, 하드웨어 관련 파싱)는 새로 작성/조정**했습니다.

## 4. 두 방식의 차이점 정리

| 항목 | AP(1번) | ESP32(2번) | 같은가 |
|---|---|---|---|
| 원본 파싱 로직 | MediaTek CSI 이벤트 파서 | ESP-IDF 자체 pcap 포맷 파서 | ❌ 다름 (하드웨어 상이) |
| subcarrier 수 | 256 | 192 | ❌ 다름 |
| 시간축 T | 512 | 128 | ❌ 다름 (캡처 길이 차이) |
| 안테나 수 | 2 (RX0+RX1) | 1 (RX0만) | ❌ 다름 |
| 입력 채널(iq_amp) | 6채널 | 3채널 | ❌ 다름 |
| 리샘플링 방식(timestamp 선형보간) | 사용 | 사용 | ✅ 동일 |
| z-normalization 방식(train 통계만) | 사용 | 사용 | ✅ 동일 |
| CNN 아키텍처 | ConvBlock×4+AdaptiveAvgPool+FC | 동일 구조 | ✅ 동일 |
| Augmentation | 없음 | 없음 | ✅ 동일 |
| Norm 방식 | BatchNorm | BatchNorm | ✅ 동일 |
| Optimizer 종류 | AdamW | AdamW | ✅ 동일 |
| Learning rate | 3e-4 | 1e-3 | ❌ 다름 |
| Epochs | 40 | 80 | ❌ 다름 |
| Early-stop patience | 8 | 15 | ❌ 다름 |
| Scheduler patience | 3 | 5 | ❌ 다름 |
| 학습 장비 | GPU(CUDA) | CPU | ❌ 다름 |
| 스크립트 파일 | `train_csi_cnn.py` | `train_csi_cnn_esp32.py` (파생) | ❌ 파일은 다름 |

**요약**: 스크립트 파일은 플랫폼별로 따로 존재하고, **모델 구조·augmentation 미사용·BatchNorm·optimizer 종류 같은 핵심 설계 원칙은 동일**하게 이식됐지만, **learning rate·epochs·patience 같은 세부 하이퍼파라미터, 그리고 학습에 쓴 하드웨어(GPU vs CPU)는 실제로 다릅니다.** 이 차이가 결과에 얼마나 영향을 줬는지는 별도로 검증되지 않았습니다.

**주의**: 이 설명은 "전처리·학습 파이프라인"만 다룹니다. 실제 촬영(캡처) 당시의 거리·시간·트리거 방식 등 물리적 프로토콜은 스크립트/문서 어디에도 기록되어 있지 않습니다.

---
*작성: 2026-09-28, IIC Lab (하이퍼파라미터 실측 대조 후 정정)*

## 5. AP↔ESP32를 양극단으로 잡으면 성능 저하가 불가피한가? — 2026-09-29

### 왜 자연스럽게 떨어지는가

- AP와 ESP32는 subcarrier 수(256 vs 192), 안테나 수(2 vs 1), raw 신호 스케일이 근본적으로 다른 하드웨어다.
- 지금까지 진행한 실험(리샘플링만 하고 그대로 학습)은 "포즈의 본질적 패턴"이 아니라 "그 하드웨어 특유의 신호 특성"까지 같이 학습해버려서, 장비가 바뀌면 그 특성이 안 맞아 정확도가 붕괴한다. 이는 CSI 센싱 분야에서 이미 잘 알려진 도메인 시프트 문제이며, 크로스 하드웨어 홀드아웃 실험(test 14.3%/15.6%로 붕괴)이 이를 직접 확인해줬다.

### 그렇다고 "원래 안 되는 것"은 아님

문헌(Widar3.0, DATTA, Wi-SFDAGR 등)에서 이미 이 문제를 어느 정도 해결한 사례가 있다. 방법은:

- **도메인 불변 feature**: 절대 신호값 대신 "패턴의 모양"만 남기는 feature 설계 (예: BVP처럼 속도/변화량 기반 표현)
- **도메인 적대적 학습(domain-adversarial)**: 모델이 "어느 장비인지"를 구분 못 하게 강제로 학습시키는 방법
- **소량 fine-tuning**: 타깃 장비 데이터를 아주 조금만 써서 모델을 미세조정

이런 기법을 쓰면 "완전히 동일한 정확도"까지는 아니어도 격차를 상당히 줄일 수 있다는 것이 이 분야의 정설이다.

### 결론

**"지금 방식(그냥 리샘플링해서 같은 모델에 넣기)"으로는 격차가 나는 게 맞고 사실상 불가피하다.** 하지만 "AP-ESP32처럼 양극단 장비 간에는 원천적으로 절대 안 된다"는 것은 아니며, **도메인 적응 기법을 추가해야만 격차를 줄일 수 있다**는 것이 정확한 결론이다. 따라서 프로젝트 목표("병행해도 잘 나옴")는 지금 파이프라인만으로는 달성되지 않고, 위의 도메인 적응 기법들을 추가로 적용해야 도달 가능한 목표다.
