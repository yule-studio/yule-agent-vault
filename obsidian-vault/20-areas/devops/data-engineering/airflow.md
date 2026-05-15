---
title: "Airflow — DAG / Operator / Scheduler"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:06:00+09:00
tags: [devops, data-engineering, airflow]
---

# Airflow — DAG / Operator / Scheduler

**[[data-engineering|↑ data-engineering]]**

---

## 1. 무엇

```
data pipeline 의 orchestrator.
Python 으로 workflow (DAG) 정의.

DAG = Directed Acyclic Graph
  - task 들의 의존성
  - 시간 (schedule)
  - retry / SLA / notify
```

→ data 엔지니어링 표준 (Netflix / Airbnb 시작).

---

## 2. 대안

| | 무엇 |
| --- | --- |
| **Airflow** | 가장 표준, mature |
| **Dagster** (★ modern) | type-safe, asset-oriented |
| **Prefect** | Python native, cloud first |
| **Argo Workflows** | k8s native |
| **Step Functions** | AWS managed |
| **Cloud Composer / MWAA** | managed Airflow |

→ 기존 = Airflow. 새 = Dagster.

---

## 3. DAG 예

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'email_on_failure': True,
    'email': ['data@example.com'],
    'sla': timedelta(hours=2)
}

with DAG(
    dag_id='daily_user_metrics',
    default_args=default_args,
    description='Daily user metrics ETL',
    schedule_interval='0 2 * * *',         # 매일 02:00
    start_date=datetime(2026, 1, 1),
    catchup=False,                          # missed run 건너뜀
    max_active_runs=1,
    tags=['data', 'daily', 'metrics']
) as dag:

    extract = PythonOperator(
        task_id='extract_from_db',
        python_callable=extract_users,
        op_kwargs={'date': '{{ ds }}'}      # macro: execution date
    )

    transform = PythonOperator(
        task_id='transform_user_data',
        python_callable=transform_users
    )

    load = PythonOperator(
        task_id='load_to_warehouse',
        python_callable=load_to_snowflake
    )

    extract >> transform >> load            # 의존성
```

---

## 4. Operator 종류

```
PythonOperator        — Python 함수
BashOperator           — shell command
SQLOperator (Postgres/MySQL/BQ) — SQL
KubernetesPodOperator  — k8s pod 실행 (★ scalable)
DockerOperator         — Docker container
EmailOperator          — 메일
HttpOperator           — REST API
SimpleHttpOperator     — light HTTP
SparkSubmitOperator    — Spark job
S3KeySensor            — S3 file 대기
ExternalTaskSensor     — 다른 DAG 대기
BranchPythonOperator   — 조건 분기
TriggerDagRunOperator  — 다른 DAG trigger
```

→ Operator + Sensor + Hook.

---

## 5. TaskFlow API (★ modern)

```python
from airflow.decorators import dag, task
from datetime import datetime

@dag(
    schedule_interval='@daily',
    start_date=datetime(2026, 1, 1),
    catchup=False,
    tags=['modern']
)
def user_metrics():
    
    @task
    def extract(date: str):
        return query_db(date)
    
    @task
    def transform(rows: list):
        return [process(r) for r in rows]
    
    @task
    def load(rows: list):
        save_to_warehouse(rows)
    
    load(transform(extract("{{ ds }}")))

user_metrics()
```

→ Python 의 type / return value 자동 연결 (XCom).

→ Airflow 2.0+ 의 표준.

---

## 6. XCom (task 간 data)

```python
@task
def extract():
    return {"count": 100}     # 자동 XCom push

@task
def report(data):
    print(data["count"])      # 자동 pull

report(extract())
```

→ 작은 data 만 (max 48KB SQLite / 1GB Postgres backend).

→ 큰 data = S3 / 외부 storage 의 path 만 pass.

---

## 7. KubernetesPodOperator (★ scalable)

```python
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import KubernetesPodOperator

extract = KubernetesPodOperator(
    task_id='extract',
    name='extract-pod',
    namespace='airflow',
    image='myorg/extract:1.0',
    cmds=['python', 'extract.py'],
    arguments=['--date', '{{ ds }}'],
    resources={'request_cpu': '500m', 'request_memory': '1Gi'},
    is_delete_operator_pod=True,
    in_cluster=True
)
```

→ Airflow worker 가 직접 처리 안 함. pod 띄움 → 자원 격리.

---

## 8. scheduler / executor

```
executor 종류:
  - SequentialExecutor   — 한 번에 1 task (SQLite)
  - LocalExecutor        — 단일 host parallel
  - CeleryExecutor       — 분산 (Redis / RabbitMQ broker)
  - KubernetesExecutor (★) — task 마다 pod
  - CeleryKubernetes     — 둘 다
```

→ production = K8s executor 권장 (자원 격리 + autoscale).

---

## 9. backfill / catchup

```
catchup=True + start_date past:
  → 과거 모든 schedule 자동 run

catchup=False (★ 권장 default):
  → 최신만 run

manual backfill:
  airflow dags backfill my_dag -s 2026-01-01 -e 2026-01-31

→ 신중. 큰 batch 가 production load 영향.
```

---

## 10. monitoring / SLA

```python
default_args = {
    'sla': timedelta(hours=2),         # 2시간 안에 끝나야
    'sla_miss_callback': notify_oncall
}

# 또는 task 별
task = PythonOperator(
    task_id='heavy_task',
    sla=timedelta(minutes=30)
)
```

```
UI:
  - DAG run status (success / failed / running)
  - task duration trend
  - SLA miss log
  - dependency graph
```

---

## 11. dynamic DAG / task

```python
# task mapping (★ 2.3+)
@task
def process(file: str):
    return load(file)

files = ["file1.csv", "file2.csv", "file3.csv"]
process.expand(file=files)
# → 3 task 자동 생성, parallel
```

```python
# dynamic DAG from config
for client in ["clientA", "clientB", "clientC"]:
    @dag(dag_id=f"etl_{client}", schedule='@daily', ...)
    def make_dag():
        @task
        def extract(): ...
        @task
        def load(): ...
        extract() >> load()
    
    make_dag()
```

---

## 12. testing

```python
# unit test
def test_transform():
    result = transform_users([{"id": 1, "name": "alice"}])
    assert result[0]["normalized_name"] == "ALICE"

# DAG test
def test_dag_loaded():
    dagbag = DagBag(include_examples=False)
    assert "user_metrics" in dagbag.dags
    assert len(dagbag.dags["user_metrics"].tasks) == 3
```

```bash
# CLI test (no scheduler 안 띄움)
airflow tasks test user_metrics extract_from_db 2026-05-15
```

---

## 13. CI / CD

```
DAG 변경:
  PR review
  → unit test + DAG test
  → linter (black / isort / mypy)
  → deploy (git pull on Airflow webserver/scheduler)
  → 또는 GitSync sidecar
  
DAG 의 secret:
  - Variable (UI 의 secret)
  - Connection (UI 또는 env var)
  - Vault / Secrets Manager 통합
```

---

## 14. 운영 (★ production)

```
☐ HA scheduler (Airflow 2 multi-scheduler)
☐ DB = managed Postgres
☐ executor = Kubernetes
☐ secret = Vault / SM
☐ DAG sync = GitSync
☐ logs = S3 / GCS
☐ metric = StatsD → Prometheus
☐ alert = PagerDuty (SLA miss / task fail)
☐ backup (DB)
☐ ResourceQuota (namespace)
☐ owner / team tag
```

---

## 15. 함정

1. **`catchup=True` + 큰 start_date** — 수백 backfill 폭주.
2. **XCom 큰 data** — DB 부담. S3 path 전달.
3. **task 의 side effect** — non-idempotent.
4. **scheduler 1대** — SPOF. HA 2+.
5. **PythonOperator + heavy library** — worker memory 부담. K8s operator.
6. **DAG file 의 top-level code 무거움** — every parse 마다 실행.
7. **time zone 헷갈림** — 명시적 timezone 사용.

---

## 16. 관련

- [[data-engineering|↑ data-engineering]]
- [[spark]]
- [[dbt]]
- [[../platform-engineering/scaffolding|↗ scaffolding]]
