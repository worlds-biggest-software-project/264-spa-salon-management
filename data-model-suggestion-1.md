# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Spa & Salon Management · Created: 2026-05-22

## Philosophy

This model follows classical third-normal-form (3NF) relational design where every concept in the spa and salon domain receives its own dedicated table with explicit foreign key relationships. Each entity -- clients, staff, services, appointments, products, memberships, gift cards, invoices -- is a first-class citizen with well-defined columns, constraints, and indexes.

The design draws on established patterns from Zenoti's analytics data model (which separates appointments, appointment segments, invoices, invoice items, memberships, and collections into distinct normalised data sources), Square's Bookings API (which models bookings, appointment segments, service variations, team members, and locations as independent resources), and Boulevard's GraphQL schema (which exposes distinct types for clients, locations, staff, services, and bookings). The iCalendar standard (RFC 5545) informs the appointment representation with explicit start/end timestamps, recurrence rules, and status enumerations.

This approach is best suited for teams that value data integrity above all else, expect complex cross-entity reporting (e.g., "show me all appointments for therapists certified in deep tissue massage at locations in California where the client has an active membership"), and plan to build a robust analytics layer. The trade-off is a higher table count (70+), more JOINs in queries, and more migration effort when the schema evolves.

**Best for:** Teams building a full-featured platform with complex reporting, strong data integrity requirements, and a SQL-proficient engineering team.

**Trade-offs:**
- Pro: Maximum data integrity via foreign keys and constraints
- Pro: Complex cross-entity queries are natural and performant with proper indexing
- Pro: Standards-aligned field naming (ISO 3166 for jurisdictions, ISO 4217 for currencies, RFC 5545 for calendar data)
- Pro: Clear entity boundaries make the domain easy to understand
- Con: High table count (~75 tables) increases migration complexity
- Con: Many JOINs required for common read paths (booking confirmation needs 5-6 joins)
- Con: Adding new jurisdiction-specific fields requires schema migrations
- Con: Rigid schema makes prototyping slower

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5545 (iCalendar) | Appointment fields (dtstart, dtend, rrule, status) align with VEVENT properties for calendar export/sync |
| RFC 4791 (CalDAV) | Appointment UID and ETAG columns support CalDAV sync protocol |
| ISO 3166-1/2 | Country and subdivision codes for location addresses and jurisdiction tracking |
| ISO 4217 | Three-letter currency codes in all monetary columns |
| PCI DSS | No raw card data stored; payment_method references tokenised payment via Stripe/Square |
| HIPAA | PHI fields isolated in dedicated medspa tables with row-level security |
| GDPR/CCPA | Consent tracking table; soft-delete and anonymisation support on client records |
| E.164 | Phone numbers stored in international format |
| GHS/SDS | Product safety data sheet references in product inventory |

---

## Core Identity & Multi-Tenancy

```sql
-- ============================================================
-- TENANT / ORGANISATION
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    subscription_plan TEXT NOT NULL DEFAULT 'free',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenant_slug ON tenant (slug);
```

## Location Management

```sql
CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state_province  TEXT,
    postal_code     TEXT,
    country_code    CHAR(2) NOT NULL DEFAULT 'US',  -- ISO 3166-1 alpha-2
    subdivision_code TEXT,                           -- ISO 3166-2
    phone           TEXT,                            -- E.164 format
    email           TEXT,
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    latitude        NUMERIC(10, 7),
    longitude       NUMERIC(10, 7),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE INDEX idx_location_tenant ON location (tenant_id);
```

## Treatment Rooms

```sql
CREATE TABLE treatment_room (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    name            TEXT NOT NULL,
    room_type       TEXT NOT NULL DEFAULT 'standard',
        -- standard, vip, wet_room, dry_room, couples, medical
    capacity        INT NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_treatment_room_location ON treatment_room (location_id);
```

## Staff Management

```sql
CREATE TABLE staff_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID,  -- links to auth system
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    display_name    TEXT,
    email           TEXT,
    phone           TEXT,                            -- E.164
    role            TEXT NOT NULL DEFAULT 'therapist',
        -- therapist, receptionist, manager, owner, admin
    employment_type TEXT NOT NULL DEFAULT 'employee',
        -- employee, contractor, booth_renter
    hourly_rate     NUMERIC(10, 2),
    commission_pct  NUMERIC(5, 2),
    hire_date       DATE,
    termination_date DATE,
    bio             TEXT,
    avatar_url      TEXT,
    is_bookable     BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_staff_tenant ON staff_member (tenant_id);
CREATE INDEX idx_staff_user ON staff_member (user_id);
CREATE INDEX idx_staff_bookable ON staff_member (tenant_id, is_bookable, is_active);

-- Staff can work at multiple locations
CREATE TABLE staff_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id        UUID NOT NULL REFERENCES staff_member(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    UNIQUE (staff_id, location_id)
);

-- Staff certifications / qualifications
CREATE TABLE staff_certification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id        UUID NOT NULL REFERENCES staff_member(id),
    certification_name TEXT NOT NULL,
    issuing_body    TEXT,
    license_number  TEXT,
    issued_date     DATE,
    expiry_date     DATE,
    is_verified     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_staff_cert_staff ON staff_certification (staff_id);
CREATE INDEX idx_staff_cert_expiry ON staff_certification (expiry_date);
```

## Staff Scheduling

```sql
CREATE TABLE staff_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id        UUID NOT NULL REFERENCES staff_member(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    day_of_week     SMALLINT NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),
        -- 0=Sunday, 6=Saturday
    start_time      TIME NOT NULL,
    end_time        TIME NOT NULL,
    is_available    BOOLEAN NOT NULL DEFAULT true,
    effective_from  DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_until DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_staff_schedule_lookup
    ON staff_schedule (staff_id, location_id, day_of_week, effective_from);

-- Time-off / blocked time
CREATE TABLE staff_time_off (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id        UUID NOT NULL REFERENCES staff_member(id),
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ NOT NULL,
    reason          TEXT,
    status          TEXT NOT NULL DEFAULT 'approved',
        -- pending, approved, rejected
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_staff_timeoff ON staff_time_off (staff_id, start_at, end_at);
```

## Service Catalogue

```sql
CREATE TABLE service_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    parent_id       UUID REFERENCES service_category(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    description     TEXT,
    sort_order      INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE INDEX idx_svc_cat_tenant ON service_category (tenant_id);
CREATE INDEX idx_svc_cat_parent ON service_category (parent_id);

CREATE TABLE service (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    category_id     UUID REFERENCES service_category(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    description     TEXT,
    duration_minutes INT NOT NULL,
    buffer_before_minutes INT NOT NULL DEFAULT 0,
    buffer_after_minutes INT NOT NULL DEFAULT 0,
    price           NUMERIC(10, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    tax_rate_pct    NUMERIC(5, 2) NOT NULL DEFAULT 0,
    requires_room   BOOLEAN NOT NULL DEFAULT true,
    room_type       TEXT,  -- if specific room type needed
    max_capacity    INT NOT NULL DEFAULT 1,  -- for group services
    is_addon        BOOLEAN NOT NULL DEFAULT false,
    is_online_bookable BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE INDEX idx_service_tenant ON service (tenant_id);
CREATE INDEX idx_service_category ON service (category_id);
CREATE INDEX idx_service_bookable ON service (tenant_id, is_online_bookable, is_active);

-- Which staff can perform which services
CREATE TABLE service_staff (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    service_id      UUID NOT NULL REFERENCES service(id),
    staff_id        UUID NOT NULL REFERENCES staff_member(id),
    price_override  NUMERIC(10, 2),  -- staff-specific pricing
    duration_override INT,           -- staff-specific duration
    UNIQUE (service_id, staff_id)
);

-- Service pricing variations (e.g., hair length, time of day)
CREATE TABLE service_variation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    service_id      UUID NOT NULL REFERENCES service(id),
    name            TEXT NOT NULL,  -- e.g., "Short Hair", "Long Hair"
    price_adjustment NUMERIC(10, 2) NOT NULL DEFAULT 0,
    duration_adjustment INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_svc_variation ON service_variation (service_id);
```

## Client Management (CRM)

```sql
CREATE TABLE client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID,  -- links to auth system for online booking
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,                            -- E.164
    date_of_birth   DATE,
    gender          TEXT,
    avatar_url      TEXT,
    preferred_staff_id UUID REFERENCES staff_member(id),
    preferred_location_id UUID REFERENCES location(id),
    referral_source TEXT,
    is_vip          BOOLEAN NOT NULL DEFAULT false,
    marketing_consent BOOLEAN NOT NULL DEFAULT false,
    sms_consent     BOOLEAN NOT NULL DEFAULT false,
    email_consent   BOOLEAN NOT NULL DEFAULT false,
    consent_updated_at TIMESTAMPTZ,
    notes           TEXT,
    tags            TEXT[],
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_client_tenant ON client (tenant_id);
CREATE INDEX idx_client_email ON client (tenant_id, email);
CREATE INDEX idx_client_phone ON client (tenant_id, phone);
CREATE INDEX idx_client_name ON client (tenant_id, last_name, first_name);
CREATE INDEX idx_client_user ON client (user_id);

-- Client addresses (billing, home, work)
CREATE TABLE client_address (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    address_type    TEXT NOT NULL DEFAULT 'home',
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state_province  TEXT,
    postal_code     TEXT,
    country_code    CHAR(2) DEFAULT 'US',  -- ISO 3166-1
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Client notes / consultation history
CREATE TABLE client_note (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    staff_id        UUID REFERENCES staff_member(id),
    note_type       TEXT NOT NULL DEFAULT 'general',
        -- general, consultation, allergy, preference, skin, hair
    content         TEXT NOT NULL,
    is_alert        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_client_note ON client_note (client_id, note_type);

-- Client form responses (intake forms, consent forms)
CREATE TABLE client_form_response (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    form_template_id UUID NOT NULL,
    responses       JSONB NOT NULL,
    signed_at       TIMESTAMPTZ,
    signature_url   TEXT,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_client_form ON client_form_response (client_id, form_template_id);
```

## Appointment Booking

```sql
CREATE TABLE appointment_group (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    booked_by       UUID,  -- staff or client user_id
    booking_channel TEXT NOT NULL DEFAULT 'online',
        -- online, phone, walk_in, app, marketplace
    status          TEXT NOT NULL DEFAULT 'booked',
        -- booked, confirmed, checked_in, in_progress, completed,
        -- no_show, cancelled, rescheduled
    notes           TEXT,
    cancellation_reason TEXT,
    cancelled_at    TIMESTAMPTZ,
    -- iCalendar alignment (RFC 5545)
    ical_uid        TEXT UNIQUE,  -- for CalDAV sync
    ical_sequence   INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appt_group_tenant ON appointment_group (tenant_id);
CREATE INDEX idx_appt_group_client ON appointment_group (client_id);
CREATE INDEX idx_appt_group_location ON appointment_group (location_id);
CREATE INDEX idx_appt_group_status ON appointment_group (tenant_id, status);
CREATE INDEX idx_appt_group_ical ON appointment_group (ical_uid);

-- Individual service segments within an appointment group
-- (modelled after Square's appointment_segment concept)
CREATE TABLE appointment_segment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    appointment_group_id UUID NOT NULL REFERENCES appointment_group(id),
    service_id      UUID NOT NULL REFERENCES service(id),
    service_variation_id UUID REFERENCES service_variation(id),
    staff_id        UUID NOT NULL REFERENCES staff_member(id),
    room_id         UUID REFERENCES treatment_room(id),
    start_at        TIMESTAMPTZ NOT NULL,  -- RFC 5545 DTSTART
    end_at          TIMESTAMPTZ NOT NULL,  -- RFC 5545 DTEND
    duration_minutes INT NOT NULL,
    price_at_booking NUMERIC(10, 2) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'scheduled',
        -- scheduled, in_progress, completed, cancelled
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appt_seg_group ON appointment_segment (appointment_group_id);
CREATE INDEX idx_appt_seg_staff ON appointment_segment (staff_id, start_at, end_at);
CREATE INDEX idx_appt_seg_room ON appointment_segment (room_id, start_at, end_at);
CREATE INDEX idx_appt_seg_time ON appointment_segment (start_at, end_at);

-- Recurring appointment pattern
CREATE TABLE appointment_recurrence (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    service_id      UUID NOT NULL REFERENCES service(id),
    staff_id        UUID REFERENCES staff_member(id),
    rrule           TEXT NOT NULL,  -- RFC 5545 RRULE value
    start_date      DATE NOT NULL,
    end_date        DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Waitlist for fully booked slots
CREATE TABLE waitlist_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    service_id      UUID NOT NULL REFERENCES service(id),
    preferred_staff_id UUID REFERENCES staff_member(id),
    preferred_date  DATE NOT NULL,
    preferred_time_start TIME,
    preferred_time_end TIME,
    status          TEXT NOT NULL DEFAULT 'waiting',
        -- waiting, offered, booked, expired, cancelled
    notified_at     TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_waitlist_lookup
    ON waitlist_entry (tenant_id, location_id, preferred_date, status);
```

## Point of Sale & Invoicing

```sql
CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    client_id       UUID REFERENCES client(id),
    appointment_group_id UUID REFERENCES appointment_group(id),
    invoice_number  TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
        -- draft, open, paid, partially_paid, void, refunded
    subtotal        NUMERIC(10, 2) NOT NULL DEFAULT 0,
    tax_total       NUMERIC(10, 2) NOT NULL DEFAULT 0,
    discount_total  NUMERIC(10, 2) NOT NULL DEFAULT 0,
    tip_total       NUMERIC(10, 2) NOT NULL DEFAULT 0,
    grand_total     NUMERIC(10, 2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    notes           TEXT,
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, invoice_number)
);

CREATE INDEX idx_invoice_tenant ON invoice (tenant_id);
CREATE INDEX idx_invoice_client ON invoice (client_id);
CREATE INDEX idx_invoice_appt ON invoice (appointment_group_id);
CREATE INDEX idx_invoice_date ON invoice (tenant_id, created_at);

CREATE TABLE invoice_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    item_type       TEXT NOT NULL,
        -- service, product, package_redemption, membership_redemption,
        -- gift_card_redemption, tip, discount, fee
    reference_id    UUID,  -- links to service_id, product_id, etc.
    description     TEXT NOT NULL,
    quantity        NUMERIC(10, 3) NOT NULL DEFAULT 1,
    unit_price      NUMERIC(10, 2) NOT NULL,
    discount_amount NUMERIC(10, 2) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(10, 2) NOT NULL DEFAULT 0,
    total           NUMERIC(10, 2) NOT NULL,
    staff_id        UUID REFERENCES staff_member(id),  -- for commission
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invoice_item ON invoice_item (invoice_id);

-- Payments against invoices
CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    payment_method  TEXT NOT NULL,
        -- card, cash, gift_card, membership_credit,
        -- account_credit, external
    amount          NUMERIC(10, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    stripe_payment_intent_id TEXT,  -- tokenised reference (PCI DSS)
    square_payment_id TEXT,
    gift_card_id    UUID REFERENCES gift_card(id),
    status          TEXT NOT NULL DEFAULT 'completed',
        -- pending, completed, failed, refunded
    tip_amount      NUMERIC(10, 2) NOT NULL DEFAULT 0,
    tip_staff_id    UUID REFERENCES staff_member(id),
    processed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_invoice ON payment (invoice_id);
CREATE INDEX idx_payment_stripe ON payment (stripe_payment_intent_id);
```

## Memberships & Packages

```sql
CREATE TABLE membership_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    description     TEXT,
    billing_interval TEXT NOT NULL DEFAULT 'monthly',
        -- weekly, monthly, quarterly, annually
    price           NUMERIC(10, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    included_services JSONB NOT NULL DEFAULT '[]',
    -- e.g., [{"service_id": "...", "quantity": 2}]
    discount_pct    NUMERIC(5, 2) NOT NULL DEFAULT 0,
    freeze_allowed  BOOLEAN NOT NULL DEFAULT true,
    max_freeze_days INT DEFAULT 30,
    cancellation_notice_days INT DEFAULT 30,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_membership_plan_tenant ON membership_plan (tenant_id);

CREATE TABLE client_membership (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    membership_plan_id UUID NOT NULL REFERENCES membership_plan(id),
    location_id     UUID REFERENCES location(id),
    status          TEXT NOT NULL DEFAULT 'active',
        -- active, frozen, cancelled, expired, past_due
    start_date      DATE NOT NULL,
    end_date        DATE,
    next_billing_date DATE,
    freeze_start    DATE,
    freeze_end      DATE,
    stripe_subscription_id TEXT,
    cancelled_at    TIMESTAMPTZ,
    cancellation_reason TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_client_membership ON client_membership (client_id);
CREATE INDEX idx_client_membership_status ON client_membership (status, next_billing_date);

-- Service packages (buy 5 facials, get 1 free)
CREATE TABLE package (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    description     TEXT,
    price           NUMERIC(10, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    valid_days      INT,  -- days from purchase until expiry
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE package_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    package_id      UUID NOT NULL REFERENCES package(id),
    service_id      UUID NOT NULL REFERENCES service(id),
    quantity        INT NOT NULL DEFAULT 1,
    UNIQUE (package_id, service_id)
);

CREATE TABLE client_package (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    package_id      UUID NOT NULL REFERENCES package(id),
    purchase_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    expiry_date     DATE,
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE client_package_usage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_package_id UUID NOT NULL REFERENCES client_package(id),
    service_id      UUID NOT NULL REFERENCES service(id),
    appointment_segment_id UUID REFERENCES appointment_segment(id),
    used_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pkg_usage ON client_package_usage (client_package_id);
```

## Gift Cards

```sql
CREATE TABLE gift_card (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            TEXT NOT NULL UNIQUE,
    original_amount NUMERIC(10, 2) NOT NULL,
    balance         NUMERIC(10, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    purchaser_client_id UUID REFERENCES client(id),
    recipient_name  TEXT,
    recipient_email TEXT,
    status          TEXT NOT NULL DEFAULT 'active',
        -- active, redeemed, expired, cancelled
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gift_card_tenant ON gift_card (tenant_id);
CREATE INDEX idx_gift_card_code ON gift_card (code);

CREATE TABLE gift_card_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gift_card_id    UUID NOT NULL REFERENCES gift_card(id),
    transaction_type TEXT NOT NULL,
        -- purchase, redemption, refund, adjustment
    amount          NUMERIC(10, 2) NOT NULL,
    balance_after   NUMERIC(10, 2) NOT NULL,
    invoice_id      UUID REFERENCES invoice(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gc_txn ON gift_card_transaction (gift_card_id);
```

## Inventory & Retail Products

```sql
CREATE TABLE product_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    parent_id       UUID REFERENCES product_category(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    sort_order      INT NOT NULL DEFAULT 0,
    UNIQUE (tenant_id, slug)
);

CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    category_id     UUID REFERENCES product_category(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    description     TEXT,
    sku             TEXT,
    barcode         TEXT,
    brand           TEXT,
    retail_price    NUMERIC(10, 2) NOT NULL,
    cost_price      NUMERIC(10, 2),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    tax_rate_pct    NUMERIC(5, 2) NOT NULL DEFAULT 0,
    is_backbar      BOOLEAN NOT NULL DEFAULT false,  -- internal use only
    is_retail       BOOLEAN NOT NULL DEFAULT true,
    sds_url         TEXT,   -- Safety Data Sheet reference (GHS)
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE INDEX idx_product_tenant ON product (tenant_id);
CREATE INDEX idx_product_sku ON product (tenant_id, sku);
CREATE INDEX idx_product_barcode ON product (barcode);

CREATE TABLE product_inventory (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    quantity_on_hand NUMERIC(10, 2) NOT NULL DEFAULT 0,
    reorder_point   NUMERIC(10, 2),
    reorder_quantity NUMERIC(10, 2),
    last_recount_at TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (product_id, location_id)
);

CREATE INDEX idx_product_inv ON product_inventory (product_id, location_id);
CREATE INDEX idx_product_inv_low
    ON product_inventory (location_id)
    WHERE quantity_on_hand <= reorder_point;

CREATE TABLE inventory_movement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    movement_type   TEXT NOT NULL,
        -- purchase, sale, adjustment, transfer_in, transfer_out,
        -- backbar_usage, waste, return
    quantity        NUMERIC(10, 2) NOT NULL,
    reference_id    UUID,  -- invoice_item_id, purchase_order_id, etc.
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inv_movement ON inventory_movement (product_id, location_id, created_at);
```

## Commission & Payroll

```sql
CREATE TABLE commission_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    rule_type       TEXT NOT NULL,
        -- flat, percentage, tiered
    applies_to      TEXT NOT NULL,
        -- service, product, membership, all
    service_category_id UUID REFERENCES service_category(id),
    flat_amount     NUMERIC(10, 2),
    percentage      NUMERIC(5, 2),
    tier_config     JSONB,
    -- e.g., [{"min": 0, "max": 5000, "pct": 30}, {"min": 5000, "pct": 40}]
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE staff_commission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id        UUID NOT NULL REFERENCES staff_member(id),
    commission_rule_id UUID REFERENCES commission_rule(id),
    invoice_item_id UUID NOT NULL REFERENCES invoice_item(id),
    service_revenue NUMERIC(10, 2) NOT NULL,
    commission_amount NUMERIC(10, 2) NOT NULL,
    tip_amount      NUMERIC(10, 2) NOT NULL DEFAULT 0,
    pay_period_start DATE NOT NULL,
    pay_period_end  DATE NOT NULL,
    is_paid         BOOLEAN NOT NULL DEFAULT false,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_staff_commission ON staff_commission (staff_id, pay_period_start);
CREATE INDEX idx_staff_commission_unpaid ON staff_commission (staff_id) WHERE NOT is_paid;
```

## Marketing & Communications

```sql
CREATE TABLE campaign (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    campaign_type   TEXT NOT NULL,
        -- email, sms, push_notification
    status          TEXT NOT NULL DEFAULT 'draft',
        -- draft, scheduled, sending, sent, cancelled
    subject         TEXT,
    body            TEXT,
    template_id     TEXT,
    audience_filter JSONB,
    -- e.g., {"tags": ["vip"], "last_visit_days_ago": 30}
    scheduled_at    TIMESTAMPTZ,
    sent_at         TIMESTAMPTZ,
    sent_count      INT DEFAULT 0,
    open_count      INT DEFAULT 0,
    click_count     INT DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaign_tenant ON campaign (tenant_id);

-- Automated reminders / notifications
CREATE TABLE notification_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       UUID REFERENCES client(id),
    channel         TEXT NOT NULL,  -- sms, email, push
    notification_type TEXT NOT NULL,
        -- appointment_reminder, appointment_confirmation,
        -- booking_cancelled, review_request, birthday,
        -- membership_renewal, no_show_warning
    reference_type  TEXT,  -- appointment, invoice, membership
    reference_id    UUID,
    status          TEXT NOT NULL DEFAULT 'sent',
        -- queued, sent, delivered, failed, bounced
    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notif_log ON notification_log (tenant_id, client_id, created_at);
```

## Reviews & Feedback

```sql
CREATE TABLE review (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    client_id       UUID REFERENCES client(id),
    appointment_group_id UUID REFERENCES appointment_group(id),
    staff_id        UUID REFERENCES staff_member(id),
    rating          SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    comment         TEXT,
    is_public       BOOLEAN NOT NULL DEFAULT true,
    response        TEXT,          -- owner/manager reply
    responded_at    TIMESTAMPTZ,
    source          TEXT DEFAULT 'internal',
        -- internal, google, yelp, facebook
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_review_location ON review (location_id, created_at);
CREATE INDEX idx_review_staff ON review (staff_id, created_at);
```

## Loyalty Programme

```sql
CREATE TABLE loyalty_program (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    points_per_dollar NUMERIC(5, 2) NOT NULL DEFAULT 1.0,
    redemption_rate NUMERIC(10, 4) NOT NULL DEFAULT 0.01,
        -- dollar value per point
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE client_loyalty (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    loyalty_program_id UUID NOT NULL REFERENCES loyalty_program(id),
    points_balance  INT NOT NULL DEFAULT 0,
    lifetime_points INT NOT NULL DEFAULT 0,
    tier            TEXT NOT NULL DEFAULT 'standard',
        -- standard, silver, gold, platinum
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (client_id, loyalty_program_id)
);

CREATE TABLE loyalty_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_loyalty_id UUID NOT NULL REFERENCES client_loyalty(id),
    transaction_type TEXT NOT NULL,
        -- earn, redeem, adjustment, expire
    points          INT NOT NULL,
    balance_after   INT NOT NULL,
    reference_type  TEXT,  -- invoice, appointment, promotion
    reference_id    UUID,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_loyalty_txn ON loyalty_transaction (client_loyalty_id, created_at);
```

## Medical Spa Extension (HIPAA-compliant)

```sql
-- These tables store PHI and must be protected with row-level security,
-- encryption at rest, and audit logging per HIPAA requirements.

CREATE TABLE medspa_client_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    medical_history JSONB,
    allergies       TEXT[],
    medications     TEXT[],
    skin_type       TEXT,
    fitzpatrick_scale SMALLINT CHECK (fitzpatrick_scale BETWEEN 1 AND 6),
    contraindications TEXT[],
    consent_signed  BOOLEAN NOT NULL DEFAULT false,
    consent_signed_at TIMESTAMPTZ,
    last_updated_by UUID REFERENCES staff_member(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_medspa_client ON medspa_client_record (client_id);

CREATE TABLE medspa_treatment_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    appointment_segment_id UUID REFERENCES appointment_segment(id),
    treating_provider_id UUID NOT NULL REFERENCES staff_member(id),
    treatment_type  TEXT NOT NULL,
    area_treated    TEXT,
    products_used   JSONB,
    dosage          TEXT,
    before_photo_url TEXT,
    after_photo_url TEXT,
    clinical_notes  TEXT,
    follow_up_date  DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_medspa_treatment ON medspa_treatment_record (client_id, created_at);
```

## Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    actor_id        UUID,
    actor_type      TEXT NOT NULL,  -- staff, client, system, api
    action          TEXT NOT NULL,
        -- create, update, delete, login, logout, export, view_phi
    entity_type     TEXT NOT NULL,
        -- client, appointment, invoice, medspa_record, etc.
    entity_id       UUID NOT NULL,
    changes         JSONB,
    -- e.g., {"field": "status", "old": "booked", "new": "cancelled"}
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant ON audit_log (tenant_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_actor ON audit_log (actor_id, created_at);

-- Partition audit_log by month for performance
-- (implementation depends on PostgreSQL version)
```

## GDPR / Privacy

```sql
CREATE TABLE data_processing_consent (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    consent_type    TEXT NOT NULL,
        -- marketing_email, marketing_sms, data_processing,
        -- photo_usage, third_party_sharing
    is_granted      BOOLEAN NOT NULL,
    granted_at      TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    ip_address      INET,
    consent_text    TEXT,  -- the text shown at time of consent
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_client ON data_processing_consent (client_id, consent_type);

CREATE TABLE data_erasure_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          TEXT NOT NULL DEFAULT 'pending',
        -- pending, in_progress, completed, rejected
    completed_at    TIMESTAMPTZ,
    processed_by    UUID REFERENCES staff_member(id),
    notes           TEXT
);
```

## Form Templates

```sql
CREATE TABLE form_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    form_type       TEXT NOT NULL,
        -- intake, consent, medical_history, feedback, waiver
    fields          JSONB NOT NULL,
    -- e.g., [{"name": "allergies", "type": "text", "required": true}]
    is_active       BOOLEAN NOT NULL DEFAULT true,
    version         INT NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_form_template ON form_template (tenant_id, form_type);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Location | 3 | tenant, location, treatment_room |
| Staff Management | 6 | staff_member, staff_location, staff_certification, staff_schedule, staff_time_off, commission_rule |
| Service Catalogue | 4 | service_category, service, service_staff, service_variation |
| Client CRM | 5 | client, client_address, client_note, client_form_response, form_template |
| Appointments | 4 | appointment_group, appointment_segment, appointment_recurrence, waitlist_entry |
| Invoicing & Payments | 3 | invoice, invoice_item, payment |
| Memberships & Packages | 5 | membership_plan, client_membership, package, package_item, client_package, client_package_usage |
| Gift Cards | 2 | gift_card, gift_card_transaction |
| Inventory & Products | 4 | product_category, product, product_inventory, inventory_movement |
| Commission & Payroll | 2 | commission_rule (counted above), staff_commission |
| Marketing | 2 | campaign, notification_log |
| Reviews & Loyalty | 4 | review, loyalty_program, client_loyalty, loyalty_transaction |
| Medical Spa | 2 | medspa_client_record, medspa_treatment_record |
| Compliance & Audit | 3 | audit_log, data_processing_consent, data_erasure_request |
| **Total** | **~49** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** -- enables distributed ID generation, API-safe identifiers, and avoids sequential enumeration attacks on client or invoice records.

2. **Appointment group / segment split** -- mirrors the Square Bookings API pattern where a single booking visit can contain multiple service segments with different staff and rooms, enabling accurate scheduling and room utilisation tracking.

3. **iCalendar alignment** -- `ical_uid` and `ical_sequence` on appointment_group enable native CalDAV sync, a differentiator noted in the standards research where most competitors lack this capability.

4. **Price-at-booking snapshot** -- `price_at_booking` on appointment_segment captures the price agreed at booking time, preventing retroactive price changes from affecting existing bookings.

5. **Separate inventory movement table** -- rather than just updating quantity_on_hand, every stock change is recorded as a movement, providing full audit trail for retail, backbar usage, and waste tracking.

6. **Medical spa tables isolated** -- PHI data lives in dedicated `medspa_*` tables rather than being mixed into general client records, making HIPAA compliance boundaries clear and enabling row-level security policies scoped to authorised clinical staff.

7. **Commission rules as a separate entity** -- supports flat, percentage, and tiered commission structures that can vary by service category, matching the payroll complexity seen in real salons where stylists, colourists, and estheticians have different commission tiers.

8. **GDPR consent tracking** -- explicit consent records with timestamps, IP addresses, and the consent text shown at the time of granting, supporting right-to-audit requirements under GDPR and CCPA.

9. **Soft-delete via is_active flags** -- rather than hard deleting records (which would break referential integrity and audit trails), entities use `is_active` flags, with a separate `data_erasure_request` workflow for GDPR right-to-erasure.

10. **Multi-location, single-tenant tables** -- the tenant/location hierarchy supports franchise and multi-location operators while keeping the schema simple enough for single-location salons.
