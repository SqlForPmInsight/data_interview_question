# Kafka 면접 질문 5개 ⭐⭐⭐

## Q1. Kafka Offset을 Latest로 설정하면 장애 시 어떻게 되나요?

**답변 포인트:**
1. 장애 구간 데이터 유실
2. Earliest로 바꾸면 중복 처리
3. Checkpointing으로 해결

**1분 답변:**
"Latest는 현재 시점부터 읽기 때문에, 장애 구간 데이터는 유실됩니다. 실제 프로젝트에서 광고 성과 분석 시 PoC 특성상 실시간성을 우선해서 Latest를 선택했습니다. 유실을 막으려면 Earliest로 바꾸고 Manual Commit을 활용해서 마지막 성공 지점부터 다시 읽으면 됩니다. 다만 하위 시스템에서 Idempotency를 보장해야 중복 처리를 막을 수 있습니다."

**실무 고려사항:**
```java
// Latest 선택 시 트레이드오프
Properties props = new Properties();
props.put("auto.offset.reset", "latest");  // 실시간성 우선
props.put("enable.auto.commit", "false");  // Manual Commit으로 정확성 보완

// 장애 대응 코드
try {
    processMessage(record);
    consumer.commitSync();  // 성공 시에만 커밋
} catch (Exception e) {
    // 처리 실패 시 커밋하지 않음 → 재처리됨
    logger.error("Processing failed: " + record.value(), e);
}
```

---

## Q2. Consumer Group Rebalancing은 언제 발생하나요?

**답변 포인트:**
1. Consumer 추가/제거 시
2. Consumer 장애/응답 없음
3. Partition 수 변경

**1분 답변:**
"세 가지 상황에서 발생합니다. 첫째, Consumer가 추가되거나 제거될 때입니다. 둘째, 기존 Consumer가 heartbeat를 보내지 못해 장애로 간주될 때입니다. 셋째, Topic의 Partition 수가 변경될 때입니다. Rebalancing 중에는 모든 Consumer가 메시지 처리를 중단하므로 실시간 시스템에서는 session timeout과 heartbeat 간격을 최적화해서 빈도를 줄여야 합니다."

**최적화 설정:**
```java
props.put("session.timeout.ms", "30000");      // 30초
props.put("heartbeat.interval.ms", "10000");   // 10초  
props.put("max.poll.interval.ms", "300000");   // 5분
props.put("group.instance.id", "consumer-01"); // Static Membership
```

---

## Q3. Kafka Lag은 어떻게 모니터링하고 대응하나요?

**답변 포인트:**
1. JMX Metrics로 실시간 모니터링
2. Lag 임계값 설정하여 자동 알림
3. Consumer 인스턴스 증가 또는 처리 최적화

**1분 답변:**
"Consumer Lag을 JMX 메트릭으로 실시간 모니터링하고 임계값을 설정해서 자동 알림을 받습니다. Lag이 지속적으로 증가하면 Producer 속도가 Consumer 속도를 앞서는 상황이므로 Consumer 인스턴스를 늘리거나 처리 로직을 최적화합니다. Consumer 수는 최대 Partition 수까지만 늘릴 수 있으므로 필요하면 Topic의 Partition도 확장해야 합니다."

**모니터링 코드:**
```java
// Consumer Lag 확인
AdminClient admin = AdminClient.create(props);
Map<TopicPartition, OffsetAndMetadata> offsets = 
    admin.listConsumerGroupOffsets("analytics-group").all().get();

for (TopicPartition tp : offsets.keySet()) {
    long currentOffset = offsets.get(tp).offset();
    long endOffset = getEndOffset(tp);
    long lag = endOffset - currentOffset;
    
    if (lag > 10000) {  // 1만건 이상 Lag
        sendAlert("High lag detected: " + tp + ", lag: " + lag);
    }
}
```

---

## Q4. At-least-once vs Exactly-once 차이는?

**답변 포인트:**
1. At-least-once: 중복 가능, 유실 없음
2. Exactly-once: 중복과 유실 모두 방지
3. 구현 복잡도와 성능 트레이드오프

**1분 답변:**
"At-least-once는 메시지 유실은 방지하지만 중복 처리가 발생할 수 있습니다. Manual Commit과 재시도 로직으로 구현합니다. Exactly-once는 중복과 유실을 모두 방지하는 가장 강력한 보장이지만 Kafka Transactions을 사용해야 해서 성능 오버헤드가 있습니다. 금융 거래처럼 중복이 절대 허용되지 않으면 Exactly-once, 일반적인 분석 업무는 At-least-once로도 충분합니다."

**구현 방식:**
```java
// At-least-once
props.put("enable.auto.commit", "false");
props.put("acks", "all");
consumer.commitSync();  // 처리 완료 후 커밋

// Exactly-once (Kafka Streams)
props.put("processing.guarantee", "exactly_once");
props.put("isolation.level", "read_committed");
```

---

## Q5. Partition 개수는 어떻게 정하나요?

**답변 포인트:**
1. Consumer 최대 병렬도 결정
2. Producer 처리량 고려
3. 브로커 리소스와 운영 복잡도

**1분 답변:**
"Partition 수는 원하는 Consumer 병렬도와 Producer 처리량을 고려해서 결정합니다. Consumer는 Partition 수만큼만 병렬 처리가 가능하므로 예상되는 최대 Consumer 수를 고려해야 합니다. 또한 각 Producer는 Partition별로 순차 쓰기하므로 Partition이 많을수록 처리량이 증가합니다. 다만 너무 많으면 브로커 메모리 사용량과 Leader Election 시간이 증가하므로 보통 브로커당 수천 개 이하로 유지합니다."

**계산 예시:**
```
Target Throughput: 1M msgs/sec
Producer per partition: 10K msgs/sec
Required partitions: 1M / 10K = 100 partitions

Max consumers: 100 (partition 수와 동일)
Broker count: 3
Partitions per broker: 100 / 3 ≈ 33
```