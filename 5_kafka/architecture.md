# Kafka Architecture

## 1. Broker와 Cluster
- broker 역할
- cluster metadata 관리

## 2. Topic과 Log 구조
- partition 로그 append-only 구조
- segment 파일 구조 (index, log)

## 3. Zookeeper / KRaft
- 기존 Zookeeper 기반 구조
- KRaft(내장 consensus)로의 전환

## 4. Producer / Consumer 흐름
- 메시지 write path
- 메시지 read path

## 5. 면접 질문
- Kafka에서 메시지가 저장되는 구조를 설명해보라.
- broker, topic, partition 관계는?
