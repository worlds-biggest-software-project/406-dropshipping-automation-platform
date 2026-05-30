# Dropshipping Automation Platform — Phased Development Plan

> Project: 406-dropshipping-automation-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

> **Research note:** This plan was generated from `research.md`, `README.md`, and four `data-model-suggestion-*.md` documents. No `features.md` or `standards.md` exist for this project. MVP scope is therefore derived from the core features in `research.md`/`README.md` and the competitor landscape (Inventory Source, AutoDS, Spark Shipping, Dropified, DSers, Spocket, Zendrop, Flxpoint). Feature prioritisation reflects "table stakes vs differentiator" reasoning from the research rather than a formal competitor feature matrix. Standards alignment is limited to commonly used ecommerce formats (HMAC webhook verification, OAuth 2.0 for storefront APIs, RFC 4180 CSV, X12 EDI 850/856) inferred from the domain.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (Node.js 22 LTS) | The domain is integration-heavy (REST APIs to Shopify/WooCommerce/eBay, webhooks, feed parsing), not compute-heavy. TypeScript gives end-to-end type safety from API to DB, and every target storefront publishes a maintained Node SDK. A single language across API, workers, and (future) frontend reduces context-switching. |
| Runtime/package manager | pnpm + tsx/tsup | pnpm's content-addressed store keeps the monorepo-style `packages/` lean. tsx for dev, tsup for production bundles. |
| API framework | Fastify 5 | Faster than Express, first-class JSON Schema validation (we validate every webhook and feed record), built-in plugin encapsulation, and `@fastify/swagger` auto-generates an OpenAPI 3.1 spec from route schemas. |
| Database | PostgreSQL 16 | Chosen per **Data Model Suggestion 3 (Hybrid Relational + JSONB)**. Core transactional entities (orders, POs, shipments) get FK integrity; variable data (supplier feed fields, channel metadata, routing/margin rule conditions) lives in JSONB. Single database avoids the operational cost of the polyglot Neo4j model (Suggestion 4) at MVP. |
| ORM / query layer | Drizzle ORM | Type-safe schema-as-code, first-class `jsonb()` columns and GIN index declarations, generates SQL migrations, and drops to raw SQL cleanly for JSONB containment queries. Lighter and more transparent than Prisma for a JSONB-heavy schema. |
| Migrations | Drizzle Kit | Generates and applies SQL migrations from the Drizzle schema; supports partitioned tables and GIN indexes. |
| Schema validation | Zod | Validates JSONB payloads at every boundary (webhooks, feed records, rule definitions) before they enter the DB, compensating for JSONB's lack of DB-level enforcement. Zod schemas also produce the JSON Schemas Fastify uses for OpenAPI. |
| Task queue | BullMQ on Redis | All async work (feed polling, order→PO conversion, repricing, tracking sync, storefront pushes) runs as durable, retryable jobs with concurrency control and scheduled/repeatable jobs for polling. |
| Cache | Redis 7 | Hot inventory levels (sub-ms stock checks to prevent overselling), idempotency keys for at-least-once webhooks, and BullMQ backing store. |
| LLM provider | Anthropic Claude via the official SDK, behind a provider interface | Powers the AI-native features: price-anomaly explanation, stock-out prediction narratives, and field-mapping inference for new supplier feeds. Wrapped in an interface so a local model can be swapped in for self-hosted deployments. |
| HTTP client | undici | Native, fast, supports per-supplier connection pools and timeouts. |
| Feed parsing | `csv-parse` (RFC 4180), `fast-xml-parser` (XML), `basic-ftp`/`ssh2-sftp-client` (FTP/SFTP), `node-x12` (EDI 850/856) | Covers every feed format named in the README: CSV, XML, API, EDI, FTP. |
| Testing | Vitest + Testcontainers | Vitest for unit/integration with fast watch mode; Testcontainers spins up ephemeral Postgres + Redis for real integration tests. `nock`/`msw` mock external storefront and supplier APIs. |
| Code quality | Biome | Single fast tool for lint + format, replacing ESLint + Prettier. |
| Type checking | `tsc --noEmit` in CI | Strict mode enforced. |
| Containerisation | Docker + docker-compose | Self-hosted is a stated deployment mode. compose brings up api, worker, postgres, redis for one-command local and self-hosted runs. |
| Secrets / crypto | Node `crypto` AES-256-GCM with a `MASTER_KEY` env var | Supplier and storefront credentials are encrypted at rest in JSONB `credentials` fields. |
| Config | `dotenv` + a Zod-validated `env.ts` | Fail fast on missing/invalid config at boot. |
| Observability | `pino` structured logging + OpenTelemetry traces | Correlation IDs thread through webhook → job → external call for support debugging. |

### Project Structure

```
dropshipping-automation-platform/
├── package.json                  # pnpm workspace root
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── biome.json
├── docker-compose.yml            # api, worker, postgres, redis
├── Dockerfile                    # multi-stage build for api + worker
├── .env.example
├── drizzle.config.ts
├── packages/
│   ├── core/                     # shared domain: types, db schema, config, utils
│   │   ├── src/
│   │   │   ├── env.ts            # Zod-validated environment config
│   │   │   ├── db/
│   │   │   │   ├── schema.ts     # Drizzle table definitions (the data model)
│   │   │   │   ├── client.ts     # pooled pg connection
│   │   │   │   └── migrations/   # generated SQL migrations
│   │   │   ├── crypto.ts         # AES-256-GCM encrypt/decrypt for credentials
│   │   │   ├── audit.ts          # append-only audit_events writer
│   │   │   ├── money.ts          # integer-cents money helpers
│   │   │   ├── logger.ts         # pino instance + correlation context
│   │   │   └── types.ts          # shared domain types & Zod schemas
│   │   └── package.json
│   ├── suppliers/                # supplier connectors + feed parsing + sync
│   │   ├── src/
│   │   │   ├── connectors/       # csv.ts, xml.ts, api.ts, ftp.ts, edi.ts
│   │   │   ├── field-mapper.ts   # supplier field → canonical field mapping
│   │   │   ├── ai-mapper.ts      # LLM-assisted field mapping inference
│   │   │   ├── sync-service.ts   # diff & upsert supplier_product_mappings
│   │   │   └── index.ts
│   │   └── package.json
│   ├── channels/                 # storefront connectors (Shopify, WooCommerce, ...)
│   │   ├── src/
│   │   │   ├── adapters/         # shopify.ts, woocommerce.ts, bigcommerce.ts, wix.ts, ebay.ts
│   │   │   ├── channel-adapter.ts# ChannelAdapter interface
│   │   │   ├── webhook-verify.ts # HMAC/OAuth signature verification per platform
│   │   │   └── index.ts
│   │   ├── pricing/              # margin rule engine + repricing
│   │   │   ├── margin-engine.ts
│   │   │   └── anomaly.ts        # price-change anomaly detection
│   │   ├── routing/              # supplier selection / order splitting
│   │   │   ├── routing-engine.ts
│   │   │   └── scorer.ts
│   │   ├── fulfilment/           # order→PO, PO submission, tracking sync, returns
│   │   │   ├── order-service.ts
│   │   │   ├── po-service.ts
│   │   │   ├── tracking-service.ts
│   │   │   └── returns-service.ts
│   │   └── package.json
│   ├── ai/                       # LLM provider interface + ML predictors
│   │   ├── src/
│   │   │   ├── provider.ts       # LlmProvider interface + Anthropic impl
│   │   │   ├── stockout.ts       # stock-out prediction
│   │   │   ├── supplier-scoring.ts
│   │   │   └── prompts.ts        # prompt templates
│   │   └── package.json
│   ├── api/                      # Fastify HTTP server
│   │   ├── src/
│   │   │   ├── server.ts
│   │   │   ├── plugins/          # auth, db, swagger, error-handler
│   │   │   └── routes/           # tenants, suppliers, products, orders, webhooks, ...
│   │   └── package.json
│   └── worker/                   # BullMQ workers + schedulers
│       ├── src/
│       │   ├── index.ts
│       │   ├── queues.ts         # queue + repeatable-job definitions
│       │   └── processors/       # feed-poll, order-route, reprice, tracking-sync
│       └── package.json
└── tests/
    ├── fixtures/                 # sample CSV/XML/EDI feeds, webhook payloads
    └── e2e/                      # full-flow tests
```

---

## Phase 1: Foundation — Project Skeleton, Config, Database & Audit Log

### Purpose
Establish the monorepo, the validated configuration layer, the database schema and migration tooling, the credential-encryption utility, and the append-only audit log. Nothing functional ships to a user, but every later phase depends on these primitives. After this phase the database can be created from migrations, credentials can be safely stored, and every state change has a place to be recorded.

### Tasks

#### 1.1 — Monorepo & tooling skeleton

**What**: Set up the pnpm workspace, TypeScript base config, Biome, Docker, and docker-compose.

**Design**:
- `pnpm-workspace.yaml` includes `packages/*`.
- `tsconfig.base.json`: `strict: true`, `noUncheckedIndexedAccess: true`, `module: NodeNext`, `target: ES2023`.
- `docker-compose.yml` services: `postgres` (postgres:16, volume-backed), `redis` (redis:7), `api`, `worker`. `api`/`worker` depend on healthy `postgres` and `redis`.
- `Dockerfile`: multi-stage — `deps` (pnpm install), `build` (tsup bundle), `runtime` (node:22-slim). One image, entrypoint chooses `api` or `worker` via `APP_ROLE` env.
- Root scripts: `dev`, `build`, `lint`, `typecheck`, `test`, `db:generate`, `db:migrate`.

**Testing**:
- `Unit: pnpm typecheck` passes on the empty skeleton.
- `Unit: pnpm lint` passes (Biome clean).
- `Integration: docker compose up` brings postgres + redis to healthy state; `docker compose run api node -e "process.exit(0)"` exits 0.

#### 1.2 — Validated environment config

**What**: A single `env.ts` that parses and validates all environment variables at boot.

**Design**:
```typescript
// packages/core/src/env.ts
import { z } from 'zod';

const EnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  APP_ROLE: z.enum(['api', 'worker']).default('api'),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  MASTER_KEY: z.string().length(64),          // 32-byte key, hex-encoded
  ANTHROPIC_API_KEY: z.string().optional(),
  AI_ENABLED: z.coerce.boolean().default(false),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});
export type Env = z.infer<typeof EnvSchema>;
export const env: Env = EnvSchema.parse(process.env);
```

**Testing**:
- `Unit: valid env → parsed Env object with defaults applied`.
- `Unit: missing DATABASE_URL → ZodError naming DATABASE_URL`.
- `Unit: MASTER_KEY of wrong length → ZodError`.

#### 1.3 — Database schema & migrations (Data Model Suggestion 3)

**What**: Implement the full hybrid relational+JSONB schema in Drizzle, generate the initial migration.

**Design**: Translate the Suggestion-3 DDL into `packages/core/src/db/schema.ts`. Tables: `tenants`, `storefronts`, `suppliers`, `tenant_suppliers`, `products`, `product_variants`, `supplier_product_mappings`, `storefront_listings`, `product_images`, `customer_orders`, `order_line_items`, `supplier_purchase_orders`, `purchase_order_line_items`, `shipments`, `order_returns`, `supplier_return_authorisations`, `margin_rules`, `routing_rules`, `price_changes`, `inventory_updates`, `sync_logs`. Plus one new table for audit (task 1.5).

Money fields: store as `integer` cents columns (`unit_price_cents`, `supplier_price_cents`, etc.) rather than `DECIMAL`, with `currency CHAR(3)`, to avoid floating-point drift; expose `DECIMAL`-style values only at API boundaries via `money.ts`.

Indexes per Suggestion 3, including GIN indexes:
```typescript
// e.g.
export const products = pgTable('products', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id, { onDelete: 'cascade' }),
  sku: varchar('sku', { length: 100 }).notNull(),
  // ...
  attributes: jsonb('attributes').notNull().default({}),
  metadata: jsonb('metadata').notNull().default({}),
}, (t) => ({
  tenantSkuUq: unique().on(t.tenantId, t.sku),
  tenantIdx: index('idx_products_tenant').on(t.tenantId),
  attrsGin: index('idx_products_attributes').using('gin', t.attributes),
}));
```
`price_changes` and `inventory_updates` declared as monthly **range-partitioned** tables on their timestamp columns.

**Testing**:
- `Integration (Testcontainers): apply migration to fresh Postgres → all tables exist (query information_schema)`.
- `Integration: insert tenant → product with tenant_id FK; deleting tenant cascades to product`.
- `Integration: GIN containment query products.attributes @> '{"brand":"Samsung"}' uses the GIN index (EXPLAIN shows Bitmap Index Scan)`.
- `Unit: money.ts round-trips 1999 cents ↔ "19.99" without drift`.

#### 1.4 — Credential encryption

**What**: AES-256-GCM helpers to encrypt/decrypt credential blobs stored in JSONB.

**Design**:
```typescript
// packages/core/src/crypto.ts
export function encryptSecret(plaintext: string): string; // returns base64(iv|tag|ciphertext)
export function decryptSecret(blob: string): string;
```
Key derived from `env.MASTER_KEY` (hex → 32 bytes). Storefront/supplier `credentials` JSONB stores values as `{ "<field>": { "enc": "<blob>" } }`; a `sealCredentials(obj)`/`openCredentials(obj)` pair walks the object.

**Testing**:
- `Unit: encrypt → decrypt round-trips arbitrary UTF-8`.
- `Unit: tampering with ciphertext → decrypt throws (auth tag failure)`.
- `Unit: sealCredentials({apiKey:"x"}) → openCredentials yields {apiKey:"x"}`.

#### 1.5 — Append-only audit log

**What**: An `audit_events` table and a writer capturing every significant state change (inspired by Suggestion 2's event log, used as an audit trail rather than the source of truth).

**Design**:
```sql
CREATE TABLE audit_events (
  id          BIGSERIAL PRIMARY KEY,
  tenant_id   UUID NOT NULL,
  entity_type VARCHAR(50) NOT NULL,   -- 'order','purchase_order','shipment','price','inventory','return'
  entity_id   UUID,
  event_type  VARCHAR(100) NOT NULL,  -- 'CustomerOrderReceived','SupplierPriceChanged',...
  data        JSONB NOT NULL,
  correlation_id UUID,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_entity ON audit_events(entity_type, entity_id, created_at);
CREATE INDEX idx_audit_tenant ON audit_events(tenant_id, created_at);
```
```typescript
// packages/core/src/audit.ts
export async function recordEvent(e: {
  tenantId: string; entityType: string; entityId?: string;
  eventType: string; data: unknown; correlationId?: string;
}): Promise<void>;
```

**Testing**:
- `Integration: recordEvent → row present with correct event_type and JSONB data`.
- `Unit: recordEvent never throws on serialisable data; logs and swallows on DB error (audit must not break business flow)`.

---

## Phase 2: Tenancy, Auth & Core CRUD API

### Purpose
Stand up the Fastify server with tenant-scoped authentication and CRUD for the foundational entities (tenants, suppliers, products, storefront connections). This gives an operator a way to model their business in the system before any automation runs, and produces the auto-generated OpenAPI spec that all integrations will reference.

### Tasks

#### 2.1 — Fastify server, plugins, OpenAPI

**What**: Boot a Fastify app with db, logging, error handling, and Swagger plugins.

**Design**:
- `plugins/db.ts` decorates `app.db` with the Drizzle client.
- `plugins/error-handler.ts` maps Zod errors → 422, `NotFoundError` → 404, `AuthError` → 401, unknown → 500 with a correlation id.
- `@fastify/swagger` + `@fastify/swagger-ui` expose `GET /openapi.json` (OpenAPI 3.1) and `/docs`. Every route declares Zod-derived JSON Schema for params/body/response.
- `GET /health` returns `{ status, db: 'up'|'down', redis: 'up'|'down' }`.

**Testing**:
- `Integration: GET /health → 200 with db/redis up (Testcontainers)`.
- `Integration: GET /openapi.json → valid OpenAPI 3.1 document containing declared routes`.
- `Integration: route throwing ZodError → 422 with field names in body`.

#### 2.2 — Tenant API-key authentication

**What**: API-key auth scoping every request to a tenant.

**Design**:
- `api_keys` table: `id`, `tenant_id`, `key_hash` (sha256), `name`, `last_used_at`, `created_at`, `revoked_at`.
- `Authorization: Bearer <key>` → hash → lookup → set `request.tenantId`. Missing/invalid/revoked → 401.
- `POST /v1/tenants` (bootstrap, unauthenticated in self-host or admin-guarded) creates a tenant and returns a one-time plaintext API key.
- All `/v1/*` routes except tenant bootstrap and `/health` require auth via a preHandler.

**Testing**:
- `Integration: request with valid key → request.tenantId set, 200`.
- `Integration: revoked key → 401`.
- `Integration: tenant A key cannot read tenant B's product (404, not 403, to avoid existence disclosure)`.

#### 2.3 — Suppliers, products, storefronts CRUD

**What**: REST endpoints for the model-your-business entities.

**Design**: Standard tenant-scoped CRUD.
```
POST   /v1/suppliers           body: { name, feedType, connectionConfig, fieldMapping, contactInfo? }
GET    /v1/suppliers           list (paginated, ?limit&cursor)
GET    /v1/suppliers/:id
PATCH  /v1/suppliers/:id
POST   /v1/storefronts         body: { platform, storeUrl, credentials, platformConfig }
GET    /v1/storefronts/:id     (credentials never returned in responses)
POST   /v1/products            body: { sku, title, description?, category?, attributes?, basePrice? }
GET    /v1/products            ?category=&q=  (q does ILIKE on title; category exact)
PATCH  /v1/products/:id
POST   /v1/products/:id/mappings   body: { supplierId, supplierSku, supplierPrice, stockQuantity, leadTimeDays? }
```
`connectionConfig`/`credentials` validated by a per-platform/per-feed-type Zod discriminated union; secrets sealed via `sealCredentials` before insert. Cursor pagination on `created_at, id`.

**Testing**:
- `Unit: CSV supplier connectionConfig missing 'url' → 422`.
- `Integration: create supplier with credentials → DB row has sealed (encrypted) values; GET response omits them`.
- `Integration: create product then mapping → mapping linked, unique (product,supplier) enforced (duplicate → 409)`.
- `Integration: list products with cursor → stable pagination, no dupes across pages`.

---

## Phase 3: Supplier Catalogue Sync (Core Value #1)

### Purpose
The heart of the platform: ingest supplier feeds in any format, map their fields to the canonical model, and keep `supplier_product_mappings` (price, stock) current. This phase delivers the "never oversell" guarantee by deactivating products when stock hits zero, and records every price and inventory change. After this phase, an operator's catalogue stays in sync with suppliers automatically.

### Tasks

#### 3.1 — Feed connector interface & parsers

**What**: A uniform connector interface plus concrete parsers for CSV, XML, API, FTP/SFTP, and EDI.

**Design**:
```typescript
// packages/suppliers/src/connectors/index.ts
export interface RawFeedRecord { [field: string]: string | number | null; }
export interface FeedConnector {
  readonly type: 'csv' | 'xml' | 'api' | 'ftp' | 'edi';
  fetch(config: unknown): AsyncIterable<RawFeedRecord>;  // streams records
}
```
- `csv.ts`: `csv-parse` streaming, honours `delimiter`, `encoding`, `skipRows`.
- `xml.ts`: `fast-xml-parser`; `recordPath` (e.g. `Catalog.Products.Product`) selects the repeated node.
- `api.ts`: undici; `baseUrl`, `authType` (bearer/basic/none), pagination strategy (`pageParam`/`cursorPath`), `rateLimit`.
- `ftp.ts`: `basic-ftp` / `ssh2-sftp-client`; downloads file then delegates to csv/xml parser.
- `edi.ts`: `node-x12`; parses inventory/price advice; MVP supports the common 846 (inventory) and 832 (price/sales catalog) hooks behind the same `RawFeedRecord` shape.
- Each connector streams to bound memory (no full-feed buffering).

**Testing**:
- `Fixture: parse sample 1,000-row CSV → 1,000 RawFeedRecords with expected fields`.
- `Fixture: malformed XML → throws FeedParseError with line context`.
- `Fixture: paginated mock API (msw, 3 pages) → all records across pages yielded`.
- `Unit: CSV with skipRows:1 ignores header preamble row`.

#### 3.2 — Field mapper

**What**: Transform a `RawFeedRecord` into canonical fields using the supplier's `fieldMapping`.

**Design**:
```typescript
// canonical fields: sku, title, price, stockQuantity, currency, leadTimeDays, weightGrams, ...
export interface CanonicalProduct {
  supplierSku: string; title?: string; priceCents: number;
  stockQuantity: number; currency: string; leadTimeDays?: number;
  raw: RawFeedRecord;     // preserved into supplier_product_mappings.raw_supplier_data
}
export function mapRecord(raw: RawFeedRecord, mapping: FieldMapping): CanonicalProduct;
```
`FieldMapping` = `{ canonicalField: sourceFieldName }` plus optional transforms (`{ field, transform: 'cents'|'trim'|'currencySymbolStrip' }`). Missing required source field → `MappingError` collected per-record (not fatal to the whole feed).

**Testing**:
- `Unit: mapping {sku:"item_no", price:"unit_cost"} → CanonicalProduct with correct values`.
- `Unit: price "$28.50" with cents+symbolStrip transform → 2850`.
- `Unit: missing required 'sku' field → MappingError naming the record`.

#### 3.3 — Sync service (diff & upsert, stock-out deactivation)

**What**: Compare incoming canonical records to current `supplier_product_mappings`; upsert changes; record price/inventory deltas; deactivate stock-out products.

**Design**:
Algorithm per feed run:
1. Open a `sync_logs` row (`status='running'`, direction `inbound`).
2. Stream records; for each, `mapRecord`; collect `MappingError`s into `error_details`.
3. Upsert `supplier_product_mappings` by `(product_id via supplier_sku resolution, supplier_id)`; store `raw_supplier_data`.
4. If `supplier_price_cents` changed → append `price_changes` (compute `change_pct`; flag anomaly per Phase 6 hook, default false) and `recordEvent('SupplierPriceChanged')`.
5. If `stock_quantity` changed → append `inventory_updates` (`source='feed_sync'`), update Redis `inv:{mappingId}` hot key.
6. If a product's stock across **all** active suppliers hits 0 → set `products.is_active=false`, enqueue `storefront.deactivate` jobs, `recordEvent('ProductDeactivatedDueToStockout')`.
7. Close `sync_logs` (`success`/`partial`/`failed`, counts, `duration_ms`).
Idempotent: re-running an identical feed produces zero changes and zero new history rows.

**Testing**:
- `Integration: feed lowers a SKU's stock 10→0 (sole supplier) → product.is_active=false, deactivate job enqueued, inventory_updates row written`.
- `Integration: feed raises price 28.50→31.00 → price_changes row with change_pct≈8.77, audit event`.
- `Integration: identical feed re-run → 0 price_changes, 0 inventory_updates added (idempotent)`.
- `Integration: 50-record feed with 3 unmappable rows → sync_logs.status='partial', items_failed=3`.

#### 3.4 — Scheduled feed polling (BullMQ)

**What**: Repeatable jobs that poll each active supplier feed on its configured interval.

**Design**:
- On supplier create/update, register a BullMQ repeatable job keyed by `supplierId` with `every: pollingIntervalSec*1000` (from `connectionConfig.pollingIntervalSec`, default 3600; min enforced 60).
- Processor `feed-poll`: resolve connector by `feedType`, run sync service, retries `3` with exponential backoff, `removeOnComplete: 100`.
- Concurrency limited per supplier (no overlapping runs for the same supplier — use a Redis lock `lock:feed:{supplierId}`).
- Manual trigger endpoint: `POST /v1/suppliers/:id/sync` enqueues an immediate job.

**Testing**:
- `Integration (Testcontainers redis): registering supplier creates a repeatable job with correct cron/every`.
- `Integration: overlapping triggers for same supplier → second waits/skips on lock`.
- `Integration: POST /v1/suppliers/:id/sync → job enqueued, 202 with jobId`.
- `Integration: connector throws → job retried, sync_logs shows failed after final attempt`.

---

## Phase 4: Storefront Integration & Order Ingestion (Core Value #2)

### Purpose
Connect to the storefront platforms (Shopify, WooCommerce, BigCommerce, Wix, eBay) to ingest customer orders in near-real-time via webhooks (with polling fallback) and to push catalogue/price/stock changes outward. After this phase, real customer orders flow into the system the moment they are placed.

### Tasks

#### 4.1 — ChannelAdapter interface

**What**: A common interface every storefront adapter implements.

**Design**:
```typescript
export interface ChannelAdapter {
  readonly platform: 'shopify' | 'woocommerce' | 'bigcommerce' | 'wix' | 'ebay';
  verifyWebhook(headers: Record<string,string>, rawBody: Buffer, secret: string): boolean;
  parseOrderWebhook(rawBody: Buffer): NormalizedOrder;
  fetchOrdersSince(cursor: string | null): AsyncIterable<NormalizedOrder>; // polling fallback
  pushTracking(listingRef: string, externalOrderId: string, tracking: TrackingInfo): Promise<void>;
  updateListingPrice(listingRef: string, priceCents: number): Promise<void>;
  setListingActive(listingRef: string, active: boolean): Promise<void>;
}
export interface NormalizedOrder {
  externalOrderId: string;
  lineItems: { externalLineId: string; sku: string; quantity: number; unitPriceCents: number }[];
  customerInfo: { name?: string; email?: string; phone?: string };
  shippingAddress: { line1: string; line2?: string; city: string; state?: string; postalCode: string; country: string };
  orderTotalCents: number; currency: string; orderedAt: string;
  channelMetadata: Record<string, unknown>;
}
```

**Testing**:
- `Unit: each adapter satisfies the interface (type-level + a conformance test array iterating adapters)`.

#### 4.2 — Shopify & WooCommerce adapters (MVP platforms)

**What**: Implement two adapters fully; stub the other three with `NotImplemented` so the interface is exercised.

**Design**:
- **Shopify**: webhook verify = HMAC-SHA256 of raw body with shared secret, base64, compared to `X-Shopify-Hmac-Sha256` (constant-time). OAuth 2.0 token in `credentials`. `pushTracking` → Fulfillment API; `updateListingPrice` → Product Variant API.
- **WooCommerce**: webhook verify = HMAC-SHA256 with `X-WC-Webhook-Signature`. REST API with consumer key/secret (basic auth over HTTPS).
- Rate-limit handling: respect `429` + `Retry-After`, back off, surface as retryable job errors.

**Testing**:
- `Fixture: real Shopify order webhook payload + correct HMAC → verifyWebhook true; tampered body → false`.
- `Fixture: Shopify payload → NormalizedOrder with line items, address, total mapped`.
- `Integration (msw): pushTracking issues correct Fulfillment API call with tracking number`.
- `Integration (msw): 429 then 200 → call retried and succeeds`.

#### 4.3 — Webhook ingestion endpoint & idempotency

**What**: A single hardened endpoint to receive storefront order webhooks.

**Design**:
- `POST /v1/webhooks/:storefrontId` — reads **raw** body (Fastify raw-body plugin), resolves storefront → adapter → `verifyWebhook`. Invalid signature → `401`, nothing enqueued.
- Idempotency: dedupe key `wh:{storefrontId}:{externalOrderId}` in Redis (TTL 24h). Duplicate → `200` no-op.
- Valid+new → persist `customer_orders` + `order_line_items` (status `received`), `recordEvent('CustomerOrderReceived')`, enqueue `order-route` job, return `200` fast (< 50ms target; heavy work is in the job).

**Testing**:
- `Integration: valid signature, new order → 200, order row created (status received), order-route job enqueued`.
- `Integration: invalid signature → 401, no row, no job`.
- `Integration: same webhook twice → one order row, second returns 200 no-op`.
- `Integration: malformed body but valid signature → 422, order not created, error logged`.

#### 4.4 — Polling fallback & outbound push worker

**What**: Scheduled order polling for platforms/setups without webhooks, and a worker that performs outbound pushes (price, stock, tracking).

**Design**:
- Repeatable `order-poll` job per storefront (default every 5 min) calling `fetchOrdersSince(last_sync_cursor)`; same idempotent persistence path as webhooks.
- `storefront-push` queue with job types `tracking`, `price`, `deactivate`; processor resolves adapter and calls the matching method; retries with backoff; updates `storefront_listings.last_pushed_at`.

**Testing**:
- `Integration (msw): order-poll ingests 2 new orders, skips 1 already-seen`.
- `Integration: deactivate job (from Phase 3.3) → setListingActive(false) called`.
- `Integration: push failure → retried; permanent failure recorded in sync_logs`.

---

## Phase 5: Fulfilment Routing & Auto-Ordering (Core Value #3)

### Purpose
Turn ingested orders into supplier purchase orders within seconds, choosing the optimal supplier per line item and splitting multi-supplier orders. This is the operational core that lets a retailer scale without manual order placement. After this phase, an order placed on a storefront results automatically in confirmed POs to the right suppliers.

### Tasks

#### 5.1 — Routing engine & scorer

**What**: Select the best supplier for each line item using real-time stock, cost, lead time, and reliability, honouring tenant `routing_rules`.

**Design**:
```typescript
export interface RoutingCandidate {
  supplierId: string; supplierSku: string; costCents: number;
  stockQuantity: number; leadTimeDays: number; reliabilityScore: number;
}
export interface RoutingDecision {
  lineItemId: string; selectedSupplierId: string;
  reason: 'lowest_cost' | 'fastest_shipping' | 'preferred_supplier' | 'fallback' | 'ai_recommended';
  costCents: number; estimatedShipDays: number; alternativesConsidered: number;
}
export function routeLineItem(item, candidates, rules): RoutingDecision | null; // null = unfulfillable
```
Scoring (lower = better, weights configurable per tenant, defaults shown):
`score = 0.4*normCost + 0.4*normLeadTime + 0.2*(1 - reliabilityScore)`.
Rule evaluation: `routing_rules.conditions` (JSONB) matched against order context (country, category, order value); first matching rule by `priority` may force `preferred_supplier` with a `fallback` strategy. Candidates filtered to `stockQuantity >= qty`. If preferred is out of stock → fallback strategy. If no candidate has stock → decision `null` (line item flagged `unfulfillable`).

> Post-MVP: this scorer is the natural seam to swap in the Neo4j graph routing from Data Model Suggestion 4; keep the candidate-fetch behind a `getCandidates(productId, region, qty)` function so the source can change without touching the scorer.

**Testing**:
- `Unit: two suppliers, A cheaper+slower, B pricier+faster, default weights → A wins (cost+lead balanced)`.
- `Unit: preferred-supplier rule with stock → preferred chosen, reason 'preferred_supplier'`.
- `Unit: preferred-supplier out of stock, fallback 'lowest_cost' → cheapest in-stock chosen, reason 'fallback'`.
- `Unit: no supplier has stock → null (unfulfillable)`.

#### 5.2 — Order routing job & PO creation

**What**: The `order-route` processor that runs the engine and creates supplier POs (split per supplier), transactionally.

**Design**:
- Processor loads order + line items, fetches candidates per item (joining `supplier_product_mappings` with live Redis stock), runs `routeLineItem` for each.
- Groups decisions by `selectedSupplierId` → one `supplier_purchase_orders` row per supplier with its `purchase_order_line_items`. `routing_decision` JSONB snapshot stored on the PO.
- Wrapped in a single DB transaction; decrements reserved stock in Redis atomically (Lua script) to prevent double-allocation across concurrent orders.
- Sets `customer_orders.status` → `routed` (or `partially_unfulfillable` if any null). `recordEvent('OrderRoutingDecisionMade')` + `recordEvent('SupplierPurchaseOrderCreated')` per PO. Enqueues `po-submit` jobs.

**Testing**:
- `Integration: order with items from 2 suppliers → 2 POs created with correct line items, order status 'routed'`.
- `Integration: concurrent orders for last unit → exactly one PO gets it (Redis Lua reservation), other line flagged unfulfillable`.
- `Integration: transaction rollback on mid-creation error → no partial POs persisted`.

#### 5.3 — PO submission to suppliers

**What**: Submit POs to suppliers over their connection type and record confirmation.

**Design**:
- `po-submit` processor resolves the supplier connection: `api` → POST order to supplier endpoint; `edi` → emit X12 850 over the configured transport; `email`/manual → render and send an order email; `ftp` → drop an order file.
- On success: `supplier_purchase_orders.status='submitted'`, store `submission_details` JSONB (request id / ISA control / message id), then on async confirmation → `confirmed` + `supplier_order_ref`. `recordEvent('SupplierPurchaseOrderConfirmed')`.
- Target: order-received → PO-submitted in < 60s (assert in an e2e timing test, generously bounded in CI).

**Testing**:
- `Integration (msw): API supplier → POST payload matches PO; 200 → status 'submitted', submission_details captured`.
- `Unit: EDI 850 generation for a PO → valid X12 envelope (parse round-trip with node-x12)`.
- `Integration: supplier returns 500 → job retried; final failure → PO status 'failed', alert event`.
- `E2E: webhook → routed → submitted within timing budget`.

---

## Phase 6: Price Monitoring, Margin Protection & Anomaly Detection

### Purpose
Protect margins automatically: detect supplier cost changes (already captured in `price_changes`), apply tenant markup rules to reprice storefront listings, and flag anomalous cost shifts before they silently erode profit. After this phase, storefront prices track supplier costs per the operator's rules, and suspicious changes raise alerts.

### Tasks

#### 6.1 — Margin rule engine

**What**: Compute a target storefront price from a supplier cost and the tenant's `margin_rules`.

**Design**:
```typescript
export type MarginAction =
  | { type: 'percentage_markup'; value: number }
  | { type: 'fixed_markup'; value: number }          // dollars; convert to cents
  | { type: 'min_margin'; value: number; fallback?: 'deactivate' };
export function computePrice(costCents: number, rules: MarginRule[], ctx: PriceContext):
  { priceCents: number; ruleApplied: string } | { action: 'deactivate'; ruleApplied: string };
```
Rules ordered by `priority`; first whose JSONB `conditions` match `ctx` (category, supplierId, price range) wins. `min_margin` with `fallback:'deactivate'` returns a deactivate instruction when no price satisfies the floor.

**Testing**:
- `Unit: cost 2850, percentage_markup 40 → 3990, ruleApplied named`.
- `Unit: conditions {category:'electronics'} matches electronics product, skips others`.
- `Unit: min_margin floor unsatisfiable with fallback deactivate → deactivate instruction`.

#### 6.2 — Repricing pipeline

**What**: When a supplier price changes, recompute the storefront price and push it.

**Design**:
- The Phase 3.3 price-change hook enqueues a `reprice` job for affected `storefront_listings`.
- Processor: load active margin rules, `computePrice`, and if the new price differs from `storefront_listings.listed_price` by more than a configurable epsilon → enqueue `storefront-push` `price` job, update `listed_price`, `recordEvent('StorefrontPriceAdjusted')`. Deactivate instruction → `deactivate` push.

**Testing**:
- `Integration: supplier cost rise → reprice job → listing price updated and price push enqueued`.
- `Integration: cost change below epsilon → no push (avoids churn)`.

#### 6.3 — Price anomaly detection

**What**: Flag price changes that deviate from a SKU's normal pattern.

**Design**:
- Statistical baseline: rolling z-score over the last N `price_changes` for the mapping; `|z| > 3` OR single-step `change_pct > configurable threshold (default 25%)` → `anomaly_flag=true`, suppress auto-reprice, enqueue an alert and **hold** the listing at its current price pending operator review.
- AI explanation (optional, `AI_ENABLED`): when flagged, call the LLM to produce a short human-readable rationale stored on the alert. Prompt template in `ai/prompts.ts`:
  > "A supplier's cost for SKU {sku} changed from {old} to {new} ({pct}%). Recent cost history: {series}. In 2 sentences, state whether this looks like a data/feed error, a genuine cost change, or a currency artefact, and recommend hold or accept."

**Testing**:
- `Unit: stable history then +60% jump → anomaly_flag true, reprice suppressed`.
- `Unit: gradual 2% drift over many records → not flagged`.
- `Integration (mocked LLM): flagged change → alert row contains LLM rationale; with AI_ENABLED=false → alert created without rationale`.

---

## Phase 7: Tracking Synchronisation & Returns

### Purpose
Close the fulfilment loop: import tracking numbers from suppliers, push them to storefronts and customers, and automate the returns/RMA path that incumbents leave manual. After this phase, customers see tracking automatically and returns flow through supplier RMAs and refunds without manual chasing.

### Tasks

#### 7.1 — Tracking ingestion & storefront push

**What**: Receive shipment/tracking data from suppliers and propagate it.

**Design**:
- Supplier tracking arrives via webhook (`POST /v1/webhooks/supplier/:supplierId`, signature-verified) or via the next feed poll (EDI 856 ASN / API). Creates/updates `shipments` (carrier, tracking number, url), appends to `tracking_timeline` JSONB, `recordEvent('ShipmentCreated'|'TrackingEventReceived')`.
- Enqueues `storefront-push` `tracking` job → `pushTracking` on the channel adapter; on success set `pushed_to_storefront=true`, `recordEvent('TrackingPushedToStorefront')`.

**Testing**:
- `Integration: supplier tracking webhook → shipment row + timeline entry + storefront tracking push enqueued`.
- `Fixture: EDI 856 ASN parsed → shipment with tracking number`.
- `Integration: storefront push success → pushed_to_storefront=true`.

#### 7.2 — Returns & supplier RMA

**What**: Model customer returns, request supplier RMAs, and process refunds.

**Design**:
```
POST /v1/orders/:id/returns   body: { lineItems:[{lineItemId,quantity,reason}], customerNotes? }
```
- Creates `order_returns` (status `requested`) + `return_details` JSONB; `recordEvent('ReturnRequested')`.
- Enqueues `rma-request` job → for each involved supplier, call its RMA mechanism (API/email), create `supplier_return_authorisations` (status `pending` → `issued` with `rma_number`), `recordEvent('SupplierRMAIssued')`.
- Refund step (`refund` job): once RMA issued/approved, compute `refund_amount`, mark return `refunded`, `recordEvent('RefundProcessed')`. (Actual payment capture/refund is delegated to the storefront platform's API where supported; otherwise recorded for operator action.)
- State machine: `requested → rma_pending → rma_issued → refunded` (+ `rejected`).

**Testing**:
- `Integration: create return → return row + rma-request job enqueued`.
- `Integration (msw): supplier RMA API returns number → authorisation status 'issued' with rma_number`.
- `Unit: return state machine rejects invalid transition (requested → refunded skipping rma)`.

---

## Phase 8: AI-Native Intelligence — Stock-Out Prediction & Supplier Scoring

### Purpose
Deliver the differentiator no incumbent offers: learn from the history captured in `inventory_updates`, `price_changes`, and shipment outcomes to predict stock-outs before they happen, score supplier reliability dynamically, and feed those signals back into routing. After this phase, the platform shifts from reactive automation to proactive operations management.

### Tasks

#### 8.1 — Supplier reliability scoring

**What**: Compute a rolling `reliability_score` per supplier from fulfilment outcomes.

**Design**:
- Scheduled `supplier-scoring` job (daily) aggregates per supplier over a trailing window: on-time delivery % (delivered_at vs estimated), confirmation latency, return rate, price stability (variance of `change_pct`). Combine into `reliability_score ∈ [0,1]` and `avg_ship_days`; write back to `suppliers`. `recordEvent` for auditability.
- These values feed the Phase 5 scorer directly (no schema change needed).

**Testing**:
- `Integration: supplier with 96% on-time, low returns → high score; supplier with late shipments → lower score`.
- `Unit: scoring is deterministic given the same input window`.

#### 8.2 — Stock-out prediction

**What**: Predict which supplier-product mappings will hit zero stock within a horizon.

**Design**:
- `LlmProvider` interface in `ai/provider.ts`; default Anthropic impl; usable only when `AI_ENABLED`.
- MVP predictor is a transparent statistical model (no black box): per mapping, fit recent depletion rate from `inventory_updates` (units/day via linear regression over the trailing window), project `days_to_zero`. If `< horizon` (default 3 days) → emit a `stockout_risk` signal (severity by proximity). Optional LLM layer summarises replenishment patterns and writes a narrative recommendation ("supplier restocks ~weekly; risk high before weekend").
- Signals surface via `GET /v1/insights/stockout-risks` and bias routing (lower score for at-risk suppliers).

**Testing**:
- `Unit: steady -10 units/day from 30 → days_to_zero=3, flagged at horizon 3`.
- `Unit: flat/replenishing stock → not flagged`.
- `Integration (mocked LLM): flagged mapping → narrative present; AI disabled → numeric signal only`.

#### 8.3 — AI-assisted supplier feed field mapping

**What**: When onboarding a new supplier, propose a `fieldMapping` from a feed sample.

**Design**:
- `POST /v1/suppliers/:id/suggest-mapping` with a sample feed (or fetch first N records). Builds a prompt listing detected source columns + sample values and the canonical target fields; LLM returns a proposed `FieldMapping` (validated by Zod before being offered). Operator reviews/accepts; nothing auto-applied.
- Prompt template kept in `ai/prompts.ts`; deterministic JSON output enforced via tool/output schema.

**Testing**:
- `Integration (mocked LLM): sample columns [item_no, qty_avail, unit_cost] → proposed mapping {sku:item_no, stockQuantity:qty_avail, price:unit_cost}, schema-valid`.
- `Unit: LLM returns invalid JSON / unknown canonical field → rejected with clear error, no mapping applied`.

---

## Phase 9: Operator Dashboard (Web UI)

### Purpose
Provide a single interface — the README's "manage operations across channels from a single interface" promise — for operators to connect suppliers/storefronts, watch sync status, review routing decisions, handle anomalies and returns, and see margin/stock insights. Until now everything is API-only; this makes it usable by non-developers.

### Tasks

#### 9.1 — Frontend app scaffold

**What**: A React + Vite SPA served behind the API, talking to `/v1` with the OpenAPI types.

**Design**:
- React 19 + Vite + TanStack Query + Tailwind. Types generated from `openapi.json` via `openapi-typescript`. Auth via API key stored in an httpOnly-cookie-backed session (login exchanges key → session).
- Pages: Dashboard (KPIs), Suppliers, Storefronts, Products, Orders, Anomalies/Alerts, Returns, Insights.

**Testing**:
- `Unit: generated API client typechecks against openapi.json`.
- `E2E (Playwright): login → dashboard renders KPI cards from API`.

#### 9.2 — Core operator screens

**What**: Implement the supplier/storefront connect flows, order/PO timeline, anomaly review, and returns handling.

**Design**:
- Supplier connect wizard uses the Phase 8.3 mapping suggestion. Order detail shows the routing decision + PO + shipment timeline (from `audit_events` and entity tables). Anomaly screen lets an operator accept/hold a flagged price. Returns screen drives the Phase 7.2 state machine.

**Testing**:
- `E2E (Playwright, mocked API): add CSV supplier via wizard → supplier appears in list`.
- `E2E: open an order → see routing reason and PO statuses`.
- `E2E: accept a held anomaly → reprice push triggered (API called)`.

---

## Phase 10: Hardening, Observability & Deployment

### Purpose
Make the platform production-ready for both self-hosted and cloud deployment: rate limiting, retries/dead-letter handling, structured tracing, backups, and a clean one-command deploy. After this phase the platform can be operated reliably at scale.

### Tasks

#### 10.1 — Resilience & dead-letter handling

**What**: Standardise retry/backoff and capture permanently failed jobs.

**Design**:
- All queues: exponential backoff, max attempts, `failed` events route to a `dead-letter` queue persisted in a `failed_jobs` table with payload + last error, surfaced in the dashboard for manual replay (`POST /v1/jobs/:id/replay`).
- Outbound calls wrapped in a circuit breaker per external host; open breaker → fast-fail + alert.

**Testing**:
- `Integration: job failing past max attempts → row in failed_jobs, replayable`.
- `Unit: circuit breaker opens after N consecutive failures, half-opens after cooldown`.

#### 10.2 — Observability & rate limiting

**What**: Tracing, metrics, and protective rate limits.

**Design**:
- OpenTelemetry spans: webhook → job → external call share a `correlation_id` (also written to `audit_events`). pino logs include it.
- `@fastify/rate-limit` per tenant API key. Per-supplier and per-storefront outbound rate limiters honour documented platform limits.
- `/metrics` Prometheus endpoint (queue depths, sync success rate, p95 order-to-PO latency).

**Testing**:
- `Integration: a webhook traced end-to-end shares one correlation_id across order row, job, and audit events`.
- `Integration: exceeding tenant rate limit → 429 with Retry-After`.

#### 10.3 — Deployment & data lifecycle

**What**: Production Docker/compose, migrations on deploy, partition maintenance, backups.

**Design**:
- `docker-compose.prod.yml` with separate api/worker scaling, healthchecks, and a migration init container (`drizzle-kit migrate`) that runs before app start.
- Scheduled job creates next-month partitions for `price_changes`/`inventory_updates` and archives partitions older than 90 days (research's lean-operational-DB guidance).
- Documented backup/restore (pg_dump + WAL) and a `make deploy` target. `.env.example` documents every variable from `env.ts`.

**Testing**:
- `Integration: migration init container brings a fresh DB fully up to head`.
- `Integration: partition-maintenance job creates next month's partition and detaches a 91-day-old one`.
- `E2E (smoke): docker compose -f docker-compose.prod.yml up → /health green, a seeded webhook produces a confirmed PO`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (skeleton, config, DB schema, crypto, audit)
    │  required by everything
    ▼
Phase 2: Tenancy, Auth & Core CRUD API
    │
    ▼
Phase 3: Supplier Catalogue Sync ─────────────┐ (core value #1)
    │                                          │
    ▼                                          │
Phase 4: Storefront Integration & Orders ──────┤ (can begin once Phase 2 done;
    │                                          │  full e2e needs Phase 3 stock data)
    ▼                                          │
Phase 5: Routing & Auto-Ordering ──────────────┘ (requires 3 + 4)
    │
    ├──► Phase 6: Pricing & Anomaly (requires 3; parallel with 7)
    ├──► Phase 7: Tracking & Returns (requires 4,5; parallel with 6)
    │
    ▼
Phase 8: AI Intelligence (requires history from 3,5,7)
    │
    ▼
Phase 9: Operator Dashboard (requires 2-7 APIs; 8 enhances it)
    │
    ▼
Phase 10: Hardening, Observability & Deployment (cross-cutting; finalises all)
```

**Parallelism opportunities**
- Phases **6** and **7** can be developed concurrently once Phases 3–5 are complete (independent: pricing vs tracking/returns).
- Within Phase 4, individual **channel adapters** (Shopify, WooCommerce, then BigCommerce/Wix/eBay) can be built in parallel against the shared `ChannelAdapter` interface.
- Within Phase 3, individual **feed connectors** (CSV, XML, API, FTP, EDI) can be built in parallel against the shared `FeedConnector` interface.
- Phase **9 (frontend)** can begin as soon as the Phase 2 OpenAPI spec exists and progress alongside backend phases, mocking unfinished endpoints.

---

## Definition of Done (per phase)

A phase is complete only when **all** of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`pnpm test`).
3. Biome lint and format pass (`pnpm lint`).
4. `tsc --noEmit` passes in strict mode (`pnpm typecheck`).
5. Docker image builds and `docker compose up` reaches a healthy state.
6. The phase's headline capability works end-to-end (demonstrated by at least one integration or e2e test against Testcontainers Postgres + Redis and mocked external APIs).
7. Any new configuration variables are added to `env.ts` (with validation) and documented in `.env.example`.
8. Any new API endpoints appear in the auto-generated `GET /openapi.json` with request/response schemas.
9. Any schema changes have a generated, committed Drizzle migration that applies cleanly to a fresh database.
10. Significant new state transitions write to `audit_events` with a correlation id.
```
