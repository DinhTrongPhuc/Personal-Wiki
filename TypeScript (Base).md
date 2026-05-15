## Định nghĩa

### JavaScript (JS)

- Là ngôn ngữ gốc, chạy trực tiếp trên trình duyệt và Node.js
- Không cần khai báo kiểu dữ liệu
- Viết nhanh, linh hoạt nhưng dễ bug nếu code lớn

### TypeScript (TS) — do Microsoft phát triển

- Là JavaScript + kiểu dữ liệu (type)
- Không chạy trực tiếp → tự **compile** (biên dịch) sang JS khi run
- Giúp phát hiện lỗi ngay lúc code, gợi ý code thông minh, dễ debug hơn

---

## Data Types

### Primitive: `number`, `string`, `boolean`

```ts
let age: number = 20;
let name: string = "name_string";
let isOnline: boolean = true;
```

### Type Inference — TS tự đoán kiểu dữ liệu

```ts
let score = 10;
// TS tự hiểu score là number
```

### `any` / `unknown` — tránh sử dụng

> Sử dụng `any`/`unknown` là tự loại bỏ lợi thế của TS về kiểu dữ liệu và debug.

```ts
let data: any = 10;
data = "hello";
data = true;
```

```ts
let value: unknown = "hello";

if (typeof value === "string") {
  console.log(value.toUpperCase());
}
```

---

## Array

```ts
let numbers: number[] = [1, 2, 3];
let names: string[] = ["A", "B"];
```

```ts
let users: Array<string> = ["Phúc", "Hiền"];

let users2: Array<string | number>;
users2 = ["Hiền", 1];

let user3: [string, string]; // Tuple — chỉ nhận đúng 2 giá trị string
user3 = ["Hiền", "Phúc"];
```

---

## Object

```ts
let user: {
  name: string;
  age: number;
  hobbies?: string[];  // optional property
  role: {
    value: string;
    description: string;
  };
} = {
  name: "Phúc",
  age: 20,
  hobbies: ["coding", "sleeping"],
  role: {
    value: "none",
    description: "nothing",
  },
};
```

```ts
const User = {
  name: "Hiền",
  age: 20,
}; // TS tự infer type
```

---

## Function

### Cấu trúc hàm truyền thống

```ts
// Function Declaration
function name(param: Type): ReturnType {
  return value;
}

// Function Expression
const name = function (param: Type): ReturnType {
  return value;
};
```

### Arrow Function

```ts
// Đầy đủ
const name = (param: Type): ReturnType => {
  return value;
};

// Rút gọn
const name = (param: Type): ReturnType => value;
```

> **Lưu ý:** Nếu không trả về giá trị, kiểu trả về là `void`.

---

### `function` vs `const` (Arrow Function)

#### Hoisting

```ts
// ✅ Hợp lệ với Function Declaration
sayHello();
function sayHello() { console.log("Chào bạn!"); }
```

```ts
// ❌ Lỗi với Arrow Function
sayHi();
const sayHi = () => { console.log("Chào nhé!"); };
// ReferenceError: Cannot access 'sayHi' before initialization
```

#### `this` context

- **Arrow function**: không có `this` riêng → kế thừa `this` từ phạm vi ngoài
- **Regular function**: `this` phụ thuộc vào cách gọi hàm

```ts
class Counter {
  count: number = 0;

  // Cách 1: Regular Function — dễ mất 'this'
  increaseRegular() {
    this.count++;
  }

  // Cách 2: Arrow Function — tự động giữ 'this'
  increaseArrow = () => {
    this.count++;
  };
}

const myCounter = new Counter();
const action1 = myCounter.increaseRegular;
const action2 = myCounter.increaseArrow;

// action1(); ❌ Lỗi: 'this' bị undefined
action2();   // ✅ Chạy tốt
```

---

### `this` trong TypeScript

#### `this` trong class

```ts
class SmartPhone {
  brand: string;

  constructor(brand: string) {
    this.brand = brand;
  }

  showBrand() {
    console.log(`Điện thoại này thuộc hãng: ${this.brand}`);
  }
}

const myPhone = new SmartPhone("iPhone");
myPhone.showBrand(); // Điện thoại này thuộc hãng: iPhone
```

#### Explicit `this` trong hàm

```ts
interface User {
  name: string;
}

function sayHello(this: User, message: string) {
  console.log(`${this.name} gửi lời chào: ${message}`);
}

const member = { name: "Hoàng" };

sayHello.call(member, "Chào mừng bạn!"); // ✅ Hợp lệ
// sayHello("Chào!");                     // ❌ Lỗi: 'this' không phải kiểu 'User'
```

---

### Các loại hàm

#### Hàm có return

```ts
function sum(a: number, b: number): number {
  return a + b;
}

const greet = (name: string): string => {
  return `Hello ${name}`;
};
```

#### Hàm không return (`void`)

```ts
function log(message: string): void {
  console.log(message);
}

const sayHello = () => {
  console.log("Hello");
};
```

#### Optional parameter (`?`)

```ts
function greet(name: string, greeting?: string) {
  if (greeting) {
    return `${greeting}, ${name}!`;
  }
  return `Hello, ${name}!`;
}

console.log(greet("Alice"));              // "Hello, Alice!"
console.log(greet("Bob", "Good morning")); // "Good morning, Bob!"
```

#### Default parameter

```ts
function sayHi(name: string = "Guest") {
  console.log(name);
}
```

---

### Export / Import

#### Named export — dùng `{ }`

```ts
// math.ts
export const add = (a: number, b: number) => a + b;
export const sub = (a: number, b: number) => a - b;
```

```ts
// app.ts
import { add, sub } from "./math";

// Đổi tên khi import
import { add as sum } from "./math";
```

> **Lưu ý:**
> 
> - Một file có thể có nhiều **named export**, nhưng chỉ **1 `export default`**
> - Hàm không có `export` chỉ tồn tại trong phạm vi file đó (private)
> - Hàm có `export` là public, có thể được `import` từ file khác

---

## Type và Interface

> Cả hai đều dùng để định nghĩa **cấu trúc dữ liệu** (shape of object) và kiểm tra kiểu.

### Type (Type Alias)

```ts
type User = {
  id: number;
  name: string;
};

// Union type
type ID = string | number;

// Intersection type
type User = { name: string } & { age: number };
```

**Dùng `type` khi:** union type, function type, kiểu phức tạp/linh hoạt.

### Interface

```ts
interface Person {
  name: string;
  age: number;
}
```

**Dùng `interface` khi:** định nghĩa object, API, class. Hỗ trợ **Declaration Merging**.

---

### So sánh `type` vs `interface`

||Interface|Type|
|---|---|---|
|Khai báo|`interface User { id: number; }`|`type User = { id: number; }`|
|Mở rộng|`interface Admin extends User { role: string; }`|`type Admin = User & { role: string; }`|
|Implements|`class X implements User {}`|`class X implements User {}`|

#### Điểm khác biệt: Override property

```ts
// ❌ Interface — báo lỗi khi override kiểu
interface PersonInterface { name: string; }
interface Person extends PersonInterface {
  name: string[]; // Lỗi: không thể override
}
```

```ts
// ✅ Type — cho phép override
type Student = { name: string; }
type StudentProfile = Student & { name: string[]; } // Không lỗi
```

---

## Type Nâng Cao

### Function type

```ts
type Callback = (value: number) => void;

type Greet = (name: string) => string;

const sayHello: Greet = (name) => {
  return `Hello, ${name}`;
};
```

### Type Assertion — ép kiểu

```ts
let value: unknown = "hello";
let str = value as string;
```

### `readonly`

```ts
type User = {
  readonly id: number;
  name: string;
};

interface User {
  readonly id: number;
  name: string;
}

const user: User = { id: 1, name: "Alice" };
// user.id = 2; // ❌ Lỗi: read-only property
```

#### `Readonly<T>` — Utility Type

```ts
interface Todo { title: string; }

const todo: Readonly<Todo> = { title: "Delete" };
// todo.title = "Hello"; // ❌ Lỗi
```

---

## Class

```ts
// Backend / OOP style
class User {
  constructor(
    public name: string,
    private age: number
  ) {}

  getAge() {
    return this.age;
  }
}
```

```ts
class Car {
  readonly model: string;

  constructor(model: string) {
    this.model = model;
  }
}
```

---

## Enum

```ts
enum UserRole {
  Admin,
  Viewer,
  Editor,
  Guest,
}

// Giá trị mặc định: Admin=0, Viewer=1, Editor=2, Guest=3

let userRole: UserRole = UserRole.Admin;

// Dùng let để có thể thay đổi sau
userRole = UserRole.Guest;
```

> **Tip:** Dùng `UserRole.Admin` thay vì số `0` → IDE tự gợi ý, dễ đọc hơn.

---

## Generics

> Generics cho phép viết hàm/class/interface **tái sử dụng được với nhiều kiểu dữ liệu** mà vẫn đảm bảo type safety. Thay vì dùng `any`, dùng `<T>` để TS tự suy ra kiểu.

### Generic Function

```ts
// Không dùng generic → phải viết lại cho từng kiểu
function getFirstNumber(arr: number[]): number { return arr[0]; }
function getFirstString(arr: string[]): string { return arr[0]; }

// ✅ Dùng generic — một hàm cho tất cả
function getFirst<T>(arr: T[]): T {
  return arr[0];
}

getFirst<number>([1, 2, 3]);   // 1
getFirst<string>(["a", "b"]); // "a"
getFirst([true, false]);       // TS tự infer T = boolean
```

### Generic với nhiều type parameter

```ts
function pair<T, U>(first: T, second: U): [T, U] {
  return [first, second];
}

pair<string, number>("tuổi", 20); // ["tuổi", 20]
```

### Generic Interface

```ts
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
}

// Dùng với kiểu cụ thể
const userResponse: ApiResponse<{ name: string; age: number }> = {
  data: { name: "Phúc", age: 20 },
  status: 200,
  message: "OK",
};
```

### Generic với Constraint (`extends`)

> Dùng `extends` để giới hạn `T` chỉ nhận những kiểu có thuộc tính nhất định.

```ts
// T phải có thuộc tính 'length'
function logLength<T extends { length: number }>(value: T): void {
  console.log(value.length);
}

logLength("hello");    // ✅ string có .length
logLength([1, 2, 3]);  // ✅ array có .length
// logLength(123);     // ❌ number không có .length
```

### Generic Class

```ts
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }
}

const numStack = new Stack<number>();
numStack.push(1);
numStack.push(2);
numStack.pop(); // 2
```

---

## Utility Types

> Các kiểu có sẵn trong TS giúp **biến đổi type** mà không cần định nghĩa lại từ đầu.

### `Partial<T>` — tất cả property thành optional

```ts
interface User {
  id: number;
  name: string;
  email: string;
}

// Dùng khi update — không cần truyền đủ tất cả field
function updateUser(id: number, changes: Partial<User>) {
  // changes có thể chỉ có { name } hoặc { email } hoặc cả hai
}

updateUser(1, { name: "Phúc" });         // ✅
updateUser(1, { email: "a@b.com" });     // ✅
```

### `Required<T>` — tất cả property thành bắt buộc

```ts
interface Config {
  host?: string;
  port?: number;
}

const config: Required<Config> = {
  host: "localhost", // ✅ bắt buộc phải có
  port: 3000,        // ✅ bắt buộc phải có
};
```

### `Pick<T, K>` — chọn một số property

```ts
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
}

// Chỉ lấy id và name — dùng khi trả về API không cần password
type PublicUser = Pick<User, "id" | "name">;

const publicUser: PublicUser = { id: 1, name: "Phúc" };
```

### `Omit<T, K>` — bỏ một số property

```ts
// Loại bỏ password — ngược lại với Pick
type SafeUser = Omit<User, "password">;

const safeUser: SafeUser = { id: 1, name: "Phúc", email: "a@b.com" };
```

### `Record<K, V>` — tạo object với key/value type cụ thể

```ts
// Tạo map từ string → number
type ScoreBoard = Record<string, number>;

const scores: ScoreBoard = {
  Phúc: 90,
  Hiền: 85,
};

// Dùng với union type làm key
type Role = "admin" | "viewer" | "editor";
type RoleConfig = Record<Role, boolean>;

const config: RoleConfig = {
  admin: true,
  viewer: false,
  editor: true,
};
```

### `Exclude<T, U>` — loại bỏ kiểu khỏi union

```ts
type Status = "active" | "inactive" | "banned";

type ActiveStatus = Exclude<Status, "banned">;
// = "active" | "inactive"
```

### `Extract<T, U>` — chỉ giữ lại kiểu trong union

```ts
type Status = "active" | "inactive" | "banned";

type NegativeStatus = Extract<Status, "inactive" | "banned">;
// = "inactive" | "banned"
```

### `NonNullable<T>` — loại bỏ `null` và `undefined`

```ts
type MaybeString = string | null | undefined;

type DefiniteString = NonNullable<MaybeString>;
// = string
```

### `ReturnType<T>` — lấy kiểu trả về của hàm

```ts
function getUser() {
  return { id: 1, name: "Phúc" };
}

type User = ReturnType<typeof getUser>;
// = { id: number; name: string }
```

### `Parameters<T>` — lấy kiểu tham số của hàm

```ts
function createUser(name: string, age: number): void {}

type CreateUserParams = Parameters<typeof createUser>;
// = [string, number]
```

---

## Type Guards

> Type Guards giúp TS **thu hẹp kiểu** (narrow type) tại runtime, từ kiểu rộng (`string | number`) về kiểu cụ thể hơn.

### `typeof` — kiểm tra primitive

```ts
function format(value: string | number): string {
  if (typeof value === "string") {
    return value.toUpperCase(); // TS biết value là string ở đây
  }
  return value.toFixed(2);     // TS biết value là number ở đây
}
```

### `instanceof` — kiểm tra class instance

```ts
class Dog { bark() { console.log("Woof!"); } }
class Cat { meow() { console.log("Meow!"); } }

function makeSound(animal: Dog | Cat) {
  if (animal instanceof Dog) {
    animal.bark(); // ✅ TS biết là Dog
  } else {
    animal.meow(); // ✅ TS biết là Cat
  }
}
```

### `in` — kiểm tra property tồn tại

```ts
interface Admin { role: string; permissions: string[]; }
interface User  { name: string; email: string; }

function greet(person: Admin | User) {
  if ("role" in person) {
    console.log(`Admin: ${person.role}`); // TS biết là Admin
  } else {
    console.log(`User: ${person.name}`);  // TS biết là User
  }
}
```

### Custom Type Guard — `is`

> Viết hàm guard riêng cho kiểu phức tạp. Trả về `value is Type` thay vì `boolean`.

```ts
interface Cat { meow(): void; }
interface Dog { bark(): void; }

// Hàm guard tự định nghĩa
function isCat(animal: Cat | Dog): animal is Cat {
  return (animal as Cat).meow !== undefined;
}

function handleAnimal(animal: Cat | Dog) {
  if (isCat(animal)) {
    animal.meow(); // ✅ TS biết chắc là Cat
  } else {
    animal.bark(); // ✅ TS biết chắc là Dog
  }
}
```

### Discriminated Union — pattern phổ biến nhất

> Thêm một field chung (thường là `type` hoặc `kind`) để phân biệt các kiểu trong union.

```ts
interface Circle  { kind: "circle";  radius: number; }
interface Square  { kind: "square";  side: number; }
interface Triangle { kind: "triangle"; base: number; height: number; }

type Shape = Circle | Square | Triangle;

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":   return Math.PI * shape.radius ** 2;
    case "square":   return shape.side ** 2;
    case "triangle": return 0.5 * shape.base * shape.height;
  }
}
```

---

## Decorators

> Decorators là cú pháp `@` đặt trước class, method, property hoặc parameter. Phổ biến trong **NestJS**, **Angular**.
> 
> Cần bật trong `tsconfig.json`: `"experimentalDecorators": true`

### Class Decorator

```ts
function Logger(constructor: Function) {
  console.log(`Class ${constructor.name} được khởi tạo`);
}

@Logger
class UserService {
  getUser() { return "Phúc"; }
}
// Log: "Class UserService được khởi tạo"
```

### Method Decorator

```ts
function Log(target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  descriptor.value = function (...args: any[]) {
    console.log(`Gọi ${key} với args:`, args);
    return original.apply(this, args);
  };
  return descriptor;
}

class MathService {
  @Log
  add(a: number, b: number) {
    return a + b;
  }
}

const math = new MathService();
math.add(2, 3);
// Log: "Gọi add với args: [2, 3]"
// Kết quả: 5
```

### Property Decorator

```ts
function Required(target: any, key: string) {
  let value: any;
  Object.defineProperty(target, key, {
    get: () => value,
    set: (v) => {
      if (!v) throw new Error(`${key} là bắt buộc!`);
      value = v;
    },
  });
}

class Product {
  @Required
  name: string = "";
}
```

### Decorator trong NestJS (thực tế)

```ts
import { Controller, Get, Post, Body } from "@nestjs/common";

@Controller("users")         // Class decorator — định nghĩa route prefix
export class UserController {

  @Get()                     // Method decorator — GET /users
  findAll() { return []; }

  @Post()                    // Method decorator — POST /users
  create(@Body() dto: any) { // Parameter decorator — lấy request body
    return dto;
  }
}
```

---

## Namespace & Module

### Module (ES Module) — cách hiện đại, khuyên dùng

> Mỗi file `.ts` là một module. Dùng `export`/`import` để chia sẻ code.

```ts
// utils/math.ts
export function add(a: number, b: number) { return a + b; }
export const PI = 3.14159;

// main.ts
import { add, PI } from "./utils/math";
```

### Namespace — cách cũ, dùng khi không có module system

> Namespace nhóm các kiểu/hàm liên quan vào một "không gian tên" để tránh xung đột tên.

```ts
namespace Validation {
  export interface StringValidator {
    isValid(s: string): boolean;
  }

  export class EmailValidator implements StringValidator {
    isValid(s: string): boolean {
      return s.includes("@");
    }
  }
}

const validator = new Validation.EmailValidator();
validator.isValid("test@email.com"); // true
```

### Namespace lồng nhau

```ts
namespace App {
  export namespace Models {
    export interface User { id: number; name: string; }
  }

  export namespace Services {
    export function getUser(id: number): Models.User {
      return { id, name: "Phúc" };
    }
  }
}

const user = App.Services.getUser(1);
```

> **Khuyến nghị:** Dùng **ES Module** (`import`/`export`) trong dự án hiện đại. Namespace chủ yếu gặp trong code cũ hoặc khi viết `.d.ts` declaration file.

---

## tsconfig.json

> File cấu hình compiler của TypeScript. Đặt ở root project.

### Cấu hình cơ bản

```json
{
  "compilerOptions": {
    // --- Phiên bản ---
    "target": "ES2020",          // JS version sau khi compile
    "module": "commonjs",        // Module system (commonjs cho Node, ESNext cho browser)
    "lib": ["ES2020", "DOM"],    // Thư viện built-in được dùng

    // --- Đường dẫn ---
    "rootDir": "./src",          // Thư mục source TS
    "outDir": "./dist",          // Thư mục output JS sau compile

    // --- Type checking ---
    "strict": true,              // Bật tất cả strict checks (khuyến nghị)
    "noImplicitAny": true,       // Không cho phép 'any' ngầm định
    "strictNullChecks": true,    // Phân biệt null/undefined với các kiểu khác

    // --- Module resolution ---
    "esModuleInterop": true,     // Cho phép import CommonJS module dễ hơn
    "moduleResolution": "node",  // Cách TS tìm module (node / bundler)
    "baseUrl": ".",              // Base URL cho path alias
    "paths": {                   // Path alias
      "@/*": ["src/*"]
    },

    // --- Output ---
    "sourceMap": true,           // Tạo .map file để debug
    "declaration": true,         // Tạo .d.ts file
    "removeComments": true,      // Xóa comment khỏi output JS

    // --- Decorators (NestJS/Angular) ---
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,

    // --- Khác ---
    "skipLibCheck": true         // Bỏ qua check .d.ts của thư viện bên ngoài
  },

  "include": ["src/**/*"],       // File nào được compile
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

### Các `strict` checks quan trọng

|Option|Ý nghĩa|
|---|---|
|`strict`|Bật tất cả strict options bên dưới|
|`noImplicitAny`|Lỗi khi TS không thể infer type → phải khai báo rõ|
|`strictNullChecks`|`null`/`undefined` không được gán vào kiểu khác|
|`strictFunctionTypes`|Kiểm tra chặt kiểu của function parameter|
|`noUnusedLocals`|Cảnh báo biến local khai báo nhưng không dùng|
|`noUnusedParameters`|Cảnh báo tham số hàm không dùng|
|`noImplicitReturns`|Lỗi nếu hàm có thể không return giá trị|

### `target` phổ biến

|Target|Dùng khi|
|---|---|
|`ES5`|Cần hỗ trợ IE11 hoặc trình duyệt cũ|
|`ES2017`|Node.js 8+|
|`ES2020`|Node.js 14+, trình duyệt hiện đại|
|`ESNext`|Dùng mới nhất, không cần hỗ trợ cũ|

### Path alias (`@/`)

```json
// tsconfig.json
"baseUrl": ".",
"paths": { "@/*": ["src/*"] }
```

```ts
// Thay vì import đường dài
import { UserService } from "../../../services/user.service";

// Dùng alias
import { UserService } from "@/services/user.service";
```

> **Lưu ý:** Path alias trong `tsconfig.json` chỉ cho TS hiểu. Khi bundle (webpack/vite/jest), cần cấu hình thêm ở phía bundler để runtime cũng hiểu.