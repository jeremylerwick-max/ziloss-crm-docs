# ZILOSS CRM - Master Build Plan

> **One document to rule them all.** Any Claude instance reading this should understand the full project context.

## 🎯 What We're Building

**Ziloss CRM** - An AI-native CRM that competes with GoHighLevel (GHL) but with a 10x differentiator:

### The Autonomous Growth Operator (AGO)
```
"Set outcomes. The system plans, executes, verifies, and improves—without brittle workflows."
```

**What users define:**
```yaml
goal:
  target: 40 appointments/month
  max_cpa: $110
  quality: "homeowners, 600+ credit"
```

**What AGO does automatically:**
- Plans campaigns across channels
- Executes with compliance gates
- Measures against targets
- Self-corrects without human intervention

**Why this wins:** GHL requires humans to build/maintain workflows. AGO operates autonomously toward business outcomes.

---

## 📁 Project Location

```
/Users/mac/Desktop/agent-orchestrator/
```

**Owner:** Jeremy Boler (Ziloss Technologies, Taylorsville, Utah)

---

## 📚 Documentation Map

### Architecture Documents
Location: `/docs/architecture/`

| File | Purpose | Lines |
|------|---------|-------|
| `01_OVERVIEW.md` | System architecture, module map, data flow | ~400 |
| `02_MODULE_M8_AI_ENGINE.md` | Intent classification, sentiment, response gen | ~350 |
| `03_MODULE_M10_APPOINTMENTS.md` | Booking system, availability, reminders | ~400 |
| `04_MODULE_M11_13_WORKERS.md` | Background workers, outbox, retry logic | ~350 |
| `05_MODULE_M19_WORKFLOW_ENGINE.md` | Visual workflow builder, triggers | ~300 |
| `06_FRONTEND_STRUCTURE.md` | React dashboard, components | ~250 |
| `07_MASTER_ESTIMATE.md` | Full timeline, 25K lines, 8-10 weeks | ~200 |
| `08_GHL_COMPETITIVE_ANALYSIS.md` | Gemini research on GHL weaknesses | ~300 |
| `09_TECHNICAL_ARCHITECTURE_COMPARISON.md` | ChatGPT deep dive on architecture | ~350 |
| `10_MVP_SPECIFICATION.md` | Week 1-2 MVP scope | ~250 |
| `11_AGO_FLAGSHIP_FEATURE.md` | AGO concept and value prop | ~200 |
| `11_AUTONOMOUS_GROWTH_OPERATOR.md` | Full AGO specification | ~400 |
| `12_AGO_DATA_MODEL.md` | Goals, plans, executions schema | ~300 |
| `13_AGO_EXECUTION_ARCHITECTURE.md` | How AGO runs autonomously | ~350 |

### Claude Code Prompts
Location: `/docs/prompts/`

| File | Purpose | Lines | Est. Days |
|------|---------|-------|-----------|
| `README.md` | Index and execution guide | ~165 | - |
| `CC_PROMPT_EVENT_LEDGER.md` | **FOUNDATION** - Append-only event log | ~677 | 1 |
| `CC_PROMPT_M8_AI_ENGINE.md` | Intent/Sentiment/Response generation | ~196 | 2-3 |
| `CC_PROMPT_M10_APPOINTMENTS.md` | Booking system | ~634 | 2 |
| `CC_PROMPT_M11_13_WORKERS.md` | Background workers | ~1011 | 2 |

---

## 🔨 Build Execution Order

### Phase 1: Foundation (Day 1)
```
CC_PROMPT_EVENT_LEDGER.md
└── Creates: src/core/events/
└── Why first: Everything else writes to this log
└── Key: Append-only, checksums, replay capability
```

### Phase 2: Core Modules (Days 2-4, can run in parallel)
```
CC_PROMPT_M8_AI_ENGINE.md        CC_PROMPT_M10_APPOINTMENTS.md
└── src/modules/ai_engine/       └── src/modules/appointments/
└── Depends on: Event Ledger     └── Depends on: Event Ledger
└── 17 intent types              └── Slot calculation
└── Claude Haiku/Sonnet          └── Google Calendar sync
```

### Phase 3: Workers (Days 5-7, after AI Engine)
```
CC_PROMPT_M11_13_WORKERS.md
└── Creates: src/workers/
└── Depends on: Event Ledger, AI Engine
└── Outbox processor, stale monitor, hot lead detector
```

### Phase 4: AGO (Week 2+)
```
Build on top of all the above
└── Goal setting UI
└── Autonomous planning
└── Self-correction loops
```

---

## 🗄️ Database Schema Summary

### Event Ledger (Foundation)
```sql
CREATE TABLE events (
    event_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    location_id UUID NOT NULL,
    lead_id UUID,
    actor VARCHAR(20),  -- system/agent/rep/lead/external/workflow
    event_type VARCHAR(100),
    payload JSONB,
    correlation_id UUID,
    causation_id UUID,
    sequence_number BIGSERIAL,
    checksum VARCHAR(64),
    occurred_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ
);
-- CRITICAL: Trigger prevents UPDATE/DELETE (append-only)
```

### Key Tables
- `leads` - Contact records with lifecycle state
- `conversations` - Message threads
- `appointments` - Bookings with EXCLUDE constraint
- `message_outbox` - Transactional outbox pattern
- `goals` / `plans` / `executions` - AGO tables

---

## 🔑 Key Technical Decisions

### 1. Event Sourcing (Event Ledger)
**Why:** Enables AGO to replay decisions, audit everything, debug AI behavior
**GHL doesn't have this** - gives us architectural advantage

### 2. Transactional Outbox Pattern
**Why:** Guaranteed message delivery without distributed transactions
```
BEGIN TRANSACTION
  INSERT INTO leads...
  INSERT INTO message_outbox...
COMMIT
-- Worker picks up from outbox
```

### 3. Consent as Hard Gate
**Why:** TCPA compliance, legal protection
```python
if not lead.has_consent:
    raise ConsentRequiredError()  # Never skip this
```

### 4. Claude Haiku for Speed, Sonnet for Quality
**Why:** Cost optimization + quality balance
- Haiku: Intent classification, quick responses
- Sonnet: Complex reasoning, goal planning

---

## 🚀 How to Continue Building

### Start Event Ledger (First Priority)
```bash
cd /Users/mac/Desktop/agent-orchestrator
claude "Build Event Ledger following docs/prompts/CC_PROMPT_EVENT_LEDGER.md"
```

### Start Any Module
```bash
claude "Build [MODULE] following docs/prompts/CC_PROMPT_[MODULE].md"
```

### Error Correction
```bash
claude "Read ZILOSS_BUILD_PLAN.md for context. Then fix [describe issue]"
```

### Continue From Anywhere (Slack, Web, etc.)
```
Read /Users/mac/Desktop/agent-orchestrator/ZILOSS_BUILD_PLAN.md for full context.
The project is Ziloss CRM with AGO (Autonomous Growth Operator).
Current status: [describe where you are]
```

---

## 📊 Current Project Status

### Completed ✅
- [x] Full architecture documentation (14 docs, ~4,700 lines)
- [x] Claude Code prompts (5 prompts, ~2,683 lines)
- [x] Directory structure for Event Ledger
- [x] Competitive analysis (GHL weaknesses identified)
- [x] AGO specification complete

### In Progress 🔨
- [ ] Event Ledger module (NEXT UP)
- [ ] AI Engine module
- [ ] Appointments module
- [ ] Workers module

### Planned 📋
- [ ] Frontend dashboard
- [ ] AGO goal-setting UI
- [ ] Multi-channel campaigns
- [ ] Full GHL feature parity

---

## 🧠 Memory References

Key information is stored in Claude's memory system. Search for:
- "Ziloss" - Project overview
- "AGO" - Autonomous Growth Operator
- "Event Ledger" - Foundation module
- "GHL" - Competitor analysis

---

## 📞 Quick Reference

**Project:** Ziloss CRM  
**Owner:** Jeremy Boler  
**Location:** `/Users/mac/Desktop/agent-orchestrator/`  
**Tech Stack:** Python, FastAPI, PostgreSQL, Redis, Claude AI  
**Target:** Beat GoHighLevel with AI-native architecture  
**10x Feature:** Autonomous Growth Operator (AGO)

---

## 🎯 The One Thing to Remember

> **Event Ledger is the foundation.** Build it first. Everything else (AI decisions, appointments, workers, AGO) writes to and reads from this append-only log. If you get this right, GHL can't catch you without a rewrite.

---

*Last Updated: January 15, 2025*
*Document Version: 1.0*

---

## 📦 DETAILED MODULE SPECIFICATIONS

### M8: AI Engine (`src/modules/ai_engine/`)

**Purpose:** All AI-powered text processing

**Components:**
```
ai_engine/
├── intent_classifier.py    # 17 intent types
├── sentiment_analyzer.py   # -1.0 to +1.0 scoring
├── response_generator.py   # Claude integration
├── prompt_templates.py     # Versioned prompts
└── rate_limiter.py         # Token bucket per tenant
```

**Intent Types:**
```python
class IntentType(Enum):
    BOOKING_REQUEST = "booking_request"      # "Can I schedule..."
    RESCHEDULE = "reschedule"                # "Need to move my..."
    CANCELLATION = "cancellation"            # "Cancel my..."
    PRICING_INQUIRY = "pricing_inquiry"      # "How much..."
    SERVICE_QUESTION = "service_question"    # "Do you offer..."
    AVAILABILITY_CHECK = "availability"      # "Are you open..."
    CONFIRMATION = "confirmation"            # "Yes", "Sounds good"
    REJECTION = "rejection"                  # "No thanks"
    COMPLAINT = "complaint"                  # Negative sentiment
    COMPLIMENT = "compliment"                # Positive feedback
    STOP_REQUEST = "stop_request"            # "STOP", "Unsubscribe"
    HUMAN_REQUEST = "human_request"          # "Talk to person"
    UNCLEAR = "unclear"                      # Can't determine
    GREETING = "greeting"                    # "Hi", "Hello"
    GOODBYE = "goodbye"                      # "Thanks bye"
    LOCATION_INQUIRY = "location"            # "Where are you"
    HOURS_INQUIRY = "hours"                  # "What time..."
```

**Rate Limiting:**
```python
# Token bucket per tenant
LIMITS = {
    "free": {"rpm": 10, "tpm": 10_000},
    "pro": {"rpm": 60, "tpm": 100_000},
    "enterprise": {"rpm": 300, "tpm": 1_000_000}
}
```

---

### M10: Appointments (`src/modules/appointments/`)

**Purpose:** Complete booking system

**Components:**
```
appointments/
├── availability.py      # Slot calculation
├── booking.py           # Create/modify bookings
├── calendar_sync.py     # Google Calendar integration
├── reminders.py         # SMS/email reminders
└── assignment.py        # Round-robin rep assignment
```

**Key Features:**
1. **Double-booking prevention** (PostgreSQL EXCLUDE constraint)
2. **Buffer time** between appointments
3. **Timezone handling** (all UTC internally)
4. **Reminder sequences** (24h, 2h, 30min)

**Database:**
```sql
CREATE TABLE appointments (
    appointment_id UUID PRIMARY KEY,
    lead_id UUID NOT NULL,
    rep_id UUID,
    calendar_id UUID,
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    status VARCHAR(20),  -- pending/confirmed/completed/no_show/cancelled
    
    -- Prevent overlapping bookings
    EXCLUDE USING gist (
        calendar_id WITH =,
        tstzrange(start_time, end_time) WITH &&
    ) WHERE (status NOT IN ('cancelled'))
);
```

---

### M11-13: Workers (`src/workers/`)

**Purpose:** Background processing with reliability

**Components:**
```
workers/
├── outbox_processor.py      # Send queued messages
├── stale_monitor.py         # Detect stale conversations
├── hot_lead_detector.py     # Identify high-intent leads
├── reminder_scheduler.py    # Appointment reminders
├── retry_handler.py         # Exponential backoff
└── dead_letter.py           # Failed message handling
```

**Outbox Pattern:**
```python
# Guarantees: Either both succeed or neither
async with db.transaction():
    await db.execute("INSERT INTO leads...")
    await db.execute("INSERT INTO message_outbox...")

# Worker picks up separately
async def process_outbox():
    rows = await db.fetch("""
        SELECT * FROM message_outbox
        WHERE status = 'pending'
        FOR UPDATE SKIP LOCKED
        LIMIT 100
    """)
    for row in rows:
        await send_message(row)
        await mark_sent(row.id)
```

**Retry Logic:**
```python
RETRY_DELAYS = [60, 300, 900, 3600, 14400]  # 1m, 5m, 15m, 1h, 4h

async def retry_with_backoff(message_id: UUID, attempt: int):
    if attempt >= len(RETRY_DELAYS):
        await move_to_dead_letter(message_id)
        return
    
    delay = RETRY_DELAYS[attempt]
    await schedule_retry(message_id, delay, attempt + 1)
```

---

### M19: Workflow Engine (`src/modules/workflow_engine/`)

**Purpose:** Visual automation builder (GHL competitor feature)

**Components:**
```
workflow_engine/
├── triggers.py          # Event-based triggers
├── conditions.py        # If/then logic
├── actions.py           # What to do
├── executor.py          # Run workflows
└── visual_builder.py    # React component data
```

**Trigger Types:**
```python
class TriggerType(Enum):
    LEAD_CREATED = "lead.created"
    FORM_SUBMITTED = "form.submitted"
    TAG_ADDED = "tag.added"
    APPOINTMENT_BOOKED = "appointment.booked"
    SMS_RECEIVED = "sms.received"
    EMAIL_OPENED = "email.opened"
    CUSTOM_EVENT = "custom.event"
    SCHEDULED = "scheduled"
```

---

## 🤖 AGO: Autonomous Growth Operator (Full Spec)

### What Makes AGO Different

**Traditional CRM (GHL):**
```
Human defines workflow → System executes exactly → Human monitors → Human adjusts
```

**AGO:**
```
Human defines outcome → AGO plans strategy → AGO executes → AGO measures → AGO adjusts
```

### AGO Data Model

```sql
-- What the user wants
CREATE TABLE ago_goals (
    goal_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    location_id UUID NOT NULL,
    
    -- The target
    goal_type VARCHAR(50),        -- appointments, leads, revenue
    target_value NUMERIC,         -- 40
    target_unit VARCHAR(20),      -- per_month
    
    -- Constraints
    max_cpa NUMERIC,              -- $110
    quality_criteria JSONB,       -- {"credit_score": "600+", "homeowner": true}
    
    -- Channels allowed
    channels_enabled JSONB,       -- ["sms", "email", "voice"]
    
    -- Time bounds
    start_date DATE,
    end_date DATE,
    
    status VARCHAR(20),           -- active, paused, completed
    created_at TIMESTAMPTZ
);

-- How AGO will achieve it
CREATE TABLE ago_plans (
    plan_id UUID PRIMARY KEY,
    goal_id UUID REFERENCES ago_goals,
    
    -- The strategy
    channel_allocation JSONB,     -- {"sms": 0.6, "email": 0.3, "voice": 0.1}
    daily_budget NUMERIC,
    daily_volume_target INT,
    
    -- Messaging strategy
    message_templates JSONB,
    follow_up_cadence JSONB,      -- [1, 3, 7, 14] days
    
    -- AI parameters
    response_style VARCHAR(20),   -- professional, casual, urgent
    escalation_threshold NUMERIC, -- When to involve human
    
    version INT,
    reasoning TEXT,               -- Why AGO chose this plan
    created_at TIMESTAMPTZ
);

-- What AGO actually did
CREATE TABLE ago_executions (
    execution_id UUID PRIMARY KEY,
    plan_id UUID REFERENCES ago_plans,
    goal_id UUID REFERENCES ago_goals,
    
    -- Period
    period_start TIMESTAMPTZ,
    period_end TIMESTAMPTZ,
    
    -- Results
    leads_contacted INT,
    responses_received INT,
    appointments_booked INT,
    cost_incurred NUMERIC,
    
    -- Calculated
    cpa_achieved NUMERIC,
    conversion_rate NUMERIC,
    
    -- AGO's assessment
    performance_score NUMERIC,    -- 0-100
    learnings JSONB,              -- What AGO learned
    adjustments_made JSONB,       -- What AGO changed
    
    created_at TIMESTAMPTZ
);
```

### AGO Execution Loop

```python
async def ago_daily_loop():
    """Runs every day at midnight for each active goal."""
    
    for goal in await get_active_goals():
        # 1. Measure yesterday
        metrics = await measure_performance(goal)
        
        # 2. Compare to target
        gap = goal.target_value - metrics.achieved
        
        # 3. Decide if plan needs adjustment
        if should_adjust_plan(metrics, goal):
            new_plan = await generate_new_plan(goal, metrics)
            await save_plan(new_plan)
            
            # Log the decision
            await event_ledger.append(EventCreate(
                event_type="agent.planned",
                payload={
                    "goal_id": str(goal.goal_id),
                    "old_plan": current_plan.dict(),
                    "new_plan": new_plan.dict(),
                    "reasoning": new_plan.reasoning,
                    "metrics_that_triggered": metrics.dict()
                }
            ))
        
        # 4. Execute today's actions
        await execute_daily_plan(goal)

async def should_adjust_plan(metrics: Metrics, goal: Goal) -> bool:
    """AGO decides if it needs to change strategy."""
    
    # Below target pace?
    if metrics.run_rate < goal.target_value * 0.8:
        return True
    
    # CPA too high?
    if metrics.cpa > goal.max_cpa * 1.2:
        return True
    
    # Conversion rate dropping?
    if metrics.conversion_trend < -0.1:
        return True
    
    return False

async def generate_new_plan(goal: Goal, metrics: Metrics) -> Plan:
    """Use Claude to generate a new strategy."""
    
    prompt = f"""
    Goal: {goal.target_value} {goal.goal_type} per {goal.target_unit}
    Max CPA: ${goal.max_cpa}
    
    Current Performance:
    - Achieved: {metrics.achieved}
    - CPA: ${metrics.cpa}
    - Best channel: {metrics.best_channel}
    - Worst channel: {metrics.worst_channel}
    
    Generate a new plan that:
    1. Shifts budget toward {metrics.best_channel}
    2. Reduces spend on {metrics.worst_channel}
    3. Stays within CPA constraint
    4. Explains reasoning
    
    Return JSON with: channel_allocation, daily_budget, reasoning
    """
    
    response = await claude.complete(prompt)
    return Plan(**response)
```

### AGO Guardrails

```python
class AGOGuardrails:
    """Hard limits AGO cannot exceed, even if it thinks it should."""
    
    # Budget limits
    MAX_DAILY_SPEND = 500  # Never spend more than this per day
    MAX_MESSAGES_PER_LEAD = 10  # Don't spam
    
    # Timing limits
    QUIET_HOURS_START = 21  # 9 PM
    QUIET_HOURS_END = 8     # 8 AM
    
    # Consent is absolute
    async def check_consent(self, lead_id: UUID) -> bool:
        """NEVER skip this check. Legal requirement."""
        consent = await get_consent_status(lead_id)
        if not consent.sms_allowed:
            raise ConsentRequired("Cannot message without consent")
        return True
    
    # Human escalation
    async def check_escalation(self, lead_id: UUID) -> bool:
        """Some situations require human intervention."""
        # Complaint detected
        # Legal language detected
        # High-value opportunity
        # Lead explicitly requested human
        pass
```

---

## 🔗 Integration Points

### External Services

| Service | Purpose | Credentials Location |
|---------|---------|---------------------|
| Claude API | AI responses | `.env` ANTHROPIC_API_KEY |
| Twilio | SMS/Voice | `.env` TWILIO_* |
| Google Calendar | Appointment sync | OAuth in `secrets/` |
| Stripe | Billing | `.env` STRIPE_* |
| PostgreSQL | Primary database | `.env` DATABASE_URL |
| Redis | Caching, pub/sub | `.env` REDIS_URL |

### GHL Migration Path

For users migrating from GHL:
```python
# Import their data
async def import_from_ghl(api_key: str, location_id: str):
    ghl = GHLClient(api_key)
    
    # Import contacts
    contacts = await ghl.get_contacts(location_id)
    for contact in contacts:
        await create_lead_from_ghl(contact)
    
    # Import conversations
    conversations = await ghl.get_conversations(location_id)
    for conv in conversations:
        await import_conversation(conv)
    
    # Note: Workflows cannot be directly imported
    # AGO replaces them with outcome-based goals
```

---

## 🧪 Testing Strategy

### Unit Tests
```bash
pytest tests/unit/ -v
```

### Integration Tests
```bash
pytest tests/integration/ -v --db-url=$TEST_DATABASE_URL
```

### E2E Tests
```bash
pytest tests/e2e/ -v
```

### Coverage Target
- Event Ledger: >95% (critical path)
- AI Engine: >90%
- Appointments: >90%
- Workers: >85%
- Overall: >85%

---

## 🚨 Error Handling Patterns

### Retry with Backoff
```python
@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=4, max=60)
)
async def send_sms(to: str, body: str):
    return await twilio.messages.create(to=to, body=body)
```

### Dead Letter Queue
```python
async def handle_permanent_failure(message_id: UUID, error: str):
    await db.execute("""
        INSERT INTO dead_letter_queue (message_id, error, created_at)
        VALUES ($1, $2, NOW())
    """, message_id, error)
    
    # Alert operations team
    await notify_ops(f"Message {message_id} permanently failed: {error}")
```

### Circuit Breaker
```python
circuit_breaker = CircuitBreaker(
    failure_threshold=5,
    recovery_timeout=60,
    expected_exception=TwilioException
)

@circuit_breaker
async def send_with_circuit_breaker(message):
    return await twilio.send(message)
```

---

## 📈 Monitoring & Observability

### Key Metrics to Track
```python
# Business metrics
appointments_booked_total = Counter("appointments_booked_total")
messages_sent_total = Counter("messages_sent_total", ["channel"])
ai_response_time = Histogram("ai_response_seconds")

# AGO metrics
ago_goal_progress = Gauge("ago_goal_progress", ["goal_id"])
ago_cpa_current = Gauge("ago_cpa_current", ["goal_id"])
ago_plan_adjustments = Counter("ago_plan_adjustments_total")
```

### Dashboards
- **Operations:** Message queues, error rates, latency
- **Business:** Appointments, conversion rates, revenue
- **AGO:** Goal progress, CPA trends, autonomous decisions

---

## 🔐 Security Checklist

- [ ] All secrets in environment variables
- [ ] Database connections use SSL
- [ ] API endpoints require authentication
- [ ] Rate limiting on all public endpoints
- [ ] Audit logging for sensitive operations
- [ ] TCPA consent verified before every message
- [ ] PII encrypted at rest
- [ ] Regular security scans

---


## 🎮 Quick Commands Reference

### For Claude Code (Terminal)
```bash
# Start Event Ledger build
cd /Users/mac/Desktop/agent-orchestrator
claude "Build Event Ledger following docs/prompts/CC_PROMPT_EVENT_LEDGER.md"

# Start AI Engine build (after Event Ledger)
claude "Build AI Engine following docs/prompts/CC_PROMPT_M8_AI_ENGINE.md"

# Start Appointments build (after Event Ledger)
claude "Build Appointments following docs/prompts/CC_PROMPT_M10_APPOINTMENTS.md"

# Start Workers build (after Event Ledger + AI Engine)
claude "Build Workers following docs/prompts/CC_PROMPT_M11_13_WORKERS.md"

# Check build status
cat /Users/mac/Desktop/agent-orchestrator/ZILOSS_BUILD_PLAN.md

# Run tests
cd /Users/mac/Desktop/agent-orchestrator && pytest tests/ -v
```

### For Claude (Slack/Web/Any)
```
Context: Read /Users/mac/Desktop/agent-orchestrator/ZILOSS_BUILD_PLAN.md

This is the Ziloss CRM project with AGO (Autonomous Growth Operator).
Location: /Users/mac/Desktop/agent-orchestrator/

Architecture docs: /docs/architecture/
Claude Code prompts: /docs/prompts/
Source code: /src/

Current task: [describe what you need]
```

### Error Correction Template
```
I'm working on Ziloss CRM (AGO project).
Repo: /Users/mac/Desktop/agent-orchestrator/

Read ZILOSS_BUILD_PLAN.md for full context.

The issue is: [describe problem]
File affected: [path]
Error message: [paste error]

Please fix this.
```

---

## 📋 Build Checklist

### Day 1: Event Ledger ⬜
- [ ] `src/core/events/models.py` - Event types + Pydantic models
- [ ] `src/core/events/ledger.py` - Append-only ledger class
- [ ] `src/core/events/publisher.py` - Event publishing
- [ ] `src/core/events/replay.py` - Debug replay
- [ ] `src/core/events/projections.py` - State from events
- [ ] `schema/020_event_ledger.sql` - Database migration
- [ ] Tests pass with >90% coverage

### Days 2-3: AI Engine ⬜
- [ ] `src/modules/ai_engine/intent_classifier.py`
- [ ] `src/modules/ai_engine/sentiment_analyzer.py`
- [ ] `src/modules/ai_engine/response_generator.py`
- [ ] `src/modules/ai_engine/rate_limiter.py`
- [ ] Integration with Event Ledger
- [ ] Tests pass

### Days 2-3: Appointments ⬜
- [ ] `src/modules/appointments/availability.py`
- [ ] `src/modules/appointments/booking.py`
- [ ] `src/modules/appointments/reminders.py`
- [ ] `src/modules/appointments/calendar_sync.py`
- [ ] Database with EXCLUDE constraint
- [ ] Tests pass

### Days 4-5: Workers ⬜
- [ ] `src/workers/outbox_processor.py`
- [ ] `src/workers/stale_monitor.py`
- [ ] `src/workers/hot_lead_detector.py`
- [ ] `src/workers/retry_handler.py`
- [ ] Tests pass

### Week 2: AGO Foundation ⬜
- [ ] Goal setting API
- [ ] Plan generation
- [ ] Execution loop
- [ ] Self-correction logic

---

## 📝 Session Handoff Template

When ending a session, update this section:

```
### Last Session: [DATE]

**Completed:**
- [What was built]

**In Progress:**
- [What's partially done]

**Next Steps:**
1. [Immediate next task]
2. [Following task]

**Issues/Blockers:**
- [Any problems encountered]

**Files Modified:**
- [List of files changed]
```

---

## 🎯 Success Metrics

### Technical
- [ ] Event Ledger processes 10K events/second
- [ ] AI response time <500ms p95
- [ ] Message delivery >99.9%
- [ ] Zero data loss (append-only guarantee)

### Business
- [ ] AGO achieves target within 10% accuracy
- [ ] CPA stays within constraint
- [ ] User can set goal in <1 minute
- [ ] Migration from GHL in <1 hour

---

## 🏁 Definition of Done

### For Each Module
1. Code complete with types
2. Unit tests >85% coverage
3. Integration tests passing
4. Documentation updated
5. Event Ledger integration working
6. Error handling complete
7. Logging in place

### For AGO
1. User can set a goal
2. System generates a plan
3. System executes the plan
4. System measures results
5. System adjusts automatically
6. Human can override at any point

---

## 💡 Key Insights for Any Claude

1. **Event Ledger is sacred** - Never skip it. Everything flows through it.

2. **Consent is non-negotiable** - TCPA compliance. Check before every message.

3. **AGO is the differentiator** - This is what makes Ziloss beat GHL.

4. **Append-only enables everything** - Replay, audit, debugging, self-correction.

5. **Start simple, iterate** - MVP first, then enhance.

---

## 📞 Contact

**Project Owner:** Jeremy Boler
**Company:** Ziloss Technologies
**Location:** Taylorsville, Utah

---

*This document is the single source of truth for the Ziloss CRM build.*
*Any Claude instance should read this first before working on the project.*

---

**END OF ZILOSS BUILD PLAN**
