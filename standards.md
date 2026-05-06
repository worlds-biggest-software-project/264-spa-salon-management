# Standards & API Reference

> Project: Spa & Salon Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The internationally recognised standard for establishing, implementing, and maintaining an information security management system (ISMS). Directly relevant for spa and salon platforms that store personal client data, medical history (for medspas), payment records, and employee information. Several practice-management platforms (e.g. Zanda) hold ISO 27001 certification to signal compliance to health-adjacent clients.

**ISO/IEC 27701:2019 (updated 2025) — Privacy Information Management Systems**
- URL: https://www.iso.org/standard/71670.html
- An extension to ISO 27001 and ISO 27002 that adds requirements and guidance for establishing a Privacy Information Management System (PIMS). Relevant when the platform handles personally identifiable information (PII) for clients and staff, particularly under GDPR obligations in EU markets.

**ISO 9001:2015 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- The global quality management standard applicable to service businesses including salons and spas. Software vendors serving the sector may adopt ISO 9001 as a marketing differentiator, and salons themselves may seek certification. Service-oriented software should support quality process documentation, customer feedback loops, and audit trails — all ISO 9001 requirements.

**ISO/IEC 27701:2025 — Privacy Information Management (revised)**
- URL: https://www.iso.org/standard/27701
- The 2025 revision refines the PIMS requirements, with increased relevance for businesses processing biometric or health-adjacent data (e.g. skin condition records in medspas).

---

### W3C & IETF Standards

**RFC 5545 — Internet Calendaring and Scheduling (iCalendar)**
- URL: https://datatracker.ietf.org/doc/html/rfc5545
- Defines the `.ics` / `.ical` data format used universally for representing appointments, events, and free/busy information. Salon management platforms that export or sync appointments with Google Calendar, Apple Calendar, Outlook, and similar tools must produce RFC 5545-compliant iCalendar objects. The MIME type is `text/calendar`.

**RFC 4791 — Calendaring Extensions to WebDAV (CalDAV)**
- URL: https://datatracker.ietf.org/doc/html/rfc4791
- Extends WebDAV to allow clients to access and manage calendar data on a server. CalDAV enables multi-user calendar sharing and synchronisation — relevant when the platform needs to sync appointment calendars with staff personal calendars or third-party scheduling tools.

**RFC 6638 — Scheduling Extensions to CalDAV**
- URL: https://datatracker.ietf.org/doc/rfc6638/
- Extends CalDAV with scheduling capabilities such as meeting invitations, replies, cancellations, and free/busy queries. Relevant for platforms supporting multi-provider booking where automated schedule negotiation is required.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The standard authorisation framework underpinning modern API security. All third-party integrations (payment processors, loyalty platforms, marketing tools) in the salon software ecosystem rely on OAuth 2.0. Platforms offering an open API should implement OAuth 2.0 for partner authentication.

**RFC 7636 — Proof Key for Code Exchange (PKCE)**
- URL: https://datatracker.ietf.org/doc/html/rfc7636
- Extends the OAuth 2.0 authorisation code flow for public clients (mobile apps, SPAs) that cannot securely store a client secret. Required for any salon booking mobile application that uses OAuth 2.0. OAuth 2.1 will mandate PKCE for all public clients.

**WCAG 2.2 — Web Content Accessibility Guidelines**
- URL: https://www.w3.org/WAI/standards-guidelines/wcag/
- Published as a W3C Recommendation in October 2023. The US Department of Justice's 2024 final rule (ADA Title II) mandates WCAG 2.1 Level AA compliance for web content and mobile apps. Online booking flows and client-facing portals must meet these requirements to serve customers with disabilities and avoid legal risk.

---

### Data Model & API Specifications

**OpenAPI Specification v3.1.0**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The industry standard for describing RESTful APIs. Any publicly exposed spa/salon management API should publish an OpenAPI 3.1 specification to enable automated client generation, interactive documentation (Swagger UI, ReDoc), and integration testing. The current stable version is 3.1.0, fully aligned with JSON Schema.

**JSON:API v1.1**
- URL: https://jsonapi.org/format/
- A convention-over-configuration specification for building APIs in JSON. Useful for maintaining consistent request/response structures (compound documents, sparse fieldsets, pagination links) across all API endpoints of a salon management platform.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/
- Used to define and validate the shape of JSON data models — client records, appointment objects, service catalogues, inventory items, and so on. Integral to OpenAPI 3.1 definitions and API contract testing.

---

### Security & Authentication Standards

**PCI DSS (Payment Card Industry Data Security Standard)**
- URL: https://www.pcisecuritystandards.org/
- Mandatory for any platform that processes, stores, or transmits cardholder data. Salon and spa platforms that accept credit card payments — either directly or via integrated payment processors — must comply with PCI DSS. Most platforms delegate card processing to PCI DSS-compliant providers (Stripe, Square) and avoid storing card data themselves, achieving SAQ A or SAQ A-EP self-assessment compliance.

**HIPAA (Health Insurance Portability and Accountability Act)**
- URL: https://www.hhs.gov/hipaa/index.html
- Applies to medspas and wellness centres that collect, use, or disclose protected health information (PHI) — for example, treatment records, skin condition photos, or medical histories. The 2025 OCR Security Rule updates introduced strengthened cybersecurity requirements. HIPAA-compliant spa software must provide encryption, role-based access, audit trails, and business associate agreements (BAAs).

**GDPR (General Data Protection Regulation)**
- URL: https://gdpr.eu/
- EU regulation governing personal data processing. Applicable to any platform operating in or serving EU-based salons and their clients. Key requirements: lawful basis for data processing, right to erasure, data portability, privacy-by-design, and breach notification within 72 hours. Salon software must provide configurable data retention, client consent tracking, and data export.

**OWASP Top 10**
- URL: https://owasp.org/www-project-top-ten/
- The de facto standard reference for web application security risks. Salon management platforms — which handle payment data, personal health information, and authentication credentials — should assess their implementations against the OWASP Top 10, particularly injection attacks, broken authentication, and sensitive data exposure.

**OAuth 2.0 Security Best Practices (IETF BCP)**
- URL: https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics
- IETF best current practices for deploying OAuth 2.0 securely, covering token storage, redirect URI validation, PKCE enforcement, and refresh token rotation. Required reading for any platform offering third-party API access.

---

### MCP Server Specifications

The Model Context Protocol (MCP) is relevant if the platform exposes AI-native capabilities. An MCP server interface for a spa/salon management system could expose:
- Tool calls for querying appointment availability, client records, and service menus
- Resource endpoints for staff schedules and inventory data
- Prompt templates for AI-assisted booking recommendations and client communication

MCP specification: https://modelcontextprotocol.io/specification

---

## Similar Products — Developer Documentation & APIs

### Zenoti
- **Description:** Enterprise cloud platform for salons, spas, and medspas, serving 10,000+ customers in 50 countries. Covers appointments, memberships, inventory, POS, marketing, and analytics.
- **API Documentation:** https://docs.zenoti.com/
- **Authentication:** API keys (1-year validity) and Bearer tokens (10-hour validity); OAuth playground available
- **API Types:** REST (JSON); also supports GraphQL playground and webhook event subscriptions
- **SDKs/Libraries:** Postman and Insomnia collections provided; no official language SDKs listed publicly
- **Developer Guide:** https://docs.zenoti.com/docs/create-the-backend-app-and-generate-a-new-api-key
- **Webhooks:** Yes — events include guest creation, appointment creation, invoice closure
- **Standards:** REST/JSON, OpenAPI; sandbox environment available

### Boulevard (BLVD)
- **Description:** Intelligent scheduling and POS platform targeting independent salons and multi-location enterprises. Known for its self-booking client experience and enterprise integrations.
- **API Documentation:** https://developers.joinblvd.com/
- **Authentication:** OAuth 2.0 with API key; sandbox accounts available via developer portal
- **API Types:** GraphQL (Admin API and unauthenticated Client/Booking API); webhooks via GraphQL mutations
- **SDKs/Libraries:** `@boulevard/blvd-book-sdk` (JavaScript/TypeScript) — https://github.com/Boulevard/book-sdk
- **Developer Guide:** https://www.joinblvd.com/features/api
- **Standards:** GraphQL; OpenAPI specs; sandbox environment

### Mindbody / Booker
- **Description:** Leading platform for fitness, wellness, and salon businesses. Booker (acquired by Mindbody) focuses specifically on salon and spa booking. Combined platform serves tens of thousands of businesses.
- **API Documentation:** https://developers.mindbodyonline.com/
- **Authentication:** API key + site credentials; OAuth 2.0 for consumer-facing flows
- **API Types:** REST (JSON); webhooks for near real-time event notifications; Booker API available via Azure API Management
- **SDKs/Libraries:** Postman collection for Consumer API; no official language SDKs
- **Developer Guide:** https://developers.mindbodyonline.com/ui/documentation/booker-api
- **Affiliate/Consumer API:** https://developers.mindbodyonline.com/AffiliateDocumentation
- **Standards:** REST/JSON; Swagger documentation available

### Phorest
- **Description:** Salon and spa software targeting growth-focused multi-location businesses, with strong client retention and marketing tools. Widely used in UK, Ireland, and US markets.
- **API Documentation:** https://developer.phorest.com/docs/getting-started
- **Authentication:** HTTP Basic Auth (API username and password supplied by Phorest)
- **API Types:** REST (JSON); resource-oriented URLs with standard HTTP verbs
- **SDKs/Libraries:** None publicly listed; integration requires own development team
- **Developer Guide:** https://support.phorest.com/hc/en-us/articles/360018509300-Getting-Started-with-Phorest-API
- **Standards:** REST/JSON; standard HTTP response codes
- **Note:** Card payment API not exposed; third-party integrations may incur charges

### Meevo (Millennium Systems International)
- **Description:** Cloud-based salon, spa, and medspa management platform with enterprise features including multi-location support, a Daily Data Stream, and a public REST API.
- **API Documentation:** https://www.meevo.com/developer-tools
- **Authentication:** Not publicly detailed; API key based
- **API Types:** REST (Public API for CRUD operations); Daily Data Stream (DDS) for incremental change feeds
- **SDKs/Libraries:** None publicly listed
- **Developer Guide:** Available via the developer tools portal
- **Key Data Objects:** Appointments, business info, client and employee profiles, memberships, packages, gift cards, services, products
- **Standards:** REST/JSON

### Vagaro
- **Description:** All-in-one platform for salons, spas, and fitness businesses. Serves SMBs and enterprise clients with booking, POS, marketing, and forms management.
- **API Documentation:** https://docs.vagaro.com/public/reference/api-introduction
- **Authentication:** API key; enterprise API access requires contact with Enterprise Sales Team
- **API Types:** REST; webhooks available ($10/month for 5,000 calls)
- **SDKs/Libraries:** None publicly listed
- **Developer Guide:** https://support.vagaro.com/hc/en-us/categories/34949493949851-Developer-Features
- **Webhook Events:** Appointments, customers, transactions, form responses
- **Standards:** REST/JSON

### Square Bookings API
- **Description:** Square's embedded booking API that allows developers to build custom appointment-scheduling experiences on top of Square's payment and business infrastructure. Generally available since May 2022.
- **API Documentation:** https://developer.squareup.com/docs/bookings-api/what-it-is
- **API Reference:** https://developer.squareup.com/reference/square/bookings-api
- **Authentication:** OAuth 2.0 (seller authorisation)
- **API Types:** REST (JSON); webhooks for booking events
- **SDKs/Libraries:** Official SDKs in Python, Ruby, Java, .NET, PHP, Go, JavaScript/TypeScript — https://developer.squareup.com/docs/sdks
- **Developer Guide:** https://developer.squareup.com/docs/bookings-api/use-the-api
- **Standards:** REST/JSON; OpenAPI spec available; OAuth 2.0

### Acuity Scheduling (Squarespace)
- **Description:** Online appointment scheduling platform used by independent practitioners and small salons. Acquired by Squarespace in 2019.
- **API Documentation:** https://developers.acuityscheduling.com/
- **Authentication:** HTTP Basic Auth (User ID + API Key)
- **API Types:** REST (JSON); base URL `https://acuityscheduling.com/api/v1/`
- **SDKs/Libraries:** Official JavaScript SDK (`acuity-js`) and PHP SDK (`acuity-php`) — https://github.com/AcuityScheduling
- **Developer Guide:** https://developers.acuityscheduling.com/
- **Standards:** REST/JSON; HTTPS required

### Stripe Payments API
- **Description:** Payment processing platform used by most salon management systems for card charging, deposit collection, no-show fee processing, and subscription/membership billing.
- **API Documentation:** https://docs.stripe.com/api
- **Authentication:** API keys (publishable and secret); OAuth 2.0 for Connect platform integrations
- **API Types:** REST (JSON); webhooks for payment lifecycle events
- **SDKs/Libraries:** Official SDKs in Python, Ruby, PHP, Java, .NET, Go, Node.js — https://docs.stripe.com/libraries
- **Developer Guide:** https://docs.stripe.com/payments
- **Key APIs:** Payment Intents, Payment Methods, Subscriptions, Connect (for marketplace/platform models)
- **Standards:** REST/JSON; OpenAPI spec available; PCI DSS compliant

### Twilio (SMS/Voice Reminders)
- **Description:** Cloud communications platform used by salon management platforms to send appointment reminders, confirmations, and cancellation notifications via SMS and voice.
- **API Documentation:** https://www.twilio.com/docs/sms
- **Authentication:** Account SID + Auth Token (HTTP Basic Auth); API Key pairs for production
- **API Types:** REST (JSON); webhook callbacks for delivery status
- **SDKs/Libraries:** Official SDKs in Python, Ruby, PHP, Java, .NET, Go, Node.js — https://www.twilio.com/docs/libraries
- **Developer Guide:** https://www.twilio.com/docs/studio/tutorials/how-to-send-appointment-reminders
- **Standards:** REST/JSON; E.164 phone number format (ITU-T E.164)

---

## Notes

**Fragmented API landscape:** There is no single industry-standard API schema for salon or spa management. Each major platform (Zenoti, Boulevard, Mindbody, Phorest, Meevo, Vagaro) has designed its own proprietary REST or GraphQL API with different authentication models, data models, and endpoint conventions. An AI-native open-source platform has an opportunity to define a clean, well-documented OpenAPI 3.1 schema as a reference implementation.

**GraphQL adoption is growing:** Boulevard's choice of GraphQL for both its admin and client-booking APIs is noteworthy. GraphQL allows clients to request exactly the data they need — valuable for complex scheduling UIs — and may become more prevalent in next-generation salon platforms.

**Calendar interoperability gap:** Most salon platforms do not implement CalDAV (RFC 4791) natively. Appointment sync with Google Calendar, Apple Calendar, and Outlook typically relies on iCalendar export links (RFC 5545) or third-party integration middleware (Zapier, Make). Native CalDAV support would be a meaningful differentiator.

**HIPAA is increasingly material:** As medspas grow (injections, laser treatments, medical aesthetics), the distinction between a beauty salon and a healthcare provider is blurring. Platforms that handle treatment records or before/after photos may need to comply with HIPAA, requiring BAAs, audit logging, and encryption standards that exceed typical SMB software.

**PCI DSS scope minimisation:** The dominant pattern is to tokenise card data via Stripe or Square and never store raw card numbers, keeping the platform in PCI DSS SAQ A scope. An open-source alternative should follow this pattern.

**Emerging AI/MCP opportunity:** None of the surveyed platforms expose an MCP server interface. An AI-native platform offering a well-defined MCP server would allow AI agents to book appointments, check availability, manage client records, and trigger communications — a capability not currently available in the market.
