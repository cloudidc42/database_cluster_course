# Part 99: Load Testing กับ Database Cluster

## บทนำ: ทำไมต้อง Load Test ก่อน Go-Live?

```
ไม่มี Load Test:
  - ไม่รู้ว่า system รับ load ได้แค่ไหน
  - Production เกิดปัญหาเมื่อ traffic จริงมา
  - Database connection pool exhausted
  - Query ที่ทำงานได้ใน dev พัง production
  - ไม่รู้ว่า bottleneck อยู่ที่ไหน

มี Load Test:
  - รู้ capacity ล่วงหน้า
  - หา bottleneck และแก้ก่อน go-live
  - Validate SLO (P99 < 200ms)
  - Tune connection pool, cache, indexes
  - มั่นใจได้ว่า system พร้อม
```

---

## 1. ประเภทของ Load Testing

### 1.1 Smoke Test

ทดสอบ basic functionality ว่าระบบทำงานได้

```javascript
// smoke-test.js
export const options = {
  vus: 1,           // 1 virtual user
  duration: '1m',   // 1 นาที
  thresholds: {
    http_req_duration: ['p(99)<500'],  // รอได้นานขึ้นสำหรับ smoke test
    http_req_failed: ['rate<0.01'],
  },
};

export default function() {
  // ทดสอบแค่ว่า endpoint ทำงาน
  const res = http.get('http://api:3000/health');
  check(res, { 'health check OK': (r) => r.status === 200 });
  sleep(1);
}
```

### 1.2 Load Test

ทดสอบที่ traffic คาดหวัง (normal load)

```javascript
// load-test.js - Expected production traffic
export const options = {
  stages: [
    { duration: '5m', target: 100 },   // Ramp up to 100 VUs
    { duration: '30m', target: 100 },  // Stay at 100 VUs (normal load)
    { duration: '5m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(99)<200'],  // SLO: P99 < 200ms
    http_req_failed: ['rate<0.001'],   // < 0.1% error rate
  },
};
```

### 1.3 Stress Test

ทดสอบเกิน capacity เพื่อหา breaking point

```javascript
// stress-test.js - Push beyond capacity
export const options = {
  stages: [
    { duration: '10m', target: 200 },   // Normal load
    { duration: '10m', target: 500 },   // High load
    { duration: '10m', target: 1000 },  // Very high load
    { duration: '10m', target: 2000 },  // Extreme load
    { duration: '5m', target: 0 },      // Recovery
  ],
};
```

### 1.4 Soak Test (Endurance Test)

ทดสอบ long duration เพื่อหา memory leaks, connection leaks

```javascript
// soak-test.js - Run for 8+ hours
export const options = {
  stages: [
    { duration: '1h', target: 100 },   // Ramp up
    { duration: '8h', target: 100 },   // Sustained load
    { duration: '1h', target: 0 },     // Cool down
  ],
  thresholds: {
    http_req_duration: ['p(99)<200'],
    // Monitor memory usage over time
  },
};
```

### 1.5 Spike Test

ทดสอบ sudden traffic spike

```javascript
// spike-test.js - Simulate flash sale
export const options = {
  stages: [
    { duration: '1m', target: 100 },    // Normal baseline
    { duration: '30s', target: 5000 },  // Sudden spike!
    { duration: '5m', target: 5000 },   // Stay at spike
    { duration: '30s', target: 100 },   // Drop back to normal
    { duration: '5m', target: 100 },    // Recovery check
    { duration: '1m', target: 0 },
  ],
};
```

---

## 2. k6: Deep Dive

### 2.1 ติดตั้ง k6

```bash
# macOS
brew install k6

# Ubuntu/Debian
sudo gpg -k
sudo gpg --no-default-keyring \
  --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
  --keyserver hkp://keyserver.ubuntu.com:80 \
  --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" \
  | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update && sudo apt-get install k6

# Docker
docker run --rm -i grafana/k6 run - < script.js

# Verify
k6 version
```

### 2.2 k6 Script Structure

```javascript
// complete-k6-example.js
import http from 'k6/http';
import { sleep, check, group, fail } from 'k6';
import { Counter, Rate, Trend, Gauge } from 'k6/metrics';
import { SharedArray } from 'k6/data';
import { randomIntBetween, randomItem } from 'https://jslib.k6.io/k6-utils/1.4.0/index.js';

// ==================== Custom Metrics ====================
const dbQueryDuration = new Trend('db_query_duration', true);  // true = milliseconds
const cacheHitRate = new Rate('cache_hit_rate');
const orderCreated = new Counter('orders_created');
const activeUsers = new Gauge('active_users');

// ==================== Test Data ====================
// SharedArray: load ครั้งเดียว, share ระหว่าง VUs
const users = new SharedArray('users', function() {
  return JSON.parse(open('./test-data/users.json'));
});

const products = new SharedArray('products', function() {
  return JSON.parse(open('./test-data/products.json'));
});

// ==================== Options ====================
export const options = {
  // Scenarios: different workload patterns
  scenarios: {
    // 70% ของ traffic เป็น browse
    browse_products: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '5m', target: 70 },
        { duration: '20m', target: 70 },
        { duration: '5m', target: 0 },
      ],
      exec: 'browseProducts',
    },
    
    // 20% เป็น search
    search: {
      executor: 'constant-vus',
      vus: 20,
      duration: '30m',
      exec: 'searchProducts',
    },
    
    // 10% เป็น checkout
    checkout: {
      executor: 'per-vu-iterations',
      vus: 10,
      iterations: 100,
      maxDuration: '30m',
      exec: 'checkoutFlow',
    },
  },

  // SLO Thresholds
  thresholds: {
    // HTTP
    'http_req_duration': ['p(50)<50', 'p(95)<150', 'p(99)<200'],
    'http_req_failed': ['rate<0.001'],  // < 0.1% errors
    
    // Custom
    'db_query_duration': ['p(99)<100'],  // Database P99 < 100ms
    'cache_hit_rate': ['rate>0.80'],     // Cache hit > 80%
    
    // Per group
    'http_req_duration{scenario:checkout}': ['p(99)<500'],
  },
};

// ==================== Scenarios ====================

export function browseProducts() {
  const user = randomItem(users);
  
  activeUsers.add(1);
  
  group('browse products', function() {
    // GET product list
    const listRes = http.get(
      `http://api:3000/api/products?page=1&limit=20`,
      { headers: authHeaders(user) }
    );
    
    check(listRes, {
      'list status 200': (r) => r.status === 200,
      'list has products': (r) => JSON.parse(r.body).data.length > 0,
    });
    
    // Check if from cache
    const fromCache = listRes.headers['X-Cache'] === 'HIT';
    cacheHitRate.add(fromCache);
    
    dbQueryDuration.add(
      parseFloat(listRes.headers['X-DB-Query-Time'] || '0')
    );
    
    sleep(randomIntBetween(1, 3));
    
    // View specific product
    const product = randomItem(products);
    const detailRes = http.get(
      `http://api:3000/api/products/${product.id}`,
      { headers: authHeaders(user) }
    );
    
    check(detailRes, {
      'detail status 200': (r) => r.status === 200,
    });
    
    sleep(randomIntBetween(2, 5));
  });
  
  activeUsers.add(-1);
}

export function searchProducts() {
  const searchTerms = [
    'laptop', 'phone', 'headphones', 'keyboard', 'monitor'
  ];
  
  group('search', function() {
    const term = randomItem(searchTerms);
    const res = http.get(
      `http://api:3000/api/products/search?q=${term}&limit=20`
    );
    
    check(res, {
      'search status 200': (r) => r.status === 200,
      'search returns results': (r) => {
        const body = JSON.parse(r.body);
        return body.total > 0;
      },
    });
    
    sleep(randomIntBetween(2, 4));
  });
}

export function checkoutFlow() {
  const user = randomItem(users);
  
  group('checkout flow', function() {
    // 1. Add to cart
    const product = randomItem(products);
    const cartRes = http.post(
      'http://api:3000/api/cart/items',
      JSON.stringify({
        productId: product.id,
        quantity: randomIntBetween(1, 3),
      }),
      { headers: { ...authHeaders(user), 'Content-Type': 'application/json' } }
    );
    
    check(cartRes, {
      'add to cart 200': (r) => r.status === 200 || r.status === 201,
    });
    
    sleep(randomIntBetween(1, 2));
    
    // 2. Place order
    const orderRes = http.post(
      'http://api:3000/api/orders',
      JSON.stringify({
        shippingAddressId: user.defaultAddressId,
        paymentMethodId: 'pm_test_visa',
      }),
      { headers: { ...authHeaders(user), 'Content-Type': 'application/json' } }
    );
    
    const orderSuccess = check(orderRes, {
      'order placed 201': (r) => r.status === 201,
      'order has id': (r) => JSON.parse(r.body).orderId !== undefined,
    });
    
    if (orderSuccess) {
      orderCreated.add(1);
    }
    
    sleep(randomIntBetween(2, 5));
  });
}

// Helper
function authHeaders(user) {
  return {
    'Authorization': `Bearer ${user.token}`,
    'X-User-ID': user.id,
  };
}
```

### 2.3 Scenarios: ปรับรูปแบบ Traffic

```javascript
// scenarios-demo.js

export const options = {
  scenarios: {
    // Pattern 1: Constant VUs
    constant_load: {
      executor: 'constant-vus',
      vus: 100,
      duration: '10m',
    },
    
    // Pattern 2: Ramping VUs (load test)
    ramp_up: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '5m', target: 100 },
        { duration: '10m', target: 100 },
        { duration: '5m', target: 200 },
        { duration: '10m', target: 200 },
        { duration: '5m', target: 0 },
      ],
    },
    
    // Pattern 3: Constant Arrival Rate (RPS)
    constant_rps: {
      executor: 'constant-arrival-rate',
      rate: 1000,           // 1000 iterations/sec
      timeUnit: '1s',
      duration: '10m',
      preAllocatedVUs: 200, // VUs ที่ pre-allocate
      maxVUs: 500,          // VUs สูงสุด
    },
    
    // Pattern 4: Ramping Arrival Rate
    ramping_rps: {
      executor: 'ramping-arrival-rate',
      startRate: 100,
      timeUnit: '1s',
      stages: [
        { duration: '5m', target: 500 },
        { duration: '10m', target: 500 },
        { duration: '5m', target: 1000 },
        { duration: '10m', target: 1000 },
        { duration: '5m', target: 0 },
      ],
      preAllocatedVUs: 500,
    },
    
    // Pattern 5: Per-VU Iterations (each VU runs N times)
    per_vu: {
      executor: 'per-vu-iterations',
      vus: 50,
      iterations: 100,  // each VU runs 100 iterations
      maxDuration: '10m',
    },
  },
};
```

### 2.4 Output: InfluxDB + Grafana

```bash
# รัน k6 ด้วย InfluxDB output
k6 run \
  --out influxdb=http://influxdb:8086/k6 \
  --tag testname=load-test-2024-01-15 \
  --tag environment=staging \
  load-test.js

# หรือใช้ k6 Cloud
k6 run \
  --out cloud \
  load-test.js

# Multiple outputs
k6 run \
  --out influxdb=http://influxdb:8086/k6 \
  --out json=results.json \
  --out csv=results.csv \
  load-test.js
```

```yaml
# docker-compose สำหรับ k6 + InfluxDB + Grafana
version: '3.8'
services:
  influxdb:
    image: influxdb:1.8
    ports:
      - "8086:8086"
    environment:
      INFLUXDB_DB: k6
      INFLUXDB_HTTP_AUTH_ENABLED: "false"
    volumes:
      - influxdb_data:/var/lib/influxdb

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: Admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards

  k6:
    image: grafana/k6:latest
    command: run --out influxdb=http://influxdb:8086/k6 /scripts/load-test.js
    volumes:
      - ./k6-scripts:/scripts
      - ./test-data:/scripts/test-data
    environment:
      - K6_INFLUXDB_PUSH_INTERVAL=5s
    depends_on:
      - influxdb

volumes:
  influxdb_data:
  grafana_data:
```

---

## 3. Database-Specific Load Testing

### 3.1 pgbench: PostgreSQL Benchmark

```bash
# ติดตั้ง pgbench (มาพร้อมกับ postgresql-client)
sudo apt-get install postgresql-client

# Initialize test database
pgbench -i \
  -s 100 \          # scale factor: 100 * 100000 = 10M rows
  -U shopuser \
  -d shopcluster \
  -h localhost

# Basic benchmark: read/write mix
pgbench \
  -c 50 \          # 50 concurrent clients
  -j 4 \           # 4 worker threads
  -t 1000 \        # 1000 transactions per client
  -P 10 \          # print progress every 10 seconds
  -U shopuser \
  -d shopcluster \
  -h localhost

# Read-only benchmark
pgbench \
  -c 100 \
  -j 4 \
  -T 300 \         # 5 minutes duration
  -S \             # read-only mode
  -U shopuser \
  -d shopcluster \
  -h localhost

# Custom test script
cat > pgbench-custom.sql << 'EOF'
\set user_id random(1, 10000)
\set product_id random(1, 100000)

BEGIN;

-- Read user
SELECT id, email, username
FROM users
WHERE id = :user_id;

-- Read product
SELECT id, name, price, stock_quantity
FROM products
WHERE id = :product_id;

-- Update inventory (write)
UPDATE products
SET stock_quantity = stock_quantity - 1,
    updated_at = NOW()
WHERE id = :product_id
  AND stock_quantity > 0;

COMMIT;
EOF

pgbench \
  -c 50 -j 4 -T 300 \
  -f pgbench-custom.sql \
  -U shopuser -d shopcluster \
  -r \             # report per-command latencies
  -h localhost
```

### 3.2 redis-benchmark: Redis Benchmark

```bash
# Basic benchmark
redis-benchmark \
  -h redis \
  -p 6379 \
  -a "your-password" \
  -n 1000000 \      # 1M operations
  -c 100 \          # 100 concurrent connections
  -d 128 \          # 128 bytes per value
  --csv             # CSV output

# Specific commands
redis-benchmark \
  -h redis \
  -n 100000 \
  -c 50 \
  -t SET,GET,LPUSH,LRANGE,HSET,HGETALL

# Pipeline mode
redis-benchmark \
  -h redis \
  -n 1000000 \
  -c 100 \
  -P 16 \           # pipeline 16 commands
  -t SET,GET

# Test specific pattern: session operations
redis-benchmark \
  -h redis \
  -n 100000 \
  -c 50 \
  --dbnum 1 \       # use db 1 for sessions
  -t HSET,HGETALL,EXPIRE

# Latency test
redis-cli \
  -h redis \
  --latency \
  --latency-history \
  -i 1              # 1 second intervals
```

### 3.3 MinIO Benchmark

```bash
# Warp: MinIO benchmark tool
docker run --rm \
  -e WARP_HOST=minio:9000 \
  -e WARP_ACCESS_KEY=minioadmin \
  -e WARP_SECRET_KEY=minioadmin \
  minio/warp:latest \
  mixed \
  --duration 5m \
  --concurrent 20 \
  --obj.size 512KB \
  --bucket-prefix warp-benchmark

# S3 benchmark ด้วย s3bench
s3bench \
  -endpoint http://minio:9000 \
  -bucket benchmark \
  -prefix test/ \
  -numClients 50 \
  -numSamples 1000 \
  -objectSize 1mb \
  -accessKey minioadmin \
  -accessSecret minioadmin
```

---

## 4. Full Load Test Suite สำหรับ API + Database

### 4.1 Test Suite Structure

```
k6-tests/
├── config/
│   ├── environments.json    # URL, thresholds per env
│   └── scenarios.json       # scenario definitions
├── helpers/
│   ├── auth.js             # authentication helpers
│   ├── metrics.js          # custom metrics
│   └── data.js             # test data helpers
├── scenarios/
│   ├── browse.js           # browse products
│   ├── search.js           # search products
│   ├── cart.js             # shopping cart
│   ├── checkout.js         # checkout flow
│   └── admin.js            # admin operations
├── test-data/
│   ├── users.json          # test users
│   └── products.json       # test products
├── smoke-test.js
├── load-test.js
├── stress-test.js
├── soak-test.js
└── spike-test.js
```

### 4.2 Complete Load Test Script

```javascript
// load-test.js - Full production load test
import http from 'k6/http';
import { sleep, check, group, fail } from 'k6';
import { Counter, Rate, Trend, Gauge } from 'k6/metrics';
import { SharedArray } from 'k6/data';
import { randomIntBetween, randomItem, uuidv4 } from 'https://jslib.k6.io/k6-utils/1.4.0/index.js';

// ========== Configuration ==========
const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';
const ENV = __ENV.ENVIRONMENT || 'dev';

// ========== Custom Metrics ==========
const dbQueryTime = new Trend('db_query_time');
const cacheHit = new Rate('cache_hit');
const orderSuccess = new Counter('order_success');
const orderFailed = new Counter('order_failed');
const inventoryConflicts = new Counter('inventory_conflicts');

// ========== Test Data ==========
const users = new SharedArray('users', () => {
  return JSON.parse(open('./test-data/users.json'));
});

const products = new SharedArray('products', () => {
  return JSON.parse(open('./test-data/products.json'));
});

// ========== Options ==========
export const options = {
  scenarios: {
    browsing: {
      executor: 'ramping-vus',
      startVUs: 10,
      stages: [
        { duration: '5m', target: 200 },
        { duration: '15m', target: 200 },
        { duration: '3m', target: 0 },
      ],
      exec: 'browsing',
      tags: { scenario: 'browsing' },
    },
    shopping: {
      executor: 'ramping-arrival-rate',
      startRate: 10,
      timeUnit: '1s',
      stages: [
        { duration: '5m', target: 50 },
        { duration: '15m', target: 50 },
        { duration: '3m', target: 0 },
      ],
      preAllocatedVUs: 100,
      maxVUs: 300,
      exec: 'shopping',
      tags: { scenario: 'shopping' },
    },
  },

  thresholds: {
    'http_req_duration': ['p(50)<50', 'p(95)<150', 'p(99)<200'],
    'http_req_duration{scenario:shopping}': ['p(99)<500'],
    'http_req_failed': ['rate<0.001'],
    'db_query_time': ['p(95)<50', 'p(99)<100'],
    'cache_hit': ['rate>0.75'],
  },
};

// ========== Browsing Scenario ==========
export function browsing() {
  const user = randomItem(users);
  const token = login(user);
  if (!token) return;

  group('browse catalog', () => {
    // List products
    const listRes = http.get(
      `${BASE_URL}/api/products?page=${randomIntBetween(1, 10)}&limit=20&sort=created_at`,
      { headers: authHeaders(token), tags: { operation: 'list_products' } }
    );
    
    recordMetrics(listRes, 'list_products');
    
    check(listRes, {
      'list: status 200': (r) => r.status === 200,
      'list: has data': (r) => {
        try {
          return JSON.parse(r.body).data.length > 0;
        } catch { return false; }
      },
    }) || fail('List products failed');

    sleep(randomIntBetween(1, 3));

    // View product detail
    const product = randomItem(products);
    const detailRes = http.get(
      `${BASE_URL}/api/products/${product.id}`,
      { headers: authHeaders(token), tags: { operation: 'get_product' } }
    );
    
    recordMetrics(detailRes, 'get_product');
    
    check(detailRes, {
      'detail: status 200': (r) => r.status === 200,
      'detail: has price': (r) => {
        try { return JSON.parse(r.body).price > 0; }
        catch { return false; }
      },
    });

    sleep(randomIntBetween(2, 5));

    // Search
    const terms = ['laptop', 'phone', 'headphones', 'tablet', 'camera'];
    const searchRes = http.get(
      `${BASE_URL}/api/products/search?q=${randomItem(terms)}`,
      { headers: authHeaders(token), tags: { operation: 'search' } }
    );
    
    recordMetrics(searchRes, 'search');
    
    check(searchRes, {
      'search: status 200': (r) => r.status === 200,
    });

    sleep(randomIntBetween(1, 3));
  });
}

// ========== Shopping Scenario ==========
export function shopping() {
  const user = randomItem(users);
  const token = login(user);
  if (!token) return;

  group('shopping flow', () => {
    // View cart
    const cartRes = http.get(
      `${BASE_URL}/api/cart`,
      { headers: authHeaders(token) }
    );
    
    check(cartRes, {
      'cart: status 200': (r) => r.status === 200,
    });

    sleep(1);

    // Add item to cart
    const product = randomItem(products);
    const addRes = http.post(
      `${BASE_URL}/api/cart/items`,
      JSON.stringify({ productId: product.id, quantity: 1 }),
      { headers: { ...authHeaders(token), 'Content-Type': 'application/json' } }
    );
    
    check(addRes, {
      'add to cart: 200/201': (r) => r.status === 200 || r.status === 201,
    });

    sleep(randomIntBetween(2, 5));

    // Place order
    const orderRes = http.post(
      `${BASE_URL}/api/orders`,
      JSON.stringify({
        shippingAddressId: user.defaultAddressId,
        paymentToken: 'tok_visa_test',
      }),
      { headers: { ...authHeaders(token), 'Content-Type': 'application/json' } }
    );
    
    recordMetrics(orderRes, 'create_order');
    
    const orderOk = check(orderRes, {
      'order: status 201': (r) => r.status === 201,
      'order: has orderId': (r) => {
        try { return !!JSON.parse(r.body).orderId; }
        catch { return false; }
      },
    });
    
    if (orderOk) {
      orderSuccess.add(1);
    } else {
      orderFailed.add(1);
      // Check for inventory conflict
      if (orderRes.status === 409) {
        inventoryConflicts.add(1);
      }
    }

    sleep(randomIntBetween(1, 3));
  });
}

// ========== Helpers ==========
function login(user) {
  const res = http.post(
    `${BASE_URL}/api/auth/login`,
    JSON.stringify({ email: user.email, password: user.password }),
    { headers: { 'Content-Type': 'application/json' }, tags: { operation: 'login' } }
  );
  
  if (res.status !== 200) return null;
  return JSON.parse(res.body).accessToken;
}

function authHeaders(token) {
  return {
    'Authorization': `Bearer ${token}`,
    'Accept': 'application/json',
  };
}

function recordMetrics(res, operation) {
  // DB query time from header
  const dbTime = parseFloat(res.headers['X-DB-Query-Time'] || '0');
  if (dbTime > 0) dbQueryTime.add(dbTime);
  
  // Cache hit
  const fromCache = res.headers['X-Cache'] === 'HIT';
  cacheHit.add(fromCache);
}
```

### 4.3 Generate Test Data Script

```javascript
// scripts/generate-test-data.js
const { Pool } = require('pg');
const { faker } = require('@faker-js/faker');
const fs = require('fs');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL
});

async function generateTestData() {
  console.log('Generating test data...');
  
  // Create test users
  const users = [];
  for (let i = 0; i < 1000; i++) {
    const user = {
      id: faker.string.uuid(),
      email: faker.internet.email(),
      username: faker.internet.userName(),
      password: 'TestPassword123!',
      defaultAddressId: faker.string.uuid(),
    };
    users.push(user);
  }
  
  // Insert users to DB
  const client = await pool.connect();
  try {
    for (const user of users) {
      await client.query(
        `INSERT INTO users (id, email, username, password_hash, created_at)
         VALUES ($1, $2, $3, crypt($4, gen_salt('bf')), NOW())
         ON CONFLICT DO NOTHING`,
        [user.id, user.email, user.username, user.password]
      );
    }
    
    // Get tokens for each user (via API)
    const usersWithTokens = await Promise.all(
      users.slice(0, 200).map(async (user) => {
        try {
          const response = await fetch('http://localhost:3000/api/auth/login', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ email: user.email, password: user.password }),
          });
          const data = await response.json();
          return { ...user, token: data.accessToken };
        } catch {
          return null;
        }
      })
    );
    
    const validUsers = usersWithTokens.filter(Boolean);
    fs.writeFileSync(
      './test-data/users.json',
      JSON.stringify(validUsers, null, 2)
    );
    
    console.log(`Generated ${validUsers.length} test users`);
    
    // Get products
    const { rows: products } = await client.query(
      'SELECT id, name, price FROM products LIMIT 1000'
    );
    
    fs.writeFileSync(
      './test-data/products.json',
      JSON.stringify(products, null, 2)
    );
    
    console.log(`Found ${products.length} products`);
    
  } finally {
    client.release();
    await pool.end();
  }
}

generateTestData().catch(console.error);
```

---

## 5. Monitoring During Load Test

### 5.1 PostgreSQL Metrics during Load Test

```sql
-- เปิด terminal ใหม่ขณะ load test กำลังรัน

-- 1. ดู active connections และ wait events
SELECT 
  state,
  wait_event_type,
  wait_event,
  COUNT(*) as count,
  MAX(EXTRACT(EPOCH FROM (NOW() - query_start))) as max_duration_sec
FROM pg_stat_activity
WHERE state != 'idle'
GROUP BY state, wait_event_type, wait_event
ORDER BY count DESC;

-- 2. ดู slowest queries ตอนนี้
SELECT
  now() - pg_stat_activity.query_start AS duration,
  query,
  state,
  wait_event_type,
  wait_event
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '1 seconds'
  AND state != 'idle'
ORDER BY duration DESC
LIMIT 20;

-- 3. ดู lock waits
SELECT
  blocked.pid AS blocked_pid,
  blocking.pid AS blocking_pid,
  blocked.query AS blocked_query,
  blocking.query AS blocking_query,
  blocked.wait_event_type,
  blocked.wait_event
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE NOT blocked.granted;

-- 4. ดู connection pool usage
SELECT
  count(*) AS total,
  count(*) FILTER (WHERE state = 'active') AS active,
  count(*) FILTER (WHERE state = 'idle') AS idle,
  count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_txn,
  max(now() - query_start) AS longest_query
FROM pg_stat_activity
WHERE backend_type = 'client backend';

-- 5. ดู table-level statistics (cache hit ratio)
SELECT
  relname AS table,
  heap_blks_read,
  heap_blks_hit,
  ROUND(100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0), 2) AS cache_hit_ratio,
  seq_scan,
  idx_scan,
  n_tup_ins + n_tup_upd + n_tup_del AS write_ops
FROM pg_statio_user_tables
WHERE heap_blks_hit + heap_blks_read > 0
ORDER BY heap_blks_hit + heap_blks_read DESC
LIMIT 20;

-- 6. Top queries by total time
SELECT
  round(total_exec_time::numeric, 2) AS total_exec_ms,
  calls,
  round(mean_exec_time::numeric, 2) AS mean_ms,
  round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS percentage,
  left(query, 100) AS query_preview
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

### 5.2 Redis Metrics during Load Test

```bash
# Monitor Redis ขณะ load test
watch -n 1 redis-cli -h redis -a password info stats | grep -E 'instantaneous|keyspace|hits|misses'

# ดู slowlog
redis-cli -h redis -a password SLOWLOG GET 10

# Monitor ทุก commands ใน real-time (ระวัง: ช้าลงถ้าใช้ production)
redis-cli -h redis -a password MONITOR | head -1000

# Key statistics
redis-cli -h redis -a password INFO all | grep -E '
  used_memory_human|
  mem_fragmentation_ratio|
  connected_clients|
  instantaneous_ops_per_sec|
  keyspace_hits|
  keyspace_misses|
  expired_keys|
  evicted_keys
'
```

### 5.3 System Metrics (Prometheus Queries ระหว่าง Load Test)

```promql
# CPU Usage
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)

# Memory Usage
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100

# Disk I/O
rate(node_disk_read_bytes_total[1m])
rate(node_disk_written_bytes_total[1m])

# Network I/O
rate(node_network_receive_bytes_total{device!~"lo|veth.*"}[1m])
rate(node_network_transmit_bytes_total{device!~"lo|veth.*"}[1m])

# PostgreSQL transactions per second
rate(pg_stat_database_xact_commit_total[1m]) + rate(pg_stat_database_xact_rollback_total[1m])

# Redis operations per second
rate(redis_commands_processed_total[1m])

# Connection pool usage
pgbouncer_pool_client_active_connections / pgbouncer_pool_server_active_connections
```

---

## 6. Analyzing Results

### 6.1 k6 Results Analysis

```bash
# รัน test และ save output
k6 run --out json=results.json load-test.js

# Analyze ด้วย Python
python3 << 'EOF'
import json
import statistics

with open('results.json') as f:
    data = [json.loads(line) for line in f if line.strip()]

# Filter HTTP request metrics
req_durations = [
    d['metric']['value'] 
    for d in data 
    if d.get('type') == 'Point' 
    and d.get('metric') == 'http_req_duration'
]

if req_durations:
    print("=== HTTP Request Duration ===")
    print(f"Count: {len(req_durations):,}")
    print(f"Min: {min(req_durations):.2f}ms")
    print(f"Max: {max(req_durations):.2f}ms")
    print(f"Mean: {statistics.mean(req_durations):.2f}ms")
    print(f"Median (P50): {statistics.median(req_durations):.2f}ms")
    
    sorted_durations = sorted(req_durations)
    p95_idx = int(len(sorted_durations) * 0.95)
    p99_idx = int(len(sorted_durations) * 0.99)
    
    print(f"P95: {sorted_durations[p95_idx]:.2f}ms")
    print(f"P99: {sorted_durations[p99_idx]:.2f}ms")
    
    # SLO check
    p99 = sorted_durations[p99_idx]
    print(f"\nSLO Check (P99 < 200ms): {'✅ PASS' if p99 < 200 else '❌ FAIL'} ({p99:.2f}ms)")

EOF
```

### 6.2 Finding Bottlenecks

```bash
# Script: analyze-bottleneck.sh
#!/bin/bash

echo "=== Bottleneck Analysis ==="

# 1. Connection Pool Exhaustion
echo ""
echo "1. Connection Pool"
kubectl exec -n prod-database \
  $(kubectl get pods -n prod-database -l app=pgbouncer -o name | head -1) \
  -- psql -p 6432 pgbouncer -c "SHOW POOLS;" -t

# 2. Slow Queries
echo ""
echo "2. Top Slow Queries"
kubectl exec -n prod-database \
  $(kubectl get pods -n prod-database -l app=postgresql,role=primary -o name | head -1) \
  -- psql -U postgres shopcluster -c "
    SELECT
      calls,
      round(mean_exec_time::numeric, 2) as mean_ms,
      round(total_exec_time::numeric, 2) as total_ms,
      left(query, 80) as query
    FROM pg_stat_statements
    ORDER BY mean_exec_time DESC
    LIMIT 5;
  "

# 3. Lock Contention
echo ""
echo "3. Lock Waits"
kubectl exec -n prod-database \
  $(kubectl get pods -n prod-database -l app=postgresql,role=primary -o name | head -1) \
  -- psql -U postgres shopcluster -c "
    SELECT
      pg_blocking_pids(pid) as blocked_by,
      query as blocked_query
    FROM pg_stat_activity
    WHERE cardinality(pg_blocking_pids(pid)) > 0
    LIMIT 10;
  "

# 4. I/O Stats
echo ""
echo "4. I/O Saturation"
kubectl exec -n prod-database \
  $(kubectl get pods -n prod-database -l app=postgresql,role=primary -o name | head -1) \
  -- iostat -x 1 5 | tail -10

# 5. Missing Indexes
echo ""
echo "5. Sequential Scans (possible missing indexes)"
kubectl exec -n prod-database \
  $(kubectl get pods -n prod-database -l app=postgresql,role=primary -o name | head -1) \
  -- psql -U postgres shopcluster -c "
    SELECT
      relname AS table,
      seq_scan,
      idx_scan,
      round(100.0 * seq_scan / NULLIF(seq_scan + idx_scan, 0), 1) AS seq_pct
    FROM pg_stat_user_tables
    WHERE seq_scan > 100
    ORDER BY seq_scan DESC
    LIMIT 10;
  "
```

---

## 7. Performance Tuning จาก Load Test Results

### 7.1 Connection Pool Tuning

```javascript
// config/database.js - tuned connection pool
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  
  // หลัง load test: จาก pgbench results
  max: 20,          // max connections per app instance
                    // 3 instances × 20 = 60 max connections
                    // ต้องน้อยกว่า postgres max_connections (200) × 0.8
  
  min: 2,           // keep 2 connections warm
  idleTimeoutMillis: 10000,    // close idle after 10s
  connectionTimeoutMillis: 3000, // timeout ถ้า get connection ไม่ได้ใน 3s
  
  // Statement timeout ป้องกัน long-running queries
  statement_timeout: 30000,   // 30s
  
  // Application name สำหรับ monitoring
  application_name: 'shopcluster-api',
});

// Monitor pool health
pool.on('connect', () => {
  console.debug('New DB connection created');
});

pool.on('error', (err) => {
  console.error('Unexpected DB pool error:', err);
});

// Expose pool metrics
setInterval(() => {
  console.info('DB Pool stats:', {
    total: pool.totalCount,
    idle: pool.idleCount,
    waiting: pool.waitingCount,
  });
  
  // Prometheus metrics
  dbPoolTotal.set(pool.totalCount);
  dbPoolIdle.set(pool.idleCount);
  dbPoolWaiting.set(pool.waitingCount);
}, 10000);
```

### 7.2 Query Optimization จาก Load Test

```sql
-- หลัง load test พบ slow query นี้:
-- SELECT * FROM products WHERE category_id = $1 ORDER BY created_at DESC
-- ใช้เวลา 350ms เพราะ seq scan

-- สร้าง composite index
CREATE INDEX CONCURRENTLY idx_products_category_created
ON products(category_id, created_at DESC)
WHERE deleted_at IS NULL;  -- partial index

-- ก่อน: EXPLAIN ANALYZE
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT id, name, price
FROM products
WHERE category_id = 'cat-electronics'
ORDER BY created_at DESC
LIMIT 20;

-- หลัง index: 350ms → 2ms
```

---

## สรุป Load Testing Best Practices

| Test Type | เมื่อทำ | Duration | VUs |
|-----------|---------|----------|-----|
| Smoke | ทุก PR | 1m | 1-5 |
| Load | ก่อน release | 30m | Expected |
| Stress | ก่อน major feature | 1h | 2-3x expected |
| Soak | ก่อน go-live | 8h+ | Normal |
| Spike | ก่อน marketing campaign | 30m | 10-50x normal |

**กฎหลัก:**
1. Load test ใน environment ที่ใกล้เคียง production มากที่สุด
2. ใช้ data ที่ realistic (ไม่ใช่ fake empty DB)
3. Monitor database ระหว่าง test ด้วย
4. Fix bottlenecks ก่อน go-live ไม่ใช่หลัง
5. เก็บ baseline เพื่อ compare regression
