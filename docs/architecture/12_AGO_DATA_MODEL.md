# AGO Data Model - The Foundation GHL Can't Copy

**Source:** ChatGPT Deep Research (January 2026)
**Priority:** CRITICAL - Get this right, GHL can't catch you without a rewrite

---

## Why This Data Model Matters

> "If you get this right, GHL can't catch you without a rewrite."

Traditional CRMs use:
- Mutable records
- Tag-based states
- Chat history logs

AI-native CRM needs:
- Append-only event ledger
- Deterministic state machine
- Structured memory graph

---

## 1. Canonical Event Ledger (MANDATORY)

**Rule:** Append-only. No mutations. Ever.

```sql
CREATE TABLE events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    tenant_id UUID NOT NULL,
    location_id UUID NOT NULL,
    lead_id UUID NOT NULL,
    
    -- Who did this
    actor VARCHAR(20) NOT NULL 
        CHECK (actor IN ('system', 'agent', 'rep', 'external', 'lead')),
    
    -- What happened
    event_type VARCHAR(100) NOT NULL,  -- 'lead.created', 'sms.sent', 'reply.received'
    payload JSONB NOT NULL DEFAULT '{}',
    
    -- Traceability
    correlation_id UUID,      -- Ties events to a plan/execution
    causation_id UUID,        -- What event caused this one
    source VARCHAR(50),       -- 'twilio', 'mailgun', 'ui', 'agent', 'webhook'
    
    -- Integrity
    checksum VARCHAR(64),     -- For replay integrity
    
    -- Indexing
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Critical indexes
CREATE INDEX idx_events_lead ON events(lead_id, occurred_at DESC);
CREATE INDEX idx_events_location ON events(location_id, occurred_at DESC);
CREATE INDEX idx_events_type ON events(event_type, occurred_at DESC);
CREATE INDEX idx_events_correlation ON events(correlation_id);
```

### Why This Matters

| Capability | Enabled By Event Ledger |
|------------|-------------------------|
| Deterministic replay | ✅ Rebuild any state |
| Auditability | ✅ Full history |
| Debugging AI decisions | ✅ See what AI saw |
| Legal defensibility | ✅ Prove compliance |
| Self-healing | ✅ Detect anomalies |

**GHL does not have this as a first-class primitive.**

### Event Types

```
-- Lead lifecycle
lead.created
lead.enriched
lead.qualified
lead.assigned
lead.state_changed

-- Communication
sms.queued
sms.sent
sms.delivered
sms.failed
sms.received
email.queued
email.sent
email.opened
email.clicked
email.bounced
email.received
call.initiated
call.answered
call.voicemail
call.completed

-- Appointments
appointment.requested
appointment.booked
appointment.confirmed
appointment.reminded
appointment.completed
appointment.no_show
appointment.cancelled
appointment.rescheduled

-- Agent actions
agent.planned
agent.decided
agent.acted
agent.learned

-- Compliance
consent.granted
consent.revoked
dnd.enabled
dnd.expired
```

---

## 2. Lead State Model (NOT Tags)

**Rule:** No tags. No triggers. No spaghetti. State transitions are validated, not assumed.

```sql
CREATE TABLE lead_state (
    lead_id UUID PRIMARY KEY REFERENCES contacts(id),
    location_id UUID NOT NULL,
    
    -- Lifecycle (finite state machine)
    lifecycle VARCHAR(20) NOT NULL DEFAULT 'new'
        CHECK (lifecycle IN (
            'new',           -- Just created
            'contacting',    -- Outreach in progress
            'engaged',       -- Has responded
            'qualified',     -- Meets criteria
            'booked',        -- Appointment scheduled
            'no_show',       -- Missed appointment
            'won',           -- Converted
            'lost',          -- Did not convert
            'nurture',       -- Long-term follow-up
            'dnc'            -- Do not contact
        )),
    
    -- AI scoring
    intent_score FLOAT DEFAULT 0.0,       -- 0-1, likelihood to book
    qualification_score FLOAT DEFAULT 0.0, -- 0-1, fit for service
    urgency_score FLOAT DEFAULT 0.0,       -- 0-1, time sensitivity
    
    -- Timing
    last_touch_at TIMESTAMPTZ,
    next_action_at TIMESTAMPTZ,
    last_response_at TIMESTAMPTZ,
    
    -- Assignment
    owner_rep_id UUID REFERENCES users(id),
    
    -- AI confidence
    confidence FLOAT DEFAULT 0.0,
    
    -- Tracking
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_lead_state_lifecycle ON lead_state(location_id, lifecycle);
CREATE INDEX idx_lead_state_next_action ON lead_state(next_action_at) 
    WHERE next_action_at IS NOT NULL;
```

### State Transition Rules

```yaml
# Validated transitions (not arbitrary tag changes)
transitions:
  new:
    - contacting  # Started outreach
    - dnc         # Opted out immediately
    
  contacting:
    - engaged     # Responded
    - nurture     # No response after attempts
    - dnc         # Opted out
    
  engaged:
    - qualified   # Meets criteria
    - nurture     # Not ready
    - dnc         # Opted out
    
  qualified:
    - booked      # Appointment set
    - nurture     # Changed mind
    - lost        # Declined
    - dnc         # Opted out
    
  booked:
    - won         # Showed + converted
    - no_show     # Missed appointment
    - lost        # Showed but declined
    
  no_show:
    - booked      # Rescheduled
    - nurture     # Too many no-shows
    - lost        # Gave up
    - dnc         # Opted out
```

---

## 3. Consent & Compliance Model (Hard Gate)

**Rule:** No action executes unless consent is valid at execution time.

```sql
CREATE TABLE consent (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lead_id UUID NOT NULL REFERENCES contacts(id),
    location_id UUID NOT NULL,
    
    -- Channel-specific consent
    channel VARCHAR(20) NOT NULL 
        CHECK (channel IN ('sms', 'email', 'voice', 'whatsapp')),
    
    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'unknown'
        CHECK (status IN ('opted_in', 'opted_out', 'unknown', 'pending')),
    
    -- Legal basis
    legal_basis VARCHAR(20) 
        CHECK (legal_basis IN ('express', 'transactional', 'legitimate_interest')),
    
    -- Provenance (for TCPA compliance)
    source VARCHAR(100),           -- 'web_form', 'sms_keyword', 'import'
    source_url TEXT,               -- Where they opted in
    source_ip VARCHAR(45),         -- IP address
    source_timestamp TIMESTAMPTZ,  -- When they opted in
    
    -- Audit
    granted_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ,
    revocation_reason TEXT,
    
    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    UNIQUE(lead_id, channel)
);

CREATE INDEX idx_consent_lead ON consent(lead_id);
CREATE INDEX idx_consent_status ON consent(location_id, channel, status);
```

### Consent Invariant (Enforced in Code)

```python
async def check_consent(lead_id: str, channel: str) -> bool:
    """
    INVARIANT: No message sent without valid consent.
    This is a HARD GATE, not a soft check.
    """
    consent = await get_consent(lead_id, channel)
    
    if consent is None:
        return False
    
    if consent.status != 'opted_in':
        return False
    
    if consent.revoked_at is not None:
        return False
    
    # Check quiet hours
    if is_quiet_hours(lead_id):
        return False
    
    return True
```

---

## 4. Agent Memory (Structured, Not Chat Logs)

**Rule:** This is how the AI remembers without hallucinating.

```sql
CREATE TABLE agent_memory (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Entity (lead, household, or account)
    entity_type VARCHAR(20) NOT NULL 
        CHECK (entity_type IN ('lead', 'household', 'account')),
    entity_id UUID NOT NULL,
    location_id UUID NOT NULL,
    
    -- Memory key-value
    key VARCHAR(100) NOT NULL,    -- 'objection.price', 'preferred_time', 'sentiment'
    value JSONB NOT NULL,
    
    -- Confidence and source
    confidence FLOAT DEFAULT 1.0,
    source VARCHAR(50),           -- 'extraction', 'inference', 'user_input'
    source_event_id UUID REFERENCES events(event_id),
    
    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ,       -- Some memories decay
    
    UNIQUE(entity_type, entity_id, key)
);

CREATE INDEX idx_memory_entity ON agent_memory(entity_type, entity_id);
CREATE INDEX idx_memory_key ON agent_memory(key);
```

### Memory Keys (Standardized)

```yaml
# Lead-level memory
lead:
  - first_name
  - preferred_name
  - preferred_channel       # sms, email, call
  - preferred_time          # morning, afternoon, evening
  - timezone
  - language
  - sentiment               # positive, neutral, negative, frustrated
  - objection.price
  - objection.timing
  - objection.trust
  - budget_range
  - decision_timeline
  - competitor_mentioned
  - best_call_time
  - voicemail_preference
  
# Household-level memory (Real Estate)
household:
  - decision_maker
  - other_members
  - property_type_interest
  - price_range
  - preferred_areas
  - motivation              # upsizing, downsizing, relocating
  - timeline
  
# Account-level memory
account:
  - company_name
  - industry
  - employee_count
  - annual_revenue
  - decision_process
  - key_stakeholders
```

---

## 5. Action Outbox (Guaranteed Delivery)

```sql
CREATE TABLE action_outbox (
    action_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Tenant
    location_id UUID NOT NULL,
    lead_id UUID NOT NULL,
    
    -- Action details
    action_type VARCHAR(50) NOT NULL,  -- 'send_sms', 'send_email', 'initiate_call'
    payload JSONB NOT NULL,
    
    -- Scheduling
    execute_after TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    execute_before TIMESTAMPTZ,        -- Deadline (optional)
    
    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'processing', 'sent', 'delivered', 'failed', 'cancelled')),
    
    -- Retry logic
    attempts INT DEFAULT 0,
    max_attempts INT DEFAULT 3,
    last_attempt_at TIMESTAMPTZ,
    last_error TEXT,
    next_retry_at TIMESTAMPTZ,
    
    -- Traceability
    correlation_id UUID,
    created_by VARCHAR(50),            -- 'agent', 'workflow', 'manual'
    
    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_outbox_pending ON action_outbox(execute_after) 
    WHERE status = 'pending';
CREATE INDEX idx_outbox_location ON action_outbox(location_id, status);
CREATE INDEX idx_outbox_lead ON action_outbox(lead_id);
```

---

## 6. Experiment Assignments (A/B Testing)

```sql
CREATE TABLE experiment_assignments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    lead_id UUID NOT NULL,
    location_id UUID NOT NULL,
    
    -- Experiment
    experiment_name VARCHAR(100) NOT NULL,
    variant VARCHAR(50) NOT NULL,      -- 'control', 'treatment_a', 'treatment_b'
    
    -- Assignment
    assigned_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- Outcome tracking
    outcome_measured BOOLEAN DEFAULT FALSE,
    outcome_value JSONB,               -- {'booked': true, 'showed': true, 'won': false}
    outcome_measured_at TIMESTAMPTZ,
    
    UNIQUE(lead_id, experiment_name)
);

CREATE INDEX idx_experiments_name ON experiment_assignments(experiment_name, variant);
```

---

## 7. Agent Plans (What AI Decided)

```sql
CREATE TABLE agent_plans (
    plan_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Context
    location_id UUID NOT NULL,
    lead_id UUID NOT NULL,
    goal_id UUID,                      -- Which goal this serves
    
    -- Plan details
    proposed_actions JSONB NOT NULL,   -- Array of actions
    reasoning TEXT,                    -- Why these actions
    confidence FLOAT,
    risk_score FLOAT,
    
    -- Status
    status VARCHAR(20) DEFAULT 'proposed'
        CHECK (status IN ('proposed', 'approved', 'executing', 'completed', 'failed', 'cancelled')),
    
    -- Approval (if needed)
    requires_approval BOOLEAN DEFAULT FALSE,
    approved_by UUID REFERENCES users(id),
    approved_at TIMESTAMPTZ,
    
    -- Execution
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    
    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_plans_lead ON agent_plans(lead_id, created_at DESC);
CREATE INDEX idx_plans_status ON agent_plans(status) WHERE status = 'executing';
```

---

## Complete Schema Summary

| Table | Purpose | Key Innovation |
|-------|---------|----------------|
| `events` | Append-only truth | Deterministic replay |
| `lead_state` | Finite state machine | No tags, validated transitions |
| `consent` | Hard compliance gates | Legal defensibility |
| `agent_memory` | Structured recall | No hallucination |
| `action_outbox` | Guaranteed delivery | Self-healing retry |
| `experiment_assignments` | A/B testing | Incrementality measurement |
| `agent_plans` | AI decision log | Explainable AI |

---

## Why GHL Can't Catch Up

To implement this, GHL would need to:

1. **Rebuild their event system** - They mutate records, we append
2. **Replace tags with state machine** - Breaking change for users
3. **Add consent provenance** - Retrofitting is legally risky
4. **Build structured memory** - Not just chat history
5. **Add experimentation** - Requires holdouts, uplift modeling

> "That rebuild is expensive and risky for them because it changes how users interact with the system."
