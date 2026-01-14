# ZILOSS AI CRM - PHASE 6: UI DASHBOARD

## 🎯 OBJECTIVE
Complete the web UI for Ziloss AI CRM - a GoHighLevel replacement.

**Project Location:** `/Users/mac/Desktop/agent-orchestrator`
**UI Location:** `/Users/mac/Desktop/agent-orchestrator/ui`
**API:** `http://localhost:8000`

---

## ✅ WHAT'S ALREADY BUILT (Don't Recreate)

### Backend (Phase 1-5 Complete)
- FastAPI at `localhost:8000`
- PostgreSQL + Redis via Docker
- Full CRM schema (contacts, conversations, messages, appointments)
- Twilio SMS send/receive (+1 801-212-9267)
- AI auto-response (Claude Haiku)
- DND/compliance enforcement
- Natural language command API (`/nl/command`)

### UI Scaffolding (Exists but needs completion)
- React + Vite + TypeScript + TailwindCSS
- React Query for data fetching
- React Router for navigation
- Lucide icons

### Existing Pages (`/ui/src/pages/`)
- `ContactsPage.tsx` - Contact list + detail view ✅
- `ConversationsPage.tsx` - Message threads ✅
- `AppointmentsPage.tsx` - Calendar view ✅
- `CommandPage.tsx` - NL interface ✅
- `Layout.tsx` - Sidebar navigation ✅

### API Client (`/ui/src/api/client.ts`)
- contactsApi, conversationsApi, messagesApi, appointmentsApi, nlApi

---

## 🚀 WHAT TO BUILD

### 1. DASHBOARD PAGE (Priority #1)
Create `/ui/src/pages/DashboardPage.tsx`

**Metrics to Display:**
- Total contacts count
- New leads today
- Open conversations count
- Appointments this week
- Messages sent/received today
- Response rate (% of inbound messages answered)

**Layout:**
- Grid of metric cards at top
- Recent activity feed below
- Quick action buttons (New Contact, Send SMS, Schedule)

**API Calls Needed:**
```typescript
// Add to client.ts
GET /crm/stats - Returns dashboard metrics
GET /crm/activity?limit=10 - Returns recent activity
```

### 2. INBOX VIEW UPGRADE (Priority #2)
Improve `/ui/src/pages/ConversationsPage.tsx`

**Add:**
- Contact name display (not just ID)
- Channel icon (SMS, Email, etc.)
- Unread badge styling
- Quick reply templates
- Mark as read/unread
- Assign to user
- Close/reopen conversation

**Real-time Updates:**
- WebSocket connection for new messages
- Toast notifications for inbound SMS

### 3. CONTACT DETAIL ENHANCEMENT (Priority #3)
Improve contact detail panel in `ContactsPage.tsx`

**Add:**
- Conversation history for this contact
- Tag management (add/remove tags inline)
- DND toggle
- Send SMS button (opens compose modal)
- Custom fields display
- Activity timeline

### 4. WEBSOCKET INTEGRATION (Priority #4)
Create `/ui/src/hooks/useWebSocket.ts`

```typescript
// Connect to: ws://localhost:8000/ws/{location_id}
// Events to handle:
// - new_message: Add to conversation list
// - message_status: Update delivery status
// - new_contact: Refresh contact list
// - appointment_reminder: Show notification
```

### 5. GLOBAL COMMAND BAR (Priority #5)
Add Cmd+K / Ctrl+K shortcut that opens command palette

**Features:**
- Search contacts by name/phone/email
- Quick actions: "Send SMS to John", "Schedule meeting"
- Navigate: "Go to conversations"
- Uses existing NL API

---

## 📁 FILE STRUCTURE TO CREATE

```
ui/src/
├── pages/
│   ├── DashboardPage.tsx      # NEW
│   └── (existing pages)
├── components/
│   ├── Dashboard/
│   │   ├── MetricCard.tsx     # NEW
│   │   ├── ActivityFeed.tsx   # NEW
│   │   └── QuickActions.tsx   # NEW
│   ├── Inbox/
│   │   ├── ConversationItem.tsx  # NEW (extract from page)
│   │   ├── MessageBubble.tsx     # NEW
│   │   └── QuickReplyBar.tsx     # NEW
│   ├── Contact/
│   │   ├── TagManager.tsx     # NEW
│   │   ├── ContactTimeline.tsx # NEW
│   │   └── SendSMSModal.tsx   # NEW
│   └── CommandPalette.tsx     # NEW (Cmd+K)
├── hooks/
│   ├── useWebSocket.ts        # NEW
│   └── useCommandPalette.ts   # NEW
└── api/
    └── client.ts              # UPDATE (add stats endpoint)
```

---

## 🎨 DESIGN GUIDELINES

**Color Palette:**
- Primary: Blue-500 (#3B82F6)
- Success: Green-500 (#22C55E)
- Warning: Yellow-500 (#EAB308)
- Error: Red-500 (#EF4444)
- Background: Gray-50 (#F9FAFB)
- Sidebar: Gray-900 (#111827)

**Typography:**
- Headings: font-bold
- Body: font-normal
- Muted: text-gray-500

**Components:**
- Cards: bg-white rounded-lg border shadow-sm
- Buttons: rounded-lg with hover states
- Inputs: rounded-lg border focus:ring-2 focus:ring-blue-500
- Badges: rounded-full px-2 py-0.5 text-xs

---

## 🔧 COMMANDS

```bash
# Start backend
cd /Users/mac/Desktop/agent-orchestrator
docker-compose up -d
source .venv/bin/activate
uvicorn orchestrator.api:app --host 0.0.0.0 --port 8000 --reload

# Start UI dev server  
cd /Users/mac/Desktop/agent-orchestrator/ui
npm run dev
# Opens at http://localhost:5173

# API docs
http://localhost:8000/docs
```

---

## 📋 ACCEPTANCE CRITERIA

### Dashboard
- [ ] Shows 6+ key metrics with icons
- [ ] Metrics update when data changes
- [ ] Recent activity shows last 10 events
- [ ] Quick action buttons work

### Inbox
- [ ] Shows contact name (not just ID)
- [ ] Real-time message updates via WebSocket
- [ ] Can mark conversations as read
- [ ] Shows channel type icon

### Contacts
- [ ] Can add/remove tags inline
- [ ] Can toggle DND status
- [ ] Shows conversation history
- [ ] Send SMS modal works

### Command Palette
- [ ] Opens with Cmd+K
- [ ] Searches contacts in real-time
- [ ] Can execute NL commands
- [ ] Keyboard navigable

---

## ⚠️ IMPORTANT NOTES

1. **Don't break existing functionality** - Pages already work, enhance them
2. **Use existing API client** - Add to `client.ts`, don't create new files
3. **Match existing code style** - React Query, TypeScript, Tailwind
4. **Default location ID:** `00000000-0000-0000-0000-000000000001`
5. **Backend is already running** - Just start the UI

---

## 🚦 START HERE

1. Verify backend is running: `curl http://localhost:8000/health`
2. Start UI: `cd ui && npm run dev`
3. Open http://localhost:5173
4. Begin with DashboardPage.tsx

---

*Phase 6 of Ziloss AI CRM*
*Target: Complete functional UI*
*Time estimate: 2-3 days*
