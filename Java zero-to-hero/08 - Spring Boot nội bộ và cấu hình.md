---
title: Spring Boot nội bộ và cấu hình
aliases: [Spring Boot, Auto-configuration, Starter, Actuator, Profiles]
tags: [spring-boot, autoconfig, config, senior, hoc-tap]
created: 2026-10-08
---

# 08 - Spring Boot nội bộ và cấu hình

⬅️ [[07 - Spring Core, IoC và AOP]] | [[00 - Lộ trình Senior 1|Mục lục]] | [[09 - Spring MVC và thiết kế REST API]] ➡️

## 1. Spring Boot làm gì cho bạn

**Spring Boot = Spring Framework + auto-configuration + starter + server nhúng + cấu hình ngoài (externalized config) + Actuator.**

Triết lý: *opinionated defaults*: có mặc định hợp lý, bạn chỉ ghi đè cái cần khác.

## 2. Quá trình khởi động

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

`@SpringBootApplication` = `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`.

Các bước chính của `SpringApplication.run`:
1. Xác định loại ứng dụng (servlet, reactive, không web)
2. Nạp **Environment** (properties, biến môi trường, tham số dòng lệnh, profile)
3. Tạo `ApplicationContext`
4. Chạy **auto-configuration** và component scan, tạo bean
5. Khởi động **web server nhúng** (Tomcat mặc định)
6. Gọi `CommandLineRunner`/`ApplicationRunner`; phát sự kiện `ApplicationReadyEvent`

> [!tip]
> Đặt lớp main ở **package gốc** để component scan thấy toàn bộ code bên dưới.

## 3. Auto-configuration

Cơ chế: Boot đọc danh sách lớp cấu hình trong file
`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
của từng thư viện, rồi chỉ **bật khi điều kiện thỏa**.

```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean(DataSource.class)       // bạn tự khai báo thì Boot lùi ra
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration { }
```

| Annotation điều kiện | Bật khi |
|---------------------|---------|
| `@ConditionalOnClass` | Lớp có trên classpath |
| `@ConditionalOnMissingBean` | Chưa có bean kiểu đó (nhường quyền cho bạn) |
| `@ConditionalOnProperty` | Property có giá trị chỉ định |
| `@ConditionalOnWebApplication` | Là ứng dụng web |
| `@ConditionalOnBean` | Đã có bean kiểu đó |

### Xem Boot đã bật/tắt gì

```yaml
debug: true          # in "Conditions Evaluation Report" khi khởi động
```

hoặc Actuator: `/actuator/conditions`, `/actuator/beans`, `/actuator/configprops`.

### Tự viết một starter (để hiểu cơ chế)

1. Module `my-lib` chứa logic
2. Module `my-lib-spring-boot-autoconfigure` chứa `@AutoConfiguration` + `@ConfigurationProperties`
3. Khai báo trong file `AutoConfiguration.imports`
4. Module `my-lib-spring-boot-starter` chỉ gom dependency

## 4. Cấu hình ngoài (Externalized Configuration)

### Thứ tự ưu tiên (cao thắng thấp, rút gọn)

1. Tham số dòng lệnh (`--server.port=9000`)
2. Biến môi trường (`SERVER_PORT=9000`)
3. `application-{profile}.yml`
4. `application.yml`
5. Giá trị mặc định trong code

```yaml
# application.yml
server:
  port: 8080
  shutdown: graceful              # đóng êm: chờ request đang xử lý

spring:
  application:
    name: order-service
  lifecycle:
    timeout-per-shutdown-phase: 30s
  datasource:
    url: jdbc:postgresql://localhost:5432/shop
    username: ${DB_USER}
    password: ${DB_PASSWORD}      # từ biến môi trường, KHÔNG commit mật khẩu
    hikari:
      maximum-pool-size: 20
      connection-timeout: 3000
  jpa:
    open-in-view: false           # nên tắt, xem [[10 - JPA, Hibernate, SQL và Transaction]]
    hibernate:
      ddl-auto: validate

app:
  payment:
    base-url: https://pay.example.com
    timeout: 3s
```

### Gắn cấu hình an toàn kiểu (type-safe)

```java
@ConfigurationProperties(prefix = "app.payment")
@Validated
public record PaymentProperties(
        @NotBlank String baseUrl,
        @DefaultValue("3s") Duration timeout) {}

@SpringBootApplication
@ConfigurationPropertiesScan
class App {}
```

> [!note]
> Ưu tiên `@ConfigurationProperties` hơn rải rác `@Value`: có kiểm tra hợp lệ lúc khởi động, hỗ trợ `Duration`, `DataSize`, danh sách, đối tượng lồng nhau.

### Profile

```bash
java -jar app.jar --spring.profiles.active=prod
```

```yaml
# application-prod.yml chỉ ghi phần khác biệt
logging:
  level:
    root: WARN
```

Đừng để mật khẩu/bí mật trong file cấu hình: dùng biến môi trường, **Kubernetes Secret**, **Vault**, **AWS Secrets Manager**, hoặc **Spring Cloud Config**.

## 5. Quản lý phụ thuộc

- `spring-boot-starter-parent` hoặc BOM `spring-boot-dependencies` quản lý **phiên bản đồng bộ** của hàng trăm thư viện.
- Ghi đè phiên bản có chủ đích bằng property (ví dụ `<jackson-bom.version>`), nhưng cẩn thận xung đột.
- Xem cây phụ thuộc để xử lý xung đột: `mvn dependency:tree`.
- Loại starter không cần: `<exclusions>`; đổi Tomcat sang Jetty/Undertow nếu cần.

## 6. Actuator

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      probes:
        enabled: true             # /actuator/health/liveness và /readiness cho K8s
      show-details: when-authorized
```

| Endpoint | Công dụng |
|----------|-----------|
| `/actuator/health` | Sức khỏe, kèm liveness/readiness |
| `/actuator/metrics` | Metric (JVM, HTTP, DB pool) |
| `/actuator/prometheus` | Xuất metric cho Prometheus |
| `/actuator/loggers` | Đổi mức log lúc chạy |
| `/actuator/env`, `/beans`, `/mappings` | Chẩn đoán cấu hình |

> [!danger] Bảo mật
> Chỉ phơi bày endpoint cần thiết, đặt sau xác thực hoặc cổng quản trị riêng (`management.server.port`). `env`, `heapdump` có thể lộ thông tin nhạy cảm.

Chi tiết quan sát hệ thống: [[15 - Observability và Production]].

## 7. Logging

Mặc định Logback qua SLF4J.

```java
private static final Logger log = LoggerFactory.getLogger(OrderService.class);

log.info("Tạo đơn {} cho user {}", orderId, userId);      // dùng placeholder, không nối chuỗi
log.error("Thanh toán lỗi, orderId={}", orderId, ex);     // exception là tham số cuối
```

Thực hành tốt:
- Mức log đúng: `ERROR` (cần người xử lý), `WARN` (bất thường nhưng tự phục hồi), `INFO` (sự kiện nghiệp vụ chính), `DEBUG/TRACE` (chẩn đoán)
- **Log có cấu trúc (JSON)** ở production; Spring Boot 3.4+ hỗ trợ sẵn `logging.structured.format.console`
- Gắn **correlation/trace ID** vào mọi dòng log (MDC)
- **Không log** mật khẩu, token, số thẻ, dữ liệu cá nhân

## 8. Các tính năng đáng biết

- **Graceful shutdown** (`server.shutdown=graceful`): không cắt ngang request
- **Devtools:** tự restart khi code đổi (chỉ dev)
- **Testcontainers + `@ServiceConnection`** (Boot 3.1+): tự nối dịch vụ test
- **Docker Compose support** (Boot 3.1+): `spring-boot-docker-compose` tự khởi động dịch vụ khi chạy dev
- **Virtual thread:** `spring.threads.virtual.enabled=true` (xem [[05 - Concurrency và Multithreading]])
- **Buildpacks:** `./mvnw spring-boot:build-image` tạo image Docker không cần Dockerfile
- **GraalVM native image** và **AOT**: khởi động nhanh, ít RAM (đánh đổi thời gian build và hạn chế reflection)
- **Layered jar:** tối ưu cache lớp Docker

## 9. Lỗi khởi động thường gặp

| Triệu chứng | Nguyên nhân thường gặp |
|-------------|-----------------------|
| `NoSuchBeanDefinitionException` | Bean ngoài phạm vi component scan, thiếu annotation, sai profile |
| `NoUniqueBeanDefinitionException` | Nhiều bean cùng kiểu: dùng `@Primary`/`@Qualifier` |
| `APPLICATION FAILED TO START` + dependency vòng | Circular dependency: thiết kế lại |
| `Failed to configure a DataSource` | Thiếu URL/driver |
| `Port already in use` | Cổng bị chiếm |
| Property không nạp | Sai prefix/relaxed binding, sai profile, YAML thụt lề sai |

> [!question] Tự kiểm tra
> 1. Auto-configuration biết nên bật cấu hình nào nhờ cơ chế gì? Làm sao ghi đè nó?
> 2. Liệt kê thứ tự ưu tiên của cấu hình trong Spring Boot.
> 3. Vì sao nên dùng `@ConfigurationProperties` thay `@Value`?
> 4. Liveness và readiness khác nhau thế nào, vì sao Kubernetes cần cả hai?

⬅️ [[07 - Spring Core, IoC và AOP]] | [[00 - Lộ trình Senior 1|Mục lục]] | [[09 - Spring MVC và thiết kế REST API]] ➡️
