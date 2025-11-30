# Shuffle in Spark

## 1. Shuffle 개념
- partition 재조정이 필요한 연산
- cluster-wide I/O 발생

## 2. Shuffle가 느린 이유
- disk write
- network I/O
- sort/merge 비용

## 3. Shuffle가 발생하는 연산
- groupByKey
- join
- distinct
- sort

## 4. Shuffle 최적화 전략
- reduceByKey 사용
- map-side combine
- broadcast join
- partition 전략 최적화

## 5. 면접 질문
- shuffle이 병목이 되는 이유는?
- groupByKey vs reduceByKey 차이?
