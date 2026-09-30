# Part 102: Complete Monitoring Dashboard Setup

## บทนำ

การ monitoring ที่ดีเป็นกุญแจสำคัญของ production system ในบทนี้เราจะสร้าง complete monitoring stack ด้วย Prometheus, Grafana และ Alertmanager ที่ครอบคลุมทุก layer ของ Database Cluster ตั้งแต่ infrastructure จนถึง application

---

## สารบัญ

1. Monitoring Stack Overview
2. Docker Compose Setup
3. Prometheus Configuration
4. Grafana Dashboards
5. Alertmanager Configuration
6. Alert Rules
7. Loki Log Aggregation
8. Deployment และ Operations

---

## 1. Monitoring Stack Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Monitoring Stack                              │
│                                                                  │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐    │
│  │  Application │   │  PostgreSQL  │   │    Redis         │    │
│  │  :3000/metrics│   │  :5432       │   │    :6379         │    │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────────┘    │
│         │                  │                   │                 │
│         │            ┌─────▼─────┐       ┌─────▼─────┐         │
│         │            │ postgres_ │       │ redis_    │         │
│         │            │ exporter  │       │ exporter  │         │
│         │            │ :9187     │       │ :9121     │         │
│         │            └─────┬─────┘       └─────┬─────┘         │
│         │                  │                   │                 │
│         │      ┌───────────▼───────────────────▼──────────┐    │
│         └──────►          Prometheus :9090                 │    │
│                │          Scrape every 15s                  │    │
│                └───────────────────┬───────────────────────┘    │
│                                    │                             │
│                         ┌──────────▼──────────┐                │
│                         │   Alertmanager :9093  │                │
│                         │  Slack / PagerDuty    │                │
│                         └───────────────────────┘               │
│                                    │                             │
│                         ┌──────────▼──────────┐                │
│                         │   Grafana :3001       │                │
│                         │   Dashboards          │                │
│                         └───────────────────────┘               │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │   Loki :3100  ◄── Promtail  ◄── Application Logs         │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Docker Compose Setup

### 2.1 Full docker-compose.yml

```yaml
# docker-compose.monitoring.yml
version: '3.8'

networks:
  monitoring:
    driver: bridge
  app_network:
    external: true

volumes:
  prometheus_data:
    driver: local
  grafana_data:
    driver: local
  alertmanager_data:
    driver: local
  loki_data:
    driver: local

services:

  # ═══════════════════════════════════════
  # Prometheus - Metrics Collection
  # ═══════════════════════════════════════
  prometheus:
    image: prom/prometheus:v2.47.2
    container_name: prometheus
    restart: unless-stopped
    
    ports:
      - "9090:9090"
    
    volumes:
      - ./monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./monitoring/prometheus/rules:/etc/prometheus/rules:ro
      - prometheus_data:/prometheus
    
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--storage.tsdb.retention.size=20GB'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
      - '--query.timeout=2m'
      - '--query.max-concurrency=20'
    
    networks:
      - monitoring
      - app_network
    
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:9090/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3
    
    labels:
      - "traefik.enable=false"

  # ═══════════════════════════════════════
  # Grafana - Visualization
  # ═══════════════════════════════════════
  grafana:
    image: grafana/grafana:10.2.2
    container_name: grafana
    restart: unless-stopped
    
    ports:
      - "3001:3000"
    
    environment:
      - GF_SECURITY_ADMIN_USER=${GRAFANA_ADMIN_USER:-admin}
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_AUTH_ANONYMOUS_ENABLED=false
      - GF_SERVER_DOMAIN=${GRAFANA_DOMAIN:-localhost}
      - GF_SERVER_ROOT_URL=https://${GRAFANA_DOMAIN:-localhost}:3001
      - GF_SMTP_ENABLED=true
      - GF_SMTP_HOST=${SMTP_HOST}
      - GF_SMTP_USER=${SMTP_USER}
      - GF_SMTP_PASSWORD=${SMTP_PASSWORD}
      - GF_SMTP_FROM_ADDRESS=${SMTP_FROM}
      - GF_RENDERING_SERVER_URL=http://renderer:8081/render
      - GF_RENDERING_CALLBACK_URL=http://grafana:3000/
      - GF_UNIFIED_ALERTING_ENABLED=true
      - GF_ALERTING_ENABLED=false
      - GF_FEATURE_TOGGLES_ENABLE=traceqlEditor
      - GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-worldmap-panel,grafana-piechart-panel
    
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning:ro
      - ./monitoring/grafana/dashboards:/var/lib/grafana/dashboards:ro
    
    networks:
      - monitoring
    
    depends_on:
      prometheus:
        condition: service_healthy
    
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:3000/api/health | grep -q 'ok'"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ═══════════════════════════════════════
  # Grafana Image Renderer
  # ═══════════════════════════════════════
  renderer:
    image: grafana/grafana-image-renderer:3.9.2
    container_name: grafana-renderer
    restart: unless-stopped
    
    environment:
      ENABLE_METRICS: "true"
      HTTP_PORT: "8081"
    
    networks:
      - monitoring

  # ═══════════════════════════════════════
  # Alertmanager
  # ═══════════════════════════════════════
  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    restart: unless-stopped
    
    ports:
      - "9093:9093"
    
    volumes:
      - ./monitoring/alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager_data:/alertmanager
    
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--cluster.listen-address=0.0.0.0:9094'
      - '--web.external-url=https://${ALERTMANAGER_DOMAIN:-localhost}:9093'
    
    networks:
      - monitoring
    
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:9093/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ═══════════════════════════════════════
  # postgres_exporter
  # ═══════════════════════════════════════
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:v0.15.0
    container_name: postgres-exporter
    restart: unless-stopped
    
    ports:
      - "9187:9187"
    
    environment:
      DATA_SOURCE_NAME: "postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:5432/${POSTGRES_DB}?sslmode=require"
      PG_EXPORTER_DISABLE_DEFAULT_METRICS: "false"
      PG_EXPORTER_DISABLE_SETTINGS_METRICS: "false"
      PG_EXPORTER_AUTO_DISCOVER_DATABASES: "true"
    
    volumes:
      - ./monitoring/postgres-exporter/queries.yaml:/etc/postgres_exporter/queries.yaml:ro
    
    command:
      - '--extend.query-path=/etc/postgres_exporter/queries.yaml'
      - '--log.level=warn'
    
    networks:
      - monitoring
      - app_network
    
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:9187/metrics"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ═══════════════════════════════════════
  # redis_exporter
  # ═══════════════════════════════════════
  redis-exporter:
    image: oliver006/redis_exporter:v1.55.0
    container_name: redis-exporter
    restart: unless-stopped
    
    ports:
      - "9121:9121"
    
    environment:
      REDIS_ADDR: "redis://${REDIS_HOST}:6379"
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      REDIS_EXPORTER_LOG_FORMAT: json
    
    networks:
      - monitoring
      - app_network
    
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:9121/metrics"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ═══════════════════════════════════════
  # node_exporter
  # ═══════════════════════════════════════
  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    restart: unless-stopped
    
    ports:
      - "9100:9100"
    
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    
    networks:
      - monitoring
    
    pid: host

  # ═══════════════════════════════════════
  # blackbox_exporter - Endpoint monitoring
  # ═══════════════════════════════════════
  blackbox-exporter:
    image: prom/blackbox-exporter:v0.24.0
    container_name: blackbox-exporter
    restart: unless-stopped
    
    ports:
      - "9115:9115"
    
    volumes:
      - ./monitoring/blackbox/blackbox.yml:/etc/blackbox_exporter/config.yml:ro
    
    networks:
      - monitoring
      - app_network

  # ═══════════════════════════════════════
  # Loki - Log Aggregation
  # ═══════════════════════════════════════
  loki:
    image: grafana/loki:2.9.2
    container_name: loki
    restart: unless-stopped
    
    ports:
      - "3100:3100"
    
    volumes:
      - ./monitoring/loki/loki.yml:/etc/loki/local-config.yaml:ro
      - loki_data:/loki
    
    command: -config.file=/etc/loki/local-config.yaml
    
    networks:
      - monitoring
    
    healthcheck:
      test: ["CMD-SHELL", "wget -q --spider http://localhost:3100/ready"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ═══════════════════════════════════════
  # Promtail - Log Shipper
  # ═══════════════════════════════════════
  promtail:
    image: grafana/promtail:2.9.2
    container_name: promtail
    restart: unless-stopped
    
    volumes:
      - ./monitoring/promtail/promtail.yml:/etc/promtail/config.yml:ro
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    
    command: -config.file=/etc/promtail/config.yml
    
    networks:
      - monitoring
    
    depends_on:
      - loki
```

---

## 3. Prometheus Configuration

### 3.1 prometheus.yml

```yaml
# monitoring/prometheus/prometheus.yml

global:
  scrape_interval: 15s
  evaluation_interval: 15s
  scrape_timeout: 10s
  
  external_labels:
    cluster: 'myapp-production'
    region: 'ap-southeast-1'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
      timeout: 10s
      api_version: v2

# Load alert rules
rule_files:
  - "/etc/prometheus/rules/recording_rules.yml"
  - "/etc/prometheus/rules/alert_rules_critical.yml"
  - "/etc/prometheus/rules/alert_rules_warning.yml"
  - "/etc/prometheus/rules/alert_rules_db.yml"

# Scrape configurations
scrape_configs:

  # ─────────────────────────
  # Prometheus self-monitoring
  # ─────────────────────────
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
    
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: 'prometheus'

  # ─────────────────────────
  # Node Exporter
  # ─────────────────────────
  - job_name: 'node'
    static_configs:
      - targets:
        - 'node-exporter:9100'
        labels:
          host: 'app-server-01'
    
    relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):\d+'
        target_label: instance

  # ─────────────────────────
  # PostgreSQL
  # ─────────────────────────
  - job_name: 'postgresql'
    static_configs:
      - targets:
        - 'postgres-exporter:9187'
        labels:
          database: 'myapp'
          environment: 'production'
    
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'pg_.*'
        action: keep

  # ─────────────────────────
  # Redis
  # ─────────────────────────
  - job_name: 'redis'
    static_configs:
      - targets:
        - 'redis-exporter:9121'
        labels:
          instance: 'redis-cluster'
          environment: 'production'

  # ─────────────────────────
  # Application (Node.js)
  # ─────────────────────────
  - job_name: 'nodejs-app'
    static_configs:
      - targets:
        - 'app:3000'
        labels:
          service: 'api'
          version: '2.0.0'
    
    metrics_path: '/metrics'
    
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance

  # ─────────────────────────
  # Blackbox Exporter
  # ─────────────────────────
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    
    static_configs:
      - targets:
        - https://api.myapp.com/health
        - https://api.myapp.com/api/v1/status
        - https://www.myapp.com
        labels:
          type: 'http'
    
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

  - job_name: 'blackbox-tcp'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    
    static_configs:
      - targets:
        - 'postgres:5432'
        - 'redis:6379'
        labels:
          type: 'tcp'
    
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

  # ─────────────────────────
  # Alertmanager
  # ─────────────────────────
  - job_name: 'alertmanager'
    static_configs:
      - targets: ['alertmanager:9093']
    
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: 'alertmanager'

  # ─────────────────────────
  # Grafana
  # ─────────────────────────
  - job_name: 'grafana'
    static_configs:
      - targets: ['grafana:3000']
    
    metrics_path: '/metrics'
```

### 3.2 Recording Rules (Pre-aggregated Metrics)

```yaml
# monitoring/prometheus/rules/recording_rules.yml

groups:
  - name: postgresql_recording_rules
    interval: 1m
    rules:
    
    # Cache hit ratio
    - record: pg:blks_hit:ratio5m
      expr: |
        rate(pg_stat_database_blks_hit[5m])
        /
        (rate(pg_stat_database_blks_hit[5m]) + rate(pg_stat_database_blks_read[5m]) + 0.001)
    
    # Transaction rate
    - record: pg:xact_commit:rate5m
      expr: rate(pg_stat_database_xact_commit[5m])
    
    - record: pg:xact_rollback:rate5m
      expr: rate(pg_stat_database_xact_rollback[5m])
    
    # Connection usage
    - record: pg:connections:ratio
      expr: |
        pg_stat_activity_count
        /
        pg_settings_max_connections
    
    # Replication lag
    - record: pg:replication_lag:seconds
      expr: |
        pg_replication_lag
    
    # Query rate
    - record: pg:queries:rate5m
      expr: |
        rate(pg_stat_statements_calls_total[5m])
    
    # Lock wait
    - record: pg:locks:waiting
      expr: |
        pg_locks_count{mode="ExclusiveLock", granted="false"}

  - name: redis_recording_rules
    interval: 1m
    rules:
    
    # Hit rate
    - record: redis:hits:ratio5m
      expr: |
        rate(redis_keyspace_hits_total[5m])
        /
        (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]) + 0.001)
    
    # Memory usage ratio
    - record: redis:memory:ratio
      expr: |
        redis_memory_used_bytes
        /
        redis_memory_max_bytes
    
    # Operations per second
    - record: redis:ops:rate5m
      expr: rate(redis_commands_processed_total[5m])
    
    # Connected clients ratio
    - record: redis:clients:ratio
      expr: |
        redis_connected_clients
        /
        redis_config_maxclients
    
    # Eviction rate
    - record: redis:evictions:rate5m
      expr: rate(redis_evicted_keys_total[5m])

  - name: nodejs_recording_rules
    interval: 1m
    rules:
    
    # Request rate
    - record: http:requests:rate5m
      expr: |
        sum(rate(http_requests_total[5m])) by (method, route, status)
    
    # Error rate
    - record: http:errors:rate5m
      expr: |
        sum(rate(http_requests_total{status=~"5.."}[5m])) by (route)
    
    # Error ratio
    - record: http:error:ratio5m
      expr: |
        sum(rate(http_requests_total{status=~"5.."}[5m]))
        /
        sum(rate(http_requests_total[5m]))
    
    # P95 latency
    - record: http:latency:p95
      expr: |
        histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route))
    
    # P99 latency
    - record: http:latency:p99
      expr: |
        histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route))
```

---

## 4. Alert Rules

### 4.1 Critical Alerts

```yaml
# monitoring/prometheus/rules/alert_rules_critical.yml

groups:
  - name: critical_database_alerts
    rules:
    
    # Database Down
    - alert: PostgreSQLDown
      expr: pg_up == 0
      for: 1m
      labels:
        severity: critical
        team: platform
        pagerduty: "true"
      annotations:
        summary: "PostgreSQL is DOWN"
        description: "PostgreSQL instance {{ $labels.instance }} has been down for more than 1 minute."
        runbook_url: "https://wiki.myapp.com/runbooks/postgresql-down"
    
    # Replication Broken
    - alert: PostgreSQLReplicationBroken
      expr: pg_replication_lag > 300  # 5 minutes
      for: 5m
      labels:
        severity: critical
        team: platform
      annotations:
        summary: "PostgreSQL replication lag critical"
        description: "Replication lag is {{ $value }}s on {{ $labels.instance }}. Threshold is 300s."
        runbook_url: "https://wiki.myapp.com/runbooks/replication-lag"
    
    # High Error Rate
    - alert: HighErrorRate
      expr: http:error:ratio5m > 0.05  # 5%
      for: 5m
      labels:
        severity: critical
        team: backend
        pagerduty: "true"
      annotations:
        summary: "High HTTP error rate: {{ $value | humanizePercentage }}"
        description: "Error rate is {{ $value | humanizePercentage }} for the last 5 minutes."
    
    # Redis Down
    - alert: RedisDown
      expr: redis_up == 0
      for: 1m
      labels:
        severity: critical
        team: platform
        pagerduty: "true"
      annotations:
        summary: "Redis is DOWN"
        description: "Redis instance {{ $labels.instance }} is not responding."
    
    # Endpoint Down
    - alert: EndpointDown
      expr: probe_success == 0
      for: 3m
      labels:
        severity: critical
        team: backend
        pagerduty: "true"
      annotations:
        summary: "Endpoint {{ $labels.instance }} is DOWN"
        description: "HTTP endpoint {{ $labels.instance }} has been unreachable for 3 minutes."
    
    # Disk Space Critical
    - alert: DiskSpaceCritical
      expr: |
        (node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.lxcfs"} /
         node_filesystem_size_bytes{fstype!~"tmpfs|fuse.lxcfs"}) * 100 < 10
      for: 5m
      labels:
        severity: critical
        team: platform
      annotations:
        summary: "Disk space critically low: {{ $value | humanize }}% free"
        description: "Disk {{ $labels.device }} on {{ $labels.instance }} has only {{ $value | humanize }}% free space."

  - name: critical_application_alerts
    rules:
    
    # Application Response Time Critical
    - alert: AppResponseTimeCritical
      expr: |
        histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) > 5
      for: 10m
      labels:
        severity: critical
        team: backend
      annotations:
        summary: "P95 response time > 5 seconds"
        description: "P95 response time is {{ $value }}s which exceeds SLA."
    
    # Too Many Active Connections
    - alert: PostgreSQLTooManyConnections
      expr: pg:connections:ratio > 0.9
      for: 5m
      labels:
        severity: critical
        team: backend
      annotations:
        summary: "PostgreSQL connections at {{ $value | humanizePercentage }} capacity"
        description: "PostgreSQL connection pool is nearly full. Consider connection pooling."
```

### 4.2 Warning Alerts

```yaml
# monitoring/prometheus/rules/alert_rules_warning.yml

groups:
  - name: warning_database_alerts
    rules:
    
    # High CPU
    - alert: HighCPUUsage
      expr: |
        100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
      for: 15m
      labels:
        severity: warning
        team: platform
      annotations:
        summary: "High CPU usage: {{ $value | humanize }}%"
        description: "CPU usage on {{ $labels.instance }} is {{ $value | humanize }}% for 15 minutes."
    
    # High Memory
    - alert: HighMemoryUsage
      expr: |
        (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
      for: 10m
      labels:
        severity: warning
        team: platform
      annotations:
        summary: "High memory usage: {{ $value | humanize }}%"
        description: "Memory usage on {{ $labels.instance }} is {{ $value | humanize }}%."
    
    # Slow Queries
    - alert: PostgreSQLSlowQueries
      expr: |
        rate(pg_stat_statements_mean_exec_time_seconds[5m]) > 1
      for: 10m
      labels:
        severity: warning
        team: backend
      annotations:
        summary: "PostgreSQL slow queries detected"
        description: "Average query execution time is {{ $value }}s."
    
    # Low Cache Hit Ratio
    - alert: PostgreSQLLowCacheHitRatio
      expr: pg:blks_hit:ratio5m < 0.95
      for: 15m
      labels:
        severity: warning
        team: backend
      annotations:
        summary: "PostgreSQL cache hit ratio low: {{ $value | humanizePercentage }}"
        description: "Cache hit ratio {{ $value | humanizePercentage }} is below 95% threshold."
    
    # Redis High Memory
    - alert: RedisHighMemoryUsage
      expr: redis:memory:ratio > 0.80
      for: 10m
      labels:
        severity: warning
        team: platform
      annotations:
        summary: "Redis memory usage: {{ $value | humanizePercentage }}"
        description: "Redis memory is {{ $value | humanizePercentage }} full."
    
    # Redis Evictions
    - alert: RedisHighEvictions
      expr: redis:evictions:rate5m > 100
      for: 5m
      labels:
        severity: warning
        team: backend
      annotations:
        summary: "Redis evicting {{ $value | humanize }} keys/second"
        description: "High eviction rate may indicate insufficient memory."
    
    # Replication Lag Warning
    - alert: PostgreSQLReplicationLagWarning
      expr: pg_replication_lag > 60
      for: 5m
      labels:
        severity: warning
        team: platform
      annotations:
        summary: "PostgreSQL replication lag: {{ $value }}s"
        description: "Replication is falling behind on {{ $labels.instance }}."
    
    # Disk Space Warning
    - alert: DiskSpaceWarning
      expr: |
        (node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.lxcfs"} /
         node_filesystem_size_bytes{fstype!~"tmpfs|fuse.lxcfs"}) * 100 < 20
      for: 10m
      labels:
        severity: warning
        team: platform
      annotations:
        summary: "Disk space warning: {{ $value | humanize }}% free"
```

---

## 5. Alertmanager Configuration

### 5.1 alertmanager.yml

```yaml
# monitoring/alertmanager/alertmanager.yml

global:
  resolve_timeout: 5m
  smtp_smarthost: '${SMTP_HOST}:587'
  smtp_from: 'alerts@myapp.com'
  smtp_auth_username: '${SMTP_USER}'
  smtp_auth_password: '${SMTP_PASSWORD}'
  smtp_require_tls: true
  slack_api_url: '${SLACK_WEBHOOK_URL}'
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

# Inhibit rules
inhibit_rules:
  # Inhibit warnings when critical is firing for same instance
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['instance', 'job']
  
  # If database is down, suppress all db-related warnings
  - source_match:
      alertname: 'PostgreSQLDown'
    target_match_re:
      alertname: 'PostgreSQL.*'
    equal: ['instance']

# Templates
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Routes
route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'slack-notifications'
  
  routes:
    # Critical alerts -> PagerDuty + Slack
    - match:
        severity: critical
      receiver: 'critical-alerts'
      group_wait: 10s
      repeat_interval: 1h
      continue: true
    
    # PagerDuty for explicitly marked alerts
    - match:
        pagerduty: "true"
      receiver: 'pagerduty-critical'
      group_wait: 10s
      repeat_interval: 30m
    
    # Platform team alerts
    - match:
        team: platform
      receiver: 'platform-team-slack'
      continue: true
    
    # Backend team alerts
    - match:
        team: backend
      receiver: 'backend-team-slack'
      continue: true
    
    # Warning alerts - Slack only
    - match:
        severity: warning
      receiver: 'slack-warnings'
      group_interval: 30m
      repeat_interval: 12h

# Receivers
receivers:

  - name: 'slack-notifications'
    slack_configs:
      - channel: '#alerts-all'
        username: 'Prometheus'
        icon_emoji: ':bell:'
        send_resolved: true
        title: '{{ template "slack.default.title" . }}'
        text: '{{ template "slack.default.text" . }}'
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
        actions:
          - type: button
            text: 'View in Grafana'
            url: '{{ (index .Alerts 0).GeneratorURL }}'
          - type: button
            text: 'Silence'
            url: '{{ template "__alertmanagerURL" . }}/#/silences/new?filter=%7B{{ range .CommonLabels.SortedPairs }}{{ .Name }}%3D%22{{ .Value }}%22%2C{{ end }}%7D'

  - name: 'critical-alerts'
    slack_configs:
      - channel: '#alerts-critical'
        username: 'Prometheus CRITICAL'
        icon_emoji: ':fire:'
        send_resolved: true
        title: ':fire: CRITICAL: {{ .GroupLabels.alertname }}'
        text: |
          *Severity:* {{ .CommonLabels.severity }}
          *Environment:* {{ .CommonLabels.cluster }}
          {{ range .Alerts }}
          *Alert:* {{ .Labels.alertname }}
          *Description:* {{ .Annotations.description }}
          *Runbook:* {{ .Annotations.runbook_url }}
          {{ end }}
        color: 'danger'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: '${PAGERDUTY_ROUTING_KEY}'
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'
        severity: 'critical'
        details:
          cluster: '{{ .CommonLabels.cluster }}'
          environment: 'production'
          runbook: '{{ (index .Alerts 0).Annotations.runbook_url }}'
          description: '{{ .CommonAnnotations.description }}'

  - name: 'platform-team-slack'
    slack_configs:
      - channel: '#platform-alerts'
        username: 'Prometheus'
        icon_emoji: ':warning:'
        send_resolved: true
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'backend-team-slack'
    slack_configs:
      - channel: '#backend-alerts'
        username: 'Prometheus'
        icon_emoji: ':warning:'
        send_resolved: true
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#alerts-warnings'
        username: 'Prometheus'
        icon_emoji: ':warning:'
        send_resolved: true
        title: ':warning: {{ .GroupLabels.alertname }}'
        text: '{{ .CommonAnnotations.description }}'
        color: 'warning'

  - name: 'email-oncall'
    email_configs:
      - to: 'oncall@myapp.com'
        subject: '[CRITICAL] {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Labels.alertname }}
          Severity: {{ .Labels.severity }}
          Description: {{ .Annotations.description }}
          Started: {{ .StartsAt.Format "2006-01-02 15:04:05 UTC" }}
          {{ end }}
```

---

## 6. Grafana Dashboards

### 6.1 Grafana Provisioning

```yaml
# monitoring/grafana/provisioning/datasources/datasources.yml

apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
    jsonData:
      httpMethod: POST
      exemplarTraceIdDestinations:
        - name: traceID
          datasourceUid: tempo
      timeInterval: "15s"
  
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false
    jsonData:
      derivedFields:
        - matcherRegex: "traceId=(\\w+)"
          name: TraceID
          url: "${__value.raw}"
          datasourceUid: tempo
  
  - name: AlertManager
    type: alertmanager
    access: proxy
    url: http://alertmanager:9093
    editable: false
    jsonData:
      implementation: prometheus
```

```yaml
# monitoring/grafana/provisioning/dashboards/dashboards.yml

apiVersion: 1

providers:
  - name: 'Default'
    orgId: 1
    folder: 'Database Cluster'
    type: file
    disableDeletion: false
    editable: true
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
```

### 6.2 PostgreSQL Dashboard JSON (ตัวอย่าง panels สำคัญ)

```json
{
  "title": "PostgreSQL Overview",
  "uid": "postgres-overview",
  "tags": ["postgresql", "database"],
  "timezone": "browser",
  "refresh": "30s",
  "time": {
    "from": "now-3h",
    "to": "now"
  },
  "panels": [
    {
      "id": 1,
      "type": "stat",
      "title": "Database Status",
      "gridPos": { "x": 0, "y": 0, "w": 4, "h": 4 },
      "targets": [
        {
          "expr": "pg_up",
          "legendFormat": "{{instance}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "mappings": [
            { "options": { "0": { "text": "DOWN", "color": "red" }, "1": { "text": "UP", "color": "green" } }, "type": "value" }
          ],
          "thresholds": {
            "steps": [
              { "color": "red", "value": 0 },
              { "color": "green", "value": 1 }
            ]
          }
        }
      }
    },
    {
      "id": 2,
      "type": "gauge",
      "title": "Active Connections",
      "gridPos": { "x": 4, "y": 0, "w": 4, "h": 4 },
      "targets": [
        {
          "expr": "pg_stat_activity_count{state='active'}",
          "legendFormat": "Active"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "min": 0,
          "max": 500,
          "thresholds": {
            "steps": [
              { "color": "green", "value": 0 },
              { "color": "yellow", "value": 300 },
              { "color": "red", "value": 450 }
            ]
          }
        }
      }
    },
    {
      "id": 3,
      "type": "timeseries",
      "title": "Cache Hit Ratio",
      "gridPos": { "x": 8, "y": 0, "w": 8, "h": 4 },
      "targets": [
        {
          "expr": "pg:blks_hit:ratio5m * 100",
          "legendFormat": "Cache Hit %"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "min": 0,
          "max": 100,
          "thresholds": {
            "steps": [
              { "color": "red", "value": 0 },
              { "color": "yellow", "value": 90 },
              { "color": "green", "value": 95 }
            ]
          }
        }
      }
    },
    {
      "id": 4,
      "type": "timeseries",
      "title": "Transaction Rate",
      "gridPos": { "x": 0, "y": 4, "w": 12, "h": 6 },
      "targets": [
        {
          "expr": "pg:xact_commit:rate5m",
          "legendFormat": "Commits/s"
        },
        {
          "expr": "pg:xact_rollback:rate5m",
          "legendFormat": "Rollbacks/s"
        }
      ],
      "fieldConfig": {
        "defaults": { "unit": "ops" }
      }
    },
    {
      "id": 5,
      "type": "timeseries",
      "title": "Replication Lag",
      "gridPos": { "x": 12, "y": 4, "w": 12, "h": 6 },
      "targets": [
        {
          "expr": "pg_replication_lag",
          "legendFormat": "{{replica}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "s",
          "thresholds": {
            "steps": [
              { "color": "green", "value": 0 },
              { "color": "yellow", "value": 30 },
              { "color": "red", "value": 60 }
            ]
          }
        }
      }
    },
    {
      "id": 6,
      "type": "table",
      "title": "Table Sizes",
      "gridPos": { "x": 0, "y": 10, "w": 12, "h": 8 },
      "targets": [
        {
          "expr": "topk(10, pg_stat_user_tables_n_live_tup)",
          "legendFormat": "{{relname}}",
          "instant": true
        }
      ]
    },
    {
      "id": 7,
      "type": "timeseries",
      "title": "Lock Waits",
      "gridPos": { "x": 12, "y": 10, "w": 12, "h": 8 },
      "targets": [
        {
          "expr": "pg_locks_count{granted='false'}",
          "legendFormat": "Waiting Locks - {{mode}}"
        }
      ]
    }
  ]
}
```

### 6.3 Redis Dashboard JSON

```json
{
  "title": "Redis Overview",
  "uid": "redis-overview",
  "tags": ["redis", "cache"],
  "panels": [
    {
      "id": 1,
      "type": "stat",
      "title": "Redis Status",
      "targets": [{ "expr": "redis_up", "legendFormat": "Status" }]
    },
    {
      "id": 2,
      "type": "gauge",
      "title": "Memory Usage",
      "targets": [
        {
          "expr": "redis:memory:ratio * 100",
          "legendFormat": "Memory %"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "thresholds": {
            "steps": [
              { "color": "green", "value": 0 },
              { "color": "yellow", "value": 70 },
              { "color": "red", "value": 85 }
            ]
          }
        }
      }
    },
    {
      "id": 3,
      "type": "timeseries",
      "title": "Hit Rate",
      "targets": [
        {
          "expr": "redis:hits:ratio5m * 100",
          "legendFormat": "Hit Rate %"
        }
      ]
    },
    {
      "id": 4,
      "type": "timeseries",
      "title": "Operations Per Second",
      "targets": [
        {
          "expr": "redis:ops:rate5m",
          "legendFormat": "OPS"
        }
      ]
    },
    {
      "id": 5,
      "type": "timeseries",
      "title": "Connected Clients",
      "targets": [
        {
          "expr": "redis_connected_clients",
          "legendFormat": "Clients"
        }
      ]
    },
    {
      "id": 6,
      "type": "timeseries",
      "title": "Evictions",
      "targets": [
        {
          "expr": "redis:evictions:rate5m",
          "legendFormat": "Evictions/s"
        }
      ]
    },
    {
      "id": 7,
      "type": "timeseries",
      "title": "Network I/O",
      "targets": [
        {
          "expr": "rate(redis_net_input_bytes_total[5m])",
          "legendFormat": "Input bytes/s"
        },
        {
          "expr": "rate(redis_net_output_bytes_total[5m])",
          "legendFormat": "Output bytes/s"
        }
      ],
      "fieldConfig": {
        "defaults": { "unit": "bytes" }
      }
    }
  ]
}
```

---

## 7. Loki Log Aggregation

### 7.1 loki.yml

```yaml
# monitoring/loki/loki.yml

auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  log_level: warn

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2023-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://alertmanager:9093

ingester:
  wal:
    enabled: true
    dir: /loki/wal
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 1h
  max_chunk_age: 1h
  chunk_target_size: 1048576
  chunk_retain_period: 30s

limits_config:
  retention_period: 720h  # 30 days
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20
  max_query_series: 5000
  max_query_parallelism: 32

compactor:
  working_directory: /loki/compactor
  shared_store: filesystem
  retention_enabled: true
  retention_delete_delay: 2h
  retention_delete_worker_count: 150

chunk_store_config:
  max_look_back_period: 720h

table_manager:
  retention_deletes_enabled: true
  retention_period: 720h

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100
```

### 7.2 promtail.yml

```yaml
# monitoring/promtail/promtail.yml

server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push
    batchwait: 1s
    batchsize: 1048576
    timeout: 10s

scrape_configs:

  # Application logs (Docker containers)
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'stream'
      - source_labels: ['__meta_docker_container_label_com_docker_compose_service']
        target_label: 'service'
    
    pipeline_stages:
      # Parse JSON logs
      - json:
          expressions:
            level: level
            message: message
            timestamp: timestamp
            requestId: requestId
            userId: userId
            duration: duration
            statusCode: statusCode
            path: path
            method: method
      
      # Extract timestamp
      - timestamp:
          source: timestamp
          format: RFC3339
          fallback_formats:
            - "2006-01-02T15:04:05.000Z07:00"
      
      # Add labels from JSON fields
      - labels:
          level:
          service:
      
      # Set log level
      - match:
          selector: '{level="error"}'
          stages:
            - labels:
                severity: error
      
      - match:
          selector: '{level="warn"}'
          stages:
            - labels:
                severity: warning
      
      # Drop debug logs in production
      - match:
          selector: '{level="debug"}'
          action: drop

  # System logs
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          host: app-server
          __path__: /var/log/*.log
    
    pipeline_stages:
      - multiline:
          firstline: '^\d{4}-\d{2}-\d{2}'
          max_wait_time: 3s

  # PostgreSQL logs
  - job_name: postgresql
    static_configs:
      - targets:
          - localhost
        labels:
          job: postgresql
          service: postgresql
          __path__: /var/log/postgresql/*.log
    
    pipeline_stages:
      - regex:
          expression: '(?P<timestamp>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}.\d{3} \w+) \[(?P<pid>\d+)\] (?P<level>\w+): (?P<message>.*)'
      - labels:
          level: pid
      - timestamp:
          source: timestamp
          format: "2006-01-02 15:04:05.000 MST"
```

### 7.3 Blackbox Exporter Configuration

```yaml
# monitoring/blackbox/blackbox.yml

modules:
  http_2xx:
    prober: http
    timeout: 10s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200, 201, 204]
      method: GET
      follow_redirects: true
      preferred_ip_protocol: "ip4"
      tls_config:
        insecure_skip_verify: false
  
  http_post_2xx:
    prober: http
    timeout: 10s
    http:
      method: POST
      valid_status_codes: [200, 201]
      headers:
        Content-Type: application/json
      body: '{}'
  
  tcp_connect:
    prober: tcp
    timeout: 5s
  
  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: "ip4"
```

---

## 8. Grafana Annotations: Deployment Markers

```bash
#!/bin/bash
# deploy-marker.sh

GRAFANA_URL="http://grafana:3001"
GRAFANA_API_KEY="${GRAFANA_API_KEY}"
DEPLOYMENT_VERSION="${1:-unknown}"
DEPLOYED_BY="${2:-CI/CD}"

# Create annotation in Grafana
curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${GRAFANA_API_KEY}" \
  "${GRAFANA_URL}/api/annotations" \
  -d "{
    \"text\": \"Deployment: ${DEPLOYMENT_VERSION} by ${DEPLOYED_BY}\",
    \"tags\": [\"deployment\", \"${DEPLOYMENT_VERSION}\"],
    \"time\": $(date +%s000),
    \"timeEnd\": $(date +%s000),
    \"isRegion\": false
  }"

echo "Deployment annotation created for version ${DEPLOYMENT_VERSION}"
```

---

## 9. Custom postgres_exporter Queries

```yaml
# monitoring/postgres-exporter/queries.yaml

pg_replication:
  query: |
    SELECT
      CASE WHEN pg_is_in_recovery() THEN 1 ELSE 0 END AS is_replica,
      CASE WHEN NOT pg_is_in_recovery() AND EXISTS (SELECT 1 FROM pg_stat_replication) 
           THEN 1 ELSE 0 END AS has_replicas,
      COALESCE(EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())), 0) AS lag_seconds
  metrics:
    - is_replica:
        usage: "GAUGE"
        description: "Whether this is a replica"
    - has_replicas:
        usage: "GAUGE"
        description: "Whether primary has replicas"
    - lag_seconds:
        usage: "GAUGE"
        description: "Replication lag in seconds"

pg_table_bloat:
  query: |
    SELECT
      schemaname || '.' || tablename AS table_name,
      ROUND(CASE WHEN otta=0 THEN 0.0
                 ELSE sml.relpages/otta::NUMERIC END, 1) AS bloat_ratio,
      CASE WHEN relpages < otta THEN 0
           ELSE (relpages-otta)::BIGINT * bs END AS bloat_size
    FROM (
      SELECT
        schemaname, tablename, bs,
        CEIL((reltuples*((datahdr+ma-(CASE WHEN datahdr%ma=0 THEN ma ELSE datahdr%ma END))+nullhdr2+4))/(bs-20::FLOAT)) AS otta,
        relpages
      FROM (
        SELECT
          ma, bs, schemaname, tablename,
          (datawidth+(hdr+ma-(CASE WHEN hdr%ma=0 THEN ma ELSE hdr%ma END)))::NUMERIC AS datahdr,
          (maxfracsum*(nullhdr+ma-(CASE WHEN nullhdr%ma=0 THEN ma ELSE nullhdr%ma END))) AS nullhdr2,
          relpages
        FROM pg_class c
        JOIN pg_namespace n ON n.oid = c.relnamespace
        CROSS JOIN (
          SELECT
            current_setting('block_size')::NUMERIC AS bs,
            23 AS hdr, 8 AS ma
        ) AS constants
        CROSS JOIN (
          SELECT
            SUM((1-null_frac)*avg_width) AS datawidth,
            MAX(null_frac) AS maxfracsum,
            (SELECT 1+COUNT(*)/8 FROM pg_stats WHERE tablename=s.tablename) AS nullhdr
          FROM pg_stats s
          WHERE s.tablename = c.relname
          GROUP BY s.tablename
        ) AS dc
        WHERE c.relkind = 'r' AND n.nspname NOT IN ('pg_catalog', 'information_schema')
      ) AS foo
    ) AS sml
    WHERE relpages > 100
    ORDER BY bloat_size DESC
    LIMIT 20
  metrics:
    - table_name:
        usage: "LABEL"
        description: "Table name with schema"
    - bloat_ratio:
        usage: "GAUGE"
        description: "Table bloat ratio"
    - bloat_size:
        usage: "GAUGE"
        description: "Bloat size in bytes"

pg_long_running_queries:
  query: |
    SELECT
      COUNT(*) FILTER (WHERE state = 'active' AND query_start < now() - '30 seconds'::INTERVAL) AS slow_queries_30s,
      COUNT(*) FILTER (WHERE state = 'active' AND query_start < now() - '1 minute'::INTERVAL) AS slow_queries_1m,
      COUNT(*) FILTER (WHERE state = 'active' AND query_start < now() - '5 minutes'::INTERVAL) AS slow_queries_5m,
      COUNT(*) FILTER (WHERE wait_event_type = 'Lock') AS lock_waits
    FROM pg_stat_activity
    WHERE query NOT LIKE '%pg_stat_activity%'
  metrics:
    - slow_queries_30s:
        usage: "GAUGE"
        description: "Queries running > 30 seconds"
    - slow_queries_1m:
        usage: "GAUGE"
        description: "Queries running > 1 minute"
    - slow_queries_5m:
        usage: "GAUGE"
        description: "Queries running > 5 minutes"
    - lock_waits:
        usage: "GAUGE"
        description: "Sessions waiting for locks"
```

---

## 10. Operations Guide

### 10.1 Deployment Script

```bash
#!/bin/bash
# deploy-monitoring.sh

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
MONITORING_DIR="${SCRIPT_DIR}/monitoring"

echo "=== Deploying Monitoring Stack ==="

# Check prerequisites
command -v docker >/dev/null 2>&1 || { echo "Docker is required"; exit 1; }
command -v docker-compose >/dev/null 2>&1 || { echo "Docker Compose is required"; exit 1; }

# Check environment variables
: "${GRAFANA_ADMIN_PASSWORD:?GRAFANA_ADMIN_PASSWORD is required}"
: "${POSTGRES_PASSWORD:?POSTGRES_PASSWORD is required}"
: "${REDIS_PASSWORD:?REDIS_PASSWORD is required}"
: "${SLACK_WEBHOOK_URL:?SLACK_WEBHOOK_URL is required}"

# Create directories
mkdir -p ${MONITORING_DIR}/{prometheus/{rules},grafana/{provisioning/{datasources,dashboards},dashboards},alertmanager,loki,promtail,blackbox,postgres-exporter}

# Deploy
docker-compose -f docker-compose.monitoring.yml up -d

# Wait for services
echo "Waiting for services to start..."
sleep 30

# Check health
services=(prometheus grafana alertmanager postgres-exporter redis-exporter loki)
for service in "${services[@]}"; do
  status=$(docker-compose -f docker-compose.monitoring.yml ps ${service} | grep -o "Up\|Exit\|running" | head -1)
  echo "${service}: ${status}"
done

echo ""
echo "=== Monitoring Stack Deployed ==="
echo "Grafana:      http://localhost:3001"
echo "Prometheus:   http://localhost:9090"
echo "Alertmanager: http://localhost:9093"
```

### 10.2 Useful Prometheus Queries

```promql
# Top 10 tables by size
topk(10, pg_stat_user_tables_n_live_tup)

# Connections by state
pg_stat_activity_count by (state)

# Most frequent queries (requires pg_stat_statements)
topk(10, rate(pg_stat_statements_calls_total[5m]))

# Slow queries per database
pg_stat_statements_mean_exec_time_seconds > 0.1

# Redis key distribution
redis_db_keys by (db)

# Application error rate by route
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) by (route) * 100

# System load
node_load1 / count(node_cpu_seconds_total{mode="idle"}) without (cpu) * 100

# Disk write throughput
rate(node_disk_written_bytes_total[5m]) by (device)
```

---

## สรุป

Monitoring stack ที่สมบูรณ์ประกอบด้วย:

1. **Prometheus** - เก็บ metrics จากทุก component
2. **Grafana** - แสดง dashboards และ alerts
3. **Alertmanager** - จัดการ alert notifications ไปยัง Slack/PagerDuty
4. **postgres_exporter** - PostgreSQL metrics
5. **redis_exporter** - Redis metrics
6. **node_exporter** - System metrics
7. **blackbox_exporter** - Endpoint health checks
8. **Loki + Promtail** - Log aggregation

การ monitor ทุก layer ช่วยให้ detect ปัญหาได้ไว ลด MTTR (Mean Time to Recovery) และทำให้ team มีข้อมูลเพียงพอในการ troubleshoot ปัญหา

---

*เนื้อหาส่วนนี้เป็นส่วนหนึ่งของ Database Cluster Course - Advanced Content*
