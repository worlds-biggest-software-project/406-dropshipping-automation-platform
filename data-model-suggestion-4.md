# Data Model Suggestion 4: Graph-Enhanced Supply Chain Model (PostgreSQL + Neo4j)

## Summary

A polyglot persistence architecture that pairs PostgreSQL for transactional order management with a Neo4j graph database for modelling the supplier-product-storefront relationship network. Dropshipping is fundamentally a supply chain coordination problem: a single product can be sourced from multiple suppliers at different prices and lead times, listed on multiple storefronts with different pricing and metadata, and fulfilled through different routing paths depending on customer location, stock levels, and supplier reliability. These relationships form a dense, multi-dimensional graph that relational joins model poorly but graph traversal handles naturally. This approach is especially well-suited for the platform's AI-native ambitions -- graph-based features like supplier similarity, product affinity, and routing path optimisation become first-class operations rather than expensive SQL queries.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                     Application Layer                        │
│   (Order API, Sync Workers, Routing Engine, Price Monitor)   │
└───────────┬────────────────────────────┬─────────────────────┘
            │                            │
            ▼                            ▼
   ┌─────────────────┐        ┌──────────────────────┐
   │   PostgreSQL     │        │      Neo4j           │
   │                  │        │                      │
   │  - Orders        │        │  - Supplier network  │
   │  - Purchase POs  │        │  - Product catalogue │
   │  - Shipments     │        │  - Routing graph     │
   │  - Returns       │        │  - Channel mappings  │
   │  - Price history │        │  - Supplier scoring  │
   │  - Tenants/auth  │        │  - AI feature store  │
   └─────────────────┘        └──────────────────────┘
            │                            │
            └────────────┬───────────────┘
                         ▼
                  ┌──────────────┐
                  │    Redis     │
                  │  (inventory  │
                  │   cache,     │
                  │   rate       │
                  │   limiting)  │
                  └──────────────┘
```

**Division of responsibility:**
- **PostgreSQL** handles transactional, write-heavy, ACID-critical data: orders, purchase orders, shipments, payments, price history, and tenant authentication.
- **Neo4j** handles the relationship graph: which suppliers carry which products, at what cost, with what reliability, and how they connect to storefronts. It serves the routing engine, recommendation system, and AI feature extraction.
- **Redis** caches real-time inventory levels for sub-millisecond stock checks.

---

## Graph Data Model (Neo4j)

### Node Types

```cypher
// ── Core Nodes ──────────────────────────────────────────

// Tenant (retailer)
CREATE (t:Tenant {
  id: 'uuid',
  name: 'Acme Dropship Store',
  plan_tier: 'pro',
  created_at: datetime()
})

// Supplier
CREATE (s:Supplier {
  id: 'uuid',
  name: 'Pacific Wholesale',
  feed_type: 'api',
  reliability_score: 0.94,
  avg_ship_days: 3.2,
  region: 'US-West',
  categories: ['electronics', 'accessories']
})

// Product (canonical, tenant-scoped)
CREATE (p:Product {
  id: 'uuid',
  tenant_id: 'uuid',
  sku: 'WIDGET-001',
  title: 'Wireless Bluetooth Speaker',
  category: 'electronics',
  base_price: 49.99,
  weight_grams: 350,
  is_active: true
})

// ProductVariant
CREATE (v:ProductVariant {
  id: 'uuid',
  variant_sku: 'WIDGET-001-BLK-L',
  size: 'L',
  colour: 'Black',
  price_override: 54.99
})

// Storefront (channel)
CREATE (sf:Storefront {
  id: 'uuid',
  tenant_id: 'uuid',
  platform: 'shopify',
  store_url: 'https://acme-store.myshopify.com',
  is_active: true
})

// Region (for routing optimisation)
CREATE (r:Region {
  code: 'US-West',
  name: 'US West Coast',
  countries: ['US'],
  states: ['CA', 'OR', 'WA', 'NV', 'AZ']
})

// Category (product taxonomy)
CREATE (c:Category {
  name: 'electronics',
  parent: 'all',
  depth: 1
})
```

### Relationship Types

```cypher
// ── Supplier-Product Relationships ──────────────────────

// A supplier CARRIES a product at a specific price and stock level
CREATE (s:Supplier)-[:CARRIES {
  supplier_sku: 'PW-BT-SPK-001',
  cost_price: 28.50,
  currency: 'USD',
  stock_quantity: 150,
  min_order_qty: 1,
  lead_time_days: 2.5,
  last_synced_at: datetime(),
  margin_pct: 43.0
}]->(p:Product)

// ── Storefront-Product Relationships ────────────────────

// A product is LISTED_ON a storefront
CREATE (p:Product)-[:LISTED_ON {
  external_id: 'shopify_product_789',
  listed_price: 49.99,
  listing_status: 'active',
  last_pushed_at: datetime()
}]->(sf:Storefront)

// ── Tenant Relationships ────────────────────────────────

// A tenant USES a supplier
CREATE (t:Tenant)-[:USES_SUPPLIER {
  priority: 1,
  is_active: true,
  since: date('2025-06-01')
}]->(s:Supplier)

// A tenant OPERATES a storefront
CREATE (t:Tenant)-[:OPERATES]->(sf:Storefront)

// A tenant OWNS a product
CREATE (t:Tenant)-[:OWNS]->(p:Product)

// ── Product Structure ───────────────────────────────────

// A product HAS_VARIANT
CREATE (p:Product)-[:HAS_VARIANT]->(v:ProductVariant)

// A product BELONGS_TO a category
CREATE (p:Product)-[:BELONGS_TO]->(c:Category)

// ── Supplier Geography ──────────────────────────────────

// A supplier SHIPS_FROM a region
CREATE (s:Supplier)-[:SHIPS_FROM {
  avg_days_to_customer: 3.2,
  shipping_methods: ['standard', 'express']
}]->(r:Region)

// A supplier SHIPS_TO a region
CREATE (s:Supplier)-[:SHIPS_TO {
  avg_days: 5.0,
  cost_range: [4.99, 12.99]
}]->(r:Region)

// ── Supplier-Supplier Relationships ─────────────────────

// Suppliers that carry overlapping products (computed)
CREATE (s1:Supplier)-[:COMPETES_WITH {
  overlap_product_count: 47,
  avg_price_diff_pct: 8.3
}]->(s2:Supplier)

// ── Historical Performance ──────────────────────────────

// Fulfilment performance between supplier and region
CREATE (s:Supplier)-[:FULFILLED_TO {
  total_orders: 1250,
  on_time_pct: 96.2,
  avg_days: 3.1,
  return_rate_pct: 2.4,
  last_30d_orders: 89,
  trend: 'stable'
}]->(r:Region)
```

### Graph Queries for Core Operations

```cypher
// ── Intelligent Order Routing ───────────────────────────

// Find the best supplier for a product shipping to US-West,
// considering cost, stock, reliability, and shipping speed
MATCH (p:Product {id: $productId})<-[carries:CARRIES]-(s:Supplier)
      -[ships:SHIPS_TO]->(r:Region {code: 'US-West'})
WHERE carries.stock_quantity >= $requiredQty
  AND s.reliability_score > 0.85
RETURN s.name, s.id,
       carries.cost_price,
       carries.stock_quantity,
       s.reliability_score,
       ships.avg_days AS estimated_days,
       // Composite routing score: lower is better
       (carries.cost_price * 0.4) +
       (ships.avg_days * 2.0) +
       ((1 - s.reliability_score) * 50) AS routing_score
ORDER BY routing_score ASC
LIMIT 3

// ── Multi-Supplier Order Splitting ──────────────────────

// For a multi-item order, find the optimal supplier assignment
// that minimises total cost while ensuring all items are in stock
UNWIND $lineItems AS item
MATCH (p:Product {id: item.productId})<-[c:CARRIES]-(s:Supplier)
      -[:SHIPS_TO]->(r:Region {code: $customerRegion})
WHERE c.stock_quantity >= item.quantity
RETURN item.productId, s.id AS supplierId, s.name,
       c.cost_price, c.stock_quantity, s.reliability_score
ORDER BY item.productId, c.cost_price ASC

// ── Supplier Substitution (fallback routing) ────────────

// When preferred supplier is out of stock, find alternatives
// that carry the same product with similar attributes
MATCH (p:Product {id: $productId})<-[:CARRIES]-(preferred:Supplier {id: $preferredSupplierId})
MATCH (p)<-[alt_carries:CARRIES]-(alt:Supplier)
WHERE alt.id <> preferred.id
  AND alt_carries.stock_quantity > 0
  AND alt.reliability_score > 0.80
RETURN alt.name, alt.id,
       alt_carries.cost_price,
       alt_carries.stock_quantity,
       alt.reliability_score,
       abs(alt_carries.cost_price - preferred.cost_price) AS price_diff
ORDER BY price_diff ASC, alt.reliability_score DESC
LIMIT 5

// ── Product Affinity (AI feature) ───────────────────────

// Find products frequently ordered together via shared supplier paths
MATCH (p1:Product {id: $productId})<-[:CARRIES]-(s:Supplier)
      -[:CARRIES]->(p2:Product)
WHERE p2.id <> p1.id AND p2.is_active = true
WITH p2, count(s) AS shared_suppliers, collect(s.name) AS supplier_names
RETURN p2.title, p2.sku, shared_suppliers, supplier_names
ORDER BY shared_suppliers DESC
LIMIT 10

// ── Supplier Network Analysis ───────────────────────────

// Find suppliers that are single points of failure
// (products only available from one supplier)
MATCH (t:Tenant {id: $tenantId})-[:OWNS]->(p:Product)<-[:CARRIES]-(s:Supplier)
WITH p, count(s) AS supplier_count, collect(s.name) AS suppliers
WHERE supplier_count = 1
RETURN p.title, p.sku, suppliers[0] AS sole_supplier
ORDER BY p.title

// ── Competitive Pricing Intelligence ────────────────────

// Compare pricing across suppliers for a category
MATCH (p:Product)-[:BELONGS_TO]->(c:Category {name: $category})
MATCH (p)<-[carries:CARRIES]-(s:Supplier)
WHERE p.tenant_id = $tenantId
RETURN p.title, p.sku,
       min(carries.cost_price) AS min_cost,
       max(carries.cost_price) AS max_cost,
       avg(carries.cost_price) AS avg_cost,
       count(s) AS supplier_count
ORDER BY (max(carries.cost_price) - min(carries.cost_price)) DESC
```

---

## Relational Schema (PostgreSQL)

PostgreSQL handles the transactional side -- data that requires ACID guarantees and sequential processing.

```sql
-- Tenants and authentication (transactional)
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    email           VARCHAR(255) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Customer orders (ACID-critical)
CREATE TABLE customer_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    storefront_id   UUID NOT NULL,          -- references Neo4j storefront node
    external_order_id VARCHAR(255) NOT NULL,
    customer_info   JSONB NOT NULL,
    shipping_address JSONB NOT NULL,
    order_total     DECIMAL(12,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    routing_data    JSONB,                   -- snapshot of graph routing decision
    ordered_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE order_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES customer_orders(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL,           -- references Neo4j product node
    variant_id      UUID,
    quantity        INT NOT NULL,
    unit_price      DECIMAL(12,2) NOT NULL,
    line_total      DECIMAL(12,2) NOT NULL,
    assigned_supplier_id UUID               -- resolved by graph routing engine
);

CREATE TABLE supplier_purchase_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES customer_orders(id),
    supplier_id     UUID NOT NULL,           -- references Neo4j supplier node
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    supplier_order_ref VARCHAR(255),
    total_cost      DECIMAL(12,2),
    routing_score   DECIMAL(8,4),            -- score from graph routing algorithm
    routing_reason  VARCHAR(100),
    submitted_at    TIMESTAMPTZ,
    confirmed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE purchase_order_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id UUID NOT NULL REFERENCES supplier_purchase_orders(id) ON DELETE CASCADE,
    order_line_item_id UUID REFERENCES order_line_items(id),
    supplier_sku    VARCHAR(100) NOT NULL,
    quantity        INT NOT NULL,
    unit_cost       DECIMAL(12,2) NOT NULL
);

CREATE TABLE shipments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id UUID NOT NULL REFERENCES supplier_purchase_orders(id),
    carrier         VARCHAR(100),
    tracking_number VARCHAR(255),
    tracking_url    VARCHAR(1000),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    tracking_timeline JSONB NOT NULL DEFAULT '[]',
    shipped_at      TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    pushed_to_storefront BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE order_returns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES customer_orders(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'requested',
    reason          TEXT,
    refund_amount   DECIMAL(12,2),
    return_details  JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Price change log (time-series)
CREATE TABLE price_changes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_id     UUID NOT NULL,
    supplier_sku    VARCHAR(100) NOT NULL,
    product_id      UUID NOT NULL,
    old_price       DECIMAL(12,2) NOT NULL,
    new_price       DECIMAL(12,2) NOT NULL,
    change_pct      DECIMAL(6,3),
    anomaly_flag    BOOLEAN NOT NULL DEFAULT false,
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Inventory sync log (time-series)
CREATE TABLE inventory_updates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_id     UUID NOT NULL,
    supplier_sku    VARCHAR(100) NOT NULL,
    product_id      UUID NOT NULL,
    old_quantity    INT,
    new_quantity    INT NOT NULL,
    source          VARCHAR(50),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Data Synchronisation Between PostgreSQL and Neo4j

```
┌─────────────────────────────────────────────────────────┐
│               Sync Strategy                             │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Supplier feed ingested                                 │
│    │                                                    │
│    ├─► Neo4j: Update CARRIES relationship               │
│    │   (cost_price, stock_quantity, last_synced_at)      │
│    │                                                    │
│    ├─► Redis: Update inventory cache                    │
│    │   (hot path for stock checks)                      │
│    │                                                    │
│    └─► PostgreSQL: Append to inventory_updates          │
│        and price_changes (if price changed)             │
│                                                         │
│  Order placed                                           │
│    │                                                    │
│    ├─► Neo4j: Query routing graph for optimal supplier  │
│    │                                                    │
│    └─► PostgreSQL: Create order, PO, line items         │
│        (with routing decision snapshot)                 │
│                                                         │
│  Delivery completed                                     │
│    │                                                    │
│    ├─► PostgreSQL: Update shipment status               │
│    │                                                    │
│    └─► Neo4j: Update FULFILLED_TO relationship          │
│        (on_time_pct, avg_days recalculated)             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Pros

- **Routing optimisation is a graph problem**: Finding the best supplier considering cost, stock, reliability, shipping speed, and region is a multi-dimensional graph traversal -- exactly what Neo4j excels at. The same query in SQL would require multiple joins and subqueries across five or more tables.
- **Supplier network visibility**: Graph queries trivially answer business-critical questions: "Which products have only one supplier?" (single points of failure), "Which suppliers compete on the same products?" (leverage for negotiation), and "What is the cheapest path from supplier to customer?" Relational databases make these queries expensive or impractical.
- **AI/ML feature extraction**: Graph embeddings and centrality algorithms (PageRank, betweenness centrality) can score supplier importance, identify product clusters, and detect anomalous supply chain patterns. Neo4j's Graph Data Science library provides these algorithms out of the box.
- **Natural model for the domain**: The mental model of "suppliers carry products which are listed on storefronts" is inherently a graph. Developers and stakeholders can visualise and query the supply chain intuitively using Neo4j Browser or Bloom.
- **Dynamic routing without complex rule engines**: Instead of maintaining a separate routing rules engine, the graph itself encodes routing preferences through relationship weights. Adding a new routing criterion (e.g., environmental impact score) is just adding a property to a relationship.
- **Transactional integrity preserved**: Order processing, financial calculations, and shipment tracking remain in PostgreSQL with full ACID guarantees. The graph handles reads and relationship queries where eventual consistency is acceptable.

## Cons

- **Operational complexity**: Running and maintaining two database systems (PostgreSQL + Neo4j) increases infrastructure complexity, monitoring burden, and the number of failure modes. Backups, upgrades, and disaster recovery must be coordinated across both systems.
- **Data synchronisation risk**: The same entities exist in both databases (e.g., supplier IDs, product IDs). Keeping them in sync requires careful event-driven or change-data-capture pipelines. Inconsistencies between the two stores can cause routing decisions based on stale graph data.
- **Team skill requirements**: Neo4j and Cypher are less widely known than SQL and PostgreSQL. Hiring and onboarding developers requires additional training. The graph model also requires different mental models for schema design and query optimisation.
- **Neo4j licensing**: Neo4j Community Edition is open source (GPLv3) but lacks clustering, role-based access control, and some enterprise features. Neo4j Enterprise or AuraDB (cloud) is commercially licensed. Alternatives like Memgraph or Apache AGE (PostgreSQL extension) exist but have smaller ecosystems.
- **Write throughput limitations**: Neo4j handles write-heavy workloads less efficiently than PostgreSQL. High-frequency inventory updates (hundreds per second across thousands of supplier-product mappings) may stress the graph database. This is mitigated by keeping inventory updates in Redis/PostgreSQL and batch-syncing to Neo4j.
- **Transaction spanning two databases**: An order placement that writes to both PostgreSQL (order record) and Neo4j (routing decision) cannot be wrapped in a single atomic transaction. Saga patterns or compensating transactions are needed.

---

## Technology Recommendations

| Component | Recommendation |
|---|---|
| Transactional DB | PostgreSQL 16+ |
| Graph DB | Neo4j 5.x (Community or AuraDB Free for development; Enterprise or AuraDB Pro for production) |
| Alternative graph | Apache AGE (PostgreSQL extension) if a single-database constraint is required |
| Inventory cache | Redis with pub/sub for real-time stock level distribution |
| Event bus | Apache Kafka for syncing state between PostgreSQL, Neo4j, and Redis |
| Graph algorithms | Neo4j Graph Data Science library for PageRank, community detection, similarity |
| API layer | Node.js with neo4j-driver and pg (or Prisma for PostgreSQL) |
| Monitoring | Neo4j Ops Manager + Prometheus/Grafana for PostgreSQL |
| Visualisation | Neo4j Bloom for supply chain network visualisation |

---

## Migration and Scaling Considerations

- **Start with Apache AGE, graduate to Neo4j**: If running two databases feels premature, start with Apache AGE (a PostgreSQL extension that adds Cypher/graph query support). This keeps everything in a single PostgreSQL instance. When graph query complexity or performance demands exceed AGE's capabilities, migrate the graph layer to a dedicated Neo4j instance.
- **Change Data Capture (CDC) for sync**: Use Debezium to capture changes from PostgreSQL and propagate them to Neo4j, rather than dual-writing from the application. This reduces consistency risks and decouples the systems.
- **Graph as a read-optimised layer**: Treat Neo4j as a read-optimised materialisation of the supply chain relationships. PostgreSQL remains the source of truth. If Neo4j goes down, the system can fall back to SQL-based routing queries (slower but functional).
- **Batch graph updates for inventory**: Rather than updating Neo4j on every inventory change, batch-update CARRIES relationship properties every 30-60 seconds from the Redis cache. This keeps the graph current enough for routing while avoiding write contention.
- **Neo4j clustering**: For production resilience, deploy a Neo4j causal cluster (3+ nodes) with read replicas to handle routing query load. Routing queries are read-only and embarrassingly parallelisable.
- **Graph schema evolution**: Neo4j is schema-optional, so adding new node labels, relationship types, or properties requires no migration. However, maintain a data dictionary to ensure consistency.
- **Cost management**: Neo4j AuraDB pricing scales with graph size and query throughput. Monitor relationship count growth (supplier-product mappings are the primary driver) and archive inactive supplier relationships to control costs.
- **Fallback strategy**: Design the routing engine to function (with degraded performance) using PostgreSQL-only queries if Neo4j is unavailable. This ensures the platform does not have a hard dependency on graph database availability for order processing.
