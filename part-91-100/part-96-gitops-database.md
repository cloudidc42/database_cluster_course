# Part 96: GitOps สำหรับ Database

## บทนำ: ทำไม GitOps ถึงสำคัญสำหรับ Database?

ในยุคของ Cloud-Native Applications การจัดการ Database Configuration และ Schema ด้วยมือกลายเป็นปัญหาใหญ่ เมื่อทีมมีขนาดใหญ่ขึ้น การ deploy บ่อยขึ้น และ environment มีหลายชั้น (dev, staging, prod) การติดตามว่า Database อยู่ในสถานะอะไร ใครเปลี่ยนอะไร และเมื่อไหร่ กลายเป็นฝันร้าย

**GitOps** แก้ปัญหานี้โดยใช้ Git เป็น Single Source of Truth สำหรับทุกอย่าง รวมถึง Database Configuration

---

## 1. GitOps คืออะไร?

GitOps เป็น operational framework ที่ใช้ Git best practices สำหรับ infrastructure automation โดยมีหลักการ 4 ข้อ:

### หลักการที่ 1: Declarative (ประกาศ Desired State)

แทนที่จะบอกว่า "ทำสิ่งนี้" (imperative) ให้บอกว่า "ต้องการให้เป็นแบบนี้" (declarative)

```yaml
# ❌ Imperative: บอกวิธีทำ
kubectl create deployment postgres --image=postgres:15
kubectl scale deployment postgres --replicas=3
kubectl expose deployment postgres --port=5432

# ✅ Declarative: บอกว่าต้องการอะไร
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
```

### หลักการที่ 2: Versioned and Immutable (Git เป็น Single Source of Truth)

ทุก configuration เก็บใน Git พร้อม history สมบูรณ์

```bash
# ดู history ของ database configuration
git log --oneline -- kubernetes/database/

# ดูว่า schema เปลี่ยนเมื่อไหร่
git log --oneline -- migrations/

# ดูความแตกต่างระหว่าง versions
git diff v1.0.0..v1.1.0 -- migrations/

# rollback ง่ายด้วย git revert
git revert abc123  # revert specific migration
```

### หลักการที่ 3: Pulled Automatically (Agent Pull จาก Git)

Agent (เช่น ArgoCD) คอยดู Git repository และ pull changes ลงมา apply อัตโนมัติ ไม่ต้อง push เข้า cluster

```
Git Repository          ArgoCD Agent            Kubernetes Cluster
      │                       │                        │
      │  ← Poll every 3 min  │                        │
      │                       │                        │
  [commit]                    │                        │
      │                       │                        │
      │  ← Detect changes ─── │                        │
      │                       │                        │
      │                       │  ── Apply changes ──→  │
      │                       │                        │
      │                       │  ← Report status ───   │
```

### หลักการที่ 4: Continuously Reconciled (ตรวจจับ Drift)

หาก state ใน cluster แตกต่างจาก Git (drift), agent จะแก้ไขให้ตรงโดยอัตโนมัติ

```yaml
# ArgoCD จะตรวจสอบ actual state vs desired state
# และแก้ไขให้ตรง (หรือ alert) เสมอ
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  syncPolicy:
    automated:
      selfHeal: true  # แก้ drift อัตโนมัติ
      prune: true     # ลบ resources ที่ไม่มีใน Git
```

---

## 2. ArgoCD: GitOps for Kubernetes

ArgoCD เป็น declarative GitOps continuous delivery tool สำหรับ Kubernetes

### 2.1 การติดตั้ง ArgoCD

```bash
# สร้าง namespace
kubectl create namespace argocd

# ติดตั้ง ArgoCD
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# รอ pods ขึ้น
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd \
  --timeout=300s

# ดู initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Port forward เพื่อเข้า UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Login ด้วย CLI
argocd login localhost:8080 \
  --username admin \
  --password <password> \
  --insecure
```

### 2.2 โครงสร้าง Repository สำหรับ GitOps

```
gitops-repo/
├── apps/
│   ├── dev/
│   │   ├── database/
│   │   │   ├── postgresql.yaml
│   │   │   ├── redis.yaml
│   │   │   └── migrations-job.yaml
│   │   └── api/
│   │       └── deployment.yaml
│   ├── staging/
│   │   └── database/
│   │       ├── postgresql.yaml  # staging config
│   │       └── migrations-job.yaml
│   └── prod/
│       └── database/
│           ├── postgresql.yaml  # prod config (HA)
│           └── migrations-job.yaml
├── base/
│   ├── postgresql/
│   │   ├── statefulset.yaml
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   └── kustomization.yaml
│   └── redis/
│       ├── statefulset.yaml
│       └── kustomization.yaml
├── overlays/
│   ├── dev/
│   │   └── kustomization.yaml
│   ├── staging/
│   │   └── kustomization.yaml
│   └── prod/
│       └── kustomization.yaml
├── migrations/
│   ├── V001__create_users.sql
│   ├── V002__create_products.sql
│   └── V003__add_indexes.sql
└── argocd/
    ├── projects/
    │   └── database-project.yaml
    └── applications/
        ├── dev-database.yaml
        ├── staging-database.yaml
        └── prod-database.yaml
```

### 2.3 ArgoCD Application CRD

```yaml
# argocd/applications/prod-database.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prod-database
  namespace: argocd
  labels:
    environment: production
    component: database
  finalizers:
    - resources-finalizer.argocd.argoproj.io  # ลบ resources เมื่อ delete app
spec:
  project: database-project

  source:
    repoURL: https://github.com/mycompany/gitops-repo.git
    targetRevision: main
    path: apps/prod/database

  destination:
    server: https://kubernetes.default.svc
    namespace: prod-database

  syncPolicy:
    automated:
      prune: true      # ลบ resources ที่ไม่มีใน Git
      selfHeal: true   # แก้ drift อัตโนมัติ
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - RespectIgnoreDifferences=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  # ไม่ต้อง sync ส่วนนี้ (managed โดย runtime)
  ignoreDifferences:
    - group: apps
      kind: StatefulSet
      jsonPointers:
        - /spec/volumeClaimTemplates
    - group: ""
      kind: Secret
      jsonPointers:
        - /data
```

### 2.4 ArgoCD Project: จำกัด Permissions

```yaml
# argocd/projects/database-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: database-project
  namespace: argocd
spec:
  description: "Database cluster management"

  # อนุญาต deploy จาก repo นี้เท่านั้น
  sourceRepos:
    - https://github.com/mycompany/gitops-repo.git

  # อนุญาต deploy ไป namespaces เหล่านี้
  destinations:
    - namespace: dev-database
      server: https://kubernetes.default.svc
    - namespace: staging-database
      server: https://kubernetes.default.svc
    - namespace: prod-database
      server: https://kubernetes.default.svc

  # อนุญาต resource types เหล่านี้
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace

  namespaceResourceWhitelist:
    - group: apps
      kind: StatefulSet
    - group: apps
      kind: Deployment
    - group: batch
      kind: Job
    - group: ""
      kind: Service
    - group: ""
      kind: ConfigMap
    - group: ""
      kind: PersistentVolumeClaim

  # RBAC สำหรับ project
  roles:
    - name: db-admin
      description: Database administrators
      policies:
        - p, proj:database-project:db-admin, applications, *, database-project/*, allow
      groups:
        - mycompany:db-admins
    - name: developer
      description: Application developers
      policies:
        - p, proj:database-project:developer, applications, get, database-project/*, allow
        - p, proj:database-project:developer, applications, sync, database-project/dev-*, allow
      groups:
        - mycompany:developers
```

### 2.5 Sync Policies

**Manual Sync**: ต้อง approve ก่อน deploy

```bash
# ดู out-of-sync applications
argocd app list --sync-status OutOfSync

# ดู diff ก่อน sync
argocd app diff prod-database

# Sync manually
argocd app sync prod-database

# Sync และรอให้เสร็จ
argocd app sync prod-database --timeout 300

# Sync specific resource
argocd app sync prod-database --resource apps:StatefulSet:postgresql
```

**Auto Sync**: ตรวจจับและ deploy อัตโนมัติ

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
    allowEmpty: false  # ไม่ sync ถ้า source ว่าง (ป้องกัน accident delete)
```

### 2.6 Auto-Healing: Revert Manual Changes

เมื่อ selfHeal เปิดอยู่ ArgoCD จะตรวจสอบทุก 3 นาที และ revert การเปลี่ยนแปลงที่ไม่ได้มาจาก Git

```bash
# ทดสอบ auto-healing
# 1. Scale down database replica manually
kubectl scale statefulset postgresql --replicas=1 -n prod-database

# 2. รอ ArgoCD ตรวจสอบ (ประมาณ 3 นาที)
# 3. ArgoCD จะ scale กลับเป็น 3 replicas ตาม Git

# ดู healing events
kubectl get events -n prod-database | grep "Self heal"

# ใน ArgoCD logs
kubectl logs -n argocd deployment/argocd-application-controller | grep "selfHeal"
```

### 2.7 ArgoCD Notifications

```yaml
# argocd-notifications-config
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
    username: ArgoCD
    icon: ":argo:"

  template.app-deployed: |
    message: |
      {{if eq .ServiceType "slack"}}:white_check_mark:{{end}}
      Application *{{.app.metadata.name}}* has been successfully synced.
      Revision: {{.app.status.sync.revision}}
      Environment: {{.app.metadata.labels.environment}}

  template.app-sync-failed: |
    message: |
      {{if eq .ServiceType "slack"}}:red_circle:{{end}}
      Application *{{.app.metadata.name}}* sync FAILED.
      Error: {{.app.status.conditions | map .message | join ","}}

  template.database-migration-started: |
    message: |
      {{if eq .ServiceType "slack"}}:hourglass:{{end}}
      Database migration started for *{{.app.metadata.name}}*.

  trigger.on-deployed: |
    - description: Notify when application is synced and healthy
      send: [app-deployed]
      when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'

  trigger.on-sync-failed: |
    - description: Notify on sync failure
      send: [app-sync-failed]
      when: app.status.operationState.phase in ['Error', 'Failed']

---
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
stringData:
  slack-token: "xoxb-your-slack-token"
```

---

## 3. Database Migrations ใน GitOps

### 3.1 ปัญหา: Migrations เป็น Imperative

GitOps ชอบ declarative state แต่ migrations เป็น imperative operations

```
ปัญหา:
  - Migration V001 สร้าง users table
  - Migration V002 เพิ่ม column email
  - ไม่สามารถ "declare" final state ได้ง่ายๆ
  - ต้องรันในลำดับที่ถูกต้อง
  - ต้อง track ว่า run แล้วหรือยัง

วิธีแก้:
  1. Kubernetes Job: รัน migration เป็น one-time job
  2. Init Container: รันก่อน app container เริ่ม
  3. Flyway Operator: CRD สำหรับ migrations
  4. Atlas Operator: modern schema management
```

### 3.2 วิธีที่ 1: Kubernetes Job

```yaml
# migrations/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-v2024-01-15
  namespace: prod-database
  annotations:
    # บอก ArgoCD ให้ sync job ก่อน application
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  ttlSecondsAfterFinished: 3600  # ลบ job หลังสำเร็จ 1 ชม.
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      initContainers:
        - name: wait-for-postgres
          image: postgres:15-alpine
          command:
            - sh
            - -c
            - |
              until pg_isready -h $DB_HOST -p $DB_PORT -U $DB_USER; do
                echo "Waiting for PostgreSQL..."
                sleep 2
              done
          env:
            - name: DB_HOST
              value: postgresql-service
            - name: DB_PORT
              value: "5432"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
      containers:
        - name: flyway-migration
          image: flyway/flyway:10-alpine
          command:
            - flyway
            - -url=jdbc:postgresql://$(DB_HOST):$(DB_PORT)/$(DB_NAME)
            - -user=$(DB_USER)
            - -password=$(DB_PASSWORD)
            - -locations=filesystem:/migrations
            - migrate
          env:
            - name: DB_HOST
              value: postgresql-service
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: database
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
          volumeMounts:
            - name: migrations
              mountPath: /migrations
      volumes:
        - name: migrations
          configMap:
            name: db-migrations
```

```yaml
# migrations ConfigMap สำหรับ SQL files
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-migrations
  namespace: prod-database
data:
  V001__create_users.sql: |
    CREATE TABLE IF NOT EXISTS users (
      id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      email VARCHAR(255) UNIQUE NOT NULL,
      username VARCHAR(100) UNIQUE NOT NULL,
      password_hash VARCHAR(255) NOT NULL,
      created_at TIMESTAMP DEFAULT NOW(),
      updated_at TIMESTAMP DEFAULT NOW()
    );
    
    CREATE INDEX idx_users_email ON users(email);
    CREATE INDEX idx_users_username ON users(username);

  V002__create_products.sql: |
    CREATE TABLE IF NOT EXISTS products (
      id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      name VARCHAR(500) NOT NULL,
      description TEXT,
      price DECIMAL(12, 2) NOT NULL,
      stock_quantity INTEGER NOT NULL DEFAULT 0,
      category_id UUID,
      created_at TIMESTAMP DEFAULT NOW(),
      updated_at TIMESTAMP DEFAULT NOW()
    );
    
    CREATE INDEX idx_products_category ON products(category_id);
    CREATE INDEX idx_products_price ON products(price);

  V003__add_indexes.sql: |
    -- Performance indexes
    CREATE INDEX CONCURRENTLY IF NOT EXISTS 
      idx_products_name_gin ON products USING gin(to_tsvector('english', name));
    
    CREATE INDEX CONCURRENTLY IF NOT EXISTS 
      idx_products_created_at ON products(created_at DESC);
```

### 3.3 วิธีที่ 2: Init Container

```yaml
# deployment ที่มี init container รัน migration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: prod-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      initContainers:
        # Init container 1: รอ database พร้อม
        - name: wait-for-db
          image: postgres:15-alpine
          command:
            - sh
            - -c
            - |
              until pg_isready -h postgresql.prod-database.svc.cluster.local -p 5432; do
                echo "$(date) - Waiting for database..."
                sleep 3
              done
              echo "Database is ready!"
        
        # Init container 2: รัน migration
        - name: run-migrations
          image: mycompany/api-server:latest
          command:
            - node
            - dist/migrate.js
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url
            - name: MIGRATION_LOCK_TIMEOUT
              value: "60000"
      
      containers:
        - name: api-server
          image: mycompany/api-server:latest
          ports:
            - containerPort: 3000
```

```javascript
// migrate.js - รัน migrations ก่อน app เริ่ม
const { Pool } = require('pg');
const fs = require('fs');
const path = require('path');
const crypto = require('crypto');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 1
});

async function runMigrations() {
  const client = await pool.connect();
  
  try {
    // สร้าง migrations table ถ้ายังไม่มี
    await client.query(`
      CREATE TABLE IF NOT EXISTS schema_migrations (
        version VARCHAR(255) PRIMARY KEY,
        checksum VARCHAR(64) NOT NULL,
        applied_at TIMESTAMP DEFAULT NOW(),
        execution_time_ms INTEGER
      )
    `);
    
    // Lock เพื่อป้องกัน concurrent migrations
    await client.query('SELECT pg_advisory_lock(12345678)');
    
    // อ่าน migration files
    const migrationsDir = path.join(__dirname, '../migrations');
    const files = fs.readdirSync(migrationsDir)
      .filter(f => f.endsWith('.sql'))
      .sort();
    
    for (const file of files) {
      const version = file.replace('.sql', '');
      
      // ตรวจสอบว่า migration นี้ run แล้วหรือยัง
      const existing = await client.query(
        'SELECT version FROM schema_migrations WHERE version = $1',
        [version]
      );
      
      if (existing.rows.length > 0) {
        console.log(`[SKIP] ${version} - already applied`);
        continue;
      }
      
      const sql = fs.readFileSync(path.join(migrationsDir, file), 'utf8');
      const checksum = crypto.createHash('sha256').update(sql).digest('hex');
      
      const startTime = Date.now();
      
      await client.query('BEGIN');
      try {
        await client.query(sql);
        await client.query(
          'INSERT INTO schema_migrations(version, checksum, execution_time_ms) VALUES($1, $2, $3)',
          [version, checksum, Date.now() - startTime]
        );
        await client.query('COMMIT');
        console.log(`[DONE] ${version} - ${Date.now() - startTime}ms`);
      } catch (err) {
        await client.query('ROLLBACK');
        throw new Error(`Migration ${version} failed: ${err.message}`);
      }
    }
    
    console.log('All migrations completed successfully!');
    
  } finally {
    await client.query('SELECT pg_advisory_unlock(12345678)');
    client.release();
    await pool.end();
  }
}

runMigrations().catch(err => {
  console.error('Migration failed:', err);
  process.exit(1);
});
```

### 3.4 วิธีที่ 3: Atlas Operator

Atlas เป็น modern database schema management tool ที่รองรับ GitOps ได้ดีมาก

```bash
# ติดตั้ง Atlas Operator
helm repo add ariga https://ariga.github.io/helm-charts
helm install atlas-operator ariga/atlas-operator \
  --namespace atlas-operator \
  --create-namespace
```

```yaml
# atlas-schema.yaml - Declarative schema definition
apiVersion: db.atlasgo.io/v1alpha1
kind: AtlasSchema
metadata:
  name: shopcluster-schema
  namespace: prod-database
spec:
  # Secret ที่มี connection string
  credentials:
    secretKeyRef:
      name: atlas-db-credentials
      key: url

  # Schema definition ใน HCL format
  schema:
    hcl: |
      table "users" {
        schema = schema.public
        column "id" {
          type = uuid
          default = sql("gen_random_uuid()")
        }
        column "email" {
          type = varchar(255)
          null = false
        }
        column "username" {
          type = varchar(100)
          null = false
        }
        column "password_hash" {
          type = varchar(255)
          null = false
        }
        column "created_at" {
          type    = timestamp
          default = sql("NOW()")
        }
        primary_key {
          columns = [column.id]
        }
        index "idx_users_email" {
          columns = [column.email]
          unique  = true
        }
        index "idx_users_username" {
          columns = [column.username]
          unique  = true
        }
      }

      table "products" {
        schema = schema.public
        column "id" {
          type = uuid
          default = sql("gen_random_uuid()")
        }
        column "name" {
          type = varchar(500)
          null = false
        }
        column "price" {
          type = decimal(12, 2)
          null = false
        }
        column "stock_quantity" {
          type    = integer
          default = 0
        }
        primary_key {
          columns = [column.id]
        }
      }

  # Atlas จะ generate migration plan
  # และ apply เฉพาะที่แตกต่างจาก current schema
  policy:
    lint:
      # ไม่อนุญาต destructive changes
      destructive:
        error: true
    diff:
      skip:
        drop_table: true  # ไม่ drop table อัตโนมัติ
```

---

## 4. Schema Drift Detection

Schema drift คือเมื่อ actual database schema แตกต่างจาก expected schema ใน Git

### 4.1 Atlas Drift Detection

```yaml
# atlas-drift-check.yaml - CronJob ตรวจสอบ drift ทุกชั่วโมง
apiVersion: batch/v1
kind: CronJob
metadata:
  name: schema-drift-check
  namespace: prod-database
spec:
  schedule: "0 * * * *"  # ทุกชั่วโมง
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: drift-checker
              image: arigaio/atlas:latest
              command:
                - sh
                - -c
                - |
                  # ตรวจสอบ drift
                  atlas schema diff \
                    --from "file:///schemas/schema.hcl" \
                    --to "$DATABASE_URL" \
                    --format "{{ json . }}" > /tmp/drift-result.json
                  
                  # ถ้ามี drift ส่ง alert
                  if [ -s /tmp/drift-result.json ]; then
                    echo "DRIFT DETECTED!"
                    cat /tmp/drift-result.json
                    
                    # ส่ง Slack notification
                    curl -X POST "$SLACK_WEBHOOK" \
                      -H 'Content-type: application/json' \
                      -d "{\"text\": \"⚠️ Schema drift detected in prod-database!\"}"
                    
                    exit 1
                  fi
                  
                  echo "No drift detected."
              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: atlas-db-credentials
                      key: url
                - name: SLACK_WEBHOOK
                  valueFrom:
                    secretKeyRef:
                      name: notification-secrets
                      key: slack-webhook
              volumeMounts:
                - name: schemas
                  mountPath: /schemas
          volumes:
            - name: schemas
              configMap:
                name: expected-schema
```

### 4.2 Prometheus Alert สำหรับ Drift

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: database-drift-alerts
  namespace: monitoring
spec:
  groups:
    - name: database.drift
      interval: 60s
      rules:
        - alert: DatabaseSchemaDrift
          expr: |
            kube_job_status_failed{
              namespace="prod-database",
              job_name=~"schema-drift-check.*"
            } > 0
          for: 5m
          labels:
            severity: critical
            team: database
          annotations:
            summary: "Database schema drift detected"
            description: "The actual database schema differs from the expected schema in Git"
            runbook: "https://wiki.company.com/runbooks/database-drift"

        - alert: MigrationJobFailed
          expr: |
            kube_job_status_failed{
              namespace=~".*-database",
              job_name=~"db-migration.*"
            } > 0
          for: 2m
          labels:
            severity: critical
            team: database
          annotations:
            summary: "Database migration job failed"
            description: "Migration job {{ $labels.job_name }} in {{ $labels.namespace }} failed"
```

---

## 5. Secrets Management ใน GitOps

### 5.1 ปัญหา: Secrets ใน Git

```
❌ อย่าทำแบบนี้:
  git add kubernetes/database/secret.yaml  # มี password ใน plaintext
  
✅ ทำแบบนี้:
  1. Sealed Secrets: encrypt ใน Git
  2. External Secrets Operator: reference จาก Vault
  3. SOPS: encrypt files ด้วย KMS
```

### 5.2 Sealed Secrets: Encrypt ใน Git

```bash
# ติดตั้ง Sealed Secrets Controller
helm repo add sealed-secrets \
  https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  -n kube-system

# ติดตั้ง kubeseal CLI
brew install kubeseal  # macOS
# หรือ
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-0.24.0-linux-amd64.tar.gz
tar xzf kubeseal-*.tar.gz
mv kubeseal /usr/local/bin/
```

```bash
# สร้าง regular secret ก่อน
kubectl create secret generic db-credentials \
  --from-literal=username=shopcluster_user \
  --from-literal=password=SuperSecure!Pass123 \
  --from-literal=database=shopcluster_prod \
  --dry-run=client \
  -o yaml > /tmp/db-credentials.yaml

# Encrypt ด้วย Sealed Secrets
kubeseal \
  --controller-namespace kube-system \
  --controller-name sealed-secrets \
  --format yaml \
  < /tmp/db-credentials.yaml \
  > kubernetes/database/sealed-db-credentials.yaml

# ไฟล์นี้ commit ได้ปลอดภัย!
git add kubernetes/database/sealed-db-credentials.yaml
git commit -m "Add encrypted database credentials"
```

```yaml
# ผลลัพธ์: sealed-db-credentials.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: prod-database
spec:
  encryptedData:
    # ค่าเหล่านี้ encrypt ด้วย public key ของ controller
    username: AgBy8I3D7eXPHOe5s...  # encrypted
    password: AgCkPQ9LmN5xRt7v...   # encrypted
    database: AgB3fY8kMz2wQn...     # encrypted
  template:
    metadata:
      name: db-credentials
      namespace: prod-database
    type: Opaque
```

### 5.3 External Secrets Operator: Vault Integration

```bash
# ติดตั้ง External Secrets Operator
helm repo add external-secrets \
  https://charts.external-secrets.io
helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets-operator \
  --create-namespace
```

```yaml
# vault-secret-store.yaml - เชื่อม ESO กับ Vault
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.company.internal"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets-operator

---
# external-secret.yaml - ดึง secret จาก Vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: prod-database
spec:
  refreshInterval: "15m"  # refresh ทุก 15 นาที
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
    template:
      type: Opaque
      data:
        # transform Vault data เป็น format ที่ต้องการ
        DATABASE_URL: |
          postgresql://{{ .username }}:{{ .password }}@postgresql:5432/{{ .database }}
  data:
    - secretKey: username
      remoteRef:
        key: database/prod-credentials
        property: username
    - secretKey: password
      remoteRef:
        key: database/prod-credentials
        property: password
    - secretKey: database
      remoteRef:
        key: database/prod-credentials
        property: database_name
```

### 5.4 SOPS: Encrypt ด้วย AWS KMS

```bash
# ติดตั้ง SOPS
brew install sops  # macOS

# สร้าง .sops.yaml config
cat > .sops.yaml << 'EOF'
creation_rules:
  - path_regex: kubernetes/.*\.yaml$
    kms: arn:aws:kms:us-east-1:123456789012:key/mrk-abc123def456
  - path_regex: terraform/.*\.tfvars$
    kms: arn:aws:kms:us-east-1:123456789012:key/mrk-abc123def456
EOF

# Encrypt ไฟล์
sops --encrypt kubernetes/database/secrets.yaml \
  > kubernetes/database/secrets.enc.yaml

# Decrypt เมื่อต้องการ
sops --decrypt kubernetes/database/secrets.enc.yaml
```

```yaml
# ArgoCD + SOPS: ใช้ argocd-vault-plugin
# ใน Application spec
spec:
  source:
    repoURL: https://github.com/company/gitops-repo.git
    path: kubernetes/database
    plugin:
      name: argocd-vault-plugin
      env:
        - name: AVP_TYPE
          value: awssecretsmanager
        - name: AWS_REGION
          value: us-east-1
```

---

## 6. Full Pipeline: ArgoCD + Sealed Secrets + Atlas

### 6.1 Pipeline Architecture

```
Developer                Git Repository            ArgoCD              Kubernetes
    │                          │                      │                     │
    │── git commit + push ──→  │                      │                     │
    │   (migration SQL)        │                      │                     │
    │                          │  ← Poll (3 min) ─── │                     │
    │                          │                      │                     │
    │                          │── Detect change ──→  │                     │
    │                          │                      │── PreSync Hook ──→  │
    │                          │                      │   (Migration Job)   │
    │                          │                      │                     │── Run SQL migration
    │                          │                      │                     │── Update flyway_schema_history
    │                          │                      │← Migration done ─── │
    │                          │                      │                     │
    │                          │                      │── Sync App ───────→ │
    │                          │                      │   (Deploy new       │
    │                          │                      │    app version)     │
    │← Slack notification ─── │                      │                     │
```

### 6.2 Complete GitOps Setup

```bash
# scripts/setup-gitops.sh - ติดตั้ง full stack

#!/bin/bash
set -euo pipefail

echo "=== Setting up GitOps for Database Cluster ==="

# 1. สร้าง namespaces
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace prod-database --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace external-secrets-operator --dry-run=client -o yaml | kubectl apply -f -

# 2. ติดตั้ง ArgoCD
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=300s

# 3. ติดตั้ง Sealed Secrets
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm upgrade --install sealed-secrets sealed-secrets/sealed-secrets \
  -n kube-system \
  --set fullnameOverride=sealed-secrets

# 4. ติดตั้ง External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm upgrade --install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets-operator \
  --create-namespace

# 5. ติดตั้ง Atlas Operator
helm repo add ariga https://ariga.github.io/helm-charts
helm upgrade --install atlas-operator ariga/atlas-operator \
  -n atlas-operator \
  --create-namespace

# 6. สร้าง ArgoCD App of Apps
kubectl apply -f argocd/app-of-apps.yaml

echo "=== GitOps setup complete! ==="
echo "ArgoCD UI: https://argocd.company.internal"
```

### 6.3 App of Apps Pattern

```yaml
# argocd/app-of-apps.yaml - Parent app ที่ manage ทุก apps
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prod-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/gitops-repo.git
    targetRevision: main
    path: argocd/applications
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 7. Rollback: Git Revert + ArgoCD Sync

### 7.1 Rollback Process

```bash
# สมมติ migration V003 ทำให้เกิดปัญหา

# 1. ตรวจสอบ history
git log --oneline migrations/
# abc123 V003__add_new_column.sql  ← ปัญหา
# def456 V002__create_products.sql
# ghi789 V001__create_users.sql

# 2. Revert migration file
git revert abc123
# สร้าง commit ใหม่ที่ลบ V003 ออก

# 3. สร้าง rollback migration
cat > migrations/V004__rollback_v003.sql << 'EOF'
-- Rollback V003: remove added column
ALTER TABLE products DROP COLUMN IF EXISTS new_column;
EOF

# 4. Commit และ push
git add migrations/V004__rollback_v003.sql
git commit -m "Rollback V003: remove problematic column"
git push origin main

# 5. ArgoCD จะตรวจพบ change และ run migration job
# ซึ่งจะ execute V004__rollback_v003.sql

# ตรวจสอบ status
argocd app get prod-database
argocd app history prod-database
```

### 7.2 Automated Rollback Script

```bash
#!/bin/bash
# scripts/emergency-rollback.sh

set -euo pipefail

TARGET_REVISION=${1:-HEAD~1}
APP_NAME=${2:-prod-database}

echo "=== Emergency Rollback ==="
echo "Reverting to: $TARGET_REVISION"
echo "Application: $APP_NAME"

# 1. ตรวจสอบว่า revision มีอยู่
if ! git rev-parse "$TARGET_REVISION" > /dev/null 2>&1; then
  echo "ERROR: Revision $TARGET_REVISION not found"
  exit 1
fi

# 2. สร้าง revert commit
git revert --no-edit $TARGET_REVISION
git push origin main

# 3. Force sync ArgoCD ทันที (ไม่รอ polling)
argocd app sync "$APP_NAME" --force

# 4. รอ sync เสร็จ
argocd app wait "$APP_NAME" --timeout 300

# 5. ตรวจสอบสถานะ
STATUS=$(argocd app get "$APP_NAME" -o json | jq -r '.status.health.status')
if [ "$STATUS" != "Healthy" ]; then
  echo "ERROR: Application is not healthy after rollback: $STATUS"
  exit 1
fi

echo "=== Rollback completed successfully! ==="
```

---

## 8. Environment Promotion

### 8.1 Kustomize สำหรับ Multi-Environment

```yaml
# base/postgresql/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - statefulset.yaml
  - service.yaml
  - configmap.yaml

# overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base/postgresql
patches:
  - target:
      kind: StatefulSet
      name: postgresql
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 1
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/memory
        value: "256Mi"
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/memory
        value: "512Mi"
configMapGenerator:
  - name: postgresql-config
    literals:
      - POSTGRES_MAX_CONNECTIONS=50
      - POSTGRES_SHARED_BUFFERS=128MB

# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base/postgresql
patches:
  - target:
      kind: StatefulSet
      name: postgresql
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/memory
        value: "4Gi"
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/memory
        value: "8Gi"
configMapGenerator:
  - name: postgresql-config
    literals:
      - POSTGRES_MAX_CONNECTIONS=200
      - POSTGRES_SHARED_BUFFERS=2GB
      - POSTGRES_EFFECTIVE_CACHE_SIZE=6GB
```

### 8.2 Promotion Flow ด้วย GitHub Actions

```yaml
# .github/workflows/promote-database.yml
name: Promote Database to Environment

on:
  pull_request:
    types: [closed]
    branches:
      - main
    paths:
      - 'migrations/**'
      - 'kubernetes/base/**'

jobs:
  promote-to-staging:
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Update staging image tag
        run: |
          # อัพเดท image tag ใน staging overlay
          NEW_TAG="${{ github.sha }}"
          
          cd overlays/staging
          kustomize edit set image \
            mycompany/api-server=mycompany/api-server:${NEW_TAG}
          
          git config user.email "gitops-bot@company.com"
          git config user.name "GitOps Bot"
          git add .
          git commit -m "Promote staging to ${NEW_TAG}"
          git push

      - name: Wait for staging sync
        run: |
          # รอ ArgoCD sync staging
          argocd app wait staging-database --timeout 300
          argocd app wait staging-api --timeout 300

      - name: Run smoke tests against staging
        run: |
          # รัน smoke tests
          npm run test:smoke -- --env staging

  promote-to-prod:
    needs: promote-to-staging
    runs-on: ubuntu-latest
    environment: production  # ต้องมี manual approval ใน GitHub
    steps:
      - uses: actions/checkout@v4

      - name: Update prod image tag
        run: |
          NEW_TAG="${{ github.sha }}"
          
          cd overlays/prod
          kustomize edit set image \
            mycompany/api-server=mycompany/api-server:${NEW_TAG}
          
          git config user.email "gitops-bot@company.com"
          git config user.name "GitOps Bot"
          git add .
          git commit -m "Promote production to ${NEW_TAG}"
          git push

      - name: Notify team
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: 'C0XXXXX'
          slack-message: |
            :rocket: Production deployment started!
            Version: ${{ github.sha }}
            Deployed by: ${{ github.actor }}
```

---

## 9. Full GitOps Database Cluster Setup

### 9.1 Repository Structure (สมบูรณ์)

```
gitops-database/
├── .github/
│   └── workflows/
│       ├── validate.yml        # ตรวจ syntax, lint
│       ├── promote-dev.yml     # auto-promote เมื่อ merge to main
│       └── promote-prod.yml    # manual-approve ก่อน prod
├── argocd/
│   ├── app-of-apps.yaml
│   ├── projects/
│   │   └── database-project.yaml
│   └── applications/
│       ├── dev-database.yaml
│       ├── staging-database.yaml
│       └── prod-database.yaml
├── base/
│   ├── postgresql/
│   │   ├── statefulset.yaml
│   │   ├── primary-service.yaml
│   │   ├── replica-service.yaml
│   │   ├── configmap.yaml
│   │   └── kustomization.yaml
│   ├── redis/
│   │   ├── statefulset.yaml
│   │   └── kustomization.yaml
│   └── migration-job/
│       ├── job-template.yaml
│       └── kustomization.yaml
├── overlays/
│   ├── dev/
│   ├── staging/
│   └── prod/
├── migrations/
│   ├── V001__initial_schema.sql
│   └── V002__add_indexes.sql
├── schemas/
│   └── expected-schema.hcl     # Atlas schema definition
├── secrets/
│   ├── dev/
│   │   └── sealed-db-creds.yaml
│   ├── staging/
│   │   └── sealed-db-creds.yaml
│   └── prod/
│       └── external-secret.yaml  # ดึงจาก Vault
├── monitoring/
│   ├── prometheus-rules.yaml
│   └── grafana-dashboard.json
└── scripts/
    ├── setup-gitops.sh
    ├── emergency-rollback.sh
    └── validate-schemas.sh
```

### 9.2 ตรวจสอบ Full Setup

```bash
# scripts/validate-schemas.sh - ตรวจสอบทุกอย่างก่อน deploy

#!/bin/bash
set -euo pipefail

echo "=== Validating GitOps Setup ==="

# 1. Validate YAML syntax
echo "Checking YAML syntax..."
find . -name "*.yaml" -not -path "./.git/*" | while read file; do
  if ! kubectl apply --dry-run=client -f "$file" > /dev/null 2>&1; then
    yamllint "$file" || true
  fi
done

# 2. Validate Kustomize builds
echo "Validating Kustomize overlays..."
for env in dev staging prod; do
  kubectl kustomize overlays/$env > /dev/null
  echo "  ✓ $env overlay is valid"
done

# 3. Validate migration files
echo "Validating migration files..."
for sql_file in migrations/*.sql; do
  # ตรวจสอบชื่อไฟล์ format
  if [[ ! "$sql_file" =~ ^migrations/V[0-9]+__.+\.sql$ ]]; then
    echo "ERROR: Invalid migration filename: $sql_file"
    exit 1
  fi
  echo "  ✓ $sql_file"
done

# 4. ตรวจสอบ Sealed Secrets ว่า valid
echo "Checking Sealed Secrets..."
find secrets -name "*.yaml" | while read file; do
  if grep -q "SealedSecret" "$file"; then
    kubectl apply --dry-run=server -f "$file" || true
  fi
done

echo "=== All validations passed! ==="
```

---

## สรุป GitOps สำหรับ Database

| ด้าน | ก่อน GitOps | หลัง GitOps |
|------|-------------|-------------|
| Schema changes | Manual SSH + SQL | PR → Review → Merge → Auto-deploy |
| Audit trail | "ใครทำนะ?" | Git history ครบ |
| Rollback | Manual, เสี่ยง | git revert + auto-sync |
| Environments | ต่าง config | Kustomize overlays, consistent |
| Secrets | Hardcoded / shared | Sealed Secrets / Vault |
| Drift detection | ไม่มี | ArgoCD selfHeal + alerts |
| Approval process | ปากเปล่า | PR review + environment protection |

GitOps เปลี่ยน Database Management จาก "art" เป็น "engineering" - predictable, auditable, automated และ safe
