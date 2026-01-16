# MODULES M11-13: BACKGROUND WORKERS

**Estimated Lines:** 1,900
**Estimated Time:** 2 days
**Dependencies:** M9 (outbox), M5 (conversations), M4 (contacts)

---

## PURPOSE

Background workers that run continuously to:
- M11: Process outbox queue (send SMS/email)
- M12: Monitor stale conversations (no reply in 24h+)
- M13: Detect positive responses (hot leads)

---

## FILE STRUCTURE

```
/workers/
├── __init__.py
├── base.py                     # Base worker class
├── config.py                   # Worker configuration
├── health.py                   # Health check endpoints
│
├── outbox/                     # M11: Outbox Worker
│   ├── __init__.py
│   ├── worker.py              # Main processing loop
│   ├── sender.py              # Twilio/Email sending
│   ├── retry.py               # Exponential backoff
│   └── dead_letter.py         # DLQ management
│
├── monitors/                   # M12 & M13: Monitors
│   ├── __init__.py
│   ├── stale_conversations.py  # Find no-response convos
│   ├── positive_responses.py   # Find hot leads
│   ├── appointment_reminders.py # Send upcoming reminders
│   └── dnd_expiry.py          # Re-enable after DND period
│
├── scheduled/                  # Scheduled jobs
│   ├── __init__.py
│   ├── daily_digest.py        # Send daily summary
│   ├── weekly_report.py       # Analytics email
│   └── cleanup.py             # Archive old data
│
└── runner.py                   # Supervisor/runner script
```

---

## BASE WORKER CLASS

```python
from abc import ABC, abstractmethod
import asyncio
import signal
from datetime import datetime
import structlog

logger = structlog.get_logger()

class BaseWorker(ABC):
    """Base class for all background workers"""
    
    def __init__(
        self,
        name: str,
        poll_interval_seconds: float = 5.0,
        batch_size: int = 10,
        max_retries: int = 3,
    ):
        self.name = name
        self.poll_interval = poll_interval_seconds
        self.batch_size = batch_size
        self.max_retries = max_retries
        self.running = False
        self.processed_count = 0
        self.error_count = 0
        self.started_at: datetime = None
        
    async def start(self):
        """Start the worker loop"""
        self.running = True
        self.started_at = datetime.utcnow()
        
        # Handle graceful shutdown
        for sig in (signal.SIGTERM, signal.SIGINT):
            asyncio.get_event_loop().add_signal_handler(
                sig, lambda: asyncio.create_task(self.stop())
            )
        
        logger.info(f"Worker {self.name} starting")
        
        while self.running:
            try:
                items = await self.fetch_items()
                
                if items:
                    for item in items:
                        try:
                            await self.process_item(item)
                            self.processed_count += 1
                        except Exception as e:
                            self.error_count += 1
                            await self.handle_error(item, e)
                else:
                    await asyncio.sleep(self.poll_interval)
                    
            except Exception as e:
                logger.error(f"Worker {self.name} error: {e}")
                await asyncio.sleep(self.poll_interval * 2)
    
    async def stop(self):
        """Graceful shutdown"""
        logger.info(f"Worker {self.name} stopping")
        self.running = False
    
    @abstractmethod
    async def fetch_items(self) -> list:
        """Fetch items to process"""
        pass
    
    @abstractmethod
    async def process_item(self, item) -> None:
        """Process a single item"""
        pass
    
    async def handle_error(self, item, error: Exception) -> None:
        """Handle processing error"""
        logger.error(f"Error processing item: {error}", item=item)
    
    def health_check(self) -> dict:
        """Return health status"""
        return {
            "name": self.name,
            "running": self.running,
            "started_at": self.started_at.isoformat() if self.started_at else None,
            "processed_count": self.processed_count,
            "error_count": self.error_count,
            "error_rate": self.error_count / max(self.processed_count, 1),
        }
```

---

## M11: OUTBOX WORKER

```python
class OutboxWorker(BaseWorker):
    """Process pending messages in outbox"""
    
    def __init__(self, db_pool, twilio_client, email_client):
        super().__init__(
            name="outbox_worker",
            poll_interval_seconds=2.0,
            batch_size=20,
            max_retries=3
        )
        self.db = db_pool
        self.twilio = twilio_client
        self.email = email_client
    
    async def fetch_items(self) -> list[OutboxItem]:
        """Fetch pending messages, locked for processing"""
        async with self.db.acquire() as conn:
            rows = await conn.fetch("""
                SELECT * FROM outbox
                WHERE status = 'pending'
                  AND scheduled_at <= NOW()
                  AND attempts < $1
                ORDER BY priority DESC, created_at ASC
                LIMIT $2
                FOR UPDATE SKIP LOCKED
            """, self.max_retries, self.batch_size)
            
            return [OutboxItem(**row) for row in rows]
    
    async def process_item(self, item: OutboxItem) -> None:
        """Send message via appropriate channel"""
        
        try:
            if item.channel == 'sms':
                result = await self.twilio.send_sms(
                    to=item.to_address,
                    body=item.body,
                    from_=item.from_address
                )
                external_id = result.sid
                
            elif item.channel == 'email':
                result = await self.email.send(
                    to=item.to_address,
                    subject=item.subject,
                    body=item.body,
                    from_=item.from_address
                )
                external_id = result.message_id
            
            await self._mark_sent(item.id, external_id)
            
        except Exception as e:
            await self._mark_failed(item.id, str(e))
            raise
    
    async def _mark_sent(self, item_id: str, external_id: str):
        async with self.db.acquire() as conn:
            await conn.execute("""
                UPDATE outbox
                SET status = 'sent',
                    sent_at = NOW(),
                    external_id = $2,
                    updated_at = NOW()
                WHERE id = $1
            """, item_id, external_id)
    
    async def _mark_failed(self, item_id: str, error: str):
        async with self.db.acquire() as conn:
            result = await conn.fetchrow("""
                UPDATE outbox
                SET attempts = attempts + 1,
                    last_error = $2,
                    next_retry_at = NOW() + (INTERVAL '1 minute' * POW(2, attempts)),
                    updated_at = NOW()
                WHERE id = $1
                RETURNING attempts
            """, item_id, error)
            
            if result['attempts'] >= self.max_retries:
                await self._move_to_dlq(item_id)
    
    async def _move_to_dlq(self, item_id: str):
        async with self.db.acquire() as conn:
            await conn.execute("""
                INSERT INTO dead_letter_queue 
                SELECT *, NOW() as moved_at FROM outbox WHERE id = $1;
                DELETE FROM outbox WHERE id = $1;
            """, item_id)
```

---

## M12: STALE CONVERSATION MONITOR

```python
class StaleConversationMonitor(BaseWorker):
    """Find conversations with no reply in 24+ hours"""
    
    def __init__(self, db_pool, settings):
        super().__init__(
            name="stale_monitor",
            poll_interval_seconds=300.0,  # Every 5 minutes
            batch_size=50
        )
        self.db = db_pool
        self.settings = settings
    
    async def fetch_items(self) -> list[Conversation]:
        async with self.db.acquire() as conn:
            rows = await conn.fetch("""
                SELECT c.* FROM conversations c
                JOIN messages m ON m.conversation_id = c.id
                WHERE c.status = 'open'
                  AND c.is_dnd = false
                  AND m.direction = 'outbound'
                  AND m.created_at = (
                      SELECT MAX(created_at) 
                      FROM messages 
                      WHERE conversation_id = c.id
                  )
                  AND m.created_at < NOW() - INTERVAL '24 hours'
                  AND NOT EXISTS (
                      SELECT 1 FROM contact_tags ct
                      JOIN tags t ON t.id = ct.tag_id
                      WHERE ct.contact_id = c.contact_id
                      AND t.name = 'stale_notified'
                  )
                LIMIT $1
            """, self.batch_size)
            return [Conversation(**row) for row in rows]
    
    async def process_item(self, convo: Conversation) -> None:
        # Add tag to contact
        await self._add_tag(convo.contact_id, 'needs_followup')
        
        # Record notification
        await self._add_tag(convo.contact_id, 'stale_notified')
        
        # Trigger webhook if configured
        if self.settings.stale_webhook_url:
            await self._trigger_webhook(convo)
        
        logger.info(f"Marked stale: conversation {convo.id}")
```

---

## M13: POSITIVE RESPONSE MONITOR

```python
class PositiveResponseMonitor(BaseWorker):
    """Detect hot leads from message content"""
    
    POSITIVE_PATTERNS = [
        r'\byes\b',
        r'\binterested\b',
        r'\bbook\b',
        r'\bcall me\b',
        r'\bhow much\b',
        r'\bwhen can\b',
        r'\bschedule\b',
        r'\bavailable\b',
        r'\bsign me up\b',
        r"\blet's do it\b",
    ]
    
    def __init__(self, db_pool, ai_engine=None):
        super().__init__(
            name="positive_monitor",
            poll_interval_seconds=60.0,  # Every minute
            batch_size=100
        )
        self.db = db_pool
        self.ai_engine = ai_engine
        self.patterns = [re.compile(p, re.IGNORECASE) for p in self.POSITIVE_PATTERNS]
    
    async def fetch_items(self) -> list[Message]:
        async with self.db.acquire() as conn:
            rows = await conn.fetch("""
                SELECT m.* FROM messages m
                WHERE m.direction = 'inbound'
                  AND m.created_at > NOW() - INTERVAL '5 minutes'
                  AND m.analyzed_at IS NULL
                LIMIT $1
            """, self.batch_size)
            return [Message(**row) for row in rows]
    
    async def process_item(self, message: Message) -> None:
        score = self._calculate_score(message.body)
        
        if self.ai_engine and score > 0.3:
            # Use AI for higher confidence
            ai_result = await self.ai_engine.analyze_sentiment(message.body)
            if ai_result.sentiment == 'positive':
                score = min(score + 0.3, 1.0)
        
        # Update message as analyzed
        await self._mark_analyzed(message.id, score)
        
        if score >= 0.7:
            await self._create_hot_lead_alert(message, score)
    
    def _calculate_score(self, text: str) -> float:
        matches = sum(1 for p in self.patterns if p.search(text))
        return min(matches * 0.2, 1.0)
    
    async def _create_hot_lead_alert(self, message: Message, score: float):
        # Add tag
        await self._add_tag(message.contact_id, 'hot_lead')
        
        # Send notification
        await self._notify_team(message, score)
```

---

## RUNNING WORKERS

```python
# runner.py
import asyncio
from workers.outbox import OutboxWorker
from workers.monitors import StaleConversationMonitor, PositiveResponseMonitor

async def main():
    # Initialize dependencies
    db_pool = await create_pool()
    twilio = TwilioClient()
    email = EmailClient()
    
    # Create workers
    workers = [
        OutboxWorker(db_pool, twilio, email),
        StaleConversationMonitor(db_pool, settings),
        PositiveResponseMonitor(db_pool, ai_engine),
    ]
    
    # Run all workers concurrently
    await asyncio.gather(*[w.start() for w in workers])

if __name__ == "__main__":
    asyncio.run(main())
```

**Run command:**
```bash
python -m workers.runner
```

---

## LINE ESTIMATES

| Directory | Files | Lines |
|-----------|-------|-------|
| base.py | 1 | 150 |
| config.py | 1 | 50 |
| health.py | 1 | 100 |
| outbox/ | 4 | 500 |
| monitors/ | 4 | 600 |
| scheduled/ | 3 | 400 |
| runner.py | 1 | 100 |
| **TOTAL** | **15** | **1,900** |
