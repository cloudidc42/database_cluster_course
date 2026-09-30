# Part 27: Database Migrations ด้วย Flyway/Liquibase และ Custom Solutions

## ทำไมต้องใช้ Database Migrations?

การพัฒนา application ที่ใช้ฐานข้อมูลมักเจอปัญหา:

1. **Schema drift**: Database ใน dev, staging, production ต่างกัน
2. **Collaboration**: ทีม 5 คน แก้ schema พร้อมกัน ใครทำก่อน-หลัง?
3. **Rollback**: เมื่อ deploy แล้วมีปัญหา ต้อง rollback schema ได้
4. **Audit trail**: รู้ว่า schema เปลี่ยนอะไร เมื่อไหร่ โดยใคร
5. **Automation**: Deploy schema พร้อมกับ code โดยอัตโนมัติใน CI/CD

**Database Migrations** แก้ทุกปัญหาเหล่านี้ด้วยการ:
- เก็บ schema changes เป็น versioned files
- รัน migrations ตามลำดับ
- Track ว่า migration ไหนรันแล้ว
- รองรับ rollback (down migrations)

---

## Migration File Naming Convention

```
V{version}__{description}.sql

ตัวอย่าง:
V001__create_users_table.sql
V002__add_email_to_users.sql
V003__create_products_table.sql
V004__add_index_on_users_email.sql
V005__create_orders_and_order_items.sql
```

**Rules:**
- `V` = Version prefix
- `001`, `002` = Sequential version number (zero-padded)
- `__` = Double underscore separator
- `create_users_table` = Description (snake_case)
- `.sql` = Extension

สำหรับ node-pg-migrate ใช้ timestamp:
```
1699000001000_create-users.js
1699000002000_add-email-to-users.js
```

---

## เปรียบเทียบ Migration Tools

| Tool | Language | Auto-migrate | Rollback | Transactions | ความนิยม |
|------|----------|-------------|---------|-------------|---------|
| Flyway | Java/Any | ✓ | Pro only | ✓ | สูงมาก |
| Liquibase | Java/Any | ✓ | ✓ | ✓ | สูง |
| node-pg-migrate | Node.js | ✓ | ✓ | ✓ | ปานกลาง |
| db-migrate | Node.js | ✓ | ✓ | ✓ | ปานกลาง |
| Prisma Migrate | Node.js | ✓ | ✓ | ✓ | สูง (TypeScript) |
| golang-migrate | Go | ✓ | ✓ | ✓ | สูง (Go) |

---

## node-pg-migrate

### Installation และ Setup

```bash
# Install
npm install --save-dev node-pg-migrate pg

# หรือ
yarn add --dev node-pg-migrate pg
```

### Configuration: database.json

สร้างไฟล์ `database.json` ที่ root ของ project:

```json
{
  "dev": {
    "driver": "pg",
    "host": "localhost",
    "port": 5432,
    "database": "myapp_development",
    "user": "postgres",
    "password": "secret"
  },
  "test": {
    "driver": "pg",
    "host": "localhost",
    "port": 5432,
    "database": "myapp_test",
    "user": "postgres",
    "password": "secret"
  },
  "production": {
    "driver": "pg",
    "url": { "ENV": "DATABASE_URL" }
  }
}
```

### package.json scripts

```json
{
  "scripts": {
    "migrate": "node-pg-migrate",
    "migrate:create": "node-pg-migrate create",
    "migrate:up": "node-pg-migrate up",
    "migrate:down": "node-pg-migrate down",
    "migrate:status": "node-pg-migrate status",
    "migrate:redo": "node-pg-migrate redo"
  }
}
```

### สร้าง Migration ใหม่

```bash
# สร้าง migration file
npm run migrate:create -- create-users-table

# ผลลัพธ์: migrations/1699000001000_create-users-table.js
```

### Migration File Structure

```javascript
// migrations/1699000001000_create-users-table.js

/**
 * @param { import("node-pg-migrate").MigrationBuilder } pgm
 */
exports.up = (pgm) => {
  // สร้างตาราง users
  pgm.createTable('users', {
    id: {
      type: 'uuid',
      primaryKey: true,
      default: pgm.func('gen_random_uuid()'),
    },
    email: {
      type: 'varchar(255)',
      notNull: true,
      unique: true,
    },
    name: {
      type: 'varchar(255)',
      notNull: true,
    },
    password_hash: {
      type: 'varchar(255)',
      notNull: true,
    },
    role: {
      type: 'varchar(50)',
      notNull: true,
      default: 'user',
    },
    email_verified: {
      type: 'boolean',
      default: false,
    },
    created_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
    updated_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
  });

  // สร้าง indexes
  pgm.createIndex('users', 'email');
  pgm.createIndex('users', 'created_at');
};

/**
 * @param { import("node-pg-migrate").MigrationBuilder } pgm
 */
exports.down = (pgm) => {
  pgm.dropTable('users');
};
```

```javascript
// migrations/1699000002000_create-categories-table.js
exports.up = (pgm) => {
  pgm.createTable('categories', {
    id: {
      type: 'uuid',
      primaryKey: true,
      default: pgm.func('gen_random_uuid()'),
    },
    name: {
      type: 'varchar(255)',
      notNull: true,
    },
    slug: {
      type: 'varchar(255)',
      notNull: true,
      unique: true,
    },
    parent_id: {
      type: 'uuid',
      references: 'categories',
      onDelete: 'SET NULL',
    },
    description: { type: 'text' },
    image_url: { type: 'varchar(512)' },
    sort_order: { type: 'integer', default: 0 },
    active: { type: 'boolean', default: true },
    created_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
    updated_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
  });

  pgm.createIndex('categories', 'slug');
  pgm.createIndex('categories', 'parent_id');
};

exports.down = (pgm) => {
  pgm.dropTable('categories');
};
```

```javascript
// migrations/1699000003000_create-products-table.js
exports.up = (pgm) => {
  // สร้าง enum สำหรับ product status
  pgm.createType('product_status', ['active', 'inactive', 'out_of_stock', 'discontinued']);

  pgm.createTable('products', {
    id: {
      type: 'uuid',
      primaryKey: true,
      default: pgm.func('gen_random_uuid()'),
    },
    sku: {
      type: 'varchar(100)',
      notNull: true,
      unique: true,
    },
    name: {
      type: 'varchar(500)',
      notNull: true,
    },
    slug: {
      type: 'varchar(500)',
      notNull: true,
      unique: true,
    },
    description: { type: 'text' },
    price: {
      type: 'numeric(12, 2)',
      notNull: true,
    },
    compare_price: { type: 'numeric(12, 2)' },
    cost_price: { type: 'numeric(12, 2)' },
    stock_quantity: {
      type: 'integer',
      notNull: true,
      default: 0,
    },
    category_id: {
      type: 'uuid',
      notNull: true,
      references: 'categories',
      onDelete: 'RESTRICT',
    },
    brand_id: { type: 'uuid' },
    status: {
      type: 'product_status',
      notNull: true,
      default: 'active',
    },
    weight: { type: 'numeric(8,3)' },
    images: {
      type: 'jsonb',
      default: "'[]'::jsonb",
    },
    attributes: {
      type: 'jsonb',
      default: "'{}'::jsonb",
    },
    tags: {
      type: 'text[]',
      default: "ARRAY[]::text[]",
    },
    featured: { type: 'boolean', default: false },
    created_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
    updated_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
  });

  pgm.createIndex('products', 'sku');
  pgm.createIndex('products', 'slug');
  pgm.createIndex('products', 'category_id');
  pgm.createIndex('products', 'status');
  pgm.createIndex('products', 'featured');
  pgm.createIndex('products', 'price');
  
  // GIN index สำหรับ full-text search
  pgm.createIndex('products', pgm.func("to_tsvector('thai', name || ' ' || COALESCE(description, ''))"), {
    name: 'products_search_idx',
    method: 'gin',
  });
  
  // GIN index สำหรับ tags
  pgm.createIndex('products', 'tags', { method: 'gin' });
};

exports.down = (pgm) => {
  pgm.dropTable('products');
  pgm.dropType('product_status');
};
```

```javascript
// migrations/1699000004000_create-orders-table.js
exports.up = (pgm) => {
  pgm.createType('order_status', [
    'pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded'
  ]);
  
  pgm.createType('payment_status', [
    'pending', 'paid', 'failed', 'refunded', 'partially_refunded'
  ]);

  pgm.createTable('orders', {
    id: {
      type: 'uuid',
      primaryKey: true,
      default: pgm.func('gen_random_uuid()'),
    },
    order_number: {
      type: 'varchar(50)',
      notNull: true,
      unique: true,
    },
    user_id: {
      type: 'uuid',
      notNull: true,
      references: 'users',
      onDelete: 'RESTRICT',
    },
    status: {
      type: 'order_status',
      notNull: true,
      default: 'pending',
    },
    payment_status: {
      type: 'payment_status',
      notNull: true,
      default: 'pending',
    },
    subtotal: {
      type: 'numeric(12,2)',
      notNull: true,
    },
    discount_amount: {
      type: 'numeric(12,2)',
      default: 0,
    },
    shipping_fee: {
      type: 'numeric(12,2)',
      default: 0,
    },
    tax_amount: {
      type: 'numeric(12,2)',
      default: 0,
    },
    total_amount: {
      type: 'numeric(12,2)',
      notNull: true,
    },
    shipping_address: {
      type: 'jsonb',
      notNull: true,
    },
    billing_address: { type: 'jsonb' },
    payment_method: { type: 'varchar(50)' },
    payment_reference: { type: 'varchar(255)' },
    notes: { type: 'text' },
    metadata: {
      type: 'jsonb',
      default: "'{}'::jsonb",
    },
    created_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
    updated_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
    confirmed_at: { type: 'timestamptz' },
    shipped_at: { type: 'timestamptz' },
    delivered_at: { type: 'timestamptz' },
    cancelled_at: { type: 'timestamptz' },
  });

  pgm.createTable('order_items', {
    id: {
      type: 'uuid',
      primaryKey: true,
      default: pgm.func('gen_random_uuid()'),
    },
    order_id: {
      type: 'uuid',
      notNull: true,
      references: 'orders',
      onDelete: 'CASCADE',
    },
    product_id: {
      type: 'uuid',
      notNull: true,
      references: 'products',
      onDelete: 'RESTRICT',
    },
    sku: {
      type: 'varchar(100)',
      notNull: true,
    },
    product_name: {
      type: 'varchar(500)',
      notNull: true,
    },
    quantity: {
      type: 'integer',
      notNull: true,
    },
    unit_price: {
      type: 'numeric(12,2)',
      notNull: true,
    },
    total_price: {
      type: 'numeric(12,2)',
      notNull: true,
    },
    product_snapshot: {
      type: 'jsonb',
      default: "'{}'::jsonb",
    },
    created_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
  });

  // Indexes
  pgm.createIndex('orders', 'order_number');
  pgm.createIndex('orders', 'user_id');
  pgm.createIndex('orders', 'status');
  pgm.createIndex('orders', 'payment_status');
  pgm.createIndex('orders', 'created_at');
  pgm.createIndex('order_items', 'order_id');
  pgm.createIndex('order_items', 'product_id');
};

exports.down = (pgm) => {
  pgm.dropTable('order_items');
  pgm.dropTable('orders');
  pgm.dropType('payment_status');
  pgm.dropType('order_status');
};
```

```javascript
// migrations/1699000005000_add-cart-tables.js
exports.up = (pgm) => {
  pgm.createTable('carts', {
    id: {
      type: 'uuid',
      primaryKey: true,
      default: pgm.func('gen_random_uuid()'),
    },
    user_id: {
      type: 'uuid',
      unique: true,
      references: 'users',
      onDelete: 'CASCADE',
    },
    session_id: {
      type: 'varchar(255)',
      unique: true,
    },
    expires_at: { type: 'timestamptz' },
    created_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
    updated_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
  });

  pgm.createTable('cart_items', {
    id: {
      type: 'uuid',
      primaryKey: true,
      default: pgm.func('gen_random_uuid()'),
    },
    cart_id: {
      type: 'uuid',
      notNull: true,
      references: 'carts',
      onDelete: 'CASCADE',
    },
    product_id: {
      type: 'uuid',
      notNull: true,
      references: 'products',
      onDelete: 'CASCADE',
    },
    quantity: {
      type: 'integer',
      notNull: true,
    },
    added_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
  });

  // ป้องกัน duplicate items ใน cart
  pgm.addConstraint('cart_items', 'cart_items_cart_product_unique', 
    'UNIQUE(cart_id, product_id)'
  );

  pgm.createIndex('carts', 'user_id');
  pgm.createIndex('carts', 'session_id');
  pgm.createIndex('cart_items', 'cart_id');
};

exports.down = (pgm) => {
  pgm.dropTable('cart_items');
  pgm.dropTable('carts');
};
```

### รัน Migrations

```bash
# รัน migrations ทั้งหมดที่ยังไม่ได้รัน
npm run migrate:up

# รัน แค่ 1 migration
npm run migrate:up -- --count 1

# Rollback migration ล่าสุด
npm run migrate:down

# Rollback 3 migrations
npm run migrate:down -- --count 3

# ดู status ว่า migration ไหนรันแล้ว
npm run migrate:status

# Redo (down แล้ว up อีกที)
npm run migrate:redo
```

**Output ตัวอย่าง:**

```
> node-pg-migrate up

Migrating files:
- 1699000001000_create-users-table
- 1699000002000_create-categories-table
- 1699000003000_create-products-table
- 1699000004000_create-orders-table
- 1699000005000_add-cart-tables

Running migration 1699000001000_create-users-table
Running migration 1699000002000_create-categories-table
Running migration 1699000003000_create-products-table
Running migration 1699000004000_create-orders-table
Running migration 1699000005000_add-cart-tables

Migrations complete!
```

### Tips: Idempotent Migrations

```javascript
// migrations/1699000006000_add-discount-code-to-orders.js
exports.up = (pgm) => {
  // ใช้ IF NOT EXISTS เพื่อให้ idempotent
  pgm.addColumns('orders', {
    discount_code: {
      type: 'varchar(50)',
    },
    coupon_id: {
      type: 'uuid',
    },
  }, {
    ifNotExists: true,  // ถ้ามี column อยู่แล้ว ไม่ error
  });
};

exports.down = (pgm) => {
  pgm.dropColumns('orders', ['discount_code', 'coupon_id']);
};
```

```javascript
// migrations/1699000007000_create-reviews-table.js
exports.up = (pgm) => {
  // ใช้ SQL โดยตรงสำหรับ complex operations
  pgm.sql(`
    CREATE TABLE IF NOT EXISTS reviews (
      id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
      user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
      rating INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
      title VARCHAR(255),
      content TEXT,
      verified_purchase BOOLEAN DEFAULT FALSE,
      helpful_count INTEGER DEFAULT 0,
      status VARCHAR(20) NOT NULL DEFAULT 'published',
      created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
      updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
      CONSTRAINT reviews_unique_user_product UNIQUE(user_id, product_id)
    );

    CREATE INDEX IF NOT EXISTS reviews_product_id_idx ON reviews(product_id);
    CREATE INDEX IF NOT EXISTS reviews_user_id_idx ON reviews(user_id);
    CREATE INDEX IF NOT EXISTS reviews_rating_idx ON reviews(rating);
    CREATE INDEX IF NOT EXISTS reviews_status_idx ON reviews(status);
  `);
};

exports.down = (pgm) => {
  pgm.dropTable('reviews');
};
```

---

## Prisma Migrations

### Installation

```bash
npm install @prisma/client
npm install --save-dev prisma
```

### schema.prisma

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
  // shadowDatabaseUrl = env("SHADOW_DATABASE_URL")  // สำหรับ environments ที่ไม่ใช่ local
}

model User {
  id            String    @id @default(uuid()) @db.Uuid
  email         String    @unique @db.VarChar(255)
  name          String    @db.VarChar(255)
  passwordHash  String    @map("password_hash") @db.VarChar(255)
  role          String    @default("user") @db.VarChar(50)
  emailVerified Boolean   @default(false) @map("email_verified")
  createdAt     DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt     DateTime  @updatedAt @map("updated_at") @db.Timestamptz

  orders   Order[]
  reviews  Review[]
  cart     Cart?

  @@index([createdAt])
  @@map("users")
}

model Category {
  id          String    @id @default(uuid()) @db.Uuid
  name        String    @db.VarChar(255)
  slug        String    @unique @db.VarChar(255)
  description String?
  imageUrl    String?   @map("image_url") @db.VarChar(512)
  sortOrder   Int       @default(0) @map("sort_order")
  active      Boolean   @default(true)
  parentId    String?   @map("parent_id") @db.Uuid
  createdAt   DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt   DateTime  @updatedAt @map("updated_at") @db.Timestamptz

  parent    Category?  @relation("CategoryToCategory", fields: [parentId], references: [id])
  children  Category[] @relation("CategoryToCategory")
  products  Product[]

  @@map("categories")
}

model Product {
  id            String        @id @default(uuid()) @db.Uuid
  sku           String        @unique @db.VarChar(100)
  name          String        @db.VarChar(500)
  slug          String        @unique @db.VarChar(500)
  description   String?
  price         Decimal       @db.Decimal(12, 2)
  comparePrice  Decimal?      @map("compare_price") @db.Decimal(12, 2)
  costPrice     Decimal?      @map("cost_price") @db.Decimal(12, 2)
  stockQuantity Int           @default(0) @map("stock_quantity")
  categoryId    String        @map("category_id") @db.Uuid
  status        ProductStatus @default(active)
  images        Json          @default("[]")
  attributes    Json          @default("{}")
  tags          String[]
  featured      Boolean       @default(false)
  createdAt     DateTime      @default(now()) @map("created_at") @db.Timestamptz
  updatedAt     DateTime      @updatedAt @map("updated_at") @db.Timestamptz

  category   Category    @relation(fields: [categoryId], references: [id])
  orderItems OrderItem[]
  reviews    Review[]
  cartItems  CartItem[]

  @@index([categoryId])
  @@index([status])
  @@index([featured])
  @@index([price])
  @@map("products")
}

enum ProductStatus {
  active
  inactive
  out_of_stock
  discontinued

  @@map("product_status")
}

model Order {
  id               String        @id @default(uuid()) @db.Uuid
  orderNumber      String        @unique @map("order_number") @db.VarChar(50)
  userId           String        @map("user_id") @db.Uuid
  status           OrderStatus   @default(pending)
  paymentStatus    PaymentStatus @default(pending) @map("payment_status")
  subtotal         Decimal       @db.Decimal(12, 2)
  discountAmount   Decimal       @default(0) @map("discount_amount") @db.Decimal(12, 2)
  shippingFee      Decimal       @default(0) @map("shipping_fee") @db.Decimal(12, 2)
  taxAmount        Decimal       @default(0) @map("tax_amount") @db.Decimal(12, 2)
  totalAmount      Decimal       @map("total_amount") @db.Decimal(12, 2)
  shippingAddress  Json          @map("shipping_address")
  billingAddress   Json?         @map("billing_address")
  paymentMethod    String?       @map("payment_method") @db.VarChar(50)
  notes            String?
  createdAt        DateTime      @default(now()) @map("created_at") @db.Timestamptz
  updatedAt        DateTime      @updatedAt @map("updated_at") @db.Timestamptz
  confirmedAt      DateTime?     @map("confirmed_at") @db.Timestamptz
  shippedAt        DateTime?     @map("shipped_at") @db.Timestamptz
  deliveredAt      DateTime?     @map("delivered_at") @db.Timestamptz

  user       User        @relation(fields: [userId], references: [id])
  items      OrderItem[]

  @@index([userId])
  @@index([status])
  @@index([createdAt])
  @@map("orders")
}

enum OrderStatus {
  pending
  confirmed
  processing
  shipped
  delivered
  cancelled
  refunded

  @@map("order_status")
}

enum PaymentStatus {
  pending
  paid
  failed
  refunded
  partially_refunded

  @@map("payment_status")
}

model OrderItem {
  id              String   @id @default(uuid()) @db.Uuid
  orderId         String   @map("order_id") @db.Uuid
  productId       String   @map("product_id") @db.Uuid
  sku             String   @db.VarChar(100)
  productName     String   @map("product_name") @db.VarChar(500)
  quantity        Int
  unitPrice       Decimal  @map("unit_price") @db.Decimal(12, 2)
  totalPrice      Decimal  @map("total_price") @db.Decimal(12, 2)
  productSnapshot Json     @default("{}") @map("product_snapshot")
  createdAt       DateTime @default(now()) @map("created_at") @db.Timestamptz

  order   Order   @relation(fields: [orderId], references: [id], onDelete: Cascade)
  product Product @relation(fields: [productId], references: [id])

  @@index([orderId])
  @@index([productId])
  @@map("order_items")
}

model Review {
  id               String   @id @default(uuid()) @db.Uuid
  productId        String   @map("product_id") @db.Uuid
  userId           String   @map("user_id") @db.Uuid
  rating           Int
  title            String?  @db.VarChar(255)
  content          String?
  verifiedPurchase Boolean  @default(false) @map("verified_purchase")
  helpfulCount     Int      @default(0) @map("helpful_count")
  status           String   @default("published") @db.VarChar(20)
  createdAt        DateTime @default(now()) @map("created_at") @db.Timestamptz
  updatedAt        DateTime @updatedAt @map("updated_at") @db.Timestamptz

  product Product @relation(fields: [productId], references: [id], onDelete: Cascade)
  user    User    @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([userId, productId])
  @@index([productId])
  @@index([rating])
  @@map("reviews")
}

model Cart {
  id        String    @id @default(uuid()) @db.Uuid
  userId    String?   @unique @map("user_id") @db.Uuid
  sessionId String?   @unique @map("session_id") @db.VarChar(255)
  expiresAt DateTime? @map("expires_at") @db.Timestamptz
  createdAt DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt DateTime  @updatedAt @map("updated_at") @db.Timestamptz

  user  User?      @relation(fields: [userId], references: [id], onDelete: Cascade)
  items CartItem[]

  @@map("carts")
}

model CartItem {
  id        String   @id @default(uuid()) @db.Uuid
  cartId    String   @map("cart_id") @db.Uuid
  productId String   @map("product_id") @db.Uuid
  quantity  Int
  addedAt   DateTime @default(now()) @map("added_at") @db.Timestamptz

  cart    Cart    @relation(fields: [cartId], references: [id], onDelete: Cascade)
  product Product @relation(fields: [productId], references: [id], onDelete: Cascade)

  @@unique([cartId, productId])
  @@index([cartId])
  @@map("cart_items")
}
```

### Prisma Migration Commands

```bash
# สร้าง migration และ apply ใน dev
npx prisma migrate dev --name create-initial-schema

# ดู status
npx prisma migrate status

# Deploy migrations ใน production (ไม่สร้าง migration ใหม่)
npx prisma migrate deploy

# Reset database (dev only - ลบทั้งหมดแล้วสร้างใหม่)
npx prisma migrate reset

# Generate Prisma Client หลัง schema เปลี่ยน
npx prisma generate

# ดู database ผ่าน UI
npx prisma studio
```

### Shadow Database

Prisma ใช้ shadow database เพื่อ:
1. Detect schema drift
2. Generate accurate migration diff
3. Validate migration SQL

```
DATABASE_URL="postgresql://user:password@localhost:5432/mydb"
SHADOW_DATABASE_URL="postgresql://user:password@localhost:5432/mydb_shadow"
```

### Baseline Existing Database

เมื่อ project มี database อยู่แล้วและต้องการเริ่มใช้ Prisma Migrate:

```bash
# 1. สร้าง migration folder โดยไม่รัน SQL
npx prisma migrate diff \
  --from-empty \
  --to-schema-datamodel prisma/schema.prisma \
  --script > prisma/migrations/0001_initial/migration.sql

# 2. Mark migration นี้ว่าได้ apply แล้ว (baseline)
npx prisma migrate resolve --applied "0001_initial"

# 3. ตรวจสอบ
npx prisma migrate status
```

---

## Zero-Downtime Migration Techniques

### Expand-Contract Pattern

วิธีที่ปลอดภัยที่สุดสำหรับ production migrations ที่ไม่ต้อง downtime

**ตัวอย่าง: เปลี่ยน column name จาก `full_name` เป็น `first_name` + `last_name`**

**Phase 1: Expand (เพิ่ม columns ใหม่)**

```sql
-- Migration V001: Add new columns (backward compatible)
ALTER TABLE users ADD COLUMN IF NOT EXISTS first_name VARCHAR(150);
ALTER TABLE users ADD COLUMN IF NOT EXISTS last_name VARCHAR(150);
```

```javascript
// Application code: เขียนทั้ง old และ new columns
async function updateUserName(userId, firstName, lastName) {
  await db.query(`
    UPDATE users 
    SET full_name = $1,
        first_name = $2,
        last_name = $3
    WHERE id = $4
  `, [`${firstName} ${lastName}`, firstName, lastName, userId]);
}
```

**Phase 2: Migrate data**

```sql
-- Migration V002: Backfill data
UPDATE users
SET
  first_name = split_part(full_name, ' ', 1),
  last_name = CASE
    WHEN strpos(full_name, ' ') > 0
    THEN substring(full_name FROM strpos(full_name, ' ') + 1)
    ELSE ''
  END
WHERE first_name IS NULL OR last_name IS NULL;

-- ตรวจสอบว่า migrate ครบ
SELECT COUNT(*) FROM users WHERE first_name IS NULL OR last_name IS NULL;
```

**Phase 3: Add constraints (เมื่อ data migrate ครบ)**

```sql
-- Migration V003: Make new columns NOT NULL
ALTER TABLE users ALTER COLUMN first_name SET NOT NULL;
ALTER TABLE users ALTER COLUMN last_name SET NOT NULL;
```

**Phase 4: Update application ให้ใช้ columns ใหม่เท่านั้น**

```javascript
// Application code: ใช้แค่ new columns แล้ว
async function updateUserName(userId, firstName, lastName) {
  await db.query(`
    UPDATE users 
    SET first_name = $1,
        last_name = $2,
        full_name = $3  -- ยังเขียน old column ไว้ระหว่างนี้
    WHERE id = $4
  `, [firstName, lastName, `${firstName} ${lastName}`, userId]);
}
```

**Phase 5: Contract (ลบ column เก่า)**

```sql
-- Migration V004: Drop old column (หลัง deploy แล้ว stable)
ALTER TABLE users DROP COLUMN IF EXISTS full_name;
```

### Zero-Downtime Add Column

```sql
-- ✅ Safe: เพิ่ม nullable column ก่อน
ALTER TABLE products ADD COLUMN IF NOT EXISTS barcode VARCHAR(50);

-- ✅ Safe: เพิ่ม column พร้อม default value
ALTER TABLE products ADD COLUMN IF NOT EXISTS view_count INTEGER DEFAULT 0;

-- ⚠️ Dangerous: เพิ่ม NOT NULL column โดยไม่มี default
-- จะ lock table ทั้ง table!
-- ALTER TABLE products ADD COLUMN barcode VARCHAR(50) NOT NULL;

-- ✅ Safe approach:
-- 1. เพิ่ม column แบบ nullable ก่อน
ALTER TABLE products ADD COLUMN IF NOT EXISTS barcode VARCHAR(50);

-- 2. Set default value
ALTER TABLE products ALTER COLUMN barcode SET DEFAULT '';

-- 3. Backfill data
UPDATE products SET barcode = '' WHERE barcode IS NULL;

-- 4. Add NOT NULL constraint
ALTER TABLE products ALTER COLUMN barcode SET NOT NULL;
```

### Adding Index Without Locking

```sql
-- ⚠️ Dangerous: locks table ระหว่าง create index
CREATE INDEX orders_user_id_idx ON orders(user_id);

-- ✅ Safe: CREATE INDEX CONCURRENTLY (ไม่ lock table)
CREATE INDEX CONCURRENTLY IF NOT EXISTS orders_user_id_idx ON orders(user_id);

-- ✅ Safe: DROP INDEX CONCURRENTLY
DROP INDEX CONCURRENTLY IF EXISTS orders_old_idx;
```

### Large Table Data Backfill

```sql
-- ⚠️ Dangerous: UPDATE ทั้ง table ครั้งเดียว
UPDATE orders SET status_new = status WHERE status_new IS NULL;

-- ✅ Safe: Batch update ทีละ 1000 rows
DO $$
DECLARE
  batch_size INT := 1000;
  offset_val INT := 0;
  updated_count INT;
BEGIN
  LOOP
    UPDATE orders
    SET status_new = status
    WHERE id IN (
      SELECT id FROM orders
      WHERE status_new IS NULL
      LIMIT batch_size
    );
    
    GET DIAGNOSTICS updated_count = ROW_COUNT;
    EXIT WHEN updated_count = 0;
    
    COMMIT;  -- Commit each batch
    PERFORM pg_sleep(0.01);  -- ให้เวลา database หายใจ
    
    RAISE NOTICE 'Updated % rows', updated_count;
  END LOOP;
END $$;
```

---

## Migration ใน CI/CD

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  migrate-and-deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run database migrations
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: npm run migrate:up
      
      - name: Run tests
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
        run: npm test
      
      - name: Deploy application
        run: |
          # deploy steps here
          echo "Deploying..."
```

### Docker Entrypoint

```dockerfile
# Dockerfile
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

```bash
#!/bin/sh
# docker-entrypoint.sh

set -e

echo "Running database migrations..."
npm run migrate:up

echo "Starting application..."
exec "$@"
```

---

## Migration Testing

```typescript
// tests/migrations.test.ts
import { Client } from 'pg';

describe('Database Migrations', () => {
  let client: Client;

  beforeAll(async () => {
    client = new Client({ connectionString: process.env.TEST_DATABASE_URL });
    await client.connect();
    
    // รัน migrations ใน test database
    await runMigrations();
  });

  afterAll(async () => {
    // Rollback ทุก migrations หลัง test
    await rollbackMigrations();
    await client.end();
  });

  it('should create users table with correct columns', async () => {
    const result = await client.query(`
      SELECT column_name, data_type, is_nullable
      FROM information_schema.columns
      WHERE table_name = 'users'
      ORDER BY ordinal_position
    `);

    expect(result.rows).toContainEqual(
      expect.objectContaining({ column_name: 'id', data_type: 'uuid' })
    );
    expect(result.rows).toContainEqual(
      expect.objectContaining({ column_name: 'email', is_nullable: 'NO' })
    );
  });

  it('should have correct indexes on users table', async () => {
    const result = await client.query(`
      SELECT indexname, indexdef
      FROM pg_indexes
      WHERE tablename = 'users'
    `);

    const indexNames = result.rows.map(r => r.indexname);
    expect(indexNames).toContain('users_pkey');
    expect(indexNames).toContain('users_email_key');
  });

  it('should enforce foreign key constraints', async () => {
    // Insert user
    await client.query(`
      INSERT INTO users (id, email, name, password_hash)
      VALUES ('test-id', 'test@test.com', 'Test', 'hash')
    `);

    // Order ที่ user ไม่มีอยู่ควร fail
    await expect(client.query(`
      INSERT INTO orders (user_id, order_number, subtotal, total_amount, shipping_address)
      VALUES ('non-existent-id', 'ORD-001', 100, 100, '{}')
    `)).rejects.toThrow(/foreign key constraint/);

    // Cleanup
    await client.query(`DELETE FROM users WHERE id = 'test-id'`);
  });

  it('should rollback migrations correctly', async () => {
    // ตรวจสอบว่า down migration ทำงาน
    const tablesBefore = await getTableList();
    
    await runMigration('down', 1);
    
    const tablesAfter = await getTableList();
    expect(tablesAfter.length).toBeLessThan(tablesBefore.length);
    
    // Run up อีกครั้ง
    await runMigration('up', 1);
  });
});

async function getTableList(): Promise<string[]> {
  const result = await client.query(`
    SELECT tablename FROM pg_tables WHERE schemaname = 'public'
  `);
  return result.rows.map(r => r.tablename);
}
```

---

## สรุป Migration Best Practices

1. **Always write idempotent migrations** — ใช้ `IF NOT EXISTS`, `IF EXISTS`
2. **Never modify existing migrations** — สร้าง migration ใหม่แทน
3. **Test up and down** — ตรวจสอบว่า rollback ทำงานได้
4. **Use transactions** — ทุก migration ควรอยู่ใน transaction
5. **Avoid long-running locks** — ใช้ `CONCURRENTLY` สำหรับ indexes
6. **Batch large updates** — ไม่ UPDATE ทั้ง table ครั้งเดียว
7. **Expand-Contract for renames** — ค่อยๆ migrate แบบ backward compatible
8. **Run migrations before deploy** — migration ก่อน app เสมอ
9. **Keep migrations small** — หนึ่ง migration = หนึ่ง logical change
10. **Document breaking changes** — comment ใน migration file
