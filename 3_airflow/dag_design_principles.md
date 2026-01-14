# Airflow DAG 설계 원칙과 최적화

## DAG 설계 원칙

### 1. TaskGroup 활용

#### 개념
- **논리적으로 관련된 Task들을 그룹핑**
- **DAG 가독성 향상**
- **SubDAG의 현대적 대안** (Airflow 2.0+)

#### 기본 사용법
```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.utils.task_group import TaskGroup
from datetime import datetime, timedelta

def extract_user_data():
    # 사용자 데이터 추출
    pass

def extract_order_data():
    # 주문 데이터 추출  
    pass

def transform_user_data():
    # 사용자 데이터 변환
    pass

def transform_order_data():
    # 주문 데이터 변환
    pass

with DAG(
    'ecommerce_etl',
    default_args={'retries': 2},
    schedule_interval='0 2 * * *',  # 매일 새벽 2시
    start_date=datetime(2023, 1, 1),
    catchup=False
) as dag:
    
    # Extract TaskGroup
    with TaskGroup('extract_data') as extract_group:
        extract_users = PythonOperator(
            task_id='extract_users',
            python_callable=extract_user_data
        )
        
        extract_orders = PythonOperator(
            task_id='extract_orders',
            python_callable=extract_order_data
        )
    
    # Transform TaskGroup  
    with TaskGroup('transform_data') as transform_group:
        transform_users = PythonOperator(
            task_id='transform_users',
            python_callable=transform_user_data
        )
        
        transform_orders = PythonOperator(
            task_id='transform_orders', 
            python_callable=transform_order_data
        )
    
    # TaskGroup 간 의존성
    extract_group >> transform_group
```

#### 복잡한 TaskGroup 예시
```python
# 실무 사례: 다중 소스 데이터 파이프라인
with TaskGroup('extract_sources') as extract_group:
    # DB 소스별 병렬 추출
    extract_mysql = PythonOperator(
        task_id='extract_mysql',
        python_callable=extract_from_mysql
    )
    
    extract_postgres = PythonOperator(
        task_id='extract_postgres',
        python_callable=extract_from_postgres
    )
    
    extract_api = PythonOperator(
        task_id='extract_api',
        python_callable=extract_from_api
    )

with TaskGroup('quality_checks') as quality_group:
    # 데이터 품질 검증
    check_completeness = PythonOperator(
        task_id='check_completeness',
        python_callable=check_data_completeness
    )
    
    check_accuracy = PythonOperator(
        task_id='check_accuracy', 
        python_callable=check_data_accuracy
    )
    
    # TaskGroup 내부 의존성
    check_completeness >> check_accuracy

# 복잡한 의존성 관리
extract_group >> quality_group
```

### 2. Sensor vs Operator 선택

#### Sensor 사용 사례

##### FileSensor: 파일 도착 대기
```python
from airflow.sensors.filesystem import FileSensor
from airflow.operators.bash import BashOperator

# 외부 시스템에서 파일 생성까지 대기
wait_for_file = FileSensor(
    task_id='wait_for_daily_report',
    filepath='/data/input/daily_report_{{ ds }}.csv',
    poke_interval=60,  # 60초마다 확인
    timeout=3600,      # 1시간 후 타임아웃
    mode='poke'        # 지속적으로 확인
)

process_file = BashOperator(
    task_id='process_daily_report',
    bash_command='python /scripts/process_report.py {{ ds }}'
)

wait_for_file >> process_file
```

##### S3KeySensor: 클라우드 파일 대기
```python
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor

wait_for_s3_file = S3KeySensor(
    task_id='wait_for_s3_data',
    bucket_name='data-lake',
    bucket_key='raw/{{ ds }}/events.parquet',
    aws_conn_id='aws_default',
    poke_interval=300,  # 5분마다 확인
    timeout=7200        # 2시간 타임아웃
)
```

##### 커스텀 Sensor: API 상태 확인
```python
from airflow.sensors.base import BaseSensorOperator
from airflow.utils.context import Context
import requests

class APIReadySensor(BaseSensorOperator):
    def __init__(self, endpoint, **kwargs):
        super().__init__(**kwargs)
        self.endpoint = endpoint
    
    def poke(self, context: Context) -> bool:
        try:
            response = requests.get(self.endpoint)
            return response.status_code == 200 and response.json()['status'] == 'ready'
        except:
            return False

# ETL 시작 전 API 서비스 준비 상태 확인
wait_for_api = APIReadySensor(
    task_id='wait_for_api_ready',
    endpoint='http://api.company.com/health',
    poke_interval=30,
    timeout=600
)
```

#### Operator 사용 사례

##### PythonOperator: 복잡한 비즈니스 로직
```python
def complex_data_transformation(**context):
    """복잡한 데이터 변환 로직"""
    execution_date = context['ds']
    
    # 여러 소스에서 데이터 읽기
    user_data = read_from_mysql(f"SELECT * FROM users WHERE date = '{execution_date}'")
    order_data = read_from_postgres(f"SELECT * FROM orders WHERE date = '{execution_date}'")
    
    # 복잡한 조인 및 집계
    result = user_data.merge(order_data, on='user_id')\
                     .groupby(['category', 'region'])\
                     .agg({'amount': 'sum', 'quantity': 'count'})
    
    # BigQuery에 저장
    result.to_gbq('analytics.daily_summary', if_exists='replace')
    
    return f"Processed {len(result)} rows for {execution_date}"

transform_data = PythonOperator(
    task_id='transform_daily_data',
    python_callable=complex_data_transformation
)
```

##### BashOperator: 시스템 명령어
```python
# Spark Job 실행
submit_spark_job = BashOperator(
    task_id='submit_spark_analysis',
    bash_command="""
    gcloud dataproc jobs submit pyspark \
        gs://bucket/scripts/analysis.py \
        --cluster=analytics-cluster \
        --region=asia-northeast1 \
        --properties='spark.executor.memory=4g,spark.executor.cores=2'
    """
)

# 데이터 품질 검증
quality_check = BashOperator(
    task_id='run_data_quality_check',
    bash_command='python /scripts/data_quality.py --date {{ ds }}'
)
```

### 3. 의존성 관리 (>>)

#### 기본 의존성
```python
# 순차 실행
task_a >> task_b >> task_c

# 병렬 후 합류
task_a >> [task_b, task_c] >> task_d

# 복잡한 분기
task_a >> task_b
task_a >> task_c
[task_b, task_c] >> task_d
```

#### XCom을 활용한 데이터 전달
```python
def extract_daily_count(**context):
    # 데이터 추출 및 개수 반환
    count = query_database(f"SELECT COUNT(*) FROM orders WHERE date = '{context['ds']}'")
    return count

def validate_count(**context):
    # 이전 Task의 결과 가져오기
    count = context['task_instance'].xcom_pull(task_ids='extract_count')
    
    if count < 1000:
        raise ValueError(f"Data count too low: {count}")
    
    return f"Validation passed: {count} records"

extract_task = PythonOperator(
    task_id='extract_count',
    python_callable=extract_daily_count
)

validate_task = PythonOperator(
    task_id='validate_count', 
    python_callable=validate_count
)

extract_task >> validate_task
```

#### 조건부 실행 (BranchPythonOperator)
```python
from airflow.operators.python import BranchPythonOperator
from airflow.operators.dummy import DummyOperator

def choose_processing_path(**context):
    """데이터 크기에 따라 처리 방법 결정"""
    execution_date = context['ds']
    data_size = get_data_size(execution_date)
    
    if data_size > 10_000_000:  # 1천만 건 이상
        return 'spark_processing'
    else:
        return 'pandas_processing'

branch_task = BranchPythonOperator(
    task_id='choose_processing',
    python_callable=choose_processing_path
)

spark_task = BashOperator(
    task_id='spark_processing',
    bash_command='gcloud dataproc jobs submit pyspark gs://bucket/spark_job.py'
)

pandas_task = PythonOperator(
    task_id='pandas_processing',
    python_callable=process_with_pandas
)

join_task = DummyOperator(
    task_id='join_processing',
    trigger_rule='one_success'  # 둘 중 하나만 성공하면 실행
)

branch_task >> [spark_task, pandas_task] >> join_task
```

### 4. Dynamic DAG 생성

#### 설정 파일 기반 DAG 생성
```python
# config.yaml
tables:
  - name: users
    source: mysql
    schedule: "0 1 * * *"
  - name: orders  
    source: postgres
    schedule: "0 2 * * *"
  - name: products
    source: api
    schedule: "0 3 * * *"
```

```python
import yaml
from airflow import DAG
from airflow.operators.python import PythonOperator

def create_etl_dag(table_name, source_type, schedule):
    """테이블별 ETL DAG 동적 생성"""
    
    def extract_data(**context):
        if source_type == 'mysql':
            return extract_from_mysql(table_name)
        elif source_type == 'postgres':
            return extract_from_postgres(table_name)
        elif source_type == 'api':
            return extract_from_api(table_name)
    
    def transform_data(**context):
        return transform_table_data(table_name)
    
    def load_data(**context):
        return load_to_warehouse(table_name)
    
    dag = DAG(
        f'{table_name}_etl',
        default_args={
            'owner': 'data-team',
            'retries': 2,
            'retry_delay': timedelta(minutes=5)
        },
        schedule_interval=schedule,
        start_date=datetime(2023, 1, 1),
        catchup=False
    )
    
    with dag:
        extract = PythonOperator(
            task_id='extract',
            python_callable=extract_data
        )
        
        transform = PythonOperator(
            task_id='transform',
            python_callable=transform_data  
        )
        
        load = PythonOperator(
            task_id='load',
            python_callable=load_data
        )
        
        extract >> transform >> load
    
    return dag

# 설정 파일 읽어서 DAG 생성
with open('/opt/airflow/config/tables.yaml', 'r') as f:
    config = yaml.safe_load(f)

# 각 테이블별로 DAG 동적 생성
for table in config['tables']:
    dag_id = f"{table['name']}_etl"
    globals()[dag_id] = create_etl_dag(
        table['name'], 
        table['source'], 
        table['schedule']
    )
```

## 스케줄링 최적화

### 1. Cron 표현식 활용

```python
# 기본 스케줄링
dag = DAG(
    'daily_report',
    schedule_interval='0 2 * * *',  # 매일 새벽 2시
    start_date=datetime(2023, 1, 1)
)

# 복잡한 스케줄링
dag = DAG(
    'weekly_summary',
    schedule_interval='0 6 * * 1',  # 매주 월요일 새벽 6시
    start_date=datetime(2023, 1, 2)  # 월요일부터 시작
)

# 분 단위 스케줄링  
dag = DAG(
    'realtime_sync',
    schedule_interval='*/15 * * * *',  # 15분마다
    start_date=datetime(2023, 1, 1)
)

# 커스텀 스케줄링 (마지막 영업일)
from airflow.timetables.base import Timetable

class LastBusinessDayTimetable(Timetable):
    def next_dagrun_info(self, last_automated_run, restriction):
        # 마지막 영업일 로직 구현
        pass

dag = DAG(
    'month_end_report',
    timetable=LastBusinessDayTimetable(),
    start_date=datetime(2023, 1, 1)
)
```

### 2. catchup 설정

```python
# catchup=True: 과거 미실행 구간 모두 실행
dag = DAG(
    'historical_backfill',
    schedule_interval='@daily',
    start_date=datetime(2023, 1, 1),  # 2023-01-01부터
    catchup=True  # 과거 모든 날짜 실행
)

# catchup=False: 현재부터만 실행 (권장)
dag = DAG(
    'daily_etl',
    schedule_interval='@daily', 
    start_date=datetime(2023, 1, 1),
    catchup=False  # 과거 실행 안 함
)
```

### 3. max_active_runs 제어

```python
# 동시 실행 제한
dag = DAG(
    'resource_intensive_job',
    schedule_interval='@hourly',
    max_active_runs=1,  # 한 번에 하나의 DAG Run만
    start_date=datetime(2023, 1, 1)
)

# 병렬 실행 허용
dag = DAG(
    'lightweight_job',
    schedule_interval='@hourly',
    max_active_runs=3,  # 최대 3개까지 동시 실행
    start_date=datetime(2023, 1, 1)
)
```

### 4. depends_on_past 설정

```python
# 이전 실행이 성공해야 다음 실행
task = PythonOperator(
    task_id='sequential_task',
    python_callable=my_function,
    depends_on_past=True,  # 이전 실행 성공 필요
    wait_for_downstream=False
)

# 독립 실행 (권장)
task = PythonOperator(
    task_id='independent_task', 
    python_callable=my_function,
    depends_on_past=False  # 이전 실행과 무관
)
```

## 최적화 전략

### 1. Parallelism 설정

#### Airflow Configuration
```python
# airflow.cfg 설정
[core]
parallelism = 32  # 전체 Airflow에서 동시 실행 가능한 Task 수
max_active_runs_per_dag = 16  # DAG당 최대 동시 실행

[celery]  # CeleryExecutor 사용 시
worker_concurrency = 16  # Worker당 동시 실행 Task 수
```

#### DAG Level 설정
```python
dag = DAG(
    'parallel_processing',
    default_args={
        'pool': 'data_processing_pool'  # Resource Pool 지정
    },
    max_active_runs=2,  # 이 DAG는 최대 2개까지 동시 실행
    concurrency=8,      # 한 번에 최대 8개 Task 동시 실행
    start_date=datetime(2023, 1, 1)
)
```

### 2. Pool 사용

```python
# Pool 생성 (Airflow UI 또는 CLI)
# airflow pools create data_processing 5 "Data processing tasks"

# Task에 Pool 할당
extract_task = PythonOperator(
    task_id='extract_large_dataset',
    python_callable=extract_data,
    pool='data_processing',  # Pool 지정
    pool_slots=2            # 2개 슬롯 사용
)

transform_task = PythonOperator(
    task_id='transform_data',
    python_callable=transform_data,
    pool='data_processing',
    pool_slots=1  # 1개 슬롯 사용
)

# 리소스 제어
# - data_processing Pool: 총 5개 슬롯
# - extract_task: 2개 슬롯 사용  
# - transform_task: 1개 슬롯 사용
# → 동시 실행 시 2개 슬롯 남음
```

### 3. SubDAG vs TaskGroup 비교

#### SubDAG (Legacy, 비권장)
```python
from airflow.operators.subdag import SubDagOperator

def create_subdag(parent_dag_id, child_dag_id, args):
    subdag = DAG(
        f'{parent_dag_id}.{child_dag_id}',
        default_args=args,
        schedule_interval=None  # Parent DAG에서 관리
    )
    
    # SubDAG 내부 Task들
    with subdag:
        task1 = PythonOperator(task_id='task1', python_callable=func1)
        task2 = PythonOperator(task_id='task2', python_callable=func2)
        task1 >> task2
    
    return subdag

# 메인 DAG에서 SubDAG 사용
subdag_task = SubDagOperator(
    task_id='sub_process',
    subdag=create_subdag('main_dag', 'sub_process', default_args)
)
```

**SubDAG 문제점:**
- **Deadlock 위험**: SubDAG가 별도 Executor로 실행
- **복잡한 모니터링**: 중첩된 구조로 UI에서 추적 어려움
- **리소스 경합**: Parent와 Child DAG가 동시에 리소스 사용

#### TaskGroup (권장)
```python
# TaskGroup 사용 (Airflow 2.0+)
with TaskGroup('data_processing') as processing_group:
    extract = PythonOperator(task_id='extract', python_callable=extract_func)
    transform = PythonOperator(task_id='transform', python_callable=transform_func)
    load = PythonOperator(task_id='load', python_callable=load_func)
    
    extract >> transform >> load

# TaskGroup 장점:
# - 같은 DAG 내에서 실행 (Deadlock 없음)
# - 간단한 모니터링 (그룹 단위로 접기/펴기)
# - 리소스 공유 용이
```

## 실무 사례: 대규모 데이터 파이프라인

### 요구사항
```python
# 이커머스 일간 데이터 파이프라인
# 1. 10개 소스에서 데이터 추출 (병렬)
# 2. 데이터 품질 검증
# 3. 변환 및 집계 (Spark)
# 4. 여러 목적지로 로드
# 5. 실행 시간: 2시간 이내
```

### 최적화된 DAG 구조
```python
from airflow.utils.task_group import TaskGroup
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator

def create_optimized_etl_dag():
    with DAG(
        'ecommerce_daily_etl',
        default_args={
            'owner': 'data-team',
            'retries': 2,
            'retry_delay': timedelta(minutes=5),
            'pool': 'etl_pool'
        },
        schedule_interval='0 2 * * *',  # 새벽 2시
        max_active_runs=1,
        concurrency=12,  # 최대 12개 Task 동시 실행
        catchup=False,
        start_date=datetime(2023, 1, 1)
    ) as dag:
        
        # 1단계: 병렬 데이터 추출 (10분)
        with TaskGroup('extract_sources') as extract_group:
            sources = ['users', 'orders', 'products', 'categories', 
                      'payments', 'reviews', 'inventory', 'shipping',
                      'promotions', 'analytics']
            
            extract_tasks = []
            for source in sources:
                task = PythonOperator(
                    task_id=f'extract_{source}',
                    python_callable=extract_from_source,
                    op_args=[source],
                    pool_slots=1
                )
                extract_tasks.append(task)
        
        # 2단계: 데이터 품질 검증 (5분)  
        with TaskGroup('quality_checks') as quality_group:
            completeness_check = PythonOperator(
                task_id='check_completeness',
                python_callable=check_data_completeness,
                pool_slots=1
            )
            
            accuracy_check = PythonOperator(
                task_id='check_accuracy',
                python_callable=check_data_accuracy, 
                pool_slots=1
            )
            
            # 병렬 검증
            [completeness_check, accuracy_check]
        
        # 3단계: Spark 변환 (60분)
        spark_processing = BashOperator(
            task_id='spark_transformation',
            bash_command="""
            gcloud dataproc jobs submit pyspark \
                gs://bucket/scripts/daily_etl.py \
                --cluster=etl-cluster \
                --region=asia-northeast1 \
                --properties='spark.executor.instances=20,spark.executor.memory=8g'
            """,
            pool_slots=3  # 리소스 집약적 작업
        )
        
        # 4단계: 병렬 로드 (15분)
        with TaskGroup('load_destinations') as load_group:
            load_warehouse = PythonOperator(
                task_id='load_to_warehouse',
                python_callable=load_to_bigquery,
                pool_slots=1
            )
            
            load_cache = PythonOperator(
                task_id='load_to_cache',
                python_callable=load_to_redis,
                pool_slots=1
            )
            
            load_analytics = PythonOperator(
                task_id='load_to_analytics',
                python_callable=load_to_elasticsearch,
                pool_slots=1
            )
            
            # 병렬 로드
            [load_warehouse, load_cache, load_analytics]
        
        # 5단계: 알림
        notify_completion = PythonOperator(
            task_id='send_completion_notification',
            python_callable=send_slack_notification,
            pool_slots=1
        )
        
        # 의존성 정의
        extract_group >> quality_group >> spark_processing >> load_group >> notify_completion
        
    return dag

# DAG 등록
ecommerce_etl = create_optimized_etl_dag()
```

### 성능 최적화 결과
```python
# 최적화 전
# - 총 실행시간: 4시간
# - 병목: 순차적 추출 (2시간)
# - 리소스 사용률: 30%

# 최적화 후  
# - 총 실행시간: 1시간 30분 (62.5% 단축)
# - 병목 해소: 병렬 추출 (10분)
# - 리소스 사용률: 85%

# 주요 개선사항:
# 1. 데이터 추출 병렬화: 2시간 → 10분
# 2. TaskGroup으로 가독성 향상
# 3. Pool 기반 리소스 제어
# 4. Spark 클러스터 크기 최적화
```

## 면접 단골 질문과 답변

### Q: Airflow와 Crontab 차이는?
**1분 답변**: "Airflow는 의존성 관리, 재시도, 모니터링, 백필 기능을 제공하는 워크플로우 오케스트레이션 도구입니다. Crontab은 단순한 스케줄러로 실패 시 처리, 작업 간 의존성, 로그 관리가 어렵습니다. 데이터 파이프라인에서는 Task 실패 시 재시도, 이전 Task 성공 여부 확인, 전체 파이프라인 모니터링이 필요하기 때문에 Airflow를 사용합니다."

### Q: TaskGroup은 왜 사용하나요?
**답변 포인트**:
1. SubDAG의 Deadlock 문제 해결
2. DAG 가독성 향상으로 유지보수 용이
3. 논리적 그룹핑으로 모니터링 효율성 증대

### Q: Dynamic DAG는 어떻게 만드나요?
**답변 포인트**:
1. 설정 파일 기반으로 DAG 동적 생성
2. globals()를 사용해 런타임에 DAG 등록
3. 코드 중복 줄이고 유지보수성 향상

### Q: DAG 실패 시 재실행 전략은?
**답변 포인트**:
1. retries와 retry_delay로 자동 재시도
2. depends_on_past=False로 독립 실행
3. trigger_rule로 조건부 실행 제어