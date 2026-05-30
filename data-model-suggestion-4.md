# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Spa & Salon Management · Created: 2026-05-22

## Philosophy

This model uses a dual-layer architecture: a relational layer for core CRUD operations (appointments, invoices, inventory) and a property graph layer for relationship-heavy queries that are awkward or expensive in pure relational systems. The graph layer models relationships between clients, staff, services, products, and locations as first-class edges with properties, enabling queries like "find the best-matching therapist for this client based on past service satisfaction, certification overlap, and availability" or "recommend products that clients with similar treatment histories purchased."

The graph layer is implemented using PostgreSQL's `ltree` extension for hierarchies and a general-purpose `graph_node` / `graph_edge` pattern for arbitrary relationships. This avoids the operational complexity of a separate graph database (Neo4j, Amazon Neptune) while providing graph traversal capabilities that would require complex recursive CTEs in a purely relational model. For deployments that need deeper graph analytics, the same edge data can be exported to a dedicated graph database.

This approach is inspired by social network data models (Facebook's TAO, LinkedIn's graph platform) and recommendation engines (Netflix, Spotify) where the relationship between entities is as important as the entities themselves. In the spa/salon context, the graph captures: client-staff affinity (which therapist does each client prefer, and how strongly?), service co-purchase patterns (clients who book facials often add eye treatments), product-service associations (which products are used in which treatments?), and referral networks (who referred whom?).

**Best for:** Teams building an AI-native platform with personalised recommendations, smart therapist matching, client affinity scoring, and network-effect features like referral tracking -- especially if the roadmap includes a consumer marketplace.

**Trade-offs:**
- Pro: Relationship-heavy queries (recommendations, matching, affinity) are natural and performant
- Pro: Referral networks, client clusters, and therapist specialisation patterns are first-class data
- Pro: AI recommendation engines can traverse the graph directly for feature extraction
- Pro: Flexible: new relationship types can be added without schema changes
- Pro: Service hierarchy and category navigation use ltree for efficient subtree queries
- Con: Two mental models (relational + graph) increase cognitive load
- Con: Graph edge tables can grow large (N clients x M services x T staff = many edges)
- Con: Maintaining graph consistency alongside relational data requires careful application logic
- Con: No ACID transactions across graph and relational layers if using an external graph database
- Con: Fewer off-the-shelf tools for graph analytics compared to pure relational

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5545 (iCalendar) | Appointment fields align with VEVENT properties for calendar export |
| ISO 3166-1/2 | Location jurisdiction modelled as ltree paths (e.g., `us.ca.la`) |
| ISO 4217 | Currency codes on monetary fields |
| PCI DSS | Payment tokens only; graph edges never contain card data |
| HIPAA | PHI stored in relational layer with RLS; graph layer contains only anonymised affinity scores |
| GDPR | Client graph edges can be bulk-deleted on erasure request |
| Property Graph Model | Edges follow the labelled property graph pattern (source, target, label, properties) |

---

## Graph Layer

```sql
-- ============================================================
-- PROPERTY GRAPH INFRASTRUCTURE
-- ============================================================
-- Generic graph layer using labelled property graph pattern.
-- Nodes map to relational entities; edges capture relationships.

CREATE EXTENSION IF NOT EXISTS ltree;

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY,  -- same as the relational entity PK
    tenant_id       UUID NOT NULL,
    node_type       TEXT NOT NULL,
        -- client, staff, service, product, location, category
    label           TEXT NOT NULL,     -- display name
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Properties vary by node_type:
    -- client:   {"vip": true, "loyalty_tier": "gold", "segment": "regular"}
    -- staff:    {"role": "therapist", "specialties": ["massage", "facial"]}
    -- service:  {"category": "massage", "duration": 60, "price": 120}
    -- product:  {"brand": "Aveda", "category": "hair_care"}
    -- location: {"city": "Los Angeles", "country": "US"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gn_tenant ON graph_node (tenant_id, node_type);
CREATE INDEX idx_gn_props ON graph_node USING GIN (properties jsonb_path_ops);

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_id       UUID NOT NULL REFERENCES graph_node(id),
    target_id       UUID NOT NULL REFERENCES graph_node(id),
    edge_type       TEXT NOT NULL,
    weight          NUMERIC(8, 4) NOT NULL DEFAULT 1.0,
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ge_source ON graph_edge (source_id, edge_type);
CREATE INDEX idx_ge_target ON graph_edge (target_id, edge_type);
CREATE INDEX idx_ge_type ON graph_edge (tenant_id, edge_type);
CREATE INDEX idx_ge_weight ON graph_edge (edge_type, weight DESC);
CREATE INDEX idx_ge_props ON graph_edge USING GIN (properties jsonb_path_ops);

-- Prevent duplicate edges of the same type between the same nodes
CREATE UNIQUE INDEX idx_ge_unique
    ON graph_edge (source_id, target_id, edge_type);
```

## Edge Type Catalogue

```sql
-- ============================================================
-- EDGE TYPES
-- ============================================================
-- Documented relationship types in the graph.

-- CLIENT -> STAFF edges:
--   'prefers'        — client prefers this staff member
--                      weight = affinity score (0-1 based on visit frequency, ratings)
--                      properties: {"visit_count": 12, "avg_rating": 4.8, "last_visit": "2026-05-01"}
--
--   'referred_by'    — client was referred by this staff member
--                      properties: {"referral_date": "2026-01-15"}

-- CLIENT -> SERVICE edges:
--   'has_received'   — client has received this service
--                      weight = frequency score
--                      properties: {"count": 8, "last_at": "2026-05-01", "avg_rating": 4.5}
--
--   'interested_in'  — client has expressed interest (viewed, added to cart, enquired)
--                      properties: {"source": "online_browse", "at": "2026-05-10"}

-- CLIENT -> PRODUCT edges:
--   'has_purchased'  — client has purchased this product
--                      weight = purchase frequency
--                      properties: {"count": 3, "total_spent": 105.00, "last_at": "2026-04-15"}

-- CLIENT -> CLIENT edges:
--   'referred'       — one client referred another
--                      properties: {"referral_date": "2026-02-01", "referral_code": "JANE20"}

-- STAFF -> SERVICE edges:
--   'certified_for'  — staff is certified to perform this service
--                      properties: {"cert_number": "LMT-12345", "expires": "2027-12-31"}
--
--   'specialises_in' — staff specialises in this service (higher skill/preference)
--                      weight = expertise level (0-1)

-- SERVICE -> SERVICE edges:
--   'commonly_paired' — services frequently booked together
--                       weight = co-occurrence score
--                       properties: {"pair_count": 45, "pair_pct": 0.32}
--
--   'upgrade_from'   — this service is a premium upgrade from target
--                      properties: {"price_diff": 50.00}
--
--   'addon_for'      — this service is commonly added to target
--                      weight = addon frequency

-- PRODUCT -> SERVICE edges:
--   'used_in'        — product is used during this service
--                      properties: {"quantity_per_use": 0.5, "unit": "oz"}
--
--   'recommended_after' — product is recommended for purchase after service
--                         weight = recommendation strength

-- SERVICE -> CATEGORY (ltree) edges:
--   Handled by the ltree path column on the service, not graph_edge table.
```

## Service Category Hierarchy (ltree)

```sql
-- ============================================================
-- SERVICE CATEGORY HIERARCHY
-- ============================================================
-- Uses PostgreSQL ltree for efficient hierarchical queries.

CREATE TABLE service_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    path            ltree NOT NULL,
    -- Examples:
    --   'body'
    --   'body.massage'
    --   'body.massage.deep_tissue'
    --   'body.massage.swedish'
    --   'body.massage.hot_stone'
    --   'body.scrub'
    --   'face'
    --   'face.facial'
    --   'face.facial.anti_aging'
    --   'face.facial.hydrating'
    --   'hair'
    --   'hair.cut'
    --   'hair.color'
    --   'hair.color.highlights'
    --   'nails'
    --   'nails.manicure'
    --   'nails.pedicure'
    --   'medspa'
    --   'medspa.injectables'
    --   'medspa.laser'
    name            TEXT NOT NULL,
    description     TEXT,
    sort_order      INT NOT NULL DEFAULT 0,
    UNIQUE (tenant_id, path)
);

CREATE INDEX idx_svc_cat_path ON service_category USING GIST (path);
CREATE INDEX idx_svc_cat_tenant ON service_category (tenant_id);

-- Example queries:
--
-- All subcategories of 'body':
-- SELECT * FROM service_category WHERE path <@ 'body' AND tenant_id = $1;
--
-- All ancestors of 'body.massage.deep_tissue':
-- SELECT * FROM service_category WHERE 'body.massage.deep_tissue' <@ path;
--
-- Depth of a category:
-- SELECT nlevel(path) FROM service_category WHERE id = $1;
```

## Relational Layer: Core Entities

```sql
-- ============================================================
-- TENANT & LOCATION
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    country_code    CHAR(2) NOT NULL DEFAULT 'US',
    jurisdiction    ltree,
    -- Examples: 'us.ca.los_angeles', 'gb.england.london', 'au.nsw.sydney'
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    address         JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE INDEX idx_location_tenant ON location (tenant_id);
CREATE INDEX idx_location_jurisdiction ON location USING GIST (jurisdiction);

-- ============================================================
-- TREATMENT ROOMS
-- ============================================================

CREATE TABLE treatment_room (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    name            TEXT NOT NULL,
    room_type       TEXT NOT NULL DEFAULT 'standard',
    capacity        INT NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_room_location ON treatment_room (location_id);

-- ============================================================
-- STAFF
-- ============================================================

CREATE TABLE staff (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,
    role            TEXT NOT NULL DEFAULT 'therapist',
    is_bookable     BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    profile         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_staff_tenant ON staff (tenant_id, is_active);

-- ============================================================
-- CLIENT
-- ============================================================

CREATE TABLE client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,
    profile         JSONB NOT NULL DEFAULT '{}',
    consents        JSONB NOT NULL DEFAULT '{}',
    medspa_profile  JSONB,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_client_tenant ON client (tenant_id);
CREATE INDEX idx_client_email ON client (tenant_id, email);
CREATE INDEX idx_client_phone ON client (tenant_id, phone);

-- ============================================================
-- SERVICE
-- ============================================================

CREATE TABLE service (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    category_path   ltree,  -- references service_category.path
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    duration_minutes INT NOT NULL,
    price           NUMERIC(10, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    is_online_bookable BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE INDEX idx_service_tenant ON service (tenant_id, is_active);
CREATE INDEX idx_service_category ON service USING GIST (category_path);
```

## Relational Layer: Appointments

```sql
CREATE TABLE appointment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    status          TEXT NOT NULL DEFAULT 'booked',
    booking_channel TEXT NOT NULL DEFAULT 'online',
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ NOT NULL,
    ical_uid        TEXT UNIQUE,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appt_tenant ON appointment (tenant_id, status);
CREATE INDEX idx_appt_client ON appointment (client_id);
CREATE INDEX idx_appt_location ON appointment (location_id, start_at);
CREATE INDEX idx_appt_time ON appointment (start_at, end_at);

CREATE TABLE appointment_segment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    appointment_id  UUID NOT NULL REFERENCES appointment(id),
    service_id      UUID NOT NULL REFERENCES service(id),
    staff_id        UUID NOT NULL REFERENCES staff(id),
    room_id         UUID REFERENCES treatment_room(id),
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ NOT NULL,
    duration_minutes INT NOT NULL,
    price           NUMERIC(10, 2) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'scheduled',
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_seg_appt ON appointment_segment (appointment_id);
CREATE INDEX idx_seg_staff ON appointment_segment (staff_id, start_at, end_at);
CREATE INDEX idx_seg_room ON appointment_segment (room_id, start_at, end_at);
```

## Relational Layer: Invoicing

```sql
CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    client_id       UUID REFERENCES client(id),
    appointment_id  UUID REFERENCES appointment(id),
    invoice_number  TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    subtotal        NUMERIC(10, 2) NOT NULL DEFAULT 0,
    tax_total       NUMERIC(10, 2) NOT NULL DEFAULT 0,
    discount_total  NUMERIC(10, 2) NOT NULL DEFAULT 0,
    tip_total       NUMERIC(10, 2) NOT NULL DEFAULT 0,
    grand_total     NUMERIC(10, 2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, invoice_number)
);

CREATE INDEX idx_invoice_tenant ON invoice (tenant_id, created_at);
CREATE INDEX idx_invoice_client ON invoice (client_id);

CREATE TABLE invoice_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    item_type       TEXT NOT NULL,
    reference_id    UUID,
    description     TEXT NOT NULL,
    quantity        NUMERIC(10, 3) NOT NULL DEFAULT 1,
    unit_price      NUMERIC(10, 2) NOT NULL,
    discount_amount NUMERIC(10, 2) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(10, 2) NOT NULL DEFAULT 0,
    total           NUMERIC(10, 2) NOT NULL,
    staff_id        UUID REFERENCES staff(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inv_item ON invoice_item (invoice_id);

CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    payment_method  TEXT NOT NULL,
    amount          NUMERIC(10, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    external_id     TEXT,  -- Stripe PI, Square payment ID (PCI DSS compliant)
    status          TEXT NOT NULL DEFAULT 'completed',
    processed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_invoice ON payment (invoice_id);
```

## Relational Layer: Products & Memberships

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_product_tenant ON product (tenant_id);
CREATE INDEX idx_product_sku ON product (tenant_id, sku);

CREATE TABLE product_inventory (
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    quantity_on_hand NUMERIC(10, 2) NOT NULL DEFAULT 0,
    reorder_point   NUMERIC(10, 2),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (product_id, location_id)
);

CREATE TABLE membership_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    price           NUMERIC(10, 2) NOT NULL,
    billing_interval TEXT NOT NULL DEFAULT 'monthly',
    config          JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE client_membership (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id),
    plan_id         UUID NOT NULL REFERENCES membership_plan(id),
    status          TEXT NOT NULL DEFAULT 'active',
    start_date      DATE NOT NULL,
    next_billing_date DATE,
    state           JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_membership_client ON client_membership (client_id, status);
```

## Graph-Powered Features

### Therapist-Client Matching

```sql
-- ============================================================
-- SMART THERAPIST MATCHING
-- ============================================================
-- Uses graph edges to find the best therapist for a client
-- based on affinity, certification, and availability.

CREATE OR REPLACE FUNCTION find_best_therapist(
    p_client_id UUID,
    p_service_id UUID,
    p_location_id UUID,
    p_start_at TIMESTAMPTZ,
    p_end_at TIMESTAMPTZ
) RETURNS TABLE (
    staff_id UUID,
    staff_name TEXT,
    affinity_score NUMERIC,
    expertise_score NUMERIC,
    combined_score NUMERIC
) AS $$
BEGIN
    RETURN QUERY
    WITH
    -- Staff certified for the requested service
    certified_staff AS (
        SELECT ge.source_id AS sid, ge.weight AS expertise
        FROM graph_edge ge
        WHERE ge.target_id = p_service_id
          AND ge.edge_type IN ('certified_for', 'specialises_in')
    ),
    -- Client's affinity for each staff member
    client_affinity AS (
        SELECT ge.target_id AS sid, ge.weight AS affinity
        FROM graph_edge ge
        WHERE ge.source_id = p_client_id
          AND ge.edge_type = 'prefers'
    ),
    -- Staff working at the requested location
    location_staff AS (
        SELECT ge.source_id AS sid
        FROM graph_edge ge
        WHERE ge.target_id = p_location_id
          AND ge.edge_type = 'works_at'
    ),
    -- Staff who are available (no conflicting appointment segments)
    available_staff AS (
        SELECT s.id AS sid
        FROM staff s
        WHERE s.is_bookable AND s.is_active
          AND s.id NOT IN (
              SELECT aseg.staff_id
              FROM appointment_segment aseg
              JOIN appointment a ON a.id = aseg.appointment_id
              WHERE a.status NOT IN ('cancelled', 'no_show')
                AND aseg.start_at < p_end_at
                AND aseg.end_at > p_start_at
          )
    )
    SELECT
        s.id,
        s.first_name || ' ' || s.last_name,
        COALESCE(ca.affinity, 0.0),
        COALESCE(cs.expertise, 0.5),
        -- Combined score: 60% affinity + 40% expertise
        (COALESCE(ca.affinity, 0.0) * 0.6 + COALESCE(cs.expertise, 0.5) * 0.4)
    FROM certified_staff cs
    JOIN location_staff ls ON ls.sid = cs.sid
    JOIN available_staff avs ON avs.sid = cs.sid
    JOIN staff s ON s.id = cs.sid
    LEFT JOIN client_affinity ca ON ca.sid = cs.sid
    ORDER BY (COALESCE(ca.affinity, 0.0) * 0.6 + COALESCE(cs.expertise, 0.5) * 0.4) DESC;
END;
$$ LANGUAGE plpgsql;
```

### Service Recommendations

```sql
-- ============================================================
-- SERVICE RECOMMENDATIONS
-- ============================================================
-- "Clients who booked [service X] also booked..."
-- Uses collaborative filtering via graph edges.

CREATE OR REPLACE FUNCTION recommend_services(
    p_client_id UUID,
    p_tenant_id UUID,
    p_limit INT DEFAULT 5
) RETURNS TABLE (
    service_id UUID,
    service_name TEXT,
    recommendation_score NUMERIC,
    reason TEXT
) AS $$
BEGIN
    RETURN QUERY
    WITH
    -- Services this client has already received
    client_services AS (
        SELECT ge.target_id AS service_id
        FROM graph_edge ge
        WHERE ge.source_id = p_client_id
          AND ge.edge_type = 'has_received'
    ),
    -- Services commonly paired with client's past services
    paired_services AS (
        SELECT
            ge.target_id AS service_id,
            ge.weight AS pair_score,
            'commonly paired with ' || s_source.name AS rec_reason
        FROM client_services cs
        JOIN graph_edge ge ON ge.source_id = cs.service_id
            AND ge.edge_type = 'commonly_paired'
        JOIN service s_source ON s_source.id = cs.service_id
        WHERE ge.target_id NOT IN (SELECT service_id FROM client_services)
    ),
    -- Services similar clients have received (collaborative filtering)
    -- Find clients who share service history, then find their other services
    similar_client_services AS (
        SELECT
            ge2.target_id AS service_id,
            AVG(ge2.weight) AS collab_score,
            'popular with similar clients' AS rec_reason
        FROM client_services cs
        -- Other clients who received the same service
        JOIN graph_edge ge1 ON ge1.target_id = cs.service_id
            AND ge1.edge_type = 'has_received'
            AND ge1.source_id != p_client_id
        -- Other services those clients received
        JOIN graph_edge ge2 ON ge2.source_id = ge1.source_id
            AND ge2.edge_type = 'has_received'
            AND ge2.target_id NOT IN (SELECT service_id FROM client_services)
        GROUP BY ge2.target_id
        HAVING COUNT(DISTINCT ge1.source_id) >= 3  -- minimum support
    )
    SELECT
        s.id,
        s.name,
        COALESCE(ps.pair_score, 0) * 0.5 + COALESCE(scs.collab_score, 0) * 0.5,
        COALESCE(ps.rec_reason, scs.rec_reason)
    FROM service s
    LEFT JOIN paired_services ps ON ps.service_id = s.id
    LEFT JOIN similar_client_services scs ON scs.service_id = s.id
    WHERE s.tenant_id = p_tenant_id
      AND s.is_active
      AND (ps.service_id IS NOT NULL OR scs.service_id IS NOT NULL)
    ORDER BY COALESCE(ps.pair_score, 0) * 0.5 + COALESCE(scs.collab_score, 0) * 0.5 DESC
    LIMIT p_limit;
END;
$$ LANGUAGE plpgsql;
```

### Product Recommendations

```sql
-- ============================================================
-- RETAIL PRODUCT RECOMMENDATIONS
-- ============================================================
-- Recommend products based on the service just completed.

CREATE OR REPLACE FUNCTION recommend_products_after_service(
    p_client_id UUID,
    p_service_id UUID,
    p_limit INT DEFAULT 3
) RETURNS TABLE (
    product_id UUID,
    product_name TEXT,
    retail_price NUMERIC,
    recommendation_reason TEXT,
    score NUMERIC
) AS $$
BEGIN
    RETURN QUERY
    WITH
    -- Products the client has already purchased (avoid re-recommending)
    owned_products AS (
        SELECT ge.target_id AS product_id
        FROM graph_edge ge
        WHERE ge.source_id = p_client_id
          AND ge.edge_type = 'has_purchased'
    ),
    -- Products recommended after this service
    service_recs AS (
        SELECT
            ge.source_id AS prod_id,
            ge.weight AS rec_score,
            'Recommended after ' || s.name
        FROM graph_edge ge
        JOIN service s ON s.id = ge.target_id
        WHERE ge.target_id = p_service_id
          AND ge.edge_type = 'recommended_after'
          AND ge.source_id NOT IN (SELECT product_id FROM owned_products)
    ),
    -- Products used during this service (client may want to buy)
    used_products AS (
        SELECT
            ge.source_id AS prod_id,
            ge.weight * 0.8 AS rec_score,
            'Used during your ' || s.name || ' today'
        FROM graph_edge ge
        JOIN service s ON s.id = ge.target_id
        WHERE ge.target_id = p_service_id
          AND ge.edge_type = 'used_in'
          AND ge.source_id NOT IN (SELECT product_id FROM owned_products)
    )
    SELECT
        p.id, p.name, p.retail_price,
        COALESCE(sr.rec_reason, up.rec_reason),
        GREATEST(COALESCE(sr.rec_score, 0), COALESCE(up.rec_score, 0))
    FROM (
        SELECT prod_id, rec_score, rec_reason FROM service_recs
        UNION ALL
        SELECT prod_id, rec_score, rec_reason FROM used_products
    ) combined(prod_id, rec_score, rec_reason)
    JOIN product p ON p.id = combined.prod_id
    LEFT JOIN service_recs sr ON sr.prod_id = p.id
    LEFT JOIN used_products up ON up.prod_id = p.id
    WHERE p.is_active
    ORDER BY GREATEST(COALESCE(sr.rec_score, 0), COALESCE(up.rec_score, 0)) DESC
    LIMIT p_limit;
END;
$$ LANGUAGE plpgsql;
```

### Referral Network Analysis

```sql
-- ============================================================
-- REFERRAL NETWORK
-- ============================================================
-- Track and analyse client referral chains.

-- Find all clients referred (directly or transitively) by a given client
CREATE OR REPLACE FUNCTION get_referral_tree(
    p_client_id UUID,
    p_max_depth INT DEFAULT 5
) RETURNS TABLE (
    client_id UUID,
    client_name TEXT,
    referred_by UUID,
    depth INT,
    referral_date DATE
) AS $$
BEGIN
    RETURN QUERY
    WITH RECURSIVE referral_chain AS (
        -- Direct referrals
        SELECT
            ge.target_id AS cid,
            c.first_name || ' ' || c.last_name AS cname,
            ge.source_id AS ref_by,
            1 AS d,
            (ge.properties ->> 'referral_date')::DATE AS ref_date
        FROM graph_edge ge
        JOIN client c ON c.id = ge.target_id
        WHERE ge.source_id = p_client_id
          AND ge.edge_type = 'referred'

        UNION ALL

        -- Transitive referrals
        SELECT
            ge.target_id,
            c.first_name || ' ' || c.last_name,
            ge.source_id,
            rc.d + 1,
            (ge.properties ->> 'referral_date')::DATE
        FROM referral_chain rc
        JOIN graph_edge ge ON ge.source_id = rc.cid
            AND ge.edge_type = 'referred'
        JOIN client c ON c.id = ge.target_id
        WHERE rc.d < p_max_depth
    )
    SELECT * FROM referral_chain
    ORDER BY d, ref_date;
END;
$$ LANGUAGE plpgsql;
```

## Graph Maintenance

```sql
-- ============================================================
-- GRAPH EDGE UPDATE FUNCTIONS
-- ============================================================
-- Called by application layer after appointments complete
-- and invoices close.

-- Update client-staff affinity after appointment completion
CREATE OR REPLACE FUNCTION update_client_staff_affinity(
    p_tenant_id UUID,
    p_client_id UUID,
    p_staff_id UUID,
    p_rating SMALLINT DEFAULT NULL
) RETURNS VOID AS $$
DECLARE
    v_visit_count INT;
    v_avg_rating NUMERIC;
    v_affinity NUMERIC;
BEGIN
    -- Count visits between this client and staff
    SELECT COUNT(*), AVG(
        CASE WHEN ge.properties ? 'rating'
             THEN (ge.properties ->> 'rating')::NUMERIC
             ELSE NULL END
    )
    INTO v_visit_count, v_avg_rating
    FROM appointment_segment aseg
    JOIN appointment a ON a.id = aseg.appointment_id
    WHERE a.client_id = p_client_id
      AND aseg.staff_id = p_staff_id
      AND a.status = 'completed';

    -- Compute affinity: log-scaled visit count * rating factor
    v_affinity := LEAST(1.0,
        ln(v_visit_count + 1) / 4.0 * COALESCE(v_avg_rating / 5.0, 0.8)
    );

    -- Upsert the 'prefers' edge
    INSERT INTO graph_edge (tenant_id, source_id, target_id, edge_type, weight, properties)
    VALUES (p_tenant_id, p_client_id, p_staff_id, 'prefers', v_affinity,
            jsonb_build_object(
                'visit_count', v_visit_count,
                'avg_rating', v_avg_rating,
                'last_visit', now()::DATE
            ))
    ON CONFLICT (source_id, target_id, edge_type)
    DO UPDATE SET
        weight = EXCLUDED.weight,
        properties = EXCLUDED.properties,
        updated_at = now();
END;
$$ LANGUAGE plpgsql;

-- Update service co-purchase patterns
CREATE OR REPLACE FUNCTION update_service_pairing(
    p_tenant_id UUID,
    p_service_id_a UUID,
    p_service_id_b UUID
) RETURNS VOID AS $$
DECLARE
    v_pair_count INT;
    v_total_a INT;
    v_pair_pct NUMERIC;
BEGIN
    -- Count how many appointments include both services
    SELECT COUNT(DISTINCT a.id) INTO v_pair_count
    FROM appointment a
    JOIN appointment_segment s1 ON s1.appointment_id = a.id AND s1.service_id = p_service_id_a
    JOIN appointment_segment s2 ON s2.appointment_id = a.id AND s2.service_id = p_service_id_b
    WHERE a.tenant_id = p_tenant_id AND a.status = 'completed';

    SELECT COUNT(DISTINCT a.id) INTO v_total_a
    FROM appointment a
    JOIN appointment_segment s ON s.appointment_id = a.id AND s.service_id = p_service_id_a
    WHERE a.tenant_id = p_tenant_id AND a.status = 'completed';

    v_pair_pct := CASE WHEN v_total_a > 0
                       THEN v_pair_count::NUMERIC / v_total_a
                       ELSE 0 END;

    -- Upsert bidirectional 'commonly_paired' edges
    INSERT INTO graph_edge (tenant_id, source_id, target_id, edge_type, weight, properties)
    VALUES (p_tenant_id, p_service_id_a, p_service_id_b, 'commonly_paired', v_pair_pct,
            jsonb_build_object('pair_count', v_pair_count, 'pair_pct', v_pair_pct))
    ON CONFLICT (source_id, target_id, edge_type)
    DO UPDATE SET weight = EXCLUDED.weight, properties = EXCLUDED.properties, updated_at = now();

    -- Reverse direction
    INSERT INTO graph_edge (tenant_id, source_id, target_id, edge_type, weight, properties)
    VALUES (p_tenant_id, p_service_id_b, p_service_id_a, 'commonly_paired', v_pair_pct,
            jsonb_build_object('pair_count', v_pair_count, 'pair_pct', v_pair_pct))
    ON CONFLICT (source_id, target_id, edge_type)
    DO UPDATE SET weight = EXCLUDED.weight, properties = EXCLUDED.properties, updated_at = now();
END;
$$ LANGUAGE plpgsql;
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    actor_id        UUID,
    actor_type      TEXT NOT NULL,
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant ON audit_log (tenant_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Infrastructure | 2 | graph_node, graph_edge |
| Service Hierarchy | 1 | service_category (ltree) |
| Tenant & Location | 3 | tenant, location (ltree jurisdiction), treatment_room |
| Staff | 1 | staff |
| Client | 1 | client |
| Service | 1 | service (ltree category_path) |
| Appointments | 2 | appointment, appointment_segment |
| Invoicing | 3 | invoice, invoice_item, payment |
| Products & Inventory | 2 | product, product_inventory |
| Memberships | 2 | membership_plan, client_membership |
| Audit | 1 | audit_log |
| **Total** | **~19** | Plus graph_node/graph_edge which encode all relationships |

---

## Key Design Decisions

1. **Separate graph layer, not graph database** -- the `graph_node` / `graph_edge` tables live in the same PostgreSQL instance as the relational tables, avoiding the operational complexity of synchronising a separate graph database. For teams that outgrow PostgreSQL's graph query performance, the edge data can be exported to Neo4j or Amazon Neptune.

2. **Edges are the AI feature store** -- graph edges with weights and properties (affinity scores, visit counts, co-purchase frequencies) serve as a pre-computed feature store for AI models. The recommendation functions demonstrate how the graph enables collaborative filtering, content-based recommendations, and hybrid approaches without a separate ML pipeline.

3. **ltree for service categories and jurisdictions** -- PostgreSQL's `ltree` extension provides efficient hierarchical queries (subtree, ancestor, depth) without recursive CTEs. Service categories use paths like `body.massage.deep_tissue` and location jurisdictions use `us.ca.los_angeles`, enabling queries like "all massage services" or "all California locations" with simple path operators.

4. **Graph edges maintained by application triggers** -- affinity scores, co-purchase patterns, and referral links are updated by stored functions called after appointment completion and invoice closure. This keeps the graph fresh without requiring a separate batch processing pipeline.

5. **Unique constraint on (source, target, edge_type)** -- each pair of nodes can have at most one edge of each type. This simplifies upsert logic and prevents edge accumulation. Historical edge changes are tracked in the audit log if needed.

6. **Relational layer for ACID-critical operations** -- appointments, invoices, and payments use standard relational tables with foreign keys and constraints. The graph layer is eventually consistent and optimised for read-heavy recommendation queries, not transactional writes.

7. **Graph-powered therapist matching** -- the `find_best_therapist` function combines graph traversal (client affinity, staff expertise) with relational queries (availability checking) in a single SQL query. This hybrid approach delivers personalised scheduling that would require multiple round-trips in a pure relational model.

8. **Referral network as recursive graph traversal** -- the referral tree function uses recursive CTEs on graph edges to trace multi-level referral chains, enabling features like "your referral network has generated $X in revenue" and identifying influential clients.

9. **Properties on edges, not just nodes** -- edge properties carry rich metadata (visit counts, ratings, referral dates, co-occurrence percentages) that make the graph useful for analytics without needing to join back to relational tables.

10. **Graph node IDs match relational PKs** -- `graph_node.id` is the same UUID as the corresponding relational entity PK (`client.id`, `staff.id`, `service.id`). This eliminates the need for a mapping table and makes it trivial to join graph results back to relational data.
