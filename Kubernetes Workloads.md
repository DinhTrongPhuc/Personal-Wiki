# Kubernetes Workloads

> Thuộc chuỗi: [[Kubernetes]] → **Kubernetes Workloads** → [[Kubernetes Storage & Config]] → [[Kubernetes Production]]

---

## Pod

> Đơn vị nhỏ nhất trong K8s. Một Pod chứa **1 hoặc nhiều container** dùng chung network và storage.

### Pod cơ bản

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
spec:
  containers:
    - name: app                        # tên container
      image: node:20-alpine            # image
      ports:
        - containerPort: 3000          # port container lắng nghe
      env:
        - name: NODE_ENV
          value: "production"
      resources:
        requests:                      # tài nguyên tối thiểu cần
          memory: "64Mi"
          cpu: "100m"                  # 100 millicores = 0.1 CPU
        limits:                        # tài nguyên tối đa được dùng
          memory: "128Mi"
          cpu: "500m"
```

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod my-pod
kubectl delete pod my-pod
```

> **Lưu ý:** Pod không nên tạo trực tiếp — dùng **Deployment** để K8s tự quản lý restart, scale. Pod được tạo thẳng sẽ không được tự restart khi crash.

### Pod với nhiều container (Sidecar pattern)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
    # Container chính
    - name: app
      image: my-app:latest
      ports:
        - containerPort: 3000

    # Sidecar — log collector
    - name: log-collector
      image: fluent/fluent-bit:latest
      volumeMounts:
        - name: logs
          mountPath: /var/log/app

  volumes:
    - name: logs
      emptyDir: {}               # volume tạm, chia sẻ giữa các container trong pod
```

---

## Deployment

> Quản lý tập hợp Pod giống nhau. Xử lý **rolling update**, **rollback**, **scaling**.

### Deployment cơ bản

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 3                          # số Pod muốn chạy
  selector:
    matchLabels:
      app: my-app                      # chọn Pod theo label này
  template:                            # template để tạo Pod
    metadata:
      labels:
        app: my-app                    # label phải khớp với selector
    spec:
      containers:
        - name: app
          image: username/my-app:v1.0
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: "production"
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          # Health checks
          livenessProbe:               # K8s restart container nếu probe fail
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30    # chờ 30s sau khi start rồi mới check
            periodSeconds: 10          # check mỗi 10s
            failureThreshold: 3        # fail 3 lần liên tiếp mới restart
          readinessProbe:              # K8s chỉ gửi traffic khi probe pass
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
```

### Rolling Update Strategy

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate                # mặc định
    rollingUpdate:
      maxSurge: 1                      # tạo thêm tối đa 1 Pod mới khi update
      maxUnavailable: 0                # không để Pod nào unavailable → zero downtime
  # hoặc
  strategy:
    type: Recreate                     # xóa hết Pod cũ rồi mới tạo mới (có downtime)
```

### Lệnh quản lý Deployment

```bash
# Tạo/cập nhật
kubectl apply -f deployment.yaml

# Xem trạng thái
kubectl get deployments
kubectl describe deployment my-app
kubectl rollout status deployment/my-app     # xem tiến trình rolling update

# Scale
kubectl scale deployment my-app --replicas=5

# Cập nhật image (không cần sửa file YAML)
kubectl set image deployment/my-app app=username/my-app:v1.1

# Rollback
kubectl rollout undo deployment/my-app              # về version trước
kubectl rollout undo deployment/my-app --to-revision=2  # về revision cụ thể
kubectl rollout history deployment/my-app           # xem lịch sử revision

# Restart tất cả Pod (pull image mới với tag :latest)
kubectl rollout restart deployment/my-app

# Xóa
kubectl delete deployment my-app
```

---

## Service

> Expose Pod ra network. Vì Pod có IP thay đổi mỗi khi restart, Service cung cấp **IP và DNS ổn định**.

### Các loại Service

```
ClusterIP     → chỉ trong cluster (mặc định)
NodePort      → expose qua port của Node (học local)
LoadBalancer  → tạo cloud load balancer (production)
ExternalName  → map đến external DNS
```

### ClusterIP — giao tiếp nội bộ

```yaml
# service-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: ClusterIP                      # mặc định, không cần ghi cũng được
  selector:
    app: my-app                        # chọn Pod có label app=my-app
  ports:
    - port: 80                         # port của Service (bên trong cluster)
      targetPort: 3000                 # port của Pod/container
      protocol: TCP
```

```bash
# Trong cluster, các Pod khác có thể gọi:
# http://my-app-service          → port 80
# http://my-app-service.default  → có namespace
# http://my-app-service.default.svc.cluster.local  → FQDN đầy đủ
```

### NodePort — expose ra ngoài (dùng khi học local)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-nodeport
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80                         # port trong cluster
      targetPort: 3000                 # port của Pod
      nodePort: 30080                  # port trên Node (30000-32767)
```

```bash
# Truy cập qua: http://<node-ip>:30080
# Với Minikube:
minikube service my-app-nodeport       # tự mở browser
minikube service my-app-nodeport --url # chỉ in URL
```

### LoadBalancer — production (cloud)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-lb
spec:
  type: LoadBalancer                   # cloud provider tạo LB tự động
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 3000
```

```bash
kubectl get service my-app-lb
# EXTERNAL-IP sẽ có sau ~1-2 phút (cloud provider cấp)
```

### Service cho nhiều port

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-multi
spec:
  selector:
    app: my-app
  ports:
    - name: http
      port: 80
      targetPort: 3000
    - name: metrics
      port: 9090
      targetPort: 9090
```

---

## Ingress

> **HTTP/HTTPS router** — điều hướng request từ ngoài vào đúng Service dựa trên host/path. Thay thế cho LoadBalancer khi có nhiều service.

```
Internet → Ingress Controller → Ingress Rules → Service → Pod
                (Nginx/Traefik)
```

### Cài Ingress Controller (Minikube)

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx      # kiểm tra controller đang chạy
```

### Ingress cơ bản — route theo path

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: my-app.local               # domain (thêm vào /etc/hosts khi test local)
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

### Ingress với nhiều domain

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80

    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

### Ingress với TLS/HTTPS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"  # tự động lấy cert
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - example.com
        - www.example.com
      secretName: example-tls           # Secret chứa certificate
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

### Test Ingress local với Minikube

```bash
# Lấy IP của Minikube
minikube ip                            # VD: 192.168.49.2

# Thêm vào /etc/hosts
echo "192.168.49.2 my-app.local" | sudo tee -a /etc/hosts

# Truy cập
curl http://my-app.local
```

---

## ConfigMap

> Lưu **config không nhạy cảm** (biến môi trường, config file). Tách config ra khỏi image.

### Tạo ConfigMap

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Dạng key-value
  NODE_ENV: "production"
  API_URL: "https://api.example.com"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"

  # Dạng file
  app.properties: |
    server.port=3000
    server.timeout=30
    cache.enabled=true

  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://localhost:3000;
      }
    }
```

```bash
# Tạo từ file trực tiếp (không cần viết YAML)
kubectl create configmap app-config --from-file=./config/
kubectl create configmap app-config --from-env-file=.env
kubectl create configmap app-config --from-literal=NODE_ENV=production --from-literal=PORT=3000

kubectl get configmaps
kubectl describe configmap app-config
```

### Dùng ConfigMap trong Pod

```yaml
# deployment-with-config.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: username/my-app:latest

          # Cách 1: Inject từng key thành env var
          env:
            - name: NODE_ENV
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: NODE_ENV
            - name: API_URL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: API_URL

          # Cách 2: Inject tất cả keys thành env vars
          envFrom:
            - configMapRef:
                name: app-config

          # Cách 3: Mount file config vào volume
          volumeMounts:
            - name: config-volume
              mountPath: /app/config     # thư mục trong container
              readOnly: true

      volumes:
        - name: config-volume
          configMap:
            name: app-config            # mount tất cả key thành file
            # hoặc chỉ mount một số key:
            items:
              - key: app.properties
                path: app.properties    # tên file trong container
              - key: nginx.conf
                path: nginx.conf
```

---

## Secret

> Lưu **thông tin nhạy cảm** (password, API key, certificate). Được encode base64, có thể encrypt at rest.

> **Lưu ý bảo mật:** Secret mặc định chỉ encode base64, không encrypt. Để secure hơn cần dùng **Sealed Secrets**, **External Secrets Operator**, hoặc bật encryption at rest trong K8s config.

### Tạo Secret

```bash
# Tạo từ literal (K8s tự encode base64)
kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD=supersecret123 \
  --from-literal=JWT_SECRET=my-jwt-secret

# Tạo từ file
kubectl create secret generic tls-secret \
  --from-file=tls.crt=./cert.pem \
  --from-file=tls.key=./key.pem

# TLS Secret chuyên dụng
kubectl create secret tls my-tls \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem

kubectl get secrets
kubectl describe secret app-secret     # không hiện giá trị
```

### Viết Secret bằng YAML

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  # Giá trị phải encode base64
  # echo -n "supersecret123" | base64
  DB_PASSWORD: c3VwZXJzZWNyZXQxMjM=
  JWT_SECRET: bXktand0LXNlY3JldA==

# hoặc dùng stringData — K8s tự encode
stringData:
  DB_PASSWORD: "supersecret123"        # không cần encode
  JWT_SECRET: "my-jwt-secret"
```

```bash
# Decode để kiểm tra
kubectl get secret app-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode
```

### Dùng Secret trong Pod

```yaml
spec:
  containers:
    - name: app
      image: username/my-app:latest

      # Cách 1: Inject từng key
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD

      # Cách 2: Inject tất cả
      envFrom:
        - secretRef:
            name: app-secret

      # Cách 3: Mount file (cho TLS cert, SSH key...)
      volumeMounts:
        - name: secret-volume
          mountPath: /app/secrets
          readOnly: true

  volumes:
    - name: secret-volume
      secret:
        secretName: app-secret
```

---

## Ví dụ thực tế — Node.js API hoàn chỉnh

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: my-app
data:
  NODE_ENV: "production"
  PORT: "3000"
  LOG_LEVEL: "info"
---
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-secret
  namespace: my-app
type: Opaque
stringData:
  DATABASE_URL: "postgresql://postgres:password@postgres-service:5432/mydb"
  JWT_SECRET: "your-super-secret-jwt-key"
---
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: my-app
  labels:
    app: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: username/my-api:v1.0
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: api-config
            - secretRef:
                name: api-secret
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: my-app
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 3000
---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: my-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

```bash
# Deploy toàn bộ
kubectl apply -f k8s/

# Kiểm tra
kubectl get all -n my-app

# Xem logs
kubectl logs -f deployment/api -n my-app
```

---

## Cheat Sheet

```bash
# POD
kubectl get pods [-n namespace] [-o wide]
kubectl describe pod <name>
kubectl logs <pod> [-f] [--tail=100]
kubectl exec -it <pod> -- sh
kubectl delete pod <name>

# DEPLOYMENT
kubectl get deployments
kubectl apply -f deployment.yaml
kubectl scale deployment <name> --replicas=3
kubectl set image deployment/<name> app=image:tag
kubectl rollout status deployment/<name>
kubectl rollout undo deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout restart deployment/<name>

# SERVICE
kubectl get services
kubectl apply -f service.yaml
kubectl port-forward service/<name> 3000:80

# INGRESS
kubectl get ingress
kubectl describe ingress <name>

# CONFIGMAP & SECRET
kubectl get configmaps
kubectl get secrets
kubectl describe configmap <name>
kubectl create configmap <name> --from-env-file=.env
kubectl create secret generic <name> --from-literal=KEY=VALUE
```

---

## Liên kết

- [[Kubernetes]] — tổng quan, kiến trúc, kubectl cơ bản
- [[Kubernetes Storage & Config]] — Volume, PV, PVC, StatefulSet
- [[Kubernetes Production]] — Helm, CI/CD, cloud deploy, HPA