# Part 93: Compliance - GDPR และ PDPA

## บทนำ

ในยุคดิจิทัล การเก็บและประมวลผลข้อมูลส่วนบุคคลต้องปฏิบัติตามกฎหมาย การไม่ปฏิบัติตามอาจนำมาซึ่งค่าปรับสูงและความเสียหายต่อชื่อเสียง บทนี้จะครอบคลุม GDPR (สหภาพยุโรป) และ PDPA (ไทย) พร้อม technical implementations

---

## 1. GDPR: General Data Protection Regulation

### 1.1 ภาพรวม GDPR

GDPR เป็นกฎหมายของสหภาพยุโรปที่มีผลบังคับใช้ตั้งแต่ 25 พฤษภาคม 2018 บังคับใช้กับ:
- องค์กรที่ตั้งอยู่ใน EU/EEA
- องค์กรนอก EU/EEA ที่ประมวลผลข้อมูลของผู้อยู่ใน EU

**ค่าปรับ:**
- สูงสุด €20 ล้าน หรือ 4% ของ global revenue (เลือกแบบที่สูงกว่า)

### 1.2 หลักการสำคัญ (Principles)

```markdown
1. Lawfulness, Fairness, Transparency
   - มีฐานทางกฎหมาย (Legal Basis) ในการประมวลผล
   - ต้องโปร่งใสกับ data subjects

2. Purpose Limitation
   - เก็บข้อมูลเพื่อวัตถุประสงค์เฉพาะที่แจ้งไว้
   - ไม่นำไปใช้เพื่อวัตถุประสงค์อื่น

3. Data Minimization
   - เก็บเฉพาะข้อมูลที่จำเป็น
   - ห้ามเก็บข้อมูลที่ไม่เกี่ยวข้อง

4. Accuracy
   - ข้อมูลต้องถูกต้องและ up-to-date
   - มีกระบวนการแก้ไขข้อมูลที่ผิด

5. Storage Limitation
   - เก็บข้อมูลไม่นานกว่าที่จำเป็น
   - มี retention policy ที่ชัดเจน

6. Integrity and Confidentiality
   - ปกป้องข้อมูลจากการเข้าถึงโดยไม่ได้รับอนุญาต
   - ใช้ encryption และ access controls

7. Accountability
   - สามารถพิสูจน์ได้ว่าปฏิบัติตาม GDPR
   - Documentation, policies, training
```

### 1.3 Legal Basis (ฐานทางกฎหมาย)

```markdown
1. Consent: ได้รับความยินยอม
2. Contract: จำเป็นสำหรับสัญญา
3. Legal Obligation: ข้อผูกพันตามกฎหมาย
4. Vital Interests: เพื่อชีวิตและความปลอดภัย
5. Public Task: งานสาธารณะประโยชน์
6. Legitimate Interests: ประโยชน์โดยชอบธรรม
```

### 1.4 สิทธิของ Data Subjects

```markdown
1. Right of Access: สิทธิในการเข้าถึงข้อมูล
2. Right to Rectification: สิทธิในการแก้ไขข้อมูล
3. Right to Erasure (Right to be Forgotten): สิทธิในการลบข้อมูล
4. Right to Restriction: สิทธิในการจำกัดการประมวลผล
5. Right to Data Portability: สิทธิในการนำข้อมูลไป
6. Right to Object: สิทธิในการคัดค้าน
```

---

## 2. PDPA: Personal Data Protection Act (ไทย)

### 2.1 ภาพรวม PDPA

PDPA (พ.ร.บ. คุ้มครองข้อมูลส่วนบุคคล) มีผลบังคับใช้เต็มรูปแบบ 1 มิถุนายน 2565 สร้างแบบอ้างอิง GDPR แต่มีบางข้อแตกต่าง

**บังคับใช้กับ:**
- นิติบุคคลหรือบุคคลธรรมดาที่เก็บ ใช้ เผยแพร่ข้อมูลส่วนบุคคล
- ผู้ประกอบกิจการในไทย หรือเสนอสินค้า/บริการแก่คนในไทย

**โทษ:**
- ทางแพ่ง: ค่าสินไหมทดแทน + ค่าเสียหายเพิ่มขึ้น 2 เท่า
- ทางปกครอง: ปรับไม่เกิน 5 ล้านบาท
- ทางอาญา: จำคุกไม่เกิน 1 ปี และ/หรือปรับไม่เกิน 1 ล้านบาท

### 2.2 ข้อมูลส่วนบุคคลตาม PDPA

```markdown
ข้อมูลส่วนบุคคล (Personal Data):
- ชื่อ-นามสกุล
- เลขบัตรประชาชน / เลขหนังสือเดินทาง
- ที่อยู่, อีเมล, เบอร์โทรศัพท์
- วันเดือนปีเกิด, อายุ, เพศ
- IP Address, Cookie ID, Device ID
- ข้อมูลตำแหน่งที่ตั้ง (Location)

ข้อมูลส่วนบุคคลอ่อนไหว (Sensitive Personal Data):
- เชื้อชาติ, ชาติพันธุ์
- ความคิดเห็นทางการเมือง
- ความเชื่อ ศาสนา หรือปรัชญา
- พฤติกรรมทางเพศ
- ประวัติอาชญากรรม
- ข้อมูลสุขภาพ, ความพิการ
- ข้อมูลสหภาพแรงงาน
- ข้อมูลพันธุกรรม
- ข้อมูลชีวมิติ (Biometric)
```

---

## 3. Personal Data Categories ใน Database

### 3.1 Data Classification

```sql
-- สร้าง catalog ของ personal data
CREATE TABLE compliance.data_catalog (
    id SERIAL PRIMARY KEY,
    schema_name TEXT NOT NULL,
    table_name TEXT NOT NULL,
    column_name TEXT NOT NULL,
    data_category TEXT NOT NULL, -- PII, Sensitive, Financial, etc.
    data_type TEXT,              -- name, email, phone, etc.
    is_encrypted BOOLEAN DEFAULT false,
    retention_days INTEGER,
    legal_basis TEXT,
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ตัวอย่างการ catalog ข้อมูล
INSERT INTO compliance.data_catalog VALUES
    (DEFAULT, 'app', 'users', 'email', 'PII', 'email', true, 730, 'contract', NULL, DEFAULT),
    (DEFAULT, 'app', 'users', 'phone', 'PII', 'phone', true, 730, 'consent', NULL, DEFAULT),
    (DEFAULT, 'app', 'users', 'full_name', 'PII', 'name', false, 730, 'contract', NULL, DEFAULT),
    (DEFAULT, 'app', 'users', 'id_card_number', 'PII', 'national_id', true, 365, 'legal_obligation', 'KYC', DEFAULT),
    (DEFAULT, 'app', 'health_records', 'diagnosis', 'Sensitive', 'health', true, 3650, 'consent', NULL, DEFAULT),
    (DEFAULT, 'app', 'payments', 'card_number', 'Financial', 'credit_card', true, 90, 'contract', 'PCI-DSS', DEFAULT);
```

---

## 4. Technical Implementations

### 4.1 Consent Management

```sql
-- Consent records
CREATE TABLE compliance.consents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    purpose TEXT NOT NULL,  -- marketing, analytics, necessary, etc.
    legal_basis TEXT NOT NULL,  -- consent, contract, legitimate_interest
    version TEXT NOT NULL,  -- version ของ privacy policy
    given_at TIMESTAMPTZ,
    withdrawn_at TIMESTAMPTZ,
    ip_address INET,
    user_agent TEXT,
    is_active BOOLEAN GENERATED ALWAYS AS (
        given_at IS NOT NULL AND withdrawn_at IS NULL
    ) STORED
);

CREATE INDEX idx_consents_user_purpose
    ON compliance.consents(user_id, purpose)
    WHERE is_active = true;

-- Functions
CREATE OR REPLACE FUNCTION compliance.give_consent(
    p_user_id UUID,
    p_purpose TEXT,
    p_legal_basis TEXT,
    p_version TEXT,
    p_ip INET DEFAULT NULL,
    p_user_agent TEXT DEFAULT NULL
) RETURNS UUID AS $$
DECLARE
    v_consent_id UUID;
BEGIN
    -- Withdraw existing active consent for same purpose
    UPDATE compliance.consents
    SET withdrawn_at = NOW()
    WHERE user_id = p_user_id
    AND purpose = p_purpose
    AND withdrawn_at IS NULL;
    
    -- Insert new consent
    INSERT INTO compliance.consents (
        user_id, purpose, legal_basis, version,
        given_at, ip_address, user_agent
    ) VALUES (
        p_user_id, p_purpose, p_legal_basis, p_version,
        NOW(), p_ip, p_user_agent
    ) RETURNING id INTO v_consent_id;
    
    RETURN v_consent_id;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION compliance.withdraw_consent(
    p_user_id UUID,
    p_purpose TEXT
) RETURNS BOOLEAN AS $$
BEGIN
    UPDATE compliance.consents
    SET withdrawn_at = NOW()
    WHERE user_id = p_user_id
    AND purpose = p_purpose
    AND withdrawn_at IS NULL;
    
    RETURN FOUND;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION compliance.has_consent(
    p_user_id UUID,
    p_purpose TEXT
) RETURNS BOOLEAN AS $$
BEGIN
    RETURN EXISTS (
        SELECT 1 FROM compliance.consents
        WHERE user_id = p_user_id
        AND purpose = p_purpose
        AND is_active = true
    );
END;
$$ LANGUAGE plpgsql STABLE;
```

```javascript
// services/consent.service.js
class ConsentService {
  async giveConsent(userId, purposes, meta) {
    const { ipAddress, userAgent, policyVersion } = meta;
    const consentIds = [];
    
    for (const purpose of purposes) {
      const result = await pool.query(
        `SELECT compliance.give_consent($1, $2, $3, $4, $5, $6) AS consent_id`,
        [userId, purpose, 'consent', policyVersion, ipAddress, userAgent]
      );
      consentIds.push(result.rows[0].consent_id);
    }
    
    return consentIds;
  }
  
  async withdrawConsent(userId, purpose) {
    const result = await pool.query(
      `SELECT compliance.withdraw_consent($1, $2) AS success`,
      [userId, purpose]
    );
    return result.rows[0].success;
  }
  
  async getUserConsents(userId) {
    const result = await pool.query(`
      SELECT 
        purpose,
        legal_basis,
        version,
        given_at,
        withdrawn_at,
        is_active
      FROM compliance.consents
      WHERE user_id = $1
      ORDER BY purpose, given_at DESC
    `, [userId]);
    
    return result.rows;
  }
  
  async checkConsent(userId, purpose) {
    const result = await pool.query(
      `SELECT compliance.has_consent($1, $2) AS has_consent`,
      [userId, purpose]
    );
    return result.rows[0].has_consent;
  }
}
```

### 4.2 Right to Access: Export User Data

```sql
-- Function: export all personal data for a user
CREATE OR REPLACE FUNCTION compliance.export_user_data(p_user_id UUID)
RETURNS JSONB AS $$
DECLARE
    v_result JSONB := '{}';
    v_profile JSONB;
    v_orders JSONB;
    v_consents JSONB;
    v_activities JSONB;
BEGIN
    -- Profile data
    SELECT jsonb_build_object(
        'id', id,
        'email', email,
        'full_name', full_name,
        'created_at', created_at,
        'last_login', last_login
    ) INTO v_profile
    FROM app.users
    WHERE id = p_user_id;
    
    -- Orders
    SELECT jsonb_agg(jsonb_build_object(
        'id', id,
        'amount', amount,
        'status', status,
        'created_at', created_at
    )) INTO v_orders
    FROM app.orders
    WHERE user_id = p_user_id;
    
    -- Consents
    SELECT jsonb_agg(jsonb_build_object(
        'purpose', purpose,
        'given_at', given_at,
        'withdrawn_at', withdrawn_at,
        'is_active', is_active
    )) INTO v_consents
    FROM compliance.consents
    WHERE user_id = p_user_id;
    
    -- Activities
    SELECT jsonb_agg(jsonb_build_object(
        'action', action,
        'resource', resource,
        'created_at', created_at
    )) INTO v_activities
    FROM audit.user_activities
    WHERE user_id = p_user_id
    ORDER BY created_at DESC
    LIMIT 1000;
    
    v_result := jsonb_build_object(
        'export_date', NOW(),
        'user_id', p_user_id,
        'profile', COALESCE(v_profile, 'null'::jsonb),
        'orders', COALESCE(v_orders, '[]'::jsonb),
        'consents', COALESCE(v_consents, '[]'::jsonb),
        'activities', COALESCE(v_activities, '[]'::jsonb)
    );
    
    -- Log the access request
    INSERT INTO compliance.data_access_requests (
        user_id, request_type, status, completed_at
    ) VALUES (p_user_id, 'access', 'completed', NOW());
    
    RETURN v_result;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

```javascript
// controllers/gdpr.controller.js
const { pool } = require('../db');
const json2csv = require('json2csv');

class GDPRController {
  // GET /api/gdpr/export
  async exportUserData(req, res) {
    const userId = req.user.id;
    
    try {
      // Log request
      await pool.query(`
        INSERT INTO compliance.data_access_requests 
        (user_id, request_type, status, requested_at)
        VALUES ($1, 'access', 'processing', NOW())
      `, [userId]);
      
      // Get all user data
      const result = await pool.query(
        'SELECT compliance.export_user_data($1) AS data',
        [userId]
      );
      
      const userData = result.rows[0].data;
      
      // Return as JSON (GDPR requires machine-readable format)
      res.setHeader('Content-Type', 'application/json');
      res.setHeader(
        'Content-Disposition',
        `attachment; filename="my-data-${new Date().toISOString().split('T')[0]}.json"`
      );
      res.json(userData);
      
    } catch (err) {
      console.error('Error exporting user data:', err);
      res.status(500).json({ error: 'Failed to export data' });
    }
  }
  
  // DELETE /api/gdpr/delete
  async deleteUserData(req, res) {
    const userId = req.user.id;
    const { reason, password } = req.body;
    
    // Verify password before deletion
    const valid = await authService.verifyPassword(userId, password);
    if (!valid) {
      return res.status(401).json({ error: 'Invalid password' });
    }
    
    try {
      await pool.query('BEGIN');
      
      // Soft delete + anonymize
      await pool.query(
        'SELECT compliance.anonymize_user($1, $2)',
        [userId, reason || 'user_request']
      );
      
      await pool.query('COMMIT');
      
      // Revoke JWT tokens
      await tokenService.revokeAllUserTokens(userId);
      
      res.json({ message: 'Your data has been deleted' });
      
    } catch (err) {
      await pool.query('ROLLBACK');
      console.error('Error deleting user data:', err);
      res.status(500).json({ error: 'Failed to delete data' });
    }
  }
}
```

### 4.3 Right to Erasure (Right to be Forgotten)

```sql
-- Data Access Requests tracking
CREATE TABLE compliance.data_access_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    request_type TEXT NOT NULL, -- access, erasure, portability, restriction
    status TEXT NOT NULL DEFAULT 'pending', -- pending, processing, completed, rejected
    reason TEXT,
    requested_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    notes TEXT
);

-- Anonymization function
CREATE OR REPLACE FUNCTION compliance.anonymize_user(
    p_user_id UUID,
    p_reason TEXT DEFAULT 'user_request'
) RETURNS VOID AS $$
DECLARE
    v_anon_email TEXT;
    v_anon_id UUID;
BEGIN
    v_anon_id := gen_random_uuid();
    v_anon_email := 'deleted_' || encode(gen_random_bytes(8), 'hex') || '@deleted.invalid';
    
    -- Anonymize user record (ไม่ลบ - เก็บ audit trail)
    UPDATE app.users SET
        email = v_anon_email,
        full_name = 'Deleted User',
        phone_encrypted = NULL,
        phone_hash = NULL,
        id_card_encrypted = NULL,
        id_card_hash = NULL,
        password_hash = encode(gen_random_bytes(60), 'hex'), -- random unusable hash
        is_active = false,
        deleted_at = NOW(),
        deletion_reason = p_reason,
        anonymized = true
    WHERE id = p_user_id;
    
    -- Remove from marketing lists
    DELETE FROM marketing.email_subscriptions WHERE user_id = p_user_id;
    DELETE FROM marketing.push_subscriptions WHERE user_id = p_user_id;
    
    -- Anonymize order history (เก็บ business records แต่ remove PII)
    UPDATE app.orders SET
        shipping_name = 'Deleted User',
        shipping_address = NULL,
        billing_name = 'Deleted User',
        billing_address = NULL
    WHERE user_id = p_user_id;
    
    -- Revoke active sessions
    DELETE FROM app.user_sessions WHERE user_id = p_user_id;
    
    -- Log the deletion
    INSERT INTO compliance.data_access_requests (
        user_id, request_type, status, reason, completed_at
    ) VALUES (p_user_id, 'erasure', 'completed', p_reason, NOW());
    
    -- Add to deletion log (for audit, anonymous reference)
    INSERT INTO compliance.deletion_log (
        original_user_id, anonymized_email, deleted_at, reason
    ) VALUES (p_user_id, v_anon_email, NOW(), p_reason);
    
END;
$$ LANGUAGE plpgsql;

-- Deletion log (เก็บ audit trail โดยไม่มี PII)
CREATE TABLE compliance.deletion_log (
    id SERIAL PRIMARY KEY,
    original_user_id UUID NOT NULL, -- เก็บไว้สำหรับ internal audit
    anonymized_email TEXT,
    deleted_at TIMESTAMPTZ DEFAULT NOW(),
    reason TEXT,
    -- RLS: เฉพาะ compliance team เท่านั้น
    CONSTRAINT deletion_log_reason_check
        CHECK (reason IN ('user_request', 'admin_action', 'retention_policy', 'legal_hold'))
);
```

### 4.4 Hard Delete vs Soft Delete vs Anonymization

```sql
-- Option 1: Hard Delete (อาจไม่สอดคล้อง business requirement)
CREATE OR REPLACE FUNCTION compliance.hard_delete_user(p_user_id UUID)
RETURNS VOID AS $$
BEGIN
    -- ต้องลบ dependent records ก่อน
    DELETE FROM app.user_sessions WHERE user_id = p_user_id;
    DELETE FROM app.notifications WHERE user_id = p_user_id;
    DELETE FROM compliance.consents WHERE user_id = p_user_id;
    
    -- ลบ user (cascade จะลบ records อื่น)
    DELETE FROM app.users WHERE id = p_user_id;
    
    -- Log (ไม่มี user_id แล้ว - เก็บ timestamp เท่านั้น)
    INSERT INTO compliance.deletion_log (original_user_id, reason, deleted_at)
    VALUES (p_user_id, 'hard_delete', NOW());
END;
$$ LANGUAGE plpgsql;

-- Option 2: Soft Delete (เก็บ records แต่ mark ว่า deleted)
ALTER TABLE app.users ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMPTZ;
ALTER TABLE app.users ADD COLUMN IF NOT EXISTS deletion_reason TEXT;
ALTER TABLE app.users ADD COLUMN IF NOT EXISTS anonymized BOOLEAN DEFAULT false;

-- Option 3: Anonymization (recommended สำหรับ GDPR)
-- ดู function anonymize_user ด้านบน
-- - เก็บ record structure ไว้ (สำหรับ business logic)
-- - แทนที่ PII ด้วยข้อมูล fake/generic
-- - ยังคง integrity ของ related records
```

### 4.5 Data Portability: Export ในรูปแบบ Machine-readable

```javascript
// services/data-portability.service.js
const { pool } = require('../db');
const { Parser } = require('json2csv');
const archiver = require('archiver');
const { PassThrough } = require('stream');

class DataPortabilityService {
  async generateExport(userId, format = 'json') {
    const data = await this.collectUserData(userId);
    
    switch (format) {
      case 'json':
        return this.exportAsJSON(data);
      case 'csv':
        return this.exportAsCSV(data);
      default:
        throw new Error(`Unsupported format: ${format}`);
    }
  }
  
  async collectUserData(userId) {
    const [profile, orders, consents, addresses] = await Promise.all([
      this.getProfile(userId),
      this.getOrders(userId),
      this.getConsents(userId),
      this.getAddresses(userId),
    ]);
    
    return { profile, orders, consents, addresses };
  }
  
  exportAsJSON(data) {
    return {
      contentType: 'application/json',
      data: JSON.stringify(data, null, 2),
    };
  }
  
  exportAsCSV(data) {
    const archive = archiver('zip');
    const stream = new PassThrough();
    archive.pipe(stream);
    
    // Profile CSV
    const profileParser = new Parser({ fields: Object.keys(data.profile) });
    archive.append(profileParser.parse([data.profile]), { name: 'profile.csv' });
    
    // Orders CSV
    if (data.orders.length > 0) {
      const ordersParser = new Parser({ fields: Object.keys(data.orders[0]) });
      archive.append(ordersParser.parse(data.orders), { name: 'orders.csv' });
    }
    
    archive.finalize();
    
    return {
      contentType: 'application/zip',
      stream,
    };
  }
  
  async getProfile(userId) {
    const result = await pool.query(`
      SELECT 
        id,
        email,
        full_name,
        created_at,
        last_login
      FROM app.users WHERE id = $1
    `, [userId]);
    return result.rows[0];
  }
  
  async getOrders(userId) {
    const result = await pool.query(`
      SELECT
        id,
        total_amount,
        status,
        created_at
      FROM app.orders WHERE user_id = $1
      ORDER BY created_at DESC
    `, [userId]);
    return result.rows;
  }
  
  async getConsents(userId) {
    const result = await pool.query(`
      SELECT purpose, legal_basis, given_at, withdrawn_at, is_active
      FROM compliance.consents WHERE user_id = $1
      ORDER BY given_at DESC
    `, [userId]);
    return result.rows;
  }
  
  async getAddresses(userId) {
    const result = await pool.query(`
      SELECT type, street, city, state, country, postal_code
      FROM app.addresses WHERE user_id = $1
    `, [userId]);
    return result.rows;
  }
}
```

### 4.6 Data Minimization

```sql
-- เก็บเฉพาะ fields ที่จำเป็น
-- ❌ ไม่ดี: เก็บทุกอย่าง
CREATE TABLE user_registrations_bad (
    id UUID PRIMARY KEY,
    email TEXT,
    full_name TEXT,
    phone TEXT,
    date_of_birth DATE,
    gender TEXT,
    nationality TEXT,
    id_card_number TEXT,
    passport_number TEXT,
    address TEXT,
    city TEXT,
    country TEXT,
    ip_address INET,
    browser TEXT,
    os TEXT,
    device_fingerprint TEXT,
    referrer TEXT,
    utm_source TEXT,
    utm_campaign TEXT,
    -- ... 50 more columns
);

-- ✅ ดี: เก็บเฉพาะที่จำเป็น
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email TEXT NOT NULL UNIQUE,  -- required for login
    full_name TEXT NOT NULL,     -- required for service
    -- เพิ่มเฉพาะ field ที่มี documented purpose
);

-- Phone เฉพาะถ้า user ให้ consent และใช้ 2FA
CREATE TABLE user_phone_verifications (
    user_id UUID PRIMARY KEY REFERENCES users(id),
    phone_encrypted BYTEA NOT NULL,
    phone_hash TEXT NOT NULL,
    verified_at TIMESTAMPTZ,
    -- ลบอัตโนมัติถ้าไม่ได้ใช้
    expires_at TIMESTAMPTZ DEFAULT NOW() + INTERVAL '1 year'
);
```

### 4.7 Data Retention: ลบข้อมูลหลังหมดอายุ

```sql
-- Retention policies
CREATE TABLE compliance.retention_policies (
    id SERIAL PRIMARY KEY,
    data_type TEXT NOT NULL UNIQUE,
    retention_days INTEGER NOT NULL,
    action TEXT NOT NULL DEFAULT 'anonymize', -- delete, anonymize, archive
    legal_basis TEXT,
    notes TEXT
);

INSERT INTO compliance.retention_policies VALUES
    (DEFAULT, 'user_account', 730, 'anonymize', 'contract', 'Account inactive for 2 years'),
    (DEFAULT, 'user_session', 30, 'delete', 'legitimate_interest', 'Security sessions'),
    (DEFAULT, 'audit_log', 2555, 'archive', 'legal_obligation', '7 years for financial audit'),
    (DEFAULT, 'marketing_email', 365, 'delete', 'consent', '1 year after consent withdrawal'),
    (DEFAULT, 'payment_data', 2555, 'archive', 'legal_obligation', '7 years for financial records'),
    (DEFAULT, 'analytics_raw', 90, 'delete', 'legitimate_interest', 'Raw logs');

-- Retention job
CREATE OR REPLACE FUNCTION compliance.apply_retention_policies()
RETURNS TABLE(policy TEXT, records_affected BIGINT) AS $$
DECLARE
    v_cutoff TIMESTAMPTZ;
    v_count BIGINT;
BEGIN
    -- User accounts: anonymize inactive users
    SELECT retention_days INTO v_cutoff
    FROM compliance.retention_policies
    WHERE data_type = 'user_account';
    
    v_cutoff := NOW() - (v_cutoff::TEXT || ' days')::INTERVAL;
    
    WITH anonymized AS (
        SELECT id FROM app.users
        WHERE last_login < v_cutoff
        AND is_active = false
        AND anonymized = false
        AND deleted_at IS NULL
    )
    SELECT count(*) INTO v_count FROM anonymized;
    
    -- Perform anonymization
    UPDATE app.users SET
        email = 'deleted_' || encode(gen_random_bytes(8), 'hex') || '@deleted.invalid',
        full_name = 'Deleted User',
        phone_encrypted = NULL,
        anonymized = true
    WHERE last_login < v_cutoff
    AND is_active = false
    AND anonymized = false;
    
    RETURN QUERY SELECT 'user_account'::TEXT, v_count;
    
    -- Sessions: delete old sessions
    DELETE FROM app.user_sessions
    WHERE created_at < NOW() - INTERVAL '30 days';
    
    GET DIAGNOSTICS v_count = ROW_COUNT;
    RETURN QUERY SELECT 'user_session'::TEXT, v_count;
    
    -- Marketing emails: delete withdrawn consents
    DELETE FROM marketing.email_subscriptions
    WHERE user_id IN (
        SELECT user_id FROM compliance.consents
        WHERE purpose = 'marketing_email'
        AND withdrawn_at < NOW() - INTERVAL '365 days'
    );
    
    GET DIAGNOSTICS v_count = ROW_COUNT;
    RETURN QUERY SELECT 'marketing_email'::TEXT, v_count;
    
END;
$$ LANGUAGE plpgsql;
```

```javascript
// jobs/retention-cleanup.job.js
const { Pool } = require('pg');
const { createLogger } = require('../lib/logger');

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const logger = createLogger('retention-cleanup');

async function runRetentionCleanup() {
  logger.info('Starting retention policy cleanup...');
  
  const client = await pool.connect();
  
  try {
    const result = await client.query(`
      SELECT * FROM compliance.apply_retention_policies()
    `);
    
    for (const row of result.rows) {
      logger.info(`Retention policy applied`, {
        policy: row.policy,
        records_affected: row.records_affected,
      });
    }
    
    logger.info('Retention cleanup completed');
    
  } catch (err) {
    logger.error('Retention cleanup failed', { error: err.message });
    throw err;
  } finally {
    client.release();
  }
}

// Schedule: รันทุกวันเวลา 02:00
// cron: '0 2 * * *'
module.exports = { runRetentionCleanup };
```

### 4.8 Pseudonymization และ Anonymization

```sql
-- Pseudonymization: แทน real ID ด้วย pseudo ID
-- ยังคง re-identify ได้ถ้ามี mapping table

CREATE TABLE compliance.pseudonym_mapping (
    real_id UUID NOT NULL,
    pseudo_id UUID NOT NULL DEFAULT gen_random_uuid(),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(real_id),
    UNIQUE(pseudo_id)
);

-- Analytics view ที่ใช้ pseudonymized data
CREATE VIEW analytics.user_behaviors AS
SELECT
    m.pseudo_id AS user_id,  -- ใช้ pseudo_id แทน real user_id
    ua.action,
    ua.resource,
    ua.created_at
FROM audit.user_activities ua
JOIN compliance.pseudonym_mapping m ON m.real_id = ua.user_id;

-- Anonymization: ลบ identifiers ทั้งหมด
-- ไม่สามารถ re-identify ได้

CREATE OR REPLACE FUNCTION analytics.anonymize_for_research(
    p_start_date DATE,
    p_end_date DATE
) RETURNS TABLE(
    cohort TEXT,
    action_count BIGINT,
    unique_sessions BIGINT
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        to_char(created_at, 'YYYY-MM') AS cohort,
        COUNT(*) AS action_count,
        COUNT(DISTINCT session_id) AS unique_sessions
        -- ไม่รวม user_id หรือ personal data
    FROM audit.user_activities
    WHERE created_at BETWEEN p_start_date AND p_end_date
    GROUP BY to_char(created_at, 'YYYY-MM');
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

## 5. PostgreSQL Audit Log

### 5.1 pg_audit Extension

```bash
# ติดตั้ง pg_audit
apt-get install postgresql-16-pgaudit

# เพิ่มใน postgresql.conf
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'read,write,ddl,role'
pgaudit.log_catalog = off
pgaudit.log_parameter = on
pgaudit.log_relation = on
pgaudit.log_statement_once = off
```

```sql
-- เปิด audit สำหรับ specific tables
CREATE EXTENSION pgaudit;

-- Audit specific objects
SELECT pgaudit.audit_table('app.users');
SELECT pgaudit.audit_table('app.payments');
SELECT pgaudit.audit_table('compliance.consents');

-- Session-level audit
SET pgaudit.log = 'read';  -- Log SELECT queries
```

### 5.2 Custom Audit Log

```sql
-- Audit log table
CREATE TABLE audit.data_access_log (
    id BIGSERIAL PRIMARY KEY,
    event_time TIMESTAMPTZ DEFAULT NOW(),
    user_id UUID,
    db_user TEXT DEFAULT current_user,
    ip_address INET,
    operation TEXT NOT NULL,
    table_name TEXT NOT NULL,
    record_id TEXT,
    old_values JSONB,
    new_values JSONB,
    query TEXT,
    success BOOLEAN DEFAULT true
) PARTITION BY RANGE (event_time);

-- Monthly partitions
CREATE TABLE audit.data_access_log_2024_01
    PARTITION OF audit.data_access_log
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE audit.data_access_log_2024_02
    PARTITION OF audit.data_access_log
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Trigger function
CREATE OR REPLACE FUNCTION audit.log_data_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit.data_access_log (
        user_id,
        ip_address,
        operation,
        table_name,
        record_id,
        old_values,
        new_values
    ) VALUES (
        current_setting('app.user_id', true)::UUID,
        inet_client_addr(),
        TG_OP,
        TG_TABLE_SCHEMA || '.' || TG_TABLE_NAME,
        CASE
            WHEN TG_OP = 'DELETE' THEN OLD.id::TEXT
            ELSE NEW.id::TEXT
        END,
        CASE WHEN TG_OP IN ('UPDATE', 'DELETE') THEN to_jsonb(OLD) ELSE NULL END,
        CASE WHEN TG_OP IN ('UPDATE', 'INSERT') THEN to_jsonb(NEW) ELSE NULL END
    );
    
    IF TG_OP = 'DELETE' THEN RETURN OLD; END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Attach to sensitive tables
CREATE TRIGGER audit_users
    AFTER INSERT OR UPDATE OR DELETE ON app.users
    FOR EACH ROW EXECUTE FUNCTION audit.log_data_changes();
```

---

## 6. Redis: TTL สำหรับ Personal Data

```javascript
// lib/session-store.js - Redis session ที่ GDPR-compliant
const Redis = require('ioredis');

const redis = new Redis(process.env.REDIS_URL);

class GDPRCompliantSessionStore {
  // Session TTL: 24 ชั่วโมง (ตาม retention policy)
  static SESSION_TTL = 24 * 60 * 60;
  
  // Personal data TTL: สั้นกว่า
  static PERSONAL_DATA_TTL = 60 * 60; // 1 hour
  
  async createSession(userId, userData) {
    const sessionId = require('crypto').randomBytes(32).toString('hex');
    
    // เก็บ minimal data เท่านั้น
    const sessionData = {
      userId: userData.userId,
      tenantId: userData.tenantId,
      role: userData.role,
      // ไม่เก็บ email, name, phone ใน session
    };
    
    await redis.setex(
      `session:${sessionId}`,
      GDPRCompliantSessionStore.SESSION_TTL,
      JSON.stringify(sessionData)
    );
    
    // Track sessions สำหรับ user (สำหรับ revoke all)
    await redis.sadd(`user_sessions:${userId}`, sessionId);
    await redis.expire(`user_sessions:${userId}`, GDPRCompliantSessionStore.SESSION_TTL);
    
    return sessionId;
  }
  
  async deleteUserSessions(userId) {
    const sessionIds = await redis.smembers(`user_sessions:${userId}`);
    
    if (sessionIds.length > 0) {
      const pipeline = redis.pipeline();
      for (const sessionId of sessionIds) {
        pipeline.del(`session:${sessionId}`);
      }
      pipeline.del(`user_sessions:${userId}`);
      await pipeline.exec();
    }
    
    return sessionIds.length;
  }
  
  // Cache personal data ด้วย TTL สั้น
  async cacheUserProfile(userId, profile) {
    await redis.setex(
      `profile:${userId}`,
      GDPRCompliantSessionStore.PERSONAL_DATA_TTL,
      JSON.stringify(profile)
    );
  }
  
  // ลบ cache ทันทีเมื่อ user ขอลบข้อมูล
  async clearUserCache(userId) {
    const keys = await redis.keys(`*:${userId}`);
    if (keys.length > 0) {
      await redis.del(...keys);
    }
  }
}
```

---

## 7. Breach Notification

### 7.1 GDPR: 72 Hours Rule

```markdown
ถ้าเกิด Data Breach ต้องแจ้ง:
1. DPA (Data Protection Authority) ภายใน 72 ชั่วโมง
2. Data Subjects (ถ้า risk สูง) โดยเร็วที่สุด

ข้อมูลที่ต้องแจ้ง:
- ลักษณะของ breach
- ข้อมูลที่ได้รับผลกระทบ
- จำนวนผู้ได้รับผลกระทบ
- ผลกระทบที่เป็นไปได้
- มาตรการที่ดำเนินการแล้ว
- ชื่อและข้อมูลติดต่อ DPO
```

```javascript
// services/breach-notification.service.js
class BreachNotificationService {
  async reportBreach(breachData) {
    const {
      type,           // unauthorized_access, ransomware, accidental_disclosure
      discoveredAt,
      affectedRecords,
      dataTypes,      // ['email', 'phone', 'id_card']
      riskLevel,      // low, medium, high, critical
      description,
    } = breachData;
    
    // Log breach
    const breachId = await this.logBreach(breachData);
    
    // Assess risk
    const requiresNotification = riskLevel === 'high' || riskLevel === 'critical';
    const requiresDPANotification = true; // Always for GDPR
    
    // Notify DPO immediately
    await this.notifyDPO(breachId, breachData);
    
    // Check 72-hour deadline
    const deadline = new Date(discoveredAt);
    deadline.setHours(deadline.getHours() + 72);
    
    const hoursRemaining = (deadline - new Date()) / (1000 * 60 * 60);
    
    console.warn(`⚠️ GDPR Breach Notification Required!`);
    console.warn(`Deadline: ${deadline.toISOString()}`);
    console.warn(`Hours remaining: ${hoursRemaining.toFixed(1)}`);
    
    if (requiresDPANotification) {
      // Schedule DPA notification
      await this.scheduleDPANotification(breachId, deadline);
    }
    
    if (requiresNotification && affectedRecords > 0) {
      // Notify affected users
      await this.scheduleUserNotifications(breachId, affectedRecords);
    }
    
    return { breachId, deadline, hoursRemaining };
  }
  
  async logBreach(data) {
    const result = await pool.query(`
      INSERT INTO compliance.data_breaches (
        type, discovered_at, affected_records,
        data_types, risk_level, description, status
      ) VALUES ($1, $2, $3, $4, $5, $6, 'investigating')
      RETURNING id
    `, [
      data.type,
      data.discoveredAt,
      data.affectedRecords,
      data.dataTypes,
      data.riskLevel,
      data.description,
    ]);
    
    return result.rows[0].id;
  }
}
```

---

## 8. Privacy by Design

### 8.1 หลักการ

```markdown
1. Proactive not Reactive: ป้องกันก่อน ไม่ใช่แก้ไขทีหลัง
2. Privacy as Default: privacy เป็น default setting
3. Privacy Embedded: รวม privacy ใน system design
4. Full Functionality: privacy ไม่ทำให้ functionality ลดลง
5. End-to-End Security: ปกป้องตลอด lifecycle
6. Visibility and Transparency: open standards
7. Respect for User Privacy: user-centric
```

### 8.2 Technical Practices

```javascript
// ✅ Privacy by Design: collect minimal data

// Registration form
const registrationSchema = {
  // Required fields (legitimate purpose)
  email: { required: true, purpose: 'login' },
  password: { required: true, purpose: 'authentication' },
  
  // Optional fields (user consent required)
  fullName: { required: false, consentRequired: true, purpose: 'personalization' },
  phone: { required: false, consentRequired: true, purpose: '2fa_optional' },
  
  // Never collect without explicit necessity
  // dateOfBirth: NEVER unless legally required
  // gender: NEVER unless explicit purpose
  // nationality: NEVER unless KYC required
};

// API response: return minimal data
function sanitizeUserForResponse(user) {
  // ไม่ return sensitive fields กลับไปยัง client
  const { 
    password_hash,      // ไม่ return
    phone_encrypted,    // ไม่ return (return แค่ last4)
    id_card_encrypted,  // ไม่ return
    ...safeFields 
  } = user;
  
  return {
    ...safeFields,
    hasPhone: !!user.phone_encrypted, // แค่บอกว่ามีหรือไม่
  };
}
```

---

## 9. DPO Requirements

### 9.1 เมื่อไหร่ต้องมี DPO

```markdown
GDPR กำหนดให้ต้องมี DPO เมื่อ:
1. เป็น public authority หรือ body
2. Core activities ต้องการ systematic monitoring ของ data subjects
3. Core activities เกี่ยวกับ special categories of data ขนาดใหญ่

PDPA: กำหนดให้มี DPO สำหรับ:
- ผู้ควบคุมข้อมูลส่วนบุคคลหรือผู้ประมวลผลที่มีการประมวลผลข้อมูลจำนวนมาก
- กิจกรรมที่ต้องติดตามตรวจสอบข้อมูลส่วนบุคคลอย่างสม่ำเสมอและเป็นระบบในขนาดใหญ่
- ข้อมูลส่วนบุคคลอ่อนไหวในขนาดใหญ่
```

---

## 10. Full Compliance Implementation

### 10.1 Complete Schema

```sql
-- compliance schema
CREATE SCHEMA compliance;

-- Data inventory
CREATE TABLE compliance.data_inventory (
    id SERIAL PRIMARY KEY,
    system_name TEXT NOT NULL,
    data_category TEXT NOT NULL,
    description TEXT,
    legal_basis TEXT NOT NULL,
    retention_period TEXT NOT NULL,
    encryption_required BOOLEAN DEFAULT true,
    cross_border_transfer BOOLEAN DEFAULT false,
    third_party_sharing BOOLEAN DEFAULT false,
    dpia_required BOOLEAN DEFAULT false, -- Data Protection Impact Assessment
    review_date DATE,
    owner TEXT,
    notes TEXT
);

-- Privacy notices sent to users
CREATE TABLE compliance.privacy_notices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version TEXT NOT NULL UNIQUE,
    effective_date DATE NOT NULL,
    content_url TEXT,
    changes_summary TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- User acknowledgments of privacy notices
CREATE TABLE compliance.privacy_notice_acknowledgments (
    user_id UUID NOT NULL REFERENCES app.users(id),
    notice_version TEXT NOT NULL REFERENCES compliance.privacy_notices(version),
    acknowledged_at TIMESTAMPTZ DEFAULT NOW(),
    ip_address INET,
    PRIMARY KEY (user_id, notice_version)
);

-- Data subject requests
CREATE TABLE compliance.dsar_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID,
    email TEXT NOT NULL, -- ใช้ email เพราะ user อาจถูก delete แล้ว
    request_type TEXT NOT NULL CHECK (
        request_type IN ('access', 'rectification', 'erasure', 
                        'restriction', 'portability', 'objection')
    ),
    status TEXT NOT NULL DEFAULT 'received' CHECK (
        status IN ('received', 'verifying_identity', 'processing', 
                  'completed', 'rejected', 'extended')
    ),
    received_at TIMESTAMPTZ DEFAULT NOW(),
    identity_verified_at TIMESTAMPTZ,
    due_date DATE GENERATED ALWAYS AS (
        (received_at + INTERVAL '30 days')::DATE  -- GDPR: 1 month
    ) STORED,
    completed_at TIMESTAMPTZ,
    rejection_reason TEXT,
    notes TEXT,
    processed_by TEXT
);

-- Data breaches
CREATE TABLE compliance.data_breaches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    breach_type TEXT NOT NULL,
    discovered_at TIMESTAMPTZ NOT NULL,
    occurred_at TIMESTAMPTZ,
    affected_records INTEGER,
    data_categories TEXT[],
    risk_level TEXT NOT NULL CHECK (risk_level IN ('low', 'medium', 'high', 'critical')),
    description TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'investigating',
    dpa_notified_at TIMESTAMPTZ,
    users_notified_at TIMESTAMPTZ,
    containment_measures TEXT,
    prevention_measures TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 10.2 Compliance Checklist SQL

```sql
-- View: compliance status dashboard
CREATE VIEW compliance.status_dashboard AS
SELECT
    'Consent Rate' AS metric,
    ROUND(
        100.0 * COUNT(DISTINCT c.user_id) / NULLIF(COUNT(DISTINCT u.id), 0),
        2
    ) AS value,
    '%' AS unit
FROM app.users u
LEFT JOIN compliance.consents c ON c.user_id = u.id AND c.is_active = true

UNION ALL

SELECT
    'Users with Privacy Notice Acknowledgment',
    ROUND(
        100.0 * COUNT(DISTINCT pna.user_id) / NULLIF(COUNT(DISTINCT u.id), 0),
        2
    ),
    '%'
FROM app.users u
LEFT JOIN compliance.privacy_notice_acknowledgments pna ON pna.user_id = u.id
JOIN compliance.privacy_notices pn ON pn.version = pna.notice_version
    AND pn.effective_date = (SELECT MAX(effective_date) FROM compliance.privacy_notices)

UNION ALL

SELECT
    'Pending DSAR Requests',
    COUNT(*)::NUMERIC,
    'requests'
FROM compliance.dsar_requests
WHERE status NOT IN ('completed', 'rejected')

UNION ALL

SELECT
    'DSAR Overdue',
    COUNT(*)::NUMERIC,
    'requests'
FROM compliance.dsar_requests
WHERE status NOT IN ('completed', 'rejected')
AND due_date < CURRENT_DATE;
```

---

## 11. Full Compliance Checklist

```markdown
## GDPR/PDPA Compliance Checklist

### Legal Foundation
- [ ] ระบุ Legal Basis สำหรับการประมวลผลข้อมูลทุกประเภท
- [ ] Privacy Policy ที่ชัดเจนและเข้าถึงได้ง่าย
- [ ] จัดทำ ROPA (Record of Processing Activities)
- [ ] ระบุตัว Data Controller และ Data Processor
- [ ] สัญญากับ Data Processor ที่ครบถ้วน

### Technical Controls
- [ ] Encryption at rest สำหรับ sensitive data
- [ ] Encryption in transit (TLS)
- [ ] Access control ที่เข้มงวด (RBAC + RLS)
- [ ] Audit logging ที่สมบูรณ์
- [ ] Vulnerability management program

### Data Subject Rights
- [ ] Right of Access: export ข้อมูลได้ภายใน 30 วัน
- [ ] Right to Rectification: แก้ไขข้อมูลได้
- [ ] Right to Erasure: ลบ/anonymize ข้อมูลได้
- [ ] Right to Portability: export ในรูปแบบ machine-readable
- [ ] Right to Restriction: หยุดประมวลผลได้
- [ ] DSAR request tracking system

### Consent Management
- [ ] Consent ก่อนประมวลผลข้อมูล (ถ้า legal basis = consent)
- [ ] Consent ที่ชัดเจน เฉพาะเจาะจง และ informed
- [ ] ถอน consent ได้ง่ายพอๆ กับการให้
- [ ] เก็บ consent records พร้อม timestamp, IP, version
- [ ] Cookie consent สำหรับ non-essential cookies

### Data Retention
- [ ] Retention policy สำหรับข้อมูลทุกประเภท
- [ ] Automated deletion/anonymization
- [ ] Backup retention aligned กับ retention policy
- [ ] Secure disposal ของ storage media

### Breach Response
- [ ] Breach detection และ monitoring
- [ ] Incident response plan
- [ ] 72-hour notification process (GDPR)
- [ ] Communication templates
- [ ] Regular drills

### Organization
- [ ] DPO แต่งตั้ง (ถ้าจำเป็น)
- [ ] Privacy training สำหรับ staff
- [ ] Data protection policies
- [ ] Vendor management / Third-party assessment
- [ ] Privacy by Design ใน development process
```

---

*เนื้อหานี้เป็นส่วนหนึ่งของ Database Cluster Course - World-Class Level*
*Part 93/100: Compliance - GDPR และ PDPA*
