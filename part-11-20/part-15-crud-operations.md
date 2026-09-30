# Part 15: CRUD Operations ผ่าน API

## บทนำ

ในบทนี้เราจะเรียนรู้การออกแบบและ implement CRUD operations ที่สมบูรณ์ผ่าน REST API โดยใช้ Node.js + PostgreSQL + Redis ครอบคลุมทั้ง pagination, filtering, sorting, transactions และ HTTP caching

---

## 1. ออกแบบ RESTful Endpoints

### Naming Conventions
```
# Collection (plural noun)
GET    /api/v1/posts              # List all posts
POST   /api/v1/posts              # Create a post
GET    /api/v1/posts/:id          # Get single post
PUT    /api/v1/posts/:id          # Full update
PATCH  /api/v1/posts/:id          # Partial update
DELETE /api/v1/posts/:id          # Delete

# Nested resources
GET    /api/v1/posts/:id/comments        # Post's comments
POST   /api/v1/posts/:id/comments        # Add comment to post
DELETE /api/v1/posts/:id/comments/:cid   # Delete specific comment

# Actions (use verbs only when necessary)
POST   /api/v1/posts/:id/publish         # Publish post
POST   /api/v1/posts/:id/archive         # Archive post
POST   /api/v1/auth/login                # Login action
POST   /api/v1/auth/logout               # Logout action

# Bulk operations
POST   /api/v1/posts/bulk                # Bulk create
PATCH  /api/v1/posts/bulk                # Bulk update
DELETE /api/v1/posts/bulk                # Bulk delete
```

---

## 2. Database Schema

```sql
-- migrations/004_blog_schema.sql
CREATE TABLE categories (
  id          SERIAL PRIMARY KEY,
  name        VARCHAR(100) UNIQUE NOT NULL,
  slug        VARCHAR(100) UNIQUE NOT NULL,
  description TEXT,
  parent_id   INTEGER REFERENCES categories(id) ON DELETE SET NULL,
  post_count  INTEGER NOT NULL DEFAULT 0,
  created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE posts (
  id           SERIAL PRIMARY KEY,
  title        VARCHAR(200) NOT NULL,
  slug         VARCHAR(220) UNIQUE NOT NULL,
  content      TEXT NOT NULL,
  excerpt      VARCHAR(500),
  author_id    INTEGER NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
  category_id  INTEGER REFERENCES categories(id) ON DELETE SET NULL,
  status       VARCHAR(20) NOT NULL DEFAULT 'draft'
                 CHECK (status IN ('draft', 'published', 'archived')),
  tags         TEXT[] NOT NULL DEFAULT '{}',
  view_count   INTEGER NOT NULL DEFAULT 0,
  published_at TIMESTAMP WITH TIME ZONE,
  created_at   TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  deleted_at   TIMESTAMP WITH TIME ZONE
);

CREATE TABLE comments (
  id          SERIAL PRIMARY KEY,
  post_id     INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  author_id   INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  parent_id   INTEGER REFERENCES comments(id) ON DELETE CASCADE,
  content     TEXT NOT NULL CHECK (length(content) > 0 AND length(content) <= 5000),
  is_approved BOOLEAN NOT NULL DEFAULT false,
  created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_posts_author_id ON posts(author_id);
CREATE INDEX idx_posts_category_id ON posts(category_id);
CREATE INDEX idx_posts_status ON posts(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_posts_published_at ON posts(published_at DESC) WHERE status = 'published';
CREATE INDEX idx_posts_tags ON posts USING GIN(tags);
CREATE INDEX idx_posts_deleted_at ON posts(deleted_at) WHERE deleted_at IS NULL;

-- Full text search
CREATE INDEX idx_posts_fts ON posts USING GIN(
  to_tsvector('english', title || ' ' || coalesce(content, ''))
);

CREATE INDEX idx_comments_post_id ON comments(post_id);
CREATE INDEX idx_comments_author_id ON comments(author_id);
CREATE INDEX idx_comments_parent_id ON comments(parent_id);

-- Function to update updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER posts_updated_at
  BEFORE UPDATE ON posts
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER categories_updated_at
  BEFORE UPDATE ON categories
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER comments_updated_at
  BEFORE UPDATE ON comments
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

---

## 3. GET /posts - List with Pagination, Filtering, Sorting

```typescript
// src/controllers/post.controller.ts
import { Request, Response, NextFunction } from 'express';
import { z } from 'zod';
import { postService } from '../services/post.service';
import { ResponseUtils } from '../utils/response.utils';
import { AppError } from '../utils/errors';
import crypto from 'crypto';

// Query schema for listing
const listQuerySchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  status: z.enum(['draft', 'published', 'archived']).optional(),
  categoryId: z.coerce.number().int().positive().optional(),
  authorId: z.coerce.number().int().positive().optional(),
  search: z.string().trim().max(100).optional(),
  tags: z.string().optional(),
  sortBy: z.enum(['created_at', 'updated_at', 'view_count', 'title']).default('created_at'),
  sortOrder: z.enum(['ASC', 'DESC']).default('DESC'),
  fields: z.string().optional(), // field selection: ?fields=id,title,excerpt
});

export class PostController {
  // GET /posts
  async list(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const query = listQuerySchema.parse(req.query);

      // Parse tags (comma-separated)
      const tags = query.tags
        ? query.tags.split(',').map(t => t.trim()).filter(Boolean)
        : undefined;

      const result = await postService.listPosts({
        page: query.page,
        limit: query.limit,
        status: query.status,
        categoryId: query.categoryId,
        authorId: query.authorId,
        search: query.search,
        tags,
        sortBy: query.sortBy,
        sortOrder: query.sortOrder,
      });

      // HTTP Caching for public post lists
      if (!req.user) {
        const cacheKey = JSON.stringify(query);
        const etag = `"${crypto.createHash('md5').update(cacheKey + result.total).digest('hex')}"`;
        
        res.setHeader('Cache-Control', 'public, max-age=60, s-maxage=120');
        res.setHeader('ETag', etag);
        res.setHeader('Vary', 'Accept-Encoding');
        
        if (req.headers['if-none-match'] === etag) {
          res.status(304).send();
          return;
        }
      }

      ResponseUtils.paginated(res, result.posts, {
        page: result.page,
        limit: result.limit,
        total: result.total,
      });
    } catch (error) {
      next(error);
    }
  }

  // GET /posts/:id
  async getById(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const id = parseInt(req.params.id, 10);
      const post = await postService.getPost(id);

      if (!post) throw AppError.notFound('Post');

      // Track view count (async, don't await)
      postService.incrementViewCount(id).catch(console.error);

      // HTTP Caching
      if (post.updatedAt) {
        const etag = `"${crypto.createHash('md5').update(`${id}-${post.updatedAt.getTime()}`).digest('hex')}"`;
        
        res.setHeader('Last-Modified', post.updatedAt.toUTCString());
        res.setHeader('ETag', etag);
        
        if (post.status === 'published' && !req.user) {
          res.setHeader('Cache-Control', 'public, max-age=300');
        }
        
        // Check conditional request
        if (req.headers['if-none-match'] === etag) {
          res.status(304).send();
          return;
        }
        
        if (req.headers['if-modified-since']) {
          const ifModifiedSince = new Date(req.headers['if-modified-since']);
          if (post.updatedAt <= ifModifiedSince) {
            res.status(304).send();
            return;
          }
        }
      }

      ResponseUtils.success(res, post);
    } catch (error) {
      next(error);
    }
  }

  // POST /posts
  async create(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const authorId = req.user!.userId;
      const post = await postService.createPost({ ...req.body, authorId });
      ResponseUtils.created(res, post);
    } catch (error) {
      next(error);
    }
  }

  // PUT /posts/:id - Full update
  async update(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const id = parseInt(req.params.id, 10);
      const userId = req.user!.userId;
      
      const post = await postService.updatePost(id, req.body, userId);
      if (!post) throw AppError.notFound('Post');

      ResponseUtils.success(res, post);
    } catch (error) {
      next(error);
    }
  }

  // PATCH /posts/:id - Partial update
  async partialUpdate(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const id = parseInt(req.params.id, 10);
      const userId = req.user!.userId;
      
      // Only allow specific fields for PATCH
      const allowedFields = ['title', 'excerpt', 'tags', 'categoryId'];
      const updates = Object.fromEntries(
        Object.entries(req.body).filter(([key]) => allowedFields.includes(key))
      );
      
      if (Object.keys(updates).length === 0) {
        throw AppError.badRequest('No valid fields to update');
      }
      
      const post = await postService.updatePost(id, updates, userId);
      if (!post) throw AppError.notFound('Post');

      ResponseUtils.success(res, post);
    } catch (error) {
      next(error);
    }
  }

  // DELETE /posts/:id
  async delete(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const id = parseInt(req.params.id, 10);
      const userId = req.user!.userId;
      const userRole = req.user!.role;

      const deleted = await postService.deletePost(id, userId, userRole);
      if (!deleted) throw AppError.notFound('Post');

      ResponseUtils.noContent(res);
    } catch (error) {
      next(error);
    }
  }

  // POST /posts/:id/publish
  async publish(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const id = parseInt(req.params.id, 10);
      const post = await postService.publishPost(id, req.user!.userId);
      if (!post) throw AppError.notFound('Post');
      ResponseUtils.success(res, post);
    } catch (error) {
      next(error);
    }
  }

  // GET /posts/:id/comments
  async getComments(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const postId = parseInt(req.params.id, 10);
      const page = parseInt(req.query.page as string || '1', 10);
      const limit = parseInt(req.query.limit as string || '20', 10);

      const result = await postService.getPostComments(postId, page, limit);
      ResponseUtils.paginated(res, result.comments, {
        page,
        limit,
        total: result.total,
      });
    } catch (error) {
      next(error);
    }
  }

  // POST /posts/bulk
  async bulkCreate(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const { posts } = req.body;
      
      if (!Array.isArray(posts) || posts.length === 0) {
        throw AppError.badRequest('posts array is required');
      }
      
      if (posts.length > 100) {
        throw AppError.badRequest('Cannot create more than 100 posts at once');
      }
      
      const results = await postService.bulkCreatePosts(
        posts,
        req.user!.userId
      );
      
      res.status(207).json({
        success: true,
        data: {
          created: results.created,
          failed: results.failed,
          total: posts.length,
        },
      });
    } catch (error) {
      next(error);
    }
  }
}

export const postController = new PostController();
```

---

## 4. Service Layer

```typescript
// src/services/post.service.ts
import { postRepository, PostListOptions, PaginatedPosts } from '../repositories/post.repository';
import { commentRepository } from '../repositories/comment.repository';
import { postCache } from '../utils/redis-cache';
import { withTransaction } from '../db/transactions';
import { Post, PostWithRelations } from '../models/post.model';
import { AppError } from '../utils/errors';
import { PoolClient } from 'pg';

const POST_CACHE_TTL = 300; // 5 minutes
const LIST_CACHE_TTL = 60;  // 1 minute

export class PostService {
  async listPosts(options: PostListOptions): Promise<PaginatedPosts> {
    // Cache key based on query params
    const cacheKey = `list:${JSON.stringify(options)}`;
    
    // Only cache public post lists
    if (options.status === 'published' || !options.status) {
      const cached = await postCache.get<PaginatedPosts>(cacheKey);
      if (cached) return cached;
    }

    const result = await postRepository.list(options);

    if (options.status === 'published' || !options.status) {
      await postCache.set(cacheKey, result, LIST_CACHE_TTL);
    }

    return result;
  }

  async getPost(id: number): Promise<PostWithRelations | null> {
    const cacheKey = `id:${id}`;
    
    const cached = await postCache.get<PostWithRelations>(cacheKey);
    if (cached) return cached;

    const post = await postRepository.findById(id);
    if (!post) return null;

    await postCache.set(cacheKey, post, POST_CACHE_TTL);
    return post;
  }

  async createPost(data: {
    title: string;
    content: string;
    excerpt?: string;
    categoryId?: number;
    status?: 'draft' | 'published';
    tags?: string[];
    authorId: number;
  }): Promise<Post> {
    // Use transaction for creating post and updating category count
    return withTransaction(async (client) => {
      // Create the post
      const result = await client.query<Post>(
        `INSERT INTO posts (title, slug, content, excerpt, author_id, category_id, status, tags, created_at, updated_at)
         VALUES ($1, $2, $3, $4, $5, $6, $7, $8, NOW(), NOW())
         RETURNING *`,
        [
          data.title,
          await this.generateUniqueSlug(data.title, client),
          data.content,
          data.excerpt || null,
          data.authorId,
          data.categoryId || null,
          data.status || 'draft',
          data.tags || [],
        ]
      );

      const post = result.rows[0];

      // Update category post count if category specified
      if (data.categoryId) {
        await client.query(
          'UPDATE categories SET post_count = post_count + 1 WHERE id = $1',
          [data.categoryId]
        );
      }

      // Invalidate list cache
      await postCache.deletePattern('list:*');

      return post;
    });
  }

  async updatePost(
    id: number,
    data: Partial<Post>,
    userId: number,
    userRole?: string
  ): Promise<PostWithRelations | null> {
    // Get existing post
    const existing = await postRepository.findById(id);
    if (!existing) return null;

    // Check ownership
    if (existing.authorId !== userId && !['admin', 'moderator'].includes(userRole || '')) {
      throw AppError.forbidden('You can only edit your own posts');
    }

    return withTransaction(async (client) => {
      const oldCategoryId = existing.categoryId;
      const newCategoryId = data.categoryId;

      // Update category counts if category changed
      if (newCategoryId !== undefined && newCategoryId !== oldCategoryId) {
        if (oldCategoryId) {
          await client.query(
            'UPDATE categories SET post_count = GREATEST(0, post_count - 1) WHERE id = $1',
            [oldCategoryId]
          );
        }
        if (newCategoryId) {
          await client.query(
            'UPDATE categories SET post_count = post_count + 1 WHERE id = $1',
            [newCategoryId]
          );
        }
      }

      // Update post
      const post = await postRepository.update(id, data);

      // Invalidate cache
      await postCache.invalidateMany([
        `id:${id}`,
        `slug:${existing.slug}`,
      ]);
      await postCache.deletePattern('list:*');

      return post ? await postRepository.findById(id) : null;
    });
  }

  async deletePost(
    id: number,
    userId: number,
    userRole: string
  ): Promise<boolean> {
    const post = await postRepository.findById(id);
    if (!post) return false;

    // Check ownership
    if (post.authorId !== userId && !['admin', 'moderator'].includes(userRole)) {
      throw AppError.forbidden('You can only delete your own posts');
    }

    return withTransaction(async (client) => {
      // Soft delete
      await client.query(
        'UPDATE posts SET deleted_at = NOW() WHERE id = $1',
        [id]
      );

      // Update category count
      if (post.categoryId) {
        await client.query(
          'UPDATE categories SET post_count = GREATEST(0, post_count - 1) WHERE id = $1',
          [post.categoryId]
        );
      }

      // Invalidate cache
      await postCache.invalidateMany([
        `id:${id}`,
        `slug:${post.slug}`,
      ]);
      await postCache.deletePattern('list:*');

      return true;
    });
  }

  async publishPost(id: number, userId: number): Promise<Post | null> {
    const post = await postRepository.findById(id);
    if (!post) return null;

    if (post.status === 'published') {
      throw AppError.conflict('Post is already published');
    }

    const result = await postRepository.update(id, {
      status: 'published',
      publishedAt: new Date(),
    });

    if (result) {
      await postCache.invalidateMany([`id:${id}`, `slug:${post.slug}`]);
      await postCache.deletePattern('list:*');
    }

    return result;
  }

  async incrementViewCount(id: number): Promise<void> {
    // Use Redis to buffer view counts
    const bufferKey = `viewbuf:${id}`;
    const count = await (await import('../db/redis')).redis.incr(bufferKey);
    
    // Flush to DB every 10 views
    if (count % 10 === 0) {
      await postRepository.incrementViewCount(id);
      await (await import('../db/redis')).redis.set(bufferKey, '0');
    }
  }

  async getPostComments(
    postId: number,
    page: number,
    limit: number
  ): Promise<{ comments: any[]; total: number }> {
    return commentRepository.listByPost(postId, page, limit);
  }

  async bulkCreatePosts(
    postsData: any[],
    authorId: number
  ): Promise<{ created: Post[]; failed: Array<{ index: number; error: string }> }> {
    const created: Post[] = [];
    const failed: Array<{ index: number; error: string }> = [];

    for (let i = 0; i < postsData.length; i++) {
      try {
        const post = await this.createPost({ ...postsData[i], authorId });
        created.push(post);
      } catch (error) {
        failed.push({
          index: i,
          error: error instanceof Error ? error.message : 'Unknown error',
        });
      }
    }

    return { created, failed };
  }

  private async generateUniqueSlug(title: string, client?: PoolClient): Promise<string> {
    const baseSlug = title
      .toLowerCase()
      .replace(/[^a-z0-9\s-]/g, '')
      .replace(/\s+/g, '-')
      .replace(/-+/g, '-')
      .trim()
      .slice(0, 190);

    let slug = baseSlug;
    let counter = 0;

    const queryFn = client
      ? (q: string, p: unknown[]) => client.query(q, p)
      : (q: string, p: unknown[]) => (postRepository as any).db.query(q, p);

    while (true) {
      const result = await (client
        ? client.query('SELECT id FROM posts WHERE slug = $1', [slug])
        : (import('../db/pool').then(m => m.db.query('SELECT id FROM posts WHERE slug = $1', [slug])))
      );
      
      // If using client directly
      if (client) {
        const r = await client.query('SELECT id FROM posts WHERE slug = $1', [slug]);
        if (r.rowCount === 0) break;
      }
      
      counter++;
      slug = `${baseSlug}-${counter}`;
      
      if (counter > 100) {
        slug = `${baseSlug}-${Date.now()}`;
        break;
      }
    }

    return slug;
  }
}

export const postService = new PostService();
```

---

## 5. Comment Operations

```typescript
// src/repositories/comment.repository.ts
import { db } from '../db/pool';
import { withTransaction } from '../db/transactions';
import { Comment } from '../models/comment.model';

export interface CommentWithAuthor extends Comment {
  authorUsername: string;
  replies?: CommentWithAuthor[];
}

export class CommentRepository {
  async listByPost(
    postId: number,
    page: number,
    limit: number,
    parentId: number | null = null
  ): Promise<{ comments: CommentWithAuthor[]; total: number }> {
    const offset = (page - 1) * limit;

    const countResult = await db.query<{ count: string }>(
      `SELECT COUNT(*) as count
       FROM comments c
       WHERE c.post_id = $1
         AND c.parent_id IS NOT DISTINCT FROM $2
         AND c.is_approved = true`,
      [postId, parentId]
    );
    const total = parseInt(countResult.rows[0].count, 10);

    const result = await db.query<CommentWithAuthor>(
      `SELECT c.*, u.username as author_username
       FROM comments c
       JOIN users u ON c.author_id = u.id
       WHERE c.post_id = $1
         AND c.parent_id IS NOT DISTINCT FROM $2
         AND c.is_approved = true
       ORDER BY c.created_at ASC
       LIMIT $3 OFFSET $4`,
      [postId, parentId, limit, offset]
    );

    return { comments: result.rows, total };
  }

  async create(data: {
    postId: number;
    authorId: number;
    content: string;
    parentId?: number;
  }): Promise<Comment> {
    const result = await db.query<Comment>(
      `INSERT INTO comments (post_id, author_id, content, parent_id, is_approved, created_at, updated_at)
       VALUES ($1, $2, $3, $4, $5, NOW(), NOW())
       RETURNING *`,
      [
        data.postId,
        data.authorId,
        data.content,
        data.parentId || null,
        false, // Require approval by default
      ]
    );
    return result.rows[0];
  }

  async update(id: number, content: string, authorId: number): Promise<Comment | null> {
    const result = await db.query<Comment>(
      `UPDATE comments
       SET content = $2, updated_at = NOW()
       WHERE id = $1 AND author_id = $3
       RETURNING *`,
      [id, content, authorId]
    );
    return result.rows[0] || null;
  }

  async delete(id: number, authorId: number, isAdmin: boolean = false): Promise<boolean> {
    let query: string;
    let params: unknown[];

    if (isAdmin) {
      query = 'DELETE FROM comments WHERE id = $1 RETURNING id';
      params = [id];
    } else {
      query = 'DELETE FROM comments WHERE id = $1 AND author_id = $2 RETURNING id';
      params = [id, authorId];
    }

    const result = await db.query(query, params);
    return (result.rowCount || 0) > 0;
  }

  async approve(id: number): Promise<boolean> {
    const result = await db.query(
      'UPDATE comments SET is_approved = true, updated_at = NOW() WHERE id = $1 RETURNING id',
      [id]
    );
    return (result.rowCount || 0) > 0;
  }
}

export const commentRepository = new CommentRepository();
```

---

## 6. Pagination Utilities

```typescript
// src/utils/pagination.utils.ts
export interface PaginationParams {
  page: number;
  limit: number;
}

export interface PaginationMeta {
  page: number;
  limit: number;
  total: number;
  totalPages: number;
  hasNext: boolean;
  hasPrev: boolean;
  nextPage: number | null;
  prevPage: number | null;
}

export function buildPaginationMeta(
  page: number,
  limit: number,
  total: number
): PaginationMeta {
  const totalPages = Math.ceil(total / limit);
  
  return {
    page,
    limit,
    total,
    totalPages,
    hasNext: page < totalPages,
    hasPrev: page > 1,
    nextPage: page < totalPages ? page + 1 : null,
    prevPage: page > 1 ? page - 1 : null,
  };
}

export function buildPaginationLinks(
  baseUrl: string,
  page: number,
  limit: number,
  total: number
): Record<string, string | null> {
  const totalPages = Math.ceil(total / limit);
  
  const buildUrl = (p: number) => `${baseUrl}?page=${p}&limit=${limit}`;
  
  return {
    self: buildUrl(page),
    first: buildUrl(1),
    last: totalPages > 0 ? buildUrl(totalPages) : null,
    next: page < totalPages ? buildUrl(page + 1) : null,
    prev: page > 1 ? buildUrl(page - 1) : null,
  };
}

// Cursor-based pagination (more efficient for large datasets)
export interface CursorPaginationParams {
  cursor?: string;
  limit: number;
  direction?: 'next' | 'prev';
}

export interface CursorPaginationResult<T> {
  items: T[];
  nextCursor: string | null;
  prevCursor: string | null;
  hasNext: boolean;
  hasPrev: boolean;
}

export function encodeCursor(id: number, createdAt: Date): string {
  const data = JSON.stringify({ id, createdAt: createdAt.toISOString() });
  return Buffer.from(data).toString('base64url');
}

export function decodeCursor(cursor: string): { id: number; createdAt: Date } | null {
  try {
    const data = JSON.parse(Buffer.from(cursor, 'base64url').toString());
    return { id: data.id, createdAt: new Date(data.createdAt) };
  } catch {
    return null;
  }
}
```

---

## 7. HTTP Caching Headers

```typescript
// src/middlewares/cache.middleware.ts
import { Request, Response, NextFunction } from 'express';
import crypto from 'crypto';

export interface CacheOptions {
  maxAge?: number;           // Cache-Control max-age (seconds)
  sMaxAge?: number;          // s-maxage for CDN
  mustRevalidate?: boolean;
  noCache?: boolean;
  noStore?: boolean;
  public?: boolean;
  private?: boolean;
  staleWhileRevalidate?: number;
  staleIfError?: number;
}

export function cacheControl(options: CacheOptions = {}) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const directives: string[] = [];

    if (options.noStore) {
      directives.push('no-store');
    } else if (options.noCache) {
      directives.push('no-cache');
    } else {
      if (options.public) directives.push('public');
      if (options.private) directives.push('private');

      if (options.maxAge !== undefined) {
        directives.push(`max-age=${options.maxAge}`);
      }

      if (options.sMaxAge !== undefined) {
        directives.push(`s-maxage=${options.sMaxAge}`);
      }

      if (options.mustRevalidate) {
        directives.push('must-revalidate');
      }

      if (options.staleWhileRevalidate !== undefined) {
        directives.push(`stale-while-revalidate=${options.staleWhileRevalidate}`);
      }

      if (options.staleIfError !== undefined) {
        directives.push(`stale-if-error=${options.staleIfError}`);
      }
    }

    if (directives.length > 0) {
      res.setHeader('Cache-Control', directives.join(', '));
    }

    next();
  };
}

// ETag generator middleware
export function generateETag(data: unknown): string {
  const content = JSON.stringify(data);
  return `"${crypto.createHash('md5').update(content).digest('hex')}"`;
}

export function setETag(data: unknown, res: Response): void {
  const etag = generateETag(data);
  res.setHeader('ETag', etag);
}

export function checkETag(req: Request, etag: string): boolean {
  const ifNoneMatch = req.headers['if-none-match'];
  return ifNoneMatch === etag;
}

// Cache presets
export const cachePresets = {
  noCache: cacheControl({ noCache: true }),
  noStore: cacheControl({ noStore: true }),
  short: cacheControl({ public: true, maxAge: 60, sMaxAge: 120 }),    // 1/2 min
  medium: cacheControl({ public: true, maxAge: 300, sMaxAge: 600 }),  // 5/10 min
  long: cacheControl({ public: true, maxAge: 3600, sMaxAge: 7200 }), // 1/2 hr
  private: cacheControl({ private: true, maxAge: 300 }),
};
```

---

## 8. Soft Delete Implementation

```typescript
// src/repositories/base.repository.ts
import { db } from '../db/pool';
import { QueryResult } from 'pg';

export abstract class BaseRepository<T extends { id: number; deletedAt?: Date | null }> {
  protected abstract tableName: string;

  async findById(id: number, includeDeleted = false): Promise<T | null> {
    const where = includeDeleted
      ? 'id = $1'
      : 'id = $1 AND deleted_at IS NULL';

    const result = await db.query<T>(
      `SELECT * FROM ${this.tableName} WHERE ${where}`,
      [id]
    );
    return result.rows[0] || null;
  }

  async softDelete(id: number): Promise<boolean> {
    const result = await db.query(
      `UPDATE ${this.tableName}
       SET deleted_at = NOW(), updated_at = NOW()
       WHERE id = $1 AND deleted_at IS NULL
       RETURNING id`,
      [id]
    );
    return (result.rowCount || 0) > 0;
  }

  async restore(id: number): Promise<boolean> {
    const result = await db.query(
      `UPDATE ${this.tableName}
       SET deleted_at = NULL, updated_at = NOW()
       WHERE id = $1 AND deleted_at IS NOT NULL
       RETURNING id`,
      [id]
    );
    return (result.rowCount || 0) > 0;
  }

  async hardDelete(id: number): Promise<boolean> {
    const result = await db.query(
      `DELETE FROM ${this.tableName} WHERE id = $1 RETURNING id`,
      [id]
    );
    return (result.rowCount || 0) > 0;
  }

  async listDeleted(page: number, limit: number): Promise<{ items: T[]; total: number }> {
    const offset = (page - 1) * limit;

    const countResult = await db.query<{ count: string }>(
      `SELECT COUNT(*) as count FROM ${this.tableName} WHERE deleted_at IS NOT NULL`,
      []
    );
    const total = parseInt(countResult.rows[0].count, 10);

    const result = await db.query<T>(
      `SELECT * FROM ${this.tableName}
       WHERE deleted_at IS NOT NULL
       ORDER BY deleted_at DESC
       LIMIT $1 OFFSET $2`,
      [limit, offset]
    );

    return { items: result.rows, total };
  }

  // Auto-purge deleted records older than X days
  async purgeOldDeleted(daysOld: number): Promise<number> {
    const result = await db.query(
      `DELETE FROM ${this.tableName}
       WHERE deleted_at IS NOT NULL
         AND deleted_at < NOW() - INTERVAL '${daysOld} days'
       RETURNING id`
    );
    return result.rowCount || 0;
  }
}
```

---

## 9. Bulk Operations

```typescript
// src/services/bulk.service.ts
import { db } from '../db/pool';
import { withTransaction } from '../db/transactions';

export interface BulkResult<T> {
  created?: T[];
  updated?: T[];
  deleted?: number[];
  failed: Array<{ index: number; error: string; data?: unknown }>;
  totalProcessed: number;
  successful: number;
}

export class BulkService {
  // Bulk create with error handling per item
  async bulkCreate<T>(
    tableName: string,
    items: Record<string, unknown>[],
    columns: string[]
  ): Promise<BulkResult<T>> {
    const created: T[] = [];
    const failed: Array<{ index: number; error: string }> = [];

    // Process in chunks of 100
    const chunkSize = 100;

    for (let start = 0; start < items.length; start += chunkSize) {
      const chunk = items.slice(start, start + chunkSize);

      try {
        await withTransaction(async (client) => {
          for (let i = 0; i < chunk.length; i++) {
            try {
              const item = chunk[i];
              const values = columns.map(col => item[col]);
              const placeholders = columns.map((_, idx) => `$${idx + 1}`).join(', ');

              const result = await client.query<T>(
                `INSERT INTO ${tableName} (${columns.join(', ')}, created_at, updated_at)
                 VALUES (${placeholders}, NOW(), NOW())
                 RETURNING *`,
                values
              );
              created.push(result.rows[0]);
            } catch (error) {
              failed.push({
                index: start + i,
                error: error instanceof Error ? error.message : 'Unknown error',
              });
            }
          }
        });
      } catch (chunkError) {
        // Chunk transaction failed, mark all as failed
        chunk.forEach((_, i) => {
          failed.push({
            index: start + i,
            error: 'Chunk transaction failed',
          });
        });
      }
    }

    return {
      created,
      failed,
      totalProcessed: items.length,
      successful: created.length,
    };
  }

  // Bulk update (upsert)
  async bulkUpsert<T>(
    tableName: string,
    items: Array<{ id: number } & Record<string, unknown>>,
    columns: string[],
    conflictColumn: string = 'id'
  ): Promise<BulkResult<T>> {
    if (items.length === 0) {
      return { created: [], failed: [], totalProcessed: 0, successful: 0 };
    }

    const updated: T[] = [];
    const failed: Array<{ index: number; error: string }> = [];

    try {
      await withTransaction(async (client) => {
        for (let i = 0; i < items.length; i++) {
          try {
            const item = items[i];
            const updateCols = columns.filter(c => c !== conflictColumn);
            const setClauses = updateCols
              .map((col, idx) => `${col} = $${idx + 2}`)
              .join(', ');

            const allValues = [
              item[conflictColumn],
              ...updateCols.map(col => item[col]),
            ];

            const result = await client.query<T>(
              `UPDATE ${tableName}
               SET ${setClauses}, updated_at = NOW()
               WHERE ${conflictColumn} = $1
               RETURNING *`,
              allValues
            );

            if (result.rowCount === 0) {
              failed.push({ index: i, error: `${conflictColumn} ${item[conflictColumn]} not found` });
            } else {
              updated.push(result.rows[0]);
            }
          } catch (error) {
            failed.push({
              index: i,
              error: error instanceof Error ? error.message : 'Unknown error',
            });
          }
        }
      });
    } catch (txError) {
      return {
        failed: items.map((_, i) => ({ index: i, error: 'Transaction failed' })),
        totalProcessed: items.length,
        successful: 0,
      };
    }

    return {
      updated,
      failed,
      totalProcessed: items.length,
      successful: updated.length,
    };
  }

  // Bulk delete
  async bulkDelete(
    tableName: string,
    ids: number[],
    soft: boolean = true
  ): Promise<BulkResult<never>> {
    if (ids.length === 0) {
      return { failed: [], totalProcessed: 0, successful: 0, deleted: [] };
    }

    try {
      const query = soft
        ? `UPDATE ${tableName}
           SET deleted_at = NOW()
           WHERE id = ANY($1::int[]) AND deleted_at IS NULL
           RETURNING id`
        : `DELETE FROM ${tableName}
           WHERE id = ANY($1::int[])
           RETURNING id`;

      const result = await db.query<{ id: number }>(query, [ids]);
      const deletedIds = result.rows.map(r => r.id);

      const notFoundIds = ids.filter(id => !deletedIds.includes(id));
      const failed = notFoundIds.map(id => ({
        index: ids.indexOf(id),
        error: `ID ${id} not found or already deleted`,
      }));

      return {
        deleted: deletedIds,
        failed,
        totalProcessed: ids.length,
        successful: deletedIds.length,
      };
    } catch (error) {
      return {
        failed: ids.map((_, i) => ({
          index: i,
          error: error instanceof Error ? error.message : 'Delete failed',
        })),
        totalProcessed: ids.length,
        successful: 0,
      };
    }
  }
}

export const bulkService = new BulkService();
```

---

## 10. Complete Routes with All CRUD Operations

```typescript
// src/routes/post.routes.ts
import { Router } from 'express';
import { postController } from '../controllers/post.controller';
import { commentController } from '../controllers/comment.controller';
import {
  authenticate,
  authorize,
  optionalAuth
} from '../middlewares/auth.middleware';
import { validateBody, validateQuery, validateParams } from '../middlewares/validate.middleware';
import { idParamSchema } from '../middlewares/validate.middleware';
import { createPostSchema, updatePostSchema, postQuerySchema } from '../schemas/post.schema';
import { cachePresets } from '../middlewares/cache.middleware';
import { createRateLimiter } from '../middlewares/rate-limiter';

const router = Router();

// Rate limiters
const createLimiter = createRateLimiter({ windowMs: 60000, maxRequests: 10, keyPrefix: 'post-create' });

// ============================================================
// PUBLIC ROUTES
// ============================================================

// GET /posts - List posts with pagination, filtering, sorting
router.get('/',
  cachePresets.short,
  validateQuery(postQuerySchema),
  postController.list.bind(postController)
);

// GET /posts/:id - Get single post
router.get('/:id',
  validateParams(idParamSchema),
  optionalAuth,
  postController.getById.bind(postController)
);

// GET /posts/slug/:slug - Get post by slug
router.get('/slug/:slug',
  optionalAuth,
  postController.getBySlug.bind(postController)
);

// GET /posts/:id/comments - Post comments
router.get('/:id/comments',
  validateParams(idParamSchema),
  cachePresets.short,
  postController.getComments.bind(postController)
);

// ============================================================
// AUTHENTICATED ROUTES
// ============================================================

// POST /posts - Create post
router.post('/',
  authenticate,
  createLimiter,
  validateBody(createPostSchema),
  postController.create.bind(postController)
);

// POST /posts/bulk - Bulk create
router.post('/bulk',
  authenticate,
  postController.bulkCreate.bind(postController)
);

// PUT /posts/:id - Full update
router.put('/:id',
  authenticate,
  validateParams(idParamSchema),
  validateBody(updatePostSchema),
  postController.update.bind(postController)
);

// PATCH /posts/:id - Partial update
router.patch('/:id',
  authenticate,
  validateParams(idParamSchema),
  postController.partialUpdate.bind(postController)
);

// DELETE /posts/:id - Soft delete
router.delete('/:id',
  authenticate,
  validateParams(idParamSchema),
  postController.delete.bind(postController)
);

// POST /posts/:id/comments - Add comment to post
router.post('/:id/comments',
  authenticate,
  validateParams(idParamSchema),
  commentController.create.bind(commentController)
);

// DELETE /posts/:id/comments/:cid - Delete comment
router.delete('/:id/comments/:cid',
  authenticate,
  commentController.delete.bind(commentController)
);

// ============================================================
// ADMIN/MODERATOR ROUTES
// ============================================================

// POST /posts/:id/publish
router.post('/:id/publish',
  authenticate,
  authorize('admin', 'moderator'),
  validateParams(idParamSchema),
  postController.publish.bind(postController)
);

// POST /posts/:id/archive
router.post('/:id/archive',
  authenticate,
  authorize('admin', 'moderator'),
  validateParams(idParamSchema),
  postController.archive.bind(postController)
);

// POST /posts/:id/restore - Restore soft-deleted post
router.post('/:id/restore',
  authenticate,
  authorize('admin'),
  validateParams(idParamSchema),
  postController.restore.bind(postController)
);

// DELETE /posts/:id/hard - Hard delete (admin only)
router.delete('/:id/hard',
  authenticate,
  authorize('admin'),
  validateParams(idParamSchema),
  postController.hardDelete.bind(postController)
);

// PATCH /posts/bulk - Bulk update
router.patch('/bulk',
  authenticate,
  authorize('admin', 'moderator'),
  postController.bulkUpdate.bind(postController)
);

// DELETE /posts/bulk - Bulk delete
router.delete('/bulk',
  authenticate,
  authorize('admin'),
  postController.bulkDelete.bind(postController)
);

export default router;
```

---

## 11. Request/Response Standards

```typescript
// Response Examples

// Success List Response
{
  "success": true,
  "data": [
    {
      "id": 1,
      "title": "Hello World",
      "slug": "hello-world",
      "excerpt": "This is my first post",
      "status": "published",
      "tags": ["nodejs", "typescript"],
      "viewCount": 1234,
      "publishedAt": "2024-01-15T10:00:00Z",
      "createdAt": "2024-01-15T09:00:00Z",
      "author": { "id": 1, "username": "john" },
      "category": { "id": 2, "name": "Tech", "slug": "tech" },
      "commentCount": 5
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100,
    "totalPages": 5,
    "hasNext": true,
    "hasPrev": false
  },
  "meta": {
    "timestamp": "2024-01-15T10:00:00Z"
  }
}

// Success Single Response
{
  "success": true,
  "data": { ... },
  "meta": { "timestamp": "..." }
}

// Created Response (201)
{
  "success": true,
  "data": { "id": 123, ... },
  "meta": { "timestamp": "..." }
}

// Error Response
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Post not found"
  },
  "meta": { "timestamp": "..." }
}

// Validation Error Response (422)
{
  "success": false,
  "error": {
    "code": "UNPROCESSABLE_ENTITY",
    "message": "Validation failed",
    "details": [
      { "field": "title", "message": "Title is required" },
      { "field": "content", "message": "Content must be at least 10 characters" }
    ]
  },
  "meta": { "timestamp": "..." }
}

// Bulk Operation Response (207)
{
  "success": true,
  "data": {
    "created": [...],
    "failed": [
      { "index": 2, "error": "Email already in use" }
    ],
    "total": 10,
    "successful": 9,
    "failed_count": 1
  }
}
```

---

## 12. Field Selection

```typescript
// src/utils/field-selection.ts
export function selectFields<T extends Record<string, unknown>>(
  obj: T,
  fields: string[]
): Partial<T> {
  if (!fields || fields.length === 0) return obj;
  
  return fields.reduce((acc, field) => {
    if (field in obj) {
      acc[field as keyof T] = obj[field as keyof T];
    }
    return acc;
  }, {} as Partial<T>);
}

export function parseFieldsParam(fieldsParam?: string): string[] {
  if (!fieldsParam) return [];
  return fieldsParam.split(',').map(f => f.trim()).filter(Boolean);
}

// Usage in controller
async getById(req: Request, res: Response, next: NextFunction): Promise<void> {
  const post = await postService.getPost(id);
  
  // Field selection
  const fields = parseFieldsParam(req.query.fields as string);
  const data = fields.length > 0 ? selectFields(post, fields) : post;
  
  ResponseUtils.success(res, data);
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. การออกแบบ RESTful endpoints ที่ถูกต้อง
2. GET /posts: pagination, filtering (status, category, search, tags), sorting ครบ
3. POST /posts: create with validation
4. PUT/PATCH: full update และ partial update
5. DELETE: soft delete และ hard delete
6. Nested resources: /posts/:id/comments
7. Bulk operations: create, update, delete
8. Soft delete implementation พร้อม restore
9. HTTP caching ด้วย ETag และ Last-Modified
10. Database transactions ใน service layer
11. Redis cache ใน service layer
12. Field selection: ?fields=id,title,excerpt
13. Response format standards

เป็นการสรุป Part 11-15 ซึ่งครอบคลุม Node.js connection กับ PostgreSQL, Redis, MinIO/S3 และการสร้าง REST API พร้อม CRUD operations ที่สมบูรณ์
