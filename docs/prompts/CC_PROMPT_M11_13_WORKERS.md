# Claude Code Prompt: M11-13 Background Workers

## Context

You are building the Background Workers module for Ziloss CRM, a GoHighLevel competitor. This module handles all asynchronous processing: message delivery, conversation monitoring, and lead scoring.

**Project Location:** `/Users/mac/Desktop/agent-orchestrator`
**Module Location:** `/Users/mac/Desktop/agent-orchestrator/src/workers/`
**Estimated Lines:** 1,900
**Time Estimate:** 2 days

## Tech Stack

- **Language:** Python 3.11+
- **Queue:** Redis (via redis-py with asyncio)
- **Database:** PostgreSQL (async via asyncpg)
- **Background Framework:** Custom worker pattern (no Celery)
- **SMS:** Twilio
- **Email:** SendGrid/Mailgun
- **Testing:** pytest + pytest-asyncio

## Module Purpose

Three workers that power the "Autonomous Growth Operator":

1. **M11: Outbox Worker** - Process pending messages (SMS/email) from the action_outbox table
2. **M12: Stale Conversation Monitor** - Find conversations with no reply in 24+ hours
3. **M13: Positive Response Monitor** - Detect hot leads based on sentiment/keywords

## File Structure to Create

```
src/workers/
├── __init__.py
├── runner.py                  # Main entry point: python -m workers.runner
├── base.py                    # BaseWorker class
├── config.py                  # Worker configuration
├── outbox/
│   ├── __init__.py
│   ├── worker.py              # M11: Outbox processor
│   ├── sms_sender.py          # Twilio integration
│   ├── email_sender.py        # SendGrid integration
│   └── retry.py               # Retry logic with backoff
├── monitors/
│   ├── __init__.py
│   ├── stale_conversation.py  # M12: Find stale conversations
│   └── positive_response.py   # M13: Detect hot leads
├── utils/
│   ├── __init__.py
│   ├── rate_limiter.py        # Per-tenant rate limiting
│   └── health_check.py        # Worker health endpoints
└── tests/
    ├── __init__.py
    ├── test_outbox.py
    ├── test_stale_monitor.py
    ├── test_positive_monitor.py
    └── conftest.py
```

## Database Tables

### action_outbox (from AGO architecture)

```sql
CREATE TABLE action_outbox (
    action_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id),
    lead_id UUID NOT NULL REFERENCES contacts(id),
    
    action_type VARCHAR(50) NOT NULL,
    payload JSONB NOT NULL DEFAULT '{}',
    
    execute_after TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    execute_before TIMESTAMPTZ,  -- Deadline
    
    status VARCHAR(20) NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'processing', 'sent', 'delivered', 'failed', 'cancelled')),
    
    attempts INT DEFAULT 0,
    max_attempts INT DEFAULT 3,
    last_attempt_at TIMESTAMPTZ,
    last_error TEXT,
    next_retry_at TIMESTAMPTZ,
    
    correlation_id UUID,
    created_by VARCHAR(50),
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_outbox_pending ON action_outbox(execute_after) 
    WHERE status = 'pending';
CREATE INDEX idx_outbox_retry ON action_outbox(next_retry_at) 
    WHERE status = 'failed' AND attempts < max_attempts;
```

### dead_letter_queue (for failed messages)

```sql
CREATE TABLE dead_letter_queue (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    original_action_id UUID NOT NULL,
    location_id UUID NOT NULL,
    lead_id UUID NOT NULL,
    
    action_type VARCHAR(50) NOT NULL,
    payload JSONB NOT NULL,
    
    error_message TEXT,
    error_details JSONB,
    
    attempts INT,
    first_attempt_at TIMESTAMPTZ,
    last_attempt_at TIMESTAMPTZ,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    resolved_at TIMESTAMPTZ,
    resolved_by UUID,
    resolution_notes TEXT
);

CREATE INDEX idx_dlq_location ON dead_letter_queue(location_id);
CREATE INDEX idx_dlq_unresolved ON dead_letter_queue(created_at) 
    WHERE resolved_at IS NULL;
```

### worker_health (for monitoring)

```sql
CREATE TABLE worker_health (
    worker_id VARCHAR(100) PRIMARY KEY,
    worker_type VARCHAR(50) NOT NULL,
    
    last_heartbeat TIMESTAMPTZ NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'running'
        CHECK (status IN ('running', 'stopping', 'stopped', 'error')),
    
    processed_count BIGINT DEFAULT 0,
    error_count BIGINT DEFAULT 0,
    last_error TEXT,
    last_error_at TIMESTAMPTZ,
    
    started_at TIMESTAMPTZ NOT NULL,
    metadata JSONB DEFAULT '{}'
);
```

## Base Worker Implementation

```python
# src/workers/base.py

import asyncio
import signal
from abc import ABC, abstractmethod
from datetime import datetime, timedelta
from typing import Optional
from uuid import uuid4
import logging

logger = logging.getLogger(__name__)

class BaseWorker(ABC):
    """
    Base class for all background workers.
    Implements graceful shutdown, health checks, and error handling.
    """
    
    def __init__(
        self,
        name: str,
        poll_interval: float = 1.0,
        batch_size: int = 10,
        max_retries: int = 3
    ):
        self.name = name
        self.worker_id = f"{name}-{uuid4().hex[:8]}"
        self.poll_interval = poll_interval
        self.batch_size = batch_size
        self.max_retries = max_retries
        
        self._running = False
        self._shutdown_event = asyncio.Event()
        self._processed_count = 0
        self._error_count = 0
        self._last_error: Optional[str] = None
        self._started_at: Optional[datetime] = None
    
    async def start(self):
        """Start the worker loop."""
        self._running = True
        self._started_at = datetime.utcnow()
        
        # Register signal handlers
        loop = asyncio.get_event_loop()
        for sig in (signal.SIGINT, signal.SIGTERM):
            loop.add_signal_handler(sig, self._handle_shutdown)
        
        logger.info(f"Worker {self.worker_id} starting...")
        
        try:
            await self._register_health()
            
            while self._running:
                try:
                    # Process a batch
                    processed = await self.process_batch()
                    self._processed_count += processed
                    
                    # Update health
                    await self._update_health()
                    
                    # Sleep if nothing to process
                    if processed == 0:
                        await asyncio.sleep(self.poll_interval)
                    
                except Exception as e:
                    self._error_count += 1
                    self._last_error = str(e)
                    logger.exception(f"Worker {self.worker_id} error: {e}")
                    await asyncio.sleep(self.poll_interval * 2)  # Back off on error
                
                # Check for shutdown
                if self._shutdown_event.is_set():
                    break
        
        finally:
            await self._deregister_health()
            logger.info(f"Worker {self.worker_id} stopped. Processed: {self._processed_count}")
    
    def _handle_shutdown(self):
        """Handle graceful shutdown."""
        logger.info(f"Worker {self.worker_id} received shutdown signal")
        self._running = False
        self._shutdown_event.set()
    
    @abstractmethod
    async def process_batch(self) -> int:
        """
        Process a batch of items.
        Returns the number of items processed.
        """
        pass
    
    async def _register_health(self):
        """Register worker in health table."""
        await self.db.execute("""
            INSERT INTO worker_health (worker_id, worker_type, last_heartbeat, status, started_at)
            VALUES ($1, $2, NOW(), 'running', NOW())
            ON CONFLICT (worker_id) 
            DO UPDATE SET last_heartbeat = NOW(), status = 'running', started_at = NOW()
        """, self.worker_id, self.name)
    
    async def _update_health(self):
        """Update health heartbeat."""
        await self.db.execute("""
            UPDATE worker_health 
            SET last_heartbeat = NOW(),
                processed_count = $2,
                error_count = $3,
                last_error = $4,
                last_error_at = CASE WHEN $4 IS NOT NULL THEN NOW() ELSE last_error_at END
            WHERE worker_id = $1
        """, self.worker_id, self._processed_count, self._error_count, self._last_error)
    
    async def _deregister_health(self):
        """Mark worker as stopped."""
        await self.db.execute("""
            UPDATE worker_health SET status = 'stopped' WHERE worker_id = $1
        """, self.worker_id)
```

## M11: Outbox Worker

```python
# src/workers/outbox/worker.py

from src.workers.base import BaseWorker
from src.workers.outbox.sms_sender import SMSSender
from src.workers.outbox.email_sender import EmailSender
from src.workers.outbox.retry import calculate_next_retry

class OutboxWorker(BaseWorker):
    """
    Processes pending actions from action_outbox table.
    Implements the outbox pattern for reliable message delivery.
    """
    
    def __init__(self, db, redis, twilio_client, sendgrid_client):
        super().__init__(
            name="outbox",
            poll_interval=0.5,  # Fast polling for real-time feel
            batch_size=20
        )
        self.db = db
        self.redis = redis
        self.sms_sender = SMSSender(twilio_client)
        self.email_sender = EmailSender(sendgrid_client)
    
    async def process_batch(self) -> int:
        """Fetch and process pending actions."""
        
        # Use SELECT FOR UPDATE SKIP LOCKED for concurrent workers
        actions = await self.db.fetch_all("""
            SELECT * FROM action_outbox
            WHERE status = 'pending'
              AND execute_after <= NOW()
              AND (execute_before IS NULL OR execute_before > NOW())
            ORDER BY execute_after
            LIMIT $1
            FOR UPDATE SKIP LOCKED
        """, self.batch_size)
        
        if not actions:
            # Also check for retries
            actions = await self.db.fetch_all("""
                SELECT * FROM action_outbox
                WHERE status = 'failed'
                  AND attempts < max_attempts
                  AND next_retry_at <= NOW()
                ORDER BY next_retry_at
                LIMIT $1
                FOR UPDATE SKIP LOCKED
            """, self.batch_size)
        
        for action in actions:
            await self._process_action(action)
        
        return len(actions)
    
    async def _process_action(self, action: dict):
        """Process a single action."""
        action_id = action['action_id']
        
        try:
            # Mark as processing
            await self.db.execute("""
                UPDATE action_outbox 
                SET status = 'processing', 
                    attempts = attempts + 1,
                    last_attempt_at = NOW()
                WHERE action_id = $1
            """, action_id)
            
            # Check consent before sending
            if not await self._check_consent(action):
                await self._mark_cancelled(action_id, "No consent")
                return
            
            # Check rate limit
            if not await self._check_rate_limit(action):
                await self._schedule_retry(action_id, "Rate limited", delay_seconds=60)
                return
            
            # Execute based on action type
            result = await self._execute_action(action)
            
            # Mark as sent
            await self.db.execute("""
                UPDATE action_outbox 
                SET status = 'sent',
                    updated_at = NOW()
                WHERE action_id = $1
            """, action_id)
            
            # Emit event
            await self._emit_event(action, result)
            
        except Exception as e:
            logger.exception(f"Action {action_id} failed: {e}")
            await self._handle_failure(action, e)
    
    async def _execute_action(self, action: dict) -> dict:
        """Execute the action based on type."""
        action_type = action['action_type']
        payload = action['payload']
        
        if action_type == 'send_sms':
            return await self.sms_sender.send(
                to=payload['to'],
                body=payload['body'],
                from_number=payload.get('from')
            )
        
        elif action_type == 'send_email':
            return await self.email_sender.send(
                to=payload['to'],
                subject=payload['subject'],
                body=payload['body'],
                from_email=payload.get('from')
            )
        
        elif action_type == 'initiate_call':
            # Voice AI integration
            return await self._initiate_call(payload)
        
        else:
            raise ValueError(f"Unknown action type: {action_type}")
    
    async def _check_consent(self, action: dict) -> bool:
        """
        CRITICAL: Check consent before sending any message.
        This is a hard gate - no exceptions.
        """
        lead_id = action['lead_id']
        action_type = action['action_type']
        
        channel_map = {
            'send_sms': 'sms',
            'send_email': 'email',
            'initiate_call': 'voice'
        }
        channel = channel_map.get(action_type)
        
        if not channel:
            return True  # Non-communication actions don't need consent
        
        consent = await self.db.fetch_one("""
            SELECT status FROM consent
            WHERE lead_id = $1 AND channel = $2
        """, lead_id, channel)
        
        return consent and consent['status'] == 'opted_in'
    
    async def _check_rate_limit(self, action: dict) -> bool:
        """Check per-tenant rate limits."""
        location_id = str(action['location_id'])
        action_type = action['action_type']
        
        # Rate limit key: location:action_type
        key = f"ratelimit:{location_id}:{action_type}"
        
        # 10 per second per location per action type
        current = await self.redis.incr(key)
        if current == 1:
            await self.redis.expire(key, 1)
        
        return current <= 10
    
    async def _handle_failure(self, action: dict, error: Exception):
        """Handle action failure with retry or dead letter."""
        action_id = action['action_id']
        attempts = action['attempts'] + 1
        max_attempts = action['max_attempts']
        
        if attempts >= max_attempts:
            # Move to dead letter queue
            await self._move_to_dlq(action, error)
        else:
            # Schedule retry with exponential backoff
            delay = calculate_next_retry(attempts)
            await self._schedule_retry(action_id, str(error), delay)
    
    async def _schedule_retry(self, action_id: str, error: str, delay_seconds: int):
        """Schedule action for retry."""
        await self.db.execute("""
            UPDATE action_outbox 
            SET status = 'failed',
                last_error = $2,
                next_retry_at = NOW() + INTERVAL '$3 seconds'
            WHERE action_id = $1
        """, action_id, error, delay_seconds)
    
    async def _move_to_dlq(self, action: dict, error: Exception):
        """Move failed action to dead letter queue."""
        await self.db.execute("""
            INSERT INTO dead_letter_queue (
                original_action_id, location_id, lead_id, action_type, payload,
                error_message, attempts, first_attempt_at, last_attempt_at
            ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, NOW())
        """, 
            action['action_id'],
            action['location_id'],
            action['lead_id'],
            action['action_type'],
            action['payload'],
            str(error),
            action['attempts'],
            action['created_at']
        )
        
        # Mark original as failed permanently
        await self.db.execute("""
            UPDATE action_outbox SET status = 'failed' WHERE action_id = $1
        """, action['action_id'])
    
    async def _emit_event(self, action: dict, result: dict):
        """Emit event to the event ledger."""
        event_type = f"{action['action_type'].replace('send_', '')}.sent"
        
        await self.db.execute("""
            INSERT INTO events (
                tenant_id, lead_id, actor, event_type, payload, correlation_id, source
            ) VALUES ($1, $2, 'system', $3, $4, $5, $6)
        """,
            action['location_id'],
            action['lead_id'],
            event_type,
            {**action['payload'], 'result': result},
            action['correlation_id'],
            'outbox_worker'
        )
```

## M11: SMS Sender

```python
# src/workers/outbox/sms_sender.py

from twilio.rest import Client
from twilio.base.exceptions import TwilioRestException

class SMSSender:
    """Sends SMS via Twilio."""
    
    def __init__(self, client: Client):
        self.client = client
    
    async def send(
        self,
        to: str,
        body: str,
        from_number: str = None
    ) -> dict:
        """
        Send SMS message.
        Returns message SID and status.
        """
        try:
            message = self.client.messages.create(
                to=to,
                body=body,
                from_=from_number
            )
            
            return {
                'sid': message.sid,
                'status': message.status,
                'to': to,
                'segments': len(body) // 160 + 1
            }
        
        except TwilioRestException as e:
            # Map Twilio errors to our error types
            if e.code == 21211:  # Invalid phone number
                raise InvalidPhoneError(to)
            elif e.code == 21610:  # Unsubscribed
                raise OptOutError(to)
            elif e.code == 21408:  # Permission denied
                raise PermissionError("SMS permission denied")
            else:
                raise SMSDeliveryError(str(e))
```

## M12: Stale Conversation Monitor

```python
# src/workers/monitors/stale_conversation.py

from src.workers.base import BaseWorker

class StaleConversationMonitor(BaseWorker):
    """
    Monitors for conversations that haven't received a reply.
    Triggers follow-up actions or alerts.
    """
    
    STALE_THRESHOLD_HOURS = 24
    
    def __init__(self, db, redis):
        super().__init__(
            name="stale_conversation",
            poll_interval=60,  # Check every minute
            batch_size=50
        )
        self.db = db
        self.redis = redis
    
    async def process_batch(self) -> int:
        """Find and process stale conversations."""
        
        stale_conversations = await self.db.fetch_all("""
            SELECT 
                c.id as conversation_id,
                c.location_id,
                c.contact_id,
                c.last_message_at,
                c.last_message_direction,
                cont.first_name,
                cont.phone,
                ls.lifecycle,
                ls.intent_score
            FROM conversations c
            JOIN contacts cont ON c.contact_id = cont.id
            LEFT JOIN lead_state ls ON c.contact_id = ls.lead_id
            WHERE c.last_message_direction = 'outbound'
              AND c.last_message_at < NOW() - INTERVAL '$1 hours'
              AND c.status = 'active'
              AND (ls.lifecycle IS NULL OR ls.lifecycle NOT IN ('won', 'lost', 'dnc'))
              AND NOT EXISTS (
                  SELECT 1 FROM conversation_stale_checks 
                  WHERE conversation_id = c.id 
                    AND checked_at > NOW() - INTERVAL '24 hours'
              )
            ORDER BY c.last_message_at
            LIMIT $2
        """, self.STALE_THRESHOLD_HOURS, self.batch_size)
        
        for conv in stale_conversations:
            await self._process_stale_conversation(conv)
        
        return len(stale_conversations)
    
    async def _process_stale_conversation(self, conv: dict):
        """Process a single stale conversation."""
        
        # Record that we checked this conversation
        await self.db.execute("""
            INSERT INTO conversation_stale_checks (conversation_id, checked_at)
            VALUES ($1, NOW())
        """, conv['conversation_id'])
        
        # Add "stale" tag to contact
        await self._add_tag(conv['contact_id'], 'stale_conversation')
        
        # Emit event for workflow triggers
        await self._emit_event(conv)
        
        # Check if we should trigger auto-follow-up
        if conv['intent_score'] and conv['intent_score'] > 0.5:
            await self._trigger_follow_up(conv)
    
    async def _add_tag(self, contact_id: str, tag: str):
        """Add a tag to a contact."""
        await self.db.execute("""
            INSERT INTO contact_tags (contact_id, tag, added_at)
            VALUES ($1, $2, NOW())
            ON CONFLICT (contact_id, tag) DO NOTHING
        """, contact_id, tag)
    
    async def _emit_event(self, conv: dict):
        """Emit stale conversation event."""
        await self.db.execute("""
            INSERT INTO events (
                tenant_id, lead_id, actor, event_type, payload, source
            ) VALUES ($1, $2, 'system', 'conversation.stale', $3, 'stale_monitor')
        """,
            conv['location_id'],
            conv['contact_id'],
            {
                'conversation_id': str(conv['conversation_id']),
                'hours_since_reply': self.STALE_THRESHOLD_HOURS,
                'last_message_at': conv['last_message_at'].isoformat()
            }
        )
    
    async def _trigger_follow_up(self, conv: dict):
        """Trigger automated follow-up for high-intent leads."""
        # Queue a follow-up action
        await self.db.execute("""
            INSERT INTO action_outbox (
                location_id, lead_id, action_type, payload, 
                execute_after, created_by
            ) VALUES ($1, $2, 'send_sms', $3, NOW(), 'stale_monitor')
        """,
            conv['location_id'],
            conv['contact_id'],
            {
                'template': 'follow_up_no_reply',
                'to': conv['phone'],
                'context': {'first_name': conv['first_name']}
            }
        )
```

## M13: Positive Response Monitor

```python
# src/workers/monitors/positive_response.py

from src.workers.base import BaseWorker
import re

class PositiveResponseMonitor(BaseWorker):
    """
    Detects hot leads based on positive keywords and sentiment.
    Alerts team and updates lead scoring.
    """
    
    # Keywords that indicate high intent
    POSITIVE_KEYWORDS = [
        r'\byes\b',
        r'\binterested\b',
        r'\bready\b',
        r'\bwant to\b',
        r'\blet\'?s do it\b',
        r'\bsign me up\b',
        r'\bbook\b',
        r'\bschedule\b',
        r'\bappointment\b',
        r'\bhow much\b',
        r'\bprice\b',
        r'\bavailable\b',
        r'\btoday\b',
        r'\btomorrow\b',
        r'\basap\b',
        r'\bsoon\b',
    ]
    
    def __init__(self, db, redis, ai_service):
        super().__init__(
            name="positive_response",
            poll_interval=5,  # Fast polling for hot leads
            batch_size=20
        )
        self.db = db
        self.redis = redis
        self.ai_service = ai_service
        self._keyword_pattern = re.compile(
            '|'.join(self.POSITIVE_KEYWORDS), 
            re.IGNORECASE
        )
    
    async def process_batch(self) -> int:
        """Find and score new inbound messages."""
        
        # Get recent inbound messages not yet scored
        messages = await self.db.fetch_all("""
            SELECT 
                m.id as message_id,
                m.conversation_id,
                m.body,
                m.created_at,
                c.location_id,
                c.contact_id,
                cont.first_name,
                cont.phone,
                cont.email
            FROM messages m
            JOIN conversations c ON m.conversation_id = c.id
            JOIN contacts cont ON c.contact_id = cont.id
            WHERE m.direction = 'inbound'
              AND m.created_at > NOW() - INTERVAL '1 hour'
              AND NOT EXISTS (
                  SELECT 1 FROM message_scores 
                  WHERE message_id = m.id
              )
            ORDER BY m.created_at DESC
            LIMIT $1
        """, self.batch_size)
        
        for msg in messages:
            await self._score_message(msg)
        
        return len(messages)
    
    async def _score_message(self, msg: dict):
        """Score a message for positive intent."""
        
        body = msg['body']
        
        # Quick keyword check first
        keyword_score = self._calculate_keyword_score(body)
        
        # If keywords found, do full AI analysis
        ai_score = 0.0
        if keyword_score > 0.3:
            ai_result = await self.ai_service.analyze_message(
                location_id=msg['location_id'],
                message=body
            )
            ai_score = self._calculate_ai_score(ai_result)
        
        # Combined score
        final_score = (keyword_score * 0.3) + (ai_score * 0.7)
        
        # Record the score
        await self.db.execute("""
            INSERT INTO message_scores (message_id, keyword_score, ai_score, final_score)
            VALUES ($1, $2, $3, $4)
        """, msg['message_id'], keyword_score, ai_score, final_score)
        
        # Update lead state if high score
        if final_score > 0.7:
            await self._handle_hot_lead(msg, final_score)
    
    def _calculate_keyword_score(self, body: str) -> float:
        """Calculate score based on keyword matches."""
        matches = self._keyword_pattern.findall(body.lower())
        
        if not matches:
            return 0.0
        
        # More matches = higher score, capped at 1.0
        return min(len(matches) * 0.2, 1.0)
    
    def _calculate_ai_score(self, ai_result) -> float:
        """Convert AI analysis to a 0-1 score."""
        score = 0.0
        
        # Intent scoring
        intent_scores = {
            'booking_request': 1.0,
            'positive_interest': 0.9,
            'question_pricing': 0.7,
            'question_general': 0.4,
        }
        score += intent_scores.get(ai_result.intent.intent.value, 0) * 0.5
        
        # Sentiment scoring
        if ai_result.sentiment.sentiment == 'positive':
            score += 0.3
        elif ai_result.sentiment.sentiment == 'urgent':
            score += 0.4
        
        # Confidence factor
        score *= ai_result.intent.confidence
        
        return min(score, 1.0)
    
    async def _handle_hot_lead(self, msg: dict, score: float):
        """Handle a detected hot lead."""
        
        # Update lead state
        await self.db.execute("""
            UPDATE lead_state 
            SET intent_score = GREATEST(intent_score, $2),
                lifecycle = CASE 
                    WHEN lifecycle IN ('new', 'contacting') THEN 'engaged'
                    ELSE lifecycle
                END,
                updated_at = NOW()
            WHERE lead_id = $1
        """, msg['contact_id'], score)
        
        # Add hot lead tag
        await self.db.execute("""
            INSERT INTO contact_tags (contact_id, tag, added_at)
            VALUES ($1, 'hot_lead', NOW())
            ON CONFLICT (contact_id, tag) DO NOTHING
        """, msg['contact_id'])
        
        # Emit event
        await self.db.execute("""
            INSERT INTO events (
                tenant_id, lead_id, actor, event_type, payload, source
            ) VALUES ($1, $2, 'system', 'lead.hot_detected', $3, 'positive_monitor')
        """,
            msg['location_id'],
            msg['contact_id'],
            {
                'score': score,
                'message_preview': msg['body'][:100],
                'contact_name': msg['first_name'],
                'phone': msg['phone']
            }
        )
        
        # Send team notification
        await self._notify_team(msg, score)
    
    async def _notify_team(self, msg: dict, score: float):
        """Send notification to team about hot lead."""
        
        # Get team members to notify
        team = await self.db.fetch_all("""
            SELECT u.id, u.email, u.phone, u.notification_preferences
            FROM users u
            JOIN user_locations ul ON u.id = ul.user_id
            WHERE ul.location_id = $1
              AND u.notification_preferences->>'hot_leads' = 'true'
        """, msg['location_id'])
        
        for member in team:
            # Queue notification
            await self.db.execute("""
                INSERT INTO action_outbox (
                    location_id, lead_id, action_type, payload, 
                    execute_after, created_by
                ) VALUES ($1, $2, 'internal_notification', $3, NOW(), 'positive_monitor')
            """,
                msg['location_id'],
                msg['contact_id'],
                {
                    'type': 'hot_lead',
                    'recipient_id': str(member['id']),
                    'recipient_email': member['email'],
                    'contact_name': msg['first_name'],
                    'score': score
                }
            )
```

## Retry Logic

```python
# src/workers/outbox/retry.py

def calculate_next_retry(attempt: int) -> int:
    """
    Calculate delay for next retry using exponential backoff.
    Returns delay in seconds.
    
    Attempt 1: 30 seconds
    Attempt 2: 2 minutes
    Attempt 3: 8 minutes
    """
    base_delay = 30  # seconds
    max_delay = 3600  # 1 hour
    
    delay = base_delay * (2 ** (attempt - 1))
    return min(delay, max_delay)
```

## Worker Runner

```python
# src/workers/runner.py

import asyncio
import argparse
from src.workers.outbox.worker import OutboxWorker
from src.workers.monitors.stale_conversation import StaleConversationMonitor
from src.workers.monitors.positive_response import PositiveResponseMonitor

async def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--worker', choices=['outbox', 'stale', 'positive', 'all'], default='all')
    args = parser.parse_args()
    
    # Initialize dependencies
    db = await create_db_pool()
    redis = await create_redis_pool()
    twilio = create_twilio_client()
    sendgrid = create_sendgrid_client()
    ai_service = create_ai_service()
    
    workers = []
    
    if args.worker in ['outbox', 'all']:
        workers.append(OutboxWorker(db, redis, twilio, sendgrid))
    
    if args.worker in ['stale', 'all']:
        workers.append(StaleConversationMonitor(db, redis))
    
    if args.worker in ['positive', 'all']:
        workers.append(PositiveResponseMonitor(db, redis, ai_service))
    
    # Run all workers
    await asyncio.gather(*[w.start() for w in workers])

if __name__ == "__main__":
    asyncio.run(main())
```

## Testing Requirements

1. **test_outbox.py**
   - Test message sending (SMS, email)
   - Test consent checking
   - Test rate limiting
   - Test retry logic
   - Test dead letter queue

2. **test_stale_monitor.py**
   - Test stale detection
   - Test tag addition
   - Test event emission
   - Test follow-up triggering

3. **test_positive_monitor.py**
   - Test keyword scoring
   - Test AI score calculation
   - Test hot lead handling
   - Test team notification

## Environment Variables

```env
# Redis
REDIS_URL=redis://localhost:6379

# Twilio
TWILIO_ACCOUNT_SID=xxx
TWILIO_AUTH_TOKEN=xxx
TWILIO_FROM_NUMBER=+1234567890

# SendGrid
SENDGRID_API_KEY=xxx
SENDGRID_FROM_EMAIL=noreply@example.com

# Worker settings
OUTBOX_POLL_INTERVAL=0.5
OUTBOX_BATCH_SIZE=20
STALE_THRESHOLD_HOURS=24
```

## Success Criteria

- [ ] Outbox processes messages reliably
- [ ] Consent checked before every send
- [ ] Rate limiting works
- [ ] Retry with exponential backoff
- [ ] Dead letter queue for failed messages
- [ ] Stale conversations detected
- [ ] Hot leads identified and scored
- [ ] Team notifications sent
- [ ] Events emitted to ledger
- [ ] All tests pass
