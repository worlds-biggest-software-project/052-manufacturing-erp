# Manufacturing ERP - Comprehensive Phased Development Plan

> Project: Manufacturing ERP (Candidate #052)
> Created: 2026-05-25
> Status: Development Planning

---

## 1. Technology Decisions & Rationale

### 1.1 Data Model Selection: Hybrid Relational + JSONB (Data Model Suggestion 3)

**Decision:** Adopt Data Model Suggestion 3 (Hybrid Relational + JSONB) as the primary data model, with selective adoption of elements from Suggestions 1 and 2.

**Rationale:**

- **Suggestion 1 (Normalized Relational)** provides maximum data integrity but its 43-table schema introduces excessive rigidity for an early-stage project. Every new field requires a migration. However, its ISA-95 alignment and quality management table design are exemplary and will be adopted for the quality module.
- **Suggestion 2 (Event-Sourced CQRS)** is architecturally elegant for audit trails and AI event streaming, but introduces prohibitive write-path complexity for MVP. The event type catalog and AI subscription patterns will be adopted later (Phase 8) as an event bus layered on top of the relational core.
- **Suggestion 3 (Hybrid Relational + JSONB)** provides the fastest path to a working MVP: 21 tables, relational columns for all MRP/scheduling-critical fields, JSONB for tenant-specific and compliance-specific variation. The core manufacturing fields (quantities, dates, statuses, foreign keys) remain typed and constrained. Variable data (compliance, machine specs, custom fields) uses JSONB with application-level JSON Schema validation.
- **Suggestion 4 (Graph-Relational)** adds unnecessary complexity at MVP but its graph query patterns for where-used analysis, lot genealogy, and supplier impact analysis are compelling for later phases. A graph query layer will be introduced in Phase 9 once the BOM and traceability data exists.

**Selective adoptions from other models:**

| Adopted From | What | When |
|---|---|---|
| Suggestion 1 | Separate `inspection_template_check` and `inspection_result` tables (relational, not JSONB) for quality module | Phase 4 |
| Suggestion 1 | `engineering_change_order` and `eco_affected_item` tables | Phase 5 |
| Suggestion 2 | Event bus for AI agent subscriptions; `event_store` table (append-only, partitioned) alongside relational tables | Phase 8 |
| Suggestion 2 | Event type catalog for production, quality, telemetry domains | Phase 8 |
| Suggestion 4 | `graph_node` / `graph_edge` tables for where-used analysis and lot genealogy | Phase 9 |

### 1.2 Technology Stack

| Layer | Technology | Rationale |
|---|---|---|
| **Language** | TypeScript (backend + frontend) | Single language across stack; strong typing; large talent pool; excellent PostgreSQL ecosystem (Drizzle ORM, Prisma) |
| **Backend Framework** | Node.js with Fastify | High-performance HTTP server; first-class OpenAPI/Swagger support via `@fastify/swagger`; plugin architecture maps well to ERP module boundaries |
| **Database** | PostgreSQL 16+ | Industry-leading hybrid relational-document database; JSONB with GIN indexes; recursive CTEs for BOM explosion; row-level security for multi-tenancy; time-series partitioning for telemetry |
| **ORM / Query Builder** | Drizzle ORM | Type-safe SQL; schema-as-code migrations; supports JSONB operators; lighter than Prisma for complex manufacturing queries |
| **Frontend** | React 19 + Next.js 15 (App Router) | Server Components for dashboard performance; streaming for real-time shop floor views; widely adopted; strong component ecosystem |
| **UI Components** | shadcn/ui + Tailwind CSS 4 | Accessible, composable component primitives; Tailwind for rapid UI iteration; good tablet/kiosk responsive patterns |
| **Authentication** | Auth.js (NextAuth v5) + JWT | OAuth 2.0 / OIDC compliant; supports API key auth for shop floor kiosks and IoT gateways; JWT tokens per RFC 7519 |
| **API Protocol** | REST with OpenAPI 3.1 | Machine-readable API spec enables AI agent integration and SDK code generation; Fastify generates OpenAPI from route schemas; RFC 7807 error responses |
| **Real-time** | Server-Sent Events (SSE) + WebSocket (via Socket.IO) | SSE for shop floor dashboards (one-way); WebSocket for operator chat interface and live scheduling updates |
| **IoT Connectivity** | node-opcua (OPC-UA client) + custom MTConnect HTTP poller | node-opcua is the most mature OPC-UA library for Node.js; MTConnect uses standard HTTP/XML polling per the MTConnect spec |
| **AI Integration** | Anthropic Claude API (via @anthropic-ai/sdk) | Scheduling agent, BOM assistant, quality predictor, and NL operator interface all use Claude as the reasoning engine; MCP server exposes ERP data to AI agents |
| **Message Queue** | BullMQ (Redis-backed) | Job queue for MRP computation, AI scheduling runs, telemetry ingestion batching, and background report generation |
| **Caching** | Redis 7+ | Session cache, MRP intermediate results, equipment status cache for scheduler, rate limiting |
| **Search** | PostgreSQL full-text search (tsvector) | Sufficient for item/part number search; avoids adding Elasticsearch complexity at MVP |
| **Testing** | Vitest (unit) + Playwright (E2E) + Testcontainers (integration) | Vitest is fast and TypeScript-native; Testcontainers provides real PostgreSQL for integration tests; Playwright validates shop floor tablet UI |
| **CI/CD** | GitHub Actions | Standard; matrix builds for Node versions; Testcontainers in CI |
| **Container** | Docker + Docker Compose (dev) | Single `docker compose up` for local development with PostgreSQL, Redis, and the application |
| **Deployment** | Docker images; Kubernetes Helm chart for production | Self-hosted and cloud deployment targets; Helm chart for manufacturers with existing K8s infrastructure |
| **Licence** | MIT | Most permissive open-source licence; matches ERPNext; no copyleft restrictions for manufacturers who want to customize |

### 1.3 Project Structure

```
manufacturing-erp/
|-- packages/
|   |-- api/                    # Fastify REST API server
|   |   |-- src/
|   |   |   |-- modules/        # Domain modules (bom, mrp, work-order, quality, ...)
|   |   |   |-- middleware/     # Auth, tenant isolation, audit logging
|   |   |   |-- plugins/       # Fastify plugins (db, redis, auth)
|   |   |   |-- routes/        # OpenAPI route definitions
|   |   |   |-- services/      # Business logic layer
|   |   |   |-- db/
|   |   |   |   |-- schema/    # Drizzle schema definitions
|   |   |   |   |-- migrations/ # Database migrations
|   |   |   |   |-- seeds/     # Seed data for development
|   |   |-- test/
|   |-- web/                    # Next.js frontend
|   |   |-- src/
|   |   |   |-- app/           # Next.js App Router pages
|   |   |   |-- components/    # Shared UI components
|   |   |   |   |-- shop-floor/ # Tablet-optimized operator components
|   |   |   |   |-- planning/  # MRP and scheduling views
|   |   |   |   |-- quality/   # Inspection and NCR components
|   |   |   |-- hooks/         # React hooks for ERP data
|   |   |   |-- lib/           # API client, utilities
|   |-- iot-gateway/            # OPC-UA / MTConnect data collection service
|   |   |-- src/
|   |   |   |-- connectors/    # MTConnect poller, OPC-UA client
|   |   |   |-- processors/    # Telemetry normalization and batching
|   |-- ai-agents/              # AI scheduling, quality, BOM agents
|   |   |-- src/
|   |   |   |-- scheduler/     # Dynamic scheduling agent
|   |   |   |-- quality/       # Predictive quality agent
|   |   |   |-- bom-assistant/ # BOM construction assistant
|   |   |   |-- nl-interface/  # Natural language operator interface
|   |-- mcp-server/             # MCP server for AI agent access to ERP data
|   |   |-- src/
|   |   |   |-- tools/         # MCP tools (create_work_order, get_bom, etc.)
|   |   |   |-- resources/     # MCP resources (items, work_orders, etc.)
|   |-- shared/                 # Shared types, utilities, constants
|       |-- src/
|           |-- types/          # TypeScript type definitions
|           |-- constants/      # Status enums, ISA-95 constants
|           |-- utils/          # Date, currency, UOM conversion utilities
|-- docker/
|   |-- docker-compose.yml      # Local development stack
|   |-- Dockerfile.api
|   |-- Dockerfile.web
|   |-- Dockerfile.iot-gateway
|-- helm/                       # Kubernetes Helm chart
|-- docs/
|   |-- api/                    # Generated OpenAPI documentation
|   |-- architecture/           # Architecture decision records
|-- .github/
|   |-- workflows/              # CI/CD pipelines
|-- turbo.json                  # Turborepo configuration
|-- package.json                # Root workspace
```

### 1.4 Monorepo Strategy

The project uses a **Turborepo monorepo** with the following workspaces:

- `packages/api` - Backend API server
- `packages/web` - Frontend application
- `packages/iot-gateway` - IoT data collection service (deployed independently)
- `packages/ai-agents` - AI agent processes (deployed independently)
- `packages/mcp-server` - MCP server (deployed alongside API or independently)
- `packages/shared` - Shared types and utilities

This structure allows independent deployment of the IoT gateway and AI agents while sharing TypeScript types and business logic constants across all packages.

---

## 2. Phase Dependency Graph

```
Phase 1: Foundation & Infrastructure
  |
  v
Phase 2: Items, BOM & Routing --------+
  |                                     |
  v                                     v
Phase 3: Work Orders & Shop Floor   Phase 4: Quality Management
  |                                     |
  +------------------+------------------+
                     |
                     v
              Phase 5: MRP Engine
                     |
                     v
              Phase 6: Job Costing & Financials
                     |
          +----------+----------+
          |                     |
          v                     v
Phase 7: IoT Integration    Phase 8: AI Scheduling & Event Bus
          |                     |
          +----------+----------+
                     |
                     v
          Phase 9: AI Quality & BOM Assistant
                     |
                     v
          Phase 10: NL Operator Interface & MCP Server
                     |
                     v
          Phase 11: Advanced Features (ECO, CMMS, EDI)
                     |
                     v
          Phase 12: Hardening & Production Readiness
```

**Critical path:** Phases 1 -> 2 -> 3 -> 5 -> 6 form the critical path to a functional MVP (work order lifecycle with costing). Phases 4, 7, and 8 can be developed in parallel with critical path phases once their dependencies are met.

---

## 3. Phase Definitions

---

### Phase 1: Foundation & Infrastructure

**Goal:** Establish the development environment, database schema infrastructure, authentication, multi-tenancy, and API framework so that all subsequent phases build on a stable, tested foundation.

**Duration:** 3-4 weeks

**Dependencies:** None (starting phase)

#### Task 1.1: Repository & Tooling Setup

**What:** Initialize the Turborepo monorepo with all package workspaces, TypeScript configuration, linting, formatting, and CI pipeline.

**Design:**

```typescript
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["build"] },
    "test:integration": { "dependsOn": ["build"], "env": ["DATABASE_URL"] },
    "lint": {},
    "typecheck": { "dependsOn": ["^build"] },
    "db:migrate": { "cache": false },
    "db:seed": { "cache": false, "dependsOn": ["db:migrate"] }
  }
}
```

```yaml
# docker/docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: manufacturing_erp
      POSTGRES_USER: erp
      POSTGRES_PASSWORD: erp_dev
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

volumes:
  pgdata:
```

**Testing:**
- `turbo build` completes without errors for all packages
- `turbo lint` and `turbo typecheck` pass with zero warnings
- `docker compose up` starts PostgreSQL and Redis successfully
- GitHub Actions CI pipeline runs lint, typecheck, and build on push
- Each package can be built independently

#### Task 1.2: Database Schema Infrastructure & Multi-Tenancy

**What:** Create the foundational database tables (tenant, user, audit_log), implement Row-Level Security for multi-tenancy, and establish the migration workflow.

**Design:**

```typescript
// packages/api/src/db/schema/tenant.ts
import { pgTable, uuid, text, boolean, jsonb, timestamp } from 'drizzle-orm/pg-core';

export const tenant = pgTable('tenant', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  timezone: text('timezone').notNull().default('UTC'),
  locale: text('locale').notNull().default('en-US'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const user = pgTable('user', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenant.id),
  email: text('email').notNull(),
  displayName: text('display_name').notNull(),
  passwordHash: text('password_hash'),
  isActive: boolean('is_active').notNull().default(true),
  roles: jsonb('roles').notNull().default([]),
  preferences: jsonb('preferences').notNull().default({}),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueEmail: unique().on(table.tenantId, table.email),
}));
```

```typescript
// packages/api/src/middleware/tenant-isolation.ts
import { FastifyRequest, FastifyReply } from 'fastify';
import { sql } from 'drizzle-orm';

export async function tenantIsolation(request: FastifyRequest, reply: FastifyReply) {
  const tenantId = request.headers['x-tenant-id'] || request.user?.tenantId;
  if (!tenantId) {
    return reply.status(400).send({ type: 'missing_tenant', title: 'Tenant ID required' });
  }
  // Set PostgreSQL session variable for RLS
  await request.server.db.execute(
    sql`SET LOCAL app.current_tenant = ${tenantId}`
  );
  request.tenantId = tenantId;
}
```

```sql
-- Audit log trigger function (applied to all regulated tables)
CREATE OR REPLACE FUNCTION audit_trigger_func()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO audit_log (tenant_id, table_name, record_id, action, changed_by, old_values, new_values)
  VALUES (
    COALESCE(NEW.tenant_id, OLD.tenant_id),
    TG_TABLE_NAME,
    COALESCE(NEW.id, OLD.id),
    TG_OP,
    current_setting('app.current_user', true)::UUID,
    CASE WHEN TG_OP IN ('UPDATE', 'DELETE') THEN to_jsonb(OLD) END,
    CASE WHEN TG_OP IN ('INSERT', 'UPDATE') THEN to_jsonb(NEW) END
  );
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;
```

**Testing:**
- Create two tenants; verify each tenant can only see their own data via RLS
- Attempt cross-tenant data access and confirm it returns empty results
- Insert, update, and delete records; verify audit_log captures old_values and new_values correctly
- Run `drizzle-kit generate` and `drizzle-kit migrate`; verify migration applies cleanly to a fresh database
- Test that RLS policies enforce isolation even when application middleware is bypassed (direct SQL)
- Load test: create 10 tenants with 100 users each; verify query performance is unaffected by tenant count

#### Task 1.3: Authentication & Authorization

**What:** Implement JWT-based authentication with API key support for kiosks and IoT devices. Role-based access control using the JSONB roles array on the user table.

**Design:**

```typescript
// packages/api/src/plugins/auth.ts
import fp from 'fastify-plugin';
import jwt from '@fastify/jwt';

export default fp(async function authPlugin(fastify) {
  fastify.register(jwt, {
    secret: process.env.JWT_SECRET!,
    sign: { expiresIn: '8h' },
  });

  fastify.decorate('authenticate', async (request: FastifyRequest, reply: FastifyReply) => {
    const apiKey = request.headers['x-api-key'];
    if (apiKey) {
      // API key auth for kiosks and IoT gateways
      const keyRecord = await fastify.db.query.apiKey.findFirst({
        where: eq(apiKey.key, apiKey),
      });
      if (!keyRecord || !keyRecord.isActive) {
        return reply.status(401).send({ type: 'invalid_api_key', title: 'Invalid API key' });
      }
      request.user = { id: keyRecord.userId, tenantId: keyRecord.tenantId, roles: keyRecord.roles };
      return;
    }
    // JWT auth for web and mobile clients
    await request.jwtVerify();
  });

  fastify.decorate('authorize', (...requiredRoles: string[]) => {
    return async (request: FastifyRequest, reply: FastifyReply) => {
      const userRoles = request.user.roles as string[];
      const hasRole = requiredRoles.some(role => userRoles.includes(role));
      if (!hasRole) {
        return reply.status(403).send({
          type: 'forbidden',
          title: 'Insufficient permissions',
          detail: `Requires one of: ${requiredRoles.join(', ')}`,
        });
      }
    };
  });
});
```

**Testing:**
- Login with valid credentials returns a JWT token; token contains tenantId and roles
- Expired JWT returns 401 with clear error message
- API key authentication works for kiosk-type requests
- Role-based authorization: operator cannot access admin endpoints; plant_manager can access scheduling endpoints
- Invalid API key returns 401
- Missing both JWT and API key returns 401
- JWT token refresh works before expiration

#### Task 1.4: API Framework & Error Handling

**What:** Configure Fastify with OpenAPI 3.1 auto-generation, RFC 7807 Problem Details error responses, request validation via JSON Schema, and structured logging.

**Design:**

```typescript
// packages/api/src/app.ts
import Fastify from 'fastify';
import swagger from '@fastify/swagger';
import swaggerUi from '@fastify/swagger-ui';

const app = Fastify({
  logger: {
    level: process.env.LOG_LEVEL || 'info',
    serializers: {
      req: (req) => ({ method: req.method, url: req.url, tenantId: req.tenantId }),
    },
  },
});

app.register(swagger, {
  openapi: {
    openapi: '3.1.0',
    info: {
      title: 'Manufacturing ERP API',
      version: '0.1.0',
      description: 'Open-source AI-native manufacturing ERP',
    },
    components: {
      securitySchemes: {
        bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' },
        apiKey: { type: 'apiKey', in: 'header', name: 'X-API-Key' },
      },
    },
  },
});

// RFC 7807 error handler
app.setErrorHandler((error, request, reply) => {
  const statusCode = error.statusCode || 500;
  reply.status(statusCode).send({
    type: error.type || `https://manufacturing-erp.dev/errors/${statusCode}`,
    title: error.message,
    status: statusCode,
    detail: error.detail || undefined,
    instance: request.url,
    traceId: request.id,
  });
});
```

**Testing:**
- `GET /docs` serves Swagger UI with complete OpenAPI 3.1 spec
- All API responses include RFC 7807 `type`, `title`, `status` fields on error
- Request validation rejects invalid payloads with detailed error messages listing every invalid field
- Structured JSON logs include tenantId, request ID, and timing
- Health check endpoint `GET /health` returns database and Redis connectivity status
- OpenAPI spec can be downloaded as JSON/YAML and imported into Postman/Insomnia

#### Definition of Done - Phase 1

- [ ] Turborepo monorepo builds all packages successfully
- [ ] Docker Compose starts PostgreSQL 16 and Redis 7
- [ ] Database migrations create tenant, user, api_key, and audit_log tables
- [ ] Row-Level Security isolates tenants; verified with cross-tenant access tests
- [ ] JWT and API key authentication both work; role-based authorization enforced
- [ ] OpenAPI 3.1 spec auto-generated and served at `/docs`
- [ ] RFC 7807 error responses returned for all error cases
- [ ] Audit log captures all INSERT/UPDATE/DELETE with old and new values
- [ ] CI pipeline runs lint, typecheck, unit tests, and integration tests
- [ ] `docker compose up` followed by `turbo db:migrate && turbo db:seed` produces a working environment
- [ ] All integration tests pass against a real PostgreSQL instance via Testcontainers

---

### Phase 2: Items, BOM & Routing

**Goal:** Implement the item master, multi-level Bill of Materials with revision control, and routing (operation sequences) -- the core master data that every subsequent module depends on.

**Duration:** 4-5 weeks

**Dependencies:** Phase 1

#### Task 2.1: Item Master

**What:** Create the `item` table and full CRUD API with search, filtering, and JSONB properties for variable item attributes, compliance data, and custom fields.

**Design:**

```typescript
// packages/api/src/modules/item/item.schema.ts
import { z } from 'zod';

export const createItemSchema = z.object({
  itemNumber: z.string().min(1).max(50),
  name: z.string().min(1).max(200),
  description: z.string().optional(),
  itemType: z.enum(['raw_material', 'component', 'sub_assembly', 'finished_good', 'consumable', 'tooling']),
  uom: z.string().min(1).max(10),
  gtin: z.string().length(14).optional(),
  standardCost: z.number().nonnegative().optional(),
  leadTimeDays: z.number().int().nonnegative().default(0),
  safetyStock: z.number().nonnegative().default(0),
  reorderPoint: z.number().nonnegative().default(0),
  lotTracked: z.boolean().default(false),
  serialTracked: z.boolean().default(false),
  properties: z.record(z.unknown()).default({}),
  complianceData: z.record(z.unknown()).default({}),
  customFields: z.record(z.unknown()).default({}),
});

// GET /api/items?type=finished_good&search=bracket&compliance.itar_controlled=true
export const listItemsSchema = z.object({
  type: z.enum(['raw_material', 'component', 'sub_assembly', 'finished_good', 'consumable', 'tooling']).optional(),
  search: z.string().optional(),
  isActive: z.boolean().optional(),
  page: z.number().int().positive().default(1),
  pageSize: z.number().int().min(1).max(100).default(25),
});
```

```typescript
// packages/api/src/modules/item/item.service.ts
export class ItemService {
  async create(tenantId: string, data: CreateItem): Promise<Item> {
    const [item] = await this.db.insert(items).values({
      tenantId,
      ...data,
    }).returning();
    return item;
  }

  async search(tenantId: string, filters: ListItemsFilters): Promise<PaginatedResult<Item>> {
    let query = this.db.select().from(items)
      .where(eq(items.tenantId, tenantId));

    if (filters.type) query = query.where(eq(items.itemType, filters.type));
    if (filters.search) {
      query = query.where(
        or(
          ilike(items.itemNumber, `%${filters.search}%`),
          ilike(items.name, `%${filters.search}%`),
        )
      );
    }

    const total = await this.count(tenantId, filters);
    const results = await query
      .limit(filters.pageSize)
      .offset((filters.page - 1) * filters.pageSize)
      .orderBy(items.itemNumber);

    return { data: results, total, page: filters.page, pageSize: filters.pageSize };
  }
}
```

**Testing:**
- Create item with all fields populated; verify returned item matches input
- Create item with only required fields; verify defaults applied (safetyStock=0, lotTracked=false, etc.)
- Duplicate item_number within same tenant returns 409 Conflict
- Same item_number in different tenants succeeds (tenant isolation)
- Search by partial item_number returns matching items
- Filter by item_type returns only items of that type
- JSONB `properties` can store arbitrary nested objects; GIN index enables `@>` containment queries
- JSONB `complianceData` can store ITAR, REACH, PPAP fields; query `compliance_data @> '{"itar_controlled": true}'` returns correct results
- Pagination: 100 items with pageSize=25 returns 4 pages; page navigation works correctly
- Deactivating an item sets `is_active=false`; deactivated items excluded from default search results
- Audit log records item creation and updates with full before/after values

#### Task 2.2: Bill of Materials (BOM)

**What:** Implement multi-level BOM with revision control, effectivity dates, phantom assemblies, and BOM explosion via recursive CTE. BOM lines reference component items with quantity, scrap factor, and operation sequence linkage.

**Design:**

```typescript
// packages/api/src/modules/bom/bom.service.ts
export class BomService {
  async explode(bomId: string, options?: { asOfDate?: Date }): Promise<BomExplosionLine[]> {
    const asOf = options?.asOfDate || new Date();
    const result = await this.db.execute(sql`
      WITH RECURSIVE bom_explosion AS (
        SELECT
          bl.component_item_id,
          i.item_number,
          i.name,
          bl.quantity,
          bl.uom,
          bl.scrap_factor,
          bl.is_phantom,
          1 AS level,
          ARRAY[b.item_id] AS path
        FROM bom b
        JOIN bom_line bl ON bl.bom_id = b.id
        JOIN item i ON i.id = bl.component_item_id
        WHERE b.id = ${bomId}
          AND b.status = 'active'
          AND (bl.effective_to IS NULL OR bl.effective_to > ${asOf})

        UNION ALL

        SELECT
          bl.component_item_id,
          i.item_number,
          i.name,
          bl.quantity * be.quantity,
          bl.uom,
          bl.scrap_factor,
          bl.is_phantom,
          be.level + 1,
          be.path || b.item_id
        FROM bom_explosion be
        JOIN bom b ON b.item_id = be.component_item_id AND b.status = 'active'
        JOIN bom_line bl ON bl.bom_id = b.id
        JOIN item i ON i.id = bl.component_item_id
        WHERE NOT (b.item_id = ANY(be.path))
          AND be.level < 20
      )
      SELECT * FROM bom_explosion ORDER BY level, item_number
    `);
    return result.rows;
  }

  async createRevision(bomId: string, data: CreateBomRevision): Promise<Bom> {
    return this.db.transaction(async (tx) => {
      // Supersede the current active BOM
      await tx.update(bom)
        .set({ status: 'superseded', updatedAt: new Date() })
        .where(eq(bom.id, bomId));

      // Create new BOM revision with copied lines
      const [newBom] = await tx.insert(bom).values({
        ...data,
        status: 'active',
      }).returning();

      // Copy BOM lines from previous revision
      const oldLines = await tx.select().from(bomLine).where(eq(bomLine.bomId, bomId));
      if (oldLines.length > 0) {
        await tx.insert(bomLine).values(
          oldLines.map(line => ({ ...line, id: undefined, bomId: newBom.id }))
        );
      }
      return newBom;
    });
  }
}
```

**Testing:**
- Create a 3-level BOM (finished good -> sub-assembly -> components); explosion returns all levels with accumulated quantities
- Scrap factor correctly inflates quantities at each level (e.g., 2% scrap on 100 = 102 required)
- Phantom assemblies pass through: their components appear in the explosion as if they belonged to the parent
- BOM revision: create revision B from revision A; A's status becomes 'superseded'; B's status is 'active'
- Effectivity dates: BOM line with effective_to in the past is excluded from explosion
- Circular BOM detection: inserting a component that creates a cycle is rejected with a clear error
- BOM explosion performance: 10-level BOM with 500 total components completes in < 200ms
- BOM comparison: API endpoint returns differences between two revisions (added/removed/changed lines)
- BOM line operation_seq links to routing operation for material staging

#### Task 2.3: Routing & Operations

**What:** Implement routing (production process definition) with ordered operations, work center assignments, time standards, and overlap percentages.

**Design:**

```typescript
// packages/api/src/modules/routing/routing.schema.ts
export const createRoutingOperationSchema = z.object({
  sequence: z.number().int().positive().multipleOf(10), // 10, 20, 30...
  name: z.string().min(1).max(200),
  workCenterId: z.string().uuid(),
  setupTimeMins: z.number().nonnegative().default(0),
  runTimeMins: z.number().nonnegative().default(0),
  teardownTimeMins: z.number().nonnegative().default(0),
  overlapPercent: z.number().min(0).max(100).default(0),
  inspectionRequired: z.boolean().default(false),
  properties: z.record(z.unknown()).default({}),
});
```

```typescript
// packages/api/src/modules/routing/routing.service.ts
export class RoutingService {
  async calculateLeadTime(routingId: string, quantity: number): Promise<LeadTimeResult> {
    const operations = await this.db.select()
      .from(routingOperation)
      .where(eq(routingOperation.routingId, routingId))
      .orderBy(routingOperation.sequence);

    let totalMins = 0;
    for (let i = 0; i < operations.length; i++) {
      const op = operations[i];
      const opTime = op.setupTimeMins + (op.runTimeMins * quantity) + op.teardownTimeMins;
      if (i > 0 && operations[i - 1].overlapPercent > 0) {
        const prevOpTime = operations[i - 1].setupTimeMins +
          (operations[i - 1].runTimeMins * quantity) + operations[i - 1].teardownTimeMins;
        totalMins += opTime - (prevOpTime * operations[i - 1].overlapPercent / 100);
      } else {
        totalMins += opTime;
      }
    }
    return { totalMins, operations: operations.length, quantity };
  }
}
```

**Testing:**
- Create routing with 5 operations in sequence 10, 20, 30, 40, 50; operations returned in order
- Lead time calculation: setup 30min + run 2min/unit * 100 units + teardown 10min = 240min per operation
- Overlap: 50% overlap reduces total time by expected amount
- Routing linked to item; only one active routing per item at a time
- Operation references a valid work center; invalid work_center_id returns 400
- Routing revision: similar to BOM revision workflow (supersede + copy)
- Delete operation re-validates sequence gaps are acceptable

#### Task 2.4: Work Center & Equipment Master

**What:** Implement work center and equipment tables with capacity configuration, IoT connectivity configuration in JSONB properties, and status tracking.

**Design:**

```typescript
// packages/api/src/modules/work-center/work-center.service.ts
export class WorkCenterService {
  async getCapacityCalendar(workCenterId: string, startDate: Date, endDate: Date): Promise<CapacitySlot[]> {
    const wc = await this.db.query.workCenter.findFirst({
      where: eq(workCenter.id, workCenterId),
    });
    const shifts = wc.properties?.shift_schedule || [
      { shift: 'day', start: '06:00', end: '14:00', days: [1, 2, 3, 4, 5] },
    ];
    // Generate capacity slots for each day in the range based on shift schedule
    return generateCapacitySlots(startDate, endDate, shifts, wc.availableHoursPerDay);
  }
}
```

**Testing:**
- Create work center with shift schedule in properties; capacity calendar returns correct available hours per day
- Create equipment linked to work center; equipment list for work center returns all assigned machines
- Equipment status changes (operational -> maintenance -> operational) are tracked with timestamps
- Equipment properties can store MTConnect URL and OPC-UA endpoint
- Capacity query for a week returns 5 working days with correct available hours (excluding weekends)
- List all equipment by status (e.g., all machines currently in maintenance)

#### Definition of Done - Phase 2

- [ ] Item CRUD API with search, filtering, pagination, and JSONB property queries
- [ ] Multi-level BOM with revision control, effectivity dates, and phantom assemblies
- [ ] BOM explosion (recursive CTE) returns accumulated quantities across all levels
- [ ] BOM comparison between revisions shows added/removed/changed lines
- [ ] Routing with ordered operations, time standards, and overlap calculations
- [ ] Work center and equipment master data with capacity calendar
- [ ] All entities support audit logging
- [ ] OpenAPI spec includes all new endpoints with request/response schemas
- [ ] Integration tests verify BOM explosion, routing lead time, and capacity calendar
- [ ] Seed data creates a realistic 3-level BOM with routing for development use

---

### Phase 3: Work Orders & Shop Floor Execution

**Goal:** Implement the work order lifecycle (draft -> released -> in_progress -> completed -> closed), job card reporting for operators, material issuance, and backflushing -- the operational heart of the manufacturing ERP.

**Duration:** 5-6 weeks

**Dependencies:** Phase 2

#### Task 3.1: Work Order Lifecycle

**What:** Implement work order creation from BOM and routing, status transitions with validation, and material requirement generation from BOM lines.

**Design:**

```typescript
// packages/api/src/modules/work-order/work-order.service.ts
export class WorkOrderService {
  private static readonly VALID_TRANSITIONS: Record<string, string[]> = {
    draft: ['released', 'cancelled'],
    released: ['in_progress', 'on_hold', 'cancelled'],
    in_progress: ['on_hold', 'completed', 'cancelled'],
    on_hold: ['released', 'in_progress', 'cancelled'],
    completed: ['closed'],
    closed: [],
    cancelled: [],
  };

  async create(tenantId: string, data: CreateWorkOrder): Promise<WorkOrder> {
    return this.db.transaction(async (tx) => {
      // Create work order
      const [wo] = await tx.insert(workOrder).values({
        tenantId,
        woNumber: await this.generateNumber(tenantId, 'WO'),
        ...data,
        status: 'draft',
      }).returning();

      // Generate material requirements from BOM
      const bomLines = await tx.select().from(bomLine)
        .where(eq(bomLine.bomId, data.bomId));
      for (const line of bomLines) {
        await tx.insert(workOrderMaterial).values({
          workOrderId: wo.id,
          itemId: line.componentItemId,
          quantityRequired: line.quantity * data.quantityPlanned * (1 + (line.scrapFactor || 0)),
          uom: line.uom,
          backflush: line.properties?.supply_type === 'backflush',
        });
      }

      // Generate job cards from routing operations
      if (data.routingId) {
        const operations = await tx.select().from(routingOperation)
          .where(eq(routingOperation.routingId, data.routingId))
          .orderBy(routingOperation.sequence);
        for (const op of operations) {
          await tx.insert(jobCard).values({
            tenantId,
            jcNumber: await this.generateNumber(tenantId, 'JC'),
            workOrderId: wo.id,
            operationId: op.id,
            workCenterId: op.workCenterId,
            status: 'pending',
            quantityPlanned: data.quantityPlanned,
          });
        }
      }

      return wo;
    });
  }

  async transition(workOrderId: string, newStatus: string, userId: string): Promise<WorkOrder> {
    const wo = await this.findById(workOrderId);
    const allowed = WorkOrderService.VALID_TRANSITIONS[wo.status];
    if (!allowed?.includes(newStatus)) {
      throw new BusinessError(
        'invalid_transition',
        `Cannot transition from '${wo.status}' to '${newStatus}'`,
        400,
      );
    }
    // Additional validation per transition
    if (newStatus === 'completed') {
      await this.validateCompletion(wo);
    }
    const [updated] = await this.db.update(workOrder)
      .set({
        status: newStatus,
        actualStart: newStatus === 'in_progress' && !wo.actualStart ? new Date() : wo.actualStart,
        actualEnd: newStatus === 'completed' ? new Date() : wo.actualEnd,
        updatedAt: new Date(),
      })
      .where(eq(workOrder.id, workOrderId))
      .returning();
    return updated;
  }
}
```

**Testing:**
- Create work order from BOM and routing; materials and job cards auto-generated
- Material quantities include scrap factor (100 units with 2% scrap = 102 required)
- Status transitions follow the state machine: draft->released->in_progress->completed->closed
- Invalid transition (e.g., draft->completed) returns 400 with clear error
- Completing a work order validates that quantity_completed > 0
- Cancelling a work order sets status to 'cancelled' and releases allocated materials
- Work order list filtered by status, date range, and item
- Lot number assigned on work order creation when item is lot_tracked

#### Task 3.2: Job Card Reporting

**What:** Implement the operator-facing job card interface: start operation, report completions and scrap, pause/resume, and capture time.

**Design:**

```typescript
// packages/api/src/modules/job-card/job-card.service.ts
export class JobCardService {
  async startOperation(jobCardId: string, data: StartOperation): Promise<JobCard> {
    const jc = await this.findById(jobCardId);
    if (!['pending', 'ready'].includes(jc.status)) {
      throw new BusinessError('invalid_state', 'Job card must be pending or ready to start');
    }
    const [updated] = await this.db.update(jobCard)
      .set({
        status: 'in_progress',
        equipmentId: data.equipmentId,
        operatorId: data.operatorId,
        actualStart: new Date(),
        updatedAt: new Date(),
      })
      .where(eq(jobCard.id, jobCardId))
      .returning();

    // If this is the first job card started, mark work order as in_progress
    const wo = await this.db.query.workOrder.findFirst({
      where: eq(workOrder.id, jc.workOrderId),
    });
    if (wo.status === 'released') {
      await this.workOrderService.transition(wo.id, 'in_progress', data.operatorId);
    }
    return updated;
  }

  async reportCompletion(jobCardId: string, data: ReportCompletion): Promise<JobCard> {
    return this.db.transaction(async (tx) => {
      const jc = await this.findById(jobCardId);
      const newCompleted = jc.quantityCompleted + data.quantityCompleted;
      const newScrapped = jc.quantityScrapped + (data.quantityScrapped || 0);

      const [updated] = await tx.update(jobCard).set({
        quantityCompleted: newCompleted,
        quantityScrapped: newScrapped,
        runTimeMins: (jc.runTimeMins || 0) + (data.runTimeMins || 0),
        properties: {
          ...jc.properties,
          ...(data.operatorNotes && { operator_notes: data.operatorNotes }),
          ...(data.scrapDetails && { scrap_details: [
            ...(jc.properties?.scrap_details || []),
            ...data.scrapDetails,
          ]}),
        },
        status: newCompleted >= jc.quantityPlanned ? 'completed' : 'in_progress',
        actualEnd: newCompleted >= jc.quantityPlanned ? new Date() : undefined,
        updatedAt: new Date(),
      }).where(eq(jobCard.id, jobCardId)).returning();

      // Backflush materials if configured
      if (updated.status === 'completed') {
        await this.backflushMaterials(tx, jc.workOrderId, data.quantityCompleted);
      }

      // Update work order totals
      await this.updateWorkOrderTotals(tx, jc.workOrderId);

      return updated;
    });
  }
}
```

**Testing:**
- Start operation sets status to in_progress and records actual_start timestamp
- Report completion increments quantity_completed; when quantity_completed >= quantity_planned, status becomes 'completed'
- Scrap reporting increments quantity_scrapped and stores scrap details in properties JSONB
- Backflush: completing an operation auto-issues materials marked as backflush from inventory
- Operator notes stored in properties JSONB alongside structured data
- Pause/resume captures idle time
- Cannot report completion on a job card that is not in_progress
- Work order quantity_completed updated as sum of all job card completions
- Time tracking: setup_time_mins and run_time_mins accumulated across multiple reports

#### Task 3.3: Inventory Transactions (Material Issue & Receipt)

**What:** Implement inventory balance tracking with double-entry transaction logging: material receipt, issue to work orders, transfers, adjustments, and scrap.

**Design:**

```typescript
// packages/api/src/modules/inventory/inventory.service.ts
export class InventoryService {
  async issueToWorkOrder(data: MaterialIssue): Promise<InventoryTransaction> {
    return this.db.transaction(async (tx) => {
      // Validate sufficient available quantity
      const balance = await this.getBalance(tx, data.itemId, data.warehouseId, data.lotNumber);
      if (balance.quantityAvailable < data.quantity) {
        throw new BusinessError('insufficient_stock',
          `Available: ${balance.quantityAvailable} ${balance.uom}, Requested: ${data.quantity}`);
      }

      // Decrement inventory balance
      await tx.update(inventoryBalance).set({
        quantityOnHand: sql`quantity_on_hand - ${data.quantity}`,
        updatedAt: new Date(),
      }).where(eq(inventoryBalance.id, balance.id));

      // Create transaction record
      const [txn] = await tx.insert(inventoryTransaction).values({
        tenantId: data.tenantId,
        itemId: data.itemId,
        transactionType: 'issue',
        quantity: -data.quantity,
        uom: balance.uom,
        warehouseId: data.warehouseId,
        lotNumber: data.lotNumber,
        workOrderId: data.workOrderId,
        cost: balance.unitCost * data.quantity,
        transactedBy: data.userId,
      }).returning();

      // Update work_order_material issued quantity
      await tx.update(workOrderMaterial).set({
        quantityIssued: sql`quantity_issued + ${data.quantity}`,
      }).where(
        and(
          eq(workOrderMaterial.workOrderId, data.workOrderId),
          eq(workOrderMaterial.itemId, data.itemId),
        )
      );

      return txn;
    });
  }
}
```

**Testing:**
- Issue material to work order decrements inventory balance and creates transaction record
- Issue exceeding available quantity returns 400 with available quantity in error message
- Receipt increases inventory balance; transaction record shows positive quantity
- Transfer between warehouses: source decremented, destination incremented, two transaction records
- Adjustment: can increase or decrease balance with reason code
- Backflush (triggered by job card completion): auto-issues BOM quantities proportional to completed quantity
- Lot tracking: issue from specific lot; balance tracked per lot
- Inventory transaction history filterable by item, warehouse, date range, and transaction type
- Concurrent issues to the same balance are serialized (no negative balances due to race conditions)

#### Task 3.4: Shop Floor UI (Tablet-Optimized)

**What:** Build the tablet-optimized operator interface in Next.js: job card list, start/complete operations, report scrap, and view work order status. Designed for operators with no ERP expertise.

**Design:**

```typescript
// packages/web/src/components/shop-floor/JobCardPanel.tsx
'use client';

import { useState } from 'react';
import { Button } from '@/components/ui/button';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';

export function JobCardPanel({ jobCard }: { jobCard: JobCard }) {
  const [isReporting, setIsReporting] = useState(false);

  return (
    <Card className="min-h-[200px] touch-manipulation">
      <CardHeader className="pb-2">
        <CardTitle className="text-2xl font-bold">{jobCard.jcNumber}</CardTitle>
        <p className="text-lg text-muted-foreground">{jobCard.operationName}</p>
      </CardHeader>
      <CardContent className="space-y-4">
        {/* Large, touch-friendly status indicator */}
        <StatusBadge status={jobCard.status} size="lg" />

        {/* Progress bar: completed / planned */}
        <ProgressBar
          completed={jobCard.quantityCompleted}
          planned={jobCard.quantityPlanned}
          scrapped={jobCard.quantityScrapped}
        />

        {/* Large action buttons for operator touch */}
        <div className="grid grid-cols-2 gap-4">
          {jobCard.status === 'pending' && (
            <Button size="lg" className="h-16 text-xl" onClick={() => startOperation(jobCard.id)}>
              Start
            </Button>
          )}
          {jobCard.status === 'in_progress' && (
            <>
              <Button size="lg" className="h-16 text-xl" onClick={() => setIsReporting(true)}>
                Report
              </Button>
              <Button size="lg" variant="outline" className="h-16 text-xl"
                      onClick={() => pauseOperation(jobCard.id)}>
                Pause
              </Button>
            </>
          )}
        </div>

        {isReporting && (
          <CompletionReportForm
            jobCard={jobCard}
            onSubmit={handleCompletion}
            onCancel={() => setIsReporting(false)}
          />
        )}
      </CardContent>
    </Card>
  );
}
```

**Testing:**
- Shop floor page loads and displays active job cards for the logged-in operator's work center
- Job card list is scrollable on tablet viewport (768px-1024px width)
- Start button sets job card to in_progress; button changes to Report/Pause
- Completion report form accepts quantity completed, quantity scrapped, and operator notes
- Scrap reporting includes a dropdown of configurable scrap reasons
- All touch targets are minimum 48x48px per WCAG 2.1 guidelines
- Page updates in real-time when another operator completes an operation on a shared work center
- Offline-resilient: form data saved to local storage if network is temporarily unavailable
- Work order summary visible from job card view without navigating away

#### Definition of Done - Phase 3

- [ ] Work order creation from BOM and routing auto-generates materials and job cards
- [ ] Work order status machine enforced with validated transitions
- [ ] Job card start/complete/pause/resume with time tracking
- [ ] Scrap reporting with reason codes and operator notes
- [ ] Inventory issue, receipt, transfer, adjustment, and backflush transactions
- [ ] Inventory balance maintained with double-entry transaction logging
- [ ] Lot tracking through material issue and work order completion
- [ ] Tablet-optimized shop floor UI with large touch targets and progress visualization
- [ ] Concurrent inventory operations serialized to prevent negative balances
- [ ] Integration tests cover full work order lifecycle from creation through closure

---

### Phase 4: Quality Management

**Goal:** Implement ISO 9001-compliant quality management: configurable inspection templates, inspection execution and recording, non-conformance reports (NCR), and corrective action requests (CAR/CAPA) -- with inspection gates that block stock movement until passed.

**Duration:** 4-5 weeks

**Dependencies:** Phase 2 (items), Phase 3 (work orders, job cards, inventory)

#### Task 4.1: Inspection Templates

**What:** Create inspection templates with configurable checks (numeric with tolerances, pass/fail, text, visual) that can be linked to specific items and operations. Templates gate stock movement: items requiring inspection cannot move to available inventory until inspection passes.

**Design:**

```typescript
// packages/api/src/modules/quality/inspection-template.schema.ts
const inspectionCheckSchema = z.object({
  seq: z.number().int().positive(),
  name: z.string(),
  type: z.enum(['numeric', 'pass_fail', 'text', 'visual']),
  uom: z.string().optional(),
  nominal: z.number().optional(),       // for numeric: target value
  tolUpper: z.number().optional(),      // upper tolerance
  tolLower: z.number().optional(),      // lower tolerance (negative for bilateral)
  maxValue: z.number().optional(),      // max acceptable value (unilateral)
  minValue: z.number().optional(),      // min acceptable value (unilateral)
  gauge: z.string().optional(),         // measurement instrument
  sampleSize: z.number().int().positive().optional(),
  frequency: z.string().optional(),     // 'every_part', 'every_10_parts', 'first_and_last'
  instructions: z.string().optional(),
  criticalToQuality: z.boolean().default(false),
});
```

**Testing:**
- Create inspection template with 5 checks: 2 numeric (bilateral tolerance), 1 pass/fail, 1 text, 1 visual
- Template linked to specific item: inspection auto-triggered when item is received or operation completes
- Template with `inspection_type = 'incoming'`: triggered on material receipt
- Template with `inspection_type = 'in_process'`: triggered when a routing operation with `inspection_required = true` completes
- Numeric check: nominal 50.000, tol_upper 0.025, tol_lower -0.025; measured 50.012 passes; measured 50.030 fails
- Template deactivation prevents new inspections from being created with it
- Sample size and frequency configuration stored and retrievable

#### Task 4.2: Inspection Execution & Recording

**What:** Implement the inspection workflow: create inspection record, record results per check, auto-evaluate pass/fail for numeric checks, and disposition the lot based on overall inspection result.

**Design:**

```typescript
// packages/api/src/modules/quality/inspection.service.ts
export class InspectionService {
  async recordResult(inspectionId: string, data: RecordResult): Promise<InspectionRecord> {
    const inspection = await this.findById(inspectionId);
    const template = await this.templateService.findById(inspection.templateId);
    const check = template.checks.find(c => c.seq === data.checkSeq);

    // Auto-evaluate numeric checks
    let result = data.result;
    if (check.type === 'numeric' && data.numericValue !== undefined) {
      if (check.nominal !== undefined && check.tolUpper !== undefined && check.tolLower !== undefined) {
        const upper = check.nominal + check.tolUpper;
        const lower = check.nominal + check.tolLower;
        result = (data.numericValue >= lower && data.numericValue <= upper) ? 'pass' : 'fail';
      }
    }

    // Append result to inspection record JSONB results array
    const updatedResults = [
      ...inspection.results,
      { checkSeq: data.checkSeq, measured: data.measured, numericValue: data.numericValue, result, notes: data.notes },
    ];

    // Check if all checks are completed
    const allChecksRecorded = template.checks.every(
      c => updatedResults.some(r => r.checkSeq === c.seq)
    );

    // Determine overall status
    let overallStatus = inspection.status;
    if (allChecksRecorded) {
      const anyFailed = updatedResults.some(r => r.result === 'fail');
      const anyCriticalFailed = updatedResults.some(
        r => r.result === 'fail' && template.checks.find(c => c.seq === r.checkSeq)?.criticalToQuality
      );
      overallStatus = anyFailed ? 'failed' : 'passed';

      if (overallStatus === 'failed' && anyCriticalFailed) {
        // Auto-create NCR for critical quality failures
        await this.ncrService.createFromInspection(inspection, updatedResults);
      }

      if (overallStatus === 'passed') {
        // Release lot from quarantine to available
        await this.inventoryService.releaseLot(inspection.lotId);
      }
    }

    const [updated] = await this.db.update(inspectionRecord).set({
      results: updatedResults,
      status: overallStatus,
      inspectedAt: allChecksRecorded ? new Date() : undefined,
      quantityAccepted: overallStatus === 'passed' ? inspection.quantityInspected : undefined,
      quantityRejected: overallStatus === 'failed' ? inspection.quantityInspected : undefined,
      updatedAt: new Date(),
    }).where(eq(inspectionRecord.id, inspectionId)).returning();

    return updated;
  }
}
```

**Testing:**
- Recording all check results as 'pass' sets overall inspection to 'passed' and releases the lot
- Any check failing sets overall inspection to 'failed'
- Critical-to-quality check failure auto-creates NCR
- Numeric check auto-evaluation: value within tolerance passes; value outside tolerance fails
- Partial recording: some checks recorded, status remains 'in_progress'
- Multiple samples for a single check: all sample values stored; pass/fail based on acceptance criteria
- Inspection linked to work order and lot; traceability chain maintained
- Inspection gate: lot remains in 'quarantine' status until inspection passes
- First-article inspection (FAI) type includes AS9100-specific compliance_data fields

#### Task 4.3: Non-Conformance & Corrective Actions

**What:** Implement the NCR/CAR workflow per ISO 9001: report non-conformance, investigate root cause, assign corrective actions, verify effectiveness, and close.

**Design:**

```typescript
// packages/api/src/modules/quality/ncr.service.ts
export class NcrService {
  async updateAnalysis(ncrId: string, data: UpdateAnalysis): Promise<NonConformance> {
    const ncr = await this.findById(ncrId);
    const updatedAnalysis = {
      ...ncr.analysis,
      rootCauseMethod: data.rootCauseMethod,  // '5_why', 'ishikawa', 'fault_tree'
      rootCause: data.rootCause,
      contributingFactors: data.contributingFactors,
      correctiveActions: [
        ...(ncr.analysis?.correctiveActions || []),
        ...(data.newActions || []).map(action => ({
          ...action,
          id: generateId(),
          status: 'open',
          createdAt: new Date().toISOString(),
        })),
      ],
    };

    const [updated] = await this.db.update(nonConformance).set({
      analysis: updatedAnalysis,
      status: 'corrective_action',
      updatedAt: new Date(),
    }).where(eq(nonConformance.id, ncrId)).returning();
    return updated;
  }

  async verifyEffectiveness(ncrId: string, data: VerifyEffectiveness): Promise<NonConformance> {
    const ncr = await this.findById(ncrId);
    const updatedAnalysis = {
      ...ncr.analysis,
      effectivenessCheck: {
        ...ncr.analysis.effectivenessCheck,
        result: data.result,        // 'effective', 'partially_effective', 'ineffective'
        verifiedBy: data.userId,
        verifiedAt: new Date().toISOString(),
        notes: data.notes,
      },
    };

    const newStatus = data.result === 'effective' ? 'closed' : 'corrective_action';

    const [updated] = await this.db.update(nonConformance).set({
      analysis: updatedAnalysis,
      status: newStatus,
      closedAt: newStatus === 'closed' ? new Date() : undefined,
      updatedAt: new Date(),
    }).where(eq(nonConformance.id, ncrId)).returning();
    return updated;
  }
}
```

**Testing:**
- Create NCR from inspection failure; NCR linked to inspection, work order, lot, and item
- NCR status flow: open -> investigating -> corrective_action -> verification -> closed
- Root cause analysis stored with method (5 Why, Ishikawa) and contributing factors
- Corrective actions: add multiple actions with assignees and due dates
- Mark corrective action as implemented; track implementation date
- Effectiveness verification: effective -> close NCR; ineffective -> return to corrective_action
- NCR disposition: use_as_is, rework, repair, scrap, return_to_supplier, concession
- Scrap disposition auto-creates inventory scrap transaction for affected quantity
- NCR dashboard: count by severity, status, source, and age
- All NCR actions audited for ISO 9001 compliance

#### Definition of Done - Phase 4

- [ ] Inspection templates with configurable checks (numeric, pass/fail, text, visual)
- [ ] Inspection execution with auto-evaluation of numeric checks against tolerances
- [ ] Inspection gate: lots held in quarantine until inspection passes
- [ ] Critical-to-quality failures auto-create NCRs
- [ ] Full NCR lifecycle: open -> investigate -> corrective_action -> verify -> close
- [ ] Corrective action tracking with assignees, due dates, and effectiveness verification
- [ ] NCR disposition handling (scrap, rework, use-as-is, return-to-supplier)
- [ ] ISO 9001 audit trail for all quality records
- [ ] Quality dashboard with first-pass yield, NCR aging, and trend analysis
- [ ] Integration tests cover inspection-to-NCR-to-CAR workflow end-to-end

---

### Phase 5: MRP Engine

**Goal:** Implement Material Requirements Planning (MRP): net-change processing that explodes BOMs against demand (work orders, sales forecasts), nets against on-hand inventory and open purchase orders, and generates planned production and purchase orders with time-phased requirements.

**Duration:** 5-6 weeks

**Dependencies:** Phase 2 (BOM, items), Phase 3 (work orders, inventory)

#### Task 5.1: Demand Aggregation

**What:** Aggregate demand from work orders, sales order lines (external input), and manual forecasts into a time-bucketed gross requirements schedule per item.

**Design:**

```typescript
// packages/api/src/modules/mrp/demand.service.ts
export class DemandService {
  async aggregateGrossRequirements(
    tenantId: string,
    planningHorizon: { start: Date; end: Date },
    bucketSize: 'day' | 'week',
  ): Promise<Map<string, TimeBucket[]>> {
    // Gross requirements = work order material needs + external demand
    const woMaterials = await this.db.select({
      itemId: workOrderMaterial.itemId,
      quantity: workOrderMaterial.quantityRequired,
      quantityIssued: workOrderMaterial.quantityIssued,
      dueDate: workOrder.plannedStart,
    })
    .from(workOrderMaterial)
    .innerJoin(workOrder, eq(workOrder.id, workOrderMaterial.workOrderId))
    .where(
      and(
        eq(workOrder.tenantId, tenantId),
        inArray(workOrder.status, ['draft', 'released', 'in_progress']),
        between(workOrder.plannedStart, planningHorizon.start, planningHorizon.end),
      )
    );

    // Group into time buckets per item
    return this.bucketize(woMaterials, bucketSize);
  }
}
```

**Testing:**
- Aggregate demand from 50 work orders across 10 items; verify quantities match BOM explosion
- Daily and weekly bucketing produce correct groupings
- Already-issued material (quantityIssued > 0) reduces the net requirement
- Cancelled work orders excluded from demand
- Demand from multiple sources (work orders + manual forecasts) summed correctly per bucket

#### Task 5.2: MRP Netting & Planned Order Generation

**What:** Implement the core MRP netting logic: for each item, compare gross requirements against projected available balance (on-hand + scheduled receipts - allocations), and generate planned orders when projected balance drops below safety stock.

**Design:**

```typescript
// packages/api/src/modules/mrp/mrp-engine.service.ts
export class MrpEngineService {
  async runMrp(tenantId: string, options: MrpOptions): Promise<MrpResult> {
    const plan = await this.createPlan(tenantId, options);
    const items = await this.getPlannableItems(tenantId);
    const plannedOrders: PlannedOrder[] = [];

    // Process items in BOM level order (finished goods first, then sub-assemblies, then components)
    const sortedItems = this.sortByBomLevel(items);

    for (const item of sortedItems) {
      const grossReqs = await this.demandService.getGrossRequirements(item.id, options.horizon);
      const onHand = await this.inventoryService.getAvailableBalance(item.id);
      const scheduledReceipts = await this.getScheduledReceipts(item.id, options.horizon);

      let projectedBalance = onHand;
      const buckets = this.mergeTimeline(grossReqs, scheduledReceipts, options.bucketSize);

      for (const bucket of buckets) {
        projectedBalance += bucket.scheduledReceipts - bucket.grossRequirement;

        if (projectedBalance < item.safetyStock) {
          // Generate planned order
          const orderQty = this.calculateOrderQuantity(
            item.safetyStock - projectedBalance,
            item.lotSizeMin,
            item.lotSizeMax,
            item.lotSizeMultiple,
          );

          const orderType = item.itemType === 'raw_material' ? 'purchase' : 'production';
          const leadTime = orderType === 'purchase'
            ? item.leadTimeDays
            : await this.routingService.calculateLeadTimeDays(item.id, orderQty);

          const planned: PlannedOrder = {
            productionPlanId: plan.id,
            itemId: item.id,
            orderType,
            quantity: orderQty,
            uom: item.uom,
            plannedEnd: bucket.date,
            plannedStart: subtractBusinessDays(bucket.date, leadTime),
            priority: 500,
            source: 'mrp',
          };

          plannedOrders.push(planned);
          projectedBalance += orderQty;

          // For production orders: generate dependent demand (BOM explosion)
          if (orderType === 'production') {
            await this.explodeDependentDemand(item.id, orderQty, planned.plannedStart);
          }
        }
      }
    }

    await this.db.insert(plannedOrder).values(plannedOrders);
    return { planId: plan.id, ordersGenerated: plannedOrders.length };
  }
}
```

**Testing:**
- MRP run with 0 on-hand and 100 demand generates planned order for 100 (adjusted by lot sizing)
- Safety stock: 50 on-hand, 20 safety stock, 40 demand -> projected balance = 10 -> no planned order; 50 on-hand, 20 safety stock, 35 demand -> projected balance = 15 -> no planned order; 50 on-hand, 20 safety stock, 40 demand -> projected balance = 10 < 20 -> planned order for 10
- Lot sizing: min=50, max=200, multiple=25; requirement of 37 generates order for 50 (min); requirement of 180 generates order for 200 (rounded up to multiple)
- Lead time offset: purchase order with 14-day lead time; demand on June 1 -> planned start May 18
- Multi-level BOM: finished good demand generates dependent demand for sub-assemblies and components at offset dates
- Scheduled receipts (open purchase orders, in-progress work orders) reduce net requirements
- Net-change MRP: only re-processes items where demand, supply, or BOM has changed since last run
- MRP run completes in < 30 seconds for 1000 items with 5000 demand lines
- Planned orders can be firmed (converted to actual work orders or purchase orders)
- MRP exception report lists items where demand exceeds capacity or lead time is insufficient

#### Task 5.3: MRP UI & Planned Order Management

**What:** Build the planner-facing MRP interface: run MRP, view time-phased requirements, review planned orders, firm orders, and view exception reports.

**Design:**

```typescript
// packages/web/src/app/(protected)/planning/mrp/page.tsx
export default async function MrpPage() {
  return (
    <div className="space-y-6">
      <MrpRunPanel />        {/* Trigger MRP run with date range and options */}
      <MrpResultsSummary />  {/* Orders generated, exceptions, items processed */}
      <PlannedOrderTable />  {/* Sortable/filterable table with firm/release actions */}
      <TimePhasedGrid />     {/* Item x Week grid showing requirements/supply/balance */}
      <ExceptionsList />     {/* Items with insufficient supply, late orders, capacity issues */}
    </div>
  );
}
```

**Testing:**
- MRP run triggers from UI; progress indicator shows items processed
- Planned order table displays order type, item, quantity, dates, and source
- Firming a planned production order creates a work order (via Phase 3 work order creation)
- Firming a planned purchase order creates a purchase requisition record
- Time-phased grid shows gross requirements, scheduled receipts, projected balance, and planned orders per week
- Exception report highlights items with negative projected balance, past-due demand, and insufficient lead time
- Planner can manually adjust planned order dates and quantities before firming
- MRP results persist; re-running MRP replaces previous unfirmed planned orders

#### Definition of Done - Phase 5

- [ ] MRP engine processes gross requirements against inventory and generates planned orders
- [ ] Safety stock, lot sizing (min/max/multiple), and lead time offset all implemented correctly
- [ ] Multi-level BOM explosion generates dependent demand at offset dates
- [ ] Net-change MRP processes only changed items for performance
- [ ] Planned orders can be firmed into work orders or purchase requisitions
- [ ] Time-phased requirements grid shows supply/demand balance per item per bucket
- [ ] MRP exception report identifies items with supply gaps
- [ ] MRP run completes in < 30 seconds for 1000 items
- [ ] Integration tests verify MRP netting logic with various lot sizing and lead time scenarios

---

### Phase 6: Job Costing & Financials

**Goal:** Implement job costing with actual vs. standard cost variance reporting, WIP valuation, and quoting/estimating from BOM and routing -- providing CFOs with the financial visibility they need to manage manufacturing profitability.

**Duration:** 3-4 weeks

**Dependencies:** Phase 3 (work orders, job cards, inventory transactions), Phase 5 (MRP)

#### Task 6.1: Cost Element Configuration & Standard Cost Roll-Up

**What:** Define cost elements (material, labor, overhead, subcontract) and implement standard cost roll-up from BOM materials and routing operations.

**Design:**

```typescript
// packages/api/src/modules/costing/cost-rollup.service.ts
export class CostRollupService {
  async rollUpStandardCost(itemId: string): Promise<CostBreakdown> {
    const bom = await this.bomService.getActiveBom(itemId);
    const routing = await this.routingService.getActiveRouting(itemId);

    let materialCost = 0;
    let laborCost = 0;
    let overheadCost = 0;

    // Material cost from BOM
    if (bom) {
      const lines = await this.bomService.getLines(bom.id);
      for (const line of lines) {
        const componentCost = await this.getItemStandardCost(line.componentItemId);
        materialCost += componentCost * line.quantity * (1 + (line.scrapFactor || 0));
      }
    }

    // Labor and overhead from routing
    if (routing) {
      const operations = await this.routingService.getOperations(routing.id);
      for (const op of operations) {
        const wc = await this.workCenterService.findById(op.workCenterId);
        const laborTime = op.setupTimeMins + op.runTimeMins; // per unit
        laborCost += (laborTime / 60) * (op.properties?.labor_rate_override || wc.costRate || 0);
        overheadCost += (laborTime / 60) * (wc.overheadRate || 0);
      }
    }

    return { materialCost, laborCost, overheadCost, totalCost: materialCost + laborCost + overheadCost };
  }
}
```

**Testing:**
- Standard cost roll-up for a 3-level BOM correctly accumulates material costs at each level
- Labor cost calculated from routing time standards * work center cost rate
- Overhead cost calculated from routing time * overhead rate
- Scrap factor inflates material cost appropriately
- Roll-up for item with no BOM returns zero material cost
- Cost breakdown returned as separate material/labor/overhead components

#### Task 6.2: Actual Cost Collection & Variance Reporting

**What:** Collect actual costs from inventory transactions (material cost), job card time reports (labor cost), and work center overhead rates. Compare actual vs. standard for variance analysis.

**Design:**

```typescript
// packages/api/src/modules/costing/variance.service.ts
export class VarianceService {
  async getWorkOrderVariance(workOrderId: string): Promise<VarianceReport> {
    const wo = await this.workOrderService.findById(workOrderId);

    // Planned (standard) costs
    const planned = await this.costRollupService.rollUpStandardCost(wo.itemId);
    const plannedTotal = planned.totalCost * wo.quantityPlanned;

    // Actual material cost from inventory transactions
    const materialTxns = await this.db.select({
      total: sql<number>`SUM(ABS(cost))`,
    }).from(inventoryTransaction)
      .where(and(
        eq(inventoryTransaction.workOrderId, workOrderId),
        eq(inventoryTransaction.transactionType, 'issue'),
      ));

    // Actual labor cost from job cards
    const jobCards = await this.db.select().from(jobCard)
      .where(eq(jobCard.workOrderId, workOrderId));
    const actualLabor = jobCards.reduce((sum, jc) => {
      const wc = this.workCenterCache.get(jc.workCenterId);
      return sum + ((jc.setupTimeMins + jc.runTimeMins) / 60) * (wc?.costRate || 0);
    }, 0);

    return {
      workOrderNumber: wo.woNumber,
      quantityPlanned: wo.quantityPlanned,
      quantityCompleted: wo.quantityCompleted,
      planned: { material: planned.materialCost * wo.quantityPlanned, labor: planned.laborCost * wo.quantityPlanned, overhead: planned.overheadCost * wo.quantityPlanned },
      actual: { material: materialTxns[0].total || 0, labor: actualLabor, overhead: actualOverhead },
      variance: {
        material: (materialTxns[0].total || 0) - (planned.materialCost * wo.quantityPlanned),
        labor: actualLabor - (planned.laborCost * wo.quantityPlanned),
        total: actualTotal - plannedTotal,
        percentVariance: ((actualTotal - plannedTotal) / plannedTotal) * 100,
      },
    };
  }
}
```

**Testing:**
- Work order with exact standard costs shows zero variance
- Higher actual material prices produce unfavorable material variance
- Extra scrap increases actual material cost and creates unfavorable variance
- Labor variance: actual setup took longer than standard -> unfavorable labor variance
- WIP valuation: sum of actual costs on uncompleted work orders
- Variance report by item, by work center, and by date range
- Cost variance dashboard with trend charts
- Unit cost calculation: actual total cost / quantity completed

#### Task 6.3: Quoting & Estimating

**What:** Implement quoting from BOM and routing: cost roll-up with configurable margin to generate customer quotes for job shop and contract manufacturing scenarios.

**Design:**

```typescript
// packages/api/src/modules/costing/quote.service.ts
export class QuoteService {
  async generateQuote(data: QuoteRequest): Promise<Quote> {
    const costBreakdown = await this.costRollupService.rollUpStandardCost(data.itemId);
    const unitCost = costBreakdown.totalCost;
    const totalCost = unitCost * data.quantity;
    const margin = data.marginPercent || 25; // default 25% margin
    const unitPrice = unitCost * (1 + margin / 100);

    return {
      itemId: data.itemId,
      quantity: data.quantity,
      unitCost,
      totalCost,
      marginPercent: margin,
      unitPrice,
      totalPrice: unitPrice * data.quantity,
      breakdown: costBreakdown,
      validUntil: addDays(new Date(), data.validityDays || 30),
    };
  }
}
```

**Testing:**
- Quote generated from BOM/routing with 25% margin matches expected price
- Quote with custom margin percentage calculates correctly
- Quote includes breakdown by material, labor, and overhead
- Quote validity date defaults to 30 days from creation
- Quote for item with multi-level BOM includes all sub-component costs
- Multiple quotes for the same item with different quantities show volume pricing effects

#### Definition of Done - Phase 6

- [ ] Standard cost roll-up from BOM and routing
- [ ] Actual cost collection from inventory transactions and job card time
- [ ] Variance reporting: actual vs. standard by material, labor, overhead
- [ ] WIP valuation for uncompleted work orders
- [ ] Quoting from BOM/routing with configurable margin
- [ ] Cost variance dashboard with trend analysis
- [ ] Integration tests verify cost calculation accuracy across multi-level BOMs

---

### Phase 7: IoT Integration (OPC-UA & MTConnect)

**Goal:** Build the IoT gateway service that connects to CNC machines and PLCs via OPC-UA and MTConnect, ingests real-time telemetry (execution state, spindle speed, vibration, cycle counts), and stores it in the partitioned `equipment_telemetry` table for use by the AI quality predictor and shop floor dashboards.

**Duration:** 4-5 weeks

**Dependencies:** Phase 2 (equipment master)

#### Task 7.1: MTConnect Poller

**What:** Implement an HTTP poller that discovers MTConnect agents configured on equipment records, polls their `/current` and `/sample` endpoints at configurable intervals, and normalizes the XML responses into telemetry records.

**Design:**

```typescript
// packages/iot-gateway/src/connectors/mtconnect-poller.ts
import { XMLParser } from 'fast-xml-parser';

export class MTConnectPoller {
  private parser = new XMLParser({ ignoreAttributes: false, attributeNamePrefix: '@_' });

  async poll(equipment: Equipment): Promise<TelemetryReading[]> {
    const agentUrl = equipment.properties?.connectivity?.mtconnect_url;
    if (!agentUrl) return [];

    const response = await fetch(`${agentUrl}/current`);
    const xml = await response.text();
    const parsed = this.parser.parse(xml);

    const streams = parsed.MTConnectStreams?.Streams?.DeviceStream;
    if (!streams) return [];

    return this.extractReadings(equipment.id, streams);
  }

  private extractReadings(equipmentId: string, stream: any): TelemetryReading[] {
    const readings: TelemetryReading[] = [];
    const components = stream.ComponentStream || [];

    for (const component of Array.isArray(components) ? components : [components]) {
      const events = component.Events || {};
      const samples = component.Samples || {};

      readings.push({
        equipmentId,
        timestamp: new Date(),
        source: 'mtconnect',
        executionState: events.Execution?.['#text'],
        mode: events.ControllerMode?.['#text'],
        spindleSpeed: parseFloat(samples.SpindleSpeed?.['#text']) || null,
        feedRate: parseFloat(samples.PathFeedrate?.['#text']) || null,
        spindleLoad: parseFloat(samples.Load?.['#text']) || null,
        readings: this.extractAllDataItems(component),
      });
    }
    return readings;
  }
}
```

**Testing:**
- Poll a mock MTConnect agent; verify telemetry records created with correct execution state and spindle speed
- Handle MTConnect agent timeout gracefully (log warning, continue polling other machines)
- Parse real MTConnect XML from at least two different CNC controller types (Fanuc, Mazak format)
- Polling interval configurable per equipment (default 5 seconds)
- Duplicate suppression: do not store a reading if execution state and all numeric values are unchanged from the previous reading
- Connection recovery: poller reconnects automatically after agent goes offline and comes back

#### Task 7.2: OPC-UA Client

**What:** Implement an OPC-UA client using `node-opcua` that subscribes to equipment data nodes and receives real-time value changes via OPC-UA subscription (pub/sub model).

**Design:**

```typescript
// packages/iot-gateway/src/connectors/opcua-client.ts
import { OPCUAClient, ClientSession, ClientSubscription, DataValue } from 'node-opcua';

export class OpcuaConnector {
  async connect(equipment: Equipment): Promise<void> {
    const endpointUrl = equipment.properties?.connectivity?.opcua_endpoint;
    if (!endpointUrl) return;

    const client = OPCUAClient.create({
      endpointMustExist: false,
      securityMode: MessageSecurityMode.None, // configurable per equipment
    });

    await client.connect(endpointUrl);
    const session = await client.createSession();

    const subscription = ClientSubscription.create(session, {
      requestedPublishingInterval: 1000,
      requestedMaxKeepAliveCount: 10,
    });

    // Monitor standard machine data nodes
    const nodesToMonitor = [
      { nodeId: 'ns=2;s=SpindleSpeed', attributeId: AttributeIds.Value },
      { nodeId: 'ns=2;s=FeedRate', attributeId: AttributeIds.Value },
      { nodeId: 'ns=2;s=SpindleLoad', attributeId: AttributeIds.Value },
      { nodeId: 'ns=2;s=ExecutionState', attributeId: AttributeIds.Value },
    ];

    for (const node of nodesToMonitor) {
      const monitoredItem = await subscription.monitor(node, { samplingInterval: 500 });
      monitoredItem.on('changed', (dataValue: DataValue) => {
        this.handleValueChange(equipment.id, node.nodeId, dataValue);
      });
    }
  }
}
```

**Testing:**
- Connect to a mock OPC-UA server; verify subscription receives value changes
- Telemetry records created on value change with correct equipment ID and timestamp
- Handle OPC-UA server disconnect gracefully; reconnect with exponential backoff
- Security: test with MessageSecurityMode.SignAndEncrypt when configured
- Browse OPC-UA server node tree to discover available data items
- Batch telemetry writes: buffer readings and insert in bulk every 5 seconds for performance

#### Task 7.3: Telemetry Storage & Shop Floor Dashboard

**What:** Store telemetry in the partitioned `equipment_telemetry` table and build a real-time shop floor dashboard showing machine states, utilization, and alerts.

**Design:**

```typescript
// packages/iot-gateway/src/processors/telemetry-writer.ts
export class TelemetryWriter {
  private buffer: TelemetryReading[] = [];
  private flushInterval = 5000; // 5 seconds

  async flush(): Promise<void> {
    if (this.buffer.length === 0) return;
    const batch = this.buffer.splice(0, this.buffer.length);
    await this.db.insert(equipmentTelemetry).values(batch);

    // Publish to Redis for real-time dashboard
    for (const reading of batch) {
      await this.redis.publish(`telemetry:${reading.equipmentId}`, JSON.stringify(reading));
    }
  }
}
```

**Testing:**
- Telemetry records written to correct monthly partition
- Dashboard shows real-time machine state (Active/Idle/Stopped) with < 3 second latency
- Machine utilization calculation: (active time / available time) * 100 over rolling 24-hour window
- Alert when machine enters unexpected state (breakdown, alarm)
- Historical telemetry queryable by equipment, date range, and parameter
- Partition creation: verify that new monthly partitions are created automatically before each month

#### Definition of Done - Phase 7

- [ ] MTConnect poller connects to agents and stores telemetry readings
- [ ] OPC-UA client subscribes to equipment data nodes and receives value changes
- [ ] Telemetry stored in partitioned table with monthly partitions
- [ ] Real-time shop floor dashboard showing machine states and utilization
- [ ] Telemetry data published to Redis for < 3 second dashboard latency
- [ ] Graceful handling of disconnects with automatic reconnection
- [ ] Integration tests with mock MTConnect agent and mock OPC-UA server

---

### Phase 8: AI Dynamic Scheduling & Event Bus

**Goal:** Build the AI dynamic scheduling agent that continuously re-optimizes the production sequence using live capacity, skills, tooling, and material availability -- and introduce the event bus (from Data Model Suggestion 2) that enables AI agents to subscribe to production events in real time.

**Duration:** 5-6 weeks

**Dependencies:** Phase 3 (work orders, job cards), Phase 5 (MRP planned orders), Phase 7 (equipment telemetry)

#### Task 8.1: Event Bus Infrastructure

**What:** Implement the append-only `event_store` table alongside the existing relational tables. Production state changes (work order status transitions, job card completions, equipment state changes) emit events to the event store. AI agents subscribe to filtered event streams.

**Design:**

```typescript
// packages/api/src/modules/events/event-bus.service.ts
export class EventBusService {
  async emit(event: DomainEvent): Promise<void> {
    // Write to event store
    await this.db.insert(eventStore).values({
      streamId: event.aggregateId,
      streamType: event.aggregateType,
      streamVersion: event.version,
      eventType: event.type,
      eventCategory: event.category,
      tenantId: event.tenantId,
      data: event.data,
      metadata: event.metadata,
      occurredAt: event.occurredAt,
    });

    // Publish to Redis for real-time consumers
    await this.redis.publish(
      `events:${event.category}`,
      JSON.stringify(event),
    );

    // Enqueue BullMQ jobs for async consumers
    await this.eventQueue.add(event.type, event);
  }
}

// Usage in work order service:
async transition(workOrderId: string, newStatus: string, userId: string) {
  const wo = await this.updateStatus(workOrderId, newStatus);
  await this.eventBus.emit({
    aggregateId: wo.id,
    aggregateType: 'work_order',
    type: `WorkOrder${capitalize(newStatus)}`,
    category: 'production',
    tenantId: wo.tenantId,
    data: { woNumber: wo.woNumber, itemId: wo.itemId, status: newStatus, previousStatus: wo.status },
    metadata: { actorId: userId, actorType: 'user', source: 'web_ui' },
    occurredAt: new Date(),
  });
  return wo;
}
```

**Testing:**
- Work order status transition emits event to event store with correct category and data
- Job card completion emits OperationCompleted event with quantities and times
- Equipment state change emits MachineStateChanged event
- Redis pub/sub delivers events to subscribers in < 100ms
- BullMQ worker processes events asynchronously
- Event store query by stream_id returns all events for an aggregate in version order
- Temporal query: filter events by occurred_at range
- Event_subscription table tracks each consumer's last_sequence for resumption

#### Task 8.2: AI Dynamic Scheduling Agent

**What:** Build the scheduling agent that uses Claude to re-optimize the production sequence when conditions change (machine breakdown, material delay, operator absence, new urgent order). The agent subscribes to production and telemetry events, evaluates whether rescheduling is needed, and proposes schedule changes with explainable reasoning.

**Design:**

```typescript
// packages/ai-agents/src/scheduler/scheduler-agent.ts
import Anthropic from '@anthropic-ai/sdk';

export class SchedulerAgent {
  private anthropic = new Anthropic();

  async evaluateAndReschedule(trigger: SchedulingTrigger): Promise<ScheduleProposal | null> {
    // Gather current state
    const openWorkOrders = await this.api.getWorkOrders({ status: ['released', 'in_progress'] });
    const equipmentStatus = await this.api.getEquipmentStatus();
    const personnelAvailable = await this.api.getAvailablePersonnel();
    const materialAvailability = await this.api.getMaterialAvailability(
      openWorkOrders.map(wo => wo.id)
    );

    const systemPrompt = `You are a manufacturing production scheduler for a discrete manufacturing plant.
Your role is to optimize the production sequence to maximize throughput while respecting constraints.

CONSTRAINTS:
- Each work order must be assigned to a work center with the required capability
- Equipment must be in 'operational' status to be scheduled
- Operators must have qualifications matching the operation requirements
- Materials must be available (or scheduled to arrive) before the operation start date
- Customer priority work orders (priority >= 800) take precedence
- Setup time is reduced when consecutive operations use the same tooling

CURRENT TRIGGER: ${JSON.stringify(trigger)}

Respond with a JSON object containing:
- "shouldReschedule": boolean
- "reason": string explaining why or why not
- "proposals": array of { workOrderId, newPlannedStart, newPlannedEnd, assignedEquipment, assignedOperator, explanation }`;

    const response = await this.anthropic.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 4096,
      system: systemPrompt,
      messages: [
        {
          role: 'user',
          content: JSON.stringify({
            workOrders: openWorkOrders,
            equipment: equipmentStatus,
            personnel: personnelAvailable,
            materials: materialAvailability,
          }),
        },
      ],
    });

    const proposal = JSON.parse(response.content[0].text);
    if (proposal.shouldReschedule) {
      // Emit scheduling events for each proposed change
      for (const change of proposal.proposals) {
        await this.eventBus.emit({
          type: 'WorkOrderRescheduled',
          category: 'production',
          data: {
            ...change,
            scheduledBy: 'ai_scheduler',
            scheduleConfidence: proposal.confidence,
          },
          metadata: { actorType: 'ai_agent', source: 'scheduler' },
        });
      }
    }
    return proposal;
  }
}
```

**Testing:**
- Machine breakdown triggers rescheduling: affected work orders moved to available equipment
- New urgent order (priority 800+) inserted into schedule; lower-priority orders shifted
- Material delay: work orders requiring delayed material pushed back; independent work orders unaffected
- Operator absence: operations requiring that operator's qualifications reassigned or deferred
- Setup time optimization: consecutive same-tooling operations grouped on the same machine
- Schedule proposal includes human-readable explanation for each change
- Scheduler respects equipment capability constraints (does not assign lathe work to a mill)
- Scheduler does not propose changes when current schedule is still valid (trigger evaluated but no reschedule needed)
- Schedule changes logged as WorkOrderRescheduled events with AI attribution
- Integration test: simulate 20 work orders, 5 machines, 1 breakdown; verify rescheduled work orders are feasible

#### Definition of Done - Phase 8

- [ ] Event bus emits domain events for all work order, job card, and equipment state changes
- [ ] Event store (append-only, partitioned) stores all events alongside relational tables
- [ ] AI scheduler subscribes to production and telemetry events
- [ ] Machine breakdown triggers automatic schedule re-optimization
- [ ] Schedule proposals include explainable reasoning for each change
- [ ] Schedule changes emitted as WorkOrderRescheduled events with AI attribution
- [ ] Planner can review, accept, or reject AI schedule proposals before they take effect
- [ ] Integration tests verify scheduling across constraint scenarios (breakdown, material delay, priority change)

---

### Phase 9: AI Quality Prediction & BOM Assistant

**Goal:** Build two AI-powered capabilities: (1) a predictive quality agent that correlates equipment telemetry with inspection results to predict quality failures before they occur, and (2) a BOM construction assistant that parses engineering documents to generate draft BOM entries.

**Duration:** 5-6 weeks

**Dependencies:** Phase 4 (quality management), Phase 7 (telemetry), Phase 8 (event bus)

#### Task 9.1: Predictive Quality Agent

**What:** Build an AI agent that ingests equipment telemetry events and historical quality data, identifies patterns that precede quality failures, and emits QualityPredictionAlert events that trigger automatic work order holds when confidence is high.

**Design:**

```typescript
// packages/ai-agents/src/quality/quality-predictor.ts
export class QualityPredictorAgent {
  async analyzeRecentTelemetry(equipmentId: string): Promise<QualityPrediction | null> {
    // Get recent telemetry (last 2 hours)
    const telemetry = await this.api.getEquipmentTelemetry(equipmentId, {
      since: subtractHours(new Date(), 2),
    });

    // Get historical correlations: telemetry patterns that preceded inspection failures
    const historicalFailures = await this.api.getInspectionFailures({
      equipmentId,
      since: subtractMonths(new Date(), 6),
      limit: 50,
    });

    const response = await this.anthropic.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 2048,
      system: `You are a manufacturing quality prediction agent. Analyze equipment telemetry
data and compare it against historical patterns that preceded quality failures.

Respond with JSON:
- "predictionType": string (e.g., "surface_finish_degradation", "dimensional_drift")
- "confidence": number 0-1
- "contributingFactors": array of { factor, currentValue, typicalRange, trend }
- "recommendedAction": string
- "urgency": "immediate" | "next_tool_change" | "next_shift"`,
      messages: [
        {
          role: 'user',
          content: JSON.stringify({ currentTelemetry: telemetry, historicalFailures }),
        },
      ],
    });

    const prediction = JSON.parse(response.content[0].text);
    if (prediction.confidence >= 0.80) {
      await this.eventBus.emit({
        type: 'QualityPredictionAlert',
        category: 'quality',
        data: { equipmentId, ...prediction },
        metadata: { actorType: 'ai_agent', source: 'quality_predictor' },
      });

      // Auto-hold work order if confidence >= 0.90 and urgency is immediate
      if (prediction.confidence >= 0.90 && prediction.urgency === 'immediate') {
        const activeWo = await this.api.getActiveWorkOrderOnEquipment(equipmentId);
        if (activeWo) {
          await this.api.transitionWorkOrder(activeWo.id, 'on_hold');
        }
      }
    }
    return prediction;
  }
}
```

**Testing:**
- Increasing vibration trend triggers surface_finish_degradation prediction
- Spindle load exceeding historical threshold triggers prediction with high confidence
- Low confidence prediction (< 0.80) does not emit alert event
- High confidence prediction (>= 0.90) with immediate urgency auto-holds the work order
- Prediction includes contributing factors with current values and typical ranges
- Historical pattern matching: known failure patterns from past inspection data recognized in current telemetry
- False positive tracking: predicted failures that do not materialize are tracked for model improvement
- Alert surfaced on shop floor dashboard and sent to quality manager notification channel

#### Task 9.2: BOM Construction Assistant

**What:** Build an AI agent that accepts engineering documents (CAD file metadata, PDF datasheets, spreadsheets) and proposes draft BOM entries with confidence scores, flagging discrepancies with existing purchasing and inventory data.

**Design:**

```typescript
// packages/ai-agents/src/bom-assistant/bom-assistant.ts
export class BomAssistantAgent {
  async generateBomFromDocument(
    tenantId: string,
    documentBase64: string,
    documentType: 'pdf' | 'image' | 'csv',
    parentItemId?: string,
  ): Promise<BomDraft> {
    const existingItems = await this.api.getItems(tenantId, { isActive: true });

    const response = await this.anthropic.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 4096,
      system: `You are a manufacturing BOM construction assistant. Parse the engineering document
and extract component information to generate a Bill of Materials.

For each component identified:
1. Extract part number, description, quantity, and unit of measure
2. Match against existing items in inventory: ${JSON.stringify(existingItems.map(i => ({ id: i.id, number: i.itemNumber, name: i.name })))}
3. Flag discrepancies (different description, different UOM)
4. Assign a confidence score (0-1) for each line

Respond with JSON:
{
  "bomLines": [
    { "lineNumber": 1, "partNumber": "...", "description": "...", "quantity": N,
      "uom": "...", "matchedItemId": "uuid or null", "confidence": 0.95,
      "flags": ["description_mismatch", "new_item_needed"] }
  ],
  "warnings": ["..."],
  "unrecognizedContent": ["..."]
}`,
      messages: [
        {
          role: 'user',
          content: [
            { type: 'text', text: `Parse this ${documentType} document and generate BOM entries.` },
            documentType === 'pdf' || documentType === 'image'
              ? { type: 'image', source: { type: 'base64', media_type: 'application/pdf', data: documentBase64 } }
              : { type: 'text', text: Buffer.from(documentBase64, 'base64').toString('utf-8') },
          ],
        },
      ],
    });

    return JSON.parse(response.content[0].text);
  }
}
```

**Testing:**
- Parse a PDF drawing with a parts list; extract 10+ components with quantities
- Match extracted part numbers against existing item master; matched items have high confidence
- Flag new items not in the system (confidence lower, flag: 'new_item_needed')
- Description mismatch: drawing says "Hex Bolt M8x30" but item master has "Hex Cap Screw M8x30" -> flag
- CSV spreadsheet import: parse a BOM spreadsheet with columns mapped to part number, description, qty, uom
- Draft BOM lines can be reviewed and accepted by the engineer before creating the actual BOM
- Unrecognized content reported in the response for manual review

#### Task 9.3: Graph Query Layer (from Data Model 4)

**What:** Introduce the `graph_node` and `graph_edge` tables to enable where-used analysis, lot genealogy, and supplier impact analysis. Graph is synced from relational data via triggers.

**Design:**

As specified in Data Model Suggestion 4, implement the graph tables and sync triggers. The graph layer enables:
- Where-used analysis: "Which finished goods use component X?"
- Lot genealogy: forward and backward traceability
- Supplier impact: "If supplier Y fails, which work orders are affected?"

**Testing:**
- Creating an item auto-creates a graph_node via trigger
- Adding a BOM line creates a CONTAINS edge between parent and child item nodes
- Where-used query returns all finished goods containing a specific component (multi-level)
- Lot genealogy: issuing material from lot L1 to work order WO1 creates CONSUMED_BY edge; WO1 completion creates PRODUCED_FROM edge to output lot L2
- Supplier impact query: identify all active work orders affected by a supplier disruption
- Graph stays in sync after BOM revision (old edges expired, new edges created)

#### Definition of Done - Phase 9

- [ ] Predictive quality agent correlates telemetry with inspection history
- [ ] High-confidence quality predictions auto-hold work orders
- [ ] Quality alerts surfaced on dashboard and via notifications
- [ ] BOM assistant parses PDF drawings and CSV spreadsheets to generate draft BOM entries
- [ ] BOM draft lines matched against existing items with confidence scores
- [ ] Graph layer synced from relational data for where-used, genealogy, and impact analysis
- [ ] Integration tests verify quality prediction pipeline and BOM parsing accuracy

---

### Phase 10: Natural Language Operator Interface & MCP Server

**Goal:** Build the natural-language interface for shop floor operators (voice or chat) that handles ERP transactions in plain language, and publish an MCP server that exposes manufacturing ERP resources to AI agents.

**Duration:** 4-5 weeks

**Dependencies:** Phase 3 (work orders, job cards), Phase 8 (event bus)

#### Task 10.1: Natural Language Operator Interface

**What:** Build a chat-based interface where operators can report production events in plain language. The system interprets the input, maps it to ERP transactions (complete job card, report scrap, request material), confirms with the operator, and executes.

**Design:**

```typescript
// packages/ai-agents/src/nl-interface/nl-operator.ts
export class NLOperatorInterface {
  async processInput(
    operatorId: string,
    input: string,
    context: OperatorContext,
  ): Promise<NLResponse> {
    const response = await this.anthropic.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      system: `You are a shop floor assistant for manufacturing operators. Parse operator input and
identify the intended ERP transaction.

OPERATOR CONTEXT:
- Work Center: ${context.workCenterCode}
- Active Job Cards: ${JSON.stringify(context.activeJobCards)}
- Current Equipment: ${context.equipmentCode}

SUPPORTED TRANSACTIONS:
- complete_operation: { jobCardId, quantityCompleted, quantityScrapped, scrapReason, notes }
- report_scrap: { jobCardId, quantity, reason, notes }
- request_material: { itemNumber, quantity, workOrderId }
- pause_operation: { jobCardId, reason }
- resume_operation: { jobCardId }
- log_issue: { description, severity, equipmentId }

Respond with JSON:
{
  "intent": "complete_operation",
  "params": { ... },
  "confirmationMessage": "Complete 48 pieces on JC-2026-00142 with 2 scrapped (surface finish)?",
  "confidence": 0.95
}`,
      messages: [{ role: 'user', content: input }],
    });

    const parsed = JSON.parse(response.content[0].text);

    return {
      intent: parsed.intent,
      params: parsed.params,
      confirmationMessage: parsed.confirmationMessage,
      confidence: parsed.confidence,
      requiresConfirmation: parsed.confidence < 0.95,
      rawTranscript: input,
    };
  }

  async executeConfirmed(intent: string, params: Record<string, any>): Promise<void> {
    switch (intent) {
      case 'complete_operation':
        await this.jobCardService.reportCompletion(params.jobCardId, {
          quantityCompleted: params.quantityCompleted,
          quantityScrapped: params.quantityScrapped,
          operatorNotes: params.notes,
          scrapDetails: params.scrapReason
            ? [{ quantity: params.quantityScrapped, reason: params.scrapReason }]
            : undefined,
        });
        break;
      // ... other intents
    }
  }
}
```

**Testing:**
- "Finished 48 pieces on the Mazak, scrapped 2 for surface finish" -> correct complete_operation with quantities
- "Need more 100K resistors for work order 142" -> request_material with correct item lookup
- "Pause, going to break" -> pause_operation on the operator's active job card
- Ambiguous input: "done with the job" -> confirmation prompt specifying which job card
- Low confidence (< 0.95) requires operator confirmation before executing
- Transcript stored in job card properties for audit trail
- Multi-language support: basic Spanish and German input recognized
- Invalid intent: "what is the weather?" -> friendly response that the system handles production transactions only

#### Task 10.2: MCP Server

**What:** Publish an MCP server that exposes manufacturing ERP resources (items, BOMs, work orders, inventory, quality records, equipment status) as MCP tools and resources, enabling external AI agents to interact with the ERP.

**Design:**

```typescript
// packages/mcp-server/src/tools/work-order-tools.ts
export const workOrderTools = [
  {
    name: 'get_work_orders',
    description: 'List work orders with filtering by status, item, date range',
    inputSchema: {
      type: 'object',
      properties: {
        status: { type: 'array', items: { type: 'string' } },
        itemNumber: { type: 'string' },
        fromDate: { type: 'string', format: 'date' },
        toDate: { type: 'string', format: 'date' },
      },
    },
    handler: async (params) => {
      return api.getWorkOrders(params);
    },
  },
  {
    name: 'create_work_order',
    description: 'Create a new work order from a BOM and routing',
    inputSchema: {
      type: 'object',
      required: ['itemId', 'bomId', 'quantityPlanned', 'plannedStart', 'plannedEnd'],
      properties: {
        itemId: { type: 'string' },
        bomId: { type: 'string' },
        routingId: { type: 'string' },
        quantityPlanned: { type: 'number' },
        plannedStart: { type: 'string', format: 'date-time' },
        plannedEnd: { type: 'string', format: 'date-time' },
      },
    },
    handler: async (params) => {
      return api.createWorkOrder(params);
    },
  },
  {
    name: 'get_equipment_telemetry',
    description: 'Get real-time or historical telemetry for an equipment',
    inputSchema: { /* ... */ },
    handler: async (params) => { /* ... */ },
  },
  {
    name: 'explode_bom',
    description: 'Explode a BOM to all levels showing accumulated quantities',
    inputSchema: { /* ... */ },
    handler: async (params) => { /* ... */ },
  },
];
```

**Testing:**
- MCP client can list available tools and resources
- `get_work_orders` returns filtered work orders with correct data
- `create_work_order` creates a work order with materials and job cards
- `explode_bom` returns multi-level BOM with accumulated quantities
- `get_equipment_telemetry` returns recent sensor readings
- Authentication: MCP server requires valid API key or OAuth token
- Rate limiting: MCP tools enforce per-tenant rate limits
- Error responses follow MCP error format with useful detail messages

#### Definition of Done - Phase 10

- [ ] Natural language interface parses operator input into ERP transactions
- [ ] Confirmation flow for low-confidence interpretations
- [ ] Transcripts stored in job card properties for audit trail
- [ ] MCP server exposes items, BOMs, work orders, inventory, quality, and equipment as tools and resources
- [ ] MCP server authenticated via API key or OAuth
- [ ] Integration tests verify NL parsing accuracy for common operator phrases
- [ ] MCP server tested with a standard MCP client

---

### Phase 11: Advanced Features (ECO, CMMS, EDI, Subcontracting)

**Goal:** Implement engineering change orders, computerized maintenance management, EDI connectivity for OEM document exchange, and subcontracting workflow -- features that move the product from viable to competitive.

**Duration:** 5-6 weeks

**Dependencies:** Phase 2 (BOM), Phase 3 (work orders), Phase 7 (equipment telemetry)

#### Task 11.1: Engineering Change Orders (ECO)

**What:** Implement the full ECO lifecycle: submit change request, review affected items and BOMs, approve/reject, implement changes across affected BOMs and routings, and close.

**Design:**

Adopt the `engineering_change_order` and `eco_affected_item` tables from Data Model Suggestion 1. ECO implementation creates new BOM and/or routing revisions and supersedes the old ones.

**Testing:**
- Submit ECO affecting 3 items; affected items listed with current and proposed revisions
- Approve ECO; implement ECO creates new BOM revisions for all affected items
- Reject ECO; no changes to BOMs or routings
- ECO with effectivity date: new revisions become active only on the target date
- ECO audit trail: full history of who submitted, reviewed, approved, and implemented

#### Task 11.2: CMMS (Maintenance Management)

**What:** Implement preventive and predictive maintenance scheduling linked to production planning: maintenance plans, maintenance work orders, and downtime tracking that feeds back to the AI scheduler.

**Design:**

Implement `maintenance_plan` and `maintenance_work_order` tables. Preventive maintenance generates recurring maintenance work orders at configured intervals. Predictive maintenance triggers from AI telemetry analysis (Phase 9).

**Testing:**
- Preventive maintenance plan generates maintenance work order when interval expires
- Maintenance work order places equipment in 'maintenance' status, blocking production scheduling
- Maintenance completion restores equipment to 'operational' status
- Downtime tracked in minutes; feeds into equipment utilization calculation
- AI predictive maintenance alert creates a predictive maintenance work order
- Parts used in maintenance tracked (items consumed from inventory)

#### Task 11.3: EDI Connectivity

**What:** Implement X12 EDI document exchange for receiving OEM customer purchase orders (850), sending advance ship notices (856), and processing shipping schedules (862).

**Testing:**
- Parse incoming X12 850 purchase order; create corresponding sales order record
- Generate X12 856 ASN from completed shipment data
- EDI mapping configuration per trading partner (field mapping, segment qualifiers)
- Validation: reject malformed EDI documents with clear error report
- EDI transaction log for audit and troubleshooting

#### Task 11.4: Subcontracting Workflow

**What:** Implement the subcontracting workflow for outsourced manufacturing operations: generate purchase order for subcontract operation, ship materials to subcontractor, receive finished goods back, and track costs.

**Testing:**
- Work order with subcontracted operation creates a purchase order for the supplier
- Material transfer to subcontractor tracked as inventory issue with 'subcontract' reference
- Receipt from subcontractor completes the operation and creates inventory receipt
- Subcontract costs captured in work order costing
- Quality inspection triggered on receipt from subcontractor

#### Definition of Done - Phase 11

- [ ] ECO lifecycle with multi-item BOM revision management
- [ ] Preventive maintenance scheduling with recurring work orders
- [ ] Predictive maintenance triggered by AI telemetry analysis
- [ ] EDI X12 850/856/862 document exchange
- [ ] Subcontracting workflow with material transfer and receipt
- [ ] All features integrated with existing work order, inventory, and costing modules

---

### Phase 12: Hardening & Production Readiness

**Goal:** Security hardening, performance optimization, deployment automation, documentation, and final integration testing to prepare the system for production use by real manufacturers.

**Duration:** 4-5 weeks

**Dependencies:** All previous phases

#### Task 12.1: Security Hardening

**What:** Implement OWASP API Security Top 10 mitigations, OPC-UA transport encryption (TLS), input sanitization, rate limiting, and security audit.

**Testing:**
- OWASP API Security Top 10: all 10 risks mitigated and verified
- SQL injection: parameterized queries verified across all endpoints
- Rate limiting: API returns 429 after threshold exceeded
- OPC-UA communication encrypted with TLS when configured
- RBAC: exhaustive test of every endpoint with every role; unauthorized access returns 403
- GDPR: employee data deletion (right to erasure) verified
- Dependency audit: `npm audit` returns no critical or high vulnerabilities

#### Task 12.2: Performance Optimization

**What:** Database query optimization, connection pooling, caching strategy, and load testing for production-scale workloads.

**Testing:**
- MRP run for 5000 items with 25000 demand lines completes in < 60 seconds
- BOM explosion for 15-level BOM completes in < 500ms
- 100 concurrent shop floor operators reporting job card completions: p95 response time < 200ms
- Telemetry ingestion: 50 machines at 5-second intervals (600 writes/min) with no queue backlog
- Dashboard page load: < 2 seconds for work order list with 1000 records
- Database connection pool sized correctly; no connection exhaustion under load
- Redis caching: equipment status and item master cached; cache invalidation on update

#### Task 12.3: Deployment & Operations

**What:** Production Docker images, Kubernetes Helm chart, database backup strategy, monitoring, and alerting.

**Testing:**
- `docker compose up` starts the full stack in < 30 seconds
- Helm chart deploys to a Kubernetes cluster with configurable replicas, resource limits, and secrets
- Database backup: automated daily backup; restore tested and verified
- Health check endpoints used by Kubernetes liveness and readiness probes
- Structured logs forwarded to stdout for container log aggregation
- Prometheus metrics exposed: API response times, database query duration, telemetry ingestion rate, queue depth
- Alerting: notification when API error rate > 1%, database connections > 80% pool, MRP run fails

#### Task 12.4: Documentation & Seed Data

**What:** API documentation (auto-generated from OpenAPI), administrator guide, operator quick-start guide, and realistic seed data for demonstrations.

**Testing:**
- OpenAPI spec validates against OpenAPI 3.1 linter
- Seed data creates a realistic manufacturing scenario: 200 items, 15 BOMs (3-5 levels), 10 work centers, 30 pieces of equipment, 50 work orders in various states, quality records
- Admin guide covers: installation, tenant creation, user management, work center setup, BOM creation
- Operator guide covers: logging in to the shop floor tablet, starting an operation, reporting completions and scrap

#### Definition of Done - Phase 12

- [ ] OWASP API Security Top 10 mitigated and verified
- [ ] Load tested: 100 concurrent users, 50 machines, 5000 MRP items
- [ ] Kubernetes Helm chart deploys successfully
- [ ] Database backup and restore verified
- [ ] Monitoring and alerting operational
- [ ] OpenAPI documentation complete and published
- [ ] Seed data creates a realistic demo environment
- [ ] Administrator and operator guides written
- [ ] End-to-end integration test covers: create items -> build BOM -> create routing -> run MRP -> create work order -> report job cards -> complete inspection -> close work order -> verify costing

---

## 4. Summary

| Phase | Title | Duration | Dependencies | Key Deliverables |
|-------|-------|----------|-------------|------------------|
| 1 | Foundation & Infrastructure | 3-4 weeks | None | Monorepo, DB, auth, multi-tenancy, API framework |
| 2 | Items, BOM & Routing | 4-5 weeks | Phase 1 | Item master, multi-level BOM, routing, work centers |
| 3 | Work Orders & Shop Floor | 5-6 weeks | Phase 2 | Work order lifecycle, job cards, inventory, shop floor UI |
| 4 | Quality Management | 4-5 weeks | Phases 2, 3 | Inspections, NCR, CAR/CAPA, inspection gates |
| 5 | MRP Engine | 5-6 weeks | Phases 2, 3 | MRP netting, planned order generation, time-phased requirements |
| 6 | Job Costing & Financials | 3-4 weeks | Phases 3, 5 | Cost roll-up, variance reporting, WIP valuation, quoting |
| 7 | IoT Integration | 4-5 weeks | Phase 2 | MTConnect poller, OPC-UA client, telemetry storage, dashboard |
| 8 | AI Scheduling & Event Bus | 5-6 weeks | Phases 3, 5, 7 | Event bus, AI dynamic scheduler, schedule proposals |
| 9 | AI Quality & BOM Assistant | 5-6 weeks | Phases 4, 7, 8 | Predictive quality, BOM parser, graph query layer |
| 10 | NL Interface & MCP Server | 4-5 weeks | Phases 3, 8 | Operator chat interface, MCP server for AI agents |
| 11 | Advanced Features | 5-6 weeks | Phases 2, 3, 7 | ECO, CMMS, EDI, subcontracting |
| 12 | Hardening & Production | 4-5 weeks | All phases | Security, performance, deployment, documentation |

**Total estimated duration:** 52-68 weeks (12-16 months)

**MVP milestone (Phases 1-6):** 24-31 weeks (6-8 months) -- a functional manufacturing ERP with BOM, MRP, work orders, quality, and costing.

**AI-differentiated milestone (Phases 7-10):** 42-53 weeks (10-13 months) -- the AI scheduling, quality prediction, BOM assistant, and NL operator interface that differentiate this project from every existing open-source manufacturing ERP.
