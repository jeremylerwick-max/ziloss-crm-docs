# Claude Code Prompt: Event Ledger (AGO Foundation)

## Context

You are building the Event Ledger module for Ziloss CRM. This is the FOUNDATIONAL component for the Autonomous Growth Operator (AGO). Every action in the system writes to this append-only log.

**Project Location:** `/Users/mac/Desktop/agent-orchestrator`
**Module Location:** `/Users/mac/Desktop/agent-orchestrator/src/core/events/`
**Estimated Lines:** 800
**Time Estimate:** 1 day

## Why This Matters

> "If you get this right, GHL can't catch you without a rewrite."

The Event Ledger enables:
- **Deterministic replay** - Rebuild any state from events
- **Auditability** - Full history of every action
- **Debugging AI decisions** - See exactly what the AI saw
- **Legal defensibility** - Prove compliance
- **Self-healing** - Detect and repair anomalies

**GHL does not have this as a first-class primitive.**

## File Structure

```
src/core/events/
├── __init__.py
├── ledger.py              # Main EventLedger class
├── models.py              # Event models
├── publisher.py           # Event publishing
├── subscriber.py          # Event subscription
├── replay.py              # Event replay for debugging
├── projections.py         # Build state from events
└── tests/
    ├── __init__.py
    ├── test_ledger.py
    ├── test_replay.py
    └── conftest.py
```

## Database Schema

```sql
-- The canonical event ledger (append-only, never mutate)
CREATE TABLE events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- When
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    -- Who/Where
    tenant_id UUID NOT NULL,      -- agency_id
    location_id UUID NOT NULL,    -- sub-account
    lead_id UUID,                 -- Contact this relates to (nullable for system events)
    
    -- Who did this
    actor VARCHAR(20) NOT NULL CHECK (actor IN (
        'system',      -- Automated system action
        'agent',       -- AI agent action
        'rep',         -- Human rep action
        'lead',        -- The lead/contact themselves
        'external',    -- External webhook/integration
        'workflow'     -- Workflow engine
    )),
    
    -- What happened
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL DEFAULT '{}',
    
    -- Traceability
    correlation_id UUID,   -- Groups related events (e.g., all events from one workflow run)
    causation_id UUID,     -- The event that caused this one
    source VARCHAR(50),    -- Which service/worker created this
    
    -- Integrity
    sequence_number BIGSERIAL,  -- Monotonic within tenant
    checksum VARCHAR(64),       -- SHA256 for integrity verification
    
    -- Metadata
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes for fast queries
CREATE INDEX idx_events_tenant_lead ON events(tenant_id, lead_id, occurred_at DESC);
CREATE INDEX idx_events_location ON events(location_id, occurred_at DESC);
CREATE INDEX idx_events_type ON events(event_type, occurred_at DESC);
CREATE INDEX idx_events_correlation ON events(correlation_id) WHERE correlation_id IS NOT NULL;
CREATE INDEX idx_events_causation ON events(causation_id) WHERE causation_id IS NOT NULL;
CREATE INDEX idx_events_sequence ON events(tenant_id, sequence_number);

-- Prevent any updates or deletes (append-only enforcement)
CREATE OR REPLACE FUNCTION prevent_event_modification()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'Events table is append-only. Modifications are not allowed.';
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER events_append_only
    BEFORE UPDATE OR DELETE ON events
    FOR EACH ROW
    EXECUTE FUNCTION prevent_event_modification();
```

## Event Types

```python
# src/core/events/models.py

from enum import Enum

class EventType(str, Enum):
    # Lead lifecycle
    LEAD_CREATED = "lead.created"
    LEAD_UPDATED = "lead.updated"
    LEAD_ENRICHED = "lead.enriched"
    LEAD_QUALIFIED = "lead.qualified"
    LEAD_ASSIGNED = "lead.assigned"
    LEAD_STATE_CHANGED = "lead.state_changed"
    LEAD_WON = "lead.won"
    LEAD_LOST = "lead.lost"
    LEAD_HOT_DETECTED = "lead.hot_detected"
    
    # Communication - SMS
    SMS_QUEUED = "sms.queued"
    SMS_SENT = "sms.sent"
    SMS_DELIVERED = "sms.delivered"
    SMS_FAILED = "sms.failed"
    SMS_RECEIVED = "sms.received"
    
    # Communication - Email
    EMAIL_QUEUED = "email.queued"
    EMAIL_SENT = "email.sent"
    EMAIL_DELIVERED = "email.delivered"
    EMAIL_OPENED = "email.opened"
    EMAIL_CLICKED = "email.clicked"
    EMAIL_BOUNCED = "email.bounced"
    EMAIL_RECEIVED = "email.received"
    
    # Communication - Voice
    CALL_INITIATED = "call.initiated"
    CALL_ANSWERED = "call.answered"
    CALL_VOICEMAIL = "call.voicemail"
    CALL_COMPLETED = "call.completed"
    CALL_FAILED = "call.failed"
    
    # Appointments
    APPOINTMENT_REQUESTED = "appointment.requested"
    APPOINTMENT_BOOKED = "appointment.booked"
    APPOINTMENT_CONFIRMED = "appointment.confirmed"
    APPOINTMENT_REMINDED = "appointment.reminded"
    APPOINTMENT_COMPLETED = "appointment.completed"
    APPOINTMENT_NO_SHOW = "appointment.no_show"
    APPOINTMENT_CANCELLED = "appointment.cancelled"
    APPOINTMENT_RESCHEDULED = "appointment.rescheduled"
    
    # Consent
    CONSENT_GRANTED = "consent.granted"
    CONSENT_REVOKED = "consent.revoked"
    
    # Agent actions
    AGENT_PLANNED = "agent.planned"
    AGENT_DECIDED = "agent.decided"
    AGENT_ACTED = "agent.acted"
    AGENT_LEARNED = "agent.learned"
    
    # Conversation
    CONVERSATION_STARTED = "conversation.started"
    CONVERSATION_STALE = "conversation.stale"
    CONVERSATION_CLOSED = "conversation.closed"
    
    # Workflow
    WORKFLOW_ENROLLED = "workflow.enrolled"
    WORKFLOW_STEP_EXECUTED = "workflow.step_executed"
    WORKFLOW_COMPLETED = "workflow.completed"
    WORKFLOW_FAILED = "workflow.failed"
```

## Pydantic Models

```python
from pydantic import BaseModel, Field
from datetime import datetime
from uuid import UUID
from typing import Optional, Any

class Actor(str, Enum):
    SYSTEM = "system"
    AGENT = "agent"
    REP = "rep"
    LEAD = "lead"
    EXTERNAL = "external"
    WORKFLOW = "workflow"

class EventCreate(BaseModel):
    """Model for creating a new event."""
    tenant_id: UUID
    location_id: UUID
    lead_id: Optional[UUID] = None
    actor: Actor
    event_type: str
    payload: dict = Field(default_factory=dict)
    correlation_id: Optional[UUID] = None
    causation_id: Optional[UUID] = None
    source: Optional[str] = None

class Event(BaseModel):
    """Full event model."""
    event_id: UUID
    occurred_at: datetime
    tenant_id: UUID
    location_id: UUID
    lead_id: Optional[UUID]
    actor: Actor
    event_type: str
    payload: dict
    correlation_id: Optional[UUID]
    causation_id: Optional[UUID]
    source: Optional[str]
    sequence_number: int
    checksum: Optional[str]
    created_at: datetime

class EventQuery(BaseModel):
    """Query parameters for fetching events."""
    tenant_id: Optional[UUID] = None
    location_id: Optional[UUID] = None
    lead_id: Optional[UUID] = None
    event_types: Optional[list[str]] = None
    actor: Optional[Actor] = None
    correlation_id: Optional[UUID] = None
    after: Optional[datetime] = None
    before: Optional[datetime] = None
    limit: int = Field(default=100, le=1000)
    offset: int = 0
```

## Event Ledger Implementation

```python
# src/core/events/ledger.py

import hashlib
import json
from datetime import datetime
from typing import Optional, AsyncGenerator
from uuid import UUID, uuid4

class EventLedger:
    """
    Append-only event ledger.
    The single source of truth for everything that happens in the system.
    """
    
    def __init__(self, db):
        self.db = db
    
    async def append(self, event: EventCreate) -> Event:
        """
        Append a new event to the ledger.
        This is the ONLY way to record that something happened.
        """
        event_id = uuid4()
        occurred_at = datetime.utcnow()
        
        # Calculate checksum for integrity
        checksum = self._calculate_checksum(event, event_id, occurred_at)
        
        # Insert (this is the only write operation allowed)
        row = await self.db.fetch_one("""
            INSERT INTO events (
                event_id, occurred_at, tenant_id, location_id, lead_id,
                actor, event_type, payload, correlation_id, causation_id,
                source, checksum
            ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12)
            RETURNING *
        """,
            event_id, occurred_at,
            event.tenant_id, event.location_id, event.lead_id,
            event.actor.value, event.event_type, json.dumps(event.payload),
            event.correlation_id, event.causation_id,
            event.source, checksum
        )
        
        return Event(**row)
    
    async def query(self, query: EventQuery) -> list[Event]:
        """
        Query events from the ledger.
        Supports filtering by tenant, location, lead, type, time range.
        """
        conditions = []
        params = []
        param_count = 0
        
        if query.tenant_id:
            param_count += 1
            conditions.append(f"tenant_id = ${param_count}")
            params.append(query.tenant_id)
        
        if query.location_id:
            param_count += 1
            conditions.append(f"location_id = ${param_count}")
            params.append(query.location_id)
        
        if query.lead_id:
            param_count += 1
            conditions.append(f"lead_id = ${param_count}")
            params.append(query.lead_id)
        
        if query.event_types:
            param_count += 1
            conditions.append(f"event_type = ANY(${param_count})")
            params.append(query.event_types)
        
        if query.actor:
            param_count += 1
            conditions.append(f"actor = ${param_count}")
            params.append(query.actor.value)
        
        if query.correlation_id:
            param_count += 1
            conditions.append(f"correlation_id = ${param_count}")
            params.append(query.correlation_id)
        
        if query.after:
            param_count += 1
            conditions.append(f"occurred_at > ${param_count}")
            params.append(query.after)
        
        if query.before:
            param_count += 1
            conditions.append(f"occurred_at < ${param_count}")
            params.append(query.before)
        
        where_clause = " AND ".join(conditions) if conditions else "TRUE"
        
        rows = await self.db.fetch_all(f"""
            SELECT * FROM events
            WHERE {where_clause}
            ORDER BY occurred_at DESC
            LIMIT ${param_count + 1} OFFSET ${param_count + 2}
        """, *params, query.limit, query.offset)
        
        return [Event(**row) for row in rows]
    
    async def get_lead_history(
        self,
        lead_id: UUID,
        limit: int = 100
    ) -> list[Event]:
        """Get complete event history for a lead."""
        return await self.query(EventQuery(lead_id=lead_id, limit=limit))
    
    async def get_correlation_chain(
        self,
        correlation_id: UUID
    ) -> list[Event]:
        """Get all events in a correlation chain (e.g., one workflow run)."""
        return await self.query(EventQuery(correlation_id=correlation_id, limit=1000))
    
    async def stream_events(
        self,
        query: EventQuery
    ) -> AsyncGenerator[Event, None]:
        """Stream events for real-time processing."""
        last_sequence = 0
        
        while True:
            events = await self.db.fetch_all("""
                SELECT * FROM events
                WHERE tenant_id = $1
                  AND sequence_number > $2
                ORDER BY sequence_number
                LIMIT 100
            """, query.tenant_id, last_sequence)
            
            if not events:
                await asyncio.sleep(0.1)
                continue
            
            for row in events:
                event = Event(**row)
                last_sequence = event.sequence_number
                yield event
    
    def _calculate_checksum(
        self,
        event: EventCreate,
        event_id: UUID,
        occurred_at: datetime
    ) -> str:
        """Calculate SHA256 checksum for event integrity."""
        data = {
            'event_id': str(event_id),
            'occurred_at': occurred_at.isoformat(),
            'tenant_id': str(event.tenant_id),
            'location_id': str(event.location_id),
            'lead_id': str(event.lead_id) if event.lead_id else None,
            'actor': event.actor.value,
            'event_type': event.event_type,
            'payload': event.payload
        }
        
        content = json.dumps(data, sort_keys=True)
        return hashlib.sha256(content.encode()).hexdigest()
    
    async def verify_integrity(
        self,
        event: Event
    ) -> bool:
        """Verify event hasn't been tampered with."""
        expected = self._calculate_checksum(
            EventCreate(
                tenant_id=event.tenant_id,
                location_id=event.location_id,
                lead_id=event.lead_id,
                actor=Actor(event.actor),
                event_type=event.event_type,
                payload=event.payload,
                correlation_id=event.correlation_id,
                causation_id=event.causation_id,
                source=event.source
            ),
            event.event_id,
            event.occurred_at
        )
        return event.checksum == expected
```

## Event Publisher

```python
# src/core/events/publisher.py

from typing import Callable, Awaitable

class EventPublisher:
    """
    Publishes events to the ledger and notifies subscribers.
    Use this instead of directly calling ledger.append() to ensure
    subscribers are notified.
    """
    
    def __init__(self, ledger: EventLedger, redis=None):
        self.ledger = ledger
        self.redis = redis
        self._handlers: dict[str, list[Callable]] = {}
    
    async def publish(self, event: EventCreate) -> Event:
        """
        Publish an event: append to ledger and notify subscribers.
        """
        # Append to ledger (source of truth)
        stored_event = await self.ledger.append(event)
        
        # Notify in-process handlers
        await self._notify_handlers(stored_event)
        
        # Publish to Redis for cross-process subscribers
        if self.redis:
            await self._publish_to_redis(stored_event)
        
        return stored_event
    
    def subscribe(self, event_type: str, handler: Callable[[Event], Awaitable[None]]):
        """Subscribe to events of a specific type."""
        if event_type not in self._handlers:
            self._handlers[event_type] = []
        self._handlers[event_type].append(handler)
    
    def subscribe_all(self, handler: Callable[[Event], Awaitable[None]]):
        """Subscribe to all events."""
        self.subscribe('*', handler)
    
    async def _notify_handlers(self, event: Event):
        """Notify all registered handlers."""
        # Specific handlers
        handlers = self._handlers.get(event.event_type, [])
        # Wildcard handlers
        handlers += self._handlers.get('*', [])
        
        for handler in handlers:
            try:
                await handler(event)
            except Exception as e:
                # Log but don't fail - event is already persisted
                logger.exception(f"Handler error for {event.event_type}: {e}")
    
    async def _publish_to_redis(self, event: Event):
        """Publish event to Redis pub/sub for other processes."""
        channel = f"events:{event.tenant_id}"
        await self.redis.publish(channel, event.json())
```

## Event Replay (for debugging)

```python
# src/core/events/replay.py

class EventReplayer:
    """
    Replay events to debug or rebuild state.
    Essential for understanding "why did the AI do that?"
    """
    
    def __init__(self, ledger: EventLedger):
        self.ledger = ledger
    
    async def replay_lead_journey(
        self,
        lead_id: UUID,
        until: Optional[datetime] = None
    ) -> list[dict]:
        """
        Replay all events for a lead to show their journey.
        Returns a timeline with state at each point.
        """
        events = await self.ledger.get_lead_history(lead_id)
        
        if until:
            events = [e for e in events if e.occurred_at <= until]
        
        # Sort chronologically
        events.sort(key=lambda e: e.occurred_at)
        
        timeline = []
        state = {
            'lifecycle': 'unknown',
            'intent_score': 0,
            'messages_sent': 0,
            'messages_received': 0,
            'appointments': []
        }
        
        for event in events:
            # Update state based on event
            self._apply_event_to_state(event, state)
            
            timeline.append({
                'timestamp': event.occurred_at.isoformat(),
                'event_type': event.event_type,
                'actor': event.actor,
                'state_snapshot': state.copy(),
                'payload': event.payload
            })
        
        return timeline
    
    async def explain_decision(
        self,
        correlation_id: UUID
    ) -> dict:
        """
        Explain why the AI made a particular decision.
        Shows all events that led to the decision.
        """
        events = await self.ledger.get_correlation_chain(correlation_id)
        events.sort(key=lambda e: e.occurred_at)
        
        # Find the decision event
        decision_event = None
        for e in events:
            if e.event_type == 'agent.decided':
                decision_event = e
                break
        
        if not decision_event:
            return {'error': 'No decision event found'}
        
        # Build explanation
        context_events = [e for e in events if e.occurred_at < decision_event.occurred_at]
        
        return {
            'decision': {
                'timestamp': decision_event.occurred_at.isoformat(),
                'action': decision_event.payload.get('proposed_action'),
                'reasoning': decision_event.payload.get('reasoning'),
                'confidence': decision_event.payload.get('confidence')
            },
            'context': [
                {
                    'timestamp': e.occurred_at.isoformat(),
                    'event_type': e.event_type,
                    'summary': self._summarize_event(e)
                }
                for e in context_events
            ]
        }
    
    def _apply_event_to_state(self, event: Event, state: dict):
        """Apply an event to the state snapshot."""
        if event.event_type == 'lead.state_changed':
            state['lifecycle'] = event.payload.get('to', state['lifecycle'])
        elif event.event_type.startswith('sms.sent') or event.event_type.startswith('email.sent'):
            state['messages_sent'] += 1
        elif event.event_type.startswith('sms.received') or event.event_type.startswith('email.received'):
            state['messages_received'] += 1
        elif event.event_type == 'appointment.booked':
            state['appointments'].append(event.payload)
        elif event.event_type == 'lead.hot_detected':
            state['intent_score'] = event.payload.get('score', state['intent_score'])
    
    def _summarize_event(self, event: Event) -> str:
        """Create human-readable summary of an event."""
        summaries = {
            'sms.received': lambda e: f"Lead replied: \"{e.payload.get('body', '')[:50]}...\"",
            'sms.sent': lambda e: f"Sent SMS to lead",
            'lead.state_changed': lambda e: f"State changed: {e.payload.get('from')} → {e.payload.get('to')}",
            'appointment.booked': lambda e: f"Appointment booked for {e.payload.get('start_time')}",
            'lead.hot_detected': lambda e: f"Hot lead detected (score: {e.payload.get('score', 0):.2f})",
        }
        
        formatter = summaries.get(event.event_type, lambda e: event.event_type)
        return formatter(event)
```

## API Endpoints

```python
# Add to main router

@router.get("/events", response_model=list[Event])
async def list_events(
    lead_id: Optional[UUID] = None,
    event_type: Optional[str] = None,
    after: Optional[datetime] = None,
    before: Optional[datetime] = None,
    limit: int = 100,
    location_id: UUID = Depends(get_current_location)
):
    """Query events from the ledger."""
    pass

@router.get("/events/{event_id}", response_model=Event)
async def get_event(event_id: UUID):
    """Get a single event by ID."""
    pass

@router.get("/events/lead/{lead_id}/timeline")
async def get_lead_timeline(lead_id: UUID):
    """Get complete timeline for a lead (for debugging)."""
    pass

@router.get("/events/explain/{correlation_id}")
async def explain_decision(correlation_id: UUID):
    """Explain why the AI made a decision."""
    pass
```

## Testing Requirements

1. **test_ledger.py**
   - Test append (only allowed write)
   - Test query with all filters
   - Test checksum calculation
   - Test integrity verification
   - Test append-only enforcement (no updates/deletes)

2. **test_replay.py**
   - Test lead journey replay
   - Test decision explanation
   - Test state reconstruction

## Success Criteria

- [ ] Append-only enforced at DB level
- [ ] All events have checksums
- [ ] Query supports all filters
- [ ] Integrity verification works
- [ ] Replay shows accurate history
- [ ] Decision explanation works
- [ ] Tests pass with >90% coverage
