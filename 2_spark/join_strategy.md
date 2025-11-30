# Join Optimization in Spark

## 1. Join 발생 시 shuffle 구조
- shuffle read/write 과정
- wide transformation

## 2. Broadcast Join
- 조건
- 장점 및 한계

## 3. Sort-Merge Join vs Shuffle-Hash Join
- 내부 동작 비교

## 4. Large table join 최적화
- 사전 filter
- partition pruning
- salting 기법

## 5. 면접 질문
- broadcast join의 조건?
- spark에서 join 성능이 나쁜 이유?
