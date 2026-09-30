# Part 97: Platform Engineering สำหรับ Database

## บทนำ: ทำไมต้องมี Platform Engineering?

องค์กรที่มี developer หลายร้อยคน ทุกทีมต้องการ database ของตัวเอง การ provision ด้วยมือทุกครั้งทำให้เกิดปัญหา:
- ใช้เวลา 2-3 วันเพื่อได้ database ใหม่
- Config ไม่สม่ำเสมอ (บาง env ไม่มี backup)
- Developer ต้องรู้ Kubernetes, Helm, Terraform
- ไม่มี visibility ว่า database ไหนเป็นของทีมไหน

**Platform Engineering** แก้ปัญหานี้โดยสร้าง "paved road" - ทางที่ง่าย, ปลอดภัย, และเป็นมาตรฐาน

---

## 1. Platform Engineering คืออะไร?

### 1.1 Developer Experience (DevEx)

DevEx คือประสบการณ์ของ developer ในการทำงาน

```
ก่อน Platform Engineering:
  Developer ต้องการ PostgreSQL database
  → ส่ง email หา DBA team
  → รอ 2-3 วัน
  → รับ credentials ผ่าน email (ไม่ปลอดภัย)
  → Setup monitoring ด้วยตัวเอง
  → ไม่มีใครรู้ว่า database นี้ทำอะไร

หลัง Platform Engineering:
  Developer ต้องการ PostgreSQL database
  → เปิด Backstage portal
  → กรอก form: team, environment, size
  → กด "Create"
  → ได้ credentials ใน 5 นาที
  → Monitoring, backup, alerts ตั้งค่าให้อัตโนมัติ
```

### 1.2 Golden Paths

Golden Path คือ opinionated, well-maintained way to do common tasks

```yaml
# Golden Path สำหรับ Create New PostgreSQL Database
# Developer กรอกแค่นี้:

name: my-service-db
team: checkout-team
environment: production
size: small  # small/medium/large/xlarge

# Platform จัดการให้อัตโนมัติ:
# ✓ PostgreSQL 15 HA setup
# ✓ Automated backups ทุกวัน
# ✓ Prometheus monitoring
# ✓ Grafana dashboard
# ✓ Alerts (disk, connections, replication)
# ✓ Vault credentials rotation
# ✓ Cost allocation tag
# ✓ Network policy
```

### 1.3 Reduce Cognitive Load

```
❌ Developer ต้องรู้:
   - Kubernetes StatefulSet
   - Persistent Volume Claims
   - PostgreSQL replication setup
   - Backup strategies
   - Network policies
   - Secret management
   - Monitoring setup

✅ Developer ต้องรู้:
   - ต้องการ database ขนาดไหน?
   - ใช้ใน environment ไหน?
```

---

## 2. Internal Developer Platform (IDP)

### 2.1 Backstage: Developer Portal

Backstage (by Spotify) เป็น open platform สำหรับสร้าง Internal Developer Portal

```
Backstage Components:
├── Software Catalog     - catalog ของทุก services, databases, APIs
├── TechDocs             - technical documentation
├── Templates            - สร้าง service/database ใหม่
├── Tech Radar           - technology recommendations
├── Plugins              - integrations (GitHub, Jira, PagerDuty, etc.)
└── APIs                 - CI/CD, deployment, monitoring
```

### 2.2 ติดตั้ง Backstage

```bash
# สร้าง Backstage app ใหม่
npx @backstage/create-app@latest

# ตั้งชื่อ app
? Enter a name for the app [required]: database-platform

cd database-platform

# Start development server
yarn dev
```

```yaml
# app-config.yaml - Backstage configuration
app:
  title: Database Platform Portal
  baseUrl: http://localhost:3000

backend:
  baseUrl: http://localhost:7007
  listen:
    port: 7007

integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}

catalog:
  import:
    entityFilename: catalog-info.yaml
    pullRequestBranchName: backstage-integration
  rules:
    - allow: [Component, System, API, Resource, Location, Template]
  locations:
    # Load component definitions
    - type: file
      target: ../../examples/entities.yaml
    # Load all catalog files from GitOps repo
    - type: url
      target: https://github.com/company/gitops-repo/blob/main/catalog/all.yaml

proxy:
  '/grafana/api':
    target: https://grafana.company.internal
    headers:
      Authorization: Bearer ${GRAFANA_TOKEN}

  '/prometheus/api':
    target: https://prometheus.company.internal

auth:
  environment: development
  providers:
    github:
      development:
        clientId: ${AUTH_GITHUB_CLIENT_ID}
        clientSecret: ${AUTH_GITHUB_CLIENT_SECRET}

techdocs:
  builder: 'local'
  generator:
    runIn: 'local'
  publisher:
    type: 'local'
```

### 2.3 Database Catalog Integration

```yaml
# catalog-info.yaml - สำหรับทุก database ที่ platform manage
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: shopcluster-prod-db
  description: Production PostgreSQL database for ShopCluster
  annotations:
    backstage.io/managed-by-location: url:https://github.com/company/gitops-repo/blob/main/catalog/databases.yaml
    grafana/dashboard-selector: "title%20contains%20'ShopCluster'"
    prometheus.io/alert: 'DatabaseAvailability'
    vault.io/secret-path: 'database/prod/shopcluster'
  labels:
    environment: production
    team: platform
    criticality: high
    database-type: postgresql
  tags:
    - postgresql
    - production
    - ecommerce
spec:
  type: database
  lifecycle: production
  owner: group:database-platform
  system: shopcluster
  dependencyOf:
    - component:shopcluster-api
    - component:shopcluster-worker
  profile:
    displayName: ShopCluster Production DB
    email: db-platform@company.com

---
# ทุก team's database
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: checkout-team-db
  description: PostgreSQL database for Checkout Team
  labels:
    environment: production
    team: checkout
    database-type: postgresql
spec:
  type: database
  lifecycle: production
  owner: group:checkout-team
  system: checkout-service
```

### 2.4 Backstage Templates: สร้าง New PostgreSQL Database

```yaml
# templates/new-postgresql-database/template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: new-postgresql-database
  title: New PostgreSQL Database
  description: Provision a new PostgreSQL database with HA, backup, and monitoring
  tags:
    - database
    - postgresql
    - recommended
spec:
  owner: group:database-platform
  type: database

  parameters:
    - title: Database Information
      required:
        - databaseName
        - team
        - environment
        - purpose
      properties:
        databaseName:
          title: Database Name
          type: string
          description: Name of the database (lowercase, hyphens allowed)
          pattern: '^[a-z][a-z0-9-]{2,50}$'
          ui:autofocus: true
        team:
          title: Team
          type: string
          description: Your team name
          enum:
            - checkout
            - catalog
            - orders
            - payments
            - users
            - platform
        environment:
          title: Environment
          type: string
          enum:
            - development
            - staging
            - production
          default: development
        purpose:
          title: Purpose
          type: string
          description: What is this database used for?
          maxLength: 200

    - title: Database Configuration
      properties:
        size:
          title: Database Size
          type: string
          enum:
            - small   # 1 CPU, 2GB RAM, 50GB storage
            - medium  # 2 CPU, 8GB RAM, 200GB storage
            - large   # 4 CPU, 16GB RAM, 500GB storage
            - xlarge  # 8 CPU, 32GB RAM, 1TB storage
          default: small
          ui:widget: radio
        highAvailability:
          title: High Availability (Primary + 2 Replicas)
          type: boolean
          default: false
          description: Requires medium or larger size
        extensions:
          title: PostgreSQL Extensions
          type: array
          items:
            type: string
            enum:
              - uuid-ossp
              - pgcrypto
              - pg_stat_statements
              - postgis
              - pg_trgm
          uniqueItems: true

    - title: Backup Configuration
      properties:
        backupSchedule:
          title: Backup Schedule
          type: string
          enum:
            - daily    # เก็บ 30 วัน
            - hourly   # เก็บ 7 วัน
            - disabled
          default: daily
        pointInTimeRecovery:
          title: Enable Point-in-Time Recovery
          type: boolean
          default: false
          description: Enables WAL archiving (requires additional storage)

  steps:
    - id: fetch-template
      name: Fetch Database Template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          databaseName: ${{ parameters.databaseName }}
          team: ${{ parameters.team }}
          environment: ${{ parameters.environment }}
          size: ${{ parameters.size }}
          highAvailability: ${{ parameters.highAvailability }}
          backupSchedule: ${{ parameters.backupSchedule }}

    - id: publish-gitops
      name: Publish to GitOps Repository
      action: publish:github:pull-request
      input:
        repoUrl: github.com?repo=gitops-repo&owner=company
        branchName: create-db-${{ parameters.team }}-${{ parameters.databaseName }}
        title: "chore: provision ${{ parameters.databaseName }} database for ${{ parameters.team }}"
        description: |
          ## New Database Request
          
          **Database:** `${{ parameters.databaseName }}`
          **Team:** ${{ parameters.team }}
          **Environment:** ${{ parameters.environment }}
          **Size:** ${{ parameters.size }}
          **HA:** ${{ parameters.highAvailability }}
          
          ### What this creates:
          - PostgreSQL StatefulSet in `${{ parameters.environment }}-${{ parameters.team }}-db` namespace
          - Automated backups (${{ parameters.backupSchedule }})
          - Prometheus monitoring
          - Grafana dashboard
          - PagerDuty alerts
          
          Provisioned via Backstage by ${{ user.entity.metadata.name }}
        sourcePath: ./gitops-output

    - id: register-catalog
      name: Register in Service Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps['publish-gitops'].output.remoteUrl }}
        catalogInfoPath: '/catalog-info.yaml'

  output:
    links:
      - title: Pull Request
        url: ${{ steps['publish-gitops'].output.remoteUrl }}
      - title: Grafana Dashboard
        url: https://grafana.company.internal/d/database-${{ parameters.databaseName }}
      - title: Documentation
        icon: docs
        entityRef: ${{ steps['register-catalog'].output.entityRef }}
```

### 2.5 Template Skeleton (ไฟล์ที่สร้างอัตโนมัติ)

```yaml
# templates/new-postgresql-database/skeleton/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: ${{ values.environment }}-${{ values.team }}-db

resources:
  - namespace.yaml
  - statefulset.yaml
  - services.yaml
  - configmap.yaml
  - external-secret.yaml
  - podmonitor.yaml
  {% if values.backupSchedule != 'disabled' %}
  - backup-cronjob.yaml
  {% endif %}
  {% if values.highAvailability %}
  - replica-statefulset.yaml
  {% endif %}
```

---

## 3. Database-as-a-Service Patterns

### 3.1 Shared Cluster with Namespace Isolation

```
Kubernetes Cluster
├── namespace: dev-checkout-db
│   ├── PostgreSQL (checkout-dev)
│   └── Credentials (Vault)
├── namespace: dev-catalog-db
│   ├── PostgreSQL (catalog-dev)
│   └── Credentials (Vault)
├── namespace: prod-checkout-db
│   ├── PostgreSQL HA (checkout-prod)
│   ├── 2 Replicas
│   └── Credentials (Vault)
└── namespace: prod-catalog-db
    ├── PostgreSQL HA (catalog-prod)
    ├── 2 Replicas
    └── Credentials (Vault)
```

```yaml
# Network Policy: แต่ละ namespace isolate กัน
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-isolation
  namespace: prod-checkout-db
spec:
  podSelector:
    matchLabels:
      app: postgresql
  policyTypes:
    - Ingress
  ingress:
    # อนุญาตแค่ checkout app
    - from:
        - namespaceSelector:
            matchLabels:
              team: checkout
              environment: production
      ports:
        - protocol: TCP
          port: 5432
    # อนุญาต monitoring
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 9187  # postgres_exporter
```

### 3.2 Serverless: PgBouncer + Connection Pooling

```yaml
# pgbouncer.yaml - Connection pooler ที่ share ได้
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
  namespace: database-platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pgbouncer
  template:
    metadata:
      labels:
        app: pgbouncer
    spec:
      containers:
        - name: pgbouncer
          image: pgbouncer/pgbouncer:1.22
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: config
              mountPath: /etc/pgbouncer
      volumes:
        - name: config
          configMap:
            name: pgbouncer-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: pgbouncer-config
  namespace: database-platform
data:
  pgbouncer.ini: |
    [databases]
    # แต่ละ database ของแต่ละ team
    checkout_prod = host=checkout-prod-postgresql.prod-checkout-db.svc port=5432 dbname=checkout
    catalog_prod = host=catalog-prod-postgresql.prod-catalog-db.svc port=5432 dbname=catalog
    orders_prod = host=orders-prod-postgresql.prod-orders-db.svc port=5432 dbname=orders
    
    [pgbouncer]
    listen_port = 5432
    listen_addr = *
    auth_type = md5
    auth_file = /etc/pgbouncer/userlist.txt
    
    # Pool settings
    pool_mode = transaction
    max_client_conn = 10000
    default_pool_size = 25
    min_pool_size = 5
    reserve_pool_size = 5
    reserve_pool_timeout = 3
    
    # Timeouts
    client_idle_timeout = 600
    server_idle_timeout = 600
    query_timeout = 300
    
    # Logging
    log_connections = 1
    log_disconnections = 1
    log_pooler_errors = 1
    stats_period = 60
```

### 3.3 Database Platform Metrics

```go
// platform/metrics/collector.go
package metrics

import (
	"context"
	"time"
	
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
)

var (
	// Provisioning time
	databaseProvisioningDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "database_platform_provisioning_duration_seconds",
			Help:    "Time taken to provision a new database",
			Buckets: []float64{30, 60, 120, 300, 600},
		},
		[]string{"environment", "size", "type"},
	)

	// Migration success rate
	migrationTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name: "database_platform_migration_total",
			Help: "Total number of migration attempts",
		},
		[]string{"environment", "status"},
	)

	// MTTR for database issues
	incidentResolutionDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "database_platform_incident_resolution_seconds",
			Help:    "Time to resolve database incidents",
			Buckets: prometheus.DefBuckets,
		},
		[]string{"severity", "cause"},
	)

	// Active databases by team
	activeDatabases = promauto.NewGaugeVec(
		prometheus.GaugeOpts{
			Name: "database_platform_active_databases",
			Help: "Number of active databases",
		},
		[]string{"team", "environment", "size"},
	)

	// Developer satisfaction (from surveys)
	developerSatisfactionScore = promauto.NewGauge(
		prometheus.GaugeOpts{
			Name: "database_platform_developer_satisfaction_score",
			Help: "Developer satisfaction score (1-10) from quarterly surveys",
		},
	)
)

// RecordProvisioning บันทึก metrics เมื่อ provision database
func RecordProvisioning(env, size, dbType string, duration time.Duration, success bool) {
	databaseProvisioningDuration.WithLabelValues(env, size, dbType).
		Observe(duration.Seconds())
}

// RecordMigration บันทึก metrics เมื่อ run migration
func RecordMigration(env, status string) {
	migrationTotal.WithLabelValues(env, status).Inc()
}
```

---

## 4. Crossplane: Infrastructure Provisioning via Kubernetes

Crossplane ช่วยให้ provision cloud resources ผ่าน Kubernetes CRDs เหมือนกับ Terraform แต่ declarative และ Kubernetes-native

### 4.1 ติดตั้ง Crossplane

```bash
# ติดตั้ง Crossplane
helm repo add crossplane-stable \
  https://charts.crossplane.io/stable
helm install crossplane \
  --namespace crossplane-system \
  --create-namespace \
  crossplane-stable/crossplane

# ติดตั้ง AWS Provider
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/upbound/provider-aws:v0.40.0
EOF

# รอ provider พร้อม
kubectl wait provider.pkg.crossplane.io/provider-aws \
  --for=condition=Healthy \
  --timeout=300s
```

### 4.2 PostgreSQL Composite Resource

```yaml
# crossplane/compositions/postgresql-composition.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xpostgresqlinstances.database.example.com
  labels:
    provider: aws
    service: rds
spec:
  compositeTypeRef:
    apiVersion: database.example.com/v1alpha1
    kind: XPostgreSQLInstance

  resources:
    # RDS Subnet Group
    - name: subnet-group
      base:
        apiVersion: rds.aws.upbound.io/v1beta1
        kind: SubnetGroup
        spec:
          forProvider:
            region: us-east-1
            description: Subnet group for PostgreSQL
            subnetIdSelector:
              matchLabels:
                access: private
      patches:
        - fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.region

    # RDS Parameter Group
    - name: parameter-group
      base:
        apiVersion: rds.aws.upbound.io/v1beta1
        kind: ParameterGroup
        spec:
          forProvider:
            region: us-east-1
            family: postgres15
            description: PostgreSQL 15 parameters
            parameter:
              - name: shared_buffers
                value: "{DBInstanceClassMemory/4}"
              - name: max_connections
                value: "200"
              - name: log_min_duration_statement
                value: "1000"
              - name: pg_stat_statements.track
                value: "all"

    # RDS Instance
    - name: rds-instance
      base:
        apiVersion: rds.aws.upbound.io/v1beta1
        kind: Instance
        spec:
          forProvider:
            region: us-east-1
            engine: postgres
            engineVersion: "15"
            skipFinalSnapshot: false
            publiclyAccessible: false
            storageEncrypted: true
            multiAz: false
            backupRetentionPeriod: 7
            deletionProtection: true
            enabledCloudwatchLogsExports:
              - postgresql
              - upgrade
            performanceInsightsEnabled: true
            performanceInsightsRetentionPeriod: 7
            monitoringInterval: 60
            autoMinorVersionUpgrade: true
      patches:
        - fromFieldPath: spec.parameters.size
          toFieldPath: spec.forProvider.instanceClass
          transforms:
            - type: map
              map:
                small: db.t3.medium
                medium: db.r6g.xlarge
                large: db.r6g.2xlarge
                xlarge: db.r6g.4xlarge
        - fromFieldPath: spec.parameters.storageGB
          toFieldPath: spec.forProvider.allocatedStorage
        - fromFieldPath: spec.parameters.multiAz
          toFieldPath: spec.forProvider.multiAz

---
# XRD: Custom Resource Definition สำหรับ Platform
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xpostgresqlinstances.database.example.com
spec:
  group: database.example.com
  names:
    kind: XPostgreSQLInstance
    plural: xpostgresqlinstances
  claimNames:
    kind: PostgreSQLInstance
    plural: postgresqlinstances
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  required:
                    - size
                    - region
                  properties:
                    size:
                      type: string
                      enum: [small, medium, large, xlarge]
                    region:
                      type: string
                    storageGB:
                      type: integer
                      minimum: 20
                      maximum: 65536
                      default: 100
                    multiAz:
                      type: boolean
                      default: false
                    backupRetentionDays:
                      type: integer
                      minimum: 0
                      maximum: 35
                      default: 7
```

### 4.3 Developer สร้าง Database ด้วย Crossplane Claim

```yaml
# Developer ต้องรู้แค่นี้!
apiVersion: database.example.com/v1alpha1
kind: PostgreSQLInstance
metadata:
  name: checkout-database
  namespace: checkout-team
spec:
  parameters:
    size: medium
    region: us-east-1
    storageGB: 200
    multiAz: true
    backupRetentionDays: 14
  writeConnectionSecretToRef:
    name: checkout-db-secret
    namespace: checkout-team

# Crossplane จะสร้าง:
# - RDS Subnet Group
# - RDS Parameter Group
# - RDS Instance (db.r6g.xlarge, Multi-AZ)
# - เก็บ connection string ใน Secret
```

---

## 5. Terraform Modules: Reusable Database Infrastructure

### 5.1 Module: rds-postgresql

```hcl
# modules/rds-postgresql/main.tf

variable "name" {
  description = "Database name"
  type        = string
}

variable "team" {
  description = "Team name"
  type        = string
}

variable "environment" {
  description = "Environment (dev/staging/prod)"
  type        = string
}

variable "size" {
  description = "Database size tier"
  type        = string
  default     = "small"
  
  validation {
    condition     = contains(["small", "medium", "large", "xlarge"], var.size)
    error_message = "Size must be small, medium, large, or xlarge."
  }
}

variable "multi_az" {
  description = "Enable Multi-AZ deployment"
  type        = bool
  default     = false
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "private_subnet_ids" {
  description = "List of private subnet IDs"
  type        = list(string)
}

# Instance size mapping
locals {
  instance_class_map = {
    small  = "db.t3.medium"
    medium = "db.r6g.xlarge"
    large  = "db.r6g.2xlarge"
    xlarge = "db.r6g.4xlarge"
  }
  
  storage_map = {
    small  = 50
    medium = 200
    large  = 500
    xlarge = 1000
  }
  
  common_tags = {
    Name        = "${var.team}-${var.name}-${var.environment}"
    Team        = var.team
    Environment = var.environment
    ManagedBy   = "terraform"
    Module      = "rds-postgresql"
  }
}

# Security Group
resource "aws_security_group" "rds" {
  name_prefix = "${var.team}-${var.name}-"
  description = "Security group for ${var.team} ${var.name} RDS"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = var.allowed_security_group_ids
    description     = "Allow PostgreSQL from application"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = local.common_tags
}

# DB Subnet Group
resource "aws_db_subnet_group" "main" {
  name_prefix = "${var.team}-${var.name}-"
  subnet_ids  = var.private_subnet_ids
  tags        = local.common_tags
}

# DB Parameter Group
resource "aws_db_parameter_group" "main" {
  name_prefix = "${var.team}-${var.name}-"
  family      = "postgres15"
  description = "Custom parameters for ${var.team} ${var.name}"

  parameter {
    name  = "shared_buffers"
    value = "{DBInstanceClassMemory/4}"
  }

  parameter {
    name  = "max_connections"
    value = var.environment == "prod" ? "200" : "100"
  }

  parameter {
    name  = "log_min_duration_statement"
    value = var.environment == "prod" ? "1000" : "500"
  }

  parameter {
    name  = "pg_stat_statements.track"
    value = "all"
  }

  parameter {
    name  = "auto_explain.log_min_duration"
    value = "3000"
  }

  tags = local.common_tags
}

# KMS Key for encryption
resource "aws_kms_key" "rds" {
  description             = "KMS key for ${var.team} ${var.name} RDS"
  deletion_window_in_days = 7
  enable_key_rotation     = true
  tags                    = local.common_tags
}

# Random password
resource "random_password" "master" {
  length           = 32
  special          = true
  override_special = "!#$%^&*"
}

# Store password in Secrets Manager
resource "aws_secretsmanager_secret" "rds" {
  name_prefix             = "${var.team}/${var.name}/${var.environment}/"
  description             = "RDS credentials for ${var.team} ${var.name}"
  recovery_window_in_days = 7
  kms_key_id              = aws_kms_key.rds.arn
  tags                    = local.common_tags
}

resource "aws_secretsmanager_secret_version" "rds" {
  secret_id = aws_secretsmanager_secret.rds.id
  secret_string = jsonencode({
    username = "master"
    password = random_password.master.result
    host     = aws_db_instance.main.address
    port     = aws_db_instance.main.port
    dbname   = replace(var.name, "-", "_")
    url      = "postgresql://master:${random_password.master.result}@${aws_db_instance.main.address}:5432/${replace(var.name, "-", "_")}"
  })
}

# RDS Instance
resource "aws_db_instance" "main" {
  identifier_prefix = "${var.team}-${var.name}-"

  # Engine
  engine               = "postgres"
  engine_version       = "15.4"
  instance_class       = local.instance_class_map[var.size]

  # Storage
  allocated_storage     = local.storage_map[var.size]
  max_allocated_storage = local.storage_map[var.size] * 3
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds.arn

  # Database
  db_name  = replace(var.name, "-", "_")
  username = "master"
  password = random_password.master.result

  # Network
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  publicly_accessible    = false
  multi_az               = var.multi_az || var.environment == "prod"

  # Backup
  backup_retention_period   = var.environment == "prod" ? 14 : 7
  backup_window             = "03:00-04:00"
  maintenance_window        = "mon:04:00-mon:05:00"
  delete_automated_backups  = var.environment != "prod"
  skip_final_snapshot       = var.environment != "prod"
  final_snapshot_identifier = var.environment == "prod" ? "${var.team}-${var.name}-final" : null

  # Parameter group
  parameter_group_name = aws_db_parameter_group.main.name

  # Monitoring
  monitoring_interval             = 60
  monitoring_role_arn             = aws_iam_role.rds_monitoring.arn
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  performance_insights_enabled    = true
  performance_insights_kms_key_id = aws_kms_key.rds.arn

  # Protection
  deletion_protection       = var.environment == "prod"
  auto_minor_version_upgrade = true
  copy_tags_to_snapshot     = true

  tags = local.common_tags

  lifecycle {
    prevent_destroy = true
  }
}

# IAM Role for Enhanced Monitoring
resource "aws_iam_role" "rds_monitoring" {
  name_prefix = "${var.team}-${var.name}-rds-monitoring-"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "monitoring.rds.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "rds_monitoring" {
  role       = aws_iam_role.rds_monitoring.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole"
}

# Outputs
output "instance_id" {
  value = aws_db_instance.main.id
}

output "endpoint" {
  value = aws_db_instance.main.endpoint
}

output "secret_arn" {
  description = "ARN of Secrets Manager secret containing credentials"
  value       = aws_secretsmanager_secret.rds.arn
}
```

### 5.2 Module: elasticache-redis

```hcl
# modules/elasticache-redis/main.tf

variable "name" { type = string }
variable "team" { type = string }
variable "environment" { type = string }

variable "size" {
  type    = string
  default = "small"
}

variable "cluster_mode" {
  description = "Enable Redis Cluster mode"
  type        = bool
  default     = false
}

variable "num_shards" {
  description = "Number of shards (cluster mode)"
  type        = number
  default     = 1
}

locals {
  node_type_map = {
    small  = "cache.t3.medium"
    medium = "cache.r6g.large"
    large  = "cache.r6g.xlarge"
    xlarge = "cache.r6g.2xlarge"
  }
}

resource "aws_elasticache_replication_group" "main" {
  replication_group_id       = "${var.team}-${var.name}"
  description                = "Redis for ${var.team} ${var.name}"
  
  node_type            = local.node_type_map[var.size]
  port                 = 6379
  
  # High availability
  num_cache_clusters   = var.environment == "prod" ? 3 : 1
  automatic_failover_enabled = var.environment == "prod"
  multi_az_enabled     = var.environment == "prod"
  
  # Security
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = random_password.redis.result
  
  # Cluster mode
  cluster_mode {
    num_node_groups         = var.cluster_mode ? var.num_shards : 1
    replicas_per_node_group = var.environment == "prod" ? 2 : 0
  }
  
  # Maintenance
  maintenance_window         = "sun:05:00-sun:06:00"
  snapshot_retention_limit   = var.environment == "prod" ? 7 : 1
  snapshot_window            = "04:00-05:00"
  auto_minor_version_upgrade = true
  
  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]
  
  log_delivery_configuration {
    destination      = aws_cloudwatch_log_group.redis.name
    destination_type = "cloudwatch-logs"
    log_format       = "json"
    log_type         = "slow-log"
  }
  
  tags = {
    Team        = var.team
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "random_password" "redis" {
  length  = 32
  special = false
}

output "primary_endpoint" {
  value = aws_elasticache_replication_group.main.primary_endpoint_address
}

output "reader_endpoint" {
  value = aws_elasticache_replication_group.main.reader_endpoint_address
}
```

---

## 6. Backstage + Crossplane: Full Implementation

### 6.1 Backstage Plugin สำหรับ Database Status

```typescript
// plugins/database-platform/src/components/DatabaseList/DatabaseList.tsx
import React, { useEffect, useState } from 'react';
import {
  Table,
  TableColumn,
  Progress,
  StatusOK,
  StatusError,
  StatusWarning,
} from '@backstage/core-components';
import { useApi, configApiRef } from '@backstage/core-plugin-api';

interface Database {
  name: string;
  team: string;
  environment: string;
  size: string;
  status: 'healthy' | 'degraded' | 'down';
  connections: number;
  maxConnections: number;
  storageUsed: number;
  storageTotal: number;
  replicationLag: number;
  lastBackup: string;
}

const StatusBadge = ({ status }: { status: string }) => {
  switch (status) {
    case 'healthy':
      return <StatusOK>Healthy</StatusOK>;
    case 'degraded':
      return <StatusWarning>Degraded</StatusWarning>;
    default:
      return <StatusError>Down</StatusError>;
  }
};

export const DatabaseList = () => {
  const [databases, setDatabases] = useState<Database[]>([]);
  const [loading, setLoading] = useState(true);
  const configApi = useApi(configApiRef);

  useEffect(() => {
    const fetchDatabases = async () => {
      try {
        const backendUrl = configApi.getString('backend.baseUrl');
        const response = await fetch(
          `${backendUrl}/api/database-platform/databases`
        );
        const data = await response.json();
        setDatabases(data.databases);
      } finally {
        setLoading(false);
      }
    };

    fetchDatabases();
    const interval = setInterval(fetchDatabases, 30000); // refresh ทุก 30 วินาที
    return () => clearInterval(interval);
  }, [configApi]);

  const columns: TableColumn[] = [
    {
      title: 'Database',
      field: 'name',
      render: (row: any) => (
        <a href={`/database-platform/databases/${row.name}`}>
          {row.name}
        </a>
      ),
    },
    { title: 'Team', field: 'team' },
    { title: 'Environment', field: 'environment' },
    { title: 'Size', field: 'size' },
    {
      title: 'Status',
      field: 'status',
      render: (row: any) => <StatusBadge status={row.status} />,
    },
    {
      title: 'Connections',
      field: 'connections',
      render: (row: any) => (
        <span>
          {row.connections}/{row.maxConnections}{' '}
          ({Math.round((row.connections / row.maxConnections) * 100)}%)
        </span>
      ),
    },
    {
      title: 'Storage',
      field: 'storageUsed',
      render: (row: any) => (
        <span>
          {row.storageUsed}GB/{row.storageTotal}GB{' '}
          ({Math.round((row.storageUsed / row.storageTotal) * 100)}%)
        </span>
      ),
    },
    {
      title: 'Last Backup',
      field: 'lastBackup',
      render: (row: any) => new Date(row.lastBackup).toLocaleString(),
    },
  ];

  if (loading) return <Progress />;

  return (
    <Table
      title="Database Platform"
      options={{ search: true, paging: true, pageSize: 20 }}
      columns={columns}
      data={databases}
    />
  );
};
```

### 6.2 Backend API สำหรับ Backstage Plugin

```typescript
// plugins/database-platform-backend/src/router.ts
import { Router } from 'express';
import { Logger } from 'winston';
import { PrometheusClient } from './prometheus';
import { KubernetesClient } from './kubernetes';

export function createRouter(options: {
  logger: Logger;
  prometheus: PrometheusClient;
  kubernetes: KubernetesClient;
}): Router {
  const router = Router();
  const { logger, prometheus, kubernetes } = options;

  // ดึง list ของทุก databases
  router.get('/databases', async (req, res) => {
    try {
      // ดึงจาก Kubernetes (Crossplane claims)
      const claims = await kubernetes.listResources({
        group: 'database.example.com',
        version: 'v1alpha1',
        plural: 'postgresqlinstances',
        namespace: undefined, // all namespaces
      });

      const databases = await Promise.all(
        claims.map(async (claim: any) => {
          const metrics = await prometheus.queryDatabase(
            claim.metadata.name
          );
          
          return {
            name: claim.metadata.name,
            team: claim.metadata.labels?.team || 'unknown',
            environment: claim.metadata.labels?.environment || 'unknown',
            size: claim.spec?.parameters?.size || 'unknown',
            status: determineStatus(metrics),
            connections: metrics.connections || 0,
            maxConnections: metrics.maxConnections || 100,
            storageUsed: metrics.storageUsedGB || 0,
            storageTotal: claim.spec?.parameters?.storageGB || 100,
            replicationLag: metrics.replicationLagSeconds || 0,
            lastBackup: metrics.lastBackupTime || null,
          };
        })
      );

      res.json({ databases });
    } catch (error) {
      logger.error('Failed to list databases:', error);
      res.status(500).json({ error: 'Failed to list databases' });
    }
  });

  // ดึง details ของ database เดียว
  router.get('/databases/:name', async (req, res) => {
    const { name } = req.params;
    
    try {
      const [claim, metrics, slowQueries] = await Promise.all([
        kubernetes.getResource({
          group: 'database.example.com',
          version: 'v1alpha1',
          plural: 'postgresqlinstances',
          name,
        }),
        prometheus.queryDatabaseDetail(name),
        prometheus.getSlowQueries(name),
      ]);

      res.json({
        name,
        claim,
        metrics,
        slowQueries,
      });
    } catch (error) {
      logger.error(`Failed to get database ${name}:`, error);
      res.status(500).json({ error: 'Failed to get database details' });
    }
  });

  // สร้าง database ใหม่
  router.post('/databases', async (req, res) => {
    const { name, team, environment, size, multiAz } = req.body;
    
    // Validate input
    if (!name || !team || !environment) {
      return res.status(400).json({ error: 'name, team, environment required' });
    }
    
    try {
      // สร้าง Crossplane Claim
      await kubernetes.createResource({
        apiVersion: 'database.example.com/v1alpha1',
        kind: 'PostgreSQLInstance',
        metadata: {
          name: `${team}-${name}`,
          namespace: `${environment}-${team}-db`,
          labels: {
            team,
            environment,
            'created-by': req.user?.name || 'backstage',
          },
        },
        spec: {
          parameters: {
            size: size || 'small',
            region: 'us-east-1',
            multiAz: multiAz || environment === 'production',
          },
          writeConnectionSecretToRef: {
            name: `${team}-${name}-credentials`,
            namespace: `${environment}-${team}-db`,
          },
        },
      });

      logger.info(`Database ${name} created for team ${team} in ${environment}`);
      
      res.status(201).json({
        message: `Database ${name} provisioning started`,
        estimatedTime: '5 minutes',
      });
    } catch (error) {
      logger.error('Failed to create database:', error);
      res.status(500).json({ error: 'Failed to create database' });
    }
  });

  return router;
}

function determineStatus(metrics: any): 'healthy' | 'degraded' | 'down' {
  if (!metrics.isUp) return 'down';
  if (
    metrics.replicationLagSeconds > 30 ||
    metrics.connections / metrics.maxConnections > 0.9 ||
    metrics.storageUsedPercent > 90
  ) {
    return 'degraded';
  }
  return 'healthy';
}
```

---

## 7. Database Platform Metrics Dashboard

```json
{
  "dashboard": {
    "title": "Database Platform Overview",
    "panels": [
      {
        "title": "Total Databases by Environment",
        "type": "stat",
        "targets": [
          {
            "expr": "count(database_platform_active_databases) by (environment)"
          }
        ]
      },
      {
        "title": "Provisioning Time (p50/p95/p99)",
        "type": "timeseries",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, database_platform_provisioning_duration_seconds_bucket)",
            "legendFormat": "p50"
          },
          {
            "expr": "histogram_quantile(0.95, database_platform_provisioning_duration_seconds_bucket)",
            "legendFormat": "p95"
          },
          {
            "expr": "histogram_quantile(0.99, database_platform_provisioning_duration_seconds_bucket)",
            "legendFormat": "p99"
          }
        ]
      },
      {
        "title": "Migration Success Rate (7d)",
        "type": "gauge",
        "targets": [
          {
            "expr": "sum(rate(database_platform_migration_total{status='success'}[7d])) / sum(rate(database_platform_migration_total[7d])) * 100"
          }
        ],
        "fieldConfig": {
          "min": 0,
          "max": 100,
          "thresholds": {
            "steps": [
              { "color": "red", "value": 0 },
              { "color": "yellow", "value": 95 },
              { "color": "green", "value": 99 }
            ]
          }
        }
      },
      {
        "title": "Developer Satisfaction Score",
        "type": "stat",
        "targets": [
          {
            "expr": "database_platform_developer_satisfaction_score"
          }
        ]
      }
    ]
  }
}
```

---

## สรุป Platform Engineering สำหรับ Database

| ความสามารถ | ก่อน | หลัง |
|------------|------|------|
| Provisioning time | 2-3 วัน | 5 นาที |
| Self-service | ไม่มี | Backstage portal |
| Consistency | ต่าง config | Crossplane/Terraform modules |
| Visibility | ไม่รู้ว่ามีอะไรบ้าง | Backstage catalog |
| Cost allocation | ไม่ชัดเจน | Tags ทุก resource |
| Compliance | manual check | automated policy |
| On-call | DBA ทำทุกอย่าง | Runbooks + automation |

Platform Engineering ทำให้ developer ทำงานได้เร็วขึ้น โดยไม่ต้องรู้ detail ของ infrastructure ขณะที่ platform team สามารถ enforce standards ได้โดยอัตโนมัติ
