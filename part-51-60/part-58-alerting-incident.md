# Part 58: Alerting และ Incident Response

## บทนำ: จาก Metrics สู่ Action

Metrics และ Tracing บอกเราว่าระบบเป็นอย่างไร แต่ **Alerting** คือสิ่งที่ทำให้เราตื่นตัวเมื่อมีปัญหา และ **Incident Response** คือกระบวนการที่ทำให้เราจัดการปัญหาได้อย่างมีระเบียบ

```
Metrics → Alert Rules → Alertmanager → Notification
                                    ↓
                            On-call Engineer
                                    ↓
                          Incident Response Process
                                    ↓
                          Resolution + Post-mortem
```

---

## Alertmanager: Prometheus Alerting

### ติดตั้ง Alertmanager ด้วย Docker

```yaml
# docker-compose.alertmanager.yml
version: '3.8'

services:
  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    ports:
      - "9093:9093"
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--web.external-url=http://alertmanager.example.com:9093'
      - '--cluster.advertise-address=0.0.0.0:9093'
      # HA mode (multiple instances)
      # - '--cluster.peer=alertmanager-2:9094'
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - ./alertmanager/templates:/etc/alertmanager/templates
      - alertmanager-data:/alertmanager
    restart: unless-stopped

volumes:
  alertmanager-data:
```

### alertmanager.yml: Full Configuration

```yaml
# alertmanager/alertmanager.yml

global:
  # Default resolve timeout (ถ้า alert หายไปนานกว่านี้ = resolved)
  resolve_timeout: 5m
  
  # Slack webhook (ใช้ใน receivers)
  slack_api_url: 'https://hooks.slack.com/services/XXXX/YYYY/ZZZZ'
  
  # SMTP settings
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@example.com'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: '${SMTP_PASSWORD}'
  smtp_require_tls: true

# Templates สำหรับ custom notification format
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# ==================== Routing ====================
route:
  # Default receiver (ถ้าไม่ match rule ใดเลย)
  receiver: 'team-slack-general'
  
  # Group alerts ที่ label เหมือนกันเข้าด้วยกัน
  group_by: ['alertname', 'cluster', 'service']
  
  # รอ 30s ก่อนส่ง alert แรก (เผื่อ alerts หลายอัน group เข้ากัน)
  group_wait: 30s
  
  # รอ 5m ก่อนส่ง alert ใหม่ในกลุ่มเดิม
  group_interval: 5m
  
  # ส่ง reminder ทุก 4h ถ้า alert ยังไม่ resolve
  repeat_interval: 4h
  
  # Child routes (match ก่อน จาก บนลงล่าง)
  routes:
    # P1: Critical - ส่ง PagerDuty ทันที
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      group_wait: 0s        # ส่งทันที
      repeat_interval: 30m  # Repeat ทุก 30 นาที
      routes:
        # Database P1 alerts → DBA on-call
        - match:
            team: database
          receiver: 'pagerduty-dba'
          
        # Payment P1 alerts → Payment team on-call
        - match:
            team: payment
          receiver: 'pagerduty-payment'
    
    # P2: Warning - ส่ง Slack แต่ไม่ wake up
    - match:
        severity: warning
      receiver: 'team-slack-alerts'
      group_wait: 1m
      repeat_interval: 2h
    
    # Deadman switch - ถ้าหยุดยิง = Prometheus ล่ม
    - match:
        alertname: DeadManSwitch
      receiver: 'deadman-switch'
      group_wait: 0s
      repeat_interval: 1m
    
    # Database alerts
    - match:
        team: database
      receiver: 'team-slack-database'
      group_by: ['alertname', 'database', 'instance']
    
    # Maintenance window: silence ทุก alerts
    - match_re:
        severity: "warning|critical"
      receiver: 'null'
      matchers:
        - name: maintenance
          value: "true"
          isEqual: true

# ==================== Inhibition Rules ====================
inhibit_rules:
  # ถ้า critical alert ยิง → suppress warning ที่ labels เหมือนกัน
  - source_matchers:
      - severity = critical
    target_matchers:
      - severity = warning
    equal: ['alertname', 'cluster', 'service']
  
  # ถ้า node down → suppress service alerts บน node นั้น
  - source_matchers:
      - alertname = NodeDown
    target_matchers:
      - alertname =~ "Service.*"
    equal: ['instance']
  
  # ถ้า postgres primary down → suppress replica alerts
  - source_matchers:
      - alertname = PostgresPrimaryDown
    target_matchers:
      - alertname = PostgresReplicationLag
    equal: ['cluster']

# ==================== Receivers ====================
receivers:
  # Null receiver (ทิ้งทุกอย่าง)
  - name: 'null'
  
  # General Slack channel
  - name: 'team-slack-general'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/XXXX/YYYY/ZZZZ'
        channel: '#monitoring'
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'
        icon_emoji: ':bell:'
        send_resolved: true
  
  # Alert Slack channel
  - name: 'team-slack-alerts'
    slack_configs:
      - channel: '#alerts'
        title: '[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Details:*{{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`{{ end }}
          *Description:* {{ .Annotations.description }}
          *Runbook:* {{ .Annotations.runbook_url }}
          {{ end }}
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
        send_resolved: true
        
  # Database team Slack
  - name: 'team-slack-database'
    slack_configs:
      - channel: '#db-alerts'
        title: '🗄️ DB Alert: {{ .CommonLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *{{ .Annotations.summary }}*
          Instance: `{{ .Labels.instance }}`
          {{ .Annotations.description }}
          Runbook: {{ .Annotations.runbook_url }}
          {{ end }}
        send_resolved: true
  
  # PagerDuty Critical
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: '${PAGERDUTY_SERVICE_KEY}'
        description: '{{ template "pagerduty.default.description" . }}'
        client: 'Prometheus Alertmanager'
        client_url: 'https://prometheus.example.com'
        details:
          firing: '{{ template "pagerduty.default.instances" .Alerts.Firing }}'
          resolved: '{{ template "pagerduty.default.instances" .Alerts.Resolved }}'
          num_firing: '{{ .Alerts.Firing | len }}'
          num_resolved: '{{ .Alerts.Resolved | len }}'
          
  # PagerDuty DBA
  - name: 'pagerduty-dba'
    pagerduty_configs:
      - service_key: '${PAGERDUTY_DBA_KEY}'
        severity: 'critical'
    slack_configs:
      - channel: '#dba-oncall'
        title: '🚨 DBA Alert: {{ .CommonLabels.alertname }}'
        send_resolved: true
  
  # Dead Man Switch (ไม่ใช้ Slack เพราะ Prometheus อาจล่มอยู่)
  - name: 'deadman-switch'
    webhook_configs:
      - url: 'https://healthchecks.io/ping/${HEALTHCHECK_ID}'
        send_resolved: false
  
  # Email for SLA reports
  - name: 'email-sla'
    email_configs:
      - to: 'sla-team@example.com'
        from: 'alerts@example.com'
        subject: 'SLA Alert: {{ .CommonLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          
          Details:
          {{ range .Labels.SortedPairs }}- {{ .Name }}: {{ .Value }}
          {{ end }}
          
          Description: {{ .Annotations.description }}
          
          Started: {{ .StartsAt }}
          {{ if .EndsAt }}Ended: {{ .EndsAt }}{{ end }}
          {{ end }}
        send_resolved: true
```

---

## Alert Templates

```
{{/* alertmanager/templates/slack.tmpl */}}

{{ define "slack.title" }}
  [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}]
  {{ .CommonLabels.alertname }} 
  ({{ .CommonLabels.cluster }})
{{ end }}

{{ define "slack.text" }}
{{ range .Alerts }}
  {{ if eq .Status "firing" }}:red_circle:{{ else }}:large_green_circle:{{ end }}
  *{{ .Annotations.summary }}*
  
  {{ if .Annotations.description }}
  > {{ .Annotations.description }}
  {{ end }}
  
  *Labels:*{{ range .Labels.SortedPairs }}
  • `{{ .Name }}`: {{ .Value }}{{ end }}
  
  {{ if .Annotations.runbook_url }}
  :book: <{{ .Annotations.runbook_url }}|Runbook>
  {{ end }}
  
  :clock1: Started: {{ .StartsAt | since }}
{{ end }}
{{ end }}
```

---

## Prometheus Alert Rules

### PostgreSQL Alert Rules

```yaml
# prometheus/rules/postgres_alerts.yml
groups:
  - name: postgresql
    rules:
      # ==================== Connections ====================
      
      # High connection count (>80% of max)
      - alert: PostgresHighConnections
        expr: |
          (
            sum by (instance) (pg_stat_database_numbackends)
            / on(instance) pg_settings_max_connections
          ) > 0.80
        for: 5m
        labels:
          severity: warning
          team: database
        annotations:
          summary: "PostgreSQL high connection count on {{ $labels.instance }}"
          description: |
            Connection usage is {{ $value | humanizePercentage }} of max connections.
            Current: {{ $labels.instance }} is using {{ $value | humanizePercentage }} connections.
            Consider increasing max_connections or using PgBouncer.
          runbook_url: "https://wiki.example.com/runbooks/postgres-high-connections"
          
      - alert: PostgresConnectionsCritical
        expr: |
          (
            sum by (instance) (pg_stat_database_numbackends)
            / on(instance) pg_settings_max_connections
          ) > 0.95
        for: 2m
        labels:
          severity: critical
          team: database
        annotations:
          summary: "PostgreSQL connections near limit on {{ $labels.instance }}"
          description: |
            Connection usage is {{ $value | humanizePercentage }}.
            New connections will fail if this reaches 100%.
            IMMEDIATE ACTION REQUIRED.
          runbook_url: "https://wiki.example.com/runbooks/postgres-connections-critical"
      
      # ==================== Replication ====================
      
      # Replication lag > 10s
      - alert: PostgresReplicationLagWarning
        expr: pg_replication_replication_delay_seconds > 10
        for: 3m
        labels:
          severity: warning
          team: database
        annotations:
          summary: "PostgreSQL replication lag on {{ $labels.instance }}"
          description: |
            Replication lag is {{ $value | humanizeDuration }}.
            Replica may serve stale data.
          runbook_url: "https://wiki.example.com/runbooks/postgres-replication-lag"
          
      # Replication lag > 30s
      - alert: PostgresReplicationLagCritical
        expr: pg_replication_replication_delay_seconds > 30
        for: 1m
        labels:
          severity: critical
          team: database
        annotations:
          summary: "CRITICAL: PostgreSQL replication lag > 30s on {{ $labels.instance }}"
          description: |
            Replication lag is {{ $value | humanizeDuration }}.
            Failover may result in data loss.
          runbook_url: "https://wiki.example.com/runbooks/postgres-replication-critical"
      
      # Replication broken (no replica connected)
      - alert: PostgresNoReplication
        expr: pg_stat_replication_pg_current_wal_lsn_bytes == 0
        for: 5m
        labels:
          severity: critical
          team: database
        annotations:
          summary: "No PostgreSQL replication on {{ $labels.instance }}"
          description: "No replicas are connected. Check replica health immediately."
      
      # ==================== Performance ====================
      
      # Slow queries (>5 seconds)
      - alert: PostgresSlowQueries
        expr: pg_slow_queries_count > 5
        for: 2m
        labels:
          severity: warning
          team: database
        annotations:
          summary: "PostgreSQL slow queries detected on {{ $labels.instance }}"
          description: |
            {{ $value }} queries running for more than 5 seconds.
            Check pg_stat_activity for details.
          runbook_url: "https://wiki.example.com/runbooks/postgres-slow-queries"
      
      # Cache hit ratio low
      - alert: PostgresLowCacheHitRatio
        expr: |
          (
            sum by (datname) (pg_stat_database_blks_hit)
            / (
              sum by (datname) (pg_stat_database_blks_hit)
              + sum by (datname) (pg_stat_database_blks_read)
            )
          ) < 0.95
        for: 10m
        labels:
          severity: warning
          team: database
        annotations:
          summary: "Low cache hit ratio for {{ $labels.datname }}"
          description: |
            Cache hit ratio is {{ $value | humanizePercentage }}.
            Consider increasing shared_buffers.
      
      # ==================== Disk ====================
      
      # Disk space < 30%
      - alert: PostgresDiskSpaceWarning
        expr: |
          node_filesystem_avail_bytes{mountpoint="/var/lib/postgresql"}
          / node_filesystem_size_bytes{mountpoint="/var/lib/postgresql"}
          < 0.30
        for: 5m
        labels:
          severity: warning
          team: database
        annotations:
          summary: "PostgreSQL disk space low on {{ $labels.instance }}"
          description: |
            Available disk space: {{ $value | humanizePercentage }}.
            Plan disk expansion soon.
      
      # Disk space < 10%
      - alert: PostgresDiskSpaceCritical
        expr: |
          node_filesystem_avail_bytes{mountpoint="/var/lib/postgresql"}
          / node_filesystem_size_bytes{mountpoint="/var/lib/postgresql"}
          < 0.10
        for: 2m
        labels:
          severity: critical
          team: database
        annotations:
          summary: "CRITICAL: PostgreSQL disk space critical on {{ $labels.instance }}"
          description: |
            Available disk space: {{ $value | humanizePercentage }}.
            DATABASE MAY STOP ACCEPTING WRITES SOON.
      
      # ==================== Dead Tuples ====================
      
      # High dead tuples (bloat)
      - alert: PostgresHighDeadTuples
        expr: |
          pg_stat_user_tables_n_dead_tup
          / (pg_stat_user_tables_n_live_tup + pg_stat_user_tables_n_dead_tup + 1)
          > 0.20
        for: 30m
        labels:
          severity: warning
          team: database
        annotations:
          summary: "High dead tuples on {{ $labels.tablename }}"
          description: |
            Table {{ $labels.schemaname }}.{{ $labels.tablename }} has
            {{ $value | humanizePercentage }} dead tuples.
            Manual VACUUM may be needed.
      
      # ==================== Backup ====================
      
      # Backup not completed in 24 hours
      - alert: PostgresBackupMissing
        expr: |
          time() - postgres_last_backup_timestamp > 86400
        for: 0m
        labels:
          severity: critical
          team: database
        annotations:
          summary: "PostgreSQL backup is overdue on {{ $labels.instance }}"
          description: |
            Last backup was {{ $value | humanizeDuration }} ago.
            Check backup job immediately.
          runbook_url: "https://wiki.example.com/runbooks/postgres-backup-missing"
      
      # Postgres instance down
      - alert: PostgresDown
        expr: pg_up == 0
        for: 1m
        labels:
          severity: critical
          team: database
        annotations:
          summary: "PostgreSQL instance is down: {{ $labels.instance }}"
          description: "PostgreSQL instance {{ $labels.instance }} is not responding."
          runbook_url: "https://wiki.example.com/runbooks/postgres-down"
```

### Redis Alert Rules

```yaml
# prometheus/rules/redis_alerts.yml
groups:
  - name: redis
    rules:
      # Redis down
      - alert: RedisDown
        expr: redis_up == 0
        for: 1m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Redis instance is down: {{ $labels.instance }}"
          description: "Redis instance {{ $labels.instance }} is not responding."
          runbook_url: "https://wiki.example.com/runbooks/redis-down"
      
      # Memory usage > 80%
      - alert: RedisHighMemoryUsage
        expr: |
          redis_memory_used_bytes / redis_memory_max_bytes > 0.80
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Redis high memory usage on {{ $labels.instance }}"
          description: |
            Memory usage is {{ $value | humanizePercentage }}.
            Used: {{ $labels.instance }} redis_memory_used_bytes
            Consider increasing maxmemory or scaling Redis.
          runbook_url: "https://wiki.example.com/runbooks/redis-high-memory"
      
      # Memory usage > 95%
      - alert: RedisMemoryCritical
        expr: |
          redis_memory_used_bytes / redis_memory_max_bytes > 0.95
        for: 2m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "CRITICAL: Redis memory almost full on {{ $labels.instance }}"
          description: |
            Memory usage is {{ $value | humanizePercentage }}.
            Redis will start evicting or rejecting writes.
      
      # High eviction rate
      - alert: RedisHighEvictionRate
        expr: |
          rate(redis_evicted_keys_total[5m]) > 100
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "High Redis eviction rate on {{ $labels.instance }}"
          description: |
            Evicting {{ $value | humanize }} keys/s.
            Data is being lost. Increase maxmemory or reduce cache usage.
      
      # Replication broken
      - alert: RedisReplicationBroken
        expr: |
          redis_connected_slaves < 1
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Redis has no replicas: {{ $labels.instance }}"
          description: |
            Redis master {{ $labels.instance }} has no connected replicas.
            Failover will not be possible.
          runbook_url: "https://wiki.example.com/runbooks/redis-replication"
      
      # Replication lag > 10s
      - alert: RedisReplicationLag
        expr: |
          (redis_master_repl_offset - redis_replica_repl_offset) > 100000
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Redis replication lag on {{ $labels.instance }}"
          description: |
            Replication offset difference: {{ $value }} bytes.
            Replica may serve stale data.
      
      # Too many connected clients
      - alert: RedisTooManyClients
        expr: |
          redis_connected_clients > redis_config_maxclients * 0.80
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Redis client limit approaching on {{ $labels.instance }}"
          description: |
            {{ $value }} clients connected ({{ $labels.instance }}).
            Maximum: {{ redis_config_maxclients }}
      
      # High latency
      - alert: RedisHighLatency
        expr: redis_latency_percentiles_usec{percentile="p99"} > 10000
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "High Redis latency on {{ $labels.instance }}"
          description: |
            P99 latency: {{ $value }}µs ({{ $value | humanizeDuration }}).
            Expected < 1ms. Check for slow commands with SLOWLOG.
      
      # Low cache hit ratio
      - alert: RedisLowHitRatio
        expr: |
          rate(redis_keyspace_hits_total[5m])
          / (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
          < 0.70
        for: 15m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Low Redis cache hit ratio on {{ $labels.instance }}"
          description: |
            Cache hit ratio: {{ $value | humanizePercentage }}.
            Expected > 70%. Review caching strategy.
```

### Application Alert Rules

```yaml
# prometheus/rules/application_alerts.yml
groups:
  - name: application
    rules:
      # High error rate > 1%
      - alert: HighErrorRate
        expr: |
          (
            sum by (service) (rate(http_requests_total{status_code=~"5.."}[5m]))
            / sum by (service) (rate(http_requests_total[5m]))
          ) > 0.01
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate for {{ $labels.service }}"
          description: |
            Error rate is {{ $value | humanizePercentage }}.
            Check application logs immediately.
          runbook_url: "https://wiki.example.com/runbooks/high-error-rate"
      
      # P99 latency > 2s
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum by (service, le) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 2
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High P99 latency for {{ $labels.service }}"
          description: |
            P99 latency is {{ $value | humanizeDuration }}.
            99% of requests are taking longer than 2 seconds.
          runbook_url: "https://wiki.example.com/runbooks/high-latency"
      
      # P95 latency > 1s (warning)
      - alert: ElevatedLatencyP95
        expr: |
          histogram_quantile(0.95,
            sum by (service, le) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 1
        for: 10m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Elevated P95 latency for {{ $labels.service }}"
          description: |
            P95 latency is {{ $value | humanizeDuration }}.
      
      # Anomaly: Request rate too low (possible outage)
      - alert: LowRequestRate
        expr: |
          sum by (service) (rate(http_requests_total[5m]))
          < sum by (service) (rate(http_requests_total[5m] offset 1w)) * 0.50
        for: 10m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Unusually low request rate for {{ $labels.service }}"
          description: |
            Current rate: {{ $value | humanize }} req/s.
            This is less than 50% of last week at this time.
            Possible traffic issue or outage.
      
      # Health check failing
      - alert: HealthCheckFailing
        expr: |
          probe_success{job="blackbox"} == 0
        for: 2m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Health check failing for {{ $labels.instance }}"
          description: |
            Health check endpoint is not responding.
            Service may be down.
          runbook_url: "https://wiki.example.com/runbooks/health-check-failing"
      
      # Queue size too high
      - alert: QueueSizeHigh
        expr: queue_size > 10000
        for: 10m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Queue {{ $labels.queue_name }} size is high"
          description: |
            Queue has {{ $value | humanize }} items.
            Processing may be falling behind.
      
      # High DB connection pool usage
      - alert: DBPoolExhausted
        expr: |
          db_pool_size{state="waiting"} > 0
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "DB connection pool exhausted for {{ $labels.service }}"
          description: |
            {{ $value }} requests waiting for DB connections.
            Consider increasing pool size or optimizing queries.
      
      # Dead Man Switch (ตรวจสอบว่า Prometheus ยังทำงาน)
      - alert: DeadManSwitch
        expr: vector(1)
        labels:
          severity: none
        annotations:
          summary: "Prometheus is alive"
          description: "This alert is always firing to verify alerting works."
```

---

## Runbooks: คู่มือรับมือเมื่อ Alert ยิง

### Runbook: PostgreSQL High Connections

```markdown
# Runbook: PostgreSQL High Connections

## Summary
Connection count has exceeded 80% of max_connections.

## Severity: Warning → Critical

## Immediate Actions (< 5 minutes)

### 1. ตรวจสอบ connections ปัจจุบัน
```sql
-- ดู connections ทั้งหมด
SELECT state, count(*) 
FROM pg_stat_activity 
GROUP BY state;

-- ดู queries ที่ใช้นานที่สุด
SELECT pid, usename, state, query_start, 
       EXTRACT(EPOCH FROM (now() - query_start)) AS duration_secs,
       LEFT(query, 100) AS query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration_secs DESC
LIMIT 20;
```

### 2. Kill idle connections (ถ้าจำเป็น)
```sql
-- Kill connections ที่ idle นานกว่า 10 นาที
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND query_start < now() - interval '10 minutes';
```

### 3. ตรวจสอบ PgBouncer (ถ้ามี)
```bash
# ดู PgBouncer pools
psql -h pgbouncer-host -p 6432 -U pgbouncer pgbouncer -c "SHOW POOLS;"

# Reload PgBouncer config
psql -h pgbouncer-host -p 6432 -U pgbouncer pgbouncer -c "RELOAD;"
```

## Root Cause Analysis
- Connection leak ใน application code
- PgBouncer pool misconfiguration  
- Sudden traffic spike
- Long-running transactions

## Resolution
1. Fix connection leak ใน application
2. Add connection timeout
3. Tune PgBouncer pool_size
4. Scale horizontally (read replicas)
```

### Runbook: Redis High Memory

```markdown
# Runbook: Redis High Memory Usage

## Summary
Redis memory usage > 80% of maxmemory.

## Immediate Actions

### 1. ตรวจสอบ memory usage
```bash
redis-cli -h redis-host INFO memory | grep -E "used_memory|maxmemory|fragmentation"

redis-cli -h redis-host MEMORY DOCTOR

# หา big keys
redis-cli -h redis-host --bigkeys --scan
```

### 2. ตรวจสอบ eviction policy
```bash
redis-cli -h redis-host CONFIG GET maxmemory-policy
```

### 3. หา keys ที่ไม่มี TTL
```bash
# Check keyspace stats
redis-cli -h redis-host INFO keyspace

# Sample keys without TTL (อย่าใช้ KEYS * ใน production)
redis-cli -h redis-host --scan --pattern "user:*" | head -20 | \
  xargs -I {} redis-cli -h redis-host TTL {}
```

### 4. ลบ keys ที่ไม่จำเป็น
```bash
# ลบ keys ตาม pattern (ระวัง!)
redis-cli -h redis-host --scan --pattern "temp:*" | xargs redis-cli DEL

# หรือใช้ UNLINK (non-blocking)
redis-cli -h redis-host --scan --pattern "temp:*" | xargs redis-cli UNLINK
```

### 5. เพิ่ม maxmemory (temporary fix)
```bash
redis-cli -h redis-host CONFIG SET maxmemory 8gb
redis-cli -h redis-host CONFIG REWRITE  # บันทึกถาวร
```

## Long-term Solutions
1. Add TTL ให้ทุก cache key
2. Implement cache eviction strategy
3. Scale Redis vertically หรือ horizontally (Cluster)
4. Review data that shouldn't be in cache
```

---

## On-Call Rotation

### การตั้งค่า On-Call

```yaml
# oncall-schedule.yml (PagerDuty-style)
schedules:
  - name: "Backend On-Call"
    timezone: "Asia/Bangkok"
    layers:
      - name: "Primary"
        rotation_type: "weekly"
        rotation_start: "Monday 09:00"
        users:
          - "engineer-a"
          - "engineer-b"
          - "engineer-c"
          - "engineer-d"
        
      - name: "Secondary (Escalation)"
        rotation_type: "weekly"
        escalation_delay: 15m  # ถ้า primary ไม่รับใน 15 นาที
        users:
          - "team-lead-1"
          - "team-lead-2"

  - name: "DBA On-Call"
    timezone: "Asia/Bangkok"
    layers:
      - name: "Primary"
        rotation_type: "weekly"
        users:
          - "dba-senior"
          - "dba-junior"
          
escalation_policies:
  - name: "Critical Database"
    rules:
      - escalation_delay: 0
        targets:
          - type: "schedule"
            name: "DBA On-Call"
      - escalation_delay: 15m
        targets:
          - type: "user"
            name: "cto"
```

---

## Incident Management Process

### Incident Declaration

```markdown
## When to Declare an Incident

ประกาศ Incident เมื่อ:
1. Service down หรือ severely degraded > 5 minutes
2. Error rate > 5% เป็นเวลา > 2 minutes
3. All critical alerts ยิงพร้อมกัน
4. User-visible impact (ลูกค้ารายงานปัญหา)
5. Security breach detected

ไม่ต้องประกาศ Incident เมื่อ:
- Brief blip < 1 minute
- Single non-critical service down
- Internal tools down (ไม่กระทบ users)
```

### Severity Levels

```markdown
## P1 (Critical) - Wake Up!
- **Definition**: Complete service outage หรือ severe data loss
- **Impact**: ลูกค้าทุกคนได้รับผลกระทบ
- **Response Time**: < 5 minutes
- **Examples**:
  - ระบบ payment ล่มทั้งหมด
  - Database primary down และ failover ล้มเหลว
  - Security breach
- **Actions**: Wake on-call, ประกาศ incident, escalate ทันที

## P2 (High) - Urgent
- **Definition**: Major feature unavailable หรือ significant degradation
- **Impact**: ลูกค้าบางส่วนได้รับผลกระทบ
- **Response Time**: < 30 minutes
- **Examples**:
  - Search functionality down
  - Checkout slow (>5s p99)
  - Replica down (Primary ยังทำงาน)
- **Actions**: ติดต่อ on-call, ประกาศใน Slack

## P3 (Medium) - Non-urgent  
- **Definition**: Minor feature unavailable หรือ minor degradation
- **Impact**: ลูกค้าบางส่วนได้รับผลกระทบเล็กน้อย
- **Response Time**: < 4 hours (within business hours)
- **Examples**:
  - Email notification slow
  - Cache hit ratio low
  - Minor performance degradation
- **Actions**: Create ticket, ดูแลใน working hours

## P4 (Low) - Informational
- **Definition**: No impact แต่ควรติดตาม
- **Response Time**: < 24 hours
- **Examples**:
  - Certificate expiry in 30 days
  - Disk usage at 60%
  - Dependency reaching EOL
- **Actions**: Create ticket, plan fix
```

### Incident Response Workflow

```markdown
## Standard Incident Response Process

### Phase 1: Detection (0-5 minutes)
1. Alert ยิง → On-call รับ notification
2. ตรวจสอบว่า alert จริงหรือ false positive
3. ประกาศ Incident ใน #incidents Slack channel
4. สร้าง Incident ticket

### Phase 2: Triage (5-15 minutes)
1. Incident Commander (IC) รับหน้าที่
2. ประเมิน severity และ impact
3. ส่งข้อความอัปเดตใน status page (ถ้า P1/P2)
4. ระบุ domain ที่รับผิดชอบ (Database, Backend, Infra)

### Phase 3: Mitigation (15-60 minutes)
1. ใช้ mitigation strategies:
   - Rollback deployment
   - Enable maintenance mode
   - Scale up instances
   - Failover to backup
2. อัปเดต status page ทุก 15 นาที
3. Document actions ที่ทำไปแล้ว

### Phase 4: Resolution
1. ยืนยันว่าระบบกลับมาปกติ
2. Monitor 30 นาที หลัง fix
3. ปิด Incident
4. อัปเดต status page "resolved"

### Phase 5: Post-mortem (24-72 hours)
1. เขียน post-mortem document
2. ประชุม blameless post-mortem
3. สร้าง action items
4. ติดตาม action items ใน sprint
```

### Incident Communication Template

```markdown
# Slack: #incidents channel

## Incident Declared
🚨 **INCIDENT DECLARED**: [Brief description]
- **Severity**: P1/P2/P3
- **Started**: 2024-01-15 14:30 ICT
- **Impact**: [Who is affected and how]
- **Incident Commander**: @engineer-name
- **Status Page**: https://status.example.com

## Status Updates (every 15 min for P1)
🔄 **UPDATE** [14:45]: Investigation ongoing.
Finding: [What we found so far]
Next steps: [What we're trying]
ETA: [Best estimate to resolution]

## Resolution
✅ **RESOLVED** [15:20]: 
Root cause: [Brief explanation]
Fix: [What was done]
Duration: 50 minutes
Post-mortem: [Link to document]
```

---

## RCA และ Post-Mortem Template

```markdown
# Post-Mortem: [Incident Title]
**Date**: 2024-01-15
**Authors**: [Names]
**Severity**: P1
**Duration**: 50 minutes (14:30 - 15:20 ICT)

## Summary
[2-3 sentences describing what happened, impact, and how it was resolved]

ตัวอย่าง:
ระบบ payment ล่มเป็นเวลา 50 นาที เนื่องจาก database migration ที่ไม่ได้ทดสอบ
บน production data volume ทำให้ lock ทั้ง table เป็นเวลานาน ส่งผลให้ 
ลูกค้าไม่สามารถ checkout ได้ มีผลกระทบต่อ ~500 transactions

## Timeline
| เวลา (ICT) | เหตุการณ์ |
|------------|-----------|
| 14:25      | Deploy migration script ขึ้น production |
| 14:28      | Payment error rate เพิ่มขึ้นจาก 0% → 100% |
| 14:32      | Alert ยิง, on-call รับ PagerDuty |
| 14:35      | Incident declared, triage เริ่ม |
| 14:45      | Root cause identified: ALTER TABLE lock |
| 14:50      | ตัดสินใจ rollback migration |
| 15:10      | Rollback complete, service recovering |
| 15:20      | Error rate กลับสู่ 0%, incident closed |

## Root Cause
Migration script `ALTER TABLE orders ADD COLUMN discount_code VARCHAR(50)`
เมื่อรันบน table ที่มีข้อมูล 50 ล้าน rows จะ lock table ทั้งหมด
นาน ~45 นาที ทำให้ทุก query บน orders table รอ

**Contributing Factors**:
1. ไม่ได้ทดสอบ migration บน production-size data
2. ไม่มี procedure สำหรับ zero-downtime migrations
3. Monitoring ไม่ได้ alert เมื่อ table lock เกิน 5 วินาที

## Impact
- **Users affected**: ~2,000 users
- **Failed transactions**: ~500 orders
- **Revenue impact**: ประมาณ ฿150,000
- **Duration**: 50 minutes

## What Went Well
1. Alert ยิงภายใน 4 นาทีหลัง incident เริ่ม
2. Root cause หาได้ภายใน 13 นาที
3. Rollback procedure ทำงานได้ตามที่คาด

## What Went Wrong
1. Migration ไม่ได้ทดสอบ performance บน production data volume
2. ไม่มี canary deployment สำหรับ migrations
3. ไม่มี automatic rollback trigger

## Action Items
| Priority | Action | Owner | Due Date |
|----------|--------|-------|----------|
| P1 | เพิ่ม database lock monitoring | DBA Team | 2024-01-22 |
| P1 | สร้าง zero-downtime migration guide | Platform | 2024-01-25 |
| P2 | ทดสอบ migration บน production-size data ก่อน deploy | Backend | 2024-01-29 |
| P2 | Implement migration review checklist | Platform | 2024-02-05 |
| P3 | ปรับปรุง alert: lock wait > 5s | SRE | 2024-02-12 |

## Lessons Learned
1. Schema migration บน large tables ต้องใช้ techniques เช่น:
   - `ALTER TABLE ... USING concurrent` (PostgreSQL 11+)
   - Add column จาก application side ก่อน
   - pg_repack สำหรับ table restructuring
   
2. ทุก migration ควรมี:
   - Estimated execution time บน production data size
   - Rollback plan
   - Testing บน staging ด้วย production dump
```

---

## SLA/SLO/SLI Framework

### กำหนด SLI

```yaml
# sli-definitions.yaml

slis:
  # Availability SLI
  - name: availability
    description: "% of successful requests"
    query: |
      1 - (
        sum(rate(http_requests_total{status_code=~"5.."}[5m]))
        / sum(rate(http_requests_total[5m]))
      )
    good_events: "non-5xx responses"
    total_events: "all responses"

  # Latency SLI  
  - name: latency_fast
    description: "% of requests completing < 200ms"
    query: |
      sum(rate(http_request_duration_seconds_bucket{le="0.2"}[5m]))
      / sum(rate(http_request_duration_seconds_count[5m]))
    good_events: "requests < 200ms"
    total_events: "all requests"

  - name: latency_acceptable
    description: "% of requests completing < 1s"
    query: |
      sum(rate(http_request_duration_seconds_bucket{le="1.0"}[5m]))
      / sum(rate(http_request_duration_seconds_count[5m]))

  # Freshness SLI (replication lag)
  - name: freshness
    description: "Replication lag < 10s"
    query: |
      pg_replication_replication_delay_seconds < 10
```

### กำหนด SLO

```yaml
# slo-definitions.yaml

slos:
  - service: "Order API"
    slis:
      - name: availability
        target: 99.9      # 99.9% = 43.8 min/month downtime budget
        
      - name: latency_fast
        target: 90        # 90% of requests < 200ms
        
      - name: latency_acceptable  
        target: 99        # 99% of requests < 1s
    
    error_budget:
      window: 30d
      availability_budget: 0.001  # 43.8 minutes/month
      
  - service: "Database Reads"
    slis:
      - name: freshness
        target: 99.9      # 99.9% of time lag < 10s
```

### Error Budget Tracking

```promql
# Error Budget Remaining (%)
# Target: 99.9% availability over 30 days

# Step 1: Calculate error ratio for the window
error_ratio_30d = (
  increase(http_requests_total{status_code=~"5.."}[30d])
  / increase(http_requests_total[30d])
)

# Step 2: Calculate error budget consumed
# Error budget = 1 - 0.999 = 0.001 (0.1%)
budget_consumed = error_ratio_30d / 0.001

# Step 3: Budget remaining
budget_remaining = 1 - budget_consumed

# Step 4: Convert to time
# 30 days = 43,200 minutes
# Budget remaining in minutes
budget_remaining_minutes = budget_remaining * 43200
```

---

## Chaos Engineering Basics

### Chaos Experiments

```bash
# ==================== Network Failures ====================

# Simulate network latency (Linux tc)
# เพิ่ม 100ms latency ที่ network interface
tc qdisc add dev eth0 root netem delay 100ms 10ms 25%

# ลบ latency
tc qdisc del dev eth0 root

# Simulate packet loss 5%
tc qdisc add dev eth0 root netem loss 5%

# ==================== Process Failures ====================

# Kill random worker process
kill -9 $(pgrep -f "node worker" | shuf -n 1)

# Simulate memory pressure
stress-ng --vm 1 --vm-bytes 80% --timeout 60s

# ==================== Disk Failures ====================

# Fill disk (CAREFUL!)
dd if=/dev/zero of=/tmp/fill-disk bs=1M count=1000

# ==================== Database Failures ====================

# Kill PostgreSQL connections
psql -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE pid != pg_backend_pid() LIMIT 5;"

# Simulate slow queries
psql -c "SELECT pg_sleep(10);" &
```

### Chaos Experiments with Chaos Toolkit

```yaml
# chaos-experiment.json
{
  "version": "1.0.0",
  "title": "What happens when Redis is unavailable?",
  "description": "Test application resilience when Redis fails",
  "steady-state-hypothesis": {
    "title": "Application responds normally",
    "probes": [
      {
        "name": "app-responds-ok",
        "type": "probe",
        "provider": {
          "type": "http",
          "url": "http://app:3000/health",
          "expected_status": 200,
          "timeout": 3
        }
      }
    ]
  },
  "method": [
    {
      "name": "stop-redis",
      "type": "action",
      "provider": {
        "type": "process",
        "path": "docker",
        "arguments": "stop redis"
      }
    },
    {
      "name": "wait-for-impact",
      "type": "pauses",
      "value": 30
    }
  ],
  "rollbacks": [
    {
      "name": "start-redis",
      "type": "action",
      "provider": {
        "type": "process",
        "path": "docker",
        "arguments": "start redis"
      }
    }
  ]
}
```

---

## สรุป: Alerting และ Incident Response Checklist

```markdown
## Pre-Incident Preparation

### Alerting Setup
- [ ] Alert rules configured สำหรับ critical paths
- [ ] Alertmanager routing ถูกต้อง
- [ ] Notification channels ทดสอบแล้ว (Slack, PagerDuty)
- [ ] Inhibition rules ป้องกัน alert storm
- [ ] Silences สำหรับ planned maintenance
- [ ] Dead man switch ทำงาน

### Runbooks
- [ ] ทุก critical alert มี runbook
- [ ] Runbooks ทดสอบแล้ว (อย่าเขียนแล้วไม่ใช้!)
- [ ] Runbooks accessible ระหว่าง incident

### On-Call Process
- [ ] On-call rotation กำหนดแล้ว
- [ ] Escalation policy ชัดเจน
- [ ] Contact information up-to-date

## During Incident
- [ ] ประกาศ incident ใน Slack ทันที
- [ ] ระบุ Incident Commander
- [ ] อัปเดต status page
- [ ] Document ทุก action ที่ทำ
- [ ] อัปเดตทุก 15 นาที (P1/P2)

## Post-Incident
- [ ] Post-mortem ภายใน 24-72 ชั่วโมง
- [ ] Action items มี owner และ due date
- [ ] Alert rule ปรับปรุงถ้าจำเป็น
- [ ] ติดตาม action items จนเสร็จ
```

---

*Part 58 เสร็จสมบูรณ์ - ต่อไป Part 59: Performance Tuning PostgreSQL*
