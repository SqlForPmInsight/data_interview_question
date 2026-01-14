# Data Skew 대응 전략 ⭐ 중요!

## 문제 상황: 특정 파티션에 데이터가 쏠림

### Data Skew란?
- **정의**: 특정 Key에 데이터가 집중되어 일부 파티션만 과부하
- **증상**: 
  - 전체 Job은 느리지만 대부분 Task는 빠르게 완료
  - Spark UI에서 몇 개 Task만 오래 실행
  - OutOfMemoryError 발생

### 실무 사례
```scala
// 7천만 건 데이터에서 발생한 Skew
val orders = spark.read.parquet("orders/")  // 7천만 건
val result = orders.groupBy("user_id").count()

// 문제: 특정 user_id(bot, 테스트 계정)에 데이터 90% 집중
// 결과: 1개 Task만 90% 처리하고 나머지는 대기
```

## 해결 방법

### 1. Salting 기법

#### 개념
- **Key에 랜덤 값 추가**하여 인위적으로 분산
- **2단계 집계**: Salt별 집계 → 최종 집계

#### 언제 사용?
- 특정 Key에 데이터 90% 이상 집중
- Broadcast Join을 쓸 수 없는 대용량 테이블 조인
- groupBy 연산에서 Skew 발생

#### 코드 예시
```scala
import org.apache.spark.sql.functions._

// 원본 데이터 (Skew 발생)
val skewedData = spark.createDataFrame(Seq(
  ("user1", "action1"), ("user1", "action2"),  // user1에 데이터 집중
  ("user1", "action3"), ("user1", "action4"),
  ("user2", "action5")
)).toDF("user_id", "action")

// 1단계: Salt 추가 (0~9 랜덤 값)
val saltedData = skewedData.withColumn("salt", 
  (rand() * 10).cast("int"))
  .withColumn("salted_key", 
    concat(col("user_id"), lit("_"), col("salt")))

// 2단계: Salt별 집계
val saltedResult = saltedData
  .groupBy("user_id", "salted_key")
  .count()

// 3단계: 최종 집계 (Salt 제거)
val finalResult = saltedResult
  .groupBy("user_id")
  .agg(sum("count").as("total_count"))

finalResult.show()
```

#### 실무 적용 예시
```scala
// 실제 프로덕션 코드
def handleSkewedGroupBy(df: DataFrame, 
                       skewedCol: String, 
                       saltRange: Int = 10): DataFrame = {
  // Skew가 심한 Key 탐지
  val keyStats = df.groupBy(skewedCol).count()
  val threshold = df.count() * 0.1  // 10% 이상이면 Skew
  
  val skewedKeys = keyStats
    .filter(col("count") > threshold)
    .select(skewedCol)
    .collect()
    .map(_.getString(0))
  
  if (skewedKeys.nonEmpty) {
    // Salting 적용
    df.withColumn("salt", 
      when(col(skewedCol).isin(skewedKeys: _*), 
           (rand() * saltRange).cast("int"))
      .otherwise(lit(0)))
      .withColumn("salted_key", 
        concat(col(skewedCol), lit("_"), col("salt")))
  } else {
    df  // Skew 없으면 원본 반환
  }
}
```

### 2. Broadcast Join

#### 개념
- **작은 테이블을 모든 Executor에 복제**
- **Shuffle 완전 제거**로 성능 향상

#### 작은 테이블 기준
- **< 10MB**: 자동으로 Broadcast Join 적용
- **10MB ~ 100MB**: 수동으로 broadcast() 함수 사용
- **> 100MB**: 메모리 부족 위험, 사용 금지

#### 코드 예시
```scala
import org.apache.spark.sql.functions.broadcast

// 대용량 주문 데이터 (7천만 건)
val orders = spark.read.parquet("orders/")

// 소용량 사용자 정보 (1만 명, 약 5MB)
val users = spark.read.parquet("users/")

// 일반 Join (Shuffle 발생)
val normalJoin = orders.join(users, "user_id")

// Broadcast Join (Shuffle 제거)
val broadcastJoin = orders.join(broadcast(users), "user_id")

// 성능 비교
// Normal Join: 5분
// Broadcast Join: 30초
```

#### 자동 Broadcast 설정
```scala
// spark.sql.autoBroadcastJoinThreshold 설정
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "50MB")

// -1로 설정하면 자동 Broadcast 비활성화
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "-1")
```

### 3. 파티션 재분배

#### repartition vs coalesce
```scala
val df = spark.read.parquet("input/")

// repartition: 전체 Shuffle 발생, 파티션을 늘리거나 줄일 때
val repartitioned = df.repartition(100)

// coalesce: 파티션만 줄일 때, Shuffle 최소화
val coalesced = df.coalesce(50)
```

#### 적절한 파티션 개수 계산
```scala
def calculateOptimalPartitions(df: DataFrame, 
                              targetSizePerPartition: Long = 128 * 1024 * 1024): Int = {
  // 128MB per partition이 권장사항
  val totalSize = df.queryExecution.logical.stats.sizeInBytes
  math.max(1, (totalSize / targetSizePerPartition).toInt)
}

val optimalPartitions = calculateOptimalPartitions(df)
val optimizedDf = df.repartition(optimalPartitions)
```

## 실무 사례: 7천만 건 처리 시 스큐 해결

### 문제 상황
```scala
// 이커머스 주문 데이터 처리
val orders = spark.read.parquet("orders_70m/")  // 7천만 건

// 카테고리별 매출 집계 (Skew 발생)
val salesByCategory = orders
  .groupBy("category")
  .agg(sum("amount").as("total_sales"))

// 문제: "Electronics" 카테고리에 50% 데이터 집중
// 실행 시간: 2시간 → 30분으로 단축 필요
```

### 해결 과정

#### 1단계: 데이터 분포 분석
```scala
// 카테고리별 데이터 분포 확인
val categoryStats = orders
  .groupBy("category")
  .count()
  .orderBy(desc("count"))

categoryStats.show()
/*
+------------+----------+
|    category|     count|
+------------+----------+
| Electronics|  35000000|  // 50% 집중!
|     Fashion|  10000000|
|        Home|   8000000|
|       Books|   7000000|
|      Sports|   5000000|
|      Beauty|   5000000|
+------------+----------+
*/
```

#### 2단계: Salting 적용
```scala
// Electronics 카테고리만 Salting
val saltedOrders = orders.withColumn("salt",
  when(col("category") === "Electronics", 
       (rand() * 20).cast("int"))  // 20개로 분할
  .otherwise(lit(0)))
  .withColumn("salted_category",
    when(col("category") === "Electronics",
         concat(col("category"), lit("_"), col("salt")))
    .otherwise(col("category")))

// 1차 집계 (Salt별)
val saltedSales = saltedOrders
  .groupBy("salted_category")
  .agg(sum("amount").as("sales"))

// 2차 집계 (최종)
val finalSales = saltedSales
  .withColumn("category", 
    regexp_replace(col("salted_category"), "_\\d+", ""))
  .groupBy("category")
  .agg(sum("sales").as("total_sales"))
```

#### 3단계: 성능 검증
```scala
// Spark UI에서 확인
// 기존: 1개 Task 2시간, 나머지 Task 5분
// 개선: 모든 Task 6분 내외로 균등하게 분산
```

### 최종 결과
- **실행 시간**: 2시간 → 30분 (75% 단축)
- **리소스 활용률**: 20% → 90% 향상
- **안정성**: OOM 에러 해결

## 면접 단골 질문과 답변

### Q: Data Skew를 어떻게 해결하나요?
**1분 답변**: "세 가지 방법이 있습니다. 첫째, 작은 테이블이면 Broadcast Join으로 Shuffle을 아예 없앱니다. 둘째, Salting으로 Skew된 Key에 랜덤 값을 추가해 인위적으로 분산시킵니다. 셋째, 파티션 수를 늘려서 부하를 분산합니다. 이커머스 프로젝트에서 Electronics 카테고리 Skew를 Salting으로 해결해 처리시간을 75% 단축했습니다."

### Q: Salting은 언제 사용하나요?
**답변 포인트**:
1. 특정 Key에 데이터 50% 이상 집중
2. Broadcast Join을 쓸 수 없는 대용량 테이블
3. groupBy 연산의 성능 병목

### Q: Broadcast Join의 한계는?
**답변 포인트**:
1. 작은 테이블만 가능 (<100MB 권장)
2. 메모리 부족 시 OOM 위험
3. 동적 데이터에는 부적합