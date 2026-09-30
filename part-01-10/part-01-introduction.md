# บทที่ 1: บทนำ — Database Cluster คืออะไร และทำไมต้องใช้

> **หลักสูตร:** PostgreSQL + Redis + S3/MinIO Database Cluster  
> **ระดับ:** เริ่มต้น → ขั้นกลาง  
> **เวลาเรียน:** 3-4 ชั่วโมง

---

## สารบัญ

1. [Database Cluster คืออะไร](#1-database-cluster-คืออะไร)
2. [Single vs Cluster Architecture](#2-single-vs-cluster-architecture)
3. [เปรียบเทียบ Database ประเภทต่างๆ](#3-เปรียบเทียบ-database-ประเภทตางๆ)
4. [Use Cases จริงในอุตสาหกรรม](#4-use-cases-จริงในอุตสาหกรรม)
5. [Scalability Patterns](#5-scalability-patterns)
6. [CAP Theorem](#6-cap-theorem)
7. [ACID vs BASE](#7-acid-vs-base)
8. [Tech Stack บริษัทใหญ่](#8-tech-stack-บริษัทใหญ่)
9. [Prerequisites หลักสูตร](#9-prerequisites-หลักสูตร)
10. [Setup Environment](#10-setup-environment)
11. [Workshop](#11-workshop)

---

## 1. Database Cluster คืออะไร

**Database Cluster** คือการนำ database หลายๆ ตัวมาทำงานร่วมกันเป็นระบบเดียว เพื่อเพิ่มความสามารถในการรับมือกับข้อมูลขนาดใหญ่ ปริมาณ request สูง และความทนทานต่อความล้มเหลว

### ทำไมต้องใช้ Database Cluster?

ลองนึกภาพร้านอาหารเล็กๆ ที่มีพนักงานเสิร์ฟคนเดียว — เมื่อลูกค้าน้อย ทุกอย่างราบรื่น แต่เมื่อมีลูกค้าเต็มร้าน พนักงานคนเดียวรับมือไม่ได้ Database ก็เช่นกัน

**ปัญหาของ Single Database:**
- **Single Point of Failure:** ถ้า server ล่ม ทั้งระบบหยุดทำงาน
- **Limited Capacity:** RAM และ CPU ของ server เดียวมีจำกัด
- **Geographic Latency:** ถ้า server อยู่ที่อเมริกา แต่ user อยู่ที่เอเชีย ก็จะช้า
- **Maintenance Downtime:** ต้องหยุดระบบเพื่ออัปเดตหรือซ่อมบำรุง

**ประโยชน์ของ Database Cluster:**
- **High Availability (HA):** ถ้า node หนึ่งล่ม node อื่นยังทำงานต่อได้
- **Horizontal Scalability:** เพิ่ม node ใหม่ได้ตามต้องการ
- **Load Balancing:** กระจาย query ไปหลาย node ลดภาระแต่ละตัว
- **Geographic Distribution:** วาง node ใกล้ user แต่ละภูมิภาค
- **Zero-Downtime Maintenance:** อัปเดตทีละ node โดยไม่หยุดระบบ

---

## 2. Single vs Cluster Architecture

### 2.1 Single Database Architecture

```
┌─────────────────────────────────────────┐
│           Single Database Server        │
│                                         │
│  ┌──────────┐    ┌──────────────────┐  │
│  │ App 1    │───▶│                  │  │
│  └──────────┘    │   PostgreSQL     │  │
│  ┌──────────┐    │   (Single Node)  │  │
│  │ App 2    │───▶│                  │  │
│  └──────────┘    │   /data          │  │
│  ┌──────────┐    │   RAM: 16GB      │  │
│  │ App 3    │───▶│   CPU: 8 cores   │  │
│  └──────────┘    └──────────────────┘  │
│                                         │
│  ❌ ถ้า server ล่ม = ทุกอย่างหยุด       │
└─────────────────────────────────────────┘
```

**ข้อจำกัด:**
- ขยายได้เฉพาะแนวตั้ง (Vertical Scaling) — ซื้อ RAM/CPU เพิ่ม
- เมื่อ server เต็ม ต้องย้ายไป server ใหม่ทั้งหมด
- Backup ขณะที่ system ทำงานอยู่อาจมีปัญหา

### 2.2 Primary-Replica Cluster (ขั้นพื้นฐาน)

```
                    ┌─────────────────┐
  Write Requests    │                 │   Read Requests
        ┌──────────▶│  PRIMARY NODE   │◀──────┐
        │           │  (Read+Write)   │       │
        │           │                 │       │
        │           └────────┬────────┘       │
        │                    │                │
        │           WAL Streaming             │
        │           Replication               │
        │                    │                │
  ┌─────┴─────┐    ┌─────────▼───────┐   ┌───┴───────┐
  │           │    │                 │   │           │
  │  App 1    │    │  REPLICA 1      │   │  App 2    │
  │           │    │  (Read-Only)    │   │           │
  └─────┬─────┘    └─────────────────┘   └───────────┘
        │
        │           ┌─────────────────┐
        │           │                 │
        └──────────▶│  REPLICA 2      │
                    │  (Read-Only)    │
                    └─────────────────┘

  ✅ Read requests กระจายไป Replicas
  ✅ ถ้า Primary ล่ม Replica รับหน้าที่แทนได้ (Failover)
  ✅ สามารถทำ Backup บน Replica โดยไม่กระทบ Primary
```

### 2.3 Full Cluster Architecture (Production)

```
┌─────────────────────────────────────────────────────────────┐
│                    Load Balancer / HAProxy                    │
│                  (192.168.1.1 : 5000)                        │
└──────────────────────┬──────────────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │PostgreSQL│  │PostgreSQL│  │PostgreSQL│
   │Primary   │  │Replica 1 │  │Replica 2 │
   │:5432     │  │:5432     │  │:5432     │
   └──────────┘  └──────────┘  └──────────┘
         │              │              │
         └──────────────┴──────────────┘
                        │
                        ▼
              ┌─────────────────┐
              │  Shared Storage │
              │  or WAL Archive │
              │  (S3/MinIO)     │
              └─────────────────┘

   ┌─────────────────────────────────────────┐
   │              Redis Cluster               │
   │                                          │
   │  Master-1 ◀──▶ Replica-1               │
   │     │                                    │
   │  Master-2 ◀──▶ Replica-2               │
   │     │                                    │
   │  Master-3 ◀──▶ Replica-3               │
   └─────────────────────────────────────────┘

   ┌─────────────────────────────────────────┐
   │            MinIO Object Storage          │
   │                                          │
   │  Node-1  Node-2  Node-3  Node-4          │
   │    │       │       │       │             │
   │    └───────┴───────┴───────┘             │
   │         Erasure Coding                   │
   └─────────────────────────────────────────┘
```

---

## 3. เปรียบเทียบ Database ประเภทต่างๆ

### 3.1 ตารางเปรียบเทียบหลัก

| Feature | PostgreSQL | MySQL | MongoDB | Redis | S3/MinIO |
|---------|-----------|-------|---------|-------|----------|
| ประเภท | Relational | Relational | Document | Key-Value/Cache | Object Storage |
| ACID | ✅ Full | ✅ InnoDB | ✅ v4.0+ | ✅ Partial | ❌ |
| Schema | Strict | Strict | Flexible | Flexible | None |
| SQL | ✅ Full | ✅ Full | ❌ | ❌ | ❌ |
| JSON Support | ✅ Native JSONB | ✅ Limited | ✅ Native | ✅ JSON type | ✅ Metadata |
| Full-Text Search | ✅ Built-in | ✅ Limited | ✅ Atlas | ❌ | ❌ |
| Geospatial | ✅ PostGIS | ✅ Limited | ✅ 2dsphere | ✅ GEO commands | ❌ |
| Clustering | ✅ Patroni/Citus | ✅ Galera | ✅ Native Sharding | ✅ Redis Cluster | ✅ Native |
| Max DB Size | Unlimited | 64TB | Unlimited | RAM-limited | Unlimited |
| License | MIT-like | GPL/Commercial | SSPL | BSD | Apache 2.0 |
| Thai Usage | สูงมาก | สูงมาก | ปานกลาง | สูง | ปานกลาง |

### 3.2 เมื่อไหร่ควรใช้อะไร?

#### PostgreSQL — เหมาะสำหรับ:
```
✅ ข้อมูลมีโครงสร้าง (structured data)
✅ ต้องการ transactions ที่ซับซ้อน
✅ Reporting และ Analytics
✅ Financial systems ที่ต้องการความแม่นยำ
✅ ข้อมูลที่มี relationships ซับซ้อน
✅ ต้องการ stored procedures
✅ เมื่อต้องการ JSON + SQL ผสมกัน
```

#### Redis — เหมาะสำหรับ:
```
✅ Session management
✅ Caching (ลด load บน PostgreSQL)
✅ Rate limiting
✅ Real-time leaderboards
✅ Pub/Sub messaging
✅ Job queues
✅ Temporary data ที่ต้องการเร็วมาก
```

#### S3/MinIO — เหมาะสำหรับ:
```
✅ รูปภาพและ media files
✅ Backups และ archives
✅ Log files ขนาดใหญ่
✅ Static website hosting
✅ Data lake
✅ ML model artifacts
✅ PDF, Excel, documents
```

### 3.3 ตัวอย่าง Architecture ที่ใช้ทั้งสาม

```
สมมติว่าเราสร้าง E-commerce Platform:

┌─────────────────────────────────────────────────────────┐
│                    User Request                          │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                  API Gateway / Nginx                     │
└────────┬─────────────┬────────────────┬─────────────────┘
         │             │                │
         ▼             ▼                ▼
   ┌──────────┐  ┌──────────┐    ┌──────────┐
   │ User     │  │ Product  │    │ File     │
   │ Service  │  │ Service  │    │ Service  │
   └────┬─────┘  └────┬─────┘    └────┬─────┘
        │              │               │
        ▼              ▼               ▼
   ┌──────────┐  ┌──────────┐    ┌──────────┐
   │  Redis   │  │PostgreSQL│    │  MinIO   │
   │ Sessions │  │ Products │    │  Images  │
   │ Cache    │  │ Orders   │    │  Files   │
   │ Cart     │  │ Users    │    │  Backups │
   └──────────┘  └──────────┘    └──────────┘
```

---

## 4. Use Cases จริงในอุตสาหกรรม

### 4.1 E-Commerce Platform

**ปัญหา:** วันหยุดพิเศษ (เช่น 11.11) มี traffic พุ่งสูงถึง 100x จากปกติ

**Solution Architecture:**
```sql
-- PostgreSQL: เก็บ transaction data ที่ต้องการ ACID
-- ตาราง orders ต้องการ consistency อย่างเคร่งครัด
CREATE TABLE orders (
    order_id    UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL,
    status      VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Redis: เก็บ shopping cart (temporary, เร็ว)
-- HSET cart:user_123 product_456 {"qty": 2, "price": 299}
-- EXPIRE cart:user_123 3600  -- cart หมดอายุใน 1 ชั่วโมง

-- MinIO: เก็บรูปสินค้า, PDF invoices
-- bucket: products-images/category/product_id/main.jpg
-- bucket: invoices/2024/01/INV-000001.pdf
```

**Scaling Strategy:**
- PostgreSQL Primary + 2 Replicas (read replicas for product browsing)
- Redis Cluster 6 nodes (3 masters + 3 replicas) for cart/session
- MinIO 4 nodes with erasure coding for product images

### 4.2 Social Media Platform

**ปัญหา:** มี feed, notifications, messages ที่ต้องแสดงผลแบบ real-time

**Data Flow:**
```
User โพสต์รูป:
1. Upload รูปไปยัง MinIO → ได้ URL
2. บันทึก post metadata ใน PostgreSQL
3. Publish event ไปยัง Redis Pub/Sub
4. Redis Pub/Sub แจ้ง followers แบบ real-time
5. Redis Cache เก็บ feed ของแต่ละ user (Fan-out)

User ดู Feed:
1. ตรวจสอบ Redis Cache ก่อน
2. ถ้าไม่มีใน cache → query PostgreSQL
3. บันทึก result ลง Redis cache (TTL 5 นาที)
4. Return feed พร้อม image URLs จาก MinIO
```

### 4.3 FinTech / Banking

**ปัญหา:** ต้องการ ACID ที่เข้มงวด แต่ก็ต้องการ performance สูง

**Critical Requirements:**
```
- Consistency: ยอดเงินต้องถูกต้องเสมอ (ACID REQUIRED)
- Availability: ระบบต้องทำงาน 99.99% uptime
- Auditability: ทุก transaction ต้องมี audit trail

PostgreSQL:
- Primary DB: Transactions, Accounts, Ledger
- Synchronous Replication ไปยัง Replica (ไม่ยอม data loss)
- Point-in-Time Recovery สำหรับ compliance

Redis:
- Rate limiting (ป้องกัน fraud)
- Session management
- OTP caching (ระยะเวลาสั้น)

MinIO:
- เก็บ statements PDF
- KYC documents (ID card, selfie)
- Audit logs archives
```

### 4.4 Healthcare System

```
Patient Data:
- PostgreSQL: ประวัติผู้ป่วย, การรักษา, ยา (structured, HIPAA compliant)
- JSONB: ผลการตรวจที่มีโครงสร้างแตกต่างกัน
- Redis: Queue สำหรับนัดหมาย, real-time alerts
- MinIO: X-ray, MRI images, PDF reports (DICOM support)

ตัวอย่าง Query ที่ใช้:
```sql
-- ดูประวัติการรักษาผู้ป่วยในช่วงเวลาที่กำหนด
SELECT 
    p.patient_id,
    p.name,
    v.visit_date,
    v.diagnosis,
    v.lab_results  -- JSONB column
FROM patients p
JOIN visits v ON p.patient_id = v.patient_id
WHERE v.visit_date BETWEEN '2024-01-01' AND '2024-12-31'
  AND v.lab_results @> '{"glucose_high": true}'  -- JSONB query
ORDER BY v.visit_date DESC;
```

---

## 5. Scalability Patterns

### 5.1 Vertical Scaling (Scale Up)

```
Before:                    After:
┌──────────────┐          ┌──────────────┐
│ Server       │          │ Server       │
│ CPU: 4 cores │  ──────▶ │ CPU: 32 cores│
│ RAM: 8 GB    │          │ RAM: 128 GB  │
│ SSD: 500 GB  │          │ NVMe: 4 TB   │
└──────────────┘          └──────────────┘

장점:
✅ ง่าย ไม่ต้องเปลี่ยน application code
✅ ไม่มีปัญหา consistency
✅ Latency ต่ำ

ข้อเสีย:
❌ มีขีดจำกัด (hardware limit)
❌ ราคาแพงขึ้นแบบ exponential
❌ Single point of failure ยังมีอยู่
❌ Downtime ระหว่างอัปเกรด
```

### 5.2 Horizontal Scaling (Scale Out)

```
Before:
┌──────────────┐
│ PostgreSQL   │ ◀── 10,000 req/sec (overwhelmed)
│ Single Node  │
└──────────────┘

After (Read Scaling):
                    ┌──────────────┐
Write ──────────────▶│  Primary     │
                    └──────┬───────┘
                           │ Replication
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
        ┌─────────┐  ┌─────────┐  ┌─────────┐
Read ──▶│Replica 1│  │Replica 2│  │Replica 3│
        └─────────┘  └─────────┘  └─────────┘
        3,333 r/s    3,333 r/s    3,333 r/s

Total Read Capacity: ~10,000 req/sec ✅
```

### 5.3 Database Sharding

```
แนวคิด: แบ่งข้อมูลออกเป็น "shard" หลายๆ ชิ้น

Shard ตาม User ID:
User ID 1-1M    → Shard 1 (PostgreSQL Node 1)
User ID 1M-2M   → Shard 2 (PostgreSQL Node 2)
User ID 2M-3M   → Shard 3 (PostgreSQL Node 3)

                ┌─────────────────┐
  Request       │  Shard Router   │
  user_id=     │  (PgBouncer /   │
  1,500,000    │   Application)  │
               └────────┬────────┘
                         │ user_id=1,500,000 → Shard 2
                         ▼
               ┌─────────────────┐
               │   PostgreSQL    │
               │   Shard 2       │
               │ (User 1M-2M)    │
               └─────────────────┘
```

**Sharding Strategies:**
```
1. Range Sharding: ตาม ID range (ง่าย แต่อาจ uneven)
2. Hash Sharding: hash(user_id) % N (สม่ำเสมอ แต่ range queries ยาก)
3. Directory Sharding: มี lookup table (lifelike แต่ lookup overhead)
4. Geographic Sharding: ตาม region (ลด latency แต่ซับซ้อน)
```

### 5.4 CQRS Pattern (Command Query Responsibility Segregation)

```
┌─────────────┐          ┌─────────────────┐
│   Command   │          │   Write Model   │
│  (Write)    │─────────▶│   PostgreSQL    │
│  Create     │          │   Primary       │
│  Update     │          └────────┬────────┘
│  Delete     │                   │ Event Sourcing
└─────────────┘                   │ / Replication
                                  ▼
┌─────────────┐          ┌─────────────────┐
│   Query     │          │   Read Model    │
│  (Read)     │─────────▶│   PostgreSQL    │
│  GetUser    │          │   Replica       │
│  ListOrders │          │   (Optimized    │
│  SearchProd │          │    for reads)   │
└─────────────┘          └─────────────────┘

ประโยชน์:
- Write operations ไม่กระทบ Read performance
- Read model สามารถ optimize แยกกันได้
- Scaling แยกกันตาม workload
```

---

## 6. CAP Theorem

### อธิบายแบบเข้าใจง่าย

CAP Theorem บอกว่า ระบบ distributed ไม่สามารถมีคุณสมบัติทั้งสามอย่างพร้อมกันได้ 100%:

```
         C
        / \
       /   \
      /     \
     /  เลือก \
    /  ได้แค่  \
   /   2 จาก 3 \
  A─────────────P

C = Consistency (ความสม่ำเสมอ)
A = Availability (ความพร้อมใช้งาน)  
P = Partition Tolerance (ทนต่อการแตกแยกของเครือข่าย)
```

### คำอธิบายแต่ละข้อ:

**Consistency (C):** ทุก node เห็นข้อมูลเดียวกันในเวลาเดียวกัน
```
ตัวอย่าง: ถ้าโอนเงิน 100 บาท แล้ว query balance ทันที
ต้องเห็นว่า balance ลดลง 100 บาทเสมอ ไม่ว่าจะ query จาก node ไหน
```

**Availability (A):** ระบบตอบสนองต่อ request เสมอ (ไม่ timeout)
```
ตัวอย่าง: แม้บาง node จะล่ม ระบบยังตอบ request ได้
แต่อาจได้ข้อมูลเก่าบ้าง
```

**Partition Tolerance (P):** ระบบทำงานต่อได้แม้เครือข่ายระหว่าง node จะขาด
```
ตัวอย่าง: ถ้าเครือข่ายระหว่าง DataCenter A และ B ขาด
ระบบแต่ละ DC ยังทำงานได้
```

### CAP ของ Database แต่ละตัว:

```
CP Systems (Consistency + Partition Tolerance):
├── PostgreSQL (in cluster mode)
├── HBase
├── MongoDB (strong consistency mode)
└── Redis (in cluster mode, with WAIT)

AP Systems (Availability + Partition Tolerance):
├── CouchDB
├── Cassandra
├── DynamoDB
└── Redis (without WAIT, eventual consistency)

CA Systems (Consistency + Availability):
└── Traditional RDBMS บน single node (ไม่มี partition)
    เช่น PostgreSQL standalone, MySQL standalone
```

### ตัวอย่างในชีวิตจริง:

```
สถานการณ์: เครือข่ายระหว่าง Node A (กรุงเทพ) และ Node B (เชียงใหม่) ขาด

CP System (เช่น PostgreSQL with synchronous replication):
- Node B จะปฏิเสธ write operations
- ตอบ error: "cannot write, primary not reachable"  
- แต่ข้อมูลที่อ่านได้จะถูกต้องเสมอ
- เหมาะกับ: banking, payment systems

AP System (เช่น Cassandra):
- ทั้ง Node A และ B ยังรับ write ได้
- เมื่อเครือข่ายกลับมา จะ merge ข้อมูลทั้งสอง
- อาจมี conflict ที่ต้องแก้ไข
- เหมาะกับ: social media posts, user preferences
```

---

## 7. ACID vs BASE

### ACID (สำหรับ Relational Databases)

```
A = Atomicity    (ทำทั้งหมดหรือไม่ทำเลย)
C = Consistency  (ข้อมูลถูกต้องเสมอ)
I = Isolation    (transactions ไม่กวนกัน)
D = Durability   (ข้อมูลที่ commit แล้วไม่หาย)
```

**ตัวอย่าง Atomicity:**
```sql
-- การโอนเงิน: ต้องทำทั้งสองอย่างหรือไม่ทำเลย
BEGIN;
    UPDATE accounts SET balance = balance - 1000 WHERE user_id = 1;
    UPDATE accounts SET balance = balance + 1000 WHERE user_id = 2;
COMMIT;

-- ถ้า server ล่มระหว่าง transaction: Rollback อัตโนมัติ
-- ไม่มีทางที่ account 1 จะถูกหักแต่ account 2 ไม่ได้รับ
```

**ตัวอย่าง Isolation Levels:**
```sql
-- Read Uncommitted (ต่ำสุด): อ่านข้อมูลที่ยังไม่ commit ได้
-- Read Committed (default ใน PostgreSQL): อ่านได้เฉพาะที่ commit แล้ว
-- Repeatable Read: อ่านซ้ำในใน transaction เดียวได้ผลเหมือนเดิม
-- Serializable (สูงสุด): transactions ทำงานราวกับทำทีละอัน

SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
    -- ใน serializable mode จะไม่มี phantom reads
    SELECT * FROM products WHERE price > 100;
    -- ... ทำงานอื่นๆ ...
    SELECT * FROM products WHERE price > 100;  -- ผลเหมือนเดิมแน่นอน
COMMIT;
```

### BASE (สำหรับ NoSQL / Distributed Systems)

```
BA = Basically Available  (พร้อมใช้งานพื้นฐาน)
S  = Soft State           (state อาจเปลี่ยนได้โดยไม่มี input)
E  = Eventually Consistent (จะสม่ำเสมอในที่สุด)
```

**ตัวอย่าง Eventually Consistent:**
```
สถานการณ์: User โพสต์รูปบน Social Media

1. รูปอัปโหลดสำเร็จ → ผู้โพสต์เห็นรูปทันที
2. Replication ไปยัง DC อื่นๆ (ใช้เวลาไม่กี่วินาที)
3. ในช่วงนั้น user อื่นๆ อาจยังไม่เห็นรูป
4. หลังจากนั้นไม่นาน ทุกคนเห็นรูปเดียวกัน

นี่คือ "Eventually Consistent" — ข้อมูลจะสม่ำเสมอในที่สุด
แต่ไม่รับประกันว่าจะทันทีทันใด
```

### เมื่อไหร่ใช้อะไร:

```
ใช้ ACID เมื่อ:
✅ Financial transactions (โอนเงิน, ชำระเงิน)
✅ Inventory management (สต็อกสินค้า)
✅ Medical records
✅ เมื่อ data loss หรือ inconsistency มีผลกระทบรุนแรง

ใช้ BASE เมื่อ:
✅ Social media feeds
✅ Recommendation systems
✅ Analytics dashboards
✅ เมื่อ availability สำคัญกว่า perfect consistency
✅ เมื่อ scale สูงมากจนรักษา strong consistency ไม่ไหว
```

---

## 8. Tech Stack บริษัทใหญ่

### 8.1 Airbnb

```
Challenge: 7+ ล้าน listings ทั่วโลก, search ที่ต้องเร็วมาก

Stack:
- PostgreSQL: Core business data (listings, bookings, users, payments)
- Redis: Sessions, search query cache, rate limiting
- S3: Property photos, user avatars, documents
- Apache Hive + Presto: Analytics / Data Warehouse
- Elasticsearch: Full-text search สำหรับ property search

Architecture Pattern:
- Service-oriented architecture (SOA)
- ใช้ PgBouncer สำหรับ connection pooling
- Database sharding ตาม geographic region
- Multi-region deployment (US, EU, Asia)
```

### 8.2 Uber

```
Challenge: Real-time location tracking, surge pricing, matching algorithm

Stack:
- PostgreSQL → ย้ายไป Schemaless (MongoDB-like บน MySQL) → ปัจจุบันใช้ MySQL
- Redis: Real-time driver locations, surge calculations, ETA cache
- S3: Driver documents, trip receipts
- Kafka: Real-time event streaming

ข้อมูลที่น่าสนใจ:
- Uber เคยใช้ PostgreSQL แล้วย้ายไป MySQL เพราะปัญหา replication
- บทความ Uber ปี 2016 "Why Uber Engineering Switched from Postgres to MySQL"
  เป็นที่ถกเถียงมาก ในชุมชน PostgreSQL

Lesson learned:
- การ migrate database ขนาดใหญ่มีความเสี่ยงสูงมาก
- ทางเลือก (PostgreSQL vs MySQL) ขึ้นอยู่กับ use case และ expertise ของทีม
```

### 8.3 Shopify

```
Challenge: Black Friday traffic spikes (100x+), millions of stores

Stack:
- MySQL (Vitess): Core e-commerce data
- Redis: Sessions, shopping cart, rate limiting, pub/sub
- S3 (GCS): Product images, shop assets
- Kafka: Event streaming
- Elasticsearch: Product search

Architecture:
- Pod architecture: แบ่ง stores เป็น "pods" หลายๆ ชุด
- แต่ละ pod เป็น isolated cluster
- ถ้า pod หนึ่งมีปัญหา ไม่กระทบ pod อื่น
- Redis Cluster: หลาย hundreds ของ nodes

Key Insight:
"ไม่ใช่แค่เรื่อง database เดียว แต่เป็น ecosystem ของ services"
```

### 8.4 Netflix

```
Challenge: 250+ ล้าน subscribers, streaming content globally

Stack:
- PostgreSQL: User metadata, billing
- Cassandra: Viewing history, recommendations data
- Redis: A/B testing config, session tokens
- S3: Video content, thumbnails, metadata
- Elasticsearch: Content search

Key Insights:
- ใช้ active-active multi-region (2+ datacenters รับ traffic พร้อมกัน)
- Chaos Engineering: จงใจปิด servers เพื่อทดสอบ resilience
- Open-source contributions: Hystrix, Conductor, Spinnaker
```

---

## 9. Prerequisites หลักสูตร

### 9.1 ความรู้พื้นฐานที่ต้องมี

```
ระดับ 1 - จำเป็นต้องมี:
□ Command line / Terminal พื้นฐาน (cd, ls, mkdir, cat, nano/vim)
□ ความเข้าใจเรื่อง IP address, port, localhost
□ การติดตั้งและใช้งาน software พื้นฐาน
□ ภาษาโปรแกรมมิ่งอย่างน้อย 1 ภาษา (Python/Node.js/Go)

ระดับ 2 - มีจะดีมาก:
□ SQL พื้นฐาน (SELECT, INSERT, UPDATE, DELETE)
□ Docker พื้นฐาน
□ Git พื้นฐาน
□ Network concepts: TCP/IP, HTTP/HTTPS

ระดับ 3 - ไม่จำเป็น แต่เป็นประโยชน์:
□ Linux system administration
□ Cloud platforms (AWS, GCP, Azure)
□ Kubernetes พื้นฐาน
```

### 9.2 Hardware Requirements

```
Minimum (พอทำ workshop ได้):
- CPU: 4 cores
- RAM: 8 GB (16 GB แนะนำ)
- Storage: 50 GB free space (SSD แนะนำ)
- OS: Ubuntu 20.04+, macOS 12+, Windows 10+ with WSL2

Recommended (ทำ full cluster ได้):
- CPU: 8+ cores
- RAM: 16-32 GB
- Storage: 100+ GB SSD
- OS: Ubuntu 22.04 LTS หรือ macOS 13+
```

---

## 10. Setup Environment

### 10.1 Install Docker Desktop

#### Ubuntu/Debian:
```bash
# อัปเดต package list
sudo apt-get update

# ติดตั้ง dependencies
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# เพิ่ม Docker's official GPG key
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# เพิ่ม repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# ติดตั้ง Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# เพิ่ม user ปัจจุบันเข้า docker group (ไม่ต้อง sudo ทุกครั้ง)
sudo usermod -aG docker $USER

# Reload group (หรือ logout แล้ว login ใหม่)
newgrp docker

# ทดสอบ
docker --version
docker compose version
```

#### macOS:
```bash
# วิธีที่ 1: Download Docker Desktop จาก https://www.docker.com/products/docker-desktop/

# วิธีที่ 2: ใช้ Homebrew
brew install --cask docker

# เปิด Docker Desktop application
open /Applications/Docker.app

# รอให้ Docker daemon เริ่มทำงาน (icon ที่ menu bar หยุดกระพริบ)

# ทดสอบ
docker --version
docker compose version
```

### 10.2 First Docker Commands

```bash
# ทดสอบว่า Docker ทำงาน
docker run hello-world

# ดู output:
# Hello from Docker!
# This message shows that your installation appears to be working correctly.

# ดู images ที่มีในเครื่อง
docker images

# ดู containers ที่กำลังทำงาน
docker ps

# ดู containers ทั้งหมด (รวมที่หยุดแล้ว)
docker ps -a

# รัน PostgreSQL container เป็นครั้งแรก
docker run --name my-postgres \
    -e POSTGRES_PASSWORD=mypassword \
    -p 5432:5432 \
    -d postgres:16

# ดูว่า container รันอยู่
docker ps

# เข้าไปใช้ psql ใน container
docker exec -it my-postgres psql -U postgres

# ใน psql:
# \l   -- list databases
# \q   -- quit

# หยุด container
docker stop my-postgres

# ลบ container
docker rm my-postgres
```

### 10.3 Docker Compose Setup สำหรับหลักสูตรนี้

สร้างไฟล์ `docker-compose.yml` สำหรับใช้ตลอดหลักสูตร:

```yaml
# docker-compose.yml
version: '3.8'

services:
  # PostgreSQL Primary
  postgres-primary:
    image: postgres:16
    container_name: pg-primary
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin123
      POSTGRES_DB: mydb
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "5432:5432"
    volumes:
      - postgres_primary_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    networks:
      - db-cluster
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis
  redis:
    image: redis:7-alpine
    container_name: redis-single
    command: redis-server --requirepass redis123 --appendonly yes
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - db-cluster
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "redis123", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # MinIO (S3-compatible)
  minio:
    image: minio/minio:latest
    container_name: minio-single
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
    ports:
      - "9000:9000"   # API
      - "9001:9001"   # Console
    volumes:
      - minio_data:/data
    networks:
      - db-cluster
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  # pgAdmin (PostgreSQL GUI)
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin123
    ports:
      - "8080:80"
    networks:
      - db-cluster
    depends_on:
      - postgres-primary

  # Redis Commander (Redis GUI)
  redis-commander:
    image: rediscommander/redis-commander:latest
    container_name: redis-commander
    environment:
      REDIS_HOSTS: local:redis:6379:0:redis123
    ports:
      - "8081:8081"
    networks:
      - db-cluster
    depends_on:
      - redis

volumes:
  postgres_primary_data:
  redis_data:
  minio_data:

networks:
  db-cluster:
    driver: bridge
```

```bash
# รัน Stack ทั้งหมด
docker compose up -d

# ดูว่าทุก service รันอยู่
docker compose ps

# ดู logs
docker compose logs -f

# หยุดทั้งหมด
docker compose down

# หยุดและลบ volumes ด้วย (ระวัง! ข้อมูลจะหาย)
docker compose down -v
```

### 10.4 Verify Installation

```bash
# ทดสอบ PostgreSQL
docker exec -it pg-primary psql -U admin -d mydb -c "SELECT version();"

# Expected output:
# PostgreSQL 16.x on x86_64-pc-linux-gnu, compiled by gcc...

# ทดสอบ Redis
docker exec -it redis-single redis-cli -a redis123 ping

# Expected output:
# PONG

# ทดสอบ MinIO
curl http://localhost:9000/minio/health/live

# Expected: HTTP 200 OK

# เปิด pgAdmin: http://localhost:8080
# Login: admin@admin.com / admin123

# เปิด Redis Commander: http://localhost:8081

# เปิด MinIO Console: http://localhost:9001
# Login: minioadmin / minioadmin123
```

### 10.5 Environment Variables Setup

สร้างไฟล์ `.env` สำหรับจัดการ credentials:

```bash
# .env
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=admin
POSTGRES_PASSWORD=admin123
POSTGRES_DB=mydb
POSTGRES_URL=postgresql://admin:admin123@localhost:5432/mydb

REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=redis123
REDIS_URL=redis://:redis123@localhost:6379/0

MINIO_ENDPOINT=localhost:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin123
MINIO_BUCKET=mybucket

# Development settings
NODE_ENV=development
LOG_LEVEL=debug
```

```bash
# ทดสอบ connection ด้วย Python
pip install psycopg2-binary redis boto3

python3 << 'EOF'
import psycopg2
import redis
import boto3

# Test PostgreSQL
try:
    conn = psycopg2.connect(
        host="localhost",
        port=5432,
        user="admin",
        password="admin123",
        database="mydb"
    )
    cur = conn.cursor()
    cur.execute("SELECT version();")
    print(f"✅ PostgreSQL: {cur.fetchone()[0][:50]}...")
    conn.close()
except Exception as e:
    print(f"❌ PostgreSQL Error: {e}")

# Test Redis
try:
    r = redis.Redis(host='localhost', port=6379, password='redis123', decode_responses=True)
    r.ping()
    r.set('test_key', 'Hello from Python!')
    print(f"✅ Redis: {r.get('test_key')}")
except Exception as e:
    print(f"❌ Redis Error: {e}")

# Test MinIO
try:
    s3 = boto3.client(
        's3',
        endpoint_url='http://localhost:9000',
        aws_access_key_id='minioadmin',
        aws_secret_access_key='minioadmin123'
    )
    # สร้าง test bucket
    try:
        s3.create_bucket(Bucket='test-bucket')
    except Exception:
        pass
    s3.put_object(Bucket='test-bucket', Key='test.txt', Body=b'Hello MinIO!')
    obj = s3.get_object(Bucket='test-bucket', Key='test.txt')
    print(f"✅ MinIO: {obj['Body'].read().decode()}")
except Exception as e:
    print(f"❌ MinIO Error: {e}")
EOF
```

---

## 11. Workshop

### Workshop 1.1: วาด Architecture Diagram

ให้ผู้เรียนวาด architecture diagram สำหรับ system ที่ต้องการสร้าง โดยระบุ:

**คำถามที่ต้องตอบก่อนวาด:**
1. ระบบที่จะสร้างคืออะไร? (e-commerce, blog, fintech, IoT, etc.)
2. มี user กี่คน? (100, 10,000, 1,000,000?)
3. มี read หรือ write มากกว่ากัน? (read-heavy vs write-heavy)
4. ต้องการ real-time หรือไม่? (notifications, chat, live updates)
5. มีไฟล์ขนาดใหญ่ไหม? (images, videos, documents)
6. ต้องการ high availability ระดับไหน? (99.9%, 99.99%, 99.999%)

**Template ที่ให้เติม:**
```
[Your System Name]

Frontend Layer:
- Web App: _______________
- Mobile App: _______________

API Layer:
- Tech: _______________
- Authentication: _______________

Database Layer:
- PostgreSQL ใช้สำหรับ: _______________
- Redis ใช้สำหรับ: _______________
- MinIO/S3 ใช้สำหรับ: _______________

Scaling Strategy:
- Read Scaling: _______________
- Write Scaling: _______________
- File Storage: _______________

Expected Load:
- Users: _______________
- Requests/second: _______________
- Data size: _______________
```

### Workshop 1.2: ทดสอบ Setup Environment

```bash
# Checklist สำหรับ verify environment

echo "=== Checking Docker ==="
docker --version && echo "✅ Docker OK" || echo "❌ Docker NOT OK"
docker compose version && echo "✅ Docker Compose OK" || echo "❌ Docker Compose NOT OK"

echo ""
echo "=== Starting Stack ==="
cd /path/to/your/project
docker compose up -d

echo ""
echo "=== Waiting for services to be ready ==="
sleep 10

echo ""
echo "=== Checking Services ==="
docker compose ps

echo ""
echo "=== Testing Connections ==="
# PostgreSQL
docker exec pg-primary pg_isready -U admin -d mydb && \
    echo "✅ PostgreSQL Ready" || echo "❌ PostgreSQL NOT Ready"

# Redis
docker exec redis-single redis-cli -a redis123 ping | grep -q PONG && \
    echo "✅ Redis Ready" || echo "❌ Redis NOT Ready"

# MinIO
curl -s http://localhost:9000/minio/health/live | \
    { read response; [ "$response" = "OK" ] && echo "✅ MinIO Ready" || echo "✅ MinIO Ready (HTTP 200)"; }

echo ""
echo "=== Web UIs ==="
echo "pgAdmin:         http://localhost:8080"
echo "Redis Commander: http://localhost:8081"
echo "MinIO Console:   http://localhost:9001"
```

### Workshop 1.3: สร้าง Database แรก

```sql
-- เชื่อมต่อ PostgreSQL
-- docker exec -it pg-primary psql -U admin -d mydb

-- สร้าง Schema สำหรับ Workshop
CREATE SCHEMA IF NOT EXISTS workshop;

-- สร้าง Table แรก
CREATE TABLE workshop.users (
    user_id     BIGSERIAL PRIMARY KEY,
    username    VARCHAR(50) NOT NULL UNIQUE,
    email       VARCHAR(255) NOT NULL UNIQUE,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- เพิ่มข้อมูลทดสอบ
INSERT INTO workshop.users (username, email) VALUES
    ('alice', 'alice@example.com'),
    ('bob', 'bob@example.com'),
    ('charlie', 'charlie@example.com');

-- Query ทดสอบ
SELECT * FROM workshop.users;

-- ทดสอบ Redis
-- docker exec -it redis-single redis-cli -a redis123
-- SET mykey "Hello Database Cluster Course"
-- GET mykey
-- DEL mykey
```

---

## สรุปบทที่ 1

ในบทนี้เราได้เรียนรู้:

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Database Cluster | คืออะไร ทำไมต้องใช้ ประโยชน์หลัก |
| Architecture | Single vs Cluster, Primary-Replica, Full Cluster |
| Database Comparison | PostgreSQL, Redis, S3/MinIO แตกต่างอย่างไร |
| Use Cases | E-commerce, Social Media, FinTech, Healthcare |
| Scalability | Vertical vs Horizontal, Sharding, CQRS |
| CAP Theorem | C, A, P คืออะไร เลือก 2 จาก 3 |
| ACID vs BASE | เมื่อไหร่ใช้อะไร |
| Tech Stacks | Airbnb, Uber, Shopify, Netflix |
| Setup | Docker, docker-compose, verify connections |

**บทต่อไป:** ติดตั้ง PostgreSQL อย่างละเอียด และเริ่มใช้งานจริง

---

## แหล่งอ้างอิงเพิ่มเติม

- [PostgreSQL Official Documentation](https://www.postgresql.org/docs/)
- [Redis Documentation](https://redis.io/docs/)
- [MinIO Documentation](https://min.io/docs/)
- [CAP Theorem - Martin Fowler](https://martinfowler.com/articles/cap-theorem.html)
- [Designing Data-Intensive Applications - Martin Kleppmann](https://dataintensive.net/)
- [USE Method - Brendan Gregg](https://www.brendangregg.com/usemethod.html)
- [Uber Engineering Blog - Postgres to MySQL](https://www.uber.com/blog/postgres-to-mysql-migration/)
