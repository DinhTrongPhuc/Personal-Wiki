

### 1. Không lưu mật khẩu dạng plain text

Luôn **hash password** bằng thuật toán chuyên dụng như **Argon2, bcrypt hoặc scrypt**.

```
❌ password: "123456"
✅ password: "$2b$10$..."
```

Không nên dùng MD5/SHA-1 để hash password.

---

### 2. Chống SQL Injection

Không nối chuỗi trực tiếp vào SQL query.

```
// ❌ Không an toàn
`SELECT * FROM users WHERE email = '${email}'`

// ✅ Dùng parameterized query / ORM
SELECT * FROM users WHERE email = ?
```

Với Node.js có thể sử dụng ORM/query builder hoặc prepared statements.

---

### 3. Chống XSS (Cross-Site Scripting)

Không render dữ liệu người dùng nhập vào HTML một cách trực tiếp.

```
<!-- ❌ -->
<div>${userInput}</div>
```

Cần **escape/sanitize output** và áp dụng CSP khi phù hợp.

---

### 4. Chống CSRF

Đặc biệt quan trọng khi authentication sử dụng **cookie/session**.

Có thể sử dụng:

- CSRF token
- `SameSite` cookie
- `Secure`
- `HttpOnly`

---

### 5. Không lưu Secret/API Key trong source code

```
// ❌
const API_KEY = "sk-xxxxxxxx";

// ✅
const API_KEY = process.env.API_KEY;
```

Không commit `.env` lên Git.

---

### 6. Validate và sanitize input

**Không tin tưởng dữ liệu từ client.**

Kiểm tra:

- Type
- Format
- Length
- Range
- Required fields
- Allowed values

Ví dụ:

```
email → phải đúng format
age → number và nằm trong khoảng hợp lệ
role → chỉ cho phép USER/ADMIN
```

---

### 7. Áp dụng Authentication và Authorization đúng cách

Hai khái niệm khác nhau:

```
Authentication → Bạn là ai?
Authorization  → Bạn được phép làm gì?
```

Ví dụ:

```
GET /users/123
```

Không phải cứ đăng nhập là được xem dữ liệu của user `123`.

---

### 8. Sử dụng HTTPS

Không truyền thông tin nhạy cảm qua HTTP thường.

```
❌ http://example.com/login
✅ https://example.com/login
```

HTTPS bảo vệ dữ liệu trên đường truyền khỏi bị đọc hoặc sửa đổi.

---

### 9. Bảo vệ Cookie

Với cookie chứa session/token nên cân nhắc:

```
HttpOnly
Secure
SameSite=Lax
```

Ví dụ:

```
Set-Cookie: session=abc;
HttpOnly;
Secure;
SameSite=Lax
```

`HttpOnly` giúp JavaScript phía client không đọc được cookie.

---

### 10. Không trả về thông tin nhạy cảm

API không nên trả password hash, secret hoặc thông tin nội bộ không cần thiết.

```
// ❌
{
  "id": 1,
  "email": "a@gmail.com",
  "passwordHash": "...",
  "internalRole": "..."
}
```

Chỉ trả những dữ liệu client thực sự cần.

---

### 11. Rate Limiting

Giới hạn số request để chống:

- Brute-force
- Spam API
- Credential stuffing
- DoS ở mức ứng dụng

Ví dụ:

```
POST /login
→ tối đa 5 lần / phút / IP hoặc account
```

---

### 12. Không để lộ lỗi hệ thống

Không trả stack trace cho client production.

```
// ❌
{
  "error": "MongoServerError ... /home/server/src/..."
}
```

Nên:

```
{
  "message": "Internal server error"
}
```

Chi tiết lỗi được ghi vào **server log**.

---

### 13. Kiểm tra và cập nhật dependencies

Các package có thể tồn tại lỗ hổng bảo mật.

Định kỳ kiểm tra:

```
npm audit
```

Và cập nhật những dependency có vulnerability phù hợp.

---

### 14. Phân quyền theo nguyên tắc Least Privilege

Mỗi user/service chỉ được cấp **quyền tối thiểu cần thiết**.

Ví dụ:

```
Application
   ↓
Database user
   ↓
Chỉ có SELECT/INSERT/UPDATE cần thiết
```

Không nên để application sử dụng database account có toàn quyền nếu không cần.

---

### 15. Log và monitor các hành vi bất thường

Nên ghi nhận các sự kiện quan trọng:

```
Login failed
Login successful
Password changed
Permission denied
Admin action
Suspicious API requests
```

Sau đó sử dụng monitoring/alerting để phát hiện bất thường.

---

