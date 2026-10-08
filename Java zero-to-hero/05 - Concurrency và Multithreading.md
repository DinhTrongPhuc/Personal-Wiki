---
title: Concurrency và Multithreading
aliases: [Đa luồng, Concurrency, Java Memory Model, Virtual Thread]
tags: [java, concurrency, senior, hoc-tap]
created: 2026-10-08
---

# 05 - Concurrency và Multithreading

⬅️ [[04 - Stream, Exception, I-O và Date-Time]] | [[00 - Lộ trình Senior 1|Mục lục]] | [[06 - JVM, Bộ nhớ và Garbage Collection]] ➡️

> [!important] Vì sao quan trọng
> Mọi server Spring đều xử lý nhiều request đồng thời. Bean singleton được **dùng chung giữa các luồng**, nên lỗi đa luồng là loại lỗi senior phải tránh được và chẩn đoán được.

## 1. Ba vấn đề cốt lõi

| Vấn đề | Ý nghĩa |
|--------|---------|
| **Atomicity** (nguyên tử) | Thao tác có bị chen ngang giữa chừng không (`count++` là 3 bước: đọc, cộng, ghi) |
| **Visibility** (nhìn thấy) | Luồng khác có thấy giá trị mới ghi không (cache CPU, reorder) |
| **Ordering** (thứ tự) | Trình biên dịch/CPU có thể sắp xếp lại lệnh |

## 2. Java Memory Model và happens-before

JMM định nghĩa khi nào một luồng **chắc chắn thấy** kết quả ghi của luồng khác. Các quan hệ *happens-before* chính:

- Mở khóa (`unlock`/thoát `synchronized`) happens-before khóa tiếp theo cùng monitor
- Ghi `volatile` happens-before đọc `volatile` cùng biến
- `Thread.start()` happens-before mọi lệnh trong luồng đó; mọi lệnh trong luồng happens-before `join()` trả về
- Ghi vào `final` field trong constructor được công bố an toàn

```java
class Flag {
    private volatile boolean stop;          // không có volatile: luồng đọc có thể không bao giờ thấy thay đổi
    void stop() { stop = true; }
    void run() { while (!stop) { /* làm việc */ } }
}
```

> [!note]
> `volatile` đảm bảo **visibility và ordering**, **không** đảm bảo atomicity: `volatile int n; n++` vẫn sai.

## 3. Đồng bộ hóa

### `synchronized`

```java
class Counter {
    private int n;
    public synchronized void inc() { n++; }
    public synchronized int get() { return n; }
}
```

Khóa theo **monitor của đối tượng** (hoặc `Class` nếu `static`). Có tính **reentrant** (cùng luồng vào lại được).

### `Lock` (java.util.concurrent.locks)

```java
private final ReentrantLock lock = new ReentrantLock();

void transfer() {
    lock.lock();
    try {
        // vùng găng
    } finally {
        lock.unlock();              // LUÔN trong finally
    }
}
// tryLock(timeout) tránh treo vô hạn; ReadWriteLock, StampedLock cho đọc nhiều
```

### Atomic và CAS

```java
AtomicInteger n = new AtomicInteger();
n.incrementAndGet();
n.compareAndSet(5, 6);          // CAS: lock-free, dựa lệnh CPU

LongAdder hits = new LongAdder();   // tốt hơn AtomicLong khi tranh chấp cao
hits.increment();
```

## 4. Deadlock, livelock, starvation

**Deadlock** xảy ra khi 4 điều kiện đồng thời: loại trừ tương hỗ, giữ-và-chờ, không bị cưỡng đoạt, chờ vòng tròn.

```java
// Luồng 1: khóa A rồi B      Luồng 2: khóa B rồi A  -> deadlock
```

Cách tránh:
1. **Luôn khóa theo thứ tự cố định**
2. Dùng `tryLock` với timeout
3. Giữ khóa **ngắn nhất** có thể, không gọi code lạ trong khi giữ khóa
4. Ưu tiên cấu trúc bất biến và thuần hàm

Chẩn đoán: `jstack <pid>` hoặc `jcmd <pid> Thread.print` sẽ báo "Found one Java-level deadlock".

## 5. Thread pool và Executor

Không tự `new Thread()` trong code nghiệp vụ. Dùng `ExecutorService`.

```java
ExecutorService pool = new ThreadPoolExecutor(
    4, 16,                                   // core, max
    60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(200),           // hàng đợi CÓ GIỚI HẠN
    new ThreadPoolExecutor.CallerRunsPolicy());
```

Cơ chế nhận việc của `ThreadPoolExecutor`:
1. Còn dưới `core` thì tạo luồng mới
2. Đủ `core` thì **đưa vào hàng đợi**
3. Hàng đợi đầy thì tạo thêm luồng đến `max`
4. Vượt `max` thì chạy chính sách từ chối (reject)

> [!danger] Bẫy
> `Executors.newFixedThreadPool` dùng hàng đợi **không giới hạn** (có thể cạn bộ nhớ). `newCachedThreadPool` có thể tạo vô số luồng. Trong production hãy tự cấu hình `ThreadPoolExecutor` hoặc dùng virtual thread.

Cỡ pool gợi ý: CPU-bound khoảng `số core`; I/O-bound khoảng `core * (1 + thời gian chờ / thời gian tính)`. Luôn **đo** để xác nhận.

## 6. CompletableFuture

```java
CompletableFuture<User> userF = CompletableFuture.supplyAsync(() -> userClient.get(id), pool);
CompletableFuture<List<Order>> ordersF = CompletableFuture.supplyAsync(() -> orderClient.list(id), pool);

CompletableFuture<Profile> profile = userF.thenCombine(ordersF, Profile::new)
    .orTimeout(2, TimeUnit.SECONDS)
    .exceptionally(ex -> Profile.empty());

Profile p = profile.join();
```

Lưu ý: luôn truyền **executor riêng** (mặc định dùng `commonPool`), luôn xử lý lỗi và timeout.

## 7. Virtual Thread (Java 21)

Luồng ảo do JVM lập lịch, rất nhẹ (có thể tạo hàng triệu), phù hợp **I/O-bound**: mỗi request một luồng ảo, viết code đồng bộ dễ đọc mà vẫn mở rộng tốt.

```java
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10_000; i++) {
        exec.submit(() -> httpCall());      // chặn I/O thoải mái
    }
}
```

```yaml
# Spring Boot 3.2+: bật virtual thread cho Tomcat và task executor
spring:
  threads:
    virtual:
      enabled: true
```

| Nên | Không nên |
|-----|-----------|
| Công việc chờ I/O (DB, HTTP) | Công việc CPU nặng kéo dài |
| Không cần pool cho virtual thread | Pool virtual thread (đừng pool chúng) |
| Dùng `ReentrantLock` | Giữ `synchronized` quanh I/O lâu (có thể ghim luồng, *pinning*, ở các bản JDK cũ) |

Cần bao ngoài kết nối DB: vẫn bị giới hạn bởi **connection pool**, nên virtual thread không thay thế việc kiểm soát tài nguyên.

## 8. ThreadLocal và bối cảnh

`ThreadLocal` lưu dữ liệu theo từng luồng (Spring dùng cho `SecurityContext`, transaction, MDC log). Nguy cơ:
- **Rò rỉ** khi luồng trong pool được tái sử dụng: phải `remove()` sau khi xong
- Không tự lan truyền sang luồng `@Async`/executor khác (phải truyền tay)

## 9. Concurrency trong Spring

- Bean singleton **không được giữ trạng thái thay đổi** (field mutable) trừ khi an toàn luồng.
- `@Async`: cần `@EnableAsync` và nên cấu hình `TaskExecutor` riêng.
- `@Transactional` gắn với luồng hiện tại: sang luồng khác là **transaction khác** (xem [[10 - JPA, Hibernate, SQL và Transaction]]).
- Tăng đồng thời ở DB: dùng khóa lạc quan/bi quan, không chỉ khóa trong JVM (nhiều instance thì khóa JVM vô tác dụng).

## 10. Quy tắc thực chiến

- [ ] Ưu tiên **bất biến** và **không chia sẻ trạng thái**
- [ ] Dùng cấu trúc cấp cao (`ConcurrentHashMap`, `BlockingQueue`, `Atomic*`) thay vì tự khóa
- [ ] Hàng đợi và pool **luôn có giới hạn**; đặt tên luồng để dễ debug
- [ ] Luôn có **timeout** cho mọi lời gọi chặn
- [ ] Test race condition: lặp nhiều lần, dùng công cụ như jcstress; đừng tin "chạy thử thấy ổn"

> [!question] Tự kiểm tra
> 1. `volatile` giải quyết được gì và không giải quyết được gì?
> 2. Nêu 4 điều kiện của deadlock và cách phá từng điều kiện.
> 3. Mô tả thứ tự `ThreadPoolExecutor` nhận một task mới.
> 4. Virtual thread có thay thế được connection pool không? Vì sao?

⬅️ [[04 - Stream, Exception, I-O và Date-Time]] | [[00 - Lộ trình Senior 1|Mục lục]] | [[06 - JVM, Bộ nhớ và Garbage Collection]] ➡️
