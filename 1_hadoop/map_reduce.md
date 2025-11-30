# MapReduce

## 1. MapReduce 개요
- Map → Shuffle → Reduce 구조
- parallelism 보장 방식

## 2. Map 단계
- input split
- mapper 실행

## 3. Shuffle 단계
- 가장 비용 높은 단계
- spill, sort, merge 과정

## 4. Reduce 단계
- key 단위 aggregation
- combiner 역할

## 5. MapReduce 한계
- 디스크 기반 처리로 느림
- iterative algorithm 부적합
- Spark와의 차이

## 6. Fault Tolerance
- task 재시도
- speculative execution

## 7. 면접 질문
- Shuffle 단계가 왜 병목인가?
- MapReduce의 단점 3가지
- combiner 역할 설명
