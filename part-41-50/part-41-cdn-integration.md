# Part 41: CDN Integration กับ S3

## บทนำ: CDN คืออะไร?

CDN (Content Delivery Network) คือเครือข่ายเซิร์ฟเวอร์ที่กระจายอยู่ทั่วโลก ทำหน้าที่จัดส่งเนื้อหา (content) ให้กับผู้ใช้ โดยเลือกเซิร์ฟเวอร์ที่ใกล้กับผู้ใช้ที่สุด แทนที่จะต้องดึงข้อมูลจาก origin server โดยตรงทุกครั้ง

### ทำไมต้องใช้ CDN?

```
ไม่มี CDN:
ผู้ใช้ในไทย → ข้ามมหาสมุทร → Server ที่ US East → latency ~200ms

มี CDN:
ผู้ใช้ในไทย → Edge Server ในสิงคโปร์ → latency ~20ms (10x เร็วกว่า)
```

### ปัญหาที่ CDN แก้ไข

1. **High Latency**: ผู้ใช้อยู่ไกลจาก origin server
2. **High Origin Load**: ทุก request ต้องผ่าน origin ทำให้ server รับภาระหนัก
3. **Bandwidth Cost**: การส่งข้อมูลจาก origin มีค่าใช้จ่ายสูง
4. **Availability**: ถ้า origin ล่ม CDN ยังสามารถ serve cached content ได้
5. **DDoS Protection**: CDN ดูดซับ traffic ที่ผิดปกติได้

## CDN Architecture: ทำงานอย่างไร?

```
┌─────────────────────────────────────────────────────────┐
│                    CDN Architecture                       │
│                                                           │
│  Origin Server (S3)                                       │
│       │                                                   │
│       ├── PoP (Point of Presence) - Singapore             │
│       │         └── Edge Cache                            │
│       │                                                   │
│       ├── PoP - Tokyo                                     │
│       │         └── Edge Cache                            │
│       │                                                   │
│       ├── PoP - Frankfurt                                 │
│       │         └── Edge Cache                            │
│       │                                                   │
│       └── PoP - New York                                  │
│                 └── Edge Cache                            │
│                                                           │
│  Users → ใกล้ PoP ไหน → request ไปที่นั่น                │
└─────────────────────────────────────────────────────────┘
```

### CDN Cache Flow

```
1. User requests: cdn.example.com/images/photo.jpg
2. CDN checks Edge Cache:
   - HIT: return cached file immediately (fast!)
   - MISS: fetch from origin (S3), cache it, return to user
3. Next request: CDN serves from cache (until TTL expires)
```

## CDN Use Cases

### Static Assets ที่เหมาะกับ CDN

```
✅ Images: JPEG, PNG, WebP, AVIF
✅ Videos: MP4, WebM, HLS streams
✅ CSS stylesheets
✅ JavaScript bundles
✅ HTML files (static sites)
✅ Fonts: WOFF, WOFF2, TTF
✅ Documents: PDF, Excel, Word
✅ Software downloads: ZIP, EXE, DMG
✅ Game assets: textures, audio files
✅ API responses (ที่ cache ได้)
```

### ไม่เหมาะกับ CDN

```
❌ Dynamic API responses ที่ต้องการ real-time data
❌ Authentication endpoints
❌ File uploads
❌ WebSocket connections
❌ Server-Sent Events
```

## CDN Providers เปรียบเทียบ

### 1. AWS CloudFront

```
ข้อดี:
- Integrate กับ AWS ecosystem (S3, Lambda, EC2)
- 450+ PoP ทั่วโลก
- Lambda@Edge สำหรับ edge computing
- Origin Shield เพิ่ม cache hit rate
- Real-time logs ด้วย Kinesis

ข้อเสีย:
- ค่าใช้จ่ายซับซ้อน
- Configuration ยุ่งยาก

ราคา (2024):
- Data transfer: $0.0085-0.12/GB (ขึ้นกับ region)
- HTTPS requests: $0.0100/10,000 requests
- Lambda@Edge: $0.60/1M requests
```

### 2. Cloudflare

```
ข้อดี:
- Free tier ใช้งานได้จริง
- Workers สำหรับ edge computing
- R2 Storage: zero egress fees!
- DDoS protection ระดับ enterprise
- 310+ cities

ข้อเสีย:
- ต้องย้าย DNS ไป Cloudflare

ราคา:
- Free: unlimited requests, bandwidth
- Pro: $20/month
- Business: $200/month
- R2: $0.015/GB stored, $0 egress
```

### 3. Fastly

```
ข้อดี:
- Real-time purging (<150ms)
- VCL (Varnish Configuration Language)
- Compute@Edge

ข้อเสีย:
- ราคาแพงกว่า CloudFront
- Setup ยุ่งยาก

ราคา: $0.12/GB (อ้างอิง)
```

### 4. BunnyCDN

```
ข้อดี:
- ราคาถูกมาก
- ง่ายต่อการใช้งาน
- Storage (BunnyCDN Storage) รวมอยู่ด้วย
- 114 PoP

ราคา:
- Europe & North America: $0.01/GB
- Asia: $0.03/GB
- Storage: $0.02/GB/month
```

### สรุปการเลือก CDN

```
Use case                → CDN แนะนำ
AWS S3 origin          → CloudFront (ecosystem integration)
Cost-sensitive         → BunnyCDN หรือ Cloudflare R2
Enterprise security    → Cloudflare Business
Real-time purge        → Fastly
Global startup         → Cloudflare Free
```

## CloudFront + S3 Setup แบบละเอียด

### Prerequisites

```bash
# ติดตั้ง AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure credentials
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region: ap-southeast-1
# Default output format: json
```

### Step 1: สร้าง S3 Bucket (Private)

```bash
# สร้าง bucket (ต้องเป็น globally unique name)
aws s3 mb s3://my-app-assets-prod-2024 --region ap-southeast-1

# Block all public access
aws s3api put-public-access-block \
  --bucket my-app-assets-prod-2024 \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Verify
aws s3api get-public-access-block --bucket my-app-assets-prod-2024
```

### Step 2: Origin Access Control (OAC) - วิธีใหม่

OAC คือวิธีใหม่ที่ AWS แนะนำ แทน OAI (Origin Access Identity) แบบเดิม

```bash
# สร้าง OAC
aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "my-app-s3-oac",
    "Description": "OAC for my-app S3 bucket",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }'
```

Response จะได้ OAC ID เช่น `E2QWRUHEXAMPLE`

### Step 3: สร้าง CloudFront Distribution

```json
// cloudfront-distribution-config.json
{
  "Comment": "My App CDN",
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "S3-my-app-assets",
        "DomainName": "my-app-assets-prod-2024.s3.ap-southeast-1.amazonaws.com",
        "S3OriginConfig": {
          "OriginAccessIdentity": ""
        },
        "OriginAccessControlId": "E2QWRUHEXAMPLE"
      }
    ]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3-my-app-assets",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true,
    "AllowedMethods": {
      "Quantity": 2,
      "Items": ["GET", "HEAD"]
    }
  },
  "Enabled": true,
  "PriceClass": "PriceClass_200",
  "HttpVersion": "http2and3"
}
```

```bash
aws cloudfront create-distribution \
  --distribution-config file://cloudfront-distribution-config.json
```

### Step 4: อัปเดต S3 Bucket Policy

หลังสร้าง distribution แล้ว ต้องให้สิทธิ์ CloudFront เข้าถึง S3

```json
// s3-bucket-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-app-assets-prod-2024/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
        }
      }
    }
  ]
}
```

```bash
aws s3api put-bucket-policy \
  --bucket my-app-assets-prod-2024 \
  --policy file://s3-bucket-policy.json
```

## Cache Behaviors Configuration

### Cache Policy: Managed policies ของ AWS

```
CachingOptimized (สำหรับ S3):
ID: 658327ea-f89d-4fab-a63d-7e88639e58f6
- TTL Default: 86400 (1 day)
- TTL Max: 31536000 (1 year)
- TTL Min: 1
- Compress: yes

CachingDisabled (สำหรับ API):
ID: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad
- TTL: 0
- ใช้สำหรับ dynamic content

CachingOptimizedForUncompressedObjects:
ID: b2884449-e4de-46a7-ac36-70bc7f1ddd6d
- เหมือน CachingOptimized แต่ไม่ compress
```

### Custom Cache Behavior สำหรับ paths ต่างๆ

```typescript
// infrastructure/cloudfront.ts
import * as aws from "@pulumi/aws";

const distribution = new aws.cloudfront.Distribution("app-cdn", {
  origins: [{
    originId: "s3-origin",
    domainName: bucket.bucketRegionalDomainName,
    s3OriginConfig: {
      originAccessIdentity: "",
    },
    originAccessControlId: oac.id,
  }],
  
  // Default behavior: cache ทุกอย่าง
  defaultCacheBehavior: {
    targetOriginId: "s3-origin",
    viewerProtocolPolicy: "redirect-to-https",
    compress: true,
    allowedMethods: ["GET", "HEAD"],
    cachedMethods: ["GET", "HEAD"],
    defaultTtl: 86400,       // 1 day
    maxTtl: 31536000,        // 1 year  
    minTtl: 0,
    forwardedValues: {
      queryString: false,
      cookies: { forward: "none" },
    },
  },
  
  // Ordered cache behaviors (ตรวจสอบตามลำดับ)
  orderedCacheBehaviors: [
    // Images: cache นานมาก
    {
      pathPattern: "/images/*",
      targetOriginId: "s3-origin",
      viewerProtocolPolicy: "redirect-to-https",
      compress: true,
      allowedMethods: ["GET", "HEAD"],
      cachedMethods: ["GET", "HEAD"],
      defaultTtl: 604800,    // 1 week
      maxTtl: 31536000,      // 1 year
      minTtl: 0,
      forwardedValues: {
        queryString: true,   // forward query strings (สำหรับ image transforms)
        queryStringCacheKeys: ["w", "h", "format", "q"],
        cookies: { forward: "none" },
      },
    },
    // Videos: cache นาน แต่ support Range requests
    {
      pathPattern: "/videos/*",
      targetOriginId: "s3-origin",
      viewerProtocolPolicy: "redirect-to-https",
      compress: false,       // videos ไม่ต้อง compress ซ้ำ
      allowedMethods: ["GET", "HEAD"],
      cachedMethods: ["GET", "HEAD"],
      defaultTtl: 86400,
      maxTtl: 2592000,       // 30 days
      minTtl: 0,
      forwardedValues: {
        queryString: false,
        headers: ["Range"],  // Support video seek
        cookies: { forward: "none" },
      },
    },
    // Static assets ที่มี hash: cache ยาวมาก
    {
      pathPattern: "/static/*",
      targetOriginId: "s3-origin",
      viewerProtocolPolicy: "redirect-to-https",
      compress: true,
      allowedMethods: ["GET", "HEAD"],
      cachedMethods: ["GET", "HEAD"],
      defaultTtl: 31536000,  // 1 year (เพราะ filename มี hash)
      maxTtl: 31536000,
      minTtl: 31536000,
      forwardedValues: {
        queryString: false,
        cookies: { forward: "none" },
      },
    },
  ],
  
  enabled: true,
  httpVersion: "http2and3",
  priceClass: "PriceClass_200",
  
  restrictions: {
    geoRestriction: {
      restrictionType: "none",
    },
  },
  
  viewerCertificate: {
    cloudfrontDefaultCertificate: true,
    // หรือใช้ ACM certificate สำหรับ custom domain
  },
});
```

## TTL Strategy

### Cache-Control Headers Strategy

```typescript
// upload กับ S3 พร้อมกำหนด Cache-Control
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

const s3 = new S3Client({ region: "ap-southeast-1" });

interface UploadOptions {
  key: string;
  body: Buffer;
  contentType: string;
  cacheControl?: string;
}

function getCacheControl(filePath: string): string {
  // Static assets ที่มี content hash ใน filename: cache ยาวมาก
  if (/\.[a-f0-9]{8,}\.(js|css|woff2?)$/.test(filePath)) {
    return "public, max-age=31536000, immutable";
  }
  
  // Images: cache 1 สัปดาห์
  if (/\.(jpg|jpeg|png|webp|avif|gif|svg)$/.test(filePath)) {
    return "public, max-age=604800, stale-while-revalidate=86400";
  }
  
  // Videos: cache 1 วัน
  if (/\.(mp4|webm|ogg|m3u8|ts)$/.test(filePath)) {
    return "public, max-age=86400";
  }
  
  // Documents: cache 1 ชั่วโมง
  if (/\.(pdf|docx|xlsx)$/.test(filePath)) {
    return "public, max-age=3600";
  }
  
  // HTML: no-cache (always validate)
  if (/\.html$/.test(filePath)) {
    return "public, no-cache";
  }
  
  // Default: 1 วัน
  return "public, max-age=86400";
}

async function uploadToS3(options: UploadOptions) {
  const command = new PutObjectCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: options.key,
    Body: options.body,
    ContentType: options.contentType,
    CacheControl: options.cacheControl ?? getCacheControl(options.key),
    // Metadata
    Metadata: {
      "uploaded-at": new Date().toISOString(),
      "uploaded-by": "app-server",
    },
  });
  
  return s3.send(command);
}
```

## CloudFront Signed URLs

### เมื่อไหร่ต้องใช้ Signed URLs?

```
Use cases:
- Premium content ที่ต้องการ authentication
- Time-limited download links
- User-specific content
- Pay-per-download

ตัวอย่าง:
- วิดีโอ course (ดูได้แค่ผู้ที่ subscribe)
- เอกสาร confidential
- Invoice PDF
- ไฟล์ที่ download ได้แค่ 24 ชั่วโมง
```

### Setup Key Pairs

```bash
# สร้าง RSA key pair
openssl genrsa -out cloudfront-private-key.pem 2048
openssl rsa -pubout -in cloudfront-private-key.pem -out cloudfront-public-key.pem

# อัปโหลด public key ไป CloudFront
aws cloudfront create-public-key \
  --public-key-config '{
    "CallerReference": "my-app-key-2024",
    "Name": "MyAppSigningKey",
    "EncodedKey": "'$(cat cloudfront-public-key.pem)'"
  }'

# สร้าง Key Group
aws cloudfront create-key-group \
  --key-group-config '{
    "Name": "MyAppKeyGroup",
    "Items": ["K2JCJMDEHXQW5F"]
  }'
```

### Signed URL Generation ใน Node.js/TypeScript

```typescript
// src/services/cdn.service.ts
import { getSignedUrl } from "@aws-sdk/cloudfront-signer";
import { readFileSync } from "fs";
import { join } from "path";

const CLOUDFRONT_URL = process.env.CLOUDFRONT_URL!; // https://d1234abcd.cloudfront.net
const KEY_PAIR_ID = process.env.CLOUDFRONT_KEY_PAIR_ID!;
const PRIVATE_KEY = process.env.NODE_ENV === "production"
  ? process.env.CLOUDFRONT_PRIVATE_KEY!  // จาก environment variable
  : readFileSync(join(__dirname, "../../keys/cloudfront-private-key.pem"), "utf-8");

interface SignedUrlOptions {
  key: string;           // S3 object key
  expiresIn?: number;    // seconds (default: 3600)
  ipAddress?: string;    // restrict to specific IP
}

export async function generateSignedUrl(options: SignedUrlOptions): Promise<string> {
  const { key, expiresIn = 3600, ipAddress } = options;
  
  const url = `${CLOUDFRONT_URL}/${key}`;
  const dateLessThan = new Date(Date.now() + expiresIn * 1000).toISOString();
  
  // Custom policy สำหรับ IP restriction
  if (ipAddress) {
    const policy = JSON.stringify({
      Statement: [{
        Resource: url,
        Condition: {
          DateLessThan: {
            "AWS:EpochTime": Math.floor(Date.now() / 1000) + expiresIn,
          },
          IpAddress: {
            "AWS:SourceIp": `${ipAddress}/32`,
          },
        },
      }],
    });
    
    return getSignedUrl({
      url,
      keyPairId: KEY_PAIR_ID,
      privateKey: PRIVATE_KEY,
      policy,
    });
  }
  
  // Canned policy (ง่ายกว่า, ไม่มี IP restriction)
  return getSignedUrl({
    url,
    keyPairId: KEY_PAIR_ID,
    privateKey: PRIVATE_KEY,
    dateLessThan,
  });
}

// ตัวอย่างการใช้งาน
export async function getSecureDownloadUrl(
  userId: string,
  fileKey: string,
  userIp?: string
): Promise<{ url: string; expiresAt: Date }> {
  const expiresIn = 3600; // 1 ชั่วโมง
  
  const url = await generateSignedUrl({
    key: `users/${userId}/documents/${fileKey}`,
    expiresIn,
    ipAddress: userIp,
  });
  
  const expiresAt = new Date(Date.now() + expiresIn * 1000);
  
  return { url, expiresAt };
}
```

### Signed Cookies (สำหรับ Video Streaming)

```typescript
// Signed Cookies ดีกว่าสำหรับ HLS/DASH streams เพราะ URL เปลี่ยนตลอด
import { getSignedCookies } from "@aws-sdk/cloudfront-signer";

interface SignedCookiesOptions {
  resourcePattern: string;  // wildcard pattern: "https://cdn.example.com/course/123/*"
  expiresIn?: number;
}

export async function generateSignedCookies(
  options: SignedCookiesOptions
): Promise<Record<string, string>> {
  const { resourcePattern, expiresIn = 43200 } = options; // 12 ชั่วโมง
  
  const policy = JSON.stringify({
    Statement: [{
      Resource: resourcePattern,
      Condition: {
        DateLessThan: {
          "AWS:EpochTime": Math.floor(Date.now() / 1000) + expiresIn,
        },
      },
    }],
  });
  
  const cookies = getSignedCookies({
    keyPairId: KEY_PAIR_ID,
    privateKey: PRIVATE_KEY,
    policy,
  });
  
  return cookies;
}

// ใน Express route
app.get("/api/courses/:courseId/watch", authenticate, async (req, res) => {
  const { courseId } = req.params;
  const userId = req.user.id;
  
  // ตรวจสอบ subscription
  const hasAccess = await checkCourseAccess(userId, courseId);
  if (!hasAccess) {
    return res.status(403).json({ error: "Access denied" });
  }
  
  // สร้าง signed cookies
  const cookies = await generateSignedCookies({
    resourcePattern: `${CLOUDFRONT_URL}/courses/${courseId}/*`,
    expiresIn: 43200, // 12 ชั่วโมง
  });
  
  // Set cookies
  res.cookie("CloudFront-Policy", cookies["CloudFront-Policy"], {
    httpOnly: true,
    secure: true,
    sameSite: "strict",
    maxAge: 43200 * 1000,
  });
  res.cookie("CloudFront-Signature", cookies["CloudFront-Signature"], {
    httpOnly: true,
    secure: true,
    sameSite: "strict",
    maxAge: 43200 * 1000,
  });
  res.cookie("CloudFront-Key-Pair-Id", cookies["CloudFront-Key-Pair-Id"], {
    httpOnly: true,
    secure: true,
    sameSite: "strict",
    maxAge: 43200 * 1000,
  });
  
  // Return video manifest URL
  res.json({
    manifestUrl: `${CLOUDFRONT_URL}/courses/${courseId}/master.m3u8`,
    expiresAt: new Date(Date.now() + 43200 * 1000).toISOString(),
  });
});
```

## Cache Invalidation

### เมื่อไหร่ต้อง Invalidate?

```
1. อัปเดตไฟล์ที่มี URL เดิม (เช่น logo.png)
2. ลบเนื้อหา (GDPR requests)
3. แก้ไขข้อผิดพลาดที่ไฟล์ถูก cache แล้ว
4. Force update ทันที (ไม่รอ TTL หมด)
```

### CloudFront Invalidation API

```typescript
// src/services/cdn-invalidation.service.ts
import {
  CloudFrontClient,
  CreateInvalidationCommand,
} from "@aws-sdk/client-cloudfront";

const cloudfront = new CloudFrontClient({ region: "us-east-1" });

const DISTRIBUTION_ID = process.env.CLOUDFRONT_DISTRIBUTION_ID!;

interface InvalidationOptions {
  paths: string[];  // เช่น ["/images/logo.png", "/static/*"]
  callerReference?: string;
}

export async function invalidateCache(options: InvalidationOptions): Promise<string> {
  const { paths, callerReference = `invalidation-${Date.now()}` } = options;
  
  // Normalize paths (ต้องขึ้นต้นด้วย /)
  const normalizedPaths = paths.map(p => p.startsWith("/") ? p : `/${p}`);
  
  const command = new CreateInvalidationCommand({
    DistributionId: DISTRIBUTION_ID,
    InvalidationBatch: {
      Paths: {
        Quantity: normalizedPaths.length,
        Items: normalizedPaths,
      },
      CallerReference: callerReference,
    },
  });
  
  const response = await cloudfront.send(command);
  
  return response.Invalidation!.Id!;
}

// ตัวอย่างการใช้งาน
async function afterFileUpdate(key: string) {
  // Invalidate specific file
  await invalidateCache({ paths: [`/${key}`] });
}

async function afterBatchUpdate(prefix: string) {
  // Invalidate ทุกไฟล์ใน prefix
  await invalidateCache({ paths: [`/${prefix}/*`] });
}

// ⚠️ ระวัง: Wildcard invalidation ราคาแพง
// AWS คิด $0.005 per invalidation path (min 1000 paths/month free)
// /*  = count เป็น 1 path ราคาถูกกว่า list ทุกไฟล์
```

### Versioned Files (วิธีที่ดีกว่า Invalidation)

```typescript
// แทนที่จะ invalidate, ใช้ content-hash ใน filename
// เช่น logo.abc12345.png แทน logo.png

import { createHash } from "crypto";
import { readFileSync } from "fs";

function getContentHash(buffer: Buffer): string {
  return createHash("md5").update(buffer).digest("hex").slice(0, 8);
}

async function uploadVersionedFile(
  filePath: string,
  fileBuffer: Buffer,
  mimeType: string
): Promise<string> {
  const hash = getContentHash(fileBuffer);
  const ext = filePath.split(".").pop();
  const baseName = filePath.replace(`.${ext}`, "");
  
  // เช่น: images/logo.png → images/logo.abc12345.png
  const versionedKey = `${baseName}.${hash}.${ext}`;
  
  await uploadToS3({
    key: versionedKey,
    body: fileBuffer,
    contentType: mimeType,
    cacheControl: "public, max-age=31536000, immutable", // cache 1 ปี
  });
  
  return versionedKey;
}
```

## Cloudflare R2 + CDN (Zero Egress Fees)

### ทำไม Cloudflare R2 น่าสนใจ?

```
AWS S3 Egress fees:
- ข้อมูลออก S3 → Internet: $0.09/GB (us-east-1)
- 1TB/month = $90 แค่ค่า egress!

Cloudflare R2:
- Egress fees: $0 !!!
- Storage: $0.015/GB/month
- Operations: $4.50/million Class A, $0.36/million Class B

เหมาะสำหรับ: media-heavy applications, large file storage
```

### Setup Cloudflare R2

```typescript
// ใช้ S3-compatible API
import { S3Client, PutObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

// R2 ใช้ S3-compatible API
const r2Client = new S3Client({
  region: "auto",
  endpoint: `https://${process.env.CLOUDFLARE_ACCOUNT_ID}.r2.cloudflarestorage.com`,
  credentials: {
    accessKeyId: process.env.R2_ACCESS_KEY_ID!,
    secretAccessKey: process.env.R2_SECRET_ACCESS_KEY!,
  },
});

// Upload ไป R2 (เหมือน S3)
async function uploadToR2(key: string, body: Buffer, contentType: string) {
  const command = new PutObjectCommand({
    Bucket: process.env.R2_BUCKET_NAME!,
    Key: key,
    Body: body,
    ContentType: contentType,
  });
  
  return r2Client.send(command);
}

// สร้าง presigned URL สำหรับ direct upload จาก browser
async function createPresignedUploadUrl(key: string, contentType: string) {
  const command = new PutObjectCommand({
    Bucket: process.env.R2_BUCKET_NAME!,
    Key: key,
    ContentType: contentType,
  });
  
  return getSignedUrl(r2Client, command, { expiresIn: 3600 });
}

// Public URL (ถ้า bucket เปิด public access)
function getPublicUrl(key: string): string {
  return `https://${process.env.CLOUDFLARE_CUSTOM_DOMAIN}/${key}`;
  // เช่น: https://assets.example.com/images/photo.jpg
}
```

### Cloudflare Workers: Image Transformation

```javascript
// workers/image-transform.js
// Deploy ด้วย Wrangler CLI

export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    
    // Parse transform parameters
    const width = parseInt(url.searchParams.get("w") || "0");
    const height = parseInt(url.searchParams.get("h") || "0");
    const format = url.searchParams.get("format") || "auto";
    const quality = parseInt(url.searchParams.get("q") || "85");
    const fit = url.searchParams.get("fit") || "cover";
    
    // Strip query params to get original image path
    const imagePath = url.pathname;
    
    // Cloudflare Image Resizing (ต้องการ paid plan)
    const imageRequest = new Request(
      `https://your-r2-bucket.example.com${imagePath}`,
      {
        cf: {
          image: {
            width: width || undefined,
            height: height || undefined,
            fit,
            format,
            quality,
          },
        },
      }
    );
    
    const response = await fetch(imageRequest);
    
    // Add cache headers
    const headers = new Headers(response.headers);
    headers.set("Cache-Control", "public, max-age=31536000");
    headers.set("Vary", "Accept");
    
    return new Response(response.body, {
      status: response.status,
      headers,
    });
  },
};
```

## MinIO + Nginx as CDN Proxy (Self-hosted)

### Use case: On-premise หรือ Private cloud

```yaml
# docker-compose.yml
version: "3.8"

services:
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
    volumes:
      - minio-data:/data
    ports:
      - "9000:9000"
      - "9001:9001"
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 30s
      timeout: 20s
      retries: 3

  nginx-cdn:
    image: nginx:alpine
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/cache:/var/cache/nginx
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - minio

volumes:
  minio-data:
```

```nginx
# nginx/nginx.conf
proxy_cache_path /var/cache/nginx 
  levels=1:2 
  keys_zone=cdn_cache:10m 
  max_size=10g 
  inactive=1d 
  use_temp_path=off;

server {
    listen 80;
    server_name cdn.example.local;
    
    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name cdn.example.local;
    
    ssl_certificate /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;
    
    # Cache zone
    proxy_cache cdn_cache;
    proxy_cache_valid 200 7d;
    proxy_cache_valid 404 1m;
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    proxy_cache_background_update on;
    proxy_cache_lock on;
    
    # Add cache status header
    add_header X-Cache-Status $upstream_cache_status;
    
    location / {
        # Proxy to MinIO
        proxy_pass http://minio:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        
        # Cache key: URL only (ignore auth headers for public content)
        proxy_cache_key $scheme$proxy_host$request_uri;
        
        # Gzip compression
        gzip on;
        gzip_types text/plain text/css application/json application/javascript image/svg+xml;
        
        # Browser cache headers
        expires 7d;
        add_header Cache-Control "public, max-age=604800, stale-while-revalidate=86400";
    }
    
    # Private content: no CDN cache
    location /private/ {
        proxy_cache off;
        proxy_pass http://minio:9000;
        # Validate auth here
    }
}
```

## Image Optimization CDN: Cloudinary vs Self-hosted

### Cloudinary (Managed)

```typescript
// ง่ายมาก แต่ราคาแพง
import { v2 as cloudinary } from "cloudinary";

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET,
});

// Upload
const result = await cloudinary.uploader.upload(fileBuffer, {
  folder: "user-avatars",
  transformation: [
    { width: 400, height: 400, crop: "fill", gravity: "face" },
    { quality: "auto", fetch_format: "auto" },
  ],
});

// URL-based transformation (ไม่ต้อง process ล่วงหน้า)
const url = cloudinary.url("user-avatars/photo123", {
  width: 200,
  height: 200,
  crop: "thumb",
  gravity: "face",
  quality: "auto",
  format: "webp",
});
// → https://res.cloudinary.com/myapp/image/upload/w_200,h_200,c_thumb,g_face,q_auto,f_webp/user-avatars/photo123.webp
```

### CloudFront + Lambda@Edge (Self-hosted)

```typescript
// lambda-edge/image-resize.ts
// Deploy ที่ us-east-1 (Lambda@Edge requirement)

import { CloudFrontRequestEvent, CloudFrontRequestResult } from "aws-lambda";
import sharp from "sharp";
import { S3Client, GetObjectCommand } from "@aws-sdk/client-s3";

const s3 = new S3Client({ region: "us-east-1" });

export const handler = async (
  event: CloudFrontRequestEvent
): Promise<CloudFrontRequestResult> => {
  const request = event.Records[0].cf.request;
  const { uri, querystring } = request;
  
  // Parse query parameters
  const params = new URLSearchParams(querystring || "");
  const width = parseInt(params.get("w") || "0");
  const height = parseInt(params.get("h") || "0");
  const format = params.get("format") as "webp" | "avif" | "jpeg" | "png" | null;
  const quality = parseInt(params.get("q") || "85");
  const fit = (params.get("fit") || "cover") as sharp.FitEnum[keyof sharp.FitEnum];
  
  // ถ้าไม่มี transform params ให้ pass through
  if (!width && !height && !format) {
    return request;
  }
  
  try {
    // ดึงรูปจาก S3
    const s3Response = await s3.send(new GetObjectCommand({
      Bucket: process.env.S3_BUCKET!,
      Key: uri.slice(1), // remove leading /
    }));
    
    const inputBuffer = Buffer.from(await (s3Response.Body as any).transformToByteArray());
    
    // Process image
    let pipeline = sharp(inputBuffer);
    
    if (width || height) {
      pipeline = pipeline.resize(width || null, height || null, {
        fit,
        withoutEnlargement: true,
      });
    }
    
    if (format) {
      pipeline = pipeline.toFormat(format, { quality });
    }
    
    const outputBuffer = await pipeline.toBuffer();
    
    // Return processed image
    return {
      status: "200",
      statusDescription: "OK",
      headers: {
        "content-type": [{ key: "Content-Type", value: `image/${format || "jpeg"}` }],
        "cache-control": [{ key: "Cache-Control", value: "public, max-age=31536000" }],
        "vary": [{ key: "Vary", value: "Accept" }],
      },
      body: outputBuffer.toString("base64"),
      bodyEncoding: "base64",
    };
  } catch (error) {
    console.error("Image processing error:", error);
    return request; // Fallback: serve original
  }
};
```

## Monitoring: CDN Hit Rate

```typescript
// src/monitoring/cdn-metrics.ts
import {
  CloudWatchClient,
  GetMetricStatisticsCommand,
} from "@aws-sdk/client-cloudwatch";

const cloudwatch = new CloudWatchClient({ region: "us-east-1" });

export async function getCDNMetrics(distributionId: string, hours: number = 24) {
  const endTime = new Date();
  const startTime = new Date(endTime.getTime() - hours * 60 * 60 * 1000);
  
  const [cacheHitRate, totalRequests, bytesDownloaded, originRequests] =
    await Promise.all([
      // Cache Hit Rate
      cloudwatch.send(new GetMetricStatisticsCommand({
        Namespace: "AWS/CloudFront",
        MetricName: "CacheHitRate",
        Dimensions: [{ Name: "DistributionId", Value: distributionId }],
        StartTime: startTime,
        EndTime: endTime,
        Period: 3600,
        Statistics: ["Average"],
      })),
      
      // Total Requests
      cloudwatch.send(new GetMetricStatisticsCommand({
        Namespace: "AWS/CloudFront",
        MetricName: "Requests",
        Dimensions: [{ Name: "DistributionId", Value: distributionId }],
        StartTime: startTime,
        EndTime: endTime,
        Period: 3600,
        Statistics: ["Sum"],
      })),
      
      // Bytes Downloaded
      cloudwatch.send(new GetMetricStatisticsCommand({
        Namespace: "AWS/CloudFront",
        MetricName: "BytesDownloaded",
        Dimensions: [{ Name: "DistributionId", Value: distributionId }],
        StartTime: startTime,
        EndTime: endTime,
        Period: 3600,
        Statistics: ["Sum"],
      })),
      
      // Origin Requests (cache misses)
      cloudwatch.send(new GetMetricStatisticsCommand({
        Namespace: "AWS/CloudFront",
        MetricName: "OriginLatency",
        Dimensions: [{ Name: "DistributionId", Value: distributionId }],
        StartTime: startTime,
        EndTime: endTime,
        Period: 3600,
        Statistics: ["Average"],
      })),
    ]);
  
  return {
    cacheHitRate: cacheHitRate.Datapoints?.map(d => ({
      time: d.Timestamp,
      value: d.Average,
    })),
    totalRequests: totalRequests.Datapoints?.reduce(
      (sum, d) => sum + (d.Sum || 0), 0
    ),
    bytesDownloaded: bytesDownloaded.Datapoints?.reduce(
      (sum, d) => sum + (d.Sum || 0), 0
    ),
  };
}
```

## Bandwidth Cost Optimization

### กลยุทธ์ลดค่าใช้จ่าย

```
1. เพิ่ม Cache Hit Rate:
   - ตั้ง TTL นานๆ สำหรับ static assets
   - ใช้ content-hashing แทน invalidation
   - ลด query string forwarding
   
2. เลือก Price Class ที่เหมาะสม:
   - PriceClass_100: US, Canada, Europe (ถูกสุด)
   - PriceClass_200: + Asia, Middle East, Africa
   - PriceClass_All: ทุก PoP (แพงสุด แต่ performance ดีสุด)
   
3. Compress:
   - เปิด Gzip/Brotli: ลด bandwidth 50-70%
   - WebP แทน JPEG: ลดขนาด 25-35%
   - AVIF แทน JPEG: ลดขนาด 50%
   
4. Origin Shield:
   - เพิ่ม cache layer ระหว่าง CDN กับ origin
   - ลด origin requests
   - เพิ่มค่าใช้จ่าย CDN เล็กน้อย แต่ลดค่า origin มากกว่า
```

## Full Working Example: Media Serving Architecture

```typescript
// src/services/media.service.ts
import { S3Client, PutObjectCommand, DeleteObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { CloudFrontClient, CreateInvalidationCommand } from "@aws-sdk/client-cloudfront";
import sharp from "sharp";
import { createHash } from "crypto";
import { db } from "../db";

const s3 = new S3Client({ region: process.env.AWS_REGION! });
const cloudfront = new CloudFrontClient({ region: "us-east-1" });

interface MediaFile {
  id: string;
  userId: string;
  originalKey: string;
  variants: {
    thumbnail?: string;
    medium?: string;
    large?: string;
    webp?: string;
  };
  metadata: {
    width: number;
    height: number;
    size: number;
    format: string;
  };
  cdnUrl: string;
  createdAt: Date;
}

class MediaService {
  private readonly bucket = process.env.S3_BUCKET!;
  private readonly cdnUrl = process.env.CLOUDFRONT_URL!;
  private readonly distributionId = process.env.CLOUDFRONT_DISTRIBUTION_ID!;
  
  async uploadImage(
    userId: string,
    buffer: Buffer,
    originalFilename: string
  ): Promise<MediaFile> {
    // ดึง metadata
    const metadata = await sharp(buffer).metadata();
    
    // สร้าง content hash
    const hash = createHash("md5").update(buffer).digest("hex").slice(0, 8);
    const ext = "jpg"; // เราจะ convert ทุกอย่างเป็น JPEG/WebP
    
    const baseKey = `users/${userId}/images/${Date.now()}-${hash}`;
    
    // Process variants ใน parallel
    const [original, thumbnail, medium, large, webp] = await Promise.all([
      // Original (ใช้ sharp เพื่อ strip EXIF)
      sharp(buffer)
        .jpeg({ quality: 90, progressive: true })
        .toBuffer(),
      
      // Thumbnail: 150x150
      sharp(buffer)
        .resize(150, 150, { fit: "cover" })
        .jpeg({ quality: 80 })
        .toBuffer(),
      
      // Medium: max 800px
      sharp(buffer)
        .resize(800, 600, { fit: "inside", withoutEnlargement: true })
        .jpeg({ quality: 85, progressive: true })
        .toBuffer(),
      
      // Large: max 1920px
      sharp(buffer)
        .resize(1920, 1080, { fit: "inside", withoutEnlargement: true })
        .jpeg({ quality: 90, progressive: true })
        .toBuffer(),
      
      // WebP สำหรับ modern browsers
      sharp(buffer)
        .resize(1920, 1080, { fit: "inside", withoutEnlargement: true })
        .webp({ quality: 85 })
        .toBuffer(),
    ]);
    
    // Upload ทุก variants ไป S3
    const keys = {
      original: `${baseKey}.${ext}`,
      thumbnail: `${baseKey}_thumb.${ext}`,
      medium: `${baseKey}_medium.${ext}`,
      large: `${baseKey}_large.${ext}`,
      webp: `${baseKey}.webp`,
    };
    
    await Promise.all([
      this.uploadBuffer(keys.original, original, "image/jpeg", "public, max-age=31536000, immutable"),
      this.uploadBuffer(keys.thumbnail, thumbnail, "image/jpeg", "public, max-age=31536000, immutable"),
      this.uploadBuffer(keys.medium, medium, "image/jpeg", "public, max-age=31536000, immutable"),
      this.uploadBuffer(keys.large, large, "image/jpeg", "public, max-age=31536000, immutable"),
      this.uploadBuffer(keys.webp, webp, "image/webp", "public, max-age=31536000, immutable"),
    ]);
    
    // Save to database
    const mediaFile = await db.mediaFile.create({
      data: {
        userId,
        originalKey: keys.original,
        variants: keys,
        metadata: {
          width: metadata.width || 0,
          height: metadata.height || 0,
          size: buffer.length,
          format: metadata.format || "jpeg",
        },
        cdnUrl: `${this.cdnUrl}/${keys.original}`,
      },
    });
    
    return mediaFile;
  }
  
  private async uploadBuffer(
    key: string,
    body: Buffer,
    contentType: string,
    cacheControl: string
  ): Promise<void> {
    await s3.send(new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      Body: body,
      ContentType: contentType,
      CacheControl: cacheControl,
    }));
  }
  
  getVariantUrl(key: string, variant: "original" | "thumbnail" | "medium" | "large" | "webp"): string {
    return `${this.cdnUrl}/${key}`;
  }
  
  async deleteMedia(mediaFile: MediaFile): Promise<void> {
    // ลบทุก variants จาก S3
    const keysToDelete = [
      mediaFile.originalKey,
      ...Object.values(mediaFile.variants),
    ].filter(Boolean);
    
    await Promise.all(
      keysToDelete.map(key =>
        s3.send(new DeleteObjectCommand({
          Bucket: this.bucket,
          Key: key,
        }))
      )
    );
    
    // Invalidate CDN cache
    await cloudfront.send(new CreateInvalidationCommand({
      DistributionId: this.distributionId,
      InvalidationBatch: {
        Paths: {
          Quantity: keysToDelete.length,
          Items: keysToDelete.map(k => `/${k}`),
        },
        CallerReference: `delete-${Date.now()}`,
      },
    }));
    
    // ลบจาก database
    await db.mediaFile.delete({ where: { id: mediaFile.id } });
  }
  
  async createPresignedUploadUrl(
    userId: string,
    filename: string,
    contentType: string
  ): Promise<{ uploadUrl: string; key: string }> {
    const hash = createHash("md5").update(`${userId}-${Date.now()}`).digest("hex").slice(0, 8);
    const ext = filename.split(".").pop();
    const key = `uploads/${userId}/${hash}.${ext}`;
    
    const command = new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      ContentType: contentType,
    });
    
    const uploadUrl = await getSignedUrl(s3, command, { expiresIn: 3600 });
    
    return { uploadUrl, key };
  }
}

export const mediaService = new MediaService();
```

## สรุป: CDN Best Practices

```
1. ใช้ HTTPS everywhere: redirect HTTP → HTTPS
2. ตั้ง Cache-Control headers ที่ถูกต้องที่ origin
3. ใช้ content-hash ใน filenames สำหรับ immutable assets
4. เปิด compression (gzip/brotli)
5. เลือก Price Class ตาม target audience
6. Monitor cache hit rate (เป้าหมาย >90%)
7. ใช้ Signed URLs/Cookies สำหรับ private content
8. พิจารณา Cloudflare R2 สำหรับ zero egress costs
9. ใช้ Origin Shield ถ้า origin รับ load ไม่ไหว
10. Test CDN configuration ด้วย curl -I และดู X-Cache header
```

```bash
# ทดสอบ CDN
curl -I https://cdn.example.com/images/photo.jpg

# ดู headers ที่สำคัญ:
# X-Cache: Hit from cloudfront (หรือ HIT)
# Age: 3600 (อยู่ใน cache 1 ชั่วโมงแล้ว)
# Cache-Control: public, max-age=31536000, immutable
# Content-Encoding: gzip
# Vary: Accept-Encoding
```
