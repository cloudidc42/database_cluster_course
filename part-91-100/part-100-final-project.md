# Part 100: Final Project - สร้าง Production-Grade E-Commerce System

# ShopCluster: Production-Grade E-Commerce Platform

## บทนำ

ยินดีต้อนรับสู่ Final Project ของ Database Cluster Course คุณจะสร้าง **ShopCluster** ซึ่งเป็น production-grade e-commerce platform ที่รองรับ:

- 100,000+ users พร้อมกัน
- 1,000,000+ products
- 10,000+ orders ต่อวัน
- 99.9% availability
- P99 latency < 200ms
- Global deployment (US + EU + APAC)

---

## Architecture Overview

```
                         ┌─────────────────────────────────────────────┐
                         │              Internet                        │
                         └────────────────────┬────────────────────────┘
                                              │
                         ┌────────────────────▼────────────────────────┐
                         │           CloudFlare CDN                     │
                         └────────────────────┬────────────────────────┘
                                              │
                    ┌─────────────────────────▼─────────────────────────┐
                    │                   NGINX (Ingress)                  │
                    │              Load Balancer + TLS                   │
                    └──────┬──────────────────────────────┬─────────────┘
                           │                              │
         ┌─────────────────▼──────────┐   ┌──────────────▼───────────────┐
         │      API Server (3x)        │   │    Static Assets (S3/MinIO)   │
         │      Node.js + TypeScript   │   └──────────────────────────────┘
         └──┬─────────────────────┬───┘
            │                     │
    ┌───────▼──────┐    ┌──────────▼──────────┐
    │  PostgreSQL   │    │    Redis Cluster     │
    │  Primary (W)  │    │  Cache + Session     │
    │  2 Replicas(R)│    │  + Job Queue         │
    └───────────────┘    └─────────────────────┘
            │
    ┌───────▼──────────────────────────────────┐
    │          Kafka + Debezium                  │
    │          Event Streaming                   │
    └───────┬─────────────────┬─────────────────┘
            │                 │
    ┌───────▼───────┐ ┌───────▼───────────────┐
    │ Elasticsearch │ │  BullMQ Workers (3x)  │
    │ Product Search│ │  Background Jobs       │
    └───────────────┘ └───────────────────────┘
            
    ┌───────────────────────────────────────────┐
    │         MinIO Object Storage              │
    │         Product Images + Invoices         │
    └───────────────────────────────────────────┘
    
    ┌───────────────────────────────────────────┐
    │         Prometheus + Grafana              │
    │         Monitoring + Alerting             │
    └───────────────────────────────────────────┘
```

---

## Deliverable 1: Database Schema (Complete SQL)

```sql
-- ============================================
-- ShopCluster Database Schema
-- PostgreSQL 15+
-- ============================================

-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- ============================================
-- USERS & AUTHENTICATION
-- ============================================

CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email         VARCHAR(255) UNIQUE NOT NULL,
  username      VARCHAR(100) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  full_name     VARCHAR(255),
  phone         VARCHAR(20),
  avatar_url    VARCHAR(500),
  role          VARCHAR(20) NOT NULL DEFAULT 'customer'
                  CHECK (role IN ('customer', 'admin', 'seller')),
  is_verified   BOOLEAN NOT NULL DEFAULT false,
  is_active     BOOLEAN NOT NULL DEFAULT true,
  last_login_at TIMESTAMP WITH TIME ZONE,
  created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  deleted_at    TIMESTAMP WITH TIME ZONE  -- soft delete
);

CREATE TABLE user_sessions (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  refresh_token VARCHAR(255) UNIQUE NOT NULL,
  user_agent    TEXT,
  ip_address    INET,
  expires_at    TIMESTAMP WITH TIME ZONE NOT NULL,
  created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE addresses (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  label         VARCHAR(50) NOT NULL DEFAULT 'Home',  -- Home, Office, etc.
  recipient_name VARCHAR(255) NOT NULL,
  phone         VARCHAR(20) NOT NULL,
  address_line1 VARCHAR(255) NOT NULL,
  address_line2 VARCHAR(255),
  city          VARCHAR(100) NOT NULL,
  state         VARCHAR(100) NOT NULL,
  postal_code   VARCHAR(20) NOT NULL,
  country       CHAR(2) NOT NULL DEFAULT 'TH',
  is_default    BOOLEAN NOT NULL DEFAULT false,
  created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ============================================
-- PRODUCT CATALOG
-- ============================================

CREATE TABLE categories (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id   UUID REFERENCES categories(id),
  slug        VARCHAR(255) UNIQUE NOT NULL,
  name        VARCHAR(255) NOT NULL,
  description TEXT,
  image_url   VARCHAR(500),
  sort_order  INTEGER NOT NULL DEFAULT 0,
  is_active   BOOLEAN NOT NULL DEFAULT true,
  created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE brands (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug        VARCHAR(255) UNIQUE NOT NULL,
  name        VARCHAR(255) NOT NULL,
  logo_url    VARCHAR(500),
  website_url VARCHAR(500),
  is_active   BOOLEAN NOT NULL DEFAULT true,
  created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE products (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  sku             VARCHAR(100) UNIQUE NOT NULL,
  slug            VARCHAR(255) UNIQUE NOT NULL,
  name            VARCHAR(500) NOT NULL,
  description     TEXT,
  short_desc      VARCHAR(500),
  category_id     UUID REFERENCES categories(id),
  brand_id        UUID REFERENCES brands(id),
  base_price      DECIMAL(12, 2) NOT NULL CHECK (base_price >= 0),
  sale_price      DECIMAL(12, 2) CHECK (sale_price >= 0),
  cost_price      DECIMAL(12, 2),
  weight_kg       DECIMAL(8, 3),
  dimensions_cm   JSONB,  -- {"length": 10, "width": 10, "height": 5}
  attributes      JSONB NOT NULL DEFAULT '{}',  -- {"color": "red", "size": "L"}
  tags            TEXT[] NOT NULL DEFAULT '{}',
  status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'active', 'inactive', 'deleted')),
  is_featured     BOOLEAN NOT NULL DEFAULT false,
  rating_avg      DECIMAL(3, 2) DEFAULT 0.00,
  rating_count    INTEGER NOT NULL DEFAULT 0,
  total_sold      INTEGER NOT NULL DEFAULT 0,
  created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  deleted_at      TIMESTAMP WITH TIME ZONE,
  
  -- Full-text search
  search_vector   TSVECTOR GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(short_desc, '')), 'B') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'C')
  ) STORED
);

CREATE TABLE product_images (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id  UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  url         VARCHAR(500) NOT NULL,
  alt_text    VARCHAR(255),
  sort_order  INTEGER NOT NULL DEFAULT 0,
  is_primary  BOOLEAN NOT NULL DEFAULT false,
  created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE inventory (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id          UUID NOT NULL UNIQUE REFERENCES products(id),
  warehouse_id        UUID,
  quantity_on_hand    INTEGER NOT NULL DEFAULT 0 CHECK (quantity_on_hand >= 0),
  quantity_reserved   INTEGER NOT NULL DEFAULT 0 CHECK (quantity_reserved >= 0),
  quantity_available  INTEGER GENERATED ALWAYS AS (
    quantity_on_hand - quantity_reserved
  ) STORED,
  reorder_point       INTEGER NOT NULL DEFAULT 10,
  reorder_quantity    INTEGER NOT NULL DEFAULT 100,
  low_stock_threshold INTEGER NOT NULL DEFAULT 5,
  updated_at          TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ============================================
-- ORDERS
-- ============================================

CREATE TABLE orders (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_number    VARCHAR(20) UNIQUE NOT NULL,
  user_id         UUID NOT NULL REFERENCES users(id),
  status          VARCHAR(30) NOT NULL DEFAULT 'pending'
                    CHECK (status IN (
                      'pending', 'payment_pending', 'payment_confirmed',
                      'processing', 'shipped', 'delivered',
                      'cancelled', 'refunded', 'partially_refunded'
                    )),
  
  -- Amounts
  subtotal        DECIMAL(12, 2) NOT NULL CHECK (subtotal >= 0),
  discount_amount DECIMAL(12, 2) NOT NULL DEFAULT 0,
  shipping_fee    DECIMAL(12, 2) NOT NULL DEFAULT 0,
  tax_amount      DECIMAL(12, 2) NOT NULL DEFAULT 0,
  total_amount    DECIMAL(12, 2) NOT NULL CHECK (total_amount >= 0),
  currency        CHAR(3) NOT NULL DEFAULT 'THB',
  
  -- Shipping
  shipping_address  JSONB NOT NULL,
  shipping_method   VARCHAR(50),
  tracking_number   VARCHAR(100),
  shipped_at        TIMESTAMP WITH TIME ZONE,
  delivered_at      TIMESTAMP WITH TIME ZONE,
  
  -- Promotion
  coupon_code       VARCHAR(50),
  coupon_discount   DECIMAL(12, 2) DEFAULT 0,
  
  -- Metadata
  notes             TEXT,
  ip_address        INET,
  user_agent        TEXT,
  
  created_at        TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at        TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE order_items (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id      UUID NOT NULL REFERENCES orders(id),
  product_id    UUID NOT NULL REFERENCES products(id),
  product_name  VARCHAR(500) NOT NULL,  -- snapshot ชื่อตอน order
  product_sku   VARCHAR(100) NOT NULL,
  product_image VARCHAR(500),
  unit_price    DECIMAL(12, 2) NOT NULL,
  quantity      INTEGER NOT NULL CHECK (quantity > 0),
  total_price   DECIMAL(12, 2) GENERATED ALWAYS AS (unit_price * quantity) STORED,
  refunded_qty  INTEGER NOT NULL DEFAULT 0,
  created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ============================================
-- PAYMENTS
-- ============================================

CREATE TABLE payments (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id            UUID NOT NULL REFERENCES orders(id),
  payment_method      VARCHAR(50) NOT NULL,  -- card, promptpay, truemoney
  status              VARCHAR(30) NOT NULL DEFAULT 'pending'
                        CHECK (status IN (
                          'pending', 'processing', 'succeeded',
                          'failed', 'cancelled', 'refunded'
                        )),
  amount              DECIMAL(12, 2) NOT NULL,
  currency            CHAR(3) NOT NULL DEFAULT 'THB',
  gateway             VARCHAR(50),            -- stripe, omise, etc.
  gateway_payment_id  VARCHAR(255) UNIQUE,    -- external ID
  gateway_response    JSONB,                   -- raw response from gateway
  failure_reason      TEXT,
  refund_amount       DECIMAL(12, 2) DEFAULT 0,
  processed_at        TIMESTAMP WITH TIME ZONE,
  created_at          TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at          TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ============================================
-- REVIEWS
-- ============================================

CREATE TABLE reviews (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id  UUID NOT NULL REFERENCES products(id),
  user_id     UUID NOT NULL REFERENCES users(id),
  order_id    UUID REFERENCES orders(id),
  rating      SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
  title       VARCHAR(255),
  body        TEXT,
  images      TEXT[] DEFAULT '{}',
  is_verified BOOLEAN NOT NULL DEFAULT false,  -- verified purchase
  helpful_count INTEGER NOT NULL DEFAULT 0,
  status      VARCHAR(20) NOT NULL DEFAULT 'pending'
                CHECK (status IN ('pending', 'approved', 'rejected')),
  created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  UNIQUE (product_id, user_id, order_id)  -- 1 review per product per order
);

-- ============================================
-- CART (stored in Redis, but has DB backup)
-- ============================================

CREATE TABLE cart_items (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  product_id  UUID NOT NULL REFERENCES products(id),
  quantity    INTEGER NOT NULL CHECK (quantity > 0),
  created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  UNIQUE (user_id, product_id)
);

-- ============================================
-- INDEXES
-- ============================================

-- Users
CREATE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_role ON users(role) WHERE is_active = true;

-- Sessions
CREATE INDEX idx_sessions_refresh_token ON user_sessions(refresh_token);
CREATE INDEX idx_sessions_user_id ON user_sessions(user_id);
CREATE INDEX idx_sessions_expires ON user_sessions(expires_at);

-- Categories (tree structure)
CREATE INDEX idx_categories_parent ON categories(parent_id) WHERE is_active = true;
CREATE INDEX idx_categories_slug ON categories(slug);

-- Products
CREATE INDEX idx_products_category ON products(category_id, status, created_at DESC)
  WHERE deleted_at IS NULL;
CREATE INDEX idx_products_brand ON products(brand_id, status)
  WHERE deleted_at IS NULL;
CREATE INDEX idx_products_status ON products(status, created_at DESC)
  WHERE deleted_at IS NULL;
CREATE INDEX idx_products_featured ON products(is_featured, created_at DESC)
  WHERE status = 'active' AND deleted_at IS NULL;
CREATE INDEX idx_products_price ON products(base_price, status)
  WHERE deleted_at IS NULL;
CREATE INDEX idx_products_search ON products USING GIN(search_vector);
CREATE INDEX idx_products_tags ON products USING GIN(tags);
CREATE INDEX idx_products_attributes ON products USING GIN(attributes);

-- Inventory
CREATE INDEX idx_inventory_low_stock ON inventory(quantity_available)
  WHERE quantity_available <= low_stock_threshold;

-- Orders
CREATE INDEX idx_orders_user ON orders(user_id, created_at DESC);
CREATE INDEX idx_orders_status ON orders(status, created_at DESC);
CREATE INDEX idx_orders_number ON orders(order_number);
CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id, created_at DESC);

-- Payments
CREATE INDEX idx_payments_order ON payments(order_id);
CREATE INDEX idx_payments_gateway_id ON payments(gateway_payment_id);
CREATE INDEX idx_payments_status ON payments(status, created_at DESC);

-- Reviews
CREATE INDEX idx_reviews_product ON reviews(product_id, status, created_at DESC);
CREATE INDEX idx_reviews_user ON reviews(user_id);

-- ============================================
-- ROW LEVEL SECURITY
-- ============================================

-- Enable RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE addresses ENABLE ROW LEVEL SECURITY;

-- Users can only see their own data
CREATE POLICY users_isolation ON users
  USING (id = current_setting('app.current_user_id')::UUID
         OR current_setting('app.current_user_role') = 'admin');

CREATE POLICY orders_isolation ON orders
  USING (user_id = current_setting('app.current_user_id')::UUID
         OR current_setting('app.current_user_role') = 'admin');

CREATE POLICY addresses_isolation ON addresses
  USING (user_id = current_setting('app.current_user_id')::UUID
         OR current_setting('app.current_user_role') = 'admin');

-- ============================================
-- TRIGGERS
-- ============================================

-- Auto update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER update_products_updated_at
  BEFORE UPDATE ON products
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER update_orders_updated_at
  BEFORE UPDATE ON orders
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- Auto generate order number
CREATE OR REPLACE FUNCTION generate_order_number()
RETURNS TRIGGER AS $$
BEGIN
  NEW.order_number = 'ORD-' ||
    to_char(NOW(), 'YYYYMMDD') || '-' ||
    LPAD(nextval('order_number_seq')::TEXT, 6, '0');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE SEQUENCE order_number_seq START 100001;

CREATE TRIGGER set_order_number
  BEFORE INSERT ON orders
  FOR EACH ROW EXECUTE FUNCTION generate_order_number();

-- Update product rating when review changes
CREATE OR REPLACE FUNCTION update_product_rating()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE products
  SET
    rating_avg = subq.avg_rating,
    rating_count = subq.count
  FROM (
    SELECT
      product_id,
      ROUND(AVG(rating)::numeric, 2) as avg_rating,
      COUNT(*) as count
    FROM reviews
    WHERE product_id = COALESCE(NEW.product_id, OLD.product_id)
      AND status = 'approved'
    GROUP BY product_id
  ) AS subq
  WHERE id = subq.product_id;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_rating_on_review
  AFTER INSERT OR UPDATE OR DELETE ON reviews
  FOR EACH ROW EXECUTE FUNCTION update_product_rating();
```

---

## Deliverable 2: Docker Compose (Full Local Dev Stack)

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ============================================
  # PostgreSQL Primary + Replica
  # ============================================
  postgres-primary:
    image: postgres:15-alpine
    container_name: shopcluster-postgres-primary
    hostname: postgres-primary
    environment:
      POSTGRES_DB: shopcluster
      POSTGRES_USER: shopcluster_admin
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-shopcluster_dev_pass}
      POSTGRES_REPLICATION_USER: replicator
      POSTGRES_REPLICATION_PASSWORD: ${REPLICATION_PASSWORD:-repl_pass_dev}
    ports:
      - "5432:5432"
    volumes:
      - postgres_primary_data:/var/lib/postgresql/data
      - ./docker/postgres/primary/postgresql.conf:/etc/postgresql/postgresql.conf
      - ./docker/postgres/primary/pg_hba.conf:/etc/postgresql/pg_hba.conf
      - ./docker/postgres/init-scripts:/docker-entrypoint-initdb.d
    command: >
      postgres
      -c config_file=/etc/postgresql/postgresql.conf
      -c hba_file=/etc/postgresql/pg_hba.conf
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U shopcluster_admin -d shopcluster"]
      interval: 10s
      timeout: 5s
      retries: 5

  postgres-replica:
    image: postgres:15-alpine
    container_name: shopcluster-postgres-replica
    hostname: postgres-replica
    environment:
      POSTGRES_USER: shopcluster_admin
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-shopcluster_dev_pass}
      PGUSER: replicator
      PGPASSWORD: ${REPLICATION_PASSWORD:-repl_pass_dev}
    ports:
      - "5433:5432"
    volumes:
      - postgres_replica_data:/var/lib/postgresql/data
      - ./docker/postgres/replica/setup-replica.sh:/docker-entrypoint-initdb.d/setup-replica.sh
    depends_on:
      postgres-primary:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U shopcluster_admin"]
      interval: 10s
      timeout: 5s
      retries: 5

  # PgBouncer connection pooler
  pgbouncer:
    image: pgbouncer/pgbouncer:1.22
    container_name: shopcluster-pgbouncer
    ports:
      - "6432:5432"
    volumes:
      - ./docker/pgbouncer/pgbouncer.ini:/etc/pgbouncer/pgbouncer.ini
      - ./docker/pgbouncer/userlist.txt:/etc/pgbouncer/userlist.txt
    depends_on:
      - postgres-primary

  # ============================================
  # Redis Cluster (3 nodes for local dev)
  # ============================================
  redis-node-1:
    image: redis:7-alpine
    container_name: shopcluster-redis-1
    ports:
      - "7001:7001"
      - "17001:17001"
    command: >
      redis-server
      --port 7001
      --cluster-enabled yes
      --cluster-config-file nodes-7001.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --requirepass ${REDIS_PASSWORD:-redis_dev_pass}
    volumes:
      - redis_1_data:/data

  redis-node-2:
    image: redis:7-alpine
    container_name: shopcluster-redis-2
    ports:
      - "7002:7002"
      - "17002:17002"
    command: >
      redis-server
      --port 7002
      --cluster-enabled yes
      --cluster-config-file nodes-7002.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --requirepass ${REDIS_PASSWORD:-redis_dev_pass}
    volumes:
      - redis_2_data:/data

  redis-node-3:
    image: redis:7-alpine
    container_name: shopcluster-redis-3
    ports:
      - "7003:7003"
      - "17003:17003"
    command: >
      redis-server
      --port 7003
      --cluster-enabled yes
      --cluster-config-file nodes-7003.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --requirepass ${REDIS_PASSWORD:-redis_dev_pass}
    volumes:
      - redis_3_data:/data

  redis-cluster-init:
    image: redis:7-alpine
    container_name: shopcluster-redis-cluster-init
    depends_on:
      - redis-node-1
      - redis-node-2
      - redis-node-3
    command: >
      sh -c "
        sleep 5 &&
        redis-cli
          --cluster create
          redis-node-1:7001
          redis-node-2:7002
          redis-node-3:7003
          -a ${REDIS_PASSWORD:-redis_dev_pass}
          --cluster-replicas 0
          --cluster-yes
      "

  # ============================================
  # MinIO Object Storage
  # ============================================
  minio:
    image: minio/minio:latest
    container_name: shopcluster-minio
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER:-minioadmin}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD:-minioadmin123}
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 30s
      timeout: 20s
      retries: 3

  minio-init:
    image: minio/mc:latest
    depends_on:
      minio:
        condition: service_healthy
    entrypoint: >
      /bin/sh -c "
        mc alias set myminio http://minio:9000 minioadmin minioadmin123;
        mc mb myminio/product-images --ignore-existing;
        mc mb myminio/invoices --ignore-existing;
        mc mb myminio/reports --ignore-existing;
        mc policy set public myminio/product-images;
        exit 0;
      "

  # ============================================
  # Kafka + Zookeeper + Debezium
  # ============================================
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: shopcluster-zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper_data:/var/lib/zookeeper/data

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: shopcluster-kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    volumes:
      - kafka_data:/var/lib/kafka/data

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: shopcluster-kafka-ui
    ports:
      - "8090:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092

  debezium:
    image: debezium/connect:2.4
    container_name: shopcluster-debezium
    ports:
      - "8083:8083"
    environment:
      BOOTSTRAP_SERVERS: kafka:29092
      GROUP_ID: debezium-group
      CONFIG_STORAGE_TOPIC: debezium-configs
      OFFSET_STORAGE_TOPIC: debezium-offsets
      STATUS_STORAGE_TOPIC: debezium-status
    depends_on:
      - kafka
      - postgres-primary

  # ============================================
  # Elasticsearch
  # ============================================
  elasticsearch:
    image: elasticsearch:8.10.0
    container_name: shopcluster-elasticsearch
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9200/_health | grep green"]
      interval: 20s
      timeout: 10s
      retries: 5

  kibana:
    image: kibana:8.10.0
    container_name: shopcluster-kibana
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
    depends_on:
      - elasticsearch

  # ============================================
  # Prometheus + Grafana
  # ============================================
  prometheus:
    image: prom/prometheus:latest
    container_name: shopcluster-prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./docker/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./docker/prometheus/rules:/etc/prometheus/rules
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--storage.tsdb.retention.time=30d'

  grafana:
    image: grafana/grafana:latest
    container_name: shopcluster-grafana
    ports:
      - "3001:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
      GF_INSTALL_PLUGINS: grafana-piechart-panel,grafana-worldmap-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./docker/grafana/provisioning:/etc/grafana/provisioning
      - ./docker/grafana/dashboards:/var/lib/grafana/dashboards
    depends_on:
      - prometheus

  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:latest
    container_name: shopcluster-postgres-exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://shopcluster_admin:${POSTGRES_PASSWORD:-shopcluster_dev_pass}@postgres-primary:5432/shopcluster?sslmode=disable"
    depends_on:
      - postgres-primary

  redis-exporter:
    image: oliver006/redis_exporter:latest
    container_name: shopcluster-redis-exporter
    environment:
      REDIS_ADDR: redis-node-1:7001
      REDIS_PASSWORD: ${REDIS_PASSWORD:-redis_dev_pass}
    depends_on:
      - redis-node-1

volumes:
  postgres_primary_data:
  postgres_replica_data:
  redis_1_data:
  redis_2_data:
  redis_3_data:
  minio_data:
  zookeeper_data:
  kafka_data:
  elasticsearch_data:
  prometheus_data:
  grafana_data:
```

---

## Deliverable 3: Key API Implementations

### 3.1 Product Catalog with Search + Cache

```typescript
// src/services/product.service.ts
import { Pool } from 'pg';
import Redis from 'ioredis';
import { Client as ElasticsearchClient } from '@elastic/elasticsearch';

interface ProductFilter {
  categoryId?: string;
  brandId?: string;
  minPrice?: number;
  maxPrice?: number;
  inStockOnly?: boolean;
  tags?: string[];
}

interface ProductListOptions {
  page: number;
  limit: number;
  sort: 'created_at' | 'price_asc' | 'price_desc' | 'rating' | 'popular';
  filter?: ProductFilter;
}

interface SearchOptions {
  query: string;
  page: number;
  limit: number;
  filter?: ProductFilter;
  facets?: boolean;
}

export class ProductService {
  private readonly CACHE_TTL = 300;        // 5 minutes
  private readonly CACHE_PREFIX = 'prod:';

  constructor(
    private readonly db: Pool,
    private readonly redis: Redis,
    private readonly es: ElasticsearchClient,
  ) {}

  // ดึง list products พร้อม caching
  async listProducts(options: ProductListOptions) {
    const { page, limit, sort, filter } = options;
    const offset = (page - 1) * limit;

    const cacheKey = `${this.CACHE_PREFIX}list:${JSON.stringify({ page, limit, sort, filter })}`;

    // ตรวจ cache ก่อน
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return { ...JSON.parse(cached), fromCache: true };
    }

    // Build query
    const conditions: string[] = ["p.status = 'active'", "p.deleted_at IS NULL"];
    const params: any[] = [];
    let paramIndex = 1;

    if (filter?.categoryId) {
      conditions.push(`p.category_id = $${paramIndex++}`);
      params.push(filter.categoryId);
    }

    if (filter?.minPrice !== undefined) {
      conditions.push(`p.base_price >= $${paramIndex++}`);
      params.push(filter.minPrice);
    }

    if (filter?.maxPrice !== undefined) {
      conditions.push(`p.base_price <= $${paramIndex++}`);
      params.push(filter.maxPrice);
    }

    if (filter?.inStockOnly) {
      conditions.push(`i.quantity_available > 0`);
    }

    if (filter?.tags && filter.tags.length > 0) {
      conditions.push(`p.tags && $${paramIndex++}`);
      params.push(filter.tags);
    }

    const sortClause = {
      created_at: 'p.created_at DESC',
      price_asc: 'p.base_price ASC',
      price_desc: 'p.base_price DESC',
      rating: 'p.rating_avg DESC, p.rating_count DESC',
      popular: 'p.total_sold DESC',
    }[sort] || 'p.created_at DESC';

    const whereClause = conditions.join(' AND ');

    params.push(limit, offset);
    const limitParam = paramIndex++;
    const offsetParam = paramIndex++;

    const query = `
      SELECT
        p.id, p.sku, p.slug, p.name, p.short_desc,
        p.base_price, p.sale_price, p.rating_avg, p.rating_count,
        p.total_sold, p.is_featured,
        COALESCE(pi.url, '') as primary_image,
        i.quantity_available,
        c.name as category_name,
        b.name as brand_name
      FROM products p
      LEFT JOIN product_images pi ON pi.product_id = p.id AND pi.is_primary = true
      LEFT JOIN inventory i ON i.product_id = p.id
      LEFT JOIN categories c ON c.id = p.category_id
      LEFT JOIN brands b ON b.id = p.brand_id
      WHERE ${whereClause}
      ORDER BY ${sortClause}
      LIMIT $${limitParam} OFFSET $${offsetParam}
    `;

    const countQuery = `
      SELECT COUNT(*) FROM products p
      LEFT JOIN inventory i ON i.product_id = p.id
      WHERE ${whereClause}
    `;

    const [productsResult, countResult] = await Promise.all([
      this.db.query(query, params),
      this.db.query(countQuery, params.slice(0, -2)), // remove limit/offset
    ]);

    const result = {
      data: productsResult.rows,
      total: parseInt(countResult.rows[0].count),
      page,
      limit,
      pages: Math.ceil(parseInt(countResult.rows[0].count) / limit),
    };

    // Cache ผลลัพธ์
    await this.redis.setex(cacheKey, this.CACHE_TTL, JSON.stringify(result));

    return { ...result, fromCache: false };
  }

  // ดึง product เดียว (cache by ID)
  async getProduct(idOrSlug: string) {
    const cacheKey = `${this.CACHE_PREFIX}detail:${idOrSlug}`;

    const cached = await this.redis.get(cacheKey);
    if (cached) return { ...JSON.parse(cached), fromCache: true };

    const result = await this.db.query(`
      SELECT
        p.*,
        COALESCE(json_agg(DISTINCT pi.*) FILTER (WHERE pi.id IS NOT NULL), '[]') as images,
        row_to_json(c.*) as category,
        row_to_json(b.*) as brand,
        row_to_json(i.*) as inventory
      FROM products p
      LEFT JOIN product_images pi ON pi.product_id = p.id
      LEFT JOIN categories c ON c.id = p.category_id
      LEFT JOIN brands b ON b.id = p.brand_id
      LEFT JOIN inventory i ON i.product_id = p.id
      WHERE (p.id = $1 OR p.slug = $1)
        AND p.deleted_at IS NULL
      GROUP BY p.id, c.id, b.id, i.id
    `, [idOrSlug]);

    if (result.rows.length === 0) return null;

    const product = result.rows[0];
    await this.redis.setex(cacheKey, this.CACHE_TTL, JSON.stringify(product));

    return { ...product, fromCache: false };
  }

  // Full-text search ด้วย Elasticsearch
  async searchProducts(options: SearchOptions) {
    const { query, page, limit, filter, facets } = options;
    const from = (page - 1) * limit;

    const must: any[] = [
      {
        multi_match: {
          query,
          fields: ['name^3', 'short_desc^2', 'description', 'tags'],
          type: 'best_fields',
          fuzziness: 'AUTO',
        },
      },
      { term: { status: 'active' } },
    ];

    const filterClauses: any[] = [];

    if (filter?.categoryId) {
      filterClauses.push({ term: { category_id: filter.categoryId } });
    }
    if (filter?.minPrice !== undefined || filter?.maxPrice !== undefined) {
      filterClauses.push({
        range: {
          base_price: {
            ...(filter.minPrice !== undefined && { gte: filter.minPrice }),
            ...(filter.maxPrice !== undefined && { lte: filter.maxPrice }),
          },
        },
      });
    }
    if (filter?.inStockOnly) {
      filterClauses.push({ range: { quantity_available: { gt: 0 } } });
    }

    const esQuery: any = {
      index: 'products',
      from,
      size: limit,
      query: {
        bool: { must, filter: filterClauses },
      },
      _source: [
        'id', 'sku', 'slug', 'name', 'short_desc',
        'base_price', 'sale_price', 'rating_avg', 'primary_image',
        'category_name', 'brand_name', 'quantity_available',
      ],
      highlight: {
        fields: {
          name: {},
          short_desc: {},
        },
      },
    };

    if (facets) {
      esQuery.aggs = {
        categories: {
          terms: { field: 'category_id', size: 20 },
          aggs: {
            category_name: { terms: { field: 'category_name', size: 1 } },
          },
        },
        brands: { terms: { field: 'brand_id', size: 20 } },
        price_ranges: {
          range: {
            field: 'base_price',
            ranges: [
              { to: 500 },
              { from: 500, to: 1000 },
              { from: 1000, to: 5000 },
              { from: 5000, to: 10000 },
              { from: 10000 },
            ],
          },
        },
        avg_rating: { avg: { field: 'rating_avg' } },
      };
    }

    const response = await this.es.search(esQuery);

    return {
      data: response.hits.hits.map(hit => ({
        ...hit._source,
        _score: hit._score,
        _highlight: hit.highlight,
      })),
      total: (response.hits.total as any).value,
      page,
      limit,
      facets: response.aggregations,
    };
  }
}
```

### 3.2 Order Placement ด้วย Saga Pattern

```typescript
// src/services/order.service.ts
// Saga Pattern: ถ้าขั้นตอนใด fail, ต้อง compensate ขั้นตอนก่อนหน้า

interface OrderItem {
  productId: string;
  quantity: number;
}

interface CreateOrderInput {
  userId: string;
  items: OrderItem[];
  shippingAddressId: string;
  paymentToken: string;
  couponCode?: string;
}

enum SagaStep {
  VALIDATE_STOCK = 'validate_stock',
  RESERVE_STOCK = 'reserve_stock',
  CALCULATE_TOTAL = 'calculate_total',
  APPLY_COUPON = 'apply_coupon',
  CREATE_ORDER_RECORD = 'create_order_record',
  PROCESS_PAYMENT = 'process_payment',
  CONFIRM_STOCK_DEDUCTION = 'confirm_stock_deduction',
  SEND_CONFIRMATION = 'send_confirmation',
}

export class OrderService {
  constructor(
    private readonly db: Pool,
    private readonly redis: Redis,
    private readonly paymentGateway: PaymentGateway,
    private readonly emailQueue: Queue,
  ) {}

  async createOrder(input: CreateOrderInput) {
    const sagaLog: Array<{ step: SagaStep; data: any }> = [];

    try {
      // Step 1: Validate stock
      const productPrices = await this.validateAndGetPrices(input.items);
      sagaLog.push({ step: SagaStep.VALIDATE_STOCK, data: productPrices });

      // Step 2: Reserve stock (temporary hold)
      const reservationId = await this.reserveStock(input.items);
      sagaLog.push({ step: SagaStep.RESERVE_STOCK, data: { reservationId } });

      // Step 3: Calculate total
      const totals = this.calculateTotals(input.items, productPrices);
      sagaLog.push({ step: SagaStep.CALCULATE_TOTAL, data: totals });

      // Step 4: Apply coupon (if any)
      let discount = 0;
      if (input.couponCode) {
        discount = await this.applyCoupon(input.couponCode, totals.subtotal);
        sagaLog.push({ step: SagaStep.APPLY_COUPON, data: { discount } });
      }

      // Step 5: Create order record
      const order = await this.createOrderRecord({
        ...input,
        ...totals,
        discount,
      });
      sagaLog.push({ step: SagaStep.CREATE_ORDER_RECORD, data: { orderId: order.id } });

      // Step 6: Process payment
      const payment = await this.paymentGateway.charge({
        amount: totals.total - discount,
        currency: 'THB',
        token: input.paymentToken,
        metadata: { orderId: order.id },
      });

      if (payment.status !== 'succeeded') {
        throw new Error(`Payment failed: ${payment.failureReason}`);
      }
      sagaLog.push({ step: SagaStep.PROCESS_PAYMENT, data: { paymentId: payment.id } });

      // Step 7: Confirm stock deduction (remove reservation, actually deduct)
      await this.confirmStockDeduction(reservationId, input.items);
      sagaLog.push({ step: SagaStep.CONFIRM_STOCK_DEDUCTION, data: {} });

      // Step 8: Send confirmation email (async)
      await this.emailQueue.add('send_order_confirmation', {
        orderId: order.id,
        userId: input.userId,
      });

      // Invalidate caches
      await this.invalidateCaches(input.userId, input.items.map(i => i.productId));

      return {
        orderId: order.id,
        orderNumber: order.orderNumber,
        total: totals.total - discount,
        paymentId: payment.id,
      };

    } catch (error) {
      // Compensate completed steps in reverse order
      await this.compensate(sagaLog, input);
      throw error;
    }
  }

  private async reserveStock(items: OrderItem[]): Promise<string> {
    const reservationId = crypto.randomUUID();
    const client = await this.db.connect();

    try {
      await client.query('BEGIN');

      for (const item of items) {
        const result = await client.query(`
          UPDATE inventory
          SET quantity_reserved = quantity_reserved + $1,
              updated_at = NOW()
          WHERE product_id = $2
            AND (quantity_on_hand - quantity_reserved) >= $1
          RETURNING id
        `, [item.quantity, item.productId]);

        if (result.rows.length === 0) {
          throw new Error(`Insufficient stock for product ${item.productId}`);
        }
      }

      // Store reservation in Redis with 15 min TTL
      await this.redis.setex(
        `reservation:${reservationId}`,
        900,  // 15 minutes
        JSON.stringify(items)
      );

      await client.query('COMMIT');
      return reservationId;

    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  private async compensate(
    sagaLog: Array<{ step: SagaStep; data: any }>,
    input: CreateOrderInput
  ) {
    // Compensate in reverse order
    for (const entry of sagaLog.reverse()) {
      try {
        switch (entry.step) {
          case SagaStep.RESERVE_STOCK:
            await this.releaseStockReservation(
              entry.data.reservationId,
              input.items
            );
            break;

          case SagaStep.CREATE_ORDER_RECORD:
            await this.db.query(
              "UPDATE orders SET status = 'cancelled' WHERE id = $1",
              [entry.data.orderId]
            );
            break;

          case SagaStep.PROCESS_PAYMENT:
            await this.paymentGateway.refund(entry.data.paymentId);
            break;
        }
      } catch (compensationError) {
        console.error(`Compensation failed for step ${entry.step}:`, compensationError);
        // Log for manual intervention
      }
    }
  }

  private async releaseStockReservation(
    reservationId: string,
    items: OrderItem[]
  ) {
    const client = await this.db.connect();
    try {
      await client.query('BEGIN');
      for (const item of items) {
        await client.query(`
          UPDATE inventory
          SET quantity_reserved = GREATEST(0, quantity_reserved - $1),
              updated_at = NOW()
          WHERE product_id = $2
        `, [item.quantity, item.productId]);
      }
      await client.query('COMMIT');
      await this.redis.del(`reservation:${reservationId}`);
    } catch (error) {
      await client.query('ROLLBACK');
    } finally {
      client.release();
    }
  }

  private calculateTotals(items: OrderItem[], prices: Map<string, number>) {
    let subtotal = 0;
    for (const item of items) {
      subtotal += (prices.get(item.productId) || 0) * item.quantity;
    }
    const shippingFee = subtotal > 1000 ? 0 : 50;  // free shipping > 1000 THB
    const taxRate = 0.07;  // 7% VAT
    const taxAmount = subtotal * taxRate;
    const total = subtotal + shippingFee + taxAmount;

    return { subtotal, shippingFee, taxAmount, total };
  }
}
```

### 3.3 Shopping Cart ด้วย Redis

```typescript
// src/services/cart.service.ts
import Redis from 'ioredis';

export class CartService {
  private readonly TTL = 86400 * 7;  // 7 days
  private readonly MAX_ITEMS = 50;

  constructor(private readonly redis: Redis) {}

  private cartKey(userId: string) {
    return `cart:${userId}`;
  }

  async getCart(userId: string) {
    const data = await this.redis.hgetall(this.cartKey(userId));

    const items = Object.entries(data).map(([productId, value]) => ({
      productId,
      ...JSON.parse(value),
    }));

    return {
      userId,
      items,
      itemCount: items.length,
      totalQuantity: items.reduce((sum, item) => sum + item.quantity, 0),
    };
  }

  async addItem(userId: string, productId: string, quantity: number, productInfo: any) {
    const key = this.cartKey(userId);

    // ตรวจ limit
    const count = await this.redis.hlen(key);
    const existing = await this.redis.hget(key, productId);

    if (!existing && count >= this.MAX_ITEMS) {
      throw new Error(`Cart cannot have more than ${this.MAX_ITEMS} items`);
    }

    const currentQty = existing ? JSON.parse(existing).quantity : 0;
    const newQty = currentQty + quantity;

    if (newQty > productInfo.maxQuantity) {
      throw new Error(`Cannot add more than ${productInfo.maxQuantity} of this item`);
    }

    await this.redis.hset(key, productId, JSON.stringify({
      quantity: newQty,
      price: productInfo.price,
      name: productInfo.name,
      image: productInfo.image,
      updatedAt: new Date().toISOString(),
    }));

    await this.redis.expire(key, this.TTL);

    return this.getCart(userId);
  }

  async updateQuantity(userId: string, productId: string, quantity: number) {
    const key = this.cartKey(userId);
    const existing = await this.redis.hget(key, productId);

    if (!existing) throw new Error('Item not found in cart');

    if (quantity <= 0) {
      return this.removeItem(userId, productId);
    }

    const item = JSON.parse(existing);
    await this.redis.hset(key, productId, JSON.stringify({
      ...item,
      quantity,
      updatedAt: new Date().toISOString(),
    }));

    return this.getCart(userId);
  }

  async removeItem(userId: string, productId: string) {
    await this.redis.hdel(this.cartKey(userId), productId);
    return this.getCart(userId);
  }

  async clearCart(userId: string) {
    await this.redis.del(this.cartKey(userId));
  }

  // Merge guest cart เมื่อ login
  async mergeGuestCart(userId: string, sessionId: string) {
    const guestCart = await this.getCart(`guest:${sessionId}`);

    for (const item of guestCart.items) {
      await this.addItem(userId, item.productId, item.quantity, {
        price: item.price,
        name: item.name,
        image: item.image,
        maxQuantity: 99,
      });
    }

    await this.redis.del(this.cartKey(`guest:${sessionId}`));
  }
}
```

---

## Deliverable 4: Background Jobs ด้วย BullMQ

```typescript
// src/workers/order-worker.ts
import { Worker, Queue, QueueScheduler } from 'bullmq';
import Redis from 'ioredis';
import { PDFDocument } from 'pdf-lib';
import { sendEmail } from '../services/email.service';

const connection = new Redis({
  host: process.env.REDIS_HOST,
  port: 6379,
  password: process.env.REDIS_PASSWORD,
  maxRetriesPerRequest: null,
});

// Queue definitions
export const emailQueue = new Queue('email', { connection });
export const invoiceQueue = new Queue('invoice', { connection });
export const inventoryQueue = new Queue('inventory', { connection });
export const reportQueue = new Queue('report', { connection });

// Schedulers (required for delayed jobs)
new QueueScheduler('email', { connection });
new QueueScheduler('invoice', { connection });
new QueueScheduler('report', { connection });

// ========== Email Worker ==========
const emailWorker = new Worker(
  'email',
  async (job) => {
    const { type, data } = job.data;

    switch (type) {
      case 'order_confirmation':
        await sendEmail({
          to: data.userEmail,
          template: 'order-confirmation',
          variables: {
            orderNumber: data.orderNumber,
            items: data.items,
            total: data.total,
            trackingUrl: `https://shopcluster.com/orders/${data.orderId}`,
          },
        });
        break;

      case 'order_shipped':
        await sendEmail({
          to: data.userEmail,
          template: 'order-shipped',
          variables: {
            orderNumber: data.orderNumber,
            trackingNumber: data.trackingNumber,
            courier: data.courier,
          },
        });
        break;

      case 'low_stock_alert':
        await sendEmail({
          to: data.adminEmail,
          template: 'low-stock-alert',
          variables: {
            products: data.products,
          },
        });
        break;
    }
  },
  {
    connection,
    concurrency: 10,
    limiter: {
      max: 100,   // max 100 emails per minute
      duration: 60000,
    },
  }
);

// ========== Invoice Worker ==========
const invoiceWorker = new Worker(
  'invoice',
  async (job) => {
    const { orderId } = job.data;

    // ดึง order details
    const order = await getOrderWithDetails(orderId);

    // Generate PDF
    const pdfDoc = await PDFDocument.create();
    const page = pdfDoc.addPage([595, 842]);  // A4

    // Header
    page.drawText('SHOPCLUSTER', {
      x: 50, y: 800,
      size: 24,
    });
    page.drawText(`Invoice #${order.orderNumber}`, {
      x: 50, y: 770,
      size: 14,
    });
    page.drawText(`Date: ${order.createdAt.toLocaleDateString()}`, {
      x: 50, y: 750,
      size: 12,
    });

    // Items table
    let y = 700;
    page.drawText('Product', { x: 50, y, size: 11 });
    page.drawText('Qty', { x: 350, y, size: 11 });
    page.drawText('Price', { x: 430, y, size: 11 });
    page.drawText('Total', { x: 510, y, size: 11 });

    y -= 20;
    for (const item of order.items) {
      page.drawText(item.productName.substring(0, 40), { x: 50, y, size: 10 });
      page.drawText(item.quantity.toString(), { x: 350, y, size: 10 });
      page.drawText(`฿${item.unitPrice.toFixed(2)}`, { x: 430, y, size: 10 });
      page.drawText(`฿${item.totalPrice.toFixed(2)}`, { x: 510, y, size: 10 });
      y -= 20;
    }

    // Total
    y -= 10;
    page.drawText(`Total: ฿${order.totalAmount.toFixed(2)}`, {
      x: 430, y,
      size: 12,
    });

    const pdfBytes = await pdfDoc.save();

    // Upload to MinIO
    const s3 = getMinIOClient();
    const fileName = `invoices/${order.orderNumber}.pdf`;

    await s3.putObject({
      Bucket: 'invoices',
      Key: fileName,
      Body: Buffer.from(pdfBytes),
      ContentType: 'application/pdf',
      Metadata: {
        orderId,
        userId: order.userId,
      },
    });

    // Update order with invoice URL
    await db.query(
      "UPDATE orders SET invoice_url = $1 WHERE id = $2",
      [`https://minio.shopcluster.com/invoices/${fileName}`, orderId]
    );

    // Queue email to send invoice
    await emailQueue.add('send_invoice', {
      type: 'invoice_ready',
      data: {
        userId: order.userId,
        orderNumber: order.orderNumber,
        invoiceUrl: `https://minio.shopcluster.com/invoices/${fileName}`,
      },
    });
  },
  { connection, concurrency: 5 }
);

// ========== Inventory Sync Worker ==========
const inventoryWorker = new Worker(
  'inventory',
  async (job) => {
    const { type, data } = job.data;

    switch (type) {
      case 'check_low_stock':
        const lowStockItems = await db.query(`
          SELECT p.id, p.name, p.sku, i.quantity_available, i.reorder_point
          FROM products p
          JOIN inventory i ON i.product_id = p.id
          WHERE i.quantity_available <= i.low_stock_threshold
            AND p.status = 'active'
        `);

        if (lowStockItems.rows.length > 0) {
          await emailQueue.add('low_stock_alert', {
            type: 'low_stock_alert',
            data: {
              adminEmail: process.env.ADMIN_EMAIL,
              products: lowStockItems.rows,
            },
          });
        }
        break;

      case 'sync_warehouse':
        // Sync with external warehouse system
        const warehouseData = await fetchWarehouseInventory();
        for (const item of warehouseData) {
          await db.query(`
            UPDATE inventory
            SET quantity_on_hand = $1,
                updated_at = NOW()
            WHERE product_id = (
              SELECT id FROM products WHERE sku = $2
            )
          `, [item.quantity, item.sku]);
        }
        break;
    }
  },
  { connection, concurrency: 3 }
);
```

---

## Deliverable 5: Kubernetes Helm Chart

```yaml
# helm/shopcluster/values.yaml
global:
  imageRegistry: registry.company.com
  imagePullSecrets:
    - name: registry-credentials

api:
  replicaCount: 3
  image:
    repository: shopcluster/api
    tag: latest
    pullPolicy: IfNotPresent
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 20
    targetCPUUtilizationPercentage: 70
    targetMemoryUtilizationPercentage: 80
  env:
    NODE_ENV: production
    PORT: "3000"
  envFrom:
    - secretRef:
        name: shopcluster-secrets

postgresql:
  enabled: true
  architecture: replication
  auth:
    existingSecret: shopcluster-db-secret
  primary:
    resources:
      requests:
        cpu: 1000m
        memory: 4Gi
      limits:
        cpu: 4000m
        memory: 8Gi
    persistence:
      size: 500Gi
      storageClass: premium-ssd
  readReplicas:
    replicaCount: 2

redis:
  enabled: true
  architecture: cluster
  cluster:
    nodes: 6  # 3 masters + 3 replicas
  auth:
    existingSecret: shopcluster-redis-secret
  resources:
    requests:
      cpu: 500m
      memory: 2Gi

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
  hosts:
    - host: api.shopcluster.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: shopcluster-tls
      hosts:
        - api.shopcluster.com
```

---

## Deliverable 7: Load Test Scripts

```javascript
// k6-tests/shopcluster-load-test.js
import http from 'k6/http';
import { sleep, check, group } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';
import { SharedArray } from 'k6/data';
import { randomItem, randomIntBetween } from 'https://jslib.k6.io/k6-utils/1.4.0/index.js';

const orderSuccessRate = new Rate('order_success_rate');
const searchLatency = new Trend('search_latency');
const checkoutLatency = new Trend('checkout_latency');
const ordersPlaced = new Counter('orders_placed');

const testUsers = new SharedArray('users', () =>
  JSON.parse(open('./test-data/users.json'))
);

const testProducts = new SharedArray('products', () =>
  JSON.parse(open('./test-data/products.json'))
);

export const options = {
  scenarios: {
    // 60% browse traffic
    browse: {
      executor: 'ramping-vus',
      startVUs: 10,
      stages: [
        { duration: '5m', target: 300 },
        { duration: '20m', target: 300 },
        { duration: '5m', target: 0 },
      ],
      exec: 'browseScenario',
      tags: { scenario: 'browse' },
    },
    // 30% search traffic
    search: {
      executor: 'constant-arrival-rate',
      rate: 200,
      timeUnit: '1s',
      duration: '30m',
      preAllocatedVUs: 100,
      exec: 'searchScenario',
      tags: { scenario: 'search' },
    },
    // 10% checkout
    checkout: {
      executor: 'ramping-vus',
      startVUs: 5,
      stages: [
        { duration: '5m', target: 50 },
        { duration: '20m', target: 50 },
        { duration: '5m', target: 0 },
      ],
      exec: 'checkoutScenario',
      tags: { scenario: 'checkout' },
    },
  },
  thresholds: {
    'http_req_duration': ['p(95)<150', 'p(99)<200'],
    'http_req_failed': ['rate<0.001'],
    'order_success_rate': ['rate>0.99'],
    'search_latency': ['p(99)<300'],
    'checkout_latency': ['p(99)<500'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://api.shopcluster.local';

export function browseScenario() {
  const user = randomItem(testUsers);
  const headers = { Authorization: `Bearer ${user.token}` };

  group('browse', () => {
    // Home page
    http.get(`${BASE_URL}/api/products?featured=true&limit=12`, { headers });
    sleep(randomIntBetween(1, 3));

    // Category
    const categories = ['electronics', 'fashion', 'home', 'sports'];
    http.get(`${BASE_URL}/api/products?category=${randomItem(categories)}&limit=20`, { headers });
    sleep(randomIntBetween(2, 5));

    // Product detail
    const product = randomItem(testProducts);
    http.get(`${BASE_URL}/api/products/${product.id}`, { headers });
    sleep(randomIntBetween(3, 8));
  });
}

export function searchScenario() {
  const terms = ['iPhone', 'laptop', 'dress', 'shoes', 'headphones', 'camera', 'watch'];
  const q = randomItem(terms);

  const start = Date.now();
  const res = http.get(`${BASE_URL}/api/products/search?q=${q}&limit=20`);
  searchLatency.add(Date.now() - start);

  check(res, {
    'search 200': (r) => r.status === 200,
    'has results': (r) => {
      try { return JSON.parse(r.body).total > 0; }
      catch { return false; }
    },
  });
  sleep(1);
}

export function checkoutScenario() {
  const user = randomItem(testUsers);
  const headers = {
    Authorization: `Bearer ${user.token}`,
    'Content-Type': 'application/json',
  };

  group('checkout flow', () => {
    // Add to cart
    const product = randomItem(testProducts);
    http.post(
      `${BASE_URL}/api/cart/items`,
      JSON.stringify({ productId: product.id, quantity: 1 }),
      { headers }
    );
    sleep(2);

    // Place order
    const start = Date.now();
    const res = http.post(
      `${BASE_URL}/api/orders`,
      JSON.stringify({
        shippingAddressId: user.addressId,
        paymentToken: 'tok_th_test_visa',
      }),
      { headers }
    );
    checkoutLatency.add(Date.now() - start);

    const ok = check(res, {
      'order placed': (r) => r.status === 201,
    });

    orderSuccessRate.add(ok ? 1 : 0);
    if (ok) ordersPlaced.add(1);
    sleep(5);
  });
}
```

---

## Deliverable 8: Security Checklist

```markdown
# ShopCluster Security Checklist

## Database Security
- [x] All credentials stored in Vault/Secrets Manager
- [x] Database network not publicly accessible
- [x] TLS enforced for all database connections
- [x] Encryption at rest (EBS/StorageClass with encryption)
- [x] Row Level Security (RLS) enabled for user data
- [x] Least privilege: app user has no DROP/TRUNCATE permissions
- [x] Audit logging enabled (pgaudit)
- [x] Regular automated backups tested
- [x] Backup encryption enabled

## Application Security
- [x] JWT tokens with short expiry (15 minutes)
- [x] Refresh token rotation
- [x] Rate limiting on all endpoints
- [x] SQL injection prevention (parameterized queries only)
- [x] XSS prevention (sanitize inputs)
- [x] CSRF protection
- [x] Helmet.js security headers
- [x] Input validation (Zod schemas)
- [x] No sensitive data in logs
- [x] Dependency vulnerability scanning (npm audit, Snyk)

## Infrastructure Security
- [x] Network policies (pods can only talk to what they need)
- [x] Pod security policies / PodSecurityAdmission
- [x] Container images scanned (Trivy)
- [x] Secrets never in environment variables (use Vault Agent)
- [x] RBAC: least privilege for service accounts
- [x] API Gateway with authentication
- [x] WAF enabled (CloudFlare)

## Compliance
- [x] PDPA: user consent for data collection
- [x] PCI DSS: no raw card data stored
- [x] GDPR: right to deletion implemented
- [x] Data retention policies configured
```

---

## สรุป: Final Project Checklist

```
ShopCluster Implementation Checklist:

Database Layer:
  ✓ PostgreSQL 15 Primary + 2 Replicas
  ✓ Complete schema with indexes, constraints, RLS
  ✓ Connection pooling (PgBouncer)
  ✓ Automated backups ทุกวัน
  ✓ Monitoring (postgres-exporter + Grafana)

Cache Layer:
  ✓ Redis Cluster (6 nodes production)
  ✓ Cart storage
  ✓ Product catalog cache (5 min TTL)
  ✓ Session management
  ✓ Rate limiting

Message Queue:
  ✓ BullMQ + Redis
  ✓ Email notifications
  ✓ Invoice generation
  ✓ Inventory sync
  ✓ Report generation

Search:
  ✓ Elasticsearch 8
  ✓ Full-text search with relevance
  ✓ Faceted search
  ✓ Debezium CDC sync from PostgreSQL

API Layer:
  ✓ Node.js + TypeScript
  ✓ Product catalog with caching
  ✓ Shopping cart (Redis)
  ✓ Order placement (Saga pattern)
  ✓ File upload (MinIO)
  ✓ Authentication (JWT + refresh)

Kubernetes:
  ✓ Helm charts for all services
  ✓ HPA (auto-scaling)
  ✓ PodDisruptionBudget
  ✓ Health checks
  ✓ Resource limits

Monitoring:
  ✓ Prometheus + Grafana
  ✓ SLI/SLO dashboards
  ✓ Error budget tracking
  ✓ Alert rules + runbooks
  ✓ PagerDuty integration

Load Testing:
  ✓ k6 test suite
  ✓ pgbench baseline
  ✓ Smoke, load, stress, spike tests
  ✓ Performance thresholds

Security:
  ✓ Secrets in Vault
  ✓ RLS on sensitive tables
  ✓ Network policies
  ✓ TLS everywhere
  ✓ Rate limiting
  ✓ Input validation

GitOps:
  ✓ ArgoCD for Kubernetes
  ✓ Database migrations in Git
  ✓ Sealed Secrets
  ✓ Environment promotion
```

**ยินดีด้วย!** คุณได้สร้าง production-grade e-commerce platform ที่รองรับ scale ระดับ enterprise โดยใช้ techniques ทั้งหมดที่เรียนมาใน course นี้ตั้งแต่ PostgreSQL clustering, Redis, MinIO, Monitoring, GitOps จนถึง SRE practices!
