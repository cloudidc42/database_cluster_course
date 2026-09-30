# Part 95: Database as Code

## บทนำ

"Database as Code" คือ philosophy ที่ treat ทุกอย่างที่เกี่ยวกับ database เหมือนกับ application code นั่นคือ version control ทุกอย่าง, review ก่อน deploy, automated testing, และ reproducible deployments

---

## 1. ทำไมต้อง Database as Code?

### 1.1 ปัญหาแบบเก่า

```markdown
## Anti-patterns ที่พบบ่อย

1. Manual schema changes
   - SSH เข้า production แล้วรัน ALTER TABLE โดยตรง
   - ไม่มี record ว่าใครทำอะไร เมื่อไหร่
   - ไม่สามารถ rollback ได้

2. "Database only I know" syndrome
   - DBA คนเดียวที่รู้ schema จริงๆ
   - ถ้า DBA ลาออก ทุกคนไม่รู้ว่า schema มาจากไหน

3. Environment drift
   - DEV, STAGING, PROD มี schema ต่างกัน
   - "It works on my machine but not in prod"

4. No review process
   - Developer สามารถ ALTER production table โดยไม่ต้องให้ใครตรวจสอบ
   - Bug ใน migration = data corruption

5. Manual seed data
   - ไม่มี reproducible test data
   - Each environment has different reference data
```

### 1.2 Database as Code แก้ปัญหาอย่างไร

```markdown
## Database as Code Principles

1. Everything in Git
   - Schema migrations → Git
   - Stored procedures → Git
   - Seed/reference data → Git
   - Database config → Git

2. No manual changes
   - ทุก change ผ่าน migration script
   - Production ห้าม SSH และ manual ALTER

3. Reproducible
   - ทุก environment ใช้ migration เดียวกัน
   - fresh install ได้ schema เดียวกันเสมอ

4. Reviewed
   - SQL changes ต้องผ่าน PR review
   - DBA/Senior review ก่อน merge

5. Tested
   - Migration ต้องมี test
   - Rollback ต้องทำงาน

6. Automated
   - CI/CD deploy migrations อัตโนมัติ
```

---

## 2. Migration Strategies

### 2.1 State-based vs Change-based

```markdown
## State-based (Declarative)
- นิยาม "desired state" ของ schema
- Tool คำนวณ diff และ generate migration
- ตัวอย่าง: Prisma, Atlas

Pros:
✅ เขียนง่าย: แค่อธิบาย schema ที่ต้องการ
✅ Tool จัดการ order ให้
✅ Drift detection อัตโนมัติ

Cons:
❌ ยาก control exact SQL ที่รัน
❌ Complex migrations อาจ generate ไม่ถูก
❌ ต้องเชื่อ tool ว่า generate migration ถูกต้อง

## Change-based (Imperative)  
- เขียน migration scripts (V001__, V002__)
- รัน script ตามลำดับ
- ตัวอย่าง: Flyway, Liquibase, node-pg-migrate

Pros:
✅ Full control ของ SQL ที่รัน
✅ Complex migrations (data migrations) ทำได้
✅ Explicit: รู้ว่า exact SQL อะไรรัน

Cons:
❌ ต้องเขียน migration เองทุกครั้ง
❌ Order management เป็นความรับผิดชอบของทีม

## เลือกอะไรดี?
- Simple schema: State-based (Prisma/Atlas)
- Complex/legacy systems: Change-based (Flyway/Liquibase)
- Hybrid: เขียน migration เอง + tool ช่วย detect drift
```

---

## 3. Migration Tools

### 3.1 Flyway

```bash
# ติดตั้ง Flyway CLI
curl -L https://download.red-gate.com/maven/release/org/flywaydb/flyway-commandline/9.22.3/flyway-commandline-9.22.3-linux-x64.tar.gz | tar -xz
mv flyway-9.22.3 /opt/flyway
ln -s /opt/flyway/flyway /usr/local/bin/flyway

# Docker
docker pull flyway/flyway:9-alpine
```

**Project Structure:**

```
project/
├── flyway.conf
└── db/
    └── migration/
        ├── V001__initial_schema.sql
        ├── V002__add_users_table.sql
        ├── V003__add_tenant_id.sql
        ├── V004__add_indexes.sql
        ├── R__stored_procedures.sql    # Repeatable
        └── U003__undo_add_tenant_id.sql  # Undo (Enterprise)
```

**flyway.conf:**

```properties
# flyway.conf
flyway.url=jdbc:postgresql://localhost:5432/myapp
flyway.user=postgres
flyway.password=${DB_PASSWORD}
flyway.schemas=public,app,audit
flyway.locations=filesystem:db/migration
flyway.baselineOnMigrate=true
flyway.validateOnMigrate=true
flyway.outOfOrder=false
```

**Migration Files:**

```sql
-- V001__initial_schema.sql
-- Flyway naming: V{version}__{description}.sql

CREATE SCHEMA IF NOT EXISTS app;
CREATE SCHEMA IF NOT EXISTS audit;
CREATE SCHEMA IF NOT EXISTS compliance;

CREATE TABLE app.tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    plan TEXT NOT NULL DEFAULT 'free',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE app.users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES app.tenants(id),
    email TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'member',
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

```sql
-- V002__add_projects_table.sql
CREATE TABLE app.projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES app.tenants(id),
    name TEXT NOT NULL,
    owner_id UUID NOT NULL REFERENCES app.users(id),
    status TEXT NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_projects_tenant_id ON app.projects(tenant_id);
CREATE INDEX idx_projects_owner_id ON app.projects(owner_id);
```

```sql
-- V003__add_rls_policies.sql
-- Enable RLS
ALTER TABLE app.tenants ENABLE ROW LEVEL SECURITY;
ALTER TABLE app.users ENABLE ROW LEVEL SECURITY;
ALTER TABLE app.projects ENABLE ROW LEVEL SECURITY;

-- Create application role
DO $$
BEGIN
    IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'app_user') THEN
        CREATE ROLE app_user LOGIN PASSWORD 'change_this';
    END IF;
END
$$;

GRANT USAGE ON SCHEMA app TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_user;

-- Policies
CREATE POLICY tenant_isolation ON app.tenants
    FOR ALL TO app_user
    USING (id = current_setting('app.tenant_id', true)::uuid);
```

```sql
-- R__stored_procedures.sql (Repeatable migration)
-- นำหน้าด้วย R__ = รันทุกครั้งที่ checksum เปลี่ยน

CREATE OR REPLACE FUNCTION compliance.has_consent(
    p_user_id UUID,
    p_purpose TEXT
) RETURNS BOOLEAN AS $$
BEGIN
    RETURN EXISTS (
        SELECT 1 FROM compliance.consents
        WHERE user_id = p_user_id
        AND purpose = p_purpose
        AND is_active = true
    );
END;
$$ LANGUAGE plpgsql STABLE;
```

**Running Flyway:**

```bash
# ดู migration status
flyway -url=jdbc:postgresql://localhost:5432/myapp \
       -user=postgres \
       -password=$DB_PASSWORD \
       info

# Apply migrations
flyway -url=jdbc:postgresql://localhost:5432/myapp \
       -user=postgres \
       -password=$DB_PASSWORD \
       migrate

# Validate (ตรวจสอบ checksums)
flyway validate

# Docker compose
docker run --rm \
  -v $(pwd)/db/migration:/flyway/sql \
  -e FLYWAY_URL=jdbc:postgresql://db:5432/myapp \
  -e FLYWAY_USER=postgres \
  -e FLYWAY_PASSWORD=$DB_PASSWORD \
  flyway/flyway:9-alpine migrate
```

### 3.2 node-pg-migrate

```bash
npm install node-pg-migrate pg
```

```javascript
// package.json
{
  "scripts": {
    "db:migrate": "node-pg-migrate up",
    "db:rollback": "node-pg-migrate down 1",
    "db:create": "node-pg-migrate create",
    "db:status": "node-pg-migrate status"
  }
}
```

```javascript
// migrations/1699000000000_initial-schema.js
exports.up = (pgm) => {
  // Create schemas
  pgm.createSchema('app', { ifNotExists: true });
  pgm.createSchema('audit', { ifNotExists: true });
  
  // Create tenants table
  pgm.createTable({ schema: 'app', name: 'tenants' }, {
    id: {
      type: 'uuid',
      primaryKey: true,
      default: pgm.func('gen_random_uuid()'),
    },
    name: { type: 'text', notNull: true },
    slug: { type: 'text', unique: true, notNull: true },
    plan: { type: 'text', notNull: true, default: "'free'" },
    created_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
  });
  
  // Create users table
  pgm.createTable({ schema: 'app', name: 'users' }, {
    id: {
      type: 'uuid',
      primaryKey: true,
      default: pgm.func('gen_random_uuid()'),
    },
    tenant_id: {
      type: 'uuid',
      notNull: true,
      references: '"app"."tenants"',
      onDelete: 'CASCADE',
    },
    email: { type: 'text', notNull: true, unique: true },
    password_hash: { type: 'text', notNull: true },
    role: { type: 'text', notNull: true, default: "'member'" },
    created_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
  });
  
  pgm.createIndex({ schema: 'app', name: 'users' }, 'tenant_id');
};

exports.down = (pgm) => {
  pgm.dropTable({ schema: 'app', name: 'users' });
  pgm.dropTable({ schema: 'app', name: 'tenants' });
  pgm.dropSchema('audit');
  pgm.dropSchema('app');
};
```

```javascript
// migrations/1699000001000_add-projects.js
exports.up = (pgm) => {
  pgm.createTable({ schema: 'app', name: 'projects' }, {
    id: {
      type: 'uuid',
      primaryKey: true,
      default: pgm.func('gen_random_uuid()'),
    },
    tenant_id: {
      type: 'uuid',
      notNull: true,
      references: '"app"."tenants"',
      onDelete: 'CASCADE',
    },
    name: { type: 'text', notNull: true },
    description: { type: 'text' },
    owner_id: {
      type: 'uuid',
      notNull: true,
      references: '"app"."users"',
    },
    status: {
      type: 'text',
      notNull: true,
      default: "'active'",
      check: "status IN ('active', 'archived', 'deleted')",
    },
    created_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
  });
  
  pgm.createIndex({ schema: 'app', name: 'projects' }, 'tenant_id');
  pgm.createIndex({ schema: 'app', name: 'projects' }, 'owner_id');
  
  // Add check constraint
  pgm.addConstraint(
    { schema: 'app', name: 'projects' },
    'projects_name_not_empty',
    "name != ''"
  );
};

exports.down = (pgm) => {
  pgm.dropTable({ schema: 'app', name: 'projects' });
};
```

### 3.3 Prisma Migrate

```bash
npm install prisma @prisma/client
npx prisma init
```

```prisma
// prisma/schema.prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["multiSchema"]
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
  schemas  = ["app", "audit"]
}

model Tenant {
  id        String    @id @default(uuid()) @db.Uuid
  name      String
  slug      String    @unique
  plan      String    @default("free")
  createdAt DateTime  @default(now()) @map("created_at")
  
  users    User[]
  projects Project[]
  
  @@map("tenants")
  @@schema("app")
}

model User {
  id           String   @id @default(uuid()) @db.Uuid
  tenantId     String   @map("tenant_id") @db.Uuid
  email        String   @unique
  passwordHash String   @map("password_hash")
  role         String   @default("member")
  createdAt    DateTime @default(now()) @map("created_at")
  
  tenant   Tenant    @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  projects Project[]
  
  @@index([tenantId])
  @@map("users")
  @@schema("app")
}

model Project {
  id          String   @id @default(uuid()) @db.Uuid
  tenantId    String   @map("tenant_id") @db.Uuid
  name        String
  description String?
  ownerId     String   @map("owner_id") @db.Uuid
  status      String   @default("active")
  createdAt   DateTime @default(now()) @map("created_at")
  
  tenant Tenant @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  owner  User   @relation(fields: [ownerId], references: [id])
  
  @@index([tenantId])
  @@map("projects")
  @@schema("app")
}
```

```bash
# Create migration
npx prisma migrate dev --name add_projects_table

# Apply in production
npx prisma migrate deploy

# Check status
npx prisma migrate status

# Reset (dev only!)
npx prisma migrate reset
```

### 3.4 Atlas: Modern Schema-as-Code

```bash
# ติดตั้ง Atlas CLI
curl -sSf https://atlasgo.sh | sh

# หรือ Homebrew
brew install ariga/tap/atlas
```

**HCL Schema Definition:**

```hcl
# schema.hcl
schema "app" {
  comment = "Application schema"
}

schema "audit" {
  comment = "Audit schema"  
}

table "tenants" {
  schema = schema.app
  
  column "id" {
    type    = uuid
    default = sql("gen_random_uuid()")
  }
  
  column "name" {
    type = text
  }
  
  column "slug" {
    type = text
  }
  
  column "plan" {
    type    = text
    default = "free"
  }
  
  column "created_at" {
    type    = timestamptz
    default = sql("NOW()")
  }
  
  primary_key {
    columns = [column.id]
  }
  
  index "tenants_slug_key" {
    unique  = true
    columns = [column.slug]
  }
}

table "users" {
  schema = schema.app
  
  column "id" {
    type    = uuid
    default = sql("gen_random_uuid()")
  }
  
  column "tenant_id" {
    type = uuid
  }
  
  column "email" {
    type = text
  }
  
  column "password_hash" {
    type = text
  }
  
  column "role" {
    type    = text
    default = "member"
  }
  
  column "created_at" {
    type    = timestamptz
    default = sql("NOW()")
  }
  
  primary_key {
    columns = [column.id]
  }
  
  foreign_key "users_tenant_id_fkey" {
    columns     = [column.tenant_id]
    ref_columns = [table.tenants.column.id]
    on_delete   = CASCADE
  }
  
  index "users_email_key" {
    unique  = true
    columns = [column.email]
  }
  
  index "users_tenant_id_idx" {
    columns = [column.tenant_id]
  }
}

table "projects" {
  schema = schema.app
  
  column "id" {
    type    = uuid
    default = sql("gen_random_uuid()")
  }
  
  column "tenant_id" {
    type = uuid
  }
  
  column "name" {
    type = text
  }
  
  column "description" {
    type = text
    null = true
  }
  
  column "owner_id" {
    type = uuid
  }
  
  column "status" {
    type    = text
    default = "active"
  }
  
  column "created_at" {
    type    = timestamptz
    default = sql("NOW()")
  }
  
  primary_key {
    columns = [column.id]
  }
  
  foreign_key "projects_tenant_id_fkey" {
    columns     = [column.tenant_id]
    ref_columns = [table.tenants.column.id]
    on_delete   = CASCADE
  }
  
  foreign_key "projects_owner_id_fkey" {
    columns     = [column.owner_id]
    ref_columns = [table.users.column.id]
  }
  
  index "projects_tenant_id_idx" {
    columns = [column.tenant_id]
  }
}
```

**Atlas Commands:**

```bash
# Inspect current schema
atlas schema inspect \
  --url "postgres://postgres:password@localhost:5432/myapp?sslmode=disable" \
  --schema app,audit \
  --format '{{ hcl . }}' > current_schema.hcl

# Apply schema (dry run)
atlas schema apply \
  --url "postgres://postgres:password@localhost:5432/myapp?sslmode=disable" \
  --to "file://schema.hcl" \
  --dry-run

# Apply schema
atlas schema apply \
  --url "postgres://postgres:password@localhost:5432/myapp?sslmode=disable" \
  --to "file://schema.hcl"

# Generate migration from schema diff
atlas migrate diff add_projects_table \
  --from "postgres://postgres:password@localhost:5432/myapp_old?sslmode=disable" \
  --to "file://schema.hcl" \
  --dev-url "docker://postgres/15/dev"

# Apply migrations
atlas migrate apply \
  --url "postgres://postgres:password@localhost:5432/myapp?sslmode=disable" \
  --dir "file://migrations"

# Lint migrations (check for issues)
atlas migrate lint \
  --dev-url "docker://postgres/15/dev" \
  --dir "file://migrations"

# Check for drift
atlas schema diff \
  --from "postgres://postgres:password@localhost:5432/myapp?sslmode=disable" \
  --to "file://schema.hcl"
```

**atlas.hcl (Project Config):**

```hcl
# atlas.hcl
variable "db_url" {
  type    = string
  default = getenv("DATABASE_URL")
}

env "local" {
  url    = "postgres://postgres:password@localhost:5432/myapp_dev?sslmode=disable"
  dev    = "docker://postgres/15/dev"
  schema = schema.hcl
  
  migration {
    dir    = "file://migrations"
    format = atlas
  }
}

env "staging" {
  url    = var.db_url
  schema = schema.hcl
  
  migration {
    dir    = "file://migrations"
    format = atlas
  }
}

env "production" {
  url    = var.db_url
  schema = schema.hcl
  
  migration {
    dir    = "file://migrations"
    format = atlas
    # Require approval in production
    revisions_schema = "atlas_migrations"
  }
}
```

---

## 4. GitOps สำหรับ Database

### 4.1 Migration PR Workflow

```yaml
# .github/workflows/db-migration-check.yml
name: Database Migration Check

on:
  pull_request:
    paths:
      - 'migrations/**'
      - 'schema.hcl'
      - 'prisma/schema.prisma'

jobs:
  validate-migration:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test_password
          POSTGRES_DB: myapp_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Atlas
        uses: ariga/setup-atlas@v0
        with:
          version: latest
      
      - name: Lint migrations
        run: |
          atlas migrate lint \
            --dev-url "postgres://postgres:test_password@localhost:5432/myapp_test?sslmode=disable" \
            --dir "file://migrations" \
            --git-base ${{ github.base_ref }}
      
      - name: Apply migrations to test DB
        run: |
          atlas migrate apply \
            --url "postgres://postgres:test_password@localhost:5432/myapp_test?sslmode=disable" \
            --dir "file://migrations"
      
      - name: Run migration tests
        run: |
          npm run test:migrations
        env:
          DATABASE_URL: postgres://postgres:test_password@localhost:5432/myapp_test
      
      - name: Post migration summary to PR
        uses: actions/github-script@v6
        with:
          script: |
            const { execSync } = require('child_process');
            const diff = execSync(
              'atlas schema diff ' +
              '--from "postgres://postgres:test_password@localhost:5432/myapp_test?sslmode=disable" ' +
              '--to "file://schema.hcl"'
            ).toString();
            
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Database Migration Summary\n\`\`\`sql\n${diff}\n\`\`\``,
            });
```

### 4.2 Auto-apply on Merge

```yaml
# .github/workflows/db-migration-deploy.yml
name: Database Migration Deploy

on:
  push:
    branches: [main]
    paths:
      - 'migrations/**'

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Atlas
        uses: ariga/setup-atlas@v0
      
      - name: Apply migrations to staging
        run: |
          atlas migrate apply \
            --url "${{ secrets.STAGING_DATABASE_URL }}" \
            --dir "file://migrations"
        env:
          ATLAS_TOKEN: ${{ secrets.ATLAS_TOKEN }}
      
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": "✅ Database migrations deployed to staging",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*DB Migrations Deployed to Staging*\nCommit: ${{ github.sha }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
  
  deploy-production:
    runs-on: ubuntu-latest
    environment: production  # Requires manual approval
    needs: deploy-staging
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Atlas
        uses: ariga/setup-atlas@v0
      
      - name: Pre-migration check
        run: |
          # Check for drift before applying
          atlas schema diff \
            --from "${{ secrets.PROD_DATABASE_URL }}" \
            --to "file://schema.hcl" \
            --schema app
      
      - name: Apply migrations to production
        run: |
          atlas migrate apply \
            --url "${{ secrets.PROD_DATABASE_URL }}" \
            --dir "file://migrations" \
            --lock-timeout 10s
```

### 4.3 Drift Detection

```bash
#!/bin/bash
# scripts/check-drift.sh

echo "Checking for schema drift..."

# Compare actual DB schema with desired state
DIFF=$(atlas schema diff \
  --from "$DATABASE_URL" \
  --to "file://schema.hcl" \
  --schema app,audit 2>&1)

if [ -z "$DIFF" ]; then
  echo "✅ No drift detected - schema is in sync"
  exit 0
else
  echo "❌ Schema drift detected!"
  echo "$DIFF"
  
  # Send alert
  curl -X POST "$SLACK_WEBHOOK_URL" \
    -H 'Content-type: application/json' \
    -d "{\"text\": \"⚠️ Database schema drift detected in production!\n\`\`\`${DIFF}\`\`\`\"}"
  
  exit 1
fi
```

```yaml
# .github/workflows/drift-detection.yml
name: Schema Drift Detection

on:
  schedule:
    - cron: '0 */6 * * *'  # Every 6 hours

jobs:
  check-drift:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Atlas
        uses: ariga/setup-atlas@v0
      
      - name: Check for drift
        run: |
          atlas schema diff \
            --from "${{ secrets.PROD_DATABASE_URL }}" \
            --to "file://schema.hcl" \
            --schema app,audit
        continue-on-error: true
        id: drift_check
      
      - name: Alert on drift
        if: steps.drift_check.outcome == 'failure'
        run: |
          echo "Schema drift detected! Sending alert..."
          # Send PagerDuty / Slack alert
```

---

## 5. Seed Data Management

### 5.1 Reference Data (Lookup Tables)

```sql
-- db/seeds/reference_data.sql

-- Plans
INSERT INTO app.plans (id, name, max_users, max_projects, monthly_price)
VALUES
    ('free', 'Free', 5, 3, 0),
    ('pro', 'Pro', 25, 50, 29),
    ('enterprise', 'Enterprise', 0, 0, 299) -- 0 = unlimited
ON CONFLICT (id) DO UPDATE SET
    name = EXCLUDED.name,
    max_users = EXCLUDED.max_users,
    max_projects = EXCLUDED.max_projects,
    monthly_price = EXCLUDED.monthly_price;

-- Roles
INSERT INTO app.roles (name, permissions)
VALUES
    ('owner', ARRAY['*']),
    ('admin', ARRAY['users:read', 'users:write', 'projects:*', 'settings:*']),
    ('manager', ARRAY['projects:*', 'tasks:*', 'users:read']),
    ('member', ARRAY['projects:read', 'tasks:*'])
ON CONFLICT (name) DO UPDATE SET
    permissions = EXCLUDED.permissions;

-- Default notification types
INSERT INTO app.notification_types (code, name, description)
VALUES
    ('task_assigned', 'Task Assigned', 'When a task is assigned to you'),
    ('task_completed', 'Task Completed', 'When a task you created is completed'),
    ('project_invite', 'Project Invitation', 'When invited to a project'),
    ('mention', 'Mentioned', 'When someone mentions you')
ON CONFLICT (code) DO UPDATE SET
    name = EXCLUDED.name,
    description = EXCLUDED.description;
```

### 5.2 Test Fixtures

```javascript
// db/seeds/test-data.js
const { Pool } = require('pg');
const bcrypt = require('bcrypt');
const crypto = require('crypto');

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function seedTestData() {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Clean existing test data
    await client.query(`
      DELETE FROM app.users WHERE email LIKE '%@test.example.com';
      DELETE FROM app.tenants WHERE slug LIKE 'test-%';
    `);
    
    // Create test tenants
    const tenant1 = await client.query(`
      INSERT INTO app.tenants (name, slug, plan)
      VALUES ('Test Company A', 'test-company-a', 'pro')
      RETURNING id
    `);
    
    const tenant2 = await client.query(`
      INSERT INTO app.tenants (name, slug, plan)
      VALUES ('Test Company B', 'test-company-b', 'free')
      RETURNING id
    `);
    
    const tenantId1 = tenant1.rows[0].id;
    const tenantId2 = tenant2.rows[0].id;
    
    // Create test users
    const passwordHash = await bcrypt.hash('Test1234!', 10);
    
    const users = [
      { email: 'admin@test.example.com', role: 'owner', tenantId: tenantId1 },
      { email: 'manager@test.example.com', role: 'manager', tenantId: tenantId1 },
      { email: 'member@test.example.com', role: 'member', tenantId: tenantId1 },
      { email: 'admin2@test.example.com', role: 'owner', tenantId: tenantId2 },
    ];
    
    for (const user of users) {
      await client.query(`
        INSERT INTO app.users (tenant_id, email, password_hash, role)
        VALUES ($1, $2, $3, $4)
      `, [user.tenantId, user.email, passwordHash, user.role]);
    }
    
    // Create test projects
    const adminUser = await client.query(
      'SELECT id FROM app.users WHERE email = $1',
      ['admin@test.example.com']
    );
    
    for (let i = 1; i <= 5; i++) {
      await client.query(`
        INSERT INTO app.projects (tenant_id, name, owner_id, status)
        VALUES ($1, $2, $3, 'active')
      `, [tenantId1, `Test Project ${i}`, adminUser.rows[0].id]);
    }
    
    await client.query('COMMIT');
    console.log('Test data seeded successfully');
    
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

if (require.main === module) {
  seedTestData()
    .then(() => process.exit(0))
    .catch(err => {
      console.error(err);
      process.exit(1);
    });
}

module.exports = { seedTestData };
```

### 5.3 Flyway Seed Migration

```sql
-- db/migration/V099__seed_reference_data.sql
-- Seed reference data as part of migrations

INSERT INTO app.plans (id, name, max_users, monthly_price)
VALUES
    ('free', 'Free', 5, 0),
    ('pro', 'Pro', 25, 29),
    ('enterprise', 'Enterprise', -1, 299)
ON CONFLICT (id) DO NOTHING;

-- Version-aware seed
DO $$
BEGIN
    IF NOT EXISTS (SELECT 1 FROM app.notification_types WHERE code = 'task_assigned') THEN
        INSERT INTO app.notification_types (code, name)
        VALUES ('task_assigned', 'Task Assigned');
    END IF;
END
$$;
```

---

## 6. Configuration Management as Code

### 6.1 postgresql.conf as Code

```hcl
# terraform/modules/rds/variables.tf

variable "postgresql_config" {
  type = map(string)
  default = {
    # Connection settings
    max_connections = "200"
    
    # Memory
    shared_buffers = "256MB"
    effective_cache_size = "1GB"
    work_mem = "8MB"
    maintenance_work_mem = "128MB"
    
    # Write performance
    wal_buffers = "16MB"
    checkpoint_completion_target = "0.9"
    max_wal_size = "4GB"
    min_wal_size = "1GB"
    
    # Query planner
    random_page_cost = "1.1"  # SSD: 1.1, HDD: 4.0
    effective_io_concurrency = "200"  # SSD: 200, HDD: 2
    
    # Logging
    log_min_duration_statement = "1000"  # log queries > 1s
    log_checkpoints = "on"
    log_connections = "off"
    log_disconnections = "off"
    log_lock_waits = "on"
    
    # Autovacuum
    autovacuum = "on"
    autovacuum_max_workers = "5"
    autovacuum_naptime = "30"
  }
}
```

```hcl
# terraform/modules/rds/main.tf

resource "aws_db_parameter_group" "main" {
  name   = "myapp-${var.environment}"
  family = "postgres15"
  
  dynamic "parameter" {
    for_each = var.postgresql_config
    content {
      name  = parameter.key
      value = parameter.value
      apply_method = "pending-reboot"
    }
  }
  
  # Special parameters requiring immediate apply
  parameter {
    name         = "log_min_duration_statement"
    value        = "1000"
    apply_method = "immediate"
  }
  
  tags = {
    Environment = var.environment
    Project     = "myapp"
  }
}
```

### 6.2 pg_hba.conf as Code

```hcl
# pg_hba.conf ใน Ansible
- name: Configure pg_hba.conf
  blockinfile:
    path: /etc/postgresql/15/main/pg_hba.conf
    block: |
      # TYPE  DATABASE        USER            ADDRESS                 METHOD
      local   all             postgres                                peer
      local   all             all                                     md5
      host    all             all             127.0.0.1/32            scram-sha-256
      hostssl all             app_user        0.0.0.0/0               scram-sha-256
      hostssl all             replication_user    10.0.0.0/8          scram-sha-256
  notify: reload postgresql
```

---

## 7. Terraform สำหรับ Cloud Databases

### 7.1 Complete Terraform Setup

```hcl
# terraform/main.tf

terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
  
  backend "s3" {
    bucket = "myapp-terraform-state"
    key    = "database/terraform.tfstate"
    region = "ap-southeast-1"
    
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = "myapp"
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}

# Variables
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "production"
}

variable "aws_region" {
  default = "ap-southeast-1"
}

# VPC
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  
  name = "myapp-${var.environment}"
  cidr = "10.0.0.0/16"
  
  azs             = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  single_nat_gateway = var.environment != "production"
}
```

```hcl
# terraform/rds.tf

# Security Group สำหรับ RDS
resource "aws_security_group" "rds" {
  name        = "myapp-rds-${var.environment}"
  description = "Security group for RDS PostgreSQL"
  vpc_id      = module.vpc.vpc_id
  
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
    description     = "PostgreSQL from app servers"
  }
  
  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]  # VPN access
    description = "PostgreSQL from VPN"
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Subnet Group
resource "aws_db_subnet_group" "main" {
  name       = "myapp-${var.environment}"
  subnet_ids = module.vpc.private_subnets
  
  tags = { Name = "myapp-${var.environment}" }
}

# Parameter Group
resource "aws_db_parameter_group" "postgres15" {
  name   = "myapp-postgres15-${var.environment}"
  family = "postgres15"
  
  parameter {
    name  = "max_connections"
    value = "200"
  }
  
  parameter {
    name  = "shared_buffers"
    value = "{DBInstanceClassMemory/4}"
  }
  
  parameter {
    name  = "log_min_duration_statement"
    value = "1000"
    apply_method = "immediate"
  }
  
  parameter {
    name  = "log_lock_waits"
    value = "1"
    apply_method = "immediate"
  }
  
  parameter {
    name  = "rds.force_ssl"
    value = "1"
  }
}

# RDS Instance
resource "aws_db_instance" "postgres" {
  identifier = "myapp-postgres-${var.environment}"
  
  engine         = "postgres"
  engine_version = "15.4"
  
  instance_class = local.db_config[var.environment].instance_class
  
  db_name  = "myapp"
  username = "postgres"
  
  manage_master_user_password = true  # AWS manages password in Secrets Manager
  
  allocated_storage     = local.db_config[var.environment].storage_gb
  max_allocated_storage = local.db_config[var.environment].storage_gb * 2
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds.arn
  
  multi_az               = local.db_config[var.environment].multi_az
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  
  parameter_group_name = aws_db_parameter_group.postgres15.name
  
  backup_retention_period   = local.db_config[var.environment].backup_days
  backup_window             = "03:00-04:00"
  maintenance_window        = "Mon:04:00-Mon:05:00"
  copy_tags_to_snapshot     = true
  delete_automated_backups  = false
  deletion_protection       = var.environment == "production"
  skip_final_snapshot       = var.environment != "production"
  final_snapshot_identifier = "myapp-postgres-${var.environment}-final"
  
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn
  
  performance_insights_enabled          = true
  performance_insights_retention_period = var.environment == "production" ? 7 : 7
  
  auto_minor_version_upgrade = true
  apply_immediately          = false
  
  lifecycle {
    prevent_destroy = true  # ป้องกันการลบ production DB
    ignore_changes  = [password]
  }
}

# Read Replica
resource "aws_db_instance" "postgres_replica" {
  count = local.db_config[var.environment].read_replicas
  
  identifier = "myapp-postgres-replica-${count.index + 1}-${var.environment}"
  
  replicate_source_db    = aws_db_instance.postgres.identifier
  instance_class         = local.db_config[var.environment].replica_instance_class
  
  multi_az              = false
  publicly_accessible   = false
  storage_encrypted     = true
  
  parameter_group_name  = aws_db_parameter_group.postgres15.name
  
  performance_insights_enabled = true
  
  auto_minor_version_upgrade = true
  apply_immediately          = false
}
```

```hcl
# terraform/elasticache.tf

# ElastiCache Redis
resource "aws_elasticache_subnet_group" "redis" {
  name       = "myapp-redis-${var.environment}"
  subnet_ids = module.vpc.private_subnets
}

resource "aws_security_group" "redis" {
  name        = "myapp-redis-${var.environment}"
  vpc_id      = module.vpc.vpc_id
  
  ingress {
    from_port       = 6379
    to_port         = 6380
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }
}

resource "aws_elasticache_replication_group" "redis" {
  replication_group_id       = "myapp-redis-${var.environment}"
  description                = "Redis cluster for myapp ${var.environment}"
  
  node_type            = local.redis_config[var.environment].node_type
  num_cache_clusters   = local.redis_config[var.environment].num_nodes
  
  engine               = "redis"
  engine_version       = "7.0"
  port                 = 6379
  
  subnet_group_name    = aws_elasticache_subnet_group.redis.name
  security_group_ids   = [aws_security_group.redis.id]
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = random_password.redis_auth.result
  
  automatic_failover_enabled = local.redis_config[var.environment].num_nodes > 1
  multi_az_enabled           = local.redis_config[var.environment].num_nodes > 1
  
  snapshot_retention_limit = 1
  snapshot_window          = "05:00-06:00"
  maintenance_window       = "Mon:06:00-Mon:07:00"
  
  parameter_group_name = aws_elasticache_parameter_group.redis7.name
}

resource "aws_elasticache_parameter_group" "redis7" {
  name   = "myapp-redis7-${var.environment}"
  family = "redis7"
  
  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"
  }
  
  parameter {
    name  = "notify-keyspace-events"
    value = "Ex"  # Expired events
  }
}
```

---

## 8. Full GitOps Pipeline

### 8.1 Complete CI/CD Flow

```yaml
# .github/workflows/full-pipeline.yml
name: Full Database GitOps Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  ATLAS_VERSION: latest

jobs:
  # ============================
  # Stage 1: Validate
  # ============================
  validate:
    name: Validate Migrations
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        options: --health-cmd pg_isready --health-interval 10s
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Atlas needs git history
      
      - name: Setup Atlas
        uses: ariga/setup-atlas@v0
      
      - name: Validate migration files
        run: |
          atlas migrate validate \
            --dir "file://migrations" \
            --dev-url "postgres://postgres:test@localhost:5432/test?sslmode=disable"
      
      - name: Lint migrations (check for issues)
        run: |
          atlas migrate lint \
            --dev-url "postgres://postgres:test@localhost:5432/test?sslmode=disable" \
            --dir "file://migrations" \
            --git-base origin/${{ github.base_ref || 'main' }}
      
      - name: Test migrations
        run: |
          # Apply all migrations to test DB
          atlas migrate apply \
            --url "postgres://postgres:test@localhost:5432/test?sslmode=disable" \
            --dir "file://migrations"
          
          # Run migration-specific tests
          npm run test:db
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/test
  
  # ============================
  # Stage 2: Preview (PR only)
  # ============================
  preview:
    name: Preview Changes
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    needs: validate
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Atlas
        uses: ariga/setup-atlas@v0
      
      - name: Generate migration preview
        id: preview
        run: |
          DIFF=$(atlas migrate diff --dry-run \
            --from "${{ secrets.STAGING_DB_URL }}" \
            --dir "file://migrations" 2>&1 || true)
          
          echo "diff<<EOF" >> $GITHUB_OUTPUT
          echo "$DIFF" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
      
      - name: Comment on PR
        uses: actions/github-script@v6
        with:
          script: |
            const diff = `${{ steps.preview.outputs.diff }}`;
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## 🗄️ Database Migration Preview\n\`\`\`sql\n${diff || 'No changes'}\n\`\`\``
            });
  
  # ============================
  # Stage 3: Deploy to Staging
  # ============================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: validate
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.myapp.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Atlas
        uses: ariga/setup-atlas@v0
      
      - name: Apply migrations to staging
        run: |
          atlas migrate apply \
            --url "${{ secrets.STAGING_DB_URL }}" \
            --dir "file://migrations" \
            --lock-timeout 30s
      
      - name: Run smoke tests
        run: |
          npm run test:smoke:staging
        env:
          API_URL: https://staging-api.myapp.com
  
  # ============================
  # Stage 4: Deploy to Production
  # ============================
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [validate, deploy-staging]
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://myapp.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Atlas
        uses: ariga/setup-atlas@v0
      
      - name: Check for drift before deployment
        run: |
          DRIFT=$(atlas schema diff \
            --from "${{ secrets.PROD_DB_URL }}" \
            --to "file://schema.hcl" 2>&1)
          
          if [ -n "$DRIFT" ]; then
            echo "⚠️ Drift detected before migration!"
            echo "$DRIFT"
            # Don't fail - just warn
          fi
      
      - name: Apply migrations to production
        run: |
          atlas migrate apply \
            --url "${{ secrets.PROD_DB_URL }}" \
            --dir "file://migrations" \
            --lock-timeout 60s \
            --tx-mode all
        
      - name: Verify deployment
        run: |
          atlas migrate status \
            --url "${{ secrets.PROD_DB_URL }}" \
            --dir "file://migrations"
      
      - name: Notify deployment
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": "✅ Database migrations deployed to production",
              "blocks": [{
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*DB Migrations Deployed to Production*\nBy: ${{ github.actor }}\nCommit: `${{ github.sha }}`"
                }
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 9. Safe Migration Patterns

### 9.1 Backward Compatible Migrations

```sql
-- ✅ Safe: ADD COLUMN with DEFAULT
ALTER TABLE users ADD COLUMN IF NOT EXISTS avatar_url TEXT;
-- ไม่ต้องการ lock ถ้า NOT NULL ไม่มี DEFAULT

-- ✅ Safe: ADD COLUMN NOT NULL with DEFAULT (PostgreSQL 11+)
ALTER TABLE users ADD COLUMN IF NOT EXISTS 
    notification_count INTEGER NOT NULL DEFAULT 0;

-- ❌ Dangerous: DROP COLUMN ที่ยังใช้อยู่
-- ต้องทำ 3 steps:
-- Step 1: Remove usage in application code
-- Step 2: Wait for all deployments
-- Step 3: Then drop column
ALTER TABLE users DROP COLUMN IF EXISTS old_column;

-- ✅ Safe: Rename column (Blue-Green approach)
-- Step 1: Add new column
ALTER TABLE users ADD COLUMN new_name TEXT;

-- Step 2: Backfill
UPDATE users SET new_name = old_name WHERE new_name IS NULL;

-- Step 3: Make NOT NULL after backfill
ALTER TABLE users ALTER COLUMN new_name SET NOT NULL;

-- Step 4: Application uses new_name
-- Step 5: Drop old column
ALTER TABLE users DROP COLUMN old_name;
```

### 9.2 Large Table Migrations

```sql
-- ❌ Dangerous: ALTER TABLE บน table ใหญ่
ALTER TABLE orders ADD COLUMN status TEXT NOT NULL DEFAULT 'active';
-- อาจ lock table หลายชั่วโมง!

-- ✅ Safe: ใช้ pg_repack หรือ pattern ต่อไปนี้

-- Option 1: Add nullable column ก่อน
ALTER TABLE orders ADD COLUMN new_status TEXT;

-- Backfill in batches (ไม่ lock table)
DO $$
DECLARE
    batch_size INT := 10000;
    last_id UUID := '00000000-0000-0000-0000-000000000000';
    rows_updated INT;
BEGIN
    LOOP
        WITH batch AS (
            SELECT id FROM orders
            WHERE id > last_id
            AND new_status IS NULL
            ORDER BY id
            LIMIT batch_size
        )
        UPDATE orders o
        SET new_status = 'active'
        FROM batch
        WHERE o.id = batch.id;
        
        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;
        
        SELECT MAX(id) INTO last_id FROM orders WHERE new_status IS NOT NULL;
        
        -- ป้องกัน vacuum lag
        PERFORM pg_sleep(0.1);
    END LOOP;
END;
$$;

-- Set NOT NULL after backfill complete
ALTER TABLE orders ALTER COLUMN new_status SET NOT NULL;
ALTER TABLE orders ALTER COLUMN new_status SET DEFAULT 'active';

-- Drop old column
ALTER TABLE orders DROP COLUMN status;
ALTER TABLE orders RENAME COLUMN new_status TO status;
```

---

## 10. สรุปและ Best Practices

```markdown
## Database as Code: Best Practices

### Version Control
✅ ทุก migration ใน Git
✅ Reviewed ก่อน merge (SQL review)
✅ Semantic versioning สำหรับ schema
✅ Branch สำหรับ feature migrations

### Migration Quality
✅ ทดสอบ up และ down migrations
✅ Idempotent migrations (IF EXISTS, ON CONFLICT)
✅ Backward compatible (zero-downtime)
✅ Batch large data migrations

### Deployment
✅ Deploy ผ่าน CI/CD เท่านั้น
✅ No manual DB changes ใน production
✅ Staging ต้อง deploy ก่อน production
✅ Drift detection scheduled

### Documentation
✅ Comment บน complex migrations
✅ Data dictionary ใน schema
✅ Migration notes (why, not just what)

### Rollback Plan
✅ Test rollback ก่อน deploy ใหญ่
✅ Snapshot ก่อน migration ใหญ่
✅ Blue-green deployment สำหรับ risky migrations
```

---

*เนื้อหานี้เป็นส่วนหนึ่งของ Database Cluster Course - World-Class Level*
*Part 95/100: Database as Code*
