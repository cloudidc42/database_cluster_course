# Part 13: เชื่อมต่อ Node.js กับ MinIO/S3

## บทนำ

ในบทนี้เราจะเรียนรู้การใช้ Node.js เชื่อมต่อกับ MinIO (self-hosted S3-compatible object storage) และ AWS S3 สำหรับจัดเก็บไฟล์, รูปภาพ, และ assets ต่างๆ

---

## 1. การติดตั้ง

```bash
# MinIO SDK (S3-compatible)
npm install minio

# AWS SDK v3 (สำหรับ AWS S3)
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner

# สำหรับ multipart upload
npm install @aws-sdk/lib-storage

# Utilities
npm install uuid
npm install sharp  # สำหรับ image processing

# TypeScript types
npm install --save-dev @types/minio
```

### เปรียบเทียบ minio SDK vs aws-sdk v3

| Feature | minio SDK | @aws-sdk/client-s3 |
|---------|-----------|-------------------|
| MinIO support | ✅ Native | ✅ Compatible |
| AWS S3 | ✅ | ✅ Native |
| TypeScript | ✅ | ✅ |
| Presigned URL | ✅ | ✅ |
| Multipart | ✅ | ✅ |
| API Style | Callback/Promise | Promise |
| Bundle size | Smaller | Modular |

**แนะนำ**: ใช้ `minio` SDK เมื่อ primary target คือ MinIO, ใช้ `@aws-sdk/client-s3` เมื่อ primary target คือ AWS S3

---

## 2. MinIO Connection Config

```typescript
// src/config/storage.ts
import * as Minio from 'minio';
import dotenv from 'dotenv';

dotenv.config();

export const minioConfig = {
  endPoint: process.env.MINIO_ENDPOINT || 'localhost',
  port: parseInt(process.env.MINIO_PORT || '9000', 10),
  useSSL: process.env.MINIO_USE_SSL === 'true',
  accessKey: process.env.MINIO_ACCESS_KEY || 'minioadmin',
  secretKey: process.env.MINIO_SECRET_KEY || 'minioadmin',
  region: process.env.MINIO_REGION || 'us-east-1',
};

export const minioClient = new Minio.Client(minioConfig);

// Default bucket name
export const DEFAULT_BUCKET = process.env.MINIO_BUCKET || 'my-app';
```

## 3. AWS S3 Connection Config

```typescript
// src/config/s3.ts
import {
  S3Client,
  S3ClientConfig,
} from '@aws-sdk/client-s3';

export const s3Config: S3ClientConfig = {
  region: process.env.AWS_REGION || 'ap-southeast-1',
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID || '',
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY || '',
  },
};

// สำหรับ MinIO ด้วย AWS SDK v3
export const minioS3Config: S3ClientConfig = {
  region: process.env.MINIO_REGION || 'us-east-1',
  endpoint: `http${process.env.MINIO_USE_SSL === 'true' ? 's' : ''}://${process.env.MINIO_ENDPOINT}:${process.env.MINIO_PORT}`,
  credentials: {
    accessKeyId: process.env.MINIO_ACCESS_KEY || 'minioadmin',
    secretAccessKey: process.env.MINIO_SECRET_KEY || 'minioadmin',
  },
  forcePathStyle: true, // สำคัญสำหรับ MinIO
};

export const s3Client = new S3Client(s3Config);
export const minioS3Client = new S3Client(minioS3Config);
```

### Environment Variables
```bash
# .env
# MinIO
MINIO_ENDPOINT=localhost
MINIO_PORT=9000
MINIO_USE_SSL=false
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin_secret
MINIO_BUCKET=my-app
MINIO_REGION=us-east-1

# AWS S3
AWS_REGION=ap-southeast-1
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
AWS_S3_BUCKET=my-app-bucket
```

---

## 4. Storage Service (Unified Interface)

```typescript
// src/services/storage.service.ts
import * as Minio from 'minio';
import {
  S3Client,
  CreateBucketCommand,
  DeleteBucketCommand,
  ListBucketsCommand,
  HeadBucketCommand,
  PutObjectCommand,
  GetObjectCommand,
  DeleteObjectCommand,
  HeadObjectCommand,
  ListObjectsV2Command,
  CopyObjectCommand,
  DeleteObjectsCommand,
  ObjectIdentifier,
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { Upload } from '@aws-sdk/lib-storage';
import { Readable, PassThrough } from 'stream';
import { createReadStream, createWriteStream } from 'fs';
import { stat } from 'fs/promises';
import path from 'path';

export interface StorageObject {
  key: string;
  size: number;
  lastModified: Date;
  etag?: string;
  contentType?: string;
}

export interface UploadOptions {
  contentType?: string;
  metadata?: Record<string, string>;
  acl?: string;
  ttl?: number;
}

export interface PresignedUrlOptions {
  ttlSeconds?: number;
  contentType?: string;
  metadata?: Record<string, string>;
}

class StorageService {
  private client: S3Client;
  private bucket: string;

  constructor(client: S3Client, bucket: string) {
    this.client = client;
    this.bucket = bucket;
  }

  // ============================================================
  // BUCKET OPERATIONS
  // ============================================================

  async bucketExists(bucketName?: string): Promise<boolean> {
    try {
      await this.client.send(new HeadBucketCommand({
        Bucket: bucketName || this.bucket,
      }));
      return true;
    } catch (error: any) {
      if (error.name === 'NotFound' || error.$metadata?.httpStatusCode === 404) {
        return false;
      }
      throw error;
    }
  }

  async createBucket(bucketName?: string, region?: string): Promise<void> {
    const targetBucket = bucketName || this.bucket;
    const exists = await this.bucketExists(targetBucket);
    
    if (exists) {
      console.log(`[Storage] Bucket already exists: ${targetBucket}`);
      return;
    }

    await this.client.send(new CreateBucketCommand({
      Bucket: targetBucket,
      CreateBucketConfiguration: region
        ? { LocationConstraint: region as any }
        : undefined,
    }));
    
    console.log(`[Storage] Created bucket: ${targetBucket}`);
  }

  async listBuckets(): Promise<string[]> {
    const result = await this.client.send(new ListBucketsCommand({}));
    return (result.Buckets || []).map(b => b.Name || '').filter(Boolean);
  }

  async deleteBucket(bucketName?: string): Promise<void> {
    await this.client.send(new DeleteBucketCommand({
      Bucket: bucketName || this.bucket,
    }));
  }

  // ============================================================
  // OBJECT OPERATIONS
  // ============================================================

  async putObject(
    key: string,
    body: Buffer | Readable | string,
    options: UploadOptions = {}
  ): Promise<string> {
    const command = new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      Body: body,
      ContentType: options.contentType || this.getContentType(key),
      Metadata: options.metadata,
    });

    const result = await this.client.send(command);
    return result.ETag || '';
  }

  async getObject(key: string): Promise<Buffer> {
    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: key,
    });

    const result = await this.client.send(command);
    
    if (!result.Body) {
      throw new Error(`Object not found: ${key}`);
    }

    // Convert stream to buffer
    const stream = result.Body as Readable;
    const chunks: Buffer[] = [];
    
    for await (const chunk of stream) {
      chunks.push(Buffer.from(chunk));
    }
    
    return Buffer.concat(chunks);
  }

  async getObjectStream(key: string): Promise<Readable> {
    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: key,
    });

    const result = await this.client.send(command);
    
    if (!result.Body) {
      throw new Error(`Object not found: ${key}`);
    }

    return result.Body as Readable;
  }

  async removeObject(key: string): Promise<void> {
    await this.client.send(new DeleteObjectCommand({
      Bucket: this.bucket,
      Key: key,
    }));
  }

  async removeObjects(keys: string[]): Promise<void> {
    if (keys.length === 0) return;
    
    const objects: ObjectIdentifier[] = keys.map(key => ({ Key: key }));
    
    await this.client.send(new DeleteObjectsCommand({
      Bucket: this.bucket,
      Delete: { Objects: objects },
    }));
  }

  async statObject(key: string): Promise<StorageObject | null> {
    try {
      const command = new HeadObjectCommand({
        Bucket: this.bucket,
        Key: key,
      });

      const result = await this.client.send(command);
      
      return {
        key,
        size: result.ContentLength || 0,
        lastModified: result.LastModified || new Date(),
        etag: result.ETag,
        contentType: result.ContentType,
      };
    } catch (error: any) {
      if (error.$metadata?.httpStatusCode === 404) return null;
      throw error;
    }
  }

  async listObjects(prefix?: string, maxKeys: number = 1000): Promise<StorageObject[]> {
    const objects: StorageObject[] = [];
    let continuationToken: string | undefined;
    
    do {
      const command = new ListObjectsV2Command({
        Bucket: this.bucket,
        Prefix: prefix,
        MaxKeys: maxKeys,
        ContinuationToken: continuationToken,
      });

      const result = await this.client.send(command);
      
      for (const obj of result.Contents || []) {
        objects.push({
          key: obj.Key || '',
          size: obj.Size || 0,
          lastModified: obj.LastModified || new Date(),
          etag: obj.ETag,
        });
      }
      
      continuationToken = result.NextContinuationToken;
    } while (continuationToken);
    
    return objects;
  }

  async copyObject(sourceKey: string, destKey: string): Promise<void> {
    await this.client.send(new CopyObjectCommand({
      Bucket: this.bucket,
      CopySource: `${this.bucket}/${sourceKey}`,
      Key: destKey,
    }));
  }

  // ============================================================
  // FILE UPLOAD/DOWNLOAD
  // ============================================================

  async uploadFile(
    filePath: string,
    key: string,
    options: UploadOptions = {}
  ): Promise<string> {
    const fileStats = await stat(filePath);
    const fileStream = createReadStream(filePath);
    
    const contentType = options.contentType || this.getContentType(filePath);
    
    // Use multipart upload for large files (>5MB)
    if (fileStats.size > 5 * 1024 * 1024) {
      return this.multipartUpload(key, fileStream, {
        ...options,
        contentType,
      });
    }
    
    const fileBuffer = await new Promise<Buffer>((resolve, reject) => {
      const chunks: Buffer[] = [];
      fileStream.on('data', chunk => chunks.push(Buffer.from(chunk)));
      fileStream.on('end', () => resolve(Buffer.concat(chunks)));
      fileStream.on('error', reject);
    });
    
    return this.putObject(key, fileBuffer, { ...options, contentType });
  }

  async downloadFile(key: string, destPath: string): Promise<void> {
    const stream = await this.getObjectStream(key);
    const writeStream = createWriteStream(destPath);
    
    await new Promise<void>((resolve, reject) => {
      stream.pipe(writeStream)
        .on('finish', resolve)
        .on('error', reject);
    });
  }

  async uploadBuffer(
    buffer: Buffer,
    key: string,
    options: UploadOptions = {}
  ): Promise<string> {
    return this.putObject(key, buffer, options);
  }

  async uploadStream(
    stream: Readable,
    key: string,
    options: UploadOptions = {}
  ): Promise<string> {
    return this.putObject(key, stream, options);
  }

  // ============================================================
  // PRESIGNED URLS
  // ============================================================

  async getPresignedDownloadUrl(
    key: string,
    options: PresignedUrlOptions = {}
  ): Promise<string> {
    const ttl = options.ttlSeconds || 3600; // 1 hour default
    
    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: key,
    });

    return getSignedUrl(this.client, command, { expiresIn: ttl });
  }

  async getPresignedUploadUrl(
    key: string,
    options: PresignedUrlOptions = {}
  ): Promise<string> {
    const ttl = options.ttlSeconds || 3600;
    
    const command = new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      ContentType: options.contentType,
      Metadata: options.metadata,
    });

    return getSignedUrl(this.client, command, { expiresIn: ttl });
  }

  // ============================================================
  // MULTIPART UPLOAD (สำหรับไฟล์ใหญ่ >5MB)
  // ============================================================

  async multipartUpload(
    key: string,
    body: Readable | Buffer,
    options: UploadOptions = {}
  ): Promise<string> {
    const upload = new Upload({
      client: this.client,
      params: {
        Bucket: this.bucket,
        Key: key,
        Body: body,
        ContentType: options.contentType || this.getContentType(key),
        Metadata: options.metadata,
      },
      queueSize: 4,        // parallel uploads
      partSize: 10 * 1024 * 1024, // 10MB per part
      leavePartsOnError: false,
    });

    upload.on('httpUploadProgress', (progress) => {
      const percent = progress.total
        ? Math.round((progress.loaded! / progress.total) * 100)
        : 0;
      console.log(`[Storage] Upload progress: ${percent}%`);
    });

    const result = await upload.done();
    return result.ETag || '';
  }

  // ============================================================
  // UTILITIES
  // ============================================================

  private getContentType(filename: string): string {
    const ext = path.extname(filename).toLowerCase();
    const types: Record<string, string> = {
      '.jpg': 'image/jpeg',
      '.jpeg': 'image/jpeg',
      '.png': 'image/png',
      '.gif': 'image/gif',
      '.webp': 'image/webp',
      '.svg': 'image/svg+xml',
      '.pdf': 'application/pdf',
      '.doc': 'application/msword',
      '.docx': 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
      '.xls': 'application/vnd.ms-excel',
      '.xlsx': 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
      '.zip': 'application/zip',
      '.mp4': 'video/mp4',
      '.mp3': 'audio/mpeg',
      '.json': 'application/json',
      '.txt': 'text/plain',
      '.html': 'text/html',
      '.css': 'text/css',
      '.js': 'application/javascript',
    };
    return types[ext] || 'application/octet-stream';
  }
}

// Create service instances
export const storageService = new StorageService(
  minioS3Client,
  process.env.MINIO_BUCKET || 'my-app'
);

export const s3StorageService = new StorageService(
  s3Client,
  process.env.AWS_S3_BUCKET || 'my-app'
);

export default storageService;
```

---

## 5. Full Working Example: File/Image Upload API

### Types
```typescript
// src/types/file.ts
export interface FileRecord {
  id: string;
  originalName: string;
  storagePath: string;
  contentType: string;
  size: number;
  userId: number;
  category: string;
  isPublic: boolean;
  metadata: Record<string, string>;
  createdAt: Date;
  thumbnailPath?: string;
}

export interface UploadResult {
  id: string;
  url: string;
  thumbnailUrl?: string;
  size: number;
  contentType: string;
  originalName: string;
}
```

### File Upload Service
```typescript
// src/services/file-upload.service.ts
import { v4 as uuidv4 } from 'uuid';
import sharp from 'sharp';
import path from 'path';
import { Readable } from 'stream';
import storageService from './storage.service';
import { db } from '../db/pool';

export interface FileUploadOptions {
  userId: number;
  category?: string;
  isPublic?: boolean;
  generateThumbnail?: boolean;
  maxWidth?: number;
  maxHeight?: number;
  quality?: number;
}

export class FileUploadService {
  private readonly maxFileSize: number = 50 * 1024 * 1024; // 50MB
  private readonly allowedImageTypes = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
  private readonly allowedDocTypes = [
    'application/pdf',
    'application/msword',
    'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
  ];

  async uploadFile(
    buffer: Buffer,
    originalName: string,
    contentType: string,
    options: FileUploadOptions
  ): Promise<UploadResult> {
    // Validate size
    if (buffer.length > this.maxFileSize) {
      throw new Error(`File size exceeds maximum limit of ${this.maxFileSize / 1024 / 1024}MB`);
    }

    const fileId = uuidv4();
    const ext = path.extname(originalName).toLowerCase();
    const storagePath = this.buildStoragePath(options.userId, options.category || 'general', fileId, ext);

    // Upload original file
    await storageService.uploadBuffer(buffer, storagePath, {
      contentType,
      metadata: {
        'original-name': originalName,
        'user-id': options.userId.toString(),
        'file-id': fileId,
      },
    });

    let thumbnailPath: string | undefined;

    // Generate thumbnail for images
    if (options.generateThumbnail && this.isImage(contentType)) {
      thumbnailPath = await this.generateThumbnail(
        buffer,
        fileId,
        options.userId,
        options.maxWidth || 300,
        options.maxHeight || 300,
        options.quality || 80
      );
    }

    // Optimize image if needed
    if (this.isImage(contentType) && buffer.length > 1024 * 1024) {
      const optimized = await this.optimizeImage(buffer, contentType, options);
      await storageService.uploadBuffer(optimized, storagePath, {
        contentType,
        metadata: {
          'original-name': originalName,
          'user-id': options.userId.toString(),
          'file-id': fileId,
          'optimized': 'true',
        },
      });
    }

    // Save to database
    await db.query(
      `INSERT INTO files (id, original_name, storage_path, content_type, size, user_id, category, is_public, thumbnail_path, created_at)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, NOW())`,
      [
        fileId,
        originalName,
        storagePath,
        contentType,
        buffer.length,
        options.userId,
        options.category || 'general',
        options.isPublic || false,
        thumbnailPath || null,
      ]
    );

    // Get download URL
    const url = options.isPublic
      ? `${process.env.STORAGE_PUBLIC_URL}/${storagePath}`
      : await storageService.getPresignedDownloadUrl(storagePath, { ttlSeconds: 3600 });

    const thumbnailUrl = thumbnailPath
      ? (options.isPublic
          ? `${process.env.STORAGE_PUBLIC_URL}/${thumbnailPath}`
          : await storageService.getPresignedDownloadUrl(thumbnailPath, { ttlSeconds: 3600 }))
      : undefined;

    return {
      id: fileId,
      url,
      thumbnailUrl,
      size: buffer.length,
      contentType,
      originalName,
    };
  }

  async getFileUrl(fileId: string, ttlSeconds: number = 3600): Promise<string> {
    const result = await db.query<{ storage_path: string; is_public: boolean }>(
      'SELECT storage_path, is_public FROM files WHERE id = $1',
      [fileId]
    );

    if (!result.rows[0]) {
      throw new Error('File not found');
    }

    const { storage_path, is_public } = result.rows[0];

    if (is_public) {
      return `${process.env.STORAGE_PUBLIC_URL}/${storage_path}`;
    }

    return storageService.getPresignedDownloadUrl(storage_path, { ttlSeconds });
  }

  async deleteFile(fileId: string, userId: number): Promise<void> {
    const result = await db.query<{
      storage_path: string;
      thumbnail_path: string | null;
      user_id: number;
    }>(
      'SELECT storage_path, thumbnail_path, user_id FROM files WHERE id = $1',
      [fileId]
    );

    if (!result.rows[0]) {
      throw new Error('File not found');
    }

    const file = result.rows[0];

    if (file.user_id !== userId) {
      throw new Error('Unauthorized');
    }

    // Delete from storage
    await storageService.removeObject(file.storage_path);
    
    if (file.thumbnail_path) {
      await storageService.removeObject(file.thumbnail_path).catch(console.error);
    }

    // Delete from database
    await db.query('DELETE FROM files WHERE id = $1', [fileId]);
  }

  private buildStoragePath(
    userId: number,
    category: string,
    fileId: string,
    ext: string
  ): string {
    const date = new Date();
    const year = date.getFullYear();
    const month = String(date.getMonth() + 1).padStart(2, '0');
    return `users/${userId}/${category}/${year}/${month}/${fileId}${ext}`;
  }

  private isImage(contentType: string): boolean {
    return this.allowedImageTypes.includes(contentType);
  }

  private async generateThumbnail(
    buffer: Buffer,
    fileId: string,
    userId: number,
    width: number,
    height: number,
    quality: number
  ): Promise<string> {
    const thumbnailBuffer = await sharp(buffer)
      .resize(width, height, {
        fit: 'inside',
        withoutEnlargement: true,
      })
      .jpeg({ quality })
      .toBuffer();

    const thumbnailPath = `users/${userId}/thumbnails/${fileId}_thumb.jpg`;
    
    await storageService.uploadBuffer(thumbnailBuffer, thumbnailPath, {
      contentType: 'image/jpeg',
    });

    return thumbnailPath;
  }

  private async optimizeImage(
    buffer: Buffer,
    contentType: string,
    options: FileUploadOptions
  ): Promise<Buffer> {
    let sharpInstance = sharp(buffer);

    if (options.maxWidth || options.maxHeight) {
      sharpInstance = sharpInstance.resize(
        options.maxWidth,
        options.maxHeight,
        { fit: 'inside', withoutEnlargement: true }
      );
    }

    if (contentType === 'image/jpeg') {
      return sharpInstance.jpeg({ quality: options.quality || 85 }).toBuffer();
    } else if (contentType === 'image/png') {
      return sharpInstance.png({ compressionLevel: 9 }).toBuffer();
    } else if (contentType === 'image/webp') {
      return sharpInstance.webp({ quality: options.quality || 85 }).toBuffer();
    }

    return buffer;
  }
}

export const fileUploadService = new FileUploadService();
```

### Upload Controller
```typescript
// src/controllers/upload.controller.ts
import { Request, Response, NextFunction } from 'express';
import multer from 'multer';
import { fileUploadService } from '../services/file-upload.service';
import storageService from '../services/storage.service';

// Use memory storage (buffer)
export const upload = multer({
  storage: multer.memoryStorage(),
  limits: {
    fileSize: 50 * 1024 * 1024, // 50MB
    files: 10,
  },
  fileFilter: (req, file, callback) => {
    const allowedTypes = [
      'image/jpeg', 'image/png', 'image/gif', 'image/webp',
      'application/pdf', 'video/mp4',
    ];
    
    if (allowedTypes.includes(file.mimetype)) {
      callback(null, true);
    } else {
      callback(new Error(`File type not allowed: ${file.mimetype}`));
    }
  },
});

export class UploadController {
  async uploadSingle(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      if (!req.file) {
        res.status(400).json({ success: false, error: 'No file provided' });
        return;
      }

      const userId = (req as any).user?.id || 1; // From auth middleware

      const result = await fileUploadService.uploadFile(
        req.file.buffer,
        req.file.originalname,
        req.file.mimetype,
        {
          userId,
          category: req.body.category || 'general',
          isPublic: req.body.isPublic === 'true',
          generateThumbnail: req.file.mimetype.startsWith('image/'),
          maxWidth: 1920,
          maxHeight: 1080,
          quality: 85,
        }
      );

      res.status(201).json({
        success: true,
        data: result,
      });
    } catch (error) {
      next(error);
    }
  }

  async uploadMultiple(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      if (!req.files || !Array.isArray(req.files) || req.files.length === 0) {
        res.status(400).json({ success: false, error: 'No files provided' });
        return;
      }

      const userId = (req as any).user?.id || 1;
      
      const results = await Promise.allSettled(
        req.files.map(file =>
          fileUploadService.uploadFile(
            file.buffer,
            file.originalname,
            file.mimetype,
            {
              userId,
              category: req.body.category || 'general',
              isPublic: req.body.isPublic === 'true',
              generateThumbnail: file.mimetype.startsWith('image/'),
            }
          )
        )
      );

      const uploaded = results
        .filter(r => r.status === 'fulfilled')
        .map(r => (r as PromiseFulfilledResult<any>).value);

      const errors = results
        .filter(r => r.status === 'rejected')
        .map(r => (r as PromiseRejectedResult).reason.message);

      res.status(207).json({
        success: true,
        data: {
          uploaded,
          errors,
          total: req.files.length,
          successful: uploaded.length,
          failed: errors.length,
        },
      });
    } catch (error) {
      next(error);
    }
  }

  async getFileUrl(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const { fileId } = req.params;
      const ttl = parseInt(req.query.ttl as string || '3600', 10);
      
      const url = await fileUploadService.getFileUrl(fileId, ttl);
      
      res.json({
        success: true,
        data: { url, expiresIn: ttl },
      });
    } catch (error) {
      next(error);
    }
  }

  async deleteFile(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const { fileId } = req.params;
      const userId = (req as any).user?.id || 1;
      
      await fileUploadService.deleteFile(fileId, userId);
      
      res.status(204).send();
    } catch (error) {
      next(error);
    }
  }

  async getPresignedUploadUrl(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const { filename, contentType } = req.body;
      
      if (!filename || !contentType) {
        res.status(400).json({
          success: false,
          error: 'filename and contentType are required',
        });
        return;
      }

      const userId = (req as any).user?.id || 1;
      const fileId = require('uuid').v4();
      const ext = require('path').extname(filename);
      const key = `uploads/${userId}/${fileId}${ext}`;
      
      const uploadUrl = await storageService.getPresignedUploadUrl(key, {
        contentType,
        ttlSeconds: 300, // 5 minutes to upload
        metadata: {
          'user-id': userId.toString(),
          'file-id': fileId,
          'original-name': filename,
        },
      });

      res.json({
        success: true,
        data: {
          uploadUrl,
          key,
          fileId,
          expiresIn: 300,
        },
      });
    } catch (error) {
      next(error);
    }
  }
}

export const uploadController = new UploadController();
```

### Routes
```typescript
// src/routes/upload.routes.ts
import { Router } from 'express';
import { uploadController, upload } from '../controllers/upload.controller';

const router = Router();

// Single file upload
router.post('/single', upload.single('file'), uploadController.uploadSingle.bind(uploadController));

// Multiple files upload
router.post('/multiple', upload.array('files', 10), uploadController.uploadMultiple.bind(uploadController));

// Get presigned URL for client-side upload
router.post('/presigned-url', uploadController.getPresignedUploadUrl.bind(uploadController));

// Get download URL
router.get('/:fileId/url', uploadController.getFileUrl.bind(uploadController));

// Delete file
router.delete('/:fileId', uploadController.deleteFile.bind(uploadController));

export default router;
```

---

## 6. Database Schema สำหรับ Files

```sql
-- migrations/003_create_files.sql
CREATE TABLE IF NOT EXISTS files (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  original_name   VARCHAR(255) NOT NULL,
  storage_path    TEXT NOT NULL,
  content_type    VARCHAR(100) NOT NULL,
  size            BIGINT NOT NULL,
  user_id         INTEGER REFERENCES users(id) ON DELETE CASCADE,
  category        VARCHAR(50) NOT NULL DEFAULT 'general',
  is_public       BOOLEAN NOT NULL DEFAULT false,
  thumbnail_path  TEXT,
  metadata        JSONB DEFAULT '{}',
  created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_files_user_id ON files(user_id);
CREATE INDEX idx_files_category ON files(category);
CREATE INDEX idx_files_created_at ON files(created_at DESC);
CREATE INDEX idx_files_is_public ON files(is_public) WHERE is_public = true;
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. การติดตั้ง MinIO SDK และ AWS SDK v3
2. การ config connection สำหรับทั้ง MinIO และ AWS S3
3. Bucket operations ครบ
4. Object operations: upload, download, delete, list, copy
5. Presigned URLs สำหรับ upload/download
6. Multipart upload สำหรับไฟล์ขนาดใหญ่
7. การ implement File Upload API ครบ
8. Image optimization ด้วย sharp
9. Thumbnail generation

ในบทต่อไปเราจะเรียนรู้การสร้าง REST API ด้วย Express.js
