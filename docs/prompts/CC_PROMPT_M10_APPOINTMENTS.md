# Claude Code Prompt: M10 Appointments System

## Context

You are building the Appointments module for Ziloss CRM, a GoHighLevel competitor. This module handles all appointment booking, scheduling, reminders, and calendar integration.

**Project Location:** `/Users/mac/Desktop/agent-orchestrator`
**Module Location:** `/Users/mac/Desktop/agent-orchestrator/src/modules/appointments/`
**Estimated Lines:** 3,000
**Time Estimate:** 2 days

## Tech Stack

- **Language:** Python 3.11+
- **Framework:** FastAPI
- **Database:** PostgreSQL (async via asyncpg)
- **Validation:** Pydantic v2
- **Background Jobs:** Redis queue (for reminders)
- **Calendar Sync:** Google Calendar API
- **Testing:** pytest + pytest-asyncio

## Module Purpose

The Appointments module provides:
1. **Appointment Types** - Define service types with duration/buffer
2. **Availability Rules** - Set weekly availability schedules
3. **Booking Engine** - Handle appointment creation with conflict detection
4. **Reminders** - Automated SMS/email reminders (24h, 1h before)
5. **Public Booking Pages** - Embeddable booking widget
6. **Calendar Sync** - Google Calendar integration

## File Structure to Create

```
src/modules/appointments/
├── __init__.py
├── router.py                  # FastAPI routes
├── service.py                 # Business logic
├── models.py                  # Pydantic models
├── schemas.py                 # Database schemas
├── availability/
│   ├── __init__.py
│   ├── rules.py               # Availability rule logic
│   ├── slots.py               # Available slot calculation
│   └── overrides.py           # Date-specific overrides
├── booking/
│   ├── __init__.py
│   ├── engine.py              # Booking logic
│   ├── conflicts.py           # Conflict detection
│   └── validator.py           # Booking validation
├── reminders/
│   ├── __init__.py
│   ├── scheduler.py           # Schedule reminders
│   └── sender.py              # Send reminders
├── calendar_sync/
│   ├── __init__.py
│   ├── google.py              # Google Calendar integration
│   └── ical.py                # iCal export
├── booking_pages/
│   ├── __init__.py
│   ├── generator.py           # Generate booking page URLs
│   └── embed.py               # Embeddable widget
├── exceptions.py
└── tests/
    ├── __init__.py
    ├── test_availability.py
    ├── test_booking.py
    ├── test_reminders.py
    └── conftest.py
```

## Database Schema

```sql
-- Appointment Types (services offered)
CREATE TABLE appointment_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    
    name VARCHAR(100) NOT NULL,
    description TEXT,
    duration_minutes INT NOT NULL DEFAULT 30,
    buffer_before_minutes INT DEFAULT 0,
    buffer_after_minutes INT DEFAULT 15,
    
    -- Pricing (optional)
    price_cents INT,
    currency VARCHAR(3) DEFAULT 'USD',
    
    -- Settings
    color VARCHAR(7) DEFAULT '#3B82F6',  -- Hex color for calendar
    is_active BOOLEAN DEFAULT TRUE,
    requires_confirmation BOOLEAN DEFAULT FALSE,
    max_per_day INT,  -- NULL = unlimited
    
    -- Booking page settings
    show_on_booking_page BOOLEAN DEFAULT TRUE,
    booking_page_order INT DEFAULT 0,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_appt_types_location ON appointment_types(location_id);

-- Availability Rules (weekly schedule)
CREATE TABLE availability_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,  -- NULL = location-wide
    
    day_of_week INT NOT NULL CHECK (day_of_week >= 0 AND day_of_week <= 6),  -- 0=Sunday
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    
    timezone VARCHAR(50) NOT NULL DEFAULT 'America/Chicago',
    is_active BOOLEAN DEFAULT TRUE,
    
    -- Optional: specific appointment types only
    appointment_type_ids JSONB,  -- NULL = all types
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT valid_time_range CHECK (end_time > start_time)
);

CREATE INDEX idx_availability_location ON availability_rules(location_id);
CREATE INDEX idx_availability_user ON availability_rules(user_id);

-- Availability Overrides (specific dates)
CREATE TABLE availability_overrides (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    
    date DATE NOT NULL,
    
    -- NULL times = entire day blocked
    start_time TIME,
    end_time TIME,
    
    is_available BOOLEAN NOT NULL,  -- TRUE = available, FALSE = blocked
    reason VARCHAR(200),
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    UNIQUE(location_id, user_id, date, start_time)
);

CREATE INDEX idx_overrides_date ON availability_overrides(date);

-- Appointments
CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    contact_id UUID NOT NULL REFERENCES contacts(id),
    appointment_type_id UUID NOT NULL REFERENCES appointment_types(id),
    assigned_to UUID REFERENCES users(id),  -- Staff member
    
    -- Time
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    timezone VARCHAR(50) NOT NULL,
    
    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'scheduled'
        CHECK (status IN ('scheduled', 'confirmed', 'completed', 'cancelled', 'no_show', 'rescheduled')),
    
    -- Details
    title VARCHAR(200),
    notes TEXT,
    internal_notes TEXT,  -- Staff-only notes
    
    -- Cancellation
    cancelled_at TIMESTAMPTZ,
    cancelled_by UUID REFERENCES users(id),
    cancellation_reason TEXT,
    
    -- Reminders
    reminder_24h_sent_at TIMESTAMPTZ,
    reminder_1h_sent_at TIMESTAMPTZ,
    confirmation_sent_at TIMESTAMPTZ,
    
    -- External sync
    google_event_id VARCHAR(200),
    
    -- Booking source
    booked_via VARCHAR(50) DEFAULT 'manual',  -- manual, booking_page, api, ai
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- CRITICAL: Prevent double-booking with database constraint
    EXCLUDE USING gist (
        location_id WITH =,
        assigned_to WITH =,
        tstzrange(start_time, end_time) WITH &&
    ) WHERE (status NOT IN ('cancelled', 'rescheduled'))
);

CREATE INDEX idx_appointments_location ON appointments(location_id);
CREATE INDEX idx_appointments_contact ON appointments(contact_id);
CREATE INDEX idx_appointments_assigned ON appointments(assigned_to);
CREATE INDEX idx_appointments_start ON appointments(start_time);
CREATE INDEX idx_appointments_status ON appointments(status);
CREATE INDEX idx_appointments_reminders ON appointments(start_time) 
    WHERE reminder_24h_sent_at IS NULL OR reminder_1h_sent_at IS NULL;

-- Booking Pages
CREATE TABLE booking_pages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    
    -- URL
    slug VARCHAR(100) NOT NULL,
    custom_domain VARCHAR(200),
    
    -- Content
    title VARCHAR(200) NOT NULL,
    description TEXT,
    logo_url TEXT,
    
    -- Appearance
    primary_color VARCHAR(7) DEFAULT '#3B82F6',
    background_color VARCHAR(7) DEFAULT '#FFFFFF',
    
    -- Settings
    show_timezone_selector BOOLEAN DEFAULT TRUE,
    require_phone BOOLEAN DEFAULT TRUE,
    require_email BOOLEAN DEFAULT TRUE,
    custom_fields JSONB DEFAULT '[]',
    
    -- Which appointment types to show
    appointment_type_ids JSONB,  -- NULL = all active types
    
    -- Which staff to show
    user_ids JSONB,  -- NULL = all staff, or round-robin
    assignment_mode VARCHAR(20) DEFAULT 'round_robin'
        CHECK (assignment_mode IN ('round_robin', 'specific', 'any_available')),
    
    is_active BOOLEAN DEFAULT TRUE,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    UNIQUE(location_id, slug)
);
```

## Pydantic Models

```python
from enum import Enum
from pydantic import BaseModel, Field
from datetime import datetime, date, time
from uuid import UUID
from typing import Optional

class AppointmentStatus(str, Enum):
    SCHEDULED = "scheduled"
    CONFIRMED = "confirmed"
    COMPLETED = "completed"
    CANCELLED = "cancelled"
    NO_SHOW = "no_show"
    RESCHEDULED = "rescheduled"

class AssignmentMode(str, Enum):
    ROUND_ROBIN = "round_robin"
    SPECIFIC = "specific"
    ANY_AVAILABLE = "any_available"

# Request models
class CreateAppointmentTypeRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    description: Optional[str] = None
    duration_minutes: int = Field(default=30, ge=5, le=480)
    buffer_before_minutes: int = Field(default=0, ge=0, le=60)
    buffer_after_minutes: int = Field(default=15, ge=0, le=60)
    price_cents: Optional[int] = Field(None, ge=0)
    color: str = Field(default="#3B82F6", pattern="^#[0-9A-Fa-f]{6}$")
    requires_confirmation: bool = False
    max_per_day: Optional[int] = Field(None, ge=1)

class CreateAvailabilityRuleRequest(BaseModel):
    user_id: Optional[UUID] = None
    day_of_week: int = Field(..., ge=0, le=6)
    start_time: time
    end_time: time
    timezone: str = "America/Chicago"
    appointment_type_ids: Optional[list[UUID]] = None

class CreateAvailabilityOverrideRequest(BaseModel):
    user_id: Optional[UUID] = None
    date: date
    start_time: Optional[time] = None
    end_time: Optional[time] = None
    is_available: bool
    reason: Optional[str] = Field(None, max_length=200)

class BookAppointmentRequest(BaseModel):
    appointment_type_id: UUID
    contact_id: UUID
    start_time: datetime
    assigned_to: Optional[UUID] = None
    title: Optional[str] = Field(None, max_length=200)
    notes: Optional[str] = None
    timezone: str = "America/Chicago"
    send_confirmation: bool = True

class RescheduleAppointmentRequest(BaseModel):
    new_start_time: datetime
    reason: Optional[str] = None
    notify_contact: bool = True

class CancelAppointmentRequest(BaseModel):
    reason: Optional[str] = None
    notify_contact: bool = True

class GetAvailableSlotsRequest(BaseModel):
    appointment_type_id: UUID
    start_date: date
    end_date: date
    user_id: Optional[UUID] = None  # Specific staff member
    timezone: str = "America/Chicago"

# Response models
class AvailableSlot(BaseModel):
    start_time: datetime
    end_time: datetime
    user_id: Optional[UUID] = None
    user_name: Optional[str] = None

class AppointmentResponse(BaseModel):
    id: UUID
    location_id: UUID
    contact_id: UUID
    appointment_type_id: UUID
    assigned_to: Optional[UUID]
    start_time: datetime
    end_time: datetime
    timezone: str
    status: AppointmentStatus
    title: Optional[str]
    notes: Optional[str]
    google_event_id: Optional[str]
    booked_via: str
    created_at: datetime
    
    # Nested
    contact_name: Optional[str] = None
    appointment_type_name: Optional[str] = None
    assigned_to_name: Optional[str] = None
```

## API Endpoints

```python
router = APIRouter(prefix="/api/appointments", tags=["Appointments"])

# Appointment Types
@router.get("/types", response_model=list[AppointmentTypeResponse])
@router.post("/types", response_model=AppointmentTypeResponse)
@router.get("/types/{type_id}", response_model=AppointmentTypeResponse)
@router.put("/types/{type_id}", response_model=AppointmentTypeResponse)
@router.delete("/types/{type_id}")

# Availability Rules
@router.get("/availability/rules", response_model=list[AvailabilityRuleResponse])
@router.post("/availability/rules", response_model=AvailabilityRuleResponse)
@router.delete("/availability/rules/{rule_id}")

# Availability Overrides
@router.get("/availability/overrides", response_model=list[AvailabilityOverrideResponse])
@router.post("/availability/overrides", response_model=AvailabilityOverrideResponse)
@router.delete("/availability/overrides/{override_id}")

# Available Slots (for booking)
@router.post("/availability/slots", response_model=list[AvailableSlot])
async def get_available_slots(request: GetAvailableSlotsRequest):
    """Get available booking slots for a date range."""
    pass

# Appointments
@router.get("/", response_model=list[AppointmentResponse])
async def list_appointments(
    start_date: Optional[date] = None,
    end_date: Optional[date] = None,
    status: Optional[AppointmentStatus] = None,
    contact_id: Optional[UUID] = None,
    assigned_to: Optional[UUID] = None
):
    """List appointments with filters."""
    pass

@router.post("/", response_model=AppointmentResponse)
async def book_appointment(request: BookAppointmentRequest):
    """Book a new appointment."""
    pass

@router.get("/{appointment_id}", response_model=AppointmentResponse)
@router.put("/{appointment_id}/reschedule", response_model=AppointmentResponse)
@router.put("/{appointment_id}/cancel", response_model=AppointmentResponse)
@router.put("/{appointment_id}/confirm", response_model=AppointmentResponse)
@router.put("/{appointment_id}/complete", response_model=AppointmentResponse)
@router.put("/{appointment_id}/no-show", response_model=AppointmentResponse)

# Booking Pages
@router.get("/booking-pages", response_model=list[BookingPageResponse])
@router.post("/booking-pages", response_model=BookingPageResponse)
@router.get("/booking-pages/{page_id}", response_model=BookingPageResponse)
@router.put("/booking-pages/{page_id}", response_model=BookingPageResponse)

# Public Booking Endpoint (no auth required)
@router.get("/book/{slug}", response_model=PublicBookingPageResponse)
@router.post("/book/{slug}", response_model=AppointmentResponse)
```

## Key Implementation Details

### Availability Slot Calculation

```python
async def get_available_slots(
    location_id: UUID,
    appointment_type_id: UUID,
    start_date: date,
    end_date: date,
    user_id: Optional[UUID] = None,
    timezone: str = "America/Chicago"
) -> list[AvailableSlot]:
    """
    Calculate available slots by:
    1. Get appointment type (duration, buffers)
    2. Get availability rules for date range
    3. Apply overrides
    4. Remove existing appointments
    5. Return available slots
    """
    
    # 1. Get appointment type
    appt_type = await get_appointment_type(appointment_type_id)
    slot_duration = appt_type.duration_minutes + appt_type.buffer_before_minutes + appt_type.buffer_after_minutes
    
    # 2. Generate all possible slots from rules
    all_slots = []
    current_date = start_date
    while current_date <= end_date:
        day_of_week = current_date.weekday()  # Adjust for Sunday=0
        
        # Get rules for this day
        rules = await get_availability_rules(location_id, user_id, day_of_week)
        
        for rule in rules:
            # Generate slots within this rule's time range
            current_time = rule.start_time
            while current_time + timedelta(minutes=slot_duration) <= rule.end_time:
                all_slots.append(AvailableSlot(
                    start_time=combine_date_time(current_date, current_time, rule.timezone),
                    end_time=combine_date_time(current_date, current_time + timedelta(minutes=appt_type.duration_minutes), rule.timezone),
                    user_id=rule.user_id
                ))
                current_time += timedelta(minutes=slot_duration)
        
        current_date += timedelta(days=1)
    
    # 3. Apply overrides (remove blocked, add available)
    overrides = await get_overrides(location_id, user_id, start_date, end_date)
    all_slots = apply_overrides(all_slots, overrides)
    
    # 4. Remove slots that conflict with existing appointments
    existing = await get_existing_appointments(location_id, user_id, start_date, end_date)
    available_slots = remove_conflicts(all_slots, existing)
    
    # 5. Remove past slots
    now = datetime.now(tz=ZoneInfo(timezone))
    available_slots = [s for s in available_slots if s.start_time > now]
    
    return available_slots
```

### Double-Booking Prevention

The database uses a PostgreSQL EXCLUDE constraint to prevent double-booking at the database level:

```sql
EXCLUDE USING gist (
    location_id WITH =,
    assigned_to WITH =,
    tstzrange(start_time, end_time) WITH &&
) WHERE (status NOT IN ('cancelled', 'rescheduled'))
```

This requires the `btree_gist` extension. Also implement application-level checking:

```python
async def check_conflicts(
    location_id: UUID,
    assigned_to: UUID,
    start_time: datetime,
    end_time: datetime,
    exclude_appointment_id: Optional[UUID] = None
) -> list[Appointment]:
    """Check for conflicting appointments."""
    
    query = """
        SELECT * FROM appointments
        WHERE location_id = $1
          AND assigned_to = $2
          AND status NOT IN ('cancelled', 'rescheduled')
          AND tstzrange(start_time, end_time) && tstzrange($3, $4)
    """
    params = [location_id, assigned_to, start_time, end_time]
    
    if exclude_appointment_id:
        query += " AND id != $5"
        params.append(exclude_appointment_id)
    
    return await db.fetch_all(query, *params)
```

### Reminder Scheduling

```python
async def schedule_reminders(appointment: Appointment):
    """Schedule reminder jobs for an appointment."""
    
    # 24-hour reminder
    reminder_24h_time = appointment.start_time - timedelta(hours=24)
    if reminder_24h_time > datetime.now(tz=UTC):
        await enqueue_job(
            "send_appointment_reminder",
            appointment_id=appointment.id,
            reminder_type="24h",
            execute_at=reminder_24h_time
        )
    
    # 1-hour reminder
    reminder_1h_time = appointment.start_time - timedelta(hours=1)
    if reminder_1h_time > datetime.now(tz=UTC):
        await enqueue_job(
            "send_appointment_reminder",
            appointment_id=appointment.id,
            reminder_type="1h",
            execute_at=reminder_1h_time
        )
```

### Round-Robin Assignment

```python
async def assign_round_robin(
    location_id: UUID,
    user_ids: list[UUID],
    start_time: datetime
) -> UUID:
    """Assign appointment to next available staff member."""
    
    # Get appointment counts for today
    today_start = start_time.replace(hour=0, minute=0, second=0)
    today_end = today_start + timedelta(days=1)
    
    counts = await db.fetch_all("""
        SELECT assigned_to, COUNT(*) as count
        FROM appointments
        WHERE location_id = $1
          AND assigned_to = ANY($2)
          AND start_time >= $3 AND start_time < $4
          AND status NOT IN ('cancelled', 'rescheduled')
        GROUP BY assigned_to
    """, location_id, user_ids, today_start, today_end)
    
    count_map = {row['assigned_to']: row['count'] for row in counts}
    
    # Find user with fewest appointments
    min_count = float('inf')
    selected_user = user_ids[0]
    
    for user_id in user_ids:
        count = count_map.get(user_id, 0)
        if count < min_count:
            min_count = count
            selected_user = user_id
    
    return selected_user
```

## Events to Emit

Integrate with the Event Ledger (AGO architecture):

```python
# Emit these events for the state machine
EVENT_TYPES = [
    "appointment.created",
    "appointment.confirmed", 
    "appointment.rescheduled",
    "appointment.cancelled",
    "appointment.completed",
    "appointment.no_show",
    "appointment.reminder_sent",
]
```

## Testing Requirements

1. **test_availability.py**
   - Test slot generation from rules
   - Test override application
   - Test conflict removal
   - Test timezone handling

2. **test_booking.py**
   - Test successful booking
   - Test double-booking prevention
   - Test round-robin assignment
   - Test reschedule flow
   - Test cancellation flow

3. **test_reminders.py**
   - Test reminder scheduling
   - Test reminder sending
   - Test reminder cancellation on appointment cancel

## Success Criteria

- [ ] Availability slots calculate correctly
- [ ] Double-booking prevented at DB level
- [ ] Round-robin assignment works
- [ ] Reminders schedule and send
- [ ] Public booking page works
- [ ] All status transitions work
- [ ] Events emitted correctly
- [ ] Tests pass with >80% coverage
