# Part 101: AWS Cloud Deployment สำหรับ Database Cluster

## บทนำ

ในบทนี้เราจะเรียนรู้การ deploy Database Cluster บน AWS Cloud Platform โดยใช้บริการต่างๆ ที่ AWS มีให้ เพื่อสร้าง production-ready database infrastructure ที่มี High Availability, Scalability และ Security

AWS มีบริการ managed database ที่ช่วยลดภาระการจัดการ infrastructure ทำให้ทีม development สามารถโฟกัสที่การพัฒนา application ได้มากขึ้น

---

## สารบัญ

1. AWS Services Overview สำหรับ Database Cluster
2. Amazon RDS PostgreSQL
3. Amazon ElastiCache Redis
4. Amazon S3 Storage
5. CloudFront CDN
6. Container Orchestration: ECS/EKS
7. ALB Load Balancer
8. Route53 DNS
9. ACM SSL Certificates
10. Secrets Manager
11. Parameter Store
12. VPC Network Design
13. Terraform Infrastructure as Code
14. Cost Estimation
15. Production Checklist

---

## 1. AWS Services Overview

### 1.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS Cloud                                 │
│                                                                  │
│  ┌─────────┐    ┌─────────┐    ┌─────────────────────────────┐ │
│  │Route53  │───▶│CloudFront│───▶│         ALB                 │ │
│  │ DNS     │    │  CDN    │    │   Load Balancer              │ │
│  └─────────┘    └─────────┘    └──────────┬──────────────────┘ │
│                                           │                      │
│  ┌────────────────────────────────────────▼──────────────────┐  │
│  │                    VPC (10.0.0.0/16)                       │  │
│  │                                                            │  │
│  │  ┌─────────────────────────┐  ┌────────────────────────┐  │  │
│  │  │   Public Subnet         │  │   Public Subnet        │  │  │
│  │  │   AZ-a (10.0.1.0/24)   │  │   AZ-b (10.0.2.0/24)  │  │  │
│  │  │   NAT Gateway           │  │   NAT Gateway          │  │  │
│  │  └─────────────────────────┘  └────────────────────────┘  │  │
│  │                                                            │  │
│  │  ┌─────────────────────────┐  ┌────────────────────────┐  │  │
│  │  │   Private Subnet        │  │   Private Subnet       │  │  │
│  │  │   AZ-a (10.0.10.0/24)  │  │   AZ-b (10.0.11.0/24) │  │  │
│  │  │   ECS/EKS Nodes         │  │   ECS/EKS Nodes       │  │  │
│  │  └─────────────────────────┘  └────────────────────────┘  │  │
│  │                                                            │  │
│  │  ┌─────────────────────────┐  ┌────────────────────────┐  │  │
│  │  │   Database Subnet       │  │   Database Subnet      │  │  │
│  │  │   AZ-a (10.0.20.0/24)  │  │   AZ-b (10.0.21.0/24) │  │  │
│  │  │   RDS Primary           │  │   RDS Standby          │  │  │
│  │  │   ElastiCache Primary   │  │   ElastiCache Replica  │  │  │
│  │  └─────────────────────────┘  └────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                   Support Services                           │ │
│  │  Secrets Manager | Parameter Store | S3 | CloudWatch        │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Services ที่ใช้งาน

| Service | วัตถุประสงค์ | Tier |
|---------|------------|------|
| RDS PostgreSQL | Database หลัก | Critical |
| ElastiCache Redis | Caching layer | Critical |
| S3 | Object/File storage | High |
| CloudFront | CDN สำหรับ static assets | Medium |
| ECS/EKS | Container runtime | Critical |
| ALB | HTTP/HTTPS load balancing | Critical |
| Route53 | DNS management | Critical |
| ACM | TLS/SSL certificates | Critical |
| Secrets Manager | Database passwords, API keys | Critical |
| Parameter Store | App configuration | High |
| VPC | Network isolation | Critical |
| CloudWatch | Monitoring & logs | High |
| IAM | Access control | Critical |

---

## 2. Amazon RDS PostgreSQL

### 2.1 RDS PostgreSQL Overview

Amazon RDS สำหรับ PostgreSQL เป็น managed database service ที่จัดการ:
- OS patching
- Database software updates
- Automated backups
- Failover
- Scaling

### 2.2 Multi-AZ Deployment

Multi-AZ deployment สร้าง standby replica ใน AZ อื่น โดย AWS จัดการ synchronous replication และ automatic failover

```
┌─────────────────────────────────────────┐
│           Multi-AZ Deployment           │
│                                         │
│  ┌─────────────────┐                    │
│  │   AZ-a          │                    │
│  │  ┌───────────┐  │                    │
│  │  │  Primary  │  │ Synchronous        │
│  │  │  RDS      │◄─┼──────────────────┐ │
│  │  └───────────┘  │ Replication      │ │
│  └─────────────────┘                  │ │
│                                        │ │
│  ┌─────────────────┐                   │ │
│  │   AZ-b          │                   │ │
│  │  ┌───────────┐  │                   │ │
│  │  │  Standby  │──┼───────────────────┘ │
│  │  │  RDS      │  │                    │
│  │  └───────────┘  │                    │
│  └─────────────────┘                    │
│                                         │
│  Failover time: 60-120 seconds          │
└─────────────────────────────────────────┘
```

**การตั้งค่า Multi-AZ ใน Terraform:**

```hcl
resource "aws_db_instance" "postgres_primary" {
  identifier        = "myapp-postgres-prod"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.r5.xlarge"
  
  # Storage
  allocated_storage     = 100
  max_allocated_storage = 1000
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds.arn
  
  # Database
  db_name  = "myapp"
  username = "myapp_admin"
  password = random_password.db_password.result
  port     = 5432
  
  # High Availability
  multi_az               = true
  availability_zone      = "ap-southeast-1a"
  
  # Networking
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  publicly_accessible    = false
  
  # Backup
  backup_retention_period = 30
  backup_window           = "02:00-03:00"
  maintenance_window      = "sun:04:00-sun:05:00"
  
  # Monitoring
  monitoring_interval    = 60
  monitoring_role_arn    = aws_iam_role.rds_monitoring.arn
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  
  # Performance
  performance_insights_enabled          = true
  performance_insights_retention_period = 7
  
  # Parameters
  parameter_group_name = aws_db_parameter_group.postgres15.name
  
  # Options
  deletion_protection = true
  skip_final_snapshot = false
  final_snapshot_identifier = "myapp-postgres-final"
  
  tags = {
    Environment = "production"
    Service     = "database"
    ManagedBy   = "terraform"
  }
}
```

### 2.3 Read Replicas

Read replicas ช่วยกระจาย read traffic ออกจาก primary database

```hcl
# Read Replica 1 - Same Region
resource "aws_db_instance" "postgres_replica_1" {
  identifier             = "myapp-postgres-replica-1"
  replicate_source_db    = aws_db_instance.postgres_primary.identifier
  instance_class         = "db.r5.large"
  
  # Storage (inherited from source)
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn
  
  # Networking
  vpc_security_group_ids = [aws_security_group.rds_replica.id]
  publicly_accessible    = false
  
  # Performance
  performance_insights_enabled = true
  
  # Monitoring
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn
  
  auto_minor_version_upgrade = true
  
  tags = {
    Environment = "production"
    Role        = "read-replica"
    Number      = "1"
  }
}

# Read Replica 2 - Different AZ
resource "aws_db_instance" "postgres_replica_2" {
  identifier             = "myapp-postgres-replica-2"
  replicate_source_db    = aws_db_instance.postgres_primary.identifier
  instance_class         = "db.r5.large"
  availability_zone      = "ap-southeast-1b"
  
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn
  
  vpc_security_group_ids = [aws_security_group.rds_replica.id]
  publicly_accessible    = false
  
  performance_insights_enabled = true
  monitoring_interval          = 60
  monitoring_role_arn          = aws_iam_role.rds_monitoring.arn
  
  tags = {
    Environment = "production"
    Role        = "read-replica"
    Number      = "2"
  }
}
```

### 2.4 Instance Types

| Family | Use Case | vCPU | RAM | Network |
|--------|----------|------|-----|---------|
| db.t3.micro | Dev/Test | 2 | 1 GB | Low |
| db.t3.small | Dev/Test | 2 | 2 GB | Low |
| db.t3.medium | Small Prod | 2 | 4 GB | Moderate |
| db.m5.large | General | 2 | 8 GB | Moderate |
| db.m5.xlarge | General | 4 | 16 GB | Moderate |
| db.m5.2xlarge | General | 8 | 32 GB | High |
| db.r5.large | Memory | 2 | 16 GB | Moderate |
| db.r5.xlarge | Memory | 4 | 32 GB | High |
| db.r5.2xlarge | Memory | 8 | 64 GB | High |
| db.r5.4xlarge | Memory | 16 | 128 GB | Very High |

### 2.5 Storage Types

```hcl
# gp3 Storage (แนะนำสำหรับ production)
resource "aws_db_instance" "postgres_gp3" {
  # ...
  storage_type          = "gp3"
  allocated_storage     = 100    # GB
  max_allocated_storage = 1000   # Auto-scaling up to 1TB
  
  # gp3 ค่าเริ่มต้น 3000 IOPS, 125 MB/s throughput
  # สามารถเพิ่มได้ถึง 16000 IOPS, 1000 MB/s
  iops       = 6000   # Optional: ระบุ IOPS
  throughput = 250    # Optional: MB/s
}

# io1 Storage (High IOPS workloads)
resource "aws_db_instance" "postgres_io1" {
  # ...
  storage_type      = "io1"
  allocated_storage = 100
  iops              = 10000  # Required for io1
}
```

### 2.6 Parameter Groups

```hcl
resource "aws_db_parameter_group" "postgres15" {
  family = "postgres15"
  name   = "myapp-postgres15-params"
  
  description = "Custom parameters for MyApp PostgreSQL 15"
  
  # Connection settings
  parameter {
    name  = "max_connections"
    value = "500"
  }
  
  # Memory settings
  parameter {
    name  = "shared_buffers"
    value = "{DBInstanceClassMemory/4}"  # 25% of RAM
  }
  
  parameter {
    name  = "effective_cache_size"
    value = "{DBInstanceClassMemory*3/4}"  # 75% of RAM
  }
  
  parameter {
    name  = "work_mem"
    value = "65536"  # 64MB per sort/hash operation
  }
  
  parameter {
    name  = "maintenance_work_mem"
    value = "524288"  # 512MB for maintenance
  }
  
  # WAL settings
  parameter {
    name  = "wal_level"
    value = "replica"
  }
  
  parameter {
    name  = "max_wal_senders"
    value = "10"
  }
  
  parameter {
    name  = "wal_keep_size"
    value = "512"  # MB
  }
  
  # Query planner
  parameter {
    name  = "random_page_cost"
    value = "1.1"  # SSD storage
  }
  
  parameter {
    name  = "effective_io_concurrency"
    value = "200"  # SSD storage
  }
  
  # Logging
  parameter {
    name  = "log_min_duration_statement"
    value = "1000"  # Log queries > 1 second
  }
  
  parameter {
    name  = "log_checkpoints"
    value = "1"
  }
  
  parameter {
    name  = "log_connections"
    value = "1"
  }
  
  parameter {
    name  = "log_disconnections"
    value = "1"
  }
  
  parameter {
    name  = "log_lock_waits"
    value = "1"
  }
  
  # Autovacuum
  parameter {
    name  = "autovacuum_vacuum_scale_factor"
    value = "0.05"  # Trigger vacuum at 5% dead tuples
  }
  
  parameter {
    name  = "autovacuum_analyze_scale_factor"
    value = "0.02"  # Trigger analyze at 2%
  }
  
  parameter {
    name  = "autovacuum_vacuum_cost_delay"
    value = "2"  # ms
  }
  
  tags = {
    Environment = "production"
  }
}
```

### 2.7 RDS Proxy

RDS Proxy จัดการ connection pooling ระหว่าง application และ database:

```hcl
resource "aws_db_proxy" "postgres" {
  name                   = "myapp-rds-proxy"
  debug_logging          = false
  engine_family          = "POSTGRESQL"
  idle_client_timeout    = 1800  # 30 minutes
  require_tls            = true
  role_arn               = aws_iam_role.rds_proxy.arn
  vpc_security_group_ids = [aws_security_group.rds_proxy.id]
  vpc_subnet_ids         = aws_subnet.private[*].id
  
  auth {
    auth_scheme               = "SECRETS"
    description               = "PostgreSQL admin credentials"
    iam_auth                  = "DISABLED"
    secret_arn                = aws_secretsmanager_secret.db_credentials.arn
  }
  
  tags = {
    Environment = "production"
  }
}

resource "aws_db_proxy_default_target_group" "postgres" {
  db_proxy_name = aws_db_proxy.postgres.name
  
  connection_pool_config {
    connection_borrow_timeout    = 120   # seconds
    max_connections_percent      = 90    # % of max_connections
    max_idle_connections_percent = 50
  }
}

resource "aws_db_proxy_target" "postgres" {
  db_instance_identifier = aws_db_instance.postgres_primary.id
  db_proxy_name          = aws_db_proxy.postgres.name
  target_group_name      = aws_db_proxy_default_target_group.postgres.name
}
```

**ประโยชน์ของ RDS Proxy:**
- ลด connection overhead บน database
- รองรับ connection spikes ได้ดีขึ้น
- IAM authentication
- TLS encryption
- Failover ไวขึ้น (ไม่ต้อง re-establish connections)

### 2.8 Automated Backups และ Point-in-Time Recovery

```hcl
resource "aws_db_instance" "postgres" {
  # ...
  
  # Automated backups
  backup_retention_period = 30        # เก็บ 30 วัน
  backup_window           = "02:00-03:00"  # UTC
  
  # Manual snapshot ก่อน maintenance
  delete_automated_backups = false
}

# สร้าง Manual Snapshot
resource "aws_db_snapshot" "before_migration" {
  db_instance_identifier = aws_db_instance.postgres.id
  db_snapshot_identifier = "myapp-before-v2-migration"
  
  tags = {
    Purpose = "Pre-migration backup"
    Version = "v2.0.0"
  }
}
```

**การทำ Point-in-Time Recovery ผ่าน AWS CLI:**

```bash
# Restore to specific point in time
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier myapp-postgres-prod \
  --target-db-instance-identifier myapp-postgres-restored \
  --restore-time 2024-01-15T10:30:00Z \
  --db-instance-class db.r5.xlarge \
  --db-subnet-group-name myapp-db-subnet-group \
  --vpc-security-group-ids sg-0123456789abcdef0

# ดู restore status
aws rds describe-db-instances \
  --db-instance-identifier myapp-postgres-restored \
  --query 'DBInstances[0].DBInstanceStatus'
```

---

## 3. Amazon ElastiCache Redis

### 3.1 ElastiCache Redis Overview

```
┌─────────────────────────────────────────────┐
│         ElastiCache Redis Cluster            │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │         Cluster Mode ON              │    │
│  │                                      │    │
│  │  ┌──────────┐  ┌──────────┐         │    │
│  │  │  Shard 0 │  │  Shard 1 │  ...    │    │
│  │  │ Primary  │  │ Primary  │         │    │
│  │  │  Slots   │  │  Slots   │         │    │
│  │  │ 0-5460   │  │5461-10922│         │    │
│  │  └────┬─────┘  └────┬─────┘         │    │
│  │       │              │               │    │
│  │  ┌────▼─────┐  ┌────▼─────┐         │    │
│  │  │ Replica  │  │ Replica  │         │    │
│  │  │  AZ-b    │  │  AZ-b    │         │    │
│  │  └──────────┘  └──────────┘         │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

### 3.2 ElastiCache Cluster Setup

```hcl
# Subnet Group
resource "aws_elasticache_subnet_group" "main" {
  name       = "myapp-redis-subnet-group"
  subnet_ids = aws_subnet.database[*].id
  
  tags = {
    Environment = "production"
  }
}

# Parameter Group
resource "aws_elasticache_parameter_group" "redis7" {
  family = "redis7"
  name   = "myapp-redis7-params"
  
  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"
  }
  
  parameter {
    name  = "maxmemory-samples"
    value = "10"
  }
  
  parameter {
    name  = "lazyfree-lazy-eviction"
    value = "yes"
  }
  
  parameter {
    name  = "lazyfree-lazy-expire"
    value = "yes"
  }
  
  parameter {
    name  = "save"
    value = ""  # Disable RDB snapshots (use AOF only)
  }
  
  parameter {
    name  = "appendonly"
    value = "yes"
  }
  
  parameter {
    name  = "appendfsync"
    value = "everysec"
  }
  
  parameter {
    name  = "activerehashing"
    value = "yes"
  }
  
  parameter {
    name  = "hz"
    value = "15"
  }
}

# Redis Replication Group (Cluster Mode ON)
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id = "myapp-redis-cluster"
  description          = "MyApp Redis Cluster with Cluster Mode"
  
  # Engine
  engine               = "redis"
  engine_version       = "7.0"
  node_type            = "cache.r6g.large"
  
  # Cluster Mode
  num_node_groups         = 3  # 3 shards
  replicas_per_node_group = 2  # 2 replicas per shard
  
  # Networking
  subnet_group_name    = aws_elasticache_subnet_group.main.name
  security_group_ids   = [aws_security_group.redis.id]
  
  # Security
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = random_password.redis_auth.result
  
  # Availability
  automatic_failover_enabled = true
  multi_az_enabled           = true
  
  # Maintenance
  maintenance_window         = "sun:05:00-sun:06:00"
  snapshot_window            = "03:00-04:00"
  snapshot_retention_limit   = 7
  
  # Parameters
  parameter_group_name = aws_elasticache_parameter_group.redis7.name
  
  # Notifications
  notification_topic_arn = aws_sns_topic.cache_events.arn
  
  # Auto minor version upgrade
  auto_minor_version_upgrade = true
  
  tags = {
    Environment = "production"
    Service     = "cache"
  }
}
```

### 3.3 Node Types สำหรับ ElastiCache

| Family | Type | vCPU | Memory | Network |
|--------|------|------|--------|---------|
| cache.t3.micro | Burstable | 2 | 0.5 GB | Low |
| cache.t3.small | Burstable | 2 | 1.37 GB | Low |
| cache.t3.medium | Burstable | 2 | 3.09 GB | Moderate |
| cache.m6g.large | General | 2 | 6.38 GB | Moderate |
| cache.m6g.xlarge | General | 4 | 12.93 GB | High |
| cache.r6g.large | Memory | 2 | 13.07 GB | High |
| cache.r6g.xlarge | Memory | 4 | 26.32 GB | High |
| cache.r6g.2xlarge | Memory | 8 | 52.82 GB | High |

### 3.4 Security Configuration

```hcl
# Security Group สำหรับ Redis
resource "aws_security_group" "redis" {
  name        = "myapp-redis-sg"
  description = "Security group for ElastiCache Redis"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
    description     = "Allow Redis from application"
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "myapp-redis-sg"
  }
}
```

---

## 4. Amazon S3

### 4.1 S3 Bucket Configuration

```hcl
# Main Application Bucket
resource "aws_s3_bucket" "app_assets" {
  bucket = "myapp-assets-${data.aws_caller_identity.current.account_id}"
  
  tags = {
    Environment = "production"
    Purpose     = "Application assets"
  }
}

# Versioning
resource "aws_s3_bucket_versioning" "app_assets" {
  bucket = aws_s3_bucket.app_assets.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "app_assets" {
  bucket = aws_s3_bucket.app_assets.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true
  }
}

# Block Public Access
resource "aws_s3_bucket_public_access_block" "app_assets" {
  bucket = aws_s3_bucket.app_assets.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Lifecycle Rules
resource "aws_s3_bucket_lifecycle_configuration" "app_assets" {
  bucket = aws_s3_bucket.app_assets.id
  
  rule {
    id     = "archive-old-uploads"
    status = "Enabled"
    
    filter {
      prefix = "uploads/"
    }
    
    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }
    
    transition {
      days          = 365
      storage_class = "GLACIER"
    }
    
    expiration {
      days = 2555  # 7 years
    }
  }
  
  rule {
    id     = "delete-temp-files"
    status = "Enabled"
    
    filter {
      prefix = "temp/"
    }
    
    expiration {
      days = 1
    }
  }
}

# CORS Configuration สำหรับ direct upload
resource "aws_s3_bucket_cors_configuration" "app_assets" {
  bucket = aws_s3_bucket.app_assets.id
  
  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["PUT", "POST", "GET"]
    allowed_origins = ["https://myapp.com", "https://www.myapp.com"]
    expose_headers  = ["ETag"]
    max_age_seconds = 3000
  }
}

# Database Backups Bucket
resource "aws_s3_bucket" "db_backups" {
  bucket = "myapp-db-backups-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_lifecycle_configuration" "db_backups" {
  bucket = aws_s3_bucket.db_backups.id
  
  rule {
    id     = "backup-retention"
    status = "Enabled"
    
    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
    
    transition {
      days          = 90
      storage_class = "GLACIER"
    }
    
    expiration {
      days = 365  # Keep 1 year
    }
  }
}
```

---

## 5. CloudFront CDN

```hcl
# CloudFront Distribution
resource "aws_cloudfront_distribution" "main" {
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  price_class         = "PriceClass_All"
  
  aliases = ["www.myapp.com", "myapp.com"]
  
  # Origin - ALB
  origin {
    origin_id   = "alb-origin"
    domain_name = aws_lb.main.dns_name
    
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
    
    custom_header {
      name  = "X-CloudFront-Secret"
      value = random_password.cloudfront_secret.result
    }
  }
  
  # Origin - S3 Static Assets
  origin {
    origin_id                = "s3-assets-origin"
    domain_name              = aws_s3_bucket.app_assets.bucket_regional_domain_name
    
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.assets.cloudfront_access_identity_path
    }
  }
  
  # Default Cache Behavior (ALB)
  default_cache_behavior {
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "alb-origin"
    viewer_protocol_policy = "redirect-to-https"
    compress               = true
    
    forwarded_values {
      query_string = true
      headers      = ["Authorization", "Host", "Accept"]
      
      cookies {
        forward = "all"
      }
    }
    
    min_ttl     = 0
    default_ttl = 0
    max_ttl     = 0
  }
  
  # Cache Behavior สำหรับ Static Assets
  ordered_cache_behavior {
    path_pattern           = "/static/*"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "s3-assets-origin"
    viewer_protocol_policy = "redirect-to-https"
    compress               = true
    
    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
    
    min_ttl     = 0
    default_ttl = 86400    # 1 day
    max_ttl     = 31536000  # 1 year
  }
  
  # SSL Certificate
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.main.arn
    minimum_protocol_version = "TLSv1.2_2021"
    ssl_support_method       = "sni-only"
  }
  
  # Geo restriction
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
  
  # WAF
  web_acl_id = aws_wafv2_web_acl.main.arn
  
  tags = {
    Environment = "production"
  }
}
```

---

## 6. VPC Network Design

### 6.1 VPC Setup

```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name        = "myapp-vpc"
    Environment = "production"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "myapp-igw"
  }
}

# Public Subnets (2 AZs)
resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  map_public_ip_on_launch = true
  
  tags = {
    Name = "myapp-public-${count.index + 1}"
    Tier = "public"
  }
}

# Private Subnets สำหรับ Application
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "myapp-private-${count.index + 1}"
    Tier = "private"
  }
}

# Database Subnets (isolated)
resource "aws_subnet" "database" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 20}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "myapp-database-${count.index + 1}"
    Tier = "database"
  }
}

# NAT Gateway (per AZ for HA)
resource "aws_eip" "nat" {
  count  = 2
  domain = "vpc"
  
  tags = {
    Name = "myapp-nat-eip-${count.index + 1}"
  }
}

resource "aws_nat_gateway" "main" {
  count         = 2
  subnet_id     = aws_subnet.public[count.index].id
  allocation_id = aws_eip.nat[count.index].id
  
  tags = {
    Name = "myapp-nat-${count.index + 1}"
  }
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "myapp-public-rt"
  }
}

resource "aws_route_table" "private" {
  count  = 2
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }
  
  tags = {
    Name = "myapp-private-rt-${count.index + 1}"
  }
}

resource "aws_route_table" "database" {
  vpc_id = aws_vpc.main.id
  
  # No internet route - database subnets are isolated
  
  tags = {
    Name = "myapp-database-rt"
  }
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

resource "aws_route_table_association" "database" {
  count          = 2
  subnet_id      = aws_subnet.database[count.index].id
  route_table_id = aws_route_table.database.id
}
```

### 6.2 Security Groups

```hcl
# ALB Security Group
resource "aws_security_group" "alb" {
  name        = "myapp-alb-sg"
  description = "ALB security group"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Application Security Group
resource "aws_security_group" "app" {
  name        = "myapp-app-sg"
  description = "Application security group"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
    description     = "Allow from ALB"
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# RDS Security Group
resource "aws_security_group" "rds" {
  name        = "myapp-rds-sg"
  description = "RDS security group"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
    description     = "Allow PostgreSQL from app"
  }
  
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.rds_proxy.id]
    description     = "Allow from RDS Proxy"
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## 7. Secrets Manager

```hcl
# Database Credentials
resource "aws_secretsmanager_secret" "db_credentials" {
  name                    = "myapp/production/db/credentials"
  description             = "RDS PostgreSQL credentials"
  recovery_window_in_days = 7
  kms_key_id              = aws_kms_key.secrets.arn
  
  tags = {
    Environment = "production"
    Service     = "database"
  }
}

resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id
  
  secret_string = jsonencode({
    username = "myapp_admin"
    password = random_password.db_password.result
    host     = aws_db_instance.postgres_primary.address
    port     = 5432
    dbname   = "myapp"
    engine   = "postgres"
  })
}

# Redis Auth Token
resource "aws_secretsmanager_secret" "redis_auth" {
  name                    = "myapp/production/redis/auth"
  description             = "ElastiCache Redis auth token"
  recovery_window_in_days = 7
}

resource "aws_secretsmanager_secret_version" "redis_auth" {
  secret_id = aws_secretsmanager_secret.redis_auth.id
  
  secret_string = jsonencode({
    auth_token = random_password.redis_auth.result
    host       = aws_elasticache_replication_group.redis.configuration_endpoint_address
    port       = 6379
  })
}

# Application Secrets
resource "aws_secretsmanager_secret" "app_secrets" {
  name = "myapp/production/app/secrets"
  
  tags = {
    Environment = "production"
  }
}

resource "aws_secretsmanager_secret_version" "app_secrets" {
  secret_id = aws_secretsmanager_secret.app_secrets.id
  
  secret_string = jsonencode({
    jwt_secret          = random_password.jwt_secret.result
    encryption_key      = random_password.encryption_key.result
    stripe_secret_key   = var.stripe_secret_key
    sendgrid_api_key    = var.sendgrid_api_key
    cloudfront_secret   = random_password.cloudfront_secret.result
  })
}
```

---

## 8. Parameter Store

```hcl
# Application Configuration
resource "aws_ssm_parameter" "app_config" {
  for_each = {
    "/myapp/production/app/log_level"          = "info"
    "/myapp/production/app/max_pool_size"      = "20"
    "/myapp/production/app/min_pool_size"      = "5"
    "/myapp/production/app/idle_timeout"       = "10000"
    "/myapp/production/app/connection_timeout" = "5000"
    "/myapp/production/redis/ttl_default"      = "3600"
    "/myapp/production/redis/ttl_session"      = "86400"
    "/myapp/production/s3/bucket_name"         = aws_s3_bucket.app_assets.bucket
    "/myapp/production/cloudfront/domain"      = "myapp.com"
  }
  
  name  = each.key
  value = each.value
  type  = "String"
  
  tags = {
    Environment = "production"
  }
}

# SecureString Parameters
resource "aws_ssm_parameter" "app_env" {
  name      = "/myapp/production/app/environment"
  value     = "production"
  type      = "SecureString"
  key_id    = aws_kms_key.ssm.arn
}
```

---

## 9. Terraform: Full Infrastructure

### 9.1 Project Structure

```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── versions.tf
├── data.tf
├── locals.tf
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── rds/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── elasticache/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── ecs/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── environments/
    ├── staging/
    │   ├── main.tf
    │   └── terraform.tfvars
    └── production/
        ├── main.tf
        └── terraform.tfvars
```

### 9.2 main.tf (Full Configuration)

```hcl
# main.tf

terraform {
  required_version = ">= 1.5.0"
  
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
    bucket         = "myapp-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "myapp-terraform-lock"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = "MyApp"
      Environment = var.environment
      ManagedBy   = "Terraform"
      Owner       = "platform-team"
    }
  }
}

# Data sources
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}
data "aws_availability_zones" "available" {
  state = "available"
}

# Random passwords
resource "random_password" "db_password" {
  length           = 32
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}

resource "random_password" "redis_auth" {
  length  = 64
  special = false
}

resource "random_password" "jwt_secret" {
  length  = 64
  special = false
}

resource "random_password" "encryption_key" {
  length  = 32
  special = false
}

resource "random_password" "cloudfront_secret" {
  length  = 32
  special = false
}

# KMS Keys
resource "aws_kms_key" "rds" {
  description             = "KMS key for RDS encryption"
  deletion_window_in_days = 7
  enable_key_rotation     = true
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      }
    ]
  })
}

resource "aws_kms_alias" "rds" {
  name          = "alias/myapp-rds"
  target_key_id = aws_kms_key.rds.key_id
}

resource "aws_kms_key" "s3" {
  description             = "KMS key for S3 encryption"
  deletion_window_in_days = 7
  enable_key_rotation     = true
}

resource "aws_kms_key" "secrets" {
  description             = "KMS key for Secrets Manager"
  deletion_window_in_days = 7
  enable_key_rotation     = true
}

resource "aws_kms_key" "ssm" {
  description             = "KMS key for SSM Parameter Store"
  deletion_window_in_days = 7
  enable_key_rotation     = true
}

# VPC
module "vpc" {
  source = "./modules/vpc"
  
  vpc_cidr           = var.vpc_cidr
  availability_zones = slice(data.aws_availability_zones.available.names, 0, 2)
  environment        = var.environment
  project            = var.project_name
}

# IAM Roles
resource "aws_iam_role" "rds_monitoring" {
  name = "myapp-rds-monitoring-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = {
        Service = "monitoring.rds.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "rds_monitoring" {
  role       = aws_iam_role.rds_monitoring.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole"
}

resource "aws_iam_role" "rds_proxy" {
  name = "myapp-rds-proxy-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = {
        Service = "rds.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy" "rds_proxy_secrets" {
  name = "rds-proxy-secrets-access"
  role = aws_iam_role.rds_proxy.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ]
      Resource = [
        aws_secretsmanager_secret.db_credentials.arn
      ]
    }]
  })
}

# DB Subnet Group
resource "aws_db_subnet_group" "main" {
  name       = "myapp-db-subnet-group"
  subnet_ids = module.vpc.database_subnet_ids
  
  tags = {
    Name = "myapp-db-subnet-group"
  }
}

# RDS Parameter Group
resource "aws_db_parameter_group" "postgres15" {
  family = "postgres15"
  name   = "myapp-postgres15-params"
  
  parameter {
    name  = "max_connections"
    value = "500"
  }
  
  parameter {
    name  = "shared_buffers"
    value = "{DBInstanceClassMemory/4}"
    apply_method = "pending-reboot"
  }
  
  parameter {
    name  = "effective_cache_size"
    value = "{DBInstanceClassMemory*3/4}"
  }
  
  parameter {
    name  = "work_mem"
    value = "65536"
  }
  
  parameter {
    name  = "maintenance_work_mem"
    value = "524288"
    apply_method = "pending-reboot"
  }
  
  parameter {
    name  = "log_min_duration_statement"
    value = "1000"
  }
  
  parameter {
    name  = "log_checkpoints"
    value = "1"
  }
  
  parameter {
    name  = "autovacuum_vacuum_scale_factor"
    value = "0.05"
  }
  
  parameter {
    name  = "autovacuum_analyze_scale_factor"
    value = "0.02"
  }
  
  parameter {
    name  = "random_page_cost"
    value = "1.1"
  }
  
  parameter {
    name  = "effective_io_concurrency"
    value = "200"
  }
}

# RDS Instance
resource "aws_db_instance" "postgres_primary" {
  identifier        = "${var.project_name}-postgres-${var.environment}"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = var.db_instance_class
  
  allocated_storage     = var.db_allocated_storage
  max_allocated_storage = var.db_max_allocated_storage
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds.arn
  
  db_name  = var.db_name
  username = "myapp_admin"
  password = random_password.db_password.result
  port     = 5432
  
  multi_az               = var.db_multi_az
  availability_zone      = data.aws_availability_zones.available.names[0]
  
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  publicly_accessible    = false
  
  backup_retention_period = var.db_backup_retention
  backup_window           = "02:00-03:00"
  maintenance_window      = "sun:04:00-sun:05:00"
  
  monitoring_interval    = 60
  monitoring_role_arn    = aws_iam_role.rds_monitoring.arn
  
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  
  performance_insights_enabled          = true
  performance_insights_retention_period = 7
  
  parameter_group_name = aws_db_parameter_group.postgres15.name
  
  deletion_protection       = var.environment == "production"
  skip_final_snapshot       = var.environment != "production"
  final_snapshot_identifier = "${var.project_name}-postgres-final-${formatdate("YYYY-MM-DD", timestamp())}"
  
  lifecycle {
    ignore_changes = [password]
  }
}

# Read Replicas (Production only)
resource "aws_db_instance" "postgres_replica" {
  count = var.environment == "production" ? var.db_read_replica_count : 0
  
  identifier          = "${var.project_name}-postgres-replica-${count.index + 1}-${var.environment}"
  replicate_source_db = aws_db_instance.postgres_primary.identifier
  instance_class      = var.db_replica_instance_class
  
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn
  
  vpc_security_group_ids = [aws_security_group.rds_replica.id]
  publicly_accessible    = false
  
  performance_insights_enabled = true
  monitoring_interval          = 60
  monitoring_role_arn           = aws_iam_role.rds_monitoring.arn
  
  auto_minor_version_upgrade = true
}

# RDS Proxy
resource "aws_db_proxy" "postgres" {
  count = var.environment == "production" ? 1 : 0
  
  name                   = "${var.project_name}-rds-proxy-${var.environment}"
  engine_family          = "POSTGRESQL"
  idle_client_timeout    = 1800
  require_tls            = true
  role_arn               = aws_iam_role.rds_proxy.arn
  vpc_security_group_ids = [aws_security_group.rds_proxy.id]
  vpc_subnet_ids         = module.vpc.private_subnet_ids
  
  auth {
    auth_scheme = "SECRETS"
    iam_auth    = "DISABLED"
    secret_arn  = aws_secretsmanager_secret.db_credentials.arn
  }
}

# ElastiCache
resource "aws_elasticache_subnet_group" "main" {
  name       = "${var.project_name}-redis-subnet-group"
  subnet_ids = module.vpc.database_subnet_ids
}

resource "aws_elasticache_parameter_group" "redis7" {
  family = "redis7"
  name   = "${var.project_name}-redis7-params"
  
  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"
  }
  
  parameter {
    name  = "appendonly"
    value = "yes"
  }
}

resource "aws_elasticache_replication_group" "redis" {
  replication_group_id = "${var.project_name}-redis-${var.environment}"
  description          = "${var.project_name} Redis Cluster"
  
  engine         = "redis"
  engine_version = "7.0"
  node_type      = var.redis_node_type
  
  num_node_groups         = var.redis_num_shards
  replicas_per_node_group = var.redis_replicas_per_shard
  
  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = random_password.redis_auth.result
  
  automatic_failover_enabled = true
  multi_az_enabled           = var.environment == "production"
  
  maintenance_window       = "sun:05:00-sun:06:00"
  snapshot_window          = "03:00-04:00"
  snapshot_retention_limit = 7
  
  parameter_group_name = aws_elasticache_parameter_group.redis7.name
  
  auto_minor_version_upgrade = true
}

# S3 Buckets
resource "aws_s3_bucket" "app_assets" {
  bucket = "${var.project_name}-assets-${data.aws_caller_identity.current.account_id}-${var.environment}"
}

resource "aws_s3_bucket_versioning" "app_assets" {
  bucket = aws_s3_bucket.app_assets.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "app_assets" {
  bucket = aws_s3_bucket.app_assets.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "app_assets" {
  bucket = aws_s3_bucket.app_assets.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Secrets Manager
resource "aws_secretsmanager_secret" "db_credentials" {
  name                    = "${var.project_name}/${var.environment}/db/credentials"
  recovery_window_in_days = var.environment == "production" ? 30 : 7
  kms_key_id              = aws_kms_key.secrets.arn
}

resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id
  
  secret_string = jsonencode({
    username = "myapp_admin"
    password = random_password.db_password.result
    host     = aws_db_instance.postgres_primary.address
    port     = 5432
    dbname   = var.db_name
    engine   = "postgres"
  })
  
  lifecycle {
    ignore_changes = [secret_string]
  }
}

# CloudWatch Alarms
resource "aws_cloudwatch_metric_alarm" "rds_cpu" {
  alarm_name          = "${var.project_name}-rds-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "CPUUtilization"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "RDS CPU utilization is too high"
  
  dimensions = {
    DBInstanceIdentifier = aws_db_instance.postgres_primary.id
  }
  
  alarm_actions = [aws_sns_topic.alerts.arn]
  ok_actions    = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "rds_connections" {
  alarm_name          = "${var.project_name}-rds-high-connections"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "DatabaseConnections"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 400
  
  dimensions = {
    DBInstanceIdentifier = aws_db_instance.postgres_primary.id
  }
  
  alarm_actions = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "elasticache_memory" {
  alarm_name          = "${var.project_name}-redis-low-memory"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "DatabaseMemoryUsagePercentage"
  namespace           = "AWS/ElastiCache"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  
  dimensions = {
    ReplicationGroupId = aws_elasticache_replication_group.redis.id
  }
  
  alarm_actions = [aws_sns_topic.alerts.arn]
}

# SNS Topic สำหรับ alerts
resource "aws_sns_topic" "alerts" {
  name = "${var.project_name}-alerts-${var.environment}"
}

resource "aws_sns_topic_subscription" "alerts_email" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email
}
```

### 9.3 variables.tf

```hcl
variable "aws_region" {
  description = "AWS Region"
  default     = "ap-southeast-1"
}

variable "environment" {
  description = "Environment name"
  default     = "production"
}

variable "project_name" {
  description = "Project name"
  default     = "myapp"
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  default     = "10.0.0.0/16"
}

variable "db_instance_class" {
  description = "RDS instance class"
  default     = "db.r5.xlarge"
}

variable "db_allocated_storage" {
  description = "Initial RDS storage in GB"
  default     = 100
}

variable "db_max_allocated_storage" {
  description = "Maximum RDS storage in GB"
  default     = 1000
}

variable "db_name" {
  description = "Database name"
  default     = "myapp"
}

variable "db_multi_az" {
  description = "Enable Multi-AZ"
  default     = true
}

variable "db_backup_retention" {
  description = "Backup retention period in days"
  default     = 30
}

variable "db_read_replica_count" {
  description = "Number of read replicas"
  default     = 2
}

variable "db_replica_instance_class" {
  description = "Read replica instance class"
  default     = "db.r5.large"
}

variable "redis_node_type" {
  description = "ElastiCache node type"
  default     = "cache.r6g.large"
}

variable "redis_num_shards" {
  description = "Number of Redis shards"
  default     = 3
}

variable "redis_replicas_per_shard" {
  description = "Replicas per shard"
  default     = 2
}

variable "alert_email" {
  description = "Email for alerts"
  type        = string
}
```

### 9.4 outputs.tf

```hcl
output "rds_endpoint" {
  description = "RDS primary endpoint"
  value       = aws_db_instance.postgres_primary.endpoint
  sensitive   = true
}

output "rds_replica_endpoints" {
  description = "RDS replica endpoints"
  value       = aws_db_instance.postgres_replica[*].endpoint
  sensitive   = true
}

output "rds_proxy_endpoint" {
  description = "RDS Proxy endpoint"
  value       = length(aws_db_proxy.postgres) > 0 ? aws_db_proxy.postgres[0].endpoint : null
  sensitive   = true
}

output "redis_cluster_endpoint" {
  description = "Redis cluster configuration endpoint"
  value       = aws_elasticache_replication_group.redis.configuration_endpoint_address
  sensitive   = true
}

output "s3_bucket_name" {
  description = "S3 bucket name"
  value       = aws_s3_bucket.app_assets.bucket
}

output "db_credentials_secret_arn" {
  description = "ARN of DB credentials secret"
  value       = aws_secretsmanager_secret.db_credentials.arn
}

output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = module.vpc.private_subnet_ids
}
```

---

## 10. Cost Estimation

### 10.1 Production Cost Breakdown (ap-southeast-1)

| Service | Configuration | Monthly Cost (USD) |
|---------|--------------|-------------------|
| RDS Primary | db.r5.xlarge, Multi-AZ, 100GB gp3 | ~$380 |
| RDS Read Replicas x2 | db.r5.large each | ~$280 |
| RDS Proxy | per connection | ~$50 |
| ElastiCache | 3 shards x 3 nodes (cache.r6g.large) | ~$540 |
| Data Transfer | ~100GB/month | ~$10 |
| S3 Storage | 1TB | ~$25 |
| CloudFront | 1TB transfer | ~$85 |
| ALB | 1 unit + data | ~$30 |
| NAT Gateway | 2 AZs + 100GB | ~$70 |
| Secrets Manager | 10 secrets | ~$4 |
| **Total (approx)** | | **~$1,474/month** |

### 10.2 Staging Cost (ลดค่าใช้จ่าย)

| Service | Configuration | Monthly Cost (USD) |
|---------|--------------|-------------------|
| RDS | db.t3.medium, Single-AZ, 20GB gp3 | ~$35 |
| ElastiCache | 1 shard, 1 replica (cache.t3.medium) | ~$50 |
| **Total (approx)** | | **~$150/month** |

### 10.3 Cost Optimization Tips

```hcl
# ใช้ Reserved Instances ลดค่าใช้จ่าย 40-60%
# ใช้ Savings Plans

# Auto-scaling สำหรับ ElastiCache
resource "aws_appautoscaling_target" "redis" {
  max_capacity       = 10
  min_capacity       = 1
  resource_id        = "replication-group/${aws_elasticache_replication_group.redis.id}"
  scalable_dimension = "elasticache:replication-group:NodeGroups"
  service_namespace  = "elasticache"
}

# S3 Intelligent-Tiering
resource "aws_s3_bucket_lifecycle_configuration" "intelligent_tiering" {
  bucket = aws_s3_bucket.app_assets.id
  
  rule {
    id     = "intelligent-tiering"
    status = "Enabled"
    
    transition {
      days          = 0
      storage_class = "INTELLIGENT_TIERING"
    }
  }
}
```

---

## 11. Production Checklist

### Pre-deployment Checklist

```bash
#!/bin/bash
# pre-deployment-check.sh

echo "=== AWS Database Cluster Pre-deployment Check ==="

# 1. Check Terraform state
echo "1. Checking Terraform state..."
cd terraform/environments/production
terraform plan -detailed-exitcode
if [ $? -eq 2 ]; then
  echo "WARNING: Terraform plan shows changes"
fi

# 2. Verify RDS accessibility
echo "2. Checking RDS connectivity..."
aws rds describe-db-instances \
  --db-instance-identifier myapp-postgres-prod \
  --query 'DBInstances[0].{Status:DBInstanceStatus,Endpoint:Endpoint.Address}' \
  --output table

# 3. Check ElastiCache status
echo "3. Checking ElastiCache status..."
aws elasticache describe-replication-groups \
  --replication-group-id myapp-redis-cluster \
  --query 'ReplicationGroups[0].{Status:Status,Endpoint:ConfigurationEndpoint.Address}' \
  --output table

# 4. Verify Secrets
echo "4. Checking Secrets Manager..."
aws secretsmanager list-secrets \
  --filter Key=name,Values=myapp/production \
  --query 'SecretList[].{Name:Name,Updated:LastChangedDate}' \
  --output table

# 5. Check CloudWatch alarms
echo "5. Checking CloudWatch alarms..."
aws cloudwatch describe-alarms \
  --state-value ALARM \
  --query 'MetricAlarms[].{Name:AlarmName,State:StateValue,Reason:StateReason}' \
  --output table

# 6. Verify backups
echo "6. Checking latest backups..."
aws rds describe-db-snapshots \
  --db-instance-identifier myapp-postgres-prod \
  --snapshot-type automated \
  --query 'DBSnapshots[-1].{Status:Status,Time:SnapshotCreateTime}' \
  --output table

echo "=== Pre-deployment check complete ==="
```

### Security Checklist

- [ ] RDS ไม่ expose publicly
- [ ] ElastiCache ไม่ expose publicly
- [ ] Security groups จำกัด ingress เฉพาะที่จำเป็น
- [ ] Storage encryption เปิดใช้งาน
- [ ] Secrets เก็บใน Secrets Manager ไม่ใช่ environment variables
- [ ] KMS keys มี rotation enabled
- [ ] VPC Flow Logs เปิดใช้งาน
- [ ] CloudTrail เปิดใช้งาน
- [ ] IAM roles ใช้ least privilege
- [ ] S3 buckets block public access
- [ ] CloudFront ใช้ HTTPS only
- [ ] ACM certificates valid

### Monitoring Checklist

- [ ] CloudWatch dashboards สร้างแล้ว
- [ ] Alarms สำหรับ RDS CPU, connections, storage
- [ ] Alarms สำหรับ ElastiCache memory, evictions
- [ ] SNS notifications ตั้งค่าแล้ว
- [ ] Performance Insights เปิดใช้งาน
- [ ] Enhanced Monitoring เปิดใช้งาน
- [ ] Log groups สร้างใน CloudWatch

### Backup Checklist

- [ ] RDS automated backups retention ≥ 30 วัน
- [ ] Manual snapshots ก่อน major changes
- [ ] ElastiCache snapshots เปิดใช้งาน
- [ ] S3 versioning เปิดใช้งาน
- [ ] ทดสอบ restore procedure แล้ว
- [ ] Point-in-time recovery ทดสอบแล้ว

---

## สรุป

AWS มีบริการครบครันสำหรับการสร้าง Database Cluster ที่ production-ready:

1. **RDS PostgreSQL** จัดการ database หลักพร้อม Multi-AZ และ Read Replicas
2. **ElastiCache Redis** จัดการ cache layer พร้อม Cluster Mode
3. **RDS Proxy** จัดการ connection pooling
4. **Secrets Manager** เก็บ credentials อย่างปลอดภัย
5. **Terraform** จัดการ infrastructure ทั้งหมด

การใช้ Terraform ช่วยให้ infrastructure เป็น code ที่ reproducible และสามารถ version control ได้ ทำให้ deployment ปลอดภัยและสม่ำเสมอ

---

*เนื้อหาส่วนนี้เป็นส่วนหนึ่งของ Database Cluster Course - Advanced Content*
