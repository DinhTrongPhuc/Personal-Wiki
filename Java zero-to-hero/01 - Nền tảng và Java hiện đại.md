---
title: Nền tảng và Java hiện đại
aliases: [Java Fundamentals, Java 8-21, Modern Java]
tags: [java, core, hoc-tap]
created: 2026-10-08
---

# 01 - Nền tảng và Java hiện đại

[[00 - Lộ trình Senior 1|Mục lục]] | [[02 - OOP, SOLID và Design Pattern]] ➡️

## 1. Hệ thống kiểu

### Primitive và Wrapper

| Primitive | Wrapper | Ghi chú |
|-----------|---------|---------|
| `int` | `Integer` | Wrapper có thể là `null` |
| `long` | `Long` | |
| `double` | `Double` | |
| `boolean` | `Boolean` | |

**Autoboxing** là việc tự chuyển qua lại giữa hai dạng, nhưng có bẫy:

```java
Integer a = 127, b = 127;
System.out.println(a == b);   // true  (cache -128..127)

Integer c = 128, d = 128;
System.out.println(c == d);   // false (đối tượng khác nhau)
System.out.println(c.equals(d)); // true

Integer x = null;
int y = x;                    // NullPointerException khi unboxing
```

> [!warning] Quy tắc
> So sánh wrapper và đối tượng bằng `equals()`. Tiền tệ dùng `BigDecimal`, **không** dùng `double`.

```java
System.out.println(0.1 + 0.2);                              // 0.30000000000000004
System.out.println(new BigDecimal("0.1").add(new BigDecimal("0.2"))); // 0.3
// Luôn tạo BigDecimal từ String, không từ double
```

### String

- `String` **bất biến** và được lưu trong **String Pool**.
- `"abc" == "abc"` là `true` (cùng pool), nhưng `new String("abc") == "abc"` là `false`.
- Nối chuỗi trong vòng lặp: dùng `StringBuilder` (không an toàn đa luồng) hoặc `StringBuffer` (đồng bộ, hiếm khi cần).

### Truyền tham số

Java **luôn truyền theo giá trị**. Với đối tượng, giá trị được truyền là *bản sao của tham chiếu*.

```java
void change(List<String> list, String s) {
    list.add("x");   // ảnh hưởng đối tượng gốc
    s = "new";       // chỉ đổi bản sao tham chiếu, bên ngoài không đổi
}
```

## 2. Khởi tạo đối tượng

Thứ tự khi `new Child()`:

1. Khởi tạo `static` của lớp cha, rồi lớp con (chỉ lần đầu nạp lớp)
2. Khối `{ }` và field của lớp cha, rồi constructor lớp cha
3. Khối `{ }` và field của lớp con, rồi constructor lớp con

> [!danger] Bẫy kinh điển
> Gọi phương thức bị override từ constructor của lớp cha có thể chạy khi field lớp con **chưa** được khởi tạo.

## 3. Bất biến (Immutability)

Đối tượng bất biến an toàn đa luồng, dễ suy luận, dễ cache. Quy tắc tạo lớp bất biến:

```java
public final class Money {
    private final BigDecimal amount;
    private final String currency;
    private final List<String> tags;

    public Money(BigDecimal amount, String currency, List<String> tags) {
        this.amount = amount;
        this.currency = currency;
        this.tags = List.copyOf(tags);          // sao chép phòng vệ
    }
    public List<String> tags() { return tags; } // List.copyOf đã bất biến
}
```

Từ Java 16 dùng `record` cho gọn (xem bên dưới).

## 4. Các tính năng Java hiện đại

| Phiên bản | Tính năng đáng nhớ |
|-----------|--------------------|
| 8 | Lambda, Stream, `Optional`, `java.time`, default method |
| 9 | Module (JPMS), `List.of`, `Map.of` |
| 10 | `var` |
| 11 (LTS) | `HttpClient`, `String.isBlank/strip/lines/repeat` |
| 14 | Switch expression |
| 15 | Text block |
| 16 | Record, pattern matching cho `instanceof` |
| 17 (LTS) | Sealed class |
| 21 (LTS) | Virtual thread, pattern matching cho `switch`, record pattern, sequenced collection |

### Record

```java
public record Point(int x, int y) {
    public Point {                       // compact constructor để kiểm tra
        if (x < 0 || y < 0) throw new IllegalArgumentException("Âm");
    }
    public double distance() { return Math.sqrt(x * x + y * y); }
}
```

Dùng record cho **DTO, value object, kết quả trả về**. Không dùng làm JPA entity.

### Sealed và pattern matching

```java
public sealed interface Shape permits Circle, Rect {}
public record Circle(double r) implements Shape {}
public record Rect(double w, double h) implements Shape {}

static double area(Shape s) {
    return switch (s) {                  // trình biên dịch kiểm tra đủ trường hợp
        case Circle c -> Math.PI * c.r() * c.r();
        case Rect(double w, double h) -> w * h;   // record pattern (Java 21)
    };
}

Object o = "hello";
if (o instanceof String str && !str.isBlank()) {
    System.out.println(str.length());
}
```

### Switch expression và text block

```java
int days = switch (month) {
    case 4, 6, 9, 11 -> 30;
    case 2 -> isLeap ? 29 : 28;
    default -> 31;
};

String sql = """
    SELECT id, name
    FROM users
    WHERE active = true
    """;
```

### Sequenced collection (Java 21)

```java
List<Integer> list = new ArrayList<>(List.of(1, 2, 3));
list.getFirst();      // 1
list.getLast();       // 3
list.reversed();      // [3, 2, 1] dạng view
```

## 5. Hợp đồng `equals` và `hashCode`

Quy tắc bắt buộc:
1. Nếu `a.equals(b)` thì `a.hashCode() == b.hashCode()`.
2. `equals` phải phản xạ, đối xứng, bắc cầu, nhất quán.
3. Đừng dùng field **có thể thay đổi** làm cơ sở cho `hashCode` của đối tượng nằm trong `HashSet` hoặc làm key `HashMap`.

Với JPA entity, đây là chủ đề tế nhị: xem [[10 - JPA, Hibernate, SQL và Transaction]].

## 6. Những bẫy hay gặp

- [ ] `==` so với `equals` trên `String` và wrapper
- [ ] `NullPointerException`: dùng `Objects.requireNonNull`, `Optional` ở giá trị trả về
- [ ] Chia số nguyên: `1 / 2 == 0`
- [ ] Tràn số `int`: dùng `Math.addExact` để phát hiện
- [ ] Sửa `List` khi đang duyệt: `ConcurrentModificationException`
- [ ] Biến `static` mutable là trạng thái toàn cục ẩn

> [!question] Tự kiểm tra
> 1. Vì sao `Integer 127 == 127` là `true` còn `128` thì không?
> 2. Khi nào nên dùng `record`, khi nào không?
> 3. Thứ tự khởi tạo khi tạo đối tượng lớp con là gì?
> 4. Vì sao không dùng `double` cho tiền?

[[00 - Lộ trình Senior 1|Mục lục]] | [[02 - OOP, SOLID và Design Pattern]] ➡️
