---
title: Stream, Exception, I/O và Date-Time
aliases: [Stream API, Exception Handling, NIO, java.time]
tags: [java, stream, exception, io, datetime, hoc-tap]
created: 2026-10-08
---

# 04 - Stream, Exception, I/O và Date-Time

⬅️ [[03 - Collections và Generics chuyên sâu]] | [[00 - Lộ trình Senior 1|Mục lục]] | [[05 - Concurrency và Multithreading]] ➡️

## 1. Lambda và Functional Interface

```java
Function<String, Integer> len = String::length;        // T -> R
Predicate<String> empty = String::isEmpty;             // T -> boolean
Consumer<String> print = System.out::println;          // T -> void
Supplier<LocalDate> today = LocalDate::now;            // () -> T
BiFunction<Integer, Integer, Integer> add = Integer::sum;
UnaryOperator<String> up = String::toUpperCase;
```

- Lambda chỉ dùng được biến cục bộ **effectively final**.
- Method reference có 4 dạng: `Class::staticMethod`, `obj::method`, `Class::instanceMethod`, `Class::new`.
- Tự tạo functional interface bằng `@FunctionalInterface` khi cần.

## 2. Stream API

Stream **không lưu trữ** dữ liệu, chỉ là một đường ống tính toán **lười (lazy)**: các thao tác trung gian chỉ chạy khi gặp thao tác kết thúc.

```java
List<String> result = orders.stream()
    .filter(o -> o.status() == Status.PAID)       // trung gian
    .map(Order::customerName)                      // trung gian
    .distinct()
    .sorted()
    .limit(10)
    .toList();                                     // kết thúc
```

### Các thao tác hay dùng

| Nhóm | Thao tác |
|------|----------|
| Trung gian | `filter`, `map`, `flatMap`, `distinct`, `sorted`, `limit`, `skip`, `peek` |
| Kết thúc | `collect`, `toList`, `forEach`, `reduce`, `count`, `findFirst`, `anyMatch`, `allMatch` |
| Số học | `mapToInt`, `sum`, `average`, `max`, `summaryStatistics` |

### Collectors

```java
// Nhóm theo
Map<String, List<Employee>> byDept =
    emps.stream().collect(Collectors.groupingBy(Employee::dept));

// Nhóm rồi tính toán
Map<String, Double> avgSalary =
    emps.stream().collect(Collectors.groupingBy(Employee::dept,
                          Collectors.averagingDouble(Employee::salary)));

// Chuyển thành Map; PHẢI xử lý trùng key
Map<Long, Employee> byId = emps.stream()
    .collect(Collectors.toMap(Employee::id, e -> e, (a, b) -> a));

// Nối chuỗi
String names = emps.stream().map(Employee::name)
    .collect(Collectors.joining(", ", "[", "]"));

// flatMap: làm phẳng
List<String> allTags = posts.stream()
    .flatMap(p -> p.tags().stream())
    .toList();
```

### Bẫy với Stream

> [!warning]
> - Không dùng stream cho thao tác có **tác dụng phụ** (sửa biến bên ngoài).
> - `Collectors.toMap` ném `IllegalStateException` khi trùng key nếu không có merge function.
> - Stream **chỉ dùng được một lần**.
> - `parallelStream()` dùng chung `ForkJoinPool.commonPool()`: dễ gây nghẽn trong ứng dụng web. Chỉ dùng khi đã đo và công việc thuần CPU, dữ liệu lớn.
> - Đừng viết chuỗi stream quá dài, khó đọc hơn vòng `for` rõ ràng thì dùng `for`.

## 3. Optional

```java
Optional<User> user = repo.findByEmail(email);

String name = user.map(User::name).orElse("Ẩn danh");
User u = user.orElseThrow(() -> new NotFoundException("Không có user"));
user.ifPresentOrElse(this::welcome, this::askSignup);
```

Nguyên tắc:
- Dùng làm **kiểu trả về**. Không dùng làm field, tham số, hay phần tử collection.
- Không gọi `get()` mà không kiểm tra; ưu tiên `orElseThrow`.
- `orElse(x)` luôn tính `x`; dùng `orElseGet(() -> ...)` nếu tính tốn kém.

## 4. Exception: thiết kế đúng

```java
// Giữ nguyên nguyên nhân gốc (cause)
try {
    repo.save(order);
} catch (DataAccessException e) {
    throw new OrderPersistenceException("Lưu đơn thất bại: " + order.id(), e);
}
```

### Quy tắc vàng
1. **Không** `catch (Exception e) {}` rỗng (nuốt lỗi). Ít nhất phải log kèm stack trace.
2. **Không** bắt `Throwable`/`Error`.
3. **Bắt ở nơi có thể xử lý được**; nếu không, để lan lên.
4. **Log một lần** ở biên (ví dụ `@RestControllerAdvice`), tránh log lặp ở mọi tầng.
5. Dùng exception cho tình huống **bất thường**, không dùng làm luồng điều khiển.
6. Thông điệp lỗi hữu ích: nói **cái gì**, **với dữ liệu nào** (nhưng không lộ thông tin nhạy cảm).
7. Checked hay unchecked: nghiệp vụ trong Spring thường dùng **unchecked** có phân cấp rõ ràng.

```java
public abstract class DomainException extends RuntimeException {
    private final String code;
    protected DomainException(String code, String msg) { super(msg); this.code = code; }
    public String code() { return code; }
}
public class OrderNotFoundException extends DomainException {
    public OrderNotFoundException(long id) { super("ORDER_NOT_FOUND", "Không tìm thấy đơn " + id); }
}
```

### try-with-resources và suppressed

```java
try (var in = Files.newBufferedReader(path);
     var out = Files.newBufferedWriter(target)) {
    in.transferTo(out);
}   // đóng theo thứ tự ngược; lỗi khi đóng được gắn vào getSuppressed()
```

Xử lý lỗi ở tầng web: xem [[09 - Spring MVC và thiết kế REST API]].

## 5. I/O hiện đại (NIO.2)

```java
Path p = Path.of("data", "users.csv");

String all = Files.readString(p);                      // file nhỏ
List<String> lines = Files.readAllLines(p);
try (Stream<String> s = Files.lines(p)) {              // file lớn: đọc lười, nhớ đóng
    s.filter(l -> l.contains("ERROR")).forEach(System.out::println);
}

Files.writeString(Path.of("out.txt"), "xin chào",
        StandardCharsets.UTF_8, StandardOpenOption.CREATE, StandardOpenOption.APPEND);

try (var walk = Files.walk(Path.of("src"))) {          // duyệt cây thư mục
    walk.filter(f -> f.toString().endsWith(".java")).forEach(System.out::println);
}
```

Lưu ý:
- **Luôn chỉ định charset** (`UTF-8`); đừng phụ thuộc mặc định của hệ thống.
- Với file khổng lồ, đọc theo dòng/khối, không `readAllBytes`.
- `Files.lines`, `Files.walk` trả về stream cần đóng.
- Phân biệt: **byte stream** (`InputStream`) và **character stream** (`Reader`).

## 6. Ngày giờ (`java.time`)

| Lớp | Dùng cho |
|-----|----------|
| `Instant` | Một thời điểm trên trục thời gian (UTC) |
| `LocalDate` / `LocalDateTime` | Ngày/giờ **không** múi giờ (ngày sinh, lịch hẹn cục bộ) |
| `ZonedDateTime` / `OffsetDateTime` | Có múi giờ/độ lệch |
| `Duration` / `Period` | Khoảng thời gian (giờ-phút / ngày-tháng) |

```java
Instant now = Instant.now();
ZonedDateTime vn = now.atZone(ZoneId.of("Asia/Ho_Chi_Minh"));
LocalDate d = LocalDate.of(2026, 10, 8);
long days = ChronoUnit.DAYS.between(d, d.plusMonths(1));

DateTimeFormatter f = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm");
String s = vn.format(f);
```

> [!tip] Quy tắc backend
> **Lưu UTC (`Instant`/`timestamptz`), hiển thị theo múi giờ người dùng.** Truyền qua API bằng ISO-8601 (`2026-10-08T03:00:00Z`). Trong code, inject `Clock` để test được thời gian.

```java
@Service
class TokenService {
    private final Clock clock;
    TokenService(Clock clock) { this.clock = clock; }
    boolean expired(Instant exp) { return Instant.now(clock).isAfter(exp); }
}
```

## 7. Tiền và số

- `BigDecimal` cho tiền: chỉ định `scale` và `RoundingMode` rõ ràng.
- So sánh `BigDecimal` bằng `compareTo`, không dùng `equals` (vì `2.0` khác `2.00`).
- Lưu tiền dưới dạng số nguyên đơn vị nhỏ nhất (cent, đồng) cũng là cách phổ biến.

```java
BigDecimal price = new BigDecimal("19.99");
BigDecimal total = price.multiply(BigDecimal.valueOf(3)).setScale(2, RoundingMode.HALF_UP);
```

> [!question] Tự kiểm tra
> 1. Stream lười nghĩa là gì? `peek` trong stream không có thao tác kết thúc thì chạy không?
> 2. Vì sao `parallelStream` có thể gây hại trong ứng dụng web?
> 3. Khi nào dùng `Instant`, khi nào dùng `LocalDateTime`?
> 4. Vì sao không nên `catch (Exception e)` rồi bỏ qua?

⬅️ [[03 - Collections và Generics chuyên sâu]] | [[00 - Lộ trình Senior 1|Mục lục]] | [[05 - Concurrency và Multithreading]] ➡️
