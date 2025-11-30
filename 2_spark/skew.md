# Data Skew in Spark

## 1. 스큐란?
- 특정 key가 과도하게 큰 경우
- partition 간 workload imbalance

## 2. 스큐 발생 원인
- 비대칭 key 분포
- one-to-many join

## 3. 해결 전략
- salting
- skew join hint
- skewed key 분리 후 2단계 처리
- broadcast join 활용

## 4. 면접 질문
- data skew가 발생하면 어떤 문제가 생기나?
- 스큐 해결 방법 3가지?
