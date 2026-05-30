# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

## Summary

A fully normalized relational data model using PostgreSQL, designed around third normal form (3NF) for transactional integrity. This approach models every core domain entity -- retailers, suppliers, products, orders, shipments, price history, and channel integrations -- as discrete tables with explicit foreign key relationships. It prioritises data consistency, referential integrity, and straightforward querying for a platform that must coordinate data across multiple suppliers and storefronts simultaneously.

---

## Key Entities and Relationships

### Entity Overview

```
tenants (retailers)
  |-- tenant_storefronts (Shopify, WooCommerce, eBay, etc.)
  |-- tenant_suppliers
  |     |-- supplier_credentials
  |     |-- supplier_feed_configs
  |
  |-- products (canonical product catalogue)
  |     |-- product_variants
  |     |-- supplier_product_mappings (links products to supplier SKUs)
  |     |-- storefront_product_listings (links products to channel listings)
  |     |-- product_images
  |
  |-- customer_orders
  |     |-- order_line_items
  |     |-- supplier_purchase_orders
  |     |     |-- purchase_order_line_items
  |     |     |-- shipments
  |     |     |     |-- tracking_events
  |     |
  |     |-- order_returns
  |           |-- return_line_items
  |           |-- supplier_return_authorisations
  |
  |-- price_history
  |-- inventory_snapshots
  |-- routing_rules
  |-- margin_rules
```

### Core Schema

```sql
-- Multi-tenant retailer accounts
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    email           VARCHAR(255) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Storefront channel connections (Shopify, WooCommerce, eBay, etc.)
CREATE TABLE tenant_storefronts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    platform        VARCHAR(50) NOT NULL,  -- 'shopify', 'woocommerce', 'ebay', 'bigcommerce', 'wix'
    store_url       VARCHAR(500),
    api_key_enc     BYTEA,                 -- encrypted credentials
    api_secret_enc  BYTEA,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Supplier definitions
CREATE TABLE suppliers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    contact_email   VARCHAR(255),
    feed_type       VARCHAR(20) NOT NULL,  -- 'csv', 'xml', 'api', 'edi', 'ftp'
    reliability_score DECIMAL(3,2) DEFAULT 0.00,
    avg_ship_days   DECIMAL(4,1),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Many-to-many: which tenants use which suppliers
CREATE TABLE tenant_suppliers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    supplier_id     UUID NOT NULL REFERENCES suppliers(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    priority        INT NOT NULL DEFAULT 0,
    UNIQUE(tenant_id, supplier_id)
);

-- Supplier feed configuration per tenant-supplier relationship
CREATE TABLE supplier_feed_configs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_supplier_id  UUID NOT NULL REFERENCES tenant_suppliers(id) ON DELETE CASCADE,
    feed_url            VARCHAR(1000),
    feed_format         VARCHAR(20) NOT NULL,
    polling_interval_sec INT NOT NULL DEFAULT 3600,
    field_mappings      JSONB,              -- maps supplier fields to canonical fields
    last_polled_at      TIMESTAMPTZ,
    last_poll_status    VARCHAR(20) DEFAULT 'pending'
);

-- Canonical product catalogue (tenant-scoped)
CREATE TABLE products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    sku             VARCHAR(100) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    category        VARCHAR(255),
    base_price      DECIMAL(12,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    weight_grams    INT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, sku)
);

-- Product variants (size, colour, etc.)
CREATE TABLE product_variants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    variant_sku     VARCHAR(100) NOT NULL,
    attributes      JSONB,                -- e.g. {"size": "L", "colour": "Blue"}
    price_override  DECIMAL(12,2),
    weight_grams    INT,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

-- Maps a canonical product to a supplier's SKU and pricing
CREATE TABLE supplier_product_mappings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    supplier_id     UUID NOT NULL REFERENCES suppliers(id),
    supplier_sku    VARCHAR(100) NOT NULL,
    supplier_price  DECIMAL(12,2) NOT NULL,
    supplier_currency CHAR(3) NOT NULL DEFAULT 'USD',
    stock_quantity  INT NOT NULL DEFAULT 0,
    min_order_qty   INT DEFAULT 1,
    lead_time_days  DECIMAL(4,1),
    last_synced_at  TIMESTAMPTZ,
    UNIQUE(product_id, supplier_id)
);

-- Maps a canonical product to a storefront listing
CREATE TABLE storefront_product_listings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    storefront_id   UUID NOT NULL REFERENCES tenant_storefronts(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),         -- Shopify product ID, eBay listing ID, etc.
    listed_price    DECIMAL(12,2),
    listing_status  VARCHAR(20) NOT NULL DEFAULT 'draft',
    last_pushed_at  TIMESTAMPTZ,
    UNIQUE(product_id, storefront_id)
);

-- Customer orders ingested from storefronts
CREATE TABLE customer_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    storefront_id   UUID NOT NULL REFERENCES tenant_storefronts(id),
    external_order_id VARCHAR(255) NOT NULL,
    customer_name   VARCHAR(255),
    customer_email  VARCHAR(255),
    shipping_address JSONB NOT NULL,
    order_total     DECIMAL(12,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    ordered_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(storefront_id, external_order_id)
);

-- Line items within a customer order
CREATE TABLE order_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES customer_orders(id) ON DELETE CASCADE,
    product_id      UUID REFERENCES products(id),
    variant_id      UUID REFERENCES product_variants(id),
    quantity        INT NOT NULL,
    unit_price      DECIMAL(12,2) NOT NULL,
    line_total      DECIMAL(12,2) NOT NULL
);

-- Purchase orders sent to suppliers (one customer order may split across multiple POs)
CREATE TABLE supplier_purchase_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES customer_orders(id),
    supplier_id     UUID NOT NULL REFERENCES suppliers(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    supplier_order_ref VARCHAR(255),
    submitted_at    TIMESTAMPTZ,
    confirmed_at    TIMESTAMPTZ,
    total_cost      DECIMAL(12,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Line items within a supplier purchase order
CREATE TABLE purchase_order_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id UUID NOT NULL REFERENCES supplier_purchase_orders(id) ON DELETE CASCADE,
    order_line_item_id UUID REFERENCES order_line_items(id),
    supplier_sku    VARCHAR(100) NOT NULL,
    quantity        INT NOT NULL,
    unit_cost       DECIMAL(12,2) NOT NULL
);

-- Shipments from suppliers
CREATE TABLE shipments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id UUID NOT NULL REFERENCES supplier_purchase_orders(id),
    carrier         VARCHAR(100),
    tracking_number VARCHAR(255),
    tracking_url    VARCHAR(1000),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    shipped_at      TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    pushed_to_storefront BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Tracking event log for shipments
CREATE TABLE tracking_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id     UUID NOT NULL REFERENCES shipments(id) ON DELETE CASCADE,
    event_status    VARCHAR(100) NOT NULL,
    location        VARCHAR(255),
    event_at        TIMESTAMPTZ NOT NULL,
    raw_data        JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Supplier price change history
CREATE TABLE price_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_product_mapping_id UUID NOT NULL REFERENCES supplier_product_mappings(id),
    old_price       DECIMAL(12,2) NOT NULL,
    new_price       DECIMAL(12,2) NOT NULL,
    change_pct      DECIMAL(6,3),
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    anomaly_flag    BOOLEAN NOT NULL DEFAULT false
);

-- Inventory level snapshots for analytics and stock-out prediction
CREATE TABLE inventory_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_product_mapping_id UUID NOT NULL REFERENCES supplier_product_mappings(id),
    quantity        INT NOT NULL,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Margin and pricing rules per tenant
CREATE TABLE margin_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    product_id      UUID REFERENCES products(id),
    category        VARCHAR(255),
    rule_type       VARCHAR(30) NOT NULL,   -- 'fixed_markup', 'percentage', 'min_margin'
    rule_value      DECIMAL(12,2) NOT NULL,
    priority        INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

-- Fulfilment routing rules per tenant
CREATE TABLE routing_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    rule_name       VARCHAR(255) NOT NULL,
    conditions      JSONB NOT NULL,         -- e.g. {"country": "US", "category": "electronics"}
    preferred_supplier_id UUID REFERENCES suppliers(id),
    fallback_strategy VARCHAR(50) NOT NULL DEFAULT 'lowest_cost',
    priority        INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

-- Returns management
CREATE TABLE order_returns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES customer_orders(id),
    reason          TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'requested',
    refund_amount   DECIMAL(12,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE supplier_return_authorisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    return_id       UUID NOT NULL REFERENCES order_returns(id),
    supplier_id     UUID NOT NULL REFERENCES suppliers(id),
    rma_number      VARCHAR(100),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Key Indexes

```sql
-- High-frequency lookups
CREATE INDEX idx_products_tenant ON products(tenant_id);
CREATE INDEX idx_spm_product ON supplier_product_mappings(product_id);
CREATE INDEX idx_spm_supplier ON supplier_product_mappings(supplier_id);
CREATE INDEX idx_orders_tenant_status ON customer_orders(tenant_id, status);
CREATE INDEX idx_orders_storefront ON customer_orders(storefront_id);
CREATE INDEX idx_spo_order ON supplier_purchase_orders(order_id);
CREATE INDEX idx_shipments_po ON shipments(purchase_order_id);
CREATE INDEX idx_price_history_mapping ON price_history(supplier_product_mapping_id, detected_at);
CREATE INDEX idx_inventory_snap ON inventory_snapshots(supplier_product_mapping_id, recorded_at);
```

---

## Pros

- **Referential integrity**: Foreign keys enforce data consistency across the order lifecycle (order -> purchase order -> shipment -> tracking). In a domain where overselling and lost orders directly cost money, this is critical.
- **Well-understood tooling**: PostgreSQL is mature, with excellent support for ORMs (Prisma, Drizzle, SQLAlchemy, TypeORM), migration tools (Flyway, Alembic, Prisma Migrate), and monitoring.
- **Query flexibility**: Complex reporting queries (margin analysis, supplier performance, order routing history) are natural in SQL with joins across normalised tables.
- **ACID transactions**: Multi-table operations (e.g., creating a customer order, splitting into purchase orders, and decrementing inventory) can run in a single transaction.
- **Mature ecosystem for AI/ML**: PostgreSQL integrates with pgvector for embeddings, and normalised data exports cleanly to ML pipelines for stock-out prediction and supplier scoring.

## Cons

- **Schema rigidity**: Adding new supplier feed fields, product attributes, or storefront-specific metadata requires ALTER TABLE migrations. Different suppliers and channels have wildly varying data shapes.
- **Join overhead at scale**: Queries spanning orders, line items, purchase orders, shipments, and tracking events involve 5+ table joins. At high order volumes this can become expensive without careful indexing and query optimisation.
- **Supplier feed diversity is a poor fit**: Supplier feeds arrive in CSV, XML, EDI, and API formats with inconsistent field naming. Mapping these into rigid columns requires extensive ETL logic.
- **Inventory sync latency**: Real-time inventory updates across multiple supplier-product mappings generate high write throughput on the `supplier_product_mappings` and `inventory_snapshots` tables. Row-level locking can create contention.
- **Multi-tenant scaling ceiling**: A shared-schema multi-tenant model with `tenant_id` filters works well to ~1,000 tenants but may require partitioning or schema-per-tenant approaches beyond that.

---

## Technology Recommendations

| Component | Recommendation |
|---|---|
| Database | PostgreSQL 16+ |
| ORM | Prisma (TypeScript) or SQLAlchemy (Python) |
| Migrations | Prisma Migrate, Flyway, or Alembic |
| Connection pooling | PgBouncer or Supabase Supavisor |
| Caching | Redis for hot inventory data and session state |
| Search | PostgreSQL full-text search or Elasticsearch for product search |
| Background jobs | BullMQ (Node.js) or Celery (Python) for feed polling and order sync |
| Monitoring | pg_stat_statements, Datadog, or Grafana + Prometheus |

---

## Migration and Scaling Considerations

- **Start simple**: Begin with a single PostgreSQL instance and shared-schema multi-tenancy. This handles the first several hundred tenants comfortably.
- **Partition early for time-series data**: Use PostgreSQL native partitioning on `price_history` and `inventory_snapshots` by month. These tables grow fastest and benefit most from partition pruning.
- **Read replicas for analytics**: Route reporting and dashboard queries to read replicas to keep the primary instance responsive for transactional workloads (order creation, inventory sync).
- **Vertical before horizontal**: PostgreSQL scales vertically well. A 16-core, 64GB instance handles millions of orders before horizontal sharding is needed.
- **Sharding path**: If horizontal scaling becomes necessary, shard by `tenant_id` using Citus (distributed PostgreSQL) to keep tenant data co-located while distributing load.
- **Archive strategy**: Move completed orders, old price history, and inventory snapshots older than 90 days to a separate analytics database (e.g., ClickHouse or TimescaleDB) to keep the operational database lean.
- **Connection management**: At scale, use PgBouncer in transaction mode to multiplex thousands of application connections over a smaller pool of database connections.
