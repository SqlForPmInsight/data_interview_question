# Dataproc 최적화 완벽 가이드

## Dataproc vs Databricks 비교

### 핵심 차이점

#### 1. 관리 방식
```bash
# Dataproc: GCP 네이티브 서비스
# 장점: GCP 서비스와 완벽 연동, BigQuery 직접 연결
# 단점: Spark 최적화 기능이 Databricks 대비 제한적

gcloud dataproc clusters create my-cluster \
    --region=asia-northeast1 \
    --zone=asia-northeast1-a \
    --num-masters=1 \
    --num-workers=4 \
    --worker-machine-type=n1-standard-4 \
    --initialization-actions=gs://bucket/init-script.sh

# Databricks: 전용 Spark 최적화 플랫폼  
# 장점: Auto Scaling, Delta Lake, ML 통합 환경
# 단점: GCP 서비스 연동 시 추가 설정 필요
```

#### 2. 비용 구조
```python
# Dataproc 비용 계산
def calculate_dataproc_cost(machine_type, num_workers, hours):
    # Compute Engine 비용 + Dataproc 서비스 비용 (10%)
    
    machine_costs = {
        'n1-standard-4': 0.1900,   # 시간당 $0.19
        'n1-highmem-4': 0.2376,    # 시간당 $0.24  
        'n1-highcpu-16': 0.5688    # 시간당 $0.57
    }
    
    base_cost = machine_costs[machine_type] * num_workers * hours
    dataproc_overhead = base_cost * 0.1  # 10% 추가
    
    return base_cost + dataproc_overhead

# 예시: n1-standard-4 × 10대 × 8시간
# = $0.19 × 10 × 8 = $15.2
# + Dataproc 수수료 $1.52 = $16.72

# Databricks 비용: 더 높은 DBU(Databricks Unit) 비용
# Standard Tier: $0.15/DBU + Compute 비용
# Premium Tier: $0.30/DBU + Compute 비용
```

#### 3. 성능 및 기능
```python
# Dataproc 설정
cluster_config = {
    'software_config': {
        'properties': {
            # Spark 설정 수동 최적화 필요
            'spark:spark.executor.memory': '6g',
            'spark:spark.executor.cores': '2', 
            'spark:spark.dynamicAllocation.enabled': 'true',
            'spark:spark.sql.adaptive.enabled': 'true'
        }
    }
}

# Databricks: 자동 최적화
# - Photon Engine: 네이티브 벡터화 실행
# - Auto Scaling: 워크로드에 따른 자동 확장/축소
# - Delta Lake: ACID 트랜잭션, Time Travel
# - Runtime 자동 튜닝
```

### 선택 기준

#### Dataproc을 선택해야 하는 경우
```python
# 1. GCP 네이티브 환경
# - BigQuery, Cloud Storage와 밀접한 연동
# - IAM, VPC 등 GCP 보안 정책 통합
# - Cloud Composer(Airflow)와 연동

# 2. 비용 최적화가 중요한 경우
# - Preemptible VM 활용 (70% 할인)
# - 단순한 배치 ETL 작업
# - 일시적인 클러스터 사용

# 실무 예시: 일간 배치 ETL
gcloud dataproc jobs submit pyspark \
    gs://bucket/daily_etl.py \
    --cluster=ephemeral-cluster \
    --region=asia-northeast1 \
    --properties='spark.executor.instances=20'
```

#### Databricks를 선택해야 하는 경우
```python
# 1. 복잡한 분석 워크로드
# - 머신러닝 파이프라인 통합
# - 스트리밍 + 배치 통합 처리
# - 협업 환경(노트북) 필요

# 2. 성능이 중요한 경우  
# - Delta Lake의 최적화된 스토리지
# - Photon Engine의 고성능 실행
# - Auto Scaling으로 리소스 효율성
```

## Ephemeral vs Long-running Cluster

### Ephemeral Cluster (임시 클러스터)

#### 개념 및 사용 사례
```bash
# Job별로 클러스터 생성 후 자동 삭제
gcloud dataproc jobs submit pyspark \
    gs://bucket/analysis.py \
    --cluster=temp-cluster-$(date +%s) \
    --region=asia-northeast1 \
    --cluster-labels=purpose=ephemeral,team=analytics \
    --max-idle=10m \  # 10분 유휴 시 자동 삭제
    --initialization-actions=gs://bucket/install-deps.sh

# 장점:
# 1. 비용 최적화: 사용한 시간만큼만 과금
# 2. 격리성: Job간 상호 영향 없음
# 3. 최신 환경: 매번 새로운 클러스터
```

#### 자동화된 Ephemeral Cluster 관리
```python
# Python으로 Ephemeral Cluster 관리
from google.cloud import dataproc_v1
import time

class EphemeralClusterManager:
    def __init__(self, project_id, region):
        self.project_id = project_id
        self.region = region
        self.client = dataproc_v1.ClusterControllerClient()
        
    def create_and_run_job(self, job_script, cluster_size='small'):
        """클러스터 생성 → Job 실행 → 자동 삭제"""
        
        cluster_name = f"ephemeral-{int(time.time())}"
        
        # 클러스터 크기별 설정
        cluster_configs = {
            'small': {'workers': 2, 'machine_type': 'n1-standard-2'},
            'medium': {'workers': 4, 'machine_type': 'n1-standard-4'}, 
            'large': {'workers': 8, 'machine_type': 'n1-highmem-4'}
        }
        
        config = cluster_configs[cluster_size]
        
        # 클러스터 생성
        cluster_config = {
            'cluster_name': cluster_name,
            'config': {
                'master_config': {
                    'num_instances': 1,
                    'machine_type_uri': 'n1-standard-2'
                },
                'worker_config': {
                    'num_instances': config['workers'],
                    'machine_type_uri': config['machine_type'],
                    'is_preemptible': True  # 70% 할인
                },
                'software_config': {
                    'image_version': '2.0-debian10',
                    'properties': {
                        'spark:spark.sql.adaptive.enabled': 'true',
                        'spark:spark.dynamicAllocation.enabled': 'true'
                    }
                },
                'lifecycle_config': {
                    'auto_delete_time': {
                        'seconds': int(time.time()) + 3600  # 1시간 후 자동 삭제
                    }
                }
            }
        }
        
        # 클러스터 생성 대기
        operation = self.client.create_cluster(
            project_id=self.project_id,
            region=self.region,
            cluster=cluster_config
        )
        operation.result()  # 생성 완료까지 대기
        
        # Job 실행
        job_client = dataproc_v1.JobControllerClient()
        job = {
            'placement': {'cluster_name': cluster_name},
            'pyspark_job': {
                'main_python_file_uri': job_script,
                'properties': {
                    'spark.executor.memory': '3g',
                    'spark.executor.cores': '2'
                }
            }
        }
        
        job_operation = job_client.submit_job(
            project_id=self.project_id,
            region=self.region,
            job=job
        )
        
        return cluster_name, job_operation.job_id

# 사용 예시
manager = EphemeralClusterManager('my-project', 'asia-northeast1')
cluster_name, job_id = manager.create_and_run_job(
    'gs://bucket/daily_analysis.py', 
    'medium'
)
```

### Long-running Cluster (상시 클러스터)

#### 개념 및 사용 사례
```bash
# 장기간 유지되는 클러스터 생성
gcloud dataproc clusters create analytics-cluster \
    --region=asia-northeast1 \
    --zone=asia-northeast1-a \
    --num-masters=1 \
    --num-workers=10 \
    --num-preemptible-workers=20 \  # Secondary Worker (70% 할인)
    --worker-machine-type=n1-highmem-4 \
    --preemptible-worker-machine-type=n1-standard-4 \
    --enable-autoscaling \  # Auto Scaling 활성화
    --max-workers=30 \
    --secondary-worker-type=preemptible

# 장점:
# 1. 즉시 Job 실행 (클러스터 생성 시간 없음)
# 2. 상태 유지 (캐시, 중간 결과)
# 3. 반복 작업에 효율적
```

#### Auto Scaling 최적화
```python
# Auto Scaling 정책 설정
def create_autoscaling_cluster():
    cluster_config = {
        'cluster_name': 'auto-scaling-cluster',
        'config': {
            'master_config': {
                'num_instances': 1,
                'machine_type_uri': 'n1-standard-4'
            },
            'worker_config': {
                'num_instances': 5,  # 기본 Worker
                'machine_type_uri': 'n1-highmem-4'
            },
            'secondary_worker_config': {  # Preemptible Worker
                'num_instances': 0,  # 초기값 (Auto Scaling으로 조절)
                'machine_type_uri': 'n1-standard-4',
                'is_preemptible': True
            },
            'software_config': {
                'properties': {
                    # Dynamic Allocation 설정
                    'spark:spark.dynamicAllocation.enabled': 'true',
                    'spark:spark.dynamicAllocation.minExecutors': '2',
                    'spark:spark.dynamicAllocation.maxExecutors': '50',
                    'spark:spark.dynamicAllocation.initialExecutors': '5',
                    
                    # Auto Scaling 최적화
                    'dataproc:dataproc.autoscaling.upscaling.policy': 'MULTIPLE',
                    'dataproc:dataproc.autoscaling.downscaling.policy': 'GRACEFUL'
                }
            }
        }
    }
    
    return cluster_config

# 실시간 스케일링 모니터링
def monitor_cluster_scaling(cluster_name):
    while True:
        # 클러스터 상태 확인
        cluster = client.get_cluster(
            project_id=PROJECT_ID,
            region=REGION,
            cluster_name=cluster_name
        )
        
        primary_workers = cluster.config.worker_config.num_instances
        secondary_workers = cluster.config.secondary_worker_config.num_instances
        
        print(f"Primary Workers: {primary_workers}")
        print(f"Secondary Workers: {secondary_workers}")
        print(f"Total Workers: {primary_workers + secondary_workers}")
        
        # 5분마다 확인
        time.sleep(300)
```

## 오토스케일링 최적화

### 스케일링 정책 설정

#### Upscaling (확장) 최적화
```python
# 점진적 확장 vs 급격한 확장
upscaling_policies = {
    'conservative': {
        # 보수적 확장 (비용 우선)
        'scale_up_factor': 0.05,  # 5%씩 확장
        'scale_up_min_worker_fraction': 0.0,
        'secondary_worker_fraction': 0.8  # Secondary 80%
    },
    'aggressive': {
        # 공격적 확장 (성능 우선)
        'scale_up_factor': 0.2,   # 20%씩 확장
        'scale_up_min_worker_fraction': 0.0,
        'secondary_worker_fraction': 0.6  # Secondary 60%
    }
}

def apply_scaling_policy(policy_type='conservative'):
    policy = upscaling_policies[policy_type]
    
    cluster_config = {
        'secondary_worker_config': {
            'is_preemptible': True,
            'num_instances': 0  # Auto Scaling으로 관리
        },
        'software_config': {
            'properties': {
                f'dataproc:dataproc.autoscaling.secondary.fraction': 
                    str(policy['secondary_worker_fraction'])
            }
        }
    }
    
    return cluster_config
```

#### Downscaling (축소) 최적화
```python
# Graceful Downscaling 설정
def configure_graceful_downscaling():
    """안전한 축소를 위한 설정"""
    
    return {
        'software_config': {
            'properties': {
                # Executor가 완료될 때까지 대기
                'spark:spark.dynamicAllocation.executorIdleTimeout': '60s',
                'spark:spark.dynamicAllocation.cachedExecutorIdleTimeout': '300s',
                
                # Dataproc Auto Scaling
                'dataproc:dataproc.autoscaling.downscaling.policy': 'GRACEFUL',
                'dataproc:dataproc.autoscaling.cooldown.period': '2m',
                
                # Preemptible 인스턴스 우선 제거
                'dataproc:dataproc.autoscaling.preemptible.downscaling': 'true'
            }
        }
    }
```

### 비용 최적화: Preemptible VM 활용

#### Preemptible VM 전략
```python
# 70% 할인된 Preemptible VM 최대 활용
def create_cost_optimized_cluster():
    """비용 최적화된 클러스터 생성"""
    
    cluster_config = {
        'master_config': {
            'num_instances': 1,
            'machine_type_uri': 'n1-standard-2',  # Master는 Primary만
            'disk_config': {
                'boot_disk_size_gb': 100,
                'boot_disk_type': 'pd-standard'  # SSD보다 저렴
            }
        },
        'worker_config': {
            # Primary Worker 최소화 (안정성 보장)
            'num_instances': 3,
            'machine_type_uri': 'n1-highmem-2'
        },
        'secondary_worker_config': {
            # Secondary Worker 최대화 (비용 절감)
            'num_instances': 0,  # Auto Scaling으로 관리
            'machine_type_uri': 'n1-standard-4',
            'is_preemptible': True,
            'disk_config': {
                'boot_disk_size_gb': 100,
                'boot_disk_type': 'pd-standard'
            }
        }
    }
    
    return cluster_config

# Preemptible 중단 대응 전략
def handle_preemptible_interruption():
    """Preemptible VM 중단 시 대응 방법"""
    
    spark_config = {
        # Checkpoint 활성화 (복구를 위한)
        'spark.sql.streaming.checkpointLocation': 'gs://bucket/checkpoints/',
        'spark.checkpoint.dir': 'gs://bucket/spark-checkpoints/',
        
        # 중간 결과 저장 주기 단축
        'spark.sql.streaming.trigger.processingTime': '30 seconds',
        
        # 실패 시 재시도 정책
        'spark.sql.streaming.maxRetries': '3',
        'spark.sql.adaptive.enabled': 'true'
    }
    
    return spark_config
```

## 실무 사례: 월 2천만원 절감 최적화

### 기존 문제 상황
```python
# 기존 클러스터 구성 (비용 비효율)
original_setup = {
    'cluster_type': 'Long-running',
    'duration': '24시간 × 30일',
    'composition': {
        'master': 'n1-highmem-4 × 1',
        'workers': 'n1-highmem-8 × 20'  # 모두 Primary Worker
    },
    'monthly_cost': '₩20,000,000',  # 월 2천만원
    'utilization': '30%'  # 70% 유휴 시간
}

# 비용 계산
def calculate_original_cost():
    # n1-highmem-4 (Master): $0.2376/hour
    # n1-highmem-8 (Worker): $0.4752/hour × 20대
    
    master_cost = 0.2376 * 24 * 30  # $171.07
    worker_cost = 0.4752 * 20 * 24 * 30  # $6,843.64
    
    total_usd = master_cost + worker_cost  # $7,014.71
    total_krw = total_usd * 1350  # ₩9,469,858 (환율 1,350원)
    
    # Dataproc 수수료 10% 추가
    return total_krw * 1.1  # ₩10,416,844
```

### 최적화 과정

#### 1단계: 워크로드 분석 및 클러스터 세분화
```python
# 워크로드 패턴 분석
workload_patterns = {
    'morning_batch': {
        'time': '02:00-06:00',
        'frequency': 'daily',
        'resource_need': 'high',
        'duration': '2-3시간'
    },
    'realtime_streaming': {
        'time': '08:00-20:00', 
        'frequency': 'continuous',
        'resource_need': 'medium',
        'duration': '12시간'
    },
    'weekly_report': {
        'time': '일요일 새벽',
        'frequency': 'weekly',
        'resource_need': 'very_high', 
        'duration': '4-5시간'
    }
}

# 워크로드별 클러스터 분리
def create_workload_specific_clusters():
    clusters = {}
    
    # 1) 일간 배치용 Ephemeral Cluster
    clusters['daily_batch'] = {
        'type': 'ephemeral',
        'schedule': '매일 02:00',
        'config': {
            'workers': 15,
            'machine_type': 'n1-highmem-4',
            'preemptible_ratio': 0.8,  # 80% Preemptible
            'auto_delete': '4시간 후'
        }
    }
    
    # 2) 스트리밍용 Long-running Cluster (최소 사이즈)
    clusters['streaming'] = {
        'type': 'long_running', 
        'schedule': '08:00-20:00',
        'config': {
            'workers': 5,  # 기존 20대 → 5대
            'machine_type': 'n1-standard-4',
            'auto_scaling': True,
            'max_workers': 15
        }
    }
    
    # 3) 주간 리포트용 Ephemeral Cluster  
    clusters['weekly_report'] = {
        'type': 'ephemeral',
        'schedule': '일요일 01:00',
        'config': {
            'workers': 30,
            'machine_type': 'n1-highmem-8', 
            'preemptible_ratio': 0.7,
            'auto_delete': '6시간 후'
        }
    }
    
    return clusters
```

#### 2단계: Preemptible VM 비율 최대화
```python
# Preemptible/Primary Worker 비율 최적화
def optimize_worker_mix():
    """안정성과 비용의 균형점 찾기"""
    
    configs = {
        'conservative': {  # 안정성 우선
            'primary_workers': 5,
            'preemptible_workers': 10,
            'preemptible_ratio': 0.67,  # 67%
            'cost_saving': 0.47  # 47% 절감
        },
        'balanced': {  # 균형
            'primary_workers': 3, 
            'preemptible_workers': 12,
            'preemptible_ratio': 0.8,  # 80%
            'cost_saving': 0.56  # 56% 절감  
        },
        'aggressive': {  # 비용 우선
            'primary_workers': 2,
            'preemptible_workers': 18, 
            'preemptible_ratio': 0.9,  # 90%
            'cost_saving': 0.63  # 63% 절감
        }
    }
    
    # 실무에서는 Balanced 선택 (80% Preemptible)
    return configs['balanced']

# Preemptible 중단 대비 Checkpointing 강화
def enhance_fault_tolerance():
    return {
        'spark.sql.streaming.checkpointLocation': 'gs://bucket/checkpoints/',
        'spark.sql.streaming.trigger.processingTime': '1 minute',  # 체크포인트 간격 단축
        'spark.serializer': 'org.apache.spark.serializer.KryoSerializer',
        'spark.sql.adaptive.enabled': 'true',
        'spark.sql.adaptive.coalescePartitions.enabled': 'true'
    }
```

#### 3단계: 스케줄링 최적화
```python
# Airflow DAG로 효율적인 클러스터 스케줄링
from airflow import DAG
from airflow.providers.google.cloud.operators.dataproc import (
    DataprocCreateClusterOperator,
    DataprocSubmitJobOperator, 
    DataprocDeleteClusterOperator
)

def create_optimized_etl_dag():
    with DAG(
        'cost_optimized_etl',
        schedule_interval='0 2 * * *',  # 새벽 2시
        catchup=False
    ) as dag:
        
        # 클러스터 생성 (필요할 때만)
        create_cluster = DataprocCreateClusterOperator(
            task_id='create_ephemeral_cluster',
            cluster_name='batch-{{ ds_nodash }}',
            cluster_config={
                'master_config': {
                    'num_instances': 1,
                    'machine_type_uri': 'n1-standard-2'
                },
                'worker_config': {
                    'num_instances': 3,  # Primary Worker 최소
                    'machine_type_uri': 'n1-highmem-4'
                },
                'secondary_worker_config': {
                    'num_instances': 12,  # Preemptible Worker 다수
                    'machine_type_uri': 'n1-standard-4',
                    'is_preemptible': True
                },
                'lifecycle_config': {
                    'auto_delete_ttl': '14400s'  # 4시간 후 자동 삭제
                }
            },
            region='asia-northeast1'
        )
        
        # ETL Job 실행
        submit_etl = DataprocSubmitJobOperator(
            task_id='run_daily_etl',
            job={
                'placement': {'cluster_name': 'batch-{{ ds_nodash }}'},
                'pyspark_job': {
                    'main_python_file_uri': 'gs://bucket/daily_etl.py',
                    'properties': {
                        'spark.executor.memory': '6g',
                        'spark.executor.cores': '2',
                        'spark.sql.adaptive.enabled': 'true'
                    }
                }
            },
            region='asia-northeast1'
        )
        
        # 클러스터 정리 (비용 절약)
        delete_cluster = DataprocDeleteClusterOperator(
            task_id='cleanup_cluster',
            cluster_name='batch-{{ ds_nodash }}',
            region='asia-northeast1'
        )
        
        create_cluster >> submit_etl >> delete_cluster
    
    return dag
```

### 최적화 결과
```python
# 최적화 후 비용 구조
optimized_setup = {
    'cluster_strategy': 'Mixed (Ephemeral + Long-running)',
    'preemptible_ratio': 0.8,  # 80% Preemptible
    'utilization_improvement': '30% → 85%',
    'cost_breakdown': {
        'daily_batch': '₩150,000/일 × 30일 = ₩4,500,000',
        'streaming': '₩100,000/일 × 30일 = ₩3,000,000', 
        'weekly_report': '₩500,000/주 × 4주 = ₩2,000,000'
    },
    'total_monthly_cost': '₩9,500,000',
    'cost_saving': '₩10,500,000 (52% 절감)'
}

def cost_optimization_summary():
    """최적화 요약"""
    
    improvements = {
        '클러스터 세분화': {
            'before': '단일 대형 클러스터 (20대)',
            'after': '워크로드별 최적화된 클러스터',
            'saving': '30%'
        },
        'Preemptible VM 활용': {
            'before': '모든 Worker가 Primary (정가)',
            'after': '80% Preemptible (70% 할인)',
            'saving': '56%'
        },
        'Auto Scaling': {
            'before': '고정 크기 (과잉 프로비저닝)',
            'after': '수요에 맞춰 동적 조절',
            'saving': '40%'
        },
        'Ephemeral 클러스터': {
            'before': '24시간 상시 운영',
            'after': '필요할 때만 생성/삭제',
            'saving': '70%'
        }
    }
    
    total_saving = 20_000_000 - 9_500_000  # ₩10,500,000
    saving_percentage = (total_saving / 20_000_000) * 100  # 52.5%
    
    return {
        'monthly_saving': f'₩{total_saving:,}',
        'saving_percentage': f'{saving_percentage:.1f}%',
        'annual_saving': f'₩{total_saving * 12:,}',  # 연간 1억 2천 6백만원 절감
        'improvements': improvements
    }

optimization_result = cost_optimization_summary()
print(f"월간 절감액: {optimization_result['monthly_saving']}")
print(f"절감률: {optimization_result['saving_percentage']}")
```

### 모니터링 및 지속적 최적화
```python
# 비용 및 성능 모니터링 시스템
import pandas as pd
from google.cloud import monitoring_v3

def monitor_dataproc_efficiency():
    """Dataproc 클러스터 효율성 모니터링"""
    
    client = monitoring_v3.MetricServiceClient()
    
    # 1) 비용 추적
    def track_daily_costs():
        # BigQuery로 Dataproc 비용 분석
        cost_query = """
        SELECT 
            DATE(usage_start_time) as date,
            service.description as service,
            location.location as region,
            SUM(cost) as daily_cost
        FROM `project.billing.gcp_billing_export`
        WHERE service.description LIKE '%Dataproc%'
            AND DATE(usage_start_time) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
        GROUP BY date, service, region
        ORDER BY date DESC
        """
        return cost_query
    
    # 2) 리소스 사용률 모니터링
    def monitor_cluster_utilization():
        metrics_to_check = [
            'compute.googleapis.com/instance/cpu/utilization',
            'compute.googleapis.com/instance/memory/utilization',
            'dataproc.googleapis.com/cluster/spark/jobs'
        ]
        
        for metric in metrics_to_check:
            # CloudWatch 스타일 메트릭 수집 로직
            pass
    
    # 3) 자동 알림 설정
    def setup_cost_alerts():
        alert_conditions = {
            'daily_budget_exceeded': {
                'threshold': 500_000,  # 일일 50만원 초과 시
                'notification': 'slack://data-team'
            },
            'cluster_idle_time': {
                'threshold': 0.2,  # 사용률 20% 미만 시
                'action': 'auto_scale_down'
            }
        }
        return alert_conditions

# 지속적 개선을 위한 A/B 테스트
def ab_test_cluster_configs():
    """클러스터 설정 A/B 테스트"""
    
    test_configs = {
        'config_a': {  # 현재 설정
            'preemptible_ratio': 0.8,
            'machine_type': 'n1-standard-4',
            'expected_cost': 300_000
        },
        'config_b': {  # 테스트 설정
            'preemptible_ratio': 0.9,
            'machine_type': 'n1-highmem-2', 
            'expected_cost': 250_000
        }
    }
    
    # 성능 지표 비교
    comparison_metrics = [
        'job_completion_time',
        'cost_per_job',
        'failure_rate',
        'resource_utilization'
    ]
    
    return test_configs, comparison_metrics
```

## 면접 단골 질문과 답변

### Q: Dataproc vs Databricks 차이는?
**1분 답변**: "Dataproc은 GCP 네이티브 서비스로 BigQuery, Cloud Storage와 완벽 연동되고 비용이 저렴합니다. Databricks는 전용 Spark 최적화 플랫폼으로 Photon Engine, Delta Lake 등 고급 기능을 제공하지만 DBU 비용이 추가됩니다. GCP 환경에서 단순한 ETL이면 Dataproc, 복잡한 ML 파이프라인이면 Databricks가 적합합니다. 실제 프로젝트에서 Dataproc으로 월 2천만원을 절감했습니다."

### Q: Ephemeral Cluster는 언제 사용하나요?
**답변 포인트**:
1. 배치 ETL처럼 일회성 작업에 최적
2. Job별 격리로 안정성 확보
3. 사용한 시간만큼만 과금되어 비용 효율적

### Q: Preemptible VM 사용 시 주의사항은?
**답변 포인트**:
1. 24시간 내 중단 가능성 (70% 할인 대가)
2. Checkpointing으로 중단 시 복구 보장
3. Primary Worker 일부 유지로 안정성 확보

### Q: Auto Scaling은 어떻게 동작하나요?
**답변 포인트**:
1. Spark Dynamic Allocation과 연동
2. Pending Task 수에 따라 Worker 자동 추가/제거
3. Preemptible Worker 우선 활용으로 비용 최적화