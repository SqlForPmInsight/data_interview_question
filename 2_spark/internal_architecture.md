# Spark 내부 동작 원리

## Driver vs Executor

### Driver 역할
- **SparkContext 생성**: 전체 애플리케이션의 진입점
- **Job 스케줄링**: 사용자 코드를 Job → Stage → Task로 분해
- **메타데이터 관리**: 클러스터 상태, 실행 계획 추적
- **결과 수집**: Executor들의 결과를 모아서 최종 결과 생성

### Executor 역할  
- **Task 실행**: Driver가 보낸 Task를 실제로 실행
- **데이터 저장**: 메모리/디스크에 RDD 파티션 캐시
- **Shuffle 데이터 제공**: 다른 Executor가 요청하는 Shuffle 데이터 서빙
- **상태 보고**: 실행 진행상황을 Driver에게 보고

## Job → Stage → Task 실행 흐름

### 1단계: Job 생성
```scala
val df = spark.read.parquet("input/")
val result = df.groupBy("category").count()  // Action → Job 생성
result.collect()
```

### 2단계: Stage 분할
- **Stage 분할 기준**: Shuffle 발생 지점
- **Narrow Transformation**: 같은 Stage 내에서 처리 (map, filter)
- **Wide Transformation**: Stage 경계 생성 (groupBy, join)

```
Stage 0: parquet 읽기 + map/filter
   ↓ (Shuffle)
Stage 1: groupBy + count
```

### 3단계: Task 실행
- **파티션 수 = Task 수**
- **각 Task는 하나의 파티션 처리**
- **병렬 실행**: Executor 코어 수만큼 동시 실행

## Shuffle 개념과 성능 영향

### Shuffle이 발생하는 경우
1. **groupBy**, **reduceByKey** - Key별 그룹핑
2. **join** - 테이블 조인
3. **repartition** - 파티션 재분배
4. **sort** - 정렬

### 성능 영향 요소

#### 1. 네트워크 I/O
```scala
// 문제가 되는 코드
df.groupBy("user_id").count()  // user_id별 데이터가 다른 노드로 이동
```

#### 2. 디스크 쓰기
- Shuffle 중간 결과를 디스크에 임시 저장
- 메모리 부족시 더 많은 디스크 I/O 발생

#### 3. 메모리 부족 시 OOM
```scala
// OOM 위험 코드
df.groupBy("category")
  .agg(collect_list("large_text"))  // 특정 카테고리에 데이터 집중
```

### Shuffle 최적화 방법

#### 1. Broadcast Join 사용
```scala
// 작은 테이블(<10MB)을 모든 Executor에 복제
import org.apache.spark.sql.functions.broadcast

val largeDf = spark.read.parquet("large_table/")
val smallDf = spark.read.parquet("small_table/")

// Shuffle 발생
largeDf.join(smallDf, "key")

// Shuffle 제거
largeDf.join(broadcast(smallDf), "key")
```

#### 2. Pre-partitioning
```scala
// 파티션 Key를 미리 맞춰서 Shuffle 최소화
val df1 = spark.read.parquet("table1/").partitionBy("user_id")
val df2 = spark.read.parquet("table2/").partitionBy("user_id")

df1.join(df2, "user_id")  // Shuffle 최소화됨
```

## Partition과 Parallelism

### 파티션 개수 결정 기준

#### 기본 공식
```
적정 파티션 수 = Executor 수 × 코어 수 × 2~3배
```

#### 실무 예시
```scala
// 클러스터 정보: 5대 서버, 각각 4코어
// 적정 파티션 수 = 5 × 4 × 2 = 40개

val df = spark.read.parquet("input/")
println(s"현재 파티션 수: ${df.rdd.getNumPartitions}")

// 파티션 수 조정
val optimizedDf = df.coalesce(40)
```

### 파티션이 너무 많으면?
- **문제점**: Task 시작 오버헤드 증가
- **증상**: 작은 파일들이 많이 생성
- **해결책**: `coalesce()`로 파티션 수 줄이기

```scala
// 1000개 파티션 → 40개로 줄이기
df.coalesce(40).write.parquet("output/")
```

### 파티션이 너무 적으면?
- **문제점**: 병렬처리 효과 감소, 메모리 부족
- **증상**: 일부 Task만 실행되고 나머지는 대기
- **해결책**: `repartition()`으로 파티션 수 늘리기

```scala
// 5개 파티션 → 40개로 늘리기
df.repartition(40).write.parquet("output/")
```

## 면접 필수 포인트

### Q: Shuffle이 왜 느린가요?
**답변**: "Shuffle은 네트워크 I/O와 디스크 쓰기가 동시에 발생하기 때문입니다. 데이터를 네트워크로 전송하기 전에 디스크에 임시 저장하고, 받는 쪽에서도 정렬/그룹핑을 위해 추가 처리가 필요합니다."

### Q: 파티션 개수는 어떻게 정하나요?
**답변**: "코어 수의 2-3배로 설정합니다. 너무 많으면 Task 오버헤드가 증가하고, 너무 적으면 병렬성이 떨어집니다. 실무에서는 Spark UI에서 Task 실행 시간을 보며 조정합니다."

### Q: Driver vs Executor 차이는?
**답변**: "Driver는 전체 애플리케이션을 조율하는 마스터이고, Executor는 실제 작업을 수행하는 워커입니다. Driver가 Job을 Stage와 Task로 나누면 Executor들이 병렬로 실행합니다."