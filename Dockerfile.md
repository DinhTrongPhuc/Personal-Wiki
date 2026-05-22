

> Thuộc chuỗi: [[Docker]] → **Dockerfile** → [[Docker Compose]] 
---

## Dockerfile là gì?

File hướng dẫn Docker từng bước build **Image**. Đặt tên là `Dockerfile` (không có extension), đặt ở root project.

### Luồng hoạt động

```
Dockerfile  →  docker build  →  Image  →  docker run  →  Container
(bản thiết kế)   (build)       (snapshot)    (chạy)       (instance)
```

---

## Các lệnh trong Dockerfile

|Lệnh|Dùng khi|Ghi chú|
|---|---|---|
|`FROM`|Chọn base image|Luôn là lệnh đầu tiên|
|`WORKDIR`|Đặt thư mục làm việc|Tạo thư mục nếu chưa có|
|`COPY`|Copy file host → container|Dùng hầu hết các trường hợp|
|`ADD`|Copy + hỗ trợ URL & giải nén `.tar`|Ưu tiên dùng `COPY`|
|`RUN`|Chạy lệnh lúc **build**|Mỗi `RUN` tạo 1 layer mới|
|`CMD`|Lệnh mặc định lúc **run**|Override được khi chạy|
|`ENTRYPOINT`|Lệnh cố định lúc **run**|Không override được|
|`ENV`|Khai báo biến môi trường|Tồn tại cả lúc run|
|`ARG`|Biến chỉ dùng lúc build|Không tồn tại lúc run|
|`EXPOSE`|Khai báo port container lắng nghe|Chỉ mang tính tài liệu|
|`VOLUME`|Tạo mount point cho volume||
|`USER`|Chuyển sang user khác|Dùng cho bảo mật|
|`HEALTHCHECK`|Kiểm tra container còn sống không||

### CMD vs ENTRYPOINT

```dockerfile
# CMD — lệnh mặc định, có thể override
CMD ["node", "server.js"]
# docker run my-app              → chạy node server.js
# docker run my-app npm run dev  → override, chạy npm run dev

# ENTRYPOINT — lệnh cố định, CMD trở thành argument
ENTRYPOINT ["node"]
CMD ["server.js"]
# docker run my-app              → node server.js
# docker run my-app other.js     → node other.js (chỉ override argument)

# Kết hợp phổ biến
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["node", "server.js"]
```

---

## Template theo ngôn ngữ / framework

### Node.js (Express, NestJS...)

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Copy package files TRƯỚC để tận dụng layer cache
# npm install chỉ chạy lại khi package.json thay đổi
COPY package*.json ./
RUN npm ci

# Copy source code SAU
COPY . .

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Node.js — Development (hot reload)

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install         # install dev deps

COPY . .

EXPOSE 3000
CMD ["npm", "run", "dev"]
```

### Node.js — Multi-stage Production

```dockerfile
# ===== Stage 1: Build =====
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# ===== Stage 2: Production =====
# Chỉ copy output đã build → image nhỏ, không có source code, không có devDependencies
FROM node:20-alpine AS runner

WORKDIR /app
ENV NODE_ENV=production

COPY --from=builder /app/dist         ./dist
COPY --from=builder /app/package*.json ./

RUN npm ci --omit=dev     # chỉ install production dependencies

# Bảo mật: chạy bằng non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Next.js

```dockerfile
# ===== Stage 1: Dependencies =====
FROM node:20-alpine AS deps

WORKDIR /app
COPY package*.json ./
RUN npm ci

# ===== Stage 2: Build =====
FROM node:20-alpine AS builder

WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN npm run build

# ===== Stage 3: Production =====
FROM node:20-alpine AS runner

WORKDIR /app
ENV NODE_ENV=production

# Next.js standalone output — tự đóng gói mọi thứ cần thiết
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static     ./.next/static
COPY --from=builder /app/public           ./public

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
```

> Cần bật `standalone` trong `next.config.js`:
> 
> ```js
> module.exports = { output: "standalone" };
> ```

### React (Vite) — Nginx serve static

```dockerfile
# ===== Stage 1: Build =====
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build         # output ra /app/dist

# ===== Stage 2: Serve với Nginx =====
FROM nginx:alpine AS runner

# Copy static files vào nginx
COPY --from=builder /app/dist /usr/share/nginx/html

# Config nginx cho SPA (React Router)
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

```nginx
# nginx.conf — xử lý React Router (SPA)
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # Redirect tất cả về index.html để React Router xử lý
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### Python / FastAPI

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Copy và cài deps trước
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Non-root user
RUN useradd -m appuser
USER appuser

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Python — Multi-stage

```dockerfile
# Stage 1: Build dependencies
FROM python:3.12-slim AS builder

WORKDIR /app
COPY requirements.txt .

RUN pip install --no-cache-dir --user -r requirements.txt

# Stage 2: Production
FROM python:3.12-slim AS runner

WORKDIR /app

# Copy chỉ installed packages
COPY --from=builder /root/.local /root/.local
COPY . .

ENV PATH=/root/.local/bin:$PATH

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Go

```dockerfile
# Stage 1: Build binary
FROM golang:1.22-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./cmd/server

# Stage 2: Minimal image — chỉ có binary
FROM scratch AS runner           # scratch = image trống, cực nhỏ

COPY --from=builder /app/server /server

EXPOSE 8080
ENTRYPOINT ["/server"]
```

---

## `.dockerignore`

> Giống `.gitignore` — bảo Docker không copy các file/thư mục này vào image. **Luôn tạo file này** — không có `.dockerignore` thì `node_modules` (700MB+) bị copy vào!

```
# .dockerignore

# Dependencies
node_modules
.pnp
.pnp.js

# Build output
dist
build
.next
out

# Version control
.git
.gitignore

# Environment files — tuyệt đối không đưa vào image
.env
.env.*
!.env.example

# Logs
*.log
logs

# Testing
coverage
.nyc_output

# IDE
.vscode
.idea
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Docker
Dockerfile*
docker-compose*

# Docs
README.md
CHANGELOG.md
docs
```

---

## Tối ưu Image

### 1. Dùng Alpine — base image nhỏ hơn nhiều

```dockerfile
FROM node:20-alpine    # ~170MB
# thay vì
FROM node:20           # ~1.1GB
# thay vì
FROM node:20-slim      # ~240MB
```

### 2. Tận dụng layer cache — copy package.json trước

```dockerfile
# ✅ ĐÚNG — npm install chỉ chạy lại khi package.json thay đổi
COPY package*.json ./
RUN npm ci
COPY . .               # code thay đổi thường xuyên → copy sau
RUN npm run build

# ❌ SAI — code thay đổi → cache miss → npm install lại mỗi lần build
COPY . .
RUN npm ci
RUN npm run build
```

### 3. Gộp RUN — giảm số layer

```dockerfile
# ✅ Gộp thành 1 layer
RUN apt-get update && \
    apt-get install -y curl wget && \
    rm -rf /var/lib/apt/lists/*    # xóa cache apt

# ❌ Tạo 3 layer riêng
RUN apt-get update
RUN apt-get install -y curl wget
RUN rm -rf /var/lib/apt/lists/*
```

### 4. Multi-stage build — không đưa dev tools vào production

```dockerfile
# Builder stage có toàn bộ dev tools, compiler, devDependencies
# Runner stage chỉ có những gì cần để chạy app
# → Image production nhỏ hơn nhiều, bề mặt tấn công nhỏ hơn
```

### 5. Non-root user — bảo mật

```dockerfile
# Alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Debian/Ubuntu
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser
```

### 6. HEALTHCHECK — tự kiểm tra container còn sống

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

---

## ARG vs ENV

```dockerfile
# ARG — chỉ dùng lúc build, không tồn tại khi container chạy
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine

ARG BUILD_DATE
LABEL build-date=${BUILD_DATE}

# Build với ARG
# docker build --build-arg BUILD_DATE=$(date) .

# ENV — tồn tại cả lúc build lẫn lúc run
ENV NODE_ENV=production
ENV PORT=3000

# Dùng trong CMD
CMD node -e "console.log(process.env.PORT)"  # in ra 3000
```

---

## Build nâng cao

### Chỉ định target stage

```bash
# Build đến stage cụ thể — hữu ích khi debug
docker build --target builder -t my-app:builder .
docker build --target runner  -t my-app:prod .
```

### Build với ARG

```bash
docker build \
  --build-arg NODE_ENV=production \
  --build-arg API_URL=https://api.example.com \
  -t my-app:latest .
```

### BuildKit — build nhanh hơn

```bash
# Bật BuildKit (mặc định trong Docker Desktop)
DOCKER_BUILDKIT=1 docker build -t my-app .

# Xem build output chi tiết
docker build --progress=plain -t my-app .

# Cache từ registry (CI/CD)
docker build \
  --cache-from username/my-app:latest \
  -t username/my-app:latest .
```

---

## Cheat Sheet

```bash
# Build
docker build -t name:tag .
docker build -t name:tag -f Dockerfile.prod .
docker build --target stage-name -t name .
docker build --no-cache -t name .             # bỏ qua cache

# Xem layer
docker history image-name
docker inspect image-name

# Debug — vào image để kiểm tra
docker run -it --rm image-name sh
docker run -it --rm --entrypoint sh image-name
```

---

## Liên kết

- [[Docker]] — tổng quan, lệnh cơ bản, volume, network
- [[Docker Compose]] — kết hợp nhiều service với Dockerfile