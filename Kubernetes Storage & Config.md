# Kubernetes Storage & Config

> Thuộc chuỗi: [[Kubernetes]] → [[Kubernetes Workloads]] → **Kubernetes Storage & Config** → [[Kubernetes Production]]

---

## Namespace

> Phân chia tài nguyên logic trong cluster — giống như folder, tách biệt môi trường hoặc team.

### Các Namespace mặc định

|Namespace|Dùng cho|
|---|---|
|`default`|Resource không chỉ định namespace|
|`kube-system`|Component của K8s (API server, scheduler...)|
|`kube-public`|Dữ liệu public, đọc được bởi mọi user|
|`kube-node-lease`|Heartbeat của các node|

### Tạo và dùng Namespace

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    env: staging
```

```bash
kubectl apply -f namespace.yaml

# Hoặc tạo nhanh
kubectl create namespace production
kubectl create namespace staging

# Xem namespaces
kubectl get namespaces

# Làm việc với namespace
kubectl get pods -n production
kubectl get all -n production

# Đặt namespace mặc định cho session hiện tại
kubectl config set-context --current --namespace=production

# Xem resource ở tất cả namespace
kubectl get pods -A
kubectl get pods --all-namespaces
```

### Pattern tổ chức namespace

```
# Theo môi trường
production, staging, development

# Theo team
team-backend, team-frontend, team-data

# Theo app
app-payments, app-notifications, app-users
```

### Resource Quota — giới hạn tài nguyên theo namespace

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "4"              # tổng CPU request không quá 4 cores
    requests.memory: 8Gi           # tổng RAM request không quá 8GB
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "50"                     # không quá 50 pods
    services: "20"
    persistentvolumeclaims: "10"
```

### LimitRange — giới hạn mặc định cho container

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:                     # giá trị limit mặc định nếu không khai báo
        memory: "256Mi"
        cpu: "500m"
      defaultRequest:              # giá trị request mặc định
        memory: "128Mi"
        cpu: "100m"
      max:                         # giới hạn tối đa có thể khai báo
        memory: "2Gi"
        cpu: "2"
      min:                         # giới hạn tối thiểu
        memory: "64Mi"
        cpu: "50m"
```

---

## Volume

> Lưu trữ data cho Pod. Khác với Docker volume — K8s volume gắn với **Pod lifecycle** (không phải container).

### emptyDir — volume tạm dùng chung giữa containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "while true; do echo $(date) >> /data/log.txt; sleep 5; done"]
      volumeMounts:
        - name: shared-data
          mountPath: /data

    - name: reader
      image: busybox
      command: ["sh", "-c", "tail -f /data/log.txt"]
      volumeMounts:
        - name: shared-data
          mountPath: /data

  volumes:
    - name: shared-data
      emptyDir: {}               # tạo khi Pod start, xóa khi Pod xóa
      # emptyDir:
      #   medium: Memory         # lưu trong RAM — nhanh hơn nhưng mất khi restart
      #   sizeLimit: "1Gi"
```

### hostPath — mount thư mục từ Node

```yaml
volumes:
  - name: host-logs
    hostPath:
      path: /var/log/app         # thư mục trên Node host
      type: DirectoryOrCreate    # tạo nếu chưa có
      # type: Directory          # phải tồn tại sẵn
      # type: File
      # type: Socket
```

> ⚠️ `hostPath` phụ thuộc vào node cụ thể — dùng chủ yếu cho debugging, không dùng production.

---

## PersistentVolume (PV) & PersistentVolumeClaim (PVC)

> Tách biệt việc **cung cấp storage** (admin) và **sử dụng storage** (developer).

```
Admin tạo PV (hoặc StorageClass tự tạo) → Developer tạo PVC → Pod dùng PVC
(nguồn storage thực tế)                   (yêu cầu storage)
```

### PersistentVolume — admin khai báo nguồn storage

```yaml
# pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce                # chỉ 1 node được mount read-write
    # - ReadOnlyMany               # nhiều node đọc được
    # - ReadWriteMany              # nhiều node đọc-ghi được (cần NFS/CephFS)
  persistentVolumeReclaimPolicy: Retain   # giữ data khi PVC bị xóa
  # Retain  = giữ data, admin phải tự dọn
  # Delete  = xóa data khi PVC bị xóa (cloud default)
  # Recycle = xóa nội dung, tái sử dụng PV (deprecated)
  storageClassName: manual
  hostPath:                        # local — chỉ dùng khi học
    path: /data/postgres
```

### PersistentVolumeClaim — developer yêu cầu storage

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: my-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi               # yêu cầu 5GB (PV phải có ít nhất 5GB)
  storageClassName: manual       # phải khớp với PV hoặc StorageClass
```

```bash
kubectl get pv
kubectl get pvc -n my-app
kubectl describe pvc postgres-pvc -n my-app

# STATUS:
# Pending  → chưa bind được PV nào
# Bound    → đã bind thành công
# Lost     → PV bị xóa nhưng PVC còn
```

### Dùng PVC trong Pod/Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          env:
            - name: POSTGRES_PASSWORD
              value: "password"
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc    # tên PVC đã tạo
```

### StorageClass — tự động tạo PV (Dynamic Provisioning)

> Thay vì admin phải tạo PV thủ công, StorageClass tự động tạo PV khi có PVC.

```yaml
# storageclass.yaml — chỉ cần trên cloud, Minikube có sẵn 'standard'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/gce-pd      # GKE
# provisioner: ebs.csi.aws.com         # EKS
# provisioner: disk.csi.azure.com      # AKS
parameters:
  type: pd-ssd
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

```yaml
# PVC dùng StorageClass — PV tự động được tạo
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: auto-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd          # dùng StorageClass đã khai báo
  # storageClassName: standard        # Minikube default
```

```bash
# Kiểm tra StorageClass có sẵn
kubectl get storageclass
```

---

## StatefulSet

> Như Deployment nhưng dành cho **app có state** (databases, message queues...). Đảm bảo:
> 
> - Tên Pod ổn định: `my-db-0`, `my-db-1`, `my-db-2`
> - Volume riêng cho mỗi Pod
> - Thứ tự start/stop có kiểm soát

### So sánh Deployment vs StatefulSet

||Deployment|StatefulSet|
|---|---|---|
|Tên Pod|Random: `app-abc12`|Có thứ tự: `app-0`, `app-1`|
|Storage|Dùng chung hoặc không|Mỗi Pod có PVC riêng|
|Start/Stop|Song song|Theo thứ tự (0→1→2)|
|Dùng cho|Stateless app (API, web)|Stateful app (DB, Redis, Kafka)|

### StatefulSet cơ bản

```yaml
# statefulset-postgres.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: my-app
spec:
  serviceName: postgres-headless    # phải có Headless Service
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              value: mydb
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"

  # volumeClaimTemplates — tạo PVC riêng cho mỗi Pod tự động
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard    # Minikube; đổi theo cloud
        resources:
          requests:
            storage: 5Gi
---
# Headless Service — cần thiết cho StatefulSet
# Không có ClusterIP, dùng để Pod tìm nhau qua DNS
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: my-app
spec:
  clusterIP: None                   # Headless — không có IP
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
---
# Service thông thường — để app khác kết nối
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: my-app
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

```bash
# Pod được đặt tên có thứ tự
kubectl get pods -n my-app
# NAME          READY   STATUS
# postgres-0    1/1     Running   ← luôn là postgres-0

# Kết nối từ app khác trong namespace my-app:
# postgresql://postgres-service:5432/mydb

# Hoặc dùng Headless Service DNS:
# postgres-0.postgres-headless.my-app.svc.cluster.local:5432
```

### StatefulSet — Redis

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis-headless
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          command: ["redis-server", "--requirepass", "$(REDIS_PASSWORD)", "--appendonly", "yes"]
          ports:
            - containerPort: 6379
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-secret
                  key: REDIS_PASSWORD
          volumeMounts:
            - name: redis-data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
    - port: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  selector:
    app: redis
  ports:
    - port: 6379
```

---

## RBAC — Phân quyền

> Role-Based Access Control — ai được làm gì với resource nào.

```
User / ServiceAccount  →  RoleBinding  →  Role  →  quyền trên resource
```

### ServiceAccount — định danh cho Pod

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: my-app
```

### Role — quyền trong 1 namespace

```yaml
# role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: my-app
rules:
  - apiGroups: [""]               # "" = core API group
    resources: ["pods", "pods/logs"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]
```

### RoleBinding — gán Role cho user/serviceaccount

```yaml
# rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: my-app
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: my-app
  # hoặc user:
  # - kind: User
  #   name: "jane@example.com"
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole & ClusterRoleBinding — quyền trên toàn cluster

```yaml
# Dùng khi cần quyền trên nhiều/tất cả namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes
subjects:
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Kiểm tra quyền
kubectl auth can-i get pods -n my-app --as system:serviceaccount:my-app:my-app-sa
kubectl auth can-i create deployments -n my-app

# Xem roles
kubectl get roles -n my-app
kubectl get rolebindings -n my-app
kubectl describe rolebinding read-pods -n my-app
```

---

## Ví dụ thực tế — App đầy đủ với Database

```
k8s/
├── namespaces/
│   └── namespace.yaml
├── config/
│   ├── configmap.yaml
│   └── secret.yaml
├── storage/
│   └── pvc.yaml              (hoặc dùng volumeClaimTemplates trong StatefulSet)
├── database/
│   ├── statefulset-postgres.yaml
│   ├── service-postgres.yaml
│   └── secret-postgres.yaml
├── api/
│   ├── deployment-api.yaml
│   ├── service-api.yaml
│   └── ingress.yaml
└── kustomization.yaml        (optional — xem [[Kubernetes Production]])
```

```bash
# Deploy toàn bộ theo thứ tự phụ thuộc
kubectl apply -f k8s/namespaces/
kubectl apply -f k8s/config/
kubectl apply -f k8s/database/
kubectl apply -f k8s/api/

# Hoặc apply tất cả một lúc (K8s tự xử lý thứ tự)
kubectl apply -f k8s/ --recursive

# Kiểm tra
kubectl get all -n my-app
kubectl get pvc -n my-app
kubectl get pv
```

---

## Cheat Sheet

```bash
# NAMESPACE
kubectl get namespaces
kubectl create namespace <name>
kubectl delete namespace <name>
kubectl config set-context --current --namespace=<name>

# PV & PVC
kubectl get pv
kubectl get pvc [-n namespace]
kubectl describe pvc <name>
kubectl delete pvc <name>

# STATEFULSET
kubectl get statefulsets [-n namespace]
kubectl scale statefulset <name> --replicas=3
kubectl rollout restart statefulset <name>

# STORAGE CLASS
kubectl get storageclass

# RBAC
kubectl get serviceaccounts [-n namespace]
kubectl get roles [-n namespace]
kubectl get rolebindings [-n namespace]
kubectl get clusterroles
kubectl auth can-i <verb> <resource> -n <namespace>
```

---

## Liên kết

- [[Kubernetes]] — tổng quan, kiến trúc, kubectl
- [[Kubernetes Workloads]] — Pod, Deployment, Service, Ingress
- [[Kubernetes Production]] — Helm, CI/CD, HPA, cloud deploy