# MongoDB – Kinh nghiệm & Lưu ý từ các Senior Developer

> Tổng hợp từ tài liệu chính thức MongoDB, kinh nghiệm cộng đồng và các chuyên gia database.  
> Cập nhật: 2026-03

---

## Mục lục

1. [Schema Design Anti-Patterns](#1-schema-design-anti-patterns)
2. [Query Anti-Patterns & Câu thay thế](#2-query-anti-patterns--câu-thay-thế)
3. [Indexing – Các lưu ý quan trọng](#3-indexing--các-lưu-ý-quan-trọng)
4. [Aggregation Pipeline](#4-aggregation-pipeline)
5. [Pagination – Skip thay bằng cursor](#5-pagination--skip-thay-bằng-cursor)
6. [Transaction & Write/Read Concern](#6-transaction--writeread-concern)
7. [Sharding](#7-sharding)
8. [Connection & Performance vận hành](#8-connection--performance-vận-hành)
9. [Checklist review trước khi lên Production](#9-checklist-review-trước-khi-lên-production)

---

## 1. Schema Design Anti-Patterns

### 1.1 Unbounded Arrays (Mảng không giới hạn)

**Vấn đề:** Mảng tăng không có giới hạn → vượt giới hạn 16MB/document, index bị phình to, write chậm dần.

```js
// ❌ Tránh: mảng comments tăng mãi theo thời gian
{
  _id: ObjectId,
  title: "Post A",
  comments: [ /* có thể lên tới hàng nghìn phần tử */ ]
}
```

**Thay thế:** Tách sang collection riêng, dùng reference.

```js
// ✅ Tách riêng
// collection: posts
{ _id: ObjectId, title: "Post A" }

// collection: comments
{ _id: ObjectId, postId: ObjectId, content: "...", createdAt: Date }
```

---

### 1.2 Deep Nesting (Lồng quá sâu)

**Vấn đề:** Lồng sâu nhiều tầng → query phức tạp, khó index, khó bảo trì.

```js
// ❌ Tránh
{ user: { address: { city: { district: { ward: "..." } } } } }
```

**Thay thế:** Làm phẳng schema, dùng prefix hoặc reference.

```js
// ✅
{ "address.city": "HCM", "address.district": "Q1", "address.ward": "P1" }
```

---

### 1.3 Bloated Documents (Document quá lớn)

**Vấn đề:** Document chứa quá nhiều field không cần thiết → working set lớn, tốn RAM, FETCH chậm.

**Thay thế:**
- Bỏ field không cần thiết
- Tách data ít dùng sang collection riêng (Pattern: **Subset**)
- Dùng projection khi query

---

### 1.4 Dùng quá nhiều Collection

**Vấn đề:** Mỗi entity nhỏ tạo 1 collection riêng → storage engine phải quản lý nhiều namespace, metadata lock tăng.

**Thay thế:** Gộp các entity có cùng lifecycle, query pattern vào chung 1 collection, phân biệt bằng field `type`.

---

### 1.5 Embedded vs Reference – Quy tắc chọn

| Nên **Embed** khi | Nên **Reference** khi |
|-------------------|-----------------------|
| Quan hệ 1:1, 1:few | Quan hệ 1:many, 1:squillions |
| Dữ liệu luôn truy cập cùng nhau | Dữ liệu truy cập độc lập |
| Dữ liệu ổn định, ít cập nhật | Dữ liệu thay đổi thường xuyên |
| Tối ưu **read** | Tối ưu **write** hoặc tránh duplicate |
| Document không vượt 16MB | Nguy cơ vượt 16MB |

---

## 2. Query Anti-Patterns & Câu thay thế

### 2.1 `$where` – JavaScript trong query

**Vấn đề:**
- **Không dùng được index** – scan toàn bộ collection
- Chạy JavaScript trên từng document → rất chậm
- Rủi ro bảo mật (injection), gây global JavaScript lock

```js
// ❌ Tránh
db.orders.find({ $where: "this.totalPrice > this.discount" })
```

**Thay thế:** Dùng `$expr` (native, hỗ trợ aggregation expression, **dùng được index**)

```js
// ✅
db.orders.find({ $expr: { $gt: ["$totalPrice", "$discount"] } })
```

---

### 2.2 `$lookup` – Dùng tràn lan như JOIN của SQL

**Vấn đề:**
- Đọc nhiều collection, không locality
- Nếu collection lớn + không có index → cực chậm
- Quá nhiều `$lookup` trong 1 pipeline → memory spike

**Chiến lược thay thế:**

| Pattern | Khi nào dùng |
|---------|-------------|
| **Embedded Document** | Dữ liệu nhỏ, luôn đọc cùng nhau |
| **Extended Reference** | Chỉ nhúng các field hay dùng (name, title, price). Tránh nhúng field hay thay đổi |
| **Subset Pattern** | Nhúng một subset (vd: 10 comment mới nhất), phần còn lại để trong collection riêng |

**Nếu bắt buộc dùng `$lookup`:**

```js
// ✅ Luôn có index trên localField và foreignField
db.orders.createIndex({ customerId: 1 })
db.customers.createIndex({ _id: 1 })  // _id đã có sẵn

// ✅ $match trước $lookup để giảm document đầu vào
db.orders.aggregate([
  { $match: { status: "pending", createdAt: { $gte: startDate } } },  // Filter đầu tiên
  { $lookup: { from: "customers", localField: "customerId", foreignField: "_id", as: "customer" } }
])
```

---

### 2.3 N+1 Query Problem

**Vấn đề:** 1 query lấy list → loop qua từng item để query thêm → N query con.

```js
// ❌ N+1
const orders = await Order.find({ status: "active" })
for (const order of orders) {
  order.customer = await Customer.findById(order.customerId) // N queries!
}
```

**Thay thế:**

```js
// ✅ Batch query
const customerIds = orders.map(o => o.customerId)
const customers = await Customer.find({ _id: { $in: customerIds } })
const customerMap = Object.fromEntries(customers.map(c => [c._id.toString(), c]))
orders.forEach(o => { o.customer = customerMap[o.customerId.toString()] })
```

---

### 2.4 `$ne` / `$nin` – Selectivity thấp

**Vấn đề:** Toán tử phủ định match rất nhiều document → không dùng index hiệu quả → gần như scan toàn collection.

```js
// ❌ Không hiệu quả trên collection lớn
db.tasks.find({ status: { $ne: "completed" } })
db.users.find({ role: { $nin: ["admin", "moderator"] } })
```

**Thay thế:** Thiết kế schema theo hướng **positive matching**

```js
// ✅ Đổi chiều query – index trên field mới
// Thêm field isActive = true cho document cần query
db.tasks.find({ isActive: true })

// ✅ Nếu buộc dùng $ne, kết hợp với điều kiện equality có selectivity cao trước
db.tasks.find({ assigneeId: userId, status: { $ne: "completed" } })
```

---

### 2.5 Không dùng Projection – Lấy toàn bộ field

**Vấn đề:** Trả về toàn bộ document khi chỉ cần vài field → tốn băng thông, serialization chậm.

```js
// ❌ Lấy toàn bộ document
const users = await User.find({ active: true })
```

**Thay thế:**

```js
// ✅ Chỉ lấy field cần thiết
const users = await User.find({ active: true }, { name: 1, email: 1, _id: 0 })

// ✅ Tốt nhất: Covered Query – toàn bộ fields cần thiết đều nằm trong index
// Khi đó totalDocsExamined = 0
db.users.createIndex({ active: 1, name: 1, email: 1 })
```

---

### 2.6 `$in` với array quá lớn

**Vấn đề:** `$in` với hàng trăm đến hàng nghìn phần tử → query planner bị chậm, index scan nhiều điểm.

```js
// ❌ Tránh nếu array > vài trăm phần tử
db.products.find({ _id: { $in: [id1, id2, /* ...1000 ids... */] } })
```

**Thay thế:**
- Chia batch query (vd: 100 items/batch)
- Thiết kế lại schema để tránh cần query nhiều ID riêng lẻ
- `$in` < 201 phần tử: MongoDB xử lý như equality (tốt); ≥ 201: xử lý như range query (kém hơn)

---

### 2.7 Sort không có Index – In-memory sort

**Vấn đề:** Sort trên field không được index → MongoDB phải load toàn bộ kết quả vào RAM để sort → giới hạn 100MB, vượt quá sẽ lỗi.

```js
// ❌ Nếu không có index trên createdAt
db.logs.find({ userId: "123" }).sort({ createdAt: -1 }).limit(20)
```

**Thay thế:**

```js
// ✅ Tạo compound index theo ESR rule
db.logs.createIndex({ userId: 1, createdAt: -1 })
```

---

### 2.8 `$regex` không có anchor

**Vấn đề:** Regex không bắt đầu bằng `^` → không dùng được index, scan toàn bộ.

```js
// ❌ Contains search – không dùng index
db.products.find({ name: /shirt/ })

// ❌ Case-insensitive không có anchor cũng không dùng index
db.products.find({ name: /shirt/i })
```

**Thay thế:**

```js
// ✅ Prefix search – dùng được index
db.products.find({ name: /^shirt/ })

// ✅ Full-text search – dùng text index
db.products.createIndex({ name: "text", description: "text" })
db.products.find({ $text: { $search: "shirt" } })
```

---

### 2.9 `countDocuments()` không có filter trên collection lớn

**Vấn đề:** `countDocuments({})` không filter → scan toàn bộ collection.

```js
// ❌ Scan toàn collection
const total = await Order.countDocuments({})
```

**Thay thế:**

```js
// ✅ Dùng estimatedDocumentCount() nếu cần count tổng (nhanh, dùng metadata)
const total = await Order.estimatedDocumentCount()

// ✅ Nếu cần count chính xác với filter, đảm bảo filter field có index
const active = await Order.countDocuments({ status: "active" })
```

---

## 3. Indexing – Các lưu ý quan trọng

### 3.1 Quy tắc ESR cho Compound Index

**Thứ tự fields trong compound index:**

```
E → S → R
Equality → Sort → Range
```

| Loại điều kiện | Ví dụ |
|----------------|-------|
| **Equality** | `status: "active"`, `userId: "abc"` |
| **Sort** | `.sort({ createdAt: -1 })` |
| **Range** | `$gte`, `$lt`, `$in` (≥201 phần tử), `$regex`, `$ne`, `$nin` |

**Ví dụ thực tế:**

```js
// Query: userId = "abc", status = "active", sort by createdAt DESC, filter createdAt > lastMonth
db.orders.find({ userId: "abc", status: "active", createdAt: { $gte: lastMonth } })
         .sort({ createdAt: -1 })

// ✅ Compound index đúng thứ tự ESR
db.orders.createIndex({ userId: 1, status: 1, createdAt: -1 })
//                       E             E          S+R (sort và range cùng 1 field)
```

**Ngoại lệ ESR → ERS:**  
Nếu range rất selective (ít kết quả) → đặt range trước sort để giảm số doc cần sort.

---

### 3.2 Không tạo Index trùng lặp / thừa

```js
// ❌ Index đơn này thừa nếu đã có compound index { userId: 1, status: 1 }
db.orders.createIndex({ userId: 1 })

// Compound index { userId: 1, status: 1 } đã "cover" query trên userId đơn lẻ
```

Dùng lệnh sau để tìm index không được dùng:
```js
db.runCommand({ serverStatus: 1 }).indexCounters
// Hoặc dùng Atlas Performance Advisor
```

---

### 3.3 Multikey Index – Cẩn thận với Array lớn

**Vấn đề:** Index trên field array → MongoDB tạo index entry cho từng phần tử → array 1000 phần tử = 1000 index entries → write chậm.

```js
// ⚠️ Cẩn thận nếu tags có nhiều phần tử
db.products.createIndex({ tags: 1 })
```

**Giới hạn Compound Multikey Index:**
> Trong 1 compound index, **chỉ được phép có tối đa 1 field là array** trong cùng 1 document.

---

### 3.4 Covered Query – Tận dụng tối đa

Covered query: tất cả fields cần thiết (filter + sort + return) đều nằm trong index → `totalDocsExamined = 0` → cực nhanh.

```js
// Index bao phủ toàn bộ fields cần dùng
db.users.createIndex({ status: 1, email: 1, name: 1 })

// Query + projection đúng → covered
db.users.find({ status: "active" }, { email: 1, name: 1, _id: 0 })
// Lưu ý: phải explicit exclude _id: 0 vì _id không nằm trong secondary index
```

---

### 3.5 Kiểm tra Query Plan với `explain()`

```js
// Luôn dùng executionStats để xem chi tiết
db.orders.find({ userId: "abc" }).explain("executionStats")

// Chú ý các chỉ số:
// - stage: "COLLSCAN" → cần index
// - stage: "IXSCAN"   → đang dùng index
// - stage: "FETCH"    → sau IXSCAN phải đọc document gốc (chưa covered)
// - totalDocsExamined: 0 → covered query (tốt nhất)
// - executionTimeMillis → thời gian thực thi
// - nReturned vs totalDocsExamined → tỉ lệ thấp = index kém selective
```

---

## 4. Aggregation Pipeline

### 4.1 Thứ tự stage tối ưu: "Filter sớm, reshape muộn, join muộn"

```js
// ❌ Sai – join trước khi filter
db.orders.aggregate([
  { $lookup: { from: "customers", ... } },  // Xử lý toàn bộ orders
  { $match: { status: "pending" } }          // Filter sau → quá muộn
])

// ✅ Đúng
db.orders.aggregate([
  { $match: { status: "pending", createdAt: { $gte: startDate } } },  // Filter đầu
  { $lookup: { from: "customers", ... } },                             // Join sau
  { $project: { ... } }                                                // Reshape cuối
])
```

---

### 4.2 `$group` và `$sort` sớm khi cần

```js
// ✅ Sort + Limit trước Lookup nếu chỉ cần top N
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $sort: { createdAt: -1 } },
  { $limit: 100 },               // Giảm dataset trước khi lookup
  { $lookup: { ... } }
])
```

---

### 4.3 Tránh `$unwind` trên array lớn không cần thiết

**Vấn đề:** `$unwind` nhân bản document theo từng phần tử array → memory tăng đột biến.

```js
// ❌ Nếu items có 100 phần tử → 1 order thành 100 documents
{ $unwind: "$items" }

// ✅ Dùng $reduce, $map, $filter trong $project nếu không cần unwind
{ $addFields: {
    totalQty: { $sum: "$items.quantity" }
  }
}
```

---

### 4.4 Giới hạn 100MB RAM trong pipeline

Mặc định aggregation pipeline giới hạn 100MB RAM. Nếu vượt quá sẽ lỗi.

```js
// ✅ Cho phép dùng disk khi cần (chậm hơn nhưng không lỗi)
db.orders.aggregate([...], { allowDiskUse: true })
```

**Tốt hơn:** Tối ưu pipeline để không cần `allowDiskUse`.

---

## 5. Pagination – Skip thay bằng cursor

### 5.1 Vấn đề với `skip()` lớn

`skip(N)` phải duyệt qua N documents rồi bỏ → O(N) → trang 1.000 chậm gấp 1.000 lần trang 1.

```js
// ❌ Chậm tuyến tính khi page lớn
db.posts.find().sort({ createdAt: -1 }).skip(10000).limit(20)
```

### 5.2 Cursor-based Pagination (Keyset Pagination)

```js
// ✅ Page đầu tiên
const page1 = await db.posts.find()
  .sort({ createdAt: -1, _id: -1 })
  .limit(20)

// ✅ Page tiếp theo – dùng giá trị cuối của page trước
const lastDoc = page1[page1.length - 1]
const nextPage = await db.posts.find({
  $or: [
    { createdAt: { $lt: lastDoc.createdAt } },
    { createdAt: lastDoc.createdAt, _id: { $lt: lastDoc._id } }  // Tiebreaker
  ]
}).sort({ createdAt: -1, _id: -1 }).limit(20)
```

**Yêu cầu:** Luôn có compound index bao phủ sort fields.

```js
db.posts.createIndex({ createdAt: -1, _id: -1 })
```

| Phương pháp | Độ phức tạp | Dùng khi |
|-------------|-------------|---------|
| `skip().limit()` | O(N) | Dataset nhỏ < 10k, cần "jump to page X" |
| Keyset Pagination | O(log N) | Dataset lớn, infinite scroll, next/prev |

---

## 6. Transaction & Write/Read Concern

### 6.1 Giữ transaction ngắn

**Vấn đề:** Transaction mặc định timeout sau 60 giây. Transaction dài → WiredTiger cache không thể evict snapshot cũ → memory pressure.

```js
// ❌ Tránh I/O bên ngoài trong transaction
const session = client.startSession()
session.startTransaction()
try {
  await Order.create([orderData], { session })
  await fetch("https://payment-gateway.com/charge") // ❌ Network call trong transaction!
  await session.commitTransaction()
} catch (e) {
  await session.abortTransaction()
}
```

```js
// ✅ Chỉ đặt database operations trong transaction
const session = client.startSession()
session.startTransaction()
try {
  await Order.create([orderData], { session })
  await Inventory.updateOne({ productId }, { $inc: { qty: -1 } }, { session })
  await session.commitTransaction()
} catch (e) {
  await session.abortTransaction()
}
// Gọi payment gateway NGOÀI transaction
```

---

### 6.2 Write Concern & Read Concern

| Concern | Giá trị | Ý nghĩa |
|---------|---------|---------|
| `writeConcern: "majority"` | Recommended for important data | Đảm bảo majority replica đã ghi trước khi acknowledge |
| `readConcern: "majority"` | Cho dữ liệu nhất quán | Chỉ đọc data đã được majority confirm |
| `readConcern: "snapshot"` | Trong transaction trên sharded cluster | Đảm bảo consistent view |

---

## 7. Sharding

### 7.1 Tránh Monotonic Shard Key (Hotspot)

**Vấn đề:** Dùng ObjectId hoặc timestamp làm shard key → tất cả write đổ vào 1 shard (hotspot).

```js
// ❌ Monotonic key → hotspot
sh.shardCollection("db.events", { _id: 1 })        // ObjectId tăng dần
sh.shardCollection("db.logs", { createdAt: 1 })    // Timestamp tăng dần
```

**Thay thế:**

```js
// ✅ Hashed shard key – phân tán đều write
sh.shardCollection("db.events", { _id: "hashed" })

// ✅ Compound shard key – kết hợp field phân tán + field range query
sh.shardCollection("db.orders", { userId: 1, createdAt: 1 })
```

---

### 7.2 Chọn Shard Key tiêu chí

- **High cardinality** – nhiều giá trị khác nhau
- **Even distribution** – phân tán đều write/read
- **Non-monotonic** – không tăng dần theo thời gian
- **Query pattern** – nên match với query thường dùng để tránh scatter-gather

---

## 8. Connection & Performance vận hành

### 8.1 Luôn dùng Connection Pool

```js
// ❌ Tạo connection mới mỗi request (Lambda / serverless không cẩn thận)
app.get("/api", async (req, res) => {
  const client = new MongoClient(uri)
  await client.connect()
  // ...
  await client.close()
})

// ✅ Kết nối 1 lần, tái sử dụng
const client = new MongoClient(uri, { maxPoolSize: 10 })
await client.connect()

app.get("/api", async (req, res) => {
  const db = client.db("mydb")
  // ...
})
```

---

### 8.2 WiredTiger Cache Sizing

Mặc định MongoDB dùng **50% RAM - 1GB** cho WiredTiger cache. Nếu working set vượt cache → page fault → chậm đột ngột.

```yaml
# mongod.conf
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4  # Điều chỉnh theo RAM và working set
```

---

### 8.3 `$unset` vs Set null

```js
// $unset: xóa hoàn toàn field khỏi document → giảm document size
db.users.updateOne({ _id: id }, { $unset: { tempToken: "" } })

// $set null: giữ field nhưng giá trị null → document size không đổi
db.users.updateOne({ _id: id }, { $set: { tempToken: null } })
```

> Dùng `$unset` khi field không còn cần thiết để tránh document phình to theo thời gian.

---

### 8.4 Bật Slow Query Log

```js
// Bật profiler level 1: log query chậm hơn 100ms
db.setProfilingLevel(1, { slowms: 100 })

// Xem slow queries
db.system.profile.find().sort({ ts: -1 }).limit(10)
```

---

## 9. Checklist review trước khi lên Production

```
Schema Design
[ ] Không có unbounded array
[ ] Document size ước tính < 1MB (an toàn) / tối đa 16MB
[ ] Không có nesting quá 3 tầng
[ ] Đã cân nhắc embed vs reference dựa trên query pattern

Indexing
[ ] Mọi field trong filter/sort đều có index
[ ] Compound index theo thứ tự ESR
[ ] Không có index trùng lặp / thừa
[ ] Index trên large array field đã được cân nhắc kỹ
[ ] Đã chạy explain("executionStats") cho các query chính

Query
[ ] Không dùng $where
[ ] Không dùng skip() lớn → dùng keyset pagination
[ ] Luôn dùng projection để tránh fetch field thừa
[ ] $lookup có index đầy đủ hoặc đã denormalize
[ ] $ne/$nin trên collection lớn đã có alternative
[ ] $in array không quá vài trăm phần tử

Aggregation
[ ] $match đứng đầu pipeline
[ ] $lookup / $unwind đặt sau $match, $limit
[ ] Không cần allowDiskUse (nếu cần → cần tối ưu thêm)

Operations
[ ] Transaction ngắn, không có network call bên trong
[ ] Connection pool được cấu hình đúng
[ ] WiredTiger cache đủ cho working set
[ ] Slow query log được bật
[ ] Monotonic shard key đã được xử lý (nếu dùng sharding)
```

---

## Tài liệu tham khảo

- [MongoDB Schema Design Anti-Patterns](https://www.mongodb.com/docs/manual/data-modeling/design-antipatterns/)
- [ESR Guideline](https://www.mongodb.com/docs/v7.0/tutorial/equality-sort-range-guideline/)
- [Performance Best Practices: Indexing](https://www.mongodb.com/blog/post/performance-best-practices-indexing)
- [Performance Best Practices: Transactions](https://www.mongodb.com/blog/post/performance-best-practices-transactions-and-read-write-concerns)
- [Aggregation Pipeline Optimization](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-optimization/)
- [6 Rules of Thumb for MongoDB Schema Design](https://www.mongodb.com/blog/post/6-rules-of-thumb-for-mongodb-schema-design)
- [Reduce $lookup Operations](https://www.mongodb.com/docs/atlas/schema-suggestions/reduce-lookup-operations/)
