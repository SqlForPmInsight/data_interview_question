# Spark 면접 질문 10개 ⭐⭐⭐

## Q1. 7천만 건 데이터 처리 시 특정 파티션에 부하가 쏠린다면?

**답변 포인트:**
1. Data Skew 문제 인식
2. Salting 또는 Broadcast Join 적용
3. 파티션 개수 재조정

**1분 답변:**
"Data Skew 문제입니다. 세 가지로 해결하겠습니다. 첫째, 작은 테이블이면 Broadcast Join으로 Shuffle을 아예 없앱니다. 둘째, Salting 기법으로 Skew된 Key에 랜덤 값을 추가해 파티션을 고르게 분산시킵니다. 셋째, Spark UI로 각 Task 실행 시간을 보며 파티션 개수를 늘립니다. 이커머스 프로젝트에서 7천만 건 처리 시 이 방법들로 실행시간을 75% 단축했습니다."

**코드 예시:**
```scala
// Salting 적용
val saltedData = skewedDF.withColumn("salt", 
  when(col("category") === "Electronics", 
       (rand() * 20).cast("int"))
  .otherwise(lit(0)))
  .withColumn("salted_key",
    concat(col("category"), lit("_"), col("salt")))

// 1차 집계 → 2차 집계
val result = saltedData
  .groupBy("salted_key")
  .agg(sum("amount"))
  .groupBy("category")
  .agg(sum("amount"))
```

---

## Q2. Spark Streaming에서 갑자기 트래픽이 10배 폭증한다면?

**답변 포인트:**
1. Backpressure 활성화
2. maxRatePerPartition 설정
3. 오토스케일링 고려

**1분 답변:**
"Backpressure로 대응하겠습니다. spark.streaming.backpressure.enabled를 true로 설정하면 시스템이 처리 가능한 만큼만 Kafka에서 데이터를 가져옵니다. maxRatePerPartition으로 파티션당 최대 레코드 수도 제한하고, Dataproc 오토스케일링으로 노드를 동적으로 늘립니다. 실제 프로젝트에서 초당 500건 처리 시 이 설정들로 안정성을 확보했습니다."

**설정 예시:**
```scala
val sparkConf = new SparkConf()
  .set("spark.streaming.backpressure.enabled", "true")
  .set("spark.streaming.kafka.maxRatePerPartition", "100")
  .set("spark.dynamicAllocation.enabled", "true")
  .set("spark.dynamicAllocation.maxExecutors", "50")
```

---

## Q3. Shuffle이 왜 느린가요? 어떻게 줄이나요?

**답변 포인트:**
1. 네트워크 I/O + 디스크 쓰기
2. Broadcast Join 활용
3. 불필요한 Shuffle 제거

**1분 답변:**
"Shuffle은 네트워크 I/O와 디스크 쓰기가 동시에 발생해서 느립니다. 줄이는 방법은 세 가지입니다. 첫째, 작은 테이블(<10MB)은 Broadcast Join으로 각 Executor에 복제해서 Shuffle을 없앱니다. 둘째, 파티션 Key를 미리 맞춰서 불필요한 재분배를 방지합니다. 셋째, 중간 집계를 먼저 하고 최종 집계는 나중에 합니다. GCP 프로젝트에서 이 방법들로 성능을 크게 개선했습니다."

**최적화 예시:**
```scala
// Broadcast Join으로 Shuffle 제거
val result = largeDf.join(broadcast(smallDf), "key")

// Pre-aggregation으로 Shuffle 데이터 감소
val preAgg = df.groupBy("key").agg(sum("amount"))
val finalResult = preAgg.join(otherDf, "key")
```

---

## Q4. repartition vs coalesce 차이는?

**답변 포인트:**
1. repartition: 전체 Shuffle 발생, 증가/감소 모두 가능
2. coalesce: 감소만 가능, Shuffle 최소화
3. 파티션 크기와 성능 영향

**1분 답변:**
"repartition은 전체 Shuffle이 발생하지만 파티션을 늘리거나 줄일 수 있고, coalesce는 파티션을 줄일 때만 사용하며 Shuffle을 최소화합니다. 많은 작은 파일을 처리한 후 큰 파일로 저장할 때 coalesce를 사용하고, 병렬성을 높이려면 repartition을 사용합니다. 파티션당 128-256MB가 적정 크기입니다."

---

## Q5. Caching은 언제 사용하나요?

**답변 포인트:**
1. 반복 접근하는 데이터
2. 복잡한 연산의 중간 결과
3. 메모리 사용량 고려

**1분 답변:**
"두 가지 상황에서 사용합니다. 첫째, 같은 데이터를 여러 번 접근할 때입니다. 복잡한 변환 결과를 cache()로 메모리에 저장하면 재계산을 피할 수 있습니다. 둘째, iterative 알고리즘에서 중간 결과를 저장할 때 사용합니다. 메모리가 충분하면 MEMORY_ONLY, 부족하면 MEMORY_AND_DISK_SER를 선택하며, 사용 후 반드시 unpersist()로 해제합니다."

---

## Q6. Executor 메모리는 어떻게 튜닝하나요?

**답변 포인트:**
1. 메모리/코어 비율 최적화
2. Storage vs Execution 메모리 조정
3. GC 튜닝

**1분 답변:**
"메모리/코어 비율을 4-8GB/core로 유지하는 것이 기본입니다. Join이 많으면 execution 메모리 비중을 높이고, 캐시를 많이 사용하면 storage 메모리를 늘립니다. Off-heap 메모리를 활용해 GC 부담을 줄이고, 메모리가 부족하면 파티션 수를 늘려서 Task당 처리 데이터를 줄입니다."

---

## Q7. Spark UI에서 어떤 지표를 주로 보나요?

**답변 포인트:**
1. Task 실행시간 편차 (Data Skew 탐지)
2. Shuffle Read/Write 크기
3. GC Time과 Memory 사용률

**1분 답변:**
"세 가지를 중점적으로 봅니다. 첫째, Task 실행시간 편차로 Data Skew를 탐지합니다. 일부 Task만 오래 걸리면 Skew 문제입니다. 둘째, Shuffle Read/Write 크기로 불필요한 Shuffle을 확인합니다. 셋째, GC Time이 전체 시간의 10%를 넘으면 메모리 튜닝이 필요합니다. 이 지표들로 병목지점을 찾아 최적화합니다."

---

## Q8. OOM이 발생하면 어떻게 대응하나요?

**답변 포인트:**
1. Executor 메모리 증가
2. 파티션 수 증가
3. 데이터 Skew 해결

**1분 답변:**
"세 단계로 접근합니다. 첫째, executor-memory를 늘려서 Task당 사용 가능한 메모리를 확보합니다. 둘째, 파티션 수를 늘려서 각 Task가 처리하는 데이터량을 줄입니다. 셋째, Spark UI에서 특정 Task만 오래 걸리면 Data Skew 문제이므로 Salting이나 Broadcast Join으로 해결합니다. 근본적으로는 불필요한 컬럼 제거와 필터 조건을 먼저 적용합니다."

---

## Q9. 파티션 개수는 어떻게 정하나요?

**답변 포인트:**
1. 코어 수의 2-3배
2. 파티션당 데이터 크기
3. 클러스터 리소스 고려

**1분 답변:**
"기본 공식은 코어 수의 2-3배입니다. 예를 들어 10대 서버에 각각 4코어면 40 × 2 = 80개 파티션을 시작점으로 잡습니다. 파티션당 128-256MB가 적정 크기이므로 전체 데이터 크기를 고려해서 조정합니다. 너무 많으면 Task 오버헤드가 증가하고, 너무 적으면 병렬성이 떨어지므로 Spark UI에서 실제 실행시간을 보며 튜닝합니다."

---

## Q10. Dynamic Allocation은 언제 사용하나요?

**답변 포인트:**
1. 워크로드가 가변적인 경우
2. 리소스 효율성 향상
3. 클라우드 환경에서 비용 절약

**1분 답변:**
"워크로드가 시간대별로 다를 때 사용합니다. 야간 배치는 많은 Executor가 필요하지만 완료 후에는 불필요하므로 Dynamic Allocation으로 자동 조절합니다. minExecutors는 항상 유지할 최소 개수, maxExecutors는 최대 확장 한계를 설정합니다. 클라우드에서는 Preemptible 인스턴스와 조합해서 비용을 크게 절약할 수 있습니다. 다만 Executor 시작 시간이 있으므로 매우 짧은 Job에는 부적합합니다."