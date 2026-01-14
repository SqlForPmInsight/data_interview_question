# Kafka Consumer Group Rebalancing & Backpressure

## Consumer Group Rebalancing ⭐ 중요!

### Rebalancing이 발생하는 경우

#### 1. Consumer 추가/제거
```java
// 기존: 3개 Partition, 2개 Consumer
// Consumer 1: Partition 0, 1
// Consumer 2: Partition 2

// 새로운 Consumer 3 추가
// → Rebalancing 발생
// Consumer 1: Partition 0
// Consumer 2: Partition 1  
// Consumer 3: Partition 2

Properties props = new Properties();
props.put("group.id", "analytics-group");
props.put("session.timeout.ms", "30000");  // 30초
props.put("heartbeat.interval.ms", "10000");  // 10초

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("events"));

// Consumer 시작 시 Rebalancing 발생
```

#### 2. Consumer 장애/응답 없음
```java
// Consumer가 heartbeat를 보내지 않으면 Rebalancing
// session.timeout.ms 시간 내에 heartbeat 없으면 죽은 것으로 간주

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, String> record : records) {
        // 처리 시간이 max.poll.interval.ms를 초과하면
        // Consumer가 Group에서 제외됨
        longRunningProcess(record);  // 5분 소요
    }
}

// 해결책: max.poll.interval.ms 증가 또는 배치 크기 감소
props.put("max.poll.interval.ms", "600000");  // 10분으로 증가
props.put("max.poll.records", "100");  // 한 번에 100개만 처리
```

#### 3. Topic Partition 수 변경
```bash
# Topic의 Partition 수 증가
kafka-topics.sh --bootstrap-server kafka:9092 \
  --topic events \
  --alter \
  --partitions 6

# 기존: 3 Partition → 6 Partition으로 증가
# 모든 Consumer Group에서 Rebalancing 발생
```

### Rebalancing 과정

#### 1단계: Group Coordinator 선택
```java
// Kafka Cluster에서 Consumer Group을 관리할 Broker 선택
// hash(group.id) % coordinator_broker_count

// Group ID "analytics-group"의 Coordinator: Broker 1
// 모든 Rebalancing 과정을 Broker 1이 조율
```

#### 2단계: JoinGroup 요청
```java
// 모든 Consumer가 Group Coordinator에게 JoinGroup 요청
// 이 시점에서 모든 Consumer는 일시적으로 메시지 처리 중단

// Consumer 1: JoinGroup 요청
// Consumer 2: JoinGroup 요청  
// Consumer 3: JoinGroup 요청 (새로 추가된 Consumer)
```

#### 3단계: Leader Consumer 선택
```java
// Group Coordinator가 첫 번째로 JoinGroup 요청한 Consumer를 Leader로 지정
// Leader Consumer는 Partition Assignment 계산

// Leader Consumer (Consumer 1)가 수행하는 작업:
// - 모든 Consumer 목록 확인
// - Topic의 Partition 목록 확인
// - Assignment Strategy에 따라 분배 계획 수립
```

#### 4단계: SyncGroup으로 분배 결과 전달
```java
// Leader Consumer가 계산한 Assignment를 Group Coordinator에게 전달
// Group Coordinator가 모든 Consumer에게 각자의 Assignment 전달

// 최종 Assignment:
// Consumer 1: [events-0]
// Consumer 2: [events-1]
// Consumer 3: [events-2]
```

### 성능 영향

#### Rebalancing 중 처리 중단
```java
// Rebalancing 시작 시점: 09:00:00
// 모든 Consumer가 메시지 처리 중단

while (true) {
    try {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
        
        if (records.isEmpty()) {
            // Rebalancing 중이면 records가 비어있음
            continue;
        }
        
        // 정상 처리
        for (ConsumerRecord<String, String> record : records) {
            processMessage(record);
        }
        
    } catch (CommitFailedException e) {
        // Rebalancing으로 인한 Offset Commit 실패
        logger.warn("Commit failed due to rebalancing", e);
    }
}

// Rebalancing 완료 시점: 09:00:05 (5초 소요)
// → 5초 동안 실시간 처리 중단
```

#### 처리량 감소
```java
// Rebalancing 빈도와 성능 영향
// 평상시: 초당 1000건 처리
// Rebalancing 발생 시: 초당 0건 (5초간)
// → 시간당 평균 처리량 1.4% 감소 (5초/3600초)

// 모니터링 지표
Map<String, Object> metrics = consumer.metrics();
double rebalanceRate = (Double) metrics.get("rebalance-rate-per-hour");
double assignmentLatency = (Double) metrics.get("last-rebalance-seconds-ago");
```

### Rebalancing 최소화 방법

#### 1. Consumer 설정 최적화
```java
Properties props = new Properties();

// Heartbeat 간격 단축 (빠른 장애 감지)
props.put("heartbeat.interval.ms", "3000");  // 3초

// Session timeout 적절히 설정
props.put("session.timeout.ms", "30000");  // 30초

// Poll 간격 여유 있게 설정
props.put("max.poll.interval.ms", "300000");  // 5분

// 한 번에 가져올 메시지 수 제한
props.put("max.poll.records", "500");
```

#### 2. Static Membership 사용 (Kafka 2.3+)
```java
// Consumer에 고유 ID 할당으로 Rebalancing 최소화
props.put("group.instance.id", "consumer-server-01");

// Consumer 재시작 시에도 동일한 Partition 유지
// session.timeout.ms 내에 재시작하면 Rebalancing 없음
```

#### 3. Graceful Shutdown
```java
// 애플리케이션 종료 시 깨끗하게 Consumer 해제
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    logger.info("Shutting down consumer gracefully");
    consumer.wakeup();  // poll() 블록 해제
}));

try {
    while (running) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
        processRecords(records);
    }
} catch (WakeupException e) {
    // 정상적인 shutdown
} finally {
    consumer.close();  // Group에서 깨끗하게 제거
}
```

## Backpressure 대응 전략

### Consumer가 느릴 때 문제점

#### 1. Lag 누적
```java
// Producer: 초당 1000건 생성
// Consumer: 초당 500건 처리
// → 초당 500건씩 Lag 누적

// Lag 모니터링
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    
    long startTime = System.currentTimeMillis();
    processRecords(records);
    long processingTime = System.currentTimeMillis() - startTime;
    
    // 처리 시간이 1초를 초과하면 Lag 누적
    if (processingTime > 1000) {
        logger.warn("Processing time exceeded: {}ms", processingTime);
    }
}
```

#### 2. Memory 부족
```java
// 메시지가 Consumer 내부 큐에 쌓임
// fetch.min.bytes, fetch.max.wait.ms 설정에 따라 큐 크기 결정

props.put("fetch.min.bytes", "1024");  // 최소 1KB
props.put("fetch.max.wait.ms", "500");  // 최대 0.5초 대기
props.put("max.partition.fetch.bytes", "1048576");  // 파티션당 1MB

// 큐가 가득 차면 OutOfMemoryError 발생
```

### Lag 모니터링 시스템

#### JMX Metrics 활용
```java
// Consumer Lag JMX Metric 수집
MBeanServer server = ManagementFactory.getPlatformMBeanServer();
ObjectName objectName = new ObjectName(
    "kafka.consumer:type=consumer-fetch-manager-metrics,client-id=*"
);

Set<ObjectInstance> beans = server.queryMBeans(objectName, null);
for (ObjectInstance bean : beans) {
    ObjectName name = bean.getObjectName();
    
    // Records Lag Max - 가장 큰 Lag
    Long recordsLagMax = (Long) server.getAttribute(name, "records-lag-max");
    
    // Bytes Lag - 처리해야 할 데이터 크기
    Long bytesLag = (Long) server.getAttribute(name, "bytes-lag");
    
    logger.info("Client: {}, Records Lag: {}, Bytes Lag: {}", 
                name.getKeyProperty("client-id"), recordsLagMax, bytesLag);
}
```

#### 자동 Lag 알림
```java
public class LagMonitor {
    private static final long LAG_THRESHOLD = 10000;  // 10K 메시지
    private final ScheduledExecutorService scheduler;
    
    public void startMonitoring() {
        scheduler.scheduleAtFixedRate(() -> {
            Map<TopicPartition, Long> lags = getCurrentLags();
            
            for (Map.Entry<TopicPartition, Long> entry : lags.entrySet()) {
                if (entry.getValue() > LAG_THRESHOLD) {
                    sendAlert(entry.getKey(), entry.getValue());
                }
            }
        }, 0, 30, TimeUnit.SECONDS);  // 30초마다 확인
    }
    
    private void sendAlert(TopicPartition partition, long lag) {
        String message = String.format(
            "High lag detected: Partition %s has %d messages behind", 
            partition, lag
        );
        
        // Slack, Email 등으로 알림
        alertService.send(message);
    }
}
```

### maxPollRecords 조정

#### 배치 크기 최적화
```java
// 처리 성능에 맞춰 배치 크기 동적 조정
public class AdaptiveConsumer {
    private int currentMaxPollRecords = 500;
    private double targetProcessingTime = 1000;  // 1초 목표
    
    public void consume() {
        props.put("max.poll.records", currentMaxPollRecords);
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        
        while (true) {
            long startTime = System.currentTimeMillis();
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            
            processRecords(records);
            
            long processingTime = System.currentTimeMillis() - startTime;
            
            // 처리 시간에 따라 배치 크기 조정
            adjustMaxPollRecords(processingTime, records.count());
            
            consumer.commitSync();
        }
    }
    
    private void adjustMaxPollRecords(long processingTime, int recordCount) {
        if (processingTime > targetProcessingTime * 1.2) {
            // 너무 느리면 배치 크기 감소
            currentMaxPollRecords = Math.max(100, currentMaxPollRecords - 50);
        } else if (processingTime < targetProcessingTime * 0.8) {
            // 여유 있으면 배치 크기 증가
            currentMaxPollRecords = Math.min(1000, currentMaxPollRecords + 50);
        }
        
        logger.info("Adjusted max.poll.records to {}, processing time: {}ms, records: {}", 
                   currentMaxPollRecords, processingTime, recordCount);
    }
}
```

### Consumer 스케일링

#### Horizontal Scaling
```java
// Consumer 인스턴스 수 = Topic Partition 수가 최대
// Topic: events, Partitions: 6
// 최대 Consumer 수: 6개

// 동적 Consumer 생성
public class ConsumerManager {
    private final List<KafkaConsumer<String, String>> consumers = new ArrayList<>();
    private final ExecutorService executorService;
    
    public void scaleUp(int additionalConsumers) {
        for (int i = 0; i < additionalConsumers; i++) {
            KafkaConsumer<String, String> consumer = createConsumer();
            consumers.add(consumer);
            
            // 별도 스레드에서 실행
            executorService.submit(() -> {
                try {
                    while (!Thread.currentThread().isInterrupted()) {
                        ConsumerRecords<String, String> records = 
                            consumer.poll(Duration.ofMillis(1000));
                        processRecords(records);
                        consumer.commitSync();
                    }
                } finally {
                    consumer.close();
                }
            });
        }
        
        logger.info("Scaled up to {} consumers", consumers.size());
    }
    
    public void scaleDown(int consumersToRemove) {
        for (int i = 0; i < consumersToRemove && !consumers.isEmpty(); i++) {
            KafkaConsumer<String, String> consumer = consumers.remove(0);
            consumer.wakeup();  // Graceful shutdown
        }
        
        logger.info("Scaled down to {} consumers", consumers.size());
    }
}
```

#### Vertical Scaling
```java
// 단일 Consumer의 처리 성능 향상
public class ParallelMessageProcessor {
    private final ExecutorService processingPool;
    
    public void processRecordsInParallel(ConsumerRecords<String, String> records) {
        Map<TopicPartition, List<ConsumerRecord<String, String>>> partitionedRecords = 
            partitionRecords(records);
        
        List<CompletableFuture<Void>> futures = new ArrayList<>();
        
        // Partition별로 병렬 처리 (순서 보장)
        for (Map.Entry<TopicPartition, List<ConsumerRecord<String, String>>> entry : 
             partitionedRecords.entrySet()) {
            
            CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
                for (ConsumerRecord<String, String> record : entry.getValue()) {
                    processMessage(record);
                }
            }, processingPool);
            
            futures.add(future);
        }
        
        // 모든 Partition 처리 완료까지 대기
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
    }
}
```

## 실무 사례: 대용량 로그 처리

### 문제 상황
```java
// 웹 서비스 로그 실시간 처리
// - Topic: web-logs, Partitions: 12
// - Producer: 초당 5000건 생성  
// - Consumer: 초당 1000건 처리 (5배 느림)
// - Lag: 시간당 14,400,000건 누적

// 기존 Consumer 구성
// - Consumer 인스턴스: 4개
// - 처리 방식: 동기 처리
// - 배치 크기: 500개
```

### 해결 과정

#### 1단계: Consumer 수 증가
```java
// Consumer 수를 Partition 수에 맞춰 증가
// 4개 → 12개 (Partition 수와 동일)

// 예상 효과: 초당 3000건 처리 (3배 향상)
// 실제 효과: 초당 2400건 처리 (2.4배 향상)
// → 여전히 부족 (초당 2600건 누적)
```

#### 2단계: 배치 크기 최적화
```java
// max.poll.records를 500 → 1000으로 증가
props.put("max.poll.records", "1000");

// fetch.min.bytes 증가로 네트워크 효율성 향상
props.put("fetch.min.bytes", "65536");  // 64KB

// 예상 효과: 네트워크 라운드트립 50% 감소
// 실제 효과: 초당 3200건 처리
// → 성능 목표 달성 (초당 1800건 Lag 감소)
```

#### 3단계: 병렬 처리 도입
```java
public class OptimizedLogConsumer {
    private final int THREAD_POOL_SIZE = 8;
    private final ExecutorService processingPool = 
        Executors.newFixedThreadPool(THREAD_POOL_SIZE);
    
    public void consume() {
        while (true) {
            ConsumerRecords<String, String> records = 
                consumer.poll(Duration.ofMillis(1000));
            
            if (records.isEmpty()) continue;
            
            // Partition 내 순서는 보장하면서 병렬 처리
            processPartitionsInParallel(records);
            
            consumer.commitSync();
        }
    }
    
    private void processPartitionsInParallel(ConsumerRecords<String, String> records) {
        Map<Integer, List<ConsumerRecord<String, String>>> partitionGroups = 
            records.records("web-logs").stream()
                   .collect(Collectors.groupingBy(r -> r.partition()));
        
        List<CompletableFuture<Void>> futures = partitionGroups.entrySet().stream()
            .map(entry -> CompletableFuture.runAsync(() -> {
                for (ConsumerRecord<String, String> record : entry.getValue()) {
                    processLogMessage(record);
                }
            }, processingPool))
            .collect(Collectors.toList());
        
        // 모든 Partition 처리 완료 대기
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
    }
}

// 최종 성능: 초당 8000건 처리
// Lag 해소: 2일 만에 누적 Lag 완전 제거
```

### 모니터링 및 알림 시스템
```java
public class KafkaLagAlertSystem {
    private static final long CRITICAL_LAG = 100000;  // 10만건
    private static final long WARNING_LAG = 50000;    // 5만건
    
    @Scheduled(fixedRate = 60000)  // 1분마다 실행
    public void checkLagAndAlert() {
        Map<TopicPartition, Long> currentLags = lagMonitor.getCurrentLags();
        
        for (Map.Entry<TopicPartition, Long> entry : currentLags.entrySet()) {
            TopicPartition tp = entry.getKey();
            Long lag = entry.getValue();
            
            if (lag > CRITICAL_LAG) {
                sendCriticalAlert(tp, lag);
                // 자동 스케일링 트리거
                consumerManager.scaleUp(2);
                
            } else if (lag > WARNING_LAG) {
                sendWarningAlert(tp, lag);
            }
        }
    }
}
```

## 면접 단골 질문과 답변

### Q: Consumer Group Rebalancing은 언제 발생하나요?
**1분 답변**: "세 가지 상황에서 발생합니다. 첫째, Consumer가 추가되거나 제거될 때입니다. 둘째, 기존 Consumer가 heartbeat를 보내지 못해 장애로 간주될 때입니다. 셋째, Topic의 Partition 수가 변경될 때입니다. Rebalancing 중에는 모든 Consumer가 메시지 처리를 중단하므로 실시간 시스템에서는 session timeout과 heartbeat 간격을 최적화해서 빈도를 줄여야 합니다."

### Q: Kafka Lag은 어떻게 모니터링하고 대응하나요?
**답변 포인트**:
1. JMX Metrics로 실시간 모니터링
2. Lag 임계값 설정하여 자동 알림
3. Consumer 인스턴스 증가 또는 처리 최적화

### Q: Consumer가 Producer보다 느리면 어떻게 해결하나요?
**답변 포인트**:
1. Consumer 수 증가 (최대 Partition 수까지)
2. max.poll.records 조정으로 배치 크기 최적화
3. 병렬 처리 도입 (Partition 내 순서 보장)

### Q: Rebalancing을 최소화하는 방법은?
**답변 포인트**:
1. Static Membership 사용 (Kafka 2.3+)
2. 적절한 heartbeat, session timeout 설정
3. Graceful shutdown으로 깨끗한 Consumer 제거