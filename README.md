# 🐐 웨어러블 센서 기반 염소 분만 조기경보 알고리즘

![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![Jupyter](https://img.shields.io/badge/Jupyter-orange?style=flat&logo=jupyter&logoColor=white)
![VSCode](https://img.shields.io/badge/VSCode-007ACC?style=flat&logo=visualstudiocode&logoColor=white)

## Project Information

- 대회명: 제4회 스마트축산 AI 경진대회
- 참가 부문: 알고리즘 부문
- 과제 유형: 기술개발 과제(상용화 이전)
- 공모 분야: 사양관리
- 축종: 염소
- 기술명: 웨어러블 센서 기반 가축 분만 조기경보 알고리즘: 염소 데이터 응용
- 기술 유형: 사양관리 / 분만 임박 조기탐지
- 팀명: 축산시그널 (대표: 최현석, 팀원: 임연주)

## Overview

본 프로젝트는 축산 현장에서 관리자가 다수의 임신 개체를 지속적으로 관찰해야 하는 부담을 줄이고, 분만 임박 개체를 조기에 선별하기 위한 **센서 기반 분만 조기경보 알고리즘**을 제안합니다.
포르투갈 Santarém 지역의 실제 농장에서 수집된 염소 목걸이형 웨어러블 센서 데이터를 활용하였으며, 3축 가속도와 온도 시계열로부터 분만 전 행동 변화를 나타내는 특징을 생성하였습니다.
이후 **Generalized Additive Model(GAM)**을 이용하여 각 시점의 분만 위험확률을 추정하고, **Exponentially Weighted Moving Average(EWMA)**를 적용하여 일시적인 위험 변동을 완화한 뒤 최종 분만 임박 경보를 발생시키는 2단계 구조를 구축하였습니다.
최종적으로 **향후 2시간 이내 분만이 예상되는 개체를 자동으로 선별**하여 관리자가 우선적으로 확인할 수 있는 데이터 기반 상시 모니터링 체계를 제안하는 것을 목표로 합니다.

## Research Topic

**Early Warning Algorithm for Goat Kidding Using Wearable Sensor Time-Series Data**

## Dataset

본 프로젝트에서는 포르투갈 Santarém 지역의 실제 농장 환경에서 수집되어 Zenodo에 공개된 염소 분만 센서 데이터를 활용했습니다.

- 염소 27개체, 개체별 CSV 파일
- 20 Hz 원시 센서 데이터
- 2024년 및 2025년 분만기 수집
- 최종 분석 개체: 26개체

### Variables

| Variable | Description |
|---|---|
| `Time` | 측정 시각 |
| `Acc_X` | 좌·우 방향 가속도 |
| `Acc_Y` | 전·후 방향 가속도 |
| `Acc_Z` | 상·하 방향 가속도 |
| `Temperature` | 목걸이형 센서 온도 |
| `Class` | 실제 분만 시점을 기준으로 한 시간 구간 라벨 |

`Class`는 정상 구간부터 분만 직전, 분만 시점 및 분만 이후를 시간적으로 나타내도록 구성되어 있으며, 본 프로젝트에서는 실제 농가의 사전 대응 시간을 고려하여 **분만 전 2시간 이내 여부**를 주요 탐지 대상으로 정의했습니다.

### Original Dataset

[📂 **Zenodo Dataset**](https://zenodo.org/records/17743794)

## Data Preprocessing

웨어러블 센서의 실제 시계열 특성을 고려하여 데이터 품질 점검부터 특징 생성까지 단계적인 전처리를 수행했습니다.

### 1. Sensor Data Quality Check

- 개체별 측정 시각 기준 정렬
- 데이터 연속성 및 중복 여부 확인
- 센서 중단 및 결측 구간 점검
- 가속도·온도 이상값 확인 및 정제
- 실제 분만 시각과 `Class` 흐름 비교 및 보정

### 2. Common Analysis Window

개체별 관측기간의 차이를 줄이고 동일한 기준에서 비교하기 위해 분만 시점을 기준으로 공통 분석 구간을 설정했습니다.

- 공통 전처리 구간: 분만 약 13시간 전 ~ 분만 시점
- 모델 입력 구간: 분만 12시간 전 ~ 분만 시점

### 3. ODBA Calculation

3축 가속도로부터 염소의 전반적인 활동 강도를 나타내는  
**Overall Dynamic Body Acceleration(ODBA)**를 산출했습니다.

ODBA 계산 과정은 다음과 같습니다.

1. 10초 time window 구성
2. 2.5초 간격 이동
3. 75% overlapping 적용
4. 각 축의 평균값을 정적가속도로 추정
5. 원 신호에서 정적가속도를 제거하여 동적가속도 계산
6. X·Y·Z축 동적가속도의 절댓값 합산
7. 각 window의 ODBA 중앙값 산출
8. 1분 단위 시계열로 집계

### 4. Time-Series Feature Engineering

현재 시점까지의 **과거 데이터만 이용하는 causal time window**를 적용하여 최종 5개의 입력변수를 구성했습니다.

| Feature | Description | Interpretation |
|---|---|---|
| `O30m` | 최근 30분 평균 ODBA | 전반적인 활동 수준 |
| `D15-30m` | 최근 15분 평균 ODBA − 최근 30분 평균 ODBA | 단기 활동 변화 |
| `MAD15m` | 최근 15분 ODBA 중앙절대편차 | 활동 변동성 |
| `Act15m` | 최근 15분 고활동 비율 | 고강도 활동 정보 |
| `Temp60m` | 최근 60분 평균 센서 온도 | 온도 변화 |

## Methodology

본 프로젝트에서는 분만 위험을 추정하는 단계와 시간에 따른 위험 변화를 반영하는 단계를 결합한 **GAM-EWMA 기반 2단계 조기경보 알고리즘**을 구성했습니다.

### Stage 1. GAM-based Kidding Risk Estimation

먼저 실제 분만 시점을 `T`, 현재 시점을 `t`라고 할 때,

**향후 2시간 이내 분만이 발생하는지 여부**를 분만 임박 상태로 정의했습니다.

5개의 시계열 입력변수를 이용하여 **GAM**이 각 시점의 분만 위험확률을 추정합니다.

GAM은 입력변수와 분만 임박 위험의 관계를 선형으로 제한하지 않고, 각 변수의 영향을 부드러운 비선형 함수로 표현할 수 있다는 장점이 있습니다.

모델의 출력은 다음과 같은 연속적인 위험확률입니다.

`p_hat(t) = estimated kidding risk probability`

### Stage 2. EWMA-based Sequential Alarm

GAM의 시점별 위험확률은 일시적인 움직임 변화에 의해 크게 변동할 수 있습니다.

따라서 **EWMA**를 적용하여 위험확률을 시간적으로 평활화했습니다.

`Z(t) = λ × p_hat(t) + (1 - λ) × Z(t-1)`

여기서

- `p_hat(t)` : GAM에서 추정한 시점별 분만 위험확률
- `Z(t)` : EWMA로 평활화된 분만 위험점수
- `λ` : 현재 위험확률에 부여하는 가중치

최종적으로 평활 위험점수 `Z(t)`가 경보 임계값 `c`를 초과하면 분만 임박 경보를 발생시킵니다.

`Alarm(t) = 1 if Z(t) > c, otherwise 0`

## Model Validation

개체 수가 제한적이고 염소마다 행동 특성이 다르다는 점을 고려하여  
**Nested Leave-One-Subject-Out Cross-Validation (Nested LOSO-CV)**을 적용했습니다.

### Outer LOSO

- 전체 26개체 중 1개체를 최종 평가용 test goat로 분리
- 나머지 25개체로 모델 학습 및 설정 결정
- 모든 개체가 한 번씩 test 개체가 되도록 총 26회 반복

### Inner LOSO

Outer fold의 25개 학습 개체 중 다시 1개체를 validation goat로 분리하고, 나머지 24개체로 모델을 학습했습니다.

이를 25회 반복하여 다음 요소를 결정했습니다.

- GAM hyperparameters
- EWMA weight `λ`
- Alarm threshold `c`

이 구조를 통해 **최종 test 개체의 정보가 모델 학습이나 하이퍼파라미터·임계값 결정 과정에 사용되지 않도록 구성**했습니다.

## Evaluation Strategy

분만 조기경보 시스템에서는 단순한 시점별 정확도보다 실제 농장에서 의미 있는 이벤트 기반 평가가 중요합니다.

따라서 다음 지표를 중심으로 성능을 평가했습니다.

- ROC-AUC
- Event Recall
- False Alarms / Animal-Day
- Lead Time

특히

- `p_hat(t)` 및 `Z(t)`는 연속적인 **score**
- `c`는 **threshold**
- `I(Z(t) > c)`는 최종 **binary alarm**

으로 구분하여 평가했습니다.

## Results

Nested LOSO-CV를 이용하여 총 26개체를 평가한 결과, 분만 시점이 가까워질수록 **EWMA 평활 위험점수 `Z(t)`가 전반적으로 상승하는 경향**을 보였으며, 특히 분만 전 2시간 이내에서 상승 폭이 커지는 양상이 관찰되었습니다.

| Metric | Result |
|---|---:|
| ROC-AUC | 0.75 |
| Detected Events | 16 / 26 |
| Event Recall | 61.5% |
| Mean Lead Time | 약 1.45 h |
| Median Lead Time | 약 1.60 h |

대표적인 탐지 성공 개체에서는 분만 시점에 가까워질수록 GAM의 위험확률이 증가하고, EWMA로 평활된 위험점수가 임계값을 초과하면서 최종 경보가 발생하는 패턴을 확인했습니다.

또한 탐지 대상 구간 이전의 일시적인 위험확률 상승은 EWMA 적용 후 완화되어, 순간적인 센서 변동보다 **지속적으로 증가하는 위험 신호를 경보에 반영**하는 효과를 확인했습니다.

## Key Contribution

- 실제 농장 환경에서 수집된 **목걸이형 3축 가속도·온도 시계열 데이터 활용**
- 3축 가속도로부터 **ODBA 기반 활동정보 산출**
- 활동 수준·단기 변화·변동성·고활동 비율·온도를 반영한 **5개 causal time-series features 구성**
- **GAM을 이용한 시점별 분만 위험확률 추정**
- **EWMA를 이용한 위험 신호 평활 및 순차적 경보 판정**
- 개체 간 데이터 누수를 방지하기 위한 **Nested LOSO-CV 적용**
- 분만 2시간 이내 위험 개체를 자동 선별하는 **현장 적용형 조기경보 구조 제안**

## Expected Impact

본 프로젝트의 분만 조기경보 알고리즘은 실제 축산 현장에서 다음과 같이 활용될 수 있습니다.

- 분만 임박 개체를 우선 선별하여 반복적인 육안 관찰 부담 완화
- 사전 대응 시간을 확보하여 분만 이상 상황에 신속하게 대응
- 관리자의 경험 중심 분만 판단을 데이터 기반 의사결정으로 보완
- 다수 개체의 분만 위험도를 지속적으로 확인하는 상시 모니터링 체계 구축
- 농가의 분만 관리 효율 향상 및 데이터 기반 정밀축산 적용 확대

## Limitations

본 프로젝트는 실제 현장 적용 가능성을 중심으로 설계되었지만 몇 가지 한계가 존재합니다.

- 분석 개체 수와 관측기간이 제한적이며 일부 개체에서 센서 결측이 발생함
- 다중분만 등 개체별 행동 차이로 탐지 성능에 편차가 나타남
- 향후 추가 개체 및 장기간 데이터 기반의 일반화 성능 검증이 필요함

따라서 현재 결과는 완성된 상용 시스템의 성능이라기보다, **웨어러블 센서 기반 분만 조기경보 시스템의 현장 적용 가능성을 검토한 기술개발 단계의 결과**로 해석할 필요가 있습니다.

## Future Work

### Short-term

- 추가 염소 개체 데이터 확보
- 장기간 시계열 데이터 수집
- 센서 결측 및 다양한 분만 형태에 대한 모델 안정성 검증
- 새로운 개체에 대한 일반화 성능 추가 평가

### Mid-term

**Collar → Gateway → Analysis Server → Mobile Application / Smart Watch**

구조를 구축하여 실시간 분만 조기경보 시스템으로 확장할 계획입니다.

모바일 환경에서는 다음 정보를 제공할 수 있도록 구성할 수 있습니다.

- 전체 개체 모니터링
- 위험 개체 목록
- 개체별 분만 위험도
- 분만 임박 경보
- 위험도 변화 추이

### Long-term

현재의 분만 조기경보 구조를 기반으로

- 발정 탐지
- 질병 이상징후 탐지
- 건강 상태 모니터링
- 다른 축종으로의 적용

등으로 탐지 범위를 확대하여 **통합 스마트축산 모니터링 플랫폼**으로 발전시키는 것을 목표로 합니다.

## Repository Structure

```text
├── data/
│   ├── raw/
│   └── processed/
│
├── notebooks/
│   ├── preprocessing.ipynb
│   └── GAM-EWMA.ipynb
│
├── docs/
│
└── README.md
