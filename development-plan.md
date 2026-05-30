# Spa & Salon Management — Phased Development Plan

> Project: 264-spa-salon-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four
`data-model-suggestion-*.md` files. The primary data model adopted is **Suggestion 3 (Hybrid
Relational + JSONB)** — the pragmatic middle ground for an MVP that must serve hair salons, day
spas, medical spas, and hotel/resort spas across jurisdictions without constant migrations. The
normalised entity boundaries from Suggestion 1 inform the table layout; the event-sourced
analytics views from Suggestion 2 inform the AI/analytics layer (Phase 9); the graph affinity
ideas from Suggestion 4 inform the recommendation engine (Phase 9) as a lightweight edge table
rather than a separate graph database.

---

## Core Requirements (synthesised)

- **What it does**: An open-source, AI-native platform for appointment booking, therapist
  scheduling, POS/retail, CRM, memberships, and inventory, serving the long tail of independent
  beauty professionals through multi-location spa/salon chains that incumbents (Mindbody, Vagaro,
  Zenoti, Boulevard) price out at $200–$470/month with 12-month lock-ins.
- **Who uses it**: solo estheticians/stylists; multi-chair salon owners (commissions + retail);
  day spa operators (room utilisation, therapist availability); medical spa practitioners (HIPAA
  clinical records); hotel/resort spa directors. Plus end clients booking online.
- **Differentiators**: open-source + self-hostable; clean OpenAPI 3.1 reference API in a
  fragmented proprietary-API market; native CalDAV/iCalendar sync (a noted competitor gap); and an
  **MCP server** exposing booking/availability/client tools to AI agents — not offered by any
  surveyed platform.
- **MVP scope** (features.md "Must-have"): scheduling & booking, POS & payments, client CRM,
  staff management, inventory, reporting.
- **Post-MVP** (v1.1 + backlog): AI automations & recommendations, marketplace discovery,
  marketing automation, contactless checkout, dynamic pricing, no-show prediction, scheduling
  optimisation, loyalty, HIPAA medspa module, MCP server.
- **Deployment model**: hybrid. Self-hosted via Docker Compose; cloud SaaS multi-tenant; the
  same artefact serves both. Single Postgres-backed monolith API + worker + web SPA.
- **Integration surface**: Stripe (payments, deposits, subscriptions via Connect), Twilio
  (SMS/voice reminders + AI receptionist telephony), an LLM provider (OpenAI-compatible) for AI
  features, SMTP/email provider, Google/Apple/Outlook calendars via iCalendar + CalDAV.
- **Standards compliance**: OpenAPI 3.1, JSON Schema 2020-12, RFC 5545 (iCalendar), RFC 4791
  (CalDAV), RFC 6749 + RFC 7636 (OAuth 2.0 + PKCE), E.164 phone, ISO 4217 currency, ISO 3166
  jurisdiction, PCI DSS SAQ-A (tokenise, never store PAN), HIPAA (medspa module), GDPR/CCPA
  (consent + erasure), WCAG 2.2 AA (client booking UI), OWASP Top 10, MCP specification.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | **TypeScript (Node 22 LTS)** | API/integration-heavy domain (Stripe, Twilio, calendars, MCP). One language across API, worker, web SPA, and the MCP server reduces context-switching. Strong typing pairs with JSON Schema/OpenAPI generation. |
| API framework | **Fastify 5 + `@fastify/swagger`** | High throughput, first-class JSON Schema validation per route, auto-generates the OpenAPI 3.1 spec required by standards.md directly from route schemas. |
| Validation / types | **Zod + `zod-to-json-schema`** | Single source of truth: Zod schemas validate request bodies and emit the JSON Schema 2020-12 used in the OpenAPI doc. Shared between API and worker. |
| Database | **PostgreSQL 16** | Hybrid relational+JSONB model (Suggestion 3) needs JSONB + GIN indexes, `gen_random_uuid()`, partial indexes, row-level security (HIPAA), and `LISTEN/NOTIFY`. Single-engine simplicity for self-hosters. |
| Query layer / migrations | **Drizzle ORM + drizzle-kit** | Type-safe SQL close to the schema, first-class JSONB columns, explicit SQL migrations (no hidden magic) — important for a schema that mixes columns and JSONB. |
| Task queue | **BullMQ on Redis 7** | Async workloads: reminders, deposit capture, membership rebilling, campaign sends, AI calls, projection rebuilds. Repeatable/cron jobs for billing and reminder sweeps. |
| Cache / queue broker | **Redis 7** | Backs BullMQ, rate limiting, availability-slot caching, idempotency keys. |
| Frontend | **React 19 + Vite + TanStack Router/Query + Tailwind + shadcn/ui** | Two SPAs from one toolchain: staff admin console and client booking portal. shadcn/Tailwind accelerates a WCAG 2.2 AA-compliant accessible UI. |
| Payments | **Stripe (Payment Intents, Setup Intents, Subscriptions, Connect)** | PCI DSS SAQ-A: card data tokenised client-side, never touches our servers. Connect supports the multi-location/marketplace model. |
| Comms | **Twilio (SMS, Voice, Verify)** | Reminders, confirmations, deposit links, and the Phase 10 AI receptionist (Voice + Media Streams). E.164 enforced. |
| Email | **Nodemailer + provider (SES/Resend) via SMTP** | Confirmations, campaigns, receipts; provider-agnostic via SMTP. |
| LLM access | **Vercel AI SDK (provider-agnostic, OpenAI-compatible)** | Recommendations, comms drafting, no-show scoring narration, AI receptionist NLU. Self-hosters can point at a local model. |
| Calendar | **`ical-generator` (RFC 5545) + custom CalDAV (RFC 4791/6638) module** | iCalendar export + native CalDAV sync — the differentiator from standards.md. |
| Auth | **Lucia-style sessions for web + OAuth 2.1 (PKCE) for API/partners** | RFC 6749/7636. PKCE mandatory for the public SPA/mobile clients. |
| MCP server | **`@modelcontextprotocol/sdk` (TypeScript)** | Exposes booking/availability/client tools + schedule/inventory resources to AI agents — the headline AI-native differentiator. |
| Testing | **Vitest (unit/integration) + Playwright (E2E) + Testcontainers (real PG/Redis)** | Vitest fast for unit; Testcontainers spins ephemeral Postgres/Redis for integration; Playwright drives both SPAs including accessibility checks (`@axe-core/playwright`). |
| Quality | **ESLint + Prettier + `tsc --noEmit` + `vitest --coverage`** | Lint, format, typecheck, coverage gates per phase. |
| Package manager | **pnpm workspaces (monorepo)** | Workspace packages: `api`, `worker`, `web-admin`, `web-booking`, `mcp`, `shared` (Zod schemas + DB). |
| Containerisation | **Docker + docker-compose** | One `docker compose up` for self-hosters: api, worker, postgres, redis, web. |
| Observability | **Pino (structured logs) + OpenTelemetry traces** | Structured audit-friendly logging; OWASP/HIPAA-aligned audit log is separate (DB table). |

### Project Structure

```
spa-salon-management/
├── pnpm-workspace.yaml
├── package.json
├── docker-compose.yml
├── Dockerfile                      # multi-stage; targets api | worker | web
├── .env.example
├── packages/
│   ├── shared/                     # Zod schemas, DB schema, domain types, utilities
│   │   ├── src/
│   │   │   ├── db/
│   │   │   │   ├── schema.ts        # Drizzle table definitions (Suggestion 3 hybrid model)
│   │   │   │   ├── client.ts        # pooled pg client factory
│   │   │   │   └── migrations/      # drizzle-kit SQL migrations
│   │   │   ├── schemas/             # Zod request/response schemas → JSON Schema
│   │   │   ├── domain/              # pricing, availability, commission calculators (pure fns)
│   │   │   ├── errors.ts            # typed error taxonomy
│   │   │   └── index.ts
│   ├── api/                        # Fastify app
│   │   ├── src/
│   │   │   ├── server.ts
│   │   │   ├── plugins/             # auth, tenancy, rate-limit, swagger, error-handler
│   │   │   ├── routes/              # one module per resource
│   │   │   ├── services/            # business logic (booking, billing, inventory, crm)
│   │   │   ├── integrations/        # stripe, twilio, email, calendar, llm
│   │   │   └── audit/               # audit-log writer + RLS helpers
│   ├── worker/                     # BullMQ processors
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── queues.ts
│   │   │   └── jobs/                # reminders, rebilling, campaigns, projections, ai-scoring
│   ├── mcp/                        # MCP server (Phase 11)
│   │   └── src/server.ts
│   ├── web-admin/                  # staff console SPA
│   └── web-booking/                # client booking SPA (WCAG 2.2 AA)
├── tests/
│   ├── integration/                # Testcontainers-backed
│   ├── e2e/                        # Playwright
│   └── fixtures/                   # seed tenants, services, clients, sample diffs
└── openapi/
    └── openapi.json                # generated artefact (CI-checked)
```

---

## Phase 1: Foundation, Tenancy & Auth

### Purpose
Stand up the monorepo, database, configuration, and the multi-tenant + authentication
foundation that every later phase assumes. After this phase a developer can run
`docker compose up`, hit a health endpoint, register a tenant, log in, and receive a session/token
scoped to that tenant. Nothing booking-specific exists yet, but the security and tenancy spine
that all later data hangs from is in place.

### Tasks

#### 1.1 — Monorepo, config, and Docker scaffolding

**What**: Create the pnpm workspace, the multi-stage Dockerfile, docker-compose, and typed env config.

**Design**:
- `pnpm-workspace.yaml` lists `packages/*`. Root scripts: `dev`, `build`, `lint`, `typecheck`, `test`, `db:migrate`, `db:seed`, `openapi:gen`.
- Typed config loader in `shared` using Zod:
```ts
export const Env = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  SESSION_SECRET: z.string().min(32),
  OAUTH_ISSUER_URL: z.string().url(),
  STRIPE_SECRET_KEY: z.string().optional(),
  STRIPE_WEBHOOK_SECRET: z.string().optional(),
  TWILIO_ACCOUNT_SID: z.string().optional(),
  TWILIO_AUTH_TOKEN: z.string().optional(),
  LLM_API_KEY: z.string().optional(),
  LLM_BASE_URL: z.string().url().default('https://api.openai.com/v1'),
  SMTP_URL: z.string().optional(),
});
export type Env = z.infer<typeof Env>;
```
- `docker-compose.yml`: services `postgres` (16), `redis` (7), `api`, `worker`, `web`. Healthchecks on PG/Redis; api `depends_on` them as healthy.
- Dockerfile build targets selected by `--target` (`api`, `worker`, `web`).

**Testing**:
- Unit: `Env.parse({})` with no DATABASE_URL → ZodError naming `DATABASE_URL`.
- Unit: valid env object → typed `Env` with defaults applied (`PORT=3000`).
- Integration: `docker compose config` validates without error (CI smoke).

#### 1.2 — Database client, migration tooling, and seed harness

**What**: Wire Drizzle + drizzle-kit, a pooled pg client, and a deterministic seed script.

**Design**:
- `createDb(env)` returns a Drizzle instance over a `pg.Pool`. Pool size from env (default 10).
- `db:migrate` runs SQL migrations under `packages/shared/src/db/migrations/`.
- Seed harness inserts: 1 demo tenant, 2 locations, 3 staff, 6 services, 20 clients, products. Used by integration + E2E fixtures.
- Testcontainers helper `withTestDb(fn)` spins ephemeral PG, applies migrations, yields a db handle, tears down.

**Testing**:
- Integration (real PG via Testcontainers): apply all migrations → `information_schema` lists expected tables; re-running migrations is idempotent.
- Integration: run seed → counts match (1 tenant, 2 locations, etc.).

#### 1.3 — Tenant model and tenant-scoped request context

**What**: `tenant` and `location` tables plus middleware that resolves and pins the active tenant.

**Design**:
- Tables (from Suggestion 3, JSONB `settings` for per-tenant config):
```ts
export const tenant = pgTable('tenant', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  subscriptionPlan: text('subscription_plan').notNull().default('free'),
  currencyCode: char('currency_code', { length: 3 }).notNull().default('USD'), // ISO 4217
  timezone: text('timezone').notNull().default('UTC'),
  businessType: text('business_type').notNull().default('salon'), // salon|day_spa|medspa|hotel_spa
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```
- Fastify plugin `tenancy`: resolves tenant from subdomain (`{slug}.host`) or `X-Tenant-Slug` header or the authenticated principal; attaches `req.tenantId`. Rejects with 404 if unknown.
- Every tenant-scoped query MUST filter by `tenant_id`; a Drizzle helper `scoped(table, tenantId)` enforces it; an ESLint rule flags raw selects on tenant tables without a tenant filter (custom rule or review checklist item).

**Testing**:
- Integration: request with unknown slug → 404, no DB leakage.
- Integration: two tenants with same client email → queries return only the active tenant's rows.

#### 1.4 — Authentication: sessions + OAuth 2.1 (PKCE) + RBAC

**What**: Staff/client identity, web sessions, the OAuth 2.1 authorization-code+PKCE flow for API/partner clients, and role-based access control.

**Design**:
- `user_account` table (auth identity, separate from `staff_member`/`client` domain rows; both reference `user_id`). Argon2id password hashing.
- Roles: `owner`, `admin`, `manager`, `receptionist`, `therapist`, `client`, `api_partner`. Permission matrix in `shared/domain/rbac.ts` mapping `(role) → Set<permission>`; route guards take a required permission.
- OAuth 2.1 endpoints (RFC 6749 + RFC 7636): `/oauth/authorize`, `/oauth/token`, `/oauth/revoke`. PKCE `code_challenge` (S256) **required** for public clients. Refresh-token rotation per OAuth security BCP. Tokens are JWTs signed with rotating keys exposed at `/.well-known/jwks.json`.
- Session cookies for web: HttpOnly, Secure, SameSite=Lax, rotating CSRF token.
- `requireAuth(perm)` Fastify preHandler verifies session or Bearer JWT, loads principal, checks `perm`, and confirms tenant match.

**Testing**:
- Unit: RBAC — `receptionist` lacks `commission.read`; `owner` has all.
- Unit (PKCE): token exchange with mismatched `code_verifier` → `invalid_grant`.
- Integration: full auth-code+PKCE flow → access+refresh tokens; refresh rotates and invalidates the old refresh token.
- Integration: Bearer token for tenant A used against tenant B resource → 403.
- Security: password stored as argon2id hash, never plaintext (assert hash prefix `$argon2id$`).

#### 1.5 — Audit log + structured logging + global error taxonomy

**What**: An append-only `audit_log` table, a Pino logger, and a typed error → HTTP mapping.

**Design**:
- `audit_log` per Suggestion 1 (tenant_id, actor_id, actor_type, action, entity_type, entity_id, changes JSONB, ip_address, user_agent, created_at), monthly partitioning planned. Actions include `view_phi` for HIPAA.
- `audit.record(ctx, {action, entityType, entityId, changes})` called from services; never blocks the request path on failure (best-effort, logged).
- Error taxonomy: `AppError` subclasses (`ValidationError`→422, `NotFoundError`→404, `AuthError`→401, `ForbiddenError`→403, `ConflictError`→409, `PaymentError`→402). Global error handler emits RFC 9457 problem+json.

**Testing**:
- Unit: each `AppError` subclass maps to the documented status + problem+json shape.
- Integration: a mutating request writes exactly one audit_log row with correct actor/entity.
- Integration: audit write failure does not fail the underlying request (mock DB error on audit insert).

---

## Phase 2: Catalogue, Staff & Scheduling Rules

### Purpose
Model the operational nouns a booking engine needs before any appointment can exist: services
(with categories, variations, durations, buffers), staff (with certifications, service eligibility,
working hours, and time off), treatment rooms, and per-tenant scheduling rules. After this phase
the system can answer "who can perform what, where, and when are they working" — the inputs to
Phase 3's availability engine.

### Tasks

#### 2.1 — Service catalogue (categories, services, variations, eligibility)

**What**: CRUD for `service_category`, `service`, `service_variation`, and `service_staff` eligibility.

**Design**:
- Tables follow Suggestion 1's columns; service carries `duration_minutes`, `buffer_before_minutes`, `buffer_after_minutes`, `price`, `currency_code`, `tax_rate_pct`, `requires_room`, `room_type`, `is_addon`, `is_online_bookable`, plus a JSONB `attributes` column (Suggestion 3) for business-type-specific fields (e.g. medspa `requires_consult`).
- `service_staff(service_id, staff_id, price_override?, duration_override?)` defines eligibility + per-staff overrides.
- Endpoints (all tenant-scoped, RBAC `catalogue.*`):
  - `GET/POST /services`, `GET/PATCH/DELETE /services/:id`
  - `GET/POST /service-categories`, `GET/POST /service-variations`
  - `PUT /services/:id/staff` (set eligible staff)
- Zod schemas in `shared/schemas/service.ts`; these emit the OpenAPI 3.1 component schemas.

**Testing**:
- Unit: create service with `duration_minutes <= 0` → ValidationError.
- Integration: create category → service in category → `GET /services?categoryId=` returns it.
- Integration: soft-delete service (`is_active=false`) → excluded from default list, still referenced by historical rows.

#### 2.2 — Staff, certifications, and service eligibility

**What**: `staff_member`, `staff_location`, `staff_certification` CRUD + linkage to services.

**Design**:
- Columns per Suggestion 1 (role, employment_type, hourly_rate, commission_pct, is_bookable). JSONB `preferences` (Suggestion 3) for therapist preferences used later by the optimiser.
- Certification carries `expiry_date`; an endpoint `GET /staff/:id/certifications?expiringWithinDays=` flags expiries.
- Eligibility query helper `eligibleStaffForService(serviceId, locationId)` → staff who (a) are bookable, (b) work at the location, (c) appear in `service_staff`, (d) hold any certification the service `attributes.required_certifications` demands and it's unexpired.

**Testing**:
- Unit: `eligibleStaffForService` excludes a therapist whose required certification expired yesterday.
- Integration: assign staff to two locations → appears in both location rosters.

#### 2.3 — Treatment rooms and resource typing

**What**: `treatment_room` CRUD with `room_type` and `capacity`.

**Design**: Rooms belong to a location. Services with `requires_room=true` and a `room_type` constrain which rooms can host them. Capacity supports couples/group rooms.

**Testing**:
- Integration: create a `couples` room (capacity 2) → eligible for a couples-massage service requiring `couples`.
- Unit: a `requires_room` service with no matching room type at a location → resolver returns empty room set (consumed by Phase 3 to block booking).

#### 2.4 — Working hours, time off, and per-tenant scheduling rules

**What**: `staff_schedule` (recurring weekly hours with effective dates), `staff_time_off`, and tenant scheduling settings.

**Design**:
- `staff_schedule(day_of_week 0-6, start_time, end_time, effective_from, effective_until?)`.
- `staff_time_off(start_at, end_at, status)`; only `approved` blocks availability.
- Tenant `settings.scheduling` JSONB: `{ slotGranularityMinutes: 15, minLeadTimeMinutes: 120, maxAdvanceDays: 60, cancellationWindowHours: 24, depositPolicy: {...} }` validated by a Zod schema.
- Pure function `workingIntervalsFor(staffId, date)` → list of `{start,end}` derived from schedule minus approved time off.

**Testing**:
- Unit: staff works Mon 09:00–17:00 with approved time off 12:00–13:00 → `workingIntervalsFor` returns two intervals.
- Unit: effective-dated schedule change → date before `effective_from` uses old hours.
- Unit: tenant scheduling settings with `slotGranularityMinutes: 7` (not a divisor of 60) → ValidationError.

---

## Phase 3: Availability Engine & Appointment Booking

### Purpose
The heart of the product. Compute bookable slots from staff working hours, room availability,
service duration + buffers, and existing bookings; then atomically create appointments without
double-booking staff or rooms. After this phase the platform can take and manage bookings via the
API — the core value proposition.

### Tasks

#### 3.1 — Availability computation

**What**: Given a service (or basket), location, date range, and optional staff preference, return open slots.

**Design**:
- `appointment_group` + `appointment_segment` tables (Suggestion 1) carry `ical_uid`, `ical_sequence`, `start_at`/`end_at` (RFC 5545), `price_at_booking`, status enums.
- Algorithm `computeAvailability(input)`:
  1. Resolve eligible staff (2.2) and eligible rooms (2.3).
  2. For each staff member, get `workingIntervalsFor(date)`; subtract existing `appointment_segment` blocks (including buffers) and `staff_time_off`.
  3. Subtract room conflicts for `requires_room` services.
  4. Slice remaining free intervals into candidate starts on `slotGranularityMinutes`, requiring `total_duration + buffers` to fit and respecting `minLeadTime`/`maxAdvance`.
  5. Return `Slot[] = { start, end, staffId, roomId? }`, deduped by start when staff is unspecified.
- Slot results cached in Redis keyed by `(tenant,location,serviceIds,date,staff?)` with short TTL, invalidated on booking writes affecting that day.
- Endpoint: `GET /availability?locationId&serviceIds&from&to&staffId?`.

**Testing**:
- Unit (fixture clocks): single staff, one existing 10:00–11:00 booking, 60-min service, 15-min granularity, buffers 0 → 10:00 and 11:00 not offered as overlapping; 09:00, 09:15, … 09:00 offered.
- Unit: service buffer_after=15 blocks a back-to-back start.
- Unit: `minLeadTimeMinutes` hides slots within the lead window relative to a fixed `now`.
- Integration: room with capacity 1 already booked → slot excluded even if staff free.

#### 3.2 — Booking creation (atomic, no double-book)

**What**: Create an `appointment_group` with one or more `appointment_segment`s atomically.

**Design**:
- `POST /appointments` body: `{ clientId, locationId, channel, segments: [{serviceId, serviceVariationId?, staffId, roomId?, startAt}], notes? }`.
- Transaction:
  1. Re-validate each segment against live availability inside a `SERIALIZABLE` transaction (or `SELECT … FOR UPDATE` on overlapping segments) to prevent TOCTOU double-booking.
  2. Compute `end_at` from service duration (+ overrides) and `price_at_booking` snapshot.
  3. Insert group (status `booked`, generate `ical_uid = {groupId}@{tenantSlug}`) + segments.
  4. Emit domain event `appointment.booked` (write to a lightweight `domain_event` outbox table for Phase 9 analytics + worker fan-out) and audit-log.
  5. Invalidate availability cache for affected staff/room/day.
- Concurrency: a unique exclusion constraint using `tstzrange` on `(staff_id, time)` and `(room_id, time)` via Postgres `EXCLUDE USING gist` guarantees DB-level no-overlap as the backstop.

**Testing**:
- Integration: two concurrent identical bookings for the same slot → exactly one succeeds, the other gets `ConflictError` (409); DB has one segment.
- Integration: booking a basket of two services with different staff → one group, two segments, contiguous times.
- Unit: `price_at_booking` captures the service price even after the catalogue price later changes.

#### 3.3 — Appointment lifecycle (status transitions)

**What**: Manage the state machine: `booked → confirmed → checked_in → in_progress → completed`, plus `cancelled`, `no_show`, `rescheduled`.

**Design**:
- State machine in `shared/domain/appointment-state.ts`; illegal transitions throw `ConflictError`.
- Endpoints: `POST /appointments/:id/confirm|check-in|start|complete|cancel|no-show`, `POST /appointments/:id/reschedule` (validates new slot, bumps `ical_sequence`).
- Cancellation applies the tenant cancellation policy; if inside the window, flags a fee (charged in Phase 5). Each transition emits a domain event + audit row.

**Testing**:
- Unit: `completed → cancelled` → ConflictError; `booked → cancelled` → allowed.
- Unit: reschedule increments `ical_sequence` and updates segment times.
- Integration: no-show transition increments client `no_show_count` projection (Phase 4).

#### 3.4 — Recurring appointments & waitlist

**What**: `appointment_recurrence` (RFC 5545 RRULE) materialisation and `waitlist_entry`.

**Design**:
- Store an `rrule` string; a worker job (Phase 6 scheduling sweep) materialises the next N occurrences as concrete groups, skipping conflicts and surfacing them to staff.
- Waitlist: when a desired slot is full, `POST /waitlist`; a booking cancellation triggers a `waitlist.match` job that offers the slot (notification in Phase 6) with an `expires_at` hold.

**Testing**:
- Unit: `FREQ=WEEKLY;COUNT=4` from a start date → 4 occurrence dates.
- Integration: cancel a booking with a matching waitlist entry → entry moves to `offered` with `expires_at` set.

---

## Phase 4: Client CRM

### Purpose
Give the business a memory of its clients: profiles, contact details (E.164), preferences, notes,
intake/consent forms, consent tracking (GDPR), and the computed client summary (visits, spend,
no-shows, next appointment) that the front desk and later the AI surface at appointment time.

### Tasks

#### 4.1 — Client profiles and contact data

**What**: `client` CRUD with E.164 phone normalisation and dedupe.

**Design**:
- Columns per Suggestion 1 (preferred_staff_id, preferred_location_id, is_vip, consent flags, tags[]). JSONB `attributes` for hair/skin notes etc.
- Phone normalised to E.164 on write (`libphonenumber-js`); reject invalid.
- Fuzzy dedupe on create: warn (not block) if a client with matching email or normalised phone exists.
- Endpoints: `GET/POST /clients`, `GET/PATCH /clients/:id`, `GET /clients?search=` (name/email/phone).

**Testing**:
- Unit: `0412 345 678` + tenant country AU → stored as `+61412345678`.
- Unit: invalid phone → ValidationError naming `phone`.
- Integration: creating a client with an existing email returns a `possibleDuplicates` array.

#### 4.2 — Notes, alerts, and consultation history

**What**: `client_note` with types (`general|consultation|allergy|preference|skin|hair`) and an `is_alert` flag.

**Design**: Alert notes are returned prominently in the client summary and at check-in. Allergy/contraindication notes feed the medspa module (Phase 8).

**Testing**:
- Integration: add an `is_alert` allergy note → appears in `GET /clients/:id/summary` `alerts[]`.

#### 4.3 — Forms, intake, and GDPR consent

**What**: `form_template`, `client_form_response`, and `data_processing_consent` + erasure workflow.

**Design**:
- `form_template.fields` JSONB (Suggestion 3) describes a dynamic form; responses validated against it.
- Consent records store `consent_type`, `is_granted`, timestamp, `ip_address`, and the exact `consent_text` shown (GDPR provenance).
- `POST /clients/:id/erasure-requests` creates a `data_erasure_request`; a worker job (Phase 6) anonymises the client row and scrubs PII while preserving aggregate/financial integrity (soft-anonymise, not hard delete).
- `GET /clients/:id/export` returns a GDPR data-portability JSON bundle.

**Testing**:
- Unit: response missing a required form field → ValidationError.
- Integration: grant then revoke `marketing_email` consent → latest state is revoked; both events retained.
- Integration: erasure job → name/email/phone anonymised, invoices retained with client reference intact.

#### 4.4 — Client summary projection

**What**: A computed `GET /clients/:id/summary` surfacing visit count, total spend, last/next visit, no-show count, active membership, loyalty, alerts, preferences.

**Design**: Maintained as a denormalised `rm_client`-style row (Suggestion 2 read-model idea) updated by the worker on appointment/invoice events, with a fallback live aggregation. This is what the front desk and the AI receptionist read.

**Testing**:
- Integration: complete an appointment + pay an invoice → summary `total_visits` and `total_spend` increment.
- Integration: no-show → `no_show_count` increments.

---

## Phase 5: POS, Payments & Stripe Integration

### Purpose
Turn completed appointments and retail sales into money: invoices with line items, tax, tips,
discounts; Stripe-tokenised card payments (PCI DSS SAQ-A); deposits and no-show fees; refunds; and
cash/gift-card/membership tenders. After this phase a salon can check a client out end-to-end.

### Tasks

#### 5.1 — Invoices and line items

**What**: `invoice` + `invoice_item` with totals computation.

**Design**:
- Tables per Suggestion 1. `invoice_item.item_type` ∈ `service|product|package_redemption|membership_redemption|gift_card_redemption|tip|discount|fee`.
- Pure calculator `computeInvoiceTotals(items)` → `{subtotal, taxTotal, discountTotal, tipTotal, grandTotal}` using per-item `tax_rate_pct`, ISO 4217 currency. All money in integer minor units internally; `NUMERIC(10,2)` at rest.
- `invoice_number` generated per tenant via a sequence (`INV-{tenant_seq}`).
- An appointment completion can auto-draft an invoice from its segments.

**Testing**:
- Unit: 2 services + 20% tip + 10% discount + tax → exact totals (golden fixture).
- Unit: mixed-currency line items → ValidationError (single currency per invoice).
- Integration: complete appointment → draft invoice pre-populated with service items at `price_at_booking`.

#### 5.2 — Stripe payments (PCI DSS SAQ-A)

**What**: Card payments via Stripe Payment Intents; cards tokenised client-side only.

**Design**:
- `integrations/stripe.ts`: create PaymentIntent (amount, currency, `metadata.invoice_id`), confirm client-side with Stripe.js (no PAN server-side → SAQ-A). `payment` rows store only `stripe_payment_intent_id`.
- Webhook `POST /webhooks/stripe` verifies signature (`STRIPE_WEBHOOK_SECRET`), is idempotent (dedupe on event id), updates `payment.status` and invoice status on `payment_intent.succeeded`/`failed`.
- Connect: each location/tenant maps to a connected account for multi-location payouts (configurable).

**Testing**:
- Integration (Stripe mock/`stripe-mock`): create PaymentIntent → `payment` row pending → webhook `succeeded` → invoice `paid`.
- Integration: webhook with bad signature → 400, no state change.
- Integration: duplicate webhook event id → processed once (idempotent).
- Security: assert no field on `payment` ever stores a raw card number/CVC.

#### 5.3 — Deposits, no-show fees, and refunds

**What**: Capture deposits at booking, charge no-show/cancellation fees per policy, and process refunds.

**Design**:
- Deposit policy in tenant `settings`; if required, booking creates a deposit PaymentIntent and blocks confirmation until paid (or uses a SetupIntent to save a card for later off-session charge of no-show fees).
- No-show transition (3.3) inside the cancellation window triggers an off-session charge job (Phase 6) against the saved payment method.
- Refunds via Stripe Refund API; `payment.status='refunded'`, invoice `refunded`/`partially_paid`.

**Testing**:
- Integration: booking with required deposit, unpaid → confirm blocked (409).
- Integration: no-show within window with saved card → off-session charge created.
- Integration: refund → Stripe refund created, statuses updated.

#### 5.4 — Non-card tenders, gift cards, and split payments

**What**: Cash, gift-card, membership-credit tenders; multiple payments per invoice.

**Design**:
- `gift_card` + `gift_card_transaction` (Suggestion 1); redemption decrements balance atomically and writes a transaction row.
- An invoice can have multiple `payment` rows; invoice closes when paid total ≥ grand total.

**Testing**:
- Unit: gift card balance 50, redeem 30 → balance 20, transaction recorded.
- Integration: split pay (card 100 + cash 45) on a 145 invoice → status `paid`.
- Integration: redeem more than gift-card balance → ConflictError, balance unchanged.

---

## Phase 6: Notifications, Reminders & Worker Jobs

### Purpose
Stand up the BullMQ worker and the asynchronous backbone: appointment confirmations and reminders
(SMS/email), no-show-fee charges, membership rebilling (Phase 7 depends on it), campaign sends
(Phase 9), recurrence materialisation, waitlist offers, GDPR erasure, and read-model projections.
After this phase the platform actively communicates and runs scheduled work.

### Tasks

#### 6.1 — Worker runtime and queue topology

**What**: BullMQ worker process, queue definitions, retry/backoff, and a dead-letter strategy.

**Design**:
- Queues: `notifications`, `payments`, `billing`, `scheduling`, `projections`, `campaigns`, `ai`. Each job typed via Zod payload schema.
- Default job opts: 3 attempts, exponential backoff, removeOnComplete with cap, failed jobs retained for inspection. Repeatable jobs (cron) for the reminder sweep, billing run, and recurrence materialisation.
- Idempotency: jobs carry a natural idempotency key; handlers no-op if already processed (e.g. reminder already sent).

**Testing**:
- Integration (real Redis via Testcontainers): enqueue a job → worker processes it → completes.
- Unit: invalid job payload → rejected before processing (schema guard).

#### 6.2 — Channel adapters: SMS (Twilio) and Email

**What**: `SmsChannel` and `EmailChannel` with templating and `notification_log`.

**Design**:
- `Channel.send({to, template, data}) → {providerId, status}`. SMS via Twilio (E.164), email via SMTP. Every send writes `notification_log` (channel, type, status, reference).
- Templates rendered from per-tenant `settings.templates` with fallbacks. Respect consent flags (no marketing SMS without `sms_consent`).

**Testing**:
- Integration (Twilio mock): send reminder → notification_log row `sent`, provider id stored.
- Unit: marketing send to a client without consent → skipped, logged as `suppressed`.

#### 6.3 — Reminder & confirmation sweeps

**What**: Scheduled sweeps that confirm bookings and send reminders at configurable offsets.

**Design**:
- Repeatable job runs every 15 min; finds appointments needing a confirmation (on create) or reminder (e.g. 24h and 2h before `start_at`), enqueues channel sends, marks reminded to avoid duplicates.
- Offsets configurable in tenant `settings.reminders`.

**Testing**:
- Integration (fixed clock): appointment 24h out → exactly one 24h reminder enqueued; running the sweep again does not re-enqueue.

#### 6.4 — Projection, billing, recurrence, waitlist, and erasure jobs

**What**: The remaining async handlers consumed by other phases.

**Design**:
- `projections`: consume `domain_event` outbox → update `rm_client`/summary, availability invalidation, inventory levels.
- `billing`: daily run charges due memberships (Phase 7), no-show fees (5.3).
- `scheduling`: materialise recurrences (3.4), expire waitlist offers.
- `erasure`: execute GDPR erasure (4.3).

**Testing**:
- Integration: emit `appointment.completed` event → projection updates client summary.
- Integration: membership with `next_billing_date = today` → billing job creates a charge (mock Stripe).

---

## Phase 7: Inventory, Retail, Memberships & Commissions

### Purpose
Complete the commercial operations layer: retail/back-bar product inventory with movement
tracking and low-stock alerts; membership plans (recurring) and prepaid service packages; and
staff commission calculation from invoice items. After this phase a multi-chair salon can run its
full retail + payroll-prep workflow.

### Tasks

#### 7.1 — Products and inventory movements

**What**: `product`, `product_inventory` (per location), `inventory_movement`.

**Design**:
- Product carries `sku`, `barcode`, `retail_price`, `cost_price`, `is_backbar`, `is_retail`, `sds_url` (GHS/SDS), JSONB `attributes`.
- Every stock change writes an `inventory_movement` (`purchase|sale|adjustment|transfer|backbar_usage|waste|return`); `quantity_on_hand` is derived/maintained from movements. Selling a product on an invoice auto-creates a `sale` movement.
- Low-stock partial index (`quantity_on_hand <= reorder_point`); a daily job raises low-stock notifications.

**Testing**:
- Unit: sum of movements equals `quantity_on_hand`.
- Integration: sell product on invoice → `sale` movement, stock decremented.
- Integration: stock crosses reorder point → low-stock alert enqueued.

#### 7.2 — Memberships (recurring) and packages (prepaid)

**What**: `membership_plan`/`client_membership` (Stripe subscriptions) and `package`/`client_package`/`client_package_usage`.

**Design**:
- Membership enroll creates a Stripe Subscription; `client_membership.status` driven by Stripe webhooks (`active|past_due|cancelled`) and the billing job (6.4). Includes `included_services` JSONB (entitlements) and `discount_pct`.
- Package purchase grants a quota of services; redemption at booking/checkout writes `client_package_usage` and decrements remaining.
- Freeze/unfreeze and cancellation-notice rules enforced from plan config.

**Testing**:
- Unit: redeem a service from a 5-pack → remaining 4; redeeming when 0 left → ConflictError.
- Integration: enroll membership → Stripe subscription created (mock), status `active`.
- Integration: Stripe `invoice.payment_failed` webhook → membership `past_due`.

#### 7.3 — Commission calculation

**What**: `commission_rule` (flat/percentage/tiered) and `staff_commission` generation per pay period.

**Design**:
- Pure `computeCommission(rule, invoiceItem, staffPeriodRevenue)` supporting flat, percentage, and tiered (`tier_config` JSONB) structures, scoped by `applies_to` and optional `service_category_id`.
- On invoice payment, generate `staff_commission` rows per service/product item attributed to a staff member; a payroll report (Phase 8) aggregates per pay period.

**Testing**:
- Unit: tiered rule (0–5000 @30%, 5000+ @40%) with period revenue crossing 5000 → blended commission correct.
- Integration: pay an invoice with a service by staff X → one `staff_commission` row for X with correct amount.

---

## Phase 8: Reporting, Analytics, Medspa (HIPAA) & Calendar Sync

### Purpose
Deliver the MVP reporting surface, the HIPAA-compliant medspa module (PHI isolation, RLS, audit),
and the standards-differentiating calendar interoperability (iCalendar export + native CalDAV).
After this phase the MVP feature set from features.md is complete and the platform has its first
true differentiators.

### Tasks

#### 8.1 — Reporting and analytics

**What**: Core reports: revenue, appointments, staff performance, retail, no-show rate, client retention.

**Design**:
- Read-optimised SQL aggregations (some materialised) exposed at `GET /reports/{revenue|appointments|staff|retail|retention}` with date-range + location filters. CSV export endpoints.
- Daily revenue view modelled on Suggestion 2's `v_daily_revenue` idea (aggregating paid invoices), feeding the dashboard.

**Testing**:
- Integration: seed a day of paid invoices → revenue report totals match the seed.
- Unit: retention report groups clients by visit recency buckets correctly.

#### 8.2 — Medspa module (HIPAA)

**What**: `medspa_client_record` and `medspa_treatment_record` with PHI isolation, RLS, encryption, and `view_phi` audit.

**Design**:
- PHI tables isolated (Suggestion 1) with Postgres **row-level security** policies scoping access to clinical roles (`provider`, `medical_admin`); enabled only when tenant `business_type='medspa'`.
- Before/after photo URLs point to encrypted object storage; access is signed, time-limited, and `view_phi`-audited.
- A Business Associate Agreement (BAA) checklist documented; encryption-at-rest assumed at the DB/storage layer.
- Endpoints behind a dedicated `phi.*` permission set; every read writes an `audit_log` `view_phi` row.

**Testing**:
- Integration: non-clinical role reading a medspa record → 403 and an RLS denial (defence in depth).
- Integration: clinical read → succeeds and writes exactly one `view_phi` audit row.
- Unit: medspa endpoints disabled when `business_type != 'medspa'`.

#### 8.3 — iCalendar export (RFC 5545)

**What**: Per-staff and per-client `.ics` feeds.

**Design**:
- `GET /calendars/staff/:id.ics?token=` returns a `text/calendar` document built with `ical-generator`; each appointment segment → a `VEVENT` with `UID`, `DTSTART`, `DTEND`, `SEQUENCE`, `STATUS`, `SUMMARY`. Token-authenticated read-only feed URL.

**Testing**:
- Unit: an appointment → a VEVENT whose `UID` equals the stored `ical_uid` and `SEQUENCE` equals `ical_sequence`.
- Integration: feed validates against an iCalendar parser; cancelled appointments emit `STATUS:CANCELLED`.

#### 8.4 — CalDAV sync (RFC 4791 / 6638)

**What**: A minimal CalDAV server endpoint enabling two-way sync with Google/Apple/Outlook calendars — the differentiator from standards.md.

**Design**:
- Implement the CalDAV verbs needed for a staff calendar collection: `PROPFIND`, `REPORT` (calendar-query, calendar-multiget), `GET`, `PUT`, `DELETE`, with `ETag` per appointment (mapped to `ical_sequence`/`updated_at`). Scoped per staff calendar; auth via the OAuth Bearer/app-password.
- Inbound `PUT` from a client (e.g. dragging an event) maps to a reschedule where permitted; conflicts return `412 Precondition Failed` on ETag mismatch.

**Testing**:
- Integration: `PROPFIND` on a staff calendar → 207 Multi-Status listing appointments.
- Integration: `GET` an event by href → the matching VEVENT; `PUT` with stale ETag → 412.

---

## Phase 9: AI-Native Layer

### Purpose
Deliver the AI augmentation that the README and research frame as the core differentiator:
personalised service/retail recommendations, automated client communication drafting, no-show
risk scoring, dynamic pricing suggestions, and client-summary intelligence at appointment time.
Built on a provider-agnostic LLM gateway plus a lightweight affinity graph (Suggestion 4) and the
event/analytics views (Suggestion 2).

### Tasks

#### 9.1 — LLM gateway and prompt infrastructure

**What**: A provider-agnostic LLM client with prompt templates, structured output, and guardrails.

**Design**:
- `integrations/llm.ts` using the Vercel AI SDK; `complete({system, prompt, schema})` returns Zod-validated structured output. Per-tenant model + key override; cost/usage logged. PHI never sent to external models unless the tenant opts in with a compliant endpoint.
- Prompt templates versioned in `shared/prompts/`. Example system prompt (recommendations):
  ```
  You are a retail and treatment advisor for {{businessType}}. Given a client's visit
  history, purchases, and notes, suggest up to 3 services and 3 retail products. Only
  recommend items in the provided catalogue. Return JSON: { services: [{id, reason}],
  products: [{id, reason}] }.
  ```

**Testing**:
- Unit (mock LLM): malformed model output → retried then surfaced as a typed error, never a raw throw.
- Unit: PHI-bearing field excluded from the outbound prompt unless opt-in flag set.

#### 9.2 — Recommendations + affinity graph

**What**: Personalised post-visit service/retail recommendations.

**Design**:
- Lightweight `graph_edge(src_type, src_id, dst_type, dst_id, weight, props)` table (Suggestion 4) capturing client→service, client→staff affinity, and service↔product co-purchase, updated by the projection worker from booking/invoice events.
- Recommendation endpoint blends graph co-occurrence candidates with an LLM re-ranking pass constrained to the live catalogue; results delivered as post-visit messaging (Phase 6 channels).

**Testing**:
- Integration: clients who bought facial+serum build a service↔product edge → a facial-only client gets the serum recommended.
- Unit: recommendations never include inactive or out-of-catalogue items.

#### 9.3 — No-show risk scoring & dynamic pricing

**What**: A no-show probability per upcoming appointment and dynamic-pricing suggestions.

**Design**:
- Feature extraction from the event/analytics views (Suggestion 2 `v_client_event_sequence`): historical no-show rate, lead time, channel, day/time, deposit status. A simple logistic model (or heuristic baseline) yields a 0–1 score stored on the appointment; high-risk appointments trigger deposit prompts/extra reminders.
- Dynamic pricing suggests off-peak discounts / peak premiums from historical demand by slot; surfaced as suggestions, never auto-applied.

**Testing**:
- Unit: a client with 3 prior no-shows scores higher than one with none (monotonicity check on the heuristic).
- Integration: an appointment scoring above threshold enqueues an extra reminder + deposit prompt.

#### 9.4 — Marketing automation & campaigns

**What**: `campaign` audience targeting, scheduling, and send via the worker.

**Design**:
- `campaign.audience_filter` JSONB (e.g. `{tags:['vip'], lastVisitDaysAgo:30, hasConsent:true}`) compiled to a SQL query over `rm_client`; LLM drafts subject/body from a brief; sends fan out through Phase 6 channels respecting consent; open/click tracked.

**Testing**:
- Integration: campaign targeting `lastVisitDaysAgo>=30` → only lapsed, consented clients enqueued.
- Unit: audience compiler rejects a filter that would include non-consented clients for a marketing channel.

---

## Phase 10: Web Applications (Admin Console + Client Booking)

### Purpose
Deliver the human-facing surfaces: the staff admin console (calendar, checkout, CRM, inventory,
reports) and the client-facing online booking portal. The booking portal must meet WCAG 2.2 AA
(ADA Title II). Both consume the same OpenAPI 3.1 API.

### Tasks

#### 10.1 — Shared web foundation and API client

**What**: Vite + React 19 + TanStack Router/Query + Tailwind + shadcn/ui base, plus a typed API client generated from `openapi.json`.

**Design**:
- Generate a typed client (`openapi-typescript`) so the SPA types track the API. Auth via OAuth PKCE (web-booking) and session (web-admin). Shared design system package.

**Testing**:
- E2E (Playwright): login flow for admin; PKCE login for booking portal.

#### 10.2 — Staff admin console

**What**: Calendar/day view with drag scheduling, checkout/POS, client CRM, inventory, staff, reports dashboards.

**Design**:
- Calendar renders per-staff/room columns from `GET /availability` + existing appointments; drag-create and drag-reschedule call lifecycle endpoints. POS screen builds an invoice, takes Stripe payment (Stripe.js Elements), prints/email receipt. Dashboard renders Phase 8 reports.

**Testing**:
- E2E: create a booking via the calendar → appears in the day view and via API.
- E2E: full checkout with a test card (Stripe test mode) → invoice paid.

#### 10.3 — Client booking portal (WCAG 2.2 AA)

**What**: Public service browse → slot pick → client details/consent → deposit/pay → confirmation, with calendar add.

**Design**:
- Accessible, keyboard-navigable flow; consent capture writes consent records (4.3); deposit via Stripe; confirmation offers an `.ics` download and reminder opt-in.

**Testing**:
- E2E: book an appointment end-to-end as an anonymous client → appointment + client created, confirmation shown.
- Accessibility (`@axe-core/playwright`): booking flow pages have zero serious/critical violations (WCAG 2.2 AA).

---

## Phase 11: MCP Server & Public API Hardening

### Purpose
Ship the headline AI-native differentiator — an MCP server exposing the platform to AI agents —
and finalise the public API for partners: published OpenAPI 3.1 spec, rate limiting, OWASP
hardening, and webhooks. After this phase the platform is integration-ready and uniquely
agent-accessible, which no surveyed competitor offers.

### Tasks

#### 11.1 — MCP server

**What**: An MCP server (`packages/mcp`) exposing tools, resources, and prompts over the MCP spec.

**Design**:
- Tools: `check_availability`, `book_appointment`, `reschedule_appointment`, `cancel_appointment`, `find_client`, `get_client_summary`, `list_services`, `send_message`. Resources: `staff_schedule`, `inventory_levels`, `service_menu`. Prompt templates: booking-recommendation, client-comms draft.
- The MCP server authenticates as a scoped OAuth client, calls the same API services (no business logic duplication), and is tenant-scoped per connection. Tool inputs/outputs use the shared Zod schemas.

**Testing**:
- Integration (MCP client harness): `check_availability` tool returns slots matching `GET /availability`.
- Integration: `book_appointment` tool creates an appointment and is blocked by RBAC if the connection lacks `booking.write`.

#### 11.2 — Public API hardening, OpenAPI publication, and webhooks

**What**: Finalise the OpenAPI 3.1 spec, add rate limiting, OWASP controls, and outbound webhooks.

**Design**:
- CI step regenerates `openapi/openapi.json` from route schemas and fails if it drifts from the committed copy; ReDoc/Swagger UI served at `/docs`.
- Rate limiting (Redis token bucket) per principal/IP; security headers; input size limits; consistent problem+json errors (OWASP A03/A05).
- Outbound webhooks (`appointment.*`, `invoice.paid`, `client.created`) with HMAC signatures and retry — matching the competitor webhook surface (Zenoti/Vagaro/Square).

**Testing**:
- Integration: committed `openapi.json` matches freshly generated spec (CI guard).
- Integration: exceed rate limit → 429 with `Retry-After`.
- Integration: subscribed webhook fires on `invoice.paid` with a valid HMAC signature; failed deliveries retry.
- Security: OWASP smoke suite — SQL injection attempt on `search` is parameterised/escaped; unauthenticated access to a protected route → 401.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Tenancy & Auth        ─── required by everything
    │
Phase 2: Catalogue, Staff & Scheduling     ─── requires 1
    │
Phase 3: Availability & Booking (CORE)      ─── requires 2
    │
    ├── Phase 4: Client CRM                  ─── requires 3 (parallel with 5)
    ├── Phase 5: POS, Payments & Stripe      ─── requires 3 (parallel with 4)
    │       │
    │   Phase 6: Notifications & Worker      ─── requires 3,5 (needed by 7,9)
    │       │
    │   Phase 7: Inventory, Memberships,     ─── requires 5,6
    │            Commissions
    │       │
    │   Phase 8: Reporting, Medspa, Calendar ─── requires 5,7  (8.3/8.4 only need 3)
    │       │
    │   Phase 9: AI-Native Layer             ─── requires 4,6,8
    │
Phase 10: Web Apps (Admin + Booking)         ─── requires 3,4,5 (grows through 8)
    │
Phase 11: MCP Server & API Hardening         ─── requires 3,4,5 (tools grow with later phases)
```

**Parallelism opportunities**:
- Phases **4 and 5** can be built concurrently once Phase 3 lands (CRM vs payments are independent).
- **8.3 (iCalendar) and 8.4 (CalDAV)** depend only on Phase 3 and can be pulled forward / done in parallel with Phases 5–7.
- **Phase 10 (web)** can begin against Phase 3–5 APIs and grow incrementally as later phases add endpoints.
- **Phase 11 (MCP)** tooling can start once Phase 3 booking endpoints exist; tools are added as later phases ship.
- The **MVP (features.md must-haves)** is fully delivered by the end of **Phase 8** (scheduling, POS, CRM, staff, inventory, reporting). Phases 9–11 are the v1.1/differentiator layer.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`pnpm test`), including Testcontainers-backed integration tests for any DB/Redis interaction.
3. Linting and formatting pass (`pnpm lint`, Prettier check).
4. Type checking passes (`pnpm typecheck` / `tsc --noEmit`) with no errors.
5. `docker compose build` succeeds and the affected service starts cleanly.
6. The phase's capability works end-to-end (manual or Playwright walkthrough for UI phases).
7. New configuration options are added to `.env.example` and documented.
8. New API endpoints appear in the generated `openapi/openapi.json`, and the CI drift check passes.
9. Database changes ship as a reversible drizzle-kit migration; `db:migrate` is idempotent and re-runnable.
10. New mutating actions write `audit_log` rows; any PHI access writes a `view_phi` audit row.
11. Test coverage for new domain logic (pricing, availability, commission, state machines) is ≥ 85% lines.
12. Security checklist for the phase reviewed against the relevant OWASP Top 10 items and, where PII/PHI/payments are touched, the GDPR / HIPAA / PCI DSS controls named in `standards.md`.
```
