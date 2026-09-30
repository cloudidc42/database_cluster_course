# Part 40: S3 Presigned URLs และ Access Control

## บทนำ

Object Storage เช่น Amazon S3 หรือ MinIO เป็นวิธีที่นิยมมากในการเก็บไฟล์ใน Cloud ในบทนี้เราจะเรียนรู้การจัดการ Access Control และการใช้ Presigned URLs เพื่อให้ Client สามารถ Upload/Download ไฟล์โดยตรงโดยไม่ต้องผ่าน API Server

ในบทนี้เราจะเรียนรู้:
- Object Storage Access Control
- Presigned URLs: GET, PUT, DELETE
- Direct Client Upload workflow
- Security best practices
- CORS configuration
- Multipart uploads
- Full implementation: Secure File Sharing System

---

## 1. Object Storage Access Control

### 1.1 Public vs Private Buckets

```
Public Bucket:
├── ทุกคนสามารถ read ได้
├── เหมาะกับ: static websites, public images
└── URL: https://bucket.s3.amazonaws.com/file.jpg → ใครก็เข้าถึงได้

Private Bucket (Default):
├── ต้องการ authentication ทุก request
├── เหมาะกับ: user uploads, sensitive documents
└── URL: https://bucket.s3.amazonaws.com/file.jpg → 403 Forbidden

ไม่ควรทำ:
❌ ทำให้ bucket เป็น public แล้วเก็บข้อมูล user ใน bucket นั้น
❌ เก็บ credentials ใน bucket

ควรทำ:
✅ Private bucket + Presigned URLs สำหรับ user files
✅ Separate public bucket สำหรับ static assets
```

### 1.2 Bucket Policies

```json
// Bucket Policy: อนุญาตแค่ role เฉพาะ
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowApplicationAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789:role/MyAppRole"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::my-app-uploads/*"
    },
    {
      "Sid": "DenyPublicAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-app-uploads/*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

### 1.3 MinIO Bucket Policy

```json
// MinIO Policy (compatible with AWS S3 format)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": ["arn:aws:iam::myapp:user/app-service"]
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::uploads",
        "arn:aws:s3:::uploads/*"
      ]
    }
  ]
}
```

---

## 2. Presigned URLs

### 2.1 แนวคิด

```
ปกติการ access private S3 object:
Client → API Server → S3 → API Server → Client

ปัญหา:
- ไฟล์ขนาดใหญ่ผ่าน API Server ทำให้ช้า
- API Server ต้องใช้ bandwidth เยอะ
- ค่าใช้จ่ายสูงขึ้น (traffic ผ่าน EC2 แพงกว่า S3)

ด้วย Presigned URL:
Client → API Server → (สร้าง signed URL)
                                    ↓
Client ← API Server               S3
Client ────────────────────────────▶ S3 (Direct!)

ข้อดี:
- API Server ไม่ต้อง proxy ไฟล์
- Bandwidth ต่ำลง
- ค่าใช้จ่ายน้อยลง
- Upload ได้เร็วขึ้น (direct to S3)
```

### 2.2 ประเภทของ Presigned URLs

```
GET Presigned URL: temporary download URL
  - User คลิก link → download ไฟล์โดยตรงจาก S3
  - ไม่ต้องผ่าน API Server
  - มี expiration time

PUT Presigned URL: temporary upload URL
  - Client upload ไปยัง S3 โดยตรง
  - ไม่ต้องส่งไฟล์ผ่าน API Server
  - สามารถจำกัด content-type และ size

DELETE Presigned URL: temporary delete URL
  - ใช้น้อย (ส่วนใหญ่ให้ backend delete ดีกว่า)
```

---

## 3. Installation

```bash
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
# หรือสำหรับ MinIO (S3 compatible)
npm install minio
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner  # ยังใช้ AWS SDK ได้กับ MinIO
```

---

## 4. Storage Service Implementation

```typescript
// src/storage/StorageService.ts

import { 
  S3Client, 
  GetObjectCommand, 
  PutObjectCommand, 
  DeleteObjectCommand,
  HeadObjectCommand,
  ListObjectsV2Command,
  CreateMultipartUploadCommand,
  UploadPartCommand,
  CompleteMultipartUploadCommand,
  AbortMultipartUploadCommand,
  CopyObjectCommand,
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { createHmac } from 'crypto';
import path from 'path';

export interface StorageConfig {
  endpoint?: string;         // For MinIO
  region: string;
  accessKeyId: string;
  secretAccessKey: string;
  bucket: string;
  bucketPublic?: string;     // Separate public bucket
  forcePathStyle?: boolean;  // Required for MinIO
  useSSL?: boolean;
}

export interface PresignedUploadOptions {
  contentType?: string;
  maxSizeBytes?: number;
  expiresIn?: number;       // seconds
  metadata?: Record<string, string>;
  prefix?: string;          // path prefix
}

export interface PresignedDownloadOptions {
  expiresIn?: number;       // seconds
  downloadFilename?: string;  // Force download with specific filename
  contentType?: string;
}

export interface ObjectMetadata {
  key: string;
  size: number;
  contentType: string;
  etag: string;
  lastModified: Date;
  metadata: Record<string, string>;
}

export interface MultipartUploadPart {
  partNumber: number;
  etag: string;
}

export class StorageService {
  private s3Client: S3Client;
  private bucket: string;
  private bucketPublic: string;
  private config: StorageConfig;

  constructor(config: StorageConfig) {
    this.config = config;
    this.bucket = config.bucket;
    this.bucketPublic = config.bucketPublic ?? config.bucket;

    this.s3Client = new S3Client({
      region: config.region,
      credentials: {
        accessKeyId: config.accessKeyId,
        secretAccessKey: config.secretAccessKey,
      },
      ...(config.endpoint && {
        endpoint: config.endpoint,
        forcePathStyle: config.forcePathStyle ?? true,  // MinIO requires path style
      }),
      ...(config.useSSL === false && {
        tls: false,
      }),
    });
  }

  /**
   * Generate GET presigned URL (download)
   */
  async getPresignedDownloadUrl(
    key: string, 
    options: PresignedDownloadOptions = {}
  ): Promise<string> {
    const { expiresIn = 3600, downloadFilename, contentType } = options;

    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: key,
      ...(downloadFilename && {
        ResponseContentDisposition: `attachment; filename="${encodeURIComponent(downloadFilename)}"`,
      }),
      ...(contentType && {
        ResponseContentType: contentType,
      }),
    });

    const url = await getSignedUrl(this.s3Client, command, { expiresIn });
    
    console.log(`[Storage] Generated download URL for ${key} (expires in ${expiresIn}s)`);
    
    return url;
  }

  /**
   * Generate PUT presigned URL (upload)
   */
  async getPresignedUploadUrl(
    key: string,
    options: PresignedUploadOptions = {}
  ): Promise<{
    url: string;
    key: string;
    expiresAt: Date;
    fields?: Record<string, string>;
  }> {
    const { 
      contentType = 'application/octet-stream', 
      expiresIn = 300,   // 5 minutes for upload
      metadata = {},
    } = options;

    const command = new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      ContentType: contentType,
      Metadata: metadata,
    });

    const url = await getSignedUrl(this.s3Client, command, { expiresIn });
    
    const expiresAt = new Date(Date.now() + expiresIn * 1000);
    
    console.log(`[Storage] Generated upload URL for ${key} (expires at ${expiresAt.toISOString()})`);
    
    return { url, key, expiresAt };
  }

  /**
   * Generate DELETE presigned URL
   */
  async getPresignedDeleteUrl(key: string, expiresIn: number = 300): Promise<string> {
    const command = new DeleteObjectCommand({
      Bucket: this.bucket,
      Key: key,
    });

    return getSignedUrl(this.s3Client, command, { expiresIn });
  }

  /**
   * Upload file directly (server-side)
   */
  async uploadFile(
    key: string,
    content: Buffer | string,
    contentType: string,
    metadata: Record<string, string> = {}
  ): Promise<{ key: string; etag: string }> {
    const command = new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      Body: content,
      ContentType: contentType,
      Metadata: metadata,
    });

    const result = await this.s3Client.send(command);
    
    return {
      key,
      etag: result.ETag ?? '',
    };
  }

  /**
   * Download file (server-side)
   */
  async downloadFile(key: string): Promise<Buffer> {
    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: key,
    });

    const response = await this.s3Client.send(command);
    const chunks: Uint8Array[] = [];
    
    for await (const chunk of response.Body as AsyncIterable<Uint8Array>) {
      chunks.push(chunk);
    }
    
    return Buffer.concat(chunks);
  }

  /**
   * Delete file
   */
  async deleteFile(key: string): Promise<void> {
    const command = new DeleteObjectCommand({
      Bucket: this.bucket,
      Key: key,
    });

    await this.s3Client.send(command);
    console.log(`[Storage] Deleted: ${key}`);
  }

  /**
   * Get object metadata without downloading
   */
  async getObjectMetadata(key: string): Promise<ObjectMetadata | null> {
    try {
      const command = new HeadObjectCommand({
        Bucket: this.bucket,
        Key: key,
      });

      const response = await this.s3Client.send(command);
      
      return {
        key,
        size: response.ContentLength ?? 0,
        contentType: response.ContentType ?? 'application/octet-stream',
        etag: response.ETag ?? '',
        lastModified: response.LastModified ?? new Date(),
        metadata: response.Metadata ?? {},
      };
    } catch (error: any) {
      if (error.name === 'NotFound' || error.$metadata?.httpStatusCode === 404) {
        return null;
      }
      throw error;
    }
  }

  /**
   * Check if object exists
   */
  async objectExists(key: string): Promise<boolean> {
    const metadata = await this.getObjectMetadata(key);
    return metadata !== null;
  }

  /**
   * Copy object
   */
  async copyObject(sourceKey: string, destinationKey: string): Promise<void> {
    const command = new CopyObjectCommand({
      Bucket: this.bucket,
      CopySource: `${this.bucket}/${sourceKey}`,
      Key: destinationKey,
    });

    await this.s3Client.send(command);
  }

  /**
   * List objects with prefix
   */
  async listObjects(prefix: string, maxKeys: number = 100): Promise<string[]> {
    const command = new ListObjectsV2Command({
      Bucket: this.bucket,
      Prefix: prefix,
      MaxKeys: maxKeys,
    });

    const response = await this.s3Client.send(command);
    return response.Contents?.map(obj => obj.Key ?? '') ?? [];
  }

  /**
   * Generate key for user files
   */
  generateUserFileKey(userId: string, filename: string, prefix?: string): string {
    const ext = path.extname(filename);
    const timestamp = Date.now();
    const random = Math.random().toString(36).slice(2, 8);
    const cleanFilename = path.basename(filename, ext)
      .replace(/[^a-zA-Z0-9-_]/g, '_')
      .toLowerCase()
      .slice(0, 50);
    
    const parts = [
      prefix ?? 'uploads',
      `user-${userId}`,
      `${timestamp}-${random}-${cleanFilename}${ext}`,
    ].filter(Boolean);
    
    return parts.join('/');
  }

  /**
   * Get public URL (for public bucket)
   */
  getPublicUrl(key: string): string {
    if (this.config.endpoint) {
      // MinIO
      return `${this.config.endpoint}/${this.bucketPublic}/${key}`;
    }
    // AWS S3
    return `https://${this.bucketPublic}.s3.${this.config.region}.amazonaws.com/${key}`;
  }
}
```

---

## 5. PUT Presigned URL Workflow

```typescript
// src/storage/FileUploadService.ts

import { StorageService } from './StorageService';
import { Pool } from 'pg';
import path from 'path';

export interface UploadRequest {
  userId: string;
  filename: string;
  contentType: string;
  fileSize: number;
  metadata?: Record<string, string>;
  folder?: string;
}

export interface UploadResponse {
  uploadId: string;
  uploadUrl: string;
  key: string;
  expiresAt: Date;
  maxSize: number;
}

export interface UploadCompleteRequest {
  uploadId: string;
  etag?: string;
}

export interface FileRecord {
  id: string;
  userId: string;
  key: string;
  filename: string;
  contentType: string;
  size: number;
  status: 'pending' | 'completed' | 'failed';
  url?: string;
  createdAt: Date;
  updatedAt: Date;
}

// Allowed content types
const ALLOWED_IMAGE_TYPES = [
  'image/jpeg', 'image/jpg', 'image/png', 'image/gif', 
  'image/webp', 'image/svg+xml',
];

const ALLOWED_DOCUMENT_TYPES = [
  'application/pdf',
  'application/msword',
  'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
  'text/plain',
];

const ALLOWED_VIDEO_TYPES = [
  'video/mp4', 'video/webm', 'video/ogg',
];

const MAX_FILE_SIZES: Record<string, number> = {
  'image': 10 * 1024 * 1024,      // 10 MB
  'document': 50 * 1024 * 1024,   // 50 MB
  'video': 500 * 1024 * 1024,     // 500 MB
  'other': 25 * 1024 * 1024,      // 25 MB
};

export class FileUploadService {
  private storage: StorageService;
  private db: Pool;

  constructor(storage: StorageService, db: Pool) {
    this.storage = storage;
    this.db = db;
  }

  /**
   * Step 1: Client requests upload URL
   * สร้าง presigned PUT URL และ record ใน DB
   */
  async requestUploadUrl(request: UploadRequest): Promise<UploadResponse> {
    const { userId, filename, contentType, fileSize, metadata, folder } = request;

    // Validate content type
    this.validateContentType(contentType, filename);
    
    // Validate file size
    const category = this.getFileCategory(contentType);
    const maxSize = MAX_FILE_SIZES[category];
    
    if (fileSize > maxSize) {
      throw new Error(`File too large. Maximum size for ${category}: ${this.formatBytes(maxSize)}`);
    }

    // Generate unique key
    const key = this.storage.generateUserFileKey(
      userId, 
      filename, 
      folder ?? this.getFolderByCategory(category)
    );

    // สร้าง presigned URL
    const { url, expiresAt } = await this.storage.getPresignedUploadUrl(key, {
      contentType,
      maxSizeBytes: fileSize,
      expiresIn: 300,  // 5 minutes to upload
      metadata: {
        'user-id': userId,
        'original-filename': encodeURIComponent(filename),
        ...metadata,
      },
    });

    // บันทึก upload record ใน DB
    const result = await this.db.query(`
      INSERT INTO file_uploads (user_id, key, filename, content_type, size, status, expires_at)
      VALUES ($1, $2, $3, $4, $5, 'pending', $6)
      RETURNING id
    `, [userId, key, filename, contentType, fileSize, expiresAt]);

    const uploadId = result.rows[0].id;

    return {
      uploadId,
      uploadUrl: url,
      key,
      expiresAt,
      maxSize,
    };
  }

  /**
   * Step 4: Client notifies upload completion
   * Verify ว่า file อยู่ใน S3 จริงๆ แล้ว update DB
   */
  async completeUpload(request: UploadCompleteRequest, userId: string): Promise<FileRecord> {
    const { uploadId } = request;

    // ดึง upload record
    const uploadResult = await this.db.query(
      `SELECT * FROM file_uploads WHERE id = $1 AND user_id = $2`,
      [uploadId, userId]
    );

    if (!uploadResult.rows.length) {
      throw new Error('Upload record not found');
    }

    const upload = uploadResult.rows[0];
    
    if (upload.status !== 'pending') {
      throw new Error(`Upload already ${upload.status}`);
    }

    if (new Date() > new Date(upload.expires_at)) {
      throw new Error('Upload URL has expired');
    }

    // Verify file อยู่ใน S3 จริง
    const metadata = await this.storage.getObjectMetadata(upload.key);
    
    if (!metadata) {
      throw new Error('File not found in storage. Please re-upload.');
    }

    // ตรวจ content type ตรงกับที่ request ไว้ไหม
    if (metadata.contentType !== upload.content_type) {
      await this.storage.deleteFile(upload.key);  // ลบไฟล์ที่ไม่ตรง
      throw new Error('Content type mismatch. File rejected for security reasons.');
    }

    // ตรวจ size ไม่เกินที่กำหนด
    if (metadata.size > upload.size * 1.1) {  // Allow 10% tolerance
      await this.storage.deleteFile(upload.key);
      throw new Error('File size exceeds limit. File rejected.');
    }

    // Update DB
    const result = await this.db.query(`
      UPDATE file_uploads 
      SET status = 'completed', actual_size = $1, etag = $2, updated_at = NOW()
      WHERE id = $3
      RETURNING *
    `, [metadata.size, metadata.etag, uploadId]);

    const fileRecord = result.rows[0];
    
    console.log(`[Upload] Completed: ${upload.key} (${this.formatBytes(metadata.size)})`);

    return {
      id: fileRecord.id,
      userId: fileRecord.user_id,
      key: fileRecord.key,
      filename: fileRecord.filename,
      contentType: fileRecord.content_type,
      size: fileRecord.actual_size,
      status: fileRecord.status,
      createdAt: fileRecord.created_at,
      updatedAt: fileRecord.updated_at,
    };
  }

  /**
   * Get download URL for a file
   */
  async getDownloadUrl(
    fileId: string, 
    userId: string,
    options: { 
      expiresIn?: number;
      forceDownload?: boolean;
    } = {}
  ): Promise<string> {
    const result = await this.db.query(
      `SELECT * FROM file_uploads WHERE id = $1 AND user_id = $2 AND status = 'completed'`,
      [fileId, userId]
    );

    if (!result.rows.length) {
      throw new Error('File not found or not accessible');
    }

    const file = result.rows[0];

    return this.storage.getPresignedDownloadUrl(file.key, {
      expiresIn: options.expiresIn ?? 3600,
      downloadFilename: options.forceDownload ? file.filename : undefined,
    });
  }

  /**
   * List user's files
   */
  async listUserFiles(
    userId: string,
    options: { limit?: number; offset?: number; contentType?: string } = {}
  ): Promise<{ files: FileRecord[]; total: number }> {
    const { limit = 20, offset = 0, contentType } = options;

    const whereClause = contentType 
      ? `WHERE user_id = $1 AND status = 'completed' AND content_type LIKE $4`
      : `WHERE user_id = $1 AND status = 'completed'`;

    const params: any[] = contentType
      ? [userId, limit, offset, `${contentType.split('/')[0]}/%`]
      : [userId, limit, offset];

    const result = await this.db.query(
      `SELECT *, COUNT(*) OVER() as total
       FROM file_uploads
       ${whereClause}
       ORDER BY created_at DESC
       LIMIT $2 OFFSET $3`,
      params
    );

    const total = result.rows[0]?.total ? parseInt(result.rows[0].total) : 0;
    const files: FileRecord[] = result.rows.map(row => ({
      id: row.id,
      userId: row.user_id,
      key: row.key,
      filename: row.filename,
      contentType: row.content_type,
      size: row.actual_size ?? row.size,
      status: row.status,
      createdAt: row.created_at,
      updatedAt: row.updated_at,
    }));

    return { files, total };
  }

  /**
   * Delete file
   */
  async deleteFile(fileId: string, userId: string): Promise<void> {
    const result = await this.db.query(
      `SELECT * FROM file_uploads WHERE id = $1 AND user_id = $2`,
      [fileId, userId]
    );

    if (!result.rows.length) {
      throw new Error('File not found');
    }

    const file = result.rows[0];

    // Delete from S3
    await this.storage.deleteFile(file.key);

    // Delete from DB
    await this.db.query('DELETE FROM file_uploads WHERE id = $1', [fileId]);

    console.log(`[Upload] Deleted file ${fileId}: ${file.key}`);
  }

  private validateContentType(contentType: string, filename: string): void {
    const allAllowed = [
      ...ALLOWED_IMAGE_TYPES,
      ...ALLOWED_DOCUMENT_TYPES,
      ...ALLOWED_VIDEO_TYPES,
    ];

    if (!allAllowed.includes(contentType)) {
      throw new Error(`Content type not allowed: ${contentType}`);
    }

    // Double-check extension vs content type
    const ext = path.extname(filename).toLowerCase();
    const extContentTypeMap: Record<string, string[]> = {
      '.jpg': ['image/jpeg', 'image/jpg'],
      '.jpeg': ['image/jpeg', 'image/jpg'],
      '.png': ['image/png'],
      '.gif': ['image/gif'],
      '.webp': ['image/webp'],
      '.pdf': ['application/pdf'],
      '.mp4': ['video/mp4'],
    };

    const allowedTypes = extContentTypeMap[ext];
    if (allowedTypes && !allowedTypes.includes(contentType)) {
      throw new Error(`Content type ${contentType} doesn't match file extension ${ext}`);
    }
  }

  private getFileCategory(contentType: string): string {
    if (ALLOWED_IMAGE_TYPES.includes(contentType)) return 'image';
    if (ALLOWED_DOCUMENT_TYPES.includes(contentType)) return 'document';
    if (ALLOWED_VIDEO_TYPES.includes(contentType)) return 'video';
    return 'other';
  }

  private getFolderByCategory(category: string): string {
    const folders: Record<string, string> = {
      image: 'uploads/images',
      document: 'uploads/documents',
      video: 'uploads/videos',
      other: 'uploads/files',
    };
    return folders[category] ?? 'uploads/files';
  }

  private formatBytes(bytes: number): string {
    if (bytes === 0) return '0 B';
    const k = 1024;
    const sizes = ['B', 'KB', 'MB', 'GB'];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    return `${parseFloat((bytes / Math.pow(k, i)).toFixed(2))} ${sizes[i]}`;
  }
}
```

---

## 6. Multipart Upload (สำหรับไฟล์ขนาดใหญ่)

```typescript
// src/storage/MultipartUploadService.ts

import { StorageService } from './StorageService';
import { S3Client, CreateMultipartUploadCommand, UploadPartCommand, CompleteMultipartUploadCommand, AbortMultipartUploadCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import Redis from 'ioredis';

export interface MultipartUploadSession {
  uploadId: string;
  s3UploadId: string;
  key: string;
  userId: string;
  totalParts: number;
  partSize: number;
  totalSize: number;
  completedParts: Array<{ partNumber: number; etag: string }>;
  expiresAt: Date;
}

// S3 multipart requirements:
// - Part size: minimum 5MB (except last part)
// - Maximum parts: 10,000
// - Maximum object size: 5TB
const MIN_PART_SIZE = 5 * 1024 * 1024;   // 5 MB
const MAX_PARTS = 10000;

export class MultipartUploadService {
  private s3Client: S3Client;
  private redis: Redis;
  private bucket: string;

  constructor(s3Client: S3Client, redis: Redis, bucket: string) {
    this.s3Client = s3Client;
    this.redis = redis;
    this.bucket = bucket;
  }

  /**
   * เริ่ม multipart upload
   */
  async initiate(
    userId: string,
    key: string,
    totalSize: number,
    contentType: string
  ): Promise<{
    uploadId: string;
    partUrls: Array<{ partNumber: number; url: string }>;
    partSize: number;
    totalParts: number;
  }> {
    // คำนวณ part size
    const partSize = Math.max(MIN_PART_SIZE, Math.ceil(totalSize / MAX_PARTS));
    const totalParts = Math.ceil(totalSize / partSize);

    // สร้าง multipart upload ใน S3
    const createCommand = new CreateMultipartUploadCommand({
      Bucket: this.bucket,
      Key: key,
      ContentType: contentType,
      Metadata: { 'user-id': userId },
    });

    const response = await this.s3Client.send(createCommand);
    const s3UploadId = response.UploadId!;

    // สร้าง presigned URLs สำหรับแต่ละ part
    const partUrlPromises = Array.from({ length: totalParts }, (_, i) => {
      const partNumber = i + 1;
      const command = new UploadPartCommand({
        Bucket: this.bucket,
        Key: key,
        UploadId: s3UploadId,
        PartNumber: partNumber,
      });
      
      return getSignedUrl(this.s3Client, command, { expiresIn: 3600 })
        .then(url => ({ partNumber, url }));
    });

    const partUrls = await Promise.all(partUrlPromises);

    // บันทึก session ใน Redis
    const uploadId = `mpu_${Date.now()}_${Math.random().toString(36).slice(2)}`;
    const session: MultipartUploadSession = {
      uploadId,
      s3UploadId,
      key,
      userId,
      totalParts,
      partSize,
      totalSize,
      completedParts: [],
      expiresAt: new Date(Date.now() + 24 * 3600 * 1000),  // 24 hours
    };

    await this.redis.setex(
      `multipart:${uploadId}`,
      24 * 3600,
      JSON.stringify(session)
    );

    return { uploadId, partUrls, partSize, totalParts };
  }

  /**
   * รายงาน part ที่ upload เสร็จ
   */
  async reportPartComplete(
    uploadId: string,
    userId: string,
    partNumber: number,
    etag: string
  ): Promise<{ completedParts: number; totalParts: number }> {
    const session = await this.getSession(uploadId, userId);

    // ตรวจ duplicate
    const alreadyCompleted = session.completedParts.some(p => p.partNumber === partNumber);
    if (!alreadyCompleted) {
      session.completedParts.push({ partNumber, etag });
    }

    // อัพเดท session
    await this.redis.setex(
      `multipart:${uploadId}`,
      24 * 3600,
      JSON.stringify(session)
    );

    return {
      completedParts: session.completedParts.length,
      totalParts: session.totalParts,
    };
  }

  /**
   * Complete multipart upload
   */
  async complete(uploadId: string, userId: string): Promise<{ key: string; etag: string }> {
    const session = await this.getSession(uploadId, userId);

    if (session.completedParts.length !== session.totalParts) {
      throw new Error(
        `Incomplete: ${session.completedParts.length}/${session.totalParts} parts completed`
      );
    }

    // Sort parts by number
    const sortedParts = [...session.completedParts].sort((a, b) => a.partNumber - b.partNumber);

    // Complete multipart upload
    const command = new CompleteMultipartUploadCommand({
      Bucket: this.bucket,
      Key: session.key,
      UploadId: session.s3UploadId,
      MultipartUpload: {
        Parts: sortedParts.map(part => ({
          PartNumber: part.partNumber,
          ETag: part.etag,
        })),
      },
    });

    const response = await this.s3Client.send(command);

    // Cleanup Redis session
    await this.redis.del(`multipart:${uploadId}`);

    return {
      key: session.key,
      etag: response.ETag ?? '',
    };
  }

  /**
   * Abort multipart upload
   */
  async abort(uploadId: string, userId: string): Promise<void> {
    const session = await this.getSession(uploadId, userId);

    const command = new AbortMultipartUploadCommand({
      Bucket: this.bucket,
      Key: session.key,
      UploadId: session.s3UploadId,
    });

    await this.s3Client.send(command);
    await this.redis.del(`multipart:${uploadId}`);
    
    console.log(`[Multipart] Aborted upload ${uploadId}`);
  }

  private async getSession(uploadId: string, userId: string): Promise<MultipartUploadSession> {
    const data = await this.redis.get(`multipart:${uploadId}`);
    if (!data) throw new Error('Upload session not found or expired');
    
    const session: MultipartUploadSession = JSON.parse(data);
    if (session.userId !== userId) throw new Error('Access denied');
    
    return session;
  }
}
```

---

## 7. CORS Configuration

```typescript
// src/storage/corsConfig.ts

// MinIO CORS configuration
export const minioCORSConfig = {
  CORSRules: [
    {
      AllowedOrigins: [
        process.env.FRONTEND_URL ?? 'http://localhost:3001',
        'https://app.example.com',
      ],
      AllowedMethods: ['GET', 'PUT', 'POST', 'DELETE', 'HEAD'],
      AllowedHeaders: [
        'Content-Type',
        'Content-Length',
        'Authorization',
        'x-amz-date',
        'x-amz-content-sha256',
        'x-amz-security-token',
        'x-amz-user-agent',
      ],
      ExposeHeaders: [
        'ETag',
        'x-amz-request-id',
      ],
      MaxAgeSeconds: 3600,
    },
  ],
};

// Apply CORS to MinIO bucket using mc (MinIO CLI)
/*
mc alias set myminio http://localhost:9000 minioadmin minioadmin

# Set CORS policy
cat > /tmp/cors.json << 'EOF'
{
  "CORSRules": [
    {
      "AllowedOrigins": ["http://localhost:3001", "https://app.example.com"],
      "AllowedMethods": ["GET", "PUT", "HEAD"],
      "AllowedHeaders": ["*"],
      "ExposeHeaders": ["ETag"],
      "MaxAgeSeconds": 3600
    }
  ]
}
EOF

mc cors set --config /tmp/cors.json myminio/uploads
*/

// Express CORS for API routes
import cors from 'cors';

export const corsOptions = cors({
  origin: (origin, callback) => {
    const allowedOrigins = [
      process.env.FRONTEND_URL ?? 'http://localhost:3001',
      'https://app.example.com',
    ];
    
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Upload-Id'],
  credentials: true,
});
```

---

## 8. Express Routes

```typescript
// src/storage/routes/fileRoutes.ts

import { Router, Request, Response } from 'express';
import { FileUploadService } from '../FileUploadService';
import { MultipartUploadService } from '../MultipartUploadService';

export function createFileRoutes(
  fileUploadService: FileUploadService,
  multipartService: MultipartUploadService
): Router {
  const router = Router();

  /**
   * POST /files/upload-url
   * Request presigned upload URL
   */
  router.post('/upload-url', async (req: Request, res: Response) => {
    try {
      const user = (req as any).user;
      const { filename, contentType, fileSize, folder } = req.body;

      if (!filename || !contentType || !fileSize) {
        return res.status(400).json({
          error: 'Missing required fields: filename, contentType, fileSize',
        });
      }

      const result = await fileUploadService.requestUploadUrl({
        userId: user.id,
        filename,
        contentType,
        fileSize: parseInt(fileSize),
        folder,
      });

      res.json(result);
    } catch (error: any) {
      res.status(400).json({ error: error.message });
    }
  });

  /**
   * POST /files/complete/:uploadId
   * Complete upload after client uploads to S3
   */
  router.post('/complete/:uploadId', async (req: Request, res: Response) => {
    try {
      const user = (req as any).user;
      const { uploadId } = req.params;
      const { etag } = req.body;

      const fileRecord = await fileUploadService.completeUpload(
        { uploadId, etag },
        user.id
      );

      res.json({
        message: 'Upload completed successfully',
        file: fileRecord,
      });
    } catch (error: any) {
      res.status(400).json({ error: error.message });
    }
  });

  /**
   * GET /files
   * List user's files
   */
  router.get('/', async (req: Request, res: Response) => {
    try {
      const user = (req as any).user;
      const { limit = '20', offset = '0', contentType } = req.query;

      const result = await fileUploadService.listUserFiles(user.id, {
        limit: parseInt(limit as string),
        offset: parseInt(offset as string),
        contentType: contentType as string,
      });

      res.json(result);
    } catch (error: any) {
      res.status(500).json({ error: error.message });
    }
  });

  /**
   * GET /files/:fileId/download
   * Get download URL
   */
  router.get('/:fileId/download', async (req: Request, res: Response) => {
    try {
      const user = (req as any).user;
      const { fileId } = req.params;
      const { expires = '3600', download } = req.query;

      const url = await fileUploadService.getDownloadUrl(fileId, user.id, {
        expiresIn: parseInt(expires as string),
        forceDownload: download === 'true',
      });

      // ส่ง URL กลับ หรือ redirect โดยตรง
      if (req.query.redirect === 'true') {
        return res.redirect(302, url);
      }

      res.json({
        url,
        expiresIn: parseInt(expires as string),
      });
    } catch (error: any) {
      res.status(404).json({ error: error.message });
    }
  });

  /**
   * DELETE /files/:fileId
   * Delete file
   */
  router.delete('/:fileId', async (req: Request, res: Response) => {
    try {
      const user = (req as any).user;
      await fileUploadService.deleteFile(req.params.fileId, user.id);
      res.json({ message: 'File deleted successfully' });
    } catch (error: any) {
      res.status(400).json({ error: error.message });
    }
  });

  /**
   * POST /files/multipart/init
   * Initialize multipart upload for large files
   */
  router.post('/multipart/init', async (req: Request, res: Response) => {
    try {
      const user = (req as any).user;
      const { filename, contentType, fileSize, folder } = req.body;

      const key = `uploads/${folder ?? 'files'}/user-${user.id}/${Date.now()}-${filename}`;

      const result = await multipartService.initiate(
        user.id,
        key,
        parseInt(fileSize),
        contentType
      );

      res.json(result);
    } catch (error: any) {
      res.status(400).json({ error: error.message });
    }
  });

  /**
   * POST /files/multipart/:uploadId/part/:partNumber
   * Report part completion
   */
  router.post('/multipart/:uploadId/part/:partNumber', async (req: Request, res: Response) => {
    try {
      const user = (req as any).user;
      const { uploadId, partNumber } = req.params;
      const { etag } = req.body;

      const result = await multipartService.reportPartComplete(
        uploadId,
        user.id,
        parseInt(partNumber),
        etag
      );

      res.json(result);
    } catch (error: any) {
      res.status(400).json({ error: error.message });
    }
  });

  /**
   * POST /files/multipart/:uploadId/complete
   * Complete multipart upload
   */
  router.post('/multipart/:uploadId/complete', async (req: Request, res: Response) => {
    try {
      const user = (req as any).user;
      const result = await multipartService.complete(req.params.uploadId, user.id);
      res.json({ message: 'Multipart upload completed', ...result });
    } catch (error: any) {
      res.status(400).json({ error: error.message });
    }
  });

  return router;
}
```

---

## 9. Frontend Implementation

```typescript
// Frontend: Direct Upload to S3/MinIO

interface UploadProgress {
  loaded: number;
  total: number;
  percentage: number;
}

class S3DirectUploader {
  private apiBaseUrl: string;

  constructor(apiBaseUrl: string) {
    this.apiBaseUrl = apiBaseUrl;
  }

  /**
   * Upload ไฟล์โดยตรงไปยัง S3
   */
  async uploadFile(
    file: File,
    folder?: string,
    onProgress?: (progress: UploadProgress) => void
  ): Promise<{ fileId: string; filename: string; size: number }> {
    // Step 1: ขอ presigned URL จาก API
    const uploadUrlResponse = await fetch(`${this.apiBaseUrl}/files/upload-url`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.getToken()}`,
      },
      body: JSON.stringify({
        filename: file.name,
        contentType: file.type,
        fileSize: file.size,
        folder,
      }),
    });

    if (!uploadUrlResponse.ok) {
      const error = await uploadUrlResponse.json();
      throw new Error(error.message || 'Failed to get upload URL');
    }

    const { uploadId, uploadUrl, key, expiresAt } = await uploadUrlResponse.json();

    // Step 2: Upload ไฟล์โดยตรงไปยัง S3 (ไม่ผ่าน API Server!)
    await this.uploadToS3(uploadUrl, file, onProgress);

    // Step 3: แจ้ง API ว่า upload เสร็จแล้ว
    const completeResponse = await fetch(
      `${this.apiBaseUrl}/files/complete/${uploadId}`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${this.getToken()}`,
        },
      }
    );

    if (!completeResponse.ok) {
      const error = await completeResponse.json();
      throw new Error(error.message || 'Failed to complete upload');
    }

    const { file: fileRecord } = await completeResponse.json();
    return fileRecord;
  }

  /**
   * Upload ไปยัง S3 โดยตรงพร้อม progress tracking
   */
  private uploadToS3(
    presignedUrl: string,
    file: File,
    onProgress?: (progress: UploadProgress) => void
  ): Promise<void> {
    return new Promise((resolve, reject) => {
      const xhr = new XMLHttpRequest();

      xhr.upload.addEventListener('progress', (event) => {
        if (event.lengthComputable && onProgress) {
          onProgress({
            loaded: event.loaded,
            total: event.total,
            percentage: Math.round((event.loaded / event.total) * 100),
          });
        }
      });

      xhr.addEventListener('load', () => {
        if (xhr.status >= 200 && xhr.status < 300) {
          resolve();
        } else {
          reject(new Error(`Upload failed: ${xhr.status} ${xhr.statusText}`));
        }
      });

      xhr.addEventListener('error', () => reject(new Error('Network error during upload')));
      xhr.addEventListener('abort', () => reject(new Error('Upload aborted')));

      xhr.open('PUT', presignedUrl);
      xhr.setRequestHeader('Content-Type', file.type);
      xhr.send(file);
    });
  }

  /**
   * Upload ไฟล์ขนาดใหญ่ด้วย multipart
   */
  async uploadLargeFile(
    file: File,
    onProgress?: (progress: UploadProgress) => void
  ): Promise<{ key: string; etag: string }> {
    // Init multipart upload
    const initResponse = await fetch(`${this.apiBaseUrl}/files/multipart/init`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.getToken()}`,
      },
      body: JSON.stringify({
        filename: file.name,
        contentType: file.type,
        fileSize: file.size,
      }),
    });

    const { uploadId, partUrls, partSize, totalParts } = await initResponse.json();

    // Upload parts
    let uploadedParts = 0;
    const etags: Array<{ partNumber: number; etag: string }> = [];

    for (const { partNumber, url } of partUrls) {
      const start = (partNumber - 1) * partSize;
      const end = Math.min(start + partSize, file.size);
      const blob = file.slice(start, end);

      // Upload part
      const partResponse = await fetch(url, {
        method: 'PUT',
        body: blob,
      });

      if (!partResponse.ok) {
        throw new Error(`Part ${partNumber} upload failed`);
      }

      const etag = partResponse.headers.get('ETag') ?? '';
      etags.push({ partNumber, etag });

      // Report completion
      await fetch(`${this.apiBaseUrl}/files/multipart/${uploadId}/part/${partNumber}`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${this.getToken()}`,
        },
        body: JSON.stringify({ etag }),
      });

      uploadedParts++;

      if (onProgress) {
        onProgress({
          loaded: uploadedParts * partSize,
          total: file.size,
          percentage: Math.round((uploadedParts / totalParts) * 100),
        });
      }
    }

    // Complete multipart upload
    const completeResponse = await fetch(
      `${this.apiBaseUrl}/files/multipart/${uploadId}/complete`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${this.getToken()}`,
        },
      }
    );

    return completeResponse.json();
  }

  private getToken(): string {
    return localStorage.getItem('token') ?? '';
  }
}

// =====================================================
// Usage Example (React component)
// =====================================================

/*
function FileUploadComponent() {
  const uploader = new S3DirectUploader('http://localhost:3000/api');
  
  const handleFileChange = async (event: React.ChangeEvent<HTMLInputElement>) => {
    const file = event.target.files?.[0];
    if (!file) return;
    
    try {
      const result = await uploader.uploadFile(file, 'documents', (progress) => {
        console.log(`Upload progress: ${progress.percentage}%`);
        setUploadProgress(progress.percentage);
      });
      
      console.log('Upload complete:', result);
    } catch (error) {
      console.error('Upload failed:', error);
    }
  };
  
  return (
    <div>
      <input type="file" onChange={handleFileChange} />
      <progress value={uploadProgress} max={100} />
    </div>
  );
}
*/
```

---

## 10. Database Schema

```sql
-- Database schema สำหรับ file uploads

CREATE TABLE file_uploads (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  key TEXT NOT NULL,              -- S3/MinIO object key
  filename TEXT NOT NULL,
  content_type TEXT NOT NULL,
  size BIGINT NOT NULL,           -- Expected size
  actual_size BIGINT,             -- Actual size after upload
  etag TEXT,                      -- S3 ETag for deduplication
  status TEXT NOT NULL DEFAULT 'pending',  -- pending, completed, failed
  expires_at TIMESTAMPTZ,         -- Upload URL expiration
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_file_uploads_user_id ON file_uploads(user_id);
CREATE INDEX idx_file_uploads_status ON file_uploads(status);
CREATE INDEX idx_file_uploads_key ON file_uploads(key);

-- Cleanup pending uploads older than 24 hours (run as cron job)
-- DELETE FROM file_uploads WHERE status = 'pending' AND expires_at < NOW();
```

---

## 11. IAM Policy (Principle of Least Privilege)

```json
// IAM Policy สำหรับ Application User (ไม่ได้ให้ permission เกิน)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowUploadUserFiles",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:HeadObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-app-uploads/uploads/*"
      ]
    },
    {
      "Sid": "AllowListForCleanup",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::my-app-uploads",
      "Condition": {
        "StringLike": {
          "s3:prefix": "uploads/*"
        }
      }
    }
  ]
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Object Storage Access Control**:
   - Public vs Private buckets
   - Bucket policies ควบคุม access

2. **Presigned URLs**:
   - **GET**: Temporary download URLs
   - **PUT**: Client direct upload
   - Expiration time ป้องกัน misuse

3. **PUT Presigned URL Workflow**:
   - Client ขอ URL จาก API → API สร้าง signed URL → Client upload ตรงไปยัง S3 → Client แจ้ง API → API verify

4. **Security**:
   - Validate content type ก่อนและหลัง upload
   - ตรวจ file size
   - ใช้ unique keys ป้องกัน collision

5. **Multipart Upload**: สำหรับไฟล์ขนาดใหญ่ (>5MB parts)

6. **CORS Configuration**: ให้ browser upload ตรงได้

7. **IAM/MinIO Policy**: Principle of Least Privilege - ให้ permission เท่าที่จำเป็น

Presigned URLs เป็น pattern ที่ดีมากสำหรับ file upload เพราะลด bandwidth ของ API Server ลงอย่างมาก และทำให้ upload เร็วขึ้นเพราะ client คุยตรงกับ Object Storage
