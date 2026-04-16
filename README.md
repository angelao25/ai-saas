# Citai

> AI-powered business communication platform. WhatsApp, Instagram, and Messenger agents that handle sales, bookings, and orders — built on Rails, deployed on Hetzner, sold as a monthly SaaS.

---

## Table of contents

1. [What is Citai](#1-what-is-citai)
2. [The three products](#2-the-three-products)
3. [Target markets](#3-target-markets)
4. [Business types supported](#4-business-types-supported)
5. [How the agent works](#5-how-the-agent-works)
6. [Agent flows](#6-agent-flows)
7. [Channels](#7-channels)
8. [Payments](#8-payments)
9. [Onboarding process](#9-onboarding-process)
10. [Data architecture](#10-data-architecture)
11. [Tech stack](#11-tech-stack)
12. [Infrastructure](#12-infrastructure)
13. [Multi-tenancy model](#13-multi-tenancy-model)
14. [Security considerations](#14-security-considerations)
15. [Pricing model](#15-pricing-model)
16. [V1 scope](#16-v1-scope)
17. [V1.5 and v2 roadmap](#17-v15-and-v2-roadmap)
18. [Key architectural decisions](#18-key-architectural-decisions)
19. [Environment variables](#19-environment-variables)
20. [Local development setup](#20-local-development-setup)

---

## 1. What is Citai

Citai is a multi-tenant SaaS platform that puts AI agents behind the communication channels small and medium businesses already use — WhatsApp, Instagram DMs, and Facebook Messenger. The agent handles the full customer interaction loop: answering product or service queries, processing orders, scheduling appointments, collecting payments, and escalating to the business owner when human judgment is needed.

The name comes from **cita** — the Spanish word for appointment or arranged meeting — reflecting the platform's bilingual DNA and primary market in Latin America and the United States.

Citai is not a chatbot builder and not a no-code tool. It is a purpose-built Rails application configured by the Citai team on behalf of each client. Clients get a dashboard to monitor activity, manage their catalog, and view reports. The Citai team handles all technical configuration.

---

## 2. The three products

### Citai Agent (core — every client gets this)

The AI agent that runs on the client's existing WhatsApp, Instagram, and/or Messenger channels. Powered by the Claude API with tool-calling. Handles conversations in Spanish and English, detects customer language automatically, remembers returning customers, and follows a configured flow based on the business type.

### Citai Catalog (included with Agent)

Product and service management inside the client dashboard. The shared database layer that powers both the agent and the storefront. Clients manage their products, variants, stock levels, images, services, professionals, and pricing here. The agent reads from this in real time — no syncing required.

### Citai Store (optional add-on — v2)

A public-facing ecommerce storefront for clients who want a web presence. Pulls inventory directly from Citai Catalog — same database, same data. Orders from the web store land in the same orders table as agent orders. Stock reservations are shared across both channels to prevent overselling.

---

## 3. Target markets

**Peru**
- Primary market for v1
- Payment providers: Mercado Pago (QR + payment links), YAPE (manual confirmation for now)
- Currency: PEN (Peruvian Sol)
- Language default: Spanish
- WhatsApp and Instagram are dominant business communication channels
- Strong demand from food businesses, clothing retail, healthcare professionals

**United States**
- Secondary market for v1
- Payment provider: Stripe
- Currency: USD
- Language default: English
- Target verticals: healthcare (dentists, chiropractors), personal care (salons, spas)

---

## 4. Business types supported

### Booking agent
For businesses that operate on appointments and schedules.

**Target clients:** dentists, doctors, chiropractors, psychologists, nail technicians, hair stylists, massage therapists, personal trainers, tutors.

**What the agent does:**
- Checks availability across one or multiple professionals' Google Calendars
- Collects patient or client details
- Books appointment and creates calendar event
- Sends confirmation to customer
- Sends reminders before appointment
- Handles cancellations and rescheduling
- Outside business hours: still accepts bookings, schedules next available slot

### Order agent
For businesses that receive and fulfill product orders.

**Target clients:** food businesses, bakeries, cake shops, clothing retail, home goods, any product-based business that currently takes orders via WhatsApp manually.

**What the agent does:**
- Presents products from the catalog
- Handles size, color, variant, and stock queries
- Collects delivery or pickup preference
- Checks delivery zone eligibility
- Processes payment via Mercado Pago or Stripe
- Creates order ticket and notifies business owner
- Outside business hours: informs customer and reopening time

### Sales agent
For retail businesses with a catalog that customers browse before purchasing.

**Target clients:** clothing stores, accessory shops, boutiques, any retail business with an Instagram-first presence.

**What the agent does:**
- Handles browsing intent — customer describes what they want
- Presents matching products with images
- Checks stock by size, color, or variant in real time
- Soft-reserves items during checkout to prevent overselling
- Processes payment
- Creates order ticket
- Remembers returning customers and their purchase history

---

## 5. How the agent works

Each client has a **tenant** record in the database. The tenant record holds all configuration: business type, agent tone, language preference, business hours, services or products, payment methods, escalation contacts, and a generated system prompt.

When a customer sends a message on any connected channel, the following happens:

1. Meta's webhook fires a POST request to Citai's webhook endpoint
2. Rails identifies the tenant from the phone number or page ID in the payload
3. The customer's conversation history is loaded from the database
4. The message is passed to the agent service with the tenant's system prompt
5. The Claude API processes the message using tool-calling
6. If a tool is called (check availability, create booking, create order, send payment link), the tool executes and returns a result to the agent
7. The agent formulates a reply
8. The reply is sent back through the Meta API on the same channel the message came from
9. The conversation turn is persisted to the database

The agent maintains **conversation state** per customer per tenant. A returning customer is recognized by their phone number or Instagram/Messenger ID. Their name, previous orders, and preferences are loaded as context.

---

## 6. Agent flows

### Booking flow

```
Customer messages → Agent greets (personalized if returning)
→ Customer states need (book / reschedule / cancel / query)
→ Agent identifies service needed
→ Agent checks availability via Google Calendar tool
→ Agent presents available slots
→ Customer selects slot
→ Agent collects required details (name, reason if needed)
→ Payment collected if required upfront (Mercado Pago / Stripe)
→ Booking confirmed → Calendar event created → Confirmation sent
→ Reminder job scheduled (24h before)
→ Owner notified (if configured for every booking)
```

### Order flow

```
Customer messages → Agent greets (personalized if returning)
→ Customer states what they want
→ Agent presents matching products from catalog
→ Customer selects item + variant (size, color, etc.)
→ Agent asks: delivery or pickup?
→ If delivery: collect address → check zone → add delivery fee
→ Agent presents order summary → Customer confirms
→ Agent requests payment
  → Mercado Pago: send payment link → await webhook → auto-confirm
  → YAPE: send number + amount → await manual owner confirmation
  → Cash: confirm pickup reservation
→ Payment confirmed → Order ticket created → Owner notified
→ Confirmation sent to customer
```

### Sales / browsing flow

```
Customer messages → Agent greets
→ Customer expresses browsing intent or specific query
→ Agent identifies intent:
  → Specific item: check catalog → present item → check stock
  → Category browse: present options from category
  → Size/stock query: check variant availability
  → Price query: respond with pricing
→ Customer selects item + variant
→ Soft stock reservation created (15 min TTL)
→ → [continues as order flow from fulfillment step]
→ If reservation expires: stock released → customer notified
```

### Escalation flow

```
Agent encounters query outside its scope
→ Agent informs customer: "Let me get someone to help you"
→ Conversation marked as escalated in database
→ Agent pauses — stops auto-replying
→ Owner notified via dashboard notification + WhatsApp message
→ Owner replies from dashboard → reply sent via Meta API to customer
→ Owner marks conversation resolved → agent resumes if appropriate
```

### Outside business hours

```
Customer messages outside configured hours
→ Booking agent: informs customer → offers to book next available slot anyway
→ Order agent: informs customer → states reopening time → closes gracefully
→ Sales agent: informs customer → offers to save their interest for follow-up
```

---

## 7. Channels

All three channels are delivered through the **Meta Business Platform (Graph API)**. One webhook endpoint in Rails receives messages from all channels. The payload identifies the source channel and the receiving page or number, which maps to a tenant.

| Channel | API | Message types supported | Notes |
|---|---|---|---|
| WhatsApp | WhatsApp Business Cloud API | Text, images, documents, buttons, payment links, template messages | Richest feature set |
| Instagram DMs | Instagram Messaging API | Text, images, buttons, URL links | No native payment links — use URL |
| Facebook Messenger | Messenger Platform API | Text, images, buttons, URL links | Same as Instagram |

**Each client connects their own existing number or page.** Citai does not provision numbers. The client's WhatsApp Business account, Instagram Business account, or Facebook Page is connected to Citai's Meta App via OAuth during onboarding.

**Webhook routing:** One Rails endpoint `/webhooks/meta` receives all incoming events. The tenant is identified by matching the `recipient.id` (WhatsApp number, Instagram page ID, or Messenger page ID) against the `channel_connections` table.

**Channel column on conversations:** Every conversation record stores the originating channel (`whatsapp`, `instagram`, `messenger`). Replies are always sent back through the same channel.

---

## 8. Payments

Citai operates as a **facilitator only — never as a merchant of record.** Client funds go directly into the client's payment account. Citai never holds or processes client money. This keeps the legal and compliance surface clean in both Peru and the US.

### Mercado Pago (Peru and Latin America)

Each client connects their own Mercado Pago account. Citai stores their access token (encrypted) and uses it to generate payment links via the Mercado Pago Preferences API. When a customer pays, Mercado Pago fires a webhook to Citai confirming the payment. The agent receives this confirmation and continues the conversation automatically.

### Stripe (United States)

Each client connects their own Stripe account via Stripe Connect. Payment intents are created on behalf of the connected account. Stripe webhooks confirm payment. Same automated flow as Mercado Pago.

### YAPE (Peru — manual for v1)

YAPE has no public merchant API. For v1, the agent sends the client's YAPE phone number and the amount to the customer with instructions to send payment. The business owner confirms receipt manually via the dashboard. The dashboard surfaces a "confirm YAPE payment" action on pending orders. Once confirmed, the order proceeds.

**Future:** If YAPE releases a merchant API, or if the client uses a QR-based YAPE business account that fires confirmations, this can be automated. Research ongoing.

### Cash on delivery / pickup

No payment processing required. Agent confirms reservation and reminds customer of amount. Order ticket created immediately.

---

## 9. Onboarding process

All onboarding is performed by the Citai team. Clients do not self-configure. The onboarding intake collects all information needed to configure the tenant record, generate the agent system prompt, and connect the client's channels and payment accounts.

### Section 1 — Business identity
- Business name
- Business type (dentist / doctor / chiro / salon / food / cake / clothing / retail / other)
- Channels customers use (WhatsApp / Instagram / Messenger — all that apply)
- WhatsApp number (if applicable — must be WhatsApp Business)
- Instagram business handle (if applicable)
- Facebook Page name (if applicable)
- City and country of operation
- Agent default language (Spanish / English)
- Agent tone (friendly and casual / professional and formal)

### Section 2 — Business hours
- Open / closed status per day of week
- Opening and closing time per open day
- Outside hours behavior:
  - Order businesses: what to say, when reopening
  - Booking businesses: accept bookings outside hours? (recommended yes)

### Section 3 — Services or products
- **Booking businesses:** full service list with name, duration, and price. Number of professionals. Which services each professional offers.
- **Order / retail businesses:** full product list with name, description, variants (size, color, etc.), price per variant, and stock count per variant. Lead time required (e.g. cakes need 48h). Minimum order if applicable.

### Section 4 — Delivery and fulfillment
- **Order businesses:** delivery, pickup, or both. Delivery zones and fee. Business address.
- **Booking businesses:** clinic/salon location. Home visits offered? Business address.

### Section 5 — Payments
- Payment methods accepted
- Payment timing (upfront or on delivery/at appointment)
- Mercado Pago access token (if applicable)
- YAPE phone number (if applicable)
- Stripe account connection (if applicable)

### Section 6 — Escalation and notifications
- Owner/manager name
- WhatsApp number for escalation notifications
- Email for daily and weekly reports
- Notify on every transaction, or only on problems?

### Section 7 — Google Calendar (booking businesses only)
- One calendar per professional, or one shared calendar
- Each professional authorizes Google access during onboarding (one-time OAuth)
- Buffer time between appointments
- Cancellation policy — allowed via WhatsApp? How much notice required?

---

## 10. Data architecture

The database is the single source of truth for all three products. The agent, the client dashboard, and the Citai Store (v2) all read from and write to the same tables. No syncing between systems.

### Core tables

```
tenants
  id, slug, name, business_type, status
  agent_tone, default_language
  timezone, country, currency
  system_prompt (generated, stored, regenerated on config change)
  fulfillment_config (jsonb)
  payment_config (jsonb — encrypted credentials)
  escalation_config (jsonb)
  created_at, updated_at

users
  id, tenant_id, email, role (owner / staff / admin)
  encrypted_password (Devise)

channel_connections
  id, tenant_id, channel (whatsapp/instagram/messenger)
  external_id (phone number or page ID)
  access_token (encrypted), status
  connected_at

business_hours
  id, tenant_id, day_of_week (0–6)
  open_time, close_time, closed (boolean)

professionals (booking tenants)
  id, tenant_id, name, bio
  google_calendar_id, google_access_token (encrypted)

services (booking tenants)
  id, tenant_id, professional_id (optional — null means all)
  name, duration_minutes, price, currency, active

products (order/retail tenants)
  id, tenant_id, name, description, category
  base_price, currency, active
  images (array of storage URLs)

product_variants
  id, product_id, sku
  size, color, material (jsonb for arbitrary attributes)
  price_override (null uses base_price)
  stock_count

stock_reservations
  id, variant_id, conversation_id
  quantity, reserved_until, released_at

customers
  id, tenant_id, channel, external_id (phone/IG/Messenger ID)
  name, phone, email
  preferred_language, notes
  last_seen_at

orders
  id, tenant_id, customer_id
  channel (whatsapp/instagram/messenger/web)
  source (agent/store)
  status (pending/confirmed/paid/dispatched/cancelled)
  fulfillment_type (delivery/pickup)
  delivery_address (jsonb)
  subtotal, delivery_fee, total, currency
  payment_method, payment_status, payment_reference
  notes, created_at

order_items
  id, order_id, product_variant_id (or service_id)
  quantity, unit_price, total_price

bookings
  id, tenant_id, customer_id, professional_id, service_id
  scheduled_at, duration_minutes
  status (pending/confirmed/cancelled/completed)
  google_event_id
  payment_status, payment_reference
  notes, created_at

conversations
  id, tenant_id, customer_id
  channel (whatsapp/instagram/messenger)
  status (active/escalated/resolved/closed)
  escalated_at, resolved_at
  current_flow, flow_state (jsonb — tracks where in the flow)
  created_at, updated_at

messages
  id, conversation_id
  role (user/assistant/tool)
  content (text)
  media_url, media_type
  tool_name, tool_result (jsonb)
  sent_at

notifications
  id, tenant_id, type (escalation/order/booking/yape_pending)
  reference_type, reference_id
  message, read_at, created_at
```

### Key design decisions

**`flow_state` on conversations (jsonb):** Tracks the current step in the agent's flow for this conversation. For an order flow it might be `{step: "awaiting_payment", order_id: 123, reserved_until: "..."}`. This allows the agent to pick up exactly where it left off when a customer replies hours later.

**`system_prompt` on tenants:** Generated and stored when the tenant is configured or updated. Not generated on every request — this keeps latency low and cost predictable. Regenerated automatically when any tenant config changes.

**`payment_config` as jsonb:** Stores whichever payment credentials apply to this tenant. Encrypted at rest using Rails credentials + ActiveRecord encryption. Structure varies by provider: `{provider: "mercado_pago", access_token: "..."}` or `{provider: "stripe", account_id: "..."}`.

**`stock_reservations` TTL:** When a customer selects a product during an agent conversation, a reservation is created with `reserved_until = now + 15 minutes`. A Sidekiq job runs every minute to release expired reservations and return stock. If the customer completes payment before expiry, the reservation is converted to a confirmed order.

---

## 11. Tech stack

| Layer | Choice | Reason |
|---|---|---|
| Framework | Ruby on Rails 8 | Full-stack, mature ecosystem, convention over configuration, built-in everything |
| Database | PostgreSQL | Multi-tenancy via Apartment gem, jsonb for flexible config, robust and proven |
| Multi-tenancy | Apartment gem (schema-per-tenant) | Clean data isolation, upgrade path to separate DBs per tenant if needed |
| Background jobs | Sidekiq + Redis | Reminder jobs, stock reservation cleanup, webhook processing, report generation |
| Real-time | ActionCable (WebSockets) | Live conversation updates in dashboard |
| Auth | Devise | Tenant admin auth, role-based (owner / staff / Citai superadmin) |
| AI | Claude API (Anthropic) | Tool-calling, multi-turn conversations, multilingual, strong instruction following |
| WhatsApp/IG/Messenger | Meta Business Platform Graph API | Official API, all three channels unified, webhooks |
| Payments Peru | Mercado Pago API | Webhook-confirmed, QR + payment links, operates in Peru |
| Payments US | Stripe Connect | Industry standard, per-connected-account, webhook-confirmed |
| Calendar | Google Calendar API (OAuth2) | Per-professional calendar, free/busy queries, event creation |
| Email | ActionMailer + Postmark (or SendGrid) | Booking confirmations, order receipts, weekly reports |
| File storage | Hetzner Object Storage (S3-compatible) | Product images, stays within infrastructure |
| Frontend | Hotwire (Turbo + Stimulus) | Stays in Rails, no separate JS framework for dashboard |
| Embeddable widget | Vanilla JS | One script tag, no framework dependency for client websites |
| Deploy | Kamal (Rails 8 default) | Docker over SSH, zero extra infra, works perfectly with Hetzner |
| Web server | Puma + Caddy (reverse proxy) | Caddy handles SSL automatically |
| Monitoring | Sentry (free tier) | Error tracking across all tenants |

---

## 12. Infrastructure

### Hetzner VPS — initial setup

**Server:** CX21 (2 vCPU, 4GB RAM, 40GB SSD) — handles 20–50 active tenants comfortably.

**Upgrade path:** CX31 (4 vCPU, 8GB RAM) when approaching 100 tenants. At 200+ tenants, move Postgres to a dedicated Hetzner DB server and keep the Rails app on the VPS. Kamal supports multi-server deployments — the config change is minimal.

**Services running on the VPS:**
- Rails app (Puma, multiple workers)
- Sidekiq workers
- Redis (job queue + ActionCable pub/sub)
- Caddy (reverse proxy + automatic SSL)

**Hetzner Object Storage:** S3-compatible storage for product images and any media files. Separate from the VPS.

**Backups:** Hetzner automated daily snapshots. Postgres WAL archiving to Object Storage for point-in-time recovery.

### Deployment

Kamal deploys via Docker over SSH. One command: `kamal deploy`. Zero downtime deployments via container rolling restart. Secrets managed via Rails encrypted credentials (`config/credentials.yml.enc`).

### Meta webhook

The Rails app exposes a single public webhook endpoint: `POST /webhooks/meta`. This endpoint must be reachable by Meta's servers. Caddy handles SSL termination so Meta's HTTPS requirement is satisfied automatically.

---

## 13. Multi-tenancy model

Citai uses **schema-per-tenant** via the Apartment gem. Each tenant gets their own PostgreSQL schema. The `public` schema contains only the `tenants` table and global configuration. All tenant data (conversations, orders, bookings, products, customers, etc.) lives in the tenant's schema.

**Why schema-per-tenant over row-level isolation:**
- Cleaner data isolation — a query in one tenant's schema cannot accidentally return another tenant's data
- Easier to export or migrate a single tenant's data
- Upgrade path to a dedicated database per tenant for enterprise clients
- Better for compliance — US healthcare clients (HIPAA adjacent) feel better knowing their data is physically separated

**Tenant switching:** Apartment handles this via middleware. The tenant is identified from the incoming request (webhook payload for Meta messages, subdomain or JWT for dashboard) and the correct schema is set for the duration of the request.

**Superadmin access:** The Citai superadmin account operates across all schemas for monitoring and support purposes. A separate admin panel (not exposed to clients) shows all tenants, their conversation volumes, error rates, and agent health.

---

## 14. Security considerations

Given the cybersecurity background of the founding developer, the following are treated as first-class requirements, not afterthoughts.

**Credentials:** All API keys, access tokens, and secrets stored in Rails encrypted credentials. Never in environment variables in plain text. Never committed to git. Tenant payment credentials additionally encrypted at the column level using ActiveRecord Encryption.

**Webhook verification:** All incoming Meta webhooks verified using the `X-Hub-Signature-256` header. Requests that fail signature verification are rejected with 401 before any processing.

**Rate limiting:** Rack::Attack configured from day one. Limits on webhook endpoint (per IP), dashboard login (per email), and API endpoints (per tenant).

**Input sanitization:** All content from Meta webhooks treated as untrusted input. Sanitized before passing to the Claude API to prevent prompt injection from end customers.

**Tenant isolation:** Apartment gem enforces schema isolation. No cross-tenant queries possible in normal application code. Superadmin queries that cross schemas are explicitly marked and audited.

**Payment credentials:** Stored encrypted using ActiveRecord Encryption (Rails 7+ built-in). Encryption keys stored in Rails credentials, not in the database.

**HTTPS everywhere:** Caddy handles automatic SSL certificate provisioning and renewal via Let's Encrypt. All traffic encrypted in transit.

**Audit log:** Sensitive actions (tenant config changes, payment credential updates, superadmin access to tenant data) logged to an `audit_logs` table with actor, action, timestamp, and IP.

---

## 15. Pricing model

Citai charges clients a flat monthly subscription fee. Citai never takes a percentage of client transactions. API costs (Meta, Claude, Mercado Pago webhooks) are absorbed into the subscription margin.

| Plan | Price | Includes |
|---|---|---|
| Starter | $97/mo | 1 channel, 1 agent type, Citai Catalog, up to 500 conversations/month, basic dashboard |
| Growth | $197/mo | All 3 channels, 1 agent type, unlimited conversations, full dashboard + reports |
| Pro | $397/mo | All 3 channels, 2 agent types (booking + orders), Citai Store (v2), priority support, custom domain |

**Cost structure per active client (estimated):**
- Claude API: ~$5–15/mo depending on conversation volume
- Meta API: ~$10–20/mo depending on conversation volume
- Hetzner VPS share: ~$3–5/mo
- Sidekiq/Redis share: negligible
- **Total cost per client: ~$20–40/mo**
- **Margin at Starter: ~$57–77/mo per client**

At 20 clients on Starter: ~$1,140–1,540/mo net. At 100 clients mixed plans: ~$8,000–12,000/mo net.

---

## 16. V1 scope

The following is in scope for v1. Everything else is explicitly deferred.

**Agent:**
- [ ] Booking agent (dentists, doctors, chiros, salons)
- [ ] Order agent (food, cakes, retail)
- [ ] Sales / browsing agent (clothing, retail with catalog)
- [ ] Customer memory (returning customer recognition)
- [ ] Multi-language (Spanish + English, auto-detect)
- [ ] Business hours enforcement
- [ ] Escalation flow with owner notification
- [ ] Outside hours behavior (per agent type)

**Channels:**
- [ ] WhatsApp (Meta Business Cloud API)
- [ ] Instagram DMs
- [ ] Facebook Messenger
- [ ] Unified Meta webhook handler

**Payments:**
- [ ] Mercado Pago (Peru) — payment link + webhook confirmation
- [ ] Stripe (US) — payment link + webhook confirmation
- [ ] YAPE — manual dashboard confirmation
- [ ] Cash on delivery / pickup — no payment processing

**Calendar:**
- [ ] Google Calendar OAuth per professional
- [ ] Free/busy availability queries
- [ ] Appointment booking and event creation
- [ ] 24h reminder via WhatsApp

**Client dashboard:**
- [ ] Real-time conversation view
- [ ] Escalation handling (reply from dashboard)
- [ ] Citai Catalog — product and service management
- [ ] Order and booking history
- [ ] Daily and weekly summary reports
- [ ] Google Calendar embed per professional

**Superadmin panel:**
- [ ] All tenant conversations in real time
- [ ] Tenant onboarding and configuration
- [ ] Agent health monitoring
- [ ] System prompt editor per tenant

**Infrastructure:**
- [ ] Single Hetzner VPS
- [ ] Kamal deploy pipeline
- [ ] Caddy SSL
- [ ] Sidekiq + Redis
- [ ] Sentry error monitoring
- [ ] Automated daily backups

---

## 17. V1.5 and v2 roadmap

**V1.5 (within 4–6 weeks of v1 launch):**
- Missed call text-back via Twilio Voice — caller doesn't get answered, receives automatic WhatsApp or SMS with agent greeting
- Stripe billing for Citai subscription management (currently manual invoicing)

**V2:**
- Citai Store — public ecommerce storefront pulling from Citai Catalog
- Custom domain support for Citai Store
- Visual flow builder — clients can modify agent flows without Citai team involvement
- InDrive / Yango delivery dispatch integration (Peru)
- YAPE automation if API becomes available
- Mercado Pago QR code generation in chat
- Multi-agent per tenant (booking agent + order agent for same business)

---

## 18. Key architectural decisions

### Why hardcoded flows for v1
The agent flows (booking, order, sales) are implemented as Rails service objects with a defined sequence, not as a visual or DB-configurable flow engine. Per-tenant customization happens through the system prompt stored in the `tenants` table. This gets v1 to market faster and avoids over-engineering before real client feedback. A flow engine is planned for v2.

### Why schema-per-tenant over row-level security
Data isolation is a selling point, not an implementation detail. Healthcare clients in the US care about this. It also gives a cleaner upgrade path to dedicated databases per tenant.

### Why Meta's official API over third-party WhatsApp providers
Third-party providers (waha, whapi.cloud) use unofficial WhatsApp Web reverse engineering. Meta actively blocks these. They are not suitable for a production SaaS serving paying clients. The official Meta Business API is more work to set up but is stable, supported, and compliant.

### Why Citai never touches client funds
Operating as a payment facilitator or merchant of record requires financial licensing in both Peru and the US. By having each client connect their own Mercado Pago or Stripe account, Citai avoids all of this. The platform facilitates payment flows but never holds money. Revenue is purely subscription-based.

### Why the catalog is shared across agent and store
A single source of truth eliminates sync complexity, prevents stock inconsistencies, and means the client manages their catalog in one place. The agent always has live stock data. The web store (v2) always reflects the same inventory.

---

## 19. Environment variables

All secrets managed via Rails encrypted credentials (`config/credentials.yml.enc`). Structure:

```yaml
secret_key_base: ...

anthropic:
  api_key: ...

meta:
  app_id: ...
  app_secret: ...
  verify_token: ...

google:
  client_id: ...
  client_secret: ...

postmark:
  api_token: ...

sentry:
  dsn: ...

hetzner:
  object_storage_access_key: ...
  object_storage_secret_key: ...
  object_storage_bucket: ...
  object_storage_endpoint: ...
```

Tenant-level credentials (Mercado Pago access tokens, Stripe account IDs, Google Calendar tokens) are stored encrypted in the database using ActiveRecord Encryption. Encryption keys are stored in Rails credentials under `active_record_encryption`.

---

## 20. Local development setup

```bash
# Prerequisites: Ruby 3.3+, PostgreSQL, Redis, Node.js

git clone https://github.com/your-username/citai.git
cd citai

bundle install

# Set up credentials
rails credentials:edit

# Database setup
rails db:create db:migrate db:seed

# Start all services
bin/dev  # starts Puma + Sidekiq + CSS watcher via Procfile.dev

# Expose local webhook to Meta (for development)
# Install ngrok or use Cloudflare Tunnel
ngrok http 3000
# Set the ngrok URL as your Meta App webhook URL during development
```

**Seeding a test tenant:**
```bash
rails db:seed
# Creates: 1 superadmin user, 1 test tenant (booking type), 1 test tenant (order type)
# Credentials in db/seeds.rb
```

---

## Project status

| Milestone | Status |
|---|---|
| Technical specification | Complete |
| Onboarding templates | Complete |
| Agent flow design | Complete |
| GitHub issues and milestones | In progress |
| Rails app initialization | Not started |
| Core data model | Not started |
| Meta webhook integration | Not started |
| AI agent core | Not started |
| Google Calendar integration | Not started |
| Payment integrations | Not started |
| Client dashboard | Not started |
| Superadmin panel | Not started |
| Production deploy | Not started |

---

*Built by [your name] · Citai · 2025*
