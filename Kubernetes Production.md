# Kubernetes Production

> Thuộc chuỗi: [[Kubernetes]] → [[Kubernetes Workloads]] → [[Kubernetes Storage & Config]] → **Kubernetes Production**

---

## Helm — Package Manager cho K8s

> Helm giúp đóng gói, cấu hình, và deploy nhiều K8s resource cùng lúc như một **chart**.

```
Chart = tập hợp template YAML + values.yaml (config)
     → giống npm package nhưng cho K8s
```

### Cài đặt Helm

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Windows
choco install kubernetes-helm

# Kiểm tra
helm version
```

### Dùng chart có sẵn (ví dụ: deploy Nginx Ingress)

```bash
# Thêm repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add bitnami       https://charts.bitnami.com/bitnami
helm repo update

# Tìm chart
helm search repo nginx
helm search repo postgres

# Xem thông tin chart
helm show values bitnami/postgresql   # xem tất cả options có thể config

# Install chart
helm install my-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Install với custom values
helm install my-postgres bitnami/postgresql \
  --namespace my-app \
  --create-namespace \
  --set auth.postgresPassword=mysecret \
  --set primary.persistence.size=10Gi

# Xem release đã cài
helm list
helm list -A                          # tất cả namespace

# Cập nhật
helm upgrade my-postgres bitnami/postgresql \
  --namespace my-app \
  --set auth.postgresPassword=mysecret

# Rollback
helm rollback my-postgres 1           # về revision 1

# Xóa
helm uninstall my-postgres -n my-app
```

### Tạo Chart riêng cho app

```bash
# Tạo chart skeleton
helm create my-app-chart

# Cấu trúc được tạo:
# my-app-chart/
# ├── Chart.yaml          — metadata của chart
# ├── values.yaml         — giá trị mặc định
# └── templates/
#     ├── deployment.yaml
#     ├── service.yaml
#     ├── ingress.yaml
#     ├── configmap.yaml
#     └── _helpers.tpl    — helper templates
```

```yaml
# Chart.yaml
apiVersion: v2
name: my-app-chart
description: My Application Helm Chart
version: 1.0.0          # phiên bản chart
appVersion: "v1.0.0"    # phiên bản app
```

```yaml
# values.yaml — config mặc định
replicaCount: 2

image:
  repository: username/my-app
  tag: "v1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: true
  host: my-app.example.com

resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "500m"

env:
  NODE_ENV: production
  LOG_LEVEL: info

secrets:
  DATABASE_URL: ""        # override khi deploy
  JWT_SECRET: ""
```

```yaml
# templates/deployment.yaml — dùng biến từ values.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
  labels:
    app: {{ .Release.Name }}
    chart: {{ .Chart.Name }}-{{ .Chart.Version }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          env:
            {{- range $key, $val := .Values.env }}
            - name: {{ $key }}
              value: {{ $val | quote }}
            {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

```bash
# Test render template (không deploy)
helm template my-app ./my-app-chart
helm template my-app ./my-app-chart --values prod-values.yaml

# Kiểm tra lỗi
helm lint ./my-app-chart

# Deploy
helm install my-app ./my-app-chart \
  --namespace production \
  --create-namespace \
  --values prod-values.yaml \
  --set secrets.DATABASE_URL="postgresql://..." \
  --set secrets.JWT_SECRET="secret"

# Upgrade khi có version mới
helm upgrade my-app ./my-app-chart \
  --namespace production \
  --values prod-values.yaml \
  --set image.tag="v1.1.0"
```

---

## HPA — Horizontal Pod Autoscaler

> Tự động tăng/giảm số Pod dựa trên CPU, RAM, hoặc custom metrics.

### Yêu cầu: Metrics Server

```bash
# Cài Metrics Server (Minikube có sẵn addon)
minikube addons enable metrics-server

# Hoặc cài thủ công
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Kiểm tra
kubectl top nodes
kubectl top pods
```

### HPA cơ bản (CPU)

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: my-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app              # tên Deployment cần scale
  minReplicas: 2              # tối thiểu 2 Pod
  maxReplicas: 10             # tối đa 10 Pod
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # scale khi CPU trung bình > 70%
```

### HPA với nhiều metrics

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    # Scale theo CPU
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70

    # Scale theo RAM
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # chờ 60s trước khi scale up thêm
      policies:
        - type: Pods
          value: 4                      # tăng tối đa 4 Pod mỗi lần
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # chờ 5 phút trước khi scale down
      policies:
        - type: Pods
          value: 2                      # giảm tối đa 2 Pod mỗi lần
          periodSeconds: 60
```

```bash
# Áp dụng
kubectl apply -f hpa.yaml

# Xem trạng thái HPA
kubectl get hpa -n my-app
kubectl describe hpa my-app-hpa -n my-app

# Test scale — tạo load giả
kubectl run -it --rm load-test --image=busybox -- sh
# Trong container:
while true; do wget -q -O- http://my-app-service/api/health; done
```

---

## Kustomize — Override YAML không cần template

> Thay đổi K8s YAML theo môi trường (dev/staging/prod) mà không dùng Helm.

```
k8s/
├── base/                     ← config gốc dùng chung
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/                  ← override cho dev
    │   ├── kustomization.yaml
    │   └── patch-dev.yaml
    └── production/           ← override cho production
        ├── kustomization.yaml
        └── patch-prod.yaml
```

```yaml
# k8s/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

```yaml
# k8s/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
namespace: production
images:
  - name: username/my-app
    newTag: "v1.2.0"          # đổi image tag
patches:
  - path: patch-prod.yaml
```

```yaml
# k8s/overlays/production/patch-prod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 5                 # override replicas cho production
  template:
    spec:
      containers:
        - name: app
          resources:
            limits:
              memory: "512Mi" # tăng resource limit
              cpu: "1"
```

```bash
# Preview kết quả merge
kubectl kustomize k8s/overlays/production

# Deploy
kubectl apply -k k8s/overlays/production
kubectl apply -k k8s/overlays/dev
```

---

## CI/CD với GitHub Actions

### Build → Push → Deploy lên K8s

```yaml
# .github/workflows/deploy.yml
name: Deploy to Kubernetes

on:
  push:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/my-app

jobs:
  # ===== Test =====
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
      - run: npm ci && npm test

  # ===== Build & Push Image =====
  build:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}

    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3

      - name: Login Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=         # tag = git commit sha
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ===== Deploy lên K8s =====
  deploy:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      # Setup kubectl với kubeconfig từ secret
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubectl
        run: |
          echo "${{ secrets.KUBECONFIG }}" | base64 -d > kubeconfig.yaml
          export KUBECONFIG=kubeconfig.yaml

      # Cách 1: Dùng kubectl set image
      - name: Deploy
        run: |
          export KUBECONFIG=kubeconfig.yaml
          kubectl set image deployment/my-app \
            app=${{ env.IMAGE_NAME }}:${{ needs.build.outputs.image-tag }} \
            -n production
          kubectl rollout status deployment/my-app -n production --timeout=5m

      # Cách 2: Dùng Kustomize
      # - name: Update image tag
      #   run: |
      #     cd k8s/overlays/production
      #     kustomize edit set image username/my-app:${{ needs.build.outputs.image-tag }}
      # - name: Deploy
      #   run: kubectl apply -k k8s/overlays/production
```

### Cấu hình GitHub Secrets

|Secret|Giá trị|
|---|---|
|`DOCKERHUB_USERNAME`|Username Docker Hub|
|`DOCKERHUB_TOKEN`|Access Token Docker Hub|
|`KUBECONFIG`|`base64 ~/.kube/config` — kubeconfig của cluster|

```bash
# Lấy kubeconfig đã encode
cat ~/.kube/config | base64 | pbcopy   # macOS
cat ~/.kube/config | base64 | xclip    # Linux
```

---

## Deploy lên Cloud

### GKE (Google Kubernetes Engine)

```bash
# Cài gcloud CLI
# https://cloud.google.com/sdk/docs/install

# Đăng nhập
gcloud auth login
gcloud config set project YOUR_PROJECT_ID

# Tạo cluster
gcloud container clusters create my-cluster \
  --region asia-southeast1 \           # Singapore — gần VN
  --num-nodes 2 \
  --machine-type e2-small \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 5

# Kết nối kubectl vào cluster
gcloud container clusters get-credentials my-cluster \
  --region asia-southeast1

# Kiểm tra
kubectl get nodes

# Xóa cluster (dừng tính phí)
gcloud container clusters delete my-cluster --region asia-southeast1
```

```bash
# Free tier GKE: 1 Autopilot cluster miễn phí (management fee)
# Chi phí chủ yếu từ node (VM)
# e2-small: ~$12/tháng/node
```

### EKS (Amazon Elastic Kubernetes Service)

```bash
# Cài eksctl
brew install eksctl         # macOS
# Linux: curl và install từ github.com/eksctl-io/eksctl/releases

# Cài aws CLI và configure
aws configure

# Tạo cluster
eksctl create cluster \
  --name my-cluster \
  --region ap-southeast-1 \     # Singapore
  --nodegroup-name standard \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 5 \
  --managed

# kubectl tự động được cấu hình
kubectl get nodes

# Xóa cluster
eksctl delete cluster --name my-cluster --region ap-southeast-1
```

### AKS (Azure Kubernetes Service)

```bash
# Cài azure CLI
# https://docs.microsoft.com/cli/azure/install-azure-cli
az login

# Tạo resource group
az group create --name my-rg --location southeastasia

# Tạo cluster (1 node free tier)
az aks create \
  --resource-group my-rg \
  --name my-cluster \
  --node-count 2 \
  --node-vm-size Standard_B2s \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 5 \
  --generate-ssh-keys

# Kết nối kubectl
az aks get-credentials --resource-group my-rg --name my-cluster

# Xóa cluster
az aks delete --resource-group my-rg --name my-cluster
```

---

## Monitoring

### Prometheus + Grafana qua Helm

```bash
# Thêm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Cài kube-prometheus-stack (Prometheus + Grafana + AlertManager)
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123

# Xem các pod được tạo
kubectl get pods -n monitoring

# Mở Grafana UI
kubectl port-forward service/monitoring-grafana 3000:80 -n monitoring
# Truy cập: http://localhost:3000
# User: admin / Pass: admin123
```

### Xem metrics cơ bản

```bash
# CPU & RAM của nodes
kubectl top nodes

# CPU & RAM của pods
kubectl top pods -n my-app
kubectl top pods -A --sort-by=memory   # sort theo RAM

# Events gần đây (debug crash, OOMKilled...)
kubectl events -n my-app --sort-by='.lastTimestamp'
kubectl describe pod <pod-name> -n my-app | grep -A 10 Events
```

---

## Debugging Production

### Pod không start được

```bash
# Xem trạng thái
kubectl get pods -n my-app

# STATUS thường gặp:
# Pending        → chưa được schedule lên node (không đủ resource?)
# CrashLoopBackOff → app crash liên tục
# ImagePullBackOff → không pull được image
# OOMKilled       → hết RAM
# Evicted         → bị node đuổi do thiếu resource

# Xem nguyên nhân
kubectl describe pod <pod-name> -n my-app
# Chú ý phần: Events, Conditions, Last State

# Xem logs của container đã crash
kubectl logs <pod-name> -n my-app
kubectl logs <pod-name> -n my-app --previous    # logs của lần chạy trước khi crash
```

### Debug với temporary pod

```bash
# Tạo pod debug trong namespace cần kiểm tra
kubectl run debug \
  --image=busybox \
  --rm -it \
  --restart=Never \
  -n my-app \
  -- sh

# Test kết nối đến service
wget -qO- http://api-service/health

# Test DNS resolution
nslookup api-service
nslookup api-service.my-app.svc.cluster.local

# Test kết nối đến database
nc -zv postgres-service 5432
```

### Network Policy — debug connectivity

```bash
# Kiểm tra service endpoints
kubectl get endpoints -n my-app

# Nếu Endpoints = <none> → selector không match với pod labels
kubectl describe service api-service -n my-app
kubectl get pods -n my-app --show-labels
```

---

## Best Practices

### Resource requests & limits — luôn khai báo

```yaml
resources:
  requests:          # đảm bảo Pod luôn có đủ resource này
    memory: "128Mi"
    cpu: "100m"
  limits:            # không để Pod ăn quá resource này
    memory: "256Mi"
    cpu: "500m"
# Không khai báo → Pod có thể bị Evicted bất cứ lúc nào
```

### Health checks — luôn có cả hai

```yaml
livenessProbe:       # restart nếu app bị hang
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:      # chỉ gửi traffic khi app sẵn sàng
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Image tag — không dùng `:latest` trên production

```yaml
# ❌ Không rõ version, không rollback được
image: username/my-app:latest

# ✅ Rõ version, rollback được
image: username/my-app:v1.2.0
image: username/my-app:abc1234   # git commit sha
```

### Pod Disruption Budget — đảm bảo luôn có Pod chạy

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2              # luôn có ít nhất 2 Pod running
  # maxUnavailable: 1          # hoặc: tối đa 1 Pod unavailable
  selector:
    matchLabels:
      app: my-app
```

### Tổ chức file YAML

```
k8s/
├── base/
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    └── production/
```

---

## Cheat Sheet

```bash
# HELM
helm repo add <name> <url>
helm repo update
helm install <release> <chart> -n <ns> --create-namespace
helm upgrade <release> <chart> -n <ns> --set key=value
helm rollback <release> <revision>
helm uninstall <release> -n <ns>
helm list -A

# HPA
kubectl get hpa [-n namespace]
kubectl describe hpa <name>
kubectl top pods [-n namespace]
kubectl top nodes

# KUSTOMIZE
kubectl apply -k <overlay-dir>
kubectl kustomize <overlay-dir>      # preview

# DEBUG
kubectl describe pod <name>
kubectl logs <pod> --previous
kubectl exec -it <pod> -- sh
kubectl run debug --image=busybox --rm -it --restart=Never -- sh
kubectl events --sort-by='.lastTimestamp' -n <ns>
kubectl get endpoints -n <ns>

# CLOUD
# GKE
gcloud container clusters create ...
gcloud container clusters get-credentials ...
# EKS
eksctl create cluster ...
# AKS
az aks create ...
az aks get-credentials ...
```

---

## Liên kết

- [[Kubernetes]] — tổng quan, kiến trúc, kubectl
- [[Kubernetes Workloads]] — Pod, Deployment, Service, Ingress
- [[Kubernetes Storage & Config]] — Volume, PV, PVC, StatefulSet
- [[Docker]] → [[Dockerfile]] → [[Docker Compose]] → [[Docker Deploy]]