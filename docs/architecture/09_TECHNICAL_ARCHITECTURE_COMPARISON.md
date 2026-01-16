# Technical Architecture Comparison

**Source:** ChatGPT Deep Research (January 2026)
**Purpose:** Architecture decisions for Ziloss CRM

---

## Executive Summary

Analysis of Salesforce, HubSpot, and GoHighLevel architectures reveals:
- **Multi-tenancy:** All use shared DB with row-level isolation (no separate DB per tenant)
- **Workflows:** Event-driven, async processing is essential at scale
- **Messaging:** Queue everything, use third-party providers (Twilio, Mailgun)
- **Storage:** Start with SQL, add search engine later
- **Backend:** Modular monolith is fine to start, microservices can come later

---

## 1. Multi-Tenancy Architecture

### Comparison

| Platform | Model | Isolation |
|----------|-------|-----------|
| **Salesforce** | Single shared tables, OrgID per row | Metadata-driven Universal Data Dictionary |
| **HubSpot** | Microservices, each multi-tenant | AccountID per service/record |
| **GoHighLevel** | Two-tier: Agency → Sub-accounts | location_id on every record |

### GHL's Two-Tier Model (What We Need)

```
Agency (Parent Tenant)
├── Location 1 (Sub-account)
│   ├── Contacts
│   ├── Conversations
│   └── Workflows
├── Location 2 (Sub-account)
└── Location N...
```

> "GHL's model resembles a two-tier multi-tenant system: the primary tenant is the agency (often the paying customer), and secondary tenants are the client accounts."

### Ziloss Implementation

```sql
-- Every table has:
CREATE TABLE contacts (
    id UUID PRIMARY KEY,
    agency_id UUID NOT NULL,      -- Parent tenant
    location_id UUID NOT NULL,    -- Sub-account
    -- ... other fields
    CONSTRAINT fk_location FOREIGN KEY (location_id) REFERENCES locations(id)
);

-- Row-level security policy
CREATE POLICY tenant_isolation ON contacts
    USING (location_id = current_setting('app.current_location_id')::uuid);
```

**Key Principle:** "None of them spin up a new database per customer by default, as that would hurt scalability."

---

## 2. Workflow Engine Architecture

### Comparison

| Platform | Execution Model | Queue Technology |
|----------|-----------------|------------------|
| **Salesforce** | Synchronous with async paths | Internal DB queue |
| **HubSpot** | Event-driven, Kafka everywhere | Kafka topics + swimlanes |
| **GoHighLevel** | Async workers | Redis/SQS likely |

### HubSpot's Kafka Swimlanes (Brilliant Design)

> "HubSpot introduced 'Kafka swimlanes' – multiple parallel Kafka topics – to isolate heavy workflows from the rest."

```
Normal Traffic → Real-time Lane (fast) → 99% of customers
Heavy Traffic → Overflow Lane (slower) → Noisy tenants

Detection: Per-customer rate limiter
If events/second > threshold → Route to overflow
```

This prevents one customer's massive workflow from lagging everyone else.

### HubSpot Scale Numbers

- "Millions of active workflows"
- "Hundreds of millions of actions per day"
- "Tens of thousands of actions per second at peak"

### Ziloss Implementation

```
Phase 1: Redis Queue (Simple)
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Trigger   │────▶│ Redis Queue │────▶│   Worker    │
│   Event     │     │             │     │   Pool      │
└─────────────┘     └─────────────┘     └─────────────┘

Phase 2: Add Swimlanes (Scale)
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Trigger   │────▶│ Rate Check  │────▶│ Real-time Q │
│   Event     │     │             │     └─────────────┘
└─────────────┘     └──────┬──────┘
                          │ If over limit
                          ▼
                   ┌─────────────┐
                   │ Overflow Q  │
                   └─────────────┘
```

---

## 3. Messaging Queue Architecture (SMS/Email)

### Comparison

| Platform | SMS Provider | Email Provider | Internal Queue |
|----------|--------------|----------------|----------------|
| **Salesforce** | Twilio integration | Own relays + Marketing Cloud | JMS/internal |
| **HubSpot** | Partners | Own SMTP network | Kafka |
| **GoHighLevel** | Twilio | Mailgun | Redis/SQS |

### GHL's Orchestrator Pattern

> "GHL's backend acts as an orchestrator: if a user launches a campaign to send 5,000 SMS, GHL will iterate through contacts and call Twilio's API for each (or in batches)."

```
Campaign Start
     │
     ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Job Queue  │────▶│   Worker    │────▶│ Twilio API  │
│  (Redis)    │     │ (rate limit)│     │             │
└─────────────┘     └──────┬──────┘     └─────────────┘
                          │
                          ▼
                   ┌─────────────┐
                   │  Webhook    │ (delivery status)
                   │  Handler    │
                   └─────────────┘
```

### Rate Limiting Strategy

> "GHL could implement a throttle (e.g. X messages per second) and a retry mechanism if Twilio responds with a temporary error."

**Ziloss Implementation:**
- Redis queue for outbound messages
- Worker pool with configurable rate limits
- Exponential backoff on provider errors
- Webhook ingestion for delivery status
- Dead letter queue for failed messages

---

## 4. Data Storage Strategy

### Comparison

| Platform | Primary Store | Search Engine | Analytics |
|----------|---------------|---------------|-----------|
| **Salesforce** | Oracle (shared tables) | Lucene (async) | Einstein Analytics (separate) |
| **HubSpot** | HBase (5+ PB) | Elasticsearch (likely) | Hadoop/Spark |
| **GoHighLevel** | PostgreSQL/MySQL | Basic indexes | Basic (export to warehouse) |

### Key Insights

**Salesforce's Async Search:**
> "As records are created/updated, a background process copies text fields into a search index outside the core transaction."

**HubSpot's Scale:**
> "HubSpot stores most of our customers' data in HBase... they manage 5+ PB of data."

**GHL's Pragmatic Approach:**
> "A SQL database is a strong bet... All data would have a tenant (sub-account) ID."

### Ziloss Implementation

```
Phase 1: PostgreSQL Only
- All tables with location_id, agency_id
- B-tree indexes on foreign keys
- pg_trgm extension for ILIKE searches
- GIN indexes on JSONB columns

Phase 2: Add Elasticsearch (when needed)
- Index: contacts, messages, notes
- Sync via PostgreSQL triggers or CDC
- Enable cross-entity search

Phase 3: Analytics (future)
- TimescaleDB for time-series metrics
- Or export to BigQuery/Snowflake
```

### Search Strategy Quote

> "A prudent strategy is likely using a scalable SQL database for core data (PostgreSQL is a common choice for its balance of reliability and JSON flexibility), and then adding Elasticsearch or a managed search service for full-text and cross-entity search capabilities."

---

## 5. Backend Tech Stack

### Comparison

| Platform | Language | Architecture | Deployment |
|----------|----------|--------------|------------|
| **Salesforce** | Java, Apex | Monolith (multitenant kernel) | Own datacenters |
| **HubSpot** | Java, Node, Go | 3000+ microservices | Kubernetes on AWS |
| **GoHighLevel** | Node.js/TypeScript | Modular monolith | Containers on GCP/AWS |

### Key Quote

> "For smaller teams, a monolithic app (e.g. built on Rails) can be advantageous until you hit a certain scale."

> "HighLevel appears to have started with a more monolithic approach and only now as they grow are they dealing with scaling that out."

### Ziloss Stack Decision

```
Language: Python 3.11+
Framework: FastAPI (async, modern)
Database: PostgreSQL 15 + pgvector
Cache: Redis
Queue: Redis (upgrade to Kafka if needed)
Search: PostgreSQL initially, Elasticsearch later
Deployment: Docker + Kubernetes

Architecture: Modular Monolith
- Clean module boundaries
- Could split to microservices later
- But don't need 3000 services to start
```

### Why Python/FastAPI over Node.js?

1. Better async/await ergonomics
2. Type hints + Pydantic validation
3. Easier AI/ML integration (Claude SDK)
4. Strong data science ecosystem
5. Team familiarity (we're already here)

---

## 6. Real-Time Capabilities

### Comparison

| Platform | Technology | Use Cases |
|----------|------------|-----------|
| **Salesforce** | CometD (long-polling) | Streaming API, Platform Events |
| **HubSpot** | WebSockets | Live chat, notifications, collab |
| **GoHighLevel** | Socket.io/WebSockets | Inbox, live dashboards |

### Key Quote

> "A client's browser might open a WebSocket connection to the HighLevel server to get immediate notifications of new chat messages or live calls."

### Ziloss Implementation

```
Technology: FastAPI WebSocket + Redis Pub/Sub

Use Cases:
1. New message arrives → Push to inbox
2. Workflow completes → Update UI
3. Team member typing → Show indicator
4. Live dashboard metrics → Real-time charts

Architecture:
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │◀───▶│  WebSocket  │◀───▶│ Redis Pub/  │
│  (Browser)  │     │   Server    │     │    Sub      │
└─────────────┘     └─────────────┘     └─────────────┘
                                              ▲
                                              │
                                    ┌─────────────┐
                                    │  Backend    │
                                    │  Services   │
                                    └─────────────┘
```

---

## 7. Architecture Recommendations Summary

### Do This (Proven Patterns)

| Area | Decision | Rationale |
|------|----------|-----------|
| Multi-tenancy | Single DB, row-level isolation | All three do this |
| Tenant model | Agency → Location hierarchy | Matches GHL, supports white-label |
| Workflows | Event-driven, async workers | Essential at scale |
| Messaging | Twilio/Mailgun + Redis queue | GHL's approach works |
| Storage | PostgreSQL + search engine | Flexible, scalable |
| Backend | Modular monolith | Start simple, split later |
| Real-time | WebSockets | Expected feature now |

### Don't Do This (Overkill for MVP)

| Anti-Pattern | Why Avoid |
|--------------|-----------|
| Separate DB per tenant | Doesn't scale, hard to maintain |
| 3000 microservices | HubSpot needed 10+ years to get there |
| Build own email servers | Use SendGrid/Mailgun |
| Kafka from day one | Redis queues are fine to start |
| HBase/NoSQL for core data | PostgreSQL handles our scale |

### Scale Triggers (When to Upgrade)

| Metric | Current Solution | Upgrade To |
|--------|------------------|------------|
| < 1M contacts | PostgreSQL search | Elasticsearch |
| < 10K workflows/day | Redis queue | Kafka |
| < 100 concurrent users | Single DB | Read replicas |
| < 1M messages/month | Twilio direct | Own SMS aggregator |

---

## Appendix: GHL Technical Details

From the research:

- **Backend:** Node.js/TypeScript (confirmed by custom code feature)
- **Database:** Cloud SQL (GCP) or RDS (AWS), likely PostgreSQL/MySQL
- **Cache:** Redis (mentioned in SDK)
- **Queue:** Redis or SQS for background jobs
- **Real-time:** Socket.io or native WebSockets
- **Deployment:** Containers on GCP/AWS, auto-scaling
- **Third-party:** Twilio (SMS/voice), Mailgun (email)

---

## Key Quotes to Remember

> "All three platforms favor shared infrastructure with logical isolation (multi-tenant SaaS) over siloed single-tenant deployments."

> "HubSpot's workflow engine uses Kafka to decouple when a workflow is triggered from when its actions are executed."

> "GHL's backend acts as an orchestrator... iterating through contacts and calling Twilio's API for each."

> "A new competitor can learn from this: if email/SMS is core, you need a robust queuing mechanism."

> "Start with a well-structured modular monolith, get product-market fit, then gradually peel off services."
