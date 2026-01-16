# Claude Code Prompts - Week 1 Modules

## Overview

These prompts are designed for Claude Code instances to build the Week 1 modules of Ziloss CRM. Each prompt is self-contained with:
- Complete context
- File structure
- Database schemas
- Implementation details
- Testing requirements
- Success criteria

## Prompt Files

| File | Module | Lines | Time |
|------|--------|-------|------|
| `CC_PROMPT_EVENT_LEDGER.md` | Event Ledger (AGO Foundation) | 800 | 1 day |
| `CC_PROMPT_M8_AI_ENGINE.md` | AI Engine (Intent/Sentiment/Response) | 2,200 | 2-3 days |
| `CC_PROMPT_M10_APPOINTMENTS.md` | Appointments System | 3,000 | 2 days |
| `CC_PROMPT_M11_13_WORKERS.md` | Background Workers | 1,900 | 2 days |

**Total: ~7,900 lines, 7-8 days**

## Execution Order

### Phase 1: Foundation (Day 1)
```
CC-1: Event Ledger
└── Creates append-only event log
└── Enables deterministic replay
└── Foundation for AGO
```

### Phase 2: Core Features (Days 2-5)
```
CC-2: AI Engine
└── Depends on: Event Ledger
└── Intent classification
└── Sentiment analysis
└── Response generation

CC-3: Appointments
└── Depends on: Event Ledger
└── Booking engine
└── Availability calculation
└── Reminders
```

### Phase 3: Workers (Days 5-7)
```
CC-4: Background Workers
└── Depends on: Event Ledger, AI Engine
└── Outbox processor
└── Stale conversation monitor
└── Hot lead detector
```

## How to Use These Prompts

### Option 1: Claude Code CLI
```bash
# In each terminal, provide the prompt file:
cd /Users/mac/Desktop/agent-orchestrator
cat docs/prompts/CC_PROMPT_EVENT_LEDGER.md | claude

# Or reference it:
claude "Read docs/prompts/CC_PROMPT_M8_AI_ENGINE.md and implement the module"
```

### Option 2: Claude Code Web Interface
1. Open Claude Code
2. Copy entire prompt content
3. Paste and execute

### Option 3: Parallel Execution
Run multiple Claude Code instances simultaneously:
```
Terminal 1: Event Ledger (CC-1)
Terminal 2: AI Engine (CC-2) [wait for CC-1 to finish DB schema]
Terminal 3: Appointments (CC-3) [wait for CC-1 to finish DB schema]
Terminal 4: Workers (CC-4) [wait for CC-1, CC-2]
```

## Dependencies Between Modules

```
Event Ledger (CC-1)
    ├── AI Engine (CC-2)
    │   └── Workers (CC-4)
    └── Appointments (CC-3)
        └── Workers (CC-4)
```

## Database Migration Order

1. Run Event Ledger migrations first
2. Run AI Engine migrations
3. Run Appointments migrations
4. Workers use existing tables

## Shared Dependencies

All modules need:
```python
# Core dependencies
fastapi>=0.104.0
pydantic>=2.0.0
asyncpg>=0.29.0
redis>=5.0.0

# AI Engine specific
anthropic>=0.18.0

# Appointments specific
pytz>=2024.1
python-dateutil>=2.8.0

# Workers specific
twilio>=8.0.0
sendgrid>=6.0.0
```

## Testing Strategy

Each module includes test files. Run in order:
```bash
# After Event Ledger
pytest src/core/events/tests/ -v

# After AI Engine
pytest src/modules/ai_engine/tests/ -v

# After Appointments
pytest src/modules/appointments/tests/ -v

# After Workers
pytest src/workers/tests/ -v

# Full suite
pytest --cov=src --cov-report=html
```

## Success Metrics

| Module | Coverage Target | Key Tests |
|--------|-----------------|-----------|
| Event Ledger | 90% | Append-only enforcement, replay |
| AI Engine | 80% | Intent classification, response generation |
| Appointments | 80% | Slot calculation, double-booking prevention |
| Workers | 80% | Consent checking, retry logic |

## Notes

1. **Event Ledger is foundational** - All other modules depend on it
2. **Append-only is non-negotiable** - No updates/deletes to events table
3. **Consent is a hard gate** - Workers must check consent before every send
4. **All events go through the ledger** - No direct database mutations elsewhere

## Architecture Reference

See `/docs/architecture/` for:
- `11_AUTONOMOUS_GROWTH_OPERATOR.md` - The 10x feature spec
- `12_AGO_DATA_MODEL.md` - Data model details
- `13_AGO_EXECUTION_ARCHITECTURE.md` - Runtime architecture
