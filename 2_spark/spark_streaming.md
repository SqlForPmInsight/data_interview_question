# Spark Streaming 실무 가이드 ⭐ 중요!

## Micro-batch 아키텍처

### 개념
- **실시간 스트림을 작은 배치로 분할 처리**
- **기본 배치 간격**: 1초 (설정 가능: 100ms ~ 수십 분)
- **실시간 vs 준실시간**: 진정한 실시간(ms)이 아닌 Near Real-time(초)

### 동작 원리
```scala
import org.apache.spark.streaming._
import org.apache.spark.streaming.kafka010._

val ssc = new StreamingContext(spark.sparkContext, Seconds(1))

// 1초마다 마이크로배치 생성
val stream = KafkaUtils.createDirectStream[String, String](
  ssc,
  PreferConsistent,
  Subscribe[String, String](topics, kafkaParams)
)

// 각 배치를 RDD로 처리
stream.foreachRDD { rdd =>
  if (!rdd.isEmpty()) {
    val df = rdd.toDF()
    df.write.mode("append").parquet("output/")
  }
}
```

### 실시간 vs 준실시간 선택 기준
```scala
// 실시간이 필요한 경우 (< 1초)
// - 이상 거래 탐지
// - 실시간 추천
// → Kafka Streams, Apache Flink 고려

// 준실시간으로 충분한 경우 (1초~1분)
// - 로그 분석
// - 배치 ETL 대체
// → Spark Streaming 적합
```

## Backpressure ⭐ 핵심 개념

### 문제 상황
```scala
// 트래픽 폭증 시나리오
// 평시: 초당 100건
// 이벤트 시: 초당 10,000건 (100배 증가)

val stream = KafkaUtils.createDirectStream(...)
stream.foreachRDD { rdd =>
  // 처리 시간: 5초
  // 배치 간격: 1초
  // → 큐 누적 발생!
}
```

### Backpressure 설정

#### 1. 기본 활성화
```scala
val spark = SparkSession.builder()
  .config("spark.streaming.backpressure.enabled", "true")
  .config("spark.streaming.receiver.maxRate", "1000")  // 초당 최대 1000건
  .config("spark.streaming.kafka.maxRatePerPartition", "100")  // 파티션당 100건
  .getOrCreate()
```

#### 2. 동적 조절
```scala
// 처리 지연 감지하여 자동으로 수신률 조절
spark.conf.set("spark.streaming.backpressure.pid.proportional", "1.0")
spark.conf.set("spark.streaming.backpressure.pid.integral", "0.2")
spark.conf.set("spark.streaming.backpressure.pid.derived", "0.0")
```

### 실무 사례: 트래픽 10배 폭증 대응

#### 상황
```scala
// 광고 클릭 데이터 처리
// 평시: 초당 500건
// 마케팅 캠페인: 초당 5,000건
// 처리 능력: 초당 1,000건 (한계)

val clickStream = KafkaUtils.createDirectStream[String, String](
  ssc,
  PreferConsistent,
  Subscribe[String, String](Array("click-events"), kafkaParams)
)
```

#### 해결책 1: Backpressure + 오토스케일링
```scala
// Dataproc 오토스케일링 설정
val cluster = DataprocCluster.builder()
  .setSecondaryWorkerConfig(
    InstanceGroupConfig.builder()
      .setNumInstances(2)  // 기본 2대
      .setIsPreemptible(true)  // 선점형 인스턴스로 비용 절약
      .setAutoScalingPolicy(
        AutoScalingPolicy.builder()
          .setMaxInstances(10)  // 최대 10대까지 확장
          .build()
      )
      .build()
  )

// Spark 설정
val sparkConf = new SparkConf()
  .set("spark.streaming.backpressure.enabled", "true")
  .set("spark.streaming.kafka.maxRatePerPartition", "200")
  .set("spark.dynamicAllocation.enabled", "true")
  .set("spark.dynamicAllocation.maxExecutors", "50")
```

#### 해결책 2: 지능형 배치 크기 조절
```scala
// 현재 처리 지연 상황에 따라 배치 크기 동적 조절
def adaptiveBatchInterval(processingDelay: Long, 
                         currentBatch: Long): Long = {
  if (processingDelay > currentBatch * 1.5) {
    // 지연이 심하면 배치 간격 늘리기
    math.min(currentBatch * 2, 60000)  // 최대 1분
  } else if (processingDelay < currentBatch * 0.5) {
    // 여유 있으면 배치 간격 줄이기
    math.max(currentBatch / 2, 1000)   // 최소 1초
  } else {
    currentBatch
  }
}
```

## Offset 관리

### Offset 전략 선택

#### 1. Latest (최신 시점부터)
```scala
val kafkaParams = Map[String, Object](
  "bootstrap.servers" -> "kafka-cluster:9092",
  "key.deserializer" -> classOf[StringDeserializer],
  "value.deserializer" -> classOf[StringDeserializer],
  "group.id" -> "spark-streaming-group",
  "auto.offset.reset" -> "latest"  // 최신 데이터부터
)
```

**언제 사용?**
- **실시간 모니터링**: 과거 데이터보다 최신 데이터가 중요
- **PoC/프로토타이핑**: 빠른 검증이 목적
- **Alert 시스템**: 실시간 알림이 우선

#### 2. Earliest (처음부터)
```scala
val kafkaParams = Map[String, Object](
  // ...
  "auto.offset.reset" -> "earliest"  // 처음 데이터부터
)
```

**언제 사용?**
- **데이터 무손실**: 모든 데이터 처리가 필수
- **배치 ETL 대체**: 완전성이 중요
- **금융 데이터**: 누락 시 큰 손실

### Checkpointing

#### 설정 방법
```scala
val ssc = new StreamingContext(spark.sparkContext, Seconds(1))

// Checkpoint 디렉토리 설정
ssc.checkpoint("hdfs://cluster/checkpoint/")

val stream = KafkaUtils.createDirectStream(...)

// 상태 저장이 필요한 연산
val windowedStream = stream
  .window(Seconds(60), Seconds(10))  // 60초 윈도우, 10초마다 갱신
  .checkpoint(Seconds(20))  // 20초마다 체크포인트
```

#### 장애 복구 시나리오
```scala
// 애플리케이션 재시작 시 체크포인트에서 복구
def createStreamingContext(): StreamingContext = {
  val checkpointDir = "hdfs://cluster/checkpoint/"
  
  // 기존 체크포인트가 있으면 복구
  StreamingContext.getOrCreate(checkpointDir, () => {
    val ssc = new StreamingContext(spark.sparkContext, Seconds(1))
    ssc.checkpoint(checkpointDir)
    // 스트리밍 로직 정의
    createStream(ssc)
    ssc
  })
}
```

## 실무 사례: 초당 500건 처리 시스템

### 프로젝트 개요
```scala
// 광고 성과 데이터 실시간 집계
// 요구사항:
// - 초당 500건 처리
// - 1분 단위 집계 결과 생성
// - 장애 시 데이터 무손실
// - 지연시간 < 5초
```

### 아키텍처 설계
```scala
object AdPerformanceStreaming {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .appName("AdPerformanceStreaming")
      .config("spark.streaming.backpressure.enabled", "true")
      .config("spark.streaming.kafka.maxRatePerPartition", "100")
      .config("spark.sql.streaming.checkpointLocation", 
              "gs://bucket/checkpoint/")
      .getOrCreate()

    import spark.implicits._

    // Kafka에서 데이터 읽기
    val clickStream = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "kafka:9092")
      .option("subscribe", "ad-clicks")
      .option("startingOffsets", "latest")  // PoC 특성상 Latest 선택
      .load()
      .selectExpr("CAST(value AS STRING) as json")
      .select(from_json(col("json"), clickSchema).as("data"))
      .select("data.*")

    // 1분 윈도우 집계
    val windowedAggregation = clickStream
      .withWatermark("timestamp", "1 minute")
      .groupBy(
        window(col("timestamp"), "1 minute", "10 seconds"),
        col("campaign_id")
      )
      .agg(
        count("*").as("clicks"),
        sum("cost").as("total_cost"),
        countDistinct("user_id").as("unique_users")
      )

    // BigQuery에 저장
    val query = windowedAggregation.writeStream
      .outputMode("append")
      .format("bigquery")
      .option("table", "analytics.ad_performance_realtime")
      .option("checkpointLocation", "gs://bucket/checkpoint/")
      .trigger(Trigger.ProcessingTime("10 seconds"))
      .start()

    query.awaitTermination()
  }
}
```

### Latest Offset 선택 이유
```scala
// PoC 환경에서 Latest를 선택한 이유:

// 1. 실시간성 우선
// - 과거 데이터보다 현재 성과가 중요
// - 마케터가 실시간 조정 필요

// 2. 인프라 안정성 검증
// - 전체 데이터 처리보다 시스템 안정성 확인이 목적
// - 누락된 데이터는 배치 ETL로 보완

// 3. 빠른 피드백
// - 30분 내 결과 확인 필요
// - Earliest 사용 시 몇 시간 소요 예상
```

### 성능 최적화 결과
```scala
// 최적화 전
// - 처리량: 초당 100건 (목표의 20%)
// - 지연시간: 15초
// - 리소스: 4 코어, 16GB

// 최적화 후  
// - 처리량: 초당 600건 (목표 달성)
// - 지연시간: 3초
// - 리소스: 8 코어, 32GB
// - 비용: 월 50만원 → 80만원 (60% 증가)
// - 안정성: 99.5% 가용률 달성
```

## 면접 단골 질문과 답변

### Q: Spark Streaming에서 갑자기 트래픽이 10배 폭증한다면?
**1분 답변**: "Backpressure로 대응하겠습니다. spark.streaming.backpressure.enabled를 true로 설정하면 시스템이 처리 가능한 만큼만 Kafka에서 데이터를 가져옵니다. maxRatePerPartition으로 파티션당 최대 레코드 수도 제한하고, Dataproc 오토스케일링으로 노드를 동적으로 늘립니다. 실제 프로젝트에서 초당 500건 처리 시 이 설정들로 안정성을 확보했습니다."

### Q: Backpressure란 무엇인가요?
**답변 포인트**:
1. 시스템 처리 능력을 초과하는 데이터 유입 제어
2. 자동으로 수신률을 조절해 큐 누적 방지
3. PID 컨트롤러로 동적 조절

### Q: Latest vs Earliest 차이는?
**답변 포인트**:
1. Latest: 실시간성 우선, 장애 시 데이터 유실
2. Earliest: 완전성 우선, 처리 지연 발생 가능  
3. 비즈니스 요구사항에 따라 선택

### Q: Checkpointing은 왜 필요한가요?
**답변 포인트**:
1. 장애 시 상태 복구를 위한 저장소
2. Window 연산 등 상태 기반 처리에 필수
3. HDFS/S3 등 내결함성 저장소 사용 권장