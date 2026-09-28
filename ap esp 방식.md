# AP(1번) ↔ ESP32(2번) 학습 구현 방식 

## 1. AP(1번, AX7800M-6E) 학습 방식

**데이터 파싱**: MediaTek `mt7915` 드라이버의 커널 CSI 이벤트를 OpenWrt에서 UDP로 덤프받아 pcap 저장. RX0/RX1 이벤트를 timestamp 기준으로 pair 매칭하는 전용 파서 사용.

**전처리**:
- 패킷 수가 파일마다 달라 timestamp 기준 선형보간으로 고정 길이 T=512로 리샘플
- subcarrier 256개, 배열 shape `(2, 256, 512)` [I, Q]
- amplitude(`sqrt(I²+Q²)`) 파생, z-normalization은 train split 통계로만 계산해 val/test에 적용

**모델**: `CSI2DCNN` — ConvBlock×4(32→64→128→192채널) + AdaptiveAvgPool2d + FC(192→128→7)

**학습 설정**: IQ+amplitude(6채널, RX0+RX1), augmentation 없음, BatchNorm, AdamW + ReduceLROnPlateau + early stopping(patience=15)

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

**학습 설정**: IQ+amplitude(3채널, RX0 단일), augmentation 없음, BatchNorm, AdamW + ReduceLROnPlateau + early stopping(patience=15)

**결과**: val 90.9%, test **52.3%** (피험자 홀드아웃, 사람1→train, 사람2→test)

**사용 스크립트**: `train_csi_cnn_esp32.py`, `preprocess_esp32_csi.py`

---

## 3. 같은 스크립트를 썼는가?

**아니다, 스크립트 파일 자체는 서로 다르다.** AP는 `train_csi_cnn.py`/`preprocess_csi_rx01_windows.py`, ESP32는 `train_csi_cnn_esp32.py`/`preprocess_esp32_csi.py`로 **별도 파일**입니다. ESP32용 스크립트는 AP용 스크립트를 베이스로 "파생(derive)"해서 만든 것이며, 파일을 그대로 재사용한 게 아니라 **로직(레시피)만 그대로 복제하고 하드웨어 관련 부분만 새로 작성**.

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
| Optimizer/스케줄/early stopping | AdamW+ReduceLROnPlateau+patience15 | 동일 | ✅ 동일 |
| 스크립트 파일 | `train_csi_cnn.py` | `train_csi_cnn_esp32.py` (파생) | ❌ 파일은 다름 |

**요약**: 스크립트 파일은 플랫폼별로 따로 존재하지만, 그 안의 **전처리 철학과 학습 레시피(모델 구조, augmentation 여부, normalization 방식, optimizer/스케줄)는 완전히 동일하게 이식**. 달라진 부분은 전부 하드웨어 스펙 차이(subcarrier 수, 안테나 수, 캡처 포맷)에서 불가피하게 발생한 것.

**주의**: 이 설명은 "전처리·학습 파이프라인"만 다룬다. 실제 촬영(캡처) 당시의 거리·시간·트리거 방식 등 물리적 프로토콜은 스크립트/문서 어디에도 기록되어 있지 않다.

---
*작성: 2026-09-28 *
