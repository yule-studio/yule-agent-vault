---
title: "실습 01 — k3s single node + first app"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:29:00+09:00
tags: [devops, k3s, practice]
---

# 실습 01 — k3s single node + first app

**[[practice|↑ practice]]**

---

## 목표

- Ubuntu VM 에 k3s 설치 (config 파일 방식)
- nginx Pod + Service + Ingress
- Traefik 으로 외부 접속
- local-path PVC 사용

---

## 1. VM 준비

```bash
# Ubuntu 22.04 server (4GB RAM, 2 vCPU, 20GB)
# 또는 EC2 t3.medium

# swap off
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# 시간 동기
sudo apt install -y chrony
sudo systemctl enable --now chrony
```

---

## 2. k3s 설치

```bash
# config 미리
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml <<EOF
write-kubeconfig-mode: "0644"
tls-san:
  - $(hostname -I | awk '{print $1}')
node-label:
  - "env=dev"
EOF

# install (version 고정)
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.29.4+k3s1 sh -

# 확인
sudo systemctl status k3s
sudo k3s kubectl get nodes
sudo k3s kubectl get pods -A
```

---

## 3. kubectl alias

```bash
echo 'alias kubectl="sudo k3s kubectl"' >> ~/.bashrc
source ~/.bashrc
kubectl version
```

또는 kubeconfig copy:
```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config

# kubectl 일반 설치 (apt)
sudo apt install -y kubectl
kubectl get nodes
```

---

## 4. 첫 app 배포

```yaml
# nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: nginx}
spec:
  replicas: 2
  selector: {matchLabels: {app: nginx}}
  template:
    metadata: {labels: {app: nginx}}
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports: [{containerPort: 80}]
          resources:
            requests: {cpu: 50m, memory: 64Mi}
            limits: {cpu: 200m, memory: 128Mi}
          readinessProbe:
            httpGet: {path: /, port: 80}
            initialDelaySeconds: 2
---
apiVersion: v1
kind: Service
metadata: {name: nginx}
spec:
  selector: {app: nginx}
  ports: [{port: 80, targetPort: 80}]
```

```bash
kubectl apply -f nginx.yaml
kubectl get pods
kubectl get svc
```

---

## 5. Ingress (Traefik)

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
    - host: nginx.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port: {number: 80}
```

```bash
kubectl apply -f ingress.yaml

# /etc/hosts 등록 (로컬 PC)
echo "<VM_IP>  nginx.local" | sudo tee -a /etc/hosts

# 검증
curl http://nginx.local
```

→ Traefik 이 자동으로 :80 listen, host header 로 routing.

---

## 6. PVC + Pod

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: app-data}
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-path
  resources: {requests: {storage: 1Gi}}

---
apiVersion: apps/v1
kind: Deployment
metadata: {name: web-stateful}
spec:
  replicas: 1
  selector: {matchLabels: {app: web-stateful}}
  template:
    metadata: {labels: {app: web-stateful}}
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          volumeMounts:
            - {mountPath: /usr/share/nginx/html, name: data}
      volumes:
        - name: data
          persistentVolumeClaim: {claimName: app-data}
```

```bash
kubectl apply -f pvc.yaml

# 파일 작성
kubectl exec -it deploy/web-stateful -- sh -c "echo 'Hello k3s' > /usr/share/nginx/html/index.html"

# pod 삭제 후 재시작 — 데이터 살아 있음
kubectl delete pod -l app=web-stateful
sleep 5
kubectl exec deploy/web-stateful -- cat /usr/share/nginx/html/index.html
```

→ local-path 가 `/var/lib/rancher/k3s/storage/<pvc-name>` 에 저장.

---

## 7. metrics-server

```bash
# 자동 설치 (k3s 내장)
kubectl top nodes
kubectl top pods
```

→ HPA 의 base.

---

## 8. dashboard (옵션)

```bash
GITHUB_URL=https://github.com/kubernetes/dashboard/releases
VERSION_KUBE_DASHBOARD=$(curl -w '%{url_effective}' -I -L -s -S ${GITHUB_URL}/latest -o /dev/null | sed -e 's|.*/||')
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/${VERSION_KUBE_DASHBOARD}/aio/deploy/recommended.yaml
```

```bash
# token 생성
kubectl create serviceaccount admin-user -n kube-system
kubectl create clusterrolebinding admin-user --clusterrole=cluster-admin --serviceaccount=kube-system:admin-user
kubectl create token admin-user -n kube-system

# port forward
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard 8443:443
# → https://localhost:8443
```

---

## 9. 검증 체크리스트

- [ ] `kubectl get nodes` → 1 node Ready
- [ ] nginx 2 pod Running
- [ ] curl http://nginx.local → 응답
- [ ] PVC Bound + 파일 persistent
- [ ] `kubectl top nodes` 동작
- [ ] dashboard token login 성공 (옵션)

---

## 10. 정리

```bash
# 모두 삭제
kubectl delete -f .

# k3s uninstall
sudo /usr/local/bin/k3s-uninstall.sh
```

---

## 11. 관련

- [[practice|↑ practice]]
- [[../installation]]
- [[../networking]]
- [[02-ha-3-server-cluster]]
