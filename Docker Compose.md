
> Thuộc chuỗi: [[Docker]] → [[Dockerfile]] → [[Docker Compose]] → [[Docker Deploy]]

---

## Docker Compose là gì?

Công cụ định nghĩa và chạy **nhiều container** cùng lúc bằng một file `docker-compose.yml`. Thay vì gõ nhiều lệnh `docker run` dài, chỉ cần `docker compose up`.

```
docker-compose.yml
       ↓
docker compose up
       ↓
┌──────────────────────────────────┐
│  frontend  │  backend  │  db     │  ← các service chạy cùng lúc
│ :3000      │  :4000    │  :5432  │
└──────────────────────────────────┘
         cùng 1 network nội bộ
```

---

## Cấu trúc file

```yaml
# docker-compose.yml
version: "3.9"

services:               # các container cần chạy
  tên-service:
    image: image:tag    # dùng image có sẵn
    # hoặc
    build:              # build từ Dockerfile — xem [[Dockerfile]]
      context: .        # thư mục chứa Dockerfile
      dockerfile: Dockerfile
      args:
        NODE_ENV: development
    container_name: tên-container   # tên tùy chỉnh (optional)
    ports:
      - "host:container"
    environment:
      - BIEN=gia_tri
      # hoặc dạng map
      BIEN: gia_tri
    env_file:
      - .env
    volumes:
      - tên-volume:/đường/dẫn/container   # named volume
      - ./host/path:/container/path        # bind mount
    depends_on:
      tên-service-khác:
        condition: service_healthy         # chờ healthy
    networks:
      - tên-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M

volumes:                # khai báo named volumes
  tên-volume:
    driver: local

networks:               # khai báo networks
  tên-network:
    driver: bridge
```

---

## Lệnh Docker Compose

```bash
# Khởi động tất cả services
docker compose up
docker compose up -d                    # chạy nền (detach)
docker compose up --build               # build lại image trước khi up
docker compose up -d --build            # build + chạy nền
docker compose up --force-recreate      # tạo lại container dù không có thay đổi

# Chỉ start một số service
docker compose up -d db redis

# Dừng
docker compose stop                     # dừng nhưng không xóa container
docker compose down                     # dừng và xóa container + network
docker compose down -v                  # ⚠️ xóa luôn volume → mất data
docker compose down --rmi all           # xóa luôn image

# Xem trạng thái
docker compose ps

# Log
docker compose logs
docker compose logs -f                  # follow tất cả
docker compose logs -f api              # follow service 'api'
docker compose logs --tail 50 api

# Vào container
docker compose exec api sh
docker compose exec db psql -U postgres -d mydb

# Chạy lệnh một lần (không start service thường trực)
docker compose run --rm api npm run migrate
docker compose run --rm api npm test

# Restart
docker compose restart
docker compose restart api

# Scale — chạy nhiều instance của 1 service
docker compose up -d --scale api=3

# Pull image mới nhất cho tất cả service
docker compose pull
docker compose pull api

# Xem config đã được merge
docker compose config
```

---

## Template: Node.js + PostgreSQL + Redis

```yaml
# docker-compose.yml
version: "3.9"

services:
  api:
    build: .
    container_name: my-api
    ports:
      - "3000:3000"
    environment:
      NODE_ENV:     development
      DATABASE_URL: postgresql://postgres:password@db:5432/mydb
      REDIS_URL:    redis://cache:6379
    volumes:
      - .:/app                   # bind mount — hot reload
      - /app/node_modules        # giữ nguyên node_modules trong container
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    networks:
      - app-network
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    container_name: my-postgres
    environment:
      POSTGRES_USER:     postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB:       mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql  # chạy khi init
    ports:
      - "5432:5432"              # expose để dùng với DB client (TablePlus, DBeaver...)
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  cache:
    image: redis:7-alpine
    container_name: my-redis
    command: redis-server --requirepass redispassword   # đặt password
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"
    networks:
      - app-network

volumes:
  postgres-data:
  redis-data:

networks:
  app-network:
    driver: bridge
```

---

## Template: Next.js + NestJS + PostgreSQL

```yaml
version: "3.9"

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      target: runner             # multi-stage: chỉ build đến stage 'runner'
    container_name: nextjs-app
    ports:
      - "3000:3000"
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:4000
    depends_on:
      - backend
    networks:
      - app-network
    restart: unless-stopped

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: nestjs-api
    ports:
      - "4000:4000"
    environment:
      NODE_ENV:     development
      DATABASE_URL: postgresql://postgres:password@db:5432/mydb
      JWT_SECRET:   ${JWT_SECRET}          # lấy từ .env
    volumes:
      - ./backend:/app
      - /app/node_modules
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    container_name: postgres-db
    environment:
      POSTGRES_USER:     ${POSTGRES_USER:-postgres}      # dùng giá trị mặc định nếu không có
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-password}
      POSTGRES_DB:       ${POSTGRES_DB:-mydb}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - app-network

volumes:
  postgres-data:

networks:
  app-network:
```

---

## Template: Full Stack + Nginx + Redis + Worker

```yaml
version: "3.9"

services:
  # Reverse proxy
  nginx:
    image: nginx:alpine
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/ssl:/etc/nginx/ssl
    depends_on:
      - api
      - frontend
    networks:
      - app-network
    restart: always

  frontend:
    build: ./frontend
    expose:
      - "3000"               # chỉ expose nội bộ — nginx làm gateway
    networks:
      - app-network
    restart: unless-stopped

  api:
    build: ./backend
    expose:
      - "4000"
    environment:
      DATABASE_URL: postgresql://postgres:password@db:5432/mydb
      REDIS_URL:    redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    networks:
      - app-network
    restart: unless-stopped

  # Background worker — xử lý queue
  worker:
    build: ./backend
    command: node dist/worker.js    # override CMD trong Dockerfile
    environment:
      REDIS_URL: redis://cache:6379
    depends_on:
      cache:
        condition: service_started
    networks:
      - app-network
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER:     postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB:       mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5
    networks:
      - app-network

  cache:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    networks:
      - app-network

volumes:
  postgres-data:
  redis-data:

networks:
  app-network:
```

---

## Environment Variables

### File `.env`

```bash
# .env — tự động đọc bởi Docker Compose
POSTGRES_USER=postgres
POSTGRES_PASSWORD=supersecret123
POSTGRES_DB=mydb
JWT_SECRET=your-super-secret-key
API_PORT=4000
```

```yaml
# docker-compose.yml — dùng biến từ .env
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER:     ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB:       ${POSTGRES_DB}

  api:
    ports:
      - "${API_PORT}:4000"
    env_file:
      - .env            # load toàn bộ .env vào container
      - .env.local      # override local (optional)
```

### Giá trị mặc định

```yaml
environment:
  # Dùng giá trị mặc định nếu biến không có trong .env
  NODE_ENV:  ${NODE_ENV:-development}
  DB_PORT:   ${DB_PORT:-5432}
  LOG_LEVEL: ${LOG_LEVEL:-info}
```

> **Bảo mật:** Luôn thêm `.env` vào `.gitignore`. Commit file `.env.example` thay thế.

---

## Dev vs Production — nhiều file Compose

### Cách tổ chức

```
project/
├── docker-compose.yml          ← base config dùng chung
├── docker-compose.dev.yml      ← override cho development
├── docker-compose.prod.yml     ← override cho production
├── .env
└── .env.production
```

### Base config

```yaml
# docker-compose.yml
version: "3.9"

services:
  api:
    build: .
    environment:
      DATABASE_URL: postgresql://postgres:password@db:5432/mydb
    networks:
      - app-network

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER:     postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB:       mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network

volumes:
  postgres-data:

networks:
  app-network:
```

### Override cho Development

```yaml
# docker-compose.dev.yml
services:
  api:
    volumes:
      - .:/app             # bind mount — hot reload
      - /app/node_modules
    command: npm run dev   # override CMD
    environment:
      NODE_ENV: development
    ports:
      - "3000:3000"
    # Thêm debugger port
    ports:
      - "3000:3000"
      - "9229:9229"        # Node.js debugger

  db:
    ports:
      - "5432:5432"        # expose DB ra ngoài để debug
```

### Override cho Production

```yaml
# docker-compose.prod.yml
services:
  api:
    image: username/my-api:latest    # dùng image đã build sẵn
    restart: always
    environment:
      NODE_ENV: production
    # không expose port — chỉ nginx mới gọi được
    expose:
      - "3000"

  db:
    restart: always
    # không expose port ra ngoài
```

### Chạy với nhiều file

```bash
# Development — gộp base + dev
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Hoặc dùng COMPOSE_FILE env variable
export COMPOSE_FILE=docker-compose.yml:docker-compose.dev.yml
docker compose up -d
```

---

## Healthcheck & depends_on

```yaml
services:
  api:
    depends_on:
      db:
        condition: service_healthy      # chờ db healthy
      cache:
        condition: service_started      # chỉ cần started
      migrations:
        condition: service_completed_successfully  # chờ job hoàn thành

  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s     # kiểm tra mỗi 10s
      timeout: 5s       # timeout sau 5s
      retries: 5        # thử lại 5 lần trước khi báo unhealthy
      start_period: 30s # grace period sau khi container start

  # One-time job — chạy migration rồi exit
  migrations:
    build: .
    command: npm run migrate
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network
```

---

## Volumes nâng cao

```yaml
volumes:
  # Named volume cơ bản — Docker quản lý
  postgres-data:

  # Named volume với driver options
  uploads:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/uploads    # map đến thư mục cụ thể trên host

  # External volume — volume đã tạo sẵn bằng docker volume create
  shared-data:
    external: true

services:
  api:
    volumes:
      # Named volume
      - postgres-data:/var/lib/postgresql/data

      # Bind mount — thư mục relative
      - ./src:/app/src
      - ./public:/app/public

      # Bind mount — anonymous (không đặt tên) — tránh dùng
      - /app/tmp

      # Read-only mount
      - ./config:/app/config:ro

      # tmpfs — lưu trong RAM, không persist
      - type: tmpfs
        target: /app/tmp
```

---

## Networking nâng cao

```yaml
networks:
  # Network mặc định — các service trong compose tự động giao tiếp
  app-network:
    driver: bridge

  # Network riêng cho DB — chỉ backend mới truy cập được
  db-network:
    driver: bridge
    internal: true    # không có internet access

services:
  frontend:
    networks:
      - app-network   # chỉ trong app-network → không gọi được db trực tiếp

  backend:
    networks:
      - app-network   # giao tiếp với frontend
      - db-network    # giao tiếp với db

  db:
    networks:
      - db-network    # chỉ trong db-network → chỉ backend gọi được
```

---

## Các pattern hay dùng

### Override entrypoint để debug

```bash
# Vào container mà không chạy app
docker compose run --rm --entrypoint sh api

# Chạy lệnh một lần
docker compose run --rm api npm run seed
docker compose run --rm api npx prisma migrate dev
```

### Watch logs nhiều service

```bash
# Xem log của nhiều service cùng lúc
docker compose logs -f api worker

# Xem log với timestamp
docker compose logs -f -t api
```

### Restart policy

```yaml
services:
  api:
    restart: "no"              # không tự restart (mặc định)
    restart: always            # luôn restart
    restart: on-failure        # chỉ restart khi exit code != 0
    restart: unless-stopped    # restart trừ khi bị stop thủ công
```

---

## Cheat Sheet

```bash
# Khởi động
docker compose up -d
docker compose up -d --build          # build lại
docker compose up -d api db           # chỉ một số service

# Dừng
docker compose stop                   # giữ container
docker compose down                   # xóa container + network
docker compose down -v                # ⚠️ xóa luôn volume

# Theo dõi
docker compose ps
docker compose logs -f [service]
docker compose stats

# Thao tác
docker compose exec service sh
docker compose run --rm service lệnh
docker compose restart [service]
docker compose pull

# Nhiều file
docker compose -f base.yml -f dev.yml up -d
```

---

## Liên kết

- [[Docker]] — tổng quan, lệnh cơ bản, volume, network
- [[Dockerfile]] — viết Dockerfile cho từng service
