# Airflow Architecture

## 1. 전체 구성도
- Webserver
- Scheduler
- Worker
- Metadata DB
- Executor (Celery / Local / Kubernetes)

## 2. 동작 흐름
- Scheduler가 DAG을 읽고 task 생성
- Executor가 task 실행
- Worker가 작업 수행
- Metadata DB 업데이트

## 3. Metadata DB 역할
- task 상태 저장
- DAG 구조 저장
- concurrency & locking 관리

## 4. Executor 종류별 차이
- Local Executor
- Celery Executor
- Kubernetes Executor

## 5. 면접 질문
- Airflow가 DAG을 어떻게 실행하는가?
- Metadata DB의 역할은?



### 아키텍쳐 (3.0)

- scheduler
- dag_processor
- trigger
- worker
- 웹서버 (api)
- meta db
    
    ![스크린샷 2025-10-06 오후 6.35.26.png](attachment:7dacdb2f-83a1-4ccc-a23f-193e80cae5e5:스크린샷_2025-10-06_오후_6.35.26.png)
    
- 3.0 업데이트
    - 웹서버(api) 방식 도입
    - edge executor 도입
        - airflow 클러스터 외부 자원 사용 가능
        - api 서버를 통한 통신을 통해 자원 컨트롤

메타 DB

- DAG, Task, Connection 등 메타정보 저장
- 공식 지원 DB: PostgreSQL, MySQL
- MSSQL은 테스트 중, MariaDB는 비공식

주요 테이블

- dag_run: DAG 실행 이력
- job: Job별 실행 이력
- log: Airflow 내 로그 기록
- task_instance: Task별 실행 이력
- task_reschedule: 재시도 이력 (poke/reschedule 방식 관련)

아키텍처 구성

- Scheduler: DAG 파싱 및 실행 시점 관리
- Worker: Task 실행
- Webserver: UI 제공
- MetaDB: 실행 및 설정 정보 저장
- Airflow 3.0부터 Webserver → API Server 구조로 변경 (보안·확장성 개선)

XCom (Cross Communication)

- Task 간 데이터 공유 기능
- 메타DB의 xcom 테이블에 JSON 직렬화 형태로 저장
- xcom_push(), xcom_pull() 사용
- 소량 데이터만 저장 (대용량은 S3, GCS 등 외부 스토리지 사용)

kwargs

- 함수에 전달되는 Airflow 실행 컨텍스트(dict)
- ti, execution_date, dag_run 등 포함
- ti.xcom_pull() 형태로 접근 가능

Template / Macro

- Jinja 템플릿: 런타임 시 변수값을 문자열에 치환
- 예시 변수: {{ ds }}, {{ data_interval_start }}, {{ data_interval_end }}
- templates_dict로 Task 간 매크로 전달 가능

https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html