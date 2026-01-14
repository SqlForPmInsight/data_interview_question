# Kafka Offset 관리 완벽 가이드 ⭐ 중요!

## Offset의 의미

### 개념
- **Partition 내에서 Message의 순차적 위치**
- **0부터 시작하는 연속된 정수**
- **불변성**: 한 번 할당되면 변경되지 않음
- **Partition별 독립**: 각 Partition마다 별도의 Offset 공간

```
Partition 0: [Offset 0] [Offset 1] [Offset 2] [Offset 3] ...
Partition 1: [Offset 0] [Offset 1] [Offset 2] [Offset 3] ...
Partition 2: [Offset 0] [Offset 1] [Offset 2] [Offset 3] ...
```

### Consumer Group과 Offset
```java
// Consumer Group이 읽은 마지막 Offset 저장
// Topic: user-events, Partition: 0, Consumer Group: analytics-group
// Current Offset: 1000 (다음에 읽을 메시지는 Offset 1001)

Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9092");
props.put("group.id", "analytics-group");
props.put("enable.auto.commit", "true");  // 자동 커밋
props.put("auto.commit.interval.ms", "1000");  // 1초마다 커밋
```

## Auto Commit vs Manual Commit

### Auto Commit (자동 커밋)
```java
// 설정
props.put("enable.auto.commit", "true");
props.put("auto.commit.interval.ms", "5000");  // 5초마다

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("user-events"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, String> record : records) {
        // 비즈니스 로직 처리
        processMessage(record.value());
        
        // Offset은 자동으로 커밋됨 (5초마다)
    }
}
```

**장점**:
- 구현 간단
- 개발자가 Offset 관리할 필요 없음

**단점**:
- **At-least-once 보장 어려움**: 처리 중 장애 시 중복 처리
- **정확한 처리 시점과 커밋 불일치**

### Manual Commit (수동 커밋)
```java
// 설정
props.put("enable.auto.commit", "false");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, String> record : records) {
        try {
            // 비즈니스 로직 처리
            processMessage(record.value());
            
            // 성공적으로 처리 후 Offset 커밋
            consumer.commitSync();  // 동기 커밋
            
        } catch (Exception e) {
            // 처리 실패 시 커밋하지 않음
            logger.error("Message processing failed", e);
        }
    }
}
```

**장점**:
- **정확한 처리 보장**: 성공적으로 처리된 경우만 커밋
- **At-least-once 시맨틱 구현 가능**

**단점**:
- 구현 복잡도 증가
- 성능 오버헤드 (커밋 호출 빈도)

## Earliest vs Latest vs Committed

### auto.offset.reset 설정

#### 1. Earliest (처음부터)
```java
props.put("auto.offset.reset", "earliest");

// 새로운 Consumer Group이 Topic을 구독할 때
// 파티션의 첫 번째 메시지(Offset 0)부터 읽기 시작
```

**언제 사용?**
- **데이터 무손실이 중요한 경우**
- **배치 ETL 시스템**
- **금융 데이터 처리**
- **감사 로그 처리**

**실무 예시**:
```java
// 주문 데이터 ETL - 모든 주문이 누락되면 안 됨
Properties orderProcessorProps = new Properties();
orderProcessorProps.put("group.id", "order-etl-processor");
orderProcessorProps.put("auto.offset.reset", "earliest");

// 장애 복구 시에도 누락된 주문 없이 전체 재처리
```

#### 2. Latest (최신 시점부터)
```java
props.put("auto.offset.reset", "latest");

// 새로운 Consumer Group이 구독 시작하는 시점의
// 가장 최신 Offset부터 읽기 시작
```

**언제 사용?**
- **실시간 모니터링**
- **Alert 시스템**
- **PoC/프로토타이핑**
- **과거 데이터보다 최신 데이터가 중요**

#### 3. None
```java
props.put("auto.offset.reset", "none");

// Consumer Group에 저장된 Offset이 없으면 Exception 발생
// 명시적으로 Offset을 관리하고 싶을 때 사용
```

### 실무 사례: Latest 선택과 트레이드오프

#### 프로젝트 상황
```java
// 광고 성과 실시간 분석 시스템
// 요구사항:
// - 실시간 대시보드 업데이트 (< 1분 지연)
// - PoC 단계로 빠른 검증 우선
// - 과거 데이터는 별도 배치로 처리

Properties adAnalyticsProps = new Properties();
adAnalyticsProps.put("group.id", "ad-analytics-realtime");
adAnalyticsProps.put("auto.offset.reset", "latest");  // Latest 선택
```

#### Latest 선택 이유
1. **실시간성 우선**: 과거 데이터보다 현재 성과가 중요
2. **빠른 피드백**: 30분 내 시스템 검증 완료 필요  
3. **인프라 안정성 검증**: 전체 데이터보다 시스템 동작 확인이 목적
4. **보완 체계 존재**: 누락 데이터는 배치 ETL로 보완

#### 장애 시 동작
```java
// 장애 상황: Consumer 애플리케이션이 1시간 다운
// Latest 설정 시:
// - 1시간 동안 생성된 메시지는 유실
// - 복구 후 최신 메시지부터 다시 처리 시작

// 유실 대응 방법:
// 1. Kafka Topic의 retention 기간 확인 (기본 7일)
// 2. 별도 Consumer Group으로 유실 구간 재처리
// 3. 배치 ETL로 일간 데이터 보정
```

### Committed Offset 기반 처리
```java
// Consumer Group이 이전에 커밋한 Offset부터 시작
// auto.offset.reset은 기존 Offset이 없을 때만 사용

// 정상적인 재시작 시나리오
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("events"));

// 이전에 커밋된 Offset: Partition 0 → 1000
// 다음 poll()에서 Offset 1001부터 읽기 시작
ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
```

## 장애 복구 시나리오별 대응

### 시나리오 1: 애플리케이션 장애
```java
// 상황: 메시지 처리 중 애플리케이션 크래시
// Latest 설정 + Auto Commit의 위험성

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, String> record : records) {
        // Auto Commit이 이미 Offset을 커밋한 상태
        
        processMessage(record.value());  // ← 여기서 크래시 발생
        
        // 처리되지 않았지만 Offset은 이미 커밋됨 → 메시지 유실!
    }
}

// 해결책: Manual Commit 사용
for (ConsumerRecord<String, String> record : records) {
    try {
        processMessage(record.value());
        consumer.commitSync();  // 처리 완료 후 커밋
    } catch (Exception e) {
        // 처리 실패 시 Offset 커밋하지 않음 → 재시작 시 재처리
        break;
    }
}
```

### 시나리오 2: Kafka Broker 장애
```java
// Kafka Cluster 일부 Broker 장애
// Producer/Consumer가 자동으로 살아있는 Broker로 연결

Properties producerProps = new Properties();
producerProps.put("retries", "3");  // 3회 재시도
producerProps.put("retry.backoff.ms", "1000");  // 1초 간격
producerProps.put("acks", "all");  // 모든 Replica에서 ACK 대기

// Consumer는 자동으로 새로운 Leader로 reconnect
// Offset은 Kafka 내부에 저장되므로 유실 없음
```

### 시나리오 3: 네트워크 파티션
```java
// Consumer가 Kafka Cluster와 연결이 끊어진 경우
Properties consumerProps = new Properties();
consumerProps.put("session.timeout.ms", "30000");  // 30초
consumerProps.put("heartbeat.interval.ms", "10000");  // 10초
consumerProps.put("max.poll.interval.ms", "300000");  // 5분

// 30초 동안 연결이 끊어지면 Consumer Group에서 제외
// → Rebalancing 발생
// → 다른 Consumer가 해당 Partition 담당
```

## Offset 모니터링 및 관리

### Lag 확인
```bash
# Consumer Lag 확인
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --group analytics-group \
  --describe

# 출력 예시:
# TOPIC     PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG     CONSUMER-ID
# events    0          1000            1500            500     consumer-1
# events    1          800             900             100     consumer-2
# events    2          1200            1200            0       consumer-3
```

### 프로그래밍 방식으로 Lag 모니터링
```java
// Admin Client로 Lag 모니터링
AdminClient adminClient = AdminClient.create(props);

Map<TopicPartition, OffsetAndMetadata> offsets = 
    adminClient.listConsumerGroupOffsets("analytics-group").all().get();

Map<TopicPartition, OffsetSpec> endOffsetRequest = offsets.keySet().stream()
    .collect(Collectors.toMap(tp -> tp, tp -> OffsetSpec.latest()));

Map<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo> endOffsets = 
    adminClient.listOffsets(endOffsetRequest).all().get();

// Lag 계산
for (Map.Entry<TopicPartition, OffsetAndMetadata> entry : offsets.entrySet()) {
    TopicPartition tp = entry.getKey();
    long currentOffset = entry.getValue().offset();
    long endOffset = endOffsets.get(tp).offset();
    long lag = endOffset - currentOffset;
    
    System.out.printf("Partition %s: Lag = %d%n", tp, lag);
}
```

### Offset Reset (수동 재설정)
```bash
# 특정 Offset으로 재설정
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --group analytics-group \
  --topic events \
  --reset-offsets \
  --to-offset 1000 \
  --execute

# 특정 시간으로 재설정
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --group analytics-group \
  --topic events \
  --reset-offsets \
  --to-datetime 2023-12-01T10:00:00.000 \
  --execute

# Earliest로 재설정
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --group analytics-group \
  --topic events \
  --reset-offsets \
  --to-earliest \
  --execute
```

## 면접 단골 질문과 답변

### Q: Kafka Offset을 Latest로 설정하면 장애 시 어떻게 되나요?
**1분 답변**: "Latest는 현재 시점부터 읽기 때문에, 장애 구간 데이터는 유실됩니다. 실제 프로젝트에서 광고 성과 분석 시 PoC 특성상 실시간성을 우선해서 Latest를 선택했습니다. 유실을 막으려면 Earliest로 바꾸고 Manual Commit을 활용해서 마지막 성공 지점부터 다시 읽으면 됩니다. 다만 하위 시스템에서 Idempotency를 보장해야 중복 처리를 막을 수 있습니다."

### Q: Auto Commit vs Manual Commit 차이는?
**답변 포인트**:
1. Auto Commit: 간편하지만 메시지 유실 위험
2. Manual Commit: 복잡하지만 정확한 처리 보장
3. At-least-once가 필요하면 Manual Commit 필수

### Q: Consumer Lag이 계속 증가하면?
**답변 포인트**:
1. Consumer 처리 속도 < Producer 생성 속도
2. Consumer 인스턴스 늘리기 (Partition 수 제한)
3. 처리 로직 최적화 또는 배치 크기 조정

### Q: Offset은 어디에 저장되나요?
**답변 포인트**:
1. Kafka 0.9 이전: Zookeeper
2. Kafka 0.9 이후: Kafka 내부 Topic (__consumer_offsets)
3. 자동 관리되며 Replication으로 고가용성 보장