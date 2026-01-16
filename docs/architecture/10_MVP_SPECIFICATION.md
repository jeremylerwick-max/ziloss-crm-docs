# Ziloss CRM - MVP Specification

**Source:** Gemini Research Validation (January 2026)
**Status:** FINAL - Ready for Development

---

## The Pitch

> **"The Agency OS that doesn't nickel-and-dime you, doesn't crash, and actually lets you own your brand."**

---

## Executive Summary

Based on comprehensive research into GHL pain points and agency usage patterns, Ziloss will compete by shifting from "all-in-one breadth" to "stability and speed depth."

**Why agencies are leaving GHL:**
1. Feature bloat
2. "Glitchy" behavior
3. Hidden rebilling costs
4. White-label leaks

**What they want:**
1. Core MVP that works flawlessly
2. Transparent pricing
3. True white-label
4. Vertical-specific features

---

## The Essential Core (Build First)

### 1. Unified Inbox (The Daily Driver)
**Priority:** #1 - Non-negotiable
**GHL Criticism:** Mobile app is generic and slow

**Requirements:**
- Consolidate SMS, Email, WhatsApp, Social DMs
- Single chronological stream per contact
- Real-time updates via WebSocket
- Fast, native mobile experience
- Quick actions (reply, tag, assign, snooze)

**Differentiator:** Speed and mobile reliability

### 2. Visual Automation Builder
**Priority:** #2 - ROI Driver
**GHL Criticism:** Complex, premium actions charge per execution

**Requirements:**
- If/then logic for lead nurturing
- Email/SMS sequences
- Appointment reminders
- Pipeline stage triggers
- NO per-execution charges

**Differentiator:** Reliable delivery, transparent pricing

### 3. CRM Pipelines
**Priority:** #3 - Visual Heartbeat
**GHL Criticism:** Can be sluggish with large datasets

**Requirements:**
- Kanban view (New → Contacted → Booked → Won)
- Drag-and-drop cards
- Custom stages per pipeline
- Filters and search
- Bulk actions

**Differentiator:** Speed with large contact lists

### 4. Calendars & Appointments
**Priority:** #4 - Calendly Replacement
**GHL Criticism:** Setup is complex

**Requirements:**
- Public booking pages
- Round-robin scheduling for teams
- Buffer times between appointments
- SMS/email reminders (24h, 1h)
- Google Calendar sync

**Differentiator:** Simple setup, just works

### 5. Funnels & Forms
**Priority:** #5 - Landing Pages
**GHL Criticism:** Templates less polished than ClickFunnels

**Requirements:**
- Drag-and-drop landing page builder
- Mobile-responsive templates
- Form → CRM instant sync
- A/B testing (Phase 2)
- Custom domains

**Differentiator:** Speed (instant form-to-CRM)

---

## The Bloat (Skip Entirely)

| Feature | Decision | Reason | Alternative |
|---------|----------|--------|-------------|
| **Social Planner** | ❌ CUT | "Shelfware" - agencies use Hootsuite/Buffer | Integration only |
| **Communities** | ❌ CUT | Skool wins, GHL is "ghost town" | None needed |
| **Memberships** | ❌ CUT | Kajabi is superior for courses | None needed |
| **Blogging** | ❌ CUT | WordPress always wins for SEO | WordPress plugin |
| **Affiliate Manager** | ❌ CUT | Niche, FirstPromoter is better | None needed |

**Impact:** Reduces development overhead by ~30,000 lines, simplifies UI

---

## The Switching Triggers (Differentiators)

### A. Transparent Pricing (No "Wallet Hell")

**GHL Problem:**
> "One of the biggest complaints is the 'hidden cost' of the SaaS Mode Wallet, where agencies must act as a bank for their clients' SMS credits."

**Ziloss Solution:**
1. **BYO Twilio/Mailgun:** Agencies connect their own API keys
2. **Flat-rate option:** Unlimited SMS/email for predictable billing
3. **No wallet system:** Simple per-seat pricing
4. **Transparent AI costs:** Claude usage included in plan

**Sales Pitch:** "No more explaining billing to clients"

### B. True White-Label (Zero-Leak Guarantee)

**GHL Problem:**
> "Agencies hate that GHL 'leaks' its identity through generic help docs, Zapier integrations, and app.gohighlevel redirects."

**Ziloss Solution:**
1. **Custom domains everywhere:** Login, app, API docs, error pages
2. **No root domain leaks:** Never see "ziloss.com" anywhere
3. **White-label Zapier app:** Template for agencies to publish
4. **Branded mobile app:** Included, not $497/month extra

**Sales Pitch:** "Zero-Leak guarantee - your brand, completely"

### C. Vertical-Specific: Real Estate

**GHL Problem:**
> "Building Native IDX (MLS integration) would instantly win over thousands of agencies who currently have to hack GHL with iFrames."

**Ziloss Solution:**
1. **Native IDX integration:** Live MLS listings on website
2. **Property view triggers:** "User viewed 123 Main St" starts workflow
3. **Behavioral tracking:** Know what properties leads browse
4. **Pre-built RE snapshots:** Pipelines, automations, templates

**Sales Pitch:** "The only CRM that speaks MLS"

### D. Native Voice AI

**GHL Problem:**
> "Instead of forcing users to use webhooks to connect to Vapi or Bland.ai (which is technical and fragile), build Voice AI natively."

**Ziloss Solution:**
1. **Built-in Voice AI:** No webhook setup required
2. **Inbound call handling:** AI answers, qualifies, books
3. **Outbound AI calls:** Speed-to-lead automation
4. **Real Estate scripts:** Pre-built qualification flows

**Sales Pitch:** "AI that actually calls your leads"

---

## Feature Comparison Matrix

| Feature | GHL | HubSpot | Ziloss MVP |
|---------|-----|---------|------------|
| Unified Inbox | ✅ (slow mobile) | ✅ | ✅ (fast) |
| Workflows | ✅ (per-action fees) | ✅ | ✅ (flat rate) |
| CRM Pipelines | ✅ | ✅ | ✅ |
| Calendars | ✅ | ✅ | ✅ |
| Landing Pages | ✅ | ✅ | ✅ |
| White-label | ⚠️ (leaks) | ❌ | ✅ (zero-leak) |
| Transparent Pricing | ❌ (hidden fees) | ❌ (per contact) | ✅ |
| Native Voice AI | ❌ | ❌ | ✅ |
| IDX/MLS Integration | ❌ | ❌ | ✅ |
| Social Planner | ✅ (unused) | ✅ | ❌ (skip) |
| Communities | ✅ (ghost town) | ❌ | ❌ (skip) |
| Blogging | ✅ (basic) | ✅ | ❌ (skip) |

---

## Development Priorities

### Week 1-2: Core Backend
- [ ] M8: AI Engine (intent, sentiment, responses)
- [ ] M10: Appointment System (booking, reminders)
- [ ] M11-13: Background Workers (outbox, monitors)

### Week 3-4: Frontend Core
- [ ] M14: Dashboard UI
- [ ] M15: Inbox UI (the daily driver)
- [ ] M16: Contacts UI
- [ ] M17: Calendar UI

### Week 5-6: Workflow Engine
- [ ] M19: Visual Workflow Builder (backend)
- [ ] M19: Workflow Canvas (frontend)

### Week 7-8: Polish & Launch
- [ ] M20: White-label system
- [ ] Testing & bug fixes
- [ ] Documentation
- [ ] Beta launch (Pine & Beckett)

### Phase 2 (Post-MVP)
- [ ] IDX/MLS Integration
- [ ] Native Voice AI (Vapi integration)
- [ ] Funnel Builder
- [ ] Mobile app

---

## Success Metrics

| Metric | Target | Why |
|--------|--------|-----|
| Inbox load time | < 500ms | "Speed" is differentiator |
| Workflow reliability | 99.9% | "Doesn't crash" is the pitch |
| White-label leaks | 0 | "Zero-leak guarantee" |
| Setup time | < 30 min | Reduce onboarding friction |
| First SMS sent | < 5 min | Speed-to-value |

---

## Target Customer

**Primary:** Real Estate agencies and brokerages
- Already frustrated with GHL IDX gaps
- High volume SMS for speed-to-lead
- Voice AI for lead qualification
- Your mom's brokerage is proof of concept

**Secondary:** Local service agencies
- Plumbers, roofers, HVAC
- Simple CRM needs
- Appointment booking focus
- Price sensitive (hate GHL fees)

---

## The Bottom Line

**Build this:**
> A lightning-fast CRM + Inbox + Calendar + Funnel builder with native Voice AI and transparent billing.

**Ignore this:**
> Social posting, blogs, courses, and complex affiliate tools.

**Win with:**
> Speed, stability, transparency, and vertical depth.
