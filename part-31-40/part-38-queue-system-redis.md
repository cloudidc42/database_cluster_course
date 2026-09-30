# Part 38: Queue System ด้วย Redis (BullMQ)

## บทนำ

Message Queue คือระบบที่ช่วยให้ Application สามารถส่งงานไปทำในเบื้องหลัง (Background) โดยไม่ต้องรอให้เสร็จทันที เป็นหนึ่งในสถาปัตยกรรมที่สำคัญที่สุดในการ Build Scalable Systems

ในบทนี้เราจะเรียนรู้:
- ทำไมต้องใช้ Message Queue
- BullMQ: queue library ที่ใช้ Redis
- Job lifecycle ทั้งหมด
- Retry strategies
- Scheduled jobs
- การ Implement ระบบ Email + Image Processing

---

## 1. ทำไมต้องใช้ Message Queue

### 1.1 ปัญหาที่ไม่มี Queue

```typescript
// ❌ ไม่ดี: ทำงานทุกอย่างใน HTTP Request
app.post('/users/register', async (req, res) => {
  const user = await createUser(req.body);
  
  // ส่ง welcome email → อาจใช้เวลา 3-5 วินาที!
  await sendWelcomeEmail(user.email);
  
  // Resize profile picture → อาจใช้เวลา 10+ วินาที!
  await resizeProfilePicture(user.avatarUrl);
  
  // ส่ง SMS verification → อาจใช้เวลา 2-3 วินาที!
  await sendSMSVerification(user.phone);
  
  // User รอ 15+ วินาที...
  res.json({ user });
});
```

### 1.2 ด้วย Queue

```typescript
// ✅ ดี: ส่งงานไปทำใน background
app.post('/users/register', async (req, res) => {
  const user = await createUser(req.body);
  
  // เพิ่มงานลง queue → เร็วมาก (milliseconds)
  await emailQueue.add('welcome-email', { userId: user.id, email: user.email });
  await imageQueue.add('resize-avatar', { userId: user.id, avatarUrl: user.avatarUrl });
  await smsQueue.add('verification-sms', { userId: user.id, phone: user.phone });
  
  // Response กลับทันที!
  res.json({ user, message: 'Account created! Check your email.' });
});

// Workers ทำงานใน background...
```

### 1.3 Use Cases หลัก

| Use Case | ทำไมต้อง Queue |
|----------|--------------|
| Email sending | SMTP อาจช้า, retry ถ้าล้มเหลว |
| Image/Video processing | CPU intensive, ใช้เวลานาน |
| Push notifications | Volume สูง, async |
| PDF generation | Memory intensive |
| Data import/export | Large files |
| Analytics events | High volume, non-critical |
| Webhook deliveries | Need retry logic |
| Scheduled reports | Run daily/weekly |

---

## 2. BullMQ Concepts

### 2.1 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        BullMQ                                │
│                                                              │
│  ┌──────────┐    ┌─────────────┐    ┌──────────────────┐   │
│  │  Queue   │───▶│    Redis    │◀───│     Worker       │   │
│  │          │    │  (Storage)  │    │  (Job Processor) │   │
│  └──────────┘    └─────────────┘    └──────────────────┘   │
│                        │                                     │
│                        ▼                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              QueueEvents                            │   │
│  │  (Monitor: completed, failed, progress, etc.)       │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Job Lifecycle

```
                    ┌──────────┐
                    │  added   │  ← queue.add()
                    └────┬─────┘
                         │
            ┌────────────┴────────────┐
            │                         │
     ┌──────▼──────┐          ┌───────▼──────┐
     │   waiting   │          │    delayed   │  ← delay option
     └──────┬──────┘          └───────┬──────┘
            │                         │ (after delay)
            └────────────┬────────────┘
                         │
                  ┌──────▼──────┐
                  │   active    │  ← worker picks up
                  └──────┬──────┘
                         │
            ┌────────────┴────────────┐
            │                         │
     ┌──────▼──────┐          ┌───────▼──────┐
     │  completed  │          │    failed    │
     └─────────────┘          └───────┬──────┘
                                      │
                         ┌────────────┴────────────┐
                         │                          │
                  ┌──────▼──────┐          ┌────────▼─────┐
                  │  retry queue│          │  dead letter │  ← max retries exceeded
                  └─────────────┘          └──────────────┘
```

---

## 3. Installation และ Setup

```bash
npm install bullmq ioredis

# Optional: Bull Board UI
npm install @bull-board/express @bull-board/api @bull-board/ui
```

### 3.1 Redis Connection

```typescript
// src/queue/config.ts

import { ConnectionOptions } from 'bullmq';

export const redisConnection: ConnectionOptions = {
  host: process.env.REDIS_HOST ?? 'localhost',
  port: parseInt(process.env.REDIS_PORT ?? '6379'),
  password: process.env.REDIS_PASSWORD,
  maxRetriesPerRequest: null,  // BullMQ ต้องการ null (ไม่ limit retries)
  enableReadyCheck: false,     // Disable ready check ใน cluster
  
  // Connection pool settings
  lazyConnect: true,
  
  // TLS (production)
  ...(process.env.REDIS_TLS === 'true' && {
    tls: {
      rejectUnauthorized: true,
    },
  }),
};

// สำหรับ Redis Cluster
export const redisClusterConnection = {
  clusters: [
    { host: 'redis-1', port: 6379 },
    { host: 'redis-2', port: 6379 },
    { host: 'redis-3', port: 6379 },
  ],
  options: {
    password: process.env.REDIS_PASSWORD,
    maxRetriesPerRequest: null,
  },
};
```

---

## 4. Queue และ Job Options

```typescript
// src/queue/emailQueue.ts

import { Queue, QueueOptions } from 'bullmq';
import { redisConnection } from './config';

export interface EmailJobData {
  to: string;
  subject: string;
  template: string;
  variables: Record<string, any>;
  userId?: string;
  priority?: number;
}

const queueOptions: QueueOptions = {
  connection: redisConnection,
  
  defaultJobOptions: {
    // Retry configuration
    attempts: 3,                 // ลอง 3 ครั้ง
    backoff: {
      type: 'exponential',
      delay: 1000,               // เริ่มที่ 1 second, แล้ว 2s, 4s, 8s...
    },
    
    // Cleanup
    removeOnComplete: {
      count: 100,                // เก็บแค่ 100 completed jobs ล่าสุด
      age: 24 * 3600,           // หรือ 24 ชั่วโมง
    },
    removeOnFail: {
      count: 1000,               // เก็บ failed jobs ไว้ debug
      age: 7 * 24 * 3600,      // 7 วัน
    },
    
    // Priority (1 = highest)
    priority: 5,
  },
};

export const emailQueue = new Queue<EmailJobData>('emails', queueOptions);

// ฟังก์ชัน helper สำหรับเพิ่ม jobs
export async function queueWelcomeEmail(userId: string, email: string): Promise<void> {
  await emailQueue.add('welcome', {
    to: email,
    subject: 'Welcome to our platform!',
    template: 'welcome',
    variables: { userId },
    userId,
  }, {
    priority: 1,  // High priority for welcome emails
    delay: 0,
  });
}

export async function queuePasswordResetEmail(
  email: string, 
  resetToken: string
): Promise<void> {
  await emailQueue.add('password-reset', {
    to: email,
    subject: 'Password Reset Request',
    template: 'password-reset',
    variables: { resetToken, expiresIn: '1 hour' },
  }, {
    priority: 1,
    attempts: 5,  // More retries for critical emails
  });
}

export async function queuePromotionalEmail(
  userIds: string[],
  campaignId: string
): Promise<void> {
  // Bulk add jobs
  const jobs = userIds.map(userId => ({
    name: 'promotional',
    data: {
      to: `${userId}@example.com`,  // fetch email from DB in worker
      subject: 'Special offer for you!',
      template: 'promotion',
      variables: { campaignId },
      userId,
    } as EmailJobData,
    opts: {
      priority: 10,   // Low priority
      delay: Math.random() * 3600000,  // Random delay up to 1 hour (avoid spam filters)
    },
  }));

  await emailQueue.addBulk(jobs);
  console.log(`Queued ${jobs.length} promotional emails`);
}
```

---

## 5. Worker Implementation

```typescript
// src/queue/workers/emailWorker.ts

import { Worker, Job, UnrecoverableError } from 'bullmq';
import { redisConnection } from '../config';
import { EmailJobData } from '../emailQueue';
import nodemailer from 'nodemailer';

// Email transporter
const transporter = nodemailer.createTransport({
  host: process.env.SMTP_HOST ?? 'localhost',
  port: parseInt(process.env.SMTP_PORT ?? '587'),
  secure: process.env.SMTP_SECURE === 'true',
  auth: {
    user: process.env.SMTP_USER,
    pass: process.env.SMTP_PASS,
  },
});

async function processEmailJob(job: Job<EmailJobData>): Promise<{ messageId: string }> {
  const { to, subject, template, variables } = job.data;
  
  console.log(`[EmailWorker] Processing job ${job.id}: ${job.name} → ${to}`);
  
  // Report progress
  await job.updateProgress(10);
  
  // Load email template
  const html = await renderEmailTemplate(template, variables);
  await job.updateProgress(30);

  // Send email
  try {
    const result = await transporter.sendMail({
      from: process.env.EMAIL_FROM ?? 'noreply@example.com',
      to,
      subject,
      html,
    });

    await job.updateProgress(100);
    
    console.log(`[EmailWorker] ✅ Email sent: ${result.messageId} → ${to}`);
    
    return { messageId: result.messageId };
    
  } catch (error: any) {
    // ถ้าเป็น permanent error (invalid email, etc.) ไม่ต้อง retry
    if (error.responseCode >= 500 && error.responseCode < 600) {
      throw new UnrecoverableError(`Email permanently rejected: ${error.message}`);
    }
    
    // รอ retry สำหรับ temporary errors
    throw error;
  }
}

async function renderEmailTemplate(
  template: string, 
  variables: Record<string, any>
): Promise<string> {
  // ในโปรเจคจริงใช้ handlebars, mjml, etc.
  const templates: Record<string, string> = {
    welcome: `
      <h1>Welcome!</h1>
      <p>Your account has been created successfully.</p>
      <p>User ID: ${variables.userId}</p>
    `,
    'password-reset': `
      <h1>Password Reset</h1>
      <p>Click <a href="https://app.example.com/reset?token=${variables.resetToken}">here</a> to reset your password.</p>
      <p>This link expires in ${variables.expiresIn}.</p>
    `,
    promotion: `
      <h1>Special Offer!</h1>
      <p>Campaign: ${variables.campaignId}</p>
    `,
  };
  
  return templates[template] ?? `<p>Template not found: ${template}</p>`;
}

// สร้าง Worker
export const emailWorker = new Worker<EmailJobData>(
  'emails',
  processEmailJob,
  {
    connection: redisConnection,
    concurrency: 5,              // Process 5 emails พร้อมกัน
    limiter: {
      max: 100,                  // ไม่เกิน 100 emails
      duration: 60000,           // ต่อ 1 นาที (rate limiting)
    },
    
    // ดึง job timeout
    lockDuration: 30000,         // 30 seconds lock ต่อ job
    
    // Graceful shutdown: รอ jobs ที่กำลังทำอยู่เสร็จ
    drainDelay: 5000,
  }
);

// Event handlers
emailWorker.on('completed', (job, result) => {
  console.log(`[EmailWorker] Job ${job.id} completed:`, result);
});

emailWorker.on('failed', (job, error) => {
  console.error(`[EmailWorker] Job ${job?.id} failed (attempt ${job?.attemptsMade}):`, error.message);
  
  // ถ้าเป็น final failure ส่งไป Dead Letter Queue
  if (job && job.attemptsMade >= (job.opts.attempts ?? 3)) {
    console.error(`[EmailWorker] Job ${job.id} moved to dead letter queue`);
  }
});

emailWorker.on('progress', (job, progress) => {
  console.log(`[EmailWorker] Job ${job.id} progress: ${progress}%`);
});

emailWorker.on('error', (error) => {
  console.error('[EmailWorker] Worker error:', error);
});

emailWorker.on('stalled', (jobId) => {
  console.warn(`[EmailWorker] Job ${jobId} stalled - will be retried`);
});

console.log('[EmailWorker] Worker started, waiting for jobs...');
```

---

## 6. Image Processing Queue

```typescript
// src/queue/workers/imageWorker.ts

import { Worker, Job, Queue } from 'bullmq';
import sharp from 'sharp';
import path from 'path';
import { redisConnection } from '../config';
import { Client as MinIOClient } from 'minio';

export interface ImageJobData {
  jobType: 'resize' | 'thumbnail' | 'compress' | 'watermark';
  inputKey: string;       // S3/MinIO object key
  outputKey?: string;
  options: {
    width?: number;
    height?: number;
    quality?: number;
    format?: 'jpeg' | 'png' | 'webp' | 'avif';
    watermarkText?: string;
    fit?: 'cover' | 'contain' | 'fill' | 'inside' | 'outside';
  };
  userId: string;
  metadata?: Record<string, any>;
}

export interface ImageJobResult {
  outputKey: string;
  originalSize: number;
  processedSize: number;
  compressionRatio: number;
  width: number;
  height: number;
  format: string;
  processingTime: number;
}

const minioClient = new MinIOClient({
  endPoint: process.env.MINIO_ENDPOINT ?? 'localhost',
  port: parseInt(process.env.MINIO_PORT ?? '9000'),
  useSSL: false,
  accessKey: process.env.MINIO_ACCESS_KEY ?? 'minioadmin',
  secretKey: process.env.MINIO_SECRET_KEY ?? 'minioadmin',
});

const BUCKET = 'images';
const THUMBNAIL_BUCKET = 'thumbnails';

async function processImageJob(job: Job<ImageJobData>): Promise<ImageJobResult> {
  const startTime = Date.now();
  const { jobType, inputKey, options, userId } = job.data;
  
  console.log(`[ImageWorker] Processing ${jobType}: ${inputKey}`);
  await job.updateProgress(5);

  // Download image from MinIO
  const inputBuffer = await downloadFromMinIO(BUCKET, inputKey);
  const originalSize = inputBuffer.length;
  await job.updateProgress(25);

  // Process image based on job type
  let outputBuffer: Buffer;
  let metadata: sharp.OutputInfo;

  switch (jobType) {
    case 'resize':
      ({ buffer: outputBuffer, metadata } = await resizeImage(inputBuffer, options, job));
      break;
    
    case 'thumbnail':
      ({ buffer: outputBuffer, metadata } = await createThumbnail(inputBuffer, options, job));
      break;
    
    case 'compress':
      ({ buffer: outputBuffer, metadata } = await compressImage(inputBuffer, options, job));
      break;
    
    case 'watermark':
      ({ buffer: outputBuffer, metadata } = await addWatermark(inputBuffer, options, job));
      break;
    
    default:
      throw new Error(`Unknown job type: ${jobType}`);
  }

  await job.updateProgress(75);

  // Upload result to MinIO
  const outputKey = job.data.outputKey ?? generateOutputKey(inputKey, jobType, options);
  const targetBucket = jobType === 'thumbnail' ? THUMBNAIL_BUCKET : BUCKET;
  
  await uploadToMinIO(targetBucket, outputKey, outputBuffer, options.format ?? 'jpeg');
  await job.updateProgress(95);

  const processingTime = Date.now() - startTime;

  const result: ImageJobResult = {
    outputKey,
    originalSize,
    processedSize: outputBuffer.length,
    compressionRatio: originalSize / outputBuffer.length,
    width: metadata.width ?? 0,
    height: metadata.height ?? 0,
    format: metadata.format ?? 'unknown',
    processingTime,
  };

  await job.updateProgress(100);
  
  console.log(`[ImageWorker] ✅ ${jobType} complete: ${inputKey} → ${outputKey}`);
  console.log(`  Compression: ${(result.compressionRatio).toFixed(2)}x, Time: ${processingTime}ms`);

  return result;
}

async function resizeImage(
  buffer: Buffer, 
  options: ImageJobData['options'],
  job: Job
): Promise<{ buffer: Buffer; metadata: sharp.OutputInfo }> {
  await job.updateProgress(40);
  
  const image = sharp(buffer).resize(options.width, options.height, {
    fit: options.fit ?? 'inside',
    withoutEnlargement: true,
  });

  if (options.format) {
    image[options.format]({ quality: options.quality ?? 80 });
  }

  await job.updateProgress(65);
  
  const outputBuffer = await image.toBuffer({ resolveWithObject: true });
  return { buffer: outputBuffer.data, metadata: outputBuffer.info };
}

async function createThumbnail(
  buffer: Buffer, 
  options: ImageJobData['options'],
  job: Job
): Promise<{ buffer: Buffer; metadata: sharp.OutputInfo }> {
  const width = options.width ?? 200;
  const height = options.height ?? 200;
  
  await job.updateProgress(40);
  
  const result = await sharp(buffer)
    .resize(width, height, { fit: 'cover', position: 'attention' })
    .webp({ quality: 80 })  // Thumbnails เป็น WebP เพื่อลดขนาด
    .toBuffer({ resolveWithObject: true });

  await job.updateProgress(65);
  
  return { buffer: result.data, metadata: result.info };
}

async function compressImage(
  buffer: Buffer, 
  options: ImageJobData['options'],
  job: Job
): Promise<{ buffer: Buffer; metadata: sharp.OutputInfo }> {
  await job.updateProgress(40);
  
  const quality = options.quality ?? 75;
  const format = options.format ?? 'webp';
  
  const image = sharp(buffer)[format]({ quality });
  const result = await image.toBuffer({ resolveWithObject: true });
  
  await job.updateProgress(65);
  
  return { buffer: result.data, metadata: result.info };
}

async function addWatermark(
  buffer: Buffer, 
  options: ImageJobData['options'],
  job: Job
): Promise<{ buffer: Buffer; metadata: sharp.OutputInfo }> {
  await job.updateProgress(40);
  
  const watermarkText = options.watermarkText ?? '© MyApp';
  
  // สร้าง SVG watermark
  const watermarkSvg = Buffer.from(`
    <svg width="200" height="50">
      <text x="100" y="35" 
            font-family="Arial" 
            font-size="24" 
            fill="white" 
            fill-opacity="0.5"
            text-anchor="middle">
        ${watermarkText}
      </text>
    </svg>
  `);
  
  const result = await sharp(buffer)
    .composite([{
      input: watermarkSvg,
      gravity: 'southeast',
    }])
    .toBuffer({ resolveWithObject: true });
  
  await job.updateProgress(65);
  
  return { buffer: result.data, metadata: result.info };
}

function generateOutputKey(inputKey: string, jobType: string, options: ImageJobData['options']): string {
  const ext = options.format ?? path.extname(inputKey).slice(1) ?? 'jpg';
  const baseName = path.basename(inputKey, path.extname(inputKey));
  const dirName = path.dirname(inputKey);
  
  const suffix = jobType === 'thumbnail' 
    ? `${options.width ?? 200}x${options.height ?? 200}` 
    : jobType;
  
  return `${dirName}/${baseName}_${suffix}.${ext}`;
}

async function downloadFromMinIO(bucket: string, key: string): Promise<Buffer> {
  return new Promise((resolve, reject) => {
    const chunks: Buffer[] = [];
    
    minioClient.getObject(bucket, key, (err, stream) => {
      if (err) return reject(err);
      
      stream.on('data', (chunk: Buffer) => chunks.push(chunk));
      stream.on('end', () => resolve(Buffer.concat(chunks)));
      stream.on('error', reject);
    });
  });
}

async function uploadToMinIO(
  bucket: string, 
  key: string, 
  buffer: Buffer, 
  format: string
): Promise<void> {
  const contentTypeMap: Record<string, string> = {
    jpeg: 'image/jpeg',
    jpg: 'image/jpeg',
    png: 'image/png',
    webp: 'image/webp',
    avif: 'image/avif',
  };
  
  await minioClient.putObject(bucket, key, buffer, buffer.length, {
    'Content-Type': contentTypeMap[format] ?? 'image/jpeg',
  });
}

// สร้าง Image Queue
export const imageQueue = new Queue<ImageJobData>('images', {
  connection: redisConnection,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 5000 },
    removeOnComplete: { count: 500, age: 7 * 24 * 3600 },
    removeOnFail: { count: 1000 },
  },
});

// สร้าง Worker
export const imageWorker = new Worker<ImageJobData, ImageJobResult>(
  'images',
  processImageJob,
  {
    connection: redisConnection,
    concurrency: 3,    // Image processing เป็น CPU intensive จำกัดไว้
    lockDuration: 120000,   // 2 minutes timeout ต่อ job
  }
);

imageWorker.on('completed', (job, result) => {
  console.log(`[ImageWorker] Job ${job.id} completed in ${result.processingTime}ms`);
});

imageWorker.on('failed', (job, error) => {
  console.error(`[ImageWorker] Job ${job?.id} failed:`, error.message);
});
```

---

## 7. Scheduled Jobs

```typescript
// src/queue/scheduledJobs.ts

import { Queue } from 'bullmq';
import { redisConnection } from './config';

export interface ReportJobData {
  reportType: 'daily-summary' | 'weekly-analytics' | 'monthly-billing';
  recipients: string[];
  parameters: Record<string, any>;
}

export const reportQueue = new Queue<ReportJobData>('reports', {
  connection: redisConnection,
});

// =====================================================
// Scheduled (Cron) Jobs
// =====================================================

// Daily summary report ทุก 8:00 AM
await reportQueue.upsertJobScheduler(
  'daily-summary-scheduler',          // scheduler id (unique)
  { pattern: '0 8 * * *' },          // cron: 8:00 AM every day
  {
    name: 'daily-summary',
    data: {
      reportType: 'daily-summary',
      recipients: ['admin@example.com'],
      parameters: {},
    },
    opts: {
      priority: 5,
    },
  }
);

// Weekly analytics report ทุกวันจันทร์ 9:00 AM
await reportQueue.upsertJobScheduler(
  'weekly-analytics-scheduler',
  { pattern: '0 9 * * 1' },    // cron: 9:00 AM every Monday
  {
    name: 'weekly-analytics',
    data: {
      reportType: 'weekly-analytics',
      recipients: ['analytics@example.com', 'cto@example.com'],
      parameters: { includeCharts: true },
    },
  }
);

// Monthly billing ทุกวันที่ 1 ของเดือน
await reportQueue.upsertJobScheduler(
  'monthly-billing-scheduler',
  { pattern: '0 10 1 * *' },   // cron: 10:00 AM on 1st of month
  {
    name: 'monthly-billing',
    data: {
      reportType: 'monthly-billing',
      recipients: ['billing@example.com'],
      parameters: { currency: 'THB' },
    },
  }
);

// =====================================================
// Delayed Jobs
// =====================================================

// Send reminder email 24 hours after registration
export async function scheduleRegistrationReminder(userId: string, email: string): Promise<void> {
  const emailQueue = new Queue('emails', { connection: redisConnection });
  
  await emailQueue.add('registration-reminder', {
    to: email,
    subject: 'Complete your profile!',
    template: 'registration-reminder',
    variables: { userId },
  }, {
    delay: 24 * 60 * 60 * 1000,   // 24 hours in milliseconds
    jobId: `reminder:${userId}`,   // deduplication key
  });
  
  console.log(`[Queue] Scheduled reminder for user ${userId} in 24 hours`);
}

// Cancel reminder if user completes profile
export async function cancelRegistrationReminder(userId: string): Promise<void> {
  const emailQueue = new Queue('emails', { connection: redisConnection });
  
  const job = await emailQueue.getJob(`reminder:${userId}`);
  if (job) {
    await job.remove();
    console.log(`[Queue] Cancelled reminder for user ${userId}`);
  }
}

// =====================================================
// Repeatable Jobs (BullMQ v4+ uses upsertJobScheduler)
// =====================================================

// Cleanup old files every night at 2:00 AM
await reportQueue.upsertJobScheduler(
  'cleanup-scheduler',
  { pattern: '0 2 * * *' },
  {
    name: 'cleanup-old-files',
    data: {
      reportType: 'daily-summary',
      recipients: [],
      parameters: {
        olderThanDays: 30,
        buckets: ['uploads', 'thumbnails', 'temp'],
      },
    },
  }
);
```

---

## 8. Job Flow (Parent-Child)

```typescript
// src/queue/flows/orderProcessingFlow.ts

import { FlowProducer, Queue } from 'bullmq';
import { redisConnection } from '../config';

export interface OrderData {
  orderId: string;
  userId: string;
  items: Array<{ productId: string; quantity: number }>;
  totalAmount: number;
}

const flowProducer = new FlowProducer({ connection: redisConnection });

/**
 * สร้าง Order Processing Flow
 * Parent job รอจน children ทั้งหมดเสร็จ
 * 
 * Flow:
 * process-order (parent)
 * ├── validate-inventory
 * ├── charge-payment
 * └── send-confirmation
 *     ├── email-receipt
 *     └── sms-notification
 */
export async function createOrderProcessingFlow(order: OrderData): Promise<void> {
  await flowProducer.add({
    name: 'process-order',
    data: order,
    queueName: 'orders',
    children: [
      {
        name: 'validate-inventory',
        data: { orderId: order.orderId, items: order.items },
        queueName: 'inventory',
      },
      {
        name: 'charge-payment',
        data: { orderId: order.orderId, amount: order.totalAmount, userId: order.userId },
        queueName: 'payments',
        opts: {
          priority: 1,  // Payment is critical
          attempts: 5,
        },
      },
      {
        name: 'send-confirmation',
        data: { orderId: order.orderId, userId: order.userId },
        queueName: 'notifications',
        children: [
          {
            name: 'email-receipt',
            data: { orderId: order.orderId, userId: order.userId },
            queueName: 'emails',
          },
          {
            name: 'sms-notification',
            data: { orderId: order.orderId, userId: order.userId },
            queueName: 'sms',
          },
        ],
      },
    ],
  });

  console.log(`[OrderFlow] Created processing flow for order ${order.orderId}`);
}
```

---

## 9. Queue Events Monitoring

```typescript
// src/queue/monitoring/QueueMonitor.ts

import { QueueEvents, Queue } from 'bullmq';
import { redisConnection } from '../config';

export class QueueMonitor {
  private queueEvents: Map<string, QueueEvents> = new Map();
  private metrics: Map<string, QueueMetrics> = new Map();

  constructor(private queueNames: string[]) {
    this.initializeMonitoring();
  }

  private initializeMonitoring(): void {
    for (const queueName of this.queueNames) {
      const queueEvents = new QueueEvents(queueName, {
        connection: redisConnection,
      });

      queueEvents.on('waiting', ({ jobId }) => {
        this.incrementMetric(queueName, 'waiting');
        console.log(`[${queueName}] Job ${jobId} waiting`);
      });

      queueEvents.on('active', ({ jobId, prev }) => {
        this.decrementMetric(queueName, 'waiting');
        this.incrementMetric(queueName, 'active');
        console.log(`[${queueName}] Job ${jobId} active`);
      });

      queueEvents.on('completed', ({ jobId, returnvalue }) => {
        this.decrementMetric(queueName, 'active');
        this.incrementMetric(queueName, 'completed');
        console.log(`[${queueName}] Job ${jobId} completed`);
      });

      queueEvents.on('failed', ({ jobId, failedReason }) => {
        this.decrementMetric(queueName, 'active');
        this.incrementMetric(queueName, 'failed');
        console.error(`[${queueName}] Job ${jobId} failed: ${failedReason}`);
      });

      queueEvents.on('stalled', ({ jobId }) => {
        console.warn(`[${queueName}] Job ${jobId} stalled`);
      });

      queueEvents.on('delayed', ({ jobId, delay }) => {
        console.log(`[${queueName}] Job ${jobId} delayed by ${delay}ms`);
      });

      queueEvents.on('progress', ({ jobId, data: progress }) => {
        // Throttle progress logging
        if (typeof progress === 'number' && progress % 25 === 0) {
          console.log(`[${queueName}] Job ${jobId} progress: ${progress}%`);
        }
      });

      this.queueEvents.set(queueName, queueEvents);
      this.metrics.set(queueName, {
        waiting: 0, active: 0, completed: 0, failed: 0, delayed: 0,
      });
    }
  }

  private incrementMetric(queue: string, metric: keyof QueueMetrics): void {
    const m = this.metrics.get(queue);
    if (m) m[metric]++;
  }

  private decrementMetric(queue: string, metric: keyof QueueMetrics): void {
    const m = this.metrics.get(queue);
    if (m && m[metric] > 0) m[metric]--;
  }

  async getQueueStatus(queueName: string): Promise<QueueStatus> {
    const queue = new Queue(queueName, { connection: redisConnection });
    
    const [waiting, active, completed, failed, delayed, paused] = await Promise.all([
      queue.getWaitingCount(),
      queue.getActiveCount(),
      queue.getCompletedCount(),
      queue.getFailedCount(),
      queue.getDelayedCount(),
      queue.getPausedCount(),
    ]);

    await queue.close();

    return { queueName, waiting, active, completed, failed, delayed, paused };
  }

  async getAllQueueStatuses(): Promise<QueueStatus[]> {
    return Promise.all(this.queueNames.map(name => this.getQueueStatus(name)));
  }

  async close(): Promise<void> {
    for (const queueEvents of this.queueEvents.values()) {
      await queueEvents.close();
    }
  }
}

interface QueueMetrics {
  waiting: number;
  active: number;
  completed: number;
  failed: number;
  delayed: number;
}

interface QueueStatus {
  queueName: string;
  waiting: number;
  active: number;
  completed: number;
  failed: number;
  delayed: number;
  paused: number;
}
```

---

## 10. Bull Board UI

```typescript
// src/queue/dashboard/bullBoard.ts

import express from 'express';
import { createBullBoard } from '@bull-board/api';
import { BullMQAdapter } from '@bull-board/api/bullMQAdapter';
import { ExpressAdapter } from '@bull-board/express';
import { emailQueue } from '../emailQueue';
import { imageQueue } from '../workers/imageWorker';
import { reportQueue } from '../scheduledJobs';

export function setupBullBoard(app: express.Application): void {
  const serverAdapter = new ExpressAdapter();
  serverAdapter.setBasePath('/admin/queues');

  createBullBoard({
    queues: [
      new BullMQAdapter(emailQueue),
      new BullMQAdapter(imageQueue),
      new BullMQAdapter(reportQueue),
    ],
    serverAdapter,
    options: {
      uiConfig: {
        boardTitle: 'My App Queues',
        favIcon: {
          default: 'static/images/logo.svg',
        },
      },
    },
  });

  app.use('/admin/queues', serverAdapter.getRouter());
  
  console.log('🎯 Bull Board UI available at: /admin/queues');
}
```

---

## 11. Complete Application Integration

```typescript
// src/index.ts - Queue System Integration

import express from 'express';
import Redis from 'ioredis';
import { emailQueue, queueWelcomeEmail, queuePromotionalEmail } from './queue/emailQueue';
import { imageQueue } from './queue/workers/imageWorker';
import { emailWorker } from './queue/workers/emailWorker';
import { imageWorker } from './queue/workers/imageWorker';
import { QueueMonitor } from './queue/monitoring/QueueMonitor';
import { setupBullBoard } from './queue/dashboard/bullBoard';

const app = express();
app.use(express.json());

// Setup Bull Board
setupBullBoard(app);

// Queue Monitor
const monitor = new QueueMonitor(['emails', 'images', 'reports']);

// ========== API Routes ==========

// User registration
app.post('/api/users/register', async (req, res) => {
  try {
    const { name, email, phone } = req.body;
    
    // สร้าง user ใน DB (simplified)
    const user = { id: Date.now().toString(), name, email, phone };
    
    // Queue background tasks
    await queueWelcomeEmail(user.id, email);
    
    res.status(201).json({
      user,
      message: 'Registration successful! Check your email.',
    });
  } catch (error) {
    res.status(500).json({ error: 'Registration failed' });
  }
});

// Upload and process image
app.post('/api/images/upload', async (req, res) => {
  try {
    const { objectKey, userId } = req.body;
    
    // Queue image processing jobs
    const [resizeJob, thumbnailJob] = await Promise.all([
      imageQueue.add('resize', {
        jobType: 'resize',
        inputKey: objectKey,
        options: { width: 1200, height: 800, format: 'webp', quality: 80 },
        userId,
      }),
      imageQueue.add('thumbnail', {
        jobType: 'thumbnail',
        inputKey: objectKey,
        options: { width: 200, height: 200 },
        userId,
      }),
    ]);
    
    res.json({
      message: 'Image upload received. Processing in background.',
      jobs: {
        resize: resizeJob.id,
        thumbnail: thumbnailJob.id,
      },
    });
  } catch (error) {
    res.status(500).json({ error: 'Upload processing failed' });
  }
});

// Check job status
app.get('/api/jobs/:queueName/:jobId', async (req, res) => {
  try {
    const { queueName, jobId } = req.params;
    
    const queueMap: Record<string, any> = {
      emails: emailQueue,
      images: imageQueue,
    };
    
    const queue = queueMap[queueName];
    if (!queue) {
      return res.status(404).json({ error: 'Queue not found' });
    }
    
    const job = await queue.getJob(jobId);
    if (!job) {
      return res.status(404).json({ error: 'Job not found' });
    }
    
    const state = await job.getState();
    const progress = job.progress;
    
    res.json({
      id: job.id,
      name: job.name,
      state,
      progress,
      data: job.data,
      returnvalue: job.returnvalue,
      failedReason: job.failedReason,
      attemptsMade: job.attemptsMade,
      timestamp: job.timestamp,
      processedOn: job.processedOn,
      finishedOn: job.finishedOn,
    });
  } catch (error) {
    res.status(500).json({ error: 'Failed to get job status' });
  }
});

// Queue status dashboard
app.get('/api/queues/status', async (req, res) => {
  try {
    const statuses = await monitor.getAllQueueStatuses();
    res.json({ queues: statuses });
  } catch (error) {
    res.status(500).json({ error: 'Failed to get queue status' });
  }
});

// Send promotional emails (bulk)
app.post('/api/campaigns/:campaignId/send', async (req, res) => {
  try {
    const { campaignId } = req.params;
    const { userIds } = req.body;
    
    await queuePromotionalEmail(userIds, campaignId);
    
    res.json({
      message: `Queued ${userIds.length} promotional emails`,
      campaignId,
    });
  } catch (error) {
    res.status(500).json({ error: 'Failed to queue campaign' });
  }
});

// ========== Graceful Shutdown ==========
process.on('SIGTERM', async () => {
  console.log('Shutting down workers...');
  
  await Promise.all([
    emailWorker.close(),
    imageWorker.close(),
    monitor.close(),
  ]);
  
  console.log('Workers closed');
  process.exit(0);
});

app.listen(3000, () => {
  console.log('🚀 Server running on port 3000');
  console.log('📊 Bull Board: http://localhost:3000/admin/queues');
});
```

---

## 12. Dead Letter Queue

```typescript
// src/queue/DeadLetterQueue.ts

import { Queue, Worker, Job } from 'bullmq';
import { redisConnection } from './config';

export interface DLQEntry {
  originalQueue: string;
  originalJobName: string;
  originalData: any;
  error: string;
  attempts: number;
  failedAt: Date;
}

const dlq = new Queue<DLQEntry>('dead-letter', {
  connection: redisConnection,
  defaultJobOptions: {
    removeOnComplete: false,  // เก็บตลอด
    removeOnFail: false,
  },
});

// สร้าง Worker ที่เฝ้า failed jobs แล้วส่งไป DLQ
export function setupDLQForWorker(worker: Worker, queueName: string): void {
  worker.on('failed', async (job, error) => {
    if (!job) return;
    
    // ตรวจสอบว่าเป็น final failure
    const maxAttempts = job.opts.attempts ?? 3;
    if (job.attemptsMade < maxAttempts) return;  // ยังมี retry เหลืออยู่
    
    console.warn(`[DLQ] Moving job ${job.id} from ${queueName} to dead letter queue`);
    
    await dlq.add('failed-job', {
      originalQueue: queueName,
      originalJobName: job.name,
      originalData: job.data,
      error: error.message,
      attempts: job.attemptsMade,
      failedAt: new Date(),
    }, {
      jobId: `dlq:${queueName}:${job.id}`,  // unique ID
    });
  });
}

// Retry jobs from DLQ manually
export async function retryFromDLQ(jobId: string): Promise<void> {
  const job = await dlq.getJob(jobId);
  if (!job) throw new Error(`DLQ job ${jobId} not found`);
  
  const { originalQueue, originalJobName, originalData } = job.data;
  
  const queue = new Queue(originalQueue, { connection: redisConnection });
  await queue.add(originalJobName, originalData, {
    priority: 1,   // High priority เพราะ manual retry
    attempts: 5,
  });
  
  await job.remove();
  await queue.close();
  
  console.log(`[DLQ] Retried job ${jobId} → ${originalQueue}:${originalJobName}`);
}

// Get all DLQ jobs
export async function getDLQJobs(limit: number = 50): Promise<any[]> {
  const jobs = await dlq.getFailed(0, limit);
  const waiting = await dlq.getWaiting(0, limit);
  
  return [...waiting, ...jobs].map(job => ({
    id: job.id,
    ...job.data,
  }));
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **ทำไมต้องใช้ Queue**: ทำให้ API Response เร็วขึ้น, Handle background tasks, Reliability

2. **BullMQ Concepts**:
   - Queue: เพิ่ม jobs
   - Worker: process jobs
   - QueueEvents: monitor events
   - FlowProducer: parent-child jobs

3. **Job Options**:
   - attempts, backoff: retry strategy
   - delay: delayed jobs
   - priority: job priority
   - jobId: deduplication

4. **Patterns**:
   - Scheduled jobs ด้วย cron
   - Delayed jobs
   - Parent-child flows
   - Dead Letter Queue

5. **Monitoring**:
   - QueueEvents listeners
   - Bull Board UI
   - Queue status API

BullMQ ทำให้ระบบ Background Processing มีความน่าเชื่อถือสูง พร้อม retry และ monitoring ครบครัน
