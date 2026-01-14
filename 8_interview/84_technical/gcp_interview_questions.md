# GCP 면접 질문 5개 ⭐⭐

## Q1. BigQuery Partition은 왜 필요한가요?

**답변 포인트:**
1. 쿼리 비용 최적화
2. 성능 향상 (스캔 데이터 감소)
3. 데이터 관리 효율성

**1분 답변:**
"BigQuery는 컬럼형 저장소로 전체 컬럼을 스캔하기 때문에 데이터량에 비례해서 비용이 발생합니다. Partition을 사용하면 날짜나 범위 기준으로 테이블을 물리적으로 분할해서 필요한 부분만 스캔할 수 있습니다. 예를 들어 1년치 데이터에서 1일만 조회할 때 파티션 없으면 365일 모두 스캔하지만, 파티션이 있으면 해당 날짜만 스캔해서 99% 비용을 절약할 수 있습니다."

**비용 절감 예시:**
```sql
-- 파티션 없는 테이블: 1년 전체 스캔 (100GB)
SELECT COUNT(*) FROM sales_table 
WHERE order_date = '2023-12-01';
-- Cost: $6.25 (100GB × $6.25/TB ÷ 1000)

-- 파티션 테이블: 해당 날짜만 스캔 (0.3GB)  
SELECT COUNT(*) FROM sales_partitioned
WHERE order_date = '2023-12-01';
-- Cost: $0.002 (99.8% 절약)
```

---

## Q2. Dataproc vs Databricks 차이는?

**답변 포인트:**
1. 관리 방식: GCP 네이티브 vs 전용 플랫폼
2. 비용 구조: Compute + 10% vs Compute + DBU
3. 기능: 기본 Spark vs 고급 최적화

**1분 답변:**
"Dataproc은 GCP 네이티브 서비스로 BigQuery, Cloud Storage와 완벽 연동되고 비용이 저렴합니다. Databricks는 전용 Spark 최적화 플랫폼으로 Photon Engine, Delta Lake 등 고급 기능을 제공하지만 DBU 비용이 추가됩니다. GCP 환경에서 단순한 ETL이면 Dataproc, 복잡한 ML 파이프라인이면 Databricks가 적합합니다. 실제 프로젝트에서 Dataproc으로 월 2천만원을 절감했습니다."

**선택 기준:**
```python
# Dataproc 선택 시
use_cases = {
    'simple_etl': 'Spark SQL 기반 변환',
    'cost_sensitive': '70% Preemptible VM 할인',
    'gcp_native': 'BigQuery 직접 연결',
    'batch_jobs': 'Ephemeral 클러스터로 비용 최적화'
}

# Databricks 선택 시
use_cases = {
    'ml_pipeline': 'MLflow 통합',
    'streaming': 'Delta Lake + Structured Streaming',
    'collaboration': '노트북 환경',
    'performance': 'Photon Engine 가속'
}
```

---

## Q3. Ephemeral Cluster는 언제 사용하나요?

**답변 포인트:**
1. 배치 작업에 최적화
2. 비용 효율성 (사용한 시간만 과금)
3. Job별 격리로 안정성 확보

**1분 답변:**
"배치 ETL처럼 일회성 작업에 최적입니다. Job이 실행될 때만 클러스터를 생성하고 완료되면 자동 삭제되어 사용한 시간만큼만 과금됩니다. Job별로 독립된 환경을 제공해서 서로 간섭 없이 안정적으로 실행되고, 매번 새로운 클러스터라 환경 오염 문제도 없습니다. 일간 배치나 주간 리포트처럼 정해진 시간에 실행되는 작업에 이상적입니다."

**구현 예시:**
```bash
# Airflow로 Ephemeral Cluster 관리
gcloud dataproc jobs submit pyspark \
    gs://bucket/daily_etl.py \
    --cluster=daily-etl-$(date +%Y%m%d) \
    --region=asia-northeast1 \
    --num-workers=10 \
    --num-preemptible-workers=20 \
    --max-idle=10m  # 10분 유휴 시 자동 삭제
```

---

## Q4. Preemptible VM 사용 시 주의사항은?

**답변 포인트:**
1. 24시간 내 중단 가능성
2. Checkpointing으로 중단 대응
3. Primary/Secondary Worker 균형

**1분 답변:**
"Preemptible VM은 70% 할인 혜택이 있지만 24시간 내 언제든 중단될 수 있습니다. 이를 대비해 Checkpointing을 활성화해서 중단 시에도 마지막 체크포인트부터 재시작할 수 있도록 해야 합니다. Primary Worker 일부는 유지해서 클러스터 안정성을 보장하고, Secondary Worker로 Preemptible을 사용하는 것이 좋습니다. 보통 80% Preemptible, 20% Primary 비율로 구성합니다."

**최적 구성:**
```python
cluster_config = {
    'primary_workers': 3,      # 안정성 보장
    'secondary_workers': 12,    # 비용 절감 (Preemptible)
    'preemptible_ratio': 0.8,   # 80% 할인
    'cost_saving': '56%'        # 전체 비용 절감률
}

# Fault Tolerance 설정
spark_config = {
    'spark.sql.streaming.checkpointLocation': 'gs://bucket/checkpoints/',
    'spark.checkpoint.dir': 'gs://bucket/spark-checkpoints/'
}
```

---

## Q5. BigQuery 비용은 어떻게 계산되나요?

**답변 포인트:**
1. On-demand: 스캔한 데이터량 × $6.25/TB
2. Flat-rate: 고정 슬롯 예약 요금  
3. 파티션/클러스터링으로 스캔량 최소화

**1분 답변:**
"On-demand 모델에서는 쿼리에서 스캔한 데이터량에 비례해서 TB당 $6.25씩 과금됩니다. Flat-rate는 월 고정 요금으로 슬롯을 예약하는 방식으로 대량 쿼리 시 비용 예측이 가능합니다. 비용을 줄이려면 파티션과 클러스터링으로 스캔 데이터량을 최소화하고, 필요한 컬럼만 SELECT하며, Materialized View로 복잡한 집계 결과를 미리 계산해두는 것이 효과적입니다."

**비용 최적화 사례:**
```sql
-- 최적화 전: 500GB 스캔, $3.125 비용
SELECT * FROM large_table 
WHERE order_date = '2023-12-01';

-- 최적화 후: 5GB 스캔, $0.03 비용 (99% 절감)
-- 1) 파티션 테이블 사용
-- 2) 필요한 컬럼만 선택  
-- 3) 클러스터링으로 추가 필터링
SELECT order_id, customer_id, amount 
FROM large_table_partitioned
WHERE order_date = '2023-12-01'
  AND customer_segment = 'Premium';
```