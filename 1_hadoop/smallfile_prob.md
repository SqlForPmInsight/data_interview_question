# Small File Problem in Hadoop

## 1. Small File Problem이란?
- NameNode 메모리 과부하
- block metadata 폭증

## 2. 왜 문제인가?
- NameNode 단일 장애점
- GC load 증가
- metadata caching 비용 증가

## 3. 해결책
- CombineFileInputFormat
- SequenceFile / Parquet
- HBase 사용
- 데이터 ingest 단계에서 병합
- Cloud object storage에서는 왜 덜 문제인지?

## 4. Cloud vs Hadoop small file 비교
- S3/GCS는 metadata 서버가 분산되어 있음
- 쿼리 성능 영향은 여전히 존재

## 5. 면접 질문
- small file 문제가 생기는 이유?
- 해결 방법 4가지 설명
