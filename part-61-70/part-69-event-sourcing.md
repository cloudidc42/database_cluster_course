# Part 69: Event Sourcing Pattern

## บทนำ

Event Sourcing เป็น architectural pattern ที่เปลี่ยนวิธีการเก็บข้อมูลในระบบ แทนที่จะเก็บแค่ "state ปัจจุบัน" (เช่น balance = 500 บาท) ระบบ Event Sourcing เก็บ "ลำดับของ events ที่เกิดขึ้น" ทั้งหมดที่นำไปสู่ state นั้น (เช่น deposit 1000 → withdraw 300 → deposit 200 → withdraw 400 = 500)

---

## 1. Traditional CRUD vs Event Sourcing

### 1.1 Traditional CRUD

```
CRUD Pattern:
━━━━━━━━━━━━━

Database มีแค่ current state:

users table:
┌──────┬────────────┬─────────┬──────────────────────┐
│  id  │   name     │ balance │    updated_at        │
├──────┼────────────┼─────────┼──────────────────────┤
│  1   │  Alice     │  500    │ 2024-01-15 14:30:00  │
└──────┴────────────┴─────────┴──────────────────────┘

คำถาม: ทำไม balance ถึงเป็น 500?
- ไม่รู้! เพราะ history ถูกทับทุกครั้งที่ UPDATE

ปัญหา:
1. ไม่มี audit trail: ใครทำอะไร เมื่อไหร่?
2. ไม่สามารถ "replay" ว่าเกิดอะไรขึ้น
3. ไม่สามารถย้อนเวลากลับไปดู state เก่า
4. Debugging ยากมากเมื่อเกิด bug
```

### 1.2 Event Sourcing

```
Event Sourcing Pattern:
━━━━━━━━━━━━━━━━━━━━━━━

events table (append-only):
┌──────────┬──────────────┬───────────┬────────────────────┬────────────────────────┐
│ event_id │ aggregate_id │ event_type│    event_data      │     created_at         │
├──────────┼──────────────┼───────────┼────────────────────┼────────────────────────┤
│  uuid-1  │     1        │ Deposited │ {"amount": 1000}   │ 2024-01-10 09:00:00    │
│  uuid-2  │     1        │ Withdrawn │ {"amount": 300}    │ 2024-01-11 10:00:00    │
│  uuid-3  │     1        │ Deposited │ {"amount": 200}    │ 2024-01-12 11:00:00    │
│  uuid-4  │     1        │ Withdrawn │ {"amount": 400}    │ 2024-01-15 14:30:00    │
└──────────┴──────────────┴───────────┴────────────────────┴────────────────────────┘

State = Replay ทุก events:
1000 - 300 + 200 - 400 = 500 ✅

ข้อดี:
1. ✅ Complete audit trail: รู้ทุกอย่างที่เกิดขึ้น
2. ✅ Time travel: ดู state ณ เวลาใดก็ได้
3. ✅ Replay: rebuild state ใหม่ได้เสมอ
4. ✅ Event-driven: trigger actions จาก events
5. ✅ Multiple projections: หลาย views จาก events เดียวกัน
```

---

## 2. Core Concepts

### 2.1 Events

```
Event คืออะไร?
━━━━━━━━━━━━━━

- Immutable fact: สิ่งที่เกิดขึ้นแล้ว ไม่เปลี่ยนแปลง
- Named in past tense: "OrderPlaced", "PaymentReceived", "ItemShipped"
- Carries all relevant data: ข้อมูลที่จำเป็นทั้งหมด
- Has timestamp: เวลาที่เกิดขึ้น

ตัวอย่าง Event:
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_type": "OrderPlaced",
  "aggregate_type": "Order",
  "aggregate_id": "order-12345",
  "sequence_number": 1,
  "event_data": {
    "customer_id": "cust-789",
    "items": [
      { "product_id": "prod-001", "quantity": 2, "price": 500 }
    ],
    "total_amount": 1000,
    "shipping_address": "123 Main St, Bangkok"
  },
  "metadata": {
    "user_id": "user-456",
    "ip_address": "192.168.1.1",
    "correlation_id": "req-98765"
  },
  "created_at": "2024-01-15T10:30:00Z"
}
```

### 2.2 Aggregate

```
Aggregate:
━━━━━━━━━━

- Entity ที่มี business rules
- Boundary ของ consistency: ทุกอย่างใน aggregate เป็น consistent กัน
- Emits events เมื่อ state เปลี่ยน
- Rebuilt จาก events (apply events ทีละตัว)

Order Aggregate States:
Created → Paid → Shipped → Delivered
                         ↘ Cancelled
```

### 2.3 Event Store

```
Event Store:
━━━━━━━━━━━━

- Append-only storage: ไม่มี UPDATE/DELETE
- Ordered: events มี sequence number
- Queryable: ดึง events ตาม aggregate_id
- Durable: ไม่สูญหาย
```

---

## 3. Event Schema Design

### 3.1 PostgreSQL Events Table

```sql
-- ════════════════════════════════════════════════
-- Events Table: Core of Event Store
-- ════════════════════════════════════════════════

CREATE TABLE events (
    -- Identity
    event_id        UUID DEFAULT gen_random_uuid() NOT NULL,
    
    -- Aggregate info
    aggregate_type  VARCHAR(100) NOT NULL,  -- 'Order', 'BankAccount', 'User'
    aggregate_id    VARCHAR(255) NOT NULL,   -- UUID หรือ domain ID ของ aggregate
    
    -- Event info
    event_type      VARCHAR(100) NOT NULL,  -- 'OrderPlaced', 'PaymentReceived'
    event_version   INT NOT NULL DEFAULT 1, -- สำหรับ schema evolution
    
    -- Data
    event_data      JSONB NOT NULL,         -- payload ของ event
    metadata        JSONB NOT NULL DEFAULT '{}', -- context: user, ip, correlation_id
    
    -- Ordering and optimistic concurrency
    sequence_number BIGINT NOT NULL,        -- version ของ aggregate (1, 2, 3, ...)
    global_position BIGSERIAL,              -- global ordering ข้าม aggregates
    
    -- Timestamps
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    -- Constraints
    CONSTRAINT events_pkey PRIMARY KEY (event_id),
    CONSTRAINT events_aggregate_sequence_unique UNIQUE (aggregate_type, aggregate_id, sequence_number)
);

-- Indexes
CREATE INDEX idx_events_aggregate 
    ON events (aggregate_type, aggregate_id, sequence_number);

CREATE INDEX idx_events_type 
    ON events (event_type, created_at DESC);

CREATE INDEX idx_events_global_position 
    ON events (global_position);

CREATE INDEX idx_events_created_at 
    ON events (created_at DESC);

-- Partial index สำหรับ recent events (hot data)
CREATE INDEX idx_events_recent 
    ON events (aggregate_id, sequence_number DESC)
    WHERE created_at > NOW() - INTERVAL '7 days';

-- ════════════════════════════════════════════════
-- Snapshots Table
-- ════════════════════════════════════════════════

CREATE TABLE snapshots (
    snapshot_id     UUID DEFAULT gen_random_uuid() NOT NULL,
    aggregate_type  VARCHAR(100) NOT NULL,
    aggregate_id    VARCHAR(255) NOT NULL,
    aggregate_version BIGINT NOT NULL,  -- sequence_number ณ ตอน snapshot
    state_data      JSONB NOT NULL,     -- serialized state
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT snapshots_pkey PRIMARY KEY (snapshot_id),
    CONSTRAINT snapshots_latest UNIQUE (aggregate_type, aggregate_id)
    -- ถ้าต้องการเก็บหลาย snapshots ให้เอา UNIQUE ออก
);

CREATE INDEX idx_snapshots_aggregate 
    ON snapshots (aggregate_type, aggregate_id, aggregate_version DESC);

-- ════════════════════════════════════════════════
-- Projections tracking
-- ════════════════════════════════════════════════

CREATE TABLE projection_positions (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_position   BIGINT NOT NULL DEFAULT 0,  -- global_position ล่าสุดที่ process แล้ว
    updated_at      TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### 3.2 Append-Only Constraints

```sql
-- ════════════════════════════════════════════════
-- Prevent Modifications: Event Store ต้อง Append-Only
-- ════════════════════════════════════════════════

-- ป้องกัน UPDATE บน events
CREATE OR REPLACE RULE events_no_update AS
    ON UPDATE TO events DO INSTEAD
    RAISE EXCEPTION 'Events are immutable - updates are not allowed';

-- ป้องกัน DELETE บน events
CREATE OR REPLACE RULE events_no_delete AS
    ON DELETE TO events DO INSTEAD
    RAISE EXCEPTION 'Events are immutable - deletes are not allowed';

-- หรือใช้ Row Security Policy
ALTER TABLE events ENABLE ROW LEVEL SECURITY;

CREATE POLICY events_insert_only ON events
    FOR ALL
    USING (true)
    WITH CHECK (true);

-- เฉพาะ SELECT และ INSERT เท่านั้น
REVOKE UPDATE, DELETE ON events FROM PUBLIC;
REVOKE UPDATE, DELETE ON events FROM app_user;
GRANT SELECT, INSERT ON events TO app_user;
```

### 3.3 Optimistic Concurrency

```sql
-- ════════════════════════════════════════════════
-- Optimistic Concurrency: ป้องกัน concurrent writes
-- ════════════════════════════════════════════════

-- Function สำหรับ append event พร้อม version check
CREATE OR REPLACE FUNCTION append_event(
    p_aggregate_type VARCHAR,
    p_aggregate_id VARCHAR,
    p_expected_version BIGINT,  -- version ที่คาดว่าจะเป็น (0 = ไม่เคยมี event)
    p_event_type VARCHAR,
    p_event_data JSONB,
    p_metadata JSONB DEFAULT '{}'
) RETURNS BIGINT AS $$
DECLARE
    v_current_version BIGINT;
    v_new_version BIGINT;
BEGIN
    -- Lock aggregate (prevent concurrent writes)
    PERFORM pg_advisory_xact_lock(
        hashtext(p_aggregate_type || ':' || p_aggregate_id)
    );
    
    -- ดู current version
    SELECT COALESCE(MAX(sequence_number), 0) INTO v_current_version
    FROM events
    WHERE aggregate_type = p_aggregate_type
      AND aggregate_id = p_aggregate_id;
    
    -- Check version match (optimistic concurrency)
    IF v_current_version != p_expected_version THEN
        RAISE EXCEPTION 'Concurrency conflict: expected version %, actual version %',
            p_expected_version, v_current_version;
    END IF;
    
    v_new_version := v_current_version + 1;
    
    -- Append event
    INSERT INTO events (
        aggregate_type, aggregate_id,
        event_type, event_data, metadata,
        sequence_number
    ) VALUES (
        p_aggregate_type, p_aggregate_id,
        p_event_type, p_event_data, p_metadata,
        v_new_version
    );
    
    RETURN v_new_version;
END;
$$ LANGUAGE plpgsql;
```

---

## 4. Implementation: Bank Account Aggregate

### 4.1 TypeScript Types และ Interfaces

```typescript
// ════════════════════════════════════════════════
// Types: Bank Account Event Sourcing
// ════════════════════════════════════════════════

// Base Event Interface
interface DomainEvent {
    eventId: string;
    eventType: string;
    aggregateType: string;
    aggregateId: string;
    sequenceNumber: number;
    eventData: Record<string, any>;
    metadata: EventMetadata;
    createdAt: Date;
}

interface EventMetadata {
    userId?: string;
    ipAddress?: string;
    correlationId?: string;
    causationId?: string;
}

// Bank Account Events
interface AccountOpenedEvent extends DomainEvent {
    eventType: 'AccountOpened';
    eventData: {
        accountId: string;
        ownerId: string;
        initialBalance: number;
        accountType: 'savings' | 'checking';
        currency: string;
    };
}

interface MoneyDepositedEvent extends DomainEvent {
    eventType: 'MoneyDeposited';
    eventData: {
        amount: number;
        description: string;
        referenceNumber: string;
    };
}

interface MoneyWithdrawnEvent extends DomainEvent {
    eventType: 'MoneyWithdrawn';
    eventData: {
        amount: number;
        description: string;
        referenceNumber: string;
    };
}

interface TransferInitiatedEvent extends DomainEvent {
    eventType: 'TransferInitiated';
    eventData: {
        amount: number;
        toAccountId: string;
        description: string;
        transferId: string;
    };
}

interface AccountFrozenEvent extends DomainEvent {
    eventType: 'AccountFrozen';
    eventData: {
        reason: string;
        frozenBy: string;
    };
}

type BankAccountEvent = 
    | AccountOpenedEvent 
    | MoneyDepositedEvent 
    | MoneyWithdrawnEvent 
    | TransferInitiatedEvent
    | AccountFrozenEvent;

// Aggregate State
interface BankAccountState {
    accountId: string;
    ownerId: string;
    balance: number;
    accountType: 'savings' | 'checking';
    currency: string;
    isFrozen: boolean;
    version: number;  // current sequence_number
    openedAt: Date;
}
```

### 4.2 Aggregate Implementation

```typescript
// ════════════════════════════════════════════════
// BankAccount Aggregate
// ════════════════════════════════════════════════

class BankAccount {
    private state: BankAccountState | null = null;
    private uncommittedEvents: BankAccountEvent[] = [];
    
    // Rebuild state จาก events
    static fromEvents(events: BankAccountEvent[]): BankAccount {
        const account = new BankAccount();
        for (const event of events) {
            account.applyEvent(event, false);  // false = don't add to uncommitted
        }
        return account;
    }
    
    // Rebuild จาก snapshot + events
    static fromSnapshot(
        snapshot: BankAccountState,
        events: BankAccountEvent[]
    ): BankAccount {
        const account = new BankAccount();
        account.state = { ...snapshot };
        
        for (const event of events) {
            account.applyEvent(event, false);
        }
        
        return account;
    }
    
    // ════════════════════════════════════════════════
    // Commands: Business operations ที่สร้าง events
    // ════════════════════════════════════════════════
    
    static open(params: {
        accountId: string;
        ownerId: string;
        initialBalance: number;
        accountType: 'savings' | 'checking';
        currency: string;
        metadata?: EventMetadata;
    }): BankAccount {
        if (params.initialBalance < 0) {
            throw new Error('Initial balance cannot be negative');
        }
        
        const account = new BankAccount();
        const event: AccountOpenedEvent = {
            eventId: crypto.randomUUID(),
            eventType: 'AccountOpened',
            aggregateType: 'BankAccount',
            aggregateId: params.accountId,
            sequenceNumber: 1,
            eventData: {
                accountId: params.accountId,
                ownerId: params.ownerId,
                initialBalance: params.initialBalance,
                accountType: params.accountType,
                currency: params.currency
            },
            metadata: params.metadata || {},
            createdAt: new Date()
        };
        
        account.applyEvent(event, true);
        return account;
    }
    
    deposit(params: {
        amount: number;
        description: string;
        referenceNumber: string;
        metadata?: EventMetadata;
    }): void {
        if (!this.state) throw new Error('Account not opened');
        if (this.state.isFrozen) throw new Error('Account is frozen');
        if (params.amount <= 0) throw new Error('Amount must be positive');
        if (params.amount > 10_000_000) throw new Error('Amount exceeds single transaction limit');
        
        const event: MoneyDepositedEvent = {
            eventId: crypto.randomUUID(),
            eventType: 'MoneyDeposited',
            aggregateType: 'BankAccount',
            aggregateId: this.state.accountId,
            sequenceNumber: this.state.version + 1,
            eventData: {
                amount: params.amount,
                description: params.description,
                referenceNumber: params.referenceNumber
            },
            metadata: params.metadata || {},
            createdAt: new Date()
        };
        
        this.applyEvent(event, true);
    }
    
    withdraw(params: {
        amount: number;
        description: string;
        referenceNumber: string;
        metadata?: EventMetadata;
    }): void {
        if (!this.state) throw new Error('Account not opened');
        if (this.state.isFrozen) throw new Error('Account is frozen');
        if (params.amount <= 0) throw new Error('Amount must be positive');
        if (params.amount > this.state.balance) {
            throw new Error(`Insufficient balance: have ${this.state.balance}, need ${params.amount}`);
        }
        
        const event: MoneyWithdrawnEvent = {
            eventId: crypto.randomUUID(),
            eventType: 'MoneyWithdrawn',
            aggregateType: 'BankAccount',
            aggregateId: this.state.accountId,
            sequenceNumber: this.state.version + 1,
            eventData: {
                amount: params.amount,
                description: params.description,
                referenceNumber: params.referenceNumber
            },
            metadata: params.metadata || {},
            createdAt: new Date()
        };
        
        this.applyEvent(event, true);
    }
    
    freeze(params: { reason: string; frozenBy: string; metadata?: EventMetadata }): void {
        if (!this.state) throw new Error('Account not opened');
        if (this.state.isFrozen) throw new Error('Account is already frozen');
        
        const event: AccountFrozenEvent = {
            eventId: crypto.randomUUID(),
            eventType: 'AccountFrozen',
            aggregateType: 'BankAccount',
            aggregateId: this.state.accountId,
            sequenceNumber: this.state.version + 1,
            eventData: {
                reason: params.reason,
                frozenBy: params.frozenBy
            },
            metadata: params.metadata || {},
            createdAt: new Date()
        };
        
        this.applyEvent(event, true);
    }
    
    // ════════════════════════════════════════════════
    // Event Application: Update state based on event
    // ════════════════════════════════════════════════
    
    private applyEvent(event: BankAccountEvent, isNew: boolean): void {
        switch (event.eventType) {
            case 'AccountOpened':
                this.state = {
                    accountId: event.eventData.accountId,
                    ownerId: event.eventData.ownerId,
                    balance: event.eventData.initialBalance,
                    accountType: event.eventData.accountType,
                    currency: event.eventData.currency,
                    isFrozen: false,
                    version: event.sequenceNumber,
                    openedAt: event.createdAt
                };
                break;
                
            case 'MoneyDeposited':
                this.state!.balance += event.eventData.amount;
                this.state!.version = event.sequenceNumber;
                break;
                
            case 'MoneyWithdrawn':
                this.state!.balance -= event.eventData.amount;
                this.state!.version = event.sequenceNumber;
                break;
                
            case 'TransferInitiated':
                this.state!.balance -= event.eventData.amount;
                this.state!.version = event.sequenceNumber;
                break;
                
            case 'AccountFrozen':
                this.state!.isFrozen = true;
                this.state!.version = event.sequenceNumber;
                break;
        }
        
        if (isNew) {
            this.uncommittedEvents.push(event);
        }
    }
    
    // ════════════════════════════════════════════════
    // Getters
    // ════════════════════════════════════════════════
    
    getState(): BankAccountState {
        if (!this.state) throw new Error('Account not opened');
        return { ...this.state };  // Return copy
    }
    
    getUncommittedEvents(): BankAccountEvent[] {
        return [...this.uncommittedEvents];
    }
    
    clearUncommittedEvents(): void {
        this.uncommittedEvents = [];
    }
    
    get version(): number {
        return this.state?.version ?? 0;
    }
}
```

### 4.3 Event Store Repository

```typescript
// ════════════════════════════════════════════════
// Event Store Repository: เขียน/อ่าน Events จาก DB
// ════════════════════════════════════════════════

import { Pool, PoolClient } from 'pg';

class EventStoreRepository {
    constructor(private db: Pool) {}
    
    async appendEvents(
        events: DomainEvent[],
        expectedVersion: number
    ): Promise<void> {
        if (events.length === 0) return;
        
        const client = await this.db.connect();
        try {
            await client.query('BEGIN');
            
            const firstEvent = events[0];
            
            // Check current version (Optimistic Concurrency Control)
            const versionResult = await client.query(
                `SELECT COALESCE(MAX(sequence_number), 0) as current_version
                 FROM events
                 WHERE aggregate_type = $1 AND aggregate_id = $2
                 FOR UPDATE`,  // Row lock
                [firstEvent.aggregateType, firstEvent.aggregateId]
            );
            
            const currentVersion = parseInt(versionResult.rows[0].current_version);
            
            if (currentVersion !== expectedVersion) {
                throw new ConcurrencyError(
                    `Concurrency conflict for ${firstEvent.aggregateType}:${firstEvent.aggregateId}. ` +
                    `Expected version ${expectedVersion}, actual ${currentVersion}`
                );
            }
            
            // Insert events
            for (const event of events) {
                await client.query(
                    `INSERT INTO events 
                     (event_id, aggregate_type, aggregate_id, event_type, 
                      event_data, metadata, sequence_number, created_at)
                     VALUES ($1, $2, $3, $4, $5, $6, $7, $8)`,
                    [
                        event.eventId,
                        event.aggregateType,
                        event.aggregateId,
                        event.eventType,
                        JSON.stringify(event.eventData),
                        JSON.stringify(event.metadata),
                        event.sequenceNumber,
                        event.createdAt
                    ]
                );
            }
            
            await client.query('COMMIT');
            
        } catch (err) {
            await client.query('ROLLBACK');
            throw err;
        } finally {
            client.release();
        }
    }
    
    async getEvents(
        aggregateType: string,
        aggregateId: string,
        fromVersion: number = 0
    ): Promise<DomainEvent[]> {
        const result = await this.db.query(
            `SELECT event_id, aggregate_type, aggregate_id, event_type,
                    event_data, metadata, sequence_number, created_at
             FROM events
             WHERE aggregate_type = $1 
               AND aggregate_id = $2
               AND sequence_number > $3
             ORDER BY sequence_number ASC`,
            [aggregateType, aggregateId, fromVersion]
        );
        
        return result.rows.map(row => ({
            eventId: row.event_id,
            aggregateType: row.aggregate_type,
            aggregateId: row.aggregate_id,
            eventType: row.event_type,
            eventData: row.event_data,
            metadata: row.metadata,
            sequenceNumber: row.sequence_number,
            createdAt: row.created_at
        }));
    }
    
    async getEventsByType(
        eventType: string,
        fromPosition: number = 0,
        limit: number = 100
    ): Promise<DomainEvent[]> {
        const result = await this.db.query(
            `SELECT event_id, aggregate_type, aggregate_id, event_type,
                    event_data, metadata, sequence_number, global_position, created_at
             FROM events
             WHERE event_type = $1
               AND global_position > $2
             ORDER BY global_position ASC
             LIMIT $3`,
            [eventType, fromPosition, limit]
        );
        
        return result.rows.map(this.mapRow);
    }
    
    async getAllEventsAfterPosition(
        position: number,
        limit: number = 500
    ): Promise<DomainEvent[]> {
        const result = await this.db.query(
            `SELECT event_id, aggregate_type, aggregate_id, event_type,
                    event_data, metadata, sequence_number, global_position, created_at
             FROM events
             WHERE global_position > $1
             ORDER BY global_position ASC
             LIMIT $2`,
            [position, limit]
        );
        
        return result.rows.map(this.mapRow);
    }
    
    private mapRow(row: any): DomainEvent {
        return {
            eventId: row.event_id,
            aggregateType: row.aggregate_type,
            aggregateId: row.aggregate_id,
            eventType: row.event_type,
            eventData: row.event_data,
            metadata: row.metadata,
            sequenceNumber: row.sequence_number,
            createdAt: row.created_at
        };
    }
}

class ConcurrencyError extends Error {
    constructor(message: string) {
        super(message);
        this.name = 'ConcurrencyError';
    }
}
```

---

## 5. Snapshots

### 5.1 Snapshot Repository

```typescript
// ════════════════════════════════════════════════
// Snapshot: เก็บ State เพื่อ optimize replay
// ════════════════════════════════════════════════

class SnapshotRepository {
    constructor(private db: Pool) {}
    
    async saveSnapshot(
        aggregateType: string,
        aggregateId: string,
        version: number,
        state: any
    ): Promise<void> {
        await this.db.query(
            `INSERT INTO snapshots 
             (aggregate_type, aggregate_id, aggregate_version, state_data)
             VALUES ($1, $2, $3, $4)
             ON CONFLICT (aggregate_type, aggregate_id) 
             DO UPDATE SET 
               aggregate_version = EXCLUDED.aggregate_version,
               state_data = EXCLUDED.state_data,
               created_at = CURRENT_TIMESTAMP`,
            [aggregateType, aggregateId, version, JSON.stringify(state)]
        );
    }
    
    async getLatestSnapshot(
        aggregateType: string,
        aggregateId: string
    ): Promise<{ version: number; state: any } | null> {
        const result = await this.db.query(
            `SELECT aggregate_version, state_data
             FROM snapshots
             WHERE aggregate_type = $1 AND aggregate_id = $2`,
            [aggregateType, aggregateId]
        );
        
        if (result.rows.length === 0) return null;
        
        return {
            version: result.rows[0].aggregate_version,
            state: result.rows[0].state_data
        };
    }
}

// ════════════════════════════════════════════════
// BankAccount Repository with Snapshot Support
// ════════════════════════════════════════════════

const SNAPSHOT_THRESHOLD = 50; // Snapshot ทุก 50 events

class BankAccountRepository {
    constructor(
        private eventStore: EventStoreRepository,
        private snapshots: SnapshotRepository
    ) {}
    
    async save(account: BankAccount): Promise<void> {
        const events = account.getUncommittedEvents();
        if (events.length === 0) return;
        
        const expectedVersion = account.version - events.length;
        await this.eventStore.appendEvents(events, expectedVersion);
        account.clearUncommittedEvents();
        
        // Create snapshot ถ้าเกิน threshold
        const state = account.getState();
        if (state.version % SNAPSHOT_THRESHOLD === 0) {
            await this.snapshots.saveSnapshot(
                'BankAccount',
                state.accountId,
                state.version,
                state
            );
            console.log(`📸 Snapshot created for ${state.accountId} at version ${state.version}`);
        }
    }
    
    async findById(accountId: string): Promise<BankAccount | null> {
        // 1. ลอง load snapshot ก่อน
        const snapshot = await this.snapshots.getLatestSnapshot('BankAccount', accountId);
        
        let fromVersion = 0;
        let account: BankAccount;
        
        if (snapshot) {
            // Load events หลัง snapshot เท่านั้น (ลด events ที่ต้อง replay)
            fromVersion = snapshot.version;
            const events = await this.eventStore.getEvents(
                'BankAccount', accountId, fromVersion
            ) as BankAccountEvent[];
            
            account = BankAccount.fromSnapshot(snapshot.state, events);
            console.log(`📸 Loaded from snapshot v${snapshot.version} + ${events.length} events`);
        } else {
            // Load ทุก events
            const events = await this.eventStore.getEvents(
                'BankAccount', accountId, 0
            ) as BankAccountEvent[];
            
            if (events.length === 0) return null;
            
            account = BankAccount.fromEvents(events);
            console.log(`📜 Loaded from ${events.length} events`);
        }
        
        return account;
    }
    
    // Time Travel: ดู state ณ เวลาหนึ่ง
    async findAtVersion(accountId: string, atVersion: number): Promise<BankAccount | null> {
        const events = await this.eventStore.getEvents('BankAccount', accountId, 0);
        const eventsUpTo = events.filter(e => e.sequenceNumber <= atVersion) as BankAccountEvent[];
        
        if (eventsUpTo.length === 0) return null;
        return BankAccount.fromEvents(eventsUpTo);
    }
}
```

---

## 6. Projections: Rebuild Current State

### 6.1 Read Model Projections

```typescript
// ════════════════════════════════════════════════
// Projections: Read Models สำหรับ Query ที่รวดเร็ว
// ════════════════════════════════════════════════

// Account Summary Projection (denormalized view สำหรับ read)
interface AccountSummary {
    account_id: string;
    owner_id: string;
    balance: number;
    account_type: string;
    currency: string;
    is_frozen: boolean;
    transaction_count: number;
    last_transaction_at: Date | null;
    version: number;
}

class AccountSummaryProjection {
    constructor(
        private db: Pool,
        private eventStore: EventStoreRepository
    ) {}
    
    // Process event หนึ่งตัว
    async handleEvent(event: BankAccountEvent): Promise<void> {
        switch (event.eventType) {
            case 'AccountOpened':
                await this.db.query(
                    `INSERT INTO account_summaries 
                     (account_id, owner_id, balance, account_type, currency, 
                      is_frozen, transaction_count, version)
                     VALUES ($1, $2, $3, $4, $5, false, 0, $6)
                     ON CONFLICT (account_id) DO NOTHING`,
                    [
                        event.eventData.accountId,
                        event.eventData.ownerId,
                        event.eventData.initialBalance,
                        event.eventData.accountType,
                        event.eventData.currency,
                        event.sequenceNumber
                    ]
                );
                break;
                
            case 'MoneyDeposited':
                await this.db.query(
                    `UPDATE account_summaries SET
                        balance = balance + $1,
                        transaction_count = transaction_count + 1,
                        last_transaction_at = $2,
                        version = $3
                     WHERE account_id = $4`,
                    [
                        event.eventData.amount,
                        event.createdAt,
                        event.sequenceNumber,
                        event.aggregateId
                    ]
                );
                break;
                
            case 'MoneyWithdrawn':
                await this.db.query(
                    `UPDATE account_summaries SET
                        balance = balance - $1,
                        transaction_count = transaction_count + 1,
                        last_transaction_at = $2,
                        version = $3
                     WHERE account_id = $4`,
                    [
                        event.eventData.amount,
                        event.createdAt,
                        event.sequenceNumber,
                        event.aggregateId
                    ]
                );
                break;
                
            case 'AccountFrozen':
                await this.db.query(
                    `UPDATE account_summaries SET
                        is_frozen = true,
                        version = $1
                     WHERE account_id = $2`,
                    [event.sequenceNumber, event.aggregateId]
                );
                break;
        }
    }
    
    // Rebuild projection จาก scratch
    async rebuild(fromPosition: number = 0): Promise<void> {
        console.log('🔄 Rebuilding AccountSummary projection...');
        
        // Clear existing projection data
        if (fromPosition === 0) {
            await this.db.query('TRUNCATE TABLE account_summaries');
        }
        
        let position = fromPosition;
        let processed = 0;
        
        while (true) {
            const events = await this.eventStore.getAllEventsAfterPosition(position, 500);
            if (events.length === 0) break;
            
            for (const event of events) {
                if (event.aggregateType === 'BankAccount') {
                    await this.handleEvent(event as BankAccountEvent);
                }
                position = (event as any).globalPosition || position + 1;
                processed++;
            }
            
            // Update checkpoint
            await this.db.query(
                `INSERT INTO projection_positions (projection_name, last_position)
                 VALUES ('AccountSummary', $1)
                 ON CONFLICT (projection_name) DO UPDATE SET last_position = $1, updated_at = NOW()`,
                [position]
            );
            
            console.log(`  Processed ${processed} events, position: ${position}`);
        }
        
        console.log(`✅ Rebuild complete: ${processed} events processed`);
    }
}
```

### 6.2 Transaction History Projection

```sql
-- ════════════════════════════════════════════════
-- Transaction History: Read Model สำหรับ Statement
-- ════════════════════════════════════════════════

CREATE TABLE transaction_history (
    transaction_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id      VARCHAR(255) NOT NULL,
    transaction_type VARCHAR(50) NOT NULL,  -- 'deposit', 'withdrawal', 'transfer'
    amount          DECIMAL(15,2) NOT NULL,
    balance_after   DECIMAL(15,2) NOT NULL,
    description     TEXT,
    reference_number VARCHAR(100),
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL,
    event_id        UUID NOT NULL REFERENCES events(event_id)
);

CREATE INDEX idx_tx_history_account ON transaction_history(account_id, created_at DESC);
CREATE INDEX idx_tx_history_date ON transaction_history(created_at DESC);

-- SQL-based projection rebuild
INSERT INTO transaction_history (
    account_id, transaction_type, amount, balance_after,
    description, reference_number, created_at, event_id
)
WITH ordered_events AS (
    SELECT 
        event_id,
        aggregate_id as account_id,
        event_type,
        event_data,
        created_at,
        ROW_NUMBER() OVER (PARTITION BY aggregate_id ORDER BY sequence_number) as rn,
        SUM(
            CASE event_type
                WHEN 'AccountOpened' THEN (event_data->>'initialBalance')::DECIMAL
                WHEN 'MoneyDeposited' THEN (event_data->>'amount')::DECIMAL
                WHEN 'MoneyWithdrawn' THEN -(event_data->>'amount')::DECIMAL
                ELSE 0
            END
        ) OVER (
            PARTITION BY aggregate_id 
            ORDER BY sequence_number 
            ROWS UNBOUNDED PRECEDING
        ) as running_balance
    FROM events
    WHERE aggregate_type = 'BankAccount'
      AND event_type IN ('AccountOpened', 'MoneyDeposited', 'MoneyWithdrawn')
)
SELECT 
    account_id,
    CASE event_type
        WHEN 'AccountOpened' THEN 'initial_deposit'
        WHEN 'MoneyDeposited' THEN 'deposit'
        WHEN 'MoneyWithdrawn' THEN 'withdrawal'
    END as transaction_type,
    CASE event_type
        WHEN 'AccountOpened' THEN (event_data->>'initialBalance')::DECIMAL
        WHEN 'MoneyDeposited' THEN (event_data->>'amount')::DECIMAL
        WHEN 'MoneyWithdrawn' THEN (event_data->>'amount')::DECIMAL
    END as amount,
    running_balance as balance_after,
    event_data->>'description' as description,
    event_data->>'referenceNumber' as reference_number,
    created_at,
    event_id
FROM ordered_events
ON CONFLICT DO NOTHING;
```

---

## 7. Event Versioning

### 7.1 Handle Schema Evolution

```typescript
// ════════════════════════════════════════════════
// Event Versioning: Handle Schema Evolution
// ════════════════════════════════════════════════

// Version 1 (เก่า)
interface MoneyDepositedV1 {
    amount: number;
    description: string;
}

// Version 2 (ใหม่: เพิ่ม fields)
interface MoneyDepositedV2 {
    amount: number;
    currency: string;          // ← ใหม่
    description: string;
    referenceNumber: string;   // ← ใหม่
}

// Upcaster: แปลง V1 → V2
function upcaseMoneyDepositedV1ToV2(eventData: MoneyDepositedV1): MoneyDepositedV2 {
    return {
        amount: eventData.amount,
        currency: 'THB',           // default value สำหรับ events เก่า
        description: eventData.description,
        referenceNumber: 'LEGACY-' + Date.now()  // generate placeholder
    };
}

// Event Registry with versioning
class EventRegistry {
    private upcasters: Map<string, Map<number, (data: any) => any>> = new Map();
    
    registerUpcaster(
        eventType: string, 
        fromVersion: number, 
        upcaster: (data: any) => any
    ): void {
        if (!this.upcasters.has(eventType)) {
            this.upcasters.set(eventType, new Map());
        }
        this.upcasters.get(eventType)!.set(fromVersion, upcaster);
    }
    
    upcast(eventType: string, eventVersion: number, eventData: any): any {
        const eventUpcasters = this.upcasters.get(eventType);
        if (!eventUpcasters) return eventData;
        
        let currentVersion = eventVersion;
        let currentData = eventData;
        
        while (eventUpcasters.has(currentVersion)) {
            currentData = eventUpcasters.get(currentVersion)!(currentData);
            currentVersion++;
        }
        
        return currentData;
    }
}

// Setup registry
const registry = new EventRegistry();
registry.registerUpcaster('MoneyDeposited', 1, upcaseMoneyDepositedV1ToV2);
```

---

## 8. Kafka as Alternative Event Store

```yaml
# ════════════════════════════════════════════════
# Kafka Event Store Setup
# ════════════════════════════════════════════════

# docker-compose สำหรับ Kafka + Schema Registry
version: '3.8'
services:
  kafka:
    image: confluentinc/cp-kafka:7.4.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_NUM_PARTITIONS: 12
      KAFKA_DEFAULT_REPLICATION_FACTOR: 1
      KAFKA_LOG_RETENTION_BYTES: -1    # ไม่มี retention limit (event store!)
      KAFKA_LOG_RETENTION_MS: -1       # เก็บ events ตลอดไป
      KAFKA_CLEANUP_POLICY: compact    # Compaction แทน deletion
    ports:
      - "9092:9092"

  schema-registry:
    image: confluentinc/cp-schema-registry:7.4.0
    environment:
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
    ports:
      - "8081:8081"
```

```typescript
// Kafka Event Store (เทียบกับ PostgreSQL)
import { Kafka, Producer, Consumer, EachMessagePayload } from 'kafkajs';

class KafkaEventStore {
    private kafka: Kafka;
    private producer: Producer;
    
    constructor(brokers: string[]) {
        this.kafka = new Kafka({ brokers });
        this.producer = this.kafka.producer({ idempotent: true });
    }
    
    async appendEvent(event: DomainEvent): Promise<void> {
        await this.producer.send({
            topic: `events.${event.aggregateType.toLowerCase()}`,
            messages: [{
                key: event.aggregateId,  // key = aggregateId เพื่อ partition ordering
                value: JSON.stringify(event),
                headers: {
                    eventType: event.eventType,
                    aggregateType: event.aggregateType,
                    sequenceNumber: String(event.sequenceNumber)
                }
            }]
        });
    }
    
    async getEvents(
        aggregateType: string,
        aggregateId: string
    ): Promise<DomainEvent[]> {
        // Kafka ไม่ได้ออกแบบมาสำหรับ query by key
        // ต้องใช้ KTable (Kafka Streams) หรือ consumer แล้วกรอง
        // หรือ query จาก read model ที่ build จาก Kafka
        
        // สำหรับ aggregate rebuild: ใช้ PostgreSQL เป็น event store
        // ใช้ Kafka สำหรับ event streaming ไปยัง projections
        throw new Error('Use PostgreSQL for aggregate queries, Kafka for streaming');
    }
}
```

---

## 9. สรุปและ Best Practices

### 9.1 When to Use Event Sourcing

```
ใช้ Event Sourcing เมื่อ:
━━━━━━━━━━━━━━━━━━━━━━━━━━

✅ Business ต้องการ complete audit trail
   (เช่น banking, insurance, healthcare, legal)

✅ ต้องการ time travel / replay capability
   (debug ปัญหา, regulatory investigation)

✅ Complex domain with many business rules
   (domain-driven design)

✅ Multiple views ของข้อมูลเดียวกัน
   (CQRS + Event Sourcing)

❌ ไม่ควรใช้เมื่อ:
━━━━━━━━━━━━━━━━━

❌ Simple CRUD application (overkill)
❌ Team ไม่คุ้นเคยกับ Event Sourcing (learning curve สูง)
❌ ต้องการ query ข้อมูลที่ซับซ้อนและ flexible มาก
❌ Performance critical + ข้อมูลเยอะมาก (replay ช้า)
```

### 9.2 Common Pitfalls

```
Pitfalls ที่ต้องระวัง:
━━━━━━━━━━━━━━━━━━━━━━━

1. Event Granularity:
   ❌ Too fine: every field change = event (event spam)
   ❌ Too coarse: "OrderUpdated" (ไม่รู้ว่า update อะไร)
   ✅ Business meaningful: "OrderShipped", "PaymentFailed"

2. Event Schema Evolution:
   ❌ เปลี่ยน event schema โดยไม่มี migration plan
   ✅ Version events, implement upcasters

3. Large Aggregates:
   ❌ Aggregate มี 10,000+ events → replay ช้ามาก
   ✅ ใช้ Snapshots

4. Projections:
   ❌ Rebuild projection ไม่ได้ (ไม่ idempotent)
   ✅ Projections ต้องสามารถ rebuild จาก scratch ได้เสมอ

5. Eventual Consistency:
   ❌ คาดหวัง strong consistency ใน read models
   ✅ Accept eventual consistency, design UI accordingly
```
