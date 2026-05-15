# React (Base)

> **Lộ trình học:** JSX → Component → Props → Hooks cơ bản → Router → Next.js cơ bản File nâng cao (HOC, Patterns, Zustand...): xem [[React Advanced]]

---

## React là gì?

- Thư viện JavaScript do **Meta** phát triển để xây dựng UI
- Hoạt động theo mô hình **Component-based** — chia giao diện thành các mảnh nhỏ, tái sử dụng được
- Dùng **Virtual DOM** — chỉ cập nhật phần thay đổi thay vì render lại toàn trang → nhanh hơn
- Hiện tại dùng **Function Component + Hooks** (không còn dùng Class Component)

---

## JSX

> JSX = JavaScript + XML. Viết giao diện kiểu HTML ngay trong file `.tsx`

```tsx
// Biểu thức JS dùng { }
const name = "Phúc";
const greeting = <p>Xin chào, {name}!</p>;

// Multi-line — bọc trong 1 tag cha hoặc Fragment <>
const ui = (
  <div>
    <h1>Tiêu đề</h1>
    <p>Nội dung</p>
  </div>
);
```

### JSX khác HTML ở chỗ nào?

|HTML|JSX|Lý do|
|---|---|---|
|`class`|`className`|`class` là từ khóa của JS|
|`for`|`htmlFor`|`for` là từ khóa của JS|
|`onclick`|`onClick`|JS dùng camelCase|
|`style="color:red"`|`style={{ color: "red" }}`|Style là object trong JSX|
|`<br>`|`<br />`|Phải tự đóng tag|
|`<!-- comment -->`|`{/* comment */}`|Comment trong JSX|

### Fragment — tránh thêm div thừa

```tsx
// ❌ Thêm div không cần thiết vào DOM
return (
  <div>
    <h1>Tiêu đề</h1>
    <p>Nội dung</p>
  </div>
);

// ✅ Fragment — không tạo element thật trong DOM
return (
  <>
    <h1>Tiêu đề</h1>
    <p>Nội dung</p>
  </>
);
```

---

## Component

> Component = hàm trả về JSX. Tên **phải bắt đầu bằng chữ hoa**.

```tsx
// Cách viết cơ bản
const Welcome = () => {
  return <h1>Xin chào!</h1>;
};

// Rút gọn khi chỉ return 1 thứ
const Welcome = () => <h1>Xin chào!</h1>;

// Dùng trong JSX như HTML tag
const App = () => (
  <div>
    <Welcome />
    <Welcome />
  </div>
);
```

### Render có điều kiện

```tsx
const UserStatus = ({ isLoggedIn }: { isLoggedIn: boolean }) => {
  // Cách 1: if/else — rõ ràng, dùng khi logic phức tạp
  if (!isLoggedIn) return <p>Vui lòng đăng nhập</p>;
  return <p>Chào mừng bạn!</p>;
};

const Banner = ({ isLoggedIn }: { isLoggedIn: boolean }) => (
  <div>
    {/* Cách 2: Ternary — dùng trong JSX */}
    {isLoggedIn ? <p>Đã đăng nhập</p> : <p>Chưa đăng nhập</p>}

    {/* Cách 3: && — chỉ hiện khi điều kiện đúng */}
    {isLoggedIn && <button>Đăng xuất</button>}
  </div>
);
```

### Render danh sách

```tsx
const fruits = ["Táo", "Xoài", "Ổi"];

const FruitList = () => (
  <ul>
    {fruits.map((fruit, index) => (
      <li key={index}>{fruit}</li>
      // key giúp React theo dõi item — nên dùng id thay index nếu có
    ))}
  </ul>
);

// Với mảng object — luôn dùng id làm key
const users = [{ id: 1, name: "Phúc" }, { id: 2, name: "Hiền" }];

const UserList = () => (
  <ul>
    {users.map((user) => (
      <li key={user.id}>{user.name}</li>
    ))}
  </ul>
);
```

---

## Props

> Props = dữ liệu truyền từ **component cha → component con**. Con **không được sửa** props nhận được.

```tsx
// Định nghĩa kiểu props bằng interface
interface CardProps {
  title: string;
  description: string;
  count?: number;       // optional — có thể không truyền
}

// Nhận props bằng destructuring
const Card = ({ title, description, count = 0 }: CardProps) => (
  <div className="card">
    <h2>{title}</h2>
    <p>{description}</p>
    <span>{count}</span>
  </div>
);

// Truyền props từ cha
const App = () => (
  <Card
    title="Thông báo"
    description="Có tin nhắn mới"
    count={5}
  />
);
```

### Props với hàm (callback)

```tsx
interface ButtonProps {
  label: string;
  onClick: () => void;   // truyền hàm từ cha xuống con
}

const Button = ({ label, onClick }: ButtonProps) => (
  <button onClick={onClick}>{label}</button>
);

const App = () => {
  const handleClick = () => console.log("Đã bấm!");

  return <Button label="Bấm tôi" onClick={handleClick} />;
};
```

### `children` prop — nội dung bên trong tag

```tsx
interface PanelProps {
  title: string;
  children: React.ReactNode; // nhận bất kỳ JSX nào
}

const Panel = ({ title, children }: PanelProps) => (
  <div className="panel">
    <h3>{title}</h3>
    <div>{children}</div>
  </div>
);

// Dùng như HTML tag thông thường
const App = () => (
  <Panel title="Thông tin người dùng">
    <p>Tên: Phúc</p>
    <p>Tuổi: 20</p>
    <button>Chỉnh sửa</button>
  </Panel>
);
```

---

## Hooks

> Hooks = hàm bắt đầu bằng `use`, cho phép dùng state và các tính năng React trong Function Component.

> **2 quy tắc bắt buộc:**
> 
> 1. Chỉ gọi ở **top level** — không gọi trong `if`, `for`, hàm lồng nhau
> 2. Chỉ gọi trong **Function Component** hoặc **Custom Hook**

---

### `useState` — lưu và cập nhật dữ liệu

```tsx
import { useState } from "react";

const Counter = () => {
  //        giá trị  hàm cập nhật   giá trị ban đầu
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Đếm: {count}</p>
      <button onClick={() => setCount(count + 1)}>Tăng</button>
      <button onClick={() => setCount(count - 1)}>Giảm</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
};
```

### State với string — ví dụ input

```tsx
const NameInput = () => {
  const [name, setName] = useState("");

  return (
    <div>
      <input
        value={name}
        onChange={(e) => setName(e.target.value)}
        placeholder="Nhập tên..."
      />
      <p>Xin chào, {name || "người lạ"}!</p>
    </div>
  );
};
```

### State với object

```tsx
interface Form {
  name: string;
  email: string;
}

const ProfileForm = () => {
  const [form, setForm] = useState<Form>({ name: "", email: "" });

  // Phải spread object cũ khi cập nhật — tránh mất field khác
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const { name, value } = e.target;
    setForm((prev) => ({ ...prev, [name]: value }));
  };

  return (
    <div>
      <input name="name"  value={form.name}  onChange={handleChange} placeholder="Tên" />
      <input name="email" value={form.email} onChange={handleChange} placeholder="Email" />
      <p>Preview: {form.name} — {form.email}</p>
    </div>
  );
};
```

> **Lưu ý:** Khi state mới phụ thuộc state cũ, dùng callback:
> 
> ```tsx
> // ❌ Có thể bị stale (giá trị cũ)
> setCount(count + 1);
> 
> // ✅ Luôn nhận giá trị mới nhất
> setCount((prev) => prev + 1);
> ```

---

### `useEffect` — chạy code khi component render / dữ liệu thay đổi

> Dùng để: **gọi API**, lắng nghe sự kiện, set timer, thao tác DOM...

```tsx
import { useEffect } from "react";

// Chạy sau MỖI lần render
useEffect(() => {
  console.log("Component vừa render");
});

// Chạy 1 lần khi component xuất hiện (mount)
useEffect(() => {
  console.log("Component đã mount");
}, []); // dependency array rỗng

// Chạy khi 'userId' thay đổi
useEffect(() => {
  console.log("userId thay đổi:", userId);
}, [userId]);
```

### Fetch API với useEffect

```tsx
interface Post { id: number; title: string; }

const PostList = () => {
  const [posts,   setPosts]   = useState<Post[]>([]);
  const [loading, setLoading] = useState(true);
  const [error,   setError]   = useState("");

  useEffect(() => {
    fetch("https://jsonplaceholder.typicode.com/posts?_limit=5")
      .then((res) => res.json())
      .then((data) => {
        setPosts(data);
        setLoading(false);
      })
      .catch(() => {
        setError("Không tải được dữ liệu");
        setLoading(false);
      });
  }, []); // [] → chỉ gọi 1 lần khi mount

  if (loading) return <p>Đang tải...</p>;
  if (error)   return <p style={{ color: "red" }}>{error}</p>;

  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
};
```

### Cleanup — dọn dẹp khi component unmount

```tsx
useEffect(() => {
  const timer = setInterval(() => {
    console.log("tick mỗi giây");
  }, 1000);

  // Hàm cleanup — chạy khi component unmount
  return () => clearInterval(timer);
}, []);
```

---

### `useRef` — tham chiếu DOM element

```tsx
import { useRef } from "react";

const InputFocus = () => {
  const inputRef = useRef<HTMLInputElement>(null);

  const handleFocus = () => {
    inputRef.current?.focus(); // truy cập DOM trực tiếp
  };

  return (
    <div>
      <input ref={inputRef} placeholder="Ấn nút để focus" />
      <button onClick={handleFocus}>Focus vào input</button>
    </div>
  );
};
```

---

### `useContext` — chia sẻ dữ liệu toàn cục (không cần truyền props nhiều tầng)

```tsx
import { createContext, useContext, useState } from "react";

// 1. Tạo Context
interface AuthContextType {
  user: string | null;
  login:  (name: string) => void;
  logout: () => void;
}

const AuthContext = createContext<AuthContextType | null>(null);

// 2. Tạo Provider — bọc bên ngoài các component cần dùng
export const AuthProvider = ({ children }: { children: React.ReactNode }) => {
  const [user, setUser] = useState<string | null>(null);

  const login  = (name: string) => setUser(name);
  const logout = ()             => setUser(null);

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};

// 3. Custom hook để dùng — tránh phải null-check mỗi nơi
export const useAuth = () => {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth phải dùng trong AuthProvider");
  return ctx;
};

// 4. Dùng trong component bất kỳ
const Header = () => {
  const { user, logout } = useAuth();
  return (
    <header>
      {user ? (
        <>
          <span>Xin chào, {user}!</span>
          <button onClick={logout}>Đăng xuất</button>
        </>
      ) : (
        <span>Chưa đăng nhập</span>
      )}
    </header>
  );
};

// 5. Bọc App trong Provider
const App = () => (
  <AuthProvider>
    <Header />
    {/* các component khác */}
  </AuthProvider>
);
```

---

## Forms

### Controlled Form — state kiểm soát input (phổ biến nhất)

```tsx
const LoginForm = () => {
  const [email,    setEmail]    = useState("");
  const [password, setPassword] = useState("");
  const [error,    setError]    = useState("");

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault(); // ngăn reload trang

    if (!email) {
      setError("Email là bắt buộc");
      return;
    }

    console.log("Đăng nhập với:", { email, password });
  };

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
        />
      </div>

      <div>
        <label htmlFor="password">Mật khẩu</label>
        <input
          id="password"
          type="password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
        />
      </div>

      {error && <p style={{ color: "red" }}>{error}</p>}

      <button type="submit">Đăng nhập</button>
    </form>
  );
};
```

---

## Styling

### CSS thông thường

```tsx
// Inline style — dùng khi style phụ thuộc vào state/props
const Box = ({ color }: { color: string }) => (
  <div style={{ backgroundColor: color, padding: 16, borderRadius: 8 }}>
    Nội dung
  </div>
);

// className — nối chuỗi có điều kiện
const Button = ({ active }: { active: boolean }) => (
  <button className={`btn ${active ? "btn-active" : ""}`}>
    Click
  </button>
);
```

### CSS Module — CSS riêng cho từng component

```css
/* Button.module.css */
.btn { background: blue; color: white; padding: 8px 16px; }
.btn:hover { background: darkblue; }
.active { background: green; }
```

```tsx
import styles from "./Button.module.css";

const Button = ({ active }: { active: boolean }) => (
  <button className={`${styles.btn} ${active ? styles.active : ""}`}>
    Click
  </button>
);
```

### Tailwind CSS (hay dùng với Next.js)

```tsx
const Card = ({ title, isActive }: { title: string; isActive: boolean }) => (
  <div className={`rounded-xl p-4 shadow ${isActive ? "bg-blue-500 text-white" : "bg-white"}`}>
    <h2 className="text-xl font-bold">{title}</h2>
  </div>
);
```

---

## React Router v6

> Dùng cho **Single Page Application (SPA)** — điều hướng trang không reload.

```bash
npm install react-router-dom
```

### Cấu hình cơ bản

```tsx
import { BrowserRouter, Routes, Route, Link } from "react-router-dom";

// Pages
const Home    = () => <h1>Trang chủ</h1>;
const About   = () => <h1>Giới thiệu</h1>;
const NotFound = () => <h1>404 — Không tìm thấy</h1>;

const App = () => (
  <BrowserRouter>
    {/* Navigation */}
    <nav>
      <Link to="/">Trang chủ</Link>
      <Link to="/about">Giới thiệu</Link>
    </nav>

    {/* Routes */}
    <Routes>
      <Route path="/"      element={<Home />} />
      <Route path="/about" element={<About />} />
      <Route path="*"      element={<NotFound />} /> {/* 404 */}
    </Routes>
  </BrowserRouter>
);
```

### `useParams` — đọc tham số trên URL

```tsx
import { useParams } from "react-router-dom";

// Route: <Route path="/users/:id" element={<UserDetail />} />
const UserDetail = () => {
  const { id } = useParams<{ id: string }>();
  return <p>Đang xem user có ID: {id}</p>;
};
```

### `useNavigate` — chuyển trang bằng code

```tsx
import { useNavigate } from "react-router-dom";

const LoginPage = () => {
  const navigate = useNavigate();

  const handleLogin = () => {
    // ... xử lý login ...
    navigate("/dashboard");    // chuyển đến /dashboard
    // navigate(-1);           // quay lại trang trước
  };

  return <button onClick={handleLogin}>Đăng nhập</button>;
};
```

### Layout dùng chung với `Outlet`

```tsx
import { Outlet, Link } from "react-router-dom";

// Layout component
const MainLayout = () => (
  <div>
    <header>
      <Link to="/">Home</Link>
      <Link to="/about">About</Link>
    </header>

    <main>
      <Outlet /> {/* Route con sẽ render ở đây */}
    </main>

    <footer>Footer</footer>
  </div>
);

// Cấu hình nested route
const App = () => (
  <BrowserRouter>
    <Routes>
      <Route element={<MainLayout />}>        {/* Layout bọc ngoài */}
        <Route path="/"      element={<Home />} />
        <Route path="/about" element={<About />} />
      </Route>
    </Routes>
  </BrowserRouter>
);
```

---

## Next.js Cơ Bản (App Router)

> Next.js = React + routing theo file + render phía server + tối ưu SEO.

### Server Component vs Client Component

```tsx
// ✅ Server Component — mặc định trong Next.js App Router
// Chạy trên server: có thể fetch data thẳng, không dùng state/event
// app/page.tsx
const HomePage = async () => {
  const res   = await fetch("https://jsonplaceholder.typicode.com/posts?_limit=3");
  const posts = await res.json();

  return (
    <ul>
      {posts.map((p: { id: number; title: string }) => (
        <li key={p.id}>{p.title}</li>
      ))}
    </ul>
  );
};

export default HomePage;
```

```tsx
// ✅ Client Component — thêm "use client" ở đầu file
// Dùng khi cần: useState, useEffect, onClick, browser API...
"use client";

import { useState } from "react";

const LikeButton = () => {
  const [liked, setLiked] = useState(false);
  return (
    <button onClick={() => setLiked(!liked)}>
      {liked ? "❤️ Đã thích" : "🤍 Thích"}
    </button>
  );
};

export default LikeButton;
```

> **Nguyên tắc:** Mặc định dùng Server Component. Chỉ thêm `"use client"` khi cần tương tác.

### Cấu trúc thư mục (App Router)

```
app/
├── layout.tsx          → Layout gốc (bọc tất cả trang)
├── page.tsx            → Trang chủ: /
├── about/
│   └── page.tsx        → /about
├── blog/
│   ├── page.tsx        → /blog
│   └── [slug]/
│       └── page.tsx    → /blog/bai-viet-1 (dynamic route)
└── dashboard/
    ├── layout.tsx      → Layout riêng cho /dashboard/*
    └── page.tsx        → /dashboard
```

### `layout.tsx` — layout dùng chung

```tsx
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="vi">
      <body>
        <header>Header dùng chung</header>
        <main>{children}</main>
        <footer>Footer dùng chung</footer>
      </body>
    </html>
  );
}
```

### Dynamic Route — `[slug]`

```tsx
// app/blog/[slug]/page.tsx
interface Props {
  params: { slug: string };
}

const BlogPost = async ({ params }: Props) => {
  // params.slug = phần URL động, VD: "bai-viet-ve-react"
  return <h1>Bài viết: {params.slug}</h1>;
};

export default BlogPost;
```

### `loading.tsx` — hiện khi trang đang tải

```tsx
// app/dashboard/loading.tsx
// Next.js tự động hiển thị file này khi route đang load
export default function Loading() {
  return <p>Đang tải trang...</p>;
}
```

### `not-found.tsx` — trang 404

```tsx
// app/not-found.tsx
export default function NotFound() {
  return (
    <div>
      <h2>404 — Không tìm thấy trang</h2>
      <a href="/">Về trang chủ</a>
    </div>
  );
}
```

### Metadata — SEO

```tsx
// app/about/page.tsx
import type { Metadata } from "next";

// Khai báo metadata cho từng trang
export const metadata: Metadata = {
  title:       "Giới thiệu | My App",
  description: "Trang giới thiệu về chúng tôi",
};

export default function AboutPage() {
  return <h1>Giới thiệu</h1>;
}
```

### Link — điều hướng trong Next.js

```tsx
import Link from "next/link";
import { useRouter } from "next/navigation"; // điều hướng bằng code

// Dùng <Link> thay <a> để điều hướng không reload trang
const Nav = () => (
  <nav>
    <Link href="/">Trang chủ</Link>
    <Link href="/about">Giới thiệu</Link>
    <Link href="/blog/bai-viet-1">Xem bài viết</Link>
  </nav>
);

// Điều hướng bằng code (trong Client Component)
"use client";
const BackButton = () => {
  const router = useRouter();
  return <button onClick={() => router.push("/")}>Về trang chủ</button>;
};
```

---

## Tóm tắt — khi nào dùng gì?

|Tình huống|Dùng|
|---|---|
|Lưu dữ liệu thay đổi trong component|`useState`|
|Gọi API, set timer, lắng nghe event|`useEffect`|
|Truy cập DOM trực tiếp|`useRef`|
|Chia sẻ data nhiều component (tránh prop drilling)|`useContext`|
|Truyền data từ cha xuống con|Props|
|Render danh sách|`.map()` + `key`|
|Điều hướng trang (SPA)|React Router|
|Render server + SEO + routing theo file|Next.js|
|Component cần state/event trong Next.js|`"use client"`|