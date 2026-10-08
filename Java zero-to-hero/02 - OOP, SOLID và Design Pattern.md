---
title: OOP, SOLID và Design Pattern
aliases: [OOP, SOLID, Design Patterns]
tags: [java, oop, solid, design-pattern, hoc-tap]
created: 2026-10-08
---

# 02 - OOP, SOLID và Design Pattern

⬅️ [[01 - Nền tảng và Java hiện đại]] | [[00 - Lộ trình Senior 1|Mục lục]] | [[03 - Collections và Generics chuyên sâu]] ➡️

## 1. Bốn trụ cột

| Trụ cột | Ý chính | Công cụ trong Java |
|---------|---------|--------------------|
| Đóng gói | Giấu trạng thái, chỉ lộ hành vi | `private`, getter có kiểm soát |
| Kế thừa | Tái sử dụng quan hệ "là một" | `extends` |
| Đa hình | Cùng giao diện, nhiều cách cài đặt | `interface`, override |
| Trừu tượng | Che chi tiết, lộ hợp đồng | `abstract`, `interface` |

> [!important] Ưu tiên composition hơn inheritance
> Kế thừa gắn chặt lớp con với chi tiết lớp cha (vấn đề *fragile base class*). Hãy hỏi: "có phải **là một** hay chỉ cần **có một**?" Phần lớn trường hợp, chọn *có một* (composition).

```java
// Composition: linh hoạt, dễ test
class OrderService {
    private final PaymentGateway gateway;      // có một
    OrderService(PaymentGateway gateway) { this.gateway = gateway; }
}
```

## 2. SOLID

### S: Single Responsibility
Một lớp chỉ có **một lý do để thay đổi**.

```java
// Sai: vừa nghiệp vụ, vừa lưu DB, vừa gửi email
class UserService { void register(User u) { /* validate + save + email */ } }

// Đúng: tách trách nhiệm
class UserRegistration {
    UserRegistration(UserRepository repo, EmailSender mail, UserValidator v) { }
}
```

### O: Open/Closed
Mở để mở rộng, đóng để sửa đổi: thêm hành vi mới bằng cách **thêm lớp**, không sửa lớp cũ.

```java
interface DiscountPolicy { BigDecimal apply(BigDecimal price); }
class VipDiscount implements DiscountPolicy { /* ... */ }
class SeasonalDiscount implements DiscountPolicy { /* ... */ }
```

### L: Liskov Substitution
Lớp con phải thay thế được lớp cha mà không làm sai hành vi. Ví dụ `Square extends Rectangle` vi phạm LSP vì đặt chiều rộng làm đổi chiều cao.

### I: Interface Segregation
Nhiều interface nhỏ, chuyên biệt tốt hơn một interface to. Đừng ép client phụ thuộc vào phương thức nó không dùng.

### D: Dependency Inversion
Module cấp cao **không** phụ thuộc cài đặt cụ thể, cả hai cùng phụ thuộc **abstraction**. Đây là nền tảng của Dependency Injection trong [[07 - Spring Core, IoC và AOP]].

## 3. Nguyên tắc khác nên nhớ

- **DRY**: đừng lặp tri thức (nhưng đừng trừu tượng hóa quá sớm)
- **KISS** và **YAGNI**: giữ đơn giản, đừng làm thứ chưa cần
- **Law of Demeter**: chỉ nói chuyện với "bạn bè gần", tránh `a.getB().getC().doX()`
- **Tell, don't ask**: bảo đối tượng làm việc, đừng hỏi trạng thái rồi tự quyết định thay nó
- **Fail fast**: kiểm tra đầu vào sớm, lỗi rõ ràng

## 4. Design Pattern thường dùng

### Creational

**Singleton**: cách an toàn nhất trong Java là `enum`.

```java
public enum AppConfig {
    INSTANCE;
    public String get(String key) { return System.getProperty(key); }
}
```

> [!note]
> Trong Spring, bean mặc định đã là singleton do container quản lý. Bạn hiếm khi tự viết Singleton.

**Builder**: dựng đối tượng phức tạp nhiều tham số tùy chọn.

```java
public class HttpRequest {
    private final String url; private final String method; private final int timeout;
    private HttpRequest(Builder b) { url = b.url; method = b.method; timeout = b.timeout; }

    public static Builder builder() { return new Builder(); }
    public static class Builder {
        private String url; private String method = "GET"; private int timeout = 30;
        public Builder url(String u) { this.url = u; return this; }
        public Builder method(String m) { this.method = m; return this; }
        public Builder timeout(int t) { this.timeout = t; return this; }
        public HttpRequest build() { return new HttpRequest(this); }
    }
}
```

**Factory Method / Abstract Factory**: ẩn việc chọn lớp cụ thể.

```java
static Notifier of(String type) {
    return switch (type) {
        case "EMAIL" -> new EmailNotifier();
        case "SMS" -> new SmsNotifier();
        default -> throw new IllegalArgumentException(type);
    };
}
```

### Structural

**Adapter**: chuyển interface này thành interface khác mà client mong đợi.
**Decorator**: bọc đối tượng để thêm hành vi (như `BufferedReader` bọc `FileReader`).
**Proxy**: đại diện kiểm soát truy cập. Spring AOP và `@Transactional` dùng proxy (xem [[07 - Spring Core, IoC và AOP]]).
**Facade**: một cửa đơn giản che hệ thống con phức tạp.

### Behavioral

**Strategy**: đổi thuật toán lúc chạy. Với lambda rất gọn.

```java
Map<String, BiFunction<BigDecimal, BigDecimal, BigDecimal>> ops = Map.of(
    "ADD", BigDecimal::add,
    "SUB", BigDecimal::subtract);
BigDecimal r = ops.get("ADD").apply(a, b);
```

**Observer**: thông báo nhiều người nghe khi có sự kiện. Spring có `ApplicationEventPublisher`.
**Template Method**: khung thuật toán cố định, bước con để lớp con điền (như `JdbcTemplate`).
**Chain of Responsibility**: chuỗi xử lý (như Servlet Filter, Spring Security filter chain).

## 5. Pattern trong hệ sinh thái Spring

| Pattern | Xuất hiện ở |
|---------|-------------|
| Dependency Injection | Toàn bộ container |
| Proxy | AOP, `@Transactional`, `@Cacheable` |
| Template Method | `JdbcTemplate`, `RestTemplate` |
| Factory | `BeanFactory`, `FactoryBean` |
| Chain of Responsibility | Filter, Interceptor |
| Observer | `ApplicationEvent` |
| Strategy | `PasswordEncoder`, `AuthenticationProvider` |

> [!warning] Đừng dùng pattern vì pattern
> Pattern là từ vựng chung để trao đổi, không phải mục tiêu. Code đơn giản giải quyết được vấn đề thì đừng thêm tầng trừu tượng.

> [!question] Tự kiểm tra
> 1. Cho ví dụ vi phạm Liskov mà không phải Square/Rectangle.
> 2. Vì sao composition thường tốt hơn inheritance?
> 3. `@Transactional` hoạt động nhờ pattern nào, và hệ quả khi tự gọi trong cùng lớp?
> 4. Khi nào Builder tốt hơn constructor?

⬅️ [[01 - Nền tảng và Java hiện đại]] | [[00 - Lộ trình Senior 1|Mục lục]] | [[03 - Collections và Generics chuyên sâu]] ➡️
