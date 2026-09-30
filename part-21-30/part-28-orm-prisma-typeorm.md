# Part 28: ORM — Prisma หรือ TypeORM

## ORM คืออะไร?

ORM (Object-Relational Mapping) คือ layer ที่ map ระหว่าง:
- **Objects** ในโปรแกรม (JavaScript/TypeScript objects)
- **Tables** ใน relational database

แทนที่จะเขียน SQL โดยตรง เราใช้ ORM method แล้วมันแปลงเป็น SQL ให้

### ข้อดีของ ORM

1. **Type safety** — รู้ว่า query return อะไร ตั้งแต่ compile time
2. **Productivity** — เขียน code น้อยกว่า raw SQL
3. **Abstraction** — เปลี่ยน database engine ได้ง่าย (บางกรณี)
4. **Migrations** — บาง ORM มี migration tool ในตัว
5. **Relations** — จัดการ JOIN, eager/lazy loading อัตโนมัติ

### ข้อเสียของ ORM

1. **N+1 problem** — ถ้าไม่ระวัง จะดึงข้อมูลมากกว่าที่ต้องการ
2. **Performance** — SQL ที่ generate อาจไม่ optimal เสมอ
3. **Complex queries** — Query ซับซ้อนมากๆ อาจต้องเขียน raw SQL
4. **Learning curve** — ต้องเรียนรู้ ORM API แทน SQL
5. **Magic** — บางครั้งไม่รู้ว่า ORM ทำอะไรอยู่

---

## เปรียบเทียบ ORM ต่างๆ

| Feature | Prisma | TypeORM | Sequelize | Knex.js | Drizzle ORM |
|---------|--------|---------|-----------|---------|-------------|
| Type Safety | ดีมาก | ดี | ปานกลาง | ต่ำ | ดีมาก |
| Auto-complete | ดีมาก | ดี | ปานกลาง | ปานกลาง | ดีมาก |
| Migrations | ดีมาก | ดี | ดี | ดี | ดี |
| Performance | ดี | ดี | ดี | ดีมาก | ดีมาก |
| Learning curve | ต่ำ | สูง | ปานกลาง | ปานกลาง | ต่ำ |
| Raw SQL | รองรับ | รองรับ | รองรับ | First-class | รองรับ |
| Schema-first | ✓ | ✗ | ✗ | ✗ | ✓ |
| Code-first | ✗ | ✓ | ✓ | N/A | ✓ |
| Community | ใหญ่ | ใหญ่ | ใหญ่มาก | ใหญ่ | กำลังโต |

---

## Prisma

### Installation

```bash
# Install Prisma
npm install @prisma/client
npm install --save-dev prisma typescript @types/node ts-node

# Initialize Prisma
npx prisma init

# หลัง init จะได้:
# prisma/schema.prisma
# .env (ถ้ายังไม่มี)
```

### ตัวอย่าง schema.prisma

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String
  posts     Post[]
  profile   Profile?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Profile {
  id     String  @id @default(uuid())
  bio    String?
  userId String  @unique
  user   User    @relation(fields: [userId], references: [id])
}

model Post {
  id         String     @id @default(uuid())
  title      String
  content    String?
  published  Boolean    @default(false)
  authorId   String
  author     User       @relation(fields: [authorId], references: [id])
  categories Category[]
  createdAt  DateTime   @default(now())
  updatedAt  DateTime   @updatedAt
}

model Category {
  id    String @id @default(uuid())
  name  String @unique
  posts Post[]
}
```

### Generate Prisma Client

```bash
# Generate client จาก schema
npx prisma generate

# หรือ migrate + generate ไปพร้อมกัน
npx prisma migrate dev --name init
```

### CRUD Operations

```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient({
  log: ['query', 'info', 'warn', 'error'],
});

// ========== CREATE ==========

// สร้าง user
const newUser = await prisma.user.create({
  data: {
    email: 'alice@example.com',
    name: 'Alice',
  },
});

// สร้าง user พร้อม nested relations
const userWithProfile = await prisma.user.create({
  data: {
    email: 'bob@example.com',
    name: 'Bob',
    profile: {
      create: {
        bio: 'Software developer',
      },
    },
  },
  include: {
    profile: true,
  },
});

// สร้างหลาย records ครั้งเดียว
const { count } = await prisma.user.createMany({
  data: [
    { email: 'user1@test.com', name: 'User 1' },
    { email: 'user2@test.com', name: 'User 2' },
    { email: 'user3@test.com', name: 'User 3' },
  ],
  skipDuplicates: true,  // ข้าม duplicate ไม่ throw error
});

// ========== READ ==========

// ดึง user ทั้งหมด
const users = await prisma.user.findMany();

// ดึง user เดียวด้วย id
const user = await prisma.user.findUnique({
  where: { id: 'some-id' },
});

// ดึง user เดียวด้วย email
const userByEmail = await prisma.user.findUnique({
  where: { email: 'alice@example.com' },
});

// findFirst: ดึง record แรกที่ match
const firstPublishedPost = await prisma.post.findFirst({
  where: { published: true },
  orderBy: { createdAt: 'desc' },
});

// ดึงพร้อม relations
const userWithPosts = await prisma.user.findUnique({
  where: { id: 'some-id' },
  include: {
    posts: {
      where: { published: true },
      orderBy: { createdAt: 'desc' },
    },
    profile: true,
  },
});

// Select เฉพาะ fields ที่ต้องการ
const userNames = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    email: true,
  },
});

// ========== UPDATE ==========

// อัพเดต user
const updatedUser = await prisma.user.update({
  where: { id: 'some-id' },
  data: {
    name: 'Alice Updated',
  },
});

// Upsert: create ถ้าไม่มี, update ถ้ามี
const upsertedUser = await prisma.user.upsert({
  where: { email: 'alice@example.com' },
  update: {
    name: 'Alice Updated Again',
  },
  create: {
    email: 'alice@example.com',
    name: 'Alice',
  },
});

// อัพเดตหลาย records
const { count: updatedCount } = await prisma.post.updateMany({
  where: {
    authorId: 'some-user-id',
    published: false,
  },
  data: {
    published: true,
  },
});

// ========== DELETE ==========

// ลบ user
const deletedUser = await prisma.user.delete({
  where: { id: 'some-id' },
});

// ลบหลาย records
const { count: deletedCount } = await prisma.post.deleteMany({
  where: {
    authorId: 'some-id',
  },
});
```

### Filtering

```typescript
// WHERE conditions
const filteredUsers = await prisma.user.findMany({
  where: {
    // Equality
    name: 'Alice',
    
    // Not equal
    email: { not: 'alice@old.com' },
    
    // IN
    id: { in: ['id1', 'id2', 'id3'] },
    
    // NOT IN
    role: { notIn: ['banned', 'deleted'] },
    
    // Comparison
    age: { gte: 18, lte: 65 },
    
    // String operations
    email: {
      contains: '@gmail.com',
      // startsWith: 'alice',
      // endsWith: '.com',
    },
    
    // Case insensitive
    name: {
      contains: 'alice',
      mode: 'insensitive',
    },
    
    // NULL checks
    deletedAt: null,          // IS NULL
    // deletedAt: { not: null }  // IS NOT NULL
  },
});

// AND, OR, NOT
const complexFilter = await prisma.post.findMany({
  where: {
    AND: [
      { published: true },
      { authorId: 'some-id' },
    ],
  },
});

const orFilter = await prisma.post.findMany({
  where: {
    OR: [
      { title: { contains: 'TypeScript' } },
      { title: { contains: 'Prisma' } },
    ],
  },
});

const notFilter = await prisma.post.findMany({
  where: {
    NOT: {
      author: {
        email: { contains: '@spam.com' },
      },
    },
  },
});

// Relation filters
const usersWithPublishedPosts = await prisma.user.findMany({
  where: {
    posts: {
      some: {
        published: true,
      },
    },
  },
});

const usersWithNoPosts = await prisma.user.findMany({
  where: {
    posts: {
      none: {},
    },
  },
});

const usersWhereAllPostsPublished = await prisma.user.findMany({
  where: {
    posts: {
      every: {
        published: true,
      },
    },
  },
});
```

### Sorting

```typescript
// Single field sort
const sortedUsers = await prisma.user.findMany({
  orderBy: { createdAt: 'desc' },
});

// Multiple fields sort
const sortedPosts = await prisma.post.findMany({
  orderBy: [
    { published: 'desc' },
    { createdAt: 'desc' },
  ],
});

// Sort by relation
const postsByAuthorName = await prisma.post.findMany({
  orderBy: {
    author: {
      name: 'asc',
    },
  },
});
```

### Pagination

```typescript
// Offset-based pagination
const page = 2;
const limit = 10;

const paginatedPosts = await prisma.post.findMany({
  skip: (page - 1) * limit,
  take: limit,
  orderBy: { createdAt: 'desc' },
});

const total = await prisma.post.count();
const totalPages = Math.ceil(total / limit);

// Cursor-based pagination (ดีกว่าสำหรับ large datasets)
const firstPage = await prisma.post.findMany({
  take: 10,
  orderBy: { id: 'asc' },
});

const lastId = firstPage[firstPage.length - 1]?.id;

const secondPage = await prisma.post.findMany({
  take: 10,
  skip: 1,  // ข้าม cursor item
  cursor: { id: lastId },
  orderBy: { id: 'asc' },
});
```

### Relations

```typescript
// One-to-One
// สร้าง profile สำหรับ user
const profile = await prisma.profile.create({
  data: {
    bio: 'Full-stack developer',
    user: {
      connect: { id: 'user-id' },
    },
  },
});

// One-to-Many
// สร้าง post สำหรับ user
const post = await prisma.post.create({
  data: {
    title: 'My First Post',
    content: 'Hello World',
    author: {
      connect: { id: 'user-id' },
    },
  },
});

// Many-to-Many
// เพิ่ม categories ให้ post
const postWithCategories = await prisma.post.update({
  where: { id: 'post-id' },
  data: {
    categories: {
      connect: [
        { id: 'cat-1' },
        { id: 'cat-2' },
      ],
    },
  },
  include: { categories: true },
});

// Disconnect relations
const postWithoutCategory = await prisma.post.update({
  where: { id: 'post-id' },
  data: {
    categories: {
      disconnect: { id: 'cat-1' },
    },
  },
});

// Set (replace all)
const setCategories = await prisma.post.update({
  where: { id: 'post-id' },
  data: {
    categories: {
      set: [{ id: 'cat-3' }],  // replace ทั้งหมด
    },
  },
});
```

### Transactions

```typescript
// Sequential transactions
const [newUser, newPost] = await prisma.$transaction([
  prisma.user.create({
    data: { email: 'new@test.com', name: 'New User' },
  }),
  prisma.post.create({
    data: { title: 'First Post', authorId: 'existing-user-id' },
  }),
]);

// Interactive transactions (สำหรับ conditional logic)
const result = await prisma.$transaction(async (tx) => {
  // ดึง account ผู้ส่ง
  const sender = await tx.account.findUnique({
    where: { id: 'sender-id' },
  });

  if (!sender || sender.balance < 100) {
    throw new Error('Insufficient balance');
  }

  // หัก balance จากผู้ส่ง
  const updatedSender = await tx.account.update({
    where: { id: 'sender-id' },
    data: { balance: { decrement: 100 } },
  });

  // เพิ่ม balance ผู้รับ
  const updatedReceiver = await tx.account.update({
    where: { id: 'receiver-id' },
    data: { balance: { increment: 100 } },
  });

  // บันทึก transaction
  await tx.transfer.create({
    data: {
      senderId: 'sender-id',
      receiverId: 'receiver-id',
      amount: 100,
    },
  });

  return { sender: updatedSender, receiver: updatedReceiver };
});
```

### Raw Queries

```typescript
// $queryRaw: ดึงข้อมูล
const users = await prisma.$queryRaw<User[]>`
  SELECT * FROM users WHERE email LIKE ${'%@gmail.com'}
`;

// Safe parameterized query
const userId = 'some-id';
const user = await prisma.$queryRaw<User[]>`
  SELECT u.*, COUNT(p.id) as post_count
  FROM users u
  LEFT JOIN posts p ON u.id = p.author_id
  WHERE u.id = ${userId}
  GROUP BY u.id
`;

// $executeRaw: สำหรับ DML (INSERT, UPDATE, DELETE)
const result = await prisma.$executeRaw`
  UPDATE products 
  SET stock_quantity = stock_quantity - ${1}
  WHERE id = ${productId}
  AND stock_quantity > 0
`;

// Unsafe raw query (ระวัง SQL injection!)
const tableName = 'users';
const unsafeQuery = await prisma.$queryRawUnsafe(
  `SELECT * FROM ${tableName} LIMIT 10`
);
```

### Prisma Client Extensions

```typescript
// เพิ่ม method ใหม่ให้ Prisma Client
const extendedPrisma = prisma.$extends({
  model: {
    user: {
      async findByEmail(email: string) {
        return prisma.user.findUnique({
          where: { email },
        });
      },
      
      async softDelete(id: string) {
        return prisma.user.update({
          where: { id },
          data: { deletedAt: new Date() },
        });
      },
    },
  },
  
  query: {
    // Middleware สำหรับ soft delete
    user: {
      async findMany({ args, query }) {
        // เพิ่ม filter ไม่รวม soft-deleted records
        args.where = {
          ...args.where,
          deletedAt: null,
        };
        return query(args);
      },
    },
  },
  
  result: {
    user: {
      // Computed fields
      fullName: {
        needs: { firstName: true, lastName: true },
        compute(user) {
          return `${user.firstName} ${user.lastName}`;
        },
      },
    },
  },
});

// การใช้งาน
const user = await extendedPrisma.user.findByEmail('alice@example.com');
await extendedPrisma.user.softDelete('some-id');
```

### Middleware

```typescript
// Logging middleware
prisma.$use(async (params, next) => {
  const before = Date.now();
  const result = await next(params);
  const after = Date.now();
  
  console.log(`Query ${params.model}.${params.action} took ${after - before}ms`);
  return result;
});

// Soft delete middleware
prisma.$use(async (params, next) => {
  if (params.model === 'User') {
    if (params.action === 'delete') {
      // แปลง delete เป็น soft delete
      params.action = 'update';
      params.args['data'] = { deletedAt: new Date() };
    }
    
    if (params.action === 'deleteMany') {
      params.action = 'updateMany';
      if (params.args.data !== undefined) {
        params.args.data['deletedAt'] = new Date();
      } else {
        params.args['data'] = { deletedAt: new Date() };
      }
    }
    
    // กรอง soft-deleted records ออก
    if (params.action === 'findMany' || params.action === 'findFirst') {
      if (params.args.where) {
        if (params.args.where.deletedAt === undefined) {
          params.args.where['deletedAt'] = null;
        }
      } else {
        params.args['where'] = { deletedAt: null };
      }
    }
  }
  
  return next(params);
});
```

### Connection Pooling

```typescript
// ตั้งค่า connection pool
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL,
    },
  },
  // PgBouncer หรือ external pooler
});

// สำหรับ serverless ใช้ @prisma/adapter-pg ร่วมกับ pg-pool
import { PrismaPg } from '@prisma/adapter-pg';
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

const adapter = new PrismaPg(pool);
const prismaWithPool = new PrismaClient({ adapter });
```

---

## TypeORM

### Installation

```bash
npm install typeorm reflect-metadata pg
npm install --save-dev @types/node typescript

# tsconfig.json ต้องมี:
# "experimentalDecorators": true
# "emitDecoratorMetadata": true
```

### Entity Definitions

```typescript
// entities/User.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  OneToMany,
  OneToOne,
} from 'typeorm';
import { Post } from './Post';
import { Profile } from './Profile';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'varchar', length: 255, unique: true })
  email: string;

  @Column({ type: 'varchar', length: 255 })
  name: string;

  @Column({ name: 'password_hash', type: 'varchar', length: 255 })
  passwordHash: string;

  @Column({ type: 'varchar', length: 50, default: 'user' })
  role: string;

  @Column({ name: 'email_verified', default: false })
  emailVerified: boolean;

  @Column({ name: 'deleted_at', type: 'timestamptz', nullable: true })
  deletedAt?: Date;

  @CreateDateColumn({ name: 'created_at', type: 'timestamptz' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at', type: 'timestamptz' })
  updatedAt: Date;

  @OneToMany(() => Post, (post) => post.author)
  posts: Post[];

  @OneToOne(() => Profile, (profile) => profile.user)
  profile: Profile;
}
```

```typescript
// entities/Post.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  ManyToOne,
  ManyToMany,
  JoinTable,
  JoinColumn,
} from 'typeorm';
import { User } from './User';
import { Category } from './Category';

@Entity('posts')
export class Post {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'varchar', length: 500 })
  title: string;

  @Column({ type: 'text', nullable: true })
  content?: string;

  @Column({ default: false })
  published: boolean;

  @Column({ name: 'author_id', type: 'uuid' })
  authorId: string;

  @ManyToOne(() => User, (user) => user.posts, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'author_id' })
  author: User;

  @ManyToMany(() => Category, (category) => category.posts)
  @JoinTable({
    name: 'post_categories',
    joinColumn: { name: 'post_id' },
    inverseJoinColumn: { name: 'category_id' },
  })
  categories: Category[];

  @CreateDateColumn({ name: 'created_at', type: 'timestamptz' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at', type: 'timestamptz' })
  updatedAt: Date;
}
```

```typescript
// entities/Category.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  ManyToMany,
} from 'typeorm';
import { Post } from './Post';

@Entity('categories')
export class Category {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'varchar', length: 255, unique: true })
  name: string;

  @ManyToMany(() => Post, (post) => post.categories)
  posts: Post[];
}
```

### Repository Pattern

```typescript
import { AppDataSource } from './data-source';
import { User } from './entities/User';

// ใช้ default repository
const userRepository = AppDataSource.getRepository(User);

// CRUD
const user = await userRepository.findOneBy({ email: 'alice@example.com' });
const users = await userRepository.find({ where: { role: 'admin' } });
const newUser = userRepository.create({ email: 'new@test.com', name: 'New' });
const saved = await userRepository.save(newUser);
await userRepository.delete({ id: 'some-id' });

// Custom Repository
class UserRepository extends Repository<User> {
  async findByEmail(email: string): Promise<User | null> {
    return this.findOneBy({ email });
  }

  async findActiveAdmins(): Promise<User[]> {
    return this.createQueryBuilder('user')
      .where('user.role = :role', { role: 'admin' })
      .andWhere('user.deleted_at IS NULL')
      .orderBy('user.created_at', 'DESC')
      .getMany();
  }

  async softDelete(id: string): Promise<void> {
    await this.update(id, { deletedAt: new Date() });
  }
}
```

### QueryBuilder

```typescript
const userRepository = AppDataSource.getRepository(User);

// Basic QueryBuilder
const users = await userRepository
  .createQueryBuilder('user')
  .where('user.name = :name', { name: 'Alice' })
  .getMany();

// Complex query with JOIN
const postsWithAuthors = await AppDataSource
  .getRepository(Post)
  .createQueryBuilder('post')
  .leftJoinAndSelect('post.author', 'author')
  .leftJoinAndSelect('post.categories', 'category')
  .where('post.published = :published', { published: true })
  .andWhere('author.role = :role', { role: 'user' })
  .orderBy('post.createdAt', 'DESC')
  .skip(0)
  .take(10)
  .getManyAndCount();

// Raw query with QueryBuilder
const result = await AppDataSource
  .createQueryBuilder()
  .select('user.email', 'email')
  .addSelect('COUNT(post.id)', 'postCount')
  .from(User, 'user')
  .leftJoin('user.posts', 'post', 'post.published = :published', { published: true })
  .groupBy('user.email')
  .having('COUNT(post.id) > :minPosts', { minPosts: 5 })
  .getRawMany();
```

### Transactions ใน TypeORM

```typescript
import { AppDataSource } from './data-source';

// Method 1: queryRunner
const queryRunner = AppDataSource.createQueryRunner();
await queryRunner.connect();
await queryRunner.startTransaction();

try {
  const user = await queryRunner.manager.save(User, {
    email: 'new@test.com',
    name: 'New User',
  });
  
  await queryRunner.manager.save(Post, {
    title: 'First Post',
    authorId: user.id,
  });
  
  await queryRunner.commitTransaction();
} catch (error) {
  await queryRunner.rollbackTransaction();
  throw error;
} finally {
  await queryRunner.release();
}

// Method 2: transaction helper
await AppDataSource.transaction(async (manager) => {
  const user = await manager.save(User, { email: 'tx@test.com', name: 'TX User' });
  await manager.save(Post, { title: 'TX Post', authorId: user.id });
});
```

---

## Full Working Example: สร้าง API ด้วย Prisma

### Setup

```typescript
// src/lib/prisma.ts
import { PrismaClient } from '@prisma/client';

declare global {
  var prisma: PrismaClient | undefined;
}

export const prisma =
  global.prisma ||
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' 
      ? ['query', 'error', 'warn'] 
      : ['error'],
  });

if (process.env.NODE_ENV !== 'production') {
  global.prisma = prisma;
}
```

### Product Repository

```typescript
// src/repositories/product.repository.ts
import { prisma } from '../lib/prisma';
import { Prisma, Product, ProductStatus } from '@prisma/client';

interface FindProductsParams {
  categoryId?: string;
  status?: ProductStatus;
  search?: string;
  minPrice?: number;
  maxPrice?: number;
  featured?: boolean;
  page?: number;
  limit?: number;
  sortBy?: 'price' | 'createdAt' | 'name';
  sortOrder?: 'asc' | 'desc';
}

interface PaginatedProducts {
  products: Product[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}

export class ProductRepository {
  async findById(id: string): Promise<Product | null> {
    return prisma.product.findUnique({
      where: { id },
      include: {
        category: true,
        reviews: {
          where: { status: 'published' },
          take: 5,
          orderBy: { createdAt: 'desc' },
          include: { user: { select: { name: true } } },
        },
      },
    });
  }

  async findMany(params: FindProductsParams): Promise<PaginatedProducts> {
    const {
      categoryId,
      status = 'active',
      search,
      minPrice,
      maxPrice,
      featured,
      page = 1,
      limit = 20,
      sortBy = 'createdAt',
      sortOrder = 'desc',
    } = params;

    const where: Prisma.ProductWhereInput = {
      status,
      ...(categoryId && { categoryId }),
      ...(featured !== undefined && { featured }),
      ...(minPrice !== undefined || maxPrice !== undefined
        ? {
            price: {
              ...(minPrice !== undefined && { gte: minPrice }),
              ...(maxPrice !== undefined && { lte: maxPrice }),
            },
          }
        : {}),
      ...(search && {
        OR: [
          { name: { contains: search, mode: 'insensitive' } },
          { description: { contains: search, mode: 'insensitive' } },
          { sku: { contains: search, mode: 'insensitive' } },
        ],
      }),
    };

    const [products, total] = await Promise.all([
      prisma.product.findMany({
        where,
        include: {
          category: { select: { id: true, name: true, slug: true } },
        },
        orderBy: { [sortBy]: sortOrder },
        skip: (page - 1) * limit,
        take: limit,
      }),
      prisma.product.count({ where }),
    ]);

    return {
      products,
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
    };
  }

  async create(data: Prisma.ProductCreateInput): Promise<Product> {
    return prisma.product.create({
      data,
      include: { category: true },
    });
  }

  async update(id: string, data: Prisma.ProductUpdateInput): Promise<Product> {
    return prisma.product.update({
      where: { id },
      data,
      include: { category: true },
    });
  }

  async delete(id: string): Promise<void> {
    await prisma.product.update({
      where: { id },
      data: { status: 'inactive' },
    });
  }

  async adjustStock(
    productId: string,
    quantity: number
  ): Promise<Product> {
    return prisma.product.update({
      where: { id: productId },
      data: {
        stockQuantity: { increment: quantity },
      },
    });
  }

  async getStatsPerCategory(): Promise<any[]> {
    return prisma.$queryRaw`
      SELECT
        c.id as category_id,
        c.name as category_name,
        COUNT(p.id)::int as product_count,
        AVG(p.price)::numeric(10,2) as avg_price,
        MIN(p.price)::numeric(10,2) as min_price,
        MAX(p.price)::numeric(10,2) as max_price,
        SUM(p.stock_quantity)::int as total_stock
      FROM categories c
      LEFT JOIN products p ON c.id = p.category_id AND p.status = 'active'
      GROUP BY c.id, c.name
      ORDER BY product_count DESC
    `;
  }
}
```

### Order Service

```typescript
// src/services/order.service.ts
import { prisma } from '../lib/prisma';
import { Order, OrderStatus, Prisma } from '@prisma/client';

interface CreateOrderInput {
  userId: string;
  items: Array<{
    productId: string;
    quantity: number;
  }>;
  shippingAddress: {
    name: string;
    address: string;
    city: string;
    postalCode: string;
    country: string;
    phone: string;
  };
  paymentMethod: string;
  notes?: string;
}

export class OrderService {
  async createOrder(input: CreateOrderInput): Promise<Order> {
    return prisma.$transaction(async (tx) => {
      // 1. ดึงข้อมูล products และตรวจ stock
      const productIds = input.items.map(i => i.productId);
      const products = await tx.product.findMany({
        where: { id: { in: productIds }, status: 'active' },
      });

      if (products.length !== productIds.length) {
        throw new Error('One or more products not found or inactive');
      }

      // ตรวจสอบ stock
      const stockErrors: string[] = [];
      for (const item of input.items) {
        const product = products.find(p => p.id === item.productId)!;
        if (product.stockQuantity < item.quantity) {
          stockErrors.push(
            `${product.name}: available ${product.stockQuantity}, requested ${item.quantity}`
          );
        }
      }

      if (stockErrors.length > 0) {
        throw new Error(`Insufficient stock:\n${stockErrors.join('\n')}`);
      }

      // 2. คำนวณราคา
      const orderItems = input.items.map(item => {
        const product = products.find(p => p.id === item.productId)!;
        return {
          productId: product.id,
          sku: product.sku,
          productName: product.name,
          quantity: item.quantity,
          unitPrice: product.price,
          totalPrice: Number(product.price) * item.quantity,
          productSnapshot: {
            id: product.id,
            name: product.name,
            price: product.price,
            sku: product.sku,
          },
        };
      });

      const subtotal = orderItems.reduce((sum, item) => sum + item.totalPrice, 0);
      const shippingFee = subtotal >= 500 ? 0 : 50;  // Free shipping over 500 baht
      const taxAmount = subtotal * 0.07;  // 7% VAT
      const totalAmount = subtotal + shippingFee + taxAmount;

      // 3. สร้าง order number
      const orderNumber = `ORD-${Date.now()}-${Math.random().toString(36).substr(2, 5).toUpperCase()}`;

      // 4. สร้าง Order
      const order = await tx.order.create({
        data: {
          orderNumber,
          userId: input.userId,
          status: 'pending',
          paymentStatus: 'pending',
          subtotal,
          shippingFee,
          taxAmount,
          totalAmount,
          shippingAddress: input.shippingAddress,
          paymentMethod: input.paymentMethod,
          notes: input.notes,
          items: {
            create: orderItems,
          },
        },
        include: {
          items: true,
          user: { select: { email: true, name: true } },
        },
      });

      // 5. หัก stock
      await Promise.all(
        input.items.map(item =>
          tx.product.update({
            where: { id: item.productId },
            data: { stockQuantity: { decrement: item.quantity } },
          })
        )
      );

      // 6. ลบ cart items (ถ้ามี cart)
      await tx.cartItem.deleteMany({
        where: {
          cart: { userId: input.userId },
          productId: { in: productIds },
        },
      });

      return order;
    });
  }

  async updateOrderStatus(
    orderId: string,
    newStatus: OrderStatus
  ): Promise<Order> {
    const order = await prisma.order.findUnique({
      where: { id: orderId },
    });

    if (!order) throw new Error('Order not found');

    // Validate status transition
    const validTransitions: Record<OrderStatus, OrderStatus[]> = {
      pending: ['confirmed', 'cancelled'],
      confirmed: ['processing', 'cancelled'],
      processing: ['shipped', 'cancelled'],
      shipped: ['delivered'],
      delivered: ['refunded'],
      cancelled: [],
      refunded: [],
    };

    if (!validTransitions[order.status].includes(newStatus)) {
      throw new Error(
        `Cannot transition from ${order.status} to ${newStatus}`
      );
    }

    const timestampField: Record<string, string> = {
      confirmed: 'confirmedAt',
      shipped: 'shippedAt',
      delivered: 'deliveredAt',
      cancelled: 'cancelledAt',
    };

    const updateData: Prisma.OrderUpdateInput = {
      status: newStatus,
    };

    if (timestampField[newStatus]) {
      (updateData as any)[timestampField[newStatus]] = new Date();
    }

    // ถ้า cancel ต้อง คืน stock
    if (newStatus === 'cancelled') {
      const orderWithItems = await prisma.order.findUnique({
        where: { id: orderId },
        include: { items: true },
      });

      await prisma.$transaction(async (tx) => {
        const updated = await tx.order.update({
          where: { id: orderId },
          data: updateData,
        });

        // คืน stock
        await Promise.all(
          (orderWithItems?.items || []).map(item =>
            tx.product.update({
              where: { id: item.productId },
              data: { stockQuantity: { increment: item.quantity } },
            })
          )
        );

        return updated;
      });
    }

    return prisma.order.update({
      where: { id: orderId },
      data: updateData,
      include: { items: true, user: { select: { email: true, name: true } } },
    });
  }

  async getUserOrders(userId: string, page = 1, limit = 10): Promise<any> {
    const [orders, total] = await Promise.all([
      prisma.order.findMany({
        where: { userId },
        include: {
          items: {
            include: {
              product: { select: { id: true, name: true, slug: true } },
            },
          },
        },
        orderBy: { createdAt: 'desc' },
        skip: (page - 1) * limit,
        take: limit,
      }),
      prisma.order.count({ where: { userId } }),
    ]);

    return {
      orders,
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
    };
  }

  async getOrderStats(): Promise<any> {
    const stats = await prisma.$queryRaw`
      SELECT
        DATE_TRUNC('month', created_at) as month,
        COUNT(*)::int as order_count,
        SUM(total_amount)::numeric(12,2) as revenue,
        AVG(total_amount)::numeric(12,2) as avg_order_value,
        COUNT(CASE WHEN status = 'delivered' THEN 1 END)::int as delivered_count,
        COUNT(CASE WHEN status = 'cancelled' THEN 1 END)::int as cancelled_count
      FROM orders
      WHERE created_at >= NOW() - INTERVAL '12 months'
      GROUP BY DATE_TRUNC('month', created_at)
      ORDER BY month DESC
    `;

    return stats;
  }
}
```

### Express API Routes

```typescript
// src/routes/products.ts
import { Router, Request, Response } from 'express';
import { ProductRepository } from '../repositories/product.repository';

const router = Router();
const productRepo = new ProductRepository();

// GET /api/products
router.get('/', async (req: Request, res: Response) => {
  try {
    const {
      categoryId,
      status,
      search,
      minPrice,
      maxPrice,
      featured,
      page,
      limit,
      sortBy,
      sortOrder,
    } = req.query;

    const result = await productRepo.findMany({
      categoryId: categoryId as string,
      status: status as any,
      search: search as string,
      minPrice: minPrice ? parseFloat(minPrice as string) : undefined,
      maxPrice: maxPrice ? parseFloat(maxPrice as string) : undefined,
      featured: featured ? featured === 'true' : undefined,
      page: page ? parseInt(page as string) : 1,
      limit: limit ? parseInt(limit as string) : 20,
      sortBy: sortBy as any,
      sortOrder: sortOrder as any,
    });

    res.json(result);
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});

// GET /api/products/:id
router.get('/:id', async (req: Request, res: Response) => {
  try {
    const product = await productRepo.findById(req.params.id);
    
    if (!product) {
      return res.status(404).json({ error: 'Product not found' });
    }
    
    res.json(product);
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});

// POST /api/products
router.post('/', async (req: Request, res: Response) => {
  try {
    const product = await productRepo.create(req.body);
    res.status(201).json(product);
  } catch (error: any) {
    if (error.code === 'P2002') {
      return res.status(409).json({ error: 'SKU or slug already exists' });
    }
    res.status(500).json({ error: 'Internal server error' });
  }
});

// PATCH /api/products/:id
router.patch('/:id', async (req: Request, res: Response) => {
  try {
    const product = await productRepo.update(req.params.id, req.body);
    res.json(product);
  } catch (error: any) {
    if (error.code === 'P2025') {
      return res.status(404).json({ error: 'Product not found' });
    }
    res.status(500).json({ error: 'Internal server error' });
  }
});

export default router;
```

---

## สรุป: เลือก ORM ไหนดี?

### ใช้ Prisma เมื่อ:
- TypeScript project ที่ต้องการ type safety สูง
- ต้องการ auto-completion ที่ดีใน IDE
- เน้น productivity มากกว่า fine-grained SQL control
- Schema-first approach
- ต้องการ built-in migration tool

### ใช้ TypeORM เมื่อ:
- ชอบ Code-first (Decorator-based)
- ต้องการ Active Record pattern
- มีประสบการณ์กับ Java/Hibernate
- ต้องการ flexibility สูงกว่า Prisma

### ใช้ Knex.js เมื่อ:
- ต้องการ Query Builder ที่ใกล้เคียง SQL มาก
- Performance critical
- ต้องการ control เต็มที่

### ใช้ Drizzle ORM เมื่อ:
- TypeScript-first, type-safe
- Lightweight, bundle size เล็ก
- Serverless/Edge environments
- ต้องการ SQL-like syntax แต่มี type safety
