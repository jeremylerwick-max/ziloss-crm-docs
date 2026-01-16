# MODULE M8: AI ENGINE

**Estimated Lines:** 2,200
**Estimated Time:** 2-3 days
**Dependencies:** M5 (conversations), M4 (contacts)

---

## PURPOSE

Analyze incoming messages, detect intent/sentiment, and generate contextual AI responses.

```
Incoming SMS → AI analyzes intent → Generates response → Sends reply
```

---

## FILE STRUCTURE

```
/modules/ai_engine/
├── __init__.py                 # Exports
├── models.py                   # Pydantic models
├── intent_classifier.py        # Detect message intent
├── sentiment_analyzer.py       # Detect emotional tone
├── response_generator.py       # Generate AI responses
├── conversation_context.py     # Build context from history
├── prompt_templates.py         # System prompts per scenario
├── settings.py                 # Per-location AI config
├── executor.py                 # Main orchestration
├── token_tracker.py            # Track usage/costs
└── tests/
    ├── test_intent.py
    ├── test_sentiment.py
    └── test_generator.py
```

---

## INTENT CLASSIFICATION

```python
class MessageIntent(Enum):
    # Booking intents
    BOOKING_REQUEST = "booking_request"      # "I want to schedule..."
    BOOKING_CONFIRM = "booking_confirm"      # "Yes that time works"
    BOOKING_CANCEL = "booking_cancel"        # "I need to cancel"
    BOOKING_RESCHEDULE = "booking_reschedule" # "Can we move it to..."
    
    # Question intents
    QUESTION_PRICING = "question_pricing"    # "How much does..."
    QUESTION_HOURS = "question_hours"        # "What time are you open"
    QUESTION_LOCATION = "question_location"  # "Where are you located"
    QUESTION_GENERAL = "question_general"    # Other questions
    
    # Interest signals
    POSITIVE_INTEREST = "positive_interest"  # "Yes I'm interested"
    NEGATIVE_INTEREST = "negative_interest"  # "No thanks"
    
    # Compliance
    OPT_OUT = "opt_out"                      # "STOP", "unsubscribe"
    OPT_IN = "opt_in"                        # "START", "subscribe"
    
    # Other
    COMPLAINT = "complaint"                   # Negative feedback
    THANKS = "thanks"                         # "Thank you"
    GREETING = "greeting"                     # "Hi", "Hello"
    SPAM = "spam"                             # Gibberish, wrong number
    UNKNOWN = "unknown"                       # Can't classify
```

---

## FUNCTION SIGNATURES

### classify_intent()
```python
async def classify_intent(
    message: str,
    conversation_history: list[Message] = None,
    contact: Contact = None,
    use_ai: bool = True  # False = regex only, True = Claude
) -> IntentResult:
    """
    Returns:
        IntentResult(
            primary_intent: MessageIntent,
            confidence: float (0-1),
            secondary_intents: list[MessageIntent],
            extracted_entities: dict  # dates, times, names, etc.
        )
    """
```

### analyze_sentiment()
```python
async def analyze_sentiment(
    message: str,
    context: list[Message] = None
) -> SentimentResult:
    """
    Returns:
        SentimentResult(
            sentiment: "positive" | "neutral" | "negative" | "urgent",
            confidence: float (0-1),
            emotions: list[str]  # happy, frustrated, confused, etc.
        )
    """
```

### generate_response()
```python
async def generate_response(
    message: str,
    contact: Contact,
    conversation: Conversation,
    intent: IntentResult,
    settings: AISettings
) -> ResponseResult:
    """
    Returns:
        ResponseResult(
            response: str,
            confidence: float (0-1),
            should_send: bool,  # False if needs human review
            suggested_actions: list[str]  # "book_appointment", "add_tag", etc.
        )
    """
```

---

## DATABASE TABLES

```sql
-- Per-location AI configuration
CREATE TABLE ai_settings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id),
    enabled BOOLEAN DEFAULT true,
    
    -- Model selection
    model VARCHAR(50) DEFAULT 'claude-3-haiku',
    fallback_model VARCHAR(50) DEFAULT 'claude-3-sonnet',
    
    -- Behavior
    tone VARCHAR(20) DEFAULT 'friendly',  -- friendly, professional, casual
    auto_respond_enabled BOOLEAN DEFAULT true,
    response_delay_seconds INT DEFAULT 30,  -- Humanize timing
    
    -- Limits
    max_tokens_per_response INT DEFAULT 150,
    max_responses_per_contact_per_day INT DEFAULT 10,
    
    -- Features
    intent_detection_enabled BOOLEAN DEFAULT true,
    sentiment_analysis_enabled BOOLEAN DEFAULT true,
    entity_extraction_enabled BOOLEAN DEFAULT true,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(location_id)
);

-- AI interaction logs
CREATE TABLE ai_conversation_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id),
    contact_id UUID REFERENCES contacts(id),
    message_id UUID REFERENCES messages(id),
    
    -- Analysis results
    detected_intent VARCHAR(50),
    intent_confidence FLOAT,
    detected_sentiment VARCHAR(20),
    sentiment_confidence FLOAT,
    extracted_entities JSONB DEFAULT '{}',
    
    -- Response
    ai_response TEXT,
    response_sent BOOLEAN DEFAULT false,
    response_message_id UUID REFERENCES messages(id),
    
    -- Performance
    model_used VARCHAR(50),
    input_tokens INT,
    output_tokens INT,
    latency_ms INT,
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_ai_log_location ON ai_conversation_log(location_id);
CREATE INDEX idx_ai_log_contact ON ai_conversation_log(contact_id);
CREATE INDEX idx_ai_log_created ON ai_conversation_log(created_at);
```

---

## API ENDPOINTS

```
POST /api/ai/analyze       - Analyze a message, return intent/sentiment
POST /api/ai/generate      - Generate a response for a message
PUT  /api/ai/settings      - Update AI settings for a location
GET  /api/ai/settings      - Get AI settings
GET  /api/ai/logs          - Get AI interaction history
GET  /api/ai/stats         - Get usage stats (tokens, costs)
```

---

## TEST CRITERIA

- [ ] Detects "I want to book an appointment" as `booking_request`
- [ ] Detects "STOP" as `opt_out`
- [ ] Detects "Yes I'm interested" as `positive_interest`
- [ ] Generates contextual response using contact history
- [ ] Respects per-location settings (tone, model, etc.)
- [ ] Logs all AI interactions with token counts
- [ ] Handles rate limits gracefully
- [ ] Falls back to regex when AI unavailable

---

## LINE ESTIMATES

| File | Lines | Complexity |
|------|-------|------------|
| intent_classifier.py | 350 | High |
| sentiment_analyzer.py | 150 | Medium |
| response_generator.py | 400 | High |
| conversation_context.py | 200 | Medium |
| prompt_templates.py | 300 | Low |
| settings.py | 150 | Low |
| executor.py | 400 | High |
| token_tracker.py | 100 | Low |
| models.py | 150 | Low |
| **TOTAL** | **2,200** | |
