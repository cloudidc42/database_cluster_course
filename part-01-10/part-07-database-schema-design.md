# Part 07: การออกแบบ Database Schema เบื้องต้น

## สารบัญ

1. [Database Design Process](#1-database-design-process)
2. [Normalization](#2-normalization)
3. [Denormalization](#3-denormalization)
4. [Entity-Relationship Diagram (ERD)](#4-entity-relationship-diagram-erd)
5. [Cardinality และ Relationships](#5-cardinality-และ-relationships)
6. [Junction / Bridge Tables](#6-junction--bridge-tables)
7. [Soft Delete vs Hard Delete](#7-soft-delete-vs-hard-delete)
8. [Audit Trail Pattern](#8-audit-trail-pattern)
9. [Versioning / History Tables](#9-versioning--history-tables)
10. [Tree / Hierarchical Data](#10-tree--hierarchical-data)
11. [Self-referencing Tables](#11-self-referencing-tables)
12. [Polymorphic Associations](#12-polymorphic-associations)
13. [Multi-tenancy Patterns](#13-multi-tenancy-patterns)
14. [Schema Naming Conventions](#14-schema-naming-conventions)
15. [Workshop: Social Media Platform Schema](#15-workshop-social-media-platform-schema)

---

## 1. Database Design Process

การออกแบบ Database Schema ที่ดีเริ่มต้นจากกระบวนการที่มีขั้นตอนชัดเจน

### 1.1 ขั้นตอนการออกแบบ

```
Requirements → Conceptual Design → Logical Design → Physical Design
     ↓                ↓                  ↓                ↓
สัมภาษณ์ user    สร้าง ERD           สร้าง Schema     เลือก Index,
วิเคราะห์        แบบ Conceptual      สมบูรณ์           Partition,
business rules                                         Storage
```

### 1.2 Requirements Gathering

```
คำถามที่ต้องถาม:
- Entity หลักๆ ในระบบมีอะไรบ้าง?
- แต่ละ entity มี attributes อะไร?
- relationships ระหว่าง entities เป็นยังไง?
- ข้อมูลไหนที่ต้อง query บ่อย?
- ขนาดข้อมูลจะเป็นเท่าไหร่? (rows ต่อวัน/เดือน)
- มี regulatory requirements ไหม? (เก็บข้อมูลนานแค่ไหน)
- Performance requirements? (SLA)
```

### 1.3 ตัวอย่าง: E-commerce System Requirements

```
Entities:
- Customers (ลูกค้า)
- Products (สินค้า)
- Orders (คำสั่งซื้อ)
- Categories (หมวดหมู่)
- Reviews (รีวิว)
- Addresses (ที่อยู่)

Relationships:
- Customer มี Orders หลายอัน (1:N)
- Order มี Products หลายชิ้น (M:N ผ่าน OrderItems)
- Product อยู่ใน Category (M:N)
- Customer มี Addresses หลายอัน (1:N)
- Customer เขียน Reviews (1:N)
- Product มี Reviews หลายอัน (1:N)
```

---

## 2. Normalization

Normalization คือกระบวนการจัดระเบียบ database เพื่อลด redundancy และปรับปรุง data integrity

### 2.1 First Normal Form (1NF)

**กฎ**: 
- ทุก column มีค่าเดียว (atomic values)
- ไม่มี repeating groups
- มี primary key

```sql
-- ❌ ไม่ผ่าน 1NF: phone numbers เก็บหลายค่าในคอลัมน์เดียว
CREATE TABLE customers_bad (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    phone_numbers TEXT  -- "0812345678, 0898765432, 0867654321"
);

-- ✅ ผ่าน 1NF: แยกออกมาเป็นตารางใหม่
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE customer_phones (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    phone_number VARCHAR(20) NOT NULL,
    phone_type VARCHAR(20) DEFAULT 'mobile'  -- mobile, home, work
);

-- ❌ ไม่ผ่าน 1NF: repeating groups
CREATE TABLE orders_bad (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    product1_id INTEGER,
    product1_qty INTEGER,
    product2_id INTEGER,
    product2_qty INTEGER,
    product3_id INTEGER,
    product3_qty INTEGER
);

-- ✅ ผ่าน 1NF
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL
);

CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(id),
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL
);
```

### 2.2 Second Normal Form (2NF)

**กฎ**: ต้องผ่าน 1NF + ทุก non-key attribute ขึ้นอยู่กับ **ทั้งหมดของ** primary key (ไม่ใช่แค่บางส่วน)

ปัญหานี้เกิดขึ้นเมื่อมี **composite primary key** และ attribute บางตัวขึ้นอยู่กับ key แค่ส่วนเดียว

```sql
-- ❌ ไม่ผ่าน 2NF: student_name ขึ้นอยู่กับ student_id เท่านั้น
-- ไม่ใช่ทั้ง (student_id, course_id)
CREATE TABLE enrollments_bad (
    student_id INTEGER,
    course_id INTEGER,
    student_name VARCHAR(100),  -- ขึ้นอยู่กับ student_id เท่านั้น!
    course_name VARCHAR(100),   -- ขึ้นอยู่กับ course_id เท่านั้น!
    grade VARCHAR(5),           -- ขึ้นอยู่กับ (student_id, course_id) ✓
    PRIMARY KEY (student_id, course_id)
);

-- ✅ ผ่าน 2NF: แยกออกเป็น 3 ตาราง
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE courses (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    credits INTEGER NOT NULL DEFAULT 3
);

CREATE TABLE enrollments (
    student_id INTEGER REFERENCES students(id),
    course_id INTEGER REFERENCES courses(id),
    grade VARCHAR(5),
    enrolled_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (student_id, course_id)
);
```

### 2.3 Third Normal Form (3NF)

**กฎ**: ต้องผ่าน 2NF + ไม่มี **transitive dependencies** (attribute ขึ้นอยู่กับ non-key attribute อื่น)

```sql
-- ❌ ไม่ผ่าน 3NF: zip_code → city → state
-- city ขึ้นอยู่กับ zip_code ไม่ใช่ customer_id
CREATE TABLE customers_bad (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    zip_code VARCHAR(10),
    city VARCHAR(50),    -- ขึ้นอยู่กับ zip_code ไม่ใช่ id!
    state VARCHAR(50)    -- ขึ้นอยู่กับ zip_code ไม่ใช่ id!
);

-- ✅ ผ่าน 3NF: แยก zip code data ออกมา
CREATE TABLE zip_codes (
    zip_code VARCHAR(10) PRIMARY KEY,
    city VARCHAR(50) NOT NULL,
    state VARCHAR(50) NOT NULL,
    country VARCHAR(3) NOT NULL DEFAULT 'THA'
);

CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    zip_code VARCHAR(10) REFERENCES zip_codes(zip_code)
);
```

### 2.4 Boyce-Codd Normal Form (BCNF)

**กฎ**: เข้มกว่า 3NF - สำหรับทุก functional dependency X → Y นั้น X ต้องเป็น superkey

```sql
-- ❌ ไม่ผ่าน BCNF
-- teacher → subject (teacher สอนวิชาเดียว)
-- (student, subject) → teacher (นักเรียนแต่ละวิชามีครู 1 คน)
-- ปัญหา: teacher ไม่ใช่ superkey แต่กำหนด subject ได้
CREATE TABLE student_subjects_bad (
    student VARCHAR(50),
    subject VARCHAR(50),
    teacher VARCHAR(50),
    PRIMARY KEY (student, subject)
    -- teacher → subject dependency ทำให้ไม่ผ่าน BCNF
);

-- ✅ ผ่าน BCNF: แยกออก
CREATE TABLE teacher_subjects (
    teacher_id INTEGER PRIMARY KEY,
    teacher_name VARCHAR(100),
    subject_id INTEGER REFERENCES subjects(id)
);

CREATE TABLE student_enrollments (
    student_id INTEGER,
    teacher_id INTEGER REFERENCES teacher_subjects(teacher_id),
    PRIMARY KEY (student_id, teacher_id)
);
```

---

## 3. Denormalization

Denormalization คือการจงใจ "ทำให้ normalized น้อยลง" เพื่อเพิ่ม performance ด้วยการลด JOINs

### 3.1 เมื่อไหร่ควร Denormalize

```
ควร Denormalize เมื่อ:
1. Query ที่สำคัญต้อง JOIN หลาย tables และช้ามาก
2. ข้อมูลถูก read มากกว่า write อย่างมีนัยสำคัญ
3. ข้อมูลที่ denormalize เปลี่ยนแปลงน้อย
4. ต้องการ analytics/reporting แบบ near-realtime
5. Caching ไม่เพียงพอ

ไม่ควร Denormalize เมื่อ:
1. ข้อมูลเปลี่ยนบ่อย (update ยาก)
2. Data consistency สำคัญมาก
3. Storage เป็นข้อจำกัด
4. ยังไม่ได้ profile จริงว่าช้า
```

### 3.2 Denormalization Patterns

```sql
-- Pattern 1: Computed/Cached Columns
-- แทนที่จะ SUM ทุกครั้ง ให้เก็บ total ไว้เลย

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    -- Denormalized: เก็บ total แทนที่จะ SUM order_items ทุกครั้ง
    total_amount DECIMAL(15, 2) NOT NULL DEFAULT 0.00,
    item_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(id),
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL
);

-- Trigger update total เมื่อมีการเปลี่ยนแปลง
CREATE OR REPLACE FUNCTION update_order_totals()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE orders SET
        total_amount = (
            SELECT COALESCE(SUM(quantity * unit_price), 0)
            FROM order_items
            WHERE order_id = COALESCE(NEW.order_id, OLD.order_id)
        ),
        item_count = (
            SELECT COUNT(*)
            FROM order_items
            WHERE order_id = COALESCE(NEW.order_id, OLD.order_id)
        )
    WHERE id = COALESCE(NEW.order_id, OLD.order_id);
    
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER order_items_totals_trigger
AFTER INSERT OR UPDATE OR DELETE ON order_items
FOR EACH ROW EXECUTE FUNCTION update_order_totals();
```

```sql
-- Pattern 2: Duplicate Columns (Copy data จาก related table)
-- แทนที่จะ JOIN products ทุกครั้ง ให้ copy ชื่อและราคา ณ เวลาสั่งซื้อ

CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(id),
    product_id INTEGER NOT NULL REFERENCES products(id),
    -- Denormalized: copy ข้อมูลตอน insert
    product_name VARCHAR(200) NOT NULL,  -- copy จาก products
    product_sku VARCHAR(50),             -- copy จาก products
    unit_price DECIMAL(10, 2) NOT NULL,  -- ราคา ณ เวลาสั่ง (ไม่เปลี่ยนตาม products)
    quantity INTEGER NOT NULL,
    subtotal DECIMAL(10, 2) GENERATED ALWAYS AS (unit_price * quantity) STORED
);
```

```sql
-- Pattern 3: Summary Tables (สำหรับ Analytics)
-- Pre-aggregate ข้อมูลสำหรับ reporting

CREATE TABLE daily_sales_summary (
    date DATE NOT NULL,
    product_id INTEGER NOT NULL,
    total_quantity INTEGER NOT NULL DEFAULT 0,
    total_revenue DECIMAL(15, 2) NOT NULL DEFAULT 0.00,
    order_count INTEGER NOT NULL DEFAULT 0,
    avg_order_value DECIMAL(15, 2),
    PRIMARY KEY (date, product_id)
);

-- Refresh ด้วย cron job หรือ materialized view
CREATE MATERIALIZED VIEW daily_sales_mv AS
SELECT 
    DATE(o.created_at) AS date,
    oi.product_id,
    SUM(oi.quantity) AS total_quantity,
    SUM(oi.quantity * oi.unit_price) AS total_revenue,
    COUNT(DISTINCT o.id) AS order_count
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.status = 'completed'
GROUP BY DATE(o.created_at), oi.product_id;

-- Refresh materialized view
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales_mv;
```

---

## 4. Entity-Relationship Diagram (ERD)

### 4.1 ERD Symbols (Crow's Foot Notation)

```
Entities:
┌──────────┐
│  TABLE   │  Rectangle = Entity/Table
└──────────┘

Relationships:
──────────  Line = relationship
──────|     One (mandatory)
──────○     One (optional)
──────<     Many (mandatory)  
──────┤<    One-to-many
──────>○    Optional many

Crow's foot notation:
|──────|    Exactly one
○──────|    Zero or one  
|──────<    One or more
○──────<    Zero or more
```

### 4.2 ตัวอย่าง ERD ใน SQL

```sql
-- ERD สำหรับ Library System
-- Books ──< BookCopies
-- Authors >──< Books (through book_authors)
-- Members ──< Loans
-- BookCopies ──< Loans

CREATE TABLE authors (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    biography TEXT,
    birth_date DATE
);

CREATE TABLE books (
    id SERIAL PRIMARY KEY,
    title VARCHAR(300) NOT NULL,
    isbn VARCHAR(20) UNIQUE NOT NULL,
    published_year INTEGER,
    genre VARCHAR(50),
    description TEXT
);

-- Junction table สำหรับ Authors ><< Books (M:N)
CREATE TABLE book_authors (
    book_id INTEGER REFERENCES books(id) ON DELETE CASCADE,
    author_id INTEGER REFERENCES authors(id) ON DELETE CASCADE,
    author_order INTEGER DEFAULT 1,  -- ลำดับผู้แต่ง
    PRIMARY KEY (book_id, author_id)
);

CREATE TABLE book_copies (
    id SERIAL PRIMARY KEY,
    book_id INTEGER NOT NULL REFERENCES books(id),
    barcode VARCHAR(50) UNIQUE NOT NULL,
    condition VARCHAR(20) DEFAULT 'good',
    available BOOLEAN DEFAULT TRUE
);

CREATE TABLE members (
    id SERIAL PRIMARY KEY,
    member_number VARCHAR(20) UNIQUE NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(200) UNIQUE NOT NULL,
    join_date DATE DEFAULT CURRENT_DATE,
    expiry_date DATE
);

CREATE TABLE loans (
    id SERIAL PRIMARY KEY,
    copy_id INTEGER NOT NULL REFERENCES book_copies(id),
    member_id INTEGER NOT NULL REFERENCES members(id),
    loan_date DATE NOT NULL DEFAULT CURRENT_DATE,
    due_date DATE NOT NULL,
    returned_date DATE,
    CONSTRAINT loan_dates_valid CHECK (due_date > loan_date)
);
```

---

## 5. Cardinality และ Relationships

### 5.1 One-to-One (1:1)

```sql
-- ตัวอย่าง: User ──── UserProfile (1:1)
-- ทุก User มี Profile อย่างมากหนึ่งอัน

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(200) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE user_profiles (
    id SERIAL PRIMARY KEY,
    user_id INTEGER UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    -- UNIQUE constraint รับประกัน 1:1
    display_name VARCHAR(100),
    bio TEXT,
    avatar_url VARCHAR(500),
    date_of_birth DATE,
    phone VARCHAR(20),
    website VARCHAR(300),
    location VARCHAR(200),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- หรือใช้ user_id เป็น PK ด้วยก็ได้
CREATE TABLE user_settings (
    user_id INTEGER PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    -- user_id ทำหน้าที่ทั้ง PK และ FK ในคราวเดียว
    theme VARCHAR(20) DEFAULT 'light',
    language VARCHAR(5) DEFAULT 'en',
    notifications_enabled BOOLEAN DEFAULT TRUE,
    email_digest_frequency VARCHAR(20) DEFAULT 'weekly'
);
```

### 5.2 One-to-Many (1:N)

```sql
-- ตัวอย่าง: Customer ──< Orders (1:N)
-- หนึ่ง Customer มี Orders ได้หลายอัน

CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    email VARCHAR(200) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    -- FK ด้านฝั่ง "many" อ้างถึง "one"
    status VARCHAR(20) DEFAULT 'pending',
    total DECIMAL(10, 2),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Query: ดู orders ของ customer
SELECT o.* FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.email = 'customer@example.com';

-- Query: ดู customers ที่มี orders เยอะที่สุด
SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
ORDER BY order_count DESC
LIMIT 10;
```

### 5.3 Many-to-Many (M:N)

```sql
-- ตัวอย่าง: Products >──< Categories (M:N)
-- Product อยู่ได้หลาย Categories
-- Category มีได้หลาย Products

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(10, 2) NOT NULL
);

CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    parent_id INTEGER REFERENCES categories(id)  -- hierarchical categories
);

-- Junction table (ไม่มี extra data)
CREATE TABLE product_categories (
    product_id INTEGER REFERENCES products(id) ON DELETE CASCADE,
    category_id INTEGER REFERENCES categories(id) ON DELETE CASCADE,
    PRIMARY KEY (product_id, category_id),
    is_primary BOOLEAN DEFAULT FALSE  -- หมวดหมู่หลัก
);

-- ตัวอย่าง M:N ที่ junction table มีข้อมูลเพิ่มเติม
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE courses (
    id SERIAL PRIMARY KEY,
    code VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    max_enrollment INTEGER DEFAULT 30
);

-- Junction table with extra data
CREATE TABLE enrollments (
    student_id INTEGER REFERENCES students(id) ON DELETE CASCADE,
    course_id INTEGER REFERENCES courses(id) ON DELETE CASCADE,
    enrolled_at TIMESTAMP DEFAULT NOW(),
    grade DECIMAL(3, 1),              -- ข้อมูลเพิ่มเติมที่เกี่ยวกับ relationship
    attendance_percent DECIMAL(5, 2),
    status VARCHAR(20) DEFAULT 'enrolled',
    PRIMARY KEY (student_id, course_id)
);
```

---

## 6. Junction / Bridge Tables

### 6.1 Junction Table พื้นฐาน

```sql
-- Roles และ Permissions (M:N)
CREATE TABLE roles (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT
);

CREATE TABLE permissions (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    resource VARCHAR(50) NOT NULL,
    action VARCHAR(50) NOT NULL,
    UNIQUE(resource, action)
);

-- Junction
CREATE TABLE role_permissions (
    role_id INTEGER REFERENCES roles(id) ON DELETE CASCADE,
    permission_id INTEGER REFERENCES permissions(id) ON DELETE CASCADE,
    granted_at TIMESTAMP DEFAULT NOW(),
    granted_by INTEGER REFERENCES users(id),
    PRIMARY KEY (role_id, permission_id)
);

-- Users และ Roles (M:N)
CREATE TABLE user_roles (
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    role_id INTEGER REFERENCES roles(id) ON DELETE CASCADE,
    assigned_at TIMESTAMP DEFAULT NOW(),
    assigned_by INTEGER REFERENCES users(id),
    expires_at TIMESTAMP,  -- temporal role assignment
    PRIMARY KEY (user_id, role_id)
);

-- Query: ดู permissions ทั้งหมดของ user
SELECT DISTINCT p.name, p.resource, p.action
FROM users u
JOIN user_roles ur ON u.id = ur.user_id
JOIN roles r ON ur.role_id = r.id
JOIN role_permissions rp ON r.id = rp.role_id
JOIN permissions p ON rp.permission_id = p.id
WHERE u.id = 1
    AND (ur.expires_at IS NULL OR ur.expires_at > NOW());
```

### 6.2 Self-referencing Junction Table

```sql
-- Friendships (M:N self-referencing)
CREATE TABLE friendships (
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    friend_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    status VARCHAR(20) DEFAULT 'pending',  -- pending, accepted, blocked
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, friend_id),
    -- เก็บแค่ (A→B) ไม่ต้องเก็บทั้ง (A→B) และ (B→A)
    CONSTRAINT no_self_friendship CHECK (user_id != friend_id),
    -- รับประกันว่าเก็บ pair ที่ user_id < friend_id เสมอ
    CONSTRAINT ordered_friendship CHECK (user_id < friend_id)
);

-- Query: ดู friends ของ user
SELECT 
    CASE WHEN f.user_id = :user_id THEN f.friend_id ELSE f.user_id END AS friend_id,
    u.name AS friend_name,
    f.status,
    f.created_at
FROM friendships f
JOIN users u ON u.id = CASE WHEN f.user_id = :user_id THEN f.friend_id ELSE f.user_id END
WHERE (f.user_id = :user_id OR f.friend_id = :user_id)
    AND f.status = 'accepted';
```

---

## 7. Soft Delete vs Hard Delete

### 7.1 Hard Delete (ลบถาวร)

```sql
-- ลบข้อมูลจริง
DELETE FROM users WHERE id = 42;

-- ข้อดี:
-- - ง่าย ไม่ต้องการ extra logic
-- - ข้อมูลไม่สะสม
-- - ไม่ต้อง filter ใน queries

-- ข้อเสีย:
-- - กู้คืนไม่ได้
-- - อาจ violate foreign keys
-- - ไม่มี audit trail
```

### 7.2 Soft Delete (ลบเสมือน)

```sql
-- ใช้ deleted_at timestamp แทนการลบจริง
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(200) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    deleted_at TIMESTAMP DEFAULT NULL  -- NULL = active, has value = deleted
);

-- Soft delete
UPDATE users SET deleted_at = NOW() WHERE id = 42;

-- Query: ต้อง filter เสมอ
SELECT * FROM users WHERE deleted_at IS NULL;

-- กู้คืน
UPDATE users SET deleted_at = NULL WHERE id = 42;

-- ปัญหา: unique constraint กับ soft delete
-- ถ้า email ต้อง unique แต่ user ถูก soft delete แล้วสมัครใหม่จะ error!

-- แก้ไขด้วย partial unique index
ALTER TABLE users DROP CONSTRAINT users_email_key;
CREATE UNIQUE INDEX users_email_active_idx 
    ON users(email) 
    WHERE deleted_at IS NULL;  -- unique เฉพาะ active users
```

### 7.3 Row-Level Security กับ Soft Delete

```sql
-- สร้าง Policy ให้ query ไม่เห็น deleted records อัตโนมัติ
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY users_not_deleted ON users
    FOR ALL
    USING (deleted_at IS NULL);

-- หรือสร้าง View
CREATE VIEW active_users AS
    SELECT * FROM users WHERE deleted_at IS NULL;

-- Application ใช้ view แทน table โดยตรง
SELECT * FROM active_users WHERE email = 'user@example.com';
```

### 7.4 Cascade Soft Delete

```sql
-- Soft delete cascade ด้วย trigger
CREATE OR REPLACE FUNCTION cascade_soft_delete()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.deleted_at IS NOT NULL AND OLD.deleted_at IS NULL THEN
        -- User ถูก soft delete, soft delete orders ด้วย
        UPDATE orders 
        SET deleted_at = NEW.deleted_at
        WHERE customer_id = NEW.id AND deleted_at IS NULL;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_soft_delete_cascade
AFTER UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION cascade_soft_delete();
```

---

## 8. Audit Trail Pattern

Audit Trail บันทึกว่าใคร ทำอะไร เมื่อไหร่ กับข้อมูลไหน

### 8.1 Simple Audit Columns

```sql
-- เพิ่ม audit columns ทุกตาราง
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    
    -- Audit columns
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    created_by INTEGER REFERENCES users(id),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    updated_by INTEGER REFERENCES users(id),
    deleted_at TIMESTAMP WITH TIME ZONE DEFAULT NULL,
    deleted_by INTEGER REFERENCES users(id)
);

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_updated_at
BEFORE UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

### 8.2 Generic Audit Log Table

```sql
-- ตาราง audit log รวมสำหรับทุกตาราง
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    table_name VARCHAR(100) NOT NULL,
    record_id BIGINT NOT NULL,
    operation VARCHAR(10) NOT NULL,  -- INSERT, UPDATE, DELETE
    old_data JSONB,                  -- ข้อมูลก่อน
    new_data JSONB,                  -- ข้อมูลหลัง
    changed_fields TEXT[],           -- fields ที่เปลี่ยน
    performed_by INTEGER REFERENCES users(id),
    performed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    ip_address INET,
    user_agent TEXT,
    session_id UUID
);

-- Index สำหรับ query
CREATE INDEX audit_logs_table_record_idx ON audit_logs(table_name, record_id);
CREATE INDEX audit_logs_performed_at_idx ON audit_logs(performed_at);
CREATE INDEX audit_logs_performed_by_idx ON audit_logs(performed_by);

-- Generic audit trigger function
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS TRIGGER AS $$
DECLARE
    audit_row audit_logs%ROWTYPE;
    changed_fields TEXT[] := '{}';
    col_name TEXT;
    old_value TEXT;
    new_value TEXT;
BEGIN
    audit_row.table_name = TG_TABLE_NAME;
    audit_row.operation = TG_OP;
    audit_row.performed_at = NOW();
    audit_row.performed_by = current_setting('app.current_user_id', true)::INTEGER;
    audit_row.ip_address = current_setting('app.current_ip', true)::INET;
    
    IF TG_OP = 'DELETE' THEN
        audit_row.record_id = OLD.id;
        audit_row.old_data = to_jsonb(OLD);
    ELSIF TG_OP = 'INSERT' THEN
        audit_row.record_id = NEW.id;
        audit_row.new_data = to_jsonb(NEW);
    ELSIF TG_OP = 'UPDATE' THEN
        audit_row.record_id = NEW.id;
        audit_row.old_data = to_jsonb(OLD);
        audit_row.new_data = to_jsonb(NEW);
        
        -- หา fields ที่เปลี่ยน
        FOR col_name IN 
            SELECT key FROM jsonb_each_text(to_jsonb(NEW))
        LOOP
            old_value = (to_jsonb(OLD) ->> col_name);
            new_value = (to_jsonb(NEW) ->> col_name);
            IF old_value IS DISTINCT FROM new_value THEN
                changed_fields = array_append(changed_fields, col_name);
            END IF;
        END LOOP;
        audit_row.changed_fields = changed_fields;
    END IF;
    
    INSERT INTO audit_logs VALUES (audit_row.*);
    
    RETURN NULL;  -- result ignored for AFTER triggers
END;
$$ LANGUAGE plpgsql;

-- Apply trigger ให้ tables ต่างๆ
CREATE TRIGGER products_audit
AFTER INSERT OR UPDATE OR DELETE ON products
FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();

CREATE TRIGGER users_audit
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();
```

### 8.3 ตั้งค่า User Context ใน Application

```sql
-- Set context ก่อน queries (ใน application code)
SET app.current_user_id = '42';
SET app.current_ip = '192.168.1.1';

-- หรือใน transaction
BEGIN;
SET LOCAL app.current_user_id = '42';
UPDATE products SET price = 999.00 WHERE id = 1;
COMMIT;
```

```javascript
// Node.js: ตั้ง audit context ก่อน query
async function withAuditContext(client, userId, ip, fn) {
  await client.query('SET LOCAL app.current_user_id = $1', [userId]);
  await client.query('SET LOCAL app.current_ip = $1', [ip]);
  return fn(client);
}

// ใช้งาน
await withTransaction(pool, async (client) => {
  await withAuditContext(client, req.user.id, req.ip, async (client) => {
    await client.query(
      'UPDATE products SET price = $1 WHERE id = $2',
      [999.00, productId]
    );
  });
});
```

---

## 9. Versioning / History Tables

### 9.1 Temporal Tables Pattern

```sql
-- เก็บ history ของ records ทุกเวอร์ชัน
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    description TEXT,
    version INTEGER NOT NULL DEFAULT 1,
    current BOOLEAN NOT NULL DEFAULT TRUE,
    valid_from TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    valid_to TIMESTAMP WITH TIME ZONE DEFAULT NULL,  -- NULL = current version
    created_by INTEGER REFERENCES users(id)
);

-- Function สำหรับอัพเดท product พร้อม history
CREATE OR REPLACE FUNCTION update_product_with_history(
    p_id INTEGER,
    p_name VARCHAR,
    p_price DECIMAL,
    p_description TEXT,
    p_user_id INTEGER
) RETURNS INTEGER AS $$
DECLARE
    v_new_version INTEGER;
BEGIN
    -- Mark current version เป็น expired
    UPDATE products 
    SET current = FALSE, valid_to = NOW()
    WHERE id = p_id AND current = TRUE
    RETURNING version + 1 INTO v_new_version;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Product % not found', p_id;
    END IF;
    
    -- Insert version ใหม่
    INSERT INTO products (id, name, price, description, version, current, valid_from, created_by)
    VALUES (p_id, p_name, p_price, p_description, v_new_version, TRUE, NOW(), p_user_id);
    
    RETURN v_new_version;
END;
$$ LANGUAGE plpgsql;

-- ดู product ปัจจุบัน
SELECT * FROM products WHERE id = 1 AND current = TRUE;

-- ดู history ทั้งหมดของ product
SELECT * FROM products WHERE id = 1 ORDER BY version DESC;

-- ดูว่า product เป็นยังไง ณ วันที่ผ่านมา
SELECT * FROM products 
WHERE id = 1 
    AND valid_from <= '2024-01-15 12:00:00'
    AND (valid_to IS NULL OR valid_to > '2024-01-15 12:00:00');
```

### 9.2 Event Sourcing Pattern

```sql
-- แทนที่จะเก็บ state ปัจจุบัน ให้เก็บ events ทั้งหมด
CREATE TABLE account_events (
    id BIGSERIAL PRIMARY KEY,
    account_id INTEGER NOT NULL REFERENCES accounts(id),
    event_type VARCHAR(50) NOT NULL,  -- CREATED, DEPOSITED, WITHDRAWN, etc.
    event_data JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_by INTEGER REFERENCES users(id),
    event_sequence BIGINT NOT NULL  -- ลำดับของ event ใน account
);

CREATE UNIQUE INDEX account_events_sequence_idx 
    ON account_events(account_id, event_sequence);

-- Function คำนวณ balance จาก events
CREATE OR REPLACE FUNCTION get_account_balance(p_account_id INTEGER)
RETURNS DECIMAL AS $$
BEGIN
    RETURN (
        SELECT COALESCE(SUM(
            CASE event_type
                WHEN 'DEPOSITED' THEN (event_data->>'amount')::DECIMAL
                WHEN 'WITHDRAWN' THEN -(event_data->>'amount')::DECIMAL
                WHEN 'CREATED' THEN (event_data->>'initial_balance')::DECIMAL
                ELSE 0
            END
        ), 0)
        FROM account_events
        WHERE account_id = p_account_id
    );
END;
$$ LANGUAGE plpgsql;
```

---

## 10. Tree / Hierarchical Data

### 10.1 Adjacency List (ง่ายสุด แต่ query ยาก)

```sql
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id INTEGER REFERENCES categories(id)  -- NULL = root
);

INSERT INTO categories (name, parent_id) VALUES
    ('Electronics', NULL),          -- id=1 root
    ('Computers', 1),               -- id=2 ลูกของ Electronics
    ('Laptops', 2),                 -- id=3 ลูกของ Computers
    ('Gaming Laptops', 3),          -- id=4 ลูกของ Laptops
    ('Business Laptops', 3),        -- id=5 ลูกของ Laptops
    ('Smartphones', 1);             -- id=6 ลูกของ Electronics

-- ดู children ทันที
SELECT * FROM categories WHERE parent_id = 2;

-- ดู tree ทั้งหมดด้วย recursive CTE
WITH RECURSIVE category_tree AS (
    -- Base case: root nodes
    SELECT id, name, parent_id, 1 AS depth, 
           name::VARCHAR(1000) AS path
    FROM categories WHERE parent_id IS NULL
    
    UNION ALL
    
    -- Recursive: children
    SELECT c.id, c.name, c.parent_id, ct.depth + 1,
           (ct.path || ' > ' || c.name)::VARCHAR(1000)
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT id, name, depth, path
FROM category_tree
ORDER BY path;
```

### 10.2 Closure Table (query เร็ว ซับซ้อนกว่า)

```sql
-- Closure table เก็บ relationships ทุกคู่ (ancestor, descendant)
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE category_closure (
    ancestor_id INTEGER NOT NULL REFERENCES categories(id),
    descendant_id INTEGER NOT NULL REFERENCES categories(id),
    depth INTEGER NOT NULL DEFAULT 0,  -- ห่างกันกี่ระดับ
    PRIMARY KEY (ancestor_id, descendant_id)
);

-- Function insert category ใหม่
CREATE OR REPLACE FUNCTION insert_category(
    p_name VARCHAR,
    p_parent_id INTEGER DEFAULT NULL
) RETURNS INTEGER AS $$
DECLARE
    v_new_id INTEGER;
BEGIN
    INSERT INTO categories (name) VALUES (p_name) RETURNING id INTO v_new_id;
    
    -- Self-reference
    INSERT INTO category_closure (ancestor_id, descendant_id, depth)
    VALUES (v_new_id, v_new_id, 0);
    
    -- Copy parent's ancestors
    IF p_parent_id IS NOT NULL THEN
        INSERT INTO category_closure (ancestor_id, descendant_id, depth)
        SELECT ancestor_id, v_new_id, depth + 1
        FROM category_closure
        WHERE descendant_id = p_parent_id;
    END IF;
    
    RETURN v_new_id;
END;
$$ LANGUAGE plpgsql;

-- ดู descendants ทั้งหมด
SELECT c.* FROM categories c
JOIN category_closure cc ON c.id = cc.descendant_id
WHERE cc.ancestor_id = 1 AND cc.depth > 0;

-- ดู ancestors ทั้งหมด (path to root)
SELECT c.*, cc.depth FROM categories c
JOIN category_closure cc ON c.id = cc.ancestor_id
WHERE cc.descendant_id = 4
ORDER BY cc.depth DESC;

-- ดู children ทันที (depth=1)
SELECT c.* FROM categories c
JOIN category_closure cc ON c.id = cc.descendant_id
WHERE cc.ancestor_id = 2 AND cc.depth = 1;
```

### 10.3 LTREE Extension (PostgreSQL specific)

```sql
-- ติดตั้ง extension
CREATE EXTENSION IF NOT EXISTS ltree;

CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    path LTREE NOT NULL  -- เช่น "Electronics.Computers.Laptops"
);

CREATE INDEX categories_path_gist ON categories USING GIST(path);
CREATE INDEX categories_path_btree ON categories USING BTREE(path);

-- Insert ข้อมูล
INSERT INTO categories (name, path) VALUES
    ('Electronics', 'Electronics'),
    ('Computers', 'Electronics.Computers'),
    ('Laptops', 'Electronics.Computers.Laptops'),
    ('Gaming Laptops', 'Electronics.Computers.Laptops.Gaming'),
    ('Smartphones', 'Electronics.Smartphones');

-- ดู descendants ทั้งหมดของ Computers
SELECT * FROM categories WHERE path <@ 'Electronics.Computers';

-- ดู ancestors ของ Gaming Laptops
SELECT * FROM categories 
WHERE path @> 'Electronics.Computers.Laptops.Gaming';

-- ดู children ทันที (depth ถัดไป)
SELECT * FROM categories 
WHERE path ~ 'Electronics.Computers.*{1}';

-- ดู level ปัจจุบัน
SELECT name, nlevel(path) AS depth FROM categories;
```

### 10.4 Nested Sets

```sql
-- Nested Sets: ทุก node มี lft และ rgt values
-- descendants อยู่ระหว่าง lft และ rgt ของ ancestor

CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    lft INTEGER NOT NULL,
    rgt INTEGER NOT NULL,
    depth INTEGER NOT NULL DEFAULT 0
);

-- ข้อมูลตัวอย่าง:
-- Electronics: lft=1, rgt=12
--   Computers: lft=2, rgt=7
--     Laptops: lft=3, rgt=6
--       Gaming: lft=4, rgt=5
--   Smartphones: lft=8, rgt=9
--   Tablets: lft=10, rgt=11

INSERT INTO categories (name, lft, rgt, depth) VALUES
    ('Electronics', 1, 12, 0),
    ('Computers', 2, 7, 1),
    ('Laptops', 3, 6, 2),
    ('Gaming Laptops', 4, 5, 3),
    ('Smartphones', 8, 9, 1),
    ('Tablets', 10, 11, 1);

-- ดู descendants ของ Computers (lft=2, rgt=7)
SELECT * FROM categories
WHERE lft BETWEEN 2 AND 7
ORDER BY lft;

-- ดู ancestors ของ Gaming Laptops (lft=4)
SELECT * FROM categories
WHERE lft < 4 AND rgt > 4
ORDER BY lft;

-- นับจำนวน descendants
SELECT name, (rgt - lft - 1) / 2 AS descendants_count
FROM categories;
```

---

## 11. Self-referencing Tables

```sql
-- Employees Hierarchy (Manager → Employee)
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    position VARCHAR(100) NOT NULL,
    manager_id INTEGER REFERENCES employees(id),  -- self-reference
    department VARCHAR(50),
    salary DECIMAL(10, 2),
    hire_date DATE DEFAULT CURRENT_DATE
);

INSERT INTO employees (id, name, position, manager_id) VALUES
    (1, 'Alice CEO', 'CEO', NULL),           -- top-level
    (2, 'Bob CTO', 'CTO', 1),               -- reports to Alice
    (3, 'Charlie Dev', 'Developer', 2),      -- reports to Bob
    (4, 'Dave Dev', 'Developer', 2),         -- reports to Bob
    (5, 'Eve Designer', 'Designer', 1);      -- reports to Alice

-- ดู org chart (recursive)
WITH RECURSIVE org_chart AS (
    -- Root: CEO
    SELECT id, name, position, manager_id, 0 AS level,
           name::TEXT AS reporting_path
    FROM employees WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Direct reports
    SELECT e.id, e.name, e.position, e.manager_id, oc.level + 1,
           oc.reporting_path || ' → ' || e.name
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT 
    REPEAT('  ', level) || name AS org_chart,
    position,
    level,
    reporting_path
FROM org_chart
ORDER BY reporting_path;

-- ดูทีมทั้งหมดของ manager
WITH RECURSIVE team AS (
    SELECT id, name FROM employees WHERE id = 2  -- Bob's team
    UNION ALL
    SELECT e.id, e.name FROM employees e
    JOIN team t ON e.manager_id = t.id
)
SELECT * FROM team;
```

---

## 12. Polymorphic Associations

### 12.1 Single Table Inheritance

```sql
-- ทุก types อยู่ในตารางเดียว ใช้ type column แยกประเภท
CREATE TABLE notifications (
    id BIGSERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id),
    type VARCHAR(50) NOT NULL,  -- 'order_shipped', 'friend_request', 'comment_reply'
    -- Common fields
    title VARCHAR(200) NOT NULL,
    message TEXT,
    read_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    -- Type-specific fields (nullable)
    -- For order notifications
    order_id INTEGER REFERENCES orders(id),
    -- For friend request notifications
    requester_id INTEGER REFERENCES users(id),
    -- For comment notifications
    comment_id INTEGER REFERENCES comments(id),
    post_id INTEGER REFERENCES posts(id)
);

-- ข้อดี: ง่าย, query รวดเร็ว
-- ข้อเสีย: null columns เยอะ, ยาก enforce constraints
```

### 12.2 Polymorphic Foreign Keys

```sql
-- ใช้ (entity_type, entity_id) แทน FK ตรงๆ
CREATE TABLE comments (
    id BIGSERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id),
    -- Polymorphic reference
    commentable_type VARCHAR(50) NOT NULL,  -- 'Post', 'Photo', 'Product'
    commentable_id INTEGER NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX comments_commentable_idx ON comments(commentable_type, commentable_id);

-- Insert
INSERT INTO comments (user_id, commentable_type, commentable_id, content)
VALUES (1, 'Post', 42, 'Great post!');

INSERT INTO comments (user_id, commentable_type, commentable_id, content)
VALUES (1, 'Product', 15, 'Nice product!');

-- Query (ต้อง JOIN แยกตามประเภท)
SELECT c.*, p.title AS post_title
FROM comments c
LEFT JOIN posts p ON c.commentable_type = 'Post' AND c.commentable_id = p.id
WHERE c.commentable_type = 'Post';
```

### 12.3 Table-per-Type (แนะนำสำหรับ PostgreSQL)

```sql
-- Base table + specialized tables
CREATE TABLE media_items (
    id BIGSERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id),
    title VARCHAR(200),
    description TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    type VARCHAR(20) NOT NULL  -- 'photo', 'video', 'document'
);

CREATE TABLE photos (
    id BIGINT PRIMARY KEY REFERENCES media_items(id) ON DELETE CASCADE,
    width INTEGER,
    height INTEGER,
    format VARCHAR(10),  -- JPEG, PNG, WebP
    file_size_bytes BIGINT,
    storage_path VARCHAR(500) NOT NULL
);

CREATE TABLE videos (
    id BIGINT PRIMARY KEY REFERENCES media_items(id) ON DELETE CASCADE,
    duration_seconds INTEGER,
    resolution VARCHAR(20),  -- 1920x1080
    format VARCHAR(10),      -- MP4, MOV
    file_size_bytes BIGINT,
    storage_path VARCHAR(500) NOT NULL,
    thumbnail_path VARCHAR(500)
);

-- Query photos
SELECT m.*, p.width, p.height, p.format
FROM media_items m
JOIN photos p ON m.id = p.id
WHERE m.user_id = 1;
```

---

## 13. Multi-tenancy Patterns

### 13.1 Separate Database per Tenant

```
ข้อดี: isolation ดีที่สุด, ง่าย backup ต่อ tenant
ข้อเสีย: resource เยอะ, ยากจัดการเมื่อ tenant มาก
```

```javascript
// Node.js: dynamic database routing
const tenantConnections = new Map();

async function getTenantPool(tenantId) {
  if (!tenantConnections.has(tenantId)) {
    const pool = new Pool({
      host: process.env.DB_HOST,
      database: `tenant_${tenantId}`,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
    });
    tenantConnections.set(tenantId, pool);
  }
  return tenantConnections.get(tenantId);
}
```

### 13.2 Separate Schema per Tenant

```sql
-- สร้าง schema สำหรับแต่ละ tenant
CREATE SCHEMA tenant_acme;
CREATE SCHEMA tenant_globex;

-- สร้างตารางใน schema
CREATE TABLE tenant_acme.users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(200) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL
);

CREATE TABLE tenant_globex.users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(200) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL
);

-- ตั้ง search_path ตาม tenant
SET search_path TO tenant_acme, public;
SELECT * FROM users;  -- ใช้ tenant_acme.users

-- Function สร้าง tenant schema
CREATE OR REPLACE FUNCTION create_tenant_schema(tenant_slug TEXT)
RETURNS VOID AS $$
BEGIN
    EXECUTE format('CREATE SCHEMA IF NOT EXISTS tenant_%I', tenant_slug);
    
    -- สร้างตารางทั้งหมดใน schema นั้น
    EXECUTE format('
        CREATE TABLE tenant_%I.users (
            id SERIAL PRIMARY KEY,
            email VARCHAR(200) UNIQUE NOT NULL,
            name VARCHAR(200) NOT NULL,
            created_at TIMESTAMP DEFAULT NOW()
        )
    ', tenant_slug);
    
    -- ...สร้างตารางอื่นๆ
END;
$$ LANGUAGE plpgsql;
```

### 13.3 Shared Tables with tenant_id (Row-level filtering)

```sql
-- ทุกตารางมี tenant_id column
CREATE TABLE tenants (
    id SERIAL PRIMARY KEY,
    slug VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    plan VARCHAR(20) DEFAULT 'free',
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL REFERENCES tenants(id),
    email VARCHAR(200) NOT NULL,
    name VARCHAR(200) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE (tenant_id, email)  -- unique per tenant
);

CREATE TABLE projects (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL REFERENCES tenants(id),
    name VARCHAR(200) NOT NULL,
    created_by INTEGER NOT NULL REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Index สำคัญมาก!
CREATE INDEX users_tenant_id_idx ON users(tenant_id);
CREATE INDEX projects_tenant_id_idx ON projects(tenant_id);

-- Row Level Security (RLS)
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_users ON users
    FOR ALL
    USING (tenant_id = current_setting('app.current_tenant_id')::INTEGER);

CREATE POLICY tenant_isolation_projects ON projects
    FOR ALL
    USING (tenant_id = current_setting('app.current_tenant_id')::INTEGER);

-- Application set ค่า tenant ก่อน query
SET app.current_tenant_id = '42';
SELECT * FROM users;  -- เห็นเฉพาะ users ของ tenant 42
```

---

## 14. Schema Naming Conventions

### 14.1 Table Naming

```sql
-- ✅ แนะนำ: snake_case, plural nouns
CREATE TABLE users (...);
CREATE TABLE user_profiles (...);
CREATE TABLE order_items (...);
CREATE TABLE product_categories (...);

-- ❌ ไม่แนะนำ
CREATE TABLE User (...);           -- PascalCase
CREATE TABLE tblUser (...);        -- Hungarian notation prefix
CREATE TABLE user (...);           -- singular
```

### 14.2 Column Naming

```sql
-- ✅ แนะนำ: snake_case
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,              -- id สำหรับ PK (ไม่ใช่ order_id)
    customer_id INTEGER NOT NULL,       -- FK: <referenced_table>_id
    total_amount DECIMAL(10, 2),        -- descriptive
    is_paid BOOLEAN DEFAULT FALSE,      -- boolean: is_, has_, can_
    created_at TIMESTAMP DEFAULT NOW(), -- timestamp: _at
    updated_at TIMESTAMP DEFAULT NOW()
);

-- ✅ FK naming: <referenced_table>_id
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),    -- ไม่ใช่ author_id
    category_id INTEGER REFERENCES categories(id)
);

-- ถ้า FK หลายอันไปตารางเดียวกัน ใช้ context prefix
CREATE TABLE transfers (
    id SERIAL PRIMARY KEY,
    from_account_id INTEGER REFERENCES accounts(id),
    to_account_id INTEGER REFERENCES accounts(id)
);
```

### 14.3 Index Naming

```sql
-- Convention: <table>_<columns>_idx
CREATE INDEX users_email_idx ON users(email);
CREATE INDEX orders_customer_id_idx ON orders(customer_id);
CREATE INDEX products_name_description_idx ON products(name, description);

-- Unique index: เพิ่ม _uniq
CREATE UNIQUE INDEX users_email_uniq ON users(email);

-- Partial index: เพิ่ม description
CREATE INDEX orders_pending_created_at_idx ON orders(created_at)
    WHERE status = 'pending';
```

### 14.4 Constraint Naming

```sql
CREATE TABLE orders (
    id SERIAL,
    customer_id INTEGER NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    
    -- Primary key: <table>_pkey
    CONSTRAINT orders_pkey PRIMARY KEY (id),
    
    -- Foreign key: <table>_<column>_fkey
    CONSTRAINT orders_customer_id_fkey FOREIGN KEY (customer_id)
        REFERENCES customers(id),
    
    -- Check constraint: <table>_<description>_check
    CONSTRAINT orders_total_positive_check CHECK (total > 0),
    CONSTRAINT orders_valid_status_check CHECK (
        status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled')
    ),
    
    -- Unique: <table>_<columns>_unique
    CONSTRAINT orders_reference_number_unique UNIQUE (reference_number)
);
```

---

## 15. Workshop: Social Media Platform Schema

### 15.1 Requirements

```
Entities:
- Users (ผู้ใช้งาน)
- Posts (โพสต์)
- Comments (ความคิดเห็น)
- Likes (ถูกใจ)
- Follows (ติดตาม)
- Messages (ข้อความ)
- Notifications (การแจ้งเตือน)
- Media (รูปภาพ/วิดีโอ)
- Hashtags (แฮชแท็ก)
```

### 15.2 Schema Design

```sql
-- ========================================
-- Core Tables
-- ========================================

CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(200) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    display_name VARCHAR(100),
    bio TEXT,
    avatar_url VARCHAR(500),
    website_url VARCHAR(300),
    location VARCHAR(100),
    is_verified BOOLEAN DEFAULT FALSE,
    is_private BOOLEAN DEFAULT FALSE,
    follower_count INTEGER DEFAULT 0,  -- denormalized
    following_count INTEGER DEFAULT 0, -- denormalized
    post_count INTEGER DEFAULT 0,      -- denormalized
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_active_at TIMESTAMP WITH TIME ZONE,
    deleted_at TIMESTAMP WITH TIME ZONE DEFAULT NULL
);

CREATE UNIQUE INDEX users_username_active_idx 
    ON users(username) WHERE deleted_at IS NULL;

-- ========================================
-- Posts
-- ========================================

CREATE TABLE posts (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    content TEXT,
    post_type VARCHAR(20) NOT NULL DEFAULT 'text',  -- text, photo, video, story, reel
    visibility VARCHAR(20) NOT NULL DEFAULT 'public', -- public, followers, private
    like_count INTEGER DEFAULT 0,     -- denormalized
    comment_count INTEGER DEFAULT 0,  -- denormalized
    share_count INTEGER DEFAULT 0,    -- denormalized
    view_count INTEGER DEFAULT 0,
    is_pinned BOOLEAN DEFAULT FALSE,
    -- For reposts/shares
    original_post_id BIGINT REFERENCES posts(id),
    -- Location
    location_name VARCHAR(200),
    location_lat DECIMAL(10, 8),
    location_lng DECIMAL(11, 8),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE DEFAULT NULL
);

CREATE INDEX posts_user_id_idx ON posts(user_id);
CREATE INDEX posts_created_at_idx ON posts(created_at DESC);
CREATE INDEX posts_user_feed_idx ON posts(user_id, created_at DESC) 
    WHERE deleted_at IS NULL;

-- ========================================
-- Media
-- ========================================

CREATE TABLE media_items (
    id BIGSERIAL PRIMARY KEY,
    post_id BIGINT REFERENCES posts(id) ON DELETE CASCADE,
    user_id BIGINT NOT NULL REFERENCES users(id),
    media_type VARCHAR(20) NOT NULL,  -- photo, video, gif
    storage_key VARCHAR(500) NOT NULL,
    thumbnail_key VARCHAR(500),
    width INTEGER,
    height INTEGER,
    duration_seconds INTEGER,  -- for video
    file_size_bytes BIGINT,
    mime_type VARCHAR(100),
    display_order SMALLINT DEFAULT 0,
    alt_text VARCHAR(500),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX media_items_post_id_idx ON media_items(post_id);

-- ========================================
-- Comments
-- ========================================

CREATE TABLE comments (
    id BIGSERIAL PRIMARY KEY,
    post_id BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    user_id BIGINT NOT NULL REFERENCES users(id),
    parent_comment_id BIGINT REFERENCES comments(id),  -- for nested comments
    content TEXT NOT NULL,
    like_count INTEGER DEFAULT 0,
    reply_count INTEGER DEFAULT 0,
    depth SMALLINT DEFAULT 0,  -- nesting level (0 = top-level)
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE DEFAULT NULL
);

CREATE INDEX comments_post_id_idx ON comments(post_id, created_at DESC);
CREATE INDEX comments_parent_id_idx ON comments(parent_comment_id);
CREATE INDEX comments_user_id_idx ON comments(user_id);

-- ========================================
-- Likes (polymorphic: Post หรือ Comment)
-- ========================================

CREATE TABLE likes (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    likeable_type VARCHAR(20) NOT NULL,  -- 'post' or 'comment'
    likeable_id BIGINT NOT NULL,
    reaction_type VARCHAR(20) DEFAULT 'like',  -- like, love, haha, wow, sad, angry
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE (user_id, likeable_type, likeable_id)  -- ถูกใจได้ครั้งเดียว
);

CREATE INDEX likes_likeable_idx ON likes(likeable_type, likeable_id);
CREATE INDEX likes_user_id_idx ON likes(user_id);

-- Trigger: อัพเดท like_count อัตโนมัติ
CREATE OR REPLACE FUNCTION update_like_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        IF NEW.likeable_type = 'post' THEN
            UPDATE posts SET like_count = like_count + 1 WHERE id = NEW.likeable_id;
        ELSIF NEW.likeable_type = 'comment' THEN
            UPDATE comments SET like_count = like_count + 1 WHERE id = NEW.likeable_id;
        END IF;
    ELSIF TG_OP = 'DELETE' THEN
        IF OLD.likeable_type = 'post' THEN
            UPDATE posts SET like_count = GREATEST(like_count - 1, 0) WHERE id = OLD.likeable_id;
        ELSIF OLD.likeable_type = 'comment' THEN
            UPDATE comments SET like_count = GREATEST(like_count - 1, 0) WHERE id = OLD.likeable_id;
        END IF;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER likes_count_trigger
AFTER INSERT OR DELETE ON likes
FOR EACH ROW EXECUTE FUNCTION update_like_count();

-- ========================================
-- Follows
-- ========================================

CREATE TABLE follows (
    follower_id BIGINT NOT NULL REFERENCES users(id),
    following_id BIGINT NOT NULL REFERENCES users(id),
    status VARCHAR(20) DEFAULT 'active',  -- active, pending (for private accounts)
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    PRIMARY KEY (follower_id, following_id),
    CONSTRAINT no_self_follow CHECK (follower_id != following_id)
);

CREATE INDEX follows_following_id_idx ON follows(following_id);
CREATE INDEX follows_follower_id_idx ON follows(follower_id);

-- Trigger: อัพเดท follower/following counts
CREATE OR REPLACE FUNCTION update_follow_counts()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' AND NEW.status = 'active' THEN
        UPDATE users SET follower_count = follower_count + 1 WHERE id = NEW.following_id;
        UPDATE users SET following_count = following_count + 1 WHERE id = NEW.follower_id;
    ELSIF TG_OP = 'DELETE' AND OLD.status = 'active' THEN
        UPDATE users SET follower_count = GREATEST(follower_count - 1, 0) WHERE id = OLD.following_id;
        UPDATE users SET following_count = GREATEST(following_count - 1, 0) WHERE id = OLD.follower_id;
    ELSIF TG_OP = 'UPDATE' THEN
        IF OLD.status != 'active' AND NEW.status = 'active' THEN
            UPDATE users SET follower_count = follower_count + 1 WHERE id = NEW.following_id;
            UPDATE users SET following_count = following_count + 1 WHERE id = NEW.follower_id;
        ELSIF OLD.status = 'active' AND NEW.status != 'active' THEN
            UPDATE users SET follower_count = GREATEST(follower_count - 1, 0) WHERE id = OLD.following_id;
            UPDATE users SET following_count = GREATEST(following_count - 1, 0) WHERE id = OLD.follower_id;
        END IF;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER follows_count_trigger
AFTER INSERT OR UPDATE OR DELETE ON follows
FOR EACH ROW EXECUTE FUNCTION update_follow_counts();

-- ========================================
-- Messages (Direct Messages)
-- ========================================

CREATE TABLE conversations (
    id BIGSERIAL PRIMARY KEY,
    conversation_type VARCHAR(20) DEFAULT 'direct',  -- direct, group
    name VARCHAR(200),  -- สำหรับ group conversations
    created_by BIGINT REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_message_at TIMESTAMP WITH TIME ZONE
);

CREATE TABLE conversation_participants (
    conversation_id BIGINT REFERENCES conversations(id) ON DELETE CASCADE,
    user_id BIGINT REFERENCES users(id),
    joined_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_read_at TIMESTAMP WITH TIME ZONE,
    is_admin BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (conversation_id, user_id)
);

CREATE TABLE messages (
    id BIGSERIAL PRIMARY KEY,
    conversation_id BIGINT NOT NULL REFERENCES conversations(id),
    sender_id BIGINT NOT NULL REFERENCES users(id),
    content TEXT,
    message_type VARCHAR(20) DEFAULT 'text',  -- text, media, sticker, reaction
    media_url VARCHAR(500),
    reply_to_id BIGINT REFERENCES messages(id),
    is_read BOOLEAN DEFAULT FALSE,
    read_at TIMESTAMP WITH TIME ZONE,
    deleted_for_sender BOOLEAN DEFAULT FALSE,
    deleted_for_all BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX messages_conversation_id_idx ON messages(conversation_id, created_at DESC);
CREATE INDEX messages_sender_id_idx ON messages(sender_id);

-- ========================================
-- Hashtags
-- ========================================

CREATE TABLE hashtags (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,  -- lowercase, no #
    post_count INTEGER DEFAULT 0,
    trending_score DECIMAL(10, 4) DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE post_hashtags (
    post_id BIGINT REFERENCES posts(id) ON DELETE CASCADE,
    hashtag_id BIGINT REFERENCES hashtags(id),
    PRIMARY KEY (post_id, hashtag_id)
);

CREATE INDEX post_hashtags_hashtag_id_idx ON post_hashtags(hashtag_id);

-- ========================================
-- Notifications
-- ========================================

CREATE TABLE notifications (
    id BIGSERIAL PRIMARY KEY,
    recipient_id BIGINT NOT NULL REFERENCES users(id),
    sender_id BIGINT REFERENCES users(id),
    notification_type VARCHAR(50) NOT NULL,
    -- Types: like_post, like_comment, comment_post, reply_comment,
    --        follow, mention_post, mention_comment, message,
    --        follow_request, follow_accepted
    title VARCHAR(200),
    body TEXT,
    -- Polymorphic reference to the relevant entity
    entity_type VARCHAR(20),  -- post, comment, message, user
    entity_id BIGINT,
    -- Push notification data
    action_url VARCHAR(300),
    image_url VARCHAR(300),
    is_read BOOLEAN DEFAULT FALSE,
    read_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX notifications_recipient_unread_idx 
    ON notifications(recipient_id, created_at DESC)
    WHERE is_read = FALSE;

-- ========================================
-- Queries สำหรับ Social Media Features
-- ========================================

-- Home Feed: โพสต์จาก users ที่ follow (ล่าสุดก่อน)
SELECT 
    p.*,
    u.username,
    u.display_name,
    u.avatar_url,
    u.is_verified,
    CASE WHEN l.id IS NOT NULL THEN TRUE ELSE FALSE END AS is_liked_by_me
FROM posts p
JOIN users u ON p.user_id = u.id
-- ติดตาม users
JOIN follows f ON f.following_id = p.user_id
LEFT JOIN likes l ON l.likeable_type = 'post' 
    AND l.likeable_id = p.id 
    AND l.user_id = :current_user_id
WHERE f.follower_id = :current_user_id
    AND f.status = 'active'
    AND p.deleted_at IS NULL
    AND (p.visibility = 'public' OR p.visibility = 'followers')
ORDER BY p.created_at DESC
LIMIT 20 OFFSET :offset;

-- Explore: trending posts
SELECT 
    p.*,
    u.username,
    u.avatar_url
FROM posts p
JOIN users u ON p.user_id = u.id
WHERE p.deleted_at IS NULL
    AND p.visibility = 'public'
    AND p.created_at > NOW() - INTERVAL '48 hours'
ORDER BY (
    p.like_count * 1.0 + 
    p.comment_count * 2.0 + 
    p.share_count * 3.0 +
    p.view_count * 0.1
) DESC
LIMIT 50;

-- Hashtag search
SELECT p.*, u.username
FROM posts p
JOIN users u ON p.user_id = u.id
JOIN post_hashtags ph ON p.id = ph.post_id
JOIN hashtags h ON ph.hashtag_id = h.id
WHERE h.name = 'postgresql'
    AND p.deleted_at IS NULL
ORDER BY p.created_at DESC
LIMIT 20;

-- User suggestions (mutual followers)
WITH my_following AS (
    SELECT following_id FROM follows 
    WHERE follower_id = :current_user_id AND status = 'active'
),
their_following AS (
    SELECT f.follower_id AS user_id, COUNT(*) AS mutual_count
    FROM follows f
    JOIN my_following mf ON f.following_id = mf.following_id
    WHERE f.follower_id != :current_user_id
        AND f.follower_id NOT IN (SELECT following_id FROM my_following)
        AND f.status = 'active'
    GROUP BY f.follower_id
)
SELECT u.*, tf.mutual_count
FROM users u
JOIN their_following tf ON u.id = tf.user_id
WHERE u.deleted_at IS NULL
ORDER BY tf.mutual_count DESC, u.follower_count DESC
LIMIT 10;
```

### 15.3 Performance Optimizations

```sql
-- Partial indexes สำหรับ common queries
CREATE INDEX posts_feed_idx ON posts(user_id, created_at DESC)
    WHERE deleted_at IS NULL AND visibility IN ('public', 'followers');

CREATE INDEX notifications_unread_idx ON notifications(recipient_id, created_at DESC)
    WHERE is_read = FALSE;

CREATE INDEX follows_active_idx ON follows(follower_id, following_id)
    WHERE status = 'active';

-- Composite indexes
CREATE INDEX messages_conversation_time_idx ON messages(conversation_id, created_at DESC)
    WHERE deleted_for_all = FALSE;

-- Full text search สำหรับ search users/posts
ALTER TABLE users ADD COLUMN search_vector TSVECTOR;
CREATE INDEX users_search_idx ON users USING GIN(search_vector);

CREATE OR REPLACE FUNCTION update_user_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector = 
        setweight(to_tsvector('english', COALESCE(NEW.username, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.display_name, '')), 'B') ||
        setweight(to_tsvector('english', COALESCE(NEW.bio, '')), 'C');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_search_vector_trigger
BEFORE INSERT OR UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION update_user_search_vector();

-- Search users
SELECT id, username, display_name, avatar_url,
       ts_rank(search_vector, query) AS rank
FROM users, to_tsquery('english', 'john | doe') query
WHERE search_vector @@ query
    AND deleted_at IS NULL
ORDER BY rank DESC
LIMIT 20;
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Database Design Process**: จาก requirements → ERD → Schema
2. **Normalization**: 1NF, 2NF, 3NF, BCNF และตัวอย่างการแก้ไข
3. **Denormalization**: เมื่อไหร่ควรทำ พร้อม patterns ต่างๆ
4. **ERD**: symbols และ cardinality (1:1, 1:N, M:N)
5. **Junction Tables**: จัดการ M:N relationships
6. **Soft Delete**: pattern ที่ดีกว่า hard delete
7. **Audit Trail**: ติดตามการเปลี่ยนแปลงข้อมูล
8. **Versioning**: เก็บ history ของข้อมูล
9. **Hierarchical Data**: Adjacency List, Closure Table, LTREE, Nested Sets
10. **Multi-tenancy**: 3 patterns สำหรับ SaaS
11. **Workshop**: Social Media Platform schema ที่สมบูรณ์

บทถัดไปจะเรียนรู้เรื่อง Redis ซึ่งเป็น in-memory database ที่ใช้ร่วมกับ PostgreSQL เพื่อเพิ่ม performance

---

*[Part 07 จบ — ไปต่อ Part 08: Redis Basics]*
