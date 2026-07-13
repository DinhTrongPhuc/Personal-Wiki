# Kubernetes

> Tài liệu này được chia thành 4 file liên kết:
> 
> - **Kubernetes** (file này) — khái niệm, kiến trúc, cài đặt, kubectl
> - [[Kubernetes Workloads]] — Pod, Deployment, Service, Ingress, ConfigMap, Secret
> - [[Kubernetes Storage & Config]] — Volume, PV, PVC, StatefulSet, Namespace
> - [[Kubernetes Production]] — Helm, CI/CD, GKE/EKS, HPA, monitoring

---

## Kubernetes là gì?

**Kubernetes** (viết tắt **K8s**) là hệ thống **orchestration** — tự động quản lý, scale, và vận hành các container trên nhiều máy chủ.

### Docker vs Kubernetes

```
Docker                          Kubernetes
──────                          ──────────
Chạy container trên 1 máy  →   Chạy container trên nhiều máy
Bạn quản lý thủ công       →   K8s tự động quản lý
docker run, docker compose  →   kubectl apply
1 host                      →   Cluster (nhiều node)
```

### Kubernetes giải quyết gì?

|Vấn đề|Kubernetes làm|
|---|---|
|App bị crash|Tự restart container|
|Traffic tăng đột biến|Tự scale thêm container|
|Cập nhật app không downtime|Rolling update tự động|
|1 server chết|Chuyển container sang server khác|
|Nhiều app cần cân bằng tải|Load balancing tích hợp|
|Quản lý config & secret|ConfigMap, Secret|

### Khi nào nên dùng K8s?

```
Nên dùng K8s khi:                   Chưa cần K8s khi:
✅ App có nhiều microservice         ❌ App nhỏ, 1-2 service
✅ Cần scale tự động                 ❌ Traffic ổn định, không cần scale
✅ Team lớn, nhiều môi trường        ❌ Team nhỏ, 1 server là đủ
✅ Cần high availability             ❌ Docker Compose đã đủ dùng
✅ Deploy nhiều lần/ngày
```

---

## Kiến trúc Kubernetes

```
┌─────────────────────────────────────────────────────────┐
│                        CLUSTER                          │
│                                                         │
│  ┌──────────────────────┐   ┌──────────────────────┐   │
│  │    Control Plane     │   │      Worker Nodes    │   │
│  │    (Master Node)     │   │                      │   │
│  │                      │   │  ┌────────────────┐  │   │
│  │  ┌────────────────┐  │   │  │   Node 1       │  │   │
│  │  │  API Server    │  │   │  │  ┌──────────┐  │  │   │
│  │  └────────────────┘  │   │  │  │  Pod     │  │  │   │
│  │  ┌────────────────┐  │   │  │  │ ┌──────┐ │  │  │   │
│  │  │   Scheduler    │  │   │  │  │ │ App  │ │  │  │   │
│  │  └────────────────┘  │   │  │  │ └──────┘ │  │  │   │
│  │  ┌────────────────┐  │   │  │  └──────────┘  │  │   │
│  │  │    etcd        │◄─┼───┼──┤  kubelet        │  │   │
│  │  └────────────────┘  │   │  └────────────────┘  │   │
│  │  ┌────────────────┐  │   │                      │   │
│  │  │ Controller Mgr │  │   │  ┌────────────────┐  │   │
│  │  └────────────────┘  │   │  │   Node 2       │  │   │
│  └──────────────────────┘   │  │  ┌──────────┐  │  │   │
│                              │  │  │  Pod     │  │  │   │
│        kubectl ──────────────┼──►  └──────────┘  │  │   │
│                              │  └────────────────┘  │   │
│                              └──────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Control Plane (não của cluster)

|Component|Vai trò|
|---|---|
|**API Server**|Cổng giao tiếp duy nhất — mọi lệnh đều qua đây|
|**etcd**|Database phân tán — lưu toàn bộ trạng thái cluster|
|**Scheduler**|Quyết định Pod chạy trên Node nào|
|**Controller Manager**|Đảm bảo trạng thái thực tế = trạng thái mong muốn|

### Worker Node (nơi app chạy)

|Component|Vai trò|
|---|---|
|**kubelet**|Agent trên mỗi node, nhận lệnh từ API Server|
|**kube-proxy**|Quản lý network rules, load balancing|
|**Container Runtime**|Chạy container thực tế (containerd, Docker)|

---

## Các đối tượng cơ bản (Objects)

> Xem chi tiết từng object tại [[Kubernetes Workloads]]

|Object|Là gì|Tương đương Docker|
|---|---|---|
|**Pod**|Đơn vị nhỏ nhất — 1+ container|`docker run`|
|**Deployment**|Quản lý Pod, rolling update, scale|`docker compose service`|
|**Service**|Expose Pod ra network, load balancing|Port mapping + load balancer|
|**Ingress**|HTTP routing từ ngoài vào cluster|Nginx reverse proxy|
|**ConfigMap**|Lưu config không nhạy cảm|`-e KEY=VALUE`|
|**Secret**|Lưu config nhạy cảm (password, token)|Docker secrets|
|**Namespace**|Phân chia tài nguyên logic|—|
|**PersistentVolume**|Lưu trữ data bền vững|Docker volume|
|**StatefulSet**|Quản lý app có state (DB)|—|

---

## Cài đặt môi trường local

### kubectl — CLI để tương tác với K8s

```bash
# macOS
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Windows (Chocolatey)
choco install kubernetes-cli

# Kiểm tra
kubectl version --client
```

### Minikube — K8s local (1 node)

> Phù hợp để học, chạy K8s đầy đủ trên máy cá nhân.

```bash
# macOS
brew install minikube

# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Windows
choco install minikube

# Khởi động cluster
minikube start
minikube start --cpus 4 --memory 8192    # nhiều resource hơn

# Kiểm tra
minikube status
kubectl get nodes

# Dashboard web UI
minikube dashboard

# Dừng / xóa cluster
minikube stop
minikube delete
```

### kind — K8s trong Docker (nhẹ hơn Minikube)

```bash
# macOS/Linux
brew install kind
# hoặc
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x kind && sudo mv kind /usr/local/bin/

# Tạo cluster
kind create cluster
kind create cluster --name my-cluster

# Tạo cluster nhiều node
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

# Xóa cluster
kind delete cluster
kind delete cluster --name my-cluster

# Liệt kê cluster
kind get clusters
```

### k9s — Terminal UI (tùy chọn, rất tiện)

```bash
# macOS
brew install k9s

# Linux
curl -sS https://webinstall.dev/k9s | bash

# Chạy
k9s
# Điều hướng bằng phím, gõ ':' để ra lệnh
# :pods, :deployments, :services, :namespaces
```

---

## kubectl — Lệnh cơ bản

### Cấu trúc lệnh

```bash
kubectl [verb] [resource] [name] [flags]
#        get     pods       my-pod  -n default
```

### Xem thông tin

```bash
# Xem nodes trong cluster
kubectl get nodes
kubectl get nodes -o wide          # thêm IP, OS, runtime

# Xem pods
kubectl get pods
kubectl get pods -A                # tất cả namespace
kubectl get pods -n kube-system    # namespace cụ thể
kubectl get pods -o wide           # thêm IP, node
kubectl get pods --watch           # real-time (như docker logs -f)

# Xem tất cả resource
kubectl get all
kubectl get all -n my-namespace

# Xem chi tiết một resource
kubectl describe pod my-pod
kubectl describe deployment my-app
kubectl describe service my-service
kubectl describe node my-node
```

### Tạo / Cập nhật resource

```bash
# Áp dụng file YAML (tạo hoặc cập nhật)
kubectl apply -f deployment.yaml
kubectl apply -f ./k8s/              # apply tất cả file trong thư mục
kubectl apply -f https://url/file.yaml

# Xóa resource
kubectl delete -f deployment.yaml
kubectl delete pod my-pod
kubectl delete deployment my-app
kubectl delete pod my-pod --grace-period=0   # force delete

# Scale deployment
kubectl scale deployment my-app --replicas=5

# Cập nhật image
kubectl set image deployment/my-app container-name=image:new-tag
```

### Debug

```bash
# Xem logs
kubectl logs my-pod
kubectl logs my-pod -f              # follow
kubectl logs my-pod --tail=100
kubectl logs my-pod -c container    # pod nhiều container
kubectl logs -l app=my-app          # theo label

# Vào trong pod (như docker exec)
kubectl exec -it my-pod -- sh
kubectl exec -it my-pod -- bash
kubectl exec -it my-pod -c container -- sh   # pod nhiều container

# Forward port từ pod ra localhost
kubectl port-forward pod/my-pod 3000:3000
kubectl port-forward service/my-service 3000:80

# Copy file vào/ra pod
kubectl cp my-pod:/app/logs ./logs
kubectl cp ./config.json my-pod:/app/config.json
```

### Context — chuyển đổi cluster

```bash
# Xem tất cả context (cluster đã cấu hình)
kubectl config get-contexts

# Xem context hiện tại
kubectl config current-context

# Chuyển sang cluster khác
kubectl config use-context minikube
kubectl config use-context my-gke-cluster

# Xem kubeconfig
kubectl config view
```

### Namespace

```bash
# Xem namespace
kubectl get namespaces

# Làm việc với namespace cụ thể
kubectl get pods -n production

# Đặt namespace mặc định (không cần gõ -n mỗi lần)
kubectl config set-context --current --namespace=production

# Tạo namespace
kubectl create namespace staging
```

---

## File YAML — Cấu trúc chung

> Mọi resource trong K8s đều được khai báo bằng YAML.

```yaml
apiVersion: apps/v1       # API version của resource
kind: Deployment          # loại resource
metadata:
  name: my-app            # tên resource
  namespace: default      # namespace (mặc định: default)
  labels:                 # nhãn — dùng để select resource
    app: my-app
    env: production
  annotations:            # metadata bổ sung (không dùng để select)
    description: "Main API service"
spec:                     # mô tả trạng thái mong muốn
  # ... nội dung tùy theo kind
```

### apiVersion theo loại resource

|Resource|apiVersion|
|---|---|
|Pod|`v1`|
|Deployment|`apps/v1`|
|Service|`v1`|
|Ingress|`networking.k8s.io/v1`|
|ConfigMap|`v1`|
|Secret|`v1`|
|PersistentVolume|`v1`|
|StatefulSet|`apps/v1`|
|HorizontalPodAutoscaler|`autoscaling/v2`|

---

## Luồng làm việc cơ bản

```bash
# 1. Viết Dockerfile cho app
#    (xem [[Dockerfile]])

# 2. Build và push image lên registry
docker build -t username/my-app:v1.0 .
docker push username/my-app:v1.0

# 3. Viết file YAML cho K8s
#    (xem [[Kubernetes Workloads]])

# 4. Apply lên cluster
kubectl apply -f k8s/

# 5. Kiểm tra
kubectl get pods
kubectl get services

# 6. Cập nhật app
docker build -t username/my-app:v1.1 .
docker push username/my-app:v1.1
kubectl set image deployment/my-app app=username/my-app:v1.1
# hoặc
kubectl apply -f k8s/deployment.yaml   # sau khi sửa image tag
```

---

## Cheat Sheet kubectl

```bash
# XEM
kubectl get pods / nodes / services / deployments / all
kubectl get pods -A                     # tất cả namespace
kubectl get pods -o wide                # thêm chi tiết
kubectl describe pod <name>             # chi tiết đầy đủ
kubectl top pods / nodes                # CPU & RAM usage

# THAO TÁC
kubectl apply -f file.yaml              # tạo/cập nhật
kubectl delete -f file.yaml             # xóa theo file
kubectl delete pod <name>               # xóa pod
kubectl scale deployment <name> --replicas=3
kubectl rollout restart deployment <name>    # restart tất cả pod
kubectl rollout status deployment <name>     # xem tiến trình rollout
kubectl rollout undo deployment <name>       # rollback

# DEBUG
kubectl logs <pod> -f
kubectl exec -it <pod> -- sh
kubectl port-forward <pod> 3000:3000
kubectl events --sort-by='.lastTimestamp'    # xem events gần nhất

# CONTEXT
kubectl config get-contexts
kubectl config use-context <name>
kubectl config set-context --current --namespace=<ns>
```

---

## Liên kết

- [[Kubernetes Workloads]] — Pod, Deployment, Service, Ingress, ConfigMap, Secret
- [[Kubernetes Storage & Config]] — Volume, PV, PVC, StatefulSet, Namespace
- [[Kubernetes Production]] — Helm, CI/CD, GKE/EKS, HPA, monitoring
- [[Docker]] — nền tảng trước khi học K8s
- [[Dockerfile]] — viết Dockerfile để build image cho K8s