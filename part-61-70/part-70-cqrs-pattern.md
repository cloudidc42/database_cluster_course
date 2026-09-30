# Part 70: CQRS (Command Query Responsibility Segregation)

## บทนำ

CQRS (Command Query Responsibility Segregation) เป็น architectural pattern ที่แยก "การเขียนข้อมูล" (Commands) ออกจาก "การอ่านข้อมูล" (Queries) อย่างชัดเจน ซึ่งตรงกันข้ามกับ traditional approach ที่ใช้ model เดียวกันสำหรับทั้ง read และ write

Pattern นี้ถูกนำเสนอโดย Greg Young และ Udi Dahan ซึ่งได้รับแรงบันดาลใจจาก Command Query Separation (CQS) principle ของ Bertrand Meyer

---

## 1. Traditional Approach: ปัญหาของ Model เดียว

### 1.1 Single Model Problem

```
Traditional CRUD Architecture:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Application
    │
    ▼
Service Layer (OrderService, UserService)
    │
    ▼
Repository (OrderRepository, UserRepository)
    │
    ▼
Database (PostgreSQL Primary)
    ↑
    │ (Same model for everything)
    ├── CREATE order
    ├── UPDATE order status
    ├── DELETE order
    ├── GET order by id
    ├── GET orders by customer
    ├── GET orders for dashboard (aggregated)
    └── GET orders for reporting

ปัญหา:
1. Read model ≠ Write model (forced to use same)
   - Write: validate business rules, normalize
   - Read: denormalize, join, aggregate
   
2. Schema ถูก compromise:
   - Write needs: normalized, referential integrity
   - Read needs: denormalized, fast JOIN, aggregated views
   
3. Scale ลำบาก:
   - Write ต้องการ ACID
   - Read ต้องการ throughput สูง, low latency
   - Cannot scale independently
```

### 1.2 ตัวอย่างปัญหา

```typescript
// Traditional: OrderService รับผิดชอบทั้ง read และ write
class OrderService {
    // Write operations
    async createOrder(data: CreateOrderDTO): Promise<Order> { ... }
    async updateOrderStatus(id: string, status: string): Promise<Order> { ... }
    async cancelOrder(id: string): Promise<void> { ... }
    
    // Read operations (ใช้ model เดิม!)
    async getOrder(id: string): Promise<Order> { ... }
    async getOrdersByCustomer(customerId: string): Promise<Order[]> { ... }
    async getDashboardStats(): Promise<DashboardStats> { ... }  // ← complex aggregation บน write model
    
    // ปัญหา: getDashboardStats ต้อง JOIN หลาย tables
    // บน Primary DB ที่รับ write อยู่ด้วย
    // → ทำให้ Primary ช้าลง
}
```

---

## 2. CQRS Benefits

```
CQRS Benefits:
━━━━━━━━━━━━━━

1. Optimize reads and writes independently:
   Write side: normalized schema, ACID, business rules
   Read side: denormalized, materialized views, optimized queries

2. Scale independently:
   Write: vertical scale Primary DB
   Read: horizontal scale (เพิ่ม replicas, Redis, Elasticsearch)

3. Different schema for reads:
   Write: orders + order_items + customers (normalized)
   Read: orders_view (denormalized single table with all data)

4. Different technology for reads:
   Complex queries → Elasticsearch
   Session data → Redis
   Analytics → ClickHouse/BigQuery
   Simple lookups → PostgreSQL Replica
```

---

## 3. CQRS Architecture

### 3.1 Overview

```
CQRS Architecture:
━━━━━━━━━━━━━━━━━━

                         Application
                              │
              ┌───────────────┴───────────────┐
              │                               │
         Commands                          Queries
    (Create, Update, Delete)          (Read, List, Search)
              │                               │
              ▼                               ▼
       Command Bus                       Query Bus
       (Dispatch to handlers)            (Dispatch to handlers)
              │                               │
              ▼                               ▼
    Command Handlers               Query Handlers
    (Business Logic)               (Data retrieval)
              │                               │
              ▼                               ▼
    PostgreSQL Primary             PostgreSQL Replica
    (Write Store)                  Redis Cache
                                   Elasticsearch
                                   (Read Stores)
              │
     Domain Events emitted
              │
    ┌─────────┴─────────┐
    ▼                   ▼
Event Bus          Event Handlers
                  (Update Read Stores)
```

### 3.2 Implementation Levels

```
Level 1: Same DB, separate code paths
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

CommandHandler → PostgreSQL Primary
QueryHandler   → PostgreSQL Primary (same DB!)

ข้อดี: เริ่มต้นง่าย, คิด structure ดีขึ้น
ข้อเสีย: ยังใช้ DB เดียวกัน, ยังไม่ scale

Level 2: Read Replica
━━━━━━━━━━━━━━━━━━━━━━

CommandHandler → PostgreSQL Primary
QueryHandler   → PostgreSQL Replica

ข้อดี: Read scale ดีขึ้น
ข้อเสีย: Eventual consistency, replica lag

Level 3: Separate Read Store
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

CommandHandler → PostgreSQL Primary
QueryHandler   → Redis / Elasticsearch / Denormalized PostgreSQL

ข้อดี: Read สามารถ optimize ได้อย่างเต็มที่
ข้อเสีย: Sync complexity, eventual consistency

Level 4: Full Event Sourcing + CQRS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

CommandHandler → Event Store (Append events)
Projections    → Build Read Models จาก events
QueryHandler   → Query Read Models

ข้อดี: Maximum flexibility, complete audit trail
ข้อเสีย: Most complex to implement and operate
```

---

## 4. TypeScript Implementation

### 4.1 Base Types และ Interfaces

```typescript
// ════════════════════════════════════════════════
// Base Types: Commands, Queries, Results
// ════════════════════════════════════════════════

// Command: สิ่งที่ต้องการให้ระบบทำ
interface Command {
    readonly commandId: string;
    readonly commandType: string;
    readonly metadata: CommandMetadata;
    readonly issuedAt: Date;
}

interface CommandMetadata {
    userId: string;
    correlationId: string;
    ipAddress?: string;
    userAgent?: string;
}

// Query: สิ่งที่ต้องการข้อมูล
interface Query {
    readonly queryId: string;
    readonly queryType: string;
    readonly requestedBy?: string;
}

// Result types
interface CommandResult<T = void> {
    success: boolean;
    data?: T;
    error?: string;
    errorCode?: string;
}

interface QueryResult<T> {
    data: T;
    metadata?: {
        total?: number;
        page?: number;
        pageSize?: number;
        executionTimeMs?: number;
    };
}

// ════════════════════════════════════════════════
// E-commerce Order Commands
// ════════════════════════════════════════════════

interface PlaceOrderCommand extends Command {
    commandType: 'PlaceOrder';
    payload: {
        customerId: string;
        items: Array<{
            productId: string;
            quantity: number;
        }>;
        shippingAddress: ShippingAddress;
        paymentMethod: PaymentMethod;
    };
}

interface UpdateOrderStatusCommand extends Command {
    commandType: 'UpdateOrderStatus';
    payload: {
        orderId: string;
        newStatus: OrderStatus;
        reason?: string;
    };
}

interface CancelOrderCommand extends Command {
    commandType: 'CancelOrder';
    payload: {
        orderId: string;
        reason: string;
    };
}

// ════════════════════════════════════════════════
// E-commerce Order Queries
// ════════════════════════════════════════════════

interface GetOrderByIdQuery extends Query {
    queryType: 'GetOrderById';
    orderId: string;
}

interface GetOrdersByCustomerQuery extends Query {
    queryType: 'GetOrdersByCustomer';
    customerId: string;
    status?: OrderStatus;
    page: number;
    pageSize: number;
}

interface GetOrderDashboardQuery extends Query {
    queryType: 'GetOrderDashboard';
    dateFrom: Date;
    dateTo: Date;
    groupBy: 'day' | 'week' | 'month';
}

interface SearchOrdersQuery extends Query {
    queryType: 'SearchOrders';
    searchText: string;
    filters?: {
        status?: OrderStatus[];
        minAmount?: number;
        maxAmount?: number;
        dateFrom?: Date;
        dateTo?: Date;
    };
    page: number;
    pageSize: number;
}

// Supporting types
type OrderStatus = 'pending' | 'confirmed' | 'paid' | 'shipped' | 'delivered' | 'cancelled';

interface ShippingAddress {
    street: string;
    city: string;
    province: string;
    postalCode: string;
    country: string;
}

interface PaymentMethod {
    type: 'credit_card' | 'bank_transfer' | 'cod';
    details?: Record<string, any>;
}
```

### 4.2 Command Bus

```typescript
// ════════════════════════════════════════════════
// Command Bus: Dispatch commands to handlers
// ════════════════════════════════════════════════

type CommandHandler<TCommand extends Command, TResult = void> = {
    handle(command: TCommand): Promise<CommandResult<TResult>>;
};

class CommandBus {
    private handlers: Map<string, CommandHandler<any, any>> = new Map();
    private middlewares: CommandMiddleware[] = [];
    
    register<TCommand extends Command, TResult = void>(
        commandType: string,
        handler: CommandHandler<TCommand, TResult>
    ): void {
        if (this.handlers.has(commandType)) {
            throw new Error(`Handler already registered for command: ${commandType}`);
        }
        this.handlers.set(commandType, handler);
    }
    
    use(middleware: CommandMiddleware): void {
        this.middlewares.push(middleware);
    }
    
    async dispatch<TCommand extends Command, TResult = void>(
        command: TCommand
    ): Promise<CommandResult<TResult>> {
        const handler = this.handlers.get(command.commandType);
        
        if (!handler) {
            throw new Error(`No handler registered for command: ${command.commandType}`);
        }
        
        // Apply middleware chain
        let index = 0;
        const next = async (): Promise<CommandResult<TResult>> => {
            if (index < this.middlewares.length) {
                const middleware = this.middlewares[index++];
                return middleware.execute(command, next);
            }
            return handler.handle(command);
        };
        
        return next();
    }
}

interface CommandMiddleware {
    execute<T extends Command>(
        command: T, 
        next: () => Promise<CommandResult<any>>
    ): Promise<CommandResult<any>>;
}

// ════════════════════════════════════════════════
// Middleware: Logging, Validation, Transaction
// ════════════════════════════════════════════════

class LoggingMiddleware implements CommandMiddleware {
    async execute<T extends Command>(
        command: T,
        next: () => Promise<CommandResult<any>>
    ): Promise<CommandResult<any>> {
        const start = Date.now();
        console.log(`[Command] ${command.commandType} started`, {
            commandId: command.commandId,
            userId: command.metadata.userId,
            correlationId: command.metadata.correlationId
        });
        
        try {
            const result = await next();
            const duration = Date.now() - start;
            
            console.log(`[Command] ${command.commandType} ${result.success ? 'succeeded' : 'failed'}`, {
                commandId: command.commandId,
                duration: `${duration}ms`,
                error: result.error
            });
            
            return result;
        } catch (error) {
            const duration = Date.now() - start;
            console.error(`[Command] ${command.commandType} threw exception`, {
                commandId: command.commandId,
                duration: `${duration}ms`,
                error
            });
            throw error;
        }
    }
}

class TransactionMiddleware implements CommandMiddleware {
    constructor(private db: Pool) {}
    
    async execute<T extends Command>(
        command: T,
        next: () => Promise<CommandResult<any>>
    ): Promise<CommandResult<any>> {
        const client = await this.db.connect();
        
        try {
            await client.query('BEGIN');
            
            // Set transaction context สำหรับ audit trail
            await client.query(
                `SET LOCAL app.current_user_id = '${command.metadata.userId}'`
            );
            await client.query(
                `SET LOCAL app.correlation_id = '${command.metadata.correlationId}'`
            );
            
            const result = await next();
            
            if (result.success) {
                await client.query('COMMIT');
            } else {
                await client.query('ROLLBACK');
            }
            
            return result;
        } catch (error) {
            await client.query('ROLLBACK');
            throw error;
        } finally {
            client.release();
        }
    }
}
```

### 4.3 Query Bus

```typescript
// ════════════════════════════════════════════════
// Query Bus: Dispatch queries to handlers
// ════════════════════════════════════════════════

type QueryHandler<TQuery extends Query, TResult> = {
    handle(query: TQuery): Promise<QueryResult<TResult>>;
};

class QueryBus {
    private handlers: Map<string, QueryHandler<any, any>> = new Map();
    private cacheEnabled: boolean;
    private cache: Map<string, { result: any; expiresAt: number }> = new Map();
    
    constructor(cacheEnabled: boolean = true) {
        this.cacheEnabled = cacheEnabled;
    }
    
    register<TQuery extends Query, TResult>(
        queryType: string,
        handler: QueryHandler<TQuery, TResult>
    ): void {
        this.handlers.set(queryType, handler);
    }
    
    async dispatch<TQuery extends Query, TResult>(
        query: TQuery,
        cacheOptions?: { ttlSeconds?: number; cacheKey?: string }
    ): Promise<QueryResult<TResult>> {
        const handler = this.handlers.get(query.queryType);
        
        if (!handler) {
            throw new Error(`No handler registered for query: ${query.queryType}`);
        }
        
        // Check cache
        if (this.cacheEnabled && cacheOptions) {
            const cacheKey = cacheOptions.cacheKey || `${query.queryType}:${JSON.stringify(query)}`;
            const cached = this.cache.get(cacheKey);
            
            if (cached && cached.expiresAt > Date.now()) {
                return cached.result;
            }
            
            const result = await handler.handle(query);
            
            if (cacheOptions.ttlSeconds) {
                this.cache.set(cacheKey, {
                    result,
                    expiresAt: Date.now() + cacheOptions.ttlSeconds * 1000
                });
            }
            
            return result;
        }
        
        return handler.handle(query);
    }
}
```

### 4.4 Command Handlers

```typescript
// ════════════════════════════════════════════════
// Command Handler: PlaceOrder
// ════════════════════════════════════════════════

class PlaceOrderCommandHandler implements CommandHandler<PlaceOrderCommand, { orderId: string }> {
    constructor(
        private db: Pool,
        private inventoryService: InventoryService,
        private pricingService: PricingService,
        private eventBus: EventBus
    ) {}
    
    async handle(command: PlaceOrderCommand): Promise<CommandResult<{ orderId: string }>> {
        const { customerId, items, shippingAddress, paymentMethod } = command.payload;
        
        try {
            // 1. Validate customer exists
            const customer = await this.db.query(
                'SELECT * FROM customers WHERE customer_id = $1 AND is_active = true',
                [customerId]
            );
            
            if (customer.rows.length === 0) {
                return { success: false, errorCode: 'CUSTOMER_NOT_FOUND', error: 'Customer not found or inactive' };
            }
            
            // 2. Get product details and check inventory
            const productIds = items.map(i => i.productId);
            const products = await this.db.query(
                `SELECT product_id, name, price, stock_quantity 
                 FROM products WHERE product_id = ANY($1)`,
                [productIds]
            );
            
            if (products.rows.length !== productIds.length) {
                return { success: false, errorCode: 'PRODUCT_NOT_FOUND', error: 'One or more products not found' };
            }
            
            // Check stock
            const productMap = new Map(products.rows.map(p => [p.product_id, p]));
            for (const item of items) {
                const product = productMap.get(item.productId);
                if (!product || product.stock_quantity < item.quantity) {
                    return { 
                        success: false, 
                        errorCode: 'INSUFFICIENT_STOCK',
                        error: `Insufficient stock for product ${item.productId}` 
                    };
                }
            }
            
            // 3. Calculate total
            const orderItems = items.map(item => {
                const product = productMap.get(item.productId)!;
                return {
                    productId: item.productId,
                    productName: product.name,
                    quantity: item.quantity,
                    unitPrice: product.price,
                    subtotal: product.price * item.quantity
                };
            });
            
            const totalAmount = orderItems.reduce((sum, item) => sum + item.subtotal, 0);
            
            // 4. Create order (WRITE to Primary DB)
            const orderId = crypto.randomUUID();
            
            await this.db.query(
                `INSERT INTO orders 
                 (order_id, customer_id, status, total_amount, shipping_address, 
                  payment_method, created_at, updated_at)
                 VALUES ($1, $2, 'pending', $3, $4, $5, NOW(), NOW())`,
                [orderId, customerId, totalAmount, 
                 JSON.stringify(shippingAddress), JSON.stringify(paymentMethod)]
            );
            
            for (const item of orderItems) {
                await this.db.query(
                    `INSERT INTO order_items 
                     (order_id, product_id, quantity, unit_price, subtotal)
                     VALUES ($1, $2, $3, $4, $5)`,
                    [orderId, item.productId, item.quantity, item.unitPrice, item.subtotal]
                );
                
                // Reserve stock
                await this.db.query(
                    `UPDATE products SET stock_quantity = stock_quantity - $1 
                     WHERE product_id = $2`,
                    [item.quantity, item.productId]
                );
            }
            
            // 5. Publish domain event (triggers read model updates)
            await this.eventBus.publish({
                eventType: 'OrderPlaced',
                aggregateType: 'Order',
                aggregateId: orderId,
                eventData: {
                    orderId,
                    customerId,
                    items: orderItems,
                    totalAmount,
                    shippingAddress
                },
                metadata: { correlationId: command.metadata.correlationId }
            });
            
            return { success: true, data: { orderId } };
            
        } catch (error) {
            console.error('PlaceOrderCommand failed:', error);
            return { 
                success: false, 
                error: 'Failed to place order',
                errorCode: 'INTERNAL_ERROR'
            };
        }
    }
}

// ════════════════════════════════════════════════
// Command Handler: UpdateOrderStatus
// ════════════════════════════════════════════════

class UpdateOrderStatusCommandHandler implements CommandHandler<UpdateOrderStatusCommand> {
    private validTransitions: Map<OrderStatus, OrderStatus[]> = new Map([
        ['pending', ['confirmed', 'cancelled']],
        ['confirmed', ['paid', 'cancelled']],
        ['paid', ['shipped', 'cancelled']],
        ['shipped', ['delivered']],
        ['delivered', []],
        ['cancelled', []]
    ]);
    
    constructor(
        private db: Pool,
        private eventBus: EventBus
    ) {}
    
    async handle(command: UpdateOrderStatusCommand): Promise<CommandResult> {
        const { orderId, newStatus, reason } = command.payload;
        
        const order = await this.db.query(
            'SELECT * FROM orders WHERE order_id = $1 FOR UPDATE',
            [orderId]
        );
        
        if (order.rows.length === 0) {
            return { success: false, error: 'Order not found', errorCode: 'ORDER_NOT_FOUND' };
        }
        
        const currentStatus = order.rows[0].status as OrderStatus;
        const allowedTransitions = this.validTransitions.get(currentStatus) || [];
        
        if (!allowedTransitions.includes(newStatus)) {
            return {
                success: false,
                error: `Cannot transition from ${currentStatus} to ${newStatus}`,
                errorCode: 'INVALID_STATUS_TRANSITION'
            };
        }
        
        await this.db.query(
            `UPDATE orders SET status = $1, updated_at = NOW() WHERE order_id = $2`,
            [newStatus, orderId]
        );
        
        await this.eventBus.publish({
            eventType: 'OrderStatusUpdated',
            aggregateType: 'Order',
            aggregateId: orderId,
            eventData: {
                orderId,
                previousStatus: currentStatus,
                newStatus,
                reason,
                updatedBy: command.metadata.userId
            },
            metadata: { correlationId: command.metadata.correlationId }
        });
        
        return { success: true };
    }
}
```

### 4.5 Query Handlers

```typescript
// ════════════════════════════════════════════════
// Query Handlers: Reading from Read Models
// ════════════════════════════════════════════════

// OrderDetail Read Model
interface OrderDetailView {
    orderId: string;
    orderNumber: string;
    customerId: string;
    customerName: string;
    customerEmail: string;
    status: OrderStatus;
    items: Array<{
        productId: string;
        productName: string;
        quantity: number;
        unitPrice: number;
        subtotal: number;
        imageUrl?: string;
    }>;
    totalAmount: number;
    shippingAddress: ShippingAddress;
    paymentMethod: string;
    createdAt: Date;
    updatedAt: Date;
    statusHistory: Array<{
        status: OrderStatus;
        changedAt: Date;
        changedBy: string;
    }>;
}

// Level 1: Query from Primary DB (simple but slow)
class GetOrderByIdQueryHandlerL1 implements QueryHandler<GetOrderByIdQuery, OrderDetailView | null> {
    constructor(private db: Pool) {}
    
    async handle(query: GetOrderByIdQuery): Promise<QueryResult<OrderDetailView | null>> {
        const start = Date.now();
        
        const result = await this.db.query(
            `SELECT 
                o.order_id, o.status, o.total_amount, o.shipping_address,
                o.payment_method, o.created_at, o.updated_at,
                c.customer_id, c.name as customer_name, c.email as customer_email,
                oi.product_id, oi.quantity, oi.unit_price, oi.subtotal,
                p.name as product_name
             FROM orders o
             JOIN customers c ON c.customer_id = o.customer_id
             JOIN order_items oi ON oi.order_id = o.order_id
             JOIN products p ON p.product_id = oi.product_id
             WHERE o.order_id = $1`,
            [query.orderId]
        );
        
        if (result.rows.length === 0) {
            return { data: null, metadata: { executionTimeMs: Date.now() - start } };
        }
        
        // Map to view
        const firstRow = result.rows[0];
        const view: OrderDetailView = {
            orderId: firstRow.order_id,
            orderNumber: `ORD-${firstRow.order_id.slice(0, 8).toUpperCase()}`,
            customerId: firstRow.customer_id,
            customerName: firstRow.customer_name,
            customerEmail: firstRow.customer_email,
            status: firstRow.status,
            items: result.rows.map(row => ({
                productId: row.product_id,
                productName: row.product_name,
                quantity: row.quantity,
                unitPrice: parseFloat(row.unit_price),
                subtotal: parseFloat(row.subtotal)
            })),
            totalAmount: parseFloat(firstRow.total_amount),
            shippingAddress: firstRow.shipping_address,
            paymentMethod: firstRow.payment_method?.type || 'unknown',
            createdAt: firstRow.created_at,
            updatedAt: firstRow.updated_at,
            statusHistory: []
        };
        
        return { data: view, metadata: { executionTimeMs: Date.now() - start } };
    }
}

// Level 2/3: Query from Denormalized Read Model
class GetOrderByIdQueryHandlerL3 implements QueryHandler<GetOrderByIdQuery, OrderDetailView | null> {
    constructor(
        private readDb: Pool,         // Read replica or read-optimized DB
        private redisCache?: RedisClient
    ) {}
    
    async handle(query: GetOrderByIdQuery): Promise<QueryResult<OrderDetailView | null>> {
        const start = Date.now();
        
        // Check Redis cache first
        if (this.redisCache) {
            const cached = await this.redisCache.get(`order:${query.orderId}`);
            if (cached) {
                return { 
                    data: JSON.parse(cached),
                    metadata: { executionTimeMs: Date.now() - start }
                };
            }
        }
        
        // Query from denormalized orders_view
        const result = await this.readDb.query(
            `SELECT 
                order_id, order_number, customer_id, customer_name, customer_email,
                status, items_json, total_amount, shipping_address_json,
                payment_method_type, created_at, updated_at, status_history_json
             FROM orders_denormalized_view
             WHERE order_id = $1`,
            [query.orderId]
        );
        
        if (result.rows.length === 0) {
            return { data: null, metadata: { executionTimeMs: Date.now() - start } };
        }
        
        const row = result.rows[0];
        const view: OrderDetailView = {
            orderId: row.order_id,
            orderNumber: row.order_number,
            customerId: row.customer_id,
            customerName: row.customer_name,
            customerEmail: row.customer_email,
            status: row.status,
            items: row.items_json,
            totalAmount: parseFloat(row.total_amount),
            shippingAddress: row.shipping_address_json,
            paymentMethod: row.payment_method_type,
            createdAt: row.created_at,
            updatedAt: row.updated_at,
            statusHistory: row.status_history_json || []
        };
        
        // Cache for 60 seconds
        if (this.redisCache) {
            await this.redisCache.setex(`order:${query.orderId}`, 60, JSON.stringify(view));
        }
        
        return { data: view, metadata: { executionTimeMs: Date.now() - start } };
    }
}

// ════════════════════════════════════════════════
// Dashboard Query Handler: Aggregated Data
// ════════════════════════════════════════════════

interface DashboardStats {
    periodLabel: string;
    totalOrders: number;
    totalRevenue: number;
    averageOrderValue: number;
    ordersByStatus: Record<OrderStatus, number>;
    topProducts: Array<{ productId: string; productName: string; quantitySold: number; revenue: number }>;
    revenueByDay: Array<{ date: string; revenue: number; orderCount: number }>;
}

class GetOrderDashboardQueryHandler implements QueryHandler<GetOrderDashboardQuery, DashboardStats> {
    constructor(
        private analyticsDb: Pool,  // Could be separate analytics DB
        private cache: RedisClient
    ) {}
    
    async handle(query: GetOrderDashboardQuery): Promise<QueryResult<DashboardStats>> {
        const start = Date.now();
        
        const cacheKey = `dashboard:${query.dateFrom.toISOString()}:${query.dateTo.toISOString()}:${query.groupBy}`;
        
        // Cache for 5 minutes (dashboard data doesn't need to be real-time)
        const cached = await this.cache.get(cacheKey);
        if (cached) {
            return { data: JSON.parse(cached), metadata: { executionTimeMs: Date.now() - start } };
        }
        
        // Query analytics (denormalized) table
        const [summaryResult, statusResult, productsResult, revenueByDayResult] = await Promise.all([
            this.analyticsDb.query(
                `SELECT 
                    COUNT(*) as total_orders,
                    SUM(total_amount) as total_revenue,
                    AVG(total_amount) as avg_order_value
                 FROM orders_analytics
                 WHERE created_at BETWEEN $1 AND $2
                   AND status = 'delivered'`,
                [query.dateFrom, query.dateTo]
            ),
            
            this.analyticsDb.query(
                `SELECT status, COUNT(*) as count
                 FROM orders_analytics
                 WHERE created_at BETWEEN $1 AND $2
                 GROUP BY status`,
                [query.dateFrom, query.dateTo]
            ),
            
            this.analyticsDb.query(
                `SELECT 
                    product_id,
                    product_name,
                    SUM(quantity) as quantity_sold,
                    SUM(subtotal) as revenue
                 FROM order_items_analytics
                 WHERE created_at BETWEEN $1 AND $2
                 GROUP BY product_id, product_name
                 ORDER BY revenue DESC
                 LIMIT 10`,
                [query.dateFrom, query.dateTo]
            ),
            
            this.analyticsDb.query(
                `SELECT 
                    DATE_TRUNC($1, created_at) as period,
                    SUM(total_amount) as revenue,
                    COUNT(*) as order_count
                 FROM orders_analytics
                 WHERE created_at BETWEEN $2 AND $3
                   AND status = 'delivered'
                 GROUP BY DATE_TRUNC($1, created_at)
                 ORDER BY period`,
                [query.groupBy, query.dateFrom, query.dateTo]
            )
        ]);
        
        const summary = summaryResult.rows[0];
        const ordersByStatus = Object.fromEntries(
            statusResult.rows.map(r => [r.status, parseInt(r.count)])
        ) as Record<OrderStatus, number>;
        
        const stats: DashboardStats = {
            periodLabel: `${query.dateFrom.toLocaleDateString()} - ${query.dateTo.toLocaleDateString()}`,
            totalOrders: parseInt(summary.total_orders),
            totalRevenue: parseFloat(summary.total_revenue || '0'),
            averageOrderValue: parseFloat(summary.avg_order_value || '0'),
            ordersByStatus,
            topProducts: productsResult.rows.map(r => ({
                productId: r.product_id,
                productName: r.product_name,
                quantitySold: parseInt(r.quantity_sold),
                revenue: parseFloat(r.revenue)
            })),
            revenueByDay: revenueByDayResult.rows.map(r => ({
                date: r.period,
                revenue: parseFloat(r.revenue),
                orderCount: parseInt(r.order_count)
            }))
        };
        
        // Cache for 5 minutes
        await this.cache.setex(cacheKey, 300, JSON.stringify(stats));
        
        return { data: stats, metadata: { executionTimeMs: Date.now() - start } };
    }
}
```

---

## 5. Sync Strategies: Write → Read Side

### 5.1 Database Replication

```
Strategy 1: Read Replica (Streaming Replication)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Write Side                              Read Side
PostgreSQL Primary                      PostgreSQL Replica
     │                                       │
     │ INSERT INTO orders                    │ (receives streaming WAL)
     │                                       │
     └─── WAL Stream ──────────────────────► │
                                             │
                                      (lag: <100ms typically)

ข้อดี: Simple, automatic, strong consistency (with synchronous_commit)
ข้อเสีย: Replica อ่านได้แต่เป็น row-for-row copy, ยัง normalize เหมือน primary
```

### 5.2 Materialized Views

```sql
-- ════════════════════════════════════════════════
-- Materialized Views: Denormalized Read Models
-- ════════════════════════════════════════════════

-- Create denormalized materialized view
CREATE MATERIALIZED VIEW orders_denormalized_view AS
SELECT 
    o.order_id,
    'ORD-' || UPPER(SUBSTRING(o.order_id::text, 1, 8)) as order_number,
    o.customer_id,
    c.name as customer_name,
    c.email as customer_email,
    o.status,
    o.total_amount,
    o.shipping_address as shipping_address_json,
    o.payment_method->>'type' as payment_method_type,
    o.created_at,
    o.updated_at,
    
    -- JSON aggregation ของ items (denormalized!)
    jsonb_agg(jsonb_build_object(
        'productId', oi.product_id,
        'productName', p.name,
        'quantity', oi.quantity,
        'unitPrice', oi.unit_price,
        'subtotal', oi.subtotal
    )) as items_json,
    
    -- Computed fields
    COUNT(oi.item_id) as item_count

FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p ON p.product_id = oi.product_id
GROUP BY 
    o.order_id, o.customer_id, c.name, c.email,
    o.status, o.total_amount, o.shipping_address,
    o.payment_method, o.created_at, o.updated_at;

-- Index บน materialized view
CREATE UNIQUE INDEX idx_mv_orders_id ON orders_denormalized_view(order_id);
CREATE INDEX idx_mv_orders_customer ON orders_denormalized_view(customer_id, created_at DESC);
CREATE INDEX idx_mv_orders_status ON orders_denormalized_view(status, created_at DESC);

-- Refresh materialized view
-- Option 1: Full refresh (ช้า, lock table)
REFRESH MATERIALIZED VIEW orders_denormalized_view;

-- Option 2: Concurrent refresh (ไม่ block reads, ต้องมี UNIQUE index)
REFRESH MATERIALIZED VIEW CONCURRENTLY orders_denormalized_view;

-- Automated refresh ด้วย pg_cron
SELECT cron.schedule(
    'refresh-orders-view',
    '*/5 * * * *',  -- ทุก 5 นาที
    $$REFRESH MATERIALIZED VIEW CONCURRENTLY orders_denormalized_view$$
);
```

### 5.3 Change Data Capture (CDC) + Event Bus

```typescript
// ════════════════════════════════════════════════
// CDC: อัพเดท Read Models เมื่อ Write Side เปลี่ยน
// ════════════════════════════════════════════════

// Event Bus Interface
interface EventBus {
    publish(event: DomainEvent): Promise<void>;
    subscribe(eventType: string, handler: EventHandler): void;
}

type EventHandler = (event: DomainEvent) => Promise<void>;

class InMemoryEventBus implements EventBus {
    private handlers: Map<string, EventHandler[]> = new Map();
    
    subscribe(eventType: string, handler: EventHandler): void {
        const handlers = this.handlers.get(eventType) || [];
        handlers.push(handler);
        this.handlers.set(eventType, handlers);
    }
    
    async publish(event: DomainEvent): Promise<void> {
        const handlers = this.handlers.get(event.eventType) || [];
        const wildcardHandlers = this.handlers.get('*') || [];
        
        await Promise.all([...handlers, ...wildcardHandlers].map(h => h(event)));
    }
}

// ════════════════════════════════════════════════
// Read Model Updaters (Event Handlers)
// ════════════════════════════════════════════════

class OrderReadModelUpdater {
    constructor(
        private readDb: Pool,
        private cache: RedisClient,
        private searchIndex: ElasticsearchClient
    ) {}
    
    registerHandlers(eventBus: EventBus): void {
        eventBus.subscribe('OrderPlaced', this.onOrderPlaced.bind(this));
        eventBus.subscribe('OrderStatusUpdated', this.onOrderStatusUpdated.bind(this));
    }
    
    private async onOrderPlaced(event: DomainEvent): Promise<void> {
        const { orderId, customerId, items, totalAmount, shippingAddress } = event.eventData;
        
        // 1. Update PostgreSQL read model (denormalized)
        await this.readDb.query(
            `INSERT INTO orders_read_model 
             (order_id, customer_id, status, items_json, total_amount, 
              shipping_address_json, created_at)
             VALUES ($1, $2, 'pending', $3, $4, $5, $6)`,
            [orderId, customerId, JSON.stringify(items), totalAmount, 
             JSON.stringify(shippingAddress), event.createdAt]
        );
        
        // 2. Invalidate cache
        await this.cache.del(`customer:orders:${customerId}`);
        await this.cache.del(`order:${orderId}`);
        
        // 3. Index in Elasticsearch for search
        await this.searchIndex.index({
            index: 'orders',
            id: orderId,
            document: {
                orderId,
                customerId,
                status: 'pending',
                totalAmount,
                items: items.map((item: any) => item.productName).join(' '),
                createdAt: event.createdAt
            }
        });
    }
    
    private async onOrderStatusUpdated(event: DomainEvent): Promise<void> {
        const { orderId, newStatus, updatedBy } = event.eventData;
        
        // 1. Update read model
        await this.readDb.query(
            `UPDATE orders_read_model 
             SET status = $1, updated_at = $2
             WHERE order_id = $3`,
            [newStatus, event.createdAt, orderId]
        );
        
        // 2. Invalidate cache
        await this.cache.del(`order:${orderId}`);
        
        // 3. Update search index
        await this.searchIndex.update({
            index: 'orders',
            id: orderId,
            doc: { status: newStatus, updatedAt: event.createdAt }
        });
    }
}
```

---

## 6. Sync Strategies: ตัวเลือกต่างๆ

### 6.1 ตารางเปรียบเทียบ

```
Sync Strategy Comparison:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Strategy            | Latency | Complexity | Consistency | Use Case
─────────────────────────────────────────────────────────────────────
Sync Replication    | <1ms    | Low        | Strong      | Financial
Async Replication   | <100ms  | Low        | Eventual    | General
Materialized Views  | Minutes | Medium     | Eventual    | Analytics
Domain Events       | <1s     | High       | Eventual    | Complex domain
CDC (Debezium)      | <1s     | Medium     | Eventual    | Integration
Manual Projection   | Custom  | High       | Custom      | Full CQRS
```

---

## 7. Elasticsearch Integration สำหรับ Search

```typescript
// ════════════════════════════════════════════════
// Search Query Handler: Elasticsearch
// ════════════════════════════════════════════════

import { Client as ESClient } from '@elastic/elasticsearch';

interface OrderSearchResult {
    orderId: string;
    orderNumber: string;
    customerName: string;
    status: OrderStatus;
    totalAmount: number;
    createdAt: Date;
    highlights?: string[];
}

class SearchOrdersQueryHandler implements QueryHandler<SearchOrdersQuery, OrderSearchResult[]> {
    constructor(private esClient: ESClient) {}
    
    async handle(query: SearchOrdersQuery): Promise<QueryResult<OrderSearchResult[]>> {
        const start = Date.now();
        
        const searchQuery: any = {
            index: 'orders',
            body: {
                from: (query.page - 1) * query.pageSize,
                size: query.pageSize,
                query: {
                    bool: {
                        must: [
                            {
                                multi_match: {
                                    query: query.searchText,
                                    fields: ['customerName^2', 'orderId', 'items', 'notes'],
                                    type: 'best_fields',
                                    fuzziness: 'AUTO'
                                }
                            }
                        ],
                        filter: []
                    }
                },
                highlight: {
                    fields: {
                        customerName: {},
                        items: {}
                    }
                },
                sort: [{ createdAt: { order: 'desc' } }]
            }
        };
        
        // Add filters
        if (query.filters?.status) {
            searchQuery.body.query.bool.filter.push({
                terms: { status: query.filters.status }
            });
        }
        
        if (query.filters?.minAmount || query.filters?.maxAmount) {
            searchQuery.body.query.bool.filter.push({
                range: {
                    totalAmount: {
                        gte: query.filters.minAmount,
                        lte: query.filters.maxAmount
                    }
                }
            });
        }
        
        if (query.filters?.dateFrom || query.filters?.dateTo) {
            searchQuery.body.query.bool.filter.push({
                range: {
                    createdAt: {
                        gte: query.filters.dateFrom?.toISOString(),
                        lte: query.filters.dateTo?.toISOString()
                    }
                }
            });
        }
        
        const response = await this.esClient.search(searchQuery);
        const hits = response.hits.hits;
        
        const results: OrderSearchResult[] = hits.map((hit: any) => ({
            orderId: hit._source.orderId,
            orderNumber: hit._source.orderNumber,
            customerName: hit._source.customerName,
            status: hit._source.status,
            totalAmount: hit._source.totalAmount,
            createdAt: new Date(hit._source.createdAt),
            highlights: hit.highlight ? [
                ...(hit.highlight.customerName || []),
                ...(hit.highlight.items || [])
            ] : undefined
        }));
        
        return {
            data: results,
            metadata: {
                total: typeof response.hits.total === 'number' 
                    ? response.hits.total 
                    : response.hits.total?.value || 0,
                page: query.page,
                pageSize: query.pageSize,
                executionTimeMs: Date.now() - start
            }
        };
    }
}
```

---

## 8. Putting It All Together

### 8.1 Application Bootstrap

```typescript
// ════════════════════════════════════════════════
// Application Setup: Wire everything together
// ════════════════════════════════════════════════

async function bootstrapApplication() {
    // Database connections
    const writeDb = new Pool({ connectionString: process.env.DATABASE_WRITE_URL });
    const readDb = new Pool({ connectionString: process.env.DATABASE_READ_URL });
    const analyticsDb = new Pool({ connectionString: process.env.DATABASE_ANALYTICS_URL });
    
    // Cache
    const redis = createClient({ url: process.env.REDIS_URL });
    await redis.connect();
    
    // Search
    const esClient = new ESClient({ node: process.env.ELASTICSEARCH_URL });
    
    // Event Bus
    const eventBus = new InMemoryEventBus();
    
    // ════════════════════════════════════════════════
    // Setup Command Side
    // ════════════════════════════════════════════════
    const commandBus = new CommandBus();
    commandBus.use(new LoggingMiddleware());
    commandBus.use(new TransactionMiddleware(writeDb));
    
    commandBus.register('PlaceOrder', new PlaceOrderCommandHandler(
        writeDb, 
        new InventoryService(writeDb),
        new PricingService(writeDb),
        eventBus
    ));
    
    commandBus.register('UpdateOrderStatus', new UpdateOrderStatusCommandHandler(
        writeDb, eventBus
    ));
    
    commandBus.register('CancelOrder', new CancelOrderCommandHandler(
        writeDb, eventBus
    ));
    
    // ════════════════════════════════════════════════
    // Setup Read Side
    // ════════════════════════════════════════════════
    const queryBus = new QueryBus(true);  // enable caching
    
    queryBus.register('GetOrderById', new GetOrderByIdQueryHandlerL3(readDb, redis));
    queryBus.register('GetOrdersByCustomer', new GetOrdersByCustomerQueryHandler(readDb, redis));
    queryBus.register('GetOrderDashboard', new GetOrderDashboardQueryHandler(analyticsDb, redis));
    queryBus.register('SearchOrders', new SearchOrdersQueryHandler(esClient));
    
    // ════════════════════════════════════════════════
    // Setup Read Model Updaters
    // ════════════════════════════════════════════════
    const readModelUpdater = new OrderReadModelUpdater(readDb, redis, esClient);
    readModelUpdater.registerHandlers(eventBus);
    
    // ════════════════════════════════════════════════
    // REST API Controllers
    // ════════════════════════════════════════════════
    const app = express();
    
    // Command endpoints (POST, PUT, DELETE)
    app.post('/api/orders', async (req, res) => {
        const command: PlaceOrderCommand = {
            commandId: crypto.randomUUID(),
            commandType: 'PlaceOrder',
            metadata: {
                userId: req.user?.id || 'anonymous',
                correlationId: req.headers['x-correlation-id'] as string || crypto.randomUUID(),
                ipAddress: req.ip
            },
            issuedAt: new Date(),
            payload: req.body
        };
        
        const result = await commandBus.dispatch(command);
        
        if (result.success) {
            res.status(201).json({ orderId: result.data?.orderId });
        } else {
            res.status(400).json({ error: result.error, code: result.errorCode });
        }
    });
    
    app.patch('/api/orders/:orderId/status', async (req, res) => {
        const command: UpdateOrderStatusCommand = {
            commandId: crypto.randomUUID(),
            commandType: 'UpdateOrderStatus',
            metadata: { userId: req.user?.id, correlationId: crypto.randomUUID() },
            issuedAt: new Date(),
            payload: {
                orderId: req.params.orderId,
                newStatus: req.body.status,
                reason: req.body.reason
            }
        };
        
        const result = await commandBus.dispatch(command);
        
        if (result.success) {
            res.json({ success: true });
        } else {
            res.status(400).json({ error: result.error });
        }
    });
    
    // Query endpoints (GET)
    app.get('/api/orders/:orderId', async (req, res) => {
        const query: GetOrderByIdQuery = {
            queryId: crypto.randomUUID(),
            queryType: 'GetOrderById',
            orderId: req.params.orderId
        };
        
        const result = await queryBus.dispatch(query, { 
            ttlSeconds: 60, 
            cacheKey: `order:${req.params.orderId}` 
        });
        
        if (result.data) {
            res.json(result.data);
        } else {
            res.status(404).json({ error: 'Order not found' });
        }
    });
    
    app.get('/api/orders', async (req, res) => {
        if (req.query.search) {
            // Use Elasticsearch for search
            const query: SearchOrdersQuery = {
                queryId: crypto.randomUUID(),
                queryType: 'SearchOrders',
                searchText: req.query.search as string,
                page: parseInt(req.query.page as string) || 1,
                pageSize: parseInt(req.query.pageSize as string) || 20,
                filters: {
                    status: req.query.status ? [req.query.status as OrderStatus] : undefined
                }
            };
            const result = await queryBus.dispatch(query);
            res.json({ data: result.data, ...result.metadata });
        } else if (req.query.customerId) {
            // Use PostgreSQL read model
            const query: GetOrdersByCustomerQuery = {
                queryId: crypto.randomUUID(),
                queryType: 'GetOrdersByCustomer',
                customerId: req.query.customerId as string,
                page: parseInt(req.query.page as string) || 1,
                pageSize: parseInt(req.query.pageSize as string) || 20
            };
            const result = await queryBus.dispatch(query, { ttlSeconds: 30 });
            res.json({ data: result.data, ...result.metadata });
        }
    });
    
    app.get('/api/dashboard', async (req, res) => {
        const query: GetOrderDashboardQuery = {
            queryId: crypto.randomUUID(),
            queryType: 'GetOrderDashboard',
            dateFrom: new Date(req.query.from as string || new Date().setMonth(new Date().getMonth() - 1)),
            dateTo: new Date(req.query.to as string || Date.now()),
            groupBy: (req.query.groupBy as 'day' | 'week' | 'month') || 'day'
        };
        
        const result = await queryBus.dispatch(query, { ttlSeconds: 300 });
        res.json(result.data);
    });
    
    return app;
}
```

---

## 9. Eventual Consistency และ User Experience

### 9.1 จัดการ Eventual Consistency ใน UI

```typescript
// ════════════════════════════════════════════════
// Frontend: รับมือกับ Eventual Consistency
// ════════════════════════════════════════════════

// Pattern 1: Optimistic UI Update
// อัพเดท UI ทันทีหลัง command โดยไม่รอ read side sync

async function placeOrder(orderData: OrderData) {
    // ส่ง command
    const response = await api.post('/api/orders', orderData);
    const { orderId } = response.data;
    
    // Optimistic update: แสดง order ใน UI ทันที
    const optimisticOrder = {
        orderId,
        status: 'pending',
        ...orderData,
        createdAt: new Date()
    };
    
    setOrders(prev => [optimisticOrder, ...prev]);
    
    // Poll เพื่อ verify ว่า order อยู่ใน read model แล้ว
    let retries = 0;
    while (retries < 5) {
        await sleep(1000);
        const order = await api.get(`/api/orders/${orderId}`);
        if (order.data) {
            // Read model updated, replace optimistic with real data
            setOrders(prev => prev.map(o => 
                o.orderId === orderId ? order.data : o
            ));
            break;
        }
        retries++;
    }
}

// Pattern 2: Write-then-Read with delay
async function updateOrderStatus(orderId: string, newStatus: string) {
    await api.patch(`/api/orders/${orderId}/status`, { status: newStatus });
    
    // Wait for read side to sync (simple but not elegant)
    await sleep(500);
    
    // Refresh order data
    const updated = await api.get(`/api/orders/${orderId}`);
    setOrder(updated.data);
}

// Pattern 3: Server-Sent Events (SSE) สำหรับ real-time updates
function subscribeToOrderUpdates(orderId: string) {
    const eventSource = new EventSource(`/api/orders/${orderId}/events`);
    
    eventSource.onmessage = (event) => {
        const orderUpdate = JSON.parse(event.data);
        setOrder(orderUpdate);
    };
    
    return () => eventSource.close();
}
```

---

## 10. สรุปและ Best Practices

### 10.1 CQRS Level Guide

```
เลือก CQRS Level ตามความต้องการ:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Level 1 (Separate code paths, same DB):
  เหมาะสำหรับ: เริ่มต้น, ทีมเล็ก, domain ยังไม่ซับซ้อน
  Complexity: Low
  Benefit: Code organization ดีขึ้น, ง่ายต่อ testing

Level 2 (Read Replica):
  เหมาะสำหรับ: Read-heavy workloads, ต้องการ scale reads
  Complexity: Medium
  Benefit: Read performance ดีขึ้นมาก, ลด load บน primary

Level 3 (Separate Read Store + Event Sync):
  เหมาะสำหรับ: Complex read requirements, multiple read technologies
  Complexity: High
  Benefit: Maximum read performance, flexible query

Level 4 (Event Sourcing + CQRS):
  เหมาะสำหรับ: Complex domain, audit requirements, financial systems
  Complexity: Very High
  Benefit: Complete audit trail, time travel, maximum flexibility
```

### 10.2 Anti-Patterns to Avoid

```
CQRS Anti-Patterns:
━━━━━━━━━━━━━━━━━━━

1. Query ข้อมูลผ่าน Command Handler
   ❌ CommandHandler.fetchAndReturn() - ไม่ใช่ CQRS!
   ✅ Commands emit events, Queries เป็นแยก

2. Read Model มี Business Logic
   ❌ OrderReadModel.calculateDiscount() - logic อยู่ผิดที่
   ✅ Business logic อยู่ใน Command Handlers และ Domain Objects เท่านั้น

3. Shared database without clear boundaries
   ❌ Command Handler เขียน table เดียวกับ Query Handler อ่าน
   ✅ Write Tables (normalized) vs Read Tables (denormalized) แยกกันชัดเจน

4. CQRS ทุก feature
   ❌ User registration, password reset ก็ใช้ CQRS เต็มรูปแบบ
   ✅ CQRS สำหรับ core domain ที่ซับซ้อนเท่านั้น
```

---

**สรุป Part 70: CQRS Pattern**

CQRS แยก read และ write models ออกจากกัน ช่วยให้:
- **Write side**: focus บน business rules และ consistency
- **Read side**: optimize สำหรับ query performance โดยใช้ denormalized data, cache, search engines
- **Scale**: Write และ Read scale ได้อิสระจากกัน

เริ่มจาก Level 1 (separate code paths) แล้วค่อยเพิ่ม complexity เมื่อจำเป็น ไม่ต้อง implement Level 4 ตั้งแต่ต้นถ้าไม่จำเป็น
