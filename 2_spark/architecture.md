# Spark Architecture

## 1. Cluster 구조
- Driver
- Executor
- Cluster Manager (YARN, Kubernetes, Standalone)

## 2. Driver 역할
- DAG 생성
- task scheduling
- job orchestration

## 3. Executor 역할
- task 실행
- shuffle read/write

## 4. Stage & Task
- narrow dependency
- wide dependency (shuffle boundary)

## 5. Execution Model
- job → stage → task
- scheduling 단위

## 6. 면접 질문
- Driver와 Executor의 역할 차이?
- Stage가 나뉘는 기준은?
