
## Định nghĩa

- **React** là thư viện JavaScript (do Meta phát triển) để xây dựng giao diện người dùng (UI)
- Hoạt động theo mô hình **Component-based** — chia UI thành các thành phần nhỏ, tái sử dụng được
- Dùng **Virtual DOM** — React so sánh DOM ảo với DOM thật, chỉ cập nhật phần thay đổi → hiệu năng cao
- Hiện tại (2024+) React khuyến khích dùng **Function Component** + **Hooks**, không còn dùng Class Component

---

## JSX

> JSX = JavaScript + XML. Cho phép viết HTML-like syntax trong file `.tsx`/`.jsx`.

```tsx
// JSX được compile thành React.createElement(...)
const element = <h1 className="title">Hello World</h1>;

// Biểu thức JS trong JSX — dùng { }
const name = "Phúc";
const greeting = <p>Xin chào, {name}!</p>;

// Multi-line — phải bọc trong 1 element cha hoặc Fragment
const ui = (
  <div>
    <h1>Tiêu đề</h1>
    <p>Nội dung</p>
  </div>
);
```

### JSX vs HTML — khác biệt quan trọng

|HTML|JSX|
|---|---|
|`class`|`className`|
|`for`|`htmlFor`|
|`onclick`|`onClick`|
|`style="color:red"`|`style={{ color: "red" }}`|
|`<br>`|`<br />` (tự đóng)|
|`<!-- comment -->`|`{/* comment */}`|

### Fragment — tránh thêm div thừa

```tsx
import { Fragment } from "react";

// Cách 1: Fragment đầy đủ
const App = () => (
  <Fragment>
    <h1>Tiêu đề</h1>
    <p>Nội dung</p>
  </Fragment>
);

// Cách 2: Shorthand (phổ biến hơn)
const App = () => (
  <>
    <h1>Tiêu đề</h1>
    <p>Nội dung</p>
  </>
);
```

---

## Component

> Component là **hàm trả về JSX**. Tên component phải bắt đầu bằng **chữ hoa**.

### Function Component cơ bản

```tsx
// Component đơn giản
function Welcome() {
  return <h1>Xin chào!</h1>;
}

// Arrow function (phổ biến hơn)
const Welcome = () => {
  return <h1>Xin chào!</h1>;
};

// Rút gọn khi chỉ return 1 dòng
const Welcome = () => <h1>Xin chào!</h1>;
```

### Render có điều kiện

```tsx
const UserCard = ({ isLoggedIn }: { isLoggedIn: boolean }) => {
  // Cách 1: if/else
  if (!isLoggedIn) return <p>Vui lòng đăng nhập</p>;

  // Cách 2: Ternary operator (phổ biến)
  return (
    <div>
      {isLoggedIn ? <p>Chào mừng!</p> : <p>Vui lòng đăng nhập</p>}
    </div>
  );
};

// Cách 3: && (short-circuit) — render khi điều kiện đúng
const Notification = ({ count }: { count: number }) => (
  <div>
    {count > 0 && <span>Bạn có {count} thông báo</span>}
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
      // key giúp React theo dõi item — nên dùng id thay vì index nếu có
    ))}
  </ul>
);

// Dùng với object — luôn dùng id làm key
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

> Props (properties) là cách truyền dữ liệu từ **component cha → component con**. Props là **read-only** — con không được sửa props nhận được.

### Truyền và nhận Props

```tsx
// Component con — nhận props
const UserCard = ({ name, age }: { name: string; age: number }) => (
  <div>
    <p>Tên: {name}</p>
    <p>Tuổi: {age}</p>
  </div>
);

// Component cha — truyền props
const App = () => (
  <UserCard name="Phúc" age={20} />
);
```

### Định nghĩa Props với Interface/Type

```tsx
// Cách 1: Interface (khuyên dùng cho props)
interface ButtonProps {
  label: string;
  color?: string;       // optional
  onClick: () => void;
  disabled?: boolean;
}

const Button = ({ label, color = "blue", onClick, disabled = false }: ButtonProps) => (
  <button
    onClick={onClick}
    disabled={disabled}
    style={{ backgroundColor: color }}
  >
    {label}
  </button>
);

// Cách 2: Type
type CardProps = {
  title: string;
  children: React.ReactNode; // nhận bất kỳ JSX nào
};

const Card = ({ title, children }: CardProps) => (
  <div className="card">
    <h2>{title}</h2>
    {children}
  </div>
);
```

### `children` prop

```tsx
interface PanelProps {
  title: string;
  children: React.ReactNode;
}

const Panel = ({ title, children }: PanelProps) => (
  <div className="panel">
    <h3>{title}</h3>
    <div className="panel-body">{children}</div>
  </div>
);

// Sử dụng
const App = () => (
  <Panel title="Thông tin">
    <p>Đây là nội dung bên trong Panel</p>
    <button>Bấm vào đây</button>
  </Panel>
);
```

### Props spreading & Rest props

```tsx
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
}

// ...rest nhận tất cả props còn lại (placeholder, type, onChange,...)
const Input = ({ label, ...rest }: InputProps) => (
  <div>
    <label>{label}</label>
    <input {...rest} />
  </div>
);

// Dùng
<Input label="Email" type="email" placeholder="abc@gmail.com" />
```

---

## Hooks

> Hooks là các hàm đặc biệt bắt đầu bằng `use`, cho phép dùng state và lifecycle trong Function Component.
> 
> **Quy tắc Hooks:**
> 
> - Chỉ gọi ở **top level** của component (không gọi trong if/loop/nested function)
> - Chỉ gọi trong **Function Component** hoặc **Custom Hook**

---

### `useState` — quản lý state

```tsx
import { useState } from "react";

const Counter = () => {
  const [count, setCount] = useState<number>(0); // [giá trị, hàm cập nhật]

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

### State với Object

```tsx
interface UserForm {
  name: string;
  email: string;
}

const ProfileForm = () => {
  const [form, setForm] = useState<UserForm>({ name: "", email: "" });

  // Cập nhật 1 field — phải spread object cũ, tránh mất field khác
  const handleChange = (field: keyof UserForm, value: string) => {
    setForm((prev) => ({ ...prev, [field]: value }));
  };

  return (
    <div>
      <input
        value={form.name}
        onChange={(e) => handleChange("name", e.target.value)}
        placeholder="Tên"
      />
      <input
        value={form.email}
        onChange={(e) => handleChange("email", e.target.value)}
        placeholder="Email"
      />
    </div>
  );
};
```

### Functional update — dùng khi state mới phụ thuộc state cũ

```tsx
// ❌ Có thể bị stale closure
setCount(count + 1);

// ✅ Luôn nhận giá trị mới nhất
setCount((prev) => prev + 1);
```

---

### `useEffect` — side effects & lifecycle

> Dùng để thực hiện side effects: fetch API, subscribe event, set timer, thao tác DOM...

```tsx
import { useEffect, useState } from "react";

useEffect(() => {
  // Code chạy sau mỗi render
});

useEffect(() => {
  // Code chỉ chạy 1 lần khi component mount
}, []); // dependency array rỗng

useEffect(() => {
  // Code chạy khi 'userId' thay đổi
}, [userId]);
```

### Cleanup function

```tsx
useEffect(() => {
  const timer = setInterval(() => {
    console.log("tick");
  }, 1000);

  // Cleanup — chạy trước khi effect chạy lại hoặc component unmount
  return () => clearInterval(timer);
}, []);
```

### Fetch data với useEffect

```tsx
interface Post { id: number; title: string; }

const PostList = () => {
  const [posts, setPosts]     = useState<Post[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError]     = useState<string | null>(null);

  useEffect(() => {
    let cancelled = false; // tránh setState sau khi unmount

    const fetchPosts = async () => {
      try {
        const res  = await fetch("https://jsonplaceholder.typicode.com/posts");
        const data = await res.json();
        if (!cancelled) setPosts(data);
      } catch {
        if (!cancelled) setError("Lỗi khi tải dữ liệu");
      } finally {
        if (!cancelled) setLoading(false);
      }
    };

    fetchPosts();
    return () => { cancelled = true; };
  }, []);

  if (loading) return <p>Đang tải...</p>;
  if (error)   return <p>{error}</p>;

  return (
    <ul>
      {posts.map((p) => <li key={p.id}>{p.title}</li>)}
    </ul>
  );
};
```

---

### `useRef` — tham chiếu DOM & giá trị không re-render

```tsx
import { useRef } from "react";

// Cách 1: Tham chiếu DOM element
const InputFocus = () => {
  const inputRef = useRef<HTMLInputElement>(null);

  const focusInput = () => {
    inputRef.current?.focus();
  };

  return (
    <div>
      <input ref={inputRef} placeholder="Click nút để focus" />
      <button onClick={focusInput}>Focus</button>
    </div>
  );
};

// Cách 2: Lưu giá trị giữa các render (không trigger re-render)
const Timer = () => {
  const [count, setCount]  = useState(0);
  const intervalRef        = useRef<ReturnType<typeof setInterval> | null>(null);

  const start = () => {
    intervalRef.current = setInterval(() => setCount((c) => c + 1), 1000);
  };

  const stop = () => {
    if (intervalRef.current) clearInterval(intervalRef.current);
  };

  return (
    <div>
      <p>{count}s</p>
      <button onClick={start}>Bắt đầu</button>
      <button onClick={stop}>Dừng</button>
    </div>
  );
};
```

---

### `useContext` — chia sẻ state toàn cục

> Giải quyết **prop drilling** — truyền props qua nhiều tầng component.

```tsx
import { createContext, useContext, useState } from "react";

// 1. Tạo Context
interface ThemeContextType {
  theme: "light" | "dark";
  toggle: () => void;
}

const ThemeContext = createContext<ThemeContextType | null>(null);

// 2. Tạo Provider — bọc các component cần dùng context
export const ThemeProvider = ({ children }: { children: React.ReactNode }) => {
  const [theme, setTheme] = useState<"light" | "dark">("light");

  const toggle = () => setTheme((t) => (t === "light" ? "dark" : "light"));

  return (
    <ThemeContext.Provider value={{ theme, toggle }}>
      {children}
    </ThemeContext.Provider>
  );
};

// 3. Custom hook để dùng context — tránh null check mỗi nơi
export const useTheme = () => {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme phải dùng trong ThemeProvider");
  return ctx;
};

// 4. Dùng trong component bất kỳ (phải nằm trong Provider)
const Header = () => {
  const { theme, toggle } = useTheme();
  return (
    <header style={{ background: theme === "dark" ? "#333" : "#fff" }}>
      <button onClick={toggle}>Đổi theme: {theme}</button>
    </header>
  );
};

// 5. Bọc app trong Provider
const App = () => (
  <ThemeProvider>
    <Header />
  </ThemeProvider>
);
```

---

### `useReducer` — quản lý state phức tạp

> Thay thế `useState` khi state có nhiều action hoặc logic phức tạp. Tương tự Redux.

```tsx
import { useReducer } from "react";

// 1. Định nghĩa kiểu
interface State {
  count: number;
  step: number;
}

type Action =
  | { type: "INCREMENT" }
  | { type: "DECREMENT" }
  | { type: "RESET" }
  | { type: "SET_STEP"; payload: number };

// 2. Reducer — hàm thuần túy, nhận state + action → state mới
function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "INCREMENT": return { ...state, count: state.count + state.step };
    case "DECREMENT": return { ...state, count: state.count - state.step };
    case "RESET":     return { count: 0, step: state.step };
    case "SET_STEP":  return { ...state, step: action.payload };
    default:          return state;
  }
}

// 3. Dùng trong component
const Counter = () => {
  const [state, dispatch] = useReducer(reducer, { count: 0, step: 1 });

  return (
    <div>
      <p>Count: {state.count} | Step: {state.step}</p>
      <button onClick={() => dispatch({ type: "INCREMENT" })}>+</button>
      <button onClick={() => dispatch({ type: "DECREMENT" })}>-</button>
      <button onClick={() => dispatch({ type: "RESET" })}>Reset</button>
      <input
        type="number"
        value={state.step}
        onChange={(e) => dispatch({ type: "SET_STEP", payload: Number(e.target.value) })}
      />
    </div>
  );
};
```

---

### `useMemo` — cache kết quả tính toán

> Chỉ tính lại khi dependency thay đổi. Dùng khi có **tính toán nặng**.

```tsx
import { useMemo, useState } from "react";

const ExpensiveList = ({ items }: { items: number[] }) => {
  const [filter, setFilter] = useState("");

  // Chỉ filter lại khi items hoặc filter thay đổi
  const filtered = useMemo(
    () => items.filter((i) => String(i).includes(filter)),
    [items, filter]
  );

  return (
    <div>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      <p>Kết quả: {filtered.length} items</p>
    </div>
  );
};
```

> **Lưu ý:** Không nên lạm dụng `useMemo` — nó cũng có chi phí. Chỉ dùng khi thực sự cần tối ưu.

---

### `useCallback` — cache hàm

> Trả về cùng một reference của hàm giữa các render. Dùng khi truyền hàm vào component con được bọc `React.memo`.

```tsx
import { useCallback, useState } from "react";

const Parent = () => {
  const [count, setCount] = useState(0);

  // Không dùng useCallback → hàm mới được tạo mỗi render → Child re-render
  // Dùng useCallback → cùng reference → Child không re-render nếu không cần
  const handleClick = useCallback(() => {
    console.log("Clicked!");
  }, []); // dependency rỗng → chỉ tạo 1 lần

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount((c) => c + 1)}>Tăng</button>
      <Child onClick={handleClick} />
    </div>
  );
};

// React.memo — chỉ re-render khi props thay đổi
const Child = React.memo(({ onClick }: { onClick: () => void }) => {
  console.log("Child render");
  return <button onClick={onClick}>Click me</button>;
});
```

---

### `useId` — tạo ID duy nhất (React 18+)

```tsx
import { useId } from "react";

const FormField = ({ label }: { label: string }) => {
  const id = useId(); // ID duy nhất, ổn định giữa server và client
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} type="text" />
    </div>
  );
};
```

---

## Custom Hook

> Custom Hook là hàm bắt đầu bằng `use`, dùng để **tái sử dụng logic** có dùng hooks.

### `useFetch` — fetch data tái sử dụng

```tsx
import { useEffect, useState } from "react";

function useFetch<T>(url: string) {
  const [data, setData]       = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError]     = useState<string | null>(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);

    fetch(url)
      .then((res) => res.json())
      .then((d) => { if (!cancelled) setData(d); })
      .catch(() => { if (!cancelled) setError("Lỗi khi tải"); })
      .finally(() => { if (!cancelled) setLoading(false); });

    return () => { cancelled = true; };
  }, [url]);

  return { data, loading, error };
}

// Dùng
const Posts = () => {
  const { data, loading, error } = useFetch<Post[]>(
    "https://jsonplaceholder.typicode.com/posts"
  );

  if (loading) return <p>Đang tải...</p>;
  if (error)   return <p>{error}</p>;
  return <ul>{data?.map((p) => <li key={p.id}>{p.title}</li>)}</ul>;
};
```

### `useLocalStorage` — đồng bộ state với localStorage

```tsx
import { useState } from "react";

function useLocalStorage<T>(key: string, initialValue: T) {
  const [stored, setStored] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = (value: T) => {
    setStored(value);
    localStorage.setItem(key, JSON.stringify(value));
  };

  return [stored, setValue] as const;
}

// Dùng
const Settings = () => {
  const [theme, setTheme] = useLocalStorage<"light" | "dark">("theme", "light");
  return <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>
    Theme: {theme}
  </button>;
};
```

### `useDebounce` — trì hoãn cập nhật giá trị

```tsx
import { useEffect, useState } from "react";

function useDebounce<T>(value: T, delay: number = 500): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debounced;
}

// Dùng — tìm kiếm không gọi API liên tục khi gõ
const Search = () => {
  const [query, setQuery]       = useState("");
  const debouncedQuery          = useDebounce(query, 500);

  useEffect(() => {
    if (debouncedQuery) {
      console.log("Tìm kiếm:", debouncedQuery); // chỉ gọi sau 500ms dừng gõ
    }
  }, [debouncedQuery]);

  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
};
```

---

## React.memo & Performance

### `React.memo` — tránh re-render không cần thiết

```tsx
// Component con sẽ re-render mỗi khi cha re-render, dù props không đổi
// React.memo giải quyết điều này
const Avatar = React.memo(({ name, url }: { name: string; url: string }) => {
  console.log("Avatar render");
  return <img src={url} alt={name} />;
});

// So sánh custom — khi props là object/array
const UserRow = React.memo(
  ({ user }: { user: { id: number; name: string } }) => <tr><td>{user.name}</td></tr>,
  (prev, next) => prev.user.id === next.user.id && prev.user.name === next.user.name
);
```

---

## Event Handling

```tsx
// onClick
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
  e.preventDefault();
  console.log("Clicked");
};

// onChange
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  console.log(e.target.value);
};

// onSubmit
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();
  // xử lý form
};

// Các event type phổ biến
// React.MouseEvent<HTMLElement>
// React.ChangeEvent<HTMLInputElement | HTMLSelectElement | HTMLTextAreaElement>
// React.KeyboardEvent<HTMLElement>
// React.FocusEvent<HTMLElement>
// React.FormEvent<HTMLFormElement>
```

---

## Forms

### Controlled Component — state kiểm soát input

```tsx
const LoginForm = () => {
  const [form, setForm] = useState({ email: "", password: "" });
  const [error, setError] = useState("");

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const { name, value } = e.target;
    setForm((prev) => ({ ...prev, [name]: value }));
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (!form.email) { setError("Email là bắt buộc"); return; }
    console.log("Đăng nhập:", form);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="email"    value={form.email}    onChange={handleChange} placeholder="Email" />
      <input name="password" value={form.password} onChange={handleChange} type="password" />
      {error && <p style={{ color: "red" }}>{error}</p>}
      <button type="submit">Đăng nhập</button>
    </form>
  );
};
```

### Uncontrolled Component — dùng `useRef`

```tsx
const QuickForm = () => {
  const inputRef = useRef<HTMLInputElement>(null);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    console.log(inputRef.current?.value);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input ref={inputRef} defaultValue="" />
      <button type="submit">Gửi</button>
    </form>
  );
};
```

---

## Component Patterns

### Compound Component — các component liên quan dùng chung

```tsx
// Ví dụ: <Tabs> chứa <Tabs.List> và <Tabs.Panel>
const TabsContext = createContext<{ active: string; setActive: (v: string) => void } | null>(null);

const Tabs = ({ children, defaultTab }: { children: React.ReactNode; defaultTab: string }) => {
  const [active, setActive] = useState(defaultTab);
  return (
    <TabsContext.Provider value={{ active, setActive }}>
      <div>{children}</div>
    </TabsContext.Provider>
  );
};

Tabs.List = ({ items }: { items: string[] }) => {
  const ctx = useContext(TabsContext)!;
  return (
    <div style={{ display: "flex", gap: 8 }}>
      {items.map((item) => (
        <button
          key={item}
          onClick={() => ctx.setActive(item)}
          style={{ fontWeight: ctx.active === item ? "bold" : "normal" }}
        >
          {item}
        </button>
      ))}
    </div>
  );
};

Tabs.Panel = ({ name, children }: { name: string; children: React.ReactNode }) => {
  const ctx = useContext(TabsContext)!;
  return ctx.active === name ? <div>{children}</div> : null;
};

// Dùng
const App = () => (
  <Tabs defaultTab="profile">
    <Tabs.List items={["profile", "settings"]} />
    <Tabs.Panel name="profile"><p>Hồ sơ</p></Tabs.Panel>
    <Tabs.Panel name="settings"><p>Cài đặt</p></Tabs.Panel>
  </Tabs>
);
```

### Render Props — chia sẻ logic qua props

```tsx
interface MousePosition { x: number; y: number; }

const MouseTracker = ({ render }: { render: (pos: MousePosition) => React.ReactNode }) => {
  const [pos, setPos] = useState<MousePosition>({ x: 0, y: 0 });

  return (
    <div
      style={{ height: 300, border: "1px solid #ccc" }}
      onMouseMove={(e) => setPos({ x: e.clientX, y: e.clientY })}
    >
      {render(pos)}
    </div>
  );
};

// Dùng
<MouseTracker render={({ x, y }) => <p>X: {x}, Y: {y}</p>} />
```

### HOC (Higher-Order Component) — bọc component để thêm logic

```tsx
// HOC thêm tính năng loading
function withLoading<P extends object>(Component: React.ComponentType<P>) {
  return function WithLoadingComponent({ isLoading, ...props }: P & { isLoading: boolean }) {
    if (isLoading) return <p>Đang tải...</p>;
    return <Component {...(props as P)} />;
  };
}

const UserList = ({ users }: { users: string[] }) => (
  <ul>{users.map((u) => <li key={u}>{u}</li>)}</ul>
);

const UserListWithLoading = withLoading(UserList);

// Dùng
<UserListWithLoading isLoading={loading} users={users} />
```

---

## Error Boundary

> Bắt lỗi JavaScript trong cây component con và hiển thị UI fallback thay vì crash toàn app. Phải dùng **Class Component** (chưa có Hook thay thế).

```tsx
import { Component, ErrorInfo, ReactNode } from "react";

interface Props { children: ReactNode; fallback?: ReactNode; }
interface State { hasError: boolean; error: Error | null; }

class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    console.error("ErrorBoundary caught:", error, info);
    // Gửi lên error tracking service (Sentry, etc.)
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? <p>Có lỗi xảy ra. Vui lòng thử lại.</p>;
    }
    return this.props.children;
  }
}

// Dùng
const App = () => (
  <ErrorBoundary fallback={<p>Lỗi trong UserProfile!</p>}>
    <UserProfile />
  </ErrorBoundary>
);
```

---

## React Router (v6)

> Routing cho Single Page Application (SPA).

```bash
npm install react-router-dom
```

### Cấu hình cơ bản

```tsx
import { BrowserRouter, Routes, Route, Link, NavLink, useNavigate, useParams } from "react-router-dom";

const App = () => (
  <BrowserRouter>
    <nav>
      <NavLink to="/"        end>Trang chủ</NavLink>
      <NavLink to="/about"      >Giới thiệu</NavLink>
      <NavLink to="/users"      >Người dùng</NavLink>
    </nav>

    <Routes>
      <Route path="/"           element={<Home />} />
      <Route path="/about"      element={<About />} />
      <Route path="/users"      element={<Users />} />
      <Route path="/users/:id"  element={<UserDetail />} />
      <Route path="*"           element={<NotFound />} />  {/* 404 */}
    </Routes>
  </BrowserRouter>
);
```

### `useParams` — đọc URL params

```tsx
const UserDetail = () => {
  const { id } = useParams<{ id: string }>();
  return <p>User ID: {id}</p>;
};
```

### `useNavigate` — điều hướng bằng code

```tsx
const LoginPage = () => {
  const navigate = useNavigate();

  const handleLogin = async () => {
    await login();
    navigate("/dashboard");          // chuyển trang
    // navigate(-1);                 // quay lại trang trước
    // navigate("/dashboard", { replace: true }); // thay vì push
  };

  return <button onClick={handleLogin}>Đăng nhập</button>;
};
```

### Nested Routes — layout lồng nhau

```tsx
import { Outlet } from "react-router-dom";

const DashboardLayout = () => (
  <div style={{ display: "flex" }}>
    <aside>
      <Link to="/dashboard">Overview</Link>
      <Link to="/dashboard/users">Users</Link>
    </aside>
    <main>
      <Outlet /> {/* Nơi render route con */}
    </main>
  </div>
);

// Trong Routes
<Route path="/dashboard" element={<DashboardLayout />}>
  <Route index          element={<Overview />} />
  <Route path="users"   element={<Users />} />
  <Route path="users/:id" element={<UserDetail />} />
</Route>
```

### Protected Route — bảo vệ route cần đăng nhập

```tsx
const ProtectedRoute = ({ children }: { children: React.ReactNode }) => {
  const { isLoggedIn } = useAuth();
  return isLoggedIn ? <>{children}</> : <Navigate to="/login" replace />;
};

// Dùng
<Route path="/dashboard" element={<ProtectedRoute><Dashboard /></ProtectedRoute>} />
```

---

## Styling

### CSS Module — scope CSS theo component

```tsx
// Button.module.css
// .btn { background: blue; color: white; }
// .btn:hover { background: darkblue; }

import styles from "./Button.module.css";

const Button = ({ label }: { label: string }) => (
  <button className={styles.btn}>{label}</button>
);
```

### Inline style

```tsx
const Box = ({ color }: { color: string }) => (
  <div style={{ backgroundColor: color, padding: 16, borderRadius: 8 }}>
    Nội dung
  </div>
);
```

### Tailwind CSS (phổ biến với Next.js)

```tsx
const Card = ({ title }: { title: string }) => (
  <div className="rounded-xl shadow-md p-4 bg-white hover:shadow-lg transition">
    <h2 className="text-xl font-bold text-gray-800">{title}</h2>
  </div>
);
```

### clsx — ghép className có điều kiện

```bash
npm install clsx
```

```tsx
import clsx from "clsx";

const Button = ({ variant, disabled }: { variant: "primary" | "ghost"; disabled?: boolean }) => (
  <button
    className={clsx(
      "px-4 py-2 rounded",
      variant === "primary" && "bg-blue-500 text-white",
      variant === "ghost"   && "bg-transparent border border-gray-300",
      disabled              && "opacity-50 cursor-not-allowed"
    )}
    disabled={disabled}
  >
    Click
  </button>
);
```

---

## State Management

### Khi nào dùng gì?

|Phạm vi|Giải pháp|
|---|---|
|1 component|`useState` / `useReducer`|
|Vài component gần nhau|Lift state up + props|
|Cây component sâu|`useContext`|
|Toàn app (phức tạp)|Zustand / Redux Toolkit|
|Server state (fetch/cache)|TanStack Query|

### Zustand — state management đơn giản

```bash
npm install zustand
```

```tsx
import { create } from "zustand";

interface CartStore {
  items: { id: number; name: string; qty: number }[];
  addItem:    (item: { id: number; name: string }) => void;
  removeItem: (id: number) => void;
  total:      () => number;
}

const useCartStore = create<CartStore>((set, get) => ({
  items: [],

  addItem: (item) =>
    set((state) => {
      const existing = state.items.find((i) => i.id === item.id);
      if (existing) {
        return { items: state.items.map((i) => i.id === item.id ? { ...i, qty: i.qty + 1 } : i) };
      }
      return { items: [...state.items, { ...item, qty: 1 }] };
    }),

  removeItem: (id) =>
    set((state) => ({ items: state.items.filter((i) => i.id !== id) })),

  total: () => get().items.reduce((sum, i) => sum + i.qty, 0),
}));

// Dùng trong component
const Cart = () => {
  const { items, removeItem, total } = useCartStore();
  return (
    <div>
      <p>Tổng: {total()} sản phẩm</p>
      {items.map((i) => (
        <div key={i.id}>
          <span>{i.name} x{i.qty}</span>
          <button onClick={() => removeItem(i.id)}>Xóa</button>
        </div>
      ))}
    </div>
  );
};
```

---

## TypeScript với React

### Typing Props phổ biến

```tsx
interface Props {
  // Kiểu cơ bản
  name:      string;
  age:       number;
  active:    boolean;

  // Optional
  nickname?: string;

  // Function
  onClick:       () => void;
  onChange:      (value: string) => void;
  onSubmit:      (e: React.FormEvent) => void;

  // Children
  children:      React.ReactNode;     // bất kỳ JSX nào
  title:         React.ReactElement;  // phải là JSX element
  render:        () => React.ReactNode; // render prop

  // Style
  style?:        React.CSSProperties;
  className?:    string;

  // Union
  variant:       "primary" | "secondary" | "danger";

  // Object & Array
  user:          { id: number; name: string };
  tags:          string[];
}
```

### Typing `useState` với union type

```tsx
type Status = "idle" | "loading" | "success" | "error";

const [status, setStatus] = useState<Status>("idle");
```

### Typing `useRef`

```tsx
const inputRef  = useRef<HTMLInputElement>(null);
const divRef    = useRef<HTMLDivElement>(null);
const timerRef  = useRef<ReturnType<typeof setTimeout> | null>(null);
```

### Typing Event Handlers

```tsx
const handleClick  = (e: React.MouseEvent<HTMLButtonElement>) => {};
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {};
const handleKey    = (e: React.KeyboardEvent<HTMLInputElement>) => {};
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {};
```

### `ComponentProps` — kế thừa props của HTML element

```tsx
import { ComponentProps } from "react";

// Kế thừa tất cả props của <button> rồi thêm vào
interface ButtonProps extends ComponentProps<"button"> {
  variant?: "primary" | "ghost";
  isLoading?: boolean;
}

const Button = ({ variant = "primary", isLoading, children, ...rest }: ButtonProps) => (
  <button
    {...rest}
    disabled={isLoading || rest.disabled}
    className={clsx(variant === "primary" && "btn-primary", rest.className)}
  >
    {isLoading ? "Đang xử lý..." : children}
  </button>
);
```

---

## React 18+ Features

### `Suspense` — loading state khai báo

```tsx
import { Suspense, lazy } from "react";

// Lazy load component — chỉ tải khi cần
const HeavyChart = lazy(() => import("./HeavyChart"));

const Dashboard = () => (
  <Suspense fallback={<p>Đang tải biểu đồ...</p>}>
    <HeavyChart />
  </Suspense>
);
```

### Concurrent Features — `useTransition`

> Đánh dấu update ít quan trọng hơn → UI vẫn responsive khi xử lý nặng.

```tsx
import { useState, useTransition } from "react";

const SearchPage = () => {
  const [query,   setQuery]   = useState("");
  const [results, setResults] = useState<string[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setQuery(value); // update ngay — giữ input responsive

    startTransition(() => {
      // update này được đánh dấu là ít quan trọng hơn
      setResults(heavySearch(value));
    });
  };

  return (
    <div>
      <input value={query} onChange={handleSearch} />
      {isPending ? <p>Đang tìm kiếm...</p> : <ResultList results={results} />}
    </div>
  );
};
```

### `useDeferredValue` — trì hoãn giá trị ít quan trọng

```tsx
import { useState, useDeferredValue } from "react";

const FilterList = ({ items }: { items: string[] }) => {
  const [filter, setFilter] = useState("");
  const deferredFilter      = useDeferredValue(filter); // trị hoãn lọc

  const filtered = items.filter((i) => i.includes(deferredFilter));

  return (
    <div>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      <ul>{filtered.map((i) => <li key={i}>{i}</li>)}</ul>
    </div>
  );
};
```

---

## Next.js (App Router)

> Next.js = React + Server-Side Rendering + File-based routing + nhiều tính năng khác.

### Server vs Client Component

```tsx
// Server Component — mặc định trong App Router
// Chạy trên server, không có state/event/browser API
// app/page.tsx
const HomePage = async () => {
  const data = await fetch("https://api.example.com/posts"); // fetch thẳng, không cần useEffect
  const posts = await data.json();

  return (
    <ul>
      {posts.map((p: any) => <li key={p.id}>{p.title}</li>)}
    </ul>
  );
};

export default HomePage;
```

```tsx
// Client Component — phải thêm "use client" ở đầu file
// Dùng khi cần: state, event handlers, browser API, hooks
"use client";

import { useState } from "react";

const Counter = () => {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount((c) => c + 1)}>{count}</button>;
};

export default Counter;
```

### File-based Routing (App Router)

```
app/
├── page.tsx              → /
├── about/
│   └── page.tsx          → /about
├── blog/
│   ├── page.tsx          → /blog
│   └── [slug]/
│       └── page.tsx      → /blog/:slug
└── dashboard/
    ├── layout.tsx        → Layout dùng chung
    ├── page.tsx          → /dashboard
    └── users/
        └── page.tsx      → /dashboard/users
```

### `layout.tsx` — layout dùng chung

```tsx
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="vi">
      <body>
        <header>Header</header>
        {children}
        <footer>Footer</footer>
      </body>
    </html>
  );
}
```

### Dynamic routes — `[slug]`

```tsx
// app/blog/[slug]/page.tsx
interface Props {
  params: { slug: string };
}

const BlogPost = async ({ params }: Props) => {
  const post = await fetchPost(params.slug);
  return <article><h1>{post.title}</h1></article>;
};

export default BlogPost;
```

### `loading.tsx` & `error.tsx`

```tsx
// app/dashboard/loading.tsx — tự động hiện khi route đang load
export default function Loading() {
  return <div className="spinner">Đang tải...</div>;
}

// app/dashboard/error.tsx — bắt lỗi của route
"use client";
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div>
      <p>Lỗi: {error.message}</p>
      <button onClick={reset}>Thử lại</button>
    </div>
  );
}
```

### Metadata — SEO

```tsx
// app/about/page.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title:       "Giới thiệu | My App",
  description: "Trang giới thiệu về chúng tôi",
  openGraph: {
    title: "Giới thiệu",
    images: ["/og-image.png"],
  },
};

export default function AboutPage() {
  return <h1>Giới thiệu</h1>;
}
```

### Server Actions — xử lý form trên server

```tsx
// app/actions.ts
"use server";

export async function createUser(formData: FormData) {
  const name = formData.get("name") as string;
  await db.user.create({ data: { name } });
  revalidatePath("/users"); // làm mới cache trang /users
}

// app/users/page.tsx (Server Component)
import { createUser } from "../actions";

export default function UsersPage() {
  return (
    <form action={createUser}>
      <input name="name" placeholder="Tên người dùng" />
      <button type="submit">Tạo</button>
    </form>
  );
}
```