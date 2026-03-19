# Evidence Narrative: 독성 예측 모델 종합 분석

## Methods Overview

본 연구는 1,247개 화학물질의 구조적 특성(molecular descriptors)과 독성 라벨(EPA ToxCast)을 활용하였다. 모델 학습에는 RDKit로 계산된 200개 분자 기술자를 사용하였으며, Nested 5-fold CV로 모델 선택과 성능 평가를 분리하였다.

## Q1: XGBoost가 Random Forest보다 예측 정확도가 높은가?

### Verdict
XGBoost가 Random Forest 대비 통계적으로 유의미하게 높은 예측 성능을 보였다.

### Evidence
XGBoost는 5-fold CV에서 평균 AUC 0.87 (95% CI: 0.84-0.90)을 달성한 반면, Random Forest는 AUC 0.82 (95% CI: 0.79-0.85)에 그쳤다. Paired t-test 결과 p = 0.003으로 유의미한 차이가 확인되었다. XGBoost의 우수한 성능은 gradient boosting의 순차적 오류 보정 메커니즘에 기인한 것으로 판단된다.

특히 class imbalance가 심한 endpoint에서 XGBoost의 우위가 두드러졌다 (positive rate < 10%인 경우 AUC 차이 0.08 vs 전체 평균 0.05). Scale_pos_weight 파라미터를 통한 자동 보정이 효과적이었다.

## Q2: Nested CV가 data leakage를 효과적으로 방지하는가?

### Verdict
Nested CV가 기존 단순 CV 대비 과적합을 효과적으로 방지하며, 성능 추정의 낙관적 편향을 제거한다.

### Evidence
단순 5-fold CV에서의 AUC는 0.91이었으나, Nested CV에서는 0.87로 나타났다. 이 0.04의 차이는 단순 CV가 하이퍼파라미터 튜닝 과정에서 발생하는 정보 누수(data leakage)를 반영하지 못함을 의미한다.

Inner loop에서 RandomizedSearchCV (100 iterations)로 최적 파라미터를 선택하고, outer loop에서 이를 독립적으로 평가하는 구조를 통해 테스트 데이터의 순수성을 보장하였다. Outer fold 간 AUC 분산이 0.003으로 안정적이었다.

## Q3: Feature selection이 모델 일반화 성능을 향상시키는가?

### Verdict
Feature selection은 변수 수를 60% 줄이면서 예측 성능을 유지하고, 모델 해석성을 크게 개선한다.

### Evidence
200개 원본 feature에서 Boruta algorithm으로 45개를 선별한 후, SHAP importance 기반으로 최종 18개로 축소하였다. 전체 feature 모델의 AUC 0.87 대비 18개 feature 모델은 AUC 0.86으로 0.01의 미미한 성능 감소만 보였다 (p = 0.42, 통계적으로 유의미하지 않음).

축소된 feature set은 logP, MW, TPSA, HBA, HBD 등 약물화학적으로 해석 가능한 descriptor 위주로 구성되어, 도메인 전문가의 모델 신뢰도를 높이는 부가 효과가 있다.

## Q4: Y-randomization null distribution은 모델 유효성을 검증하는가?

### Verdict
Y-randomization 결과, 모델이 우연이 아닌 실제 structure-activity relationship을 학습하고 있음이 통계적으로 확인되었다.

### Evidence
1,000회의 Y-randomization (라벨 무작위 셔플)을 수행한 결과, randomized 모델의 평균 AUC는 0.51 (95% CI: 0.48-0.54)로 랜덤 수준이었다. 실제 모델의 AUC 0.87은 null distribution의 99.9th percentile (AUC 0.56)을 크게 초과하며, permutation p-value < 0.001이다.

이는 모델이 단순한 통계적 artifact가 아닌 의미 있는 패턴을 학습하고 있음을 강력히 시사한다.
