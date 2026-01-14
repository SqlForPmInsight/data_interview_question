# 데이터 파이프라인 아키텍처 패턴

## Lambda Architecture

### 개념 및 구조

#### 3-Layer 아키텍처
```
Raw Data → [Batch Layer] → Batch View
    ↓      [Speed Layer] → Real-time View  
    ↓      [Serving Layer] → Unified View
```

#### Batch Layer (배치 레이어)
```python
# 대용량 데이터의 정확한 처리를 담당
# 특징: High Latency, High Throughput, Eventual Consistency

# Spark Batch Job 예시
from pyspark.sql import SparkSession
from pyspark.sql.functions import *

def daily_batch_processing():
    spark = SparkSession.builder.appName("DailyBatch").getOrCreate()
    
    # 어제 전체 데이터 처리
    raw_data = spark.read.parquet(f"gs://datalake/raw/{yesterday}/")
    
    # 복잡한 집계 및 조인 (정확성 우선)
    daily_summary = raw_data.join(dimension_tables, "key") \
                           .groupBy("category", "region") \
                           .agg(
                               sum("amount").alias("total_sales"),
                               count("*").alias("order_count"),
                               countDistinct("user_id").alias("unique_users")
                           )
    
    # 배치 뷰에 저장 (덮어쓰기)
    daily_summary.write.mode("overwrite") \
                      .partitionBy("date") \
                      .parquet("gs://datalake/batch_views/daily_summary/")

# 장점: 정확한 결과, 복잡한 처리 가능
# 단점: 높은 지연시간 (시간~일 단위)
```

#### Speed Layer (속도 레이어)
```python
# 실시간 데이터의 빠른 처리를 담당
# 특징: Low Latency, Lower Throughput, Approximate Results

# Kafka Streams 실시간 집계
from kafka import KafkaConsumer
import redis
import json

def realtime_processing():
    consumer = KafkaConsumer(
        'user_events',
        bootstrap_servers=['kafka:9092'],
        value_deserializer=lambda x: json.loads(x.decode('utf-8'))
    )
    
    redis_client = redis.Redis(host='redis', port=6379, db=0)
    
    for message in consumer:
        event = message.value
        
        # 실시간 집계 (근사치)
        key = f"hourly_count:{event['category']}:{current_hour}"
        redis_client.incr(key, event['amount'])
        redis_client.expire(key, 3600)  # 1시간 TTL
        
        # 실시간 뷰 업데이트
        update_realtime_dashboard(event)

# 장점: 낮은 지연시간 (초~분 단위)
# 단점: 근사치 결과, 복잡한 처리 제한
```

#### Serving Layer (서빙 레이어)
```python
# 배치 뷰와 실시간 뷰를 통합하여 쿼리 서비스
def query_unified_view(category, region, time_range):
    """통합 뷰 쿼리 - 배치 + 실시간 결과 조합"""
    
    # 1) 배치 뷰에서 과거 데이터 조회
    batch_query = f"""
    SELECT SUM(total_sales) as batch_sales
    FROM batch_views.daily_summary 
    WHERE category = '{category}' 
      AND region = '{region}'
      AND date < CURRENT_DATE()
    """
    batch_result = bigquery_client.query(batch_query).result()
    
    # 2) 실시간 뷰에서 오늘 데이터 조회
    realtime_key = f"today_sales:{category}:{region}"
    realtime_result = redis_client.get(realtime_key) or 0
    
    # 3) 결과 통합
    total_sales = batch_result + int(realtime_result)
    
    return {
        'category': category,
        'region': region,
        'total_sales': total_sales,
        'batch_portion': batch_result,
        'realtime_portion': realtime_result,
        'last_batch_update': get_last_batch_time(),
        'realtime_as_of': datetime.now()
    }
```

### Lambda 아키텍처 장단점

#### 장점
```python
# 1. 내결함성 (Fault Tolerance)
# - 배치 레이어에서 정확한 재계산 가능
# - 실시간 레이어 장애 시 배치 결과로 대체

# 2. 확장성 (Scalability) 
# - 각 레이어별 독립적인 확장
# - 배치: Spark 클러스터 확장
# - 실시간: Kafka/Storm 확장

def handle_speed_layer_failure():
    """속도 레이어 장애 시 대응"""
    if not is_speed_layer_healthy():
        # 배치 뷰만으로 서비스 계속 (약간의 지연 허용)
        return query_batch_view_only()
    else:
        return query_unified_view()
```

#### 단점
```python
# 1. 복잡성 증가
# - 두 개의 별도 코드베이스 유지
# - 배치용 Spark Job + 실시간용 Storm/Kafka Streams

# 2. 데이터 일관성 문제
# - 배치와 실시간 결과가 일시적으로 불일치
# - 복잡한 중복 제거 로직 필요

def reconcile_batch_speed_results():
    """배치-실시간 결과 정합성 검증"""
    batch_count = get_batch_count_for_hour(hour)
    realtime_count = get_realtime_count_for_hour(hour)
    
    discrepancy = abs(batch_count - realtime_count) / batch_count
    
    if discrepancy > 0.05:  # 5% 이상 차이
        alert_data_inconsistency(hour, batch_count, realtime_count)
```

## Kappa Architecture

### 개념 및 구조

#### Stream-only 아키텍처
```
Raw Data → [Stream Processing] → Materialized Views
    ↓           ↓
[Reprocessing] → [Multiple Materialized Views]
```

#### 핵심 원칙
```python
# 1. 모든 데이터를 스트림으로 처리
# 2. 재처리를 통한 배치 기능 대체
# 3. Immutable Event Log가 Source of Truth

# Apache Kafka + Kafka Streams 예시
from kafka import KafkaProducer, KafkaConsumer
from kafka.structs import TopicPartition

class KappaArchitecture:
    def __init__(self):
        self.producer = KafkaProducer(
            bootstrap_servers=['kafka:9092'],
            value_serializer=lambda x: json.dumps(x).encode('utf-8')
        )
        
    def ingest_data(self, event):
        """모든 데이터를 Kafka Stream으로 수집"""
        self.producer.send('raw_events', value=event)
    
    def process_stream(self, processing_logic):
        """실시간 스트림 처리"""
        consumer = KafkaConsumer(
            'raw_events',
            bootstrap_servers=['kafka:9092'],
            auto_offset_reset='latest'  # 실시간부터
        )
        
        for message in consumer:
            event = json.loads(message.value)
            result = processing_logic(event)
            
            # 결과를 별도 Topic으로 출력
            self.producer.send('processed_events', value=result)
    
    def reprocess_historical_data(self, start_time, processing_logic):
        """과거 데이터 재처리 (배치 대체)"""
        consumer = KafkaConsumer(
            'raw_events',
            bootstrap_servers=['kafka:9092'],
            auto_offset_reset='earliest'  # 처음부터
        )
        
        # 특정 시점까지만 처리
        for message in consumer:
            if message.timestamp < start_time:
                continue
                
            event = json.loads(message.value)
            result = processing_logic(event)
            
            # 재처리 결과를 별도 Topic으로
            self.producer.send('reprocessed_events', value=result)
```

### Kafka Streams 기반 구현
```python
# Kafka Streams로 복잡한 집계 처리
from kafka_streams import KafkaStreams, StreamsBuilder
from kafka_streams.serialization import Serdes

def create_kappa_pipeline():
    builder = StreamsBuilder()
    
    # 1) 원본 이벤트 스트림
    events_stream = builder.stream("raw_events", 
                                  value_serde=Serdes.Json())
    
    # 2) 실시간 집계 (Tumbling Window)
    hourly_sales = events_stream \
        .filter(lambda key, value: value['event_type'] == 'purchase') \
        .group_by(lambda key, value: value['category']) \
        .window_by(TimeWindows.of_size(Duration.of_hours(1))) \
        .aggregate(
            initializer=lambda: {'count': 0, 'amount': 0},
            aggregator=lambda key, value, aggregate: {
                'count': aggregate['count'] + 1,
                'amount': aggregate['amount'] + value['amount']
            },
            materialized=Materialized.as_("hourly-sales-store")
        )
    
    # 3) 결과를 다시 Kafka Topic으로 출력
    hourly_sales.to_stream().to("hourly_aggregates")
    
    # 4) 일간 집계 (Hopping Window)  
    daily_sales = hourly_sales.group_by_key() \
        .window_by(TimeWindows.of_size(Duration.of_days(1))) \
        .aggregate(
            initializer=lambda: {'daily_count': 0, 'daily_amount': 0},
            aggregator=lambda key, value, aggregate: {
                'daily_count': aggregate['daily_count'] + value['count'],
                'daily_amount': aggregate['daily_amount'] + value['amount']
            }
        )
    
    return KafkaStreams(builder.build())

# 재처리를 통한 '배치' 작업
def reprocess_with_new_logic():
    """새로운 비즈니스 로직으로 과거 데이터 재처리"""
    
    # 1) 새로운 Consumer Group으로 처음부터 읽기
    consumer = KafkaConsumer(
        'raw_events',
        group_id='reprocessing_group_v2',  # 새로운 그룹
        auto_offset_reset='earliest',
        bootstrap_servers=['kafka:9092']
    )
    
    # 2) 새로운 비즈니스 로직 적용
    def new_business_logic(event):
        # 기존과 다른 카테고리 분류 로직
        if event['product_id'].startswith('PREMIUM'):
            event['category'] = 'Premium'
        return event
    
    # 3) 재처리 결과를 새로운 Topic에 저장
    for message in consumer:
        event = json.loads(message.value)
        updated_event = new_business_logic(event)
        producer.send('events_v2', value=updated_event)
```

### Kappa 아키텍처 장단점

#### 장점
```python
# 1. 단순성 - Single Processing Engine
# - Kafka Streams만으로 모든 처리
# - 배치/실시간 코드 중복 없음

# 2. 낮은 지연시간
# - 스트리밍 처리로 초 단위 응답
# - 별도 배치 대기 시간 없음

def unified_processing_logic(event):
    """배치와 실시간이 동일한 로직 사용"""
    return {
        'user_id': event['user_id'],
        'category': classify_product(event['product_id']),
        'amount': event['price'] * event['quantity'],
        'timestamp': event['timestamp']
    }

# 실시간과 재처리에서 동일한 함수 사용
realtime_processor = KafkaStreams(unified_processing_logic)
batch_reprocessor = ReprocessingJob(unified_processing_logic)
```

#### 단점
```python
# 1. 제한된 처리 능력
# - 복잡한 조인, 집계에 한계
# - 스트리밍 엔진의 메모리 제약

# 2. 상태 관리 복잡성
# - 대용량 State Store 관리
# - Kafka Streams RocksDB 튜닝 필요

def handle_large_state():
    """대용량 상태 관리 최적화"""
    streams_config = {
        'state.dir': '/fast-ssd/kafka-streams',  # SSD 사용
        'rocksdb.config.setter': 'CustomRocksDBConfig',
        'num.stream.threads': 4,
        'cache.max.bytes.buffering': 64 * 1024 * 1024  # 64MB 버퍼
    }
    return streams_config

# 3. 재처리 비용
# - 전체 히스토리 재처리 시간/비용
# - Kafka 스토리지 비용 증가
```

## 실무에서의 선택 기준

### 프로젝트 특성별 아키텍처 선택

#### Lambda 아키텍처를 선택해야 하는 경우
```python
# 사용 사례 1: 금융 거래 시스템
financial_requirements = {
    'accuracy': '100% 정확성 필수',
    'compliance': '감사 로그 보관',
    'latency_tolerance': '일부 지연 허용 (정확성 우선)',
    'complexity_budget': '높음 (전문팀 보유)'
}

def financial_lambda_architecture():
    """금융 시스템 Lambda 아키텍처"""
    
    # Batch Layer: 정확한 정산
    daily_settlement = """
    SELECT 
        account_id,
        SUM(CASE WHEN transaction_type = 'CREDIT' THEN amount ELSE 0 END) as credits,
        SUM(CASE WHEN transaction_type = 'DEBIT' THEN amount ELSE 0 END) as debits,
        SUM(CASE WHEN transaction_type = 'CREDIT' THEN amount ELSE -amount END) as balance
    FROM transactions 
    WHERE DATE(transaction_time) = CURRENT_DATE() - 1
    GROUP BY account_id
    """
    
    # Speed Layer: 실시간 잔액 조회
    def get_realtime_balance(account_id):
        yesterday_balance = get_batch_balance(account_id)
        today_transactions = get_redis_transactions(account_id)
        return yesterday_balance + sum(today_transactions)

# 사용 사례 2: 대형 이커머스 분석
ecommerce_requirements = {
    'data_volume': '페타바이트급',
    'query_complexity': '복잡한 다차원 분석', 
    'team_size': '대규모 (배치/실시간 전문팀)',
    'budget': '높은 인프라 예산'
}
```

#### Kappa 아키텍처를 선택해야 하는 경우
```python
# 사용 사례 1: IoT 모니터링
iot_requirements = {
    'latency': '초 단위 응답 필요',
    'simplicity': '단순한 아키텍처 선호',
    'team_size': '소규모 팀',
    'processing': '주로 집계 및 필터링'
}

def iot_kappa_architecture():
    """IoT 모니터링 Kappa 아키텍처"""
    
    # 센서 데이터 실시간 처리
    sensor_stream = kafka_streams.stream("sensor_data")
    
    # 이상 탐지
    anomalies = sensor_stream \
        .filter(lambda key, value: value['temperature'] > 80) \
        .map_values(lambda value: {
            'alert': 'HIGH_TEMPERATURE',
            'sensor_id': value['sensor_id'],
            'value': value['temperature'],
            'threshold': 80
        })
    
    # 실시간 대시보드용 집계
    hourly_avg = sensor_stream \
        .group_by_key() \
        .window_by(TimeWindows.of_size(Duration.of_hours(1))) \
        .aggregate(
            lambda: {'sum': 0, 'count': 0},
            lambda key, value, agg: {
                'sum': agg['sum'] + value['temperature'],
                'count': agg['count'] + 1
            }
        ) \
        .map_values(lambda agg: agg['sum'] / agg['count'])

# 사용 사례 2: 소셜 미디어 실시간 분석
social_requirements = {
    'velocity': '초당 100만 이벤트',
    'latency': '밀리초 단위',
    'agility': '빠른 기능 변경',
    'cost': '비용 최적화 중요'
}
```

### 하이브리드 접근법
```python
# 실무에서는 완전한 Lambda/Kappa 보다 하이브리드 사용
def hybrid_architecture():
    """하이브리드 아키텍처 - 요구사항별 최적화"""
    
    # 핵심 비즈니스 메트릭: Lambda (정확성 우선)
    core_metrics_pipeline = {
        'batch': 'Spark 일간 정산',
        'speed': 'Kafka Streams 실시간 집계',
        'serving': 'Redis + BigQuery'
    }
    
    # 운영 모니터링: Kappa (신속성 우선)  
    monitoring_pipeline = {
        'stream_only': 'Kafka Streams',
        'storage': 'InfluxDB',
        'alerting': 'Prometheus'
    }
    
    # 분석/리포팅: Batch Only (비용 우선)
    analytics_pipeline = {
        'batch': 'Spark 야간 배치',
        'storage': 'Data Lake',
        'serving': 'BigQuery'
    }

# 마이크로서비스별 아키텍처 선택
service_architectures = {
    'payment_service': 'Lambda',      # 정확성 필수
    'recommendation': 'Kappa',        # 실시간성 중요  
    'user_analytics': 'Batch_Only',   # 비용 효율성
    'fraud_detection': 'Lambda',      # 정확성 + 실시간
    'inventory': 'Kappa'              # 재고 실시간 반영
}
```

## CDC (Change Data Capture)

### CDC 개념 및 활용

#### 기본 개념
```python
# 데이터베이스 변경사항을 실시간으로 캡처하여 스트리밍
# Transaction Log 기반으로 모든 변경사항 추적

# MySQL Binlog 기반 CDC
import pymysqlreplication
from pymysqlreplication import BinLogStreamReader
from pymysqlreplication.row_event import DeleteRowsEvent, UpdateRowsEvent, WriteRowsEvent

class MySQLCDCProducer:
    def __init__(self, mysql_settings, kafka_producer):
        self.mysql_settings = mysql_settings
        self.kafka_producer = kafka_producer
        
    def start_cdc_streaming(self):
        """MySQL Binlog를 읽어서 Kafka로 스트리밍"""
        stream = BinLogStreamReader(
            connection_settings=self.mysql_settings,
            server_id=100,
            blocking=True,
            only_events=[DeleteRowsEvent, WriteRowsEvent, UpdateRowsEvent],
            only_tables=['orders', 'users', 'products']
        )
        
        for binlogevent in stream:
            for row in binlogevent.rows:
                change_event = {
                    'operation': binlogevent.__class__.__name__,
                    'table': binlogevent.table,
                    'timestamp': binlogevent.timestamp,
                    'data': row['values'] if hasattr(row, 'values') else row['after_values'],
                    'before': row.get('before_values')  # UPDATE 시만
                }
                
                # Kafka Topic으로 전송
                topic = f"cdc_{binlogevent.table}"
                self.kafka_producer.send(topic, value=change_event)

# PostgreSQL Logical Replication 기반
import psycopg2
from psycopg2.extras import LogicalReplicationConnection

def postgres_cdc_streaming():
    """PostgreSQL WAL 기반 CDC"""
    conn = psycopg2.connect(
        host='postgres',
        database='ecommerce',
        user='replication_user',
        password='password',
        connection_factory=LogicalReplicationConnection
    )
    
    cur = conn.cursor()
    
    # Replication Slot 생성
    cur.create_replication_slot('kafka_slot', 'test_decoding')
    
    # 변경사항 스트리밍
    cur.start_replication(slot_name='kafka_slot')
    
    def consume_stream(msg):
        change_data = parse_wal_message(msg.payload)
        kafka_producer.send('postgres_changes', value=change_data)
        msg.cursor.send_feedback(flush_lsn=msg.data_start)
    
    cur.consume_stream(consume_stream)
```

### Debezium을 활용한 CDC

#### Debezium 설정
```json
// MySQL Connector 설정
{
  "name": "mysql-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.server.id": "184054",
    "database.server.name": "ecommerce",
    "database.include.list": "ecommerce",
    "table.include.list": "ecommerce.orders,ecommerce.users",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "schema-changes.ecommerce",
    "transforms": "route",
    "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.route.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
    "transforms.route.replacement": "$3"
  }
}
```

#### CDC 데이터 처리
```python
# Debezium CDC 이벤트 처리
def process_cdc_events():
    consumer = KafkaConsumer(
        'orders',  # Debezium이 생성한 Topic
        bootstrap_servers=['kafka:9092'],
        value_deserializer=lambda x: json.loads(x.decode('utf-8'))
    )
    
    for message in consumer:
        cdc_event = message.value
        
        operation = cdc_event['payload']['op']  # c(create), u(update), d(delete)
        
        if operation == 'c':  # INSERT
            handle_order_created(cdc_event['payload']['after'])
        elif operation == 'u':  # UPDATE
            handle_order_updated(
                cdc_event['payload']['before'],
                cdc_event['payload']['after']
            )
        elif operation == 'd':  # DELETE
            handle_order_deleted(cdc_event['payload']['before'])

def handle_order_created(order_data):
    """신규 주문 이벤트 처리"""
    # 1) 실시간 대시보드 업데이트
    update_realtime_metrics(order_data)
    
    # 2) 추천 시스템에 구매 정보 전달
    send_to_recommendation_service(order_data)
    
    # 3) 재고 시스템 업데이트
    update_inventory(order_data['product_id'], -order_data['quantity'])

def handle_order_updated(before, after):
    """주문 수정 이벤트 처리"""
    # 상태 변경 감지
    if before['status'] != after['status']:
        if after['status'] == 'CANCELLED':
            # 재고 복원
            restore_inventory(after['product_id'], after['quantity'])
        elif after['status'] == 'SHIPPED':
            # 배송 알림
            send_shipping_notification(after['customer_id'])
```

### 실시간 동기화 사례

#### Multi-Database 동기화
```python
# 주문 시스템(MySQL) → 분석 DB(BigQuery) 실시간 동기화
class RealTimeSyncPipeline:
    def __init__(self):
        self.kafka_consumer = KafkaConsumer('orders')
        self.bigquery_client = bigquery.Client()
        
    def sync_to_analytics_db(self):
        """CDC 이벤트를 분석 DB에 실시간 반영"""
        
        batch_rows = []
        batch_size = 100
        
        for message in self.kafka_consumer:
            cdc_event = message.value
            
            # BigQuery 스키마에 맞게 변환
            bq_row = self.transform_for_bigquery(cdc_event)
            batch_rows.append(bq_row)
            
            # 배치 크기만큼 모이면 일괄 적재
            if len(batch_rows) >= batch_size:
                self.load_to_bigquery(batch_rows)
                batch_rows.clear()
    
    def transform_for_bigquery(self, cdc_event):
        """CDC 이벤트를 BigQuery 스키마에 맞게 변환"""
        payload = cdc_event['payload']
        operation = payload['op']
        
        if operation in ['c', 'u']:  # CREATE or UPDATE
            data = payload['after']
        else:  # DELETE
            data = payload['before']
            data['_deleted'] = True
        
        return {
            'order_id': data['order_id'],
            'customer_id': data['customer_id'],
            'amount': data['amount'],
            'status': data['status'],
            'created_at': data['created_at'],
            'updated_at': datetime.now().isoformat(),
            '_operation': operation,
            '_cdc_timestamp': cdc_event['payload']['ts_ms']
        }

# 캐시 시스템 동기화
def sync_to_cache():
    """주문 정보를 Redis 캐시에 실시간 동기화"""
    
    for message in consumer:
        cdc_event = message.value
        operation = cdc_event['payload']['op']
        
        if operation == 'c' or operation == 'u':  # CREATE or UPDATE
            order = cdc_event['payload']['after']
            
            # Redis에 최신 주문 정보 캐시
            redis_client.hset(
                f"order:{order['order_id']}", 
                mapping=order
            )
            redis_client.expire(f"order:{order['order_id']}", 3600)  # 1시간 TTL
            
        elif operation == 'd':  # DELETE
            order_id = cdc_event['payload']['before']['order_id']
            redis_client.delete(f"order:{order_id}")
```

## 면접 단골 질문과 답변

### Q: Lambda vs Kappa 차이는?
**1분 답변**: "Lambda는 배치와 실시간 두 개의 처리 경로를 가진 아키텍처입니다. 정확성이 중요한 배치 결과와 낮은 지연시간의 실시간 결과를 조합해서 사용합니다. Kappa는 모든 데이터를 스트림으로 처리하고 재처리를 통해 배치를 대체합니다. 금융처럼 정확성이 중요하면 Lambda, IoT 모니터링처럼 신속성이 중요하고 처리가 단순하면 Kappa가 적합합니다."

### Q: CDC는 언제 사용하나요?
**답변 포인트**:
1. 데이터베이스 간 실시간 동기화 필요
2. 이벤트 기반 아키텍처 구축
3. 레거시 시스템을 점진적 모던화

### Q: 실시간 vs 배치 처리 선택 기준은?
**답변 포인트**:
1. SLA 요구사항: 초/분 단위면 실시간, 시/일 단위면 배치
2. 비용 고려: 배치가 일반적으로 더 저렴
3. 복잡도: 복잡한 조인/집계는 배치가 유리

### Q: Idempotency는 어떻게 보장하나요?
**답변 포인트**:
1. 고유 키 기반 중복 제거 (order_id, transaction_id)
2. 타임스탬프 기반 최신성 검증
3. 상태 전이 검증으로 잘못된 순서 방지