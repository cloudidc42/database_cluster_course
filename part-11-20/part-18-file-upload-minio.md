# Part 18: File Upload ไปยัง MinIO/S3

## สารบัญ
1. [Multipart Form Data และ Multer](#1-multipart-form-data-และ-multer)
2. [File Validation](#2-file-validation)
3. [In-Memory vs Disk Storage](#3-in-memory-vs-disk-storage)
4. [Direct Upload to MinIO/S3](#4-direct-upload-to-minios3)
5. [Pre-signed URL Upload](#5-pre-signed-url-upload)
6. [File Naming Strategy](#6-file-naming-strategy)
7. [Image Processing ด้วย sharp](#7-image-processing-ด้วย-sharp)
8. [Video Upload](#8-video-upload)
9. [Document Upload](#9-document-upload)
10. [Progress Tracking](#10-progress-tracking)
11. [Chunked Upload](#11-chunked-upload)
12. [Error Handling](#12-error-handling)
13. [Store Metadata ใน PostgreSQL](#13-store-metadata-ใน-postgresql)
14. [Serve Files](#14-serve-files)
15. [Delete File Flow](#15-delete-file-flow)
16. [Full Example: Profile Picture Upload](#16-full-example-profile-picture-upload)

---

## 1. Multipart Form Data และ Multer

### ทำไมต้องใช้ multipart/form-data?

การอัพโหลดไฟล์ต้องใช้ `multipart/form-data` เพราะ JSON ไม่รองรับ binary data โดยตรง

```
Content-Type: multipart/form-data; boundary=----FormBoundary7MA4YWxkTrZu0gW

------FormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="name"

John Doe
------FormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="avatar"; filename="photo.jpg"
Content-Type: image/jpeg

<binary file data>
------FormBoundary7MA4YWxkTrZu0gW--
```

### ติดตั้ง

```bash
npm install multer @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
npm install sharp        # image processing
npm install uuid
npm install --save-dev @types/multer
```

### Multer พื้นฐาน

```typescript
import multer from 'multer';
import path from 'path';

// Memory storage (ไฟล์อยู่ใน RAM)
const memoryStorage = multer.memoryStorage();

// Disk storage (ไฟล์บันทึกลงดิสก์)
const diskStorage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, '/tmp/uploads');
  },
  filename: (req, file, cb) => {
    const uniqueName = `${Date.now()}-${Math.random().toString(36).slice(2)}`;
    const ext = path.extname(file.originalname);
    cb(null, `${uniqueName}${ext}`);
  },
});
```

---

## 2. File Validation

```typescript
// src/config/upload.ts
import multer from 'multer';
import path from 'path';

// Allowed MIME types
export const ALLOWED_MIME_TYPES = {
  image: ['image/jpeg', 'image/png', 'image/gif', 'image/webp'],
  document: ['application/pdf', 'application/msword',
    'application/vnd.openxmlformats-officedocument.wordprocessingml.document'],
  video: ['video/mp4', 'video/quicktime', 'video/x-msvideo'],
  audio: ['audio/mpeg', 'audio/wav', 'audio/ogg'],
  spreadsheet: ['application/vnd.ms-excel',
    'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'],
} as const;

export const FILE_SIZE_LIMITS = {
  avatar: 5 * 1024 * 1024,       // 5 MB
  document: 50 * 1024 * 1024,    // 50 MB
  video: 500 * 1024 * 1024,      // 500 MB
  audio: 100 * 1024 * 1024,      // 100 MB
};

// Custom file filter
export function createFileFilter(allowedTypes: string[]) {
  return (
    req: Express.Request,
    file: Express.Multer.File,
    cb: multer.FileFilterCallback
  ) => {
    if (allowedTypes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error(`File type ${file.mimetype} ไม่ได้รับอนุญาต`));
    }
  };
}

// Validate file extension ด้วย magic bytes (ปลอดภัยกว่า MIME type)
export async function validateFileMagicBytes(
  buffer: Buffer,
  expectedType: string
): Promise<boolean> {
  const signatures: Record<string, Buffer[]> = {
    'image/jpeg': [Buffer.from([0xff, 0xd8, 0xff])],
    'image/png': [Buffer.from([0x89, 0x50, 0x4e, 0x47])],
    'image/gif': [Buffer.from('GIF87a'), Buffer.from('GIF89a')],
    'image/webp': [Buffer.from('RIFF')], // ตรวจสอบเพิ่มเติม
    'application/pdf': [Buffer.from('%PDF')],
    'video/mp4': [Buffer.from([0x00, 0x00, 0x00, 0x18, 0x66, 0x74, 0x79, 0x70])],
  };

  const sigs = signatures[expectedType];
  if (!sigs) return true; // ถ้าไม่รู้จัก type ให้ผ่าน

  return sigs.some((sig) => buffer.subarray(0, sig.length).equals(sig));
}

// Multer สำหรับ avatar upload
export const avatarUpload = multer({
  storage: multer.memoryStorage(),
  limits: {
    fileSize: FILE_SIZE_LIMITS.avatar,
    files: 1,
  },
  fileFilter: createFileFilter(ALLOWED_MIME_TYPES.image),
});

// Multer สำหรับ document upload
export const documentUpload = multer({
  storage: multer.memoryStorage(),
  limits: {
    fileSize: FILE_SIZE_LIMITS.document,
    files: 5, // อัพโหลดพร้อมกันได้สูงสุด 5 ไฟล์
  },
  fileFilter: createFileFilter([
    ...ALLOWED_MIME_TYPES.image,
    ...ALLOWED_MIME_TYPES.document,
  ]),
});
```

---

## 3. In-Memory vs Disk Storage

### In-Memory Storage

```
장점:
✓ เร็วกว่า (ไม่ต้อง disk I/O)
✓ ง่ายต่อ processing
✓ ไม่ต้อง cleanup temp files

ข้อเสีย:
✗ จำกัดด้วย RAM
✗ ไม่เหมาะสำหรับไฟล์ใหญ่ (>50MB)
✗ ข้อมูลหายถ้า server crash ก่อน upload เสร็จ
```

```typescript
// In-memory: เหมาะสำหรับไฟล์เล็ก เช่น avatar
const uploadInMemory = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: 5 * 1024 * 1024 },
});

router.post('/avatar', uploadInMemory.single('file'), async (req, res) => {
  const file = req.file!;
  // file.buffer = Buffer ของไฟล์
  // file.originalname = ชื่อไฟล์ต้นฉบับ
  // file.mimetype = MIME type
  // file.size = ขนาดไฟล์ (bytes)

  await uploadToMinio(file.buffer, file.originalname, file.mimetype);
  res.json({ success: true });
});
```

### Disk Storage

```typescript
// Disk: เหมาะสำหรับไฟล์ใหญ่
const uploadToDisk = multer({
  storage: multer.diskStorage({
    destination: '/tmp/uploads',
    filename: (req, file, cb) => {
      cb(null, `${Date.now()}-${file.originalname}`);
    },
  }),
  limits: { fileSize: 500 * 1024 * 1024 }, // 500MB
});

router.post('/video', uploadToDisk.single('video'), async (req, res) => {
  const file = req.file!;
  // file.path = path ของไฟล์บนดิสก์
  // file.filename = ชื่อไฟล์ที่บันทึก

  try {
    // Stream จากดิสก์ไปยัง MinIO (ไม่โหลดทั้งไฟล์เข้า RAM)
    await uploadFileToMinio(file.path, file.filename, file.mimetype);
    res.json({ success: true });
  } finally {
    // ลบ temp file เสมอ
    await fs.unlink(file.path).catch(console.error);
  }
});
```

---

## 4. Direct Upload to MinIO/S3

### MinIO Client Setup

```typescript
// src/config/minio.ts
import { S3Client, PutObjectCommand, GetObjectCommand,
         DeleteObjectCommand, HeadObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

export const s3Client = new S3Client({
  endpoint: process.env.MINIO_ENDPOINT || 'http://localhost:9000',
  region: process.env.MINIO_REGION || 'us-east-1',
  credentials: {
    accessKeyId: process.env.MINIO_ACCESS_KEY || 'minioadmin',
    secretAccessKey: process.env.MINIO_SECRET_KEY || 'minioadmin',
  },
  forcePathStyle: true, // สำคัญสำหรับ MinIO!
});

export const BUCKETS = {
  AVATARS: 'avatars',
  DOCUMENTS: 'documents',
  VIDEOS: 'videos',
  PUBLIC: 'public',
} as const;

// สร้าง bucket ถ้ายังไม่มี
export async function ensureBucketExists(bucket: string): Promise<void> {
  const { CreateBucketCommand, HeadBucketCommand } = await import('@aws-sdk/client-s3');

  try {
    await s3Client.send(new HeadBucketCommand({ Bucket: bucket }));
  } catch {
    await s3Client.send(new CreateBucketCommand({ Bucket: bucket }));
    console.log(`Created bucket: ${bucket}`);
  }
}
```

### Upload Function

```typescript
// src/services/storageService.ts
import { s3Client, BUCKETS } from '../config/minio';
import {
  PutObjectCommand,
  DeleteObjectCommand,
  GetObjectCommand,
  HeadObjectCommand,
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { Readable } from 'stream';

export interface UploadOptions {
  bucket: string;
  key: string;
  body: Buffer | Readable;
  contentType: string;
  metadata?: Record<string, string>;
  cacheControl?: string;
}

export interface UploadResult {
  bucket: string;
  key: string;
  url: string;
  size: number;
  etag: string;
}

export class StorageService {
  // Upload ไฟล์
  async upload(options: UploadOptions): Promise<UploadResult> {
    const { bucket, key, body, contentType, metadata, cacheControl } = options;

    const command = new PutObjectCommand({
      Bucket: bucket,
      Key: key,
      Body: body,
      ContentType: contentType,
      Metadata: metadata,
      CacheControl: cacheControl || 'max-age=31536000', // 1 year
    });

    const result = await s3Client.send(command);

    const url = `${process.env.MINIO_PUBLIC_URL}/${bucket}/${key}`;

    return {
      bucket,
      key,
      url,
      size: Buffer.isBuffer(body) ? body.length : 0,
      etag: result.ETag || '',
    };
  }

  // Download ไฟล์
  async download(bucket: string, key: string): Promise<Buffer> {
    const command = new GetObjectCommand({ Bucket: bucket, Key: key });
    const response = await s3Client.send(command);

    if (!response.Body) throw new Error('File not found');

    const chunks: Buffer[] = [];
    for await (const chunk of response.Body as Readable) {
      chunks.push(Buffer.from(chunk));
    }

    return Buffer.concat(chunks);
  }

  // ลบไฟล์
  async delete(bucket: string, key: string): Promise<void> {
    await s3Client.send(new DeleteObjectCommand({ Bucket: bucket, Key: key }));
  }

  // ตรวจสอบว่าไฟล์มีอยู่
  async exists(bucket: string, key: string): Promise<boolean> {
    try {
      await s3Client.send(new HeadObjectCommand({ Bucket: bucket, Key: key }));
      return true;
    } catch {
      return false;
    }
  }

  // สร้าง Pre-signed URL สำหรับ GET
  async getPresignedUrl(
    bucket: string,
    key: string,
    expiresIn: number = 3600
  ): Promise<string> {
    const command = new GetObjectCommand({ Bucket: bucket, Key: key });
    return getSignedUrl(s3Client, command, { expiresIn });
  }

  // สร้าง Pre-signed URL สำหรับ PUT
  async getPutPresignedUrl(
    bucket: string,
    key: string,
    contentType: string,
    expiresIn: number = 600
  ): Promise<string> {
    const command = new PutObjectCommand({
      Bucket: bucket,
      Key: key,
      ContentType: contentType,
    });
    return getSignedUrl(s3Client, command, { expiresIn });
  }
}

export const storageService = new StorageService();
```

---

## 5. Pre-signed URL Upload

### ทำไมต้องใช้ Pre-signed URL?

```
วิธีปกติ: Client → API → MinIO
ข้อเสีย: ไฟล์ผ่าน server สองครั้ง (double bandwidth)

Pre-signed URL: Client → MinIO โดยตรง
ข้อดี: ประหยัด bandwidth ของ server, เร็วกว่า

Flow:
1. Client ขอ pre-signed URL จาก API
2. API สร้าง URL ที่มีลายเซ็น (valid 10 นาที)
3. Client upload ตรงไปยัง MinIO ด้วย URL นั้น
4. Client แจ้ง API ว่า upload สำเร็จ
5. API บันทึก metadata ลง DB
```

```typescript
// src/routes/upload.ts
import { Router, Request, Response, NextFunction } from 'express';
import { z } from 'zod';
import { v4 as uuidv4 } from 'uuid';
import { storageService, BUCKETS } from '../config/minio';
import { authenticate } from '../middleware/auth';

const router = Router();

// ขอ pre-signed URL
const presignedSchema = z.object({
  filename: z.string().min(1).max(255),
  contentType: z.string().min(1),
  fileSize: z.number().min(1).max(100 * 1024 * 1024), // max 100MB
  bucket: z.enum(['documents', 'public']),
});

router.post('/presigned', authenticate, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { filename, contentType, fileSize, bucket } = presignedSchema.parse(req.body);

    // Validate content type
    const allowedTypes: Record<string, string[]> = {
      documents: ['application/pdf', 'image/jpeg', 'image/png'],
      public: ['image/jpeg', 'image/png', 'image/webp'],
    };

    if (!allowedTypes[bucket]?.includes(contentType)) {
      return res.status(400).json({
        success: false,
        error: { code: 'INVALID_FILE_TYPE', message: `ไม่รองรับไฟล์ประเภท ${contentType}` },
      });
    }

    // สร้าง key
    const ext = filename.split('.').pop();
    const key = `${req.user!.id}/${uuidv4()}.${ext}`;

    // สร้าง pre-signed URL (valid 10 นาที)
    const uploadUrl = await storageService.getPutPresignedUrl(
      bucket,
      key,
      contentType,
      600
    );

    // เก็บ "pending upload" ใน Redis (สำหรับ confirm ทีหลัง)
    await redis.set(
      `pending_upload:${key}`,
      JSON.stringify({ userId: req.user!.id, filename, contentType, fileSize, bucket }),
      'EX',
      600
    );

    res.json({
      success: true,
      data: {
        uploadUrl,
        key,
        expiresIn: 600,
        instructions: 'PUT ไฟล์ไปยัง uploadUrl พร้อม Content-Type header',
      },
    });
  } catch (error) {
    next(error);
  }
});

// Confirm upload สำเร็จ
router.post('/confirm', authenticate, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { key } = z.object({ key: z.string() }).parse(req.body);

    // ตรวจสอบ pending upload
    const pendingData = await redis.get(`pending_upload:${key}`);

    if (!pendingData) {
      return res.status(400).json({
        success: false,
        error: { code: 'NO_PENDING_UPLOAD', message: 'ไม่พบการ upload นี้ หรือหมดเวลาแล้ว' },
      });
    }

    const { userId, filename, contentType, fileSize, bucket } = JSON.parse(pendingData);

    if (userId !== req.user!.id) {
      return res.status(403).json({
        success: false,
        error: { code: 'FORBIDDEN' },
      });
    }

    // ตรวจสอบว่าไฟล์มีอยู่ใน MinIO จริง
    const exists = await storageService.exists(bucket, key);

    if (!exists) {
      return res.status(400).json({
        success: false,
        error: { code: 'FILE_NOT_FOUND', message: 'ไม่พบไฟล์ใน storage' },
      });
    }

    // บันทึก metadata ลง DB
    const fileRecord = await pool.query(
      `INSERT INTO files (user_id, bucket, key, original_name, mime_type, size, status)
       VALUES ($1, $2, $3, $4, $5, $6, 'active')
       RETURNING id`,
      [req.user!.id, bucket, key, filename, contentType, fileSize]
    );

    // ลบ pending upload
    await redis.del(`pending_upload:${key}`);

    const fileUrl = `${process.env.MINIO_PUBLIC_URL}/${bucket}/${key}`;

    res.json({
      success: true,
      data: {
        fileId: fileRecord.rows[0].id,
        url: fileUrl,
        filename,
        size: fileSize,
        contentType,
      },
    });
  } catch (error) {
    next(error);
  }
});

export default router;
```

### Client-side Upload

```typescript
// Frontend code
async function uploadFile(file: File): Promise<string> {
  // 1. ขอ pre-signed URL
  const presignedRes = await fetch('/api/upload/presigned', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      filename: file.name,
      contentType: file.type,
      fileSize: file.size,
      bucket: 'documents',
    }),
  });

  const { data } = await presignedRes.json();
  const { uploadUrl, key } = data;

  // 2. Upload ตรงไปยัง MinIO
  const uploadRes = await fetch(uploadUrl, {
    method: 'PUT',
    headers: { 'Content-Type': file.type },
    body: file,
  });

  if (!uploadRes.ok) {
    throw new Error('Upload failed');
  }

  // 3. Confirm กับ API
  const confirmRes = await fetch('/api/upload/confirm', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ key }),
  });

  const confirmData = await confirmRes.json();
  return confirmData.data.url;
}
```

---

## 6. File Naming Strategy

```typescript
// src/utils/fileNaming.ts
import { v4 as uuidv4 } from 'uuid';
import crypto from 'crypto';
import path from 'path';

// UUID-based naming (simple, unique)
export function generateUuidName(originalName: string): string {
  const ext = path.extname(originalName).toLowerCase();
  return `${uuidv4()}${ext}`;
}

// Hash-based naming (deduplication)
export function generateHashName(buffer: Buffer, originalName: string): string {
  const hash = crypto.createHash('sha256').update(buffer).digest('hex');
  const ext = path.extname(originalName).toLowerCase();
  return `${hash}${ext}`;
}

// Structured path naming (ค้นหาง่ายกว่า)
export function generateStructuredPath(
  userId: string,
  type: string,
  originalName: string
): string {
  const date = new Date();
  const year = date.getFullYear();
  const month = String(date.getMonth() + 1).padStart(2, '0');
  const day = String(date.getDate()).padStart(2, '0');

  const ext = path.extname(originalName).toLowerCase();
  const uuid = uuidv4();

  // userId/type/YYYY/MM/DD/uuid.ext
  return `${userId}/${type}/${year}/${month}/${day}/${uuid}${ext}`;
}

// Sanitize filename (ลบ special chars)
export function sanitizeFilename(filename: string): string {
  return filename
    .replace(/[^a-zA-Z0-9.-_]/g, '_')
    .replace(/_{2,}/g, '_')
    .toLowerCase()
    .slice(0, 255);
}

// ตัวอย่าง paths:
// UUID:       550e8400-e29b-41d4-a716-446655440000.jpg
// Hash:       a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a27ae3.jpg
// Structured: user_123/avatar/2024/01/15/550e8400.jpg
```

---

## 7. Image Processing ด้วย sharp

```typescript
// src/services/imageService.ts
import sharp from 'sharp';
import path from 'path';

export interface ResizeOptions {
  width?: number;
  height?: number;
  fit?: 'cover' | 'contain' | 'fill' | 'inside' | 'outside';
  quality?: number;
}

export class ImageService {
  // Resize และ optimize
  async resize(
    input: Buffer,
    options: ResizeOptions
  ): Promise<Buffer> {
    const { width, height, fit = 'cover', quality = 85 } = options;

    return sharp(input)
      .resize(width, height, { fit })
      .jpeg({ quality, progressive: true })
      .toBuffer();
  }

  // แปลงเป็น WebP (ขนาดเล็กกว่า JPEG 25-35%)
  async toWebP(input: Buffer, quality: number = 85): Promise<Buffer> {
    return sharp(input)
      .webp({ quality })
      .toBuffer();
  }

  // สร้าง thumbnail
  async createThumbnail(input: Buffer, size: number = 200): Promise<Buffer> {
    return sharp(input)
      .resize(size, size, { fit: 'cover' })
      .jpeg({ quality: 75 })
      .toBuffer();
  }

  // ดึง metadata
  async getMetadata(input: Buffer): Promise<{
    width: number;
    height: number;
    format: string;
    size: number;
  }> {
    const metadata = await sharp(input).metadata();
    return {
      width: metadata.width || 0,
      height: metadata.height || 0,
      format: metadata.format || 'unknown',
      size: input.length,
    };
  }

  // Process avatar: resize หลายขนาด
  async processAvatar(input: Buffer): Promise<{
    original: Buffer;
    large: Buffer;    // 400x400
    medium: Buffer;   // 200x200
    thumbnail: Buffer; // 50x50
  }> {
    // ตรวจสอบว่าเป็นรูปภาพจริงหรือเปล่า
    const metadata = await sharp(input).metadata();

    if (!['jpeg', 'jpg', 'png', 'webp', 'gif'].includes(metadata.format || '')) {
      throw new Error('ไฟล์ไม่ใช่รูปภาพ');
    }

    const [original, large, medium, thumbnail] = await Promise.all([
      // Original: แค่ optimize ไม่ resize
      sharp(input).jpeg({ quality: 90 }).toBuffer(),

      // Large: 400x400
      sharp(input).resize(400, 400, { fit: 'cover' }).jpeg({ quality: 85 }).toBuffer(),

      // Medium: 200x200
      sharp(input).resize(200, 200, { fit: 'cover' }).jpeg({ quality: 80 }).toBuffer(),

      // Thumbnail: 50x50
      sharp(input).resize(50, 50, { fit: 'cover' }).jpeg({ quality: 75 }).toBuffer(),
    ]);

    return { original, large, medium, thumbnail };
  }

  // Strip EXIF metadata (ความเป็นส่วนตัว)
  async stripExif(input: Buffer): Promise<Buffer> {
    return sharp(input)
      .rotate() // Auto-rotate based on EXIF, then strip EXIF
      .toBuffer();
  }
}

export const imageService = new ImageService();
```

---

## 8. Video Upload

```typescript
// src/services/videoService.ts
import { exec } from 'child_process';
import { promisify } from 'util';
import fs from 'fs/promises';
import path from 'path';
import { v4 as uuidv4 } from 'uuid';

const execAsync = promisify(exec);

export interface VideoMetadata {
  duration: number;
  width: number;
  height: number;
  codec: string;
  bitrate: number;
  fps: number;
  size: number;
}

export class VideoService {
  // ดึง metadata ด้วย ffprobe
  async getMetadata(filePath: string): Promise<VideoMetadata> {
    const command = `ffprobe -v quiet -print_format json -show_streams -show_format "${filePath}"`;

    const { stdout } = await execAsync(command);
    const data = JSON.parse(stdout);

    const videoStream = data.streams.find((s: any) => s.codec_type === 'video');
    const format = data.format;

    return {
      duration: parseFloat(format.duration),
      width: videoStream?.width || 0,
      height: videoStream?.height || 0,
      codec: videoStream?.codec_name || 'unknown',
      bitrate: parseInt(format.bit_rate) || 0,
      fps: eval(videoStream?.r_frame_rate || '0') || 0,
      size: parseInt(format.size),
    };
  }

  // สร้าง thumbnail จากวิดีโอ
  async generateThumbnail(
    videoPath: string,
    timestamp: number = 5
  ): Promise<Buffer> {
    const outputPath = `/tmp/${uuidv4()}.jpg`;

    try {
      await execAsync(
        `ffmpeg -ss ${timestamp} -i "${videoPath}" -vframes 1 -q:v 2 "${outputPath}"`
      );

      return await fs.readFile(outputPath);
    } finally {
      await fs.unlink(outputPath).catch(() => {});
    }
  }

  // แปลงวิดีโอ (transcode)
  async transcode(
    inputPath: string,
    outputFormat: 'mp4' | 'webm' = 'mp4',
    quality: 'low' | 'medium' | 'high' = 'medium'
  ): Promise<string> {
    const outputPath = `/tmp/${uuidv4()}.${outputFormat}`;

    const qualitySettings = {
      low: '-crf 28 -preset fast',
      medium: '-crf 23 -preset medium',
      high: '-crf 18 -preset slow',
    };

    const settings = qualitySettings[quality];

    if (outputFormat === 'mp4') {
      await execAsync(
        `ffmpeg -i "${inputPath}" ${settings} -c:v libx264 -c:a aac "${outputPath}"`
      );
    } else {
      await execAsync(
        `ffmpeg -i "${inputPath}" ${settings} -c:v libvpx-vp9 -c:a libopus "${outputPath}"`
      );
    }

    return outputPath;
  }
}

export const videoService = new VideoService();

// Video upload route
router.post('/video', authenticate, videoUpload.single('video'), async (req, res, next) => {
  const filePath = req.file?.path;

  try {
    if (!req.file || !filePath) {
      return res.status(400).json({ success: false, error: { code: 'NO_FILE' } });
    }

    // ดึง metadata
    const metadata = await videoService.getMetadata(filePath);

    // Validate duration (ไม่เกิน 10 นาที)
    if (metadata.duration > 600) {
      return res.status(400).json({
        success: false,
        error: { code: 'VIDEO_TOO_LONG', message: 'วิดีโอต้องไม่เกิน 10 นาที' },
      });
    }

    // สร้าง thumbnail
    const thumbnail = await videoService.generateThumbnail(filePath);

    const userId = req.user!.id;
    const videoKey = generateStructuredPath(userId, 'videos', req.file.originalname);
    const thumbKey = videoKey.replace(/\.[^.]+$/, '-thumb.jpg');

    // Upload ทั้ง video และ thumbnail พร้อมกัน
    const [videoResult, thumbResult] = await Promise.all([
      storageService.upload({
        bucket: 'videos',
        key: videoKey,
        body: await fs.readFile(filePath),
        contentType: req.file.mimetype,
      }),
      storageService.upload({
        bucket: 'videos',
        key: thumbKey,
        body: thumbnail,
        contentType: 'image/jpeg',
      }),
    ]);

    // บันทึก metadata ลง DB
    const record = await pool.query(
      `INSERT INTO files (user_id, bucket, key, original_name, mime_type, size, metadata)
       VALUES ($1, $2, $3, $4, $5, $6, $7)
       RETURNING id`,
      [
        userId,
        'videos',
        videoKey,
        req.file.originalname,
        req.file.mimetype,
        req.file.size,
        JSON.stringify({ ...metadata, thumbnailKey: thumbKey }),
      ]
    );

    res.json({
      success: true,
      data: {
        fileId: record.rows[0].id,
        videoUrl: videoResult.url,
        thumbnailUrl: thumbResult.url,
        metadata,
      },
    });
  } catch (error) {
    next(error);
  } finally {
    if (filePath) {
      await fs.unlink(filePath).catch(() => {});
    }
  }
});
```

---

## 9. Document Upload

```typescript
// src/services/documentService.ts

// PDF text extraction
export async function extractPdfText(buffer: Buffer): Promise<string> {
  // ใช้ pdfjs หรือ pdf-parse
  // npm install pdf-parse
  const pdfParse = await import('pdf-parse');
  const data = await pdfParse.default(buffer);
  return data.text;
}

// Document upload route
router.post(
  '/document',
  authenticate,
  documentUpload.single('document'),
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      if (!req.file) {
        return res.status(400).json({ success: false, error: { code: 'NO_FILE' } });
      }

      // Validate magic bytes
      const isValid = await validateFileMagicBytes(req.file.buffer, req.file.mimetype);

      if (!isValid) {
        return res.status(400).json({
          success: false,
          error: { code: 'INVALID_FILE', message: 'ไฟล์ไม่ถูกต้อง' },
        });
      }

      const userId = req.user!.id;
      const key = generateStructuredPath(userId, 'documents', req.file.originalname);

      // Extract text สำหรับ search (ถ้าเป็น PDF)
      let extractedText: string | undefined;
      if (req.file.mimetype === 'application/pdf') {
        try {
          extractedText = await extractPdfText(req.file.buffer);
        } catch {
          // ถ้า extract ไม่ได้ก็ไม่เป็นไร
        }
      }

      // Upload
      const result = await storageService.upload({
        bucket: BUCKETS.DOCUMENTS,
        key,
        body: req.file.buffer,
        contentType: req.file.mimetype,
        metadata: {
          'original-name': req.file.originalname,
          'user-id': userId,
        },
      });

      // บันทึกลง DB พร้อม full-text search
      const record = await pool.query(
        `INSERT INTO files (user_id, bucket, key, original_name, mime_type, size, search_text)
         VALUES ($1, $2, $3, $4, $5, $6, to_tsvector('english', $7))
         RETURNING id`,
        [
          userId,
          BUCKETS.DOCUMENTS,
          key,
          req.file.originalname,
          req.file.mimetype,
          req.file.size,
          extractedText || '',
        ]
      );

      res.status(201).json({
        success: true,
        data: {
          fileId: record.rows[0].id,
          filename: req.file.originalname,
          url: result.url,
          size: req.file.size,
          hasTextContent: !!extractedText,
        },
      });
    } catch (error) {
      next(error);
    }
  }
);
```

---

## 10. Progress Tracking

```typescript
// Server-Sent Events สำหรับ progress

router.get('/upload-progress/:uploadId', authenticate, (req: Request, res: Response) => {
  const { uploadId } = req.params;

  // Set SSE headers
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    Connection: 'keep-alive',
    'X-Accel-Buffering': 'no', // สำหรับ Nginx
  });

  // Subscribe to Redis pub/sub
  const subscriber = redis.duplicate();
  subscriber.subscribe(`upload_progress:${uploadId}`);

  subscriber.on('message', (channel, message) => {
    res.write(`data: ${message}\n\n`);
  });

  // Cleanup เมื่อ client disconnect
  req.on('close', () => {
    subscriber.unsubscribe();
    subscriber.quit();
  });
});

// ใน upload handler: publish progress
async function uploadWithProgress(
  uploadId: string,
  file: Buffer,
  key: string,
  bucket: string
): Promise<void> {
  const chunkSize = 1024 * 1024; // 1MB chunks
  let uploaded = 0;

  // Simulate progress (ในจริง track จาก S3 multipart upload)
  for (let i = 0; i < file.length; i += chunkSize) {
    uploaded = Math.min(i + chunkSize, file.length);
    const progress = Math.round((uploaded / file.length) * 100);

    await redis.publish(
      `upload_progress:${uploadId}`,
      JSON.stringify({ progress, uploaded, total: file.length })
    );

    // Actual upload chunk (simplified)
    await new Promise((resolve) => setTimeout(resolve, 50));
  }

  // Upload จริง
  await storageService.upload({
    bucket,
    key,
    body: file,
    contentType: 'application/octet-stream',
  });

  // Progress 100%
  await redis.publish(
    `upload_progress:${uploadId}`,
    JSON.stringify({ progress: 100, done: true })
  );
}
```

---

## 11. Chunked Upload

```typescript
// สำหรับไฟล์ขนาดใหญ่มาก

// ตาราง DB สำหรับ chunk uploads
const createChunkUploadTable = `
  CREATE TABLE IF NOT EXISTS chunked_uploads (
    id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id       UUID REFERENCES users(id) NOT NULL,
    filename      VARCHAR(255) NOT NULL,
    total_size    BIGINT NOT NULL,
    chunk_size    INTEGER NOT NULL,
    total_chunks  INTEGER NOT NULL,
    mime_type     VARCHAR(100) NOT NULL,
    bucket        VARCHAR(100) NOT NULL,
    status        VARCHAR(50) DEFAULT 'pending',
    upload_id     VARCHAR(255),  -- MinIO multipart upload ID
    created_at    TIMESTAMPTZ DEFAULT NOW(),
    completed_at  TIMESTAMPTZ
  );

  CREATE TABLE IF NOT EXISTS upload_chunks (
    id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    upload_id    UUID REFERENCES chunked_uploads(id) ON DELETE CASCADE,
    chunk_index  INTEGER NOT NULL,
    size         INTEGER NOT NULL,
    etag         VARCHAR(255),
    uploaded_at  TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(upload_id, chunk_index)
  );
`;

// src/services/chunkedUploadService.ts
import {
  CreateMultipartUploadCommand,
  UploadPartCommand,
  CompleteMultipartUploadCommand,
  AbortMultipartUploadCommand,
} from '@aws-sdk/client-s3';

export class ChunkedUploadService {
  // เริ่มต้น multipart upload
  async initUpload(params: {
    userId: string;
    filename: string;
    totalSize: number;
    chunkSize: number;
    mimeType: string;
    bucket: string;
  }): Promise<{ uploadId: string; totalChunks: number }> {
    const { userId, filename, totalSize, chunkSize, mimeType, bucket } = params;
    const totalChunks = Math.ceil(totalSize / chunkSize);
    const key = generateStructuredPath(userId, 'files', filename);

    // เริ่ม MinIO multipart upload
    const createCmd = new CreateMultipartUploadCommand({
      Bucket: bucket,
      Key: key,
      ContentType: mimeType,
    });

    const s3Upload = await s3Client.send(createCmd);

    // บันทึกลง DB
    const record = await pool.query(
      `INSERT INTO chunked_uploads
       (user_id, filename, total_size, chunk_size, total_chunks, mime_type, bucket, upload_id)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
       RETURNING id`,
      [userId, filename, totalSize, chunkSize, totalChunks, mimeType, bucket, s3Upload.UploadId]
    );

    return {
      uploadId: record.rows[0].id,
      totalChunks,
    };
  }

  // Upload chunk
  async uploadChunk(params: {
    uploadId: string;
    chunkIndex: number;
    data: Buffer;
  }): Promise<{ etag: string }> {
    const { uploadId, chunkIndex, data } = params;

    // ดึงข้อมูล upload
    const upload = await pool.query(
      'SELECT * FROM chunked_uploads WHERE id = $1',
      [uploadId]
    );

    if (!upload.rows[0]) throw new Error('Upload not found');

    const { bucket, filename, upload_id: s3UploadId } = upload.rows[0];
    const key = generateStructuredPath(upload.rows[0].user_id, 'files', filename);

    // Upload chunk ไปยัง MinIO
    const uploadCmd = new UploadPartCommand({
      Bucket: bucket,
      Key: key,
      UploadId: s3UploadId,
      PartNumber: chunkIndex + 1, // Part numbers start from 1
      Body: data,
    });

    const result = await s3Client.send(uploadCmd);

    // บันทึก chunk
    await pool.query(
      `INSERT INTO upload_chunks (upload_id, chunk_index, size, etag)
       VALUES ($1, $2, $3, $4)
       ON CONFLICT (upload_id, chunk_index) DO UPDATE SET etag = $4`,
      [uploadId, chunkIndex, data.length, result.ETag]
    );

    return { etag: result.ETag! };
  }

  // Complete multipart upload
  async completeUpload(uploadId: string): Promise<string> {
    const upload = await pool.query(
      'SELECT * FROM chunked_uploads WHERE id = $1',
      [uploadId]
    );

    const uploadInfo = upload.rows[0];

    const chunks = await pool.query(
      'SELECT * FROM upload_chunks WHERE upload_id = $1 ORDER BY chunk_index',
      [uploadId]
    );

    if (chunks.rows.length !== uploadInfo.total_chunks) {
      throw new Error(`ยังอัพโหลดไม่ครบ: ${chunks.rows.length}/${uploadInfo.total_chunks} chunks`);
    }

    const key = generateStructuredPath(uploadInfo.user_id, 'files', uploadInfo.filename);

    const completeCmd = new CompleteMultipartUploadCommand({
      Bucket: uploadInfo.bucket,
      Key: key,
      UploadId: uploadInfo.upload_id,
      MultipartUpload: {
        Parts: chunks.rows.map((c) => ({
          PartNumber: c.chunk_index + 1,
          ETag: c.etag,
        })),
      },
    });

    await s3Client.send(completeCmd);

    await pool.query(
      'UPDATE chunked_uploads SET status = $1, completed_at = NOW() WHERE id = $2',
      ['completed', uploadId]
    );

    return `${process.env.MINIO_PUBLIC_URL}/${uploadInfo.bucket}/${key}`;
  }
}
```

---

## 12. Error Handling

```typescript
// Multer error handling
export function multerErrorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) {
  if (err instanceof multer.MulterError) {
    if (err.code === 'LIMIT_FILE_SIZE') {
      return res.status(400).json({
        success: false,
        error: {
          code: 'FILE_TOO_LARGE',
          message: `ไฟล์ใหญ่เกินขนาดที่กำหนด`,
        },
      });
    }

    if (err.code === 'LIMIT_FILE_COUNT') {
      return res.status(400).json({
        success: false,
        error: { code: 'TOO_MANY_FILES', message: 'อัพโหลดได้สูงสุด 5 ไฟล์' },
      });
    }

    if (err.code === 'LIMIT_UNEXPECTED_FILE') {
      return res.status(400).json({
        success: false,
        error: { code: 'UNEXPECTED_FILE_FIELD', message: `Field name ไม่ถูกต้อง: ${err.field}` },
      });
    }
  }

  if (err.message.includes('ไม่ได้รับอนุญาต')) {
    return res.status(400).json({
      success: false,
      error: { code: 'INVALID_FILE_TYPE', message: err.message },
    });
  }

  next(err);
}
```

---

## 13. Store Metadata ใน PostgreSQL

```sql
-- Files metadata table
CREATE TABLE files (
  id            UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
  user_id       UUID REFERENCES users(id) NOT NULL,

  -- Storage location
  bucket        VARCHAR(100) NOT NULL,
  key           VARCHAR(1000) NOT NULL,

  -- File info
  original_name VARCHAR(255) NOT NULL,
  mime_type     VARCHAR(100) NOT NULL,
  size          BIGINT NOT NULL,       -- bytes
  
  -- Image specific
  width         INTEGER,
  height        INTEGER,
  
  -- Search
  search_text   TSVECTOR,

  -- Status
  status        VARCHAR(50) DEFAULT 'active',

  -- Metadata (flexible)
  metadata      JSONB DEFAULT '{}',

  -- Timestamps
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  deleted_at    TIMESTAMPTZ
);

-- Index สำหรับ search
CREATE INDEX idx_files_user_id ON files(user_id);
CREATE INDEX idx_files_search ON files USING GIN(search_text);
CREATE INDEX idx_files_metadata ON files USING GIN(metadata);

-- Full-text search example
SELECT * FROM files
WHERE user_id = $1
  AND search_text @@ plainto_tsquery('english', $2);
```

---

## 14. Serve Files

```typescript
// 1. Direct URL (public files)
const publicUrl = `${process.env.MINIO_PUBLIC_URL}/public/${key}`;

// 2. Pre-signed URL (private files, expire หลัง N นาที)
router.get('/files/:fileId/download', authenticate, async (req, res, next) => {
  try {
    const file = await pool.query(
      'SELECT * FROM files WHERE id = $1 AND user_id = $2',
      [req.params.fileId, req.user!.id]
    );

    if (!file.rows[0]) {
      return res.status(404).json({ success: false, error: { code: 'NOT_FOUND' } });
    }

    const presignedUrl = await storageService.getPresignedUrl(
      file.rows[0].bucket,
      file.rows[0].key,
      3600 // 1 hour
    );

    res.json({ success: true, data: { downloadUrl: presignedUrl, expiresIn: 3600 } });
  } catch (error) {
    next(error);
  }
});

// 3. Proxy (ผ่าน server, ซ่อน MinIO URL)
router.get('/files/:fileId/view', authenticate, async (req, res, next) => {
  try {
    const file = await pool.query(
      'SELECT * FROM files WHERE id = $1 AND user_id = $2',
      [req.params.fileId, req.user!.id]
    );

    if (!file.rows[0]) {
      return res.status(404).json({ success: false, error: { code: 'NOT_FOUND' } });
    }

    const fileBuffer = await storageService.download(
      file.rows[0].bucket,
      file.rows[0].key
    );

    res.set({
      'Content-Type': file.rows[0].mime_type,
      'Content-Length': fileBuffer.length,
      'Content-Disposition': `inline; filename="${file.rows[0].original_name}"`,
      'Cache-Control': 'private, max-age=3600',
    });

    res.send(fileBuffer);
  } catch (error) {
    next(error);
  }
});
```

---

## 15. Delete File Flow

```typescript
// ลบไฟล์ทั้ง DB และ Storage
router.delete('/files/:fileId', authenticate, async (req, res, next) => {
  try {
    const file = await pool.query(
      'SELECT * FROM files WHERE id = $1 AND user_id = $2 AND deleted_at IS NULL',
      [req.params.fileId, req.user!.id]
    );

    if (!file.rows[0]) {
      return res.status(404).json({ success: false, error: { code: 'NOT_FOUND' } });
    }

    const { bucket, key } = file.rows[0];

    // Soft delete ใน DB ก่อน
    await pool.query(
      'UPDATE files SET deleted_at = NOW(), status = $1 WHERE id = $2',
      ['deleted', req.params.fileId]
    );

    // ลบจาก Storage
    try {
      await storageService.delete(bucket, key);
    } catch (storageError) {
      // ถ้าลบจาก storage ไม่ได้ ให้ log แล้วยังคง return success
      // cleanup job จะจัดการทีหลัง
      console.error('Failed to delete from storage:', storageError);
    }

    res.json({ success: true, message: 'ลบไฟล์สำเร็จ' });
  } catch (error) {
    next(error);
  }
});

// Cleanup job: ลบ soft-deleted files ที่เก่าเกิน 30 วัน
async function cleanupDeletedFiles() {
  const oldFiles = await pool.query(
    `SELECT bucket, key FROM files
     WHERE deleted_at < NOW() - INTERVAL '30 days'
       AND status = 'deleted'
     LIMIT 100`
  );

  for (const file of oldFiles.rows) {
    try {
      await storageService.delete(file.bucket, file.key);
      await pool.query(
        'DELETE FROM files WHERE bucket = $1 AND key = $2',
        [file.bucket, file.key]
      );
    } catch (error) {
      console.error(`Failed to cleanup ${file.key}:`, error);
    }
  }
}
```

---

## 16. Full Example: Profile Picture Upload

```typescript
// src/routes/profile.ts
import { Router, Request, Response, NextFunction } from 'express';
import multer from 'multer';
import { authenticate } from '../middleware/auth';
import { storageService } from '../services/storageService';
import { imageService } from '../services/imageService';
import { validateFileMagicBytes } from '../config/upload';
import pool from '../db/pool';
import { v4 as uuidv4 } from 'uuid';

const router = Router();

const avatarUpload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: 5 * 1024 * 1024, files: 1 },
  fileFilter: (req, file, cb) => {
    const allowedTypes = ['image/jpeg', 'image/png', 'image/webp'];
    if (allowedTypes.includes(file.mimetype)) cb(null, true);
    else cb(new Error('รองรับเฉพาะ JPEG, PNG, WebP'));
  },
});

// POST /profile/avatar
router.post(
  '/avatar',
  authenticate,
  avatarUpload.single('avatar'),
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      if (!req.file) {
        return res.status(400).json({
          success: false,
          error: { code: 'NO_FILE', message: 'กรุณาเลือกไฟล์' },
        });
      }

      const userId = req.user!.id;

      // 1. Validate magic bytes
      const isValid = await validateFileMagicBytes(req.file.buffer, req.file.mimetype);
      if (!isValid) {
        return res.status(400).json({
          success: false,
          error: { code: 'INVALID_FILE', message: 'ไฟล์ไม่ถูกต้อง' },
        });
      }

      // 2. Strip EXIF (ความเป็นส่วนตัว)
      const cleanBuffer = await imageService.stripExif(req.file.buffer);

      // 3. ตรวจสอบ metadata
      const metadata = await imageService.getMetadata(cleanBuffer);

      // 4. Process รูปภาพหลายขนาด
      const processed = await imageService.processAvatar(cleanBuffer);

      // 5. สร้าง keys
      const baseKey = `avatars/${userId}/${uuidv4()}`;
      const keys = {
        original: `${baseKey}/original.jpg`,
        large: `${baseKey}/large.jpg`,
        medium: `${baseKey}/medium.jpg`,
        thumbnail: `${baseKey}/thumbnail.jpg`,
      };

      // 6. Upload ทั้งหมดพร้อมกัน
      await Promise.all([
        storageService.upload({
          bucket: 'avatars',
          key: keys.original,
          body: processed.original,
          contentType: 'image/jpeg',
          cacheControl: 'max-age=31536000',
        }),
        storageService.upload({
          bucket: 'avatars',
          key: keys.large,
          body: processed.large,
          contentType: 'image/jpeg',
          cacheControl: 'max-age=31536000',
        }),
        storageService.upload({
          bucket: 'avatars',
          key: keys.medium,
          body: processed.medium,
          contentType: 'image/jpeg',
          cacheControl: 'max-age=31536000',
        }),
        storageService.upload({
          bucket: 'avatars',
          key: keys.thumbnail,
          body: processed.thumbnail,
          contentType: 'image/jpeg',
          cacheControl: 'max-age=31536000',
        }),
      ]);

      // 7. ดึง avatar เก่ามาลบ
      const oldAvatar = await pool.query(
        'SELECT avatar_key FROM users WHERE id = $1',
        [userId]
      );

      // 8. อัพเดท DB
      const baseUrl = process.env.MINIO_PUBLIC_URL;
      const avatarUrls = {
        original: `${baseUrl}/avatars/${keys.original}`,
        large: `${baseUrl}/avatars/${keys.large}`,
        medium: `${baseUrl}/avatars/${keys.medium}`,
        thumbnail: `${baseUrl}/avatars/${keys.thumbnail}`,
      };

      await pool.query(
        `UPDATE users
         SET avatar_url = $1,
             avatar_key = $2,
             avatar_metadata = $3
         WHERE id = $4`,
        [
          avatarUrls.medium,
          baseKey,
          JSON.stringify({ ...avatarUrls, width: metadata.width, height: metadata.height }),
          userId,
        ]
      );

      // 9. ลบ avatar เก่า (background)
      const oldKey = oldAvatar.rows[0]?.avatar_key;
      if (oldKey && oldKey !== baseKey) {
        Promise.all([
          storageService.delete('avatars', `${oldKey}/original.jpg`),
          storageService.delete('avatars', `${oldKey}/large.jpg`),
          storageService.delete('avatars', `${oldKey}/medium.jpg`),
          storageService.delete('avatars', `${oldKey}/thumbnail.jpg`),
        ]).catch(console.error);
      }

      res.json({
        success: true,
        data: {
          avatarUrls,
          metadata: {
            width: metadata.width,
            height: metadata.height,
            originalSize: req.file.size,
          },
        },
        message: 'อัพโหลดรูปโปรไฟล์สำเร็จ',
      });
    } catch (error) {
      next(error);
    }
  }
);

export default router;
```

### ทดสอบ

```bash
# อัพโหลด avatar
curl -X POST http://localhost:3000/api/profile/avatar \
  -H "Authorization: Bearer <token>" \
  -F "avatar=@/path/to/photo.jpg"

# ขอ pre-signed URL
curl -X POST http://localhost:3000/api/upload/presigned \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "filename": "document.pdf",
    "contentType": "application/pdf",
    "fileSize": 1024000,
    "bucket": "documents"
  }'

# ลบไฟล์
curl -X DELETE http://localhost:3000/api/files/file-uuid-here \
  -H "Authorization: Bearer <token>"
```

---

## สรุป

| หัวข้อ | Best Practice |
|--------|--------------|
| Validation | ตรวจสอบทั้ง MIME type และ magic bytes |
| Storage | Memory สำหรับเล็ก, Disk สำหรับใหญ่ |
| Naming | UUID หรือ structured path |
| Images | Strip EXIF + resize หลายขนาด |
| Large files | Chunked/multipart upload |
| Serve | Pre-signed URL หรือ CDN |
| Delete | Soft delete ก่อน, cleanup ทีหลัง |
| Temp files | ลบเสมอใน finally block |
