# Part 92: Security - Encryption at Rest and in Transit

## บทนำ

Encryption (การเข้ารหัส) เป็นหนึ่งในมาตรการรักษาความปลอดภัยที่สำคัญที่สุดสำหรับ database systems การเข้ารหัสแบ่งออกเป็น 2 ประเภทหลัก:

1. **Encryption at Rest**: ข้อมูลที่เก็บอยู่ใน disk ถูกเข้ารหัส
2. **Encryption in Transit**: ข้อมูลที่ส่งผ่านเครือข่ายถูกเข้ารหัส

---

## 1. Encryption at Rest

### 1.1 ทำไมต้อง Encrypt at Rest?

- ป้องกันการเข้าถึงข้อมูลเมื่อ disk ถูกขโมย
- Compliance: GDPR, HIPAA, PCI-DSS กำหนดให้ต้อง encrypt
- ป้องกันเมื่อ server ถูก compromise บางส่วน
- Cloud: ป้องกันเมื่อ provider staff access

### 1.2 PostgreSQL: pgcrypto Extension

```sql
-- ติดตั้ง pgcrypto
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- ตรวจสอบ
SELECT * FROM pg_extension WHERE extname = 'pgcrypto';
```

**Column-level Encryption ด้วย pgcrypto:**

```sql
-- สร้าง table สำหรับเก็บข้อมูลที่ sensitive
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT NOT NULL UNIQUE,
    -- เก็บ encrypted data เป็น bytea
    phone_encrypted BYTEA,
    id_card_encrypted BYTEA,
    credit_card_encrypted BYTEA,
    -- เก็บ hash สำหรับ password
    password_hash TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

**pgp_sym_encrypt / pgp_sym_decrypt:**

```sql
-- Encryption key (ควรเก็บใน environment variable ไม่ใช่ใน DB)
-- ในตัวอย่างนี้ใช้ 'my-secret-key' แต่ใน production ต้องใช้ key จาก KMS

-- INSERT with encryption
INSERT INTO users (email, phone_encrypted, id_card_encrypted, password_hash)
VALUES (
    'john@example.com',
    pgp_sym_encrypt('0812345678', current_setting('app.encryption_key')),
    pgp_sym_encrypt('1234567890123', current_setting('app.encryption_key')),
    crypt('my_password', gen_salt('bf', 10))
);

-- SELECT with decryption
SELECT
    email,
    pgp_sym_decrypt(phone_encrypted, current_setting('app.encryption_key')) AS phone,
    pgp_sym_decrypt(id_card_encrypted, current_setting('app.encryption_key')) AS id_card
FROM users
WHERE email = 'john@example.com';

-- UPDATE: re-encrypt with new data
UPDATE users
SET phone_encrypted = pgp_sym_encrypt('0899999999', current_setting('app.encryption_key'))
WHERE email = 'john@example.com';
```

**crypt() สำหรับ Password Hashing:**

```sql
-- Hash password ด้วย bcrypt (bf = Blowfish = bcrypt)
-- cost factor 10 = 2^10 iterations (ยิ่งมาก ยิ่งช้า ยิ่งปลอดภัย)
SELECT crypt('my_password', gen_salt('bf', 12)) AS password_hash;
-- ได้ $2a$12$...

-- Verify password
SELECT crypt('my_password', password_hash) = password_hash AS is_valid
FROM users
WHERE email = 'john@example.com';

-- สรุป: hash algorithms ที่รองรับ
-- 'bf'  = Blowfish (bcrypt) - แนะนำ
-- 'md5' = MD5 - ไม่แนะนำ
-- 'xdes' = Extended DES - ไม่แนะนำ
-- 'des' = DES - ไม่แนะนำ
```

**gen_random_bytes() สำหรับ Random Data:**

```sql
-- สร้าง random bytes (สำหรับ tokens, keys, etc.)
SELECT encode(gen_random_bytes(32), 'hex') AS random_token;
-- ได้ 64-character hex string

-- สร้าง random UUID v4
SELECT gen_random_uuid();

-- สร้าง API key
SELECT encode(gen_random_bytes(24), 'base64') AS api_key;

-- สร้าง secure OTP
SELECT floor(random() * 900000 + 100000)::INT AS otp;
-- หรือ
SELECT lpad((floor(random() * 1000000))::TEXT, 6, '0') AS otp;
```

**Asymmetric Encryption ด้วย pgcrypto:**

```sql
-- Public key encryption (ใช้ OpenPGP format)
-- ต้องมี public key และ private key

-- Encrypt ด้วย public key (ทุกคน encrypt ได้)
SELECT pgp_pub_encrypt(
    'sensitive data',
    dearmor('-----BEGIN PGP PUBLIC KEY BLOCK-----
...your public key here...
-----END PGP PUBLIC KEY BLOCK-----')
) AS encrypted_data;

-- Decrypt ด้วย private key (เฉพาะผู้ที่มี private key)
SELECT pgp_pub_decrypt(
    encrypted_data,
    dearmor('-----BEGIN PGP PRIVATE KEY BLOCK-----
...your private key here...
-----END PGP PRIVATE KEY BLOCK-----'),
    'passphrase'
) AS decrypted_data
FROM sensitive_table;
```

### 1.3 Searchable Encryption

ปัญหาของ column encryption คือ ไม่สามารถ query/search ได้ โซลูชัน:

```sql
-- Pattern 1: เก็บ hash เพิ่มสำหรับ lookup
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT NOT NULL UNIQUE,
    -- เก็บ email hash สำหรับ lookup
    email_hash TEXT GENERATED ALWAYS AS (
        encode(digest(lower(email), 'sha256'), 'hex')
    ) STORED,
    phone_encrypted BYTEA,
    -- เก็บ phone hash สำหรับ lookup (ไม่เปิดเผยค่าจริง)
    phone_last4 TEXT, -- เก็บแค่ 4 หลักสุดท้าย
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index บน hash
CREATE INDEX idx_users_email_hash ON users(email_hash);
CREATE INDEX idx_users_phone_last4 ON users(phone_last4);

-- Search by phone last 4 digits
SELECT email, pgp_sym_decrypt(phone_encrypted, 'key') AS phone
FROM users
WHERE phone_last4 = '5678';
```

```sql
-- Pattern 2: Bloom filter (ใช้ pg_trgm สำหรับ fuzzy search)
CREATE EXTENSION pg_trgm;

-- เก็บ hmac ของ trigrams สำหรับ search
CREATE OR REPLACE FUNCTION create_search_tokens(value TEXT, hmac_key TEXT)
RETURNS TEXT[] AS $$
DECLARE
    trigrams TEXT[];
    tokens TEXT[] := '{}';
    t TEXT;
BEGIN
    SELECT array_agg(DISTINCT show_trgm) INTO trigrams
    FROM unnest(show_trgm(value)) AS show_trgm;
    
    FOREACH t IN ARRAY trigrams LOOP
        tokens := tokens || encode(hmac(t, hmac_key, 'sha256'), 'hex');
    END LOOP;
    
    RETURN tokens;
END;
$$ LANGUAGE plpgsql;
```

### 1.4 AWS RDS Encryption at Rest

```hcl
# terraform/rds.tf
resource "aws_db_instance" "main" {
  identifier        = "myapp-postgres"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.t3.medium"
  allocated_storage = 100
  
  db_name  = "myapp"
  username = "postgres"
  password = var.db_password
  
  # Enable encryption at rest
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn
  
  # Multi-AZ for production
  multi_az = true
  
  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "Mon:04:00-Mon:05:00"
  
  # Enhanced monitoring
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn
  
  # Performance Insights
  performance_insights_enabled          = true
  performance_insights_retention_period = 7
  
  tags = {
    Environment = "production"
    Project     = "myapp"
  }
}

# KMS key สำหรับ RDS
resource "aws_kms_key" "rds" {
  description             = "KMS key for RDS encryption"
  deletion_window_in_days = 7
  enable_key_rotation     = true
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      }
    ]
  })
  
  tags = {
    Purpose = "RDS Encryption"
  }
}

resource "aws_kms_alias" "rds" {
  name          = "alias/myapp-rds"
  target_key_id = aws_kms_key.rds.key_id
}
```

### 1.5 Disk-level Encryption: LUKS บน Linux

```bash
#!/bin/bash
# setup-luks-encryption.sh

# Variables
DEVICE="/dev/xvdf"          # Data disk
MAPPER_NAME="encrypted-data"
MOUNT_POINT="/data"

# 1. สร้าง LUKS container
cryptsetup luksFormat \
    --type luks2 \
    --cipher aes-xts-plain64 \
    --key-size 512 \
    --hash sha512 \
    --iter-time 5000 \
    $DEVICE

# 2. เปิด encrypted device
cryptsetup luksOpen $DEVICE $MAPPER_NAME

# 3. สร้าง filesystem
mkfs.ext4 /dev/mapper/$MAPPER_NAME

# 4. Mount
mkdir -p $MOUNT_POINT
mount /dev/mapper/$MAPPER_NAME $MOUNT_POINT

# 5. Auto-mount on boot (ต้องมี key file)
# สร้าง key file
dd if=/dev/urandom bs=512 count=4 > /root/luks-keyfile.bin
chmod 600 /root/luks-keyfile.bin

# เพิ่ม key file เข้า LUKS
cryptsetup luksAddKey $DEVICE /root/luks-keyfile.bin

# เพิ่มใน /etc/crypttab
echo "$MAPPER_NAME $DEVICE /root/luks-keyfile.bin luks" >> /etc/crypttab

# เพิ่มใน /etc/fstab
echo "/dev/mapper/$MAPPER_NAME $MOUNT_POINT ext4 defaults 0 2" >> /etc/fstab
```

### 1.6 Application-level Encryption: Node.js

```javascript
// lib/encryption.js
const crypto = require('crypto');

const ALGORITHM = 'aes-256-gcm';
const KEY_LENGTH = 32;  // 256 bits
const IV_LENGTH = 16;   // 128 bits
const TAG_LENGTH = 16;  // 128 bits authentication tag

class Encryption {
  constructor(encryptionKey) {
    if (!encryptionKey || encryptionKey.length !== KEY_LENGTH) {
      throw new Error(`Encryption key must be ${KEY_LENGTH} bytes`);
    }
    this.key = Buffer.isBuffer(encryptionKey) 
      ? encryptionKey 
      : Buffer.from(encryptionKey, 'hex');
  }
  
  /**
   * Encrypt data
   * @param {string} plaintext - Data to encrypt
   * @returns {string} - Base64 encoded encrypted data (iv:tag:ciphertext)
   */
  encrypt(plaintext) {
    const iv = crypto.randomBytes(IV_LENGTH);
    const cipher = crypto.createCipheriv(ALGORITHM, this.key, iv);
    
    let encrypted = cipher.update(plaintext, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const tag = cipher.getAuthTag();
    
    // Format: iv:tag:ciphertext (all hex encoded)
    return [
      iv.toString('hex'),
      tag.toString('hex'),
      encrypted,
    ].join(':');
  }
  
  /**
   * Decrypt data
   * @param {string} encryptedData - Encrypted string (iv:tag:ciphertext)
   * @returns {string} - Decrypted plaintext
   */
  decrypt(encryptedData) {
    const [ivHex, tagHex, ciphertext] = encryptedData.split(':');
    
    const iv = Buffer.from(ivHex, 'hex');
    const tag = Buffer.from(tagHex, 'hex');
    
    const decipher = crypto.createDecipheriv(ALGORITHM, this.key, iv);
    decipher.setAuthTag(tag);
    
    let decrypted = decipher.update(ciphertext, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
  
  /**
   * Hash data (one-way)
   * @param {string} data 
   * @param {string} salt - Optional salt
   */
  hash(data, salt = '') {
    return crypto
      .createHmac('sha256', this.key)
      .update(data + salt)
      .digest('hex');
  }
  
  /**
   * Generate secure random token
   * @param {number} bytes 
   * @returns {string} hex string
   */
  static generateToken(bytes = 32) {
    return crypto.randomBytes(bytes).toString('hex');
  }
}

// Key derivation จาก password
function deriveKey(password, salt) {
  return crypto.pbkdf2Sync(
    password,
    salt,
    100000,  // iterations
    KEY_LENGTH,
    'sha512'
  );
}

// Load key จาก environment
function loadEncryptionKey() {
  const keyHex = process.env.ENCRYPTION_KEY;
  if (!keyHex) throw new Error('ENCRYPTION_KEY environment variable not set');
  
  const key = Buffer.from(keyHex, 'hex');
  if (key.length !== KEY_LENGTH) {
    throw new Error(`ENCRYPTION_KEY must be ${KEY_LENGTH * 2} hex characters`);
  }
  
  return key;
}

module.exports = { Encryption, deriveKey, loadEncryptionKey };
```

```javascript
// lib/sensitive-data.service.js
const { Encryption, loadEncryptionKey } = require('./encryption');
const { pool } = require('./db');

const enc = new Encryption(loadEncryptionKey());

class SensitiveDataService {
  async createUser(userData) {
    const { email, phone, idCard, password } = userData;
    
    // Encrypt sensitive fields
    const encryptedPhone = phone ? enc.encrypt(phone) : null;
    const encryptedIdCard = idCard ? enc.encrypt(idCard) : null;
    
    // Hash for searchability
    const phoneHash = phone ? enc.hash(phone.replace(/\D/g, '')) : null;
    const idCardHash = idCard ? enc.hash(idCard) : null;
    
    const result = await pool.query(`
      INSERT INTO users (
        email, 
        phone_encrypted,
        phone_hash,
        id_card_encrypted,
        id_card_hash,
        password_hash
      ) VALUES ($1, $2, $3, $4, $5, crypt($6, gen_salt('bf', 12)))
      RETURNING id, email, created_at
    `, [email, encryptedPhone, phoneHash, encryptedIdCard, idCardHash, password]);
    
    return result.rows[0];
  }
  
  async getUserByPhone(phone) {
    const phoneHash = enc.hash(phone.replace(/\D/g, ''));
    
    const result = await pool.query(
      'SELECT * FROM users WHERE phone_hash = $1',
      [phoneHash]
    );
    
    if (result.rows.length === 0) return null;
    
    const user = result.rows[0];
    
    // Decrypt sensitive fields
    return {
      id: user.id,
      email: user.email,
      phone: user.phone_encrypted ? enc.decrypt(user.phone_encrypted) : null,
      idCard: user.id_card_encrypted ? enc.decrypt(user.id_card_encrypted) : null,
    };
  }
}

module.exports = new SensitiveDataService();
```

### 1.7 Key Management: AWS KMS

```javascript
// lib/kms-encryption.js
const { KMSClient, EncryptCommand, DecryptCommand, GenerateDataKeyCommand } = require('@aws-sdk/client-kms');
const crypto = require('crypto');

const kms = new KMSClient({ region: process.env.AWS_REGION || 'ap-southeast-1' });
const KEY_ID = process.env.KMS_KEY_ID;

class KMSEncryption {
  /**
   * Envelope encryption: 
   * 1. Generate data key จาก KMS
   * 2. Encrypt data ด้วย data key (local)
   * 3. เก็บ encrypted data key กับ ciphertext
   */
  async encrypt(plaintext) {
    // 1. Generate data key
    const { Plaintext: dataKey, CiphertextBlob: encryptedDataKey } = 
      await kms.send(new GenerateDataKeyCommand({
        KeyId: KEY_ID,
        KeySpec: 'AES_256',
      }));
    
    // 2. Encrypt data locally
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv('aes-256-gcm', dataKey, iv);
    
    let ciphertext = cipher.update(plaintext, 'utf8', 'hex');
    ciphertext += cipher.final('hex');
    const tag = cipher.getAuthTag();
    
    // Clear plaintext key from memory
    dataKey.fill(0);
    
    // 3. Return encrypted data + encrypted key
    return JSON.stringify({
      encryptedKey: Buffer.from(encryptedDataKey).toString('base64'),
      iv: iv.toString('hex'),
      tag: tag.toString('hex'),
      ciphertext,
    });
  }
  
  async decrypt(encryptedPayload) {
    const { encryptedKey, iv, tag, ciphertext } = JSON.parse(encryptedPayload);
    
    // 1. Decrypt data key using KMS
    const { Plaintext: dataKey } = await kms.send(new DecryptCommand({
      KeyId: KEY_ID,
      CiphertextBlob: Buffer.from(encryptedKey, 'base64'),
    }));
    
    // 2. Decrypt data locally
    const decipher = crypto.createDecipheriv(
      'aes-256-gcm',
      dataKey,
      Buffer.from(iv, 'hex')
    );
    decipher.setAuthTag(Buffer.from(tag, 'hex'));
    
    let plaintext = decipher.update(ciphertext, 'hex', 'utf8');
    plaintext += decipher.final('utf8');
    
    // Clear key from memory
    dataKey.fill(0);
    
    return plaintext;
  }
}

module.exports = new KMSEncryption();
```

### 1.8 Redis Encryption at Rest

```bash
# Redis ไม่มี built-in encryption at rest
# ต้องใช้ OS-level encryption หรือ Redis Enterprise

# Option 1: LUKS encryption สำหรับ Redis data directory
# ดูตัวอย่าง LUKS ด้านบน

# Option 2: Redis Enterprise (commercial)
# มี encryption at rest built-in

# Option 3: ใช้ application-level encryption
# Encrypt ก่อน store ใน Redis
```

```javascript
// lib/encrypted-redis.js
const Redis = require('ioredis');
const { Encryption, loadEncryptionKey } = require('./encryption');

class EncryptedRedis {
  constructor(redisOptions) {
    this.redis = new Redis(redisOptions);
    this.enc = new Encryption(loadEncryptionKey());
  }
  
  async set(key, value, options = {}) {
    const serialized = JSON.stringify(value);
    const encrypted = this.enc.encrypt(serialized);
    
    if (options.ex) {
      return this.redis.setex(key, options.ex, encrypted);
    }
    return this.redis.set(key, encrypted);
  }
  
  async get(key) {
    const encrypted = await this.redis.get(key);
    if (!encrypted) return null;
    
    const decrypted = this.enc.decrypt(encrypted);
    return JSON.parse(decrypted);
  }
  
  async del(key) {
    return this.redis.del(key);
  }
  
  // Delegate other methods to raw redis
  pipeline() {
    return this.redis.pipeline();
  }
}

module.exports = EncryptedRedis;
```

### 1.9 S3/MinIO Encryption

```javascript
// lib/s3-encryption.js
const { S3Client, PutObjectCommand, GetObjectCommand } = require('@aws-sdk/client-s3');

const s3 = new S3Client({ region: process.env.AWS_REGION });

// SSE-KMS: Server-side encryption with KMS
async function uploadWithSSEKMS(bucket, key, data) {
  return s3.send(new PutObjectCommand({
    Bucket: bucket,
    Key: key,
    Body: data,
    ServerSideEncryption: 'aws:kms',
    SSEKMSKeyId: process.env.S3_KMS_KEY_ID,
  }));
}

// SSE-S3: Server-side encryption with S3 managed keys
async function uploadWithSSES3(bucket, key, data) {
  return s3.send(new PutObjectCommand({
    Bucket: bucket,
    Key: key,
    Body: data,
    ServerSideEncryption: 'AES256',
  }));
}

// SSE-C: Customer-provided encryption key
async function uploadWithSSEC(bucket, key, data, encryptionKey) {
  const keyMD5 = require('crypto')
    .createHash('md5')
    .update(encryptionKey)
    .digest('base64');
    
  return s3.send(new PutObjectCommand({
    Bucket: bucket,
    Key: key,
    Body: data,
    SSECustomerAlgorithm: 'AES256',
    SSECustomerKey: encryptionKey.toString('base64'),
    SSECustomerKeyMD5: keyMD5,
  }));
}
```

```hcl
# terraform: S3 bucket with encryption policy
resource "aws_s3_bucket" "data" {
  bucket = "myapp-data-${random_id.suffix.hex}"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true  # ลด KMS API calls
  }
}

# Deny unencrypted uploads
resource "aws_s3_bucket_policy" "deny_unencrypted" {
  bucket = aws_s3_bucket.data.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyUnencryptedObjectUploads"
        Effect = "Deny"
        Principal = "*"
        Action = "s3:PutObject"
        Resource = "${aws_s3_bucket.data.arn}/*"
        Condition = {
          StringNotEquals = {
            "s3:x-amz-server-side-encryption" = "aws:kms"
          }
        }
      }
    ]
  })
}
```

---

## 2. Encryption in Transit

### 2.1 PostgreSQL TLS Configuration

```bash
# postgresql.conf
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'
ssl_ca_file = '/etc/ssl/certs/ca.crt'

# Minimum TLS version
ssl_min_protocol_version = 'TLSv1.2'

# Cipher suites (TLS 1.2)
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'

# สำหรับ TLS 1.3 จะใช้ cipher ที่ดีขึ้นอัตโนมัติ
```

**สร้าง Self-signed Certificate:**

```bash
#!/bin/bash
# create-ssl-certs.sh

# สร้าง CA key และ certificate
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
    -subj "/C=TH/ST=Bangkok/O=MyCompany/CN=PostgreSQL CA"

# สร้าง Server key และ CSR
openssl genrsa -out server.key 4096
openssl req -new -key server.key -out server.csr \
    -subj "/C=TH/ST=Bangkok/O=MyCompany/CN=db.myapp.com"

# Sign server certificate ด้วย CA
openssl x509 -req -days 365 -in server.csr \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -out server.crt

# สร้าง Client certificate
openssl genrsa -out client.key 4096
openssl req -new -key client.key -out client.csr \
    -subj "/C=TH/ST=Bangkok/O=MyCompany/CN=app_user"
openssl x509 -req -days 365 -in client.csr \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -out client.crt

# Set permissions
chmod 600 server.key client.key ca.key
chown postgres:postgres server.key server.crt ca.crt

echo "Certificates created successfully"
```

**pg_hba.conf สำหรับ SSL:**

```
# pg_hba.conf
# ต้องการ SSL สำหรับทุก remote connection

# Local connections ไม่ต้องการ SSL
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5

# Remote connections ต้องใช้ SSL
hostssl all             all             0.0.0.0/0               scram-sha-256

# Client certificate authentication
hostssl all             app_user        0.0.0.0/0               cert clientcert=verify-full
```

**sslmode Options:**

```javascript
// Node.js connection examples

// ❌ Disable SSL (ไม่แนะนำ production)
const pool1 = new Pool({ ssl: false });

// ✅ Require SSL (แต่ไม่ verify certificate)
const pool2 = new Pool({ ssl: { rejectUnauthorized: false } });

// ✅ Verify CA (server must have certificate signed by our CA)
const pool3 = new Pool({
  ssl: {
    rejectUnauthorized: true,
    ca: fs.readFileSync('/etc/ssl/certs/ca.crt').toString(),
  }
});

// ✅ Full verification (hostname + certificate)
const pool4 = new Pool({
  host: 'db.myapp.com',
  ssl: {
    rejectUnauthorized: true,
    ca: fs.readFileSync('/etc/ssl/certs/ca.crt').toString(),
    // Client certificate (if required)
    cert: fs.readFileSync('/etc/ssl/certs/client.crt').toString(),
    key: fs.readFileSync('/etc/ssl/private/client.key').toString(),
  }
});
```

```
# sslmode ใน connection string
postgresql://user:pass@host:5432/db?sslmode=verify-full&sslrootcert=/etc/ssl/ca.crt
```

| sslmode | Encrypt | Verify CA | Verify Hostname |
|---------|---------|-----------|-----------------|
| disable | ไม่ | ไม่ | ไม่ |
| allow | ถ้าทำได้ | ไม่ | ไม่ |
| prefer | ถ้าทำได้ | ไม่ | ไม่ |
| require | ใช่ | ไม่ | ไม่ |
| verify-ca | ใช่ | ใช่ | ไม่ |
| verify-full | ใช่ | ใช่ | ใช่ |

### 2.2 Redis TLS Configuration

```bash
# redis.conf - Redis 6+ native TLS

# Disable non-TLS port (optional, recommended)
port 0

# Enable TLS port
tls-port 6380

# TLS certificates
tls-cert-file /etc/ssl/redis/server.crt
tls-key-file /etc/ssl/redis/server.key
tls-ca-cert-file /etc/ssl/redis/ca.crt

# Require client certificates
tls-auth-clients yes

# TLS protocols
tls-protocols "TLSv1.2 TLSv1.3"

# Cipher suites (TLS 1.2)
tls-ciphers DEFAULT:!MEDIUM

# Replication TLS
tls-replication yes
tls-cluster yes
```

```javascript
// Node.js Redis TLS connection
const Redis = require('ioredis');
const fs = require('fs');

const redis = new Redis({
  host: 'redis.myapp.com',
  port: 6380,
  tls: {
    rejectUnauthorized: true,
    ca: fs.readFileSync('/etc/ssl/redis/ca.crt'),
    cert: fs.readFileSync('/etc/ssl/redis/client.crt'),
    key: fs.readFileSync('/etc/ssl/redis/client.key'),
  },
  password: process.env.REDIS_PASSWORD,
});
```

**Stunnel เป็น TLS Proxy สำหรับ Redis เก่า:**

```
# stunnel.conf
[redis-server]
accept = 6380
connect = 127.0.0.1:6379
cert = /etc/ssl/redis/server.crt
key = /etc/ssl/redis/server.key
CAfile = /etc/ssl/redis/ca.crt
verify = 2  # require client certificate
```

### 2.3 MinIO TLS

```bash
# MinIO environment variables
export MINIO_CERT_FILE=/etc/ssl/minio/server.crt
export MINIO_KEY_FILE=/etc/ssl/minio/server.key

# หรือผ่าน directory
# MinIO จะอ่าน TLS certs จาก ~/.minio/certs/ โดยอัตโนมัติ
mkdir -p ~/.minio/certs/CAs
cp server.crt ~/.minio/certs/public.crt
cp server.key ~/.minio/certs/private.key
cp ca.crt ~/.minio/certs/CAs/ca.crt

# Start MinIO
minio server /data
```

```javascript
// MinIO client with TLS
const Minio = require('minio');
const fs = require('fs');

const minioClient = new Minio.Client({
  endPoint: 'minio.myapp.com',
  port: 9000,
  useSSL: true,
  accessKey: process.env.MINIO_ACCESS_KEY,
  secretKey: process.env.MINIO_SECRET_KEY,
  // Custom CA (self-signed)
  ca: fs.readFileSync('/etc/ssl/minio/ca.crt'),
});
```

---

## 3. Key Management

### 3.1 HashiCorp Vault

```bash
# ติดตั้ง Vault (development mode)
vault server -dev -dev-root-token-id="root"

# Production: ใช้ Consul backend
vault server -config=/etc/vault.d/vault.hcl
```

```hcl
# vault.hcl
storage "consul" {
  address = "127.0.0.1:8500"
  path    = "vault/"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_cert_file = "/etc/vault/tls/server.crt"
  tls_key_file  = "/etc/vault/tls/server.key"
}

api_addr = "https://vault.myapp.com:8200"
cluster_addr = "https://vault.myapp.com:8201"

seal "awskms" {
  region     = "ap-southeast-1"
  kms_key_id = "alias/vault-unseal"
}
```

**PostgreSQL Dynamic Credentials:**

```bash
# Enable database secrets engine
vault secrets enable database

# Configure PostgreSQL connection
vault write database/config/myapp-postgres \
    plugin_name=postgresql-database-plugin \
    allowed_roles="myapp-role" \
    connection_url="postgresql://{{username}}:{{password}}@localhost:5432/myapp?sslmode=verify-full" \
    username="vault_admin" \
    password="vault_admin_password" \
    root_rotation_statements="ALTER USER '{{username}}' WITH PASSWORD '{{password}}'"

# สร้าง role: dynamic credentials ที่มี TTL
vault write database/roles/myapp-role \
    db_name=myapp-postgres \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
                         GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO \"{{name}}\"; \
                         GRANT USAGE ON ALL SEQUENCES IN SCHEMA app TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"

# Get credentials
vault read database/creds/myapp-role
# Key                Value
# ---                -----
# lease_id           database/creds/myapp-role/xyz123
# lease_duration     1h
# lease_renewable    true
# password           A1a-xxx-yyy
# username           v-myapp-role-xxxxxxxx
```

**AppRole Authentication:**

```bash
# Enable AppRole auth
vault auth enable approle

# สร้าง policy
vault policy write myapp-policy - <<EOF
path "database/creds/myapp-role" {
  capabilities = ["read"]
}
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
EOF

# สร้าง AppRole
vault write auth/approle/role/myapp \
    secret_id_ttl=24h \
    token_num_uses=10 \
    token_ttl=1h \
    token_max_ttl=4h \
    token_policies="myapp-policy"

# Get Role ID (ค่าคงที่ deploy กับ application)
vault read auth/approle/role/myapp/role-id

# Get Secret ID (หมดอายุ, generate ใหม่ได้)
vault write -f auth/approle/role/myapp/secret-id
```

```javascript
// lib/vault-client.js
const vault = require('node-vault');

class VaultClient {
  constructor() {
    this.client = vault({
      endpoint: process.env.VAULT_ADDR || 'https://vault.myapp.com:8200',
    });
    this.dbCreds = null;
    this.credRefreshTimer = null;
  }
  
  async authenticate() {
    // AppRole authentication
    const result = await this.client.approleLogin({
      role_id: process.env.VAULT_ROLE_ID,
      secret_id: process.env.VAULT_SECRET_ID,
    });
    
    this.client.token = result.auth.client_token;
    
    // Schedule token renewal
    const ttl = result.auth.lease_duration;
    setTimeout(() => this.authenticate(), (ttl * 0.8) * 1000);
  }
  
  async getDatabaseCredentials() {
    const result = await this.client.read('database/creds/myapp-role');
    
    const { username, password, lease_duration } = result;
    
    // Schedule credential renewal
    if (this.credRefreshTimer) clearTimeout(this.credRefreshTimer);
    this.credRefreshTimer = setTimeout(
      () => this.refreshDatabasePool(),
      (lease_duration * 0.8) * 1000
    );
    
    return { username, password };
  }
  
  async refreshDatabasePool() {
    console.log('Refreshing database credentials...');
    const creds = await this.getDatabaseCredentials();
    
    // Update connection pool with new credentials
    // (implementation depends on your pool library)
    await global.dbPool.updateCredentials(creds);
  }
  
  async getSecret(path) {
    const result = await this.client.read(`secret/data/${path}`);
    return result.data.data;
  }
}

module.exports = new VaultClient();
```

### 3.2 Key Rotation

```bash
# AWS KMS key rotation
aws kms enable-key-rotation \
    --key-id alias/myapp-rds

# ตรวจสอบ
aws kms get-key-rotation-status \
    --key-id alias/myapp-rds

# Manual rotation สำหรับ application keys
# 1. Generate new key
# 2. Re-encrypt ข้อมูลทั้งหมดด้วย new key
# 3. Delete old key
```

```javascript
// lib/key-rotation.js
class KeyRotation {
  async rotateEncryptionKey(oldKey, newKey) {
    const oldEnc = new Encryption(Buffer.from(oldKey, 'hex'));
    const newEnc = new Encryption(Buffer.from(newKey, 'hex'));
    
    const client = await pool.connect();
    
    try {
      await client.query('BEGIN');
      
      // Get all encrypted records
      const result = await client.query(
        'SELECT id, phone_encrypted, id_card_encrypted FROM users WHERE phone_encrypted IS NOT NULL'
      );
      
      console.log(`Rotating keys for ${result.rows.length} records...`);
      
      for (const row of result.rows) {
        // Decrypt with old key
        const phone = row.phone_encrypted 
          ? oldEnc.decrypt(row.phone_encrypted) 
          : null;
        const idCard = row.id_card_encrypted 
          ? oldEnc.decrypt(row.id_card_encrypted) 
          : null;
        
        // Re-encrypt with new key
        const newPhoneEnc = phone ? newEnc.encrypt(phone) : null;
        const newIdCardEnc = idCard ? newEnc.encrypt(idCard) : null;
        
        await client.query(
          'UPDATE users SET phone_encrypted = $1, id_card_encrypted = $2 WHERE id = $3',
          [newPhoneEnc, newIdCardEnc, row.id]
        );
      }
      
      await client.query('COMMIT');
      console.log('Key rotation completed');
      
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }
}
```

---

## 4. Secrets in Kubernetes

### 4.1 Kubernetes Secrets (Base64 Encoded เท่านั้น!)

```yaml
# ❌ Kubernetes Secret ธรรมดา - ไม่ encrypted! แค่ base64
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
data:
  # base64 encoded (ไม่ใช่ encrypted!)
  # echo -n 'mypassword' | base64
  DB_PASSWORD: bXlwYXNzd29yZA==
  DB_USER: YXBwX3VzZXI=
```

```bash
# Decode ได้ทันที - ไม่ปลอดภัยเลย
kubectl get secret db-credentials -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

### 4.2 Sealed Secrets (GitOps safe)

```bash
# ติดตั้ง Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml

# ติดตั้ง kubeseal CLI
brew install kubeseal
```

```bash
# สร้าง regular secret แล้ว seal
kubectl create secret generic db-credentials \
    --from-literal=DB_PASSWORD='mypassword' \
    --from-literal=DB_USER='app_user' \
    --dry-run=client -o yaml | \
    kubeseal \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=kube-system \
    --format yaml > sealed-secret.yaml

# sealed-secret.yaml สามารถ commit ลง Git ได้ปลอดภัย
# เพราะ decrypt ได้เฉพาะ Sealed Secrets controller ใน cluster
```

```yaml
# sealed-secret.yaml (safe to commit to Git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  encryptedData:
    DB_PASSWORD: AgA7fJ9...encrypted...
    DB_USER: AgA7fJ9...encrypted...
  template:
    metadata:
      name: db-credentials
      namespace: production
    type: Opaque
```

### 4.3 External Secrets Operator

```bash
# ติดตั้ง External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
    --namespace external-secrets \
    --create-namespace
```

```yaml
# secret-store.yaml - เชื่อม Vault
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.myapp.com:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
```

```yaml
# external-secret.yaml - sync secrets from Vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: secret/myapp/database
        property: password
    - secretKey: DB_USER
      remoteRef:
        key: secret/myapp/database
        property: username
    - secretKey: ENCRYPTION_KEY
      remoteRef:
        key: secret/myapp/encryption
        property: key
```

```yaml
# aws-secret-store.yaml - เชื่อม AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
```

---

## 5. Full Security Implementation Guide

### 5.1 Security Checklist

```markdown
## Database Encryption Security Checklist

### Encryption at Rest
- [ ] Database disk encrypted (LUKS หรือ cloud provider)
- [ ] RDS/Cloud DB: storage encryption enabled
- [ ] KMS key rotation enabled
- [ ] Sensitive columns: column-level encryption
- [ ] Redis data encrypted (application-level หรือ enterprise)
- [ ] S3/MinIO: SSE-KMS enabled
- [ ] Backups encrypted

### Encryption in Transit
- [ ] PostgreSQL: ssl = on
- [ ] PostgreSQL: sslmode = verify-full ใน application
- [ ] Redis: TLS enabled
- [ ] MinIO: TLS enabled
- [ ] Application: certificate validation ไม่ disabled
- [ ] TLS 1.2 minimum, prefer TLS 1.3
- [ ] Certificate rotation scheduled

### Key Management
- [ ] Encryption keys ไม่อยู่ใน source code
- [ ] Encryption keys ไม่อยู่ใน environment files ที่ commit ลง Git
- [ ] ใช้ KMS หรือ Vault สำหรับ key management
- [ ] Key rotation policy ตั้งไว้
- [ ] Backup keys ปลอดภัย

### Kubernetes
- [ ] ไม่ใช้ plain Kubernetes Secrets สำหรับ sensitive data
- [ ] ใช้ Sealed Secrets หรือ External Secrets Operator
- [ ] RBAC: จำกัดการเข้าถึง secrets
- [ ] Audit logging enabled
```

### 5.2 Environment Setup

```bash
#!/bin/bash
# scripts/generate-keys.sh

# Generate encryption key
ENCRYPTION_KEY=$(openssl rand -hex 32)
echo "ENCRYPTION_KEY=$ENCRYPTION_KEY"

# Generate JWT secret
JWT_SECRET=$(openssl rand -hex 64)
echo "JWT_SECRET=$JWT_SECRET"

# Store in Vault
vault kv put secret/myapp/encryption key=$ENCRYPTION_KEY
vault kv put secret/myapp/jwt secret=$JWT_SECRET

echo "Keys generated and stored in Vault"
```

```javascript
// config/security.js
const fs = require('fs');

function loadSecurityConfig() {
  return {
    // Database TLS
    db: {
      ssl: {
        rejectUnauthorized: true,
        ca: process.env.DB_CA_CERT
          ? Buffer.from(process.env.DB_CA_CERT, 'base64')
          : fs.readFileSync('/etc/ssl/db/ca.crt'),
      }
    },
    
    // Redis TLS
    redis: {
      tls: process.env.REDIS_TLS === 'true' ? {
        rejectUnauthorized: true,
        ca: process.env.REDIS_CA_CERT
          ? Buffer.from(process.env.REDIS_CA_CERT, 'base64')
          : fs.readFileSync('/etc/ssl/redis/ca.crt'),
      } : false,
    },
    
    // Encryption
    encryptionKey: Buffer.from(
      process.env.ENCRYPTION_KEY || (() => { throw new Error('ENCRYPTION_KEY not set'); })(),
      'hex'
    ),
    
    // JWT
    jwtSecret: process.env.JWT_SECRET || (() => { throw new Error('JWT_SECRET not set'); })(),
    jwtExpiry: process.env.JWT_EXPIRY || '1h',
  };
}

module.exports = loadSecurityConfig();
```

### 5.3 Testing Encryption

```javascript
// __tests__/encryption.test.js
const { Encryption } = require('../lib/encryption');
const crypto = require('crypto');

describe('Encryption', () => {
  const key = crypto.randomBytes(32);
  const enc = new Encryption(key);
  
  test('encrypt and decrypt roundtrip', () => {
    const plaintext = 'sensitive data 12345';
    const encrypted = enc.encrypt(plaintext);
    const decrypted = enc.decrypt(encrypted);
    
    expect(decrypted).toBe(plaintext);
    expect(encrypted).not.toBe(plaintext);
  });
  
  test('different IVs for same plaintext', () => {
    const plaintext = 'same data';
    const encrypted1 = enc.encrypt(plaintext);
    const encrypted2 = enc.encrypt(plaintext);
    
    // Same plaintext → different ciphertext (due to random IV)
    expect(encrypted1).not.toBe(encrypted2);
    
    // Both decrypt correctly
    expect(enc.decrypt(encrypted1)).toBe(plaintext);
    expect(enc.decrypt(encrypted2)).toBe(plaintext);
  });
  
  test('tampered ciphertext fails', () => {
    const plaintext = 'sensitive';
    const encrypted = enc.encrypt(plaintext);
    const tampered = encrypted.slice(0, -10) + 'aaaaaaaaaa';
    
    expect(() => enc.decrypt(tampered)).toThrow();
  });
  
  test('wrong key fails decryption', () => {
    const plaintext = 'sensitive';
    const encrypted = enc.encrypt(plaintext);
    
    const wrongKey = crypto.randomBytes(32);
    const wrongEnc = new Encryption(wrongKey);
    
    expect(() => wrongEnc.decrypt(encrypted)).toThrow();
  });
});
```

---

## 6. สรุป

### Layer of Defense

```
┌─────────────────────────────────────────────┐
│ Application Layer                           │
│ - Column-level encryption (pgcrypto)        │
│ - Application encryption (Node.js crypto)   │
│ - Key management (Vault/KMS)                │
├─────────────────────────────────────────────┤
│ Transport Layer                             │
│ - TLS 1.3 for all connections               │
│ - Certificate verification                  │
│ - mTLS for service-to-service               │
├─────────────────────────────────────────────┤
│ Database Layer                              │
│ - RDS encryption at rest                    │
│ - PostgreSQL SSL                            │
│ - Redis TLS                                 │
├─────────────────────────────────────────────┤
│ Infrastructure Layer                        │
│ - LUKS disk encryption                      │
│ - VPC/network isolation                     │
│ - Security groups                           │
├─────────────────────────────────────────────┤
│ Key Management Layer                        │
│ - HashiCorp Vault                           │
│ - AWS KMS                                   │
│ - Automatic key rotation                    │
└─────────────────────────────────────────────┘
```

ทุก layer มีความสำคัญ - **defense in depth** หมายความว่าถ้า layer หนึ่งถูก compromise layer อื่นยังคงป้องกันได้

---

*เนื้อหานี้เป็นส่วนหนึ่งของ Database Cluster Course - World-Class Level*
*Part 92/100: Encryption at Rest and in Transit*
