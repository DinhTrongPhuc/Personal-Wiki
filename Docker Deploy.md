
> Thuộc chuỗi: [[Docker]] → [[Dockerfile]] → [[Docker Compose]] → **Docker Deploy**

---

## Tổng quan luồng deploy

```
Code → Git push → CI/CD build image → Push registry → Pull trên server → Compose up
                     (GitHub Actions)   (Docker Hub)     (VPS/Server)
```

---

## Chuẩn bị Server (Ubuntu)

### Cài Docker

```bash
# Cài Docker Engine
curl -fsSL https://get.docker.com | sh

# Thêm user vào group docker — không cần sudo mỗi lần
sudo usermod -aG docker $USER

# Đăng xuất và đăng nhập lại
newgrp docker

# Kiểm tra
docker --version
docker compose version
```

### Bảo mật server cơ bản

```bash
# Cập nhật hệ thống
sudo apt update && sudo apt upgrade -y

# Cài ufw firewall
sudo apt install ufw -y

# Cấu hình firewall
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh           # port 22
sudo ufw allow 80            # HTTP
sudo ufw allow 443           # HTTPS
sudo ufw enable

# Xem trạng thái
sudo ufw status
```

### Cấu trúc thư mục trên server

```
/home/ubuntu/
└── my-app/
    ├── docker-compose.yml
    ├── docker-compose.prod.yml
    ├── .env                    # ⚠️ không commit — tạo thủ công trên server
    └── nginx/
        ├── conf.d/
        │   └── default.conf
        └── ssl/               # certificate (nếu không dùng Certbot)
```

---

## Workflow Deploy

### Cách 1 — Deploy thủ công

```bash
# ===== Máy local =====

# Build image production
docker build -t username/my-app:latest .
docker build -t username/my-app:$(git rev-parse --short HEAD) .   # tag bằng git commit

# Push lên Docker Hub
docker login
docker push username/my-app:latest

# ===== Trên server (SSH vào) =====
ssh ubuntu@server-ip

cd /home/ubuntu/my-app

# Pull image mới
docker compose pull

# Deploy không downtime
docker compose up -d --no-deps api     # chỉ restart service 'api'

# Hoặc restart toàn bộ
docker compose down && docker compose up -d

# Dọn image cũ
docker image prune -f
```

### Cách 2 — Deploy qua SSH script

```bash
# deploy.sh — chạy trên máy local
#!/bin/bash
set -e

SERVER="ubuntu@your-server-ip"
APP_DIR="/home/ubuntu/my-app"
IMAGE="username/my-app"
TAG=$(git rev-parse --short HEAD)

echo "Building image: $IMAGE:$TAG"
docker build -t $IMAGE:$TAG -t $IMAGE:latest .

echo "Pushing to registry..."
docker push $IMAGE:$TAG
docker push $IMAGE:latest

echo "Deploying to server..."
ssh $SERVER "
  cd $APP_DIR &&
  docker compose pull &&
  docker compose up -d --force-recreate &&
  docker image prune -f
"

echo "Deploy thành công! Version: $TAG"
```

```bash
chmod +x deploy.sh
./deploy.sh
```

---

## Nginx Reverse Proxy

> Nginx đứng trước, nhận request từ internet và chuyển tiếp vào các container.

### Docker Compose với Nginx

```yaml
# docker-compose.prod.yml
version: "3.9"

services:
  nginx:
    image: nginx:alpine
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./certbot/conf:/etc/letsencrypt          # SSL certificates
      - ./certbot/www:/var/www/certbot           # ACME challenge
    depends_on:
      - api
    networks:
      - app-network
    restart: always

  api:
    image: username/my-api:latest
    expose:
      - "4000"                  # không mở port ra ngoài — chỉ nginx gọi được
    environment:
      NODE_ENV: production
    networks:
      - app-network
    restart: always

  certbot:
    image: certbot/certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot

networks:
  app-network:
```

### Nginx config — HTTP (trước khi có SSL)

```nginx
# nginx/conf.d/default.conf
server {
    listen 80;
    server_name example.com www.example.com;

    # ACME challenge cho Certbot
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # Redirect sang HTTPS sau khi có cert
    # location / {
    #     return 301 https://$host$request_uri;
    # }

    # Hoặc proxy thẳng khi chưa có SSL
    location / {
        proxy_pass         http://api:4000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade    $http_upgrade;
        proxy_set_header   Connection "upgrade";
        proxy_set_header   Host       $host;
        proxy_set_header   X-Real-IP  $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### Nginx config — HTTPS (sau khi có SSL)

```nginx
# nginx/conf.d/default.conf
server {
    listen 80;
    server_name example.com www.example.com;

    # ACME challenge
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # Redirect tất cả HTTP → HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # SSL certificates
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # SSL settings tốt
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header Referrer-Policy "no-referrer-when-downgrade";

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    # Proxy đến API
    location /api/ {
        proxy_pass         http://api:4000/;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade    $http_upgrade;
        proxy_set_header   Connection "upgrade";
        proxy_set_header   Host       $host;
        proxy_set_header   X-Real-IP  $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;

        # Timeout
        proxy_connect_timeout 60s;
        proxy_read_timeout    60s;
    }

    # Proxy đến Frontend
    location / {
        proxy_pass         http://frontend:3000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade    $http_upgrade;
        proxy_set_header   Connection "upgrade";
        proxy_set_header   Host       $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

## SSL với Let's Encrypt (Certbot)

### Lấy certificate lần đầu

```bash
# 1. Đảm bảo nginx đang chạy với HTTP config (chưa có SSL)
docker compose up -d nginx

# 2. Lấy certificate
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path /var/www/certbot \
  --email your@email.com \
  --agree-tos \
  --no-eff-email \
  -d example.com \
  -d www.example.com

# 3. Sau khi có cert, cập nhật nginx config sang HTTPS rồi reload
docker compose exec nginx nginx -s reload
```

### Tự động gia hạn certificate

```bash
# Thêm vào crontab — gia hạn mỗi ngày lúc 3:00 AM
crontab -e

0 3 * * * cd /home/ubuntu/my-app && docker compose run --rm certbot renew && docker compose exec nginx nginx -s reload
```

---

## CI/CD với GitHub Actions

### Luồng đầy đủ: Test → Build → Push → Deploy

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  # ===== Job 1: Test =====
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - run: npm ci
      - run: npm test
      - run: npm run build    # kiểm tra build không lỗi

  # ===== Job 2: Build & Push Image =====
  build:
    needs: test               # chỉ chạy khi test pass
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      # Bật Docker BuildKit — build nhanh hơn, hỗ trợ cache
      - uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}   # dùng Access Token, không dùng password

      - name: Build và push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/my-app:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/my-app:${{ github.sha }}
          cache-from: type=gha        # cache GitHub Actions
          cache-to:   type=gha,mode=max

  # ===== Job 3: Deploy =====
  deploy:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host:     ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key:      ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /home/ubuntu/my-app

            # Pull image mới
            docker compose -f docker-compose.yml -f docker-compose.prod.yml pull

            # Deploy không downtime cho api service
            docker compose -f docker-compose.yml -f docker-compose.prod.yml \
              up -d --no-deps --force-recreate api

            # Dọn image cũ
            docker image prune -f

            echo "Deploy thành công: ${{ github.sha }}"
```

### Cài Secrets trong GitHub

Vào: `Settings → Secrets and variables → Actions → New repository secret`

|Secret|Giá trị|
|---|---|
|`DOCKERHUB_USERNAME`|username Docker Hub|
|`DOCKERHUB_TOKEN`|Access Token (không dùng password)|
|`SERVER_HOST`|IP hoặc domain của server|
|`SERVER_USER`|SSH user (thường là `ubuntu`)|
|`SERVER_SSH_KEY`|Nội dung file `~/.ssh/id_rsa` (private key)|

### Tạo SSH key cho CI/CD

```bash
# Tạo SSH key pair
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/github_actions

# Copy public key lên server
ssh-copy-id -i ~/.ssh/github_actions.pub ubuntu@server-ip

# Copy nội dung private key vào GitHub Secret
cat ~/.ssh/github_actions
```

### Deploy nhiều môi trường (Staging + Production)

```yaml
# .github/workflows/deploy.yml
on:
  push:
    branches:
      - develop    # deploy lên staging
      - main       # deploy lên production

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}

    steps:
      - name: Deploy
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}   # mỗi environment có secrets riêng
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /home/ubuntu/my-app
            docker compose pull
            docker compose up -d
```

---

## Zero Downtime Deploy

### Với Docker Compose — rolling update từng service

```bash
# Chỉ restart service 'api', không ảnh hưởng db hay nginx
docker compose up -d --no-deps --force-recreate api

# Scale lên trước, rồi scale xuống (cần load balancer)
docker compose up -d --scale api=2    # chạy 2 instance
# sau khi verify OK:
docker compose up -d --scale api=1    # scale về 1
```

### Health check trước khi chuyển traffic

```bash
#!/bin/bash
# health-check.sh

URL="http://localhost:3000/health"
MAX_RETRIES=30
RETRY_INTERVAL=2

echo "Chờ service healthy..."
for i in $(seq 1 $MAX_RETRIES); do
  if curl -sf $URL > /dev/null 2>&1; then
    echo "Service healthy!"
    exit 0
  fi
  echo "Lần $i/$MAX_RETRIES — chờ..."
  sleep $RETRY_INTERVAL
done

echo "Service không healthy sau $(($MAX_RETRIES * $RETRY_INTERVAL))s"
exit 1
```

---

## Monitoring

### Xem logs trên server

```bash
# Log của tất cả service
docker compose logs -f

# Log 24 giờ qua
docker compose logs --since 24h api

# Log theo thời gian cụ thể
docker compose logs --since "2024-01-01T00:00:00" api

# Lọc log theo từ khóa
docker compose logs api 2>&1 | grep "ERROR"
```

### Resource usage

```bash
# CPU, RAM, Network của tất cả container
docker stats

# Snapshot (không follow)
docker stats --no-stream

# Chỉ xem một container
docker stats my-api
```

### Portainer — UI quản lý Docker

```yaml
# docker-compose.yml — thêm Portainer
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer-data:/data
    restart: always

volumes:
  portainer-data:
```

```bash
# Truy cập: http://server-ip:9000
# ⚠️ Nên bảo vệ port 9000 bằng VPN hoặc chỉ mở từ IP cụ thể
sudo ufw allow from your-ip to any port 9000
```

---

## Backup Database

```bash
# Backup PostgreSQL
docker compose exec db pg_dump -U postgres mydb > backup_$(date +%Y%m%d).sql

# Restore
docker compose exec -T db psql -U postgres mydb < backup_20240101.sql

# Script backup tự động — thêm vào crontab
#!/bin/bash
BACKUP_DIR="/home/ubuntu/backups"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR
docker compose -f /home/ubuntu/my-app/docker-compose.yml exec -T db \
  pg_dump -U postgres mydb | gzip > "$BACKUP_DIR/db_$DATE.sql.gz"

# Giữ lại 7 ngày gần nhất
find $BACKUP_DIR -name "*.sql.gz" -mtime +7 -delete

echo "Backup xong: db_$DATE.sql.gz"
```

```bash
# Crontab — backup mỗi ngày lúc 2:00 AM
0 2 * * * /home/ubuntu/scripts/backup.sh >> /var/log/backup.log 2>&1
```

---

## Rollback

```bash
# Rollback về version trước
docker compose down

# Cập nhật IMAGE_TAG trong .env hoặc docker-compose.prod.yml
# VD: đổi từ username/my-app:abc123 về username/my-app:def456

docker compose up -d

# Hoặc dùng git commit hash làm tag
docker pull username/my-app:previous-commit-sha
docker tag username/my-app:previous-commit-sha username/my-app:latest
docker compose up -d --force-recreate api
```

---

## Cheat Sheet

```bash
# ===== CÀI ĐẶT SERVER =====
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# ===== DEPLOY THỦ CÔNG =====
docker build -t username/app:latest .
docker push username/app:latest
# Trên server:
docker compose pull && docker compose up -d

# ===== NGINX =====
docker compose exec nginx nginx -t        # kiểm tra config
docker compose exec nginx nginx -s reload # reload không downtime

# ===== SSL =====
docker compose run --rm certbot certonly --webroot ...
docker compose run --rm certbot renew

# ===== MONITORING =====
docker stats --no-stream
docker compose logs --since 1h api
docker compose ps

# ===== BACKUP =====
docker compose exec -T db pg_dump -U postgres mydb > backup.sql

# ===== DỌN DẸP =====
docker image prune -f
docker system prune --volumes
```

---

## Liên kết

- [[Docker]] — tổng quan, lệnh cơ bản, volume, network
- [[Dockerfile]] — viết Dockerfile, multi-stage build
- [[Docker Compose]] — cấu hình multi-service, dev vs prod