---
title: JPA, Hibernate, SQL và Transaction
aliases: [JPA, Hibernate, Transaction, Isolation, N+1, Index]
tags: [jpa, hibernate, sql, transaction, senior, hoc-tap]
created: 2026-10-08
---

# 10 - JPA, Hibernate, SQL và Transaction

⬅️ [[09 - Spring MVC và thiết kế REST API]] | [[00 - Lộ trình Senior 1|Mục lục]] | [[11 - Spring Security, JWT và OAuth2]] ➡️

> [!important]
> Phần lớn sự cố hiệu năng và dữ liệu của backend nằm ở **tầng dữ liệu**. Đây là chủ đề có giá trị nhất để đầu tư.

## 1. Các tầng

- **JPA**: đặc tả (API) ORM chuẩn
- **Hibernate**: bản cài đặt phổ biến nhất
- **Spring Data JPA**: sinh repository từ interface, tích hợp transaction

## 2. Persistence Context

`EntityManager` giữ một **persistence context** (cache cấp 1) trong phạm vi transaction:

| Trạng thái | Ý nghĩa |
|-----------|---------|
| Transient | `new` nhưng chưa được quản lý |
| Managed | Đang được theo dõi: **thay đổi tự động ghi xuống DB** lúc flush |
| Detached | Từng managed nhưng context đã đóng |
| Removed | Đã đánh dấu xóa |

```java
@Transactional
public void rename(long id, String name) {
    User u = repo.findById(id).orElseThrow();
    u.setName(name);          // KHÔNG cần save(): dirty checking tự UPDATE khi commit
}
```

- **Flush** đồng bộ thay đổi xuống DB (trước commit, trước truy vấn JPQL liên quan).
- `findById` hai lần trong cùng transaction chỉ chạy **một** SELECT (cache cấp 1).
- Context phình lớn khi xử lý hàng loạt: gọi `flush()` + `clear()` theo lô.

## 3. Mapping thực chiến

```java
@Entity
@Table(name = "orders", indexes = @Index(name = "idx_orders_customer", columnList = "customer_id"))
public class Order {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Enumerated(EnumType.STRING)           // luôn STRING, không ORDINAL
    private Status status;

    @Version                               // khóa lạc quan
    private long version;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    public void addItem(OrderItem i) { items.add(i); i.setOrder(this); }   // đồng bộ 2 chiều

    @Column(precision = 19, scale = 2)
    private BigDecimal total;

    protected Order() {}                   // JPA cần constructor không tham số

    // equals/hashCode: dựa trên khóa nghiệp vụ hoặc id đã gán; cẩn trọng với proxy
}
```

Quy tắc:
- Quan hệ **mặc định `LAZY`** (`@ManyToOne`/`@OneToOne` mặc định là EAGER, hãy đổi)
- Phía "sở hữu" quan hệ là phía có khóa ngoại; giữ hai chiều đồng bộ bằng helper
- Với `@OneToMany` lớn, tránh `List` hai chiều nạp cả bộ; cân nhắc truy vấn riêng
- Không dùng `record` làm entity; không `@Data` của Lombok trên entity (sinh `equals/hashCode/toString` gây vòng lặp)
- Khóa chính: `IDENTITY` (đơn giản, nhưng tắt batch insert), `SEQUENCE` (tốt cho batch), UUID (v7 để có thứ tự theo thời gian)

## 4. Vấn đề N+1

Nạp N đơn hàng rồi truy cập `order.getCustomer()` cho từng đơn sinh **1 + N** truy vấn.

```java
// Cách 1: JOIN FETCH
@Query("select o from Order o join fetch o.customer where o.status = :s")
List<Order> findWithCustomer(@Param("s") Status s);

// Cách 2: EntityGraph
@EntityGraph(attributePaths = {"customer"})
List<Order> findByStatus(Status s);

// Cách 3: batch size (nạp theo lô khi lazy)
// spring.jpa.properties.hibernate.default_batch_fetch_size: 50

// Cách 4: DTO projection: chỉ lấy cột cần
@Query("select new com.app.OrderSummary(o.id, c.name, o.total) from Order o join o.customer c")
List<OrderSummary> summaries();
```

> [!warning]
> `JOIN FETCH` một **collection** kết hợp phân trang khiến Hibernate phân trang **trong bộ nhớ** (cảnh báo `HHH000104`). Tách làm hai truy vấn: lấy id theo trang, rồi fetch.

Phát hiện: bật `spring.jpa.show-sql` ở dev, thống kê Hibernate, hoặc dùng thư viện như `datasource-proxy`/Hypersistence Utils để **đếm truy vấn trong test**.

## 5. Open Session In View (OSIV)

Mặc định `spring.jpa.open-in-view=true` giữ session mở đến hết request (cho phép lazy ở view/controller) nhưng **giữ kết nối DB quá lâu** và che giấu N+1. Nên **tắt** (`false`) và đảm bảo dữ liệu cần đã được nạp trong tầng service.

## 6. Transaction

### Thuộc tính ACID
Atomicity, Consistency, Isolation, Durability.

### `@Transactional` trong Spring

```java
@Service
public class TransferService {

    @Transactional                                    // mặc định: REQUIRED
    public void transfer(long from, long to, BigDecimal amt) { ... }

    @Transactional(readOnly = true)                   // tối ưu, tránh flush
    public Account get(long id) { ... }

    @Transactional(rollbackFor = Exception.class)     // mặc định chỉ rollback RuntimeException/Error
    public void importFile() throws IOException { ... }
}
```

| Propagation | Hành vi |
|-------------|---------|
| `REQUIRED` (mặc định) | Tham gia transaction hiện có, không có thì tạo mới |
| `REQUIRES_NEW` | Luôn tạo transaction mới, tạm dừng cái cũ (ví dụ ghi audit log dù rollback) |
| `SUPPORTS`, `NOT_SUPPORTED`, `NEVER`, `MANDATORY` | Các biến thể ít dùng |
| `NESTED` | Savepoint (cần hỗ trợ từ DB/driver) |

> [!danger] Các bẫy kinh điển
> 1. **Self-invocation** bỏ qua proxy: xem [[07 - Spring Core, IoC và AOP]].
> 2. Chỉ rollback với **unchecked exception** theo mặc định.
> 3. Bắt exception trong method `@Transactional` rồi nuốt đi: transaction vẫn commit (hoặc bị đánh dấu rollback-only gây `UnexpectedRollbackException`).
> 4. `@Transactional` trên method không `public` hoặc ở lớp không phải bean: vô tác dụng.
> 5. **Không gọi API ngoài/HTTP chậm trong transaction**: giữ kết nối DB và khóa lâu.
> 6. `@Async` hoặc luồng khác: là transaction **khác**.
> 7. Gửi message/sự kiện trước commit: dùng `@TransactionalEventListener(AFTER_COMMIT)` hoặc **Outbox** ([[14 - Microservices và Distributed Patterns]]).

### Mức cô lập (Isolation)

| Mức | Dirty read | Non-repeatable read | Phantom |
|-----|:---------:|:-------------------:|:-------:|
| READ UNCOMMITTED | có | có | có |
| READ COMMITTED (mặc định Postgres, Oracle) | không | có | có |
| REPEATABLE READ (mặc định MySQL InnoDB) | không | không | có thể (tùy DB) |
| SERIALIZABLE | không | không | không |

Mức cao hơn: ít bất thường hơn nhưng nhiều khóa/xung đột, hiệu năng thấp hơn. Hành vi chi tiết **phụ thuộc DB** (Postgres dùng MVCC; REPEATABLE READ của Postgres chống được phantom).

## 7. Kiểm soát đồng thời (cập nhật chồng nhau)

**Lost update**: hai người đọc cùng một số dư rồi cùng ghi đè.

```java
// Khóa lạc quan: @Version. Ghi sai phiên bản => OptimisticLockingFailureException, thử lại hoặc báo 409
// Khóa bi quan: khóa dòng ngay khi đọc
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select a from Account a where a.id = :id")
Optional<Account> findForUpdate(@Param("id") long id);

// Hoặc cập nhật nguyên tử trong SQL
@Modifying
@Query("update Account a set a.balance = a.balance - :amt where a.id = :id and a.balance >= :amt")
int debit(@Param("id") long id, @Param("amt") BigDecimal amt);
```

| | Lạc quan | Bi quan |
|--|----------|---------|
| Khi nào | Xung đột hiếm | Xung đột thường xuyên, tính toàn vẹn nghiêm ngặt |
| Chi phí | Thử lại khi lỗi | Khóa giữ lâu, nguy cơ deadlock |

Deadlock DB: khóa tài nguyên theo **thứ tự nhất quán**; transaction ngắn.

## 8. SQL và Index

### Đọc kế hoạch thực thi

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 42 ORDER BY created_at DESC LIMIT 20;
```

Tìm: `Seq Scan` trên bảng lớn (thiếu index), ước lượng hàng sai nhiều (thống kê cũ), `Sort` tốn kém.

### Thiết kế index

```sql
CREATE INDEX idx_orders_cust_created ON orders (customer_id, created_at DESC);
```

- **Composite index**: thứ tự cột quan trọng (quy tắc *leftmost prefix*): cột lọc bằng `=` trước, rồi cột sắp xếp/khoảng
- Index tăng tốc **đọc**, làm chậm **ghi** và tốn dung lượng: không index tràn lan
- Hàm trên cột (`WHERE lower(email) = ...`) vô hiệu hóa index thường: dùng index biểu thức
- `LIKE '%abc'` không dùng được B-tree thông thường
- Index phủ (*covering*), index một phần (*partial*), unique index cho ràng buộc nghiệp vụ

### Quy tắc truy vấn
- Chọn cột cần, tránh `SELECT *`
- Tránh `OFFSET` lớn (dùng keyset)
- Thao tác hàng loạt: `batch insert` (`hibernate.jdbc.batch_size`, `order_inserts`) hoặc `JdbcTemplate.batchUpdate`
- Báo cáo nặng: đọc từ **read replica** hoặc kho riêng

## 9. Connection Pool (HikariCP)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10         # KHÔNG phải càng lớn càng tốt
      minimum-idle: 10
      connection-timeout: 3000
      max-lifetime: 1500000         # nhỏ hơn timeout của DB/proxy
```

- Pool nhỏ thường **nhanh hơn**: số kết nối đồng thời hữu ích bị giới hạn bởi CPU/đĩa của DB.
- Quy tắc tham khảo: `kết nối ≈ (số core DB × 2) + số đĩa`, rồi đo.
- Tổng pool của **mọi instance** phải nhỏ hơn `max_connections` của DB.
- Triệu chứng cạn pool: request treo rồi `Connection is not available`; thường do transaction dài hoặc rò kết nối.

## 10. Migration

Dùng **Flyway** hoặc **Liquibase**, đặt `ddl-auto: validate`.

```
db/migration/V12__add_status_to_orders.sql
```

Triển khai không downtime: **expand then contract**:
1. Thêm cột mới (nullable) → 2. Ghi cả hai → 3. Backfill → 4. Chuyển đọc → 5. Xóa cột cũ ở bản sau.
Không bao giờ sửa migration đã chạy; mỗi thay đổi là một file mới.

## 11. Khi nào KHÔNG dùng JPA

- Báo cáo/truy vấn phức tạp: **jOOQ**, `JdbcClient`/`JdbcTemplate`, native query, view
- Xử lý hàng loạt khối lượng lớn
- Kết hợp được: JPA cho ghi (command), SQL thuần/jOOQ cho đọc (query), tương tự CQRS đơn giản

> [!question] Tự kiểm tra
> 1. Dirty checking là gì? Vì sao không cần gọi `save()` trong method `@Transactional`?
> 2. Nêu 4 cách xử lý N+1 và nhược điểm của `JOIN FETCH` với phân trang.
> 3. `REQUIRES_NEW` dùng khi nào? Vì sao self-invocation làm nó không hoạt động?
> 4. Hai request cùng trừ tiền một tài khoản: có những cách nào ngăn lost update?
> 5. Vì sao tăng `maximum-pool-size` lên rất lớn có thể làm hệ thống chậm đi?

⬅️ [[09 - Spring MVC và thiết kế REST API]] | [[00 - Lộ trình Senior 1|Mục lục]] | [[11 - Spring Security, JWT và OAuth2]] ➡️
