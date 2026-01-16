# MODULE M19: WORKFLOW ENGINE

**Estimated Lines:** 7,700
**Estimated Time:** 1-2 weeks
**Dependencies:** All other modules (this orchestrates everything)

---

## PURPOSE

Visual workflow builder that allows users to create automated sequences:
- Trigger on events (contact created, message received, etc.)
- Execute actions (send SMS, add tag, wait, branch)
- Support conditions and branching logic

**This is the core differentiator from basic CRMs.**

---

## FILE STRUCTURE

```
/modules/workflow_engine/
├── __init__.py
│
├── models/
│   ├── __init__.py
│   ├── workflow.py            # Workflow definition
│   ├── trigger.py             # Trigger types
│   ├── action.py              # Action types
│   ├── condition.py           # Condition/branch types
│   └── execution.py           # Execution state
│
├── triggers/
│   ├── __init__.py
│   ├── base.py                # Base trigger class
│   ├── contact_triggers.py    # Contact created/updated/tagged
│   ├── message_triggers.py    # SMS received/sent
│   ├── appointment_triggers.py # Booked/cancelled/reminded
│   ├── form_triggers.py       # Form submitted
│   ├── schedule_triggers.py   # Time-based (cron)
│   ├── webhook_triggers.py    # External webhook
│   └── manual_triggers.py     # Manual enrollment
│
├── actions/
│   ├── __init__.py
│   ├── base.py                # Base action class
│   ├── messaging/
│   │   ├── send_sms.py
│   │   ├── send_email.py
│   │   └── send_voicemail.py
│   ├── crm/
│   │   ├── update_contact.py
│   │   ├── add_tag.py
│   │   ├── remove_tag.py
│   │   ├── add_note.py
│   │   └── move_pipeline.py
│   ├── internal/
│   │   ├── wait.py            # Wait X minutes/hours/days
│   │   ├── wait_until.py      # Wait until condition
│   │   ├── branch.py          # If/else branching
│   │   ├── go_to.py           # Jump to another step
│   │   └── end.py             # End workflow
│   ├── integrations/
│   │   ├── webhook.py         # Call external webhook
│   │   ├── slack.py           # Send Slack message
│   │   └── custom_code.py     # Run custom JS/Python
│   └── ai/
│       ├── classify_intent.py  # AI classify message
│       └── generate_response.py # AI generate reply
│
├── conditions/
│   ├── __init__.py
│   ├── base.py                # Base condition class
│   ├── contact_conditions.py  # Has tag, field equals, etc.
│   ├── message_conditions.py  # Contains keyword, sentiment
│   ├── time_conditions.py     # Day of week, business hours
│   └── custom_conditions.py   # Custom expression
│
├── executor/
│   ├── __init__.py
│   ├── engine.py              # Main execution engine
│   ├── context.py             # Execution context
│   ├── queue.py               # Execution queue
│   ├── state.py               # State management
│   └── error_handler.py       # Error handling/retry
│
├── storage/
│   ├── __init__.py
│   ├── workflow_store.py      # Save/load workflows
│   ├── execution_store.py     # Save execution state
│   └── history_store.py       # Audit trail
│
├── api/
│   ├── __init__.py
│   ├── routes.py              # REST endpoints
│   ├── serializers.py         # JSON serialization
│   └── validators.py          # Input validation
│
└── tests/
    ├── test_triggers.py
    ├── test_actions.py
    ├── test_conditions.py
    └── test_executor.py
```

---

## WORKFLOW JSON STRUCTURE

```json
{
  "id": "wf_abc123",
  "name": "New Lead Follow-up",
  "location_id": "loc_xyz",
  "is_active": true,
  "version": 3,
  
  "trigger": {
    "type": "contact_created",
    "config": {
      "tags_include": ["new_lead"],
      "tags_exclude": ["existing_customer"]
    }
  },
  
  "steps": [
    {
      "id": "step_1",
      "type": "action",
      "action": "send_sms",
      "config": {
        "template": "Hi {{contact.first_name}}, thanks for reaching out!",
        "from_number": "+18012129267"
      },
      "next": "step_2"
    },
    {
      "id": "step_2", 
      "type": "action",
      "action": "wait",
      "config": {
        "duration": 24,
        "unit": "hours"
      },
      "next": "step_3"
    },
    {
      "id": "step_3",
      "type": "condition",
      "condition": {
        "type": "contact_replied",
        "config": {
          "within_hours": 24
        }
      },
      "branches": {
        "true": "step_end_replied",
        "false": "step_4"
      }
    },
    {
      "id": "step_4",
      "type": "action",
      "action": "send_sms",
      "config": {
        "template": "Hi {{contact.first_name}}, just following up!"
      },
      "next": "step_5"
    },
    {
      "id": "step_5",
      "type": "action",
      "action": "add_tag",
      "config": {
        "tag": "needs_followup"
      },
      "next": null
    },
    {
      "id": "step_end_replied",
      "type": "action",
      "action": "add_tag",
      "config": {
        "tag": "engaged"
      },
      "next": null
    }
  ],
  
  "settings": {
    "allow_re_enrollment": false,
    "exit_on_reply": true,
    "respect_dnd": true,
    "respect_quiet_hours": true,
    "max_enrollments_per_day": 100
  }
}
```

---

## DATABASE SCHEMA

```sql
-- Workflow definitions
CREATE TABLE workflows (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    
    trigger_type VARCHAR(50) NOT NULL,
    trigger_config JSONB NOT NULL DEFAULT '{}',
    
    steps JSONB NOT NULL DEFAULT '[]',
    settings JSONB NOT NULL DEFAULT '{}',
    
    is_active BOOLEAN DEFAULT false,
    version INT DEFAULT 1,
    
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Workflow executions (contact enrolled in workflow)
CREATE TABLE workflow_executions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id UUID NOT NULL REFERENCES workflows(id),
    workflow_version INT NOT NULL,
    contact_id UUID NOT NULL REFERENCES contacts(id),
    
    status VARCHAR(20) NOT NULL DEFAULT 'running'
        CHECK (status IN ('running', 'paused', 'completed', 
                          'failed', 'cancelled', 'exited')),
    
    current_step_id VARCHAR(100),
    current_step_started_at TIMESTAMPTZ,
    
    -- For wait steps
    waiting_until TIMESTAMPTZ,
    waiting_for_condition JSONB,
    
    -- Completion tracking
    started_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    exit_reason TEXT,
    
    -- Error tracking
    error_count INT DEFAULT 0,
    last_error TEXT,
    last_error_at TIMESTAMPTZ,
    
    metadata JSONB DEFAULT '{}',
    
    UNIQUE(workflow_id, contact_id) -- Prevent duplicate enrollment
);

-- Step execution history
CREATE TABLE workflow_step_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id UUID NOT NULL REFERENCES workflow_executions(id),
    step_id VARCHAR(100) NOT NULL,
    step_type VARCHAR(50) NOT NULL,
    
    status VARCHAR(20) NOT NULL
        CHECK (status IN ('started', 'completed', 'failed', 'skipped')),
    
    input_data JSONB,
    output_data JSONB,
    error_message TEXT,
    
    started_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    duration_ms INT
);

-- Indexes
CREATE INDEX idx_wf_exec_status ON workflow_executions(status) 
    WHERE status IN ('running', 'paused');
CREATE INDEX idx_wf_exec_waiting ON workflow_executions(waiting_until) 
    WHERE waiting_until IS NOT NULL AND status = 'running';
CREATE INDEX idx_wf_exec_contact ON workflow_executions(contact_id);
```

---

## EXECUTOR ENGINE

```python
class WorkflowExecutor:
    """Main workflow execution engine"""
    
    async def enroll_contact(
        self, 
        workflow_id: str, 
        contact_id: str,
        trigger_data: dict = None
    ) -> WorkflowExecution:
        """Enroll a contact in a workflow"""
        workflow = await self.store.get_workflow(workflow_id)
        
        # Check if already enrolled
        existing = await self.store.get_execution(workflow_id, contact_id)
        if existing and not workflow.settings.allow_re_enrollment:
            raise AlreadyEnrolledException()
        
        # Create execution record
        execution = WorkflowExecution(
            workflow_id=workflow_id,
            workflow_version=workflow.version,
            contact_id=contact_id,
            status='running',
            current_step_id=workflow.steps[0].id,
            metadata={'trigger_data': trigger_data}
        )
        await self.store.save_execution(execution)
        
        # Start processing first step
        await self.process_step(execution)
        
        return execution
    
    async def process_step(self, execution: WorkflowExecution):
        """Process the current step"""
        workflow = await self.store.get_workflow(execution.workflow_id)
        step = workflow.get_step(execution.current_step_id)
        
        if not step:
            await self._complete_execution(execution)
            return
        
        # Build execution context
        context = await self._build_context(execution)
        
        try:
            if step.type == 'action':
                result = await self._execute_action(step, context)
                next_step_id = step.next
                
            elif step.type == 'condition':
                result = await self._evaluate_condition(step, context)
                next_step_id = step.branches['true' if result else 'false']
                
            elif step.type == 'wait':
                await self._schedule_wait(execution, step)
                return  # Don't advance yet
            
            # Record step completion
            await self._record_step(execution, step, 'completed', result)
            
            # Advance to next step
            if next_step_id:
                execution.current_step_id = next_step_id
                await self.store.save_execution(execution)
                await self.process_step(execution)
            else:
                await self._complete_execution(execution)
                
        except Exception as e:
            await self._handle_error(execution, step, e)
    
    async def _execute_action(self, step: WorkflowStep, context: ExecutionContext):
        """Execute an action step"""
        action_class = self.action_registry.get(step.action)
        action = action_class(step.config)
        return await action.execute(context)
    
    async def _evaluate_condition(self, step: WorkflowStep, context: ExecutionContext):
        """Evaluate a condition step"""
        condition_class = self.condition_registry.get(step.condition.type)
        condition = condition_class(step.condition.config)
        return await condition.evaluate(context)
    
    async def _schedule_wait(self, execution: WorkflowExecution, step: WorkflowStep):
        """Schedule a wait step"""
        if step.action == 'wait':
            duration = timedelta(**{step.config['unit']: step.config['duration']})
            execution.waiting_until = datetime.utcnow() + duration
        elif step.action == 'wait_until':
            execution.waiting_for_condition = step.config
        
        execution.current_step_started_at = datetime.utcnow()
        await self.store.save_execution(execution)
```

---

## TRIGGER EXAMPLES

```python
# Contact Created Trigger
class ContactCreatedTrigger(BaseTrigger):
    async def should_fire(self, event: Event) -> bool:
        if event.type != 'contact.created':
            return False
        
        contact = event.data
        
        # Check tag filters
        if self.config.get('tags_include'):
            if not any(t in contact.tags for t in self.config['tags_include']):
                return False
        
        if self.config.get('tags_exclude'):
            if any(t in contact.tags for t in self.config['tags_exclude']):
                return False
        
        return True

# Message Received Trigger
class MessageReceivedTrigger(BaseTrigger):
    async def should_fire(self, event: Event) -> bool:
        if event.type != 'message.received':
            return False
        
        message = event.data
        
        # Check keyword filters
        if self.config.get('contains_keywords'):
            keywords = self.config['contains_keywords']
            if not any(kw.lower() in message.body.lower() for kw in keywords):
                return False
        
        return True
```

---

## API ENDPOINTS

```
# Workflows
GET    /api/workflows                     - List workflows
POST   /api/workflows                     - Create workflow
GET    /api/workflows/:id                 - Get workflow
PUT    /api/workflows/:id                 - Update workflow
DELETE /api/workflows/:id                 - Delete workflow
POST   /api/workflows/:id/activate        - Activate workflow
POST   /api/workflows/:id/deactivate      - Deactivate workflow
POST   /api/workflows/:id/duplicate       - Duplicate workflow

# Executions
GET    /api/workflows/:id/executions      - List executions
POST   /api/workflows/:id/enroll          - Manually enroll contact
DELETE /api/executions/:id                - Cancel execution

# Stats
GET    /api/workflows/:id/stats           - Workflow statistics
GET    /api/workflows/:id/funnel          - Step-by-step conversion
```

---

## LINE ESTIMATES

| Directory | Files | Lines |
|-----------|-------|-------|
| models/ | 5 | 800 |
| triggers/ | 8 | 1,200 |
| actions/ | 15 | 2,500 |
| conditions/ | 5 | 600 |
| executor/ | 5 | 1,500 |
| storage/ | 3 | 600 |
| api/ | 3 | 500 |
| **TOTAL** | **44** | **7,700** |
