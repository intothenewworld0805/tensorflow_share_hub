# 🏆 리그 오브 레전드 분당 기여도 분석 프로젝트 (README Draft)

> 본 프로젝트는 챌린저 티어 53,000 매치의 분단위 타임라인 데이터를 분석하여, 각 라인의 '경기 기여도'를 정량적으로 정의하고, 이를 통해 경기 데이터를 해석하는 것을 목표로 합니다.

![Python](https://img.shields.io/badge/Python-3.10%2B-blue?logo=python)
![Pandas](https://img.shields.io/badge/Pandas-blue?logo=pandas)
![Matplotlib](https://img.shields.io/badge/Matplotlib-grey?logo=matplotlib)
![Seaborn](https://img.shields.io/badge/Seaborn-darkblue?logo=seaborn)
![Scikit-learn](https://img.shields.io/badge/Scikit--learn-orange?logo=scikit-learn)

---

## 1. 프로젝트 목표

'기여도'라는 모호한 개념을 정량화하는 것이 본 프로젝트의 핵심입니다. 저희 팀은 "승리에 얼마나 기여했는가"를 측정하기 위해 분단위 데이터를 기반으로 한 단일 지표, **`ContributionScore`** (기여도 점수)를 개발했습니다.

* **최종 목표 1:** 라인별(TOP, JUNGLE, MID, BOTTOM) 분당 기여도(`ContributionScore`) 모델 개발
* **최종 목표 2:** 개발된 기여도 점수를 활용하여 유의미한 게임 패턴 분석
    * (분석 1) 라인별 파워 커브: 시간에 따른 라인별 영향력 변화
    * (분석 2) 챌린저 판독: 패배 시에도 높은 기여도를 유지하는 최상위 플레이어 식별

## 2. 데이터

* **소스:** 챌린저 티어 300명 소환사
* **규모:** 약 53,000 매치 데이터 (2025.01 – 2025.04)
* **핵심:** **분단위 타임라인 (`timeline_*.json`)** 데이터

---

## 3. 핵심 방법론: 피처 선택 및 모델링

본 프로젝트의 성패는 **어떤 피처를 선택(Feature Selection)**하고, 이를 **어떻게 조합하여 기여도로 모델링(Modeling)**하느냐에 달려있습니다.

### 3.1. 피처 엔지니어링 및 선택 (Feature Engineering & Selection)

단순히 경기 종료 시점의 데이터를 사용하지 않고, 분단위(`1min` ~ `40min`)로 데이터를 처리하여 '기여도의 누적 과정'을 추적했습니다.

1.  **데이터 가공:** `match_*.json`과 `timeline_*.json` 파일을 매칭했습니다.
2.  **상대 매칭:** `get_opponent_map` 함수를 통해 모든 분(minute)마다 정확한 라인전 상대를 특정했습니다.
3.  **핵심 피처 3가지 선택 및 근거:**
    '기여도'를 **(1) 라인전 우위**와 **(2) 팀 기여(절대 총량)**라는 두 가지 축으로 정의했습니다.

    * **`GD` (Gold Difference):** (라인전 우위) 상대 라이너 대비 누적 골드 격차. 라인전의 스노우볼을 측정하는 가장 직관적인 지표입니다.
    * **`CSD` (CS Difference):** (라인전 우위) 상대 라이너 대비 누적 CS 격차. 골드와 함께 라인전 기본기를 나타냅니다.
    * **`Dmg` (Total Damage Done to Champions):** (팀 기여) 챔피언에게 가한 누적 피해량. 골드/CS 격차를 실제 교전 영향력으로 변환했는지 측정하는 핵심 지표입니다.

    > **※ 피처 제외 근거:** `visionScore` (시야 점수)의 경우, 경기 종료 시점(`match.json`)에는 존재하지만, 분단위 타임라인 프레임(`timeline.json`)에는 존재하지 않아 모델에서 제외했습니다.

### 3.2. 기여도 모델링: 시간대별 Z-Score 정규화 모델 (Time-Segmented Z-Score Model)

선택한 3개의 피처(`GD`, `CSD`, `Dmg`)는 단위(Gold, CS, Damage)가 다르고, 게임 시간에 따라 그 가치가 완전히 다릅니다. (예: 10분 1000골드 격차 vs 40분 1000골드 격차)

이 문제를 해결하기 위해, 저희는 **"모든 경기의 같은 시간대"**와 비교하는 모델을 설계했습니다.

1.  **시간대별 분리 (Time-Segmented):**
    먼저 모든 데이터를 '분(Minute)' 단위로 그룹화합니다.

2.  **Z-Score 정규화 (Standardization):**
    각 피처를 해당 시간대의 평균($\mu$)과 표준편차($\sigma$)를 이용해 정규화(Z-Score)합니다.
    * *예시: 한 선수의 '15분' `GD` 점수는, 다른 **모든 경기의 '15분' `GD` 값**들과 비교되어 Z-Score가 계산됩니다.*

    $$Z_{metric} = \frac{X_{metric, min} - \mu_{metric, min}}{\sigma_{metric, min}}$$

3.  **기여도 점수(`ContributionScore`) 산출:**
    산출된 3개 피처의 Z-Score를 **평균**내어, 시간과 단위에 관계없이 표준화된 '종합 기여도'를 계산합니다.

    * `Z_Total = (Z_GD + Z_CSD + Z_Dmg) / 3`

4.  **최종 스케일링 (중앙값=1):**
    프로젝트 가이드라인(절대 스케일 기준: 중앙값=1)을 충족하기 위해, Z-Score 평균값에 1.0을 더합니다.

    $$ContributionScore = \frac{Z_{GD} + Z_{CSD} + Z_{Dmg}}{3} + 1.0$$

    * **해석:** `ContributionScore`가 **1.0**이면 정확히 해당 시간대의 전체 평균 기여도를 의미합니다. 1.0보다 높으면 평균 이상, 낮으면 평균 이하의 퍼포먼스입니다.

## 4. 진행 중 주요 분석 결과 (요약)

이 모델을 통해 도출된 `ContributionScore`로 다음과 같은 분석을 수행했습니다.

1.  **라인별 파워 커브:**
    * 
    * (분석 예시) JUNGLE은 5~15분 구간에서 가장 높은 기여도를 보였으며, BOTTOM(ADC) 라인은 25분 이후 기여도가 급격히 상승하여 '후반 캐리' 패턴을 데이터로 입증했습니다.

2.  **챌린저 플레이어 일관성 분석:**
    * 단순히 평균 기여도(`AvgContribution`)가 높은 선수가 아니라, **패배한 게임에서의 평균 기여도(`LossAvgContribution`)**가 1.0 이상(평균 이상)을 유지하는 선수를 '최상위 챌린저/프로'로 식별했습니다.
    * (분석 예시) 랭킹 1위 'Player A'는 전체 평균 1.35, 패배 시 평균 1.15를 기록하여, 지는 게임에서도 1인분 이상을 해내는 '방어력'을 입증했습니다.

## 5. 결론

본 프로젝트는 `GD`, `CSD`, `Dmg`라는 3가지 핵심 피처를 '시간대별 Z-Score 정규화'라는 모델을 통해 결합하여, 객관적이고 정량화된 `ContributionScore` 지표를 성공적으로 개발했습니다. 이 지표는 라인별 영향력 변화와 최상위 플레이어의 일관성을 명확하게 분석하는 강력한 도구임을 확인했습니다.
