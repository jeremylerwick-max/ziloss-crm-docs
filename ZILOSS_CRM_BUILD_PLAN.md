# ZILOSS AI CRM - BUILD PRIORITY PLAN

## Option B Confirmed: Full GHL Replacement

**Goal:** AI-native CRM that replaces GoHighLevel entirely.  
**Foundation:** agent-orchestrator codebase (70% infrastructure done)  
**Timeline:** 8-12 weeks to functional MVP

---

## PHASE 0: IMMEDIATE (This Week)
### Get the existing system running

Before building new features, verify what you have works.

```bash
cd /Users/mac/Desktop/agent-orchestrator
docker-compose up -d
uvicorn orchestrator.api:app --reload
```

**Tasks:**
- [ ] Start PostgreSQL, Redis, API
- [ ] Verify LangGraph orchestration works
- [ ] Test natural language interface module
- [ ] Confirm workflow engine executes tasks
- [ ] Document what's broken vs working

**Why first:** No point building on broken foundation.

---

## PHASE 1: CORE DATA MODEL (Week 1-2)
### Build the CRM backbone

Create GHL-equivalent data structures in YOUR database.

### 1.1 Database Schema

**File:** `schema/010_crm_core.sql`

```sql
-- Locations (tenants/workspaces)
CREATE TABLE locations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    timezone VARCHAR(50) DEFAULT 'America/Chicago',
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Contacts (leads/customers)
CREATE TABLE contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID REFERENCES locations(id) ON DELETE CASCADE,
    
    -- Core fields
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(255),
    phone VARCHAR(20),
    
    -- Address
    address1 VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(50),
    postal_code VARCHAR(20),
    country VARCHAR(50) DEFAULT 'US',
    
    -- Status
    type VARCHAR(20) DEFAULT 'lead', -- lead, customer, vendor
    source VARCHAR(100),
    assigned_to UUID,
    
    -- DND
    dnd BOOLEAN DEFAULT FALSE,
    dnd_settings JSONB DEFAULT '{}',
    
    -- Flexible fields
    custom_fields JSONB DEFAULT '{}',
    
    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    last_contacted_at TIMESTAMPTZ,
    
    -- Indexes
    CONSTRAINT unique_phone_per_location UNIQUE (location_id, phone),
    CONSTRAINT unique_email_per_location UNIQUE (location_id, email)
);

CREATE INDEX idx_contacts_location ON contacts(location_id);
CREATE INDEX idx_contacts_phone ON contacts(phone);
CREATE INDEX idx_contacts_email ON contacts(email);
CREATE INDEX idx_contacts_custom_fields ON contacts USING GIN(custom_fields);

-- Tags
CREATE TABLE tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID REFERENCES locations(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    color VARCHAR(7) DEFAULT '#3B82F6',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    CONSTRAINT unique_tag_per_location UNIQUE (location_id, name)
);

-- Contact Tags (many-to-many)
CREATE TABLE contact_tags (
    contact_id UUID REFERENCES contacts(id) ON DELETE CASCADE,
    tag_id UUID REFERENCES tags(id) ON DELETE CASCADE,
    added_at TIMESTAMPTZ DEFAULT NOW(),
    added_by VARCHAR(100), -- 'system', 'ai', 'user:xxx'
    PRIMARY KEY (contact_id, tag_id)
);

CREATE INDEX idx_contact_tags_contact ON contact_tags(contact_id);
CREATE INDEX idx_contact_tags_tag ON contact_tags(tag_id);

-- Conversations
CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID REFERENCES locations(id) ON DELETE CASCADE,
    contact_id UUID REFERENCES contacts(id) ON DELETE CASCADE,
    
    -- Channel info
    channel VARCHAR(20) NOT NULL, -- sms, email, facebook, instagram, whatsapp
    channel_id VARCHAR(255), -- external thread ID if applicable
    
    -- Status
    status VARCHAR(20) DEFAULT 'open', -- open, closed, snoozed
    unread_count INT DEFAULT 0,
    assigned_to UUID,
    
    -- Last message cache (for list views)
    last_message_body TEXT,
    last_message_direction VARCHAR(10), -- inbound, outbound
    last_message_at TIMESTAMPTZ,
    
    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_conversations_location ON conversations(location_id);
CREATE INDEX idx_conversations_contact ON conversations(contact_id);
CREATE INDEX idx_conversations_status ON conversations(status);

-- Messages
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID REFERENCES conversations(id) ON DELETE CASCADE,
    
    -- Content
    direction VARCHAR(10) NOT NULL, -- inbound, outbound
    body TEXT,
    media_urls TEXT[], -- array of attachment URLs
    
    -- Channel-specific
    channel VARCHAR(20) NOT NULL,
    external_id VARCHAR(255), -- Twilio SID, etc
    
    -- Status
    status VARCHAR(20) DEFAULT 'sent', -- queued, sent, delivered, read, failed
    error_message TEXT,
    
    -- AI metadata
    ai_generated BOOLEAN DEFAULT FALSE,
    ai_model VARCHAR(50),
    sentiment VARCHAR(20), -- positive, negative, neutral
    intent VARCHAR(100), -- booking, question, complaint, etc
    
    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    delivered_at TIMESTAMPTZ,
    read_at TIMESTAMPTZ
);

CREATE INDEX idx_messages_conversation ON messages(conversation_id);
CREATE INDEX idx_messages_created ON messages(created_at DESC);

-- Appointments
CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID REFERENCES locations(id) ON DELETE CASCADE,
    contact_id UUID REFERENCES contacts(id) ON DELETE CASCADE,
    
    -- Scheduling
    title VARCHAR(255),
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    timezone VARCHAR(50),
    
    -- Status
    status VARCHAR(20) DEFAULT 'scheduled', -- scheduled, confirmed, completed, cancelled, no_show
    
    -- Details
    notes TEXT,
    calendar_id VARCHAR(255), -- external calendar ID
    external_id VARCHAR(255), -- Google Calendar event ID, etc
    
    -- Reminders
    reminder_sent BOOLEAN DEFAULT FALSE,
    reminder_sent_at TIMESTAMPTZ,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_appointments_location ON appointments(location_id);
CREATE INDEX idx_appointments_contact ON appointments(contact_id);
CREATE INDEX idx_appointments_start ON appointments(start_time);
```

### 1.2 Python Models

**File:** `modules/crm_core/models.py`

```python
from pydantic import BaseModel, Field
from typing import Optional, List, Dict, Any
from datetime import datetime
from enum import Enum

class ContactType(str, Enum):
    LEAD = "lead"
    CUSTOMER = "customer"
    VENDOR = "vendor"

class Contact(BaseModel):
    id: Optional[str] = None
    location_id: str
    first_name: Optional[str] = None
    last_name: Optional[str] = None
    email: Optional[str] = None
    phone: Optional[str] = None
    address1: Optional[str] = None
    city: Optional[str] = None
    state: Optional[str] = None
    postal_code: Optional[str] = None
    type: ContactType = ContactType.LEAD
    source: Optional[str] = None
    dnd: bool = False
    dnd_settings: Dict[str, Any] = {}
    custom_fields: Dict[str, Any] = {}
    tags: List[str] = []
    created_at: Optional[datetime] = None
    updated_at: Optional[datetime] = None

class MessageDirection(str, Enum):
    INBOUND = "inbound"
    OUTBOUND = "outbound"

class Message(BaseModel):
    id: Optional[str] = None
    conversation_id: str
    direction: MessageDirection
    body: str
    channel: str = "sms"
    status: str = "sent"
    ai_generated: bool = False
    created_at: Optional[datetime] = None
```

### 1.3 CRUD API Endpoints

**File:** `orchestrator/crm_api.py`

```python
from fastapi import APIRouter, HTTPException
from typing import List, Optional

router = APIRouter(prefix="/crm", tags=["CRM"])

@router.post("/contacts")
async def create_contact(contact: ContactCreate) -> Contact:
    ...

@router.get("/contacts/{contact_id}")
async def get_contact(contact_id: str) -> Contact:
    ...

@router.get("/contacts")
async def list_contacts(
    location_id: str,
    tag: Optional[str] = None,
    search: Optional[str] = None,
    limit: int = 100
) -> List[Contact]:
    ...

@router.post("/contacts/{contact_id}/tags/{tag_name}")
async def add_tag(contact_id: str, tag_name: str):
    ...

@router.delete("/contacts/{contact_id}/tags/{tag_name}")
async def remove_tag(contact_id: str, tag_name: str):
    ...

@router.get("/conversations")
async def list_conversations(
    location_id: str,
    status: Optional[str] = None,
    contact_id: Optional[str] = None
) -> List[Conversation]:
    ...

@router.post("/messages")
async def send_message(message: MessageCreate) -> Message:
    # This triggers the communication layer
    ...
```

**Deliverable:** Working CRUD API for contacts, tags, conversations, messages, appointments.

---

## PHASE 2: COMMUNICATION LAYER (Week 2-3)
### Send and receive SMS via Twilio

### 2.1 Twilio Module

**File:** `modules/twilio_sms/executor.py`

```python
from twilio.rest import Client
import os

class TwilioSMS:
    def __init__(self):
        self.client = Client(
            os.getenv('TWILIO_ACCOUNT_SID'),
            os.getenv('TWILIO_AUTH_TOKEN')
        )
        self.from_number = os.getenv('TWILIO_PHONE_NUMBER')
    
    def send(self, to: str, body: str) -> dict:
        message = self.client.messages.create(
            body=body,
            from_=self.from_number,
            to=to
        )
        return {
            "sid": message.sid,
            "status": message.status,
            "to": to
        }
    
    def receive_webhook(self, payload: dict) -> dict:
        # Parse incoming SMS webhook from Twilio
        return {
            "from": payload.get("From"),
            "to": payload.get("To"),
            "body": payload.get("Body"),
            "sid": payload.get("MessageSid")
        }
```

### 2.2 Webhook Receiver

**File:** `orchestrator/webhooks.py`

```python
from fastapi import APIRouter, Request

router = APIRouter(prefix="/webhooks", tags=["Webhooks"])

@router.post("/twilio/inbound")
async def twilio_inbound(request: Request):
    """Receive inbound SMS from Twilio."""
    form_data = await request.form()
    
    # 1. Find or create contact by phone
    # 2. Find or create conversation
    # 3. Create message record
    # 4. Trigger AI response workflow
    
    return {"status": "received"}
```

### 2.3 Wire to Workflow Engine

When message comes in:
1. Save to database
2. Check if AI should respond (not DND, has active automation)
3. If yes, trigger AI conversation workflow
4. AI generates response
5. Send via Twilio
6. Save outbound message

**Deliverable:** Send SMS, receive SMS, store in conversations table.

---

## PHASE 3: AI CONVERSATION ENGINE (Week 3-4)
### Make the AI actually talk to leads

### 3.1 Connect Natural Language Interface to CRM

Your existing NLI module has intent recognition and entity extraction.
Now connect it to TAKE ACTIONS.

**File:** `modules/ai_conversation/executor.py`

```python
class AIConversation:
    def __init__(self, contact_id: str, goal: str):
        self.contact = get_contact(contact_id)
        self.conversation = get_or_create_conversation(contact_id)
        self.goal = goal  # "book_appointment", "qualify_lead", "handle_objection"
        self.history = get_message_history(self.conversation.id)
    
    async def respond(self, inbound_message: str) -> str:
        # 1. Analyze intent
        intent = analyze_intent(inbound_message)
        
        # 2. Check for stop words
        if intent == "opt_out":
            await mark_dnd(self.contact.id)
            return None  # Don't respond
        
        # 3. Generate contextual response
        prompt = self._build_prompt(inbound_message, intent)
        response = await call_llm(prompt)
        
        # 4. Extract any actions (book appointment, add tag, etc)
        actions = extract_actions(response)
        await execute_actions(actions, self.contact.id)
        
        return response.message
    
    def _build_prompt(self, message: str, intent: str) -> str:
        return f"""
You are an AI assistant helping book appointments for a window replacement company.

Contact: {self.contact.first_name} {self.contact.last_name}
Phone: {self.contact.phone}
Goal: {self.goal}

Conversation history:
{self._format_history()}

Their latest message: {message}
Detected intent: {intent}

Respond naturally. If they want to book, offer available times.
If they say stop/no/not interested, acknowledge politely and end.
Keep responses under 160 characters when possible (SMS friendly).
"""
```

### 3.2 Booking Flow

```python
async def handle_booking_intent(contact_id: str, message: str):
    # 1. Check calendar availability
    slots = await get_available_slots(days=7)
    
    # 2. If they mentioned a specific time, try to match
    requested_time = extract_time(message)
    if requested_time and is_available(requested_time):
        await create_appointment(contact_id, requested_time)
        await add_tag(contact_id, "appointment_booked")
        return f"Great! You're confirmed for {format_time(requested_time)}. See you then!"
    
    # 3. Otherwise offer options
    return f"I have openings on {format_slots(slots[:3])}. Which works best?"
```

**Deliverable:** AI can hold conversations, book appointments, handle opt-outs.

---

## PHASE 4: EVENT-DRIVEN WORKFLOWS (Week 4-5)
### Automate based on triggers

### 4.1 Event System

**File:** `orchestrator/events.py`

```python
from enum import Enum
from typing import Callable, Dict, List

class EventType(str, Enum):
    CONTACT_CREATED = "contact.created"
    CONTACT_UPDATED = "contact.updated"
    TAG_ADDED = "tag.added"
    TAG_REMOVED = "tag.removed"
    MESSAGE_RECEIVED = "message.received"
    MESSAGE_SENT = "message.sent"
    APPOINTMENT_BOOKED = "appointment.booked"
    APPOINTMENT_CANCELLED = "appointment.cancelled"
    DND_ENABLED = "dnd.enabled"

class EventBus:
    def __init__(self):
        self.handlers: Dict[EventType, List[Callable]] = {}
    
    def on(self, event_type: EventType, handler: Callable):
        if event_type not in self.handlers:
            self.handlers[event_type] = []
        self.handlers[event_type].append(handler)
    
    async def emit(self, event_type: EventType, payload: dict):
        for handler in self.handlers.get(event_type, []):
            await handler(payload)

# Global event bus
events = EventBus()

# Example: When tag added, trigger workflow
@events.on(EventType.TAG_ADDED)
async def on_tag_added(payload):
    contact_id = payload["contact_id"]
    tag_name = payload["tag_name"]
    
    if tag_name == "activate_ai":
        await start_ai_conversation(contact_id, goal="book_appointment")
    
    if tag_name == "appointment_booked":
        await send_confirmation_sms(contact_id)
        await add_tag(contact_id, "confirmation_sent")
```

### 4.2 Automation Rules (Natural Language Defined)

Instead of rigid workflow builders, define automations in plain English:

```python
automations = [
    {
        "name": "New Lead Follow-up",
        "trigger": "When a contact is created with source 'facebook'",
        "actions": [
            "Wait 2 minutes",
            "Send SMS: 'Hi {first_name}, thanks for your interest! When's a good time to chat about your window project?'",
            "Add tag 'initial_outreach_sent'"
        ]
    },
    {
        "name": "AI Activation",
        "trigger": "When tag 'activate_ai' is added",
        "actions": [
            "Start AI conversation with goal 'book_appointment'",
            "Add tag 'ai_active'"
        ]
    },
    {
        "name": "Stop Word Handler",
        "trigger": "When message received contains 'stop' or 'unsubscribe'",
        "actions": [
            "Enable DND",
            "Remove tag 'ai_active'",
            "Add tag 'opted_out'"
        ]
    }
]
```

The AI parses these into executable workflows.

**Deliverable:** Event-driven automation system with natural language rules.

---

## PHASE 5: NATURAL LANGUAGE CONTROL (Week 5-6)
### "Talk to your CRM"

### 5.1 Command Interface

Connect your existing NLI module to CRM actions:

```
User: "Show me all leads from Facebook this week"
→ Intent: query_contacts
→ Entities: source=facebook, date_range=this_week
→ Action: SELECT * FROM contacts WHERE source='facebook' AND created_at > now() - interval '7 days'
→ Response: "Found 47 leads from Facebook this week. 12 have responded, 3 booked appointments."

User: "Text everyone who hasn't responded in 3 days"
→ Intent: bulk_message
→ Entities: filter=no_response, days=3
→ Action: Find contacts, confirm count, send messages
→ Response: "Found 23 contacts. Here's the message I'll send: '...' Send now?"

User: "Book John Smith for tomorrow at 2pm"
→ Intent: create_appointment
→ Entities: contact=John Smith, date=tomorrow, time=2pm
→ Action: Create appointment, send confirmation
→ Response: "Done. John Smith is booked for tomorrow at 2:00 PM. Confirmation sent."
```

### 5.2 Query Builder

```python
def natural_language_to_query(nl_query: str) -> dict:
    """Convert natural language to CRM query."""
    
    # Use LLM to parse
    prompt = f"""
    Convert this natural language query to a structured CRM query:
    
    "{nl_query}"
    
    Return JSON with:
    - entity: contacts | conversations | appointments
    - filters: array of {{field, operator, value}}
    - sort: {{field, direction}}
    - limit: number
    """
    
    return parse_llm_response(call_llm(prompt))
```

**Deliverable:** Talk to your CRM in plain English, execute any action.

---

## PHASE 6: UI (Week 6-8)
### Minimum viable interface

### 6.1 Priority Views

1. **Inbox** - All conversations, sorted by most recent
2. **Contact Detail** - Full contact info, conversation history, tags
3. **Command Bar** - Natural language input (like Spotlight/Alfred)
4. **Dashboard** - Key metrics (leads today, appointments, response rate)

### 6.2 Tech Stack

You already have `ui/` directory with React setup.

```
ui/
├── src/
│   ├── pages/
│   │   ├── Inbox.tsx
│   │   ├── ContactDetail.tsx
│   │   └── Dashboard.tsx
│   ├── components/
│   │   ├── CommandBar.tsx
│   │   ├── ConversationList.tsx
│   │   ├── MessageThread.tsx
│   │   └── ContactCard.tsx
```

### 6.3 Real-time Updates

WebSocket for live conversation updates:

```python
# orchestrator/websocket.py
from fastapi import WebSocket

@app.websocket("/ws/{location_id}")
async def websocket_endpoint(websocket: WebSocket, location_id: str):
    await websocket.accept()
    # Subscribe to events for this location
    # Push updates when messages arrive, appointments book, etc.
```

**Deliverable:** Working web UI with inbox, contacts, command bar.

---

## PHASE 7: GHL MIGRATION (Week 7-8)
### Import existing Esler data

### 7.1 Migration Script

```python
async def migrate_from_ghl(ghl_api_key: str, location_id: str):
    """Import all data from GHL location."""
    
    # 1. Fetch all contacts
    contacts = await ghl_fetch_all_contacts(ghl_api_key, location_id)
    
    # 2. Import to Ziloss
    for contact in contacts:
        await create_contact(
            location_id=new_location_id,
            first_name=contact['firstName'],
            last_name=contact['lastName'],
            phone=contact['phone'],
            email=contact['email'],
            custom_fields=contact.get('customFields', {}),
            tags=[t['name'] for t in contact.get('tags', [])]
        )
    
    # 3. Fetch conversations
    conversations = await ghl_fetch_all_conversations(ghl_api_key, location_id)
    
    # 4. Import conversations and messages
    # ...
    
    return {"contacts": len(contacts), "conversations": len(conversations)}
```

**Deliverable:** One-click import from GHL.

---

## IMMEDIATE NEXT STEPS (Tomorrow)

1. **Verify agent-orchestrator runs**
   ```bash
   cd /Users/mac/Desktop/agent-orchestrator
   docker-compose up -d
   curl http://localhost:8000/health
   ```

2. **Create CRM schema**
   - Write `schema/010_crm_core.sql`
   - Run migration

3. **Build Contact CRUD**
   - Create contact
   - List contacts
   - Add/remove tags

4. **Test with real data**
   - Import 10 contacts from GHL manually
   - Verify storage and retrieval

---

## SUCCESS METRICS

| Milestone | Target Date | Criteria |
|-----------|-------------|----------|
| Schema + CRUD | Jan 11 | Can create/read/update contacts via API |
| Send SMS | Jan 18 | Can send SMS via Twilio, see in UI |
| Receive SMS | Jan 18 | Inbound SMS creates message record |
| AI Responds | Jan 25 | AI generates contextual responses |
| Book Appointment | Feb 1 | End-to-end booking flow works |
| NL Commands | Feb 8 | "Show me leads from today" works |
| Basic UI | Feb 15 | Inbox + Contact detail views |
| GHL Import | Feb 22 | Can migrate Esler data |
| **MVP Complete** | **Feb 22** | **Full GHL replacement functional** |

---

## WHAT TO BUILD VS BUY

| Component | Build | Buy/Use |
|-----------|-------|---------|
| CRM Data Model | ✅ Build | - |
| API Layer | ✅ Build (have FastAPI) | - |
| SMS | - | Twilio |
| Email | - | SendGrid / Postmark |
| Calendar | - | Google Calendar API |
| Voice | - | Twilio / Vapi.ai |
| AI/LLM | ✅ Build orchestration | Claude API / Local Llama |
| Database | ✅ Build schema | PostgreSQL (have it) |
| Auth | - | Clerk / Auth0 |
| Hosting | - | Railway (already using) |

---

## RISK MITIGATION

| Risk | Mitigation |
|------|------------|
| Scope creep | Strict MVP definition - no features beyond this doc |
| Integration complexity | Start with SMS only, add channels later |
| AI quality | Human-in-loop for first 100 conversations |
| Data loss | Maintain GHL as backup during migration |
| Time | You've already built 70% - this is execution, not invention |

---

*Document created: January 4, 2026*
*For: Jeremy Lerwick / Ziloss Technologies*
