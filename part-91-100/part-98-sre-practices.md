# Part 98: SRE Practices สำหรับ Database

## บทนำ: SRE คืออะไร?

Site Reliability Engineering (SRE) เป็น discipline ที่ Google สร้างขึ้น โดยใช้ software engineering principles มาแก้ปัญหา operations

> "SRE is what happens when you ask a software engineer to design an operations function"
> — Ben Treynor Sloss, ผู้ก่อตั้ง Google SRE

แนวคิดสำคัญ: **Reliability เป็น feature** ไม่ใช่แค่ requirement

---

## 1. SRE vs DevOps

```
DevOps:
  - Culture + Practices
  - Break down silos between Dev and Ops
  - Continuous Integration/Delivery
  - "You build it, you run it"

SRE:
  - Implementation ของ DevOps principles
  - Engineering-focused operations
  - Quantitative reliability (SLIs, SLOs, Error Budgets)
  - 50% rule (cap operational work at 50%)

ความเหมือน:
  - ทั้งคู่ต้องการ collaboration
  - ทั้งคู่ใช้ automation
  - ทั้งคู่ทำ post-mortems
  - ทั้งคู่ทำ monitoring และ observability

ความต่าง:
  - SRE มี prescriptive practices (SLO, Error Budget)
  - SRE focus ที่ engineering solutions
  - SRE มี 50% rule สำหรับ toil
  - SRE มี production readiness review
```

---

## 2. Error Budget: Balance Reliability vs Features

### 2.1 แนวคิด Error Budget

```
ถ้า SLO คือ 99.9% availability ต่อเดือน:
  
  Total minutes per month = 30 × 24 × 60 = 43,200 minutes
  Allowed downtime       = 43,200 × 0.1%  = 43.2 minutes
  
  นี่คือ "Error Budget" ของเดือนนี้

Error Budget policy:
  - Budget เหลือ > 50%: ทำ features ได้ปกติ
  - Budget เหลือ 10-50%: ระวัง, ต้อง review risky changes
  - Budget เหลือ < 10%: หยุด features, focus reliability
  - Budget หมด: Freeze releases จนกว่าจะ earn back
```

### 2.2 Error Budget Policy

```yaml
# error-budget-policy.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: error-budget-policy
  namespace: sre
data:
  policy.yaml: |
    # Error Budget Policy for Database Services

    service: shopcluster-database
    slo_target: 99.9  # percent

    thresholds:
      green:
        remaining_budget_percent: 50
        allowed_actions:
          - normal_feature_releases
          - schema_changes
          - major_upgrades
        
      yellow:
        remaining_budget_percent: 10
        allowed_actions:
          - critical_bug_fixes
          - security_patches
        required_approvals:
          - sre_lead
          - service_owner
        additional_monitoring:
          - increase_alert_sensitivity
          - reduce_batch_job_load
        
      red:
        remaining_budget_percent: 0
        allowed_actions:
          - critical_security_patches_only
        required_actions:
          - post_mortem_if_not_done
          - reliability_sprint
          - freeze_non_critical_releases
        notifications:
          - vp_engineering
          - product_management
        
      exhausted:
        remaining_budget_percent: -100
        required_actions:
          - executive_escalation
          - dedicated_reliability_team
          - suspend_feature_development
```

### 2.3 คำนวณ Error Budget อัตโนมัติ

```python
#!/usr/bin/env python3
# scripts/calculate_error_budget.py

import requests
from datetime import datetime, timedelta
from dataclasses import dataclass
from typing import Optional

@dataclass
class ErrorBudgetStatus:
    service: str
    period_days: int
    slo_target: float
    availability_actual: float
    total_minutes: float
    allowed_downtime_minutes: float
    actual_downtime_minutes: float
    remaining_budget_minutes: float
    remaining_budget_percent: float
    status: str  # green/yellow/red/exhausted

class ErrorBudgetCalculator:
    def __init__(self, prometheus_url: str):
        self.prometheus_url = prometheus_url
    
    def query_prometheus(self, query: str, start: datetime, end: datetime) -> dict:
        response = requests.get(
            f"{self.prometheus_url}/api/v1/query_range",
            params={
                "query": query,
                "start": start.isoformat() + "Z",
                "end": end.isoformat() + "Z",
                "step": "60s"
            }
        )
        return response.json()
    
    def calculate_availability(self, service: str, period_days: int) -> float:
        end = datetime.utcnow()
        start = end - timedelta(days=period_days)
        
        # Query successful requests / total requests
        query = f"""
            sum(rate(http_requests_total{{service="{service}",status!~"5.."}}[5m]))
            /
            sum(rate(http_requests_total{{service="{service}"}}[5m]))
        """
        
        result = self.query_prometheus(query, start, end)
        
        if not result["data"]["result"]:
            return 0.0
        
        # Average availability over period
        values = [float(v[1]) for v in result["data"]["result"][0]["values"]]
        return sum(values) / len(values) * 100
    
    def calculate_error_budget(
        self,
        service: str,
        slo_target: float,
        period_days: int = 30
    ) -> ErrorBudgetStatus:
        
        availability = self.calculate_availability(service, period_days)
        total_minutes = period_days * 24 * 60
        allowed_downtime = total_minutes * (1 - slo_target / 100)
        actual_downtime = total_minutes * (1 - availability / 100)
        remaining_budget = allowed_downtime - actual_downtime
        remaining_percent = (remaining_budget / allowed_downtime) * 100
        
        # Determine status
        if remaining_percent >= 50:
            status = "green"
        elif remaining_percent >= 10:
            status = "yellow"
        elif remaining_percent >= 0:
            status = "red"
        else:
            status = "exhausted"
        
        return ErrorBudgetStatus(
            service=service,
            period_days=period_days,
            slo_target=slo_target,
            availability_actual=availability,
            total_minutes=total_minutes,
            allowed_downtime_minutes=allowed_downtime,
            actual_downtime_minutes=actual_downtime,
            remaining_budget_minutes=remaining_budget,
            remaining_budget_percent=remaining_percent,
            status=status
        )

def main():
    calc = ErrorBudgetCalculator("http://prometheus:9090")
    
    services = [
        ("shopcluster-api", 99.9),
        ("shopcluster-database", 99.95),
        ("shopcluster-redis", 99.9),
    ]
    
    print(f"{'Service':<30} {'SLO':>6} {'Actual':>8} {'Budget Left':>12} {'Status':<12}")
    print("-" * 80)
    
    for service, slo in services:
        budget = calc.calculate_error_budget(service, slo)
        status_icon = {
            "green": "✅",
            "yellow": "⚠️",
            "red": "🔴",
            "exhausted": "💀"
        }[budget.status]
        
        print(
            f"{service:<30} "
            f"{slo:>5.2f}% "
            f"{budget.availability_actual:>7.3f}% "
            f"{budget.remaining_budget_minutes:>8.1f}m "
            f"({budget.remaining_budget_percent:>5.1f}%) "
            f"{status_icon} {budget.status}"
        )

if __name__ == "__main__":
    main()
```

---

## 3. SLI: Service Level Indicators

### 3.1 Database SLIs

```yaml
# sli-definitions.yaml
sli_definitions:
  database:
    # 1. Availability: % successful queries
    availability:
      description: "Percentage of database queries that succeed"
      measurement: |
        sum(rate(pg_stat_statements_calls_total{state!="error"}[5m]))
        /
        sum(rate(pg_stat_statements_calls_total[5m]))
        * 100
      good_event: "Query completes without error"
      valid_event: "Any query attempt"

    # 2. Latency: P50, P95, P99 query time
    query_latency_p50:
      description: "50th percentile query duration"
      measurement: |
        histogram_quantile(0.50, 
          sum(rate(pg_query_duration_seconds_bucket[5m])) by (le)
        )
      unit: seconds
    
    query_latency_p95:
      description: "95th percentile query duration"
      measurement: |
        histogram_quantile(0.95,
          sum(rate(pg_query_duration_seconds_bucket[5m])) by (le)
        )
      unit: seconds
    
    query_latency_p99:
      description: "99th percentile query duration"
      measurement: |
        histogram_quantile(0.99,
          sum(rate(pg_query_duration_seconds_bucket[5m])) by (le)
        )
      unit: seconds

    # 3. Error Rate
    error_rate:
      description: "Percentage of queries returning errors"
      measurement: |
        sum(rate(pg_stat_statements_calls_total{state="error"}[5m]))
        /
        sum(rate(pg_stat_statements_calls_total[5m]))
        * 100
    
    # 4. Durability
    durability:
      description: "Percentage of committed writes that are readable"
      measurement: "Measured by integrity checks and replication monitoring"
      # ยาก measure โดยตรง, ใช้ proxy metrics:
      proxy_metrics:
        - replication_lag_seconds: "< 30"
        - backup_age_hours: "< 25"
        - wal_receiver_status: "streaming"

    # 5. Throughput
    throughput:
      description: "Queries per second the database handles"
      measurement: |
        sum(rate(pg_stat_statements_calls_total[5m]))
      unit: "queries/second"
    
    # 6. Connection Availability
    connection_availability:
      description: "% of time connection pool has capacity"
      measurement: |
        (pg_settings_max_connections - pg_stat_activity_count)
        /
        pg_settings_max_connections
        * 100

  application:
    # Request success rate
    request_success_rate:
      description: "Percentage of HTTP requests returning 2xx/3xx"
      measurement: |
        sum(rate(http_requests_total{status=~"[23].."}[5m]))
        /
        sum(rate(http_requests_total[5m]))
        * 100

    # Request latency
    request_latency_p99:
      description: "99th percentile request duration"
      measurement: |
        histogram_quantile(0.99,
          sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
        )

    # Data freshness (replication lag)
    data_freshness:
      description: "Maximum replication lag across all replicas"
      measurement: |
        max(pg_replication_lag_seconds)
      good_threshold: "< 5s"
      acceptable_threshold: "< 30s"
```

### 3.2 Prometheus Recording Rules สำหรับ SLIs

```yaml
# prometheus-sli-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: database-sli-rules
  namespace: monitoring
spec:
  groups:
    - name: database.sli
      interval: 30s
      rules:
        # Availability SLI (5m window)
        - record: sli:database:availability:ratio_5m
          expr: |
            sum(rate(pg_stat_activity_count{state="active"}[5m]))
            /
            (
              sum(rate(pg_stat_activity_count{state="active"}[5m]))
              + sum(rate(pg_errors_total[5m]))
            )

        # Query Latency P99
        - record: sli:database:query_latency_p99:5m
          expr: |
            histogram_quantile(0.99,
              sum by(le) (
                rate(pg_query_duration_seconds_bucket[5m])
              )
            )

        # Connection Utilization
        - record: sli:database:connection_utilization:ratio
          expr: |
            pg_stat_activity_count
            /
            pg_settings_max_connections

        # Replication Lag
        - record: sli:database:replication_lag:seconds
          expr: |
            max by(instance) (pg_replication_lag_seconds)

        # Write Availability
        - record: sli:database:write_availability:ratio_5m
          expr: |
            up{job="postgresql", role="primary"}
```

---

## 4. SLO: Service Level Objectives

### 4.1 SLO Definitions

```yaml
# slo-definitions.yaml
slo_definitions:
  database_availability:
    service: shopcluster-database
    sli: database_availability
    target: 99.9  # percent
    window: 30d
    description: "99.9% of all queries succeed"
    rationale: "Based on customer impact analysis - 43 minutes downtime/month acceptable"
    
    error_budget:
      monthly_minutes: 43.2
      policy:
        green_threshold: 50    # >50% budget remaining: normal operations
        yellow_threshold: 10   # 10-50%: review risky changes
        red_threshold: 0       # <10%: freeze non-critical changes
    
    # ถ้า SLO ละเมิด ต้องทำอะไร
    consequences:
      - "Post-mortem required within 24 hours"
      - "Reliability sprint in next quarter"
      - "Executive review if 3+ violations in 6 months"

  query_latency_p99:
    service: shopcluster-database
    sli: query_latency_p99
    target: 100  # milliseconds
    window: 30d
    description: "99% of queries complete within 100ms"
    
  write_availability:
    service: shopcluster-database-primary
    sli: write_availability
    target: 99.95  # percent
    window: 30d
    description: "99.95% write availability"
    rationale: "Writes are critical - even brief downtime causes user impact"

  replication_lag:
    service: shopcluster-database
    sli: replication_lag
    target: 30   # seconds maximum
    window: 30d
    description: "Replication lag stays under 30 seconds"
    rationale: "Reads from replica must not be more than 30s stale"
```

### 4.2 SLO Dashboard Configuration

```json
{
  "dashboard": {
    "title": "SLO Dashboard - Database",
    "panels": [
      {
        "id": 1,
        "title": "Availability SLO (99.9%)",
        "type": "gauge",
        "targets": [
          {
            "expr": "sli:database:availability:ratio_5m * 100",
            "legendFormat": "Availability"
          }
        ],
        "fieldConfig": {
          "min": 99,
          "max": 100,
          "thresholds": {
            "steps": [
              { "color": "red", "value": 99 },
              { "color": "yellow", "value": 99.5 },
              { "color": "green", "value": 99.9 }
            ]
          }
        }
      },
      {
        "id": 2,
        "title": "Error Budget Remaining (30d)",
        "type": "timeseries",
        "description": "Minutes of error budget remaining this month",
        "targets": [
          {
            "expr": "43.2 - (43200 * (1 - avg_over_time(sli:database:availability:ratio_5m[30d])))",
            "legendFormat": "Budget Remaining (minutes)"
          }
        ],
        "thresholds": [
          { "color": "green", "value": 21.6 },
          { "color": "yellow", "value": 4.32 },
          { "color": "red", "value": 0 }
        ]
      },
      {
        "id": 3,
        "title": "Query Latency P99",
        "type": "timeseries",
        "targets": [
          {
            "expr": "sli:database:query_latency_p99:5m * 1000",
            "legendFormat": "P99 (ms)"
          }
        ],
        "alert": {
          "conditions": [
            {
              "query": "A",
              "reducer": "last",
              "evaluator": { "type": "gt", "params": [100] }
            }
          ]
        }
      }
    ]
  }
}
```

---

## 5. Toil: Repetitive Manual Work

### 5.1 Identifying Toil

```
Toil คือ work ที่:
  ✓ Manual
  ✓ Repetitive
  ✓ Automatable
  ✓ Tactical (short-term fix, ไม่แก้ปัญหาที่ root)
  ✓ Without enduring value
  ✓ Scales linearly with service growth

ตัวอย่าง Database Toil:
  - Manual rotation ของ database passwords ทุกเดือน
  - Manually resize storage เมื่อใกล้เต็ม
  - Manually restart stuck connection pools
  - Manually investigate "slow query" alerts ที่เป็น false positive
  - Manually clear temporary tables
  - Manually upgrade minor versions
  - Manually create read replicas เมื่อ load เพิ่ม

ไม่ใช่ Toil:
  - Post-mortem analysis
  - สร้าง new monitoring system
  - Mentoring team members
  - Architecture review
```

### 5.2 Toil Tracking

```python
#!/usr/bin/env python3
# scripts/toil_tracker.py

import json
from datetime import datetime
from pathlib import Path
from dataclasses import dataclass, asdict
from typing import List, Optional

@dataclass
class ToilRecord:
    date: str
    engineer: str
    task: str
    category: str
    duration_minutes: int
    is_automatable: bool
    automation_effort_days: Optional[int]
    notes: str

class ToilTracker:
    def __init__(self, data_file: str = "toil_records.json"):
        self.data_file = Path(data_file)
        self.records: List[ToilRecord] = []
        self.load()
    
    def load(self):
        if self.data_file.exists():
            with open(self.data_file) as f:
                data = json.load(f)
                self.records = [ToilRecord(**r) for r in data]
    
    def save(self):
        with open(self.data_file, 'w') as f:
            json.dump([asdict(r) for r in self.records], f, indent=2)
    
    def add_record(self, record: ToilRecord):
        self.records.append(record)
        self.save()
    
    def generate_report(self, week_start: datetime) -> dict:
        week_end = week_start.replace(day=week_start.day + 7)
        
        week_records = [
            r for r in self.records
            if week_start.isoformat()[:10] <= r.date <= week_end.isoformat()[:10]
        ]
        
        total_minutes = sum(r.duration_minutes for r in week_records)
        toil_minutes = total_minutes  # all records are toil
        
        # แยก by category
        by_category = {}
        for r in week_records:
            by_category.setdefault(r.category, 0)
            by_category[r.category] += r.duration_minutes
        
        # Automation opportunities
        automatable = [r for r in week_records if r.is_automatable]
        potential_savings = sum(r.duration_minutes for r in automatable)
        
        return {
            "period": f"{week_start.date()} to {week_end.date()}",
            "total_toil_minutes": total_minutes,
            "toil_hours": total_minutes / 60,
            "by_category": by_category,
            "automation_potential": {
                "items": len(automatable),
                "minutes_saveable": potential_savings,
                "roi_weeks": sum(
                    r.automation_effort_days or 0 for r in automatable
                ) * 8 * 60 / (potential_savings or 1)
            }
        }

# ตัวอย่าง database toil categories
TOIL_CATEGORIES = {
    "storage_management": "Storage หมด, resize",
    "connection_pool": "Connection pool issues",
    "performance": "Slow queries, locks",
    "backup_restore": "Backup failures, restore tests",
    "access_management": "Create/revoke users, rotate passwords",
    "replication": "Replication lag, failover",
    "monitoring": "Alert noise, false positives",
    "maintenance": "Vacuum, analyze, updates",
}
```

### 5.3 Automate Common Toil

```bash
#!/bin/bash
# scripts/auto-remediation/fix-connection-pool.sh
# Automate: restart stuck PgBouncer connections

set -euo pipefail

NAMESPACE="${1:-prod-database}"
POD_LABEL="app=pgbouncer"
MAX_IDLE_CONNECTIONS=50

echo "=== Auto-remediation: Connection Pool Cleanup ==="
echo "Namespace: $NAMESPACE"

# ดู current stats
STATS=$(kubectl exec -n "$NAMESPACE" \
  $(kubectl get pods -n "$NAMESPACE" -l "$POD_LABEL" -o name | head -1) \
  -- psql -p 6432 pgbouncer -c "SHOW STATS;" -t 2>/dev/null)

echo "Current stats:"
echo "$STATS"

# ตรวจสอบ idle connections
IDLE_COUNT=$(kubectl exec -n "$NAMESPACE" \
  $(kubectl get pods -n "$NAMESPACE" -l "$POD_LABEL" -o name | head -1) \
  -- psql -p 6432 pgbouncer -c "SHOW CLIENTS;" -t 2>/dev/null | \
  grep "idle" | wc -l)

echo "Idle connections: $IDLE_COUNT"

if [ "$IDLE_COUNT" -gt "$MAX_IDLE_CONNECTIONS" ]; then
  echo "Too many idle connections ($IDLE_COUNT > $MAX_IDLE_CONNECTIONS)"
  echo "Running RECONNECT to clean up..."
  
  kubectl exec -n "$NAMESPACE" \
    $(kubectl get pods -n "$NAMESPACE" -l "$POD_LABEL" -o name | head -1) \
    -- psql -p 6432 pgbouncer -c "RECONNECT;" 2>/dev/null
  
  echo "Reconnect completed"
  
  # ส่ง notification
  curl -s -X POST "$SLACK_WEBHOOK" \
    -H 'Content-type: application/json' \
    -d "{\"text\": \"🔧 Auto-remediated: Cleaned up $IDLE_COUNT idle connections in $NAMESPACE\"}"
else
  echo "Connection count is normal, no action needed"
fi
```

---

## 6. On-Call Practices

### 6.1 Rotation Schedule

```yaml
# oncall-schedule.yaml
schedule:
  name: "Database SRE On-Call"
  timezone: "Asia/Bangkok"
  
  primary_rotation:
    duration: 1_week
    handoff_time: "10:00"
    team_members:
      - name: "Alice"
        email: "alice@company.com"
        phone: "+66-81-xxx-xxxx"
      - name: "Bob"
        email: "bob@company.com"
        phone: "+66-82-xxx-xxxx"
      - name: "Charlie"
        email: "charlie@company.com"
        phone: "+66-83-xxx-xxxx"
  
  secondary_rotation:
    purpose: "Escalation if primary unreachable"
    duration: 1_week
    offset: 0  # Same week as primary, different person

escalation_policy:
  steps:
    - level: 1
      name: "Page Primary On-Call"
      timeout: 5_minutes
      
    - level: 2
      name: "Page Secondary On-Call"
      timeout: 10_minutes
      
    - level: 3
      name: "Page Engineering Manager"
      timeout: 20_minutes
      
    - level: 4
      name: "Page VP Engineering"
      timeout: 30_minutes
```

### 6.2 Alert Quality Framework

```yaml
# good-alert-criteria.yaml
# ทุก alert ต้องผ่าน criteria เหล่านี้:

alert_quality_checklist:
  actionable:
    description: "Alert must have clear action to take"
    bad_example: "CPU usage high"
    good_example: "CPU usage > 90% for 15 minutes - check for slow queries, lock contention"
    
  urgent:
    description: "Alert must actually require immediate attention"
    bad_example: "Disk usage > 60%"
    good_example: "Disk usage > 90% - estimated full in 2 hours"
    
  accurate:
    description: "Alert must be accurate - not false positive"
    check: "false positive rate < 5%"
    action_if_noisy: "Fix alert or acknowledge as known issue"
    
  has_runbook:
    description: "Every alert must link to a runbook"
    required_field: "annotations.runbook_url"
    
  tested:
    description: "Alert has been tested to fire correctly"
    test_method: "AlertManager test / PromQL unit test"
```

### 6.3 Complete Alert Rules พร้อม Runbooks

```yaml
# prometheus-alert-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: database-sre-alerts
  namespace: monitoring
spec:
  groups:
    - name: database.critical
      rules:
        - alert: DatabaseDown
          expr: up{job="postgresql"} == 0
          for: 1m
          labels:
            severity: critical
            team: sre
            page: "true"
          annotations:
            summary: "PostgreSQL instance {{ $labels.instance }} is DOWN"
            description: |
              PostgreSQL {{ $labels.instance }} has been unreachable for more than 1 minute.
              This impacts ALL database operations.
            runbook_url: "https://wiki.company.com/runbooks/database-down"
            dashboard_url: "https://grafana.company.com/d/database-overview"

        - alert: DatabaseReplicationBroken
          expr: |
            pg_replication_is_replica == 1
            and
            pg_replication_lag_seconds > 300
          for: 5m
          labels:
            severity: critical
            team: sre
          annotations:
            summary: "Replication lag critically high: {{ $value | humanizeDuration }}"
            description: |
              Replica {{ $labels.instance }} is {{ $value | humanizeDuration }} behind primary.
              Reads from this replica may return stale data.
              
              Possible causes:
              - Network issue between primary and replica
              - Heavy write load on primary
              - Replica CPU/IO bottleneck
            runbook_url: "https://wiki.company.com/runbooks/replication-lag"

        - alert: DatabaseConnectionsExhausted
          expr: |
            pg_stat_activity_count / pg_settings_max_connections > 0.95
          for: 5m
          labels:
            severity: critical
            team: sre
            page: "true"
          annotations:
            summary: "Database connections at {{ $value | humanizePercentage }}"
            description: |
              Connection pool is {{ $value | humanizePercentage }} full.
              New connections will be refused soon.
              
              Current: {{ $value }} connections
              Maximum: {{ with query "pg_settings_max_connections" }}{{ . | first | value }}{{ end }}
              
              Immediate actions:
              1. Check for connection leaks: SELECT * FROM pg_stat_activity WHERE state = 'idle' ORDER BY query_start;
              2. Kill idle connections if needed
              3. Check application connection pool settings
            runbook_url: "https://wiki.company.com/runbooks/connections-exhausted"

    - name: database.warning
      rules:
        - alert: DatabaseSlowQueries
          expr: |
            rate(pg_stat_statements_mean_exec_time_seconds[5m]) > 0.1
          for: 10m
          labels:
            severity: warning
            team: sre
          annotations:
            summary: "Slow queries detected: avg {{ $value | humanizeDuration }}"
            description: |
              Average query execution time is {{ $value | humanizeDuration }}.
              This may indicate missing indexes, lock contention, or resource exhaustion.
            runbook_url: "https://wiki.company.com/runbooks/slow-queries"

        - alert: DatabaseDiskSpaceLow
          expr: |
            (
              node_filesystem_free_bytes{mountpoint="/var/lib/postgresql"}
              /
              node_filesystem_size_bytes{mountpoint="/var/lib/postgresql"}
            ) < 0.20
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "Database disk space {{ $value | humanizePercentage }} remaining"
            description: |
              Disk space on {{ $labels.instance }} is running low.
              {{ $value | humanizePercentage }} ({{ with query "node_filesystem_free_bytes{mountpoint='/var/lib/postgresql'}" }}{{ . | first | value | humanize1024 }}B{{ end }}) remaining.
              
              Actions:
              1. Check for bloat: SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) FROM pg_tables ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC LIMIT 10;
              2. Run VACUUM FULL on bloated tables
              3. Consider archiving old data
              4. Increase storage if needed
            runbook_url: "https://wiki.company.com/runbooks/disk-space-low"

        - alert: DatabaseBackupMissing
          expr: |
            (time() - database_last_backup_timestamp_seconds) > 86400
          for: 1h
          labels:
            severity: warning
          annotations:
            summary: "Database backup not taken in {{ $value | humanizeDuration }}"
            runbook_url: "https://wiki.company.com/runbooks/backup-missing"
```

---

## 7. Production Readiness Review (PRR)

### 7.1 PRR Checklist

```markdown
# Production Readiness Review: Database Service

## Service Information
- **Service Name**: 
- **Team**: 
- **Launch Date**: 
- **Review Date**: 
- **Reviewer**: 

## SLO & Monitoring

### SLO Defined
- [ ] Availability SLO defined and documented
- [ ] Latency SLO defined and documented
- [ ] Error budget policy defined
- [ ] SLO dashboard created in Grafana

### Alerts Configured
- [ ] All critical SLIs have alerts
- [ ] Alerts have runbooks
- [ ] Alerts tested (verified they fire correctly)
- [ ] PagerDuty/OpsGenie integration configured
- [ ] On-call rotation defined

### Monitoring Coverage
- [ ] Connection pool metrics
- [ ] Query latency histogram
- [ ] Replication lag
- [ ] Disk usage trend
- [ ] Error rate
- [ ] Slow query log enabled

## Reliability

### High Availability
- [ ] Database has replica(s)
- [ ] Automatic failover tested
- [ ] Failover time meets SLO requirements
- [ ] Connection retry logic in application

### Backup & Recovery
- [ ] Automated backup configured
- [ ] Backup retention meets requirements
- [ ] Restore procedure documented and tested
- [ ] Recovery time objective (RTO) tested
- [ ] Recovery point objective (RPO) verified

### Load Testing
- [ ] Load test run at expected traffic
- [ ] Load test run at 2x expected traffic
- [ ] Bottlenecks identified and addressed
- [ ] Connection pool tuned
- [ ] Query performance verified under load

### Capacity Planning
- [ ] Current capacity documented
- [ ] Growth projections calculated
- [ ] Scaling triggers defined
- [ ] Max capacity tested

## Security
- [ ] Database credentials in Vault/Secrets Manager
- [ ] Network policy limiting access
- [ ] Encryption at rest enabled
- [ ] Encryption in transit (TLS) enabled
- [ ] Audit logging enabled
- [ ] No hardcoded credentials in code

## Operational Procedures
- [ ] Runbooks written for all alerts
- [ ] Common issues documented
- [ ] Escalation policy defined
- [ ] On-call team trained
- [ ] Schema migration procedure documented
- [ ] Rollback procedure tested

## Score: ___ / 35 items checked
## Status: [ ] APPROVED  [ ] CONDITIONAL  [ ] NOT READY
```

---

## 8. Post-Mortem Culture

### 8.1 Blameless Post-Mortem

```
Blameless หมายความว่า:
  ✓ ไม่โทษบุคคล
  ✓ ไม่ใช้คำว่า "ความผิด" หรือ "ผิดพลาด"
  ✓ Focus ที่ system และ process
  ✓ ทุกคนทำสิ่งที่ดีที่สุดด้วย information ที่มีในขณะนั้น
  ✓ Psychological safety: คนพูดความจริงได้

ประโยชน์:
  - คนยอมรับปัญหาเร็วขึ้น ไม่รอซ่อนปัญหา
  - ได้ข้อมูลครบถ้วน
  - แก้ root cause จริง ไม่ใช่แค่ blame คน
  - สร้างวัฒนธรรมเรียนรู้
```

### 8.2 Post-Mortem Template

```markdown
# Post-Mortem: Database Outage 2026-01-15

**Severity**: P1 (Critical)
**Duration**: 47 minutes (14:23 - 15:10 UTC)
**Impact**: 100% of write operations failed, 85% of read operations failed

## Summary
ระหว่าง routine maintenance, storage volume เกิด full ทำให้ PostgreSQL หยุดทำงาน การ alert ที่มีอยู่ไม่ได้แจ้ง warning ล่วงหน้าเพียงพอ เนื่องจาก threshold ตั้งไว้ที่ 95% แต่ temporary files จาก sort operation ทำให้ disk เต็มเร็วกว่าคาด

## Impact
- **Users affected**: ~12,000 active users
- **Transactions lost**: 0 (all buffered and replayed)
- **SLO impact**: Used 31 minutes of 43-minute monthly error budget
- **Revenue impact**: Estimated $5,000 in delayed orders

## Timeline

| Time (UTC) | Event |
|------------|-------|
| 14:00 | Routine maintenance window starts |
| 14:15 | Large batch job starts (sort operation) |
| 14:23 | PostgreSQL error: "no space left on device" |
| 14:23 | First user reports: "500 error on checkout" |
| 14:28 | Alert fires: DiskSpaceCritical (95% full) |
| 14:30 | On-call (Alice) acknowledges alert |
| 14:35 | Alice identifies temp files issue |
| 14:40 | Temporary files cleared (50GB freed) |
| 14:45 | PostgreSQL auto-recovers |
| 14:50 | Connection pool reconnects |
| 15:00 | All services reporting healthy |
| 15:10 | Full recovery confirmed, incident closed |

## Root Cause Analysis

### Contributing Factors (5 Whys)

**Why** did the service fail?  
→ PostgreSQL ran out of disk space

**Why** did it run out of disk space?  
→ Sort operation created large temp files (45GB)

**Why** was there a sort operation of this size?  
→ Batch job ran a query without proper index (ORDER BY on unindexed column)

**Why** didn't the index exist?  
→ Index was dropped during previous migration to speed it up, never recreated

**Why** wasn't this caught before production?  
→ Load test used only 1/10 of production data volume

## What Went Well
- On-call response was fast (7 minutes from alert to action)
- Recovery was fast once root cause identified
- No data was lost due to WAL journaling
- Communication to stakeholders was timely

## What Went Poorly
- Alert threshold (95%) was too late to prevent impact
- No monitoring of temp file size
- Batch job not tested with production-scale data
- Disk space trend alert missing (we had absolute, not trend)

## Action Items

| Action | Owner | Due Date | Priority |
|--------|-------|----------|----------|
| Add disk space trend alert (predict when full) | Alice | 2026-01-22 | P1 |
| Add monitoring for temp file size | Bob | 2026-01-22 | P1 |
| Restore missing index | Charlie | 2026-01-17 | P1 |
| Test batch jobs with production-scale data | Dev team | 2026-01-31 | P2 |
| Set disk alert to 80% (not 95%) | Alice | 2026-01-18 | P2 |
| Add automated storage resize when >85% | Platform team | 2026-02-15 | P3 |
| Document batch job testing requirements | Tech lead | 2026-01-31 | P3 |

## Lessons Learned
1. Absolute thresholds miss sudden changes - trend-based alerts are more predictive
2. Batch jobs must be tested with production-scale data
3. Temp file growth is hard to predict from query analysis alone
4. Adding monitoring to new metrics is better than raising thresholds on existing ones
```

---

## 9. Capacity Planning

### 9.1 Load Testing Tools

```javascript
// k6-capacity-test.js - ทดสอบ capacity ของ database
import http from 'k6/http';
import { sleep, check } from 'k6';
import { Trend, Rate } from 'k6/metrics';

// Custom metrics
const dbQueryTime = new Trend('db_query_time');
const dbErrors = new Rate('db_errors');

export const options = {
  stages: [
    { duration: '5m', target: 100 },   // ramp up
    { duration: '10m', target: 100 },  // steady state
    { duration: '5m', target: 500 },   // peak
    { duration: '10m', target: 500 },  // stress
    { duration: '5m', target: 0 },     // ramp down
  ],
  thresholds: {
    'http_req_duration': ['p(99)<200'],
    'db_query_time': ['p(99)<100'],
    'db_errors': ['rate<0.01'],
  },
};

export default function () {
  // Test: read heavy workload (70% reads, 30% writes)
  const isWrite = Math.random() < 0.3;
  
  let response;
  const startTime = Date.now();
  
  if (isWrite) {
    response = http.post(
      'http://api-service/api/orders',
      JSON.stringify({
        userId: Math.floor(Math.random() * 10000),
        items: [{ productId: 'prod-1', quantity: 1 }],
      }),
      { headers: { 'Content-Type': 'application/json' } }
    );
  } else {
    const productId = Math.floor(Math.random() * 1000) + 1;
    response = http.get(`http://api-service/api/products/${productId}`);
  }
  
  const queryTime = Date.now() - startTime;
  dbQueryTime.add(queryTime);
  
  const success = check(response, {
    'status is 2xx': (r) => r.status >= 200 && r.status < 300,
    'response time < 200ms': (r) => r.timings.duration < 200,
  });
  
  if (!success) {
    dbErrors.add(1);
  }
  
  sleep(0.1);  // 10 RPS per VU
}
```

### 9.2 Growth Projection Script

```python
#!/usr/bin/env python3
# scripts/capacity_projection.py

import requests
from datetime import datetime, timedelta
import numpy as np
from scipy import stats

class CapacityProjector:
    def __init__(self, prometheus_url: str):
        self.prometheus_url = prometheus_url
    
    def get_metric_history(self, metric: str, days: int = 90) -> list:
        """ดึง metric history จาก Prometheus"""
        end = datetime.utcnow()
        start = end - timedelta(days=days)
        
        response = requests.get(
            f"{self.prometheus_url}/api/v1/query_range",
            params={
                "query": metric,
                "start": start.timestamp(),
                "end": end.timestamp(),
                "step": "86400",  # daily
            }
        )
        
        data = response.json()
        if data["data"]["result"]:
            return [(float(ts), float(val)) 
                   for ts, val in data["data"]["result"][0]["values"]]
        return []
    
    def project_growth(self, metric_name: str, metric_query: str, 
                      capacity_limit: float, days_ahead: int = 90):
        """คำนวณว่าจะถึง capacity เมื่อไหร่"""
        history = self.get_metric_history(metric_query)
        
        if len(history) < 14:
            return {"error": "Insufficient data"}
        
        timestamps = np.array([h[0] for h in history])
        values = np.array([h[1] for h in history])
        
        # Linear regression
        slope, intercept, r_value, p_value, std_err = stats.linregress(
            timestamps, values
        )
        
        # Project future values
        future_timestamps = [
            (datetime.utcnow() + timedelta(days=i)).timestamp()
            for i in range(1, days_ahead + 1)
        ]
        future_values = [slope * ts + intercept for ts in future_timestamps]
        
        # Find when capacity is reached
        days_until_limit = None
        for i, val in enumerate(future_values):
            if val >= capacity_limit:
                days_until_limit = i + 1
                break
        
        current_value = values[-1]
        growth_rate_daily = slope * 86400  # per day
        
        return {
            "metric": metric_name,
            "current_value": round(current_value, 2),
            "capacity_limit": capacity_limit,
            "current_utilization_percent": round(current_value / capacity_limit * 100, 1),
            "daily_growth": round(growth_rate_daily, 2),
            "growth_rate_percent": round(growth_rate_daily / current_value * 100, 2),
            "days_until_limit": days_until_limit,
            "r_squared": round(r_value ** 2, 3),
            "recommendation": get_recommendation(
                current_value / capacity_limit,
                days_until_limit
            )
        }

def get_recommendation(utilization: float, days_until_limit) -> str:
    if utilization > 0.85:
        return "⚠️ URGENT: Scale up immediately"
    elif utilization > 0.70:
        return "⚡ Scale up within 1 month"
    elif days_until_limit and days_until_limit < 30:
        return f"📈 Plan scale up - {days_until_limit} days to capacity"
    elif days_until_limit and days_until_limit < 90:
        return f"📊 Monitor - {days_until_limit} days to capacity"
    else:
        return "✅ Sufficient capacity"

def main():
    projector = CapacityProjector("http://prometheus:9090")
    
    metrics_to_check = [
        (
            "Database Storage",
            'pg_database_size_bytes{datname="shopcluster"}',
            500 * 1024 * 1024 * 1024  # 500GB limit
        ),
        (
            "Active Connections",
            'pg_stat_activity_count',
            200  # max_connections
        ),
        (
            "Queries/Second",
            'sum(rate(pg_stat_statements_calls_total[1d]))',
            10000  # estimated limit
        ),
    ]
    
    print("=" * 80)
    print("DATABASE CAPACITY PROJECTION REPORT")
    print(f"Generated: {datetime.utcnow().strftime('%Y-%m-%d %H:%M UTC')}")
    print("=" * 80)
    
    for name, query, limit in metrics_to_check:
        result = projector.project_growth(name, query, limit)
        
        print(f"\n{name}")
        print(f"  Current: {result.get('current_value')} / {result.get('capacity_limit')}")
        print(f"  Utilization: {result.get('current_utilization_percent')}%")
        print(f"  Daily Growth: {result.get('daily_growth')}")
        print(f"  Days to Limit: {result.get('days_until_limit', 'N/A')}")
        print(f"  Recommendation: {result.get('recommendation')}")

if __name__ == "__main__":
    main()
```

---

## สรุป SRE Practices สำหรับ Database

| Practice | เป้าหมาย | เครื่องมือ |
|----------|----------|-----------|
| SLI/SLO | วัด reliability อย่าง objective | Prometheus + Grafana |
| Error Budget | Balance features vs reliability | Custom calculation |
| Toil Reduction | ลด manual work | Automation scripts |
| Blameless Post-mortem | เรียนรู้จากความผิดพลาด | Template + culture |
| Production Readiness | ป้องกันปัญหาก่อน launch | Checklist |
| Capacity Planning | ไม่ให้ surprise | Prometheus trends |
| On-Call | ตอบสนองเร็ว, ยั่งยืน | PagerDuty + Runbooks |

SRE transforms database operations จาก reactive ("fix when broken") เป็น proactive ("prevent before broken") ด้วยข้อมูลเชิง quantitative และ engineering mindset
