# Part 42: Image Processing Pipeline

## บทนำ: ทำไม Image Processing ถึงสำคัญ?

ในยุคที่ผู้ใช้ consume content ผ่านอุปกรณ์หลากหลาย ตั้งแต่ mobile ไปจนถึง 4K monitor การส่ง image ที่มีขนาดและ format เหมาะสมจึงสำคัญมากต่อ:

```
1. Performance: ลด page load time
   - ภาพ 5MB original → 150KB WebP = 97% ลดลง!
   - Core Web Vitals: LCP (Largest Contentful Paint)
   
2. User Experience: ไม่ต้องรอนาน
   - Progressive loading
   - Placeholder → Low quality → Full quality
   
3. Cost: ลด bandwidth/storage
   - S3 storage cost
   - CDN bandwidth cost
   - Mobile users' data plans
   
4. SEO: Google ให้ความสำคัญกับ page speed
```

## Image Formats เปรียบเทียบ

### JPEG (Joint Photographic Experts Group)

```
ข้อดี:
- รองรับในทุก browser/device
- Lossy compression ที่ดีสำหรับรูปถ่าย
- Progressive JPEG: load แบบ top-down แล้วค่อย sharper

ข้อเสีย:
- ไม่รองรับ transparency (alpha channel)
- คุณภาพลดลงทุกครั้งที่ save

Use case: รูปถ่าย, backgrounds, complex images
```

### PNG (Portable Network Graphics)

```
ข้อดี:
- Lossless compression
- รองรับ transparency (RGBA)
- คุณภาพไม่ลดลงเมื่อ save

ข้อเสีย:
- ขนาดใหญ่กว่า JPEG มาก
- ไม่เหมาะกับรูปถ่าย

Use case: logos, icons, screenshots, images ที่ต้องการ transparency
```

### WebP

```
ข้อดี:
- ขนาดเล็กกว่า JPEG 25-35%
- รองรับ transparency (lossless + lossy)
- รองรับ animation
- Browser support: 96%+ (2024)

ข้อเสีย:
- Safari รองรับแค่ iOS 14+, macOS 11+

Use case: เกือบทุกอย่าง (แทน JPEG/PNG ได้)
```

### AVIF (AV1 Image File Format)

```
ข้อดี:
- ขนาดเล็กกว่า WebP อีก 20-30%
- คุณภาพสูงมาก
- รองรับ HDR

ข้อเสีย:
- Encoding ช้ากว่า WebP มาก (5-10x)
- Browser support: ~90% (2024)
- Safari รองรับใน iOS 16+

Use case: ภาพ high-quality ที่ encoding ไม่ต้อง real-time
```

### Format Comparison (ขนาดไฟล์จริง)

```
รูปถ่าย 2000x1500px:
┌────────────┬──────────┬────────────────────────┐
│ Format     │ Size     │ Browser Support        │
├────────────┼──────────┼────────────────────────┤
│ JPEG Q90   │ 1.2 MB   │ 100%                   │
│ JPEG Q75   │ 600 KB   │ 100%                   │
│ PNG        │ 8.5 MB   │ 100%                   │
│ WebP Q85   │ 800 KB   │ 96%                    │
│ AVIF Q80   │ 350 KB   │ 90%                    │
└────────────┴──────────┴────────────────────────┘

ประหยัดได้:
JPEG Q75 vs AVIF Q80 = 41% ลดลง
```

## Sharp Library: npm install sharp

Sharp เป็น Node.js library ที่เร็วที่สุดสำหรับ image processing
ใช้ libvips ซึ่งเร็วกว่า ImageMagick 4-5x

```bash
npm install sharp
npm install -D @types/sharp
```

### Sharp: resize()

```typescript
import sharp from "sharp";

// Basic resize
await sharp(inputBuffer)
  .resize(800, 600)          // width x height (distort)
  .toFile("output.jpg");

// Fit modes
await sharp(inputBuffer)
  .resize(800, 600, {
    fit: "cover",            // crop ให้พอดี (default behavior เหมือน CSS background-size: cover)
  })
  .toFile("cover.jpg");

await sharp(inputBuffer)
  .resize(800, 600, {
    fit: "contain",          // letterbox ให้ใส่ใน box โดยไม่ crop
    background: { r: 255, g: 255, b: 255, alpha: 1 }, // white background
  })
  .toFile("contain.jpg");

await sharp(inputBuffer)
  .resize(800, 600, {
    fit: "inside",           // resize ให้อยู่ใน box แต่ไม่ letterbox
    withoutEnlargement: true // ไม่ขยายถ้าเล็กอยู่แล้ว
  })
  .toFile("inside.jpg");

await sharp(inputBuffer)
  .resize(800, 600, {
    fit: "outside",          // resize ให้ครอบ box (opposite of inside)
  })
  .toFile("outside.jpg");

await sharp(inputBuffer)
  .resize(800, null, {       // null = auto calculate
    fit: "contain",
  })
  .toFile("width-only.jpg");

// Thumbnail ที่เน้นจุดสนใจ (Attention-based crop)
await sharp(inputBuffer)
  .resize(200, 200, {
    fit: "cover",
    position: "attention",  // หรือ "entropy", "center", "top", "bottom"
  })
  .toFile("smart-crop.jpg");
```

### Sharp: Rotation และ Flip

```typescript
// Auto-rotate ตาม EXIF orientation
await sharp(inputBuffer)
  .rotate()                  // auto based on EXIF
  .toFile("auto-rotate.jpg");

// Rotate manually
await sharp(inputBuffer)
  .rotate(90)                // หมุน 90 องศาตามเข็ม
  .toFile("rotate90.jpg");

await sharp(inputBuffer)
  .rotate(45, {
    background: { r: 0, g: 0, b: 0, alpha: 0 }, // transparent background
  })
  .toFile("rotate45.png");

// Flip
await sharp(inputBuffer)
  .flip()                    // flip แนวตั้ง (vertical mirror)
  .flop()                    // flop แนวนอน (horizontal mirror)
  .toFile("flipped.jpg");
```

### Sharp: Format และ Quality

```typescript
// JPEG
await sharp(inputBuffer)
  .jpeg({
    quality: 85,             // 1-100 (default: 80)
    progressive: true,       // Progressive JPEG
    mozjpeg: true,           // ใช้ mozjpeg encoder (ขนาดเล็กกว่า 5-10%)
    chromaSubsampling: "4:2:0", // ลดขนาดเพิ่ม (default สำหรับ quality < 90)
  })
  .toBuffer();

// PNG
await sharp(inputBuffer)
  .png({
    compressionLevel: 9,     // 0-9 (default: 6)
    adaptiveFiltering: true,
    palette: true,           // ลด colors สำหรับ simple images
  })
  .toBuffer();

// WebP
await sharp(inputBuffer)
  .webp({
    quality: 85,             // 1-100 (default: 80)
    lossless: false,         // true = ไม่สูญเสียคุณภาพ แต่ขนาดใหญ่กว่า
    nearLossless: false,     // near-lossless compression
    smartSubsample: true,    // ลดขนาดอัตโนมัติ
    effort: 4,               // 0-6, สูง = ขนาดเล็กกว่า แต่ช้ากว่า
  })
  .toBuffer();

// AVIF
await sharp(inputBuffer)
  .avif({
    quality: 80,             // 1-100
    lossless: false,
    effort: 4,               // 0-9, สูง = ขนาดเล็กกว่า แต่ช้ากว่า
    chromaSubsampling: "4:2:0",
  })
  .toBuffer();

// เลือก format อัตโนมัติตาม Accept header
function getBestFormat(acceptHeader: string): "avif" | "webp" | "jpeg" {
  if (acceptHeader.includes("image/avif")) return "avif";
  if (acceptHeader.includes("image/webp")) return "webp";
  return "jpeg";
}
```

### Sharp: Composite (Watermark)

```typescript
// เพิ่ม watermark
async function addWatermark(
  imageBuffer: Buffer,
  watermarkPath: string
): Promise<Buffer> {
  // อ่าน watermark
  const watermark = await sharp(watermarkPath)
    .resize(200, null, { fit: "contain" })
    .toBuffer();
  
  return sharp(imageBuffer)
    .composite([{
      input: watermark,
      gravity: "southeast",        // ตำแหน่ง: northwest, north, northeast, etc.
      blend: "over",               // blend mode
    }])
    .toBuffer();
}

// Text watermark ด้วย SVG
async function addTextWatermark(
  imageBuffer: Buffer,
  text: string
): Promise<Buffer> {
  const { width, height } = await sharp(imageBuffer).metadata();
  
  const svgText = Buffer.from(`
    <svg width="${width}" height="${height}" xmlns="http://www.w3.org/2000/svg">
      <style>
        .watermark {
          fill: rgba(255, 255, 255, 0.5);
          font-size: 48px;
          font-family: Arial, sans-serif;
        }
      </style>
      <text
        x="50%"
        y="95%"
        text-anchor="middle"
        class="watermark"
        transform="rotate(-30, ${(width || 0) / 2}, ${(height || 0) / 2})"
      >${text}</text>
    </svg>
  `);
  
  return sharp(imageBuffer)
    .composite([{
      input: svgText,
      top: 0,
      left: 0,
    }])
    .toBuffer();
}
```

### Sharp: Metadata Extraction

```typescript
async function extractMetadata(buffer: Buffer) {
  const metadata = await sharp(buffer).metadata();
  
  return {
    format: metadata.format,          // jpeg, png, webp, etc.
    width: metadata.width,
    height: metadata.height,
    channels: metadata.channels,      // 3 = RGB, 4 = RGBA
    hasAlpha: metadata.hasAlpha,      // มี transparency ไหม
    density: metadata.density,        // DPI
    isProgressive: metadata.isProgressive,
    size: metadata.size,              // bytes (ถ้าเป็น file)
    exif: metadata.exif,              // EXIF data (Buffer)
    icc: metadata.icc,               // Color profile
    orientation: metadata.orientation, // EXIF orientation (1-8)
    
    // Parse EXIF
    ...(metadata.exif ? parseExif(metadata.exif) : {}),
  };
}

// Strip EXIF metadata (privacy/security)
async function stripExif(buffer: Buffer): Promise<Buffer> {
  return sharp(buffer)
    .rotate()          // Auto-rotate ก่อน (ใช้ EXIF orientation)
    .withMetadata({    // Keep only ที่ระบุ
      // exif: {},     // ถ้าไม่ระบุ = remove ทั้งหมด
    })
    .toBuffer();
}
```

## Image Processing Pipeline แบบสมบูรณ์

### Architecture

```
┌────────────────────────────────────────────────────────────┐
│                  Image Processing Pipeline                   │
│                                                              │
│  1. User Upload                                              │
│     Browser → Multipart/Presigned URL → S3 (original)       │
│                                                              │
│  2. Trigger Processing                                       │
│     S3 Event → SQS Queue → Worker picks up job              │
│                                                              │
│  3. Process                                                  │
│     Worker: Download from S3 → Sharp Process →              │
│     Generate variants: thumb, medium, large, webp, avif     │
│                                                              │
│  4. Store                                                    │
│     Upload variants to S3 → Save paths to PostgreSQL        │
│                                                              │
│  5. Serve                                                    │
│     CDN (CloudFront) → Serve cached variants                │
└────────────────────────────────────────────────────────────┘
```

### Database Schema

```sql
-- migrations/create_media_files.sql

CREATE TABLE media_files (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  -- Original file
  original_key    TEXT NOT NULL,        -- S3 key
  original_url    TEXT NOT NULL,        -- CDN URL
  
  -- Processed variants
  thumbnail_key   TEXT,                 -- 150x150
  medium_key      TEXT,                 -- 800x600
  large_key       TEXT,                 -- 1920x1080
  webp_key        TEXT,                 -- WebP version
  avif_key        TEXT,                 -- AVIF version
  
  -- Metadata
  width           INTEGER,
  height          INTEGER,
  size_bytes      INTEGER NOT NULL,
  mime_type       TEXT NOT NULL,
  format          TEXT NOT NULL,        -- jpeg, png, webp, etc.
  has_alpha       BOOLEAN DEFAULT false,
  
  -- Processing status
  status          TEXT NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending', 'processing', 'done', 'failed')),
  error_message   TEXT,
  processed_at    TIMESTAMPTZ,
  
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_media_files_user_id ON media_files(user_id);
CREATE INDEX idx_media_files_status ON media_files(status) WHERE status != 'done';
```

### S3 Event → SQS Setup

```typescript
// infrastructure/s3-events.ts (Pulumi/CDK)
// S3 ส่ง event เมื่อมีการ upload

// S3 Bucket notification → SQS
const s3UploadQueue = new aws.sqs.Queue("image-upload-queue", {
  visibilityTimeoutSeconds: 300,  // 5 นาที (เวลา process)
  messageRetentionSeconds: 86400, // 1 วัน
  receiveWaitTimeSeconds: 20,     // Long polling
  redrivePolicy: {
    deadLetterTargetArn: deadLetterQueue.arn,
    maxReceiveCount: 3,           // retry 3 ครั้ง แล้วไป DLQ
  },
});
```

### Worker: Process Images from Queue

```typescript
// src/workers/image-processor.worker.ts
import { SQSClient, ReceiveMessageCommand, DeleteMessageCommand } from "@aws-sdk/client-sqs";
import { S3Client, GetObjectCommand, PutObjectCommand } from "@aws-sdk/client-s3";
import sharp from "sharp";
import { db } from "../db";

const sqs = new SQSClient({ region: process.env.AWS_REGION! });
const s3 = new S3Client({ region: process.env.AWS_REGION! });

const QUEUE_URL = process.env.IMAGE_QUEUE_URL!;
const BUCKET = process.env.S3_BUCKET!;

interface S3EventRecord {
  s3: {
    bucket: { name: string };
    object: { key: string; size: number };
  };
}

async function processImage(key: string): Promise<void> {
  // 1. ดึงภาพจาก S3
  const getCommand = new GetObjectCommand({ Bucket: BUCKET, Key: key });
  const s3Object = await s3.send(getCommand);
  
  if (!s3Object.Body) throw new Error("Empty S3 object");
  
  const buffer = Buffer.from(await s3Object.Body.transformToByteArray());
  
  // 2. ดึง metadata
  const metadata = await sharp(buffer).metadata();
  
  if (!metadata.width || !metadata.height) {
    throw new Error("Cannot read image dimensions");
  }
  
  // 3. Strip EXIF แล้ว auto-rotate
  const processedOriginal = await sharp(buffer)
    .rotate()                    // Auto-rotate ตาม EXIF
    .toBuffer();
  
  // 4. สร้าง variants
  const variants = await Promise.all([
    // Thumbnail: 150x150 crop
    sharp(processedOriginal)
      .resize(150, 150, { fit: "cover", position: "attention" })
      .jpeg({ quality: 80, progressive: true, mozjpeg: true })
      .toBuffer()
      .then(buf => ({ suffix: "_thumb", ext: "jpg", buf, mime: "image/jpeg" })),
    
    // Medium: max 800x600
    sharp(processedOriginal)
      .resize(800, 600, { fit: "inside", withoutEnlargement: true })
      .jpeg({ quality: 85, progressive: true, mozjpeg: true })
      .toBuffer()
      .then(buf => ({ suffix: "_medium", ext: "jpg", buf, mime: "image/jpeg" })),
    
    // Large: max 1920x1080
    sharp(processedOriginal)
      .resize(1920, 1080, { fit: "inside", withoutEnlargement: true })
      .jpeg({ quality: 90, progressive: true, mozjpeg: true })
      .toBuffer()
      .then(buf => ({ suffix: "_large", ext: "jpg", buf, mime: "image/jpeg" })),
    
    // WebP: max 1920x1080
    sharp(processedOriginal)
      .resize(1920, 1080, { fit: "inside", withoutEnlargement: true })
      .webp({ quality: 85, effort: 4 })
      .toBuffer()
      .then(buf => ({ suffix: "", ext: "webp", buf, mime: "image/webp" })),
    
    // AVIF: max 1920x1080 (ช้ากว่าแต่ขนาดเล็กกว่า)
    sharp(processedOriginal)
      .resize(1920, 1080, { fit: "inside", withoutEnlargement: true })
      .avif({ quality: 80, effort: 4 })
      .toBuffer()
      .then(buf => ({ suffix: "", ext: "avif", buf, mime: "image/avif" })),
  ]);
  
  // 5. Upload variants ไป S3
  const baseKey = key.replace(/\.[^.]+$/, ""); // ลบ extension
  
  const uploadPromises = variants.map(async ({ suffix, ext, buf, mime }) => {
    const variantKey = `${baseKey}${suffix}.${ext}`;
    
    await s3.send(new PutObjectCommand({
      Bucket: BUCKET,
      Key: variantKey,
      Body: buf,
      ContentType: mime,
      CacheControl: "public, max-age=31536000, immutable",
      Metadata: {
        "original-key": key,
        "processed-at": new Date().toISOString(),
      },
    }));
    
    return { suffix, ext, key: variantKey };
  });
  
  const uploadedVariants = await Promise.all(uploadPromises);
  
  // 6. อัปเดต database
  const cdnBase = process.env.CLOUDFRONT_URL!;
  
  await db.mediaFile.updateMany({
    where: { originalKey: key },
    data: {
      thumbnailKey: uploadedVariants.find(v => v.suffix === "_thumb")?.key,
      mediumKey: uploadedVariants.find(v => v.suffix === "_medium")?.key,
      largeKey: uploadedVariants.find(v => v.suffix === "_large")?.key,
      webpKey: uploadedVariants.find(v => v.ext === "webp")?.key,
      avifKey: uploadedVariants.find(v => v.ext === "avif")?.key,
      width: metadata.width,
      height: metadata.height,
      status: "done",
      processedAt: new Date(),
    },
  });
  
  console.log(`Processed image: ${key} → ${uploadedVariants.length} variants`);
}

// Main worker loop
async function startWorker(): Promise<never> {
  console.log("Image processing worker started");
  
  while (true) {
    try {
      const response = await sqs.send(new ReceiveMessageCommand({
        QueueUrl: QUEUE_URL,
        MaxNumberOfMessages: 5,
        WaitTimeSeconds: 20,       // Long polling
        VisibilityTimeout: 300,
      }));
      
      if (!response.Messages?.length) continue;
      
      await Promise.all(
        response.Messages.map(async (message) => {
          try {
            const body = JSON.parse(message.Body || "{}");
            
            // S3 Event notification format
            if (body.Records) {
              await Promise.all(
                body.Records.map(async (record: S3EventRecord) => {
                  const key = decodeURIComponent(
                    record.s3.object.key.replace(/\+/g, " ")
                  );
                  
                  // Mark as processing
                  await db.mediaFile.updateMany({
                    where: { originalKey: key },
                    data: { status: "processing" },
                  });
                  
                  await processImage(key);
                })
              );
            }
            
            // Delete message after successful processing
            await sqs.send(new DeleteMessageCommand({
              QueueUrl: QUEUE_URL,
              ReceiptHandle: message.ReceiptHandle!,
            }));
          } catch (error) {
            console.error("Error processing message:", error);
            
            // Mark as failed
            const body = JSON.parse(message.Body || "{}");
            if (body.Records?.[0]?.s3?.object?.key) {
              await db.mediaFile.updateMany({
                where: { originalKey: body.Records[0].s3.object.key },
                data: {
                  status: "failed",
                  errorMessage: (error as Error).message,
                },
              });
            }
            // ไม่ลบ message → SQS จะ retry
          }
        })
      );
    } catch (error) {
      console.error("Worker error:", error);
      await new Promise(resolve => setTimeout(resolve, 5000)); // Wait 5s before retry
    }
  }
}

startWorker();
```

## On-demand Image Resizing

### Cloudflare Workers: URL-based Transformation

```javascript
// workers/image-resize.js
// URL format: /images/photo.jpg?w=800&h=600&fit=cover&format=webp&q=85

export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const params = url.searchParams;
    
    // Validate parameters
    const width = Math.min(parseInt(params.get("w") || "0"), 4096);   // max 4096
    const height = Math.min(parseInt(params.get("h") || "0"), 4096);
    const quality = Math.min(Math.max(parseInt(params.get("q") || "85"), 1), 100);
    const format = ["webp", "avif", "jpeg", "png"].includes(params.get("format") || "")
      ? params.get("format")
      : "auto";
    const fit = ["cover", "contain", "fill", "inside", "outside"].includes(params.get("fit") || "")
      ? params.get("fit")
      : "cover";
    
    // Original image URL (ลบ query params)
    const imageUrl = `${url.origin}${url.pathname}`;
    
    // ถ้าไม่มี params ให้ serve ปกติ
    if (!width && !height && format === "auto") {
      return fetch(request);
    }
    
    // Check cache ก่อน
    const cacheKey = new Request(url.toString(), request);
    const cache = caches.default;
    let response = await cache.match(cacheKey);
    
    if (!response) {
      // ดึงรูปต้นฉบับจาก R2/S3
      response = await fetch(imageUrl, {
        cf: {
          image: {
            width: width || undefined,
            height: height || undefined,
            fit,
            format,
            quality,
          },
        },
      });
      
      if (!response.ok) {
        return new Response("Image not found", { status: 404 });
      }
      
      // Set cache headers
      response = new Response(response.body, response);
      response.headers.set("Cache-Control", "public, max-age=31536000");
      response.headers.set("Vary", "Accept");
      
      // Cache ไว้
      await cache.put(cacheKey, response.clone());
    }
    
    return response;
  },
};
```

### Express.js On-demand Resize Route

```typescript
// src/routes/images.ts
import express from "express";
import sharp from "sharp";
import { S3Client, GetObjectCommand } from "@aws-sdk/client-s3";
import NodeCache from "node-cache";

const router = express.Router();
const s3 = new S3Client({ region: process.env.AWS_REGION! });

// In-memory cache (สำหรับ development; production ใช้ Redis หรือ CDN)
const imageCache = new NodeCache({ stdTTL: 3600, maxKeys: 1000 });

// /api/images/:key?w=800&h=600&fit=cover&format=webp&q=85
router.get("/:key(*)", async (req, res) => {
  const { key } = req.params;
  const { w, h, fit = "cover", format, q = "85" } = req.query as Record<string, string>;
  
  const width = w ? parseInt(w) : undefined;
  const height = h ? parseInt(h) : undefined;
  const quality = parseInt(q);
  
  // Accept header สำหรับ auto format
  const acceptsWebP = req.headers.accept?.includes("image/webp");
  const acceptsAVIF = req.headers.accept?.includes("image/avif");
  
  const outputFormat = format || (acceptsAVIF ? "avif" : acceptsWebP ? "webp" : "jpeg");
  
  // Cache key
  const cacheKey = `${key}-${width}-${height}-${fit}-${outputFormat}-${quality}`;
  const cached = imageCache.get<Buffer>(cacheKey);
  
  if (cached) {
    res.set("X-Cache", "HIT");
    res.set("Content-Type", `image/${outputFormat}`);
    res.set("Cache-Control", "public, max-age=86400");
    return res.send(cached);
  }
  
  try {
    // ดึงรูปจาก S3
    const s3Response = await s3.send(new GetObjectCommand({
      Bucket: process.env.S3_BUCKET!,
      Key: key,
    }));
    
    const inputBuffer = Buffer.from(
      await (s3Response.Body as any).transformToByteArray()
    );
    
    // Process
    let pipeline = sharp(inputBuffer).rotate(); // auto-rotate
    
    if (width || height) {
      pipeline = pipeline.resize(width, height, {
        fit: fit as any,
        withoutEnlargement: true,
      });
    }
    
    // Set output format
    switch (outputFormat) {
      case "webp":
        pipeline = pipeline.webp({ quality });
        break;
      case "avif":
        pipeline = pipeline.avif({ quality });
        break;
      case "png":
        pipeline = pipeline.png({ compressionLevel: 9 });
        break;
      default:
        pipeline = pipeline.jpeg({ quality, progressive: true, mozjpeg: true });
    }
    
    const outputBuffer = await pipeline.toBuffer();
    
    // Cache ผลลัพธ์
    imageCache.set(cacheKey, outputBuffer);
    
    res.set("X-Cache", "MISS");
    res.set("Content-Type", `image/${outputFormat}`);
    res.set("Cache-Control", "public, max-age=86400");
    res.set("Vary", "Accept");
    res.send(outputBuffer);
  } catch (error: any) {
    if (error.name === "NoSuchKey") {
      return res.status(404).json({ error: "Image not found" });
    }
    console.error("Image processing error:", error);
    res.status(500).json({ error: "Image processing failed" });
  }
});

export default router;
```

## Responsive Images: srcset และ sizes

```html
<!-- HTML: Responsive images ที่ใช้ CDN -->
<picture>
  <!-- AVIF สำหรับ browser ที่รองรับ -->
  <source
    type="image/avif"
    srcset="
      https://cdn.example.com/images/photo.avif?w=400 400w,
      https://cdn.example.com/images/photo.avif?w=800 800w,
      https://cdn.example.com/images/photo.avif?w=1200 1200w
    "
    sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
  />
  
  <!-- WebP fallback -->
  <source
    type="image/webp"
    srcset="
      https://cdn.example.com/images/photo.webp?w=400 400w,
      https://cdn.example.com/images/photo.webp?w=800 800w,
      https://cdn.example.com/images/photo.webp?w=1200 1200w
    "
    sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
  />
  
  <!-- JPEG fallback สำหรับ IE และ browser เก่า -->
  <img
    src="https://cdn.example.com/images/photo.jpg?w=800"
    srcset="
      https://cdn.example.com/images/photo.jpg?w=400 400w,
      https://cdn.example.com/images/photo.jpg?w=800 800w,
      https://cdn.example.com/images/photo.jpg?w=1200 1200w
    "
    sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
    alt="Photo description"
    loading="lazy"
    decoding="async"
    width="800"
    height="600"
  />
</picture>
```

### React Component สำหรับ Responsive Images

```tsx
// components/ResponsiveImage.tsx
import React from "react";

interface ResponsiveImageProps {
  src: string;          // S3 key หรือ CDN path
  alt: string;
  width?: number;
  height?: number;
  sizes?: string;
  className?: string;
  priority?: boolean;   // true = no lazy loading (above the fold)
  quality?: number;
}

const CDN_URL = process.env.NEXT_PUBLIC_CDN_URL || "https://cdn.example.com";

function buildUrl(src: string, width: number, format: string, quality: number): string {
  return `${CDN_URL}/${src}?w=${width}&format=${format}&q=${quality}`;
}

export function ResponsiveImage({
  src,
  alt,
  width = 1200,
  height,
  sizes = "(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw",
  className,
  priority = false,
  quality = 85,
}: ResponsiveImageProps) {
  const widths = [400, 800, 1200, 1920].filter(w => w <= width * 2);
  
  const makeSrcSet = (format: string) =>
    widths.map(w => `${buildUrl(src, w, format, quality)} ${w}w`).join(", ");
  
  return (
    <picture>
      <source
        type="image/avif"
        srcSet={makeSrcSet("avif")}
        sizes={sizes}
      />
      <source
        type="image/webp"
        srcSet={makeSrcSet("webp")}
        sizes={sizes}
      />
      <img
        src={buildUrl(src, width, "jpeg", quality)}
        srcSet={makeSrcSet("jpeg")}
        sizes={sizes}
        alt={alt}
        width={width}
        height={height}
        loading={priority ? "eager" : "lazy"}
        decoding={priority ? "sync" : "async"}
        className={className}
        style={{ maxWidth: "100%", height: "auto" }}
      />
    </picture>
  );
}
```

## Video Thumbnail Extraction ด้วย ffmpeg

```bash
npm install fluent-ffmpeg
npm install -D @types/fluent-ffmpeg
```

```typescript
// src/services/video-thumbnail.service.ts
import ffmpeg from "fluent-ffmpeg";
import { promises as fs } from "fs";
import { join } from "path";
import { tmpdir } from "os";
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import sharp from "sharp";

const s3 = new S3Client({ region: process.env.AWS_REGION! });

interface ThumbnailOptions {
  count?: number;          // จำนวน thumbnail (default: 3)
  timestamps?: string[];   // เช่น ["00:00:05", "50%", "00:02:00"]
  size?: string;           // เช่น "320x240" หรือ "?x240"
}

export async function extractVideoThumbnails(
  videoPath: string,
  videoKey: string,
  options: ThumbnailOptions = {}
): Promise<string[]> {
  const { count = 3, timestamps, size = "1280x720" } = options;
  
  const tmpDir = join(tmpdir(), `thumbnails-${Date.now()}`);
  await fs.mkdir(tmpDir, { recursive: true });
  
  try {
    // Extract frames
    await new Promise<void>((resolve, reject) => {
      let command = ffmpeg(videoPath)
        .screenshots({
          count: timestamps ? undefined : count,
          timestamps,
          filename: "thumb-%i.jpg",
          folder: tmpDir,
          size,
        })
        .on("end", resolve)
        .on("error", reject);
    });
    
    // อ่านไฟล์ที่สร้าง
    const files = (await fs.readdir(tmpDir)).filter(f => f.endsWith(".jpg"));
    
    // Upload ไป S3
    const baseKey = videoKey.replace(/\.[^.]+$/, "");
    
    const uploadedKeys: string[] = [];
    
    for (const [index, file] of files.entries()) {
      const filePath = join(tmpDir, file);
      const fileBuffer = await fs.readFile(filePath);
      
      // Process ด้วย Sharp (resize และ optimize)
      const processedBuffer = await sharp(fileBuffer)
        .resize(1280, 720, { fit: "inside", withoutEnlargement: true })
        .jpeg({ quality: 85, progressive: true, mozjpeg: true })
        .toBuffer();
      
      // WebP version
      const webpBuffer = await sharp(fileBuffer)
        .resize(1280, 720, { fit: "inside", withoutEnlargement: true })
        .webp({ quality: 85 })
        .toBuffer();
      
      const thumbKey = `${baseKey}_thumb_${index + 1}.jpg`;
      const webpKey = `${baseKey}_thumb_${index + 1}.webp`;
      
      await Promise.all([
        s3.send(new PutObjectCommand({
          Bucket: process.env.S3_BUCKET!,
          Key: thumbKey,
          Body: processedBuffer,
          ContentType: "image/jpeg",
          CacheControl: "public, max-age=31536000, immutable",
        })),
        s3.send(new PutObjectCommand({
          Bucket: process.env.S3_BUCKET!,
          Key: webpKey,
          Body: webpBuffer,
          ContentType: "image/webp",
          CacheControl: "public, max-age=31536000, immutable",
        })),
      ]);
      
      uploadedKeys.push(thumbKey);
    }
    
    return uploadedKeys;
  } finally {
    // ทำความสะอาด temp files
    await fs.rm(tmpDir, { recursive: true, force: true });
  }
}

// Extract video metadata
export async function getVideoMetadata(videoPath: string): Promise<{
  duration: number;
  width: number;
  height: number;
  fps: number;
  codec: string;
  bitrate: number;
}> {
  return new Promise((resolve, reject) => {
    ffmpeg.ffprobe(videoPath, (err, metadata) => {
      if (err) return reject(err);
      
      const videoStream = metadata.streams.find(s => s.codec_type === "video");
      
      if (!videoStream) {
        return reject(new Error("No video stream found"));
      }
      
      const fpsString = videoStream.r_frame_rate || "0/1";
      const [num, den] = fpsString.split("/").map(Number);
      
      resolve({
        duration: metadata.format.duration || 0,
        width: videoStream.width || 0,
        height: videoStream.height || 0,
        fps: den ? num / den : 0,
        codec: videoStream.codec_name || "",
        bitrate: parseInt(metadata.format.bit_rate || "0"),
      });
    });
  });
}
```

## SVG Processing: Sanitization

```typescript
// src/utils/svg-sanitize.ts
// SVG สามารถมี JavaScript ได้! ต้อง sanitize ก่อน serve
import { JSDOM } from "jsdom";
import DOMPurify from "dompurify";

const window = new JSDOM("").window;
const purify = DOMPurify(window);

export function sanitizeSVG(svgString: string): string {
  // Remove dangerous elements and attributes
  const clean = purify.sanitize(svgString, {
    USE_PROFILES: { svg: true, svgFilters: true },
    FORBID_ATTR: ["onload", "onclick", "onerror", "onmouseover"],
    FORBID_TAGS: ["script", "iframe", "object", "embed", "foreignObject"],
  });
  
  return clean;
}

// ตรวจสอบว่าเป็น SVG จริงๆ
export function isValidSVG(content: string): boolean {
  try {
    const dom = new JSDOM(content, { contentType: "image/svg+xml" });
    const svgElement = dom.window.document.querySelector("svg");
    return svgElement !== null;
  } catch {
    return false;
  }
}

// Middleware สำหรับ validate SVG upload
export function validateSVGUpload(buffer: Buffer): { valid: boolean; sanitized?: string; error?: string } {
  const content = buffer.toString("utf-8");
  
  if (!isValidSVG(content)) {
    return { valid: false, error: "Not a valid SVG file" };
  }
  
  const sanitized = sanitizeSVG(content);
  
  // ตรวจสอบว่า sanitize แล้ว content เปลี่ยนไปมากแค่ไหน
  const removedBytes = content.length - sanitized.length;
  if (removedBytes > 100) {
    console.warn(`SVG sanitization removed ${removedBytes} bytes - suspicious content`);
  }
  
  return { valid: true, sanitized };
}
```

## Full TypeScript ImageService Class

```typescript
// src/services/image.service.ts
import sharp from "sharp";
import { S3Client, PutObjectCommand, DeleteObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { createHash, randomUUID } from "crypto";
import { db } from "../db";

export interface ImageVariants {
  original: string;
  thumbnail: string;    // 150x150
  medium: string;       // 800x600
  large: string;        // 1920x1080
  webp: string;         // WebP large
  avif?: string;        // AVIF large (optional, takes time)
}

export interface ProcessedImage {
  id: string;
  userId: string;
  variants: ImageVariants;
  cdnUrls: ImageVariants;
  metadata: {
    originalWidth: number;
    originalHeight: number;
    originalSize: number;
    format: string;
  };
}

export interface ImageUploadOptions {
  generateAVIF?: boolean;       // AVIF ช้ากว่า
  watermarkText?: string;
  maxWidth?: number;
  maxHeight?: number;
  quality?: number;
}

export class ImageService {
  private readonly s3: S3Client;
  private readonly bucket: string;
  private readonly cdnBase: string;
  
  constructor() {
    this.s3 = new S3Client({ region: process.env.AWS_REGION! });
    this.bucket = process.env.S3_BUCKET!;
    this.cdnBase = process.env.CLOUDFRONT_URL!;
  }
  
  async processAndUpload(
    userId: string,
    buffer: Buffer,
    options: ImageUploadOptions = {}
  ): Promise<ProcessedImage> {
    const {
      generateAVIF = false,
      watermarkText,
      maxWidth = 1920,
      maxHeight = 1080,
      quality = 85,
    } = options;
    
    // Validate ว่าเป็นรูปจริงๆ
    const metadata = await sharp(buffer).metadata();
    
    if (!metadata.width || !metadata.height) {
      throw new Error("Invalid image: cannot read dimensions");
    }
    
    if (!["jpeg", "png", "webp", "avif", "gif", "tiff"].includes(metadata.format || "")) {
      throw new Error(`Unsupported format: ${metadata.format}`);
    }
    
    // Auto-rotate ตาม EXIF
    let baseBuffer = await sharp(buffer).rotate().toBuffer();
    
    // เพิ่ม watermark ถ้าต้องการ
    if (watermarkText) {
      baseBuffer = await this.addTextWatermark(baseBuffer, watermarkText);
    }
    
    // สร้าง ID และ keys
    const id = randomUUID();
    const hash = createHash("md5").update(buffer).digest("hex").slice(0, 8);
    const baseKey = `images/${userId}/${id}-${hash}`;
    
    // สร้างทุก variants
    const variantTasks = [
      this.createVariant(baseBuffer, `${baseKey}.jpg`, {
        width: Math.min(maxWidth, metadata.width),
        height: Math.min(maxHeight, metadata.height),
        format: "jpeg",
        quality,
        fit: "inside",
      }),
      this.createVariant(baseBuffer, `${baseKey}_thumb.jpg`, {
        width: 150,
        height: 150,
        format: "jpeg",
        quality: 80,
        fit: "cover",
      }),
      this.createVariant(baseBuffer, `${baseKey}_medium.jpg`, {
        width: 800,
        height: 600,
        format: "jpeg",
        quality: 85,
        fit: "inside",
      }),
      this.createVariant(baseBuffer, `${baseKey}_large.jpg`, {
        width: 1920,
        height: 1080,
        format: "jpeg",
        quality: 90,
        fit: "inside",
      }),
      this.createVariant(baseBuffer, `${baseKey}.webp`, {
        width: 1920,
        height: 1080,
        format: "webp",
        quality: 85,
        fit: "inside",
      }),
    ];
    
    if (generateAVIF) {
      variantTasks.push(
        this.createVariant(baseBuffer, `${baseKey}.avif`, {
          width: 1920,
          height: 1080,
          format: "avif",
          quality: 80,
          fit: "inside",
        })
      );
    }
    
    const uploadedVariants = await Promise.all(variantTasks);
    
    // Build variants map
    const variants: ImageVariants = {
      original: uploadedVariants[0].key,
      thumbnail: uploadedVariants[1].key,
      medium: uploadedVariants[2].key,
      large: uploadedVariants[3].key,
      webp: uploadedVariants[4].key,
      avif: generateAVIF ? uploadedVariants[5].key : undefined,
    };
    
    const cdnUrls: ImageVariants = {
      original: `${this.cdnBase}/${variants.original}`,
      thumbnail: `${this.cdnBase}/${variants.thumbnail}`,
      medium: `${this.cdnBase}/${variants.medium}`,
      large: `${this.cdnBase}/${variants.large}`,
      webp: `${this.cdnBase}/${variants.webp}`,
      avif: variants.avif ? `${this.cdnBase}/${variants.avif}` : undefined,
    };
    
    // Save to DB
    const dbRecord = await db.mediaFile.create({
      data: {
        id,
        userId,
        originalKey: variants.original,
        thumbnailKey: variants.thumbnail,
        mediumKey: variants.medium,
        largeKey: variants.large,
        webpKey: variants.webp,
        avifKey: variants.avif,
        width: metadata.width,
        height: metadata.height,
        sizeBytes: buffer.length,
        mimeType: "image/jpeg",
        format: "jpeg",
        status: "done",
        processedAt: new Date(),
      },
    });
    
    return {
      id: dbRecord.id,
      userId,
      variants,
      cdnUrls,
      metadata: {
        originalWidth: metadata.width,
        originalHeight: metadata.height,
        originalSize: buffer.length,
        format: metadata.format || "jpeg",
      },
    };
  }
  
  private async createVariant(
    buffer: Buffer,
    key: string,
    options: {
      width: number;
      height: number;
      format: "jpeg" | "webp" | "avif" | "png";
      quality: number;
      fit: "cover" | "contain" | "inside";
    }
  ): Promise<{ key: string }> {
    const { width, height, format, quality, fit } = options;
    
    let pipeline = sharp(buffer).resize(width, height, {
      fit,
      withoutEnlargement: true,
    });
    
    switch (format) {
      case "jpeg":
        pipeline = pipeline.jpeg({ quality, progressive: true, mozjpeg: true });
        break;
      case "webp":
        pipeline = pipeline.webp({ quality, effort: 4 });
        break;
      case "avif":
        pipeline = pipeline.avif({ quality, effort: 4 });
        break;
      case "png":
        pipeline = pipeline.png({ compressionLevel: 9 });
        break;
    }
    
    const outputBuffer = await pipeline.toBuffer();
    
    const mimeTypes = {
      jpeg: "image/jpeg",
      webp: "image/webp",
      avif: "image/avif",
      png: "image/png",
    };
    
    await this.s3.send(new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      Body: outputBuffer,
      ContentType: mimeTypes[format],
      CacheControl: "public, max-age=31536000, immutable",
    }));
    
    return { key };
  }
  
  private async addTextWatermark(buffer: Buffer, text: string): Promise<Buffer> {
    const { width = 800, height = 600 } = await sharp(buffer).metadata();
    
    const fontSize = Math.max(24, Math.min(width / 15, 72));
    
    const svgWatermark = Buffer.from(`
      <svg width="${width}" height="${height}" xmlns="http://www.w3.org/2000/svg">
        <text
          x="${width / 2}"
          y="${height - 20}"
          text-anchor="middle"
          fill="rgba(255,255,255,0.6)"
          font-size="${fontSize}"
          font-family="Arial, sans-serif"
          font-weight="bold"
        >${text}</text>
      </svg>
    `);
    
    return sharp(buffer)
      .composite([{ input: svgWatermark, top: 0, left: 0 }])
      .toBuffer();
  }
  
  async getPresignedUploadUrl(
    userId: string,
    filename: string,
    contentType: string
  ): Promise<{ uploadUrl: string; key: string; expiresAt: Date }> {
    const ext = filename.split(".").pop()?.toLowerCase() || "jpg";
    const id = randomUUID();
    const key = `uploads/${userId}/${id}.${ext}`;
    
    const command = new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      ContentType: contentType,
      Metadata: {
        "user-id": userId,
        "original-filename": filename,
      },
    });
    
    const expiresIn = 3600;
    const uploadUrl = await getSignedUrl(this.s3, command, { expiresIn });
    const expiresAt = new Date(Date.now() + expiresIn * 1000);
    
    return { uploadUrl, key, expiresAt };
  }
  
  async deleteImage(imageId: string, userId: string): Promise<void> {
    const mediaFile = await db.mediaFile.findFirst({
      where: { id: imageId, userId },
    });
    
    if (!mediaFile) {
      throw new Error("Image not found or access denied");
    }
    
    // ลบทุก variants
    const keysToDelete = [
      mediaFile.originalKey,
      mediaFile.thumbnailKey,
      mediaFile.mediumKey,
      mediaFile.largeKey,
      mediaFile.webpKey,
      mediaFile.avifKey,
    ].filter((k): k is string => k !== null);
    
    await Promise.all(
      keysToDelete.map(key =>
        this.s3.send(new DeleteObjectCommand({ Bucket: this.bucket, Key: key }))
      )
    );
    
    await db.mediaFile.delete({ where: { id: imageId } });
  }
}

export const imageService = new ImageService();
```

## สรุป: Image Processing Best Practices

```
1. Always strip EXIF: ป้องกัน privacy leak (GPS location, device info)
2. Auto-rotate: handle EXIF orientation ก่อน process
3. Use mozjpeg: ขนาดเล็กกว่า libjpeg 5-10% ด้วยคุณภาพเดียวกัน
4. WebP first: browser support 96%+ แล้ว
5. Progressive JPEG: ดีกว่า baseline สำหรับ large images
6. withoutEnlargement: ไม่ขยายรูปที่เล็กกว่า target size
7. Content-hash ใน filename: cache busting แบบ immutable
8. Process async: ไม่ block main thread ด้วย worker queue
9. Validate input: ตรวจสอบ magic bytes ไม่ใช่แค่ extension
10. Limit dimensions: ป้องกัน decompression bomb (มหา pixel attack)
```

```typescript
// Security: ตรวจสอบ magic bytes ไม่ใช่แค่ extension
function validateImageMagicBytes(buffer: Buffer): boolean {
  // JPEG: FF D8 FF
  if (buffer[0] === 0xFF && buffer[1] === 0xD8 && buffer[2] === 0xFF) return true;
  // PNG: 89 50 4E 47
  if (buffer[0] === 0x89 && buffer[1] === 0x50 && buffer[2] === 0x4E && buffer[3] === 0x47) return true;
  // GIF: 47 49 46 38
  if (buffer[0] === 0x47 && buffer[1] === 0x49 && buffer[2] === 0x46) return true;
  // WebP: 52 49 46 46 ... 57 45 42 50
  if (buffer[0] === 0x52 && buffer[1] === 0x49 && buffer[2] === 0x46 && buffer[3] === 0x46) {
    if (buffer[8] === 0x57 && buffer[9] === 0x45 && buffer[10] === 0x42 && buffer[11] === 0x50) return true;
  }
  return false;
}

// Security: ป้องกัน decompression bomb
async function validateImageSize(buffer: Buffer): Promise<void> {
  const metadata = await sharp(buffer).metadata();
  const totalPixels = (metadata.width || 0) * (metadata.height || 0);
  
  if (totalPixels > 50_000_000) { // 50 megapixels
    throw new Error("Image too large: exceeds 50 megapixel limit");
  }
}
```
