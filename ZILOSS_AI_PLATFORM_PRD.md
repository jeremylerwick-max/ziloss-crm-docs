# PRODUCT REQUIREMENTS DOCUMENT (PRD)

# ZILOSS AI AGENT PLATFORM

**Version:** 1.0  
**Author:** Jeremy Lerwick / Ziloss Technologies  
**Date:** January 4, 2026  
**Status:** Draft

---

## Executive Summary

Ziloss is an AI agent platform that connects to business systems via API and autonomously executes workflows across marketing, sales, and operations. Users provide API credentials, describe their goals in natural language, and the AI handles execution—learning and adapting over time.

**Vision:** The last software a business needs. One AI that connects to everything and runs your operations.

**Tagline Options:**
- "Plug in your APIs. Tell it what you want. It runs your business."
- "Your AI employee that never sleeps."
- "Connect. Command. Scale."

---

## Problem Statement

### The Current Reality

Businesses today use 10-50+ SaaS tools. Each requires:
- Learning a unique UI
- Building automations in that tool's rigid workflow builder
- Hiring specialists or agencies to manage
- Paying $200-2000/month per tool

**Example: A home services company running lead gen**

| Tool | Purpose | Cost/mo | Expertise Needed |
|------|---------|---------|------------------|
| GoHighLevel | CRM, SMS, automation | $497 | GHL specialist |
| Meta Ads | Lead generation | Variable | Media buyer |
| Google Ads | Lead generation | Variable | Media buyer |
| Calendly | Appointment booking | $15 | Basic |
| Zapier | Glue between tools | $50-200 | Automation knowledge |
| Spreadsheets | Reporting | Free | Manual work |
| **Agency fees** | Manage all above | $2,000-10,000 | - |

**Total:** $3,000-15,000/month + constant manual oversight

### The Problem

1. **Complexity explosion** - Each tool adds cognitive load
2. **Rigid automation** - Workflows break when conditions change
3. **No learning** - Systems repeat mistakes forever
4. **Fragmented data** - Insights trapped in silos
5. **Human bottleneck** - Someone must monitor and adjust everything

### The Opportunity

AI models can now:
- Understand natural language instructions
- Make contextual decisions
- Learn from outcomes
- Execute via APIs
- Work 24/7 without fatigue

**Why hasn't this been built?**
- LLMs just became capable enough (2024-2025)
- Most AI tools are single-purpose (chatbots, copilots)
- Integration is hard—requires deep domain knowledge
- Trust barrier—businesses scared to let AI "do things"

---

## Solution Overview

### What Ziloss Does

```
┌─────────────────────────────────────────────────────────────────┐
│                         ZILOSS AGENT                            │
│                                                                 │
│   "Run my lead nurturing. Book appointments. Report weekly."    │
│                              │                                  │
│                              ▼                                  │
│   ┌──────────────────────────────────────────────────────┐     │
│   │              NATURAL LANGUAGE INTERFACE               │     │
│   │         Understands goals, asks clarifying Qs         │     │
│   └──────────────────────────────────────────────────────┘     │
│                              │                                  │
│                              ▼                                  │
│   ┌──────────────────────────────────────────────────────┐     │
│   │              ORCHESTRATION ENGINE                     │     │
│   │    Breaks goals into tasks, routes to specialists     │     │
│   └──────────────────────────────────────────────────────┘     │
│                              │                                  │
│          ┌───────────────────┼───────────────────┐             │
│          ▼                   ▼                   ▼             │
│   ┌────────────┐     ┌────────────┐     ┌────────────┐        │
│   │  COMMS     │     │  ADS       │     │  DATA      │        │
│   │  AGENT     │     │  AGENT     │     │  AGENT     │        │
│   │            │     │            │     │            │        │
│   │ SMS/Email  │     │ Meta/Google│     │ Reporting  │        │
│   │ Follow-up  │     │ Campaigns  │     │ Analytics  │        │
│   │ Booking    │     │ Budgets    │     │ Insights   │        │
│   └─────┬──────┘     └─────┬──────┘     └─────┬──────┘        │
│         │                  │                  │                │
│         ▼                  ▼                  ▼                │
│   ┌──────────────────────────────────────────────────────┐     │
│   │                  API INTEGRATION LAYER                │     │
│   │   GHL │ Twilio │ Meta │ Google │ Sheets │ Calendars  │     │
│   └──────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### Core Concept: API-First AI Agent

1. **Connect** - User provides API keys for their existing tools
2. **Command** - User describes what they want in plain English
3. **Execute** - AI agent performs tasks autonomously
4. **Learn** - Agent improves based on outcomes
5. **Report** - User gets updates, can intervene anytime

---

## User Personas

### Persona 1: Agency Owner (Primary)

**Name:** Mike, Owner of a digital marketing agency  
**Tools:** GHL, Meta Ads, Google Ads for 15 clients  
**Pain:** Spends 60% of time on repetitive campaign management  
**Dream:** "I want to give the AI my client accounts and have it run everything. I just review and approve."

**Use Case:**
- Connects GHL + Meta Ads APIs for each client
- "Run lead nurturing for all clients. Book appointments. Pause anyone who says stop. Send me a daily summary."
- AI handles 90% of work, Mike handles exceptions

### Persona 2: SMB Owner

**Name:** Sarah, owns a window replacement company  
**Tools:** Basic CRM, runs own Facebook ads  
**Pain:** Spends evenings responding to leads, misses hot ones  
**Dream:** "I want to focus on sales calls, not chasing leads"

**Use Case:**
- Connects Facebook Lead Ads + Google Calendar
- "When a lead comes in, text them within 2 minutes. Try to book on my calendar. If no response in 24 hours, follow up."
- AI becomes her SDR

### Persona 3: Solopreneur / Creator

**Name:** Alex, runs an online coaching business  
**Tools:** Stripe, ConvertKit, Calendly  
**Pain:** Manual onboarding, missed follow-ups  
**Dream:** "Automate everything except the actual coaching calls"

**Use Case:**
- Connects Stripe + Calendar + Email
- "When someone buys, send welcome sequence, book onboarding call, remind them day before"
- AI handles entire client journey

---

## Feature Requirements

### Phase 1: Foundation (MVP)

**Target:** 3 months to working beta

#### 1.1 Natural Language Command Interface

**Description:** Users interact with the agent in plain English via chat or voice.

**Requirements:**
- Accept text commands via web UI
- Parse intent and extract entities (contacts, dates, amounts)
- Ask clarifying questions when ambiguous
- Confirm destructive actions before executing
- Remember context within conversation

**Examples:**
```
User: "Text everyone who hasn't responded in 3 days"
Agent: "I found 47 contacts who haven't responded since January 1st. 
        Should I send them: 
        A) The standard follow-up message
        B) A custom message you provide
        C) Let me draft something based on their last conversation"

User: "B - say 'Hey {first_name}, just checking if you're still interested 
       in getting a quote for your windows?'"
Agent: "Got it. I'll send that to 47 contacts. Want me to:
        - Send all now
        - Stagger over 2 hours (recommended to avoid spam filters)
        - Send to 5 first as a test"
```

#### 1.2 API Integration Framework

**Description:** Pluggable system to connect any SaaS tool via API.

**Requirements:**
- Secure credential storage (encrypted at rest)
- OAuth flow for supported platforms
- API key input for others
- Connection health monitoring
- Rate limit handling
- Automatic retry with backoff

**Initial Integrations (MVP):**
| Platform | Capabilities |
|----------|--------------|
| GoHighLevel | Contacts, conversations, tags, workflows, appointments |
| Twilio | SMS send/receive, call initiation |
| Google Sheets | Read/write data |
| Google Calendar | Read/create events |
| Meta Ads | Campaign read, basic optimization |

**Future Integrations:**
- Stripe (payments)
- Google Ads
- Slack (notifications)
- Email (SMTP/IMAP)
- Zapier (catch-all)

#### 1.3 Workflow Engine

**Description:** Executes multi-step processes with decision logic.

**Requirements:**
- Define workflows via natural language or visual builder
- Support triggers: time-based, event-based, condition-based
- Support actions: API calls, AI decisions, notifications
- Branching logic (if/then/else)
- Wait states (delay 24 hours, wait for response)
- Error handling with fallback actions

**Example Workflow: Lead Nurturing**
```yaml
name: New Lead Follow-up
trigger: new_contact_tagged("new_lead")

steps:
  - action: send_sms
    message: "Hi {{first_name}}, thanks for your interest! When's a good time to chat?"
    
  - wait: response OR 2_hours
  
  - if: response_received
      - action: ai_conversation
        goal: "Book an appointment"
        max_turns: 10
    else:
      - action: send_sms
        message: "Hey {{first_name}}, just following up..."
      - wait: response OR 24_hours
      
  - if: appointment_booked
      - action: add_tag("booked")
      - action: notify_owner
    else:
      - action: add_tag("needs_manual_review")
```

#### 1.4 Conversational AI Module

**Description:** AI that can hold natural conversations to achieve goals.

**Requirements:**
- Goal-oriented (book appointment, qualify lead, handle objection)
- Context-aware (knows conversation history, contact info)
- Persona customization (tone, style, name)
- Escalation triggers (detect frustration, complex questions)
- Multi-turn handling with memory

**Capabilities:**
- Appointment booking (check availability, confirm time)
- Lead qualification (ask qualifying questions)
- Objection handling (price, timing, competition)
- FAQ responses (based on knowledge base)
- Opt-out processing (detect stop keywords)

#### 1.5 Dashboard & Monitoring

**Description:** Real-time visibility into what the agent is doing.

**Requirements:**
- Activity feed (recent actions taken)
- Pending approvals (actions awaiting human confirmation)
- Performance metrics (messages sent, appointments booked)
- Error log (failed actions, blocked messages)
- Cost tracking (API usage, AI tokens)

---

### Phase 2: Intelligence Layer

**Target:** Months 4-6

#### 2.1 Learning & Optimization

**Description:** Agent improves based on outcomes.

**Requirements:**
- Track outcome metrics (response rate, booking rate, show rate)
- A/B test message variants automatically
- Identify best times to contact
- Learn which leads are worth pursuing
- Suggest workflow improvements

**Example:**
```
Agent: "I noticed your booking rate is 23% higher when I mention 
        'free estimate' vs 'free quote'. Want me to update all 
        messages to use 'estimate'?"
```

#### 2.2 Predictive Actions

**Description:** Agent anticipates needs and suggests actions.

**Requirements:**
- Predict lead quality from initial data
- Forecast appointment no-shows
- Identify at-risk deals
- Suggest re-engagement for cold leads

#### 2.3 Cross-Client Intelligence (Agencies)

**Description:** Learnings from one client benefit others (anonymized).

**Requirements:**
- Aggregate performance patterns across clients
- Industry-specific benchmarks
- Best practice recommendations
- Privacy-preserving (no PII shared)

---

### Phase 3: Full Autonomy

**Target:** Months 7-12

#### 3.1 Ad Campaign Management

**Description:** AI manages advertising budget and creative.

**Capabilities:**
- Create campaigns from natural language brief
- Allocate budget across platforms
- Pause underperforming ads
- Scale winners
- Generate ad creative variations

#### 3.2 Revenue Operations

**Description:** End-to-end revenue pipeline management.

**Capabilities:**
- Lead scoring and routing
- Pipeline forecasting
- Churn prediction
- Upsell identification
- Commission calculation

#### 3.3 Multi-Agent Collaboration

**Description:** Specialized agents working together.

**Architecture:**
```
                    ORCHESTRATOR (Claude)
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   ┌─────────┐      ┌─────────┐      ┌─────────┐
   │  SCOUT  │      │ ANALYST │      │ COMMS   │
   │         │      │         │      │         │
   │ Research│      │ Reports │      │ SMS/    │
   │ Monitor │      │ Insights│      │ Email   │
   └─────────┘      └─────────┘      └─────────┘
   Perplexity       Gemini           Fine-tuned
   + Web            Flash            Llama
```

---

## Technical Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
│   Web App │ Mobile App │ API │ Voice (future)                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        API GATEWAY                              │
│   Authentication │ Rate Limiting │ Request Routing              │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   COMMAND    │      │   WORKFLOW   │      │   REALTIME   │
│   SERVICE    │      │   ENGINE     │      │   EVENTS     │
│              │      │              │      │              │
│ NL parsing   │      │ Execution    │      │ Webhooks     │
│ Intent       │      │ Scheduling   │      │ WebSockets   │
│ Routing      │      │ State mgmt   │      │ Push notifs  │
└──────────────┘      └──────────────┘      └──────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     INTEGRATION LAYER                           │
│                                                                 │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │   GHL   │ │ Twilio  │ │  Meta   │ │ Google  │ │ Stripe  │   │
│  │ Adapter │ │ Adapter │ │ Adapter │ │ Adapter │ │ Adapter │   │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       DATA LAYER                                │
│                                                                 │
│  PostgreSQL     │    Redis        │    S3/R2                    │
│  (Core data)    │    (Cache/Queue)│    (Files/Logs)             │
└─────────────────────────────────────────────────────────────────┘
```

### Technology Stack

| Layer | Technology | Rationale |
|-------|------------|-----------|
| Frontend | React + Next.js | Fast, modern, good DX |
| API | FastAPI (Python) | Async, fast, AI ecosystem |
| Workflow Engine | Temporal.io | Battle-tested, durable execution |
| Database | PostgreSQL + pgvector | Relational + vector search |
| Cache/Queue | Redis | Fast, pub/sub, job queue |
| AI/LLM | Claude API + local Llama | Quality + cost optimization |
| Hosting | Railway / Vercel / Cloudflare | Scalable, affordable |

### Existing Assets (agent-orchestrator)

**Already Built:**
- LangGraph orchestration framework
- FastAPI backend structure
- Natural language interface
- Browser automation (MCP)
- Memory system with embeddings
- 31 modules, 207+ files

**Needs Work:**
- Production hardening
- Multi-tenant architecture
- Integration adapters
- Dashboard UI

---

## Business Model

### Pricing Strategy

#### Option A: Usage-Based

| Tier | Monthly Base | Included | Overage |
|------|--------------|----------|---------|
| Starter | $99 | 1,000 AI actions | $0.05/action |
| Growth | $299 | 5,000 AI actions | $0.03/action |
| Agency | $799 | 20,000 AI actions | $0.02/action |
| Enterprise | Custom | Unlimited | - |

**AI Action =** One meaningful operation (send SMS, make decision, book appointment)

#### Option B: Per-Seat + Connections

| Tier | Price | Seats | Connections |
|------|-------|-------|-------------|
| Solo | $149/mo | 1 | 3 |
| Team | $399/mo | 5 | 10 |
| Agency | $999/mo | Unlimited | Unlimited |

#### Option C: Revenue Share (Bold)

- Free to use
- 2-5% of revenue generated through platform
- Requires payment integration

**Recommendation:** Start with Option A (usage-based). Simple to understand, scales with value delivered.

### Unit Economics Target

| Metric | Target |
|--------|--------|
| CAC | < $500 |
| LTV | > $5,000 |
| LTV:CAC | > 10:1 |
| Gross Margin | > 70% |
| Payback Period | < 3 months |

### Revenue Projections

| Month | Customers | MRR | ARR |
|-------|-----------|-----|-----|
| 6 | 20 | $6,000 | $72K |
| 12 | 100 | $40,000 | $480K |
| 18 | 300 | $120,000 | $1.44M |
| 24 | 700 | $300,000 | $3.6M |

---

## Go-To-Market Strategy

### Phase 1: Design Partners (Months 1-3)

**Goal:** 5-10 agencies using beta for free

**Approach:**
- Leverage Jeremy's network
- Find agencies struggling with GHL complexity
- Offer free usage in exchange for feedback
- Weekly calls to iterate

**Success Criteria:**
- 3+ agencies using daily
- Net Promoter Score > 50
- Clear "aha moment" identified

### Phase 2: Paid Beta (Months 4-6)

**Goal:** 20-50 paying customers, $10K MRR

**Approach:**
- Convert design partners to paid
- Launch on ProductHunt
- Content marketing (YouTube, Twitter)
- GHL community outreach

**Channels:**
- GHL Facebook groups (50K+ members)
- Agency Twitter/X circles
- YouTube tutorials
- Cold outreach to agencies

### Phase 3: Scale (Months 7-12)

**Goal:** 100+ customers, $50K MRR

**Approach:**
- Paid ads (Meta, Google)
- Affiliate/referral program
- Partnerships (GHL, Twilio)
- Case studies and social proof

---

## Competitive Analysis

### Direct Competitors

| Competitor | Strength | Weakness | Our Advantage |
|------------|----------|----------|---------------|
| **GoHighLevel** | All-in-one, huge user base | Complex, rigid workflows | Natural language, adaptive |
| **Manus Pro** | AI agents, hype | Expensive, general-purpose | Vertical focus, cheaper |
| **Zapier** | Integrations | No AI intelligence | AI-native, not just glue |
| **HubSpot** | Enterprise trust | Expensive, complex | SMB-friendly, modern AI |
| **Clay** | Data enrichment | Point solution | End-to-end execution |

### Positioning

**We are NOT:**
- Another CRM
- A chatbot builder
- An integration platform
- A no-code tool

**We ARE:**
- An AI employee that executes your business processes
- The brain that connects your existing tools
- The future of business automation

### Moat Strategy

1. **Data flywheel** - More usage → better AI → more value → more usage
2. **Integration depth** - Deep GHL/Twilio expertise competitors won't match
3. **Vertical knowledge** - Home services, real estate, coaching playbooks
4. **Cost structure** - Local inference (6x RTX 6000 Ada) = lower margins for competitors

---

## Risk Analysis

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| API access revoked | Medium | High | Multi-platform support, direct integrations |
| AI mistakes damage client | Medium | High | Human-in-loop, approval workflows, insurance |
| Competitor copies | High | Medium | Speed, vertical depth, data moat |
| Regulation (AI/TCPA) | Medium | Medium | Compliance features, audit logs |
| LLM costs increase | Low | Medium | Local inference capability |

---

## Success Metrics

### North Star Metric

**"Actions Executed"** - Total AI actions taken across all customers

Rationale: Directly measures value delivered and scales with revenue.

### Supporting Metrics

| Category | Metric | Target (Month 6) |
|----------|--------|------------------|
| Growth | Customers | 50 |
| Growth | MRR | $15,000 |
| Engagement | DAU/MAU | > 60% |
| Engagement | Actions/customer/day | > 20 |
| Quality | Error rate | < 1% |
| Quality | Customer escalations | < 5% of conversations |
| Satisfaction | NPS | > 50 |
| Retention | Monthly churn | < 5% |

---

## Roadmap Summary

```
2026
│
├── Q1 (Jan-Mar): FOUNDATION
│   ├── Week 1-4: Core architecture, GHL integration
│   ├── Week 5-8: NL interface, basic workflows
│   ├── Week 9-12: Design partner onboarding
│   └── Milestone: 5 agencies in beta
│
├── Q2 (Apr-Jun): EXPANSION
│   ├── Twilio + Calendar integrations
│   ├── Conversational AI for booking
│   ├── Dashboard and monitoring
│   ├── Paid beta launch
│   └── Milestone: $10K MRR, 30 customers
│
├── Q3 (Jul-Sep): INTELLIGENCE
│   ├── Learning and optimization
│   ├── Meta Ads integration
│   ├── Multi-agent architecture
│   └── Milestone: $30K MRR, 75 customers
│
└── Q4 (Oct-Dec): SCALE
    ├── Full ad management
    ├── Enterprise features
    ├── Partner program
    └── Milestone: $80K MRR, 150 customers
```

---

## Open Questions

1. **Build vs. partner for SMS/calling?** 
   - Build own Twilio integration vs. require GHL?
   
2. **Compliance ownership?**
   - Who's liable for TCPA violations - us or customer?
   
3. **White-label for agencies?**
   - Let agencies resell under their brand?

4. **Vertical focus vs. horizontal?**
   - Start with home services only, or go broad?

5. **Pricing model validation?**
   - Need customer interviews on willingness to pay

---

## Appendix

### A. GHL API Capabilities (from audit)

Full schema documentation: `/Users/mac/Desktop/GHL_API_SCHEMA_RESEARCH.md`

### B. Existing Codebase

Location: `/Users/mac/Desktop/agent-orchestrator/`
- 207+ files
- 31 modules
- LangGraph orchestration
- FastAPI backend
- Natural language interface

### C. Hardware Resources

- 6x RTX 6000 Ada GPUs (for local inference)
- Reduces per-query cost by 90%+ vs API-only

### D. Related Documentation

- `/Users/mac/Desktop/GoHighLevel_Research_Report.md`
- `/Users/mac/Desktop/GHL_AUDIT_SYSTEM.md`
- `/Users/mac/Desktop/MASTER_PROJECT_INFO.md`

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Jan 4, 2026 | Jeremy Lerwick | Initial draft |

---

*End of Document*
