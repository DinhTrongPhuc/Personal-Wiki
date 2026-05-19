
# CI/CD — Kiến thức thực chiến qua project `vn-payment`
> [!abstract] Tóm tắt Ghi chép này tổng hợp toàn bộ kiến thức CI/CD đã trải qua khi xây dựng pipeline tự động hóa cho thư viện thanh toán `vn-payment` — một Monorepo dạng Open Source publish lên NPM.

---

## 1. CI/CD là gì?

**CI/CD** = _Continuous Integration_ + _Continuous Delivery/Deployment_

Hãy hình dung CI/CD như một **băng chuyền lắp ráp tự động trong nhà máy**:

|Giai đoạn|Vai trò|Ví dụ hành động|
|---|---|---|
|**CI** — Tích hợp liên tục|Bộ phận Kiểm định (QC)|Chạy test, lint, typecheck khi push code|
|**CD** — Phân phối liên tục|Bộ phận Vận chuyển|Build → Publish NPM / Deploy lên server|

### Vòng đời tổng quát

```
Developer push code
      │
      ▼
┌─────────────────────────────┐
│          CI Pipeline        │
│  lint → typecheck → test    │
└──────────┬──────────────────┘
           │ ✅ pass
           ▼
     Merge vào main
           │
    git tag v1.0.0
           │
           ▼
┌─────────────────────────────┐
│          CD Pipeline        │
│   build → publish to NPM   │
└─────────────────────────────┘
```

### Lợi ích cốt lõi

- **Loại bỏ thao tác thủ công lặp lại** — không cần nhớ thứ tự lệnh
- **Phát hiện lỗi sớm** — lỗi bị chặn trước khi merge vào nhánh chính
- **Nhất quán** — mọi lần release đều chạy đúng một quy trình
- **Audit trail** — lịch sử build/deploy rõ ràng trên GitHub Actions

---

## 2. Tại sao CI/CD đặc biệt quan trọng với `vn-payment`?

`vn-payment` có 3 đặc thù khiến CI/CD trở thành yêu cầu **sống còn**:

### 2.1 Thư viện xử lý tiền — không cho phép sai sót

Logic như băm chữ ký HMAC cho MoMo, ZaloPay nếu sai → giao dịch thất bại hoặc bị tấn công. CI bắt buộc toàn bộ unit test phải xanh trước khi bất kỳ dòng code nào được merge.

### 2.2 Monorepo — các package phụ thuộc lẫn nhau

```
packages/
├── core/          ← nền tảng chung
├── momo/          ← phụ thuộc core
└── zalopay/       ← phụ thuộc core
```

Thay đổi `core` có thể âm thầm làm hỏng `momo` hoặc `zalopay`. CI đảm bảo toàn bộ dependency chain luôn được kiểm tra cùng nhau.

### 2.3 Open Source — code đến từ người lạ

Bất kỳ ai cũng có thể gửi Pull Request. Không có CI, bạn phải tự tay review từng dòng logic HMAC. Với CI, máy làm việc đó thay bạn 24/7.

---

## 3. Cấu trúc file GitHub Actions

GitHub Actions lưu workflow tại `.github/workflows/`. Với `vn-payment`, có 2 file chính:

```
.github/
└── workflows/
    ├── ci.yml   ← chạy khi push / PR vào main
    └── cd.yml   ← chạy khi có git tag dạng v*.*.*
```

### 3.1 Anatomy của một workflow file

```yaml
name: CI                         # Tên hiển thị trên GitHub

on:                              # Điều kiện kích hoạt
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:                            # Các công việc cần chạy
  test:                          # Tên job
    runs-on: ubuntu-latest       # Máy ảo sử dụng

    steps:                       # Từng bước trong job
      - uses: actions/checkout@v4           # Bước 1: tải code về
      - uses: pnpm/action-setup@v4          # Bước 2: cài pnpm
        with:
          version: 11
      - uses: actions/setup-node@v4         # Bước 3: cài Node
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile # Bước 4: cài dependencies
      - run: pnpm build                     # Bước 5: build
      - run: pnpm typecheck                 # Bước 6: kiểm tra types
      - run: pnpm test                      # Bước 7: chạy test
```

> [!tip] Thứ tự quan trọng trong Monorepo **Luôn build trước, typecheck/test sau.** Vì các package con (`momo`, `zalopay`) cần file `dist/` của `core` đã được build để TypeScript có thể resolve type.

### 3.2 File `cd.yml` — workflow publish NPM

```yaml
name: CD — Publish to NPM

on:
  push:
    tags:
      - 'v*.*.*'           # Chỉ chạy khi push tag dạng v1.2.3

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: 11
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: 'https://registry.npmjs.org'
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      - run: pnpm publish -r --no-git-checks   # publish toàn bộ packages
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}  # token từ GitHub Secrets
```

---

## 4. Khái niệm quan trọng cần nắm

### 4.1 GitHub Secrets — kho lưu mật khẩu an toàn

Không bao giờ hardcode token/password vào file YAML. Thay vào đó:

1. Vào **GitHub repo → Settings → Secrets and variables → Actions**
2. Tạo secret tên `NPM_TOKEN`
3. Dùng trong YAML bằng cú pháp `${{ secrets.NPM_TOKEN }}`

Giá trị secret bị mã hóa, không ai đọc được kể cả admin repo.

### 4.2 Frozen Lockfile — đảm bảo môi trường nhất quán

```bash
pnpm install --frozen-lockfile
```

Flag này bắt máy ảo phải cài đúng version được ghi trong `pnpm-lock.yaml`, không tự động upgrade. Đảm bảo môi trường CI = môi trường local của developer.

### 4.3 Git Tag — công tắc kích hoạt CD

```bash
git tag v1.0.0          # tạo tag local
git push origin v1.0.0  # đẩy tag lên → kích hoạt CD pipeline
```

> [!warning] Tag phân biệt HOA/thường `v1.0.0` ≠ `V1.0.0`. Workflow cấu hình `'v*'` sẽ không nhận tag `V1.0.0`.

Nếu tag sai, xóa và tạo lại:

```bash
git tag -d V1.0.0              # xóa local
git push origin :refs/tags/V1.0.0  # xóa remote
git tag v1.0.0                 # tạo lại đúng
git push origin v1.0.0
```

### 4.4 Matrix Strategy — test nhiều môi trường cùng lúc

```yaml
strategy:
  matrix:
    node-version: [18, 20, 22]
    os: [ubuntu-latest, windows-latest]
```

Cách này tự động tạo ra 6 job song song để test trên mọi tổ hợp. Hữu ích khi build thư viện cần đảm bảo tương thích đa môi trường.

### 4.5 Job Dependencies — kiểm soát thứ tự chạy

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    # ...

  publish:
    needs: test          # publish CHỈ chạy khi test pass
    runs-on: ubuntu-latest
    # ...
```

---

## 5. Các lỗi thực tế đã gặp và bài học

### 🚨 Lỗi 1 — Sai thứ tự: Typecheck trước Build

**Triệu chứng:** TypeScript báo lỗi `Cannot find module '@vn-payment/core'`

**Nguyên nhân:** `momo` và `zalopay` import từ `core`, nhưng `core` chưa được build ra `dist/` nên TS không tìm thấy.

**Fix:**

```yaml
# ❌ Sai
- run: pnpm typecheck
- run: pnpm build

# ✅ Đúng
- run: pnpm build
- run: pnpm typecheck
- run: pnpm test
```

**Bài học:** Trong Monorepo, luôn build dependency trước khi chạy bất kỳ bước nào phụ thuộc vào output của nó.

---

### 🚨 Lỗi 2 — Xung đột phiên bản pnpm lockfile

**Triệu chứng:** `ERR_PNPM_NO_LOCKFILE` hoặc `ERR_PNPM_LOCKFILE_MISMATCH`

**Nguyên nhân:** Local dùng pnpm v11, CI mặc định cài pnpm v8. Format của `pnpm-lock.yaml` v11 không tương thích với v8.

**Fix:**

```yaml
- uses: pnpm/action-setup@v4
  with:
    version: 11       # ← ép đúng version
```

**Bài học:** Luôn pin version của package manager trong CI. Commit `pnpm-lock.yaml` vào repo.

---

### 🚨 Lỗi 3 — pnpm v11 chặn postinstall script của esbuild

**Triệu chứng:** `esbuild` cài đặt thất bại, build không chạy được

**Nguyên nhân:** pnpm v10+ mặc định chặn `postinstall` script (cơ chế bảo mật chống mã độc). `esbuild` cần postinstall để tải binary native.

**Fix** — thêm vào `pnpm-workspace.yaml`:

```yaml
allowBuilds:
  - esbuild
```

> [!info] Tại sao không dùng `package.json`? Từ pnpm v10+, cấu hình `allowedDeprecatedVersions` và `allowBuilds` được khuyến nghị đặt trong `pnpm-workspace.yaml` thay vì `.npmrc` hay `package.json` để rõ ràng hơn về scope.

**Bài học:** Upgrade package manager có thể kích hoạt các security policy mới. Khi CI báo lỗi install, kiểm tra thay đổi security giữa các version.

---

### 🚨 Lỗi 4 — Tag HOA/thường không khớp pattern

**Triệu chứng:** Đẩy tag lên GitHub nhưng CD pipeline không kích hoạt

**Nguyên nhân:**

```yaml
on:
  push:
    tags:
      - 'v*'   # chỉ match 'v' thường
```

Nhưng đã tạo tag `V0.2.1` (V hoa).

**Fix:**

```bash
git tag -d V0.2.1
git push origin :refs/tags/V0.2.1
git tag v0.2.1
git push origin v0.2.1
```

**Bài học:** Convention tag NPM packages luôn dùng chữ thường `vX.Y.Z` theo [Semantic Versioning](https://semver.org/).

---

### 🚨 Lỗi 5 — NPM chặn publish vì 2FA

**Triệu chứng:** `ERR_PNPM_OTP_NON_INTERACTIVE` — yêu cầu nhập mã OTP nhưng không có terminal

**Nguyên nhân:** Tài khoản NPM bật 2FA. Máy ảo CI không thể mở điện thoại để lấy OTP.

**Fix:**

1. Đăng nhập NPM → **Access Tokens → Generate New Token**
2. Chọn loại **Automation** (tự động bypass OTP trong CI)
3. Thêm vào GitHub Secrets với tên `NPM_TOKEN`
4. Dùng trong workflow:

```yaml
env:
  NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

> [!warning] Phân biệt loại NPM Token
> 
> |Loại|Dùng cho|Có bypass 2FA?|
> |---|---|---|
> |Classic (Read-only)|Đọc package private|❌|
> |Classic (Automation)|CI/CD publish|✅|
> |Granular|Giới hạn theo package/scope|Tuỳ cấu hình|

**Bài học:** Luôn dùng **Automation Token** cho CI/CD, không dùng token thường vì sẽ bị chặn bởi 2FA.

---

## 6. Checklist trước khi setup CI/CD mới

```
□ File .github/workflows/ đã được tạo
□ Version pnpm/node được pin cụ thể trong workflow
□ Thứ tự bước đúng: install → build → typecheck → test
□ NPM_TOKEN đã được tạo loại "Automation" và thêm vào GitHub Secrets
□ pnpm-workspace.yaml có cấu hình allowBuilds nếu cần
□ Pattern tag trong cd.yml khớp với convention đặt tag (v* hay V*)
□ Job publish có `needs: test` để không publish khi test fail
```

---

## 7. Roadmap mở rộng — Những thứ có thể làm tiếp

|Tính năng|Mô tả|Độ khó|
|---|---|---|
|**Code coverage report**|Tự động comment % coverage lên PR|⭐⭐|
|**Changesets**|Tự động bump version và tạo changelog|⭐⭐⭐|
|**Release drafter**|Tự động tạo Release Notes trên GitHub|⭐⭐|
|**Dependabot**|Tự động tạo PR khi dependency có version mới|⭐|
|**Branch protection**|Bắt buộc CI pass mới được merge vào main|⭐|
|**Docker publish**|Build và push Docker image lên GHCR|⭐⭐⭐|

---

## 8. Tài nguyên tham khảo

- [GitHub Actions Docs](https://docs.github.com/en/actions)
- [pnpm Workspace Docs](https://pnpm.io/workspaces)
- [Semantic Versioning](https://semver.org/)
- [NPM Automation Tokens](https://docs.npmjs.com/creating-and-viewing-access-tokens)

---

> [!success] Kết luận Sau 5 lỗi thực chiến, bạn đã nắm được bản chất của CI/CD không phải là "công nghệ phức tạp" — mà là **một hệ thống quy trình được tự động hóa**. Kiến thức này áp dụng được cho bất kỳ stack nào: Node.js, Python, Go, hay bất kỳ ngôn ngữ nào khác.

_Tags: #devops #cicd #github-actions #npm #monorepo #pnpm #vn-payment_