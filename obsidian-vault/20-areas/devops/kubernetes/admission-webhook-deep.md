---
title: "Admission Webhook deep — Validating / Mutating / Conversion"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:30:00+09:00
tags: [devops, kubernetes, admission-webhook, policy]
---

# Admission Webhook deep — Validating / Mutating / Conversion

**[[kubernetes|↑ kubernetes]]**

---

## 1. 무엇

```
kubectl apply / API request 의 lifecycle:

  authentication → authorization → mutating admission
                                    → validating admission
                                    → object schema validation
                                    → etcd 저장

admission webhook = 위 cycle 안에서:
  - mutating: 객체 변경 (default 값 / sidecar 자동 주입)
  - validating: 거부 / 허용 결정 (정책)
  - conversion: CRD version 변환 (v1 ↔ v2)
```

→ 모든 cluster policy 의 enforcement layer.

---

## 2. mutating vs validating

```
mutating:
  - object 의 spec 수정 가능
  - 같은 admission 가 여러 번 호출될 수 있음 (idempotent 필요)
  - 예: sidecar 주입 (Istio), default 값, label / annotation 추가
  
validating:
  - object 검증만 (수정 X)
  - 마지막 단계, 변경 후 검증
  - 예: 보안 정책, naming 규칙, resource limit 강제
  
순서:
  mutating → validating → etcd
```

→ 보통 같이 사용 (mutate 후 validate).

---

## 3. 흔한 사용 사례

```
1. sidecar 자동 주입
   - Istio (Envoy)
   - Linkerd (proxy)
   - Vault (secret injection)
   - DataDog agent
   
2. default 값 적용
   - resource limit 자동
   - imagePullPolicy 설정
   - probe 기본값
   
3. 정책 강제
   - 모든 pod 가 runAsNonRoot
   - image 가 signed
   - resource limit 명시
   - production namespace 의 replica >= 2
   
4. label / annotation
   - team / cost-center tag 강제
   - owner 자동 설정
   
5. 보안
   - privileged container 금지
   - hostNetwork / hostPath 금지
   - 특정 image registry 만 허용
```

---

## 4. 구조

```
[API server]
   │
   ↓ webhook 호출 (HTTPS POST)
[Webhook Server]
   │ AdmissionReview 객체 받음:
   │   - request: object 정보
   │ 처리:
   │   - allowed: bool
   │   - patch: [{op: "add", path: "...", value: ...}]
   │   - status: 메시지
   │ response 반환
[API server]
   │
   ↓ 결정 따라 진행 또는 거부
```

---

## 5. webhook server 작성 (Go)

```go
package main

import (
    "encoding/json"
    "io"
    "net/http"
    
    admissionv1 "k8s.io/api/admission/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func mutateHandler(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(r.Body)
    
    var review admissionv1.AdmissionReview
    json.Unmarshal(body, &review)
    
    var pod corev1.Pod
    json.Unmarshal(review.Request.Object.Raw, &pod)
    
    // mutation: label 추가
    patch := []byte(`[
        {"op": "add", "path": "/metadata/labels/managed-by", "value": "webhook"}
    ]`)
    
    pt := admissionv1.PatchTypeJSONPatch
    response := admissionv1.AdmissionResponse{
        UID:       review.Request.UID,
        Allowed:   true,
        Patch:     patch,
        PatchType: &pt,
    }
    
    review.Response = &response
    review.Request = nil
    
    respBytes, _ := json.Marshal(review)
    w.Header().Set("Content-Type", "application/json")
    w.Write(respBytes)
}

func validateHandler(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(r.Body)
    
    var review admissionv1.AdmissionReview
    json.Unmarshal(body, &review)
    
    var pod corev1.Pod
    json.Unmarshal(review.Request.Object.Raw, &pod)
    
    allowed := true
    var reason string
    
    // 검증: privileged container 금지
    for _, c := range pod.Spec.Containers {
        if c.SecurityContext != nil && c.SecurityContext.Privileged != nil && *c.SecurityContext.Privileged {
            allowed = false
            reason = "privileged container is not allowed"
            break
        }
    }
    
    response := admissionv1.AdmissionResponse{
        UID:     review.Request.UID,
        Allowed: allowed,
    }
    if !allowed {
        response.Result = &metav1.Status{
            Message: reason,
        }
    }
    
    review.Response = &response
    review.Request = nil
    
    respBytes, _ := json.Marshal(review)
    w.Header().Set("Content-Type", "application/json")
    w.Write(respBytes)
}

func main() {
    http.HandleFunc("/mutate", mutateHandler)
    http.HandleFunc("/validate", validateHandler)
    
    // TLS 필수
    http.ListenAndServeTLS(":8443", "/etc/tls/cert.pem", "/etc/tls/key.pem", nil)
}
```

---

## 6. webhook 등록

```yaml
# MutatingWebhookConfiguration
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata: {name: my-mutator}
webhooks:
  - name: pod-mutator.example.com
    clientConfig:
      service:
        namespace: webhook-system
        name: pod-mutator
        path: /mutate
      caBundle: <base64-encoded-ca>
    rules:
      - apiGroups: [""]
        apiVersions: [v1]
        operations: [CREATE, UPDATE]
        resources: [pods]
        scope: Namespaced
    namespaceSelector:
      matchLabels: {webhook: enabled}
    objectSelector:
      matchExpressions:
        - key: skip-webhook
          operator: DoesNotExist
    failurePolicy: Fail               # ★ Fail or Ignore
    matchPolicy: Equivalent
    sideEffects: None
    timeoutSeconds: 5
    admissionReviewVersions: [v1]
    reinvocationPolicy: Never         # Never or IfNeeded
```

→ 핵심 설정:

```
rules:
  apiGroups / apiVersions / operations / resources / scope
  → 어느 request 잡을 건지

failurePolicy:
  Fail = webhook fail = request 거부 (★ strict, 보안)
  Ignore = webhook fail = 그냥 허용 (덜 strict)

sideEffects:
  None / NoneOnDryRun / Some / Unknown
  → dry-run 시 호출 X

timeoutSeconds:
  webhook 응답 timeout (default 10s, max 30s)

reinvocationPolicy (mutating only):
  Never / IfNeeded
  → 다른 mutating webhook 가 변경 시 재호출
```

---

## 7. TLS 필수

```bash
# self-signed CA + cert
openssl req -x509 -newkey rsa:4096 -days 3650 -nodes \
    -keyout ca-key.pem -out ca.pem \
    -subj "/CN=Webhook CA"

# server cert
openssl req -newkey rsa:4096 -nodes -keyout server-key.pem \
    -out server-req.pem \
    -subj "/CN=pod-mutator.webhook-system.svc"

# SAN
cat > extfile.cnf <<EOF
subjectAltName = DNS:pod-mutator.webhook-system.svc, DNS:pod-mutator.webhook-system.svc.cluster.local
EOF

openssl x509 -req -in server-req.pem -CA ca.pem -CAkey ca-key.pem \
    -CAcreateserial -out server.pem -extfile extfile.cnf -days 365

# webhook config 의 caBundle 에 ca.pem 의 base64
base64 -w0 ca.pem
```

→ cert-manager 사용 권장 (자동 rotation).

---

## 8. cert-manager 통합 (★)

```yaml
# Issuer
apiVersion: cert-manager.io/v1
kind: Issuer
metadata: {name: webhook-issuer, namespace: webhook-system}
spec:
  selfSigned: {}

---
# CA
apiVersion: cert-manager.io/v1
kind: Certificate
metadata: {name: webhook-ca, namespace: webhook-system}
spec:
  isCA: true
  commonName: webhook-ca
  secretName: webhook-ca
  issuerRef: {name: webhook-issuer, kind: Issuer}

---
# server cert
apiVersion: cert-manager.io/v1
kind: Certificate
metadata: {name: webhook-server-cert, namespace: webhook-system}
spec:
  secretName: webhook-server-cert
  dnsNames:
    - pod-mutator.webhook-system.svc
    - pod-mutator.webhook-system.svc.cluster.local
  issuerRef: {name: webhook-ca-issuer, kind: Issuer}
```

```yaml
# WebhookConfiguration 의 caBundle 자동 주입
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: my-mutator
  annotations:
    cert-manager.io/inject-ca-from: webhook-system/webhook-server-cert
```

→ cert-manager 가 caBundle 자동 inject + cert rotation.

---

## 9. OPA Gatekeeper (★ ConstraintTemplate)

```yaml
# 1. Template (정책 정의)
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata: {name: k8srequiredlabels}
spec:
  crd:
    spec:
      names: {kind: K8sRequiredLabels}
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items: {type: string}
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        
        violation[{"msg": msg}] {
          required := input.parameters.labels
          provided := input.review.object.metadata.labels
          missing := required[_]
          not provided[missing]
          msg := sprintf("Required label missing: %v", [missing])
        }

---
# 2. Constraint (적용)
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata: {name: require-team-label}
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: [Pod, Service]
      - apiGroups: [apps]
        kinds: [Deployment]
    namespaces: [prod, staging]
  parameters:
    labels: [team, environment, owner]
```

```bash
# 위반 확인
kubectl get constraints
kubectl describe k8srequiredlabels.constraints.gatekeeper.sh require-team-label

# dryrun
spec:
  enforcementAction: dryrun     # 거부 X, log 만
```

→ Rego 언어. 정책 logic 의 portability.

---

## 10. Kyverno (★ YAML 만)

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: {name: require-labels}
spec:
  validationFailureAction: enforce       # enforce / audit
  background: true                        # 기존 자원도 audit
  rules:
    - name: check-team-label
      match:
        any:
          - resources:
              kinds: [Pod, Deployment, Service]
              namespaces: [prod, staging]
      validate:
        message: "team / environment / owner label required"
        pattern:
          metadata:
            labels:
              team: "?*"
              environment: "?*"
              owner: "?*"

    - name: add-default-label
      match: {any: [{resources: {kinds: [Pod]}}]}
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              managed-by: kyverno

    - name: generate-network-policy
      match: {any: [{resources: {kinds: [Namespace]}}]}
      generate:
        kind: NetworkPolicy
        name: default-deny
        namespace: "{{request.object.metadata.name}}"
        data:
          spec:
            podSelector: {}
            policyTypes: [Ingress, Egress]
```

→ Gatekeeper 보다 학습 ↓. validate / mutate / generate / verifyImages 모두.

---

## 11. Conversion Webhook (CRD version 변환)

```
CRD 의 v1alpha1 → v1beta1 → v1 진화:
  
  client 가 v1alpha1 요청 → API server 가 conversion webhook 호출
  → webhook 가 v1 으로 변환
  → 처리 후 다시 v1alpha1 으로
  
→ 같은 etcd 에 한 version 으로 저장 (storage version).
```

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata: {name: myapps.app.example.com}
spec:
  group: app.example.com
  versions:
    - name: v1alpha1
      served: true
      storage: false
    - name: v1
      served: true
      storage: true       # ★ etcd 저장 version
  conversion:
    strategy: Webhook
    webhook:
      clientConfig:
        service: {name: conversion-webhook, namespace: app-system, path: /convert}
        caBundle: <base64>
      conversionReviewVersions: [v1]
```

---

## 12. webhook 성능 / scale

```
issue:
  webhook 가 모든 request 검증 → cluster 의 throughput bottleneck
  
대응:
  - 빠른 응답 (< 100ms)
  - replica 3+ (HA)
  - cache (반복 쓰는 data)
  - rule 제한 (모든 resource X)
  - matchExpressions (선택적)
  - sidecar pattern (Istio 처럼 namespace selector)

timeout:
  default 10s
  너무 짧으면 fail
  너무 길면 API server 영향
  → 1-5s 권장
```

---

## 13. dry-run / impact 검증

```bash
# constraint apply 전 dryrun
spec:
  enforcementAction: dryrun

# 영향 분석
kubectl get k8srequiredlabels require-team-label -o jsonpath='{.status.violations}'

# 또는 Gatekeeper 의 audit
kubectl get config gatekeeper -n gatekeeper-system -o yaml

# 결과 review:
# 어느 자원이 위반?
# 어느 namespace?
# 즉시 enforce 시 영향?
```

→ enforce 전에 반드시 dryrun → audit log review → enforce.

---

## 14. 흔한 webhook 종류 (시니어가 관리)

```
필수:
  - cert-manager (자동 cert)
  - Istio sidecar injection
  - Vault agent injection
  
정책:
  - OPA Gatekeeper / Kyverno
  - Pod Security Admission (built-in)
  - Cosign (image signature)
  - Falco (audit)
  
운영:
  - VPA (vertical pod autoscale)
  - Kyverno (resource generation)
  - ExternalDNS

→ 너무 많은 webhook = cluster latency.
   필수만 + HA.
```

---

## 15. failurePolicy 결정

```
Fail (★ 보안 critical):
  webhook 가 fail = request 거부
  → 정책이 절대 우회 안 됨
  → 단 webhook 자체 fail 시 cluster 영향
  → kube-system 의 webhook 시 cluster 마비

Ignore:
  webhook 가 fail = 통과
  → cluster 영향 X
  → 단 정책 우회 가능
  
일반:
  - 보안 critical = Fail
  - sidecar injection = Ignore (webhook fail 시도 pod 동작)
  - kube-system 제외 (matchExpressions)
```

```yaml
# kube-system 제외 (★ 권장)
namespaceSelector:
  matchExpressions:
    - {key: kubernetes.io/metadata.name, operator: NotIn, values: [kube-system, cert-manager]}
```

---

## 16. monitoring (★)

```
metric:
  - webhook 호출 횟수
  - latency p50/p95/p99
  - error rate
  - response time
  - deny / allow ratio
  
alert:
  - webhook server down
  - latency > 1s
  - cert 만료 임박
  - error rate > 5%
```

```promql
# Gatekeeper
gatekeeper_validation_request_duration_seconds_bucket
gatekeeper_audit_last_run_time

# Kyverno
kyverno_admission_requests_total
kyverno_admission_review_duration_seconds_bucket
```

---

## 17. troubleshoot

```bash
# webhook 의 log
kubectl logs -n webhook-system deploy/my-webhook

# webhook 호출 확인 (API server log)
kubectl logs -n kube-system kube-apiserver-* | grep webhook

# 직접 webhook 테스트
curl -k -X POST https://webhook-svc:8443/validate \
    -H "Content-Type: application/json" \
    -d @admission-request.json

# Gatekeeper 의 violation
kubectl get constraints
kubectl get constrainttemplates

# Kyverno 의 report
kubectl get cpolr,polr -A
```

---

## 18. 함정

1. **failurePolicy=Fail + kube-system 포함** — cluster lockout.
2. **webhook 의 self-reference** — webhook 가 자기 자신 검증.
3. **TLS cert 만료** — webhook all fail.
4. **timeout 너무 짧음** — false negative.
5. **mutating webhook 의 non-idempotent** — reinvocationPolicy 와 충돌.
6. **caBundle 안 update** — cert rotation 시 fail.
7. **resource scope 잘못** — Pod 검증 인데 Deployment 시 안 함.
8. **dry-run 안 함** — 즉시 enforce → 운영 service 영향.
9. **webhook 의 HA 없음** — 1 replica fail = cluster 영향.
10. **rule 너무 광범위** — 모든 request 가 webhook 호출.

---

## 19. 관련

- [[kubernetes|↑ kubernetes]]
- [[operator-pattern]]
- [[rbac]]
- [[../security-ops/devsecops|↗ DevSecOps]]
- [[../security-ops/supply-chain-security|↗ supply chain]]
