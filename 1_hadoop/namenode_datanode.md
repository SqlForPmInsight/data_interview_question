# NameNode & DataNode

## 1. NameNode 역할
- metadata 관리 (inode, block mapping)
- fsimage / editlog 구조
- NameNode 메모리 이슈

## 2. DataNode 역할
- block 저장
- heartbeat 구조
- block report 메커니즘

## 3. Secondary NameNode의 진짜 역할
- checkpointing
- NameNode HA가 아니란 점

## 4. NameNode HA 구성
- active / standby
- ZKFC 역할
- failover 과정

## 5. 장애 시나리오
- NameNode 죽으면?
- DataNode 죽으면?

## 6. 면접 질문
- NameNode가 죽으면 왜 클러스터가 멈추는가?
- Secondary NameNode가 하는 일 설명
- NameNode HA에서 failover 흐름 설명
