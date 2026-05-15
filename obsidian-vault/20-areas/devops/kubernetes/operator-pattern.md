---
title: "Operator pattern — Kubebuilder / Operator SDK"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:20:00+09:00
tags: [devops, kubernetes, operator, crd]
---

# Operator pattern — Kubebuilder / Operator SDK

**[[kubernetes|↑ kubernetes]]**

---

## 1. 무엇

```
Operator = 도메인 지식을 코드화한 controller.

전통:
  Postgres 운영자가:
    - install
    - backup
    - failover
    - upgrade
    - resize

Operator:
  PostgresCluster CRD 생성 →
    operator 가 자동:
    - install
    - backup schedule
    - automatic failover
    - rolling upgrade
    - resize
```

→ "human operator" 의 지식을 software 로.

---

## 2. 흔한 operator 들

| | 무엇 |
| --- | --- |
| **Prometheus Operator** | Prom / AlertManager 자동 |
| **cert-manager** | TLS cert 자동 |
| **ArgoCD** | GitOps |
| **Strimzi** | Kafka on k8s |
| **CrunchyData / Zalando** | Postgres |
| **Percona MongoDB Operator** | MongoDB |
| **Elastic Operator (ECK)** | Elasticsearch |
| **Tekton** | CI/CD |
| **Velero** | backup |
| **External Secrets** | secret sync |
| **Karpenter** | autoscaling |
| **NVIDIA GPU Operator** | GPU |

→ OperatorHub.io 에 수백 operator.

---

## 3. controller pattern

```
[CRD]: 사용자가 정의한 resource
       (예: PostgresCluster, KafkaTopic)

[Controller]: CRD 의 spec 을 보고 → 실제 자원 (Pod, Service 등) 생성
       reconciliation loop:
       
       while True:
           current_state = read_cluster()
           desired_state = read_crd()
           if current != desired:
               make_changes()
```

→ k8s 의 모든 controller 가 이 패턴.

---

## 4. CRD (Custom Resource Definition)

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresclusters.acid.zalan.do
spec:
  group: acid.zalan.do
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                instances:
                  type: integer
                  minimum: 1
                  maximum: 10
                postgres:
                  type: object
                  properties:
                    version: {type: string}
                    parameters: {type: object}
  scope: Namespaced
  names:
    plural: postgresclusters
    singular: postgrescluster
    kind: PostgresCluster
    shortNames: [pgc]
```

```yaml
# 사용자가 만든 CR
apiVersion: acid.zalan.do/v1
kind: PostgresCluster
metadata:
  name: my-db
spec:
  instances: 3
  postgres:
    version: "15"
```

→ kubectl get postgresclusters → 표준 k8s resource 처럼.

---

## 5. Kubebuilder (★ Go 권장)

```bash
# 설치
brew install kubebuilder

# 새 operator project
mkdir my-operator && cd my-operator
kubebuilder init --domain example.com --repo github.com/myorg/my-operator

# API + Controller 생성
kubebuilder create api --group app --version v1 --kind MyApp
```

```go
// api/v1/myapp_types.go
type MyAppSpec struct {
    Replicas int32  `json:"replicas"`
    Image    string `json:"image"`
    Domain   string `json:"domain,omitempty"`
}

type MyAppStatus struct {
    Phase             string             `json:"phase,omitempty"`
    AvailableReplicas int32              `json:"availableReplicas"`
    Conditions        []metav1.Condition `json:"conditions,omitempty"`
}

type MyApp struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec              MyAppSpec   `json:"spec,omitempty"`
    Status            MyAppStatus `json:"status,omitempty"`
}
```

```go
// controllers/myapp_controller.go
func (r *MyAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var app appv1.MyApp
    if err := r.Get(ctx, req.NamespacedName, &app); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 1. desired state 계산
    deploy := makeDeployment(&app)
    
    // 2. 현재 state 와 비교 + 적용
    if err := ctrl.SetControllerReference(&app, deploy, r.Scheme); err != nil {
        return ctrl.Result{}, err
    }
    
    found := &appsv1.Deployment{}
    err := r.Get(ctx, types.NamespacedName{Name: deploy.Name, Namespace: deploy.Namespace}, found)
    if err != nil && errors.IsNotFound(err) {
        // create
        return ctrl.Result{}, r.Create(ctx, deploy)
    } else if err == nil {
        // update if diff
        if !deploymentEqual(found, deploy) {
            return ctrl.Result{}, r.Update(ctx, deploy)
        }
    }
    
    // 3. status update
    app.Status.AvailableReplicas = found.Status.AvailableReplicas
    return ctrl.Result{}, r.Status().Update(ctx, &app)
}
```

---

## 6. capability level

```
1. Basic Install     — install + uninstall
2. Seamless Upgrade  — operator 자체 upgrade
3. Full Lifecycle    — backup / restore / scale
4. Deep Insights     — metric / log / alert
5. Auto Pilot        — 자동 healing / autoscale / tuning
```

→ 좋은 operator = level 5.

---

## 7. webhook (validation / mutation)

```go
// validating webhook
func (r *MyApp) ValidateCreate() error {
    if r.Spec.Replicas < 1 {
        return errors.New("replicas must be >= 1")
    }
    if r.Spec.Image == "" {
        return errors.New("image required")
    }
    return nil
}

// mutating webhook (default 값 설정)
func (r *MyApp) Default() {
    if r.Spec.Replicas == 0 {
        r.Spec.Replicas = 1
    }
}
```

→ apply 시점에 검증 / 변환.

---

## 8. status conditions (★)

```yaml
status:
  conditions:
    - type: Available
      status: "True"
      reason: AllReplicasReady
      message: "3/3 replicas ready"
      lastTransitionTime: "2026-05-15T..."
    - type: Progressing
      status: "False"
      reason: NewReplicaSetAvailable
    - type: Degraded
      status: "False"
```

→ kubectl describe / monitoring 의 표준.

---

## 9. RBAC

```yaml
# operator 의 ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata: {name: my-operator}
rules:
  - apiGroups: ["app.example.com"]
    resources: [myapps, myapps/status]
    verbs: [get, list, watch, create, update, patch, delete]
  - apiGroups: ["apps"]
    resources: [deployments]
    verbs: [get, list, watch, create, update, patch, delete]
  - apiGroups: [""]
    resources: [services, configmaps, secrets]
    verbs: [get, list, watch, create, update, patch, delete]
```

→ operator 가 만들고 관리할 모든 resource type 의 권한 필요.

---

## 10. OLM (Operator Lifecycle Manager)

```
operator 자체 의 install / upgrade 관리:
  - OperatorHub.io 에서 install
  - subscription 으로 자동 upgrade
  - 의존성 자동 해결
  
RedHat OpenShift / OKD 의 표준.
일반 k8s 에도 install 가능.
```

---

## 11. test

```go
// envtest (etcd + apiserver 만 띄움, fast)
import "sigs.k8s.io/controller-runtime/pkg/envtest"

testEnv := &envtest.Environment{
    CRDDirectoryPaths: []string{"../config/crd/bases"},
}
cfg, _ := testEnv.Start()

// ginkgo test
It("should create deployment", func() {
    myapp := &appv1.MyApp{...}
    Expect(k8sClient.Create(ctx, myapp)).To(Succeed())
    
    deploy := &appsv1.Deployment{}
    Eventually(func() bool {
        err := k8sClient.Get(ctx, key, deploy)
        return err == nil && *deploy.Spec.Replicas == 3
    }).Should(BeTrue())
})
```

---

## 12. 도구 비교

| | 무엇 |
| --- | --- |
| **Kubebuilder** | Go, 표준 |
| **Operator SDK** | Go / Helm / Ansible base |
| **Kopf** | Python |
| **Knative Operator** | 단순 case |
| **Metacontroller** | shell / language-agnostic |
| **KubeBuilder + helm** | helm chart 의 wrapper |

→ Go 익숙 = Kubebuilder. Python = Kopf. helm chart 만 = Helm operator.

---

## 13. 함정

1. **Reconcile 가 non-idempotent** — 여러 번 호출 OK 보장.
2. **finalizer 안 만듦** — delete 시 cleanup 못 함.
3. **status 직접 mutate** — `r.Status().Update()`.
4. **watch 너무 많은 resource** — performance.
5. **leader election 없음** — 여러 instance 동시 reconcile.
6. **RBAC 너무 넓음** — least privilege.
7. **upgrade path** — CRD schema 변경 시 conversion.
8. **observability** — Prometheus metric 추가.

---

## 14. 관련

- [[kubernetes|↑ kubernetes]]
- [[concepts]]
- [[rbac]]
- [[../platform-engineering/crossplane|↗ Crossplane]]
