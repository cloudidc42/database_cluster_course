# 🗄️ หลักสูตร Database Cluster: ตั้งแต่พื้นฐานถึงระดับโลก

## สถาปัตยกรรม Database Cluster ที่เราจะเรียน

```
                    ┌─────────────────────────────────────┐
                    │         Load Balancer / HAProxy       │
                    └──────────────┬──────────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                   │                    │
    ┌─────────▼──────┐   ┌────────▼───────┐  ┌────────▼───────┐
    │  Web App        │   │  Web App        │  │  Web App        │
    │  (Node.js/      │   │  (Node.js/      │  │  (Node.js/      │
    │   Python/Go)    │   │   Python/Go)    │  │   Python/Go)    │
    └─────────┬──────┘   └────────┬───────┘  └────────┬───────┘
              │                   │                    │
              └────────────────────┼────────────────────┘
                                   │
            ┌──────────────────────┼──────────────────────┐
            │                      │                       │
   ┌─────────▼──────┐    ┌─────────▼──────┐    ┌──────────▼─────┐
   │ PostgreSQL      │    │ Redis Cache    │    │ S3/MinIO        │
   │ Primary         │    │ (Session/      │    │ Object Storage  │
   │ (Write)         │    │  Cache/Queue)  │    │ (Images/Videos/ │
   └─────────┬──────┘    └────────────────┘    │  Files)         │
             │ Replication                      └────────────────┘
   ┌─────────▼──────┐
   │ PostgreSQL      │
   │ Replica 1       │
   │ (Read)          │
   └─────────┬──────┘
             │ Replication
   ┌─────────▼──────┐
   │ PostgreSQL      │
   │ Replica 2       │
   │ (Read)          │
   └────────────────┘
```

## 📚 โครงสร้างหลักสูตร (100+ Parts)

### 🟢 ระดับพื้นฐาน (Parts 1-20): รากฐานที่แข็งแกร่ง
| Part | หัวข้อ |
|------|--------|
| 01 | บทนำ: Database Cluster คืออะไร และทำไมต้องใช้ |
| 02 | ติดตั้ง PostgreSQL บน Linux/Mac/Docker |
| 03 | SQL พื้นฐาน: CREATE, INSERT, SELECT, UPDATE, DELETE |
| 04 | Data Types และ Constraints ใน PostgreSQL |
| 05 | Primary Key, Foreign Key และ Index |
| 06 | Transaction และ ACID Properties |
| 07 | การออกแบบ Database Schema เบื้องต้น |
| 08 | ติดตั้งและใช้งาน Redis เบื้องต้น |
| 09 | Redis Data Structures: String, Hash, List, Set, ZSet |
| 10 | ติดตั้ง MinIO (S3-Compatible) สำหรับ Object Storage |
| 11 | เชื่อมต่อ Node.js กับ PostgreSQL (pg module) |
| 12 | เชื่อมต่อ Node.js กับ Redis (ioredis) |
| 13 | เชื่อมต่อ Node.js กับ MinIO/S3 |
| 14 | สร้าง REST API พื้นฐานด้วย Express.js |
| 15 | CRUD Operations ผ่าน API |
| 16 | Authentication ด้วย JWT + PostgreSQL |
| 17 | Session Management ด้วย Redis |
| 18 | File Upload ไปยัง MinIO/S3 |
| 19 | Error Handling และ Validation |
| 20 | Logging และ Monitoring เบื้องต้น |

### 🟡 ระดับกลาง (Parts 21-50): ทักษะ Production-Ready
| Part | หัวข้อ |
|------|--------|
| 21 | PostgreSQL Replication: Primary-Replica Setup |
| 22 | Connection Pooling ด้วย PgBouncer |
| 23 | Read/Write Splitting ใน Application |
| 24 | Redis Cluster และ Sentinel |
| 25 | Caching Strategies: Cache-Aside, Write-Through, Write-Behind |
| 26 | Cache Invalidation Patterns |
| 27 | Database Migrations ด้วย Flyway/Liquibase |
| 28 | ORM: Prisma หรือ TypeORM |
| 29 | Query Optimization และ EXPLAIN ANALYZE |
| 30 | Indexing Strategies: B-Tree, Hash, GIN, GiST |
| 31 | Full-Text Search ใน PostgreSQL |
| 32 | JSON/JSONB Operations ใน PostgreSQL |
| 33 | Partitioning ใน PostgreSQL |
| 34 | Backup และ Recovery Strategies |
| 35 | Docker Compose สำหรับ Database Cluster |
| 36 | Health Checks และ Readiness Probes |
| 37 | Rate Limiting ด้วย Redis |
| 38 | Queue System ด้วย Redis (Bull/BullMQ) |
| 39 | Pub/Sub Messaging ด้วย Redis |
| 40 | S3 Presigned URLs และ Access Control |
| 41 | CDN Integration กับ S3 |
| 42 | Image Processing Pipeline |
| 43 | Pagination Strategies |
| 44 | Cursor-based Pagination |
| 45 | Search และ Filtering |
| 46 | API Versioning |
| 47 | Middleware Patterns |
| 48 | Testing: Unit, Integration, E2E |
| 49 | CI/CD Pipeline |
| 50 | Security Best Practices |

### 🔴 ระดับสูง (Parts 51-75): Expert-Level Engineering
| Part | หัวข้อ |
|------|--------|
| 51 | Kubernetes Deployment |
| 52 | Helm Charts สำหรับ Database Cluster |
| 53 | PostgreSQL High Availability ด้วย Patroni |
| 54 | Redis High Availability ด้วย Sentinel/Cluster |
| 55 | Service Mesh (Istio/Linkerd) |
| 56 | Distributed Tracing (Jaeger/Zipkin) |
| 57 | Metrics ด้วย Prometheus + Grafana |
| 58 | Alerting และ Incident Response |
| 59 | Performance Tuning: PostgreSQL |
| 60 | Performance Tuning: Redis |
| 61 | Connection Pool Optimization |
| 62 | Query Planner Deep Dive |
| 63 | Vacuum และ Autovacuum Tuning |
| 64 | WAL Configuration |
| 65 | Streaming Replication Advanced |
| 66 | Logical Replication |
| 67 | Multi-Master Setup |
| 68 | Sharding Strategies |
| 69 | Event Sourcing Pattern |
| 70 | CQRS (Command Query Responsibility Segregation) |
| 71 | Saga Pattern สำหรับ Distributed Transactions |
| 72 | Outbox Pattern |
| 73 | Change Data Capture (CDC) ด้วย Debezium |
| 74 | GraphQL กับ Database Cluster |
| 75 | gRPC Services |

### 🏆 ระดับโลก (Parts 76-100+): World-Class Architecture
| Part | หัวข้อ |
|------|--------|
| 76 | Multi-Region Database Architecture |
| 77 | Global Data Distribution |
| 78 | CockroachDB / Distributed SQL |
| 79 | TimescaleDB สำหรับ Time-Series Data |
| 80 | Vector Database Integration (pgvector) |
| 81 | Real-time Features ด้วย PostgreSQL NOTIFY |
| 82 | WebSocket + Redis Pub/Sub |
| 83 | Kafka Integration กับ Database Cluster |
| 84 | Data Lake Architecture |
| 85 | OLAP vs OLTP Strategies |
| 86 | Data Warehouse ด้วย PostgreSQL |
| 87 | Analytics Pipeline |
| 88 | Machine Learning Integration |
| 89 | Zero-Downtime Migrations |
| 90 | Disaster Recovery Planning |
| 91 | Chaos Engineering |
| 92 | Security: Row-Level Security (RLS) |
| 93 | Security: Encryption at Rest/Transit |
| 94 | Compliance: GDPR, PDPA |
| 95 | Cost Optimization |
| 96 | Database as Code |
| 97 | GitOps สำหรับ Database |
| 98 | Platform Engineering |
| 99 | SRE Practices |
| 100 | Final Project: สร้าง Production-Grade System |

## 🛠️ Tech Stack ที่ใช้ในหลักสูตร

| Category | Technologies |
|----------|-------------|
| Database | PostgreSQL 16, Redis 7, MinIO |
| Backend | Node.js, Python (FastAPI), Go |
| ORM | Prisma, TypeORM, SQLAlchemy |
| Container | Docker, Docker Compose, Kubernetes |
| CI/CD | GitHub Actions, GitLab CI |
| Monitoring | Prometheus, Grafana, Jaeger |
| Cloud | AWS (RDS, ElastiCache, S3), GCP, Azure |

## 📋 Prerequisites

- Linux/Mac หรือ Windows WSL2
- Docker Desktop
- Node.js 20+ / Python 3.11+ / Go 1.21+
- Git
- VS Code / IDE ที่ชอบ

## 🚀 เริ่มต้นเรียน

```bash
# Clone repository
git clone https://github.com/cloudidc42/database_cluster_course.git
cd database_cluster_course

# เริ่มจาก Part 1
cat part-01-10/part-01-introduction.md
```

---

*หลักสูตรนี้ออกแบบมาสำหรับ Developer ทุกระดับ ตั้งแต่เริ่มต้นจนถึงระดับ Senior/Staff Engineer*
