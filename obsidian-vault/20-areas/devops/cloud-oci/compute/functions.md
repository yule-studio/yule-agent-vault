---
title: "OCI Functions"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:20:00+09:00
tags:
  - oci
  - compute
  - functions
  - serverless
---

# OCI Functions

**[[compute|↑ Compute]]** · **[[../cloud-oci|↑↑ OCI]]**

---

## 1. 한 줄

OCI 의 serverless — Fn Project (open-source) 기반. Lambda 동등.

---

## 2. 사용

```bash
# Fn CLI
brew install fn

fn init --runtime python myfunc
cd myfunc
# func.py + func.yaml

fn create app myapp --annotation oracle.com/oci/subnetIds='["<subnet-ocid>"]'
fn -v deploy --app myapp
fn invoke myapp myfunc
```

```python
# func.py
import io, json
from fdk import response

def handler(ctx, data: io.BytesIO=None):
    body = json.loads(data.getvalue()) if data else {}
    return response.Response(ctx, response_data={"hello": body.get("name", "world")})
```

```yaml
# func.yaml
schema_version: 20180708
name: myfunc
version: 0.0.1
runtime: python
build_image: fnproject/python:3.11-dev
run_image: fnproject/python:3.11
entrypoint: /python/bin/fdk /function/func.py handler
memory: 256
```

---

## 3. Trigger

- HTTP — API Gateway
- Event Service — OCI 자원 변경
- Streaming
- 다른 function

---

## 4. 가격

```
$0.0000002 / invocation
$0.0000017 / GB·s
Free: 2M invocation + 400K GB·s / month
```

→ Lambda 보다 약간 저렴.

---

## 5. 함정

- container image 기반 — 보통 100MB+ → cold start 느림
- max 5분
- VCN 통합 = 필수 (private 자원 접근)
- max concurrency / memory per function

---

## 6. 관련

- [[compute]]
- [[../messaging/streaming]]
