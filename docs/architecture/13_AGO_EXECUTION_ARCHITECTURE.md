# AGO Execution Architecture

**Source:** ChatGPT Deep Research (January 2026)
**Priority:** CRITICAL - The runtime that makes AGO work

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    AUTONOMOUS GROWTH OPERATOR                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │    Events    │───▶│    State     │───▶│    Agent     │       │
│  │    Ledger    │    │    Engine    │    │    Brain     │       │
│  └──────────────┘    └──────────────┘    └──────┬───────┘       │
│         ▲                                       │               │
│         │            ┌──────────────┐           │               │
│         │            │   Verifier   │◀──────────┤               │
│         │            └──────┬───────┘           │               │
│         │                   │                   ▼               │
│         │            ┌──────────────┐    ┌──────────────┐       │
│         └────────────│   Learning   │◀───│  Action Bus  │       │
│                      └──────────────┘    └──────────────┘       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. Deterministic State Engine (Core)

**Rule:** LLMs propose, they do not execute. State transitions are deterministic.

### State Machine Definition

```yaml
# Lead Lifecycle State Machine
name: lead_lifecycle
initial_state: new

states:
  new:
    on_enter:
      - emit_event: lead.state_changed
      - schedule_action: enrich_lead
    transitions:
      - event: outreach.started
        to: contacting
      - event: consent.revoked
        to: dnc
        
  contacting:
    on_enter:
      - emit_event: lead.state_changed
      - schedule_action: plan_outreach
    transitions:
      - event: reply.received
        to: engaged
        condition: "event.direction == 'inbound'"
      - event: contact_attempts.exhausted
        to: nurture
        condition: "state.attempts >= 5"
      - event: consent.revoked
        to: dnc
      - event: timeout
        to: contacting  # Re-evaluate, stay in state
        condition: "elapsed > 5m"
        
  engaged:
    on_enter:
      - emit_event: lead.state_changed
      - schedule_action: analyze_intent
    transitions:
      - event: qualification.passed
        to: qualified
        condition: "memory.intent_score > 0.7"
      - event: qualification.failed
        to: nurture
      - event: consent.revoked
        to: dnc
        
  qualified:
    on_enter:
      - emit_event: lead.state_changed
      - schedule_action: offer_appointment
    transitions:
      - event: appointment.booked
        to: booked
      - event: declined
        to: lost
      - event: not_ready
        to: nurture
      - event: consent.revoked
        to: dnc
        
  booked:
    on_enter:
      - emit_event: lead.state_changed
      - schedule_action: send_confirmation
      - schedule_action: schedule_reminders
    transitions:
      - event: appointment.completed
        to: won
        condition: "event.outcome == 'converted'"
      - event: appointment.completed
        to: lost
        condition: "event.outcome == 'declined'"
      - event: appointment.no_show
        to: no_show
      - event: appointment.cancelled
        to: qualified  # Back to scheduling
      - event: consent.revoked
        to: dnc
        
  no_show:
    on_enter:
      - emit_event: lead.state_changed
      - schedule_action: no_show_followup
    transitions:
      - event: appointment.rescheduled
        to: booked
      - event: max_no_shows.reached
        to: nurture
      - event: consent.revoked
        to: dnc
        
  won:
    final: true
    on_enter:
      - emit_event: lead.won
      - schedule_action: handoff_to_fulfillment
      
  lost:
    final: true
    on_enter:
      - emit_event: lead.lost
      - schedule_action: loss_analysis
      
  nurture:
    on_enter:
      - emit_event: lead.state_changed
      - schedule_action: add_to_nurture_campaign
    transitions:
      - event: reengagement.detected
        to: engaged
      - event: consent.revoked
        to: dnc
        
  dnc:
    final: true
    on_enter:
      - emit_event: lead.dnc
      - cancel_all_actions: true
```

### State Engine Implementation

```python
class StateEngine:
    """
    Deterministic state machine executor.
    LLMs never touch this directly.
    """
    
    def __init__(self, state_machine: StateMachineDefinition):
        self.sm = state_machine
        
    async def process_event(self, event: Event) -> StateTransitionResult:
        """
        Process an event and determine state transition.
        Returns the new state and any actions to schedule.
        """
        lead_state = await self.get_lead_state(event.lead_id)
        current_state = self.sm.states[lead_state.lifecycle]
        
        # Find matching transition
        for transition in current_state.transitions:
            if transition.event != event.event_type:
                continue
                
            # Evaluate condition if present
            if transition.condition:
                context = {
                    'event': event,
                    'state': lead_state,
                    'memory': await self.get_memory(event.lead_id)
                }
                if not self.evaluate_condition(transition.condition, context):
                    continue
            
            # Execute transition
            return await self.execute_transition(
                lead_id=event.lead_id,
                from_state=lead_state.lifecycle,
                to_state=transition.to,
                trigger_event=event
            )
        
        # No transition matched - stay in current state
        return StateTransitionResult(
            transitioned=False,
            current_state=lead_state.lifecycle
        )
    
    async def execute_transition(
        self,
        lead_id: str,
        from_state: str,
        to_state: str,
        trigger_event: Event
    ) -> StateTransitionResult:
        """Execute a validated state transition."""
        
        # 1. Update state
        await self.update_lead_state(lead_id, to_state)
        
        # 2. Emit transition event
        await self.emit_event(Event(
            lead_id=lead_id,
            event_type='lead.state_changed',
            payload={
                'from': from_state,
                'to': to_state,
                'trigger': trigger_event.event_id
            }
        ))
        
        # 3. Execute on_enter actions
        new_state = self.sm.states[to_state]
        scheduled_actions = []
        
        for action in new_state.on_enter:
            if action.type == 'emit_event':
                await self.emit_event(Event(
                    lead_id=lead_id,
                    event_type=action.event_type
                ))
            elif action.type == 'schedule_action':
                scheduled_actions.append(action.action_name)
            elif action.type == 'cancel_all_actions':
                await self.cancel_pending_actions(lead_id)
        
        return StateTransitionResult(
            transitioned=True,
            from_state=from_state,
            to_state=to_state,
            scheduled_actions=scheduled_actions
        )
```

---

## 2. Agent Brain (Planning, Not Execution)

**Rule:** The Agent Brain proposes actions. It never executes directly.

### Inputs to Agent Brain

```python
@dataclass
class AgentContext:
    """Everything the agent knows when making a decision."""
    
    # Current state
    lead: Lead
    lead_state: LeadState
    
    # History
    recent_events: list[Event]  # Last N events
    conversation_history: list[Message]  # Last N messages
    
    # Memory
    agent_memory: dict[str, Any]  # Structured memory
    
    # Constraints
    goal: Goal  # What we're trying to achieve
    consent: dict[str, Consent]  # Channel permissions
    quiet_hours: bool
    
    # Capacity
    rep_availability: list[Rep]
    rep_workload: dict[str, int]
    
    # Health
    deliverability: DeliverabilityHealth  # Channel health scores
    
    # Economics
    current_cpa: float
    target_cpa: float
    budget_remaining: float
```

### Agent Brain Output

```python
@dataclass
class AgentProposal:
    """What the agent proposes to do."""
    
    proposed_actions: list[ProposedAction]
    reasoning: str
    confidence: float  # 0-1
    risk_score: float  # 0-1
    
    # For human review
    requires_approval: bool
    approval_reason: str | None

@dataclass
class ProposedAction:
    action_type: str  # 'send_sms', 'send_email', 'initiate_call', 'wait'
    payload: dict
    reason: str
    expected_outcome: str
    risk: float
```

### Agent Brain Implementation

```python
class AgentBrain:
    """
    LLM-powered planning engine.
    Proposes actions, never executes.
    """
    
    def __init__(self, llm_client: Claude):
        self.llm = llm_client
        
    async def plan(self, context: AgentContext) -> AgentProposal:
        """
        Given context, propose the best next action(s).
        """
        
        # Build prompt
        prompt = self.build_planning_prompt(context)
        
        # Call LLM
        response = await self.llm.complete(
            model="claude-3-haiku",  # Fast for planning
            messages=[{"role": "user", "content": prompt}],
            system=AGENT_SYSTEM_PROMPT
        )
        
        # Parse structured output
        proposal = self.parse_proposal(response)
        
        # Apply guardrails
        proposal = await self.apply_guardrails(proposal, context)
        
        return proposal
    
    async def apply_guardrails(
        self,
        proposal: AgentProposal,
        context: AgentContext
    ) -> AgentProposal:
        """
        Enforce hard constraints on agent proposals.
        """
        
        filtered_actions = []
        
        for action in proposal.proposed_actions:
            # HARD GATE: Check consent
            if action.action_type in ['send_sms', 'send_email', 'initiate_call']:
                channel = self.action_to_channel(action.action_type)
                if not await check_consent(context.lead.id, channel):
                    continue  # Skip this action
            
            # HARD GATE: Check quiet hours
            if context.quiet_hours and action.action_type != 'wait':
                action = ProposedAction(
                    action_type='wait',
                    payload={'until': 'quiet_hours_end'},
                    reason='Quiet hours active',
                    expected_outcome='Resume outreach',
                    risk=0.0
                )
            
            # HARD GATE: Check deliverability
            if action.action_type == 'send_sms':
                if context.deliverability.sms_health < 0.5:
                    action.risk += 0.3
                    proposal.requires_approval = True
                    proposal.approval_reason = 'Low SMS deliverability'
            
            # HARD GATE: Check CPA
            action_cost = self.estimate_action_cost(action)
            if context.current_cpa + action_cost > context.target_cpa * 1.2:
                proposal.requires_approval = True
                proposal.approval_reason = f'Action exceeds CPA target'
            
            filtered_actions.append(action)
        
        proposal.proposed_actions = filtered_actions
        return proposal
```

### Agent System Prompt

```python
AGENT_SYSTEM_PROMPT = """
You are an AI sales agent optimizing for booked appointments.

RULES:
1. You PROPOSE actions, you do not execute them.
2. Respect all compliance constraints - they are hard gates.
3. Optimize for the goal, not just activity.
4. When uncertain, choose lower-risk actions.
5. Always explain your reasoning.

OUTPUT FORMAT:
{
  "proposed_actions": [
    {
      "action_type": "send_sms",
      "payload": {"template_id": "...", "personalization": {...}},
      "reason": "High intent detected, SMS is preferred channel",
      "expected_outcome": "Reply within 2 hours",
      "risk": 0.1
    }
  ],
  "reasoning": "Lead showed interest in pricing. Best to follow up quickly via preferred channel.",
  "confidence": 0.85
}
"""
```

---

## 3. Action Bus (Outbox Pattern)

```python
class ActionBus:
    """
    Reliable action execution via outbox pattern.
    """
    
    async def enqueue_action(
        self,
        action: ProposedAction,
        lead_id: str,
        correlation_id: str,
        execute_after: datetime = None
    ) -> str:
        """Add action to outbox for processing."""
        
        action_id = str(uuid4())
        
        await self.db.execute("""
            INSERT INTO action_outbox 
            (action_id, lead_id, action_type, payload, execute_after, 
             correlation_id, status)
            VALUES ($1, $2, $3, $4, $5, $6, 'pending')
        """, action_id, lead_id, action.action_type, 
           json.dumps(action.payload), execute_after or datetime.utcnow(),
           correlation_id)
        
        return action_id
    
    async def process_pending(self):
        """Worker loop to process pending actions."""
        
        while True:
            # Get next batch (with row locking)
            actions = await self.db.fetch("""
                SELECT * FROM action_outbox
                WHERE status = 'pending'
                  AND execute_after <= NOW()
                ORDER BY execute_after
                LIMIT 10
                FOR UPDATE SKIP LOCKED
            """)
            
            for action in actions:
                try:
                    await self.execute_action(action)
                    await self.mark_sent(action['action_id'])
                except Exception as e:
                    await self.handle_failure(action, e)
            
            await asyncio.sleep(1)
    
    async def execute_action(self, action: dict):
        """Execute a single action."""
        
        if action['action_type'] == 'send_sms':
            await self.sms_worker.send(action['payload'])
        elif action['action_type'] == 'send_email':
            await self.email_worker.send(action['payload'])
        elif action['action_type'] == 'initiate_call':
            await self.voice_worker.initiate(action['payload'])
        # ... etc
        
        # Emit event
        await self.emit_event(Event(
            lead_id=action['lead_id'],
            event_type=f"{action['action_type']}.executed",
            payload=action['payload'],
            correlation_id=action['correlation_id']
        ))
```

---

## 4. Verifier (Silent Killer Feature)

```python
class Verifier:
    """
    Verifies outcomes and detects anomalies.
    Writes events, not flags.
    """
    
    async def verify_delivery(self, event: Event):
        """Verify message was actually delivered."""
        
        if event.event_type == 'sms.sent':
            # Check Twilio status
            status = await self.twilio.get_status(event.payload['sid'])
            
            if status == 'delivered':
                await self.emit_event(Event(
                    lead_id=event.lead_id,
                    event_type='sms.delivered',
                    correlation_id=event.correlation_id
                ))
            elif status == 'failed':
                await self.emit_event(Event(
                    lead_id=event.lead_id,
                    event_type='sms.failed',
                    payload={'reason': status.error_code},
                    correlation_id=event.correlation_id
                ))
                
                # Trigger self-healing
                await self.trigger_repair('delivery_failure', event)
    
    async def verify_reply_authenticity(self, event: Event):
        """Detect spam/bot replies."""
        
        if event.event_type == 'sms.received':
            # Check for spam patterns
            is_spam = await self.spam_detector.check(event.payload['body'])
            
            if is_spam:
                await self.emit_event(Event(
                    lead_id=event.lead_id,
                    event_type='reply.spam_detected',
                    payload={'confidence': is_spam.confidence}
                ))
    
    async def verify_booking_validity(self, event: Event):
        """Ensure appointment is actually valid."""
        
        if event.event_type == 'appointment.booked':
            # Check for double-booking
            conflicts = await self.check_conflicts(event.payload)
            
            if conflicts:
                await self.emit_event(Event(
                    lead_id=event.lead_id,
                    event_type='appointment.conflict_detected',
                    payload={'conflicts': conflicts}
                ))
                
                # Trigger self-healing
                await self.trigger_repair('booking_conflict', event)
```

---

## 5. Learning Loop

```python
class LearningLoop:
    """
    Measure what works, update strategies.
    Start simple: variants + holdouts + uplift.
    """
    
    async def assign_experiment(
        self,
        lead_id: str,
        experiment_name: str,
        variants: list[str]
    ) -> str:
        """Randomly assign lead to experiment variant."""
        
        # Check if already assigned
        existing = await self.get_assignment(lead_id, experiment_name)
        if existing:
            return existing.variant
        
        # Random assignment
        variant = random.choice(variants)
        
        await self.db.execute("""
            INSERT INTO experiment_assignments
            (lead_id, experiment_name, variant)
            VALUES ($1, $2, $3)
        """, lead_id, experiment_name, variant)
        
        return variant
    
    async def record_outcome(
        self,
        lead_id: str,
        experiment_name: str,
        outcome: dict
    ):
        """Record experiment outcome."""
        
        await self.db.execute("""
            UPDATE experiment_assignments
            SET outcome_measured = true,
                outcome_value = $3,
                outcome_measured_at = NOW()
            WHERE lead_id = $1 AND experiment_name = $2
        """, lead_id, experiment_name, json.dumps(outcome))
    
    async def calculate_uplift(self, experiment_name: str) -> UpliftResult:
        """Calculate incremental lift from experiment."""
        
        results = await self.db.fetch("""
            SELECT variant, 
                   COUNT(*) as n,
                   AVG((outcome_value->>'booked')::boolean::int) as book_rate,
                   AVG((outcome_value->>'showed')::boolean::int) as show_rate
            FROM experiment_assignments
            WHERE experiment_name = $1
              AND outcome_measured = true
            GROUP BY variant
        """, experiment_name)
        
        control = next(r for r in results if r['variant'] == 'control')
        treatment = next(r for r in results if r['variant'] != 'control')
        
        return UpliftResult(
            experiment=experiment_name,
            control_rate=control['book_rate'],
            treatment_rate=treatment['book_rate'],
            lift=treatment['book_rate'] - control['book_rate'],
            is_significant=self.calculate_significance(control, treatment)
        )
```

---

## Complete Flow Example

```
1. Lead submits form
   └── Event: lead.created

2. State Engine processes event
   └── Transition: new → contacting
   └── Schedule: plan_outreach

3. Agent Brain plans
   └── Input: lead context, goal, constraints
   └── Output: [send_sms(template_1), wait(2h), send_sms(template_2)]

4. Action Bus executes
   └── SMS sent via Twilio
   └── Event: sms.sent

5. Verifier confirms
   └── Twilio webhook: delivered
   └── Event: sms.delivered

6. Lead replies
   └── Event: sms.received

7. State Engine processes
   └── Transition: contacting → engaged

8. Agent Brain re-plans
   └── High intent detected
   └── Output: [offer_appointment]

9. Appointment booked
   └── Event: appointment.booked
   └── Transition: engaged → booked

10. Learning Loop records
    └── Experiment outcome: booked=true

11. Repeat until: won | lost | dnc | nurture
```
