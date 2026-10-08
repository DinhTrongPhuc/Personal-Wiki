---
title: Caching, Messaging và Resilience
aliases: [Cache, Redis, Kafka, RabbitMQ, Resilience4j, Circuit Breaker]
tags: [cache, kafka, messaging, resilience, senior, hoc-tap]
created: 2026-10-08
---

# 13 - Caching, Messaging và Resilience

⬅️ [[12 - Testing chuyên sâu]] | [[00 - Lộ trình Senior|Mục lục]] | [[14 - Microservices và Distributed Patterns]] ➡️

## Phần A: Caching

> "Có hai việc khó trong khoa học máy tính: vô hiệu hóa cache và đặt tên." Cache giải quyết hiệu năng nhưng **thêm độ phức tạp về tính nhất quán**. Chỉ thêm khi đã đo được nút thắt.

### 1. Các tầng cache

| Tầng | Ví dụ | Đặc điểm |
|------|-------|----------|
| Trình duyệt/CDN | `Cache-Control`, ETag | Giảm tải server nhiều nhất |
| In-process (local) | Caffeine | Nhanh nhất, nhưng **mỗi instance một bản**, không đồng bộ |
| Phân tán | Redis, Memcached | Dùng chung giữa các instance, có độ trễ mạng |
| DB | Buffer pool, materialized view | |

### 2. Spring Cache abstraction

```java
@EnableCaching
@Configuration class CacheConfig {}

@Service
class ProductService {
    @Cacheable(cacheNames = "products", key = "#id", unless = "#result == null")
    public Product get(long id) { return repo.findById(id).orElseThrow(); }

    @CachePut(cacheNames = "products", key = "#result.id")
    public Product update(Product p) { return repo.save(p); }

    @CacheEvict(cacheNames = "products", key = "#id")
    public void delete(long id) { repo.deleteById(id); }
}
```

Nhớ rằng đây là AOP: bẫy **self-invocation** vẫn áp dụng ([[07 - Spring Core, IoC và AOP]]).

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 10m          # LUÔN đặt TTL
```

### 3. Chiến lược cache

| Chiến lược | Cách hoạt động | Ghi chú |
|-----------|----------------|---------|
| **Cache-aside** (phổ biến) | App đọc cache, miss thì đọc DB rồi ghi cache | Đơn giản, dữ liệu có thể cũ |
| Read-through | Cache tự nạp khi miss | |
| Write-through | Ghi cache và DB đồng thời | Nhất quán hơn, ghi chậm hơn |
| Write-behind | Ghi cache trước, DB sau (bất đồng bộ) | Nhanh nhưng có nguy cơ mất dữ liệu |

### 4. Các sự cố kinh điển

| Sự cố | Mô tả | Cách chống |
|-------|-------|-----------|
| **Cache stampede** | Khóa nóng hết hạn, hàng nghìn request cùng đổ xuống DB | Khóa khi nạp lại (`sync = true`), TTL ngẫu nhiên (jitter), làm mới sớm |
| **Cache penetration** | Truy vấn key không tồn tại liên tục xuyên qua cache | Cache giá trị rỗng (TTL ngắn), Bloom filter |
| **Cache avalanche** | Nhiều key hết hạn cùng lúc | TTL có jitter, chia nhỏ |
| **Dữ liệu cũ (stale)** | Cache lệch DB | TTL ngắn, evict khi ghi, sự kiện thay đổi |
| **Hot key** | Một key quá nóng | Cache local thêm một tầng, nhân bản key |

> [!warning]
> Tránh cache dữ liệu theo người dùng/quyền mà quên đưa người dùng vào key (rò rỉ dữ liệu giữa người dùng). Đừng cache thứ cần nhất quán mạnh (số dư, tồn kho cuối).

## Phần B: Messaging và hướng sự kiện

### 5. Vì sao dùng message broker

- **Tách rời** thời gian và thành phần (producer không cần consumer đang chạy)
- **Làm phẳng đỉnh tải** (buffer), xử lý bất đồng bộ
- Phát sự kiện cho nhiều người nghe (fan-out)

Đánh đổi: nhất quán cuối cùng (eventual consistency), khó debug, cần xử lý trùng lặp và sai thứ tự.

### 6. Kafka vs RabbitMQ

| | **Kafka** | **RabbitMQ** |
|--|-----------|--------------|
| Mô hình | Log phân tán, bền, **replay được** | Queue/exchange, broker đẩy message |
| Thông lượng | Rất cao | Cao vừa phải |
| Thứ tự | Đảm bảo **trong một partition** | Trong một queue (có điều kiện) |
| Dùng cho | Event streaming, event sourcing, pipeline dữ liệu | Tác vụ nền, định tuyến linh hoạt, RPC |
| Tiêu thụ | Consumer tự theo dõi **offset** | Ack/nack từng message |

### 7. Kafka cốt lõi

- **Topic** chia thành **partition**; message cùng **key** đi vào cùng partition (giữ thứ tự theo key)
- **Consumer group**: mỗi partition được đúng một consumer trong nhóm xử lý → song song hóa tối đa = số partition
- **Offset** commit sau khi xử lý xong; **rebalance** khi consumer thêm/bớt

```java
@Service
class OrderEvents {
    private final KafkaTemplate<String, OrderPlaced> kafka;
    OrderEvents(KafkaTemplate<String, OrderPlaced> kafka) { this.kafka = kafka; }

    void publish(OrderPlaced e) { kafka.send("orders.placed", String.valueOf(e.customerId()), e); }
}

@Component
class Billing {
    @KafkaListener(topics = "orders.placed", groupId = "billing")
    void on(OrderPlaced e) { /* xử lý IDEMPOTENT */ }
}
```

```yaml
spring:
  kafka:
    producer:
      acks: all                       # đợi đủ replica đồng bộ
      properties:
        enable.idempotence: true
    consumer:
      enable-auto-commit: false
      auto-offset-reset: earliest
```

### 8. Đảm bảo giao nhận

| Mức | Ý nghĩa | Thực tế |
|-----|---------|---------|
| At-most-once | Có thể mất, không trùng | Hiếm dùng |
| **At-least-once** | Không mất, **có thể trùng** | Phổ biến nhất |
| Exactly-once | Chính xác một lần | Chỉ đạt "hiệu quả" khi kết hợp idempotent consumer hoặc transaction Kafka trong phạm vi hệ thống |

> [!important] Quy tắc
> Giả định message **có thể đến trùng và sai thứ tự**. Consumer phải **idempotent**: lưu `messageId` đã xử lý (unique constraint) hoặc thao tác có tính chất đặt lại (upsert).

### 9. Xử lý lỗi consumer

- **Retry** có backoff (không vòng lặp vô hạn chặn partition)
- **Dead Letter Topic/Queue (DLT/DLQ)** cho message "độc" (poison message), có cảnh báo và quy trình xử lý lại
- Phân biệt lỗi **tạm thời** (retry) và **vĩnh viễn** (đẩy DLT ngay)
- Theo dõi **consumer lag**: chỉ số sức khỏe số một

### 10. Transactional Outbox (tránh ghi kép)

Vấn đề: vừa cập nhật DB vừa gửi message → một trong hai có thể thất bại. Chi tiết ở [[14 - Microservices và Distributed Patterns]].

## Phần C: Resilience (khả năng chịu lỗi)

Dịch vụ phụ thuộc **sẽ** chậm hoặc hỏng. Mục tiêu: lỗi cục bộ không lan thành sự cố toàn hệ thống.

### 11. Các mẫu cốt lõi (Resilience4j)

| Mẫu | Mục đích |
|-----|----------|
| **Timeout** | Không chờ vô hạn. **Luôn đặt** cho mọi lời gọi ngoài |
| **Retry** + backoff + jitter | Vượt lỗi thoáng qua; chỉ retry thao tác **idempotent** |
| **Circuit Breaker** | Ngắt mạch khi tỷ lệ lỗi cao, tránh dồn tải vào dịch vụ đang hỏng |
| **Bulkhead** | Cô lập tài nguyên (pool riêng) để một phụ thuộc chậm không nuốt hết luồng |
| **Rate Limiter** | Giới hạn tần suất gọi |
| **Fallback** | Trả giá trị thay thế (cache, mặc định) khi lỗi |

```java
@Service
class PricingClient {

    @CircuitBreaker(name = "pricing", fallbackMethod = "cached")
    @Retry(name = "pricing")
    @TimeLimiter(name = "pricing")
    public Price fetch(long productId) { return http.get(productId); }

    Price cached(long productId, Throwable t) { return cache.lastKnown(productId); }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      pricing:
        sliding-window-size: 20
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 5
  retry:
    instances:
      pricing:
        max-attempts: 3
        wait-duration: 200ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
```

Trạng thái Circuit Breaker: **CLOSED** (bình thường) → **OPEN** (từ chối ngay) → **HALF-OPEN** (thử một ít để xem phục hồi chưa).

### 12. Retry đúng cách

> [!danger] Retry storm
> Retry ở nhiều tầng nhân lên theo cấp số nhân (3 tầng × 3 lần = 27 lần gọi) và làm sập dịch vụ đang yếu. Chỉ retry ở **một tầng**, có **backoff + jitter**, giới hạn lần, kèm circuit breaker. Không retry thao tác không idempotent (trừ khi dùng idempotency key, xem [[09 - Spring MVC và thiết kế REST API]]).

### 13. Các nguyên tắc khác

- **Degrade gracefully:** tính năng phụ lỗi thì bỏ, tính năng chính vẫn chạy
- **Load shedding:** quá tải thì từ chối sớm (503/429) thay vì sụp đổ
- **Backpressure:** hàng đợi **có giới hạn** ([[05 - Concurrency và Multithreading]])
- **Health check đúng nghĩa:** readiness không nên phụ thuộc cứng vào mọi dịch vụ ngoài (tránh sập dây chuyền)
- **Chaos engineering** để kiểm chứng ([[12 - Testing chuyên sâu]])

> [!question] Tự kiểm tra
> 1. Nêu 3 sự cố của cache và cách phòng từng cái.
> 2. Vì sao consumer phải idempotent? Làm sao cài đặt?
> 3. Kafka đảm bảo thứ tự ở mức nào? Chọn partition key thế nào?
> 4. Circuit breaker, retry, timeout và bulkhead phối hợp ra sao? Retry storm là gì?
> 5. Khi nào chọn RabbitMQ thay vì Kafka?

⬅️ [[12 - Testing chuyên sâu]] | [[00 - Lộ trình Senior|Mục lục]] | [[14 - Microservices và Distributed Patterns]] ➡️
