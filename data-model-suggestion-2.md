# Data Model Suggestion 2: Event-Sourced / CQRS Approach

## Summary

An event-sourced architecture with Command Query Responsibility Segregation (CQRS) where every state change in the dropshipping lifecycle -- order placed, inventory updated, price changed, shipment dispatched, return requested -- is stored as an immutable event in an append-only event store. Read-side projections are materialised from these events to serve dashboards, product listings, and operational queries. This approach provides a complete audit trail, enables temporal queries ("what was the inventory level at 3pm yesterday?"), and naturally supports the real-time, multi-system synchronisation that dropshipping demands.

---

## Architecture Overview

```
                    ┌─────────────┐
                    │  Storefronts │  (Shopify, WooCommerce, eBay...)
                    └──────┬──────┘
                           │ webhooks / polling
                           ▼
                    ┌─────────────┐
                    │  Command API │  (validates, emits events)
                    └──────┬──────┘
                           │
                           ▼
              ┌────────────────────────┐
              │     Event Store        │  (append-only, immutable)
              │  (EventStoreDB / Kafka │
              │   + PostgreSQL)        │
              └────────┬───────────────┘
                       │
           ┌───────────┼───────────────┐
           ▼           ▼               ▼
    ┌────────────┐ ┌──────────┐ ┌──────────────┐
    │ Order View │ │ Inventory│ │ Analytics    │
    │ Projection │ │ Projection│ │ Projection   │
    │ (Postgres) │ │ (Redis)  │ │ (ClickHouse) │
    └────────────┘ └──────────┘ └──────────────┘
```

---

## Key Entities and Event Streams

### Event Streams (Aggregates)

Each aggregate root has its own event stream, identified by a stream ID (e.g., `order-{uuid}`, `supplier-product-{uuid}`).

### Core Events

```typescript
// ── Supplier Catalogue Sync ──────────────────────────────
interface SupplierFeedPolled {
  type: 'SupplierFeedPolled';
  supplierId: string;
  tenantId: string;
  feedUrl: string;
  itemCount: number;
  timestamp: string;
}

interface SupplierProductDiscovered {
  type: 'SupplierProductDiscovered';
  supplierId: string;
  supplierSku: string;
  productData: Record<string, unknown>;  // raw supplier fields
  timestamp: string;
}

interface InventoryLevelUpdated {
  type: 'InventoryLevelUpdated';
  supplierId: string;
  supplierSku: string;
  previousQuantity: number;
  newQuantity: number;
  timestamp: string;
}

interface ProductDeactivatedDueToStockout {
  type: 'ProductDeactivatedDueToStockout';
  productId: string;
  supplierId: string;
  storefrontIds: string[];
  timestamp: string;
}

// ── Price Monitoring ─────────────────────────────────────
interface SupplierPriceChanged {
  type: 'SupplierPriceChanged';
  supplierId: string;
  supplierSku: string;
  oldPrice: number;
  newPrice: number;
  currency: string;
  changePercent: number;
  anomalyDetected: boolean;
  timestamp: string;
}

interface StorefrontPriceAdjusted {
  type: 'StorefrontPriceAdjusted';
  productId: string;
  storefrontId: string;
  oldPrice: number;
  newPrice: number;
  marginRuleApplied: string;
  timestamp: string;
}

// ── Order Lifecycle ──────────────────────────────────────
interface CustomerOrderReceived {
  type: 'CustomerOrderReceived';
  orderId: string;
  tenantId: string;
  storefrontId: string;
  externalOrderId: string;
  lineItems: Array<{
    productId: string;
    variantId?: string;
    quantity: number;
    unitPrice: number;
  }>;
  shippingAddress: Record<string, string>;
  orderTotal: number;
  currency: string;
  timestamp: string;
}

interface OrderRoutingDecisionMade {
  type: 'OrderRoutingDecisionMade';
  orderId: string;
  decisions: Array<{
    lineItemId: string;
    selectedSupplierId: string;
    reason: string;           // 'lowest_cost', 'fastest_shipping', 'in_stock', 'ai_recommended'
    supplierCost: number;
    estimatedShipDays: number;
  }>;
  timestamp: string;
}

interface SupplierPurchaseOrderCreated {
  type: 'SupplierPurchaseOrderCreated';
  purchaseOrderId: string;
  orderId: string;
  supplierId: string;
  lineItems: Array<{
    supplierSku: string;
    quantity: number;
    unitCost: number;
  }>;
  totalCost: number;
  timestamp: string;
}

interface SupplierPurchaseOrderConfirmed {
  type: 'SupplierPurchaseOrderConfirmed';
  purchaseOrderId: string;
  supplierOrderRef: string;
  estimatedShipDate: string;
  timestamp: string;
}

interface ShipmentCreated {
  type: 'ShipmentCreated';
  shipmentId: string;
  purchaseOrderId: string;
  carrier: string;
  trackingNumber: string;
  trackingUrl: string;
  timestamp: string;
}

interface TrackingEventReceived {
  type: 'TrackingEventReceived';
  shipmentId: string;
  status: string;
  location: string;
  timestamp: string;
}

interface TrackingPushedToStorefront {
  type: 'TrackingPushedToStorefront';
  shipmentId: string;
  storefrontId: string;
  externalOrderId: string;
  trackingNumber: string;
  timestamp: string;
}

// ── Returns ──────────────────────────────────────────────
interface ReturnRequested {
  type: 'ReturnRequested';
  returnId: string;
  orderId: string;
  reason: string;
  lineItems: Array<{ lineItemId: string; quantity: number }>;
  timestamp: string;
}

interface SupplierRMAIssued {
  type: 'SupplierRMAIssued';
  returnId: string;
  supplierId: string;
  rmaNumber: string;
  timestamp: string;
}

interface RefundProcessed {
  type: 'RefundProcessed';
  returnId: string;
  orderId: string;
  amount: number;
  currency: string;
  timestamp: string;
}
```

### Event Store Schema (PostgreSQL-backed)

```sql
-- Append-only event log
CREATE TABLE events (
    id              BIGSERIAL PRIMARY KEY,
    stream_id       VARCHAR(255) NOT NULL,        -- e.g. 'order-abc123'
    stream_type     VARCHAR(100) NOT NULL,         -- e.g. 'CustomerOrder'
    event_type      VARCHAR(100) NOT NULL,         -- e.g. 'CustomerOrderReceived'
    event_data      JSONB NOT NULL,
    metadata        JSONB DEFAULT '{}',            -- correlation IDs, causation IDs, user
    version         INT NOT NULL,                  -- optimistic concurrency per stream
    tenant_id       UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, version)
);

-- Global ordering index
CREATE INDEX idx_events_stream ON events(stream_id, version);
CREATE INDEX idx_events_type ON events(event_type, created_at);
CREATE INDEX idx_events_tenant ON events(tenant_id, created_at);

-- Snapshot store for long-lived aggregates
CREATE TABLE snapshots (
    stream_id       VARCHAR(255) PRIMARY KEY,
    stream_type     VARCHAR(100) NOT NULL,
    snapshot_data   JSONB NOT NULL,
    version         INT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Read-Side Projections

```sql
-- Materialised view: current order status (projected from events)
CREATE TABLE order_read_model (
    order_id            UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    storefront_id       UUID NOT NULL,
    external_order_id   VARCHAR(255) NOT NULL,
    customer_name       VARCHAR(255),
    status              VARCHAR(30) NOT NULL,
    order_total         DECIMAL(12,2),
    total_cost          DECIMAL(12,2),         -- sum of supplier costs
    margin              DECIMAL(12,2),         -- order_total - total_cost
    line_item_count     INT,
    supplier_count      INT,                   -- how many suppliers involved
    has_tracking        BOOLEAN DEFAULT false,
    ordered_at          TIMESTAMPTZ,
    last_updated_at     TIMESTAMPTZ
);

-- Materialised view: real-time inventory per supplier-product
CREATE TABLE inventory_read_model (
    product_id          UUID NOT NULL,
    supplier_id         UUID NOT NULL,
    tenant_id           UUID NOT NULL,
    supplier_sku        VARCHAR(100),
    current_quantity    INT NOT NULL DEFAULT 0,
    supplier_price      DECIMAL(12,2),
    last_price_change   TIMESTAMPTZ,
    last_sync_at        TIMESTAMPTZ,
    PRIMARY KEY (product_id, supplier_id)
);

-- Materialised view: supplier performance scorecard
CREATE TABLE supplier_performance_read_model (
    supplier_id             UUID PRIMARY KEY,
    total_orders            INT DEFAULT 0,
    on_time_delivery_pct    DECIMAL(5,2),
    avg_fulfilment_hours    DECIMAL(8,2),
    return_rate_pct         DECIMAL(5,2),
    price_stability_score   DECIMAL(3,2),      -- 0-1, higher = more stable
    last_calculated_at      TIMESTAMPTZ
);

-- Materialised view: margin analysis per product
CREATE TABLE margin_read_model (
    product_id          UUID NOT NULL,
    tenant_id           UUID NOT NULL,
    avg_sell_price      DECIMAL(12,2),
    avg_cost_price      DECIMAL(12,2),
    avg_margin_pct      DECIMAL(6,2),
    units_sold_30d      INT DEFAULT 0,
    margin_trend        VARCHAR(20),           -- 'improving', 'declining', 'stable'
    last_calculated_at  TIMESTAMPTZ,
    PRIMARY KEY (product_id, tenant_id)
);
```

### Event Processing Pipeline

```
Event Store
    │
    ├──► Order Projection Handler
    │       └── Updates order_read_model
    │
    ├──► Inventory Projection Handler
    │       └── Updates inventory_read_model (and Redis cache)
    │
    ├──► Supplier Scoring Handler
    │       └── Updates supplier_performance_read_model
    │
    ├──► Margin Analysis Handler
    │       └── Updates margin_read_model
    │
    ├──► Storefront Sync Handler
    │       └── Pushes tracking/price/stock changes to Shopify/WooCommerce/etc.
    │
    ├──► Anomaly Detection Handler
    │       └── ML model for price anomalies and stock-out prediction
    │
    └──► Notification Handler
            └── Sends alerts for margin drops, stockouts, order failures
```

---

## Pros

- **Complete audit trail**: Every action is recorded as an immutable event. For a platform handling financial transactions (orders, payments, refunds) and supplier disputes, this is invaluable. You can reconstruct the exact state of any order at any point in time.
- **Natural fit for multi-system sync**: Dropshipping inherently involves multiple external systems (suppliers, storefronts, carriers). Events map directly to integration points -- `CustomerOrderReceived` triggers supplier PO creation, `ShipmentCreated` triggers storefront tracking push, `SupplierPriceChanged` triggers repricing.
- **Temporal queries for AI/ML**: The event stream is a natural training dataset. Stock-out prediction models can replay `InventoryLevelUpdated` events to learn supplier replenishment patterns. Routing optimisation can analyse historical `OrderRoutingDecisionMade` events against delivery outcomes.
- **Independent read model scaling**: Each projection can be scaled, rebuilt, or replaced independently. The inventory read model can live in Redis for sub-millisecond lookups while the analytics projection writes to ClickHouse.
- **Resilience to schema evolution**: New event types can be added without modifying existing events. New projections can be built by replaying the event history, enabling zero-downtime feature additions.
- **Debugging and support**: When a retailer reports a missing tracking number or incorrect price, the event stream shows exactly what happened and when.

## Cons

- **Complexity overhead**: Event sourcing is significantly more complex to implement than CRUD. The team needs to understand aggregate design, event versioning, projection rebuilding, and eventual consistency.
- **Eventual consistency**: Read models are asynchronously updated from events. There is a window (typically milliseconds to seconds) where a query might return stale data. For inventory sync where overselling is the risk, this must be carefully managed with optimistic concurrency or reservation patterns.
- **Event versioning is hard**: When the schema of an event changes (e.g., adding a field to `CustomerOrderReceived`), all consumers must handle both old and new versions. Upcasting strategies add maintenance burden.
- **Projection rebuild time**: If a projection is corrupted or a new one is added, rebuilding from millions of events can take hours. Snapshots mitigate this but add their own complexity.
- **Storage growth**: Every state change creates a new event rather than updating in place. A product with hourly inventory updates generates ~8,700 events per year per supplier mapping. At scale, the event store grows rapidly.
- **Tooling maturity**: While frameworks exist (Axon, EventStoreDB, Marten), the ecosystem is less mature than traditional ORM tooling. Debugging eventual consistency issues requires specialised tooling.

---

## Technology Recommendations

| Component | Recommendation |
|---|---|
| Event store | EventStoreDB (purpose-built) or PostgreSQL with append-only tables |
| Event bus | Apache Kafka or NATS JetStream for durable event distribution |
| Command API | Node.js with NestJS CQRS module, or Java with Axon Framework |
| Read model DB | PostgreSQL for transactional projections, Redis for inventory cache |
| Analytics projection | ClickHouse or TimescaleDB for time-series analytics |
| Event serialisation | JSON with schema registry (Confluent Schema Registry or custom) |
| Projection framework | Custom handlers or Eventide (Ruby), Marten (.NET), Axon (Java) |
| Monitoring | OpenTelemetry with correlation IDs across events and commands |

---

## Migration and Scaling Considerations

- **Start with PostgreSQL as the event store**: A well-indexed `events` table in PostgreSQL handles millions of events before a dedicated event store (EventStoreDB) is needed. This reduces initial infrastructure complexity.
- **Kafka for cross-service distribution**: Use Kafka as the event backbone when multiple services need to consume events (inventory sync, order processing, analytics). Kafka's consumer groups enable independent scaling of each projection handler.
- **Snapshot strategy**: For aggregates that accumulate many events (e.g., a product with years of price and inventory updates), create snapshots every N events (e.g., every 100). This bounds aggregate rehydration time.
- **Event archival**: Partition the event store by month. Events older than the retention period can be archived to cold storage (S3/GCS) while keeping snapshots for operational rehydration.
- **Projection rebuild automation**: Build tooling to replay events and rebuild projections from scratch. This is essential for deploying new features and recovering from projection bugs. Kafka's log compaction helps here.
- **Idempotency is non-negotiable**: Supplier webhooks and storefront notifications will be delivered at-least-once. Every event handler must be idempotent, using event IDs or correlation IDs to deduplicate.
- **Gradual adoption**: Not every part of the system needs event sourcing. Start with the order lifecycle (where the audit trail is most valuable) and use traditional CRUD for simpler domains like user management and billing.
- **GDPR and data retention**: Event immutability conflicts with "right to erasure". Use crypto-shredding (encrypt PII with per-customer keys; delete the key to effectively erase the data) rather than modifying events.
