# MODULE M10: APPOINTMENT SYSTEM

**Estimated Lines:** 3,000
**Estimated Time:** 2 days
**Dependencies:** M4 (contacts), M1 (schema), M9 (outbox for reminders)

---

## PURPOSE

Complete appointment booking system with availability management, reminders, and public booking pages.

```
Contact requests appointment → Check availability → Book slot → Send confirmation → Remind before
```

---

## FILE STRUCTURE

```
/modules/appointments/
├── __init__.py
├── models.py                   # Appointment, Slot, Rule models
├── availability/
│   ├── __init__.py
│   ├── rules.py               # Business hours, exceptions
│   ├── slots.py               # Generate available slots
│   ├── conflicts.py           # Check double-booking
│   └── buffer.py              # Buffer time between appts
├── booking/
│   ├── __init__.py
│   ├── create.py              # Book appointment
│   ├── update.py              # Modify appointment
│   ├── cancel.py              # Cancel with reason
│   └── status.py              # Status transitions
├── reminders/
│   ├── __init__.py
│   ├── scheduler.py           # Schedule reminder jobs
│   ├── templates.py           # Reminder message templates
│   └── sender.py              # Send via outbox
├── calendar_sync/
│   ├── __init__.py
│   ├── google.py              # Google Calendar API
│   └── ical.py                # iCal export/import
├── booking_pages/
│   ├── __init__.py
│   ├── generator.py           # Generate public booking URL
│   ├── renderer.py            # Render booking widget
│   └── embed.py               # Embeddable iframe code
├── executor.py                 # Main CRUD operations
└── tests/
    ├── test_availability.py
    ├── test_booking.py
    └── test_reminders.py
```

---

## DATABASE SCHEMA

```sql
-- Appointment Types (templates)
CREATE TABLE appointment_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id),
    name VARCHAR(100) NOT NULL,           -- "15-min Consultation"
    slug VARCHAR(100) NOT NULL,           -- "15-min-consultation"
    description TEXT,
    duration_minutes INT NOT NULL DEFAULT 30,
    buffer_before_minutes INT DEFAULT 0,
    buffer_after_minutes INT DEFAULT 15,
    color VARCHAR(7) DEFAULT '#3B82F6',   -- For calendar display
    price_cents INT DEFAULT 0,            -- For paid appointments
    requires_confirmation BOOLEAN DEFAULT false,
    max_per_day INT,                      -- Limit bookings
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(location_id, slug)
);

-- Availability Rules
CREATE TABLE availability_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id),
    appointment_type_id UUID REFERENCES appointment_types(id),
    day_of_week INT NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),
    start_time TIME NOT NULL,             -- 09:00:00
    end_time TIME NOT NULL,               -- 17:00:00
    is_available BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(location_id, appointment_type_id, day_of_week)
);

-- Availability Overrides (holidays, special days)
CREATE TABLE availability_overrides (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id),
    date DATE NOT NULL,
    is_available BOOLEAN DEFAULT false,
    start_time TIME,
    end_time TIME,
    reason VARCHAR(255),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(location_id, date)
);

-- Appointments
CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id),
    contact_id UUID NOT NULL REFERENCES contacts(id),
    appointment_type_id UUID REFERENCES appointment_types(id),
    assigned_to UUID REFERENCES users(id),
    
    title VARCHAR(255) NOT NULL,
    description TEXT,
    
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    timezone VARCHAR(50) NOT NULL,
    
    status VARCHAR(20) NOT NULL DEFAULT 'scheduled'
        CHECK (status IN ('scheduled', 'confirmed', 'completed', 
                          'cancelled', 'no_show', 'rescheduled')),
    
    -- Reminder tracking
    reminder_24h_sent_at TIMESTAMPTZ,
    reminder_1h_sent_at TIMESTAMPTZ,
    
    -- Booking source
    source VARCHAR(50) DEFAULT 'manual'
        CHECK (source IN ('manual', 'booking_page', 'api', 'ai', 'import')),
    booking_page_url TEXT,
    
    -- Cancellation
    cancelled_at TIMESTAMPTZ,
    cancelled_by UUID REFERENCES users(id),
    cancellation_reason TEXT,
    
    -- Rescheduling
    rescheduled_from UUID REFERENCES appointments(id),
    rescheduled_to UUID REFERENCES appointments(id),
    
    notes TEXT,
    metadata JSONB DEFAULT '{}',
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- Prevent double-booking
    EXCLUDE USING gist (
        location_id WITH =,
        assigned_to WITH =,
        tstzrange(start_time, end_time) WITH &&
    ) WHERE (status NOT IN ('cancelled', 'rescheduled'))
);

-- Indexes
CREATE INDEX idx_appointments_location_start ON appointments(location_id, start_time);
CREATE INDEX idx_appointments_contact ON appointments(contact_id);
CREATE INDEX idx_appointments_status ON appointments(status) WHERE status = 'scheduled';
CREATE INDEX idx_appointments_reminders ON appointments(start_time) 
    WHERE reminder_24h_sent_at IS NULL AND status = 'scheduled';
```

---

## API ENDPOINTS

```
# Appointments
GET    /api/appointments              - List appointments (with filters)
POST   /api/appointments              - Create appointment
GET    /api/appointments/:id          - Get single appointment
PUT    /api/appointments/:id          - Update appointment
DELETE /api/appointments/:id          - Cancel appointment
POST   /api/appointments/:id/confirm  - Confirm appointment
POST   /api/appointments/:id/complete - Mark as completed
POST   /api/appointments/:id/no-show  - Mark as no-show

# Availability
GET    /api/availability              - Get available slots for date range
GET    /api/availability/rules        - Get availability rules
PUT    /api/availability/rules        - Set availability rules
POST   /api/availability/overrides    - Add date override

# Appointment Types
GET    /api/appointment-types         - List appointment types
POST   /api/appointment-types         - Create type
PUT    /api/appointment-types/:id     - Update type
DELETE /api/appointment-types/:id     - Delete type

# Booking Pages
GET    /api/booking-pages             - List booking pages
POST   /api/booking-pages             - Create booking page
GET    /api/booking-pages/:slug       - Get booking page config

# Public (no auth)
GET    /book/:location_slug/:page_slug       - Public booking page
GET    /book/:location_slug/:page_slug/slots - Get available slots
POST   /book/:location_slug/:page_slug       - Book appointment
```

---

## TEST CRITERIA

- [ ] Can create appointment for contact
- [ ] Prevents double-booking same slot (uses PostgreSQL EXCLUDE)
- [ ] Returns only available slots within business hours
- [ ] Respects buffer times between appointments
- [ ] Handles timezone conversion correctly
- [ ] Reminder scheduled when appointment created
- [ ] Status transitions work (scheduled → confirmed → completed)
- [ ] Cancellation records reason and timestamp
- [ ] Public booking page works without auth
- [ ] iCal export generates valid .ics file

---

## LINE ESTIMATES

| Directory | Files | Lines |
|-----------|-------|-------|
| availability/ | 4 | 600 |
| booking/ | 4 | 500 |
| reminders/ | 3 | 400 |
| calendar_sync/ | 2 | 500 |
| booking_pages/ | 3 | 400 |
| models.py | 1 | 200 |
| executor.py | 1 | 400 |
| **TOTAL** | **18** | **3,000** |
