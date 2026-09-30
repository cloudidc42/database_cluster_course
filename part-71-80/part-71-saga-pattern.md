# Part 71: Saga Pattern สำหรับ Distributed Transactions

## บทนำ

ในระบบ Microservices หนึ่ง Business Operation อาจต้องการการเปลี่ยนแปลงข้อมูลใน Services หลายตัวพร้อมกัน เช่น การสั่งซื้อสินค้าต้องการ: ตัดสต็อกสินค้า, เรียกเก็บเงินจากบัตรเครดิต, และยืนยันคำสั่งซื้อ — ทั้งหมดนี้ต้องสำเร็จพร้อมกัน หรือไม่ก็ต้องไม่เกิดขึ้นเลย (Atomicity)

ใน Monolithic Application เราใช้ Database Transaction ได้ตรงๆ แต่ใน Microservices แต่ละ Service มี Database ของตัวเอง ทำให้ ACID Transaction ข้าม Services ทำไม่ได้โดยตรง

---

## 1. ปัญหาของ Two-Phase Commit (2PC)

### 1.1 Two-Phase Commit คืออะไร

```
Phase 1 (Prepare):
  Coordinator → Service A: "เตรียมพร้อมสำหรับ commit ได้ไหม?"
  Coordinator → Service B: "เตรียมพร้อมสำหรับ commit ได้ไหม?"
  Coordinator → Service C: "เตรียมพร้อมสำหรับ commit ได้ไหม?"
  
  Service A → Coordinator: "OK, พร้อมแล้ว"
  Service B → Coordinator: "OK, พร้อมแล้ว"  
  Service C → Coordinator: "OK, พร้อมแล้ว"

Phase 2 (Commit):
  Coordinator → Service A: "Commit!"
  Coordinator → Service B: "Commit!"
  Coordinator → Service C: "Commit!"
```

### 1.2 ปัญหาของ 2PC

**Blocking Problem**: ในระหว่าง Prepare Phase ทุก Service ต้อง Lock Resources และรอ Coordinator ตอบกลับ ถ้า Coordinator ล้มเหลว ทุก Service จะ Block อยู่ตลอดไป

**Single Point of Failure**: Coordinator คือ SPOF ถ้าล้มเหลวหลัง Phase 1 แต่ก่อน Phase 2 — ระบบค้างทันที

**Performance**: แต่ละ Transaction ต้องใช้ 2 round-trips + lock resources นาน = Latency สูงมาก

**Distributed Deadlock**: Service A ล็อครอ Service B, Service B ล็อครอ Service A

**Network Partition**: ใน CAP theorem, 2PC เลือก Consistency + Availability ไม่ได้พร้อมกัน ถ้ามี Network Partition

```
ตัวอย่างปัญหา:
1. Coordinator ส่ง Prepare ไปทุก Services ✓
2. ทุก Services ตอบ OK ✓  
3. Coordinator ตัดสินใจ Commit และส่ง Commit ไป Service A ✓
4. *** Coordinator Crash *** 
5. Service B, C ยัง Lock อยู่ รอ Commit ที่ไม่มีวันมา
6. Deadlock!
```

---

## 2. Saga Pattern: แนวทางแก้ปัญหา

### 2.1 Saga คืออะไร

**Saga** คือ Sequence ของ Local Transactions ที่แต่ละ Transaction อัปเดต Database ของตัวเอง และ Publish Event หรือ Message เพื่อ Trigger Transaction ถัดไป

ถ้า Transaction ใดล้มเหลว Saga จะ Execute **Compensating Transactions** เพื่อ Undo การเปลี่ยนแปลงที่เกิดขึ้นก่อนหน้า

```
Order Saga:
  T1: Reserve Inventory (Inventory Service)
  T2: Charge Payment   (Payment Service)  
  T3: Confirm Order    (Order Service)

ถ้า T2 ล้มเหลว:
  C1: Release Inventory (compensate T1)
  (T2 failed, no need to compensate)
  (T3 never executed)
```

### 2.2 Properties ของ Saga

- **No Distributed Locks**: แต่ละ Service ใช้ Local Transaction ของตัวเอง
- **Eventual Consistency**: ระบบ Consistent ในที่สุด แต่ไม่ใช่ทันที
- **BASE แทน ACID**: Basically Available, Soft State, Eventual Consistent
- **Compensation ไม่ใช่ Rollback**: Semantic undo, ไม่ใช่ Technical undo

---

## 3. ประเภทของ Saga

### 3.1 Choreography Saga (Event-Based)

Services สื่อสารกันผ่าน Events โดยไม่มี Central Coordinator:

```
Order Service         Inventory Service      Payment Service
      |                      |                     |
      |--OrderCreated------->|                     |
      |                      |--InventoryReserved->|
      |                      |                     |--PaymentCharged-->
      |                      |                     |
      |<--OrderConfirmed---------------------------------|
```

### 3.2 Orchestration Saga (Coordinator-Based)

มี Central Saga Orchestrator คอยสั่งแต่ละ Service:

```
Saga Orchestrator
      |
      |--ReserveInventory-->  Inventory Service
      |<--InventoryReserved--
      |
      |--ChargePayment------>  Payment Service
      |<--PaymentCharged-----
      |
      |--ConfirmOrder------->  Order Service
      |<--OrderConfirmed-----
      |
   [COMPLETE]
```

---

## 4. Choreography Saga: Deep Dive

### 4.1 Event Flow

```typescript
// order-service/src/events/order-events.ts
export interface OrderCreatedEvent {
  eventId: string;
  sagaId: string;
  orderId: string;
  customerId: string;
  items: Array<{
    productId: string;
    quantity: number;
    price: number;
  }>;
  totalAmount: number;
  timestamp: Date;
}

export interface InventoryReservedEvent {
  eventId: string;
  sagaId: string;
  orderId: string;
  reservationId: string;
  timestamp: Date;
}

export interface InventoryReservationFailedEvent {
  eventId: string;
  sagaId: string;
  orderId: string;
  reason: string;
  timestamp: Date;
}

export interface PaymentChargedEvent {
  eventId: string;
  sagaId: string;
  orderId: string;
  transactionId: string;
  amount: number;
  timestamp: Date;
}

export interface PaymentFailedEvent {
  eventId: string;
  sagaId: string;
  orderId: string;
  reason: string;
  timestamp: Date;
}

export interface InventoryReleasedEvent {
  eventId: string;
  sagaId: string;
  orderId: string;
  reservationId: string;
  timestamp: Date;
}
```

### 4.2 Order Service (Choreography)

```typescript
// order-service/src/services/order.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, DataSource } from 'typeorm';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { Order, OrderStatus } from '../entities/order.entity';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class OrderService {
  constructor(
    @InjectRepository(Order)
    private orderRepository: Repository<Order>,
    private dataSource: DataSource,
    private eventEmitter: EventEmitter2,
  ) {}

  async createOrder(customerId: string, items: any[]): Promise<Order> {
    const sagaId = uuidv4();
    const orderId = uuidv4();
    const totalAmount = items.reduce((sum, item) => sum + item.price * item.quantity, 0);

    // Local transaction: create order with PENDING status
    const order = await this.dataSource.transaction(async (manager) => {
      const newOrder = manager.create(Order, {
        id: orderId,
        sagaId,
        customerId,
        items,
        totalAmount,
        status: OrderStatus.PENDING,
      });
      return manager.save(newOrder);
    });

    // Publish event to start saga
    await this.eventEmitter.emitAsync('order.created', {
      eventId: uuidv4(),
      sagaId,
      orderId,
      customerId,
      items,
      totalAmount,
      timestamp: new Date(),
    });

    return order;
  }

  // Listen to InventoryReserved event
  async handleInventoryReserved(event: any): Promise<void> {
    // Update order to reflect inventory reserved
    await this.orderRepository.update(
      { id: event.orderId },
      { status: OrderStatus.INVENTORY_RESERVED }
    );
    // Payment service will now handle PaymentCharged event
    // (Payment service listens to InventoryReserved)
  }

  // Listen to PaymentCharged event
  async handlePaymentCharged(event: any): Promise<void> {
    await this.dataSource.transaction(async (manager) => {
      await manager.update(Order, event.orderId, {
        status: OrderStatus.CONFIRMED,
        confirmedAt: new Date(),
      });
    });

    await this.eventEmitter.emitAsync('order.confirmed', {
      eventId: uuidv4(),
      sagaId: event.sagaId,
      orderId: event.orderId,
      timestamp: new Date(),
    });
  }

  // Compensation: handle payment failure
  async handlePaymentFailed(event: any): Promise<void> {
    await this.orderRepository.update(
      { id: event.orderId },
      { status: OrderStatus.PAYMENT_FAILED, failureReason: event.reason }
    );

    // Trigger compensation: release inventory
    await this.eventEmitter.emitAsync('inventory.release.requested', {
      eventId: uuidv4(),
      sagaId: event.sagaId,
      orderId: event.orderId,
      timestamp: new Date(),
    });
  }

  // Listen to InventoryReleased (final compensation)
  async handleInventoryReleased(event: any): Promise<void> {
    await this.orderRepository.update(
      { id: event.orderId },
      { status: OrderStatus.CANCELLED }
    );
  }
}
```

### 4.3 Inventory Service (Choreography)

```typescript
// inventory-service/src/services/inventory.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, DataSource } from 'typeorm';
import { EventEmitter2, OnEvent } from '@nestjs/event-emitter';
import { Inventory } from '../entities/inventory.entity';
import { Reservation } from '../entities/reservation.entity';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class InventoryService {
  private readonly logger = new Logger(InventoryService.name);

  constructor(
    @InjectRepository(Inventory)
    private inventoryRepository: Repository<Inventory>,
    @InjectRepository(Reservation)
    private reservationRepository: Repository<Reservation>,
    private dataSource: DataSource,
    private eventEmitter: EventEmitter2,
  ) {}

  @OnEvent('order.created')
  async handleOrderCreated(event: any): Promise<void> {
    this.logger.log(`Handling OrderCreated: sagaId=${event.sagaId}`);

    try {
      const reservationId = uuidv4();

      // Local transaction: reserve inventory
      await this.dataSource.transaction(async (manager) => {
        for (const item of event.items) {
          const inventory = await manager.findOne(Inventory, {
            where: { productId: item.productId },
            lock: { mode: 'pessimistic_write' },
          });

          if (!inventory || inventory.availableQuantity < item.quantity) {
            throw new Error(`Insufficient inventory for product ${item.productId}`);
          }

          await manager.update(Inventory, inventory.id, {
            availableQuantity: inventory.availableQuantity - item.quantity,
            reservedQuantity: inventory.reservedQuantity + item.quantity,
          });
        }

        await manager.save(Reservation, {
          id: reservationId,
          sagaId: event.sagaId,
          orderId: event.orderId,
          items: event.items,
          status: 'RESERVED',
        });
      });

      // Emit success event
      await this.eventEmitter.emitAsync('inventory.reserved', {
        eventId: uuidv4(),
        sagaId: event.sagaId,
        orderId: event.orderId,
        reservationId,
        timestamp: new Date(),
      });

    } catch (error) {
      this.logger.error(`Failed to reserve inventory: ${error.message}`);

      // Emit failure event
      await this.eventEmitter.emitAsync('inventory.reservation.failed', {
        eventId: uuidv4(),
        sagaId: event.sagaId,
        orderId: event.orderId,
        reason: error.message,
        timestamp: new Date(),
      });
    }
  }

  // Compensation: release inventory
  @OnEvent('inventory.release.requested')
  async handleReleaseRequested(event: any): Promise<void> {
    this.logger.log(`Releasing inventory: sagaId=${event.sagaId}`);

    await this.dataSource.transaction(async (manager) => {
      const reservation = await manager.findOne(Reservation, {
        where: { sagaId: event.sagaId, orderId: event.orderId },
      });

      if (!reservation || reservation.status === 'RELEASED') {
        this.logger.warn(`Reservation already released or not found`);
        return; // Idempotent: already done
      }

      for (const item of reservation.items) {
        const inventory = await manager.findOne(Inventory, {
          where: { productId: item.productId },
          lock: { mode: 'pessimistic_write' },
        });

        await manager.update(Inventory, inventory.id, {
          availableQuantity: inventory.availableQuantity + item.quantity,
          reservedQuantity: inventory.reservedQuantity - item.quantity,
        });
      }

      await manager.update(Reservation, reservation.id, { status: 'RELEASED' });
    });

    await this.eventEmitter.emitAsync('inventory.released', {
      eventId: uuidv4(),
      sagaId: event.sagaId,
      orderId: event.orderId,
      timestamp: new Date(),
    });
  }
}
```

### 4.4 ข้อดีและข้อเสียของ Choreography

**ข้อดี:**
- Loose Coupling: Services ไม่รู้จักกัน รู้จักแค่ Events
- Simple: ไม่ต้องมี Orchestrator
- Resilient: ไม่มี Single Point of Failure

**ข้อเสีย:**
- Hard to Track: ยากที่จะติดตามว่า Saga อยู่ที่ขั้นตอนไหน
- Cyclic Dependencies: A→B→C→A อาจเกิดได้โดยไม่ตั้งใจ
- Distributed Logic: Business Logic กระจายอยู่ใน Services ต่างๆ

---

## 5. Orchestration Saga: Deep Dive

### 5.1 Saga Orchestrator Design

```typescript
// saga-orchestrator/src/sagas/order-saga.orchestrator.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, DataSource } from 'typeorm';
import { SagaInstance, SagaStatus, SagaStep } from '../entities/saga.entity';
import { InventoryClient } from '../clients/inventory.client';
import { PaymentClient } from '../clients/payment.client';
import { OrderClient } from '../clients/order.client';
import { v4 as uuidv4 } from 'uuid';

export enum OrderSagaSteps {
  RESERVE_INVENTORY = 'RESERVE_INVENTORY',
  CHARGE_PAYMENT = 'CHARGE_PAYMENT',
  CONFIRM_ORDER = 'CONFIRM_ORDER',
}

export enum OrderSagaCompensationSteps {
  RELEASE_INVENTORY = 'RELEASE_INVENTORY',
  REFUND_PAYMENT = 'REFUND_PAYMENT',
  CANCEL_ORDER = 'CANCEL_ORDER',
}

@Injectable()
export class OrderSagaOrchestrator {
  private readonly logger = new Logger(OrderSagaOrchestrator.name);

  constructor(
    @InjectRepository(SagaInstance)
    private sagaRepository: Repository<SagaInstance>,
    private dataSource: DataSource,
    private inventoryClient: InventoryClient,
    private paymentClient: PaymentClient,
    private orderClient: OrderClient,
  ) {}

  async startOrderSaga(payload: {
    orderId: string;
    customerId: string;
    items: any[];
    totalAmount: number;
  }): Promise<SagaInstance> {
    const sagaId = uuidv4();

    // Create saga instance in DB
    const saga = await this.dataSource.transaction(async (manager) => {
      return manager.save(SagaInstance, {
        id: sagaId,
        type: 'ORDER_SAGA',
        status: SagaStatus.STARTED,
        payload,
        currentStep: OrderSagaSteps.RESERVE_INVENTORY,
        completedSteps: [],
        compensatingSteps: [],
        createdAt: new Date(),
        updatedAt: new Date(),
      });
    });

    // Execute saga asynchronously
    this.executeSaga(saga).catch((err) => {
      this.logger.error(`Saga execution failed: ${err.message}`, err.stack);
    });

    return saga;
  }

  private async executeSaga(saga: SagaInstance): Promise<void> {
    this.logger.log(`Executing saga: ${saga.id}`);

    try {
      // Step 1: Reserve Inventory
      await this.updateSagaStatus(saga.id, SagaStatus.PROCESSING, OrderSagaSteps.RESERVE_INVENTORY);
      const reservation = await this.inventoryClient.reserveInventory({
        sagaId: saga.id,
        orderId: saga.payload.orderId,
        items: saga.payload.items,
      });
      await this.recordStepCompletion(saga.id, OrderSagaSteps.RESERVE_INVENTORY, {
        reservationId: reservation.reservationId,
      });

      // Step 2: Charge Payment
      await this.updateSagaStatus(saga.id, SagaStatus.PROCESSING, OrderSagaSteps.CHARGE_PAYMENT);
      const payment = await this.paymentClient.chargePayment({
        sagaId: saga.id,
        orderId: saga.payload.orderId,
        customerId: saga.payload.customerId,
        amount: saga.payload.totalAmount,
      });
      await this.recordStepCompletion(saga.id, OrderSagaSteps.CHARGE_PAYMENT, {
        transactionId: payment.transactionId,
      });

      // Step 3: Confirm Order
      await this.updateSagaStatus(saga.id, SagaStatus.PROCESSING, OrderSagaSteps.CONFIRM_ORDER);
      await this.orderClient.confirmOrder({
        sagaId: saga.id,
        orderId: saga.payload.orderId,
      });
      await this.recordStepCompletion(saga.id, OrderSagaSteps.CONFIRM_ORDER, {});

      // Saga completed successfully
      await this.updateSagaStatus(saga.id, SagaStatus.COMPLETED, null);
      this.logger.log(`Saga completed: ${saga.id}`);

    } catch (error) {
      this.logger.error(`Saga step failed: ${error.message}, starting compensation`);
      await this.compensateSaga(saga.id, error.message);
    }
  }

  private async compensateSaga(sagaId: string, reason: string): Promise<void> {
    const saga = await this.sagaRepository.findOne({ where: { id: sagaId } });

    await this.updateSagaStatus(sagaId, SagaStatus.COMPENSATING, null);

    // Compensate in reverse order
    const completedSteps = [...saga.completedSteps].reverse();

    for (const step of completedSteps) {
      try {
        switch (step.name) {
          case OrderSagaSteps.CONFIRM_ORDER:
            await this.orderClient.cancelOrder({
              sagaId,
              orderId: saga.payload.orderId,
              reason,
            });
            break;

          case OrderSagaSteps.CHARGE_PAYMENT:
            await this.paymentClient.refundPayment({
              sagaId,
              transactionId: step.result.transactionId,
              amount: saga.payload.totalAmount,
            });
            break;

          case OrderSagaSteps.RESERVE_INVENTORY:
            await this.inventoryClient.releaseInventory({
              sagaId,
              reservationId: step.result.reservationId,
              items: saga.payload.items,
            });
            break;
        }

        await this.recordCompensationStepCompletion(sagaId, step.name);
      } catch (compensationError) {
        this.logger.error(`Compensation failed for step ${step.name}: ${compensationError.message}`);
        // Log for manual intervention
        await this.markSagaForManualIntervention(sagaId, step.name, compensationError.message);
        throw compensationError;
      }
    }

    await this.updateSagaStatus(sagaId, SagaStatus.FAILED, null);
    this.logger.log(`Saga compensation completed: ${sagaId}`);
  }

  private async updateSagaStatus(
    sagaId: string,
    status: SagaStatus,
    currentStep: string | null,
  ): Promise<void> {
    await this.sagaRepository.update(sagaId, {
      status,
      currentStep,
      updatedAt: new Date(),
    });
  }

  private async recordStepCompletion(
    sagaId: string,
    stepName: string,
    result: any,
  ): Promise<void> {
    const saga = await this.sagaRepository.findOne({ where: { id: sagaId } });
    await this.sagaRepository.update(sagaId, {
      completedSteps: [
        ...saga.completedSteps,
        { name: stepName, result, completedAt: new Date() },
      ],
    });
  }

  private async recordCompensationStepCompletion(
    sagaId: string,
    stepName: string,
  ): Promise<void> {
    const saga = await this.sagaRepository.findOne({ where: { id: sagaId } });
    await this.sagaRepository.update(sagaId, {
      compensatingSteps: [
        ...saga.compensatingSteps,
        { name: stepName, completedAt: new Date() },
      ],
    });
  }

  private async markSagaForManualIntervention(
    sagaId: string,
    failedStep: string,
    error: string,
  ): Promise<void> {
    await this.sagaRepository.update(sagaId, {
      status: SagaStatus.MANUAL_INTERVENTION_REQUIRED,
      failureReason: `Compensation failed at step ${failedStep}: ${error}`,
    });
  }
}
```

---

## 6. Saga State Machine

### 6.1 สร้าง Schema PostgreSQL

```sql
-- migrations/001_create_saga_tables.sql

CREATE TYPE saga_status AS ENUM (
  'STARTED',
  'PROCESSING', 
  'COMPLETED',
  'COMPENSATING',
  'FAILED',
  'MANUAL_INTERVENTION_REQUIRED'
);

CREATE TABLE saga_instances (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type VARCHAR(100) NOT NULL,
  status saga_status NOT NULL DEFAULT 'STARTED',
  payload JSONB NOT NULL,
  current_step VARCHAR(100),
  completed_steps JSONB NOT NULL DEFAULT '[]',
  compensating_steps JSONB NOT NULL DEFAULT '[]',
  failure_reason TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at TIMESTAMPTZ,
  
  CONSTRAINT chk_completed_steps CHECK (jsonb_typeof(completed_steps) = 'array'),
  CONSTRAINT chk_compensating_steps CHECK (jsonb_typeof(compensating_steps) = 'array')
);

CREATE INDEX idx_saga_instances_status ON saga_instances(status);
CREATE INDEX idx_saga_instances_type ON saga_instances(type);
CREATE INDEX idx_saga_instances_created_at ON saga_instances(created_at DESC);

-- Saga log for audit trail
CREATE TABLE saga_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  saga_id UUID NOT NULL REFERENCES saga_instances(id),
  event_type VARCHAR(100) NOT NULL,
  step_name VARCHAR(100),
  payload JSONB,
  error TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_saga_events_saga_id ON saga_events(saga_id);
CREATE INDEX idx_saga_events_created_at ON saga_events(created_at DESC);
```

### 6.2 Saga Entity ใน TypeORM

```typescript
// saga-orchestrator/src/entities/saga.entity.ts
import {
  Entity,
  Column,
  PrimaryColumn,
  CreateDateColumn,
  UpdateDateColumn,
} from 'typeorm';

export enum SagaStatus {
  STARTED = 'STARTED',
  PROCESSING = 'PROCESSING',
  COMPLETED = 'COMPLETED',
  COMPENSATING = 'COMPENSATING',
  FAILED = 'FAILED',
  MANUAL_INTERVENTION_REQUIRED = 'MANUAL_INTERVENTION_REQUIRED',
}

export interface SagaStep {
  name: string;
  result: any;
  completedAt: Date;
}

@Entity('saga_instances')
export class SagaInstance {
  @PrimaryColumn('uuid')
  id: string;

  @Column({ length: 100 })
  type: string;

  @Column({
    type: 'enum',
    enum: SagaStatus,
    default: SagaStatus.STARTED,
  })
  status: SagaStatus;

  @Column({ type: 'jsonb' })
  payload: any;

  @Column({ name: 'current_step', nullable: true })
  currentStep: string;

  @Column({ name: 'completed_steps', type: 'jsonb', default: [] })
  completedSteps: SagaStep[];

  @Column({ name: 'compensating_steps', type: 'jsonb', default: [] })
  compensatingSteps: SagaStep[];

  @Column({ name: 'failure_reason', nullable: true })
  failureReason: string;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;

  @Column({ name: 'completed_at', nullable: true })
  completedAt: Date;
}
```

---

## 7. Idempotency Keys

### 7.1 ทำไมต้องใช้ Idempotency Keys

ใน Distributed System Network อาจ Timeout หรือ Fail ก็ได้ ทำให้เกิดการ Retry อยู่บ่อยครั้ง ถ้าไม่มี Idempotency การ Retry อาจทำให้เกิด Duplicate Operations เช่น เรียกเก็บเงินซ้ำ

```
Client → Service: "Charge $100"  (request times out)
Client → Service: "Charge $100"  (retry)  ← charged twice!
```

### 7.2 Idempotency Key Implementation

```typescript
// shared/src/idempotency/idempotency.service.ts
import { Injectable } from '@nestjs/common';
import { InjectDataSource } from '@nestjs/typeorm';
import { DataSource } from 'typeorm';
import crypto from 'crypto';

export interface IdempotencyRecord {
  key: string;
  status: 'PROCESSING' | 'COMPLETED' | 'FAILED';
  response?: any;
  createdAt: Date;
  expiresAt: Date;
}

@Injectable()
export class IdempotencyService {
  constructor(
    @InjectDataSource()
    private dataSource: DataSource,
  ) {}

  async executeIdempotent<T>(
    idempotencyKey: string,
    operation: () => Promise<T>,
    ttlHours = 24,
  ): Promise<{ result: T; isNew: boolean }> {
    // Check if already processed
    const existing = await this.dataSource.query(
      `SELECT * FROM idempotency_records WHERE key = $1 AND expires_at > NOW()`,
      [idempotencyKey],
    );

    if (existing.length > 0) {
      const record = existing[0];

      if (record.status === 'COMPLETED') {
        return { result: record.response, isNew: false };
      }

      if (record.status === 'PROCESSING') {
        throw new Error(`Operation with key ${idempotencyKey} is already in progress`);
      }

      if (record.status === 'FAILED') {
        // Allow retry for failed operations
        await this.dataSource.query(
          `UPDATE idempotency_records SET status = 'PROCESSING', updated_at = NOW() WHERE key = $1`,
          [idempotencyKey],
        );
      }
    } else {
      // Insert new record
      await this.dataSource.query(
        `INSERT INTO idempotency_records (key, status, expires_at, created_at)
         VALUES ($1, 'PROCESSING', NOW() + $2 * INTERVAL '1 hour', NOW())
         ON CONFLICT (key) DO NOTHING`,
        [idempotencyKey, ttlHours],
      );
    }

    try {
      const result = await operation();

      await this.dataSource.query(
        `UPDATE idempotency_records 
         SET status = 'COMPLETED', response = $1, updated_at = NOW() 
         WHERE key = $2`,
        [JSON.stringify(result), idempotencyKey],
      );

      return { result, isNew: true };
    } catch (error) {
      await this.dataSource.query(
        `UPDATE idempotency_records 
         SET status = 'FAILED', error = $1, updated_at = NOW() 
         WHERE key = $2`,
        [error.message, idempotencyKey],
      );
      throw error;
    }
  }

  generateKey(prefix: string, ...parts: string[]): string {
    const data = [prefix, ...parts].join(':');
    return crypto.createHash('sha256').update(data).digest('hex');
  }
}
```

```sql
-- migrations/002_create_idempotency_table.sql
CREATE TABLE idempotency_records (
  key VARCHAR(255) PRIMARY KEY,
  status VARCHAR(20) NOT NULL CHECK (status IN ('PROCESSING', 'COMPLETED', 'FAILED')),
  response JSONB,
  error TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_idempotency_expires_at ON idempotency_records(expires_at);
```

### 7.3 ใช้ Idempotency Key ใน Payment Service

```typescript
// payment-service/src/services/payment.service.ts
@Injectable()
export class PaymentService {
  constructor(
    private idempotencyService: IdempotencyService,
    private stripeService: StripeService,
    private paymentRepository: PaymentRepository,
  ) {}

  async chargePayment(dto: ChargePaymentDto): Promise<ChargePaymentResult> {
    // Generate idempotency key based on saga + order
    const idempotencyKey = this.idempotencyService.generateKey(
      'charge',
      dto.sagaId,
      dto.orderId,
    );

    const { result } = await this.idempotencyService.executeIdempotent(
      idempotencyKey,
      async () => {
        // Call Stripe with its own idempotency key
        const charge = await this.stripeService.charges.create(
          {
            amount: Math.round(dto.amount * 100), // in cents
            currency: 'thb',
            customer: dto.stripeCustomerId,
            metadata: {
              sagaId: dto.sagaId,
              orderId: dto.orderId,
            },
          },
          {
            idempotencyKey: `charge-${dto.sagaId}-${dto.orderId}`,
          },
        );

        // Store in our DB
        const payment = await this.paymentRepository.save({
          sagaId: dto.sagaId,
          orderId: dto.orderId,
          transactionId: charge.id,
          amount: dto.amount,
          status: 'COMPLETED',
        });

        return {
          transactionId: charge.id,
          status: 'COMPLETED',
        };
      },
    );

    return result;
  }
}
```

---

## 8. Saga Implementation ด้วย BullMQ

### 8.1 ทำไมใช้ BullMQ

BullMQ เป็น Job Queue ที่ Built on Redis ช่วยให้:
- **Durable**: Jobs ไม่หายแม้ Service Crash
- **Retry**: Automatic retry ด้วย Exponential Backoff
- **Visibility**: ดู Queue State ได้ผ่าน Bull Board
- **Rate Limiting**: จำกัดจำนวน Concurrent Jobs

### 8.2 BullMQ Setup

```typescript
// saga-orchestrator/src/queues/saga.queue.ts
import { Queue, Worker, QueueEvents } from 'bullmq';
import IORedis from 'ioredis';

const connection = new IORedis({
  host: process.env.REDIS_HOST,
  port: parseInt(process.env.REDIS_PORT),
  maxRetriesPerRequest: null,
});

// Saga execution queue
export const sagaQueue = new Queue('saga-execution', {
  connection,
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 1000,
    },
    removeOnComplete: false, // Keep completed jobs for audit
    removeOnFail: false,     // Keep failed jobs for debugging
  },
});

// Saga step queues
export const inventoryQueue = new Queue('inventory-operations', {
  connection,
  defaultJobOptions: {
    attempts: 5,
    backoff: { type: 'exponential', delay: 2000 },
  },
});

export const paymentQueue = new Queue('payment-operations', {
  connection,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 1000 },
  },
});

export const orderQueue = new Queue('order-operations', {
  connection,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 1000 },
  },
});
```

### 8.3 Saga Worker ด้วย BullMQ

```typescript
// saga-orchestrator/src/workers/saga.worker.ts
import { Worker, Job } from 'bullmq';
import { Logger } from '@nestjs/common';
import { OrderSagaOrchestrator } from '../sagas/order-saga.orchestrator';

const logger = new Logger('SagaWorker');

export function createSagaWorker(orchestrator: OrderSagaOrchestrator) {
  const worker = new Worker(
    'saga-execution',
    async (job: Job) => {
      logger.log(`Processing saga job: ${job.id}, type: ${job.name}`);

      switch (job.name) {
        case 'start-order-saga':
          return orchestrator.startOrderSaga(job.data);

        case 'compensate-saga':
          return orchestrator.compensateSaga(job.data.sagaId, job.data.reason);

        default:
          throw new Error(`Unknown saga job type: ${job.name}`);
      }
    },
    {
      connection,
      concurrency: 10,
      limiter: {
        max: 100,
        duration: 1000,
      },
    },
  );

  worker.on('completed', (job) => {
    logger.log(`Saga job completed: ${job.id}`);
  });

  worker.on('failed', (job, error) => {
    logger.error(`Saga job failed: ${job.id}, error: ${error.message}`);
  });

  worker.on('stalled', (jobId) => {
    logger.warn(`Saga job stalled: ${jobId}`);
  });

  return worker;
}
```

### 8.4 ตัวอย่าง API Controller

```typescript
// order-service/src/controllers/order.controller.ts
import { Controller, Post, Body, Get, Param, Headers } from '@nestjs/common';
import { OrderSagaOrchestrator } from '../sagas/order-saga.orchestrator';
import { CreateOrderDto } from '../dto/create-order.dto';

@Controller('orders')
export class OrderController {
  constructor(private sagaOrchestrator: OrderSagaOrchestrator) {}

  @Post()
  async createOrder(
    @Body() dto: CreateOrderDto,
    @Headers('X-Idempotency-Key') idempotencyKey: string,
  ) {
    const sagaInstance = await this.sagaOrchestrator.startOrderSaga({
      orderId: dto.orderId || uuidv4(),
      customerId: dto.customerId,
      items: dto.items,
      totalAmount: dto.totalAmount,
    });

    return {
      sagaId: sagaInstance.id,
      orderId: dto.orderId,
      status: sagaInstance.status,
    };
  }

  @Get('saga/:sagaId')
  async getSagaStatus(@Param('sagaId') sagaId: string) {
    const saga = await this.sagaOrchestrator.getSagaStatus(sagaId);
    return saga;
  }
}
```

---

## 9. Saga Log: Audit Trail

### 9.1 Design

```typescript
// saga-orchestrator/src/services/saga-log.service.ts
import { Injectable } from '@nestjs/common';
import { InjectDataSource } from '@nestjs/typeorm';
import { DataSource } from 'typeorm';

export enum SagaEventType {
  SAGA_STARTED = 'SAGA_STARTED',
  STEP_STARTED = 'STEP_STARTED',
  STEP_COMPLETED = 'STEP_COMPLETED',
  STEP_FAILED = 'STEP_FAILED',
  COMPENSATION_STARTED = 'COMPENSATION_STARTED',
  COMPENSATION_STEP_COMPLETED = 'COMPENSATION_STEP_COMPLETED',
  COMPENSATION_STEP_FAILED = 'COMPENSATION_STEP_FAILED',
  SAGA_COMPLETED = 'SAGA_COMPLETED',
  SAGA_FAILED = 'SAGA_FAILED',
  MANUAL_INTERVENTION_REQUIRED = 'MANUAL_INTERVENTION_REQUIRED',
}

@Injectable()
export class SagaLogService {
  constructor(
    @InjectDataSource()
    private dataSource: DataSource,
  ) {}

  async log(
    sagaId: string,
    eventType: SagaEventType,
    stepName?: string,
    payload?: any,
    error?: string,
  ): Promise<void> {
    await this.dataSource.query(
      `INSERT INTO saga_events (saga_id, event_type, step_name, payload, error, created_at)
       VALUES ($1, $2, $3, $4, $5, NOW())`,
      [sagaId, eventType, stepName, JSON.stringify(payload), error],
    );
  }

  async getSagaHistory(sagaId: string): Promise<any[]> {
    return this.dataSource.query(
      `SELECT * FROM saga_events WHERE saga_id = $1 ORDER BY created_at ASC`,
      [sagaId],
    );
  }

  async getFailedSagas(since: Date): Promise<any[]> {
    return this.dataSource.query(
      `SELECT 
         si.*,
         COUNT(se.id) as event_count,
         MAX(se.created_at) as last_event_at
       FROM saga_instances si
       LEFT JOIN saga_events se ON si.id = se.saga_id
       WHERE si.status IN ('FAILED', 'MANUAL_INTERVENTION_REQUIRED')
         AND si.created_at >= $1
       GROUP BY si.id
       ORDER BY si.created_at DESC`,
      [since],
    );
  }
}
```

---

## 10. Testing Sagas

### 10.1 Happy Path Test

```typescript
// tests/order-saga.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { OrderSagaOrchestrator } from '../src/sagas/order-saga.orchestrator';
import { InventoryClient } from '../src/clients/inventory.client';
import { PaymentClient } from '../src/clients/payment.client';
import { OrderClient } from '../src/clients/order.client';

describe('OrderSaga', () => {
  let orchestrator: OrderSagaOrchestrator;
  let inventoryClient: jest.Mocked<InventoryClient>;
  let paymentClient: jest.Mocked<PaymentClient>;
  let orderClient: jest.Mocked<OrderClient>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        OrderSagaOrchestrator,
        {
          provide: InventoryClient,
          useValue: {
            reserveInventory: jest.fn(),
            releaseInventory: jest.fn(),
          },
        },
        {
          provide: PaymentClient,
          useValue: {
            chargePayment: jest.fn(),
            refundPayment: jest.fn(),
          },
        },
        {
          provide: OrderClient,
          useValue: {
            confirmOrder: jest.fn(),
            cancelOrder: jest.fn(),
          },
        },
      ],
    }).compile();

    orchestrator = module.get<OrderSagaOrchestrator>(OrderSagaOrchestrator);
    inventoryClient = module.get(InventoryClient);
    paymentClient = module.get(PaymentClient);
    orderClient = module.get(OrderClient);
  });

  describe('Happy Path', () => {
    it('should complete order saga successfully', async () => {
      // Arrange
      inventoryClient.reserveInventory.mockResolvedValue({
        reservationId: 'res-123',
      });
      paymentClient.chargePayment.mockResolvedValue({
        transactionId: 'txn-456',
      });
      orderClient.confirmOrder.mockResolvedValue({ success: true });

      // Act
      const saga = await orchestrator.startOrderSaga({
        orderId: 'order-001',
        customerId: 'customer-001',
        items: [{ productId: 'prod-A', quantity: 2, price: 100 }],
        totalAmount: 200,
      });

      // Wait for saga to complete
      await new Promise(resolve => setTimeout(resolve, 100));
      const finalSaga = await orchestrator.getSagaStatus(saga.id);

      // Assert
      expect(finalSaga.status).toBe('COMPLETED');
      expect(inventoryClient.reserveInventory).toHaveBeenCalledTimes(1);
      expect(paymentClient.chargePayment).toHaveBeenCalledTimes(1);
      expect(orderClient.confirmOrder).toHaveBeenCalledTimes(1);
    });
  });

  describe('Failure Scenarios', () => {
    it('should compensate when payment fails', async () => {
      // Arrange
      inventoryClient.reserveInventory.mockResolvedValue({
        reservationId: 'res-123',
      });
      paymentClient.chargePayment.mockRejectedValue(
        new Error('Payment declined'),
      );
      inventoryClient.releaseInventory.mockResolvedValue({ success: true });

      // Act
      const saga = await orchestrator.startOrderSaga({
        orderId: 'order-002',
        customerId: 'customer-001',
        items: [{ productId: 'prod-A', quantity: 2, price: 100 }],
        totalAmount: 200,
      });

      await new Promise(resolve => setTimeout(resolve, 200));
      const finalSaga = await orchestrator.getSagaStatus(saga.id);

      // Assert
      expect(finalSaga.status).toBe('FAILED');
      expect(inventoryClient.releaseInventory).toHaveBeenCalledTimes(1);
      expect(inventoryClient.releaseInventory).toHaveBeenCalledWith(
        expect.objectContaining({ reservationId: 'res-123' }),
      );
    });

    it('should handle inventory shortage', async () => {
      // Arrange
      inventoryClient.reserveInventory.mockRejectedValue(
        new Error('Insufficient inventory'),
      );

      // Act
      const saga = await orchestrator.startOrderSaga({
        orderId: 'order-003',
        customerId: 'customer-001',
        items: [{ productId: 'prod-B', quantity: 100, price: 50 }],
        totalAmount: 5000,
      });

      await new Promise(resolve => setTimeout(resolve, 100));
      const finalSaga = await orchestrator.getSagaStatus(saga.id);

      // Assert
      expect(finalSaga.status).toBe('FAILED');
      expect(paymentClient.chargePayment).not.toHaveBeenCalled();
      expect(orderClient.confirmOrder).not.toHaveBeenCalled();
    });

    it('should handle compensation failure and mark for manual intervention', async () => {
      // Arrange
      inventoryClient.reserveInventory.mockResolvedValue({ reservationId: 'res-123' });
      paymentClient.chargePayment.mockRejectedValue(new Error('Payment failed'));
      inventoryClient.releaseInventory.mockRejectedValue(new Error('Release failed'));

      // Act
      const saga = await orchestrator.startOrderSaga({
        orderId: 'order-004',
        customerId: 'customer-001',
        items: [],
        totalAmount: 0,
      });

      await new Promise(resolve => setTimeout(resolve, 200));
      const finalSaga = await orchestrator.getSagaStatus(saga.id);

      // Assert
      expect(finalSaga.status).toBe('MANUAL_INTERVENTION_REQUIRED');
    });
  });
});
```

### 10.2 Integration Test ด้วย Docker

```yaml
# docker-compose.test.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: saga_test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    ports:
      - "5433:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6380:6379"

  inventory-service:
    build:
      context: ./inventory-service
      dockerfile: Dockerfile.test
    environment:
      DATABASE_URL: postgresql://test:test@postgres:5432/saga_test
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis

  payment-service:
    build:
      context: ./payment-service
      dockerfile: Dockerfile.test
    environment:
      DATABASE_URL: postgresql://test:test@postgres:5432/saga_test
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis

  order-service:
    build:
      context: ./order-service
      dockerfile: Dockerfile.test
    environment:
      DATABASE_URL: postgresql://test:test@postgres:5432/saga_test
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis

  saga-orchestrator:
    build:
      context: ./saga-orchestrator
      dockerfile: Dockerfile.test
    environment:
      DATABASE_URL: postgresql://test:test@postgres:5432/saga_test
      REDIS_URL: redis://redis:6379
      INVENTORY_SERVICE_URL: http://inventory-service:3001
      PAYMENT_SERVICE_URL: http://payment-service:3002
      ORDER_SERVICE_URL: http://order-service:3003
    depends_on:
      - postgres
      - redis
      - inventory-service
      - payment-service
      - order-service
    ports:
      - "3000:3000"
```

---

## 11. Monitoring และ Observability

### 11.1 Prometheus Metrics

```typescript
// saga-orchestrator/src/metrics/saga.metrics.ts
import { Counter, Histogram, Gauge } from 'prom-client';

export const sagaStartedTotal = new Counter({
  name: 'saga_started_total',
  help: 'Total number of sagas started',
  labelNames: ['type'],
});

export const sagaCompletedTotal = new Counter({
  name: 'saga_completed_total',
  help: 'Total number of sagas completed',
  labelNames: ['type', 'status'],
});

export const sagaDurationSeconds = new Histogram({
  name: 'saga_duration_seconds',
  help: 'Duration of saga execution in seconds',
  labelNames: ['type', 'status'],
  buckets: [0.1, 0.5, 1, 5, 10, 30, 60, 300],
});

export const sagasInProgress = new Gauge({
  name: 'sagas_in_progress',
  help: 'Number of sagas currently in progress',
  labelNames: ['type'],
});

export const sagaStepDuration = new Histogram({
  name: 'saga_step_duration_seconds',
  help: 'Duration of each saga step',
  labelNames: ['type', 'step', 'status'],
  buckets: [0.05, 0.1, 0.5, 1, 5, 10],
});
```

### 11.2 Dashboard Queries

```sql
-- Query: Saga success rate ใน 1 ชั่วโมงที่ผ่านมา
SELECT 
  type,
  COUNT(*) as total,
  SUM(CASE WHEN status = 'COMPLETED' THEN 1 ELSE 0 END) as completed,
  SUM(CASE WHEN status = 'FAILED' THEN 1 ELSE 0 END) as failed,
  SUM(CASE WHEN status = 'MANUAL_INTERVENTION_REQUIRED' THEN 1 ELSE 0 END) as manual,
  ROUND(
    SUM(CASE WHEN status = 'COMPLETED' THEN 1 ELSE 0 END)::numeric / COUNT(*) * 100, 2
  ) as success_rate_pct
FROM saga_instances
WHERE created_at >= NOW() - INTERVAL '1 hour'
GROUP BY type
ORDER BY type;

-- Query: Average saga duration by type
SELECT
  type,
  status,
  COUNT(*) as count,
  AVG(EXTRACT(EPOCH FROM (completed_at - created_at))) as avg_duration_sec,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (completed_at - created_at))) as p95_duration_sec
FROM saga_instances
WHERE completed_at IS NOT NULL
  AND created_at >= NOW() - INTERVAL '24 hours'
GROUP BY type, status;

-- Query: Pending sagas stuck > 5 minutes
SELECT 
  id,
  type,
  current_step,
  created_at,
  EXTRACT(EPOCH FROM (NOW() - updated_at)) / 60 as minutes_since_update
FROM saga_instances
WHERE status IN ('STARTED', 'PROCESSING', 'COMPENSATING')
  AND updated_at < NOW() - INTERVAL '5 minutes'
ORDER BY minutes_since_update DESC;
```

---

## 12. Docker Compose ครบชุด

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: microservices
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  saga-orchestrator:
    build: ./saga-orchestrator
    environment:
      DATABASE_URL: postgresql://postgres:postgres123@postgres:5432/microservices
      REDIS_URL: redis://redis:6379
      INVENTORY_SERVICE_URL: http://inventory-service:3001
      PAYMENT_SERVICE_URL: http://payment-service:3002
      ORDER_SERVICE_URL: http://order-service:3003
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    ports:
      - "3000:3000"
    restart: unless-stopped

  inventory-service:
    build: ./inventory-service
    environment:
      DATABASE_URL: postgresql://postgres:postgres123@postgres:5432/microservices
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "3001:3001"

  payment-service:
    build: ./payment-service
    environment:
      DATABASE_URL: postgresql://postgres:postgres123@postgres:5432/microservices
      REDIS_URL: redis://redis:6379
      STRIPE_SECRET_KEY: ${STRIPE_SECRET_KEY}
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "3002:3002"

  order-service:
    build: ./order-service
    environment:
      DATABASE_URL: postgresql://postgres:postgres123@postgres:5432/microservices
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "3003:3003"

  bull-board:
    image: deadly0/bull-board
    environment:
      REDIS_HOST: redis
      REDIS_PORT: 6379
    depends_on:
      - redis
    ports:
      - "3010:3000"

volumes:
  postgres_data:
  redis_data:
```

---

## 13. สรุป

### 13.1 เมื่อไหร่ใช้ Choreography

- Services น้อย (2-4 Services)
- Business Process ง่าย เส้นตรง
- Teams ต้องการ Autonomy สูง
- ไม่ต้องการ Central Visibility

### 13.2 เมื่อไหร่ใช้ Orchestration

- Services เยอะ (5+ Services)
- Business Process ซับซ้อน มีเงื่อนไข
- ต้องการ Centralized Monitoring
- ต้องการ Retry Logic ที่ชัดเจน

### 13.3 Best Practices

1. **Idempotency is mandatory**: ทุก Service operation ต้องเป็น Idempotent
2. **Design for failure**: คิดเสมอว่า Compensation จะทำงานอย่างไร
3. **Log everything**: Saga Events ต้องมี Audit Trail สมบูรณ์
4. **Monitor stuck sagas**: มี Alert สำหรับ Sagas ที่ค้างนานผิดปกติ
5. **Manual intervention plan**: มี Runbook สำหรับ Failed Compensation
6. **Test compensation paths**: ทดสอบ Failure Scenarios อย่างสม่ำเสมอ

### 13.4 Saga vs 2PC

| Feature | 2PC | Saga |
|---------|-----|------|
| Consistency | Strong (ACID) | Eventual |
| Performance | ช้า (Blocking) | เร็ว (Non-blocking) |
| Scalability | ต่ำ | สูง |
| Complexity | ง่าย (Transaction) | ซับซ้อน (Compensation) |
| Failure Handling | Lock หรือ Abort | Compensate |
| Use Case | Single DB | Multi-service |

Saga Pattern เป็น Foundation สำคัญของ Microservices Architecture ที่ต้องการ Data Consistency ข้าม Service Boundaries โดยยอมรับ Eventual Consistency แลกกับ Availability และ Scalability ที่สูงกว่า
