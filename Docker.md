
>Xem chi tiết từng phần:
> 
> - [[Dockerfile]] — viết Dockerfile, multi-stage build, tối ưu image
> - [[Docker Compose]] — chạy nhiều service, môi trường dev/prod

---

## Docker là gì?

- Công cụ đóng gói ứng dụng vào **container** — môi trường chạy độc lập, nhất quán trên mọi máy
- Giải quyết vấn đề _"chạy được trên máy tôi nhưng lỗi trên server"_
- Nhẹ hơn Virtual Machine vì dùng chung kernel của host OS

### Các khái niệm cốt lõi

| Khái niệm              | Giải thích                                                          |
| ---------------------- | ------------------------------------------------------------------- |
| **Image**              | Bản thiết kế — snapshot của app + dependencies. Không thay đổi được |
| **Container**          | Instance đang chạy từ Image — có thể start/stop/xóa                 |
| **[[Dockerfile]]**     | File hướng dẫn cách build Image                                     |
| **[[Docker Compose]]** | Công cụ chạy nhiều container cùng lúc                               |
| **Volume**             | Lưu trữ data bền vững, không mất khi container bị xóa               |
| **Network**            | Mạng nội bộ để các container giao tiếp với nhau                     |
| **Registry**           | Kho chứa Image — Docker Hub, GitHub Container Registry...           |

### VM vs Container

```
Virtual Machine                    Container
┌─────────────────────┐            ┌─────────────────────┐
│   App A  │  App B   │            │   App A  │  App B   │
│──────────┼──────────│            │──────────┼──────────│
│  OS A    │  OS B    │            │  Libs A  │  Libs B  │
│──────────┴──────────│            │──────────┴──────────│
│    Hypervisor       │            │   Docker Engine      │
│─────────────────────│            │─────────────────────│
│    Host OS          │            │    Host OS           │
│─────────────────────│            │─────────────────────│
│    Hardware         │            │    Hardware          │
└─────────────────────┘            └─────────────────────┘
Mỗi VM có OS riêng → nặng          Dùng chung OS → nhẹ, nhanh hơn
```

---

## Cài đặt

### macOS / Windows

Tải **Docker Desktop** tại: https://www.docker.com/products/docker-desktop

### Ubuntu / Debian

```bash
# Cài đặt
curl -fsSL https://get.docker.com | sh

# Thêm user vào group docker (không cần sudo mỗi lần)
sudo usermod -aG docker $USER

# Đăng xuất và đăng nhập lại, sau đó kiểm tra
docker --version
docker compose version
```

---

## Image

```bash
# Tìm image trên Docker Hub
docker search nginx

# Tải image về máy
docker pull nginx
docker pull node:20-alpine        # chỉ định tag/version

# Xem image đã có
docker images

# Xóa image
docker rmi nginx
docker rmi $(docker images -q)    # xóa tất cả

# Build image từ Dockerfile — xem [[Dockerfile]]
docker build -t my-app .
docker build -t my-app:1.0.0 .
docker build -t my-app:latest -f Dockerfile.prod .

# Xem kích thước từng layer
docker history my-app
```

---

## Container

```bash
# Chạy container từ image
docker run nginx

# Chạy với các options phổ biến
docker run \
  -d \                            # detach — chạy nền
  -p 3000:3000 \                  # map port host:container
  --name my-app \                 # đặt tên container
  --rm \                          # tự xóa khi stop
  -e NODE_ENV=production \        # biến môi trường
  --network my-network \          # gắn vào network
  -v my-data:/app/data \          # gắn volume
  node:20-alpine

# Xem container
docker ps                         # đang chạy
docker ps -a                      # tất cả kể cả đã stop

# Quản lý vòng đời
docker start   my-app
docker stop    my-app
docker restart my-app

# Xóa container
docker rm my-app
docker rm -f my-app               # force — dù đang chạy
docker rm $(docker ps -aq)        # xóa tất cả

# Log
docker logs my-app
docker logs -f my-app             # follow — real-time
docker logs --tail 50 my-app      # 50 dòng cuối

# Vào trong container
docker exec -it my-app sh         # alpine
docker exec -it my-app bash       # ubuntu/debian

# Thông tin chi tiết
docker inspect my-app

# Resource usage
docker stats
```

---

## Volume

> Data trong container **mất khi container bị xóa**. Volume lưu data bền vững bên ngoài container.

```bash
# Quản lý volume
docker volume create my-data
docker volume ls
docker volume inspect my-data
docker volume rm my-data
docker volume prune               # xóa tất cả không dùng

# Named volume — Docker quản lý, dùng cho production/DB
docker run -d \
  -v my-data:/app/data \
  --name my-app my-image

# Bind mount — map thư mục host vào container, dùng khi dev (hot reload)
docker run -d \
  -v $(pwd):/app \
  --name my-app my-image
```

### Named Volume vs Bind Mount

||Named Volume|Bind Mount|
|---|---|---|
|Khai báo|`-v my-data:/app/data`|`-v $(pwd):/app`|
|Docker quản lý|✅ Có|❌ Không|
|Dùng khi|Production, DB data|Dev local, hot reload|
|Hiệu năng|Tốt|Phụ thuộc OS|

---

## Network

```bash
# Tạo network
docker network create my-network
docker network ls
docker network inspect my-network
docker network rm my-network

# Gắn container vào network khi chạy
docker run -d --network my-network --name backend  my-backend-image
docker run -d --network my-network --name frontend my-frontend-image

# Gắn thêm network cho container đang chạy
docker network connect    my-network my-app
docker network disconnect my-network my-app
```

> Các container **cùng network** có thể gọi nhau qua **tên container** thay vì IP: `http://backend:3000` thay vì `http://localhost:3000`

---

## Registry — Push & Pull Image

```bash
# Đăng nhập Docker Hub
docker login

# Tag image trước khi push
docker tag my-app username/my-app:latest
docker tag my-app username/my-app:1.0.0

# Push lên Docker Hub
docker push username/my-app:latest

# Pull về dùng (trên server khác)
docker pull username/my-app:latest

# GitHub Container Registry
docker tag my-app ghcr.io/username/my-app:latest
docker push ghcr.io/username/my-app:latest
```

---

## Dọn dẹp

```bash
# Xem disk đang dùng
docker system df

# Xóa tất cả không dùng (container stopped, image dangling, network, cache)
docker system prune

# Xóa luôn cả volume ⚠️ mất data
docker system prune --volumes

# Xóa riêng từng loại
docker container prune            # container đã stop
docker image prune                # image dangling (không có tag)
docker image prune -a             # tất cả image không dùng
docker volume prune               # volume không gắn với container nào
docker network prune              # network không dùng
docker builder prune              # build cache
```

---

## Các lỗi thường gặp

### Port đã được dùng

```bash
# Lỗi: bind: address already in use
lsof -i :3000
kill -9 <PID>

# Hoặc đổi port mapping
docker run -p 3001:3000 my-app    # dùng port 3001 trên host
```

### Container không kết nối được với nhau

```bash
# Kiểm tra container có cùng network không
docker inspect container-a | grep -A 10 Networks
docker network inspect my-network

# Các container phải cùng network mới gọi được nhau qua tên
# http://db:5432 chỉ hoạt động khi cả hai trong cùng network
```

### Permission denied với volume (Linux)

```dockerfile
# File tạo trong container mặc định thuộc root
# Fix: tạo non-root user trong Dockerfile — xem [[Dockerfile]]
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

### Container restart liên tục

```bash
# Xem log để tìm nguyên nhân
docker logs --tail 100 my-app

# Xem exit code
docker inspect my-app | grep ExitCode
```

---

## Cheat Sheet

```bash
# IMAGE
docker pull image:tag             # tải image
docker build -t name:tag .        # build image
docker images                     # xem image
docker rmi image                  # xóa image
docker push username/image:tag    # push lên registry

# CONTAINER
docker run -d -p 3000:3000 --name app image
docker ps / docker ps -a
docker start / stop / restart app
docker rm -f app
docker logs -f app
docker exec -it app sh

# VOLUME
docker volume create name
docker volume ls / inspect / rm

# NETWORK
docker network create name
docker network ls / inspect

# DỌN DẸP
docker system prune
docker system df
```

---

## Liên kết

- [[Dockerfile]] — viết Dockerfile, multi-stage build, tối ưu image
- [[Docker Compose]] — chạy nhiều service, dev vs prod
