# Part 10: ติดตั้ง MinIO (S3-Compatible) สำหรับ Object Storage

## สารบัญ

1. [Object Storage คืออะไร](#1-object-storage-คืออะไร)
2. [S3 API Concepts](#2-s3-api-concepts)
3. [MinIO คืออะไร](#3-minio-คืออะไร)
4. [ติดตั้ง MinIO ด้วย Docker](#4-ติดตั้ง-minio-ด้วย-docker)
5. [MinIO Console (Web UI)](#5-minio-console-web-ui)
6. [mc (MinIO Client) CLI](#6-mc-minio-client-cli)
7. [Bucket Policies](#7-bucket-policies)
8. [Versioning](#8-versioning)
9. [Lifecycle Policies](#9-lifecycle-policies)
10. [Pre-signed URLs](#10-pre-signed-urls)
11. [Object Metadata และ Tags](#11-object-metadata-และ-tags)
12. [Multipart Upload](#12-multipart-upload)
13. [Server-side Encryption](#13-server-side-encryption)
14. [MinIO vs AWS S3](#14-minio-vs-aws-s3)
15. [Workshop: Upload/Download ด้วย CLI และ SDK](#15-workshop-uploaddownload-ด้วย-cli-และ-sdk)

---

## 1. Object Storage คืออะไร

### 1.1 File System vs Object Storage vs Block Storage

```
File System (NAS/NFS):
├── folder1/
│   ├── file1.jpg
│   └── file2.pdf
└── folder2/
    └── file3.mp4

- มี hierarchy (directories)
- จัดการด้วย path
- เหมาะ: application files, documents
- ข้อจำกัด: ยาก scale, ไม่มี metadata rich

Block Storage (SAN/EBS):
- แบ่ง storage เป็น blocks ขนาดเท่ากัน
- ระบบปฏิบัติการจัดการ filesystem บน blocks
- เหมาะ: database volumes, OS disks
- ข้อจำกัด: expensive, ยาก share

Object Storage (S3/MinIO):
- ไม่มี hierarchy จริงๆ (แต่จำลองได้ด้วย key prefix)
- แต่ละ object มี: Key (path), Data (binary), Metadata
- เข้าถึงผ่าน HTTP API
- เหมาะ: media files, backups, logs, ML datasets
- ข้อดี: scale ไม่จำกัด, cheap, globally accessible, versioning built-in
```

### 1.2 ทำไมต้องใช้ Object Storage

```
✅ ข้อดีของ Object Storage:
1. Scale ได้ไม่จำกัด (petabytes)
2. ราคาถูกกว่า block storage 10x
3. HTTP API ทำให้ access จากทุกที่ได้
4. Built-in versioning, lifecycle, replication
5. Durability สูง (S3: 99.999999999% - 11 nines)
6. Concurrent access จากหลาย applications
7. Global CDN integration ง่าย

✅ Use Cases ที่เหมาะ:
- รูปภาพ profile, product images
- Video/audio files
- Document storage (PDF, Excel)
- Backup files
- Log aggregation
- ML training datasets
- Static website hosting
- Software distribution

❌ ไม่เหมาะสำหรับ:
- Database files (ต้องการ low latency IOPS)
- Frequently modified files
- Real-time streaming (ต้องการ sequential access)
```

---

## 2. S3 API Concepts

### 2.1 Buckets, Objects, Keys, Prefixes

```
S3 Hierarchy:
Account
└── Bucket (my-photos)
    ├── Object: "profile/user-123/avatar.jpg"
    ├── Object: "profile/user-456/avatar.jpg"  
    ├── Object: "products/12345/main.jpg"
    └── Object: "products/12345/gallery/1.jpg"

Bucket:
- Container หลักสำหรับ objects
- ชื่อต้อง globally unique (ใน AWS S3)
- สร้าง bucket เหมือนสร้าง root folder

Object:
- ไฟล์ที่เก็บ (binary data)
- มี metadata: Content-Type, Content-Length, etc.
- สูงสุด 5 TB ต่อ object

Key:
- "path" ของ object ใน bucket
- เช่น "profile/user-123/avatar.jpg"
- ไม่มี hierarchy จริง แต่ prefix จำลองได้

Prefix:
- ส่วนหน้าของ key ที่ใช้ filter
- "profile/" คือ prefix สำหรับ profile images
- ทำงานเหมือน "folder" แต่ไม่ใช่จริงๆ
```

### 2.2 S3 URL Formats

```bash
# Path-style URL (legacy, จะถูก deprecated ใน AWS)
https://s3.amazonaws.com/my-bucket/path/to/file.jpg
https://minio.example.com/my-bucket/path/to/file.jpg

# Virtual-hosted-style URL (แนะนำ)
https://my-bucket.s3.amazonaws.com/path/to/file.jpg
https://my-bucket.minio.example.com/path/to/file.jpg

# S3 URI (สำหรับ CLI tools)
s3://my-bucket/path/to/file.jpg
```

### 2.3 HTTP Methods

```
PUT    /bucket/key          - Upload object
GET    /bucket/key          - Download object
HEAD   /bucket/key          - Get metadata (no body)
DELETE /bucket/key          - Delete object
GET    /bucket?list-type=2  - List objects in bucket
PUT    /bucket              - Create bucket
DELETE /bucket              - Delete bucket
```

---

## 3. MinIO คืออะไร

MinIO คือ open-source, S3-compatible object storage ที่:
- รองรับ S3 API ทุก operation
- ทำงานบน Linux, macOS, Windows, Kubernetes
- License: AGPL v3 (สำหรับ open-source), Commercial license สำหรับ enterprise
- Performance สูง: สูงสุด 183 GB/s read, 171 GB/s write (ใน distributed mode)

### MinIO Deployment Modes

```
1. Single Node Single Drive (SNSD):
   - Development/Testing
   - ไม่มี data redundancy

2. Single Node Multiple Drive (SNMD):
   - Better performance
   - Erasure coding ป้องกันข้อมูล

3. Multi Node Multi Drive (MNMD) - Distributed:
   - Production grade
   - High availability
   - Erasure coding across nodes
   - เหมาะสำหรับ production
```

---

## 4. ติดตั้ง MinIO ด้วย Docker

### 4.1 Single Container (Development)

```bash
# Pull MinIO image
docker pull minio/minio:latest

# Run MinIO (basic)
docker run -d \
  --name minio \
  -p 9000:9000 \
  -p 9001:9001 \
  -e MINIO_ROOT_USER=minioadmin \
  -e MINIO_ROOT_PASSWORD=minioadmin \
  minio/minio server /data --console-address ":9001"

# Run พร้อม persistent storage
docker run -d \
  --name minio \
  -p 9000:9000 \
  -p 9001:9001 \
  -e MINIO_ROOT_USER=your_access_key \
  -e MINIO_ROOT_PASSWORD=your_secret_key_min_8_chars \
  -v /path/to/minio/data:/data \
  minio/minio server /data --console-address ":9001"

# ทดสอบ
curl http://localhost:9000/minio/health/live
# HTTP 200 = healthy

# Web Console
# เปิด browser: http://localhost:9001
# Login: your_access_key / your_secret_key_min_8_chars
```

### 4.2 Docker Compose (Recommended)

```yaml
# docker-compose.yml
version: '3.8'

services:
  minio:
    image: minio/minio:latest
    container_name: minio
    restart: unless-stopped
    ports:
      - "9000:9000"   # S3 API
      - "9001:9001"   # Console (Web UI)
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER:-minioadmin}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD:-minioadmin123}
      MINIO_BROWSER_REDIRECT_URL: http://localhost:9001
      # MINIO_DOMAIN: minio.example.com  # สำหรับ virtual-host style
      # MINIO_SITE_NAME: my-minio-cluster
      # MINIO_COMPRESSION_ENABLE: "on"  # compress objects
    volumes:
      - minio_data:/data
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    networks:
      - app-network

  # MinIO Client สำหรับ initial setup
  minio-setup:
    image: minio/mc:latest
    container_name: minio-setup
    depends_on:
      minio:
        condition: service_healthy
    entrypoint: >
      /bin/sh -c "
      mc alias set local http://minio:9000 minioadmin minioadmin123;
      mc mb local/uploads --ignore-existing;
      mc mb local/avatars --ignore-existing;
      mc mb local/documents --ignore-existing;
      mc policy set download local/avatars;
      echo 'MinIO setup complete!';
      exit 0;
      "
    networks:
      - app-network

volumes:
  minio_data:
    driver: local

networks:
  app-network:
    driver: bridge
```

### 4.3 Multi-node Setup (Production)

```yaml
# docker-compose-distributed.yml
version: '3.8'

x-minio-common: &minio-common
  image: minio/minio:latest
  environment:
    MINIO_ROOT_USER: minioadmin
    MINIO_ROOT_PASSWORD: minioadmin123secret
  command: server --console-address ":9001" http://minio{1...4}/data{1...2}
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
    interval: 30s
    timeout: 10s
    retries: 3

services:
  minio1:
    <<: *minio-common
    hostname: minio1
    volumes:
      - data1-1:/data1
      - data1-2:/data2

  minio2:
    <<: *minio-common
    hostname: minio2
    volumes:
      - data2-1:/data1
      - data2-2:/data2

  minio3:
    <<: *minio-common
    hostname: minio3
    volumes:
      - data3-1:/data1
      - data3-2:/data2

  minio4:
    <<: *minio-common
    hostname: minio4
    volumes:
      - data4-1:/data1
      - data4-2:/data2

  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "9000:9000"
      - "9001:9001"
    depends_on:
      - minio1
      - minio2
      - minio3
      - minio4

volumes:
  data1-1:
  data1-2:
  data2-1:
  data2-2:
  data3-1:
  data3-2:
  data4-1:
  data4-2:
```

```nginx
# nginx.conf สำหรับ MinIO cluster
upstream minio {
    least_conn;
    server minio1:9000;
    server minio2:9000;
    server minio3:9000;
    server minio4:9000;
}

upstream console {
    least_conn;
    server minio1:9001;
    server minio2:9001;
    server minio3:9001;
    server minio4:9001;
}

server {
    listen 9000;
    
    ignore_invalid_headers off;
    client_max_body_size 0;
    proxy_buffering off;
    proxy_request_buffering off;
    
    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        proxy_connect_timeout 300;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        chunked_transfer_encoding off;
        
        proxy_pass http://minio;
    }
}

server {
    listen 9001;
    
    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        proxy_pass http://console;
    }
}
```

---

## 5. MinIO Console (Web UI)

### 5.1 เข้า Console

```
URL: http://localhost:9001
Username: minioadmin (หรือ MINIO_ROOT_USER ที่ตั้งไว้)
Password: minioadmin123 (หรือ MINIO_ROOT_PASSWORD ที่ตั้งไว้)
```

### 5.2 สิ่งที่ทำได้ใน Console

```
1. Buckets:
   - สร้าง/ลบ bucket
   - เปิด/ปิด versioning
   - ตั้ง lifecycle rules
   - ดู bucket usage

2. Objects:
   - Browse objects (file manager style)
   - Upload files (drag & drop)
   - Download files
   - Delete files
   - สร้าง folder (prefix)
   - View/Edit metadata

3. Access Management:
   - สร้าง Users
   - สร้าง Groups
   - กำหนด Policies (JSON)
   - สร้าง Service Accounts (Access Key)

4. Monitoring:
   - Storage usage graph
   - Request metrics
   - Bandwidth usage
   - Server status

5. Settings:
   - Notification targets (webhook, kafka, etc.)
   - Tiering (ILM)
   - Site Replication
```

---

## 6. mc (MinIO Client) CLI

### 6.1 ติดตั้ง mc

```bash
# Linux (AMD64)
curl https://dl.min.io/client/mc/release/linux-amd64/mc \
  --create-dirs -o $HOME/minio-binaries/mc
chmod +x $HOME/minio-binaries/mc
export PATH=$PATH:$HOME/minio-binaries/

# macOS
brew install minio/stable/mc

# ด้วย Docker (ไม่ต้องติดตั้ง)
docker run --rm -it \
  --entrypoint bash \
  minio/mc
```

### 6.2 ตั้งค่า Alias

```bash
# เพิ่ม alias สำหรับ server
mc alias set <alias> <endpoint> <access-key> <secret-key>

# ตัวอย่าง
mc alias set local http://localhost:9000 minioadmin minioadmin123
mc alias set myminio https://minio.example.com AKIAIOSFODNN7EXAMPLE wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# สำหรับ AWS S3
mc alias set aws https://s3.amazonaws.com AKIAIOSFODNN7EXAMPLE wJalrXUtnFEMI

# ดู aliases ทั้งหมด
mc alias list

# ลบ alias
mc alias rm local
```

### 6.3 Bucket Commands

```bash
# ========================================
# Bucket Management
# ========================================

# mb (make bucket)
mc mb local/my-bucket
mc mb local/uploads --with-versioning    # สร้างพร้อม versioning
mc mb local/public-images               

# ls (list)
mc ls local                             # list all buckets
mc ls local/my-bucket                  # list objects in bucket
mc ls local/my-bucket/prefix/          # list with prefix
mc ls --recursive local/my-bucket      # recursive list
mc ls --versions local/my-bucket       # list with versions

# rb (remove bucket)
mc rb local/my-bucket                  # ลบ bucket (ต้องว่าง)
mc rb --force local/my-bucket          # บังคับลบพร้อม objects
```

### 6.4 Object Commands

```bash
# ========================================
# Copy/Move/Remove Objects
# ========================================

# cp (copy)
mc cp myfile.txt local/my-bucket/              # อัพโหลดไฟล์
mc cp myfile.txt local/my-bucket/folder/       # อัพโหลดใน folder
mc cp local/my-bucket/file.txt ./              # ดาวน์โหลด
mc cp local/my-bucket/file.txt ./newname.txt   # ดาวน์โหลดพร้อมเปลี่ยนชื่อ

# Recursive copy
mc cp --recursive ./local-folder local/my-bucket/prefix/    # อัพโหลด folder
mc cp --recursive local/my-bucket/prefix/ ./local-folder/   # ดาวน์โหลด folder

# Copy ระหว่าง buckets/servers
mc cp local/bucket1/file.txt local/bucket2/
mc cp aws/prod-bucket/backup.tar local/backup-bucket/

# mv (move)
mc mv file.txt local/my-bucket/               # move (upload + delete local)
mc mv local/my-bucket/old.txt local/my-bucket/new.txt  # rename

# rm (remove)
mc rm local/my-bucket/file.txt
mc rm --recursive local/my-bucket/folder/     # ลบ folder
mc rm --recursive --force local/my-bucket/    # ลบทุกอย่างใน bucket
mc rm --versions local/my-bucket/file.txt     # ลบทุก versions

# cat (แสดงเนื้อหาไฟล์)
mc cat local/my-bucket/config.json

# head (แสดง N บรรทัดแรก)
mc head -n 5 local/my-bucket/large.log

# ========================================
# Find Objects
# ========================================

# find (ค้นหา objects)
mc find local/my-bucket --name "*.jpg"           # ค้นหาตาม name pattern
mc find local/my-bucket --larger 10MiB           # ไฟล์ใหญ่กว่า 10 MB
mc find local/my-bucket --smaller 1KiB           # ไฟล์เล็กกว่า 1 KB
mc find local/my-bucket --older 7d               # ไฟล์เก่ากว่า 7 วัน
mc find local/my-bucket --newer 1h               # ไฟล์ใหม่กว่า 1 ชั่วโมง

# ลบไฟล์เก่า
mc find local/my-bucket/logs --older 30d --exec "mc rm {}"

# ========================================
# Sync and Mirror
# ========================================

# mirror (sync local ↔ remote)
mc mirror ./local-folder local/my-bucket/prefix/   # sync ขึ้น
mc mirror local/my-bucket/prefix/ ./local-folder/  # sync ลง

# mirror options
mc mirror --watch ./uploads local/my-bucket/       # watch for changes
mc mirror --remove local/my-bucket/ ./backup/      # ลบไฟล์ที่ไม่มีในต้นทาง
mc mirror --preserve local/bucket1/ local/bucket2/ # preserve metadata

# diff (เปรียบเทียบ)
mc diff local/my-bucket/folder1/ local/my-bucket/folder2/
```

### 6.5 Policy, Tag, Metadata Commands

```bash
# ========================================
# Bucket Policies
# ========================================

# ดู policy
mc anonymous get local/my-bucket
mc anonymous get-json local/my-bucket  # ดูแบบ JSON

# ตั้ง built-in policies
mc anonymous set none local/my-bucket          # private (default)
mc anonymous set download local/my-bucket      # public read
mc anonymous set upload local/my-bucket        # public upload
mc anonymous set public local/my-bucket        # public read + write

# ตั้ง custom policy จากไฟล์
mc anonymous set-json policy.json local/my-bucket

# ลบ policy
mc anonymous remove local/my-bucket

# ========================================
# Tags
# ========================================

# ตั้ง tags ให้ bucket
mc tag set local/my-bucket "Environment=production" "Project=myapp"

# ดู tags
mc tag list local/my-bucket

# ลบ tags
mc tag remove local/my-bucket

# Tags สำหรับ object
mc tag set local/my-bucket/file.txt "author=alice" "version=1.0"
mc tag list local/my-bucket/file.txt

# ========================================
# Object Stats and Info
# ========================================

# stat (ดู metadata)
mc stat local/my-bucket/file.txt

# ผลลัพธ์:
# Name      : file.txt
# Date      : 2024-01-15 10:30:00 UTC
# Size      : 1.2 KiB
# ETag      : d41d8cd98f00b204e9800998ecf8427e
# Type      : file
# Metadata  :
#   Content-Type: text/plain
#   X-Amz-Meta-Author: Alice

# ========================================
# Encryption
# ========================================

# เข้ารหัสด้วย server-side key
mc encrypt set sse-s3 local/my-bucket      # SSE-S3 (server managed)
mc encrypt set sse-kms key-id local/my-bucket  # SSE-KMS

mc encrypt info local/my-bucket
mc encrypt clear local/my-bucket
```

---

## 7. Bucket Policies

### 7.1 Policy Types

```bash
# none (private): เฉพาะ authenticated users
# download: ทุกคน read ได้, เฉพาะ auth write
# upload: เฉพาะ auth read, ทุกคน upload ได้
# public: ทุกคน read และ write ได้

mc anonymous set download local/avatars      # profile pictures - public read
mc anonymous set none local/documents        # sensitive files - private
```

### 7.2 Custom Policy JSON

```json
// policy.json - public read สำหรับ jpg files เท่านั้น
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": ["*"]},
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::my-bucket/public/*.jpg"]
    },
    {
      "Effect": "Allow",
      "Principal": {"AWS": ["*"]},
      "Action": ["s3:ListBucket"],
      "Resource": ["arn:aws:s3:::my-bucket"],
      "Condition": {
        "StringLike": {
          "s3:prefix": ["public/*"]
        }
      }
    }
  ]
}
```

```json
// policy-upload.json - allow upload to specific prefix
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": ["*"]},
      "Action": ["s3:PutObject"],
      "Resource": ["arn:aws:s3:::uploads/temp/*"]
    }
  ]
}
```

```bash
# Apply policy
mc anonymous set-json policy.json local/my-bucket
```

### 7.3 IAM User Policies

```json
// user-policy.json - read-only access
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:ListBucket"
      ],
      "Resource": ["arn:aws:s3:::my-bucket"]
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::my-bucket/*"]
    }
  ]
}
```

```bash
# สร้าง user และ policy
mc admin user add local alice alice_password123
mc admin policy create local readonly-policy readonly-policy.json
mc admin policy attach local readonly-policy --user alice
```

---

## 8. Versioning

Versioning เก็บหลาย versions ของ object เดียวกัน ป้องกันการลบโดยไม่ตั้งใจ

```bash
# เปิด versioning
mc version enable local/my-bucket
mc version info local/my-bucket
mc version suspend local/my-bucket    # suspend (ไม่สร้าง version ใหม่)
mc version disable local/my-bucket    # ปิด (ไม่ได้ลบ versions เดิม)

# อัพโหลดไฟล์หลายครั้ง
mc cp v1.txt local/versioned-bucket/file.txt
mc cp v2.txt local/versioned-bucket/file.txt
mc cp v3.txt local/versioned-bucket/file.txt

# list versions
mc ls --versions local/versioned-bucket/file.txt
# [2024-01-15 10:00:00 UTC] 100B v1  file.txt  vid:abc123
# [2024-01-15 10:01:00 UTC] 200B v1  file.txt  vid:def456
# [2024-01-15 10:02:00 UTC] 150B v1  file.txt  vid:ghi789

# download specific version
mc cp --vid abc123 local/versioned-bucket/file.txt ./restored_v1.txt

# delete specific version (permanent)
mc rm --vid abc123 local/versioned-bucket/file.txt

# delete current version (creates delete marker)
mc rm local/versioned-bucket/file.txt

# list all versions including delete markers
mc ls --versions local/versioned-bucket/
```

---

## 9. Lifecycle Policies

Lifecycle policies จัดการ objects อัตโนมัติ เช่น ลบเมื่อครบ N วัน หรือ transition ไป cold storage

### 9.1 ด้วย mc CLI

```bash
# Lifecycle policy: ลบ objects เก่ากว่า 30 วัน
cat > lifecycle.json << 'EOF'
{
  "Rules": [
    {
      "ID": "delete-old-logs",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "logs/"
      },
      "Expiration": {
        "Days": 30
      }
    }
  ]
}
EOF

mc ilm import local/my-bucket < lifecycle.json

# ดู lifecycle rules
mc ilm ls local/my-bucket
mc ilm export local/my-bucket

# ลบ lifecycle rules
mc ilm rm local/my-bucket --id delete-old-logs
mc ilm rm local/my-bucket --all  # ลบทั้งหมด
```

### 9.2 Advanced Lifecycle Rules

```json
{
  "Rules": [
    {
      "ID": "expire-temp-files",
      "Status": "Enabled",
      "Filter": { "Prefix": "temp/" },
      "Expiration": { "Days": 1 }
    },
    {
      "ID": "expire-old-versions",
      "Status": "Enabled",
      "Filter": { "Prefix": "" },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90
      }
    },
    {
      "ID": "cleanup-incomplete-multipart",
      "Status": "Enabled",
      "Filter": { "Prefix": "" },
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    },
    {
      "ID": "archive-after-year",
      "Status": "Enabled",
      "Filter": {
        "And": {
          "Prefix": "archive/",
          "Tags": [
            { "Key": "keep", "Value": "true" }
          ]
        }
      },
      "Expiration": { "Days": 365 }
    }
  ]
}
```

---

## 10. Pre-signed URLs

Pre-signed URLs ช่วยให้ share files ชั่วคราวได้โดยไม่ต้องเปิด bucket เป็น public

### 10.1 สร้าง Pre-signed URL ด้วย mc

```bash
# Share link สำหรับ download (default 7 วัน)
mc share download local/my-bucket/private/document.pdf

# กำหนด expiry
mc share download local/my-bucket/file.txt --expire 24h     # 24 ชั่วโมง
mc share download local/my-bucket/file.txt --expire 1h30m   # 1.5 ชั่วโมง
mc share download local/my-bucket/file.txt --expire 604800s  # 7 วัน

# Share link สำหรับ upload (pre-signed PUT)
mc share upload local/my-bucket/uploads/

# ผลลัพธ์:
# URL: https://minio.example.com/my-bucket/file.txt?...
# Expire: 1 days 0 hours 0 minutes 0 seconds
```

### 10.2 สร้าง Pre-signed URL ด้วย SDK

```javascript
// Node.js: MinIO SDK
const Minio = require('minio');

const minioClient = new Minio.Client({
  endPoint: 'localhost',
  port: 9000,
  useSSL: false,
  accessKey: 'minioadmin',
  secretKey: 'minioadmin123'
});

// Pre-signed URL สำหรับ GET (download)
async function getPresignedDownloadUrl(bucketName, objectName, expirySeconds = 3600) {
  const url = await minioClient.presignedGetObject(bucketName, objectName, expirySeconds);
  return url;
}

// Pre-signed URL สำหรับ PUT (upload)
async function getPresignedUploadUrl(bucketName, objectName, expirySeconds = 3600) {
  const url = await minioClient.presignedPutObject(bucketName, objectName, expirySeconds);
  return url;
}

// Pre-signed URL พร้อม custom headers
async function getPresignedWithHeaders(bucketName, objectName) {
  const reqParams = {
    'response-content-disposition': 'attachment; filename="download.pdf"',
    'response-content-type': 'application/pdf'
  };
  
  const url = await minioClient.presignedGetObject(
    bucketName, objectName, 3600, reqParams
  );
  return url;
}

// POST policy สำหรับ browser upload
async function getPresignedPost(bucketName, objectName) {
  const policy = minioClient.newPostPolicy();
  policy.setBucket(bucketName);
  policy.setKey(objectName);
  policy.setExpires(new Date(Date.now() + 60 * 60 * 1000));  // 1 hour
  policy.setContentType('image/jpeg');
  policy.setContentLengthRange(1024, 10 * 1024 * 1024);  // 1KB - 10MB
  
  return await minioClient.presignedPostPolicy(policy);
}

// ตัวอย่าง Express endpoint
app.get('/api/upload-url', async (req, res) => {
  const { filename, contentType } = req.query;
  const userId = req.user.id;
  
  const objectName = `users/${userId}/${Date.now()}-${filename}`;
  const uploadUrl = await getPresignedUploadUrl('uploads', objectName, 300);
  
  res.json({
    uploadUrl,
    objectName,
    expiresIn: 300
  });
});

app.get('/api/download-url/:objectName', async (req, res) => {
  const downloadUrl = await getPresignedDownloadUrl(
    'documents', 
    req.params.objectName, 
    3600
  );
  
  res.json({ downloadUrl, expiresIn: 3600 });
});
```

```python
# Python: boto3 (S3-compatible)
import boto3
from botocore.config import Config
from datetime import datetime

s3_client = boto3.client(
    's3',
    endpoint_url='http://localhost:9000',
    aws_access_key_id='minioadmin',
    aws_secret_access_key='minioadmin123',
    config=Config(signature_version='s3v4'),
    region_name='us-east-1'
)

def get_presigned_download_url(bucket, key, expiry=3600):
    """สร้าง pre-signed URL สำหรับ download"""
    url = s3_client.generate_presigned_url(
        'get_object',
        Params={'Bucket': bucket, 'Key': key},
        ExpiresIn=expiry
    )
    return url

def get_presigned_upload_url(bucket, key, expiry=3600, content_type=None):
    """สร้าง pre-signed URL สำหรับ upload"""
    params = {'Bucket': bucket, 'Key': key}
    if content_type:
        params['ContentType'] = content_type
    
    url = s3_client.generate_presigned_url(
        'put_object',
        Params=params,
        ExpiresIn=expiry
    )
    return url

def get_presigned_post(bucket, key, max_size_bytes=10*1024*1024):
    """สร้าง pre-signed POST สำหรับ browser upload"""
    response = s3_client.generate_presigned_post(
        bucket,
        key,
        Fields={'Content-Type': 'image/jpeg'},
        Conditions=[
            {'Content-Type': 'image/jpeg'},
            ['content-length-range', 0, max_size_bytes]
        ],
        ExpiresIn=3600
    )
    return response  # dict ที่มี 'url' และ 'fields'
```

---

## 11. Object Metadata และ Tags

### 11.1 System Metadata

```bash
# System metadata (ถูกตั้งโดย S3/MinIO)
# Content-Type: text/plain
# Content-Length: 1234
# ETag: md5-hash ของ content
# Last-Modified: timestamp
# x-amz-version-id: version ID (ถ้าเปิด versioning)
```

### 11.2 User-defined Metadata

```bash
# อัพโหลดพร้อม metadata
mc cp file.txt local/my-bucket/ \
  --attr "Content-Type=text/plain;X-Author=Alice;X-Project=demo"

# ดู metadata
mc stat local/my-bucket/file.txt

# คำสั่ง cp พร้อม metadata ด้วย S3 API
aws s3 cp file.txt s3://my-bucket/ \
  --metadata '{"Author":"Alice","Version":"1.0"}' \
  --endpoint-url http://localhost:9000
```

```javascript
// Node.js: อัพโหลดพร้อม metadata
await minioClient.putObject(
  'my-bucket',
  'documents/report.pdf',
  fileBuffer,
  fileBuffer.length,
  {
    'Content-Type': 'application/pdf',
    'x-amz-meta-author': 'Alice Johnson',
    'x-amz-meta-department': 'Finance',
    'x-amz-meta-version': '2.1',
    'x-amz-meta-created-date': new Date().toISOString()
  }
);

// ดู metadata
const stat = await minioClient.statObject('my-bucket', 'documents/report.pdf');
console.log('Metadata:', stat.metaData);
// { 'x-amz-meta-author': 'Alice Johnson', ... }
```

### 11.3 Object Tags

```bash
# ตั้ง tags ให้ object
mc tag set local/my-bucket/file.txt "env=prod" "category=report" "retention=90days"

# ดู tags
mc tag list local/my-bucket/file.txt

# ลบ tags
mc tag remove local/my-bucket/file.txt
```

```javascript
// Node.js: Object Tags
const { S3Client, PutObjectTaggingCommand, GetObjectTaggingCommand } = require('@aws-sdk/client-s3');

const s3 = new S3Client({
  endpoint: 'http://localhost:9000',
  region: 'us-east-1',
  credentials: { accessKeyId: 'minioadmin', secretAccessKey: 'minioadmin123' },
  forcePathStyle: true  // สำคัญสำหรับ MinIO
});

// ตั้ง tags
await s3.send(new PutObjectTaggingCommand({
  Bucket: 'my-bucket',
  Key: 'file.txt',
  Tagging: {
    TagSet: [
      { Key: 'Environment', Value: 'production' },
      { Key: 'Project', Value: 'myapp' },
      { Key: 'Owner', Value: 'team-a' }
    ]
  }
}));

// ดู tags
const tagging = await s3.send(new GetObjectTaggingCommand({
  Bucket: 'my-bucket',
  Key: 'file.txt'
}));
console.log('Tags:', tagging.TagSet);
```

---

## 12. Multipart Upload

Multipart Upload ใช้สำหรับไฟล์ขนาดใหญ่ (> 100 MB แนะนำ) โดย upload เป็น parts แล้ว combine

### 12.1 กระบวนการ Multipart Upload

```
1. Initiate Upload → ได้ UploadId
2. Upload Parts (แต่ละ part ≥ 5 MB, ยกเว้น part สุดท้าย)
3. Complete Upload (รวม parts ทั้งหมด)
หรือ
4. Abort Upload (ยกเลิก, ลบ parts ที่อัพโหลดไปแล้ว)
```

### 12.2 ด้วย mc CLI

```bash
# mc จัดการ multipart upload อัตโนมัติสำหรับไฟล์ใหญ่
mc cp largefile.zip local/my-bucket/

# ตรวจสอบ incomplete multipart uploads
mc ls --incomplete local/my-bucket/

# ลบ incomplete uploads
mc rm --incomplete local/my-bucket/largefile.zip
```

### 12.3 ด้วย SDK (Python)

```python
# Python: boto3 multipart upload
import boto3
import os
from botocore.config import Config

s3 = boto3.client(
    's3',
    endpoint_url='http://localhost:9000',
    aws_access_key_id='minioadmin',
    aws_secret_access_key='minioadmin123',
    config=Config(signature_version='s3v4'),
    region_name='us-east-1'
)

def upload_large_file(file_path, bucket_name, object_key, part_size_mb=10):
    """อัพโหลดไฟล์ขนาดใหญ่ด้วย multipart"""
    part_size = part_size_mb * 1024 * 1024
    file_size = os.path.getsize(file_path)
    
    print(f"Uploading {file_path} ({file_size / 1024 / 1024:.1f} MB)")
    
    # Step 1: Initiate upload
    response = s3.create_multipart_upload(
        Bucket=bucket_name,
        Key=object_key,
        ContentType='application/octet-stream',
        Metadata={'original-filename': os.path.basename(file_path)}
    )
    upload_id = response['UploadId']
    print(f"Upload ID: {upload_id}")
    
    try:
        parts = []
        part_number = 1
        
        with open(file_path, 'rb') as f:
            while True:
                data = f.read(part_size)
                if not data:
                    break
                
                # Step 2: Upload each part
                response = s3.upload_part(
                    Bucket=bucket_name,
                    Key=object_key,
                    PartNumber=part_number,
                    UploadId=upload_id,
                    Body=data
                )
                
                etag = response['ETag']
                parts.append({'PartNumber': part_number, 'ETag': etag})
                print(f"Uploaded part {part_number} (ETag: {etag[:20]}...)")
                part_number += 1
        
        # Step 3: Complete upload
        response = s3.complete_multipart_upload(
            Bucket=bucket_name,
            Key=object_key,
            UploadId=upload_id,
            MultipartUpload={'Parts': parts}
        )
        
        print(f"Upload complete! ETag: {response['ETag']}")
        return response
        
    except Exception as e:
        # Abort ถ้าเกิด error
        s3.abort_multipart_upload(
            Bucket=bucket_name,
            Key=object_key,
            UploadId=upload_id
        )
        print(f"Upload aborted due to error: {e}")
        raise

# ใช้ boto3 transfer manager (แนะนำสำหรับ production)
from boto3.s3.transfer import TransferConfig, S3Transfer

def upload_with_transfer_manager(file_path, bucket_name, object_key):
    config = TransferConfig(
        multipart_threshold=1024 * 25,    # 25 MB
        max_concurrency=10,               # parallel parts
        multipart_chunksize=1024 * 25,    # 25 MB per part
        use_threads=True
    )
    
    transfer = S3Transfer(s3, config)
    transfer.upload_file(
        file_path, bucket_name, object_key,
        callback=ProgressPercentage(file_path)
    )

class ProgressPercentage:
    def __init__(self, filename):
        self._filename = filename
        self._size = float(os.path.getsize(filename))
        self._seen_so_far = 0
    
    def __call__(self, bytes_amount):
        self._seen_so_far += bytes_amount
        percentage = (self._seen_so_far / self._size) * 100
        print(f"\r{self._filename}: {self._seen_so_far}/{self._size:.0f} bytes ({percentage:.2f}%)", end='')
```

### 12.4 ด้วย Node.js SDK

```javascript
// Node.js: Multipart upload
const fs = require('fs');
const Minio = require('minio');

const minio = new Minio.Client({
  endPoint: 'localhost',
  port: 9000,
  useSSL: false,
  accessKey: 'minioadmin',
  secretKey: 'minioadmin123'
});

// Simple upload (MinIO SDK จัดการ multipart อัตโนมัติ)
async function uploadFile(bucketName, objectName, filePath) {
  const fileStream = fs.createReadStream(filePath);
  const fileStat = fs.statSync(filePath);
  
  await minio.putObject(
    bucketName,
    objectName,
    fileStream,
    fileStat.size,
    { 'Content-Type': getMimeType(filePath) }
  );
  
  console.log(`Uploaded ${filePath} as ${objectName}`);
}

// ติดตาม progress
async function uploadWithProgress(bucketName, objectName, filePath) {
  return new Promise((resolve, reject) => {
    const fileStream = fs.createReadStream(filePath);
    const fileSize = fs.statSync(filePath).size;
    let uploaded = 0;
    
    // Track progress ผ่าน stream
    fileStream.on('data', (chunk) => {
      uploaded += chunk.length;
      const progress = (uploaded / fileSize * 100).toFixed(1);
      process.stdout.write(`\rProgress: ${progress}%`);
    });
    
    minio.putObject(bucketName, objectName, fileStream, fileSize, 
      {}, (err, etag) => {
        if (err) {
          console.log('\nUpload failed:', err);
          reject(err);
        } else {
          console.log('\nUpload complete! ETag:', etag);
          resolve(etag);
        }
      }
    );
  });
}
```

---

## 13. Server-side Encryption

### 13.1 Encryption Types

```
SSE-S3 (Server-Side Encryption with S3-Managed Keys):
- MinIO/S3 จัดการ keys
- Transparent to user
- เปิดใช้ใน bucket level

SSE-KMS (Server-Side Encryption with KMS Keys):
- ใช้ Key Management Service
- สำหรับ MinIO ใช้ KES (Key Encryption Service)

SSE-C (Server-Side Encryption with Customer Keys):
- User ส่ง encryption key ใน request header
- S3 ไม่เก็บ key
- ปลอดภัยที่สุด แต่ต้องจัดการ key เอง
```

### 13.2 เปิด SSE-S3

```bash
# เปิด auto-encryption สำหรับ bucket
mc encrypt set sse-s3 local/sensitive-data
mc encrypt info local/sensitive-data

# อัพโหลดไฟล์ (จะถูกเข้ารหัสอัตโนมัติ)
mc cp secret-document.pdf local/sensitive-data/
```

### 13.3 SSE-C ด้วย SDK

```python
# Python: SSE-C
import base64
import hashlib
import boto3

# สร้าง encryption key (32 bytes for AES-256)
encryption_key = os.urandom(32)
encryption_key_b64 = base64.b64encode(encryption_key).decode()
encryption_key_md5 = base64.b64encode(hashlib.md5(encryption_key).digest()).decode()

# Upload พร้อม SSE-C
s3.put_object(
    Bucket='my-bucket',
    Key='encrypted-file.pdf',
    Body=open('file.pdf', 'rb'),
    SSECustomerAlgorithm='AES256',
    SSECustomerKey=encryption_key_b64,
    SSECustomerKeyMD5=encryption_key_md5
)

# Download พร้อม SSE-C (ต้องใส่ key เดิม)
s3.get_object(
    Bucket='my-bucket',
    Key='encrypted-file.pdf',
    SSECustomerAlgorithm='AES256',
    SSECustomerKey=encryption_key_b64,
    SSECustomerKeyMD5=encryption_key_md5
)

# ⚠️ ต้องเก็บ encryption_key อย่างปลอดภัย!
# ถ้าหายจะเข้าถึงไฟล์ไม่ได้
```

---

## 14. MinIO vs AWS S3

### เมื่อไหร่ใช้อะไร

```
ใช้ AWS S3 เมื่อ:
✅ ต้องการ managed service ไม่อยากดูแล infrastructure
✅ ต้องการ global CDN (CloudFront)
✅ ต้องการ integration กับ AWS services อื่นๆ
✅ ต้องการ SLA 99.99% availability
✅ ข้อมูลใหญ่มากและต้องการ auto-scale
✅ มีงบประมาณเพียงพอ (ค่าใช้จ่ายตาม usage)
✅ ต้องการ compliance certifications (HIPAA, PCI DSS)

ใช้ MinIO เมื่อ:
✅ ต้องการ on-premise storage (data sovereignty)
✅ ต้องการ S3-compatible API แต่ไม่อยากผูกกับ AWS
✅ ต้องการ dev/staging environment ที่ไม่เสียค่า
✅ ต้องการ control เต็มที่เรื่อง data
✅ งบจำกัด (hardware cost เท่านั้น)
✅ Private cloud หรือ air-gapped environment
✅ ต้องการ migrate ออกจาก AWS ได้ง่าย

ใช้ MinIO สำหรับ Dev แล้วต่อยอดไป AWS S3:
✅ ใช้ S3 API เหมือนกัน เปลี่ยนแค่ endpoint
✅ Test locally ก่อน deploy ขึ้น production
```

### Feature Comparison

| Feature | MinIO | AWS S3 |
|---------|-------|--------|
| S3 API Compatibility | 100% | Native |
| Versioning | มี | มี |
| Lifecycle Policies | มี | มี |
| Cross-region Replication | มี | มี |
| Encryption (SSE-S3, SSE-C) | มี | มี |
| Pre-signed URLs | มี | มี |
| Multipart Upload | มี | มี |
| Object Lock (WORM) | มี | มี |
| Event Notifications | มี (Kafka, AMQP, Redis) | มี (SNS, SQS) |
| Pricing | Hardware เท่านั้น | Pay-per-use |
| Max object size | 5 TB | 5 TB |
| Durability | ขึ้นอยู่กับ setup | 99.999999999% |
| Global CDN | ไม่มี built-in | CloudFront |

---

## 15. Workshop: Upload/Download ด้วย CLI และ SDK

### 15.1 เตรียม Environment

```bash
# Start MinIO
docker-compose up -d

# ติดตั้ง mc
brew install minio/stable/mc  # macOS
# หรือ
curl https://dl.min.io/client/mc/release/linux-amd64/mc -o mc
chmod +x mc

# ตั้งค่า
mc alias set local http://localhost:9000 minioadmin minioadmin123

# ทดสอบ
mc admin info local
```

### 15.2 Workshop Script

```bash
#!/bin/bash
# workshop-minio.sh

echo "=== MinIO Workshop ==="
ALIAS="local"

# ========================================
# Step 1: สร้าง Buckets
# ========================================
echo ""
echo "--- Step 1: Creating Buckets ---"

mc mb $ALIAS/uploads --ignore-existing
mc mb $ALIAS/avatars --ignore-existing
mc mb $ALIAS/documents --ignore-existing
mc mb $ALIAS/backups --ignore-existing

echo "Buckets created:"
mc ls $ALIAS

# ========================================
# Step 2: ตั้ง Policies
# ========================================
echo ""
echo "--- Step 2: Setting Policies ---"

# avatars: public read
mc anonymous set download $ALIAS/avatars
echo "avatars: public read"

# uploads: private
mc anonymous set none $ALIAS/uploads
echo "uploads: private"

# ========================================
# Step 3: Upload Files
# ========================================
echo ""
echo "--- Step 3: Uploading Files ---"

# สร้างไฟล์ทดสอบ
echo "Hello, MinIO!" > /tmp/test.txt
echo '{"name":"test","value":42}' > /tmp/data.json
dd if=/dev/zero of=/tmp/large-file.bin bs=1M count=5 2>/dev/null

# Upload ไฟล์เดี่ยว
mc cp /tmp/test.txt $ALIAS/uploads/test.txt
mc cp /tmp/data.json $ALIAS/uploads/data.json
mc cp /tmp/large-file.bin $ALIAS/uploads/large-file.bin

echo "Files uploaded:"
mc ls $ALIAS/uploads

# ========================================
# Step 4: Upload พร้อม Metadata
# ========================================
echo ""
echo "--- Step 4: Upload with Metadata ---"

mc cp /tmp/data.json $ALIAS/documents/report.json \
  --attr "Content-Type=application/json;X-Author=Alice;X-Department=Engineering"

mc stat $ALIAS/documents/report.json

# ========================================
# Step 5: Download Files
# ========================================
echo ""
echo "--- Step 5: Downloading Files ---"

mc cp $ALIAS/uploads/test.txt /tmp/downloaded-test.txt
echo "Downloaded content:"
cat /tmp/downloaded-test.txt

# ========================================
# Step 6: List และ Find
# ========================================
echo ""
echo "--- Step 6: List and Find ---"

mc ls --recursive $ALIAS/uploads
echo ""
mc find $ALIAS/uploads --name "*.txt"

# ========================================
# Step 7: Versioning
# ========================================
echo ""
echo "--- Step 7: Versioning ---"

mc version enable $ALIAS/documents
echo "v1" > /tmp/versioned.txt
mc cp /tmp/versioned.txt $ALIAS/documents/versioned.txt
echo "v2" > /tmp/versioned.txt
mc cp /tmp/versioned.txt $ALIAS/documents/versioned.txt
echo "v3" > /tmp/versioned.txt
mc cp /tmp/versioned.txt $ALIAS/documents/versioned.txt

echo "Versions:"
mc ls --versions $ALIAS/documents/versioned.txt

# ========================================
# Step 8: Lifecycle
# ========================================
echo ""
echo "--- Step 8: Lifecycle Policy ---"

cat > /tmp/lifecycle.json << 'LIFECYCLE'
{
  "Rules": [
    {
      "ID": "delete-old-uploads",
      "Status": "Enabled",
      "Filter": { "Prefix": "" },
      "Expiration": { "Days": 7 }
    }
  ]
}
LIFECYCLE

mc ilm import $ALIAS/uploads < /tmp/lifecycle.json
mc ilm ls $ALIAS/uploads

# ========================================
# Step 9: Mirror (Sync)
# ========================================
echo ""
echo "--- Step 9: Mirror ---"

mkdir -p /tmp/local-backup
mc mirror $ALIAS/uploads /tmp/local-backup/

echo "Local backup contents:"
ls -la /tmp/local-backup/

# ========================================
# Step 10: Stats
# ========================================
echo ""
echo "--- Step 10: Statistics ---"

mc admin info $ALIAS
echo ""
mc du $ALIAS/uploads     # disk usage

echo ""
echo "=== Workshop Complete ==="
```

### 15.3 Node.js Full Application

```javascript
// minio-file-service.js
const Minio = require('minio');
const fs = require('fs');
const path = require('path');
const crypto = require('crypto');

const minio = new Minio.Client({
  endPoint: process.env.MINIO_ENDPOINT || 'localhost',
  port: parseInt(process.env.MINIO_PORT) || 9000,
  useSSL: process.env.MINIO_USE_SSL === 'true',
  accessKey: process.env.MINIO_ACCESS_KEY || 'minioadmin',
  secretKey: process.env.MINIO_SECRET_KEY || 'minioadmin123'
});

const BUCKET = process.env.MINIO_BUCKET || 'uploads';

// สร้าง bucket ถ้าไม่มี
async function ensureBucket(bucketName = BUCKET) {
  const exists = await minio.bucketExists(bucketName);
  if (!exists) {
    await minio.makeBucket(bucketName, 'us-east-1');
    console.log(`Created bucket: ${bucketName}`);
  }
}

// Upload file
async function uploadFile(localPath, objectName = null, options = {}) {
  await ensureBucket();
  
  if (!objectName) {
    const ext = path.extname(localPath);
    const hash = crypto.randomBytes(8).toString('hex');
    objectName = `${Date.now()}-${hash}${ext}`;
  }
  
  const fileBuffer = fs.readFileSync(localPath);
  const metadata = {
    'Content-Type': options.contentType || 'application/octet-stream',
    ...options.metadata
  };
  
  await minio.putObject(BUCKET, objectName, fileBuffer, fileBuffer.length, metadata);
  console.log(`Uploaded: ${objectName}`);
  
  return objectName;
}

// Upload from buffer/stream
async function uploadBuffer(buffer, objectName, contentType = 'application/octet-stream') {
  await ensureBucket();
  await minio.putObject(BUCKET, objectName, buffer, buffer.length, {
    'Content-Type': contentType
  });
  return objectName;
}

// Download file
async function downloadFile(objectName, localPath) {
  return new Promise((resolve, reject) => {
    minio.getObject(BUCKET, objectName, (err, stream) => {
      if (err) return reject(err);
      
      const fileStream = fs.createWriteStream(localPath);
      stream.pipe(fileStream);
      
      fileStream.on('finish', () => {
        console.log(`Downloaded: ${objectName} → ${localPath}`);
        resolve(localPath);
      });
      
      fileStream.on('error', reject);
    });
  });
}

// Get file as buffer
async function getFileBuffer(objectName) {
  const chunks = [];
  const stream = await minio.getObject(BUCKET, objectName);
  
  for await (const chunk of stream) {
    chunks.push(chunk);
  }
  
  return Buffer.concat(chunks);
}

// Delete file
async function deleteFile(objectName) {
  await minio.removeObject(BUCKET, objectName);
  console.log(`Deleted: ${objectName}`);
}

// Delete multiple files
async function deleteFiles(objectNames) {
  await minio.removeObjects(BUCKET, objectNames);
  console.log(`Deleted ${objectNames.length} files`);
}

// List files
async function listFiles(prefix = '', recursive = true) {
  const objects = [];
  
  const stream = minio.listObjects(BUCKET, prefix, recursive);
  
  for await (const obj of stream) {
    objects.push({
      name: obj.name,
      size: obj.size,
      lastModified: obj.lastModified,
      etag: obj.etag
    });
  }
  
  return objects;
}

// Get file info (stat)
async function getFileInfo(objectName) {
  const stat = await minio.statObject(BUCKET, objectName);
  return {
    name: objectName,
    size: stat.size,
    lastModified: stat.lastModified,
    etag: stat.etag,
    contentType: stat.metaData['content-type'],
    metadata: stat.metaData
  };
}

// Get presigned download URL
async function getDownloadUrl(objectName, expirySeconds = 3600) {
  return minio.presignedGetObject(BUCKET, objectName, expirySeconds);
}

// Get presigned upload URL
async function getUploadUrl(objectName, expirySeconds = 3600) {
  return minio.presignedPutObject(BUCKET, objectName, expirySeconds);
}

// Copy object
async function copyObject(sourceObject, destObject, destBucket = BUCKET) {
  const conds = new Minio.CopyConditions();
  await minio.copyObject(destBucket, destObject, `/${BUCKET}/${sourceObject}`, conds);
  return destObject;
}

// ===========================
// Express.js Integration
// ===========================
const express = require('express');
const multer = require('multer');

const app = express();
const upload = multer({ storage: multer.memoryStorage() });

// Upload endpoint
app.post('/api/files/upload', upload.single('file'), async (req, res) => {
  try {
    const userId = req.user?.id || 'anonymous';
    const file = req.file;
    
    if (!file) {
      return res.status(400).json({ error: 'No file provided' });
    }
    
    // สร้าง object name
    const ext = path.extname(file.originalname);
    const objectName = `users/${userId}/${Date.now()}-${crypto.randomBytes(4).toString('hex')}${ext}`;
    
    // อัพโหลด
    await minio.putObject(BUCKET, objectName, file.buffer, file.size, {
      'Content-Type': file.mimetype,
      'x-amz-meta-original-name': file.originalname,
      'x-amz-meta-uploaded-by': userId
    });
    
    // สร้าง presigned URL สำหรับ view
    const viewUrl = await getDownloadUrl(objectName, 86400);
    
    res.json({
      success: true,
      objectName,
      size: file.size,
      contentType: file.mimetype,
      viewUrl,
      message: 'File uploaded successfully'
    });
    
  } catch (error) {
    console.error('Upload error:', error);
    res.status(500).json({ error: error.message });
  }
});

// List files endpoint
app.get('/api/files', async (req, res) => {
  try {
    const { prefix = '' } = req.query;
    const files = await listFiles(prefix);
    
    const filesWithUrls = await Promise.all(
      files.slice(0, 50).map(async (file) => ({
        ...file,
        downloadUrl: await getDownloadUrl(file.name, 3600)
      }))
    );
    
    res.json({ files: filesWithUrls, total: files.length });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Download endpoint (redirect to presigned URL)
app.get('/api/files/:filename(*)/download', async (req, res) => {
  try {
    const objectName = req.params.filename;
    const url = await getDownloadUrl(objectName, 300);
    res.redirect(url);
  } catch (error) {
    if (error.code === 'NoSuchKey') {
      return res.status(404).json({ error: 'File not found' });
    }
    res.status(500).json({ error: error.message });
  }
});

// Delete endpoint
app.delete('/api/files/:filename(*)', async (req, res) => {
  try {
    await deleteFile(req.params.filename);
    res.json({ success: true, message: 'File deleted' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Pre-signed upload URL endpoint
app.post('/api/files/presigned-upload', async (req, res) => {
  try {
    const { filename, contentType } = req.body;
    const userId = req.user?.id || 'anonymous';
    
    const ext = path.extname(filename);
    const objectName = `users/${userId}/${Date.now()}-${crypto.randomBytes(4).toString('hex')}${ext}`;
    
    const uploadUrl = await getUploadUrl(objectName, 3600);
    
    res.json({
      uploadUrl,
      objectName,
      expiresIn: 3600,
      instructions: `PUT ${contentType} body to ${uploadUrl}`
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Health check
app.get('/health', async (req, res) => {
  try {
    await minio.bucketExists(BUCKET);
    res.json({ status: 'healthy', minio: 'connected' });
  } catch (error) {
    res.status(503).json({ status: 'unhealthy', error: error.message });
  }
});

app.listen(3001, () => {
  console.log('File service running on port 3001');
  ensureBucket().catch(console.error);
});

module.exports = {
  uploadFile,
  uploadBuffer,
  downloadFile,
  getFileBuffer,
  deleteFile,
  listFiles,
  getFileInfo,
  getDownloadUrl,
  getUploadUrl
};
```

### 15.4 Python boto3 Complete Example

```python
# minio_service.py
import boto3
import os
import hashlib
import mimetypes
from botocore.config import Config
from botocore.exceptions import ClientError
from io import BytesIO
from pathlib import Path

class MinIOService:
    def __init__(self, 
                 endpoint='http://localhost:9000',
                 access_key='minioadmin',
                 secret_key='minioadmin123',
                 bucket_name='uploads'):
        
        self.s3 = boto3.client(
            's3',
            endpoint_url=endpoint,
            aws_access_key_id=access_key,
            aws_secret_access_key=secret_key,
            config=Config(signature_version='s3v4'),
            region_name='us-east-1'
        )
        self.bucket = bucket_name
        self._ensure_bucket()
    
    def _ensure_bucket(self):
        try:
            self.s3.head_bucket(Bucket=self.bucket)
        except ClientError as e:
            if e.response['Error']['Code'] == '404':
                self.s3.create_bucket(Bucket=self.bucket)
                print(f"Created bucket: {self.bucket}")
            else:
                raise
    
    def upload_file(self, file_path, object_key=None, metadata=None):
        """อัพโหลดไฟล์จาก local path"""
        file_path = Path(file_path)
        
        if not object_key:
            import secrets
            object_key = f"{int(__import__('time').time())}-{secrets.token_hex(4)}{file_path.suffix}"
        
        content_type = mimetypes.guess_type(str(file_path))[0] or 'application/octet-stream'
        
        extra_args = {'ContentType': content_type}
        if metadata:
            extra_args['Metadata'] = metadata
        
        self.s3.upload_file(
            str(file_path),
            self.bucket,
            object_key,
            ExtraArgs=extra_args
        )
        
        print(f"Uploaded: {file_path} → {object_key}")
        return object_key
    
    def upload_bytes(self, data: bytes, object_key: str, content_type: str = 'application/octet-stream'):
        """อัพโหลดจาก bytes"""
        self.s3.put_object(
            Bucket=self.bucket,
            Key=object_key,
            Body=data,
            ContentType=content_type
        )
        return object_key
    
    def download_file(self, object_key: str, local_path: str = None) -> bytes:
        """ดาวน์โหลดไฟล์"""
        response = self.s3.get_object(Bucket=self.bucket, Key=object_key)
        data = response['Body'].read()
        
        if local_path:
            with open(local_path, 'wb') as f:
                f.write(data)
            print(f"Downloaded: {object_key} → {local_path}")
        
        return data
    
    def delete_file(self, object_key: str):
        """ลบไฟล์"""
        self.s3.delete_object(Bucket=self.bucket, Key=object_key)
    
    def list_files(self, prefix='', max_items=1000) -> list:
        """list ไฟล์ทั้งหมด"""
        paginator = self.s3.get_paginator('list_objects_v2')
        files = []
        
        for page in paginator.paginate(Bucket=self.bucket, Prefix=prefix):
            for obj in page.get('Contents', []):
                files.append({
                    'key': obj['Key'],
                    'size': obj['Size'],
                    'last_modified': obj['LastModified'],
                    'etag': obj['ETag']
                })
        
        return files[:max_items]
    
    def get_presigned_download_url(self, object_key: str, expiry: int = 3600) -> str:
        """สร้าง pre-signed URL สำหรับ download"""
        return self.s3.generate_presigned_url(
            'get_object',
            Params={'Bucket': self.bucket, 'Key': object_key},
            ExpiresIn=expiry
        )
    
    def get_presigned_upload_url(self, object_key: str, expiry: int = 3600) -> str:
        """สร้าง pre-signed URL สำหรับ upload"""
        return self.s3.generate_presigned_url(
            'put_object',
            Params={'Bucket': self.bucket, 'Key': object_key},
            ExpiresIn=expiry
        )
    
    def get_file_info(self, object_key: str) -> dict:
        """ดู metadata ของไฟล์"""
        response = self.s3.head_object(Bucket=self.bucket, Key=object_key)
        return {
            'key': object_key,
            'size': response['ContentLength'],
            'content_type': response['ContentType'],
            'last_modified': response['LastModified'],
            'etag': response['ETag'],
            'metadata': response.get('Metadata', {})
        }
    
    def file_exists(self, object_key: str) -> bool:
        """ตรวจสอบว่าไฟล์มีอยู่ไหม"""
        try:
            self.s3.head_object(Bucket=self.bucket, Key=object_key)
            return True
        except ClientError:
            return False

# ใช้งาน
if __name__ == '__main__':
    service = MinIOService()
    
    print("=== MinIO Service Test ===\n")
    
    # Test upload
    test_data = b"Hello, MinIO from Python!"
    key = service.upload_bytes(test_data, "test/hello.txt", "text/plain")
    print(f"Uploaded: {key}")
    
    # Test download
    data = service.download_file("test/hello.txt")
    print(f"Downloaded: {data.decode()}")
    
    # Test presigned URL
    url = service.get_presigned_download_url("test/hello.txt", 300)
    print(f"Download URL (valid 5 min): {url[:80]}...")
    
    # Test list
    files = service.list_files("test/")
    print(f"Files in 'test/' prefix: {len(files)}")
    
    # Test info
    info = service.get_file_info("test/hello.txt")
    print(f"File info: size={info['size']}, type={info['content_type']}")
    
    # Test delete
    service.delete_file("test/hello.txt")
    print("Deleted test file")
    
    print("\n=== Test Complete ===")
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Object Storage**: แตกต่างจาก File System และ Block Storage อย่างไร
2. **S3 API Concepts**: Buckets, Objects, Keys, Prefixes, URL formats
3. **MinIO**: open-source S3-compatible storage
4. **ติดตั้ง MinIO**: Docker single node และ multi-node cluster
5. **MinIO Console**: Web UI สำหรับจัดการผ่าน browser
6. **mc CLI**: คำสั่งทั้งหมดสำหรับจัดการ MinIO
7. **Bucket Policies**: public, private, download, custom JSON policies
8. **Versioning**: เก็บหลาย versions ป้องกันการลบโดยไม่ตั้งใจ
9. **Lifecycle Policies**: auto-delete, transition objects
10. **Pre-signed URLs**: share files ชั่วคราวโดยไม่เปิด public
11. **Object Metadata & Tags**: เก็บข้อมูลเพิ่มเติมกับ objects
12. **Multipart Upload**: สำหรับไฟล์ขนาดใหญ่
13. **Server-side Encryption**: SSE-S3, SSE-KMS, SSE-C
14. **MinIO vs AWS S3**: เมื่อไหร่ควรใช้อะไร
15. **Workshop**: Application ครบวงจรด้วย Node.js และ Python

---

*[Part 10 จบ — ครบ Part 01-10 แล้ว!]*
