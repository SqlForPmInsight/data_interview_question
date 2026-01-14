# 데이터 아키텍처 면접 질문 5개 ⭐⭐

## Q1. Lambda vs Kappa 아키텍처 차이는?

**답변 포인트:**
1. Lambda: 배치 + 실시간 이중 경로
2. Kappa: 스트림 단일 경로
3. 정확성 vs 단순성 트레이드오프

**1분 답변:**
"Lambda는 배치와 실시간 두 개의 처리 경로를 가진 아키텍처입니다. 정확성이 중요한 배치 결과와 낮은 지연시간의 실시간 결과를 조합해서 사용합니다. Kappa는 모든 데이터를 스트림으로 처리하고 재처리를 통해 배치를 대체합니다. 금융처럼 정확성이 중요하면 Lambda, IoT 모니터링처럼 신속성이 중요하고 처리가 단순하면 Kappa가 적합합니다."

**아키텍처 비교:**
```python
# Lambda Architecture
Raw Data → [Batch Layer] → Batch View
    ↓      [Speed Layer] → Real-time View  
    ↓      [Serving Layer] → Unified View

# 장점: 정확성 + 실시간성, 내결함성
# 단점: 코드 중복, 복잡한 운영

# Kappa Architecture  
Raw Data → [Stream Processing] → Materialized Views
    ↓           ↓
[Reprocessing] → [Updated Views]

# 장점: 단순성, 낮은 지연시간
# 단점: 제한된 처리 능력, 상태 관리 복잡
```

---

## Q2. CDC는 언제 사용하나요?

**답변 포인트:**
1. 데이터베이스 간 실시간 동기화
2. 이벤트 기반 아키텍처 구축  
3. 레거시 시스템 현대화

**1분 답변:**
"CDC는 데이터베이스 변경사항을 실시간으로 캡처해서 다른 시스템으로 전달할 때 사용합니다. Transaction Log를 기반으로 모든 변경을 추적해서 INSERT/UPDATE/DELETE를 실시간 스트림으로 만듭니다. 주문 시스템 변경을 실시간 분석 DB에 동기화하거나, 레거시 시스템을 점진적으로 마이크로서비스로 분리할 때 활용합니다. Debezium 같은 도구로 Kafka Connect를 통해 쉽게 구현할 수 있습니다."

**실무 활용:**
```python
# CDC 활용 사례
mysql_orders → Debezium → Kafka → BigQuery   # 분석 동기화
postgres_users → CDC → Redis                 # 캐시 동기화  
legacy_db → CDC → microservices              # 점진적 현대화
```

---

## Q3. 실시간 vs 배치 처리 선택 기준은?

**답변 포인트:**
1. SLA 요구사항 (지연시간 허용 범위)
2. 비용 고려 (배치가 일반적으로 저렴)
3. 처리 복잡도 (배치가 복잡한 연산에 유리)

**1분 답변:**
"세 가지 기준으로 판단합니다. 첫째, SLA 요구사항을 보고 초/분 단위 응답이 필요하면 실시간, 시/일 단위면 배치를 선택합니다. 둘째, 비용을 고려해서 같은 처리량이라면 배치가 더 저렴합니다. 셋째, 복잡한 조인이나 집계는 배치가 유리하고 단순한 필터링이나 집계는 실시간으로 가능합니다. 실무에서는 핵심 메트릭은 실시간, 복잡한 분석은 배치로 분리해서 하이브리드 구성을 많이 사용합니다."

**선택 매트릭스:**
```
지연시간 | 처리복잡도 | 권장방식
---------|------------|----------
< 1분    | 단순       | 실시간 스트리밍
< 1시간  | 중간       | 마이크로배치  
< 1일    | 복잡       | 일간 배치
< 1주    | 매우복잡   | 주간 배치
```

---

## Q4. 데이터 파이프라인에서 Idempotency는 어떻게 보장하나요?

**답변 포인트:**
1. 고유 키 기반 중복 제거
2. 타임스탬프 기반 최신성 검증  
3. 상태 전이 검증

**1분 답변:**
"Idempotency는 같은 작업을 여러 번 실행해도 결과가 동일함을 보장하는 것입니다. 세 가지 방법으로 구현합니다. 첫째, order_id나 transaction_id 같은 고유 키로 중복을 제거합니다. 둘째, updated_at 타임스탬프로 더 최신 데이터만 반영합니다. 셋째, 상태 전이가 유효한지 검증해서 잘못된 순서로 처리되는 것을 방지합니다. 이를 통해 네트워크 재시도나 시스템 장애 시에도 데이터 일관성을 유지할 수 있습니다."

**구현 예시:**
```python
def upsert_order(order_data):
    """Idempotent order processing"""
    order_id = order_data['order_id']
    new_timestamp = order_data['updated_at']
    
    # 1) 중복 확인
    existing = get_order(order_id)
    if existing:
        # 2) 최신성 검증
        if new_timestamp <= existing['updated_at']:
            return existing  # 이미 최신 데이터
        
        # 3) 상태 전이 검증
        if not is_valid_state_transition(existing['status'], order_data['status']):
            raise InvalidStateTransition()
    
    # 안전한 업데이트
    return save_order(order_data)
```

---

## Q5. 데이터 품질을 어떻게 보장하나요?

**답변 포인트:**
1. 스키마 검증 (Schema validation)
2. 데이터 품질 체크 (Completeness, Accuracy)
3. 모니터링 및 알림

**1분 답변:**
"데이터 품질은 여러 단계에서 보장합니다. 첫째, 스키마 검증으로 데이터 타입과 필수 필드를 확인합니다. 둘째, 완전성 검사로 예상 데이터량과 실제량을 비교하고, 정확성 검사로 비즈니스 룰을 위반하는 데이터를 탐지합니다. 셋째, 이상치 탐지로 통계적 기준을 벗어나는 값을 찾아냅니다. 각 단계에서 문제 발견 시 자동 알림을 보내고 데이터 리니지를 추적해서 문제 원인을 빠르게 찾을 수 있도록 합니다."

**품질 체크 파이프라인:**
```python
def data_quality_check(df):
    checks = [
        # 1. 스키마 검증
        validate_schema(df, expected_schema),
        
        # 2. 완전성 검사  
        check_completeness(df, expected_row_count),
        
        # 3. 정확성 검사
        check_business_rules(df),
        
        # 4. 이상치 탐지
        detect_outliers(df, statistical_bounds)
    ]
    
    failed_checks = [c for c in checks if not c.passed]
    if failed_checks:
        send_quality_alert(failed_checks)
        quarantine_data(df)
    
    return len(failed_checks) == 0
```