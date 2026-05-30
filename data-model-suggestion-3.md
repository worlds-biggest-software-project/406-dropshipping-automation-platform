# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

## Summary

A pragmatic hybrid architecture using PostgreSQL with strategic JSONB columns, combining the referential integrity of relational tables for core transactional entities (orders, purchase orders, shipments) with the schema flexibility of JSONB for inherently variable data (supplier feed attributes, product metadata, channel-specific fields, routing rule conditions). This approach acknowledges that dropshipping platforms must ingest data from dozens of suppliers with different field schemas and push data to storefronts with different metadata requirements -- and that forcing all of this into rigid columns creates more problems than it solves.

---

## Design Philosophy

The hybrid model draws a clear line between two types of data in the dropshipping domain:

1. **Structured core data** -- entities whose shape is well-known and stable: tenants, orders, line items, purchase orders, shipments, returns. These use traditional relational columns with foreign keys.

2. **Semi-structured peripheral data** -- data whose shape varies by source or context: supplier product attributes (one supplier sends `color` and `size`, another sends `material` and `weight_kg`), storefront-specific listing metadata, raw feed records, routing rule conditions, and integration webhook payloads. These use JSONB columns.

---

## Key Entities and Relationships

### Schema

```sql
-- ================================================================
-- CORE RELATIONAL TABLES (structured, stable, FK-enforced)
-- ================================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    email           VARCHAR(255) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',       -- tenant-level preferences
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Storefront connections with channel-specific config in JSONB
CREATE TABLE storefronts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    platform        VARCHAR(50) NOT NULL,
    store_url       VARCHAR(500),
    credentials     JSONB NOT NULL DEFAULT '{}',       -- encrypted API keys, tokens
    platform_config JSONB NOT NULL DEFAULT '{}',       -- platform-specific settings
    --  e.g. Shopify: {"location_id": "xxx", "collection_ids": [...]}
    --  e.g. eBay: {"marketplace_id": "EBAY_US", "listing_format": "FixedPrice"}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Suppliers with flexible connection configuration
CREATE TABLE suppliers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    feed_type       VARCHAR(20) NOT NULL,
    contact_info    JSONB DEFAULT '{}',
    connection_config JSONB NOT NULL DEFAULT '{}',
    --  CSV: {"url": "https://...", "delimiter": ",", "encoding": "UTF-8", "skip_rows": 1}
    --  API: {"base_url": "https://...", "auth_type": "bearer", "rate_limit": 100}
    --  FTP: {"host": "ftp.supplier.com", "port": 21, "path": "/feeds/inventory.csv"}
    --  EDI: {"isa_id": "...", "gs_id": "...", "transaction_sets": ["850", "856"]}
    field_mapping   JSONB NOT NULL DEFAULT '{}',
    --  Maps supplier fields to canonical fields:
    --  {"supplier_field": "canonical_field", "item_no": "sku", "qty_avail": "stock_quantity", ...}
    reliability_score DECIMAL(3,2) DEFAULT 0.00,
    avg_ship_days   DECIMAL(4,1),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tenant_suppliers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    supplier_id     UUID NOT NULL REFERENCES suppliers(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    priority        INT NOT NULL DEFAULT 0,
    custom_config   JSONB DEFAULT '{}',                -- tenant-specific overrides
    UNIQUE(tenant_id, supplier_id)
);

-- ================================================================
-- PRODUCT CATALOGUE (relational core + JSONB attributes)
-- ================================================================

-- Canonical products: fixed columns for universal fields, JSONB for the rest
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

    -- Flexible product attributes that vary by product type
    attributes      JSONB NOT NULL DEFAULT '{}',
    --  Electronics: {"brand": "Samsung", "model": "Galaxy S25", "warranty_months": 24}
    --  Clothing:    {"material": "cotton", "sizes": ["S","M","L"], "colour": "navy"}
    --  Furniture:   {"dimensions_cm": {"l": 120, "w": 60, "h": 75}, "assembly_required": true}

    -- SEO and display metadata
    metadata        JSONB NOT NULL DEFAULT '{}',
    --  {"tags": ["wireless", "bluetooth"], "seo_title": "...", "seo_description": "..."}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, sku)
);

CREATE TABLE product_variants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    variant_sku     VARCHAR(100) NOT NULL,
    price_override  DECIMAL(12,2),
    weight_grams    INT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    attributes      JSONB NOT NULL DEFAULT '{}',      -- {"size": "L", "colour": "Blue"}
    UNIQUE(product_id, variant_sku)
);

-- Links a product to a supplier's SKU
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

    -- Raw supplier data preserved for debugging and re-mapping
    raw_supplier_data JSONB DEFAULT '{}',
    --  The original record from the supplier feed, before field mapping.
    --  Invaluable for debugging sync issues without re-fetching the feed.

    UNIQUE(product_id, supplier_id)
);

-- Links a product to a storefront listing
CREATE TABLE storefront_listings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    storefront_id   UUID NOT NULL REFERENCES storefronts(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),
    listed_price    DECIMAL(12,2),
    listing_status  VARCHAR(20) NOT NULL DEFAULT 'draft',
    last_pushed_at  TIMESTAMPTZ,

    -- Channel-specific listing data
    channel_data    JSONB NOT NULL DEFAULT '{}',
    --  Shopify: {"variant_ids": [...], "collection_id": "...", "published_scope": "web"}
    --  eBay:    {"item_id": "...", "listing_type": "FixedPrice", "condition_id": 1000}
    --  WooCommerce: {"post_id": 123, "categories": [4, 7], "cross_sells": [45, 67]}

    UNIQUE(product_id, storefront_id)
);

-- Product images with platform-specific variants
CREATE TABLE product_images (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    url             VARCHAR(1000) NOT NULL,
    position        INT NOT NULL DEFAULT 0,
    alt_text        VARCHAR(500),
    variants        JSONB DEFAULT '{}',
    --  {"thumbnail": "https://...", "shopify_cdn": "https://...", "ebay_gallery": "https://..."}
    is_primary      BOOLEAN NOT NULL DEFAULT false
);

-- ================================================================
-- ORDER MANAGEMENT (relational core, JSONB for addresses & metadata)
-- ================================================================

CREATE TABLE customer_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    storefront_id   UUID NOT NULL REFERENCES storefronts(id),
    external_order_id VARCHAR(255) NOT NULL,
    order_total     DECIMAL(12,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    ordered_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Address and customer details are semi-structured
    -- (different channels use different address formats)
    customer_info   JSONB NOT NULL DEFAULT '{}',
    --  {"name": "...", "email": "...", "phone": "..."}
    shipping_address JSONB NOT NULL,
    --  {"line1": "...", "line2": "...", "city": "...", "state": "...",
    --   "postal_code": "...", "country": "US"}
    billing_address JSONB,

    -- Channel-specific order metadata
    channel_metadata JSONB DEFAULT '{}',
    --  Shopify: {"financial_status": "paid", "fulfillment_status": null, "tags": [...]}
    --  eBay:    {"buyer_user_id": "...", "checkout_status": "COMPLETE"}

    UNIQUE(storefront_id, external_order_id)
);

CREATE TABLE order_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES customer_orders(id) ON DELETE CASCADE,
    product_id      UUID REFERENCES products(id),
    variant_id      UUID REFERENCES product_variants(id),
    quantity        INT NOT NULL,
    unit_price      DECIMAL(12,2) NOT NULL,
    line_total      DECIMAL(12,2) NOT NULL,
    properties      JSONB DEFAULT '{}',     -- customisation options, gift messages, etc.
    external_line_id VARCHAR(255)           -- Shopify line item ID, eBay transaction ID
);

CREATE TABLE supplier_purchase_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES customer_orders(id),
    supplier_id     UUID NOT NULL REFERENCES suppliers(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    supplier_order_ref VARCHAR(255),
    total_cost      DECIMAL(12,2),
    submitted_at    TIMESTAMPTZ,
    confirmed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Submission details vary by supplier type
    submission_details JSONB DEFAULT '{}',
    --  API: {"request_id": "...", "response_code": 200, "confirmation_id": "..."}
    --  EDI: {"isa_control": "000012345", "transaction_set": "850", "ack_received": true}
    --  Manual: {"submitted_via": "email", "submitted_by": "user@..."}

    routing_decision JSONB DEFAULT '{}'
    --  {"reason": "lowest_cost", "alternatives_considered": [...],
    --   "ai_confidence": 0.92, "cost_savings_vs_next": 2.50}
);

CREATE TABLE purchase_order_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id UUID NOT NULL REFERENCES supplier_purchase_orders(id) ON DELETE CASCADE,
    order_line_item_id UUID REFERENCES order_line_items(id),
    supplier_sku    VARCHAR(100) NOT NULL,
    quantity        INT NOT NULL,
    unit_cost       DECIMAL(12,2) NOT NULL
);

-- ================================================================
-- SHIPMENTS AND TRACKING
-- ================================================================

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Full tracking timeline stored as JSONB array
    tracking_timeline JSONB NOT NULL DEFAULT '[]'
    --  [
    --    {"status": "label_created", "location": "...", "timestamp": "..."},
    --    {"status": "picked_up", "location": "...", "timestamp": "..."},
    --    {"status": "in_transit", "location": "...", "timestamp": "..."},
    --    {"status": "delivered", "location": "...", "timestamp": "..."}
    --  ]
);

-- ================================================================
-- RETURNS MANAGEMENT
-- ================================================================

CREATE TABLE order_returns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES customer_orders(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'requested',
    reason          TEXT,
    refund_amount   DECIMAL(12,2),
    return_details  JSONB DEFAULT '{}',
    --  {"line_items": [{"line_item_id": "...", "qty": 1, "reason": "defective"}],
    --   "customer_notes": "...", "photos": ["url1", "url2"]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE supplier_return_authorisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    return_id       UUID NOT NULL REFERENCES order_returns(id),
    supplier_id     UUID NOT NULL REFERENCES suppliers(id),
    rma_number      VARCHAR(100),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    supplier_response JSONB DEFAULT '{}',   -- raw supplier RMA response
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ================================================================
-- PRICING AND ROUTING RULES (JSONB-heavy, inherently dynamic)
-- ================================================================

CREATE TABLE margin_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    rule_name       VARCHAR(255) NOT NULL,
    priority        INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- Rule conditions and actions are fully dynamic
    conditions      JSONB NOT NULL DEFAULT '{}',
    --  {"category": "electronics", "supplier_id": "...", "price_range": {"min": 0, "max": 50}}
    action          JSONB NOT NULL,
    --  {"type": "percentage_markup", "value": 40}
    --  {"type": "fixed_markup", "value": 15.00}
    --  {"type": "min_margin", "value": 5.00, "fallback": "deactivate"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE routing_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    rule_name       VARCHAR(255) NOT NULL,
    priority        INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,

    conditions      JSONB NOT NULL DEFAULT '{}',
    --  {"shipping_country": "US", "order_value_min": 100, "category": "apparel"}
    action          JSONB NOT NULL,
    --  {"strategy": "preferred_supplier", "supplier_id": "...",
    --   "fallback": "lowest_cost"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ================================================================
-- PRICE AND INVENTORY HISTORY (time-series with JSONB context)
-- ================================================================

CREATE TABLE price_changes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_product_mapping_id UUID NOT NULL REFERENCES supplier_product_mappings(id),
    old_price       DECIMAL(12,2) NOT NULL,
    new_price       DECIMAL(12,2) NOT NULL,
    change_pct      DECIMAL(6,3),
    anomaly_flag    BOOLEAN NOT NULL DEFAULT false,
    context         JSONB DEFAULT '{}',     -- {"feed_record": {...}, "detection_method": "zscore"}
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE inventory_updates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_product_mapping_id UUID NOT NULL REFERENCES supplier_product_mappings(id),
    old_quantity    INT,
    new_quantity    INT NOT NULL,
    source          VARCHAR(50),             -- 'feed_sync', 'order_decrement', 'manual'
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ================================================================
-- SYNC AND INTEGRATION LOGS
-- ================================================================

CREATE TABLE sync_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    source_type     VARCHAR(50) NOT NULL,    -- 'supplier_feed', 'storefront_webhook', etc.
    source_id       UUID,                    -- supplier_id or storefront_id
    direction       VARCHAR(10) NOT NULL,    -- 'inbound' or 'outbound'
    status          VARCHAR(20) NOT NULL,    -- 'success', 'partial', 'failed'
    items_processed INT DEFAULT 0,
    items_failed    INT DEFAULT 0,
    error_details   JSONB DEFAULT '[]',
    --  [{"sku": "ABC123", "error": "price_missing"}, ...]
    duration_ms     INT,
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ
);
```

### JSONB Indexing Strategy

```sql
-- GIN indexes for JSONB query patterns
CREATE INDEX idx_products_attributes ON products USING GIN (attributes jsonb_path_ops);
CREATE INDEX idx_products_metadata ON products USING GIN (metadata jsonb_path_ops);
CREATE INDEX idx_margin_rules_conditions ON margin_rules USING GIN (conditions jsonb_path_ops);
CREATE INDEX idx_routing_rules_conditions ON routing_rules USING GIN (conditions jsonb_path_ops);
CREATE INDEX idx_supplier_raw_data ON supplier_product_mappings USING GIN (raw_supplier_data jsonb_path_ops);

-- Functional indexes on commonly queried JSONB paths
CREATE INDEX idx_products_brand ON products ((attributes->>'brand'));
CREATE INDEX idx_orders_customer_email ON customer_orders ((customer_info->>'email'));

-- Standard B-tree indexes for relational columns
CREATE INDEX idx_products_tenant ON products(tenant_id);
CREATE INDEX idx_products_category ON products(tenant_id, category);
CREATE INDEX idx_orders_tenant_status ON customer_orders(tenant_id, status);
CREATE INDEX idx_spm_product ON supplier_product_mappings(product_id);
CREATE INDEX idx_spm_supplier ON supplier_product_mappings(supplier_id);
CREATE INDEX idx_price_changes_time ON price_changes(supplier_product_mapping_id, detected_at);
CREATE INDEX idx_inventory_updates_time ON inventory_updates(supplier_product_mapping_id, recorded_at);
```

### Example Queries

```sql
-- Find all "electronics" products with a specific brand from a supplier
SELECT p.id, p.title, p.attributes->>'brand' AS brand,
       spm.supplier_price, spm.stock_quantity
FROM products p
JOIN supplier_product_mappings spm ON spm.product_id = p.id
WHERE p.tenant_id = $1
  AND p.category = 'electronics'
  AND p.attributes @> '{"brand": "Samsung"}'
  AND spm.stock_quantity > 0;

-- Get full order details with channel-specific metadata
SELECT co.id, co.external_order_id, co.order_total,
       co.customer_info->>'name' AS customer_name,
       co.shipping_address->>'country' AS ship_country,
       co.channel_metadata,
       spo.routing_decision->>'reason' AS routing_reason
FROM customer_orders co
LEFT JOIN supplier_purchase_orders spo ON spo.order_id = co.id
WHERE co.tenant_id = $1
  AND co.status = 'processing';

-- Evaluate margin rules with JSONB condition matching
SELECT * FROM margin_rules
WHERE tenant_id = $1
  AND is_active = true
  AND (conditions @> '{"category": "electronics"}' OR conditions = '{}')
ORDER BY priority DESC;
```

---

## Pros

- **Best of both worlds**: Core transactional data (orders, shipments, financial amounts) gets full relational integrity with foreign keys and type safety, while variable data (product attributes, channel metadata, supplier feed fields) gets JSONB flexibility without schema migrations.
- **Supplier feed ingestion is dramatically simpler**: Raw supplier data can be stored in `raw_supplier_data` JSONB immediately upon ingestion, with field mapping applied separately. No need to pre-define columns for every possible supplier field. Adding a new supplier never requires a schema change.
- **Channel-specific data without table explosion**: Instead of `shopify_listings`, `ebay_listings`, `woocommerce_listings` tables, a single `storefront_listings` table with a `channel_data` JSONB column handles all platforms. New platform integrations require zero schema changes.
- **Routing and pricing rules are naturally dynamic**: Rules with complex conditions ("if category is electronics AND shipping country is US AND order value > $100, route to supplier X") map cleanly to JSONB condition objects that can be evaluated programmatically.
- **Single database simplicity**: No need for a separate document database. PostgreSQL's JSONB support provides document-store capabilities with GIN indexing, containment operators (`@>`), and JSONPath queries, all within a single transactional database.
- **Incremental evolution**: Start with more JSONB columns and promote frequently-queried fields to dedicated columns as patterns emerge. This is far safer than starting with rigid columns and discovering you need flexibility.

## Cons

- **JSONB is not self-documenting**: Unlike typed columns, JSONB fields have no schema enforcement at the database level. A `supplier_price` column guarantees a decimal; an `attributes` JSONB could contain anything. Application-layer validation (JSON Schema, Zod, etc.) must compensate.
- **ORM support is uneven**: While modern ORMs (Prisma, Drizzle, TypeORM) support JSONB, the developer experience for querying nested JSONB paths is less polished than querying regular columns. Complex JSONB queries often require raw SQL.
- **GIN index overhead**: GIN indexes on JSONB columns are larger than B-tree indexes and slower to update. Write-heavy tables (like `supplier_product_mappings` with frequent stock updates) may see increased index maintenance costs.
- **Reporting complexity**: Analytical queries that need to aggregate across JSONB fields (e.g., "average margin by brand" where brand is in JSONB) are slower than aggregating across indexed columns. Materialised views or ETL to a reporting layer may be needed.
- **Discipline required**: The flexibility of JSONB can lead to inconsistency if different parts of the codebase store the same concept differently. A data dictionary and application-layer schema validation are essential.

---

## Technology Recommendations

| Component | Recommendation |
|---|---|
| Database | PostgreSQL 16+ (native JSONB, GIN indexes, JSONPath) |
| ORM | Prisma with `Json` field type, or Drizzle ORM with `jsonb()` |
| Schema validation | Zod (TypeScript) or JSON Schema for JSONB field validation at the application layer |
| Migrations | Prisma Migrate or Drizzle Kit (handle JSONB columns and GIN indexes) |
| Caching | Redis for hot inventory data |
| Background jobs | BullMQ for feed polling, order sync, and price monitoring |
| Monitoring | pg_stat_user_indexes to track GIN index size and bloat |
| Search | PostgreSQL full-text search for product search, or Meilisearch for faceted product discovery |

---

## Migration and Scaling Considerations

- **Column promotion pattern**: When a JSONB field is queried frequently enough to warrant its own index, promote it to a dedicated column with a migration. For example, if `attributes->>'brand'` is queried on every product search, add a `brand VARCHAR(255)` column and backfill it. Keep the JSONB field for remaining attributes.
- **Partition time-series tables**: `price_changes` and `inventory_updates` should be partitioned by month from day one. These tables grow fastest and benefit most from partition pruning.
- **Materialised views for dashboards**: Pre-compute margin reports, supplier scorecards, and inventory summaries as materialised views refreshed on a schedule, rather than running expensive JSONB aggregation queries live.
- **JSONB size monitoring**: Monitor the average size of JSONB columns. If `raw_supplier_data` grows beyond a few KB per row, consider moving raw feed records to object storage (S3) with a reference URL in the database.
- **Read replicas**: Route dashboard and reporting queries to read replicas. The combination of GIN indexes and complex JSONB queries can be CPU-intensive and should not compete with transactional workloads.
- **Schema validation at the boundary**: Validate JSONB structure at API and feed-ingestion boundaries using Zod schemas or JSON Schema. This prevents garbage data from entering the system and makes downstream processing reliable.
- **Citus for horizontal scaling**: If the single-instance PostgreSQL ceiling is reached, Citus distributes tables by `tenant_id` while preserving JSONB functionality. This is the natural horizontal scaling path for multi-tenant PostgreSQL.
