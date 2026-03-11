# MySQL Query Optimization — Tips & Tricks Toàn Diện

> Tổng hợp từ kinh nghiệm thực chiến của các senior developer, DBA, và các bài phân tích production incidents.
> Mỗi tip đều có ví dụ **câu query tệ** → **câu query tối ưu** kèm giải thích.

---

## Mục lục

1. [Luôn dùng EXPLAIN để phân tích query](#1-luôn-dùng-explain-để-phân-tích-query)
2. [Tránh SELECT *](#2-tránh-select-)
3. [Đánh index đúng chỗ — Composite Index](#3-đánh-index-đúng-chỗ--composite-index)
4. [Tránh dùng function trong WHERE](#4-tránh-dùng-function-trong-where)
5. [Tránh LIKE với wildcard đầu chuỗi](#5-tránh-like-với-wildcard-đầu-chuỗi)
6. [Dùng EXISTS thay vì COUNT > 0](#6-dùng-exists-thay-vì-count--0)
7. [Luôn có LIMIT — Tránh lấy toàn bộ dữ liệu](#7-luôn-có-limit--tránh-lấy-toàn-bộ-dữ-liệu)
8. [Cursor-based Pagination thay vì OFFSET lớn](#8-cursor-based-pagination-thay-vì-offset-lớn)
9. [JOIN trên cột có index — Tránh JOIN trên email/string](#9-join-trên-cột-có-index--tránh-join-trên-emailstring)
10. [N+1 Query Problem](#10-n1-query-problem)
11. [Tránh Cartesian Join (thiếu điều kiện JOIN)](#11-tránh-cartesian-join-thiếu-điều-kiện-join)
12. [Subquery lồng nhau vs JOIN / CTE](#12-subquery-lồng-nhau-vs-join--cte)
13. [Covering Index — Index bao phủ toàn bộ SELECT](#13-covering-index--index-bao-phủ-toàn-bộ-select)
14. [Tối ưu kiểu dữ liệu (Data Types)](#14-tối-ưu-kiểu-dữ-liệu-data-types)
15. [Tránh aggregation không có filter (Unbounded Aggregation)](#15-tránh-aggregation-không-có-filter-unbounded-aggregation)
16. [Phân vùng / Archive bảng lớn (Partitioning)](#16-phân-vùng--archive-bảng-lớn-partitioning)
17. [Transaction — Tránh giữ transaction quá lâu](#17-transaction--tránh-giữ-transaction-quá-lâu)
18. [Cập nhật statistics thường xuyên (ANALYZE TABLE)](#18-cập-nhật-statistics-thường-xuyên-analyze-table)
19. [Bật Slow Query Log để phát hiện query chậm](#19-bật-slow-query-log-để-phát-hiện-query-chậm)
20. [Cấu hình InnoDB Buffer Pool](#20-cấu-hình-innodb-buffer-pool)

---

## 1. Luôn dùng EXPLAIN để phân tích query

**Công cụ chẩn đoán đầu tiên trước khi optimize bất kỳ query nào.**

```sql
-- Xem plan thực thi
EXPLAIN SELECT name FROM users WHERE email = 'user@example.com';

-- Chi tiết hơn (MySQL 8.0+)
EXPLAIN FORMAT=JSON SELECT name FROM users WHERE email = 'user@example.com';

-- Xem thời gian thực thi thực tế (MySQL 8.0.18+)
EXPLAIN ANALYZE SELECT name FROM users WHERE email = 'user@example.com';
```

**Các chỉ số cần chú ý trong output EXPLAIN:**

| Cột `type` | Ý nghĩa | Mức độ |
|---|---|---|
| `const` / `eq_ref` | Tìm qua primary key / unique index | Tốt nhất |
| `ref` | Tìm qua index thông thường | Tốt |
| `range` | Scan một dải index | Chấp nhận được |
| `index` | Scan toàn bộ index | Cần xem xét |
| `ALL` | Full table scan | **Nguy hiểm** |

> Nếu thấy `type: ALL` trên bảng có hơn 10.000 dòng → **phải thêm index ngay**.

---

## 2. Tránh SELECT *

**Vấn đề:** Lấy toàn bộ cột gây tốn bộ nhớ, băng thông, và làm MySQL không thể dùng covering index.

```sql
-- ❌ BAD: Lấy tất cả cột không cần thiết
SELECT * FROM users;

-- ❌ BAD: Lấy cả bảng không giới hạn
SELECT * FROM orders WHERE status = 'pending';
```

```sql
-- ✅ GOOD: Chỉ lấy cột cần dùng
SELECT id, name, email FROM users LIMIT 100;

-- ✅ GOOD: Lấy đúng cột cần thiết
SELECT id, user_id, total, created_at FROM orders WHERE status = 'pending';
```

**Lý do:**
- Giảm lượng data truyền qua network
- Cho phép MySQL sử dụng **covering index** (không cần đọc data rows)
- Tránh load các cột TEXT/BLOB nặng không cần thiết

---

## 3. Đánh index đúng chỗ — Composite Index

**Vấn đề:** Thiếu index là nguyên nhân số 1 khiến query chậm exponentially khi data tăng.

**Case thực tế:** Query chạy ổn với 10K dòng, nhưng với 50 triệu dòng → response time từ 100ms tăng lên 8 giây, CPU database 100%, bill AWS tăng từ $4,000 lên $51,000/tháng.

```sql
-- ❌ BAD: Không có index trên cột filter
SELECT * FROM orders WHERE status = 'pending' AND customer_id = 123;
-- → Full table scan trên 50 triệu dòng
```

```sql
-- ✅ GOOD: Tạo composite index theo thứ tự selectivity cao → thấp
CREATE INDEX idx_orders_status_customer ON orders(status, customer_id);

-- ✅ GOOD hơn: Thêm cột date vào để range query cũng dùng được index
CREATE INDEX idx_orders_user_status_date ON orders(user_id, status, created_at DESC);
```

**Nguyên tắc đặt thứ tự cột trong Composite Index:**
1. Cột có **equality filter** (`=`) đặt trước
2. Cột có **range filter** (`>`, `<`, `BETWEEN`) đặt sau
3. Cột dùng trong `ORDER BY` đặt cuối

> **Lưu ý:** Đừng over-index — mỗi index làm chậm INSERT/UPDATE/DELETE.

---

## 4. Tránh dùng function trong WHERE

**Vấn đề:** Bọc cột index trong function sẽ làm MySQL không dùng được index → full table scan.

```sql
-- ❌ BAD: YEAR() bọc cột index → index vô dụng
SELECT * FROM invoices WHERE YEAR(created_at) = 2024;

-- ❌ BAD: DATE() bọc cột index
SELECT * FROM orders WHERE DATE(created_at) = '2024-01-15';

-- ❌ BAD: Hàm string trên cột index
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

-- ❌ BAD: Phép tính số học trên cột index
SELECT * FROM products WHERE price * 1.1 > 100;
```

```sql
-- ✅ GOOD: So sánh trực tiếp, index được dùng
SELECT * FROM invoices 
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- ✅ GOOD: Range so sánh
SELECT * FROM orders 
WHERE created_at >= '2024-01-15 00:00:00' 
  AND created_at < '2024-01-16 00:00:00';

-- ✅ GOOD: Dùng collation phù hợp, hoặc lưu lowercase sẵn
SELECT * FROM users WHERE email = 'user@example.com';
-- (đảm bảo cột email dùng collation case-insensitive)

-- ✅ GOOD: Đưa phép tính sang vế phải
SELECT * FROM products WHERE price > 100 / 1.1;
```

---

## 5. Tránh LIKE với wildcard đầu chuỗi

**Vấn đề:** `LIKE '%keyword%'` hoặc `LIKE '%keyword'` buộc MySQL scan toàn bộ bảng, không dùng được B-tree index.

```sql
-- ❌ BAD: Wildcard đầu → full table scan
SELECT name FROM products WHERE name LIKE '%phone%';
SELECT name FROM products WHERE name LIKE '%phone';
```

```sql
-- ✅ GOOD: Wildcard cuối → vẫn dùng được index
SELECT name FROM products WHERE name LIKE 'phone%';

-- ✅ GOOD: Dùng FULLTEXT index cho tìm kiếm toàn văn
ALTER TABLE products ADD FULLTEXT(name, description);

SELECT name FROM products 
WHERE MATCH(name, description) AGAINST('phone' IN NATURAL LANGUAGE MODE);

-- ✅ GOOD: Tạo FULLTEXT index khi create table
CREATE FULLTEXT INDEX idx_products_search ON products(name, description);
```

**Khi nào dùng FULLTEXT:**
- Tìm kiếm từ khóa trong nội dung văn bản
- Hỗ trợ Boolean mode: `AGAINST('+phone -samsung' IN BOOLEAN MODE)`
- Nhanh hơn nhiều so với `LIKE '%...%'` với dữ liệu lớn

---

## 6. Dùng EXISTS thay vì COUNT > 0

**Vấn đề:** `COUNT(*)` phải đếm toàn bộ matching rows. `EXISTS` dừng ngay khi tìm thấy row đầu tiên.

```sql
-- ❌ BAD: Đếm toàn bộ để chỉ kiểm tra có tồn tại không
SELECT * FROM users u
WHERE (SELECT COUNT(*) FROM orders WHERE user_id = u.id) > 0;

-- ❌ BAD: Tương tự
IF (SELECT COUNT(*) FROM orders WHERE user_id = 10) > 0 THEN ...
```

```sql
-- ✅ GOOD: EXISTS dừng ngay khi tìm thấy match đầu tiên
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders WHERE user_id = u.id);

-- ✅ GOOD
IF EXISTS (SELECT 1 FROM orders WHERE user_id = 10) THEN ...
```

**Quy tắc chung:**
- Cần **biết số lượng chính xác** → dùng `COUNT(*)`
- Chỉ cần **kiểm tra có tồn tại** → dùng `EXISTS`

---

## 7. Luôn có LIMIT — Tránh lấy toàn bộ dữ liệu

**Vấn đề:** Không có LIMIT trên bảng lớn → load hàng triệu rows vào memory → crash hoặc timeout.

```sql
-- ❌ BAD: Lấy toàn bộ bảng users
SELECT * FROM users;

-- ❌ BAD: Không giới hạn kết quả
SELECT id, title FROM posts WHERE status = 'published';
```

```sql
-- ✅ GOOD: Luôn kèm LIMIT
SELECT id, name, email FROM users LIMIT 100;

-- ✅ GOOD: Kết hợp với pagination
SELECT id, title FROM posts 
WHERE status = 'published' 
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;
```

---

## 8. Cursor-based Pagination thay vì OFFSET lớn

**Vấn đề:** `OFFSET` càng lớn, MySQL càng phải scan nhiều rows để bỏ qua.

- `OFFSET 0` → scan 20 rows
- `OFFSET 1000` → scan 1.020 rows
- `OFFSET 10000` → scan 10.020 rows → hiệu năng tuyến tính giảm

```sql
-- ❌ BAD: OFFSET-based pagination
SELECT * FROM posts 
ORDER BY created_at DESC 
LIMIT 20 OFFSET 2000;
-- Page 100 → phải scan 2.020 rows → 22 giây với bảng lớn
```

```sql
-- ✅ GOOD: Cursor-based pagination (keyset pagination)
-- Lần đầu tiên:
SELECT id, title, created_at FROM posts 
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Các lần tiếp theo — dùng giá trị cuối của lần trước:
SELECT id, title, created_at FROM posts 
WHERE (created_at, id) < ('2024-01-15 10:30:00', 9999)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

**Kết quả:** Page 100 từ 22 giây → 50ms. Mọi page đều nhanh như nhau.

**Lưu ý:** Cursor-based pagination không hỗ trợ "nhảy đến trang X" — phù hợp với infinite scroll / "Load more".

---

## 9. JOIN trên cột có index — Tránh JOIN trên email/string

**Vấn đề:** JOIN trên cột không phải integer/primary key, hoặc không có index → nested loop scan.

```sql
-- ❌ BAD: JOIN trên email (string, không phải PK)
SELECT * FROM orders o
JOIN customers c ON o.email = c.email;

-- ❌ BAD: JOIN trên cột không có index
SELECT o.id, c.name FROM orders o
JOIN customers c ON o.customer_code = c.code;
-- Nếu customer_code không có index → full scan mỗi row
```

```sql
-- ✅ GOOD: JOIN trên integer foreign key (có index)
SELECT o.id, c.name FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- ✅ GOOD: Luôn đánh index trên foreign key
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_product_id ON orders(product_id);
```

**Case thực tế:** JOIN trên 3 bảng (10M × 2M × 500K rows) không có index → EXPLAIN cost 50 triệu → 12 giây. Sau khi thêm index → cost 50K → 80ms.

---

## 10. N+1 Query Problem

**Vấn đề:** Load N records rồi chạy thêm 1 query cho mỗi record → N+1 round trips đến database.

```sql
-- ❌ BAD pattern (giả lập ORM lazy loading):

-- Query 1: Lấy danh sách users
SELECT id, name FROM users LIMIT 100;

-- Query 2..101: Với mỗi user, lấy orders riêng lẻ
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 2;
-- ... 100 lần nữa
-- Tổng: 101 queries thay vì 1-2 queries
```

```sql
-- ✅ GOOD Solution 1: Dùng JOIN
SELECT u.id, u.name, o.id AS order_id, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.id IN (1, 2, 3, ..., 100);

-- ✅ GOOD Solution 2: Batch loading với IN
SELECT * FROM orders 
WHERE user_id IN (1, 2, 3, ..., 100);
-- Sau đó map trong application code
```

**Trong ORM (Laravel Eloquent):**
```php
// ❌ BAD: N+1
$users = User::all();
foreach ($users as $user) {
    echo $user->orders->count(); // Trigger query mỗi lần
}

// ✅ GOOD: Eager loading
$users = User::with('orders')->get();
foreach ($users as $user) {
    echo $user->orders->count(); // Không thêm query
}
```

---

## 11. Tránh Cartesian Join (thiếu điều kiện JOIN)

**Vấn đề:** JOIN nhiều bảng không có điều kiện kết hợp → kết quả là tích Descartes, bùng nổ số dòng.

```sql
-- ❌ BAD: Không có điều kiện JOIN
SELECT * FROM orders o, products p, users u;
-- Kết quả: 1.000 × 10.000 × 100.000 = 1 NGHÌN TỶ dòng
-- → Out of memory, database crash
```

```sql
-- ✅ GOOD: Explicit JOIN với đầy đủ điều kiện
SELECT o.order_id, p.name, u.email
FROM orders o
INNER JOIN products p ON o.product_id = p.id
INNER JOIN users u ON o.user_id = u.id
WHERE o.created_at >= '2024-01-01';
```

---

## 12. Subquery lồng nhau vs JOIN / CTE

**Vấn đề:** Subquery lồng nhiều tầng → mỗi tầng ngoài thực thi subquery tầng trong N lần.

```sql
-- ❌ BAD: Nested subquery 4 tầng
SELECT * FROM users u
WHERE u.id IN (
    SELECT o.user_id FROM orders o
    WHERE o.product_id IN (
        SELECT p.id FROM products p
        WHERE p.category_id IN (
            SELECT c.id FROM categories c
            WHERE c.name LIKE '%electronics%'
        )
    )
);
-- Mỗi tầng nhân thêm lần thực thi
```

```sql
-- ✅ GOOD Solution 1: Flatten với JOIN
SELECT DISTINCT u.*
FROM users u
INNER JOIN orders o ON u.id = o.user_id
INNER JOIN products p ON o.product_id = p.id
INNER JOIN categories c ON p.category_id = c.id
WHERE c.name LIKE '%electronics%';

-- ✅ GOOD Solution 2: CTE (Common Table Expressions) — dễ đọc và tối ưu tốt hơn
WITH electronics_categories AS (
    SELECT id FROM categories 
    WHERE name LIKE '%electronics%'
),
electronics_products AS (
    SELECT id FROM products 
    WHERE category_id IN (SELECT id FROM electronics_categories)
),
electronics_orders AS (
    SELECT DISTINCT user_id FROM orders 
    WHERE product_id IN (SELECT id FROM electronics_products)
)
SELECT * FROM users 
WHERE id IN (SELECT user_id FROM electronics_orders);

-- ✅ GOOD Solution 3: Dùng EXISTS thay IN cho subquery tương quan
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    INNER JOIN products p ON o.product_id = p.id
    INNER JOIN categories c ON p.category_id = c.id
    WHERE o.user_id = u.id
      AND c.name LIKE '%electronics%'
);
```

---

## 13. Covering Index — Index bao phủ toàn bộ SELECT

**Vấn đề:** MySQL thực hiện 2 bước: (1) tra index tìm row IDs, (2) đọc data rows. Nếu index chứa đủ cột cần SELECT, bước (2) bị loại bỏ hoàn toàn.

```sql
-- ❌ BAD: Index chỉ trên user_id, còn lại phải đọc data row
CREATE INDEX idx_orders_user ON orders(user_id);

SELECT user_id, status, total FROM orders WHERE user_id = 123;
-- → Tra index tìm row IDs → đọc data rows để lấy status, total
```

```sql
-- ✅ GOOD: Covering index — bao phủ tất cả cột cần dùng
CREATE INDEX idx_orders_covering ON orders(user_id, status, total, created_at);

SELECT user_id, status, total FROM orders WHERE user_id = 123;
-- → Chỉ tra index, không cần đọc data rows
-- EXPLAIN sẽ hiển thị "Using index" trong Extra column
```

**Kiểm tra covering index trong EXPLAIN:**
```sql
EXPLAIN SELECT user_id, status, total FROM orders WHERE user_id = 123;
-- Extra: "Using index" → đang dùng covering index ✅
-- Extra: "Using index condition" → partial covering
-- Extra: (empty) → phải đọc data rows
```

---

## 14. Tối ưu kiểu dữ liệu (Data Types)

**Vấn đề:** Dùng kiểu dữ liệu quá lớn → lãng phí bộ nhớ, giảm tốc độ sort/compare, giảm số rows có thể lưu trong buffer.

```sql
-- ❌ BAD: Kiểu dữ liệu quá lớn
CREATE TABLE users (
    id          INT(11),          -- Dùng INT cho status? Lãng phí
    status      INT,              -- Nên là TINYINT
    is_active   INT,              -- Nên là TINYINT(1) / BOOLEAN
    description TEXT,             -- VARCHAR(500) nếu không cần quá dài
    age         BIGINT,           -- INT là đủ
    score       DOUBLE            -- DECIMAL(8,2) nếu cần precision
);
```

```sql
-- ✅ GOOD: Kiểu dữ liệu phù hợp
CREATE TABLE users (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    status      TINYINT UNSIGNED NOT NULL DEFAULT 0,  -- 0-255, tiết kiệm
    is_active   TINYINT(1) NOT NULL DEFAULT 1,
    description VARCHAR(500),
    age         TINYINT UNSIGNED,   -- 0-255, đủ cho tuổi
    score       DECIMAL(8, 2)       -- Precision chính xác
);
```

**Bảng so sánh kích thước:**

| Kiểu | Bytes | Range |
|---|---|---|
| TINYINT | 1 | -128 đến 127 (hoặc 0-255 UNSIGNED) |
| SMALLINT | 2 | -32.768 đến 32.767 |
| MEDIUMINT | 3 | ~±8 triệu |
| INT | 4 | ~±2 tỷ |
| BIGINT | 8 | ~±9 × 10^18 |
| VARCHAR(n) | n+1 hoặc n+2 | Chỉ dùng đúng bytes cần |
| CHAR(n) | n bytes cố định | Nhanh hơn VARCHAR cho chuỗi cố định |

---

## 15. Tránh aggregation không có filter (Unbounded Aggregation)

**Vấn đề:** `COUNT`, `SUM`, `AVG` trên toàn bảng → scan hàng triệu rows mỗi lần gọi.

```sql
-- ❌ BAD: Scan toàn bộ lịch sử
SELECT AVG(order_total), COUNT(*), SUM(order_total)
FROM orders;
-- 1M rows → 10M rows → 100M rows: ngày càng chậm hơn
```

```sql
-- ✅ GOOD: Giới hạn phạm vi dữ liệu
SELECT AVG(order_total), COUNT(*), SUM(order_total)
FROM orders
WHERE created_at >= CURRENT_DATE - INTERVAL 30 DAY;

-- ✅ GOOD: Dùng summary table cho thống kê lịch sử
CREATE TABLE daily_order_stats (
    stat_date      DATE PRIMARY KEY,
    order_count    INT NOT NULL DEFAULT 0,
    total_revenue  DECIMAL(15, 2) NOT NULL DEFAULT 0,
    avg_order      DECIMAL(10, 2) NOT NULL DEFAULT 0
);

-- Chạy job định kỳ để cập nhật summary table
INSERT INTO daily_order_stats (stat_date, order_count, total_revenue, avg_order)
SELECT 
    DATE(created_at),
    COUNT(*),
    SUM(order_total),
    AVG(order_total)
FROM orders
WHERE DATE(created_at) = CURDATE() - INTERVAL 1 DAY
ON DUPLICATE KEY UPDATE
    order_count   = VALUES(order_count),
    total_revenue = VALUES(total_revenue),
    avg_order     = VALUES(avg_order);

-- Query summary table thay vì orders table
SELECT * FROM daily_order_stats 
WHERE stat_date >= CURDATE() - INTERVAL 30 DAY;
```

---

## 16. Phân vùng / Archive bảng lớn (Partitioning)

**Vấn đề:** Bảng 100+ triệu dòng → mọi query đều chậm dù có index tốt.

```sql
-- ❌ BAD: Bảng orders khổng lồ chứa toàn bộ lịch sử
SELECT * FROM orders WHERE created_at >= '2024-01-01';
-- Dù có index, MySQL vẫn phải xử lý bảng cực lớn
```

```sql
-- ✅ GOOD Strategy 1: Range Partitioning theo ngày
ALTER TABLE orders PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
-- Query theo year chỉ scan partition tương ứng

-- ✅ GOOD Strategy 2: Archive data cũ
INSERT INTO orders_archive
SELECT * FROM orders WHERE created_at < '2022-01-01';

DELETE FROM orders WHERE created_at < '2022-01-01';
-- Bảng active nhỏ gọn hơn → query nhanh hơn

-- ✅ GOOD Strategy 3: Partition by RANGE COLUMNS (MySQL 5.5+)
ALTER TABLE orders PARTITION BY RANGE COLUMNS(created_at) (
    PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
    PARTITION p2024_q1 VALUES LESS THAN ('2024-04-01'),
    PARTITION p2024_q2 VALUES LESS THAN ('2024-07-01'),
    PARTITION p_future VALUES LESS THAN (MAXVALUE)
);
```

---

## 17. Transaction — Tránh giữ transaction quá lâu

**Vấn đề:** Transaction mở quá lâu → giữ lock → block các query khác → deadlock lan rộng.

```python
# ❌ BAD: Transaction mở nhưng quên commit (Python example)
conn.begin()
result = conn.execute("SELECT * FROM users WHERE id = 1")
# Developer đi ăn trưa... transaction vẫn mở!
# 47 phút sau: mọi query liên quan đến bảng users bị block
```

```python
# ✅ GOOD: Dùng context manager, auto-commit/rollback
with conn.begin():
    result = conn.execute("SELECT * FROM users WHERE id = 1")
    # Auto-commit khi ra khỏi block, auto-rollback khi có exception
```

```sql
-- Phát hiện transaction chạy quá lâu
SELECT 
    id, 
    user, 
    host, 
    db,
    command,
    time AS seconds_running,
    state,
    info AS query
FROM information_schema.processlist
WHERE command != 'Sleep' 
  AND time > 300  -- Hơn 5 phút
ORDER BY time DESC;

-- Set timeout để tự động kill query quá chậm (MySQL)
SET SESSION wait_timeout = 300;
SET SESSION interactive_timeout = 300;

-- Kill query đang chạy quá lâu
KILL QUERY <process_id>;
```

---

## 18. Cập nhật statistics thường xuyên (ANALYZE TABLE)

**Vấn đề:** Query optimizer dựa vào statistics để chọn execution plan. Statistics cũ → plan sai → query chậm.

**Triệu chứng:** Query chạy tốt 6 tháng, bỗng dưng chậm 100x mà không có gì thay đổi → statistics stale.

```sql
-- ❌ SYMPTOM: Query từng chạy 50ms, nay chạy 40 giây
SELECT * FROM orders 
WHERE created_at > '2024-01-01'
ORDER BY total_amount DESC
LIMIT 100;
-- Bảng tăng từ 1M lên 10M rows nhưng statistics chưa cập nhật
```

```sql
-- ✅ FIX: Cập nhật statistics
ANALYZE TABLE orders;
-- Query trở lại 50ms

-- ✅ Xem statistics hiện tại
SHOW TABLE STATUS LIKE 'orders';

-- ✅ Lên lịch ANALYZE định kỳ (trong event scheduler)
CREATE EVENT daily_analyze
ON SCHEDULE EVERY 1 DAY
STARTS '2024-01-01 02:00:00'
DO BEGIN
    ANALYZE TABLE orders;
    ANALYZE TABLE users;
    ANALYZE TABLE products;
END;
```

---

## 19. Bật Slow Query Log để phát hiện query chậm

**Đây là bước đầu tiên trong mọi công việc optimize production database.**

```sql
-- Kiểm tra trạng thái slow query log
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';

-- Bật slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- Log query > 1 giây
SET GLOBAL log_queries_not_using_indexes = 'ON';  -- Log query không dùng index

-- Xem file log đang ở đâu
SHOW VARIABLES LIKE 'slow_query_log_file';
```

**Trong `my.cnf` (cấu hình vĩnh viễn):**
```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 1
log_queries_not_using_indexes = 1
log_slow_admin_statements = 1
```

```sql
-- Xem các query chậm nhất qua Performance Schema
SELECT 
    DIGEST_TEXT                              AS query,
    COUNT_STAR                               AS executions,
    ROUND(AVG_TIMER_WAIT / 1000000000, 3)   AS avg_time_sec,
    ROUND(SUM_TIMER_WAIT / 1000000000, 3)   AS total_time_sec,
    SUM_ROWS_EXAMINED                        AS total_rows_examined,
    SUM_ROWS_SENT                            AS total_rows_sent
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = 'your_database'
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

---

## 20. Cấu hình InnoDB Buffer Pool

**Đây là cấu hình có tác động lớn nhất đến hiệu năng MySQL.**

**InnoDB Buffer Pool** là vùng bộ nhớ cache data và index pages. Càng nhiều data fit vào buffer pool → càng ít đọc từ disk.

```ini
# my.cnf — Cấu hình cho dedicated MySQL server
[mysqld]

# 70-80% tổng RAM của server
innodb_buffer_pool_size = 12G   # Ví dụ server 16GB RAM

# Chia thành nhiều instances nếu pool > 1GB (giảm contention)
innodb_buffer_pool_instances = 8

# Tăng redo log để giảm checkpoint frequency
innodb_log_file_size = 1G

# Flush method tốt nhất cho Linux
innodb_flush_method = O_DIRECT

# Tắt query cache (deprecated MySQL 8.0, overhead trên write-heavy)
query_cache_type = 0
query_cache_size = 0

# Tăng sort buffer nếu có nhiều ORDER BY lớn
sort_buffer_size = 4M
join_buffer_size = 8M
```

```sql
-- Kiểm tra buffer pool hit rate (nên > 99%)
SELECT 
    (1 - (
        SELECT variable_value FROM performance_schema.global_status 
        WHERE variable_name = 'Innodb_buffer_pool_reads'
    ) / (
        SELECT variable_value FROM performance_schema.global_status 
        WHERE variable_name = 'Innodb_buffer_pool_read_requests'
    )) * 100 AS buffer_pool_hit_rate;

-- Xem hiện trạng buffer pool
SHOW STATUS LIKE 'Innodb_buffer_pool%';
```

---

## Bảng tổng hợp nhanh (Quick Reference)

| # | Anti-pattern | Solution | Tác động |
|---|---|---|---|
| 1 | Không dùng EXPLAIN | `EXPLAIN ANALYZE query` | Chẩn đoán chính xác |
| 2 | `SELECT *` | Chỉ select cột cần | Giảm memory/network |
| 3 | Thiếu index | Composite index trên WHERE/JOIN/ORDER BY | **Tác động lớn nhất** |
| 4 | Function trong WHERE | So sánh trực tiếp | Dùng được index |
| 5 | `LIKE '%keyword%'` | `LIKE 'keyword%'` hoặc FULLTEXT | Dùng được index |
| 6 | `COUNT(*) > 0` | `EXISTS (SELECT 1 ...)` | Dừng sớm hơn |
| 7 | Không có LIMIT | Thêm LIMIT | Tránh over-fetch |
| 8 | OFFSET lớn | Cursor-based pagination | O(1) thay vì O(n) |
| 9 | JOIN trên string | JOIN trên integer FK có index | Giảm nested loop |
| 10 | N+1 queries | JOIN hoặc eager loading | N queries → 1-2 queries |
| 11 | Cartesian join | Explicit JOIN với điều kiện | Tránh bùng nổ dữ liệu |
| 12 | Nested subquery | JOIN / CTE | Giảm re-execution |
| 13 | Index không covering | Covering index | Bỏ qua data row read |
| 14 | Data type quá lớn | Dùng đúng type (TINYINT vs INT) | Tiết kiệm RAM/disk |
| 15 | Aggregation không filter | Thêm WHERE + summary table | Giảm rows scan |
| 16 | Bảng khổng lồ | Partitioning / Archive | Scan ít data hơn |
| 17 | Transaction dài | Context manager + timeout | Tránh deadlock |
| 18 | Statistics cũ | `ANALYZE TABLE` định kỳ | Plan optimizer tốt hơn |
| 19 | Không monitor | Slow query log + Performance Schema | Phát hiện sớm |
| 20 | Buffer pool nhỏ | 70-80% RAM cho buffer pool | Giảm disk I/O |

---

## Checklist trước khi deploy query mới

- [ ] Chạy `EXPLAIN ANALYZE` — không có `type: ALL` trên bảng lớn
- [ ] Chỉ SELECT cột cần thiết, không dùng `SELECT *`
- [ ] Có LIMIT cho mọi query trả danh sách
- [ ] Các cột trong WHERE, JOIN, ORDER BY đã được index
- [ ] Không bọc cột index trong function
- [ ] Không dùng `LIKE '%keyword%'` — cân nhắc FULLTEXT
- [ ] Kiểm tra N+1 với ORM (dùng `debugbar` hoặc query log)
- [ ] Transaction ngắn gọn, có timeout
- [ ] Pagination dùng cursor-based nếu có thể

---

## Tài nguyên tham khảo

- [MySQL 8.0 Reference — Query Optimization](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
- [MySQL 8.0 — Optimizing InnoDB Queries](https://dev.mysql.com/doc/refman/8.0/en/optimizing-innodb-queries.html)
- [Percona — MySQL Performance Blog](https://www.percona.com/blog/)
- [Use The Index, Luke — SQL Indexing Guide](https://use-the-index-luke.com/)
