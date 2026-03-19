# Decision Brief: 독성 예측 모델 평가

## Executive Summary

3개 실험의 종합 결과, XGBoost 기반 독성 예측 모델을 스크리닝 용도로 채택하는 것을 권장한다.

## Decision Table

| 항목 | 결과 | 판정 |
|------|------|------|
| 최적 모델 | XGBoost | AUC 0.87 (RF: 0.82) |
| 검증 방법 | Nested 5-fold CV | Data leakage 없음 확인 |
| Feature 수 | 45개 → 18개 (60% 감소) | 성능 유지 (AUC 0.86) |
| Y-randomization | p < 0.001 | 모델 유효성 통계적 확인 |
| 외부 검증 | Hold-out test set | AUC 0.84 |

## Key Metrics

- **AUC 0.87**: XGBoost 최적 모델 성능
- **60% 감소**: Feature selection 후 변수 축소율
- **p < 0.001**: Y-randomization 유의성
- **AUC 0.84**: 외부 검증 성능

## Recommendations

1. XGBoost 모델을 1차 스크리닝 도구로 도입
2. 6개월 단위로 새로운 데이터로 모델 재학습
3. 예측 불확실성이 높은 화합물은 추가 실험 수행
