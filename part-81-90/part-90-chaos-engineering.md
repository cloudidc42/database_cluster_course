# Part 90: Chaos Engineering

## บทนำ

Chaos Engineering คือวินัยในการทดสอบระบบในสภาวะปกติด้วยการ inject ความผิดพลาดอย่างควบคุม เพื่อค้นหาจุดอ่อนก่อนที่จะเกิด incident จริง หลักการนี้มาจาก Netflix ที่ต้องการให้ระบบ resilient ต่อ failures ในระดับ cloud infrastructure

---

## 1. Principles of Chaos Engineering (Netflix)

### 5 หลักการสำคัญ

```
Principle 1: Build Hypothesis Around Steady State
─────────────────────────────────────────────────
กำหนด "สถานะปกติ" ของระบบก่อน เช่น:
- Error rate < 0.1%
- P99 latency < 200ms
- Throughput > 1000 rps
- Cache hit rate > 90%

แล้วตั้ง hypothesis ว่า: "เมื่อ database failover เกิดขึ้น
ระบบจะยังคงสถานะปกตินี้ได้"

Principle 2: Vary Real-World Events
─────────────────────────────────────────────────
ทดสอบด้วยเหตุการณ์ที่เกิดขึ้นจริง:
- Database crash
- Network packet loss
- High CPU/Memory
- Disk full
- Slow queries (latency injection)
- Replica lag
- Cache eviction storm

Principle 3: Run Experiments in Production
─────────────────────────────────────────────────
Staging environment ไม่เหมือน Production
- Traffic patterns ต่างกัน
- Load ต่างกัน
- Data size ต่างกัน

เริ่มจาก staging → ค่อยๆ ทำบน production
ด้วย blast radius ที่เล็กที่สุด

Principle 4: Automate Continuously
─────────────────────────────────────────────────
ไม่ใช่แค่ทำครั้งเดียวแล้วจบ
- Integrate กับ CI/CD
- Run chaos experiments อัตโนมัติ
- Alert เมื่อ system ไม่ resilient อีกต่อไป

Principle 5: Minimize Blast Radius
─────────────────────────────────────────────────
เริ่มจากเล็ก ค่อยๆ ขยาย:
- 1% ของ traffic ก่อน → 10% → 100%
- 1 region ก่อน → หลาย region
- Non-critical service ก่อน → critical service
```

---

## 2. Common Failure Scenarios

### Failure Taxonomy

```python
# failure_scenarios.py
FAILURE_SCENARIOS = {
    'database': {
        'primary_crash': {
            'description': 'PostgreSQL primary process dies',
            'inject': 'kill -9 $(pidof postgres)',
            'expected_behavior': 'Replica promotes within 60s, app reconnects',
            'metrics_to_watch': ['error_rate', 'latency_p99', 'active_connections'],
            'blast_radius': 'all database operations'
        },
        'replica_crash': {
            'description': 'One replica dies',
            'inject': 'systemctl stop postgresql on replica-1',
            'expected_behavior': 'App continues using primary and other replica',
            'metrics_to_watch': ['read_query_latency', 'connection_pool_usage'],
            'blast_radius': 'read operations on that replica'
        },
        'slow_queries': {
            'description': 'Inject 500ms latency to all queries',
            'inject': 'pg_sleep injection or tc qdisc',
            'expected_behavior': 'Circuit breaker trips, return cached data',
            'metrics_to_watch': ['latency_p99', 'timeout_rate', 'circuit_breaker_state'],
            'blast_radius': 'all database operations'
        },
        'disk_full': {
            'description': 'Fill database disk to 95%',
            'inject': 'dd if=/dev/zero of=/var/lib/postgresql/bigfile bs=1G',
            'expected_behavior': 'Alert fires, read-only mode activated',
            'metrics_to_watch': ['disk_usage', 'write_error_rate'],
            'blast_radius': 'write operations'
        },
        'max_connections': {
            'description': 'Exhaust connection pool',
            'inject': 'Open many idle connections',
            'expected_behavior': 'New connections queue or fail gracefully',
            'metrics_to_watch': ['connection_pool_exhaustion', 'query_errors'],
            'blast_radius': 'new connection attempts'
        }
    },
    
    'redis': {
        'master_crash': {
            'description': 'Redis master process dies',
            'inject': 'docker kill redis-master',
            'expected_behavior': 'Sentinel promotes replica within 30s',
            'metrics_to_watch': ['cache_hit_rate', 'redis_errors', 'latency'],
            'blast_radius': 'all cache operations'
        },
        'slow_responses': {
            'description': 'Redis responds slowly (50ms per command)',
            'inject': "CONFIG SET slowlog-log-slower-than 0; use tc netem delay",
            'expected_behavior': 'App uses database fallback for slow items',
            'metrics_to_watch': ['cache_latency', 'db_load'],
            'blast_radius': 'cache-dependent operations'
        },
        'memory_full': {
            'description': 'Redis reaches maxmemory',
            'inject': 'Fill Redis with large values',
            'expected_behavior': 'Eviction policy triggers, no crash',
            'metrics_to_watch': ['eviction_rate', 'cache_hit_rate', 'memory_usage'],
            'blast_radius': 'cached data (evicted items miss)'
        }
    },
    
    'network': {
        'packet_loss': {
            'description': '10% packet loss between app and database',
            'inject': 'tc qdisc add dev eth0 root netem loss 10%',
            'expected_behavior': 'Retries handle loss, latency increases',
            'metrics_to_watch': ['latency_p99', 'retry_rate', 'error_rate'],
            'blast_radius': 'all network traffic'
        },
        'latency_injection': {
            'description': 'Add 100ms latency to database connections',
            'inject': 'tc qdisc add dev eth0 root netem delay 100ms 20ms',
            'expected_behavior': 'Timeouts set appropriately, circuit breaker may trip',
            'metrics_to_watch': ['latency_p99', 'timeout_rate'],
            'blast_radius': 'all database/cache operations'
        }
    }
}
```

---

## 3. Tools

### LitmusChaos: Kubernetes-Native

```yaml
# litmus_chaos_experiment.yml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: postgres-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: app=postgres-primary
    appkind: deployment
  
  chaosServiceAccount: litmus-admin
  
  experiments:
    # ทดสอบ pod crash
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"  # วินาที
            - name: CHAOS_INTERVAL
              value: "10"
            - name: FORCE
              value: "false"
        
    # ทดสอบ network latency
    - name: pod-network-latency
      spec:
        components:
          env:
            - name: NETWORK_INTERFACE
              value: eth0
            - name: TARGET_CONTAINER
              value: postgres
            - name: NETWORK_LATENCY
              value: "100"  # ms
            - name: TOTAL_CHAOS_DURATION
              value: "120"
            - name: JITTER
              value: "20"
            
    # ทดสอบ CPU stress
    - name: pod-cpu-hog
      spec:
        components:
          env:
            - name: CPU_CORES
              value: "2"
            - name: TOTAL_CHAOS_DURATION
              value: "60"

  # Steady state hypothesis
  jobCleanUpPolicy: delete
```

### Chaos Mesh: CNCF Project

```yaml
# chaos_mesh_experiments.yml

# 1. Network chaos: เพิ่ม latency
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: db-network-delay
  namespace: production
spec:
  action: delay
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      app: postgres
  delay:
    latency: "100ms"
    correlation: "25"
    jitter: "10ms"
  duration: "5m"
  scheduler:
    cron: "@every 1h"  # ทำทุกชั่วโมง

---
# 2. Pod chaos: restart postgres replica
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: postgres-replica-kill
  namespace: production
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      app: postgres
      role: replica
  scheduler:
    cron: "0 2 * * 1"  # ทุกจันทร์ 02:00

---
# 3. Stress chaos: CPU/Memory stress
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: redis-cpu-stress
  namespace: production
spec:
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      app: redis
  stressors:
    cpu:
      workers: 4
      load: 80  # 80% CPU usage
  duration: "2m"
```

### Pumba: Docker Container Chaos

```bash
#!/bin/bash
# pumba_chaos.sh - Docker container chaos

# ติดตั้ง Pumba
docker pull gaiaadm/pumba

# Kill random container ที่มี prefix "redis"
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    gaiaadm/pumba kill \
    --interval 30s \
    --signal SIGTERM \
    re2:^redis.*

# เพิ่ม network latency ให้ postgres container
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    --cap-add=NET_ADMIN \
    gaiaadm/pumba netem \
    --duration 5m \
    --tc-image gaiaadm/tc-netem:latest \
    delay \
    --time 100 \
    --jitter 20 \
    postgres-primary

# Pause container ชั่วคราว (simulate hung process)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    gaiaadm/pumba pause \
    --duration 30s \
    redis-master
```

---

## 4. Database Chaos Experiments

### Kill PostgreSQL Primary: Verify Failover

```bash
#!/bin/bash
# chaos_postgres_primary_kill.sh

POSTGRES_HOST="postgres-primary"
METRICS_URL="http://prometheus:9090"
APP_URL="http://api:3000"
SLACK_WEBHOOK="${SLACK_WEBHOOK_URL}"

# Helper: check error rate
get_error_rate() {
    curl -s "${METRICS_URL}/api/v1/query?query=rate(http_requests_total{status=~'5..'}[1m])" \
        | python3 -c "import json,sys; data=json.load(sys.stdin); \
                      print(data['data']['result'][0]['value'][1] if data['data']['result'] else '0')"
}

# Helper: check latency
get_p99_latency() {
    curl -s "${METRICS_URL}/api/v1/query?query=histogram_quantile(0.99,rate(http_request_duration_seconds_bucket[1m]))" \
        | python3 -c "import json,sys; data=json.load(sys.stdin); \
                      print(data['data']['result'][0]['value'][1] if data['data']['result'] else '0')"
}

notify_slack() {
    curl -s -X POST "${SLACK_WEBHOOK}" \
        -H 'Content-type: application/json' \
        --data "{\"text\": \"$1\"}"
}

echo "=== Chaos Experiment: PostgreSQL Primary Kill ==="
echo "Time: $(date)"

# 1. Measure steady state
BASELINE_ERROR_RATE=$(get_error_rate)
BASELINE_LATENCY=$(get_p99_latency)
echo "Baseline - Error Rate: ${BASELINE_ERROR_RATE}, P99 Latency: ${BASELINE_LATENCY}s"

# 2. Start monitoring
notify_slack ":warning: Starting chaos experiment: PostgreSQL Primary Kill"

# 3. Kill primary
echo "Killing PostgreSQL primary..."
EXPERIMENT_START=$(date +%s)
ssh postgres@${POSTGRES_HOST} "sudo systemctl stop postgresql"

# 4. Monitor recovery
FAILOVER_DETECTED=false
for i in $(seq 1 30); do
    sleep 2
    
    # Check if new primary is available
    NEW_PRIMARY_RESPONSE=$(psql -h postgres-vip -U postgres -c "SELECT pg_is_in_recovery();" -t 2>/dev/null | tr -d ' ')
    
    if [ "${NEW_PRIMARY_RESPONSE}" = "f" ]; then
        FAILOVER_TIME=$(( $(date +%s) - EXPERIMENT_START ))
        echo "Failover detected at ${FAILOVER_TIME}s"
        FAILOVER_DETECTED=true
        break
    fi
    
    echo "Still waiting for failover... (${i}s)"
done

if [ "${FAILOVER_DETECTED}" = "false" ]; then
    echo "ERROR: Failover did not complete within 60s!"
    notify_slack ":alert: CHAOS EXPERIMENT FAILED: Failover took too long!"
    exit 1
fi

# 5. Wait for stabilization
echo "Waiting for system to stabilize..."
sleep 30

# 6. Measure post-chaos metrics
POST_ERROR_RATE=$(get_error_rate)
POST_LATENCY=$(get_p99_latency)
echo "Post-chaos - Error Rate: ${POST_ERROR_RATE}, P99 Latency: ${POST_LATENCY}s"

# 7. Determine success
ERROR_RATE_DIFF=$(echo "${POST_ERROR_RATE} - ${BASELINE_ERROR_RATE}" | bc)
echo "Error rate increase: ${ERROR_RATE_DIFF}"

if (( $(echo "${ERROR_RATE_DIFF} > 0.05" | bc -l) )); then
    echo "FAILED: Error rate increased by more than 5%"
    notify_slack ":x: Chaos FAILED: Error rate spike detected (${ERROR_RATE_DIFF})"
    exit 1
else
    echo "SUCCESS: System recovered within acceptable bounds"
    notify_slack ":white_check_mark: Chaos PASSED: Failover in ${FAILOVER_TIME}s, error rate OK"
fi

echo "=== Experiment Complete ==="
```

### Network Delay Simulation

```bash
#!/bin/bash
# chaos_network_delay.sh - เพิ่ม latency ระหว่าง app และ database

TARGET_INTERFACE="eth0"
DELAY_MS=200
JITTER_MS=50
DURATION_SECONDS=300  # 5 นาที

# เพิ่ม network delay
echo "Adding ${DELAY_MS}ms delay (±${JITTER_MS}ms) to ${TARGET_INTERFACE}..."
tc qdisc add dev ${TARGET_INTERFACE} root netem \
    delay ${DELAY_MS}ms ${JITTER_MS}ms \
    distribution normal

# Monitor impact
echo "Monitoring for ${DURATION_SECONDS}s..."
MONITOR_START=$(date +%s)

while [ $(( $(date +%s) - MONITOR_START )) -lt ${DURATION_SECONDS} ]; do
    # Check application health
    HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://api:3000/health)
    RESPONSE_TIME=$(curl -s -o /dev/null -w "%{time_total}" http://api:3000/health)
    
    echo "$(date): HTTP=${HTTP_STATUS}, Response=${RESPONSE_TIME}s"
    sleep 5
done

# Remove network delay
echo "Removing network delay..."
tc qdisc del dev ${TARGET_INTERFACE} root

echo "Chaos experiment complete"
```

### Max Connections Exceeded

```python
# chaos_max_connections.py
import psycopg2
import threading
import time

def exhaust_connections(
    conn_string: str,
    target_count: int = 100,
    hold_seconds: int = 60
):
    """
    เปิด connections จนถึง max_connections
    เพื่อดูว่า application handle ได้อย่างไร
    """
    
    print(f"Opening {target_count} idle connections for {hold_seconds}s")
    connections = []
    
    try:
        # เปิด connections
        for i in range(target_count):
            try:
                conn = psycopg2.connect(conn_string + " connect_timeout=5")
                connections.append(conn)
                
                if i % 10 == 0:
                    print(f"Opened {i+1} connections")
                    
            except psycopg2.OperationalError as e:
                print(f"Failed to open connection {i+1}: {e}")
                print(f"Max connections reached at {len(connections)}")
                break
        
        print(f"Holding {len(connections)} connections for {hold_seconds}s")
        
        # ทดสอบว่า application ยังทำงานได้ไหม
        import requests
        for i in range(hold_seconds // 5):
            try:
                resp = requests.get("http://api:3000/health", timeout=2)
                print(f"App health: {resp.status_code} ({resp.elapsed.total_seconds():.3f}s)")
            except Exception as e:
                print(f"App health check FAILED: {e}")
            time.sleep(5)
    
    finally:
        # ปิด connections ทั้งหมด
        print("Closing all chaos connections...")
        for conn in connections:
            try:
                conn.close()
            except:
                pass
        print("Cleanup complete")

# รัน experiment
exhaust_connections(
    conn_string="host=postgres-primary dbname=production user=chaos_user password=chaos_pass",
    target_count=100,
    hold_seconds=120
)
```

---

## 5. Redis Chaos Experiments

### Kill Redis Master

```bash
#!/bin/bash
# chaos_redis_master_kill.sh

REDIS_SENTINEL="redis-sentinel-1"
APP_URL="http://api:3000"

# Measure steady state
echo "=== Redis Master Kill Experiment ==="

CACHE_HIT_BEFORE=$(redis-cli -h redis-master info stats | grep keyspace_hits | awk -F: '{print $2}')
echo "Cache hits before: ${CACHE_HIT_BEFORE}"

# Kill master
echo "Killing Redis master..."
EXPERIMENT_START=$(date +%s)
docker kill redis-master

# Wait for Sentinel failover
echo "Waiting for Sentinel to detect failure..."
for i in $(seq 1 30); do
    sleep 2
    
    # Query sentinel for current master
    CURRENT_MASTER=$(redis-cli -h ${REDIS_SENTINEL} -p 26379 SENTINEL get-master-addr-by-name mymaster | head -1)
    
    if [ ! -z "${CURRENT_MASTER}" ] && [ "${CURRENT_MASTER}" != "redis-master" ]; then
        FAILOVER_TIME=$(( $(date +%s) - EXPERIMENT_START ))
        echo "New master: ${CURRENT_MASTER} (failover in ${FAILOVER_TIME}s)"
        break
    fi
    
    echo "Waiting... (${i})"
done

# Verify app still works
echo "Testing application..."
for i in $(seq 1 10); do
    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" ${APP_URL}/api/products)
    echo "Request ${i}: HTTP ${HTTP_CODE}"
    sleep 1
done

# Cleanup: restart old master as replica
echo "Restarting old master as replica..."
docker start redis-master
# replication will be configured by Sentinel

echo "=== Experiment Complete ==="
```

### Redis Memory Full (Eviction Test)

```python
# chaos_redis_memory.py
import redis
import time
import random
import string

def fill_redis_memory(redis_host: str = "redis-master", redis_port: int = 6379):
    """
    เติม Redis จนเกือบเต็ม memory เพื่อทดสอบ eviction
    """
    
    r = redis.Redis(host=redis_host, port=redis_port, decode_responses=True)
    
    # ตรวจสอบ maxmemory
    maxmemory = int(r.config_get('maxmemory')['maxmemory'])
    eviction_policy = r.config_get('maxmemory-policy')['maxmemory-policy']
    
    print(f"Max memory: {maxmemory / 1024 / 1024:.0f}MB")
    print(f"Eviction policy: {eviction_policy}")
    
    # Fill memory ด้วย random data
    key_count = 0
    
    try:
        while True:
            # สร้าง key ขนาด 1KB
            key = f"chaos:test:{key_count}"
            value = ''.join(random.choices(string.ascii_letters, k=1024))
            
            r.set(key, value, ex=3600)
            key_count += 1
            
            if key_count % 1000 == 0:
                info = r.info('memory')
                used_memory = info['used_memory']
                
                print(f"Keys: {key_count}, Memory: {used_memory / 1024 / 1024:.1f}MB")
                
                if maxmemory > 0 and used_memory > maxmemory * 0.95:
                    print(f"Reached 95% of maxmemory!")
                    break
                    
    except redis.exceptions.ResponseError as e:
        print(f"Redis error (expected if OOM policy = noeviction): {e}")
    
    # ทดสอบว่า SET ยังทำงานได้ไหม
    print("\nTesting SET operations under memory pressure...")
    evictions_before = int(r.info('stats').get('evicted_keys', 0))
    
    for i in range(100):
        try:
            r.set(f"test:after:eviction:{i}", "value")
        except Exception as e:
            print(f"SET failed: {e}")
    
    evictions_after = int(r.info('stats').get('evicted_keys', 0))
    new_evictions = evictions_after - evictions_before
    
    print(f"Evictions during test: {new_evictions}")
    print(f"Eviction policy: {eviction_policy}")
    
    if eviction_policy == 'noeviction' and new_evictions == 0:
        print("WARNING: Writes are failing due to noeviction policy!")
    else:
        print(f"OK: {new_evictions} keys evicted to make room")

fill_redis_memory()
```

---

## 6. Application Resilience Patterns

### Circuit Breaker Implementation

```typescript
// src/utils/circuitBreaker.ts

enum CircuitState {
    CLOSED = 'CLOSED',       // ปกติ ส่ง requests
    OPEN = 'OPEN',           // ตัด requests ทั้งหมด
    HALF_OPEN = 'HALF_OPEN'  // ทดสอบว่าปกติหรือยัง
}

interface CircuitBreakerOptions {
    failureThreshold: number;    // จำนวน failures ก่อน OPEN
    successThreshold: number;    // จำนวน successes ก่อน CLOSED
    timeout: number;             // ms ก่อนเปลี่ยนจาก OPEN → HALF_OPEN
    requestTimeout: number;      // ms timeout per request
}

export class CircuitBreaker {
    private state: CircuitState = CircuitState.CLOSED;
    private failures: number = 0;
    private successes: number = 0;
    private lastFailureTime: number = 0;
    private options: CircuitBreakerOptions;
    private name: string;
    
    constructor(name: string, options: Partial<CircuitBreakerOptions> = {}) {
        this.name = name;
        this.options = {
            failureThreshold: options.failureThreshold ?? 5,
            successThreshold: options.successThreshold ?? 2,
            timeout: options.timeout ?? 60000,  // 60s
            requestTimeout: options.requestTimeout ?? 5000  // 5s
        };
    }
    
    async execute<T>(fn: () => Promise<T>, fallback?: () => T): Promise<T> {
        if (this.state === CircuitState.OPEN) {
            // ตรวจสอบว่าถึงเวลา HALF_OPEN หรือยัง
            const timeSinceLastFailure = Date.now() - this.lastFailureTime;
            
            if (timeSinceLastFailure >= this.options.timeout) {
                console.log(`[${this.name}] Circuit HALF_OPEN: Testing...`);
                this.state = CircuitState.HALF_OPEN;
            } else {
                console.log(`[${this.name}] Circuit OPEN: Rejecting request`);
                
                if (fallback) {
                    return fallback();
                }
                throw new Error(`Circuit breaker OPEN for ${this.name}`);
            }
        }
        
        try {
            // เพิ่ม timeout
            const result = await Promise.race([
                fn(),
                new Promise<never>((_, reject) =>
                    setTimeout(
                        () => reject(new Error('Request timeout')),
                        this.options.requestTimeout
                    )
                )
            ]);
            
            this.onSuccess();
            return result;
            
        } catch (error) {
            this.onFailure();
            
            if (fallback && this.state === CircuitState.OPEN) {
                return fallback();
            }
            throw error;
        }
    }
    
    private onSuccess(): void {
        if (this.state === CircuitState.HALF_OPEN) {
            this.successes++;
            
            if (this.successes >= this.options.successThreshold) {
                console.log(`[${this.name}] Circuit CLOSED: Service recovered`);
                this.reset();
            }
        } else {
            this.failures = 0;
        }
    }
    
    private onFailure(): void {
        this.failures++;
        this.lastFailureTime = Date.now();
        
        if (
            this.state === CircuitState.HALF_OPEN ||
            this.failures >= this.options.failureThreshold
        ) {
            console.log(`[${this.name}] Circuit OPEN: ${this.failures} failures`);
            this.state = CircuitState.OPEN;
            this.successes = 0;
        }
    }
    
    private reset(): void {
        this.state = CircuitState.CLOSED;
        this.failures = 0;
        this.successes = 0;
    }
    
    getState(): CircuitState {
        return this.state;
    }
    
    getMetrics() {
        return {
            state: this.state,
            failures: this.failures,
            successes: this.successes,
            lastFailureTime: this.lastFailureTime
        };
    }
}

// การใช้งาน
const dbCircuitBreaker = new CircuitBreaker('database', {
    failureThreshold: 5,
    timeout: 30000,      // 30s ก่อน retry
    requestTimeout: 3000  // 3s timeout
});

async function getUserFromDB(userId: number) {
    return dbCircuitBreaker.execute(
        // Main operation
        async () => {
            const result = await pool.query(
                'SELECT * FROM users WHERE id = $1',
                [userId]
            );
            return result.rows[0];
        },
        // Fallback: return from cache
        () => {
            return cache.get(`user:${userId}`);
        }
    );
}
```

### Retry with Exponential Backoff

```typescript
// src/utils/retry.ts
interface RetryOptions {
    maxAttempts: number;
    initialDelayMs: number;
    maxDelayMs: number;
    backoffMultiplier: number;
    retryableErrors?: RegExp[];
}

export async function withRetry<T>(
    fn: () => Promise<T>,
    options: Partial<RetryOptions> = {}
): Promise<T> {
    const opts: RetryOptions = {
        maxAttempts: options.maxAttempts ?? 3,
        initialDelayMs: options.initialDelayMs ?? 100,
        maxDelayMs: options.maxDelayMs ?? 5000,
        backoffMultiplier: options.backoffMultiplier ?? 2,
        retryableErrors: options.retryableErrors ?? [/connection refused/i, /timeout/i]
    };
    
    let lastError: Error;
    
    for (let attempt = 1; attempt <= opts.maxAttempts; attempt++) {
        try {
            return await fn();
        } catch (error: any) {
            lastError = error;
            
            // ตรวจสอบว่า error นี้ควร retry ไหม
            const shouldRetry = opts.retryableErrors?.some(
                pattern => pattern.test(error.message)
            ) ?? true;
            
            if (!shouldRetry || attempt === opts.maxAttempts) {
                throw error;
            }
            
            // Exponential backoff with jitter
            const delay = Math.min(
                opts.initialDelayMs * Math.pow(opts.backoffMultiplier, attempt - 1) +
                Math.random() * 100,  // jitter
                opts.maxDelayMs
            );
            
            console.log(
                `Attempt ${attempt}/${opts.maxAttempts} failed: ${error.message}. ` +
                `Retrying in ${delay.toFixed(0)}ms...`
            );
            
            await new Promise(resolve => setTimeout(resolve, delay));
        }
    }
    
    throw lastError!;
}

// การใช้งาน
const user = await withRetry(
    () => pool.query('SELECT * FROM users WHERE id = $1', [userId]),
    {
        maxAttempts: 3,
        initialDelayMs: 200,
        retryableErrors: [/ECONNREFUSED/, /timeout/, /connection reset/i]
    }
);
```

### Bulkhead Pattern

```typescript
// src/utils/bulkhead.ts
// Bulkhead: จำกัด concurrent requests เพื่อป้องกัน resource exhaustion

export class Bulkhead {
    private activeCount: number = 0;
    private waitQueue: Array<{
        resolve: () => void;
        reject: (err: Error) => void;
    }> = [];
    
    constructor(
        private readonly maxConcurrent: number,
        private readonly maxQueue: number = 100
    ) {}
    
    async execute<T>(fn: () => Promise<T>): Promise<T> {
        await this.acquire();
        
        try {
            return await fn();
        } finally {
            this.release();
        }
    }
    
    private async acquire(): Promise<void> {
        if (this.activeCount < this.maxConcurrent) {
            this.activeCount++;
            return;
        }
        
        if (this.waitQueue.length >= this.maxQueue) {
            throw new Error(`Bulkhead queue full (${this.maxQueue} max)`);
        }
        
        return new Promise((resolve, reject) => {
            this.waitQueue.push({ resolve, reject });
        });
    }
    
    private release(): void {
        this.activeCount--;
        
        const next = this.waitQueue.shift();
        if (next) {
            this.activeCount++;
            next.resolve();
        }
    }
    
    getMetrics() {
        return {
            active: this.activeCount,
            queued: this.waitQueue.length,
            maxConcurrent: this.maxConcurrent,
            maxQueue: this.maxQueue
        };
    }
}

// แยก bulkhead สำหรับ operation ต่างชนิด
const dbReadBulkhead = new Bulkhead(50, 100);   // max 50 concurrent reads
const dbWriteBulkhead = new Bulkhead(20, 50);   // max 20 concurrent writes
const mlServiceBulkhead = new Bulkhead(10, 30); // max 10 ML requests

async function processOrder(order: Order) {
    // เขียน DB ผ่าน bulkhead
    await dbWriteBulkhead.execute(async () => {
        await db.query('INSERT INTO orders...', [order]);
    });
    
    // เรียก ML service ผ่าน bulkhead
    const fraudScore = await mlServiceBulkhead.execute(async () => {
        return await mlService.getFraudScore(order);
    });
    
    return fraudScore;
}
```

---

## 7. Chaos Experiment Process

### Step-by-Step Framework

```python
# chaos_framework.py
from dataclasses import dataclass, field
from datetime import datetime
from typing import Callable, Optional
import time
import logging

@dataclass
class SteadyState:
    """นิยาม "ปกติ" ของระบบ"""
    name: str
    probe: Callable[[], float]    # ฟังก์ชันวัดค่า
    threshold: float              # ค่าที่ยอมรับได้
    comparison: str               # 'lt' (less than), 'gt', 'eq'
    
    def is_satisfied(self) -> tuple[bool, float]:
        """ตรวจสอบว่า steady state ยังเป็นปกติไหม"""
        current_value = self.probe()
        
        if self.comparison == 'lt':
            satisfied = current_value < self.threshold
        elif self.comparison == 'gt':
            satisfied = current_value > self.threshold
        else:
            satisfied = abs(current_value - self.threshold) < 0.01
        
        return satisfied, current_value

@dataclass
class ChaosExperiment:
    """Chaos experiment definition"""
    name: str
    description: str
    hypothesis: str
    
    steady_states: list[SteadyState]
    
    # inject = inject the chaos
    inject: Callable[[], None]
    
    # rollback = restore normal state
    rollback: Callable[[], None]
    
    duration_seconds: int = 300
    
    # Optional: blast radius limit
    blast_radius_check: Optional[Callable[[], bool]] = None

class ChaosRunner:
    """Run chaos experiments systematically"""
    
    def __init__(self, environment: str = "staging"):
        self.environment = environment
        self.logger = logging.getLogger(__name__)
        
        if environment == "production":
            self.logger.warning("Running chaos in PRODUCTION! Proceed carefully.")
    
    def run(self, experiment: ChaosExperiment) -> dict:
        """Execute chaos experiment"""
        
        result = {
            'experiment': experiment.name,
            'environment': self.environment,
            'start_time': datetime.now().isoformat(),
            'phases': {},
            'passed': False
        }
        
        try:
            # Phase 1: Verify steady state (before)
            print(f"\n=== Phase 1: Verify Steady State ===")
            pre_steady = self._check_steady_states(experiment.steady_states)
            result['phases']['pre_experiment'] = pre_steady
            
            if not pre_steady['all_satisfied']:
                print("ABORT: System not in steady state before experiment!")
                result['abort_reason'] = 'system_not_steady'
                return result
            
            # Phase 2: Verify blast radius
            if experiment.blast_radius_check:
                print(f"\n=== Phase 2: Check Blast Radius ===")
                if not experiment.blast_radius_check():
                    print("ABORT: Blast radius check failed!")
                    result['abort_reason'] = 'blast_radius_exceeded'
                    return result
            
            # Phase 3: Inject chaos
            print(f"\n=== Phase 3: Inject Chaos ===")
            print(f"Hypothesis: {experiment.hypothesis}")
            inject_start = time.time()
            experiment.inject()
            result['phases']['inject_time'] = time.time() - inject_start
            
            # Phase 4: Monitor during chaos
            print(f"\n=== Phase 4: Monitor ({experiment.duration_seconds}s) ===")
            during_measurements = []
            
            monitor_start = time.time()
            while time.time() - monitor_start < experiment.duration_seconds:
                measurements = self._check_steady_states(experiment.steady_states)
                during_measurements.append(measurements)
                
                if not measurements['all_satisfied']:
                    print(f"DEVIATION detected at {time.time() - monitor_start:.0f}s")
                
                time.sleep(10)
            
            result['phases']['during_chaos'] = {
                'measurements': during_measurements,
                'deviations': sum(1 for m in during_measurements if not m['all_satisfied'])
            }
            
        finally:
            # Phase 5: Rollback (always)
            print(f"\n=== Phase 5: Rollback ===")
            try:
                experiment.rollback()
                print("Rollback successful")
            except Exception as e:
                print(f"Rollback FAILED: {e}")
                result['rollback_failed'] = str(e)
            
            # Phase 6: Verify steady state (after)
            print(f"\n=== Phase 6: Verify Recovery ===")
            time.sleep(30)  # Wait for recovery
            
            post_steady = self._check_steady_states(experiment.steady_states)
            result['phases']['post_experiment'] = post_steady
            result['passed'] = post_steady['all_satisfied']
        
        result['end_time'] = datetime.now().isoformat()
        return result
    
    def _check_steady_states(self, states: list[SteadyState]) -> dict:
        """Check all steady state conditions"""
        checks = {}
        all_satisfied = True
        
        for state in states:
            satisfied, value = state.is_satisfied()
            checks[state.name] = {
                'satisfied': satisfied,
                'current_value': value,
                'threshold': state.threshold
            }
            if not satisfied:
                all_satisfied = False
                print(f"  FAIL: {state.name} = {value:.3f} (threshold: {state.threshold})")
            else:
                print(f"  OK: {state.name} = {value:.3f}")
        
        return {'all_satisfied': all_satisfied, 'checks': checks}

# ตัวอย่างการใช้งาน
import requests

def get_error_rate():
    response = requests.get("http://prometheus:9090/api/v1/query", 
        params={"query": "rate(http_requests_total{status=~'5..'}[1m])"})
    data = response.json()
    if data['data']['result']:
        return float(data['data']['result'][0]['value'][1])
    return 0.0

def get_p99_latency():
    response = requests.get("http://prometheus:9090/api/v1/query",
        params={"query": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[1m]))"})
    data = response.json()
    if data['data']['result']:
        return float(data['data']['result'][0]['value'][1])
    return 0.0

# Define experiment
redis_failure_experiment = ChaosExperiment(
    name="Redis Master Failure",
    description="Kill Redis master, verify Sentinel failover and app resilience",
    hypothesis="When Redis master fails, Sentinel promotes replica within 30s and error rate stays < 5%",
    
    steady_states=[
        SteadyState(
            name="error_rate",
            probe=get_error_rate,
            threshold=0.05,   # < 5% errors
            comparison='lt'
        ),
        SteadyState(
            name="p99_latency_seconds",
            probe=get_p99_latency,
            threshold=1.0,    # < 1s
            comparison='lt'
        )
    ],
    
    inject=lambda: os.system("docker kill redis-master"),
    rollback=lambda: os.system("docker start redis-master"),
    duration_seconds=120
)

# Run experiment
runner = ChaosRunner(environment="staging")
result = runner.run(redis_failure_experiment)

print(f"\nExperiment {'PASSED' if result['passed'] else 'FAILED'}")
print(f"Results: {json.dumps(result, indent=2)}")
```

---

## 8. Chaos Dashboard และ Reporting

```python
# chaos_report.py
import json
from datetime import datetime
from typing import list

def generate_chaos_report(experiments: list[dict]) -> str:
    """สร้าง HTML report สำหรับ chaos experiments"""
    
    passed = sum(1 for e in experiments if e.get('passed'))
    failed = len(experiments) - passed
    
    rows = ""
    for exp in experiments:
        status_class = "passed" if exp.get('passed') else "failed"
        status_text = "PASSED" if exp.get('passed') else "FAILED"
        
        rows += f"""
        <tr class="{status_class}">
            <td>{exp.get('experiment', 'Unknown')}</td>
            <td>{exp.get('environment', 'Unknown')}</td>
            <td>{exp.get('start_time', '')[:19]}</td>
            <td class="status">{status_text}</td>
        </tr>"""
    
    html = f"""
<!DOCTYPE html>
<html>
<head>
    <title>Chaos Engineering Report</title>
    <style>
        body {{ font-family: Arial, sans-serif; margin: 20px; }}
        h1 {{ color: #333; }}
        .summary {{ display: flex; gap: 20px; margin: 20px 0; }}
        .metric {{ padding: 20px; border-radius: 8px; text-align: center; }}
        .passed {{ background: #d4edda; }}
        .failed {{ background: #f8d7da; }}
        table {{ border-collapse: collapse; width: 100%; }}
        th, td {{ border: 1px solid #ddd; padding: 8px; text-align: left; }}
        th {{ background: #333; color: white; }}
        tr.passed {{ background: #d4edda; }}
        tr.failed {{ background: #f8d7da; }}
        .status {{ font-weight: bold; }}
    </style>
</head>
<body>
    <h1>Chaos Engineering Report</h1>
    <p>Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}</p>
    
    <div class="summary">
        <div class="metric passed">
            <h2>{passed}</h2>
            <p>Passed</p>
        </div>
        <div class="metric failed">
            <h2>{failed}</h2>
            <p>Failed</p>
        </div>
        <div class="metric">
            <h2>{len(experiments)}</h2>
            <p>Total</p>
        </div>
    </div>
    
    <table>
        <tr>
            <th>Experiment</th>
            <th>Environment</th>
            <th>Time</th>
            <th>Result</th>
        </tr>
        {rows}
    </table>
</body>
</html>"""
    
    return html
```

---

## 9. GameDay Checklist

```markdown
# GameDay Runbook

## Before GameDay

### 1 Week Before
- [ ] เลือก scenarios สำหรับ GameDay
- [ ] แจ้ง team และ stakeholders
- [ ] ตรวจสอบ monitoring พร้อม
- [ ] ทดสอบ rollback procedures
- [ ] ตรวจสอบ on-call engineers พร้อม

### 1 Day Before
- [ ] Review runbooks
- [ ] ตรวจสอบ backup สถานะ OK
- [ ] ตรวจสอบ DR procedures up-to-date
- [ ] Setup dedicated Slack channel

### Day Of
- [ ] Status page ready
- [ ] All team members on standby
- [ ] Metrics dashboard open
- [ ] Record session (screen capture)

## During GameDay

### For Each Experiment
1. **Announce**: แจ้ง team ว่า experiment กำลังจะเริ่ม
2. **Measure**: บันทึก baseline metrics
3. **Inject**: ทำตาม inject procedure
4. **Observe**: monitor impact 5-10 นาที
5. **Rollback**: restore ตาม rollback procedure
6. **Verify**: ตรวจสอบ steady state กลับมาแล้ว
7. **Document**: บันทึก findings ทันที

### Stop Criteria (ยกเลิก experiment ทันที)
- Error rate > 20%
- P99 latency > 10 seconds
- Data loss detected
- Production customer impact
- Team consensus to stop

## After GameDay

### Post-Mortem (ภายใน 48 ชั่วโมง)
- [ ] Timeline of events
- [ ] What went well
- [ ] What needs improvement  
- [ ] Action items with owners และ deadlines
- [ ] Update runbooks
- [ ] Share learnings กับ broader team
```

---

## สรุป

Chaos Engineering เป็นวิธีที่ดีที่สุดในการตรวจสอบว่าระบบของเรา resilient จริงๆ หรือแค่คิดว่า resilient:

1. **Start Small** - เริ่มจาก staging ด้วย blast radius เล็กๆ
2. **Define Steady State** - รู้ว่า "ปกติ" คืออะไรก่อน inject chaos
3. **Automate** - Integrate กับ CI/CD เพื่อ continuous testing
4. **Fix Weaknesses** - ทุก failed experiment = โอกาสเพิ่ม reliability
5. **Circuit Breakers** - ป้องกัน cascade failures
6. **Retry + Timeout** - Handle transient failures
7. **Bulkhead** - จำกัด blast radius ของ failures
8. **GameDays** - ฝึกซ้อม team ด้วย real scenarios

**"The best time to find out your DR doesn't work is NOT during a real disaster."**
