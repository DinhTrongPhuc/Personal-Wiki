Đây là những khái niệm liên quan đến **SEO (Search Engine Optimization)** — tức là tối ưu website để Google hiểu website tốt hơn và có cơ hội xuất hiện cao hơn trên kết quả tìm kiếm.

### 1. Google Search Console là gì?

**Google Search Console (GSC)** là công cụ của Google giúp chủ website theo dõi cách website xuất hiện trên Google Search.

Có thể dùng để:

- Kiểm tra Google đã **index** trang chưa.
- Xem website xuất hiện với những **keyword** nào.
- Xem số lần website được hiển thị và được click.
- Kiểm tra lỗi crawling/indexing.
- Gửi sitemap cho Google.
- Kiểm tra một URL cụ thể đã được Google index chưa.
- Phát hiện một số vấn đề liên quan đến SEO.

Ví dụ:

```
User tìm: "học lập trình Java"

Google Search
      ↓
Website của bạn xuất hiện
      ↓
Google Search Console
      ↓
Bạn xem:
- Có bao nhiêu người thấy website
- Có bao nhiêu người click
- Keyword nào đưa người dùng đến website
```

👉 **Hiểu đơn giản:** Google Search Console = **công cụ theo dõi và quản lý website trên Google Search**.

---

## 2. `sitemap.xml` là gì?

`sitemap.xml` là một file XML chứa danh sách những URL quan trọng mà bạn muốn Google biết đến.

Ví dụ:

```
<?xml version="1.0" encoding="UTF-8"?>

<urlset>
    <url>
        <loc>https://example.com/</loc>
    </url>

    <url>
        <loc>https://example.com/products</loc>
    </url>

    <url>
        <loc>https://example.com/blog</loc>
    </url>
</urlset>
```

Thông thường nằm ở:

```
https://example.com/sitemap.xml
```

Bạn có thể gửi sitemap này vào Google Search Console để Google biết các URL quan trọng của website.

### Sitemap dùng để làm gì?

Nó giúp **search engine crawler** khám phá các URL của website dễ hơn, đặc biệt hữu ích với:

- Website lớn.
- Website có nhiều trang.
- Website mới.
- Website có cấu trúc link phức tạp.

⚠️ Sitemap **không có nghĩa là Google chắc chắn index tất cả URL**.

---

# 3. Optimize page with keywords

Câu:

> **optimise page with keyword that need to rank for**

có nghĩa là:

> **Tối ưu nội dung của trang dựa trên những từ khóa mà bạn muốn trang đó được xếp hạng trên Google.**

Ví dụ bạn có website bán laptop và muốn rank keyword:

```
"laptop gaming giá rẻ"
```

Bạn sẽ tối ưu page để nội dung thực sự liên quan đến chủ đề đó.

Ví dụ:

```
H1:
Laptop Gaming Giá Rẻ

Content:
Laptop gaming giá rẻ phù hợp với...

H2:
Top laptop gaming giá rẻ

H2:
Cách chọn laptop gaming
```

Keyword nên xuất hiện **tự nhiên và phù hợp với nội dung**, không nên nhồi nhét:

```
❌ Laptop gaming giá rẻ...
   laptop gaming giá rẻ...
   laptop gaming giá rẻ...
   laptop gaming giá rẻ...
```

Google hiện không đơn giản xếp hạng chỉ vì một keyword xuất hiện nhiều lần. **Chất lượng, mức độ hữu ích và sự phù hợp với search intent** rất quan trọng.

---

# 4. Meta Title

**Meta title** là tiêu đề mà bạn khai báo cho trang, thường được Google sử dụng làm tiêu đề trong kết quả tìm kiếm.

Ví dụ:

```
<title>
Laptop Gaming Giá Rẻ - Top Laptop Gaming 2026
</title>
```

Google có thể hiển thị:

```
Laptop Gaming Giá Rẻ - Top Laptop Gaming 2026
example.com

Laptop gaming giá rẻ với cấu hình mạnh...
```

Meta title quan trọng vì nó giúp:

- Google hiểu chủ đề của page.
- Người dùng biết page nói về gì.
- Tăng khả năng người dùng click vào kết quả.

---

# 5. Meta Description

Meta description mô tả ngắn nội dung của page.

Ví dụ:

```
<meta
  name="description"
  content="Khám phá các mẫu laptop gaming giá rẻ với cấu hình mạnh, phù hợp cho học tập, lập trình và chơi game."
/>
```

Google có thể hiển thị:

```
Laptop Gaming Giá Rẻ - Top Laptop Gaming 2026
example.com

Khám phá các mẫu laptop gaming giá rẻ với cấu hình mạnh,
phù hợp cho học tập, lập trình và chơi game.
```

Meta description chủ yếu giúp **người dùng hiểu trang nói về gì và tăng khả năng click**; Google không đảm bảo luôn hiển thị chính xác đoạn description bạn viết.

---

# 🔥 Ghép cả 4 thứ lại

Ví dụ bạn làm website:

```
example.com
```

và muốn rank:

> **"Node.js course"**

Bạn có thể làm:

### Bước 1 — Tối ưu page

```
H1:
Node.js Course for Beginners

Content:
Learn Node.js from beginner to advanced...
```

### Bước 2 — Meta title

```
<title>Node.js Course for Beginners | Example</title>
```

### Bước 3 — Meta description

```
<meta
  name="description"
  content="Learn Node.js from beginner to advanced with practical backend projects."
/>
```

### Bước 4 — Sitemap

```
https://example.com/sitemap.xml
```

Trong đó có:

```
https://example.com/
https://example.com/nodejs-course
https://example.com/about
```

### Bước 5 — Google Search Console

Bạn submit:

```
https://example.com/sitemap.xml
```

Sau đó theo dõi:

```
Keyword:
"nodejs course"

Impressions: 10,000
Clicks:       500
CTR:            5%
Average position: 8.2
```

---

