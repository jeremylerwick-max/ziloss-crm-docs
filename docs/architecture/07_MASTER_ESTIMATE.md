# ZILOSS CRM - MASTER ESTIMATE

**Created:** January 15, 2026
**Last Updated:** January 15, 2026

---

## EXECUTIVE SUMMARY

| Metric | Value |
|--------|-------|
| **MVP Target** | 80,000 lines |
| **Timeline** | 6-8 weeks |
| **Parallel Claude Code Instances** | 8-10 |
| **Tech Stack** | Python/FastAPI + React/TypeScript + PostgreSQL |

---

## COMPETITIVE CONTEXT

### GoHighLevel (The Target)
| Metric | GHL |
|--------|-----|
| Founded | 2018 |
| Revenue | $82.7M (2024) |
| Customers | 70,000 |
| Employees | 785-1,236 |
| Engineering | ~16 developers |
| Codebase | 750K - 1.3M lines |

### Our Advantage
- AI-native from day one (not bolted on)
- Modern stack (not legacy Vue.js)
- Claude Code parallelization (10x speed)
- Real estate vertical focus initially

---

## MODULE BREAKDOWN

### LAYER 0: Foundation (BUILT)
| Module | Status | Lines | Notes |
|--------|--------|-------|-------|
| M1: Database Schema | ✅ 80% | 2,500 | 14 SQL files |
| M2: FastAPI Core | ✅ 70% | 3,000 | Auth, routing, validation |
| M4: Contact Service | ✅ 90% | 8,000 | CRUD, search, import |
| M5: Conversation Engine | ✅ 90% | (in M4) | Threaded messages |
| M6: Twilio SMS | ✅ 85% | 2,000 | Send/receive, webhooks |
| M7: Compliance | ✅ 90% | 1,500 | Opt-out, timezone |
| M9: Outbox | ✅ 80% | 1,000 | Transactional queue |
| **SUBTOTAL** | | **18,000** | |

### WEEK 1: Core Services
| Module | Files | Lines | Time |
|--------|-------|-------|------|
| M8: AI Engine | 10 | 2,200 | 2-3 days |
| M10: Appointments | 18 | 3,000 | 2 days |
| M11: Outbox Worker | 4 | 500 | 0.5 days |
| M12: Stale Monitor | 4 | 600 | 0.5 days |
| M13: Positive Monitor | 4 | 600 | 0.5 days |
| **SUBTOTAL** | **40** | **6,900** | **6-7 days** |

### WEEK 2-3: Frontend
| Module | Files | Lines | Time |
|--------|-------|-------|------|
| M14: Dashboard UI | 5 | 1,500 | 1 day |
| M15: Inbox UI | 6 | 3,000 | 2 days |
| M16: Contacts UI | 7 | 2,500 | 1.5 days |
| M17: Calendar UI | 7 | 2,500 | 1.5 days |
| M18: Settings UI | 5 | 1,500 | 1 day |
| UI Components | 35 | 4,000 | 2 days |
| Layout/Routing | 8 | 1,600 | 0.5 days |
| State/Hooks | 10 | 1,300 | 1 day |
| **SUBTOTAL** | **83** | **17,900** | **10-11 days** |

### WEEK 4-5: Workflow Engine
| Module | Files | Lines | Time |
|--------|-------|-------|------|
| M19: Workflow Backend | 44 | 7,700 | 5-7 days |
| Workflow UI | 25 | 8,000 | 4-5 days |
| **SUBTOTAL** | **69** | **15,700** | **9-12 days** |

### WEEK 6-8: Polish & Scale
| Module | Files | Lines | Time |
|--------|-------|-------|------|
| M20: White-label | 10 | 2,000 | 2 days |
| M21: Billing | 10 | 2,000 | 2 days |
| M22: Campaigns | 15 | 2,500 | 2 days |
| M23-27: Integrations | 30 | 5,000 | 5 days |
| Testing | 50 | 8,000 | Ongoing |
| Infrastructure | 20 | 2,000 | Ongoing |
| **SUBTOTAL** | **135** | **21,500** | **11-14 days** |

---

## GRAND TOTAL

| Category | Files | Lines |
|----------|-------|-------|
| Built | 41 | 18,000 |
| Backend (new) | 98 | 17,100 |
| Frontend (new) | 119 | 26,400 |
| Testing/Infra | 70 | 10,000 |
| Polish/Extras | 55 | 9,500 |
| **TOTAL MVP** | **383** | **81,000** |

---

## TIMELINE

```
Week 1:  ████████░░ M8, M10, M11-13 (Core Services)
Week 2:  ████████░░ M14-16 (Dashboard, Inbox, Contacts)
Week 3:  ████████░░ M17-18, UI Polish
Week 4:  ████████░░ M19 Backend (Workflows)
Week 5:  ████████░░ M19 Frontend (Workflow Builder)
Week 6:  ████████░░ M20-22 (White-label, Billing, Campaigns)
Week 7:  ████████░░ M23-27 (Integrations)
Week 8:  ████████░░ Testing, Bug Fixes, Launch Prep
```

---

## SCALING TIERS

| Tier | Lines | Timeline | Features |
|------|-------|----------|----------|
| **MVP** | 80K | 6-8 weeks | Core CRM, AI, Workflows |
| **Competitive** | 200K | 4-6 months | + Funnels, Forms, Email |
| **Parity** | 500K+ | 12-18 months | + All GHL features |

---

## CLAUDE CODE STRATEGY

### Parallel Instance Allocation
| Instance | Module | Priority |
|----------|--------|----------|
| CC-1 | M8: AI Engine | High |
| CC-2 | M10: Appointments | High |
| CC-3 | M11-13: Workers | Medium |
| CC-4 | M14-15: Dashboard/Inbox | High |
| CC-5 | M16-17: Contacts/Calendar | Medium |
| CC-6 | M19: Workflow Backend | Critical |
| CC-7 | Workflow UI | Critical |
| CC-8 | Testing | Ongoing |

### Prompt Template
```
You are building module [MODULE] for Ziloss CRM.

CONTEXT:
- Tech stack: Python/FastAPI + PostgreSQL
- Related modules: [DEPENDENCIES]
- Architecture doc: [LINK]

TASK:
Build [COMPONENT] with these requirements:
1. [REQUIREMENT 1]
2. [REQUIREMENT 2]
3. [REQUIREMENT 3]

OUTPUT:
- All files in /modules/[module_name]/
- Include tests
- Follow existing patterns from M4 (contacts)
```

---

## RISK FACTORS

| Risk | Mitigation |
|------|------------|
| Workflow engine complexity | Start simple (5 trigger types, 10 actions) |
| Frontend drag-drop | Use ReactFlow (battle-tested) |
| Real-time sync | WebSocket + optimistic updates |
| Multi-tenancy | Location-scoped queries from day 1 |
| AI costs | Haiku default, Sonnet for complex |

---

## SUCCESS METRICS (MVP)

| Metric | Target |
|--------|--------|
| Contacts supported | 100K per location |
| Messages/day | 10K |
| Concurrent users | 100 |
| Response time (P95) | < 500ms |
| AI classification accuracy | > 90% |
| Uptime | 99.5% |

---

## NEXT STEPS

1. ✅ Architecture docs complete
2. ⏳ Await ChatGPT/Gemini research on GHL
3. ⏳ Generate Claude Code prompts for Week 1
4. ⏳ Start M8 (AI Engine) and M10 (Appointments) in parallel
5. ⏳ Set up CI/CD pipeline
