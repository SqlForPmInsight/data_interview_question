# Spark 성능 최적화 완벽 가이드

## Caching/Persist 전략

### 언제 사용해야 하나?

#### 1. 반복 접근하는 데이터
```scala
val expensiveDF = spark.read.parquet("large_table/")
  .filter(col("date") >= "2023-01-01")
  .groupBy("category")
  .agg(sum("amount").as("total"))

// 여러 번 사용할 예정
expensiveDF.cache()  // 메모리에 캐시

// 반복 사용
val result1 = expensiveDF.filter(col("total") > 1000000)
val result2 = expensiveDF.orderBy(desc("total"))
val result3 = expensiveDF.join(otherDF, "category")
```

#### 2. 복잡한 연산의 중간 결과
```scala
val complexTransformation = rawData
  .filter(complexFilter)
  .groupBy("key1", "key2", "key3")
  .agg(
    avg("value1").as("avg_val1"),
    stddev("value2").as("std_val2"),
    percentile_approx(col("value3"), 0.95).as("p95_val3")
  )

// 복잡한 연산 결과를 캐시
complexTransformation.persist(StorageLevel.MEMORY_AND_DISK_SER)

// 이후 다양한 분석에 활용
val analysis1 = complexTransformation.filter(col("avg_val1") > 100)
val analysis2 = complexTransformation.join(dimensionTable, "key1")
```

### 저장 레벨 선택

#### MEMORY_ONLY vs MEMORY_AND_DISK 비교
```scala
import org.apache.spark.storage.StorageLevel

// 1. MEMORY_ONLY - 가장 빠름, 메모리 부족 시 재계산
df.persist(StorageLevel.MEMORY_ONLY)

// 2. MEMORY_AND_DISK - 메모리 부족 시 디스크 사용
df.persist(StorageLevel.MEMORY_AND_DISK)

// 3. MEMORY_AND_DISK_SER - 직렬화로 메모리 절약
df.persist(StorageLevel.MEMORY_AND_DISK_SER)

// 4. DISK_ONLY - 메모리 사용 안 함
df.persist(StorageLevel.DISK_ONLY)
```

#### 실무 선택 기준
```scala
def chooseStorageLevel(dataSize: Long, memoryAvailable: Long): StorageLevel = {
  val ratio = dataSize.toDouble / memoryAvailable
  
  ratio match {
    case r if r < 0.3 => StorageLevel.MEMORY_ONLY
    case r if r < 0.7 => StorageLevel.MEMORY_AND_DISK
    case r if r < 1.0 => StorageLevel.MEMORY_AND_DISK_SER
    case _ => StorageLevel.DISK_ONLY
  }
}

// 사용 예시
val df = spark.read.parquet("input/")
val storageLevel = chooseStorageLevel(
  df.queryExecution.logical.stats.sizeInBytes,
  Runtime.getRuntime.maxMemory()
)
df.persist(storageLevel)
```

### 캐시 해제
```scala
// 사용 완료 후 반드시 해제
df.unpersist()

// 프로그램 종료 시 모든 캐시 해제  
spark.catalog.clearCache()
```

## Broadcast Join 심화

### 자동 Broadcast 임계값 설정
```scala
// 기본값: 10MB
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "50MB")

// 메모리 여유 있을 때 100MB까지 확장
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "100MB")

// 메모리 부족하면 비활성화
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "-1")
```

### 수동 Broadcast 최적화
```scala
// 작은 테이블 (<10MB) - 모든 Executor에 복제 가능
val smallTable = spark.read.parquet("small_table/")
val largeTable = spark.read.parquet("large_table/")

// 명시적 broadcast
val result = largeTable.join(broadcast(smallTable), "key")

// broadcast 효과 확인
result.explain()
/*
== Physical Plan ==
*(2) BroadcastHashJoin [key#1], [key#5], Inner, BuildRight
:- *(2) FileScan parquet [key#1] 
+- BroadcastExchange HashedRelationBroadcastMode(List(input[0, string, true]))
   +- *(1) FileScan parquet [key#5]
*/
```

### Broadcast 메모리 최적화
```scala
// Executor 메모리 중 Broadcast 전용 비율 조정
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.advisoryPartitionSizeInBytes", "128MB")

// Broadcast timeout 설정 (큰 테이블일 때)
spark.conf.set("spark.sql.broadcastTimeout", "600")  // 10분
```

## Partition 튜닝 전략

### 파티션 개수 최적화 공식

#### 기본 공식
```scala
def calculateOptimalPartitions(
    totalDataSize: Long,
    executorMemory: Long,
    numExecutors: Int,
    coresPerExecutor: Int
): Int = {
  
  // 방법 1: 메모리 기반 계산
  val memoryBasedPartitions = math.max(
    totalDataSize / (128 * 1024 * 1024),  // 128MB per partition
    1
  ).toInt
  
  // 방법 2: 코어 기반 계산  
  val coreBasedPartitions = numExecutors * coresPerExecutor * 3
  
  // 방법 3: 데이터 크기 기반
  val sizeBasedPartitions = totalDataSize / (256 * 1024 * 1024) // 256MB per partition
  
  // 최적값 선택 (중간값)
  Array(memoryBasedPartitions, coreBasedPartitions, sizeBasedPartitions.toInt)
    .sorted.apply(1)  // 중간값
}
```

#### 실무 적용 예시
```scala
// 클러스터 정보
val numExecutors = 10
val coresPerExecutor = 4
val executorMemory = 8 * 1024 * 1024 * 1024L  // 8GB

val df = spark.read.parquet("input/")
val dataSize = df.queryExecution.logical.stats.sizeInBytes

val optimalPartitions = calculateOptimalPartitions(
  dataSize, executorMemory, numExecutors, coresPerExecutor
)

val optimizedDF = df.repartition(optimalPartitions)
```

### Dynamic Partition Pruning 활용
```scala
// 파티션 제거로 성능 향상
spark.conf.set("spark.sql.optimizer.dynamicPartitionPruning.enabled", "true")

// 날짜 파티션 테이블
val salesData = spark.read
  .option("basePath", "sales/")
  .parquet("sales/year=*/month=*/day=*/")

// 특정 기간만 읽기 - 불필요한 파티션 제외
val recentSales = salesData.filter(
  col("year") === 2023 && 
  col("month") >= 10
)
```

## Executor 메모리 튜닝

### 메모리 구성 이해
```scala
// Executor 메모리 = Storage + Execution + Reserved + Overhead
// Storage: 60% (캐시, persist용)  
// Execution: 20% (shuffle, join용)
// Reserved: 20% (시스템용)

val totalExecutorMemory = 8 * 1024 * 1024 * 1024L  // 8GB

// 각 영역 크기 계산
val storageMemory = (totalExecutorMemory * 0.6).toLong     // 4.8GB
val executionMemory = (totalExecutorMemory * 0.2).toLong   // 1.6GB
val reservedMemory = (totalExecutorMemory * 0.2).toLong    // 1.6GB
```

### 메모리 비율 조정
```scala
// Join이 많은 작업 - Execution 메모리 늘리기
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")

// 캐시를 많이 사용 - Storage 메모리 늘리기
spark.conf.set("spark.storage.memoryFraction", "0.8")
spark.conf.set("spark.storage.safetyFraction", "0.9")

// Off-heap 메모리 사용 (GC 부담 줄이기)
spark.conf.set("spark.memory.offHeap.enabled", "true")
spark.conf.set("spark.memory.offHeap.size", "2g")
```

## 실무 사례: GCP 마이그레이션 최적화

### 문제 상황
```scala
// 기존 온프레미스 환경
// - 16코어, 64GB 서버 4대
// - 처리시간: 8시간
// - 월 비용: 400만원

// GCP 이전 후 성능 저하
// - n1-standard-4 (4코어, 15GB) 10대  
// - 처리시간: 12시간 (50% 느려짐)
// - 월 비용: 600만원 (50% 증가)
```

### 최적화 과정

#### 1단계: 병목 지점 분석
```scala
// Spark UI에서 확인한 문제점
// 1. Shuffle Read: 500GB (과도한 Shuffle)
// 2. GC Time: 30% (메모리 부족)
// 3. Task Skew: 최대 2시간, 최소 5분 (Data Skew)

// 데이터 분포 분석
val df = spark.read.parquet("input/")
val skewAnalysis = df.groupBy("partition_key")
  .count()
  .orderBy(desc("count"))

skewAnalysis.show()
// 특정 Key에 90% 데이터 집중 확인
```

#### 2단계: 인스턴스 타입 변경
```scala
// 기존: n1-standard-4 (4코어, 15GB) × 10대
// 변경: n1-highmem-2 (2코어, 13GB) × 20대
// 이유: 메모리/코어 비율 개선 (3.75GB/core → 6.5GB/core)

val optimizedConfig = Map(
  "spark.executor.cores" -> "2",
  "spark.executor.memory" -> "10g",  // 13GB - 3GB(OS용)
  "spark.executor.memoryFraction" -> "0.8",
  "spark.sql.adaptive.enabled" -> "true",
  "spark.sql.adaptive.coalescePartitions.enabled" -> "true"
)
```

#### 3단계: Data Skew 해결
```scala
// Salting 적용으로 Skew 해결
val saltedDF = df.withColumn("salt", 
  when(col("partition_key").isin(skewedKeys: _*),
       (rand() * 10).cast("int"))
  .otherwise(lit(0)))
  .withColumn("salted_key",
    concat(col("partition_key"), lit("_"), col("salt")))

// 2단계 집계로 정확한 결과 보장
val stage1 = saltedDF
  .groupBy("salted_key", "category")
  .agg(sum("amount").as("amount"))

val stage2 = stage1
  .withColumn("partition_key", 
    regexp_replace(col("salted_key"), "_\\d+", ""))
  .groupBy("partition_key", "category") 
  .agg(sum("amount").as("total_amount"))
```

#### 4단계: Broadcast Join 도입
```scala
// 차원 테이블들을 Broadcast Join으로 변경
val productDim = spark.read.parquet("product_dim/").cache()  // 50MB
val customerDim = spark.read.parquet("customer_dim/").cache() // 30MB

// 기존: Shuffle Join (5GB 네트워크 트래픽)
val result1 = factTable.join(productDim, "product_id")
                       .join(customerDim, "customer_id")

// 개선: Broadcast Join (네트워크 트래픽 없음)  
val result2 = factTable.join(broadcast(productDim), "product_id")
                       .join(broadcast(customerDim), "customer_id")
```

### 최적화 결과
```scala
// 성능 개선 결과
// 처리시간: 12시간 → 4시간 (67% 단축)
// 월 비용: 600만원 → 350만원 (42% 절감) 
// 안정성: OOM 에러 해결

// 주요 개선 지표
// - Shuffle Read: 500GB → 50GB (90% 감소)
// - GC Time: 30% → 5% (GC 부담 크게 줄음)
// - Task 실행시간: 최대 편차 95% → 20% (균등한 분산)
// - 리소스 사용률: 40% → 85% 향상
```

## Adaptive Query Execution (AQE) 활용

### AQE 핵심 기능
```scala
// AQE 활성화
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true") 
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")

// 런타임에 최적화 적용
// 1. 파티션 병합: 작은 파티션들 합치기
// 2. Join 전략 변경: 실제 크기 보고 Broadcast 결정
// 3. Skew Join 최적화: 자동으로 Skew 탐지 및 해결
```

### Join 전략 자동 최적화
```scala
val largeTable = spark.read.parquet("large_table/")
val unknownTable = spark.read.parquet("unknown_size_table/")

// AQE가 런타임에 크기 측정 후 Join 전략 결정
val result = largeTable.join(unknownTable, "key")

// Spark UI에서 확인 가능:
// "AdaptiveSparkPlan" 노드가 표시되면 AQE 적용됨
```

## 면접 단골 질문과 답변

### Q: Caching은 언제 사용하나요?
**1분 답변**: "두 가지 상황에서 사용합니다. 첫째, 같은 데이터를 여러 번 접근할 때입니다. 복잡한 변환 결과를 cache()로 메모리에 저장하면 재계산을 피할 수 있습니다. 둘째, iterative 알고리즘에서 중간 결과를 저장할 때 사용합니다. 메모리가 충분하면 MEMORY_ONLY, 부족하면 MEMORY_AND_DISK_SER를 선택하며, 사용 후 반드시 unpersist()로 해제합니다."

### Q: Executor 메모리는 어떻게 튜닝하나요?
**답변 포인트**:
1. 메모리/코어 비율을 4-8GB/core로 유지
2. Join이 많으면 execution 영역 확대  
3. Off-heap 메모리로 GC 부담 감소

### Q: 파티션 개수는 어떻게 정하나요?
**답변 포인트**:
1. 코어 수 × 2-3배가 기본
2. 파티션당 128-256MB 권장
3. Spark UI에서 Task 실행시간 모니터링

### Q: Broadcast Join이 실패하면?
**답변 포인트**:
1. 메모리 부족이 주 원인
2. timeout 연장 또는 threshold 조정
3. 대안: Bucketing이나 Sort-Merge Join 고려