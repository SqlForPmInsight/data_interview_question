# Partitioning in Spark

## 1. Partition 개념
- 병렬 처리 단위
- executor slot과의 관계

## 2. 파티션 수 결정 원칙
- CPU 코어 * 2~3
- too small vs too large 문제

## 3. Repartition vs Coalesce
- shuffle 유무
- 언제 어떤 것을 선택?

## 4. Custom partitioner
- key-based workload tuning

## 5. 면접 질문
- repartition과 coalesce 차이?
- 적절한 partition 수를 어떻게 정하나?
