# Part 61: Connection Pool Optimization

## บทนำ: ทำไม Connection Pool ถึงสำคัญ

ในระบบ PostgreSQL ที่มีผู้ใช้งานจำนวนมาก การจัดการ database connections อย่างมีประสิทธิภาพเป็นปัจจัยสำคัญที่สุดอย่างหนึ่ง เพราะ PostgreSQL ใช้ process-based model (fork a new process per connection) ต่างจาก thread-based databases

### PostgreSQL Connection Overhead

การเปิด connection ใหม่แต่ละครั้งมีค่าใช้จ่ายที่สูงมาก:

```
Memory per connection:
- Stack memory: ~8MB (default stack size)
- Shared memory structures: ~2-3MB
- Process overhead: ~1-2MB
- Working memory (work_mem): ไม่นับรวม แต่ใช้เมื่อ query ทำงาน
Total: ~5-10MB per connection
```

ตัวอย่างการคำนวณ:
```
Server RAM: 32GB
System OS: ~2GB
PostgreSQL shared_buffers: 8GB
Available for connections: 22GB
Max connections: 22GB / 8MB = ~2,750 connections (theoretical)
แต่ในทางปฏิบัติ ควรจำกัดไว้ที่ 200-400 connections
```

### ปัญหาของ Too Many Connections

```sql
-- ดู connection limits ปัจจุบัน
SELECT 
    current_setting('max_connections') as max_connections,
    COUNT(*) as current_connections,
    COUNT(*) * 100.0 / current_setting('max_connections')::int as usage_percent
FROM pg_stat_activity;

-- ดู connections แยกตาม state
SELECT 
    state,
    COUNT(*) as count,
    MAX(EXTRACT(EPOCH FROM (NOW() - state_change))) as max_seconds_in_state
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY state
ORDER BY count DESC;
```

เมื่อ connections เต็ม จะได้ error:
```
FATAL: remaining connection slots are reserved for non-replication superuser connections
```

---

## การคำนวณ Optimal Pool Size

### สูตรพื้นฐาน

```
Pool Size = (CPU Cores × 2) + Effective Disk Spindles

ตัวอย่าง:
- Server: 8 CPU cores, SSD (ถือว่า effective spindles = 1)
- Pool Size = (8 × 2) + 1 = 17 connections
```

แต่ในความเป็นจริงต้องพิจารณาเพิ่มเติม:

```python
# Python script คำนวณ optimal pool size
import psutil

def calculate_optimal_pool_size():
    cpu_count = psutil.cpu_count(logical=False)  # Physical cores
    
    # สำหรับ OLTP workload
    oltp_pool = (cpu_count * 2) + 1
    
    # สำหรับ OLAP workload (query นาน, IO-bound)
    olap_pool = cpu_count
    
    # สำหรับ mixed workload
    mixed_pool = int((cpu_count * 2.5) + 2)
    
    print(f"CPU Physical Cores: {cpu_count}")
    print(f"OLTP Pool Size: {oltp_pool}")
    print(f"OLAP Pool Size: {olap_pool}")
    print(f"Mixed Pool Size: {mixed_pool}")
    
    # ตรวจสอบว่าเพียงพอหรือไม่
    total_with_overhead = mixed_pool + (mixed_pool * 0.1)  # 10% overhead
    print(f"Total with 10% overhead: {total_with_overhead}")
    
    return mixed_pool

calculate_optimal_pool_size()
```

### Benchmark Pool Sizes

```bash
# ใช้ pgbench ทดสอบ
# ติดตั้ง pgbench database
pgbench -i -s 100 mydb

# Test ด้วย connection count ต่างๆ
for connections in 10 20 30 50 100 200; do
    echo "Testing with $connections connections..."
    pgbench -c $connections -j 4 -T 60 mydb 2>&1 | grep "tps ="
done
```

ผลลัพธ์ตัวอย่าง:
```
Testing with 10 connections...
tps = 8,234 (excluding connections establishing)
Testing with 20 connections...
tps = 14,567 (excluding connections establishing)
Testing with 30 connections...
tps = 18,901 (excluding connections establishing)
Testing with 50 connections...
tps = 19,234 (excluding connections establishing)  # Peak!
Testing with 100 connections...
tps = 17,456 (excluding connections establishing)  # Degrading
Testing with 200 connections...
tps = 12,123 (excluding connections establishing)  # Thrashing
```

---

## PgBouncer: Advanced Configuration

PgBouncer เป็น connection pooler ที่ทำงานระหว่าง application และ PostgreSQL:

```
Application → PgBouncer → PostgreSQL
(1000 clients) → (pooler) → (20 real connections)
```

### Installation และ Basic Setup

```bash
# Ubuntu/Debian
sudo apt-get install pgbouncer

# CentOS/RHEL
sudo yum install pgbouncer

# ตรวจสอบ version
pgbouncer --version
```

### PgBouncer Configuration File: pgbouncer.ini

```ini
[databases]
# Format: dbname = host=... port=... dbname=... user=...
myapp = host=localhost port=5432 dbname=myapp user=pgbouncer_user
myapp_ro = host=replica.example.com port=5432 dbname=myapp user=pgbouncer_ro
analytics = host=analytics.example.com port=5432 dbname=analytics

# Wildcard - ทุก database ที่ client ขอ
* = host=localhost port=5432

[pgbouncer]
###############################
# POOL SETTINGS
###############################

# pool_mode: session, transaction, statement
# transaction: recommended สำหรับ most web apps
# session: ต้องการ SET commands หรือ temp tables
# statement: เข้มงวดที่สุด ไม่รองรับ multi-statement transactions
pool_mode = transaction

# Maximum connections per pool (per database, per user)
default_pool_size = 20

# Minimum connections to keep warm
min_pool_size = 5

# Reserve connections สำหรับ emergency
reserve_pool_size = 5

# Time ที่รอก่อนจะใช้ reserve pool (seconds)
reserve_pool_timeout = 3.0

# Maximum total connections จาก clients
max_client_conn = 1000

###############################
# SERVER CONNECTIONS
###############################

# Maximum connections ต่อ server pool
server_pool_size = 20

# Time idle server connection ก่อน disconnect (seconds)
server_idle_timeout = 600

# Maximum connection lifetime (seconds)
server_lifetime = 3600

# ใช้ prepared statements กับ server
server_check_query = select 1

# Time ที่รอ server connection (seconds)
server_connect_timeout = 15.0

# Time ที่รอ server login (seconds)
server_login_retry = 15.0

###############################
# CLIENT CONNECTIONS
###############################

# Time ที่ client ต้องเสร็จ login (seconds)
client_login_timeout = 60.0

# Time idle client connection ก่อน disconnect (seconds)
client_idle_timeout = 0

# Maximum wait time สำหรับ connection จาก pool (seconds)
query_wait_timeout = 120.0

###############################
# QUERY TIMEOUTS
###############################

# Maximum time ที่ query ทำงานได้ (0 = disabled)
query_timeout = 0

# Maximum time ที่รอ cancel request
cancel_wait_timeout = 10.0

# Maximum time client ได้รับ connection หลัง query
client_query_timeout = 0

###############################
# TLS/SSL
###############################

# สำหรับ production ควรเปิด TLS
client_tls_sslmode = prefer
client_tls_ca_file = /etc/ssl/certs/ca.crt
client_tls_cert_file = /etc/pgbouncer/server.crt
client_tls_key_file = /etc/pgbouncer/server.key

server_tls_sslmode = require

###############################
# LOGGING
###############################

# ระดับ log
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1

# Log ทุก query (ระวัง performance impact)
verbose = 0

# Log file location
logfile = /var/log/pgbouncer/pgbouncer.log

# PID file
pidfile = /var/run/pgbouncer/pgbouncer.pid

###############################
# ADMINISTRATION
###############################

# Admin users
admin_users = postgres, pgbouncer_admin

# Stats users (can connect to pgbouncer database)
stats_users = monitoring, pgbouncer_stats

# Admin console listen address
listen_addr = *
listen_port = 6432

# Unix socket
unix_socket_dir = /var/run/postgresql

###############################
# MONITORING
###############################

# Statistics update interval (seconds)
stats_period = 60

# Maximum idle pool size
max_db_connections = 50

# Authentication
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
auth_query = SELECT usename, passwd FROM pg_shadow WHERE usename=$1
auth_dbname = pgbouncer
```

### PgBouncer Authentication

```bash
# สร้าง userlist.txt
cat > /etc/pgbouncer/userlist.txt << 'EOF'
"app_user" "md5password_hash"
"read_user" "md5password_hash"
EOF

# หรือใช้ auth_query จาก PostgreSQL
# สร้าง pgbouncer user ใน PostgreSQL
psql -c "CREATE ROLE pgbouncer_auth LOGIN PASSWORD 'secure_password';"
psql -c "GRANT SELECT ON pg_shadow TO pgbouncer_auth;"
```

### Pool Modes อธิบายละเอียด

```
Session Pooling:
- Client ได้รับ server connection ตลอด session
- เมื่อ client disconnect, connection กลับสู่ pool
- รองรับ: SET parameters, prepared statements, temp tables
- เหมาะ: applications ที่ต้องการ session-level features

Transaction Pooling:
- Client ได้รับ connection เฉพาะตอน transaction
- เมื่อ transaction เสร็จ, connection กลับสู่ pool
- ไม่รองรับ: SET (ยกเว้นบาง vars), named prepared statements
- เหมาะ: web applications (Django, Rails, Node.js)

Statement Pooling:
- ส่ง connection กลับหลังทุก statement
- ไม่รองรับ: transactions ที่มีหลาย statements
- เหมาะ: simple read-only queries
```

---

## PgBouncer SHOW Commands สำหรับ Monitoring

```sql
-- เชื่อมต่อไปยัง pgbouncer admin
psql -h localhost -p 6432 -U pgbouncer_admin pgbouncer

-- ดู stats ภาพรวม
SHOW STATS;

-- ดู pools ปัจจุบัน
SHOW POOLS;

-- ดู clients ที่เชื่อมต่ออยู่
SHOW CLIENTS;

-- ดู server connections
SHOW SERVERS;

-- ดู databases ที่ configured
SHOW DATABASES;

-- ดู configuration
SHOW CONFIG;

-- ดู version
SHOW VERSION;

-- Pause pool (ระหว่าง maintenance)
PAUSE myapp;

-- Resume pool
RESUME myapp;

-- Reload configuration
RELOAD;

-- Shutdown gracefully
SHUTDOWN;
```

### ตีความ SHOW POOLS Output

```sql
SHOW POOLS;

-- Output columns:
-- database: ชื่อ database
-- user: ชื่อ user
-- cl_active: clients ที่กำลัง query
-- cl_waiting: clients ที่รอ connection
-- cl_cancel_req: cancel requests ที่รอ
-- sv_active: server connections ที่ใช้งานอยู่
-- sv_idle: server connections ที่ว่าง
-- sv_used: server connections ที่เคยใช้
-- sv_tested: server connections กำลัง test
-- sv_login: server connections กำลัง login
-- maxwait: เวลารอนานสุด (seconds)
-- maxwait_us: เวลารอนานสุด (microseconds)
-- pool_mode: session/transaction/statement
```

---

## Application-Level Connection Pool: Node.js (pg)

### การติดตั้งและ Configuration พื้นฐาน

```typescript
// ติดตั้ง: npm install pg @types/pg

import { Pool, PoolConfig, PoolClient } from 'pg';

// Configuration สำหรับ production
const poolConfig: PoolConfig = {
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT || '5432'),
    database: process.env.DB_NAME || 'myapp',
    user: process.env.DB_USER || 'app_user',
    password: process.env.DB_PASSWORD,
    
    // Pool size settings
    max: 20,                      // Maximum connections in pool
    min: 2,                       // Minimum connections to maintain
    
    // Timeout settings
    idleTimeoutMillis: 30000,     // Close idle connections after 30s
    connectionTimeoutMillis: 5000, // Timeout waiting for connection: 5s
    
    // SSL สำหรับ production
    ssl: process.env.NODE_ENV === 'production' ? {
        rejectUnauthorized: true,
        ca: process.env.DB_SSL_CA,
    } : false,
    
    // Statement timeout
    statement_timeout: 30000,     // 30 seconds
    query_timeout: 30000,
    
    // Application name สำหรับ monitoring
    application_name: 'myapp-api',
};

const pool = new Pool(poolConfig);
```

### Pool Events: Monitoring สถานะ Pool

```typescript
// Event: เมื่อมี connection ใหม่
pool.on('connect', (client: PoolClient) => {
    console.log('New client connected to pool');
    
    // Set session-level parameters
    client.query(`
        SET statement_timeout = '30s';
        SET lock_timeout = '5s';
        SET idle_in_transaction_session_timeout = '60s';
        SET application_name = 'myapp-api';
    `);
});

// Event: เมื่อ client ถูก acquire จาก pool
pool.on('acquire', (client: PoolClient) => {
    // Logging สำหรับ debugging
    if (process.env.DEBUG_POOL) {
        console.log(`Client acquired. Pool stats: total=${pool.totalCount}, idle=${pool.idleCount}, waiting=${pool.waitingCount}`);
    }
});

// Event: เมื่อ connection ถูก remove จาก pool
pool.on('remove', (client: PoolClient) => {
    console.log('Client removed from pool');
    console.log(`Pool stats: total=${pool.totalCount}, idle=${pool.idleCount}`);
});

// Event: เมื่อเกิด error บน idle client
pool.on('error', (err: Error, client: PoolClient) => {
    console.error('Unexpected error on idle client', err);
    process.exit(-1);
});
```

### Pool Statistics: ดูสถานะ Real-time

```typescript
interface PoolStats {
    totalCount: number;      // Total connections (active + idle)
    idleCount: number;       // Idle connections
    waitingCount: number;    // Requests waiting for connection
    activeCount: number;     // Active connections (calculated)
    utilizationPercent: number;
}

function getPoolStats(): PoolStats {
    const total = pool.totalCount;
    const idle = pool.idleCount;
    const waiting = pool.waitingCount;
    const active = total - idle;
    const maxConnections = poolConfig.max || 10;
    
    return {
        totalCount: total,
        idleCount: idle,
        waitingCount: waiting,
        activeCount: active,
        utilizationPercent: (active / maxConnections) * 100
    };
}

// ตรวจสอบทุก 30 วินาที
setInterval(() => {
    const stats = getPoolStats();
    console.log('Pool Stats:', JSON.stringify(stats, null, 2));
    
    // Alert เมื่อใกล้เต็ม
    if (stats.utilizationPercent > 80) {
        console.warn(`WARNING: Pool utilization at ${stats.utilizationPercent.toFixed(1)}%`);
    }
    
    // Alert เมื่อมี waiting requests
    if (stats.waitingCount > 5) {
        console.warn(`WARNING: ${stats.waitingCount} requests waiting for connections`);
    }
}, 30000);
```

---

## Multiple Pool Strategies

### Strategy 1: แยก Pool สำหรับ OLTP และ OLAP

```typescript
// OLTP Pool: transactions สั้น, throughput สูง
const oltpPool = new Pool({
    host: process.env.DB_HOST,
    database: process.env.DB_NAME,
    user: process.env.DB_OLTP_USER,
    password: process.env.DB_OLTP_PASSWORD,
    max: 30,                        // More connections
    min: 5,
    idleTimeoutMillis: 10000,       // Short idle timeout
    connectionTimeoutMillis: 3000,  // Short connection timeout
    statement_timeout: 10000,       // 10s max query time
});

// OLAP Pool: long-running queries, fewer connections
const olapPool = new Pool({
    host: process.env.DB_ANALYTICS_HOST,  // Separate analytics server
    database: process.env.DB_NAME,
    user: process.env.DB_OLAP_USER,
    password: process.env.DB_OLAP_PASSWORD,
    max: 5,                         // Fewer connections
    min: 1,
    idleTimeoutMillis: 60000,       // Longer idle timeout
    connectionTimeoutMillis: 30000, // Longer connection timeout
    statement_timeout: 300000,      // 5 minutes max
});

// Route queries ไปยัง pool ที่เหมาะสม
async function executeQuery(
    sql: string, 
    params: any[], 
    isAnalytics: boolean = false
): Promise<any> {
    const targetPool = isAnalytics ? olapPool : oltpPool;
    return targetPool.query(sql, params);
}
```

### Strategy 2: แยก Pool สำหรับ Read และ Write

```typescript
// Write Pool: Primary database
const writePool = new Pool({
    host: process.env.PRIMARY_DB_HOST,
    database: process.env.DB_NAME,
    user: process.env.DB_WRITE_USER,
    password: process.env.DB_WRITE_PASSWORD,
    max: 15,
    min: 2,
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 5000,
});

// Read Pool: Replica database  
const readPool = new Pool({
    host: process.env.REPLICA_DB_HOST,
    database: process.env.DB_NAME,
    user: process.env.DB_READ_USER,
    password: process.env.DB_READ_PASSWORD,
    max: 25,                        // More read connections
    min: 5,
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 5000,
});

// Helper functions
async function query(sql: string, params?: any[]): Promise<any> {
    // Auto-detect read vs write
    const isReadQuery = /^\s*(SELECT|WITH\s+\w+\s+AS)/i.test(sql);
    
    if (isReadQuery) {
        try {
            return await readPool.query(sql, params);
        } catch (err) {
            // Fallback to primary if replica is unavailable
            console.warn('Replica unavailable, falling back to primary');
            return writePool.query(sql, params);
        }
    }
    
    return writePool.query(sql, params);
}

async function transaction<T>(
    callback: (client: PoolClient) => Promise<T>
): Promise<T> {
    const client = await writePool.connect();
    try {
        await client.query('BEGIN');
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

### Strategy 3: Priority Queue Pool

```typescript
import { EventEmitter } from 'events';

interface QueuedRequest {
    priority: number;
    resolve: (client: PoolClient) => void;
    reject: (error: Error) => void;
    timeout: NodeJS.Timeout;
}

class PriorityConnectionPool extends EventEmitter {
    private pool: Pool;
    private waitQueue: QueuedRequest[] = [];
    private maxWaitTime: number;
    
    constructor(config: PoolConfig, maxWaitTime: number = 10000) {
        super();
        this.pool = new Pool(config);
        this.maxWaitTime = maxWaitTime;
        
        // Listen สำหรับ released connections
        this.pool.on('connect', () => this.processQueue());
    }
    
    async acquireWithPriority(
        priority: number = 5,
        timeout?: number
    ): Promise<PoolClient> {
        return new Promise((resolve, reject) => {
            const waitTimeout = timeout || this.maxWaitTime;
            
            // พยายาม acquire ทันที
            this.pool.connect()
                .then(resolve)
                .catch(() => {
                    // เพิ่มในคิว
                    const timeoutHandle = setTimeout(() => {
                        // ลบออกจากคิว
                        const index = this.waitQueue.findIndex(r => r.timeout === timeoutHandle);
                        if (index !== -1) {
                            this.waitQueue.splice(index, 1);
                        }
                        reject(new Error(`Connection timeout after ${waitTimeout}ms`));
                    }, waitTimeout);
                    
                    this.waitQueue.push({
                        priority,
                        resolve,
                        reject,
                        timeout: timeoutHandle
                    });
                    
                    // Sort by priority (higher = first)
                    this.waitQueue.sort((a, b) => b.priority - a.priority);
                });
        });
    }
    
    private processQueue(): void {
        if (this.waitQueue.length > 0) {
            const next = this.waitQueue.shift()!;
            clearTimeout(next.timeout);
            
            this.pool.connect()
                .then(next.resolve)
                .catch(next.reject);
        }
    }
    
    async query(
        sql: string, 
        params?: any[], 
        priority: number = 5
    ): Promise<any> {
        const client = await this.acquireWithPriority(priority);
        try {
            return await client.query(sql, params);
        } finally {
            client.release();
        }
    }
    
    getStats() {
        return {
            ...getPoolStats(),
            queueLength: this.waitQueue.length,
            averagePriority: this.waitQueue.reduce((sum, r) => sum + r.priority, 0) / 
                            (this.waitQueue.length || 1)
        };
    }
}

// การใช้งาน
const priorityPool = new PriorityConnectionPool({
    host: 'localhost',
    database: 'myapp',
    user: 'app_user',
    password: 'password',
    max: 20,
});

// Critical operations (priority 10)
await priorityPool.query('SELECT * FROM critical_data', [], 10);

// Normal operations (priority 5)
await priorityPool.query('SELECT * FROM normal_data', [], 5);

// Background operations (priority 1)
await priorityPool.query('SELECT * FROM background_data', [], 1);
```

---

## Connection String Parameters สำหรับ Timeouts

```typescript
// สร้าง connection string ที่มี timeout parameters
function buildConnectionString(options: {
    host: string;
    database: string;
    user: string;
    password: string;
    statementTimeout?: number;     // ms
    lockTimeout?: number;          // ms
    idleInTransactionTimeout?: number; // ms
}): string {
    const params = new URLSearchParams({
        application_name: 'myapp',
        options: [
            options.statementTimeout ? 
                `-c statement_timeout=${options.statementTimeout}` : '',
            options.lockTimeout ? 
                `-c lock_timeout=${options.lockTimeout}` : '',
            options.idleInTransactionTimeout ? 
                `-c idle_in_transaction_session_timeout=${options.idleInTransactionTimeout}` : '',
        ].filter(Boolean).join(' ')
    });
    
    return `postgresql://${options.user}:${options.password}@${options.host}/${options.database}?${params}`;
}

// ตัวอย่างการใช้
const connectionString = buildConnectionString({
    host: 'localhost',
    database: 'myapp',
    user: 'app_user',
    password: 'secure_pass',
    statementTimeout: 30000,       // 30 seconds
    lockTimeout: 5000,             // 5 seconds
    idleInTransactionTimeout: 60000, // 1 minute
});
```

### PostgreSQL Timeout Parameters ทั้งหมด

```sql
-- statement_timeout: ยกเลิก query ที่ทำงานนานเกิน
SET statement_timeout = '30s';

-- lock_timeout: ยกเลิกถ้ารอ lock นานเกิน
SET lock_timeout = '10s';

-- idle_in_transaction_session_timeout: ยกเลิก transaction ที่ idle นานเกิน
SET idle_in_transaction_session_timeout = '5min';

-- deadlock_timeout: เวลารอก่อน detect deadlock
SET deadlock_timeout = '1s';

-- ดูค่า timeout ปัจจุบัน
SHOW statement_timeout;
SHOW lock_timeout;
SHOW idle_in_transaction_session_timeout;

-- ตั้งค่าระดับ database
ALTER DATABASE myapp SET statement_timeout = '60s';
ALTER DATABASE myapp SET idle_in_transaction_session_timeout = '5min';

-- ตั้งค่าระดับ role
ALTER ROLE app_user SET statement_timeout = '30s';
ALTER ROLE background_worker SET statement_timeout = '300s';
```

---

## Prepared Statements ใน Connection Pooling

### ปัญหา: Transaction Mode ไม่รองรับ Named Prepared Statements

```
Session mode:
  Client → Pool → Server (same connection throughout)
  PREPARE stmt1 AS SELECT...  ✓ Works
  EXECUTE stmt1               ✓ Works

Transaction mode:
  Client → Pool → Server A (transaction 1)
  PREPARE stmt1 AS SELECT...  ✓ Works on Server A
  Connection returns to pool
  Client → Pool → Server B (different connection!)
  EXECUTE stmt1               ✗ FAILS: "prepared statement does not exist"
```

### Workaround 1: Protocol-level Prepared Statements

```typescript
// pg library รองรับ protocol-level prepared statements
// ที่ไม่ต้องการ named statements ระดับ SQL

const pool = new Pool({...});

// วิธีนี้ใช้ PostgreSQL extended query protocol
// PgBouncer ใน transaction mode รองรับ anonymous prepared statements
const result = await pool.query({
    text: 'SELECT * FROM users WHERE id = $1',
    values: [userId],
    // ไม่ระบุ name = anonymous prepared statement
});
```

### Workaround 2: ปิด Prepared Statements

```typescript
// สำหรับ PgBouncer ใน transaction mode
// ใช้ simple query protocol แทน extended protocol
const pool = new Pool({
    ...poolConfig,
    // Force simple query protocol
    // ไม่รองรับ parameter binding โดยตรง แต่ปลอดภัยกับ pgbouncer
});

// หรือใช้ knex.js ที่รองรับ pgbouncer mode
const knex = require('knex')({
    client: 'postgresql',
    connection: connectionConfig,
    pool: { min: 2, max: 20 },
    // Disable prepared statements สำหรับ pgbouncer
    acquireConnectionTimeout: 10000,
});
```

### Workaround 3: Prisma กับ PgBouncer

```typescript
// Prisma datasource
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
  // เพิ่ม pgbouncer=true parameter
  // DATABASE_URL="postgresql://user:pass@pgbouncer:6432/mydb?pgbouncer=true"
}

// ใน .env file
DATABASE_URL="postgresql://user:pass@pgbouncer:6432/mydb?pgbouncer=true&connection_limit=1"

// เมื่อใช้ pgbouncer=true:
// - Prisma ปิด prepared statements
// - ใช้ connection_limit=1 สำหรับ serverless environments
```

### Workaround 4: Statement Caching ใน Application

```typescript
class PreparedStatementCache {
    private cache = new Map<string, { text: string; paramCount: number }>();
    private pool: Pool;
    
    constructor(pool: Pool) {
        this.pool = pool;
    }
    
    async query(name: string, text: string, values?: any[]): Promise<any> {
        // Cache the query text (ไม่ใช้ PostgreSQL PREPARE)
        if (!this.cache.has(name)) {
            this.cache.set(name, {
                text,
                paramCount: values?.length || 0
            });
        }
        
        const cached = this.cache.get(name)!;
        
        // Execute as regular parameterized query
        // PostgreSQL จะ cache execution plan อัตโนมัติหลัง 5 executions
        return this.pool.query(cached.text, values);
    }
    
    clearCache(): void {
        this.cache.clear();
    }
    
    getCacheStats() {
        return {
            cachedStatements: this.cache.size,
            statements: Array.from(this.cache.keys())
        };
    }
}

// การใช้งาน
const stmtCache = new PreparedStatementCache(pool);

// แทนที่จะใช้ PREPARE/EXECUTE
await stmtCache.query(
    'get_user_by_id',
    'SELECT * FROM users WHERE id = $1',
    [userId]
);
```

---

## Connection Leak Detection

### การ Detect Connection Leaks

```typescript
class LeakDetectingPool {
    private pool: Pool;
    private activeConnections = new Map<PoolClient, {
        acquiredAt: Date;
        stackTrace: string;
    }>();
    private leakThreshold: number;
    
    constructor(config: PoolConfig, leakThreshold: number = 30000) {
        this.pool = new Pool(config);
        this.leakThreshold = leakThreshold;
        
        // ตรวจสอบ leaks ทุก 10 วินาที
        setInterval(() => this.checkForLeaks(), 10000);
    }
    
    async connect(): Promise<PoolClient> {
        const client = await this.pool.connect();
        
        // Wrap release function
        const originalRelease = client.release.bind(client);
        const acquiredAt = new Date();
        const stackTrace = new Error().stack || '';
        
        this.activeConnections.set(client, { acquiredAt, stackTrace });
        
        // Override release
        (client as any).release = (err?: Error) => {
            this.activeConnections.delete(client);
            return originalRelease(err);
        };
        
        return client;
    }
    
    private checkForLeaks(): void {
        const now = new Date();
        
        for (const [client, info] of this.activeConnections) {
            const heldTime = now.getTime() - info.acquiredAt.getTime();
            
            if (heldTime > this.leakThreshold) {
                console.error(`
                    CONNECTION LEAK DETECTED!
                    Connection held for ${heldTime}ms
                    Acquired at: ${info.acquiredAt.toISOString()}
                    Stack trace:
                    ${info.stackTrace}
                `);
                
                // Optional: force release leaking connection
                // client.release(true);
            }
        }
    }
    
    async query(text: string, values?: any[]): Promise<any> {
        const client = await this.connect();
        try {
            return await client.query(text, values);
        } finally {
            (client as any).release();
        }
    }
    
    getStats() {
        return {
            activeConnections: this.activeConnections.size,
            poolStats: {
                total: this.pool.totalCount,
                idle: this.pool.idleCount,
                waiting: this.pool.waitingCount,
            }
        };
    }
}
```

### PostgreSQL Query สำหรับ Detect Idle Connections

```sql
-- ดู connections ที่ idle ใน transaction นานเกิน 5 นาที
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query_start,
    state_change,
    EXTRACT(EPOCH FROM (NOW() - state_change)) as seconds_in_state,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
    AND state_change < NOW() - INTERVAL '5 minutes'
ORDER BY state_change;

-- Kill idle in transaction connections
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
    AND state_change < NOW() - INTERVAL '5 minutes'
    AND pid != pg_backend_pid();

-- ดู long-running queries
SELECT 
    pid,
    NOW() - pg_stat_activity.query_start AS duration,
    query,
    state
FROM pg_stat_activity
WHERE (NOW() - pg_stat_activity.query_start) > INTERVAL '5 minutes'
    AND state != 'idle'
ORDER BY duration DESC;
```

---

## Pool Exhaustion Handling

### Strategy 1: Circuit Breaker Pattern

```typescript
enum CircuitState {
    CLOSED = 'CLOSED',     // Normal operation
    OPEN = 'OPEN',         // Rejecting all requests
    HALF_OPEN = 'HALF_OPEN' // Testing recovery
}

class CircuitBreakerPool {
    private pool: Pool;
    private state: CircuitState = CircuitState.CLOSED;
    private failures = 0;
    private lastFailureTime?: Date;
    private successCount = 0;
    
    private readonly failureThreshold = 5;
    private readonly resetTimeout = 30000;    // 30 seconds
    private readonly halfOpenSuccessThreshold = 3;
    
    constructor(config: PoolConfig) {
        this.pool = new Pool(config);
    }
    
    async query(text: string, values?: any[]): Promise<any> {
        if (this.state === CircuitState.OPEN) {
            const timeSinceLastFailure = this.lastFailureTime ? 
                Date.now() - this.lastFailureTime.getTime() : 0;
            
            if (timeSinceLastFailure < this.resetTimeout) {
                throw new Error('Circuit breaker is OPEN - database temporarily unavailable');
            }
            
            // ลอง half-open
            this.state = CircuitState.HALF_OPEN;
            this.successCount = 0;
        }
        
        try {
            const result = await this.pool.query(text, values);
            this.onSuccess();
            return result;
        } catch (err) {
            this.onFailure();
            throw err;
        }
    }
    
    private onSuccess(): void {
        if (this.state === CircuitState.HALF_OPEN) {
            this.successCount++;
            if (this.successCount >= this.halfOpenSuccessThreshold) {
                this.state = CircuitState.CLOSED;
                this.failures = 0;
                console.log('Circuit breaker: HALF_OPEN → CLOSED');
            }
        } else {
            this.failures = 0;
        }
    }
    
    private onFailure(): void {
        this.failures++;
        this.lastFailureTime = new Date();
        
        if (this.failures >= this.failureThreshold) {
            this.state = CircuitState.OPEN;
            console.error(`Circuit breaker: OPEN after ${this.failures} failures`);
        }
    }
    
    getState(): CircuitState {
        return this.state;
    }
}
```

### Strategy 2: Queue ด้วย Backpressure

```typescript
interface QueuedQuery {
    text: string;
    values?: any[];
    resolve: (result: any) => void;
    reject: (error: Error) => void;
    queuedAt: Date;
    maxWaitMs: number;
}

class BackpressurePool {
    private pool: Pool;
    private queryQueue: QueuedQuery[] = [];
    private processing = false;
    
    private readonly maxQueueSize: number;
    private readonly defaultMaxWait: number;
    
    constructor(
        config: PoolConfig, 
        maxQueueSize: number = 100,
        defaultMaxWait: number = 10000
    ) {
        this.pool = new Pool(config);
        this.maxQueueSize = maxQueueSize;
        this.defaultMaxWait = defaultMaxWait;
    }
    
    async query(
        text: string, 
        values?: any[], 
        maxWaitMs?: number
    ): Promise<any> {
        // ลอง execute ทันที
        try {
            return await Promise.race([
                this.pool.query(text, values),
                new Promise((_, reject) => 
                    setTimeout(() => reject(new Error('immediate_timeout')), 100)
                )
            ]);
        } catch (err: any) {
            if (err.message !== 'immediate_timeout') throw err;
        }
        
        // เพิ่มในคิว
        if (this.queryQueue.length >= this.maxQueueSize) {
            throw new Error(`Queue full: ${this.queryQueue.length}/${this.maxQueueSize} requests waiting`);
        }
        
        return new Promise((resolve, reject) => {
            const waitTime = maxWaitMs || this.defaultMaxWait;
            
            this.queryQueue.push({
                text,
                values,
                resolve,
                reject,
                queuedAt: new Date(),
                maxWaitMs: waitTime
            });
            
            // ตั้ง timeout
            setTimeout(() => {
                const index = this.queryQueue.findIndex(q => 
                    q.queuedAt === this.queryQueue[this.queryQueue.length - 1]?.queuedAt
                );
                if (index !== -1) {
                    const queued = this.queryQueue.splice(index, 1)[0];
                    const waitTime = Date.now() - queued.queuedAt.getTime();
                    queued.reject(new Error(`Query waited ${waitTime}ms, exceeded limit`));
                }
            }, waitTime);
            
            this.processQueue();
        });
    }
    
    private async processQueue(): Promise<void> {
        if (this.processing || this.queryQueue.length === 0) return;
        
        this.processing = true;
        
        while (this.queryQueue.length > 0) {
            const queued = this.queryQueue.shift()!;
            const waitTime = Date.now() - queued.queuedAt.getTime();
            
            try {
                const result = await this.pool.query(queued.text, queued.values);
                queued.resolve(result);
            } catch (err) {
                queued.reject(err as Error);
            }
        }
        
        this.processing = false;
    }
    
    getQueueStats() {
        return {
            queueLength: this.queryQueue.length,
            maxQueueSize: this.maxQueueSize,
            queueUtilization: (this.queryQueue.length / this.maxQueueSize) * 100,
            oldestInQueue: this.queryQueue[0] ? 
                Date.now() - this.queryQueue[0].queuedAt.getTime() : 0
        };
    }
}
```

---

## DatabasePool Manager Class: Full TypeScript Implementation

```typescript
import { Pool, PoolClient, PoolConfig, QueryResult } from 'pg';
import { EventEmitter } from 'events';

interface DatabasePoolConfig {
    primary: PoolConfig;
    replica?: PoolConfig;
    olap?: PoolConfig;
    poolName?: string;
    enableMetrics?: boolean;
    leakDetection?: boolean;
    leakThresholdMs?: number;
    circuitBreaker?: boolean;
}

interface QueryOptions {
    useReplica?: boolean;
    priority?: number;
    timeout?: number;
    label?: string;
}

interface PoolMetrics {
    totalQueries: number;
    failedQueries: number;
    totalWaitTime: number;
    totalQueryTime: number;
    avgWaitTime: number;
    avgQueryTime: number;
    maxWaitTime: number;
    maxQueryTime: number;
    connectionErrors: number;
}

class DatabasePoolManager extends EventEmitter {
    private primaryPool: Pool;
    private replicaPool?: Pool;
    private olapPool?: Pool;
    private config: DatabasePoolConfig;
    private metrics: PoolMetrics;
    private activeConnections = new Map<PoolClient, {
        acquiredAt: number;
        stackTrace: string;
        label?: string;
    }>();
    
    // Circuit breaker state
    private circuitOpen = false;
    private circuitFailures = 0;
    private circuitLastFailure?: number;
    
    constructor(config: DatabasePoolConfig) {
        super();
        this.config = config;
        this.metrics = this.initMetrics();
        
        // สร้าง pools
        this.primaryPool = this.createPool(config.primary, 'primary');
        
        if (config.replica) {
            this.replicaPool = this.createPool(config.replica, 'replica');
        }
        
        if (config.olap) {
            this.olapPool = this.createPool(config.olap, 'olap');
        }
        
        // ตรวจสอบ connection leaks
        if (config.leakDetection) {
            setInterval(() => this.checkLeaks(), 15000);
        }
        
        // ตรวจสอบ pool health
        setInterval(() => this.healthCheck(), 30000);
    }
    
    private initMetrics(): PoolMetrics {
        return {
            totalQueries: 0,
            failedQueries: 0,
            totalWaitTime: 0,
            totalQueryTime: 0,
            avgWaitTime: 0,
            avgQueryTime: 0,
            maxWaitTime: 0,
            maxQueryTime: 0,
            connectionErrors: 0,
        };
    }
    
    private createPool(config: PoolConfig, name: string): Pool {
        const pool = new Pool({
            ...config,
            application_name: `${this.config.poolName || 'app'}-${name}`,
        });
        
        pool.on('connect', (client) => {
            this.emit('connect', { pool: name, client });
            // ตั้งค่า session defaults
            client.query(`
                SET statement_timeout = '${config.statement_timeout || 30000}';
                SET lock_timeout = '5000';
                SET idle_in_transaction_session_timeout = '60000';
            `).catch(console.error);
        });
        
        pool.on('error', (err, client) => {
            this.metrics.connectionErrors++;
            this.emit('error', { pool: name, error: err });
            console.error(`Pool ${name} error:`, err);
        });
        
        return pool;
    }
    
    async query(
        text: string,
        values?: any[],
        options: QueryOptions = {}
    ): Promise<QueryResult> {
        const startWait = Date.now();
        
        // Circuit breaker check
        if (this.circuitOpen && this.config.circuitBreaker) {
            const timeSince = this.circuitLastFailure ? 
                Date.now() - this.circuitLastFailure : Infinity;
            
            if (timeSince < 30000) {
                throw new Error('Database circuit breaker is OPEN');
            }
            this.circuitOpen = false;
            this.circuitFailures = 0;
        }
        
        // เลือก pool
        const pool = this.selectPool(text, options);
        
        let client: PoolClient | null = null;
        
        try {
            // Acquire connection
            client = await Promise.race([
                pool.connect(),
                new Promise<never>((_, reject) => 
                    setTimeout(
                        () => reject(new Error(`Connection timeout after ${options.timeout || 5000}ms`)),
                        options.timeout || 5000
                    )
                )
            ]);
            
            const waitTime = Date.now() - startWait;
            this.updateWaitMetrics(waitTime);
            
            // Track connection
            if (this.config.leakDetection) {
                this.activeConnections.set(client, {
                    acquiredAt: Date.now(),
                    stackTrace: new Error().stack || '',
                    label: options.label,
                });
            }
            
            // Execute query
            const queryStart = Date.now();
            
            const result = await client.query(text, values);
            
            const queryTime = Date.now() - queryStart;
            this.metrics.totalQueries++;
            this.updateQueryMetrics(queryTime);
            this.circuitFailures = 0;
            
            return result;
            
        } catch (err) {
            this.metrics.failedQueries++;
            this.circuitFailures++;
            
            if (this.circuitFailures >= 5 && this.config.circuitBreaker) {
                this.circuitOpen = true;
                this.circuitLastFailure = Date.now();
                this.emit('circuit-open', { failures: this.circuitFailures });
            }
            
            throw err;
            
        } finally {
            if (client) {
                if (this.config.leakDetection) {
                    this.activeConnections.delete(client);
                }
                client.release();
            }
        }
    }
    
    private selectPool(sql: string, options: QueryOptions): Pool {
        // OLAP queries
        if (this.olapPool && options.useReplica === false) {
            return this.olapPool;
        }
        
        // Read queries to replica
        if (this.replicaPool && (
            options.useReplica === true ||
            /^\s*(SELECT|WITH|EXPLAIN)\s/i.test(sql)
        )) {
            return this.replicaPool;
        }
        
        return this.primaryPool;
    }
    
    async transaction<T>(
        callback: (client: PoolClient) => Promise<T>,
        options: { isolationLevel?: string } = {}
    ): Promise<T> {
        const client = await this.primaryPool.connect();
        
        if (this.config.leakDetection) {
            this.activeConnections.set(client, {
                acquiredAt: Date.now(),
                stackTrace: new Error().stack || '',
                label: 'transaction',
            });
        }
        
        try {
            await client.query(
                options.isolationLevel 
                    ? `BEGIN ISOLATION LEVEL ${options.isolationLevel}`
                    : 'BEGIN'
            );
            
            const result = await callback(client);
            
            await client.query('COMMIT');
            this.metrics.totalQueries++;
            
            return result;
            
        } catch (err) {
            await client.query('ROLLBACK');
            this.metrics.failedQueries++;
            throw err;
            
        } finally {
            if (this.config.leakDetection) {
                this.activeConnections.delete(client);
            }
            client.release();
        }
    }
    
    private updateWaitMetrics(waitTime: number): void {
        this.metrics.totalWaitTime += waitTime;
        this.metrics.maxWaitTime = Math.max(this.metrics.maxWaitTime, waitTime);
        
        const totalRequests = this.metrics.totalQueries + 1;
        this.metrics.avgWaitTime = this.metrics.totalWaitTime / totalRequests;
    }
    
    private updateQueryMetrics(queryTime: number): void {
        this.metrics.totalQueryTime += queryTime;
        this.metrics.maxQueryTime = Math.max(this.metrics.maxQueryTime, queryTime);
        this.metrics.avgQueryTime = this.metrics.totalQueryTime / this.metrics.totalQueries;
    }
    
    private checkLeaks(): void {
        const threshold = this.config.leakThresholdMs || 30000;
        const now = Date.now();
        
        for (const [client, info] of this.activeConnections) {
            const heldTime = now - info.acquiredAt;
            
            if (heldTime > threshold) {
                this.emit('leak-detected', {
                    heldTime,
                    label: info.label,
                    stackTrace: info.stackTrace
                });
                
                console.error(`Connection leak: held ${heldTime}ms${info.label ? ` (${info.label})` : ''}`);
            }
        }
    }
    
    private async healthCheck(): Promise<void> {
        try {
            await this.primaryPool.query('SELECT 1');
            this.emit('health', { status: 'healthy', pool: 'primary' });
        } catch (err) {
            this.emit('health', { status: 'unhealthy', pool: 'primary', error: err });
        }
        
        if (this.replicaPool) {
            try {
                await this.replicaPool.query('SELECT 1');
                this.emit('health', { status: 'healthy', pool: 'replica' });
            } catch (err) {
                this.emit('health', { status: 'unhealthy', pool: 'replica', error: err });
            }
        }
    }
    
    getMetrics(): PoolMetrics & { 
        poolStats: {
            primary: any;
            replica?: any;
            olap?: any;
        };
        circuitBreaker: {
            open: boolean;
            failures: number;
        };
        activeLeaks: number;
    } {
        return {
            ...this.metrics,
            poolStats: {
                primary: {
                    total: this.primaryPool.totalCount,
                    idle: this.primaryPool.idleCount,
                    waiting: this.primaryPool.waitingCount,
                },
                replica: this.replicaPool ? {
                    total: this.replicaPool.totalCount,
                    idle: this.replicaPool.idleCount,
                    waiting: this.replicaPool.waitingCount,
                } : undefined,
                olap: this.olapPool ? {
                    total: this.olapPool.totalCount,
                    idle: this.olapPool.idleCount,
                    waiting: this.olapPool.waitingCount,
                } : undefined,
            },
            circuitBreaker: {
                open: this.circuitOpen,
                failures: this.circuitFailures,
            },
            activeLeaks: this.activeConnections.size,
        };
    }
    
    async shutdown(): Promise<void> {
        console.log('Shutting down database pools...');
        
        await Promise.all([
            this.primaryPool.end(),
            this.replicaPool?.end(),
            this.olapPool?.end(),
        ].filter(Boolean));
        
        console.log('All database pools closed');
    }
}

// การใช้งาน
const dbManager = new DatabasePoolManager({
    poolName: 'myapp',
    primary: {
        host: process.env.DB_PRIMARY_HOST || 'localhost',
        port: 5432,
        database: 'myapp',
        user: 'app_user',
        password: process.env.DB_PASSWORD,
        max: 20,
        min: 2,
        idleTimeoutMillis: 30000,
        connectionTimeoutMillis: 5000,
        statement_timeout: 30000,
    },
    replica: {
        host: process.env.DB_REPLICA_HOST,
        port: 5432,
        database: 'myapp',
        user: 'app_readonly',
        password: process.env.DB_READONLY_PASSWORD,
        max: 30,
        min: 5,
        idleTimeoutMillis: 30000,
        connectionTimeoutMillis: 5000,
        statement_timeout: 60000,
    },
    enableMetrics: true,
    leakDetection: true,
    leakThresholdMs: 30000,
    circuitBreaker: true,
});

// Event handlers
dbManager.on('leak-detected', ({ heldTime, label, stackTrace }) => {
    console.error(`Connection leak detected: ${heldTime}ms${label ? ` [${label}]` : ''}`);
});

dbManager.on('circuit-open', ({ failures }) => {
    console.error(`Circuit breaker opened after ${failures} failures`);
});

dbManager.on('health', ({ status, pool, error }) => {
    if (status === 'unhealthy') {
        console.error(`Pool ${pool} unhealthy:`, error);
    }
});

// Graceful shutdown
process.on('SIGTERM', async () => {
    await dbManager.shutdown();
    process.exit(0);
});

export default dbManager;
```

---

## Monitoring PgBouncer ด้วย Prometheus

```yaml
# docker-compose.yml สำหรับ monitoring stack
version: '3.8'
services:
  pgbouncer:
    image: edoburu/pgbouncer:latest
    environment:
      DATABASE_URL: "postgres://user:pass@postgres:5432/myapp"
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 1000
      DEFAULT_POOL_SIZE: 20
    ports:
      - "6432:5432"
    
  pgbouncer-exporter:
    image: prometheuscommunity/pgbouncer-exporter:latest
    environment:
      PGBOUNCER_EXPORTER_HOST: pgbouncer
      PGBOUNCER_EXPORTER_PORT: 5432
      PGBOUNCER_EXPORTER_USER: pgbouncer_stats
      PGBOUNCER_EXPORTER_PASSWORD: stats_password
    ports:
      - "9127:9127"
    
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'pgbouncer'
    static_configs:
      - targets: ['pgbouncer-exporter:9127']
    
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
```

### Grafana Dashboard Queries

```
# Pool utilization
pgbouncer_pools_sv_active / pgbouncer_config_max_client_conn * 100

# Wait queue length
sum(pgbouncer_pools_cl_waiting) by (database)

# Connection error rate
rate(pgbouncer_stats_total_query_count[5m])

# Average query wait time
pgbouncer_stats_avg_wait_time

# Server utilization per pool
pgbouncer_pools_sv_active / (pgbouncer_pools_sv_active + pgbouncer_pools_sv_idle) * 100
```

---

## Best Practices สรุป

```
1. เริ่มต้นด้วย pool_mode = transaction สำหรับ web applications
2. ใช้ default_pool_size = (CPU * 2) + 1 เป็น starting point
3. ตั้ง statement_timeout และ idle_in_transaction_session_timeout เสมอ
4. Monitor pool metrics: waiting count ควรใกล้ 0
5. ใช้ PgBouncer ระหว่าง app และ PostgreSQL เสมอสำหรับ production
6. แยก pool สำหรับ read/write เมื่อมี replica
7. แยก pool สำหรับ OLAP queries ที่ใช้เวลานาน
8. ใช้ circuit breaker สำหรับ fault tolerance
9. Implement leak detection ใน development environment
10. ตั้ง max_client_conn ให้สูงกว่า max connections ของ app
```

```sql
-- Query ตรวจสอบ pool health รายวัน
SELECT 
    datname,
    numbackends as current_connections,
    xact_commit as committed_transactions,
    xact_rollback as rolled_back_transactions,
    blks_read,
    blks_hit,
    ROUND(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) as cache_hit_ratio,
    conflicts,
    deadlocks,
    temp_files,
    temp_bytes
FROM pg_stat_database
WHERE datname = current_database();
```
