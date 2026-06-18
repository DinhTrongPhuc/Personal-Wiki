

> Thuộc chuỗi: [[Docker]] → [[Dockerfile]] → [[Docker Compose]] → [[Docker Deploy]] → **Docker Deploy Free Tier**

---

## So sánh nền tảng

||Railway|Render|Fly.io|
|---|---|---|---|
|Độ khó|⭐ Dễ nhất|⭐⭐ Dễ|⭐⭐⭐ Trung bình|
|Free tier|$5 credit/tháng|750h/tháng|3 VM nhỏ miễn phí|
|Dùng Dockerfile|✅|✅|✅|
|App sleep khi không dùng|❌ Không sleep|✅ Sleep sau 15 phút|❌ Không sleep|
|PostgreSQL miễn phí|✅|✅ 1GB|✅ 3GB|
|Gần VN (Singapore)|✅|✅|✅|
|Phù hợp|Demo nhanh|App nhỏ|Học deploy thật sự|

> **Gợi ý:**
> 
> - Mới học, muốn thấy kết quả nhanh → **Railway**
> - App chạy liên tục, không muốn sleep → **Fly.io**
> - Setup đơn giản qua GitHub → **Render**

---

## Yêu cầu chung

Trước khi deploy bất kỳ nền tảng nào, project cần có:

```
my-app/
├── Dockerfile          ← bắt buộc — xem [[Dockerfile]]
├── .dockerignore       ← nên có để image nhỏ hơn
├── package.json
└── src/
```

App phải lắng nghe trên `0.0.0.0` (không phải `localhost`), và đọc `PORT` từ biến môi trường:

```js
// ✅ Đúng — lắng nghe trên tất cả interface
const PORT = process.env.PORT || 3000;
app.listen(PORT, "0.0.0.0", () => {
  console.log(`Server chạy trên port ${PORT}`);
});

// ❌ Sai — chỉ lắng nghe nội bộ, nền tảng không kết nối được
app.listen(3000, "localhost");
```

---

## Railway

> Dễ nhất — connect GitHub, Railway tự build và deploy.

### Tạo tài khoản & kết nối GitHub

1. Truy cập [railway.app](https://railway.app/)
2. **Login with GitHub**
3. Xác nhận quyền truy cập repo

### Deploy từ GitHub (cách nhanh nhất)

1. Dashboard → **New Project**
2. **Deploy from GitHub repo**
3. Chọn repo chứa `Dockerfile`
4. Railway tự detect và build — chờ ~2 phút
5. Vào **Settings → Networking → Generate Domain** để lấy URL public

### Deploy bằng CLI

```bash
# Cài Railway CLI
npm install -g @railway/cli

# Đăng nhập
railway login

# Trong thư mục project
railway init         # tạo project mới trên Railway
railway up           # build và deploy

# Lệnh hay dùng
railway logs         # xem logs real-time
railway open         # mở app trên browser
railway shell        # SSH vào container đang chạy
railway status       # xem trạng thái deploy
```

### Biến môi trường

```bash
# Qua CLI
railway variables set NODE_ENV=production
railway variables set JWT_SECRET=your-secret

# Hoặc vào Dashboard → project → Variables → paste nội dung .env
```

### Thêm PostgreSQL

```bash
# Qua CLI
railway add postgresql

# Railway tự inject DATABASE_URL vào app
# Xem giá trị: railway variables | grep DATABASE_URL
```

```yaml
# Hoặc qua Dashboard:
# New → Database → PostgreSQL
# DATABASE_URL tự động được thêm vào Variables
```

### Xem logs & debug

```bash
railway logs                    # logs real-time
railway logs --tail 100         # 100 dòng cuối
railway shell                   # vào trong container
```

### Lưu ý Railway free tier

- **$5 credit/tháng** — app nhỏ thường đủ dùng
- Không tự sleep → app luôn sẵn sàng
- Hết credit → app dừng đến đầu tháng sau
- Theo dõi usage: Dashboard → **Usage**

---

## Render

> Free tier có giới hạn sleep — phù hợp app cá nhân, portfolio.

### Tạo tài khoản

1. Truy cập [render.com](https://render.com/)
2. **Sign up with GitHub**

### Deploy Web Service

1. Dashboard → **New** → **Web Service**
2. **Connect a repository** → chọn repo
3. Cấu hình:

```
Name:          my-app
Region:        Singapore
Branch:        main
Runtime:       Docker          ← chọn Docker
Instance Type: Free
```

4. **Create Web Service** → chờ build ~3-5 phút
5. URL dạng: `https://my-app.onrender.com`

### Biến môi trường

Dashboard → service → **Environment** → **Add Environment Variable**

Hoặc paste cả file `.env` vào ô **Secret Files**.

### Thêm PostgreSQL miễn phí

1. Dashboard → **New** → **PostgreSQL**
2. Chọn **Free** tier (1GB storage)
3. Sau khi tạo xong, vào service → **Environment** → thêm:

```
DATABASE_URL = [copy từ PostgreSQL dashboard → Connection String]
```

### Xem logs

Dashboard → service → **Logs** tab — xem real-time trên web.

### Lưu ý Render free tier

- **App sleep sau 15 phút** không có request
- Lần đầu truy cập sau khi sleep: **chờ ~30 giây** để wake up
- Muốn không sleep: upgrade lên **$7/tháng**
- Giải pháp tạm: dùng dịch vụ ping định kỳ như [UptimeRobot](https://uptimerobot.com/) (miễn phí) để giữ app không sleep

```
UptimeRobot → ping app mỗi 5 phút → app không bao giờ sleep
```

---

## Fly.io

> Dùng Docker trực tiếp, gần với deploy VPS thật nhất — tốt để học.

### Cài flyctl

```bash
# macOS
brew install flyctl

# Linux
curl -L https://fly.io/install.sh | sh

# Windows (PowerShell)
pwsh -Command "iwr https://fly.io/install.ps1 -useb | iex"
```

### Tạo tài khoản & đăng nhập

```bash
fly auth signup    # tạo tài khoản mới
# hoặc
fly auth login     # đăng nhập tài khoản có sẵn
```

> Fly.io yêu cầu thêm thẻ tín dụng để xác minh, nhưng **không tính phí** nếu dùng trong free tier.

### Deploy lần đầu

```bash
# Trong thư mục project có Dockerfile
fly launch
```

Fly hỏi một loạt câu hỏi:

```
? App Name (leave blank for auto): my-app-name
? Region: sin (Singapore) ← gần VN nhất
? Would you like to set up a PostgreSQL database? No  ← thêm sau nếu cần
? Would you like to deploy now? Yes
```

Fly tạo file `fly.toml` và deploy lần đầu.

### `fly.toml` — file config

```toml
# fly.toml
app = "my-app-name"
primary_region = "sin"

[build]
  # Fly tự dùng Dockerfile trong thư mục hiện tại

[http_service]
  internal_port = 3000        # port app lắng nghe bên trong container
  force_https = true          # tự redirect HTTP → HTTPS
  auto_stop_machines = true   # tự dừng khi không có traffic (tiết kiệm credit)
  auto_start_machines = true  # tự bật khi có request

[[vm]]
  memory = "256mb"
  cpu_kind = "shared"
  cpus = 1
```

> Nếu app lắng nghe port khác (4000, 8000...) thì sửa `internal_port` cho khớp.

### Các lệnh deploy

```bash
# Deploy version mới
fly deploy

# Deploy và xem logs ngay
fly deploy --watch-logs

# Chỉ build không deploy (kiểm tra Dockerfile có lỗi không)
fly build
```

### Xem logs & debug

```bash
fly logs                    # logs real-time
fly logs --tail             # follow logs
fly status                  # trạng thái app và các machine

# SSH vào container đang chạy (như docker exec)
fly ssh console

# Chạy lệnh trong container
fly ssh console -C "node --version"
fly ssh console -C "ls /app"
```

### Biến môi trường

```bash
# Set từng biến
fly secrets set NODE_ENV=production
fly secrets set JWT_SECRET=your-secret-key
fly secrets set API_KEY=abc123

# Import từ file .env
fly secrets import < .env

# Xem danh sách (chỉ thấy tên, không thấy giá trị — bảo mật)
fly secrets list

# Xóa biến
fly secrets unset VARIABLE_NAME
```

### Thêm PostgreSQL

```bash
# Tạo PostgreSQL instance
fly postgres create \
  --name my-app-db \
  --region sin \
  --initial-cluster-size 1 \
  --vm-size shared-cpu-1x \
  --volume-size 1             # 1GB

# Gắn vào app — tự inject DATABASE_URL
fly postgres attach my-app-db --app my-app-name

# Kết nối trực tiếp vào DB để query
fly postgres connect -a my-app-db

# Xem connection string
fly postgres config show -a my-app-db
```

### Quản lý app

```bash
fly open                    # mở browser
fly status                  # trạng thái
fly info                    # thông tin app (URL, region...)
fly scale count 0           # tắt app — tiết kiệm credit
fly scale count 1           # bật lại
fly apps list               # danh sách tất cả app

# Xóa app (cẩn thận)
fly apps destroy my-app-name
```

### Lưu ý Fly.io free tier

- **3 shared-cpu-1x VM** (256MB RAM mỗi cái) miễn phí mỗi tháng
- **3GB storage** cho PostgreSQL
- Không tự sleep nếu tắt `auto_stop_machines`
- Theo dõi usage: [fly.io/dashboard](https://fly.io/dashboard)

---

## CI/CD tự động deploy khi push code

> Áp dụng được cho cả 3 nền tảng.

### Railway — GitHub tự động deploy

Railway mặc định đã tự động deploy mỗi khi push lên branch đã connect. Không cần cấu hình thêm.

```
git push origin main  →  Railway tự detect  →  build  →  deploy  ✅
```

### Render — GitHub tự động deploy

Render cũng tự động deploy khi push. Cấu hình trong Dashboard → service → **Settings** → **Auto-Deploy**.

### Fly.io — GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy to Fly.io

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: superfly/flyctl-actions/setup-flyctl@master

      - name: Deploy
        run: fly deploy --remote-only
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

```bash
# Lấy API token để cho vào GitHub Secrets
fly tokens create deploy -x 999999h

# Vào GitHub: Settings → Secrets → New secret
# Tên: FLY_API_TOKEN
# Giá trị: token vừa tạo
```

---

## Troubleshooting — Lỗi thường gặp

### App deploy thành công nhưng không mở được

```bash
# Kiểm tra app có lắng nghe đúng host và port không
# App phải dùng 0.0.0.0 và đọc PORT từ env

# Node.js
app.listen(process.env.PORT || 3000, "0.0.0.0")

# Python FastAPI
uvicorn main:app --host 0.0.0.0 --port ${PORT:-8000}
```

### Build thất bại

```bash
# Xem logs build chi tiết
railway logs --build        # Railway
fly logs                    # Fly.io

# Test build trên máy local trước
docker build -t my-app .
docker run -p 3000:3000 -e PORT=3000 my-app
```

### App crash sau khi start

```bash
# Xem logs ngay sau deploy
railway logs
fly logs

# Lỗi thường gặp:
# - Thiếu biến môi trường → thêm vào Variables/Secrets
# - Sai DATABASE_URL → kiểm tra lại connection string
# - Port không khớp → sửa internal_port trong fly.toml
```

### Kết nối database thất bại

```bash
# Kiểm tra DATABASE_URL đã được set chưa
railway variables | grep DATABASE   # Railway
fly secrets list | grep DATABASE    # Fly.io

# Test kết nối từ trong container
fly ssh console
# Trong container:
node -e "const { Pool } = require('pg'); const p = new Pool(); p.query('SELECT 1').then(console.log)"
```

---

## Cheat Sheet

```bash
# ===== RAILWAY =====
npm install -g @railway/cli
railway login
railway init
railway up
railway logs
railway open
railway variables set KEY=VALUE
railway shell

# ===== FLY.IO =====
fly launch                          # setup lần đầu
fly deploy                          # deploy version mới
fly logs                            # xem logs
fly open                            # mở browser
fly ssh console                     # SSH vào container
fly secrets set KEY=VALUE           # set biến môi trường
fly secrets import < .env           # import từ .env
fly scale count 0                   # tắt app
fly scale count 1                   # bật lại
fly postgres create --name db       # tạo postgres
fly postgres attach db              # gắn postgres vào app
```

---

## Liên kết

- [[Docker]] — tổng quan Docker
- [[Dockerfile]] — viết Dockerfile cho app
- [[Docker Compose]] — chạy nhiều service local
- [[Docker Deploy]] — deploy lên VPS thật với Nginx + SSL