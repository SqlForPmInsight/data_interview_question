# Airflow 면접 질문 5개 ⭐⭐

## Q1. Airflow와 Crontab 차이는?

**답변 포인트:**
1. 의존성 관리 기능
2. 재시도 및 오류 처리
3. 모니터링 및 백필 기능

**1분 답변:**
"Airflow는 의존성 관리, 재시도, 모니터링, 백필 기능을 제공하는 워크플로우 오케스트레이션 도구입니다. Crontab은 단순한 스케줄러로 실패 시 처리, 작업 간 의존성, 로그 관리가 어렵습니다. 데이터 파이프라인에서는 Task 실패 시 재시도, 이전 Task 성공 여부 확인, 전체 파이프라인 모니터링이 필요하기 때문에 Airflow를 사용합니다."

**비교 예시:**
```python
# Crontab: 단순 스케줄링
# 0 2 * * * python /scripts/etl.py

# Airflow: 복잡한 워크플로우
extract >> validate >> [transform_users, transform_orders] >> load >> notify
```

---

## Q2. TaskGroup은 왜 사용하나요?

**답변 포인트:**
1. SubDAG의 Deadlock 문제 해결
2. DAG 가독성 향상
3. 논리적 그룹핑으로 모니터링 효율화

**1분 답변:**
"TaskGroup은 SubDAG의 문제점을 해결하는 현대적 대안입니다. SubDAG는 별도 Executor에서 실행되어 Deadlock 위험이 있고 모니터링이 복잡합니다. TaskGroup은 같은 DAG 내에서 실행되므로 리소스 공유가 용이하고 UI에서 그룹 단위로 접기/펴기가 가능해 가독성이 좋습니다. 논리적으로 연관된 Task들을 묶어서 관리할 수 있어 대규모 DAG의 유지보수에 필수입니다."

**구현 예시:**
```python
with TaskGroup('extract_data') as extract_group:
    extract_users = PythonOperator(task_id='extract_users', ...)
    extract_orders = PythonOperator(task_id='extract_orders', ...)

with TaskGroup('transform_data') as transform_group:
    transform_users = PythonOperator(task_id='transform_users', ...)
    transform_orders = PythonOperator(task_id='transform_orders', ...)

extract_group >> transform_group
```

---

## Q3. Dynamic DAG는 어떻게 만드나요?

**답변 포인트:**
1. 설정 파일 기반 DAG 생성
2. globals()를 사용해 런타임 등록
3. 코드 중복 줄이고 유지보수성 향상

**1분 답변:**
"설정 파일이나 데이터베이스에서 메타데이터를 읽어서 DAG를 동적으로 생성합니다. 테이블별 ETL DAG가 여러 개 필요할 때 각각 만드는 대신 하나의 템플릿으로 동적 생성합니다. Python의 globals() 함수를 사용해서 런타임에 DAG를 등록하고, 설정 변경만으로 새로운 DAG를 추가할 수 있어 코드 중복을 줄이고 유지보수성을 크게 향상시킵니다."

**구현 예시:**
```python
# config.yaml에서 테이블 목록 읽기
tables = ['users', 'orders', 'products']

def create_etl_dag(table_name):
    dag = DAG(f'{table_name}_etl', ...)
    # DAG 로직 정의
    return dag

# 각 테이블별로 DAG 동적 생성
for table in tables:
    dag_id = f"{table}_etl"
    globals()[dag_id] = create_etl_dag(table)
```

---

## Q4. DAG 실패 시 재실행 전략은?

**답변 포인트:**
1. retries와 retry_delay로 자동 재시도
2. depends_on_past=False로 독립 실행
3. trigger_rule로 조건부 실행 제어

**1분 답변:**
"세 가지 전략을 조합합니다. 첫째, retries와 retry_delay로 일시적 오류에 대한 자동 재시도를 설정합니다. 둘째, depends_on_past=False로 설정해서 이전 실행 실패가 다음 스케줄 실행을 막지 않도록 합니다. 셋째, downstream Task에 trigger_rule을 설정해서 일부 upstream 실패를 허용하거나 모든 Task 완료를 기다리도록 제어합니다. 중요한 Task는 재시도 횟수를 늘리고 덜 중요한 Task는 실패를 허용합니다."

**설정 예시:**
```python
critical_task = PythonOperator(
    task_id='critical_process',
    retries=3,
    retry_delay=timedelta(minutes=5),
    depends_on_past=False
)

final_task = PythonOperator(
    task_id='final_step',
    trigger_rule='one_success',  # 하나라도 성공하면 실행
    depends_on_past=False
)
```

---

## Q5. Sensor는 언제 사용하나요?

**답변 포인트:**
1. 외부 리소스 대기 (파일, API)
2. 조건부 실행 트리거
3. 시스템 간 동기화

**1분 답변:**
"외부 시스템의 상태나 리소스를 기다려야 할 때 사용합니다. 파일이 도착하기를 기다리는 FileSensor, API가 준비될 때까지 기다리는 HttpSensor, 데이터베이스의 특정 조건을 확인하는 SqlSensor 등이 있습니다. Operator와 달리 조건이 만족될 때까지 주기적으로 확인하므로 외부 의존성이 있는 파이프라인에서 필수입니다. poke_interval과 timeout을 적절히 설정해서 시스템 부하를 조절해야 합니다."

**사용 예시:**
```python
# 파일 도착 대기
wait_for_file = FileSensor(
    task_id='wait_for_daily_file',
    filepath='/data/input/sales_{{ ds }}.csv',
    poke_interval=60,  # 1분마다 확인
    timeout=3600       # 1시간 타임아웃
)

# API 준비 상태 확인
wait_for_api = HttpSensor(
    task_id='wait_for_api',
    http_conn_id='api_connection',
    endpoint='health',
    poke_interval=30
)

wait_for_file >> wait_for_api >> process_data
```