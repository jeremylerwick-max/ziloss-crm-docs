# ZILOSS CRM - ARCHITECTURE OVERVIEW

**Created:** January 15, 2026
**Author:** Claude + Jeremy Lerwick
**Purpose:** Complete technical architecture for GHL competitor

---

## COMPETITIVE ANALYSIS

### GoHighLevel Stats
| Metric | Value |
|--------|-------|
| Founded | 2018 |
| Revenue | $82.7M (2024) |
| Customers | 70,000 |
| Employees | 785-1,236 |
| Engineering Team | ~16 developers |
| Tech Stack | Node.js, Vue.js, MongoDB, Message Queues |
| Estimated Codebase | 750K - 1.3M lines |

### Our Approach
| Metric | Ziloss Target |
|--------|---------------|
| Timeline (MVP) | 6-8 weeks |
| Timeline (Competitive) | 4-6 months |
| Claude Code Instances | 10-20 parallel |
| Tech Stack | Python/FastAPI, React/TypeScript, PostgreSQL |

---

## LAYER ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ZILOSS CRM                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   LAYER 4: UI/CLIENT (React + TypeScript)                                   │
│   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│   │  Dashboard  │ │   Inbox     │ │  Contacts   │ │  Calendar   │          │
│   │    Core     │ │    UI       │ │     UI      │ │     UI      │          │
│   │   (M14)     │ │   (M15)     │ │   (M16)     │ │   (M17)     │          │
│   └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘          │
│                                                                              │
│   LAYER 3: API GATEWAY (FastAPI)                                            │
│   ┌──────────────────────────────────────────────────────────────┐          │
│   │                      FastAPI Core (M2)                        │          │
│   │         Auth │ Routing │ Rate Limiting │ Validation           │          │
│   └──────────────────────────────────────────────────────────────┘          │
│                                                                              │
│   LAYER 2: BUSINESS LOGIC                                                   │
│   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│   │  Contact    │ │Conversation │ │    AI       │ │ Appointment │          │
│   │  Service    │ │   Engine    │ │   Engine    │ │   System    │          │
│   │   (M4)      │ │   (M5)      │ │   (M8)      │ │   (M10)     │          │
│   └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘          │
│                                                                              │
│   LAYER 1: DATA & INFRASTRUCTURE                                            │
│   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│   │  Database   │ │    Redis    │ │  Workers    │ │ Observability│         │
│   │   Schema    │ │   Cache     │ │  (M11-13)   │ │    Stack    │          │
│   │   (M1)      │ │             │ │             │ │   (M3)      │          │
│   └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## MODULE INVENTORY

### Built (18,000 lines)
| Module | Status | Lines |
|--------|--------|-------|
| M1: Database Schema | ✅ 80% | 2,500 |
| M2: FastAPI Core | ✅ 70% | 3,000 |
| M4: Contact Service | ✅ 90% | 8,000 |
| M5: Conversation Engine | ✅ 90% | (in M4) |
| M6: Twilio SMS | ✅ 85% | 2,000 |
| M7: Compliance/Timezone | ✅ 90% | 1,500 |
| M9: Outbox | ✅ 80% | 1,000 |

### Week 1 (7,100 lines)
| Module | Files | Lines |
|--------|-------|-------|
| M8: AI Engine | 10 | 2,200 |
| M10: Appointments | 18 | 3,000 |
| M11-13: Workers | 15 | 1,900 |

### Week 2-4 (55,600 lines)
| Module | Files | Lines |
|--------|-------|-------|
| M14-18: Frontend | 119 | 26,400 |
| M19: Workflow Engine | 44 | 7,700 |
| M20-21: White-label/Billing | 20 | 4,000 |
| M22: Campaign Manager | 15 | 2,500 |
| M23-27: Integrations | 30 | 5,000 |
| Testing | 50 | 8,000 |
| Infrastructure | 20 | 2,000 |

---

## MASTER ESTIMATE

| Scope | Lines | Timeline | Parallel Instances |
|-------|-------|----------|-------------------|
| **MVP** | 80K | 6-8 weeks | 8-10 |
| **Competitive** | 200K | 4-6 months | 15-20 |
| **Full Parity** | 500K+ | 12-18 months | Continuous |

---

## TECH STACK

### Backend
- **Language:** Python 3.11+
- **Framework:** FastAPI
- **Database:** PostgreSQL 15 + pgvector
- **Cache:** Redis
- **Queue:** Redis (Bull) or PostgreSQL (SKIP LOCKED)
- **AI:** Claude API (Haiku for speed, Sonnet for quality)

### Frontend
- **Language:** TypeScript
- **Framework:** React 18
- **Build:** Vite
- **Styling:** Tailwind CSS
- **Components:** Shadcn/ui + Radix
- **State:** Zustand
- **Real-time:** WebSocket

### Infrastructure
- **Hosting:** Railway / GCP Cloud Run
- **Database:** Neon / GCP Cloud SQL
- **Monitoring:** Prometheus + Grafana
- **Logging:** Loki
