# Part 74: GraphQL กับ Database Cluster

## บทนำ

**GraphQL** เป็น Query Language สำหรับ API ที่พัฒนาโดย Facebook ช่วยให้ Client ระบุได้อย่างชัดเจนว่าต้องการข้อมูลอะไร — ไม่มากเกิน (Over-fetching) และไม่น้อยเกิน (Under-fetching)

ในบทนี้เราจะเรียนรู้การ Setup GraphQL ที่ทำงานร่วมกับ Database Cluster อย่าง PostgreSQL Primary/Replica, Redis Cache, และ DataLoader เพื่อประสิทธิภาพสูงสุด

---

## 1. GraphQL vs REST: Tradeoffs

### 1.1 เปรียบเทียบ

| Feature | REST | GraphQL |
|---------|------|---------|
| Data Fetching | Fixed Endpoints | Flexible Queries |
| Over-fetching | ปัญหาบ่อย | ไม่มี |
| Under-fetching | N+1 หลาย Requests | Single Request |
| Versioning | /v1, /v2 | Schema Evolution |
| Caching | HTTP Cache (built-in) | Client-side (ซับซ้อน) |
| Learning Curve | ต่ำ | สูงกว่า |
| File Upload | ง่าย | ต้องการ Plugin |
| Real-time | WebSocket/SSE (Manual) | Subscriptions (Built-in) |

### 1.2 เมื่อไหร่ใช้ GraphQL

✅ **ใช้ GraphQL เมื่อ:**
- มี Mobile App ที่ Bandwidth จำกัด
- Frontend Team ต้องการ Flexibility สูง
- ข้อมูลมี Relationships ซับซ้อน
- ต้องการ Real-time Subscriptions

❌ **ไม่ควรใช้ GraphQL เมื่อ:**
- Simple CRUD API ที่ไม่ซับซ้อน
- File Upload เป็นหลัก
- ต้องการ HTTP Cache ง่ายๆ
- Team ยังไม่คุ้นเคย GraphQL

---

## 2. Project Setup

### 2.1 Dependencies

```bash
npm init -y
npm install @apollo/server graphql graphql-ws ws
npm install prisma @prisma/client
npm install dataloader
npm install ioredis
npm install graphql-depth-limit graphql-query-complexity
npm install @types/ws @types/node typescript ts-node
npm install -D @types/node nodemon

# เพิ่ม scripts ใน package.json:
# "dev": "nodemon src/index.ts",
# "generate": "prisma generate"
```

### 2.2 Prisma Schema

```prisma
// prisma/schema.prisma

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
  role      UserRole @default(USER)
  posts     Post[]
  comments  Comment[]
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  @@map("users")
}

model Post {
  id          String    @id @default(uuid())
  title       String
  content     String
  published   Boolean   @default(false)
  viewCount   Int       @default(0) @map("view_count")
  author      User      @relation(fields: [authorId], references: [id])
  authorId    String    @map("author_id")
  tags        Tag[]     @relation("PostToTag")
  comments    Comment[]
  createdAt   DateTime  @default(now()) @map("created_at")
  updatedAt   DateTime  @updatedAt @map("updated_at")

  @@map("posts")
}

model Comment {
  id        String   @id @default(uuid())
  content   String
  author    User     @relation(fields: [authorId], references: [id])
  authorId  String   @map("author_id")
  post      Post     @relation(fields: [postId], references: [id])
  postId    String   @map("post_id")
  createdAt DateTime @default(now()) @map("created_at")

  @@map("comments")
}

model Tag {
  id    String @id @default(uuid())
  name  String @unique
  posts Post[] @relation("PostToTag")

  @@map("tags")
}

enum UserRole {
  USER
  ADMIN
  MODERATOR
}
```

---

## 3. Database Connection: Primary + Replica

```typescript
// src/database/prisma.ts
import { PrismaClient } from '@prisma/client';

// Write to Primary
export const prismaWrite = new PrismaClient({
  datasources: {
    db: { url: process.env.DATABASE_PRIMARY_URL },
  },
  log: ['warn', 'error'],
});

// Read from Replica  
export const prismaRead = new PrismaClient({
  datasources: {
    db: { url: process.env.DATABASE_REPLICA_URL || process.env.DATABASE_PRIMARY_URL },
  },
  log: ['warn', 'error'],
});

// src/database/redis.ts
import Redis from 'ioredis';

export const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  lazyConnect: true,
  enableAutoPipelining: true, // Batch commands automatically
  maxRetriesPerRequest: 3,
  keyPrefix: 'graphql:',
});
```

---

## 4. GraphQL Schema Definition

```typescript
// src/schema/typeDefs.ts
import { gql } from 'graphql-tag';

export const typeDefs = gql`
  # Pagination (Relay Cursor Spec)
  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
  }

  # ─── User Types ──────────────────────────────────────
  type User {
    id: ID!
    email: String!
    name: String!
    role: UserRole!
    posts(first: Int, after: String): PostConnection!
    postsCount: Int!
    createdAt: String!
    updatedAt: String!
  }

  type UserEdge {
    node: User!
    cursor: String!
  }

  type UserConnection {
    edges: [UserEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }

  # ─── Post Types ──────────────────────────────────────
  type Post {
    id: ID!
    title: String!
    content: String!
    published: Boolean!
    viewCount: Int!
    author: User!
    tags: [Tag!]!
    comments(first: Int, after: String): CommentConnection!
    commentsCount: Int!
    createdAt: String!
    updatedAt: String!
  }

  type PostEdge {
    node: Post!
    cursor: String!
  }

  type PostConnection {
    edges: [PostEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }

  # ─── Comment Types ────────────────────────────────────
  type Comment {
    id: ID!
    content: String!
    author: User!
    post: Post!
    createdAt: String!
  }

  type CommentEdge {
    node: Comment!
    cursor: String!
  }

  type CommentConnection {
    edges: [CommentEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }

  # ─── Tag Types ────────────────────────────────────────
  type Tag {
    id: ID!
    name: String!
    postsCount: Int!
  }

  # ─── Enums ────────────────────────────────────────────
  enum UserRole {
    USER
    ADMIN
    MODERATOR
  }

  enum PostOrderBy {
    CREATED_AT_ASC
    CREATED_AT_DESC
    VIEW_COUNT_DESC
    TITLE_ASC
  }

  # ─── Input Types ──────────────────────────────────────
  input CreateUserInput {
    email: String!
    name: String!
    password: String!
  }

  input UpdateUserInput {
    name: String
    email: String
  }

  input CreatePostInput {
    title: String!
    content: String!
    published: Boolean
    tagIds: [ID!]
  }

  input UpdatePostInput {
    title: String
    content: String
    published: Boolean
    tagIds: [ID!]
  }

  input CreateCommentInput {
    content: String!
    postId: ID!
  }

  # ─── Query ────────────────────────────────────────────
  type Query {
    # Users
    user(id: ID!): User
    users(
      first: Int = 20
      after: String
      search: String
      role: UserRole
    ): UserConnection!

    # Posts
    post(id: ID!): Post
    posts(
      first: Int = 20
      after: String
      search: String
      published: Boolean
      authorId: ID
      tagIds: [ID!]
      orderBy: PostOrderBy = CREATED_AT_DESC
    ): PostConnection!

    # Tags
    tags: [Tag!]!

    # Stats
    stats: Stats!
  }

  type Stats {
    totalUsers: Int!
    totalPosts: Int!
    totalComments: Int!
    publishedPosts: Int!
  }

  # ─── Mutation ─────────────────────────────────────────
  type Mutation {
    # Users
    createUser(input: CreateUserInput!): User!
    updateUser(id: ID!, input: UpdateUserInput!): User!
    deleteUser(id: ID!): Boolean!

    # Posts
    createPost(input: CreatePostInput!): Post!
    updatePost(id: ID!, input: UpdatePostInput!): Post!
    publishPost(id: ID!): Post!
    deletePost(id: ID!): Boolean!

    # Comments
    createComment(input: CreateCommentInput!): Comment!
    deleteComment(id: ID!): Boolean!
  }

  # ─── Subscription ─────────────────────────────────────
  type Subscription {
    postCreated: Post!
    postUpdated(id: ID): Post!
    commentCreated(postId: ID!): Comment!
    userJoined: User!
  }
`;
```

---

## 5. DataLoader: แก้ปัญหา N+1

### 5.1 ปัญหา N+1

```typescript
// ❌ N+1 Problem: Query users → for each user query author
const posts = await prismaRead.post.findMany({ take: 100 });
for (const post of posts) {
  // 100 queries for 100 posts!
  const author = await prismaRead.user.findUnique({ where: { id: post.authorId } });
}
```

### 5.2 DataLoader Solution

```typescript
// src/dataloaders/index.ts
import DataLoader from 'dataloader';
import { PrismaClient } from '@prisma/client';

export function createDataLoaders(prisma: PrismaClient) {
  // User DataLoader: Batch load users by IDs
  const userLoader = new DataLoader<string, any>(
    async (ids: readonly string[]) => {
      const users = await prisma.user.findMany({
        where: { id: { in: [...ids] } },
      });

      // IMPORTANT: Must return in same order as ids
      const userMap = new Map(users.map((u) => [u.id, u]));
      return ids.map((id) => userMap.get(id) || null);
    },
    {
      cache: true,       // Cache within same request
      maxBatchSize: 100, // Max IDs per batch query
    },
  );

  // Post DataLoader: Batch load posts by IDs
  const postLoader = new DataLoader<string, any>(
    async (ids: readonly string[]) => {
      const posts = await prisma.post.findMany({
        where: { id: { in: [...ids] } },
        include: { tags: true },
      });
      const postMap = new Map(posts.map((p) => [p.id, p]));
      return ids.map((id) => postMap.get(id) || null);
    },
  );

  // Posts by Author DataLoader
  const postsByAuthorLoader = new DataLoader<string, any[]>(
    async (authorIds: readonly string[]) => {
      const posts = await prisma.post.findMany({
        where: { authorId: { in: [...authorIds] } },
        orderBy: { createdAt: 'desc' },
      });

      const postsByAuthor = new Map<string, any[]>();
      authorIds.forEach((id) => postsByAuthor.set(id, []));
      posts.forEach((post) => {
        postsByAuthor.get(post.authorId)!.push(post);
      });
      return authorIds.map((id) => postsByAuthor.get(id) || []);
    },
    { cache: false }, // Don't cache list queries
  );

  // Comments by Post DataLoader
  const commentsByPostLoader = new DataLoader<string, any[]>(
    async (postIds: readonly string[]) => {
      const comments = await prisma.comment.findMany({
        where: { postId: { in: [...postIds] } },
        orderBy: { createdAt: 'asc' },
      });

      const commentsByPost = new Map<string, any[]>();
      postIds.forEach((id) => commentsByPost.set(id, []));
      comments.forEach((comment) => {
        commentsByPost.get(comment.postId)!.push(comment);
      });
      return postIds.map((id) => commentsByPost.get(id) || []);
    },
    { cache: false },
  );

  // Tags by Post DataLoader
  const tagsByPostLoader = new DataLoader<string, any[]>(
    async (postIds: readonly string[]) => {
      const postsWithTags = await prisma.post.findMany({
        where: { id: { in: [...postIds] } },
        include: { tags: true },
      });

      const tagsByPost = new Map(postsWithTags.map((p) => [p.id, p.tags]));
      return postIds.map((id) => tagsByPost.get(id) || []);
    },
    { cache: false },
  );

  // Counts DataLoader (batched COUNT queries)
  const postsCountByAuthorLoader = new DataLoader<string, number>(
    async (authorIds: readonly string[]) => {
      const counts = await prisma.$queryRaw<Array<{ author_id: string; count: bigint }>>`
        SELECT author_id, COUNT(*) as count
        FROM posts
        WHERE author_id = ANY(${[...authorIds]}::uuid[])
        GROUP BY author_id
      `;
      const countMap = new Map(counts.map((c) => [c.author_id, Number(c.count)]));
      return authorIds.map((id) => countMap.get(id) || 0);
    },
  );

  return {
    userLoader,
    postLoader,
    postsByAuthorLoader,
    commentsByPostLoader,
    tagsByPostLoader,
    postsCountByAuthorLoader,
  };
}

export type DataLoaders = ReturnType<typeof createDataLoaders>;
```

---

## 6. Context Setup

```typescript
// src/context.ts
import { PrismaClient } from '@prisma/client';
import Redis from 'ioredis';
import { createDataLoaders, DataLoaders } from './dataloaders';

export interface GraphQLContext {
  prismaRead: PrismaClient;
  prismaWrite: PrismaClient;
  redis: Redis;
  dataloaders: DataLoaders;
  userId?: string;
  userRole?: string;
}

export async function createContext({
  req,
  prismaRead,
  prismaWrite,
  redis,
}: {
  req: any;
  prismaRead: PrismaClient;
  prismaWrite: PrismaClient;
  redis: Redis;
}): Promise<GraphQLContext> {
  // Create new DataLoader instances per request (important for request-scoped cache!)
  const dataloaders = createDataLoaders(prismaRead);

  // Authenticate user from JWT token
  const token = req?.headers?.authorization?.replace('Bearer ', '');
  let userId: string | undefined;
  let userRole: string | undefined;

  if (token) {
    try {
      const payload = verifyJWT(token);
      userId = payload.userId;
      userRole = payload.role;
    } catch {
      // Invalid token - unauthenticated
    }
  }

  return {
    prismaRead,
    prismaWrite,
    redis,
    dataloaders,
    userId,
    userRole,
  };
}
```

---

## 7. Resolvers

### 7.1 Query Resolvers

```typescript
// src/resolvers/query.resolvers.ts
import { GraphQLContext } from '../context';
import { encodeCursor, decodeCursor } from '../utils/cursor';

export const queryResolvers = {
  // ─── User Queries ──────────────────────────────────
  user: async (_: any, { id }: { id: string }, ctx: GraphQLContext) => {
    // Check Redis cache first
    const cacheKey = `user:${id}`;
    const cached = await ctx.redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    const user = await ctx.prismaRead.user.findUnique({ where: { id } });
    if (user) {
      await ctx.redis.setex(cacheKey, 300, JSON.stringify(user)); // Cache 5 min
    }
    return user;
  },

  users: async (_: any, args: any, ctx: GraphQLContext) => {
    const { first = 20, after, search, role } = args;
    const cursor = after ? decodeCursor(after) : undefined;

    const where: any = {};
    if (search) {
      where.OR = [
        { name: { contains: search, mode: 'insensitive' } },
        { email: { contains: search, mode: 'insensitive' } },
      ];
    }
    if (role) where.role = role;

    const [users, totalCount] = await Promise.all([
      ctx.prismaRead.user.findMany({
        where,
        take: first + 1, // +1 to check hasNextPage
        cursor: cursor ? { id: cursor } : undefined,
        skip: cursor ? 1 : 0,
        orderBy: { createdAt: 'desc' },
      }),
      ctx.prismaRead.user.count({ where }),
    ]);

    const hasNextPage = users.length > first;
    const edges = users.slice(0, first).map((user) => ({
      node: user,
      cursor: encodeCursor(user.id),
    }));

    return {
      edges,
      pageInfo: {
        hasNextPage,
        hasPreviousPage: !!after,
        startCursor: edges[0]?.cursor,
        endCursor: edges[edges.length - 1]?.cursor,
      },
      totalCount,
    };
  },

  // ─── Post Queries ──────────────────────────────────
  post: async (_: any, { id }: { id: string }, ctx: GraphQLContext) => {
    return ctx.dataloaders.postLoader.load(id);
  },

  posts: async (_: any, args: any, ctx: GraphQLContext) => {
    const { first = 20, after, search, published, authorId, tagIds, orderBy } = args;
    const cursor = after ? decodeCursor(after) : undefined;

    const where: any = {};
    if (search) {
      where.OR = [
        { title: { contains: search, mode: 'insensitive' } },
        { content: { contains: search, mode: 'insensitive' } },
      ];
    }
    if (published !== undefined) where.published = published;
    if (authorId) where.authorId = authorId;
    if (tagIds?.length) {
      where.tags = { some: { id: { in: tagIds } } };
    }

    const orderByMap: Record<string, any> = {
      CREATED_AT_ASC: { createdAt: 'asc' },
      CREATED_AT_DESC: { createdAt: 'desc' },
      VIEW_COUNT_DESC: { viewCount: 'desc' },
      TITLE_ASC: { title: 'asc' },
    };

    const [posts, totalCount] = await Promise.all([
      ctx.prismaRead.post.findMany({
        where,
        take: first + 1,
        cursor: cursor ? { id: cursor } : undefined,
        skip: cursor ? 1 : 0,
        orderBy: orderByMap[orderBy] || { createdAt: 'desc' },
      }),
      ctx.prismaRead.post.count({ where }),
    ]);

    const hasNextPage = posts.length > first;
    const edges = posts.slice(0, first).map((post) => ({
      node: post,
      cursor: encodeCursor(post.id),
    }));

    return {
      edges,
      pageInfo: {
        hasNextPage,
        hasPreviousPage: !!after,
        startCursor: edges[0]?.cursor,
        endCursor: edges[edges.length - 1]?.cursor,
      },
      totalCount,
    };
  },

  tags: async (_: any, __: any, ctx: GraphQLContext) => {
    const cacheKey = 'tags:all';
    const cached = await ctx.redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    const tags = await ctx.prismaRead.tag.findMany({
      orderBy: { name: 'asc' },
    });
    await ctx.redis.setex(cacheKey, 600, JSON.stringify(tags)); // Cache 10 min
    return tags;
  },

  stats: async (_: any, __: any, ctx: GraphQLContext) => {
    const cacheKey = 'stats:global';
    const cached = await ctx.redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    const [totalUsers, totalPosts, totalComments, publishedPosts] = await Promise.all([
      ctx.prismaRead.user.count(),
      ctx.prismaRead.post.count(),
      ctx.prismaRead.comment.count(),
      ctx.prismaRead.post.count({ where: { published: true } }),
    ]);

    const stats = { totalUsers, totalPosts, totalComments, publishedPosts };
    await ctx.redis.setex(cacheKey, 60, JSON.stringify(stats)); // Cache 1 min
    return stats;
  },
};
```

### 7.2 Mutation Resolvers

```typescript
// src/resolvers/mutation.resolvers.ts
import { GraphQLError } from 'graphql';
import { GraphQLContext } from '../context';
import { hashPassword } from '../utils/auth';
import { pubsub } from '../pubsub';

export const mutationResolvers = {
  createUser: async (_: any, { input }: any, ctx: GraphQLContext) => {
    const hashedPassword = await hashPassword(input.password);

    const user = await ctx.prismaWrite.user.create({
      data: {
        email: input.email,
        name: input.name,
        password: hashedPassword,
      },
    });

    // Publish subscription event
    pubsub.publish('USER_JOINED', { userJoined: user });

    // Invalidate cache
    await ctx.redis.del('stats:global');

    return user;
  },

  updateUser: async (_: any, { id, input }: any, ctx: GraphQLContext) => {
    // Authorization check
    if (ctx.userId !== id && ctx.userRole !== 'ADMIN') {
      throw new GraphQLError('Not authorized to update this user', {
        extensions: { code: 'FORBIDDEN' },
      });
    }

    const user = await ctx.prismaWrite.user.update({
      where: { id },
      data: input,
    });

    // Invalidate cache
    await ctx.redis.del(`user:${id}`);

    // Update DataLoader cache
    ctx.dataloaders.userLoader.clear(id);

    return user;
  },

  createPost: async (_: any, { input }: any, ctx: GraphQLContext) => {
    if (!ctx.userId) {
      throw new GraphQLError('Authentication required', {
        extensions: { code: 'UNAUTHENTICATED' },
      });
    }

    const post = await ctx.prismaWrite.post.create({
      data: {
        title: input.title,
        content: input.content,
        published: input.published ?? false,
        authorId: ctx.userId,
        tags: input.tagIds?.length
          ? { connect: input.tagIds.map((id: string) => ({ id })) }
          : undefined,
      },
      include: { tags: true },
    });

    // Publish subscription event
    pubsub.publish('POST_CREATED', { postCreated: post });

    // Invalidate caches
    await Promise.all([
      ctx.redis.del('stats:global'),
      ctx.redis.del(`user:${ctx.userId}`),
    ]);

    return post;
  },

  publishPost: async (_: any, { id }: any, ctx: GraphQLContext) => {
    const post = await ctx.prismaRead.post.findUnique({ where: { id } });

    if (!post) {
      throw new GraphQLError('Post not found', {
        extensions: { code: 'NOT_FOUND' },
      });
    }

    if (post.authorId !== ctx.userId && ctx.userRole !== 'ADMIN') {
      throw new GraphQLError('Not authorized', {
        extensions: { code: 'FORBIDDEN' },
      });
    }

    const updated = await ctx.prismaWrite.post.update({
      where: { id },
      data: { published: true },
    });

    pubsub.publish('POST_UPDATED', { postUpdated: updated });
    
    await ctx.redis.del(`post:${id}`);
    ctx.dataloaders.postLoader.clear(id);

    return updated;
  },
};
```

### 7.3 Type Resolvers (DataLoader ใช้งาน)

```typescript
// src/resolvers/type.resolvers.ts
import { GraphQLContext } from '../context';

export const typeResolvers = {
  User: {
    // DataLoader: Batch load posts for multiple users in one query
    posts: async (user: any, args: any, ctx: GraphQLContext) => {
      const posts = await ctx.dataloaders.postsByAuthorLoader.load(user.id);
      const { first = 20, after } = args;
      const edges = posts.slice(0, first).map((post: any) => ({
        node: post,
        cursor: Buffer.from(post.id).toString('base64'),
      }));
      return {
        edges,
        pageInfo: {
          hasNextPage: posts.length > first,
          hasPreviousPage: !!after,
          startCursor: edges[0]?.cursor,
          endCursor: edges[edges.length - 1]?.cursor,
        },
        totalCount: posts.length,
      };
    },

    // DataLoader: Batch count posts per user
    postsCount: async (user: any, _: any, ctx: GraphQLContext) => {
      return ctx.dataloaders.postsCountByAuthorLoader.load(user.id);
    },
  },

  Post: {
    // DataLoader: Batch load author for multiple posts in one query
    author: async (post: any, _: any, ctx: GraphQLContext) => {
      return ctx.dataloaders.userLoader.load(post.authorId);
    },

    // DataLoader: Batch load tags
    tags: async (post: any, _: any, ctx: GraphQLContext) => {
      return ctx.dataloaders.tagsByPostLoader.load(post.id);
    },

    // DataLoader: Batch load comments
    comments: async (post: any, args: any, ctx: GraphQLContext) => {
      const comments = await ctx.dataloaders.commentsByPostLoader.load(post.id);
      const { first = 20 } = args;
      const edges = comments.slice(0, first).map((comment: any) => ({
        node: comment,
        cursor: Buffer.from(comment.id).toString('base64'),
      }));
      return {
        edges,
        pageInfo: {
          hasNextPage: comments.length > first,
          hasPreviousPage: false,
          startCursor: edges[0]?.cursor,
          endCursor: edges[edges.length - 1]?.cursor,
        },
        totalCount: comments.length,
      };
    },

    commentsCount: async (post: any, _: any, ctx: GraphQLContext) => {
      const comments = await ctx.dataloaders.commentsByPostLoader.load(post.id);
      return comments.length;
    },
  },

  Comment: {
    author: async (comment: any, _: any, ctx: GraphQLContext) => {
      return ctx.dataloaders.userLoader.load(comment.authorId);
    },

    post: async (comment: any, _: any, ctx: GraphQLContext) => {
      return ctx.dataloaders.postLoader.load(comment.postId);
    },
  },
};
```

---

## 8. Subscriptions ด้วย Redis Pub/Sub

```typescript
// src/pubsub.ts
import { RedisPubSub } from 'graphql-redis-subscriptions';
import Redis from 'ioredis';

export const pubsub = new RedisPubSub({
  publisher: new Redis({
    host: process.env.REDIS_HOST,
    port: parseInt(process.env.REDIS_PORT || '6379'),
    retryStrategy: (times) => Math.min(times * 50, 2000),
  }),
  subscriber: new Redis({
    host: process.env.REDIS_HOST,
    port: parseInt(process.env.REDIS_PORT || '6379'),
    retryStrategy: (times) => Math.min(times * 50, 2000),
  }),
});

// src/resolvers/subscription.resolvers.ts
import { withFilter } from 'graphql-subscriptions';
import { pubsub } from '../pubsub';

export const subscriptionResolvers = {
  postCreated: {
    subscribe: () => pubsub.asyncIterator(['POST_CREATED']),
  },

  postUpdated: {
    subscribe: withFilter(
      () => pubsub.asyncIterator(['POST_UPDATED']),
      (payload, args) => {
        // Filter: ถ้า args.id ระบุ ให้ส่งเฉพาะ Post นั้น
        if (args.id) {
          return payload.postUpdated.id === args.id;
        }
        return true;
      },
    ),
  },

  commentCreated: {
    subscribe: withFilter(
      () => pubsub.asyncIterator(['COMMENT_CREATED']),
      (payload, args) => payload.commentCreated.postId === args.postId,
    ),
  },

  userJoined: {
    subscribe: () => pubsub.asyncIterator(['USER_JOINED']),
  },
};
```

---

## 9. Security: Query Protection

```typescript
// src/utils/query-protection.ts
import depthLimit from 'graphql-depth-limit';
import { createComplexityLimitRule } from 'graphql-query-complexity';

// ป้องกัน deeply nested queries
export const depthLimitRule = depthLimit(7);

// ป้องกัน expensive queries
export const complexityLimitRule = createComplexityLimitRule(1000, {
  scalarCost: 1,
  objectCost: 2,
  listFactor: 10,
  onCost: (cost) => {
    console.log(`Query complexity: ${cost}`);
  },
});
```

---

## 10. Apollo Server Setup (ครบชุด)

```typescript
// src/index.ts
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import { ApolloServerPluginDrainHttpServer } from '@apollo/server/plugin/drainHttpServer';
import { makeExecutableSchema } from '@graphql-tools/schema';
import express from 'express';
import http from 'http';
import cors from 'cors';
import { WebSocketServer } from 'ws';
import { useServer } from 'graphql-ws/lib/use/ws';
import { typeDefs } from './schema/typeDefs';
import { queryResolvers } from './resolvers/query.resolvers';
import { mutationResolvers } from './resolvers/mutation.resolvers';
import { subscriptionResolvers } from './resolvers/subscription.resolvers';
import { typeResolvers } from './resolvers/type.resolvers';
import { createContext } from './context';
import { prismaRead, prismaWrite } from './database/prisma';
import { redis } from './database/redis';
import { depthLimitRule, complexityLimitRule } from './utils/query-protection';

const resolvers = {
  Query: queryResolvers,
  Mutation: mutationResolvers,
  Subscription: subscriptionResolvers,
  ...typeResolvers,
};

const schema = makeExecutableSchema({ typeDefs, resolvers });

async function main() {
  const app = express();
  const httpServer = http.createServer(app);

  // WebSocket Server สำหรับ Subscriptions
  const wsServer = new WebSocketServer({
    server: httpServer,
    path: '/graphql/ws',
  });

  const serverCleanup = useServer(
    {
      schema,
      context: async (ctx) => createContext({
        req: ctx.connectionParams,
        prismaRead,
        prismaWrite,
        redis,
      }),
    },
    wsServer,
  );

  const server = new ApolloServer({
    schema,
    validationRules: [
      depthLimitRule,
      complexityLimitRule,
    ],
    plugins: [
      ApolloServerPluginDrainHttpServer({ httpServer }),
      {
        async serverWillStart() {
          return {
            async drainServer() {
              await serverCleanup.dispose();
            },
          };
        },
      },
    ],
    introspection: process.env.NODE_ENV !== 'production',
    formatError: (error) => {
      console.error(error);
      if (process.env.NODE_ENV === 'production') {
        return { message: error.message, code: error.extensions?.code };
      }
      return error;
    },
  });

  await server.start();

  app.use(
    '/graphql',
    cors<cors.CorsRequest>({
      origin: process.env.ALLOWED_ORIGINS?.split(',') || '*',
    }),
    express.json({ limit: '10mb' }),
    expressMiddleware(server, {
      context: async ({ req }) => createContext({ req, prismaRead, prismaWrite, redis }),
    }),
  );

  const PORT = parseInt(process.env.PORT || '4000');
  await new Promise<void>((resolve) => httpServer.listen({ port: PORT }, resolve));
  
  console.log(`🚀 GraphQL Server ready at http://localhost:${PORT}/graphql`);
  console.log(`📡 WebSocket ready at ws://localhost:${PORT}/graphql/ws`);
}

main().catch(console.error);
```

---

## 11. Hasura: Auto-generate GraphQL

### 11.1 Docker Setup

```yaml
# docker-compose.hasura.yml
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: hasura_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  hasura:
    image: hasura/graphql-engine:v2.36.0
    depends_on:
      - postgres
    environment:
      HASURA_GRAPHQL_DATABASE_URL: postgres://postgres:postgres123@postgres:5432/hasura_db
      HASURA_GRAPHQL_ENABLE_CONSOLE: "true"
      HASURA_GRAPHQL_ADMIN_SECRET: myadminsecret
      HASURA_GRAPHQL_JWT_SECRET: '{"type":"HS256","key":"super-secret-jwt-key-min-32-chars"}'
      HASURA_GRAPHQL_UNAUTHORIZED_ROLE: anonymous
      HASURA_GRAPHQL_LOG_LEVEL: warn
      HASURA_GRAPHQL_ENABLED_LOG_TYPES: startup, http-log, webhook-log
    ports:
      - "8080:8080"

volumes:
  postgres_data:
```

### 11.2 Hasura Metadata (hasura/metadata/tables.yaml)

```yaml
- table:
    name: users
    schema: public
  object_relationships:
  - name: posts
    using:
      foreign_key_constraint_on:
        column: author_id
        table:
          name: posts
          schema: public
  select_permissions:
  - role: anonymous
    permission:
      columns: [id, name, created_at]
      filter: {}
  - role: user
    permission:
      columns: [id, email, name, role, created_at, updated_at]
      filter:
        id:
          _eq: X-Hasura-User-Id
  - role: admin
    permission:
      columns: '*'
      filter: {}

- table:
    name: posts
    schema: public
  object_relationships:
  - name: author
    using:
      foreign_key_constraint_on: author_id
  array_relationships:
  - name: comments
    using:
      foreign_key_constraint_on:
        column: post_id
        table:
          name: comments
          schema: public
  insert_permissions:
  - role: user
    permission:
      columns: [title, content, published]
      check:
        author_id:
          _eq: X-Hasura-User-Id
      set:
        author_id: X-Hasura-User-Id
  update_permissions:
  - role: user
    permission:
      columns: [title, content, published]
      filter:
        author_id:
          _eq: X-Hasura-User-Id
  delete_permissions:
  - role: user
    permission:
      filter:
        author_id:
          _eq: X-Hasura-User-Id
```

---

## 12. Query Examples

```graphql
# Query: Get posts with author and tags (DataLoader batches efficiently)
query GetPosts($first: Int, $after: String, $search: String) {
  posts(first: $first, after: $after, search: $search, published: true) {
    edges {
      cursor
      node {
        id
        title
        viewCount
        createdAt
        author {
          id
          name
        }
        tags {
          id
          name
        }
        commentsCount
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}

# Mutation: Create post
mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) {
    id
    title
    published
    author {
      id
      name
    }
  }
}

# Subscription: Real-time comments
subscription OnCommentCreated($postId: ID!) {
  commentCreated(postId: $postId) {
    id
    content
    createdAt
    author {
      id
      name
    }
  }
}
```

---

## 13. Docker Compose ครบชุด

```yaml
# docker-compose.graphql.yml
version: '3.8'

services:
  postgres-primary:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: graphql_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
    volumes:
      - postgres_primary_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  postgres-replica:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: graphql_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
    depends_on:
      - postgres-primary
    ports:
      - "5433:5432"

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"

  graphql-api:
    build: .
    environment:
      DATABASE_PRIMARY_URL: postgresql://postgres:postgres123@postgres-primary:5432/graphql_db
      DATABASE_REPLICA_URL: postgresql://postgres:postgres123@postgres-replica:5432/graphql_db
      REDIS_HOST: redis
      REDIS_PORT: '6379'
      PORT: '4000'
      NODE_ENV: production
    depends_on:
      - postgres-primary
      - redis
    ports:
      - "4000:4000"

volumes:
  postgres_primary_data:
```

---

## 14. สรุป

### 14.1 Architecture Summary

```
Client (Browser/Mobile)
        │
        │ HTTP (Queries/Mutations)
        │ WebSocket (Subscriptions)
        ▼
Apollo Server v4
        │
        ├── DataLoader (Batch + Cache per Request)
        │         │
        │         ▼
        │   PostgreSQL Replica (Read)
        │
        ├── Redis (Response Caching)
        │
        ├── Mutation Resolvers
        │         │
        │         ▼
        │   PostgreSQL Primary (Write)
        │
        └── Subscription Resolvers
                  │
                  ▼
             Redis Pub/Sub
```

### 14.2 Performance Tips

1. **DataLoader per Request**: สร้าง DataLoader ใหม่ทุก Request เพื่อ Request-scoped Caching
2. **Read Replica**: Query Resolvers อ่านจาก Replica, Mutation เขียนที่ Primary
3. **Redis Cache**: Cache Response ที่ Expensive สำหรับ Query ที่ไม่ค่อยเปลี่ยน
4. **Depth Limiting**: ป้องกัน Malicious Query ที่ Deep Nested เกินไป
5. **Complexity Limiting**: ป้องกัน Query ที่ Logic ซับซ้อนเกินไป
6. **Persisted Queries**: ใน Production ใช้ Persisted Queries เพื่อ Performance และ Security
