# Part 91: Security - Row-Level Security (RLS) ใน PostgreSQL

## บทนำ

Row-Level Security (RLS) คือฟีเจอร์ของ PostgreSQL ที่ช่วยให้สามารถกำหนดสิทธิ์การเข้าถึงข้อมูลในระดับแถว (row) ได้ ต่างจากการควบคุมสิทธิ์ปกติที่ทำได้แค่ระดับตาราง (table-level) หรือคอลัมน์ (column-level)

ด้วย RLS เราสามารถบอกว่า "User A เห็นได้แค่แถวที่เป็นของ User A เท่านั้น" โดยที่ logic นี้อยู่ในฐานข้อมูลเอง ไม่ต้องพึ่ง application code

---

## 1. Row-Level Security คืออะไร และทำไมต้องใช้?

### 1.1 ปัญหาก่อนมี RLS

สมมติว่าเรามีระบบ Multi-tenant SaaS โดยมี table `orders` ที่เก็บออเดอร์ของลูกค้าทุกราย:

```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    customer_name TEXT NOT NULL,
    total DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

ถ้าไม่มี RLS application code ต้องทำเอง:

```javascript
// ต้องเพิ่ม WHERE ทุกครั้ง - ถ้าลืมจะ leak ข้อมูล tenant อื่น!
const orders = await db.query(
  'SELECT * FROM orders WHERE tenant_id = $1',
  [currentTenantId]
);
```

**ปัญหา:**
- ถ้าลืมใส่ `WHERE tenant_id = ?` จะ leak ข้อมูล tenant อื่น
- ต้องทำซ้ำทุก query ทุก service
- Bug ใน application code → data breach ทันที
- ยากต่อการ audit และ compliance

### 1.2 RLS แก้ปัญหาอย่างไร

ด้วย RLS เราใส่ policy ที่ database level:

```sql
-- Enable RLS
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- สร้าง policy: แต่ละ tenant เห็นได้แค่ของตัวเอง
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

ตอนนี้ไม่ว่า application จะ query ยังไงก็ตาม PostgreSQL จะ filter อัตโนมัติ:

```sql
-- Application query (ไม่มี WHERE)
SELECT * FROM orders;

-- PostgreSQL execute จริงๆ (RLS เพิ่ม WHERE ให้)
SELECT * FROM orders WHERE tenant_id = '550e8400-e29b-41d4-a716-446655440000';
```

### 1.3 ประโยชน์ของ RLS

1. **Security by default**: ไม่ต้องพึ่ง application code
2. **Defense in depth**: แม้ bug ใน application ก็ไม่ leak ข้าม tenant
3. **Centralized policy**: logic อยู่ที่เดียว
4. **Compliance**: ง่ายต่อการ audit
5. **Transparent**: application ไม่รู้สึกถึงความแตกต่าง

---

## 2. Enable RLS

### 2.1 คำสั่งพื้นฐาน

```sql
-- Enable RLS บน table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Disable RLS
ALTER TABLE orders DISABLE ROW LEVEL SECURITY;

-- ดู status ของ RLS
SELECT relname, relrowsecurity, relforcerowsecurity
FROM pg_class
WHERE relname = 'orders';
```

### 2.2 FORCE ROW LEVEL SECURITY

โดย default table owner จะ bypass RLS ซึ่งอาจเป็นปัญหา:

```sql
-- Force RLS แม้แต่ table owner
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

-- ยกเลิก force
ALTER TABLE orders NO FORCE ROW LEVEL SECURITY;
```

**ทำไมต้องใช้ FORCE?**

```sql
-- ถ้าไม่ FORCE และ connect ด้วย user ที่ own table
-- จะเห็นข้อมูลทุก row แม้มี RLS policy
SELECT * FROM orders; -- เห็นทุกแถว!

-- ถ้า FORCE แล้ว
SELECT * FROM orders; -- ถูก filter ตาม policy
```

---

## 3. CREATE POLICY

### 3.1 Syntax พื้นฐาน

```sql
CREATE POLICY policy_name ON table_name
    [AS { PERMISSIVE | RESTRICTIVE }]
    [FOR { ALL | SELECT | INSERT | UPDATE | DELETE }]
    [TO { role_name | PUBLIC | CURRENT_USER | SESSION_USER } [, ...]]
    [USING ( using_expression )]
    [WITH CHECK ( check_expression )];
```

### 3.2 USING vs WITH CHECK

- **USING**: ใช้สำหรับ SELECT, UPDATE, DELETE - กำหนดว่าแถวไหนที่ user เห็นได้
- **WITH CHECK**: ใช้สำหรับ INSERT, UPDATE - กำหนดว่าข้อมูลที่ insert/update ต้องผ่านเงื่อนไขอะไร

```sql
-- SELECT policy: เห็นได้เฉพาะ active records ของ tenant ตัวเอง
CREATE POLICY select_own_data ON orders
    FOR SELECT
    USING (
        tenant_id = current_setting('app.current_tenant')::uuid
        AND is_deleted = false
    );

-- INSERT policy: insert ได้เฉพาะ record ของ tenant ตัวเอง
CREATE POLICY insert_own_data ON orders
    FOR INSERT
    WITH CHECK (tenant_id = current_setting('app.current_tenant')::uuid);

-- UPDATE policy: update ได้เฉพาะ record ของตัวเอง และแก้ได้เฉพาะบาง field
CREATE POLICY update_own_data ON orders
    FOR UPDATE
    USING (tenant_id = current_setting('app.current_tenant')::uuid)
    WITH CHECK (tenant_id = current_setting('app.current_tenant')::uuid);

-- DELETE policy: ลบได้เฉพาะของตัวเอง
CREATE POLICY delete_own_data ON orders
    FOR DELETE
    USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

### 3.3 ALL policy

```sql
-- Policy เดียวสำหรับทุก operation
CREATE POLICY tenant_policy ON orders
    FOR ALL
    USING (tenant_id = current_setting('app.current_tenant')::uuid)
    WITH CHECK (tenant_id = current_setting('app.current_tenant')::uuid);
```

### 3.4 Modify และ Drop Policy

```sql
-- แก้ไข policy
ALTER POLICY tenant_policy ON orders
    USING (
        tenant_id = current_setting('app.current_tenant')::uuid
        AND status != 'archived'
    );

-- ลบ policy
DROP POLICY tenant_policy ON orders;

-- ดู policies ทั้งหมด
SELECT * FROM pg_policies WHERE tablename = 'orders';
```

---

## 4. Policy Expressions: USING และ WITH CHECK

### 4.1 current_user

```sql
-- ใช้ database user ในการ filter
CREATE POLICY user_policy ON user_profiles
    USING (username = current_user);

-- ตรวจสอบ role
CREATE POLICY admin_policy ON sensitive_data
    USING (pg_has_role(current_user, 'admin_role', 'member'));
```

### 4.2 current_setting()

```sql
-- Set session variable
SET LOCAL app.current_user_id = '123';
SET LOCAL app.current_tenant = '550e8400-e29b-41d4-a716-446655440000';
SET LOCAL app.user_role = 'manager';

-- ใช้ใน policy
CREATE POLICY user_data_policy ON documents
    USING (
        owner_id = current_setting('app.current_user_id')::uuid
        OR (
            tenant_id = current_setting('app.current_tenant')::uuid
            AND current_setting('app.user_role') = 'manager'
        )
    );
```

### 4.3 Custom Auth Functions

```sql
-- สร้าง function สำหรับ auth
CREATE OR REPLACE FUNCTION auth.uid() RETURNS UUID AS $$
BEGIN
    RETURN current_setting('app.user_id', TRUE)::UUID;
EXCEPTION WHEN OTHERS THEN
    RETURN NULL;
END;
$$ LANGUAGE plpgsql STABLE SECURITY DEFINER;

CREATE OR REPLACE FUNCTION auth.role() RETURNS TEXT AS $$
BEGIN
    RETURN current_setting('app.user_role', TRUE);
EXCEPTION WHEN OTHERS THEN
    RETURN 'anonymous';
END;
$$ LANGUAGE plpgsql STABLE SECURITY DEFINER;

CREATE OR REPLACE FUNCTION auth.tenant_id() RETURNS UUID AS $$
BEGIN
    RETURN current_setting('app.tenant_id', TRUE)::UUID;
EXCEPTION WHEN OTHERS THEN
    RETURN NULL;
END;
$$ LANGUAGE plpgsql STABLE SECURITY DEFINER;

-- ใช้ใน policy
CREATE POLICY documents_policy ON documents
    USING (
        owner_id = auth.uid()
        OR (tenant_id = auth.tenant_id() AND auth.role() IN ('admin', 'manager'))
    );
```

### 4.4 Complex Policy Expressions

```sql
-- Policy ที่ซับซ้อน: ตรวจสอบ permissions table
CREATE OR REPLACE FUNCTION can_access_project(project_id UUID)
RETURNS BOOLEAN AS $$
BEGIN
    RETURN EXISTS (
        SELECT 1 FROM project_members
        WHERE project_id = $1
        AND user_id = auth.uid()
        AND is_active = true
    );
END;
$$ LANGUAGE plpgsql STABLE SECURITY DEFINER;

CREATE POLICY project_access ON project_files
    USING (can_access_project(project_id));
```

---

## 5. Multi-tenancy ด้วย RLS

### 5.1 Database Schema Design

```sql
-- Schema สำหรับ multi-tenant application
CREATE SCHEMA app;

-- Tenants table
CREATE TABLE app.tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    plan TEXT NOT NULL DEFAULT 'free',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Users table
CREATE TABLE app.users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES app.tenants(id),
    email TEXT NOT NULL,
    password_hash TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'member', -- admin, manager, member
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(email)
);

-- Projects table
CREATE TABLE app.projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES app.tenants(id),
    name TEXT NOT NULL,
    description TEXT,
    owner_id UUID NOT NULL REFERENCES app.users(id),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Tasks table
CREATE TABLE app.tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES app.tenants(id),
    project_id UUID NOT NULL REFERENCES app.projects(id),
    title TEXT NOT NULL,
    assigned_to UUID REFERENCES app.users(id),
    status TEXT NOT NULL DEFAULT 'todo',
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 5.2 Enable RLS ทุก Table

```sql
-- Enable RLS
ALTER TABLE app.tenants ENABLE ROW LEVEL SECURITY;
ALTER TABLE app.users ENABLE ROW LEVEL SECURITY;
ALTER TABLE app.projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE app.tasks ENABLE ROW LEVEL SECURITY;

-- Force RLS (important!)
ALTER TABLE app.tenants FORCE ROW LEVEL SECURITY;
ALTER TABLE app.users FORCE ROW LEVEL SECURITY;
ALTER TABLE app.projects FORCE ROW LEVEL SECURITY;
ALTER TABLE app.tasks FORCE ROW LEVEL SECURITY;
```

### 5.3 สร้าง Database Role

```sql
-- Application role (ใช้ใน connection pool)
CREATE ROLE app_user LOGIN PASSWORD 'strong_password_here';

-- Grant permissions
GRANT USAGE ON SCHEMA app TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_user;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA app TO app_user;
```

### 5.4 RLS Policies

```sql
-- Tenants: เห็นได้แค่ tenant ของตัวเอง
CREATE POLICY tenant_isolation ON app.tenants
    AS PERMISSIVE
    FOR ALL
    TO app_user
    USING (id = current_setting('app.current_tenant_id')::uuid);

-- Users: เห็นได้แค่ users ใน tenant เดียวกัน
CREATE POLICY users_tenant_isolation ON app.users
    AS PERMISSIVE
    FOR SELECT
    TO app_user
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Users: insert ได้เฉพาะ tenant ของตัวเอง (admin only)
CREATE POLICY users_insert ON app.users
    AS PERMISSIVE
    FOR INSERT
    TO app_user
    WITH CHECK (
        tenant_id = current_setting('app.current_tenant_id')::uuid
        AND current_setting('app.user_role') = 'admin'
    );

-- Users: update ได้เฉพาะตัวเอง หรือ admin update ใครก็ได้ใน tenant
CREATE POLICY users_update ON app.users
    AS PERMISSIVE
    FOR UPDATE
    TO app_user
    USING (
        tenant_id = current_setting('app.current_tenant_id')::uuid
        AND (
            id = current_setting('app.current_user_id')::uuid
            OR current_setting('app.user_role') = 'admin'
        )
    )
    WITH CHECK (
        tenant_id = current_setting('app.current_tenant_id')::uuid
    );

-- Projects: tenant isolation
CREATE POLICY projects_isolation ON app.projects
    AS PERMISSIVE
    FOR ALL
    TO app_user
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid)
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Tasks: tenant isolation + project membership
CREATE POLICY tasks_isolation ON app.tasks
    AS PERMISSIVE
    FOR ALL
    TO app_user
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid)
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

### 5.5 Set Session Variables

```sql
-- ทุกครั้งที่ application connect และ authenticate user
-- ต้อง set session variables ก่อน query

-- Method 1: SET LOCAL (ใช้ใน transaction)
BEGIN;
SET LOCAL app.current_tenant_id = '550e8400-e29b-41d4-a716-446655440000';
SET LOCAL app.current_user_id = 'a3b4c5d6-e7f8-9012-b3c4-d5e6f7a8b9c0';
SET LOCAL app.user_role = 'admin';
-- queries here...
COMMIT;

-- Method 2: SET (ใช้นอก transaction, ใช้ได้ตลอด session)
SET app.current_tenant_id = '550e8400-e29b-41d4-a716-446655440000';
SET app.current_user_id = 'a3b4c5d6-e7f8-9012-b3c4-d5e6f7a8b9c0';
SET app.user_role = 'member';
```

---

## 6. Application Integration

### 6.1 Node.js + pg (node-postgres)

```javascript
// db.js - Database connection with RLS support
const { Pool } = require('pg');

const pool = new Pool({
  host: process.env.DB_HOST,
  port: 5432,
  database: process.env.DB_NAME,
  user: 'app_user', // application role
  password: process.env.DB_PASSWORD,
  max: 20, // connection pool size
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Helper function: get client with RLS context set
async function getClientWithContext(tenantId, userId, userRole) {
  const client = await pool.connect();
  
  try {
    // Set RLS context within a transaction
    await client.query('BEGIN');
    await client.query('SET LOCAL app.current_tenant_id = $1', [tenantId]);
    await client.query('SET LOCAL app.current_user_id = $1', [userId]);
    await client.query('SET LOCAL app.user_role = $1', [userRole]);
    
    return client;
  } catch (err) {
    client.release();
    throw err;
  }
}

// Middleware: set RLS context for every request
async function rlsMiddleware(req, res, next) {
  if (!req.user) {
    return next();
  }
  
  // Store context in request object
  req.dbContext = {
    tenantId: req.user.tenantId,
    userId: req.user.id,
    userRole: req.user.role,
  };
  
  next();
}

// Query helper with automatic RLS context
async function queryWithRLS(context, sql, params = []) {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    await client.query('SET LOCAL app.current_tenant_id = $1', [context.tenantId]);
    await client.query('SET LOCAL app.current_user_id = $1', [context.userId]);
    await client.query('SET LOCAL app.user_role = $1', [context.userRole]);
    
    const result = await client.query(sql, params);
    await client.query('COMMIT');
    
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

module.exports = { pool, getClientWithContext, queryWithRLS, rlsMiddleware };
```

```javascript
// projects.controller.js
const { queryWithRLS } = require('./db');

// GET /api/projects
async function getProjects(req, res) {
  try {
    // RLS จะ filter อัตโนมัติตาม tenant_id
    const result = await queryWithRLS(
      req.dbContext,
      'SELECT * FROM app.projects ORDER BY created_at DESC'
      // ไม่ต้องใส่ WHERE tenant_id - RLS จัดการให้
    );
    
    res.json(result.rows);
  } catch (err) {
    console.error('Error fetching projects:', err);
    res.status(500).json({ error: 'Internal server error' });
  }
}

// POST /api/projects
async function createProject(req, res) {
  const { name, description } = req.body;
  
  try {
    // RLS WITH CHECK จะ verify ว่า tenant_id ถูกต้อง
    const result = await queryWithRLS(
      req.dbContext,
      `INSERT INTO app.projects (tenant_id, name, description, owner_id)
       VALUES ($1, $2, $3, $4)
       RETURNING *`,
      [req.dbContext.tenantId, name, description, req.dbContext.userId]
    );
    
    res.status(201).json(result.rows[0]);
  } catch (err) {
    if (err.code === '42501') { // insufficient privilege
      return res.status(403).json({ error: 'Access denied' });
    }
    console.error('Error creating project:', err);
    res.status(500).json({ error: 'Internal server error' });
  }
}

module.exports = { getProjects, createProject };
```

### 6.2 TypeScript + Prisma

```typescript
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Project {
  id          String   @id @default(uuid())
  tenantId    String   @map("tenant_id")
  name        String
  description String?
  ownerId     String   @map("owner_id")
  createdAt   DateTime @default(now()) @map("created_at")
  
  @@map("projects")
  @@schema("app")
}
```

```typescript
// lib/prisma-rls.ts
import { PrismaClient } from '@prisma/client';
import { Pool } from 'pg';

const pgPool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

interface RLSContext {
  tenantId: string;
  userId: string;
  userRole: string;
}

// สร้าง Prisma Client ที่มี RLS context
export async function createPrismaWithRLS(context: RLSContext) {
  // ใช้ $executeRaw สำหรับ SET LOCAL
  const prisma = new PrismaClient();
  
  // Middleware ที่ set context ก่อนทุก query
  prisma.$use(async (params, next) => {
    // ต้องใช้ transaction เพื่อ SET LOCAL ให้ work
    // แต่ Prisma ไม่รองรับ interactive transaction ที่ดีนัก
    // แนะนำใช้ $queryRaw แทน
    return next(params);
  });
  
  return prisma;
}

// Better approach: use raw pg client with RLS
export async function withRLSTransaction<T>(
  context: RLSContext,
  callback: (client: any) => Promise<T>
): Promise<T> {
  const client = await pgPool.connect();
  
  try {
    await client.query('BEGIN');
    await client.query('SET LOCAL app.current_tenant_id = $1', [context.tenantId]);
    await client.query('SET LOCAL app.current_user_id = $1', [context.userId]);
    await client.query('SET LOCAL app.user_role = $1', [context.userRole]);
    
    const result = await callback(client);
    
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}
```

```typescript
// services/project.service.ts
import { withRLSTransaction } from '../lib/prisma-rls';

interface CreateProjectInput {
  name: string;
  description?: string;
}

export class ProjectService {
  async getProjects(context: RLSContext) {
    return withRLSTransaction(context, async (client) => {
      const result = await client.query(
        'SELECT * FROM app.projects ORDER BY created_at DESC'
      );
      return result.rows;
    });
  }
  
  async createProject(context: RLSContext, input: CreateProjectInput) {
    return withRLSTransaction(context, async (client) => {
      const result = await client.query(
        `INSERT INTO app.projects (tenant_id, name, description, owner_id)
         VALUES ($1, $2, $3, $4)
         RETURNING *`,
        [context.tenantId, input.name, input.description, context.userId]
      );
      return result.rows[0];
    });
  }
}
```

---

## 7. RLS Performance

### 7.1 Index ที่จำเป็น

```sql
-- Index บน tenant_id ทุก table (สำคัญมาก!)
CREATE INDEX CONCURRENTLY idx_projects_tenant_id
    ON app.projects (tenant_id);

CREATE INDEX CONCURRENTLY idx_tasks_tenant_id
    ON app.tasks (tenant_id);

CREATE INDEX CONCURRENTLY idx_users_tenant_id
    ON app.users (tenant_id);

-- Composite index สำหรับ query ที่ใช้บ่อย
CREATE INDEX CONCURRENTLY idx_tasks_tenant_project
    ON app.tasks (tenant_id, project_id);

CREATE INDEX CONCURRENTLY idx_tasks_tenant_assigned
    ON app.tasks (tenant_id, assigned_to)
    WHERE status != 'done';
```

### 7.2 EXPLAIN ANALYZE กับ RLS

```sql
-- ดูว่า RLS filter ถูก push down ไหม
SET app.current_tenant_id = '550e8400-e29b-41d4-a716-446655440000';

EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM app.projects
WHERE name ILIKE '%api%';

-- ควรเห็น:
-- Index Scan using idx_projects_tenant_id on projects
--   Index Cond: (tenant_id = '550e8400...'::uuid)
--   Filter: (name ~~* '%api%')
```

### 7.3 Security Barrier Views

```sql
-- Security barrier view: ป้องกัน policy expression leakage
CREATE VIEW app.active_projects
    WITH (security_barrier = true)
AS
SELECT * FROM app.projects
WHERE status = 'active';

-- ถ้าไม่มี security_barrier
-- PostgreSQL อาจ push down WHERE จาก outer query
-- ทำให้ policy ไม่ทำงานถูกต้อง
```

### 7.4 Policy Performance Tips

```sql
-- ✅ ดี: ใช้ cast ที่ถูกต้อง
CREATE POLICY fast_policy ON orders
    USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- ✅ ดี: ใช้ STABLE function
CREATE OR REPLACE FUNCTION get_current_tenant() RETURNS UUID AS $$
    SELECT current_setting('app.tenant_id')::UUID;
$$ LANGUAGE SQL STABLE;

CREATE POLICY fast_policy2 ON orders
    USING (tenant_id = get_current_tenant());

-- ❌ ไม่ดี: subquery ใน policy (ช้า)
CREATE POLICY slow_policy ON orders
    USING (
        EXISTS (
            SELECT 1 FROM users
            WHERE users.id = current_setting('app.user_id')::uuid
            AND users.tenant_id = orders.tenant_id
        )
    );
```

### 7.5 Table Partitioning สำหรับ Large Tenants

```sql
-- Partition by tenant สำหรับ large scale
CREATE TABLE app.orders (
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY HASH (tenant_id);

-- สร้าง partitions
CREATE TABLE app.orders_p0 PARTITION OF app.orders
    FOR VALUES WITH (MODULUS 8, REMAINDER 0);
CREATE TABLE app.orders_p1 PARTITION OF app.orders
    FOR VALUES WITH (MODULUS 8, REMAINDER 1);
CREATE TABLE app.orders_p2 PARTITION OF app.orders
    FOR VALUES WITH (MODULUS 8, REMAINDER 2);
CREATE TABLE app.orders_p3 PARTITION OF app.orders
    FOR VALUES WITH (MODULUS 8, REMAINDER 3);
-- ... และต่อไป

-- Index บน แต่ละ partition
CREATE INDEX ON app.orders_p0 (tenant_id, created_at DESC);
CREATE INDEX ON app.orders_p1 (tenant_id, created_at DESC);
-- ...
```

---

## 8. Policy Types: PERMISSIVE vs RESTRICTIVE

### 8.1 PERMISSIVE (Default)

```sql
-- PERMISSIVE: OR logic
-- ถ้าผ่าน policy ใดก็ได้ → allowed

-- Policy 1: เจ้าของ document
CREATE POLICY owner_access ON documents
    AS PERMISSIVE
    FOR SELECT
    USING (owner_id = auth.uid());

-- Policy 2: ถ้าเป็น public document
CREATE POLICY public_access ON documents
    AS PERMISSIVE
    FOR SELECT
    USING (is_public = true);

-- Result: SELECT จะได้ rows ที่ (owner หรือ public)
-- owner_id = uid() OR is_public = true
```

### 8.2 RESTRICTIVE

```sql
-- RESTRICTIVE: AND logic
-- ต้องผ่านทุก restrictive policy

-- Tenant isolation (restrictive - ต้องผ่านเสมอ)
CREATE POLICY tenant_bound ON documents
    AS RESTRICTIVE
    FOR ALL
    USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- Owner or public (permissive - OR logic)
CREATE POLICY owner_access ON documents
    AS PERMISSIVE
    FOR SELECT
    USING (owner_id = auth.uid());

CREATE POLICY public_access ON documents
    AS PERMISSIVE
    FOR SELECT
    USING (is_public = true);

-- Final result:
-- tenant_id = current_tenant AND (owner_id = uid OR is_public = true)
```

### 8.3 เมื่อใช้ RESTRICTIVE

ใช้ RESTRICTIVE เมื่อต้องการ:
- Tenant isolation ที่ไม่มีข้อยกเว้น
- Security constraint ที่ต้องผ่านเสมอ (เช่น is_deleted = false)
- Mandatory access control

```sql
-- ตัวอย่าง: ต้องไม่ใช่ deleted เสมอ (restrictive)
CREATE POLICY no_deleted_access ON documents
    AS RESTRICTIVE
    FOR SELECT
    USING (is_deleted = false);

-- Tenant isolation (restrictive)
CREATE POLICY tenant_isolation ON documents
    AS RESTRICTIVE
    FOR ALL
    USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- Role-based access (permissive)
CREATE POLICY admin_sees_all ON documents
    AS PERMISSIVE
    FOR SELECT
    USING (current_setting('app.user_role') = 'admin');

CREATE POLICY member_sees_own ON documents
    AS PERMISSIVE
    FOR SELECT
    USING (owner_id = current_setting('app.user_id')::uuid);
```

---

## 9. Bypassing RLS

### 9.1 Superuser Bypass

```sql
-- Superuser (postgres) bypass RLS โดยอัตโนมัติ
-- ตรวจสอบ bypass privilege
SELECT rolname, rolbypassrls
FROM pg_roles
WHERE rolname = 'postgres';

-- Grant BYPASSRLS (ระวัง!)
GRANT BYPASSRLS TO some_role;

-- Revoke BYPASSRLS
REVOKE BYPASSRLS FROM some_role;
```

### 9.2 ปัญหาของการ Bypass RLS

```sql
-- ❌ อันตราย: application ใช้ superuser
-- ถ้า bug ใน application → เห็นข้อมูลทุก tenant!

-- ✅ ถูกต้อง: สร้าง dedicated application role
CREATE ROLE app_user LOGIN PASSWORD 'password';
-- app_user ไม่มี BYPASSRLS
-- ต้องผ่าน RLS policies เสมอ

-- ตรวจสอบ
SELECT rolbypassrls FROM pg_roles WHERE rolname = 'app_user';
-- ควรได้ false
```

### 9.3 Service Role สำหรับ Internal Operations

```sql
-- บางครั้งต้องการ service ที่ไม่ถูก RLS บล็อก
-- เช่น background jobs, admin operations

-- ✅ วิธีที่ถูกต้อง: สร้าง service role พิเศษ
CREATE ROLE service_role LOGIN PASSWORD 'service_password';

-- BYPASSRLS เฉพาะ service role
GRANT BYPASSRLS TO service_role;

-- หรือ อาจใช้ SECURITY DEFINER function
CREATE OR REPLACE FUNCTION admin.get_all_users()
RETURNS SETOF app.users AS $$
    SELECT * FROM app.users;
$$ LANGUAGE SQL SECURITY DEFINER;

-- SECURITY DEFINER run ด้วย privilege ของ function owner
-- ถ้า owner มี BYPASSRLS → bypass RLS
-- Application ไม่ต้องมี privilege นั้น
GRANT EXECUTE ON FUNCTION admin.get_all_users() TO app_user;
```

---

## 10. Supabase RLS Patterns

Supabase ใช้ RLS อย่างหนัก เป็น reference ที่ดีสำหรับ best practices:

### 10.1 Supabase Auth Pattern

```sql
-- Supabase ใช้ JWT token สำหรับ auth
-- uid() extract จาก JWT claims

-- Supabase built-in functions
-- auth.uid() → user ID จาก JWT
-- auth.jwt() → full JWT payload
-- auth.role() → role จาก JWT

-- ตัวอย่าง Supabase-style policies
CREATE TABLE public.profiles (
    id UUID PRIMARY KEY REFERENCES auth.users(id),
    username TEXT UNIQUE,
    bio TEXT
);

ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

-- เห็น profile ทุกคน (public)
CREATE POLICY "Profiles are viewable by everyone"
    ON public.profiles FOR SELECT
    USING (true);

-- แก้ได้เฉพาะ profile ตัวเอง
CREATE POLICY "Users can update own profile"
    ON public.profiles FOR UPDATE
    USING (auth.uid() = id)
    WITH CHECK (auth.uid() = id);
```

### 10.2 Organization-based RLS (Supabase style)

```sql
-- Organizations
CREATE TABLE public.organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Organization members
CREATE TABLE public.org_members (
    org_id UUID REFERENCES public.organizations(id),
    user_id UUID REFERENCES auth.users(id),
    role TEXT NOT NULL DEFAULT 'member',
    PRIMARY KEY (org_id, user_id)
);

-- Projects
CREATE TABLE public.projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID REFERENCES public.organizations(id),
    name TEXT NOT NULL
);

ALTER TABLE public.projects ENABLE ROW LEVEL SECURITY;

-- Projects: เห็นได้เฉพาะ org ที่เป็น member
CREATE POLICY "Members can view org projects"
    ON public.projects FOR SELECT
    USING (
        EXISTS (
            SELECT 1 FROM public.org_members
            WHERE org_members.org_id = projects.org_id
            AND org_members.user_id = auth.uid()
        )
    );

-- Projects: create ได้เฉพาะ admin
CREATE POLICY "Admins can create projects"
    ON public.projects FOR INSERT
    WITH CHECK (
        EXISTS (
            SELECT 1 FROM public.org_members
            WHERE org_members.org_id = org_id
            AND org_members.user_id = auth.uid()
            AND org_members.role = 'admin'
        )
    );
```

---

## 11. Testing RLS Policies

### 11.1 Test ด้วย SET ROLE

```sql
-- Test ด้วยการ switch user
SET ROLE app_user;
SET LOCAL app.current_tenant_id = '550e8400-e29b-41d4-a716-446655440000';
SET LOCAL app.current_user_id = 'a1b2c3d4-e5f6-7890-a1b2-c3d4e5f67890';
SET LOCAL app.user_role = 'member';

-- ควรเห็นเฉพาะข้อมูลของ tenant นี้
SELECT count(*) FROM app.projects;

-- ลอง access ข้อมูล tenant อื่น (ควร error หรือ 0 rows)
SELECT * FROM app.projects
WHERE tenant_id = 'aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee';
-- ควรได้ 0 rows

RESET ROLE;
```

### 11.2 Test Functions

```sql
-- สร้าง test function
CREATE OR REPLACE FUNCTION test.rls_policies()
RETURNS TABLE(test_name TEXT, passed BOOLEAN, detail TEXT) AS $$
DECLARE
    v_count INTEGER;
    v_tenant1 UUID := '550e8400-e29b-41d4-a716-446655440000';
    v_tenant2 UUID := 'aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee';
    v_user1 UUID := 'a1b2c3d4-e5f6-7890-a1b2-c3d4e5f67890';
BEGIN
    -- Test 1: ไม่เห็นข้อมูล tenant อื่น
    PERFORM set_config('app.current_tenant_id', v_tenant1::text, true);
    PERFORM set_config('app.current_user_id', v_user1::text, true);
    PERFORM set_config('app.user_role', 'member', true);
    
    SELECT count(*) INTO v_count
    FROM app.projects
    WHERE tenant_id = v_tenant2;
    
    RETURN QUERY SELECT
        'tenant_isolation_select'::TEXT,
        (v_count = 0),
        format('Expected 0 rows from other tenant, got %s', v_count);
    
    -- Test 2: เห็นข้อมูล tenant ตัวเอง
    SELECT count(*) INTO v_count
    FROM app.projects
    WHERE tenant_id = v_tenant1;
    
    RETURN QUERY SELECT
        'own_tenant_visible'::TEXT,
        (v_count > 0),
        format('Expected >0 rows from own tenant, got %s', v_count);
    
END;
$$ LANGUAGE plpgsql;

-- Run tests
SELECT * FROM test.rls_policies();
```

### 11.3 Jest Tests สำหรับ Node.js

```javascript
// __tests__/rls.test.js
const { Pool } = require('pg');

const appPool = new Pool({
  connectionString: process.env.TEST_DATABASE_URL,
  user: 'app_user',
});

const adminPool = new Pool({
  connectionString: process.env.TEST_DATABASE_URL,
  user: 'postgres',
});

async function queryAsUser(tenantId, userId, role, sql, params = []) {
  const client = await appPool.connect();
  try {
    await client.query('BEGIN');
    await client.query('SET LOCAL app.current_tenant_id = $1', [tenantId]);
    await client.query('SET LOCAL app.current_user_id = $1', [userId]);
    await client.query('SET LOCAL app.user_role = $1', [role]);
    const result = await client.query(sql, params);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

describe('RLS Policies', () => {
  const tenant1 = '550e8400-e29b-41d4-a716-446655440000';
  const tenant2 = 'aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee';
  const user1 = 'a1b2c3d4-e5f6-7890-a1b2-c3d4e5f67890';
  
  beforeAll(async () => {
    // Setup test data
    await adminPool.query(`
      INSERT INTO app.tenants (id, name) VALUES
        ('${tenant1}', 'Tenant One'),
        ('${tenant2}', 'Tenant Two')
      ON CONFLICT DO NOTHING
    `);
    
    await adminPool.query(`
      INSERT INTO app.projects (tenant_id, name, owner_id) VALUES
        ('${tenant1}', 'Project A', '${user1}'),
        ('${tenant2}', 'Project B', '${user1}')
      ON CONFLICT DO NOTHING
    `);
  });
  
  test('User cannot see other tenant projects', async () => {
    const result = await queryAsUser(
      tenant1, user1, 'member',
      `SELECT * FROM app.projects WHERE tenant_id = '${tenant2}'`
    );
    expect(result.rows.length).toBe(0);
  });
  
  test('User can see own tenant projects', async () => {
    const result = await queryAsUser(
      tenant1, user1, 'member',
      'SELECT * FROM app.projects'
    );
    expect(result.rows.every(r => r.tenant_id === tenant1)).toBe(true);
  });
  
  test('User cannot insert project for other tenant', async () => {
    await expect(
      queryAsUser(
        tenant1, user1, 'member',
        `INSERT INTO app.projects (tenant_id, name, owner_id)
         VALUES ('${tenant2}', 'Hacked Project', '${user1}')`
      )
    ).rejects.toThrow();
  });
  
  afterAll(async () => {
    await appPool.end();
    await adminPool.end();
  });
});
```

---

## 12. Auditing: Who Accessed What

### 12.1 Audit Log Table

```sql
-- Audit log
CREATE TABLE audit.access_log (
    id BIGSERIAL PRIMARY KEY,
    table_name TEXT NOT NULL,
    operation TEXT NOT NULL, -- SELECT, INSERT, UPDATE, DELETE
    tenant_id UUID,
    user_id UUID,
    row_id TEXT,
    old_data JSONB,
    new_data JSONB,
    accessed_at TIMESTAMPTZ DEFAULT NOW(),
    client_ip INET
);

CREATE INDEX idx_audit_tenant_accessed
    ON audit.access_log (tenant_id, accessed_at DESC);

CREATE INDEX idx_audit_user_accessed
    ON audit.access_log (user_id, accessed_at DESC);
```

### 12.2 Audit Trigger

```sql
-- Generic audit trigger function
CREATE OR REPLACE FUNCTION audit.log_access()
RETURNS TRIGGER AS $$
DECLARE
    v_tenant_id UUID;
    v_user_id UUID;
BEGIN
    -- Get context
    BEGIN
        v_tenant_id := current_setting('app.current_tenant_id')::UUID;
        v_user_id := current_setting('app.current_user_id')::UUID;
    EXCEPTION WHEN OTHERS THEN
        v_tenant_id := NULL;
        v_user_id := NULL;
    END;
    
    IF TG_OP = 'DELETE' THEN
        INSERT INTO audit.access_log (
            table_name, operation, tenant_id, user_id,
            row_id, old_data, client_ip
        ) VALUES (
            TG_TABLE_NAME, TG_OP, v_tenant_id, v_user_id,
            OLD.id::TEXT, to_jsonb(OLD), inet_client_addr()
        );
        RETURN OLD;
    ELSIF TG_OP = 'INSERT' THEN
        INSERT INTO audit.access_log (
            table_name, operation, tenant_id, user_id,
            row_id, new_data, client_ip
        ) VALUES (
            TG_TABLE_NAME, TG_OP, v_tenant_id, v_user_id,
            NEW.id::TEXT, to_jsonb(NEW), inet_client_addr()
        );
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit.access_log (
            table_name, operation, tenant_id, user_id,
            row_id, old_data, new_data, client_ip
        ) VALUES (
            TG_TABLE_NAME, TG_OP, v_tenant_id, v_user_id,
            NEW.id::TEXT, to_jsonb(OLD), to_jsonb(NEW), inet_client_addr()
        );
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Attach trigger to tables
CREATE TRIGGER audit_projects
    AFTER INSERT OR UPDATE OR DELETE ON app.projects
    FOR EACH ROW EXECUTE FUNCTION audit.log_access();

CREATE TRIGGER audit_tasks
    AFTER INSERT OR UPDATE OR DELETE ON app.tasks
    FOR EACH ROW EXECUTE FUNCTION audit.log_access();
```

### 12.3 pg_audit Extension

```sql
-- Install pg_audit (ต้องติดตั้ง extension)
CREATE EXTENSION pgaudit;

-- Configure in postgresql.conf
-- pgaudit.log = 'read,write,ddl'
-- pgaudit.log_catalog = off
-- pgaudit.log_parameter = on
-- pgaudit.log_statement_once = on

-- Session audit
SET pgaudit.log = 'read,write';

-- Object audit for specific table
SELECT pgaudit.audit_table('app.projects');
```

---

## 13. Full Working Example: Multi-tenant SaaS

### 13.1 Complete Schema

```sql
-- Full multi-tenant schema
BEGIN;

CREATE SCHEMA IF NOT EXISTS app;
CREATE SCHEMA IF NOT EXISTS audit;
CREATE SCHEMA IF NOT EXISTS auth;

-- Tenants
CREATE TABLE app.tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    plan TEXT NOT NULL DEFAULT 'free' CHECK (plan IN ('free', 'pro', 'enterprise')),
    is_active BOOLEAN DEFAULT true,
    max_users INTEGER DEFAULT 5,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Users
CREATE TABLE app.users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES app.tenants(id) ON DELETE CASCADE,
    email TEXT NOT NULL,
    password_hash TEXT NOT NULL,
    full_name TEXT,
    role TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('owner', 'admin', 'manager', 'member')),
    is_active BOOLEAN DEFAULT true,
    last_login TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(email),
    UNIQUE(tenant_id, email)
);

-- Projects
CREATE TABLE app.projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES app.tenants(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT,
    owner_id UUID NOT NULL REFERENCES app.users(id),
    status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'archived', 'deleted')),
    is_public BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Tasks
CREATE TABLE app.tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES app.tenants(id) ON DELETE CASCADE,
    project_id UUID NOT NULL REFERENCES app.projects(id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    description TEXT,
    assigned_to UUID REFERENCES app.users(id),
    created_by UUID NOT NULL REFERENCES app.users(id),
    status TEXT NOT NULL DEFAULT 'todo' CHECK (status IN ('todo', 'in_progress', 'review', 'done')),
    priority INTEGER DEFAULT 3 CHECK (priority BETWEEN 1 AND 5),
    due_date DATE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_users_tenant ON app.users(tenant_id);
CREATE INDEX idx_users_email ON app.users(email);
CREATE INDEX idx_projects_tenant ON app.projects(tenant_id);
CREATE INDEX idx_projects_owner ON app.projects(owner_id);
CREATE INDEX idx_tasks_tenant ON app.tasks(tenant_id);
CREATE INDEX idx_tasks_project ON app.tasks(project_id);
CREATE INDEX idx_tasks_assigned ON app.tasks(assigned_to) WHERE assigned_to IS NOT NULL;
CREATE INDEX idx_tasks_status ON app.tasks(tenant_id, status);

COMMIT;
```

### 13.2 Enable RLS and Policies

```sql
-- Enable RLS
ALTER TABLE app.tenants ENABLE ROW LEVEL SECURITY;
ALTER TABLE app.users ENABLE ROW LEVEL SECURITY;
ALTER TABLE app.projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE app.tasks ENABLE ROW LEVEL SECURITY;

-- Force RLS
ALTER TABLE app.tenants FORCE ROW LEVEL SECURITY;
ALTER TABLE app.users FORCE ROW LEVEL SECURITY;
ALTER TABLE app.projects FORCE ROW LEVEL SECURITY;
ALTER TABLE app.tasks FORCE ROW LEVEL SECURITY;

-- Create application role
CREATE ROLE app_user LOGIN PASSWORD 'change_this_in_production';
GRANT CONNECT ON DATABASE your_db TO app_user;
GRANT USAGE ON SCHEMA app TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_user;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA app TO app_user;

-- ===== Tenant Policies =====

-- Tenant: เห็นได้แค่ tenant ของตัวเอง
CREATE POLICY "tenants_select" ON app.tenants
    FOR SELECT TO app_user
    USING (id = current_setting('app.tenant_id', true)::uuid);

-- ===== User Policies =====

-- Users: เห็นได้แค่ users ใน tenant เดียวกัน
CREATE POLICY "users_select" ON app.users
    FOR SELECT TO app_user
    USING (
        tenant_id = current_setting('app.tenant_id', true)::uuid
        AND is_active = true
    );

-- Users: insert โดย admin/owner เท่านั้น
CREATE POLICY "users_insert" ON app.users
    FOR INSERT TO app_user
    WITH CHECK (
        tenant_id = current_setting('app.tenant_id', true)::uuid
        AND current_setting('app.user_role', true) IN ('owner', 'admin')
    );

-- Users: update ตัวเองได้, admin update ใครก็ได้ใน tenant
CREATE POLICY "users_update" ON app.users
    FOR UPDATE TO app_user
    USING (
        tenant_id = current_setting('app.tenant_id', true)::uuid
        AND (
            id = current_setting('app.user_id', true)::uuid
            OR current_setting('app.user_role', true) IN ('owner', 'admin')
        )
    )
    WITH CHECK (
        tenant_id = current_setting('app.tenant_id', true)::uuid
    );

-- ===== Project Policies =====

-- Projects: เห็นได้แค่ project ใน tenant เดียวกัน
CREATE POLICY "projects_select" ON app.projects
    FOR SELECT TO app_user
    USING (
        tenant_id = current_setting('app.tenant_id', true)::uuid
        AND status != 'deleted'
    );

-- Projects: create โดย member ทุกคนได้
CREATE POLICY "projects_insert" ON app.projects
    FOR INSERT TO app_user
    WITH CHECK (
        tenant_id = current_setting('app.tenant_id', true)::uuid
    );

-- Projects: update โดย owner หรือ admin ของ project/tenant
CREATE POLICY "projects_update" ON app.projects
    FOR UPDATE TO app_user
    USING (
        tenant_id = current_setting('app.tenant_id', true)::uuid
        AND (
            owner_id = current_setting('app.user_id', true)::uuid
            OR current_setting('app.user_role', true) IN ('owner', 'admin')
        )
    )
    WITH CHECK (
        tenant_id = current_setting('app.tenant_id', true)::uuid
    );

-- Projects: delete (soft delete) โดย owner/admin
CREATE POLICY "projects_delete" ON app.projects
    FOR DELETE TO app_user
    USING (
        tenant_id = current_setting('app.tenant_id', true)::uuid
        AND current_setting('app.user_role', true) IN ('owner', 'admin')
    );

-- ===== Task Policies =====

-- Tasks: เห็นได้แค่ tasks ใน tenant และ project ที่เข้าถึงได้
CREATE POLICY "tasks_select" ON app.tasks
    FOR SELECT TO app_user
    USING (tenant_id = current_setting('app.tenant_id', true)::uuid);

-- Tasks: สร้างได้ทุกคนใน tenant
CREATE POLICY "tasks_insert" ON app.tasks
    FOR INSERT TO app_user
    WITH CHECK (
        tenant_id = current_setting('app.tenant_id', true)::uuid
    );

-- Tasks: update โดย assigned user หรือ admin
CREATE POLICY "tasks_update" ON app.tasks
    FOR UPDATE TO app_user
    USING (
        tenant_id = current_setting('app.tenant_id', true)::uuid
        AND (
            assigned_to = current_setting('app.user_id', true)::uuid
            OR created_by = current_setting('app.user_id', true)::uuid
            OR current_setting('app.user_role', true) IN ('owner', 'admin', 'manager')
        )
    )
    WITH CHECK (
        tenant_id = current_setting('app.tenant_id', true)::uuid
    );
```

### 13.3 Node.js Complete Implementation

```javascript
// src/auth/auth.service.js
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const { pool } = require('../db');

class AuthService {
  async login(email, password) {
    // Query users (RLS bypass for login - use admin connection)
    const adminPool = require('../db').adminPool;
    
    const result = await adminPool.query(
      'SELECT * FROM app.users WHERE email = $1 AND is_active = true',
      [email]
    );
    
    if (result.rows.length === 0) {
      throw new Error('Invalid credentials');
    }
    
    const user = result.rows[0];
    const validPassword = await bcrypt.compare(password, user.password_hash);
    
    if (!validPassword) {
      throw new Error('Invalid credentials');
    }
    
    // Update last login
    await adminPool.query(
      'UPDATE app.users SET last_login = NOW() WHERE id = $1',
      [user.id]
    );
    
    // Create JWT
    const token = jwt.sign(
      {
        userId: user.id,
        tenantId: user.tenant_id,
        email: user.email,
        role: user.role,
      },
      process.env.JWT_SECRET,
      { expiresIn: '1h' }
    );
    
    return { token, user: { id: user.id, email: user.email, role: user.role } };
  }
  
  verifyToken(token) {
    return jwt.verify(token, process.env.JWT_SECRET);
  }
}

module.exports = new AuthService();
```

```javascript
// src/middleware/rls.middleware.js
const authService = require('../auth/auth.service');
const { pool } = require('../db');

async function rlsAuthMiddleware(req, res, next) {
  const authHeader = req.headers.authorization;
  
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  const token = authHeader.substring(7);
  
  try {
    const payload = authService.verifyToken(token);
    
    req.rlsContext = {
      tenantId: payload.tenantId,
      userId: payload.userId,
      userRole: payload.role,
    };
    
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}

// Helper: execute query with RLS context
async function withRLS(context, callback) {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Set RLS context
    await client.query(
      "SELECT set_config('app.tenant_id', $1, true)",
      [context.tenantId]
    );
    await client.query(
      "SELECT set_config('app.user_id', $1, true)",
      [context.userId]
    );
    await client.query(
      "SELECT set_config('app.user_role', $1, true)",
      [context.userRole]
    );
    
    const result = await callback(client);
    await client.query('COMMIT');
    return result;
    
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

module.exports = { rlsAuthMiddleware, withRLS };
```

```javascript
// src/routes/projects.routes.js
const express = require('express');
const router = express.Router();
const { rlsAuthMiddleware, withRLS } = require('../middleware/rls.middleware');

// Apply auth middleware to all routes
router.use(rlsAuthMiddleware);

// GET /api/projects
router.get('/', async (req, res) => {
  try {
    const projects = await withRLS(req.rlsContext, async (client) => {
      const result = await client.query(`
        SELECT 
          p.*,
          u.full_name as owner_name,
          COUNT(t.id) as task_count
        FROM app.projects p
        LEFT JOIN app.users u ON u.id = p.owner_id
        LEFT JOIN app.tasks t ON t.project_id = p.id
        GROUP BY p.id, u.full_name
        ORDER BY p.created_at DESC
      `);
      return result.rows;
    });
    
    res.json({ projects });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Failed to fetch projects' });
  }
});

// POST /api/projects
router.post('/', async (req, res) => {
  const { name, description, isPublic } = req.body;
  
  if (!name) {
    return res.status(400).json({ error: 'Name is required' });
  }
  
  try {
    const project = await withRLS(req.rlsContext, async (client) => {
      const result = await client.query(`
        INSERT INTO app.projects (tenant_id, name, description, owner_id, is_public)
        VALUES ($1, $2, $3, $4, $5)
        RETURNING *
      `, [
        req.rlsContext.tenantId,
        name,
        description || null,
        req.rlsContext.userId,
        isPublic || false,
      ]);
      return result.rows[0];
    });
    
    res.status(201).json({ project });
  } catch (err) {
    if (err.code === '42501') { // RLS violation
      return res.status(403).json({ error: 'Access denied' });
    }
    console.error(err);
    res.status(500).json({ error: 'Failed to create project' });
  }
});

module.exports = router;
```

---

## 14. สรุปและ Best Practices

### 14.1 Checklist

- [ ] Enable RLS บน sensitive tables ทุก table
- [ ] FORCE ROW LEVEL SECURITY สำหรับ tables ที่ owner ต้องถูก filter ด้วย
- [ ] สร้าง dedicated application role (ไม่ใช้ superuser)
- [ ] Revoke BYPASSRLS จาก application role
- [ ] Index บน tenant_id ทุก table
- [ ] ใช้ RESTRICTIVE policies สำหรับ mandatory constraints
- [ ] Test policies ด้วย SET ROLE
- [ ] Audit log สำหรับ compliance

### 14.2 Performance Checklist

- [ ] Index บน columns ที่ใช้ใน USING expression
- [ ] ใช้ STABLE functions ใน policy expressions
- [ ] หลีกเลี่ยง subqueries ที่ซับซ้อนใน policies
- [ ] Monitor query plans ด้วย EXPLAIN ANALYZE
- [ ] พิจารณา table partitioning สำหรับ large tenants

### 14.3 Security Checklist

- [ ] ไม่เก็บ tenant context ใน application memory (ใช้ session variable)
- [ ] Validate RLS context ก่อน set (ป้องกัน injection)
- [ ] ทดสอบว่าไม่สามารถ access ข้าม tenant ได้
- [ ] Monitor policy violations ใน audit log
- [ ] Review policies เมื่อ schema เปลี่ยน

```sql
-- Final verification query
SELECT
    schemaname,
    tablename,
    policyname,
    permissive,
    cmd,
    qual AS using_clause,
    with_check
FROM pg_policies
WHERE schemaname = 'app'
ORDER BY tablename, policyname;
```

---

*เนื้อหานี้เป็นส่วนหนึ่งของ Database Cluster Course - World-Class Level*
*Part 91/100: Row-Level Security (RLS) ใน PostgreSQL*
