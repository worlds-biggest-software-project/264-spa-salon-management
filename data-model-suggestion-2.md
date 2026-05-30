# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Spa & Salon Management · Created: 2026-05-22

## Philosophy

This model treats every state change in the spa and salon system as an immutable event appended to an event store. The current state of any entity -- an appointment, a client record, a membership, an inventory count -- is derived by replaying its event history. Read models (projections) are materialised asynchronously from the event stream to serve queries efficiently. This is the Command Query Responsibility Segregation (CQRS) pattern with event sourcing as the persistence mechanism.

The approach is used in production by financial systems (bank ledgers), healthcare record systems (EHR audit trails), and enterprise scheduling platforms. Zenoti's analytics data model already separates "data sources" for appointments, sales, collections, and memberships with distinct granularity -- an event-sourced design formalises this separation at the persistence layer. The HIPAA requirement for complete audit trails of PHI access and modification, the PCI DSS requirement for logging all access to cardholder data, and the GDPR requirement for demonstrating lawful processing all benefit from an architecture where the audit trail IS the database rather than a secondary concern bolted on afterward.

This design is best suited for teams building an AI-native platform where historical patterns are first-class data: predicting no-shows from past booking/cancellation event sequences, analysing client behaviour trajectories for churn prediction, or training recommendation models on the full history of service choices. The event store also enables time-travel queries ("what was this client's membership status on March 15th?") without maintaining separate temporal tables.

**Best for:** Teams prioritising full audit trails, HIPAA/PCI compliance, AI-powered analytics on behavioural patterns, and the ability to reconstruct any past state.

**Trade-offs:**
- Pro: Complete, immutable audit trail by construction -- every change is permanently recorded
- Pro: Time-travel queries: reconstruct the state of any entity at any point in time
- Pro: Rich event stream feeds AI/ML pipelines directly without ETL
- Pro: HIPAA and PCI compliance are architectural properties, not afterthoughts
- Pro: New read models can be added retroactively by replaying the event store
- Con: Higher storage requirements (events accumulate, never deleted)
- Con: Eventual consistency between event store and read models adds complexity
- Con: Simple CRUD queries require read model projections (more moving parts)
- Con: Event schema evolution requires careful versioning (upcasting)
- Con: Steeper learning curve for developers unfamiliar with event sourcing

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HIPAA | Every access to and modification of PHI is an event with actor, timestamp, and IP -- satisfying audit log requirements structurally |
| PCI DSS | Payment events record tokenised references only; the event stream provides the required transaction audit trail |
| GDPR | Right-to-erasure implemented via crypto-shredding (per-client encryption keys destroyed); consent events record full provenance |
| RFC 5545 (iCalendar) | Appointment events carry iCalendar-aligned fields (dtstart, dtend, rrule, uid, sequence) |
| ISO 4217 | Currency codes in all monetary event payloads |
| ISO 3166-1/2 | Location events carry standardised jurisdiction codes |

---

## Event Store (Source of Truth)

```sql
-- ============================================================
-- CORE EVENT STORE
-- ============================================================
-- This is the single source of truth. All other tables are
-- materialised read models (projections) derived from this store.

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,       -- entity ID (client, appointment, etc.)
    stream_type     TEXT NOT NULL,        -- 'client', 'appointment', 'invoice', etc.
    tenant_id       UUID NOT NULL,
    event_type      TEXT NOT NULL,        -- e.g., 'appointment.booked', 'client.updated'
    event_version   INT NOT NULL,         -- per-stream sequence number
    payload         JSONB NOT NULL,       -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {
    --   "actor_id": "uuid",
    --   "actor_type": "staff",
    --   "ip_address": "192.168.1.1",
    --   "user_agent": "...",
    --   "correlation_id": "uuid",
    --   "causation_id": "uuid"
    -- }
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Optimistic concurrency: unique stream + version
    UNIQUE (stream_id, event_version)
);

-- Primary query pattern: replay events for a stream
CREATE INDEX idx_event_stream
    ON event_store (stream_id, event_version);

-- Query all events for a tenant (for projections, analytics)
CREATE INDEX idx_event_tenant_time
    ON event_store (tenant_id, occurred_at);

-- Query events by type (for event handlers / projectors)
CREATE INDEX idx_event_type
    ON event_store (event_type, occurred_at);

-- Partition by month for performance at scale
-- CREATE TABLE event_store_y2026m01 PARTITION OF event_store
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

## Event Type Catalogue

The following event types form the domain language of the system. Each event type has a defined payload schema.

```sql
-- ============================================================
-- EVENT TYPES (reference table, not enforced as FK)
-- ============================================================

CREATE TABLE event_type_registry (
    event_type      TEXT PRIMARY KEY,
    stream_type     TEXT NOT NULL,
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,  -- JSON Schema for payload validation
    version         INT NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types:
-- Stream: client
--   client.created, client.updated, client.consent_granted,
--   client.consent_revoked, client.merged, client.anonymised
--
-- Stream: appointment
--   appointment.booked, appointment.confirmed, appointment.checked_in,
--   appointment.started, appointment.completed, appointment.cancelled,
--   appointment.rescheduled, appointment.no_show,
--   appointment.staff_changed, appointment.room_assigned
--
-- Stream: invoice
--   invoice.created, invoice.item_added, invoice.item_removed,
--   invoice.discount_applied, invoice.tip_added, invoice.paid,
--   invoice.partially_paid, invoice.voided, invoice.refunded
--
-- Stream: membership
--   membership.enrolled, membership.renewed, membership.frozen,
--   membership.unfrozen, membership.cancelled, membership.expired,
--   membership.payment_failed, membership.plan_changed
--
-- Stream: inventory
--   inventory.received, inventory.sold, inventory.adjusted,
--   inventory.transferred, inventory.wasted, inventory.counted
--
-- Stream: gift_card
--   gift_card.issued, gift_card.redeemed, gift_card.refunded,
--   gift_card.expired, gift_card.balance_adjusted
--
-- Stream: staff
--   staff.hired, staff.updated, staff.schedule_set,
--   staff.time_off_requested, staff.time_off_approved,
--   staff.certified, staff.terminated
```

## Example Event Payloads

```sql
-- Example: appointment.booked event payload
-- {
--   "client_id": "550e8400-e29b-41d4-a716-446655440000",
--   "location_id": "660e8400-e29b-41d4-a716-446655440000",
--   "booking_channel": "online",
--   "segments": [
--     {
--       "service_id": "770e8400-...",
--       "service_name": "Deep Tissue Massage",
--       "staff_id": "880e8400-...",
--       "staff_name": "Jane Smith",
--       "room_id": "990e8400-...",
--       "start_at": "2026-06-15T10:00:00-07:00",
--       "end_at": "2026-06-15T11:00:00-07:00",
--       "duration_minutes": 60,
--       "price": 120.00,
--       "currency": "USD"
--     }
--   ],
--   "ical_uid": "appt-550e8400@spa.example.com",
--   "ical_sequence": 0,
--   "notes": "Prefers firm pressure"
-- }

-- Example: appointment.cancelled event payload
-- {
--   "reason": "client_request",
--   "cancellation_fee": 25.00,
--   "cancelled_by": "client",
--   "previous_status": "confirmed"
-- }

-- Example: invoice.paid event payload
-- {
--   "payment_method": "card",
--   "amount": 145.00,
--   "tip_amount": 25.00,
--   "tip_staff_id": "880e8400-...",
--   "currency": "USD",
--   "stripe_payment_intent_id": "pi_3abc...",
--   "items_snapshot": [
--     {"type": "service", "description": "Deep Tissue Massage", "amount": 120.00},
--     {"type": "tip", "description": "Tip for Jane", "amount": 25.00}
--   ]
-- }

-- Example: client.consent_granted event payload
-- {
--   "consent_type": "marketing_email",
--   "consent_text": "I agree to receive promotional emails...",
--   "ip_address": "203.0.113.42",
--   "user_agent": "Mozilla/5.0..."
-- }
```

## Command Handlers (Write Side)

```sql
-- ============================================================
-- COMMAND PROCESSING
-- ============================================================
-- Commands are validated and converted to events.
-- No read models are consulted during command processing --
-- only the event stream for the target aggregate is loaded.

-- Stored procedure: append event with optimistic concurrency
CREATE OR REPLACE FUNCTION append_event(
    p_stream_id     UUID,
    p_stream_type   TEXT,
    p_tenant_id     UUID,
    p_event_type    TEXT,
    p_payload       JSONB,
    p_metadata      JSONB,
    p_expected_version INT
) RETURNS UUID AS $$
DECLARE
    v_event_id UUID;
    v_current_version INT;
BEGIN
    -- Check current version for optimistic concurrency
    SELECT COALESCE(MAX(event_version), 0) INTO v_current_version
    FROM event_store
    WHERE stream_id = p_stream_id;

    IF v_current_version != p_expected_version THEN
        RAISE EXCEPTION 'Concurrency conflict: expected version %, got %',
            p_expected_version, v_current_version;
    END IF;

    INSERT INTO event_store (
        stream_id, stream_type, tenant_id,
        event_type, event_version, payload, metadata
    ) VALUES (
        p_stream_id, p_stream_type, p_tenant_id,
        p_event_type, p_expected_version + 1, p_payload, p_metadata
    )
    RETURNING event_id INTO v_event_id;

    -- Notify projectors via LISTEN/NOTIFY
    PERFORM pg_notify('domain_events', json_build_object(
        'event_id', v_event_id,
        'event_type', p_event_type,
        'stream_id', p_stream_id,
        'tenant_id', p_tenant_id
    )::text);

    RETURN v_event_id;
END;
$$ LANGUAGE plpgsql;
```

## Read Models (Projections)

Read models are materialised views optimised for specific query patterns. They are rebuilt from the event store and can be dropped and recreated at any time.

### Appointment Read Model

```sql
CREATE TABLE rm_appointment (
    id              UUID PRIMARY KEY,  -- same as stream_id
    tenant_id       UUID NOT NULL,
    client_id       UUID NOT NULL,
    location_id     UUID NOT NULL,
    status          TEXT NOT NULL,
    booking_channel TEXT NOT NULL,
    notes           TEXT,
    cancellation_reason TEXT,
    ical_uid        TEXT,
    ical_sequence   INT,
    booked_at       TIMESTAMPTZ NOT NULL,
    confirmed_at    TIMESTAMPTZ,
    checked_in_at   TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    last_event_version INT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_appt_tenant ON rm_appointment (tenant_id, status);
CREATE INDEX idx_rm_appt_client ON rm_appointment (client_id);
CREATE INDEX idx_rm_appt_location ON rm_appointment (location_id);

CREATE TABLE rm_appointment_segment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    appointment_id  UUID NOT NULL REFERENCES rm_appointment(id),
    service_id      UUID NOT NULL,
    service_name    TEXT NOT NULL,
    staff_id        UUID NOT NULL,
    staff_name      TEXT NOT NULL,
    room_id         UUID,
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ NOT NULL,
    duration_minutes INT NOT NULL,
    price           NUMERIC(10, 2) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'scheduled'
);

CREATE INDEX idx_rm_appt_seg_staff ON rm_appointment_segment (staff_id, start_at);
CREATE INDEX idx_rm_appt_seg_room ON rm_appointment_segment (room_id, start_at);
CREATE INDEX idx_rm_appt_seg_time ON rm_appointment_segment (start_at, end_at);
```

### Client Read Model

```sql
CREATE TABLE rm_client (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,
    date_of_birth   DATE,
    gender          TEXT,
    preferred_staff_id UUID,
    preferred_location_id UUID,
    is_vip          BOOLEAN NOT NULL DEFAULT false,
    tags            TEXT[],
    notes           TEXT,
    -- Consent flags (derived from consent events)
    marketing_email_consent BOOLEAN NOT NULL DEFAULT false,
    marketing_sms_consent BOOLEAN NOT NULL DEFAULT false,
    -- Computed aggregates
    total_visits     INT NOT NULL DEFAULT 0,
    total_spend      NUMERIC(12, 2) NOT NULL DEFAULT 0,
    last_visit_at    TIMESTAMPTZ,
    next_appointment_at TIMESTAMPTZ,
    average_rating_given NUMERIC(3, 2),
    no_show_count    INT NOT NULL DEFAULT 0,
    -- Membership snapshot
    active_membership_plan TEXT,
    membership_status TEXT,
    -- Loyalty snapshot
    loyalty_points   INT NOT NULL DEFAULT 0,
    loyalty_tier     TEXT DEFAULT 'standard',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_event_version INT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_client_tenant ON rm_client (tenant_id);
CREATE INDEX idx_rm_client_email ON rm_client (tenant_id, email);
CREATE INDEX idx_rm_client_phone ON rm_client (tenant_id, phone);
CREATE INDEX idx_rm_client_name ON rm_client (tenant_id, last_name, first_name);
```

### Invoice & Revenue Read Model

```sql
CREATE TABLE rm_invoice (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    location_id     UUID NOT NULL,
    client_id       UUID,
    appointment_id  UUID,
    invoice_number  TEXT NOT NULL,
    status          TEXT NOT NULL,
    subtotal        NUMERIC(10, 2) NOT NULL DEFAULT 0,
    tax_total       NUMERIC(10, 2) NOT NULL DEFAULT 0,
    discount_total  NUMERIC(10, 2) NOT NULL DEFAULT 0,
    tip_total       NUMERIC(10, 2) NOT NULL DEFAULT 0,
    grand_total     NUMERIC(10, 2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL,
    items           JSONB NOT NULL DEFAULT '[]',
    payments        JSONB NOT NULL DEFAULT '[]',
    closed_at       TIMESTAMPTZ,
    last_event_version INT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_invoice_tenant ON rm_invoice (tenant_id, created_at);
CREATE INDEX idx_rm_invoice_client ON rm_invoice (client_id);
```

### Staff Schedule Read Model

```sql
CREATE TABLE rm_staff (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    display_name    TEXT,
    role            TEXT NOT NULL,
    is_bookable     BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    locations       UUID[],
    certifications  JSONB NOT NULL DEFAULT '[]',
    commission_pct  NUMERIC(5, 2),
    last_event_version INT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_staff_tenant ON rm_staff (tenant_id, is_active);

CREATE TABLE rm_staff_availability (
    staff_id        UUID NOT NULL,
    location_id     UUID NOT NULL,
    date            DATE NOT NULL,
    slots           JSONB NOT NULL,
    -- e.g., [{"start": "09:00", "end": "12:00"}, {"start": "13:00", "end": "17:00"}]
    booked_segments JSONB NOT NULL DEFAULT '[]',
    -- e.g., [{"start": "09:00", "end": "10:00", "appointment_id": "..."}]
    is_day_off      BOOLEAN NOT NULL DEFAULT false,
    updated_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (staff_id, location_id, date)
);

CREATE INDEX idx_rm_availability_date
    ON rm_staff_availability (location_id, date) WHERE NOT is_day_off;
```

### Inventory Read Model

```sql
CREATE TABLE rm_product_inventory (
    product_id      UUID NOT NULL,
    location_id     UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    product_name    TEXT NOT NULL,
    sku             TEXT,
    barcode         TEXT,
    quantity_on_hand NUMERIC(10, 2) NOT NULL DEFAULT 0,
    reorder_point   NUMERIC(10, 2),
    retail_price    NUMERIC(10, 2),
    cost_price      NUMERIC(10, 2),
    is_low_stock    BOOLEAN NOT NULL DEFAULT false,
    last_movement_at TIMESTAMPTZ,
    last_event_version INT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (product_id, location_id)
);

CREATE INDEX idx_rm_inv_low
    ON rm_product_inventory (location_id) WHERE is_low_stock;
```

### Membership Read Model

```sql
CREATE TABLE rm_membership (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    client_id       UUID NOT NULL,
    plan_name       TEXT NOT NULL,
    plan_id         UUID NOT NULL,
    status          TEXT NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE,
    next_billing_date DATE,
    is_frozen       BOOLEAN NOT NULL DEFAULT false,
    freeze_start    DATE,
    freeze_end      DATE,
    remaining_services JSONB DEFAULT '[]',
    -- e.g., [{"service_id": "...", "remaining": 2, "total": 4}]
    last_event_version INT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_membership_client ON rm_membership (client_id, status);
CREATE INDEX idx_rm_membership_billing ON rm_membership (next_billing_date) WHERE status = 'active';
```

## Projection Engine

```sql
-- ============================================================
-- PROJECTION TRACKING
-- ============================================================
-- Tracks the last event processed by each projector to enable
-- resume and rebuild.

CREATE TABLE projection_checkpoint (
    projector_name  TEXT PRIMARY KEY,
    last_event_id   UUID,
    last_occurred_at TIMESTAMPTZ,
    events_processed BIGINT NOT NULL DEFAULT 0,
    status          TEXT NOT NULL DEFAULT 'running',
        -- running, paused, rebuilding, error
    error_message   TEXT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example projector pseudo-code (implemented in application layer):
--
-- AppointmentProjector:
--   ON appointment.booked:
--     INSERT INTO rm_appointment (...) VALUES (...)
--     INSERT INTO rm_appointment_segment (...) VALUES (...)
--     UPDATE rm_staff_availability SET booked_segments = ...
--     UPDATE rm_client SET next_appointment_at = ...
--
--   ON appointment.cancelled:
--     UPDATE rm_appointment SET status = 'cancelled', ...
--     UPDATE rm_staff_availability SET booked_segments = ...
--     UPDATE rm_client SET no_show_count = ... (if no_show)
```

## GDPR: Crypto-Shredding

```sql
-- ============================================================
-- CRYPTO-SHREDDING FOR GDPR RIGHT TO ERASURE
-- ============================================================
-- Instead of deleting events (which would break the event store),
-- PII in event payloads is encrypted with a per-client key.
-- To erase a client, destroy their encryption key.

CREATE TABLE client_encryption_key (
    client_id       UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    encrypted_key   BYTEA NOT NULL,  -- AES-256 key, encrypted with master key
    key_version     INT NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    destroyed_at    TIMESTAMPTZ  -- set when GDPR erasure is executed
);

CREATE TABLE erasure_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    client_id       UUID NOT NULL,
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          TEXT NOT NULL DEFAULT 'pending',
        -- pending, key_destroyed, projections_scrubbed, completed
    key_destroyed_at TIMESTAMPTZ,
    projections_scrubbed_at TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    processed_by    UUID
);
```

## AI / Analytics Event Stream

```sql
-- ============================================================
-- AI ANALYTICS VIEWS
-- ============================================================
-- These views provide the event stream in formats suitable for
-- ML pipelines and AI feature extraction.

-- Client behaviour sequence for churn / no-show prediction
CREATE VIEW v_client_event_sequence AS
SELECT
    es.stream_id AS entity_id,
    es.tenant_id,
    es.event_type,
    es.occurred_at,
    es.payload ->> 'booking_channel' AS booking_channel,
    es.payload ->> 'status' AS status,
    es.payload ->> 'reason' AS reason,
    LAG(es.occurred_at) OVER (
        PARTITION BY es.payload ->> 'client_id'
        ORDER BY es.occurred_at
    ) AS previous_event_at,
    EXTRACT(EPOCH FROM es.occurred_at - LAG(es.occurred_at) OVER (
        PARTITION BY es.payload ->> 'client_id'
        ORDER BY es.occurred_at
    )) AS seconds_since_previous
FROM event_store es
WHERE es.stream_type = 'appointment'
ORDER BY es.occurred_at;

-- Revenue time series for forecasting
CREATE VIEW v_daily_revenue AS
SELECT
    (es.payload ->> 'location_id')::UUID AS location_id,
    DATE(es.occurred_at) AS revenue_date,
    SUM((es.payload ->> 'amount')::NUMERIC) AS total_revenue,
    COUNT(*) AS transaction_count
FROM event_store es
WHERE es.event_type = 'invoice.paid'
GROUP BY es.payload ->> 'location_id', DATE(es.occurred_at);

-- Time-travel query: reconstruct client state at a specific point in time
CREATE OR REPLACE FUNCTION get_client_at(
    p_client_id UUID,
    p_as_of TIMESTAMPTZ
) RETURNS JSONB AS $$
DECLARE
    v_state JSONB := '{}';
    v_event RECORD;
BEGIN
    FOR v_event IN
        SELECT event_type, payload
        FROM event_store
        WHERE stream_id = p_client_id
          AND stream_type = 'client'
          AND occurred_at <= p_as_of
        ORDER BY event_version
    LOOP
        -- Apply each event to build up the state
        CASE v_event.event_type
            WHEN 'client.created' THEN
                v_state := v_event.payload;
            WHEN 'client.updated' THEN
                v_state := v_state || v_event.payload;
            WHEN 'client.consent_granted' THEN
                v_state := jsonb_set(v_state,
                    ARRAY['consents', v_event.payload ->> 'consent_type'],
                    'true'::jsonb);
            WHEN 'client.consent_revoked' THEN
                v_state := jsonb_set(v_state,
                    ARRAY['consents', v_event.payload ->> 'consent_type'],
                    'false'::jsonb);
            ELSE
                -- Unknown event type: merge payload
                v_state := v_state || v_event.payload;
        END CASE;
    END LOOP;
    RETURN v_state;
END;
$$ LANGUAGE plpgsql;
```

## Snapshot Store (Performance Optimisation)

```sql
-- ============================================================
-- SNAPSHOTS
-- ============================================================
-- For aggregates with many events, store periodic snapshots
-- to avoid replaying the full history on every load.

CREATE TABLE event_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    snapshot_version INT NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- Load aggregate: read snapshot + replay events after snapshot version
-- SELECT state, snapshot_version FROM event_snapshot
--   WHERE stream_id = $1 ORDER BY snapshot_version DESC LIMIT 1;
-- SELECT * FROM event_store
--   WHERE stream_id = $1 AND event_version > $snapshot_version
--   ORDER BY event_version;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | event_store, event_type_registry, event_snapshot |
| Projection Infrastructure | 1 | projection_checkpoint |
| Read Model: Appointments | 2 | rm_appointment, rm_appointment_segment |
| Read Model: Clients | 1 | rm_client |
| Read Model: Staff | 2 | rm_staff, rm_staff_availability |
| Read Model: Invoices | 1 | rm_invoice |
| Read Model: Inventory | 1 | rm_product_inventory |
| Read Model: Memberships | 1 | rm_membership |
| GDPR | 2 | client_encryption_key, erasure_request |
| **Total** | **~14** | Plus views; read models are disposable and rebuildable |

---

## Key Design Decisions

1. **Single event store table** -- all domain events from all stream types go into one table, partitioned by time. This simplifies infrastructure (one table to back up, monitor, replicate) and enables cross-aggregate event correlation for analytics.

2. **JSONB payloads with schema registry** -- event payloads are JSONB for flexibility, but each event type has a registered JSON Schema in `event_type_registry` for validation. This gives the flexibility of schema-free storage with the safety of contract enforcement.

3. **Optimistic concurrency via stream version** -- the `UNIQUE (stream_id, event_version)` constraint prevents concurrent conflicting writes. The `append_event` function checks expected version before inserting, raising an exception on conflict. This eliminates the need for pessimistic locks.

4. **LISTEN/NOTIFY for projection updates** -- PostgreSQL's built-in pub/sub pushes event notifications to projector processes without polling. At higher scale, this could be replaced by a message broker (Kafka, NATS), but pg_notify is sufficient for most spa/salon workloads.

5. **Read models are disposable** -- every `rm_*` table can be dropped and rebuilt from the event store. This means new query patterns can be added retroactively by creating a new projector, and schema changes to read models don't require data migration -- just rebuild.

6. **Crypto-shredding for GDPR** -- rather than deleting events (which would break the immutable event store and potentially corrupt downstream projections), PII in event payloads is encrypted with a per-client key. To exercise the right to erasure, the key is destroyed, rendering the PII unrecoverable while preserving the event stream structure for aggregate analytics.

7. **Denormalised read models** -- read models intentionally duplicate data (e.g., `staff_name` in appointment segments, computed `total_spend` in client records). This eliminates JOINs on the read path, making API queries fast single-table lookups.

8. **Built-in AI analytics views** -- the event stream is directly queryable for ML feature extraction: client behaviour sequences, revenue time series, and booking pattern analysis. No separate ETL pipeline or data warehouse is needed for initial AI use cases.

9. **Time-travel via event replay** -- the `get_client_at` function demonstrates how any entity's state at any historical timestamp can be reconstructed by replaying events up to that point. This is critical for HIPAA compliance ("what did we know about this patient's allergies when we performed the treatment on April 3rd?").

10. **Snapshots for performance** -- aggregates with long event histories (e.g., a loyal client with thousands of appointments) use periodic snapshots to avoid replaying all events from the beginning. The snapshot + subsequent events pattern keeps aggregate load time bounded.
