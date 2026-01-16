# Claude Code Prompt: M8 AI Engine

## Context

You are building the AI Engine module for Ziloss CRM, a GoHighLevel competitor. This module powers all AI-driven features: intent classification, sentiment analysis, and response generation.

**Project Location:** `/Users/mac/Desktop/agent-orchestrator`
**Module Location:** `/Users/mac/Desktop/agent-orchestrator/src/modules/ai_engine/`
**Estimated Lines:** 2,200
**Time Estimate:** 2-3 days

## Tech Stack

- **Language:** Python 3.11+
- **Framework:** FastAPI
- **Database:** PostgreSQL (async via asyncpg)
- **AI Provider:** Anthropic Claude API (claude-3-haiku for speed, claude-sonnet-4-20250514 for quality)
- **Validation:** Pydantic v2
- **Testing:** pytest + pytest-asyncio

## Module Purpose

The AI Engine provides:
1. **Intent Classification** - Categorize inbound messages (booking request, opt-out, question, etc.)
2. **Sentiment Analysis** - Detect positive/negative/neutral/urgent sentiment
3. **Response Generation** - Generate contextual replies using conversation history
4. **AI Settings** - Per-location configuration for AI behavior

## File Structure to Create

```
src/modules/ai_engine/
├── __init__.py
├── router.py              # FastAPI routes
├── service.py             # Business logic
├── models.py              # Pydantic models
├── schemas.py             # Database schemas
├── prompts.py             # System prompts for Claude
├── intent_classifier.py   # Intent classification logic
├── sentiment_analyzer.py  # Sentiment analysis logic
├── response_generator.py  # Response generation logic
├── exceptions.py          # Custom exceptions
└── tests/
    ├── __init__.py
    ├── test_intent.py
    ├── test_sentiment.py
    ├── test_response.py
    └── conftest.py
```

## Database Schema

Create migration file at `src/migrations/versions/xxx_ai_engine.py`:

```sql
-- AI Settings per location
CREATE TABLE ai_settings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    
    -- Model configuration
    intent_model VARCHAR(50) DEFAULT 'claude-3-haiku-20240307',
    response_model VARCHAR(50) DEFAULT 'claude-sonnet-4-20250514',
    
    -- Behavior settings
    auto_respond BOOLEAN DEFAULT FALSE,
    auto_respond_channels JSONB DEFAULT '["sms", "email"]',
    response_delay_seconds INT DEFAULT 30,
    
    -- Tone and style
    tone VARCHAR(20) DEFAULT 'professional' CHECK (tone IN ('professional', 'friendly', 'casual', 'formal')),
    business_context TEXT,
    
    -- Constraints
    max_response_length INT DEFAULT 160,
    forbidden_topics JSONB DEFAULT '[]',
    required_disclosures TEXT,
    
    -- Rate limiting
    max_ai_messages_per_lead_per_day INT DEFAULT 10,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    UNIQUE(location_id)
);

-- AI Conversation Log (append-only)
CREATE TABLE ai_conversation_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id),
    contact_id UUID NOT NULL REFERENCES contacts(id),
    conversation_id UUID REFERENCES conversations(id),
    
    input_message TEXT NOT NULL,
    input_channel VARCHAR(20) NOT NULL,
    
    intent VARCHAR(50),
    intent_confidence FLOAT,
    sentiment VARCHAR(20),
    sentiment_score FLOAT,
    
    response_generated TEXT,
    response_approved BOOLEAN,
    response_sent BOOLEAN DEFAULT FALSE,
    response_sent_at TIMESTAMPTZ,
    
    model_used VARCHAR(50),
    tokens_input INT,
    tokens_output INT,
    latency_ms INT,
    cost_usd DECIMAL(10, 6),
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_ai_log_location ON ai_conversation_log(location_id);
CREATE INDEX idx_ai_log_contact ON ai_conversation_log(contact_id);
CREATE INDEX idx_ai_log_intent ON ai_conversation_log(intent);
```

## Intent Types (17 categories)

```python
class IntentType(str, Enum):
    BOOKING_REQUEST = "booking_request"
    BOOKING_CONFIRM = "booking_confirm"
    BOOKING_CANCEL = "booking_cancel"
    BOOKING_RESCHEDULE = "booking_reschedule"
    QUESTION_PRICING = "question_pricing"
    QUESTION_HOURS = "question_hours"
    QUESTION_LOCATION = "question_location"
    QUESTION_GENERAL = "question_general"
    POSITIVE_INTEREST = "positive_interest"
    NEGATIVE_INTEREST = "negative_interest"
    OPT_OUT = "opt_out"
    OPT_IN = "opt_in"
    COMPLAINT = "complaint"
    THANKS = "thanks"
    GREETING = "greeting"
    SPAM = "spam"
    UNKNOWN = "unknown"
```

## API Endpoints

```
POST /api/ai/analyze          - Analyze message (intent + sentiment)
POST /api/ai/analyze/batch    - Batch analysis (max 10)
POST /api/ai/generate         - Generate response
GET  /api/ai/settings         - Get AI settings
PUT  /api/ai/settings         - Update AI settings
GET  /api/ai/stats            - Usage statistics
GET  /api/ai/logs             - Conversation logs
```

## Key Implementation Details

### Intent Classifier
- Use Claude Haiku for speed (~100ms)
- Return confidence score 0-1
- Include reasoning for debugging

### Sentiment Analyzer
- Score from -1 (negative) to +1 (positive)
- Separate urgency score 0-1
- Detect: positive, neutral, negative, urgent

### Response Generator
- Use Claude Sonnet for quality
- Respect max_length (default 160 for SMS)
- Include conversation history for context
- Check auto-send eligibility

### Rate Limiting
- Per-contact daily limit
- Configurable per location
- Return 429 when exceeded

## Testing Requirements

1. Test all 17 intent types with sample messages
2. Test sentiment edge cases
3. Test response truncation
4. Test rate limiting
5. Integration tests with mock Claude

## Success Criteria

- [ ] All intent types classified correctly
- [ ] Sentiment accuracy >85%
- [ ] Response respects max_length
- [ ] Rate limiting works
- [ ] All tests pass
- [ ] API documentation complete
