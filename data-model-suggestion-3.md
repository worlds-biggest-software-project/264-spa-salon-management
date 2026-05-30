# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Spa & Salon Management · Created: 2026-05-22

## Philosophy

This model keeps the core operational structure relational -- appointments, clients, staff, invoices have well-defined columns with foreign keys and constraints -- but pushes all variable, jurisdiction-specific, or rapidly-evolving fields into JSONB columns. The result is a smaller table count than a fully normalised model, faster iteration on new features (no schema migration needed to add a field to a JSONB column), and built-in flexibility for multi-region deployments where different jurisdictions require different fields.

This is the pragmatic middle ground used by modern SaaS platforms including Stripe (which stores payment method details in JSONB-like structures), Shopify (which uses metafields for extensible product attributes), and many multi-tenant booking platforms. The approach works particularly well for a spa and salon platform because: (a) service catalogues vary wildly between a hair salon, a day spa, a medical spa, and a hotel spa; (b) client intake forms differ by treatment type and jurisdiction; (c) commission structures, tax rules, and regulatory requirements vary by region; and (d) the platform needs to evolve rapidly during the MVP phase without constant migrations.

PostgreSQL's JSONB support provides the best of both worlds: GIN indexes on JSONB columns enable fast queries on nested fields, the `@>` containment operator supports efficient filtering, and JSON Schema validation (via CHECK constraints or application-layer validation) can enforce structure where needed while remaining optional for truly freeform data.

**Best for:** Teams building an MVP that must support diverse business types (salons, spas, medspas, hotel spas) across multiple jurisdictions, or teams that expect rapid feature iteration and want to minimise schema migrations.

**Trade-offs:**
- Pro: Fewer tables (~30) means simpler mental model and fewer JOINs
- Pro: No schema migration required to add jurisdiction-specific or business-type-specific fields
- Pro: JSONB columns with GIN indexes provide fast queries on variable fields
- Pro: Rapid MVP development -- new features can ship as JSONB fields before being promoted to columns
- Pro: Multi-region flexibility without separate schemas per jurisdiction
- Con: JSONB fields lack database-enforced constraints (nullability, type, referential integrity)
- Con: JSONB queries are slightly slower than column queries for indexed lookups
- Con: Risk of schema drift if JSONB structures are not validated at the application layer
- Con: Reporting queries on JSONB fields are more verbose (jsonb_path operators)
- Con: ORMs handle JSONB less gracefully than flat columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5545 (iCalendar) | Appointment start/end as relational columns; recurrence rules and custom calendar properties in JSONB |
| ISO 3166-1/2 | Location country_code as relational column; jurisdiction-specific tax/regulatory config in JSONB |
| ISO 4217 | Currency code as relational column on all monetary tables |
| PCI DSS | Payment tokens stored relationally; payment method details in JSONB (never raw card data) |
| HIPAA | Medical records in JSONB within dedicated medspa_profile; row-level security on the table |
| GDPR/CCPA | Consent preferences in JSONB on client; erasure via JSONB field scrubbing + anonymisation |
| JSON Schema | JSONB columns validated at application layer against registered JSON Schemas |
| GHS/SDS | Product safety information in JSONB on product table |

---

## Multi-Tenancy & Configuration

```sql
-- ============================================================
-- TENANT
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    subscription_plan TEXT NOT NULL DEFAULT 'free',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',     -- ISO 4217
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    business_type   TEXT NOT NULL DEFAULT 'salon',
        -- salon, day_spa, medical_spa, hotel_spa, barbershop, nail_salon
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "booking": {
    --     "min_advance_hours": 2,
    --     "max_advance_days": 90,
    --     "cancellation_window_hours": 24,
    --     "no_show_fee_pct": 50,
    --     "require_deposit": true,
    --     "deposit_pct": 25
    --   },
    --   "notifications": {
    --     "reminder_hours_before": [24, 2],
    --     "sms_enabled": true,
    --     "email_enabled": true
    --   },
    --   "tax": {
    --     "default_service_tax_pct": 8.25,
    --     "default_product_tax_pct": 8.25,
    --     "tax_inclusive": false
    --   },
    --   "branding": {
    --     "primary_color": "#2D3436",
    --     "logo_url": "https://..."
    --   },
    --   "compliance": {
    --     "hipaa_enabled": false,
    --     "gdpr_enabled": true,
    --     "data_retention_months": 84
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Locations

```sql
CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    country_code    CHAR(2) NOT NULL DEFAULT 'US',      -- ISO 3166-1
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "address": {
    --     "line1": "123 Main St",
    --     "line2": "Suite 200",
    --     "city": "Los Angeles",
    --     "state": "CA",
    --     "postal_code": "90001",
    --     "subdivision_code": "US-CA"
    --   },
    --   "contact": {
    --     "phone": "+13105551234",
    --     "email": "la@example.com"
    --   },
    --   "geo": {"lat": 34.0522, "lng": -118.2437},
    --   "hours": {
    --     "mon": {"open": "09:00", "close": "20:00"},
    --     "tue": {"open": "09:00", "close": "20:00"},
    --     "sun": null
    --   },
    --   "rooms": [
    --     {"id": "r1", "name": "Room 1", "type": "standard", "capacity": 1},
    --     {"id": "r2", "name": "Couples Suite", "type": "couples", "capacity": 2},
    --     {"id": "r3", "name": "Wet Room", "type": "wet_room", "capacity": 1}
    --   ],
    --   "amenities": ["sauna", "steam_room", "pool", "locker_room"],
    --   "tax_overrides": {
    --     "service_tax_pct": 9.5
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE INDEX idx_location_tenant ON location (tenant_id);
-- GIN index for querying JSONB details (e.g., amenities, room types)
CREATE INDEX idx_location_details ON location USING GIN (details jsonb_path_ops);
```

## Staff

```sql
CREATE TABLE staff (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,                                -- E.164
    role            TEXT NOT NULL DEFAULT 'therapist',
    is_bookable     BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "display_name": "Jane S.",
    --   "bio": "Licensed massage therapist with 10 years...",
    --   "avatar_url": "https://...",
    --   "employment_type": "employee",
    --   "hire_date": "2024-03-15",
    --   "hourly_rate": 28.00,
    --   "locations": ["loc-uuid-1", "loc-uuid-2"],
    --   "primary_location": "loc-uuid-1",
    --   "certifications": [
    --     {"name": "Licensed Massage Therapist", "number": "LMT-12345",
    --      "issuer": "CA Board", "expiry": "2027-12-31"}
    --   ],
    --   "commission": {
    --     "type": "tiered",
    --     "tiers": [
    --       {"min": 0, "max": 5000, "pct": 30},
    --       {"min": 5000, "pct": 40}
    --     ]
    --   },
    --   "specialties": ["deep_tissue", "hot_stone", "prenatal"],
    --   "color": "#4A90D9"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_staff_tenant ON staff (tenant_id, is_active);
CREATE INDEX idx_staff_profile ON staff USING GIN (profile jsonb_path_ops);

-- Staff schedule: relational for efficient availability queries
CREATE TABLE staff_schedule_block (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id        UUID NOT NULL REFERENCES staff(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    block_type      TEXT NOT NULL,
        -- available, time_off, break, personal
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ NOT NULL,
    recurrence      JSONB,
    -- recurrence example (RFC 5545 aligned):
    -- {"rrule": "FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR",
    --  "effective_from": "2026-01-01",
    --  "effective_until": "2026-12-31"}
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_schedule_block ON staff_schedule_block (staff_id, start_at, end_at);
CREATE INDEX idx_schedule_location ON staff_schedule_block (location_id, start_at);
```

## Service Catalogue

```sql
CREATE TABLE service (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    category        TEXT NOT NULL DEFAULT 'uncategorised',
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    description     TEXT,
    duration_minutes INT NOT NULL,
    price           NUMERIC(10, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    is_addon        BOOLEAN NOT NULL DEFAULT false,
    is_online_bookable BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    sort_order      INT NOT NULL DEFAULT 0,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "category_path": ["Body", "Massage"],
    --   "buffer_before": 0,
    --   "buffer_after": 15,
    --   "requires_room_type": "standard",
    --   "max_group_size": 1,
    --   "tax_rate_pct": 8.25,
    --   "variations": [
    --     {"name": "30 min", "duration": 30, "price": 65.00},
    --     {"name": "60 min", "duration": 60, "price": 120.00},
    --     {"name": "90 min", "duration": 90, "price": 170.00}
    --   ],
    --   "staff_pricing": {
    --     "staff-uuid-senior": {"price": 140.00, "duration": 60}
    --   },
    --   "eligible_staff": ["staff-uuid-1", "staff-uuid-2"],
    --   "products_used": [
    --     {"product_id": "prod-uuid", "quantity": 0.5, "unit": "oz"}
    --   ],
    --   "consent_form_required": false,
    --   "medspa": {
    --     "requires_consultation": true,
    --     "contraindications": ["pregnancy", "blood_thinners"],
    --     "pre_care_instructions": "Avoid sun exposure 48 hours before..."
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE INDEX idx_service_tenant ON service (tenant_id, is_active);
CREATE INDEX idx_service_category ON service (tenant_id, category);
CREATE INDEX idx_service_details ON service USING GIN (details jsonb_path_ops);
```

## Clients

```sql
CREATE TABLE client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "date_of_birth": "1990-05-15",
    --   "gender": "female",
    --   "avatar_url": "https://...",
    --   "addresses": [
    --     {"type": "home", "line1": "456 Oak Ave", "city": "LA",
    --      "state": "CA", "postal": "90002", "country": "US"}
    --   ],
    --   "referral_source": "google",
    --   "tags": ["vip", "regular"],
    --   "preferences": {
    --     "preferred_staff_id": "staff-uuid",
    --     "preferred_location_id": "loc-uuid",
    --     "pressure_preference": "firm",
    --     "temperature_preference": "warm",
    --     "aromatherapy_preference": "lavender",
    --     "music_preference": "nature_sounds"
    --   },
    --   "notes": [
    --     {"type": "allergy", "text": "Allergic to almond oil", "is_alert": true},
    --     {"type": "hair", "text": "Fine hair, prone to breakage"},
    --     {"type": "skin", "text": "Sensitive skin on back"}
    --   ]
    -- }
    consents        JSONB NOT NULL DEFAULT '{}',
    -- consents example:
    -- {
    --   "marketing_email": {"granted": true, "at": "2026-01-15T10:00:00Z",
    --                       "ip": "203.0.113.42"},
    --   "marketing_sms": {"granted": false, "at": "2026-01-15T10:00:00Z"},
    --   "data_processing": {"granted": true, "at": "2026-01-15T10:00:00Z",
    --                       "text": "I consent to..."}
    -- }
    stats           JSONB NOT NULL DEFAULT '{}',
    -- stats example (updated by triggers/background jobs):
    -- {
    --   "total_visits": 47,
    --   "total_spend": 5640.00,
    --   "last_visit_at": "2026-05-01T14:00:00Z",
    --   "next_appointment_at": "2026-06-15T10:00:00Z",
    --   "no_show_count": 1,
    --   "average_rating_given": 4.8,
    --   "loyalty_points": 2340,
    --   "loyalty_tier": "gold",
    --   "active_membership": "monthly_unlimited",
    --   "churn_risk_score": 0.12
    -- }
    medspa_profile  JSONB,  -- NULL unless tenant is medspa
    -- medspa_profile example (HIPAA-protected):
    -- {
    --   "medical_history": {"conditions": ["eczema"], "medications": ["Accutane"]},
    --   "allergies": ["latex", "almond_oil"],
    --   "skin_type": "combination",
    --   "fitzpatrick_scale": 3,
    --   "contraindications": ["pregnancy"],
    --   "consent_signed": true,
    --   "consent_signed_at": "2026-01-10T09:00:00Z",
    --   "treatments": [
    --     {"date": "2026-03-01", "type": "chemical_peel", "provider": "Dr. Kim",
    --      "notes": "Glycolic 30%, well tolerated", "before_photo": "...", "after_photo": "..."}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_client_tenant ON client (tenant_id);
CREATE INDEX idx_client_email ON client (tenant_id, email);
CREATE INDEX idx_client_phone ON client (tenant_id, phone);
CREATE INDEX idx_client_name ON client (tenant_id, last_name, first_name);
CREATE INDEX idx_client_profile ON client USING GIN (profile jsonb_path_ops);
CREATE INDEX idx_client_stats ON client USING GIN (stats jsonb_path_ops);

-- Example queries using JSONB:
--
-- Find VIP clients who haven't visited in 30 days:
-- SELECT * FROM client
-- WHERE tenant_id = $1
--   AND profile @> '{"tags": ["vip"]}'
--   AND (stats ->> 'last_visit_at')::timestamptz < now() - interval '30 days';
--
-- Find clients with allergy alerts:
-- SELECT * FROM client
-- WHERE tenant_id = $1
--   AND profile @> '{"notes": [{"type": "allergy", "is_alert": true}]}';
--
-- Find clients with active memberships at risk of churn:
-- SELECT * FROM client
-- WHERE tenant_id = $1
--   AND stats ? 'active_membership'
--   AND (stats ->> 'churn_risk_score')::numeric > 0.7;
```

## Appointments

```sql
CREATE TABLE appointment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    status          TEXT NOT NULL DEFAULT 'booked',
        -- booked, confirmed, checked_in, in_progress,
        -- completed, no_show, cancelled
    booking_channel TEXT NOT NULL DEFAULT 'online',
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ NOT NULL,
    ical_uid        TEXT UNIQUE,
    segments        JSONB NOT NULL,
    -- segments example:
    -- [
    --   {
    --     "id": "seg-uuid-1",
    --     "service_id": "svc-uuid",
    --     "service_name": "Deep Tissue Massage",
    --     "variation": "60 min",
    --     "staff_id": "staff-uuid",
    --     "staff_name": "Jane S.",
    --     "room_id": "r1",
    --     "start_at": "2026-06-15T10:00:00-07:00",
    --     "end_at": "2026-06-15T11:00:00-07:00",
    --     "duration_minutes": 60,
    --     "price": 120.00,
    --     "status": "scheduled"
    --   },
    --   {
    --     "id": "seg-uuid-2",
    --     "service_id": "svc-uuid-2",
    --     "service_name": "Express Facial",
    --     "staff_id": "staff-uuid-2",
    --     "staff_name": "Kim L.",
    --     "room_id": "r2",
    --     "start_at": "2026-06-15T11:15:00-07:00",
    --     "end_at": "2026-06-15T11:45:00-07:00",
    --     "duration_minutes": 30,
    --     "price": 65.00,
    --     "status": "scheduled"
    --   }
    -- ]
    extras          JSONB NOT NULL DEFAULT '{}',
    -- extras example:
    -- {
    --   "notes": "Prefers firm pressure, allergic to almond oil",
    --   "cancellation_reason": null,
    --   "cancelled_at": null,
    --   "confirmed_at": "2026-06-14T10:00:00Z",
    --   "checked_in_at": null,
    --   "booked_by": "staff-uuid",
    --   "deposit_amount": 30.00,
    --   "deposit_paid": true,
    --   "reminders_sent": [
    --     {"channel": "sms", "sent_at": "2026-06-14T10:00:00Z"},
    --     {"channel": "email", "sent_at": "2026-06-14T10:00:00Z"}
    --   ],
    --   "feedback": {
    --     "rating": 5,
    --     "comment": "Amazing massage!",
    --     "submitted_at": "2026-06-15T12:00:00Z"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appt_tenant ON appointment (tenant_id, status);
CREATE INDEX idx_appt_client ON appointment (client_id);
CREATE INDEX idx_appt_location_time ON appointment (location_id, start_at, end_at);
CREATE INDEX idx_appt_time ON appointment (start_at, end_at);
CREATE INDEX idx_appt_segments ON appointment USING GIN (segments jsonb_path_ops);

-- Staff-specific appointment lookup requires a functional index
-- since staff_id is inside the JSONB segments array
CREATE INDEX idx_appt_staff ON appointment USING GIN (
    (SELECT jsonb_agg(s ->> 'staff_id')
     FROM jsonb_array_elements(segments) s)
);
-- Alternative: use a generated column or a separate lookup table
-- for high-frequency staff schedule queries:

CREATE TABLE appointment_staff_lookup (
    appointment_id  UUID NOT NULL REFERENCES appointment(id) ON DELETE CASCADE,
    staff_id        UUID NOT NULL,
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (appointment_id, staff_id)
);

CREATE INDEX idx_appt_staff_time ON appointment_staff_lookup (staff_id, start_at, end_at);

-- Similarly for room availability checks:
CREATE TABLE appointment_room_lookup (
    appointment_id  UUID NOT NULL REFERENCES appointment(id) ON DELETE CASCADE,
    room_id         TEXT NOT NULL,  -- room_id from location.details.rooms
    location_id     UUID NOT NULL,
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (appointment_id, room_id)
);

CREATE INDEX idx_appt_room_time ON appointment_room_lookup (location_id, room_id, start_at, end_at);
```

## Invoices & Payments

```sql
CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    client_id       UUID REFERENCES client(id),
    appointment_id  UUID REFERENCES appointment(id),
    invoice_number  TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
        -- draft, open, paid, partially_paid, void, refunded
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    totals          JSONB NOT NULL DEFAULT '{}',
    -- totals example:
    -- {
    --   "subtotal": 185.00,
    --   "tax": 15.28,
    --   "discount": 0,
    --   "tip": 30.00,
    --   "grand_total": 230.28
    -- }
    items           JSONB NOT NULL DEFAULT '[]',
    -- items example:
    -- [
    --   {
    --     "id": "item-uuid-1",
    --     "type": "service",
    --     "reference_id": "svc-uuid",
    --     "description": "Deep Tissue Massage - 60 min",
    --     "quantity": 1,
    --     "unit_price": 120.00,
    --     "discount": 0,
    --     "tax": 9.90,
    --     "total": 129.90,
    --     "staff_id": "staff-uuid",
    --     "commission_pct": 35
    --   },
    --   {
    --     "id": "item-uuid-2",
    --     "type": "product",
    --     "reference_id": "prod-uuid",
    --     "description": "Aromatherapy Oil 8oz",
    --     "quantity": 1,
    --     "unit_price": 35.00,
    --     "discount": 0,
    --     "tax": 2.89,
    --     "total": 37.89,
    --     "staff_id": "staff-uuid"
    --   }
    -- ]
    payments        JSONB NOT NULL DEFAULT '[]',
    -- payments example:
    -- [
    --   {
    --     "id": "pay-uuid-1",
    --     "method": "card",
    --     "amount": 200.28,
    --     "stripe_pi": "pi_3abc...",
    --     "processed_at": "2026-06-15T12:00:00Z"
    --   },
    --   {
    --     "id": "pay-uuid-2",
    --     "method": "gift_card",
    --     "amount": 30.00,
    --     "gift_card_code": "GC-ABCD-1234",
    --     "processed_at": "2026-06-15T12:00:00Z"
    --   }
    -- ]
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, invoice_number)
);

CREATE INDEX idx_invoice_tenant ON invoice (tenant_id, created_at);
CREATE INDEX idx_invoice_client ON invoice (client_id);
CREATE INDEX idx_invoice_status ON invoice (tenant_id, status);
```

## Memberships, Packages & Gift Cards

```sql
CREATE TABLE membership_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    price           NUMERIC(10, 2) NOT NULL,
    billing_interval TEXT NOT NULL DEFAULT 'monthly',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "included_services": [
    --     {"service_id": "svc-uuid", "quantity": 2, "name": "Swedish Massage"}
    --   ],
    --   "discount_pct": 15,
    --   "freeze_allowed": true,
    --   "max_freeze_days": 30,
    --   "cancellation_notice_days": 30,
    --   "guest_passes_per_month": 1,
    --   "retail_discount_pct": 10
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE client_membership (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    plan_id         UUID NOT NULL REFERENCES membership_plan(id),
    status          TEXT NOT NULL DEFAULT 'active',
    start_date      DATE NOT NULL,
    next_billing_date DATE,
    stripe_subscription_id TEXT,
    state           JSONB NOT NULL DEFAULT '{}',
    -- state example:
    -- {
    --   "end_date": null,
    --   "freeze_start": null,
    --   "freeze_end": null,
    --   "cancellation_reason": null,
    --   "remaining_services": [
    --     {"service_id": "svc-uuid", "remaining": 1, "resets_at": "2026-07-01"}
    --   ],
    --   "usage_history": [
    --     {"service_id": "svc-uuid", "used_at": "2026-06-01", "appointment_id": "appt-uuid"}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_client_membership ON client_membership (client_id, status);

-- Packages and gift cards as a single flexible table
CREATE TABLE voucher (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    voucher_type    TEXT NOT NULL,
        -- package, gift_card
    code            TEXT UNIQUE,
    client_id       UUID REFERENCES client(id),
    status          TEXT NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL,
    -- Package config example:
    -- {
    --   "name": "5-Session Facial Package",
    --   "price_paid": 400.00,
    --   "services": [
    --     {"service_id": "svc-uuid", "total": 5, "remaining": 3}
    --   ],
    --   "valid_until": "2027-01-15",
    --   "usage": [
    --     {"service_id": "svc-uuid", "used_at": "2026-06-01", "appointment_id": "..."}
    --   ]
    -- }
    -- Gift card config example:
    -- {
    --   "original_amount": 100.00,
    --   "balance": 65.00,
    --   "currency": "USD",
    --   "purchaser_name": "John Doe",
    --   "recipient_name": "Jane Doe",
    --   "recipient_email": "jane@example.com",
    --   "transactions": [
    --     {"type": "purchase", "amount": 100.00, "at": "2026-05-01"},
    --     {"type": "redemption", "amount": -35.00, "at": "2026-06-01",
    --      "invoice_id": "inv-uuid"}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_voucher_tenant ON voucher (tenant_id, voucher_type);
CREATE INDEX idx_voucher_client ON voucher (client_id);
CREATE INDEX idx_voucher_code ON voucher (code);
```

## Products & Inventory

```sql
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    sku             TEXT,
    barcode         TEXT,
    retail_price    NUMERIC(10, 2) NOT NULL,
    cost_price      NUMERIC(10, 2),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "brand": "Aveda",
    --   "category": ["Hair Care", "Shampoo"],
    --   "description": "Damage Remedy Restructuring Shampoo 250ml",
    --   "is_backbar": false,
    --   "is_retail": true,
    --   "tax_rate_pct": 8.25,
    --   "sds_url": "https://...",
    --   "images": ["https://..."],
    --   "weight_oz": 8.5,
    --   "inventory": {
    --     "loc-uuid-1": {"on_hand": 12, "reorder_point": 5, "reorder_qty": 24},
    --     "loc-uuid-2": {"on_hand": 8, "reorder_point": 3, "reorder_qty": 12}
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_product_tenant ON product (tenant_id, is_active);
CREATE INDEX idx_product_sku ON product (tenant_id, sku);
CREATE INDEX idx_product_barcode ON product (barcode);
CREATE INDEX idx_product_details ON product USING GIN (details jsonb_path_ops);

-- Inventory movements remain relational for accurate stock tracking
CREATE TABLE inventory_movement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    movement_type   TEXT NOT NULL,
        -- purchase, sale, adjustment, transfer_in, transfer_out,
        -- backbar_usage, waste, return
    quantity        NUMERIC(10, 2) NOT NULL,
    reference_id    UUID,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inv_mvmt ON inventory_movement (product_id, location_id, created_at);
```

## Marketing & Notifications

```sql
CREATE TABLE campaign (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    campaign_type   TEXT NOT NULL,  -- email, sms, push
    status          TEXT NOT NULL DEFAULT 'draft',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "subject": "Spring Special: 20% off massages",
    --   "body_html": "...",
    --   "body_text": "...",
    --   "audience": {
    --     "filter": {"tags": ["regular"], "last_visit_within_days": 60},
    --     "exclude": {"tags": ["do_not_contact"]}
    --   },
    --   "scheduled_at": "2026-06-01T09:00:00Z",
    --   "metrics": {
    --     "sent": 245,
    --     "opened": 89,
    --     "clicked": 23,
    --     "unsubscribed": 2
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaign_tenant ON campaign (tenant_id);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    actor_id        UUID,
    actor_type      TEXT NOT NULL,  -- staff, client, system
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    -- changes example:
    -- {"path": "status", "old": "booked", "new": "cancelled"}
    -- or for JSONB changes:
    -- {"path": "profile.preferences.pressure", "old": "medium", "new": "firm"}
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant ON audit_log (tenant_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
```

## Form Templates

```sql
CREATE TABLE form_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    form_type       TEXT NOT NULL,
        -- intake, consent, medical_history, feedback, waiver
    is_active       BOOLEAN NOT NULL DEFAULT true,
    definition      JSONB NOT NULL,
    -- definition example:
    -- {
    --   "version": 2,
    --   "fields": [
    --     {"name": "allergies", "type": "text", "label": "Known allergies",
    --      "required": true},
    --     {"name": "medications", "type": "text_list",
    --      "label": "Current medications"},
    --     {"name": "pregnancy", "type": "boolean",
    --      "label": "Are you currently pregnant?"},
    --     {"name": "signature", "type": "signature", "required": true}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE form_response (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    form_template_id UUID NOT NULL REFERENCES form_template(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    responses       JSONB NOT NULL,
    signed_at       TIMESTAMPTZ,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_form_response ON form_response (client_id, form_template_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Location | 2 | tenant, location (rooms are JSONB inside location) |
| Staff | 2 | staff, staff_schedule_block |
| Service Catalogue | 1 | service (variations, staff pricing in JSONB) |
| Clients | 1 | client (notes, preferences, medspa profile all in JSONB) |
| Appointments | 3 | appointment, appointment_staff_lookup, appointment_room_lookup |
| Invoices | 1 | invoice (items and payments in JSONB) |
| Memberships & Vouchers | 3 | membership_plan, client_membership, voucher |
| Products & Inventory | 2 | product, inventory_movement |
| Marketing | 1 | campaign |
| Forms | 2 | form_template, form_response |
| Audit | 1 | audit_log |
| **Total** | **~19** | Plus 2 lookup tables for appointment scheduling queries |

---

## Key Design Decisions

1. **JSONB for variable fields, columns for query-critical fields** -- `status`, `start_at`, `end_at`, `client_id`, `tenant_id` are always relational columns because they drive filtering, sorting, and foreign key integrity. Everything else that varies by business type or jurisdiction goes into JSONB.

2. **Treatment rooms as JSONB inside location** -- instead of a separate `treatment_room` table, rooms are stored as a JSONB array in `location.details.rooms`. This works because rooms are always queried in the context of a location and rarely need cross-location JOINs. It eliminates a table and simplifies the location setup UI.

3. **Appointment segments as JSONB** -- the appointment's service segments are stored as a JSONB array rather than a separate table. This keeps the appointment as a single document for API responses and reduces JOINs. The trade-off is that staff/room availability queries require the separate `appointment_staff_lookup` and `appointment_room_lookup` tables, which are maintained by application-layer triggers.

4. **Unified voucher table** -- packages and gift cards share a single `voucher` table with a `voucher_type` discriminator. Their structures differ significantly, but JSONB config handles the variation without separate tables.

5. **Invoice as a self-contained document** -- items and payments are JSONB arrays inside the invoice, making the invoice a complete document that can be rendered without JOINs. This mirrors how invoices work in practice: once closed, they are immutable snapshots.

6. **Client stats as a JSONB column** -- computed aggregates (total visits, total spend, churn risk score) are stored as a denormalised JSONB blob on the client record, updated by background jobs or triggers. This avoids expensive aggregate queries on every client profile view.

7. **Medspa profile as optional JSONB** -- the `medspa_profile` column on client is NULL for non-medspa tenants. This avoids the overhead of separate PHI tables for tenants that do not need them, while still isolating the data clearly for row-level security when HIPAA compliance is enabled.

8. **GIN indexes on all JSONB columns** -- `jsonb_path_ops` GIN indexes enable fast containment queries (`@>`) on JSONB fields. This supports queries like "find all clients tagged VIP with an allergy alert" without full table scans.

9. **Inventory movements remain relational** -- while product details use JSONB, inventory movements stay as a relational append-only table because accurate stock tracking requires transactional integrity and the movement log is the source of truth for quantity_on_hand calculations.

10. **Progressive column promotion** -- the design explicitly supports promoting frequently-queried JSONB fields to dedicated columns as the product matures. For example, if `client.profile.tags` becomes critical for segmentation, it can be promoted to a `tags TEXT[]` column with a migration, while the JSONB field remains for backward compatibility.
