# BigQuery 최적화 완벽 가이드

## Partition vs Clustering

### Partition (파티션)

#### 개념
- **테이블을 물리적으로 분할**하여 저장
- **쿼리 시 필요한 파티션만 스캔**하여 비용 및 성능 최적화
- **일자, 시간, 정수 범위** 기반으로 분할

#### Date Partitioning
```sql
-- 일자 기반 파티션 테이블 생성
CREATE TABLE `project.dataset.sales_partitioned` (
    transaction_id STRING,
    customer_id STRING,
    product_id STRING,
    amount NUMERIC,
    transaction_date DATE
)
PARTITION BY DATE(transaction_date)
OPTIONS (
    description = "Sales data partitioned by transaction date",
    partition_expiration_days = 730  -- 2년 후 자동 삭제
);

-- 데이터 삽입
INSERT INTO `project.dataset.sales_partitioned` VALUES
('T001', 'C001', 'P001', 100.00, '2023-12-01'),
('T002', 'C002', 'P002', 150.00, '2023-12-02'),
('T003', 'C003', 'P003', 200.00, '2023-12-03');

-- 최적화된 쿼리 (특정 날짜만 스캔)
SELECT 
    customer_id,
    SUM(amount) as total_amount
FROM `project.dataset.sales_partitioned`
WHERE transaction_date = '2023-12-01'  -- 이 파티션만 스캔
GROUP BY customer_id;

-- 비용 효과: 전체 테이블 대신 1일 파티션만 스캔 (99% 비용 절감)
```

#### DateTime Partitioning (시간 단위)
```sql
-- 시간 기반 파티션 (실시간 데이터용)
CREATE TABLE `project.dataset.events_hourly` (
    event_id STRING,
    user_id STRING,
    event_type STRING,
    event_timestamp TIMESTAMP
)
PARTITION BY DATETIME_TRUNC(DATETIME(event_timestamp), HOUR)
OPTIONS (
    partition_expiration_days = 30  -- 30일 보관
);

-- 지난 1시간 이벤트 조회 (1시간 파티션만 스캔)
SELECT 
    event_type,
    COUNT(*) as event_count
FROM `project.dataset.events_hourly`
WHERE event_timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
GROUP BY event_type;
```

#### Integer Range Partitioning
```sql
-- 정수 범위 기반 파티션 (사용자 ID)
CREATE TABLE `project.dataset.user_data_ranged` (
    user_id INT64,
    user_name STRING,
    signup_date DATE
)
PARTITION BY RANGE_BUCKET(user_id, GENERATE_ARRAY(0, 1000000, 10000))
-- 0~9999, 10000~19999, ... 범위별 파티션

-- 특정 사용자 ID 범위 조회
SELECT *
FROM `project.dataset.user_data_ranged`
WHERE user_id BETWEEN 10000 AND 19999;  -- 해당 파티션만 스캔
```

### Clustering (클러스터링)

#### 개념
- **파티션 내에서 데이터를 정렬하여 저장**
- **자주 필터링하는 컬럼을 클러스터 키로 설정**
- **최대 4개 컬럼까지 클러스터링 가능**

#### 기본 클러스터링
```sql
-- 파티션 + 클러스터링 조합
CREATE TABLE `project.dataset.sales_optimized` (
    transaction_id STRING,
    customer_id STRING,
    product_category STRING,
    region STRING,
    amount NUMERIC,
    transaction_date DATE
)
PARTITION BY DATE(transaction_date)
CLUSTER BY customer_id, product_category  -- 자주 조회하는 컬럼들
```

#### 클러스터링 최적화 효과
```sql
-- 최적화된 쿼리 (파티션 + 클러스터 필터)
SELECT 
    customer_id,
    product_category,
    SUM(amount) as total_sales
FROM `project.dataset.sales_optimized`
WHERE transaction_date BETWEEN '2023-12-01' AND '2023-12-31'  -- 파티션 필터
  AND customer_id = 'C001'                                    -- 클러스터 필터
  AND product_category = 'Electronics'                        -- 클러스터 필터
GROUP BY customer_id, product_category;

-- 성능 향상:
-- 1. 파티션 필터: 12월 데이터만 스캔 (96% 데이터 제외)
-- 2. 클러스터 필터: customer_id, category 기준으로 정렬된 블록만 스캔
-- → 총 처리 데이터량 99.5% 감소
```

### 파티션 vs 클러스터링 선택 기준

#### 파티션을 사용해야 하는 경우
```sql
-- 1. 시간 기반 데이터 (로그, 이벤트)
-- 2. 오래된 데이터 자동 삭제 필요
-- 3. 특정 날짜/기간 기준 조회가 대부분

-- 예시: 웹 로그 분석
CREATE TABLE `project.analytics.web_logs` (
    session_id STRING,
    user_id STRING,
    page_url STRING,
    visit_time TIMESTAMP
)
PARTITION BY DATE(visit_time)
OPTIONS (partition_expiration_days = 90);  -- 90일 후 자동 삭제

-- 장점: 오래된 로그 자동 정리, 일자별 조회 시 비용 최적화
```

#### 클러스터링을 사용해야 하는 경우
```sql
-- 1. 사용자 ID, 상품 ID 등 특정 값 기준 조회
-- 2. 카테고리별 집계가 빈번한 경우
-- 3. Join 성능 최적화 필요

-- 예시: 사용자 행동 분석
CREATE TABLE `project.analytics.user_events` (
    event_id STRING,
    user_id STRING,
    product_id STRING,
    event_type STRING,
    created_at TIMESTAMP
)
CLUSTER BY user_id, product_id;

-- 장점: 특정 사용자나 상품별 조회 시 성능 최적화
```

## Slot 관리

### On-demand vs Flat-rate

#### On-demand Pricing
```sql
-- 쿼리 실행 시마다 처리된 데이터량에 따라 과금
-- $6.25 per TB (asia-northeast1 기준)

-- 비용 예측 쿼리
SELECT 
    FORMAT("%.2f", (bytes_processed / POW(10, 12)) * 6.25) as estimated_cost_usd
FROM (
    SELECT 
        SUM(total_bytes_processed) as bytes_processed
    FROM `region-asia-northeast1`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
    WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 DAY)
);
```

#### Flat-rate Pricing
```sql
-- 월 단위 고정 요금으로 슬롯 예약
-- 100 slots = $2,000/month (asia-northeast1 기준)

-- Reservation 생성
-- gcloud CLI 또는 BigQuery Reservations API 사용

-- 슬롯 사용량 모니터링
SELECT 
    creation_time,
    user_email,
    job_id,
    total_slot_ms / (1000 * 60) as slot_minutes_used
FROM `region-asia-northeast1`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
ORDER BY slot_minutes_used DESC;
```

### 슬롯 사용량 최적화

#### 쿼리 최적화로 슬롯 효율성 향상
```sql
-- 비효율적인 쿼리 (많은 슬롯 사용)
SELECT *
FROM `project.dataset.large_table` 
WHERE EXTRACT(YEAR FROM date_column) = 2023;

-- 최적화된 쿼리 (적은 슬롯 사용)
SELECT *
FROM `project.dataset.large_table`
WHERE date_column >= '2023-01-01' 
  AND date_column < '2024-01-01';

-- 성능 차이:
-- 비효율: 전체 테이블 스캔 (1000 슬롯 × 5분)
-- 최적화: 파티션 스캔 (100 슬롯 × 30초)
```

#### 동시 쿼리 수 제한
```python
# Python에서 슬롯 사용량 제어
from google.cloud import bigquery
import time

client = bigquery.Client()
max_concurrent_jobs = 5  # 최대 5개 쿼리 동시 실행

def run_queries_with_slot_control(queries):
    running_jobs = []
    
    for query in queries:
        # 실행 중인 작업이 최대치에 도달하면 대기
        while len(running_jobs) >= max_concurrent_jobs:
            # 완료된 작업 제거
            running_jobs = [job for job in running_jobs if job.state != 'DONE']
            time.sleep(10)  # 10초 대기
        
        # 새로운 쿼리 시작
        job = client.query(query)
        running_jobs.append(job)
        print(f"Started job: {job.job_id}")
    
    # 모든 작업 완료 대기
    for job in running_jobs:
        job.result()
        print(f"Completed job: {job.job_id}")
```

## 쿼리 최적화

### Bytes Processed 줄이기

#### SELECT 절 최적화
```sql
-- 비효율적: 모든 컬럼 선택
SELECT *
FROM `project.dataset.large_table`
WHERE date_column = '2023-12-01';
-- Bytes processed: 100GB

-- 최적화: 필요한 컬럼만 선택
SELECT 
    user_id,
    order_amount,
    order_date
FROM `project.dataset.large_table`
WHERE date_column = '2023-12-01';
-- Bytes processed: 5GB (95% 절감)
```

#### WHERE 절 최적화
```sql
-- 비효율적: 함수 사용
SELECT customer_id, SUM(amount)
FROM sales_table
WHERE EXTRACT(YEAR FROM order_date) = 2023
GROUP BY customer_id;

-- 최적화: 파티션 컬럼 직접 필터링
SELECT customer_id, SUM(amount)
FROM sales_table  
WHERE order_date >= '2023-01-01'
  AND order_date < '2024-01-01'
GROUP BY customer_id;
```

#### JOIN 최적화
```sql
-- 비효율적: 대형 테이블 먼저 JOIN
SELECT 
    l.order_id,
    s.customer_name
FROM `project.dataset.large_orders` l
JOIN `project.dataset.small_customers` s
  ON l.customer_id = s.customer_id;

-- 최적화: 작은 테이블을 오른쪽에 배치 (Broadcast Join)
SELECT 
    l.order_id,
    s.customer_name
FROM `project.dataset.large_orders` l
JOIN `project.dataset.small_customers` s  -- 작은 테이블 오른쪽
  ON l.customer_id = s.customer_id;

-- 더 나은 최적화: 명시적 Broadcast Hint (BigQuery 미지원, 대신 크기 순서 조정)
```

### Materialized View 활용

#### 자주 사용하는 집계 결과 미리 계산
```sql
-- 기존: 매번 집계 계산 (비용 높음)
SELECT 
    product_category,
    DATE_TRUNC(order_date, MONTH) as month,
    SUM(order_amount) as monthly_sales,
    COUNT(*) as order_count
FROM `project.ecommerce.orders`
WHERE order_date >= '2023-01-01'
GROUP BY product_category, month;
-- 매번 실행 시 10GB 처리

-- Materialized View 생성
CREATE MATERIALIZED VIEW `project.analytics.monthly_sales_mv`
PARTITION BY month
CLUSTER BY product_category
AS
SELECT 
    product_category,
    DATE_TRUNC(order_date, MONTH) as month,
    SUM(order_amount) as monthly_sales,
    COUNT(*) as order_count
FROM `project.ecommerce.orders`
GROUP BY product_category, month;

-- 최적화된 쿼리 (Materialized View 사용)
SELECT *
FROM `project.analytics.monthly_sales_mv`
WHERE month = '2023-12-01'
  AND product_category = 'Electronics';
-- 처리 데이터량: 1MB (99.99% 절감)
```

#### Materialized View 자동 갱신
```sql
-- 자동 갱신 설정 (기본적으로 자동 활성화)
CREATE MATERIALIZED VIEW `project.analytics.daily_summary_mv`
OPTIONS (
    enable_refresh = TRUE,  -- 자동 갱신 활성화
    refresh_interval_minutes = 60  -- 60분마다 갱신
)
AS
SELECT 
    DATE(created_at) as date,
    product_id,
    COUNT(*) as daily_orders,
    SUM(amount) as daily_revenue
FROM `project.ecommerce.orders`
GROUP BY date, product_id;

-- Materialized View 갱신 상태 확인
SELECT 
    table_name,
    last_refresh_time,
    refresh_watermark
FROM `project.analytics`.INFORMATION_SCHEMA.MATERIALIZED_VIEWS
WHERE table_name = 'daily_summary_mv';
```

## 실무 사례: 이커머스 분석 최적화

### 문제 상황
```sql
-- 기존 월별 매출 분석 쿼리 (문제 있는 버전)
SELECT 
    c.customer_segment,
    p.category,
    DATE_TRUNC(o.order_date, MONTH) as month,
    COUNT(*) as orders,
    SUM(oi.quantity * oi.price) as revenue
FROM `ecommerce.orders` o
JOIN `ecommerce.customers` c ON o.customer_id = c.customer_id
JOIN `ecommerce.order_items` oi ON o.order_id = oi.order_id  
JOIN `ecommerce.products` p ON oi.product_id = p.product_id
WHERE o.order_date >= '2022-01-01'
GROUP BY customer_segment, category, month
ORDER BY month, revenue DESC;

-- 문제점:
-- 1. 4개 대형 테이블 JOIN (처리 데이터량: 500GB)
-- 2. 파티션 활용 안 됨
-- 3. 매월 실행 시마다 $3,125 비용 ($6.25 × 500GB)
```

### 최적화 과정

#### 1단계: 테이블 파티셔닝
```sql
-- orders 테이블 파티셔닝 재설계
CREATE TABLE `ecommerce.orders_partitioned` (
    order_id STRING,
    customer_id STRING,
    order_date DATE,
    total_amount NUMERIC
)
PARTITION BY DATE(order_date)
CLUSTER BY customer_id;

-- order_items 테이블 파티셔닝
CREATE TABLE `ecommerce.order_items_partitioned` (
    order_id STRING,
    product_id STRING,
    quantity INT64,
    price NUMERIC,
    order_date DATE  -- 파티션 키 추가
)
PARTITION BY DATE(order_date)
CLUSTER BY order_id, product_id;
```

#### 2단계: Denormalized Table 생성
```sql
-- 자주 사용하는 조인 결과를 미리 비정규화
CREATE TABLE `ecommerce.order_details_denorm` (
    order_id STRING,
    order_date DATE,
    customer_id STRING,
    customer_segment STRING,
    product_id STRING,
    product_category STRING,
    quantity INT64,
    unit_price NUMERIC,
    total_price NUMERIC
)
PARTITION BY DATE(order_date)
CLUSTER BY customer_segment, product_category;

-- 일일 ETL로 비정규화 테이블 갱신
INSERT INTO `ecommerce.order_details_denorm`
SELECT 
    o.order_id,
    o.order_date,
    o.customer_id,
    c.customer_segment,
    oi.product_id,
    p.category as product_category,
    oi.quantity,
    oi.price as unit_price,
    oi.quantity * oi.price as total_price
FROM `ecommerce.orders_partitioned` o
JOIN `ecommerce.customers` c ON o.customer_id = c.customer_id
JOIN `ecommerce.order_items_partitioned` oi ON o.order_id = oi.order_id
JOIN `ecommerce.products` p ON oi.product_id = p.product_id
WHERE o.order_date = CURRENT_DATE() - 1;  -- 어제 데이터만 처리
```

#### 3단계: Materialized View 활용
```sql
-- 월별 집계 Materialized View
CREATE MATERIALIZED VIEW `ecommerce.monthly_revenue_mv`
PARTITION BY month
CLUSTER BY customer_segment, product_category
AS
SELECT 
    customer_segment,
    product_category,
    DATE_TRUNC(order_date, MONTH) as month,
    COUNT(*) as order_count,
    SUM(quantity) as total_quantity,
    SUM(total_price) as total_revenue
FROM `ecommerce.order_details_denorm`
GROUP BY customer_segment, product_category, month;
```

#### 4단계: 최적화된 분석 쿼리
```sql
-- 최적화된 월별 매출 분석
SELECT 
    customer_segment,
    product_category,
    month,
    order_count,
    total_revenue,
    -- 전월 대비 성장률 계산
    LAG(total_revenue) OVER (
        PARTITION BY customer_segment, product_category 
        ORDER BY month
    ) as prev_month_revenue,
    ROUND(
        ((total_revenue - LAG(total_revenue) OVER (
            PARTITION BY customer_segment, product_category 
            ORDER BY month
        )) / LAG(total_revenue) OVER (
            PARTITION BY customer_segment, product_category 
            ORDER BY month
        )) * 100, 2
    ) as growth_rate_pct
FROM `ecommerce.monthly_revenue_mv`
WHERE month >= '2023-01-01'
ORDER BY month, total_revenue DESC;

-- 성능 개선 결과:
-- 처리 데이터량: 500GB → 5MB (99.99% 절감)
-- 실행 시간: 5분 → 3초 (99.5% 단축)
-- 월 비용: $3,125 → $0.03 (99.99% 절감)
```

### 비용 모니터링 및 알림
```python
# Python으로 BigQuery 비용 모니터링
from google.cloud import bigquery
import pandas as pd

def monitor_query_costs(project_id, days=7):
    client = bigquery.Client()
    
    # 최근 7일간 쿼리 비용 분석
    query = f"""
    SELECT 
        user_email,
        DATE(creation_time) as query_date,
        COUNT(*) as query_count,
        SUM(total_bytes_processed) / POW(10, 12) as tb_processed,
        SUM(total_bytes_processed) / POW(10, 12) * 6.25 as estimated_cost_usd
    FROM `region-asia-northeast1`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
    WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL {days} DAY)
        AND job_type = 'QUERY'
        AND state = 'DONE'
    GROUP BY user_email, query_date
    ORDER BY estimated_cost_usd DESC
    """
    
    df = client.query(query).to_dataframe()
    
    # 고비용 사용자 알림
    high_cost_users = df[df['estimated_cost_usd'] > 100]  # $100 이상
    
    for _, row in high_cost_users.iterrows():
        print(f"⚠️  High cost alert: {row['user_email']}")
        print(f"   Date: {row['query_date']}")
        print(f"   Queries: {row['query_count']}")
        print(f"   Data processed: {row['tb_processed']:.2f} TB")
        print(f"   Estimated cost: ${row['estimated_cost_usd']:.2f}")
        print()
    
    return df

# 실행
cost_report = monitor_query_costs('your-project-id')
```

## 면접 단골 질문과 답변

### Q: BigQuery Partition은 왜 필요한가요?
**1분 답변**: "BigQuery는 컬럼형 저장소로 전체 컬럼을 스캔하기 때문에 데이터량에 비례해서 비용이 발생합니다. Partition을 사용하면 날짜나 범위 기준으로 테이블을 물리적으로 분할해서 필요한 부분만 스캔할 수 있습니다. 예를 들어 1년치 데이터에서 1일만 조회할 때 파티션 없으면 365일 모두 스캔하지만, 파티션이 있으면 해당 날짜만 스캔해서 99% 비용을 절약할 수 있습니다."

### Q: Materialized View는 언제 사용하나요?
**답변 포인트**:
1. 자주 실행되는 복잡한 집계 쿼리
2. JOIN이 많은 쿼리의 성능 최적화
3. 실시간성이 중요하지 않은 분석 업무

### Q: BigQuery 비용은 어떻게 계산되나요?
**답변 포인트**:
1. On-demand: 처리된 데이터량 × $6.25/TB
2. Flat-rate: 월 고정 요금으로 슬롯 예약
3. 파티션, 클러스터링으로 스캔 데이터량 최소화

### Q: Slot이란 무엇인가요?
**답변 포인트**:
1. BigQuery의 컴퓨팅 리소스 단위
2. 쿼리 복잡도에 따라 필요한 슬롯 수 결정
3. Flat-rate로 고정 슬롯 예약 시 비용 예측 가능