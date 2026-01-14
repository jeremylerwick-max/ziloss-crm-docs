################################################################################
#                                                                              #
#                    ZILOSS TECHNOLOGIES - MASTER PROJECT FILE                 #
#                                                                              #
#                         Owner: Jeremy Lerwick                                #
#                         Last Updated: January 11, 2026                       #
#                                                                              #
################################################################################

================================================================================
TABLE OF CONTENTS
================================================================================

1. ENABLEDSYNC PRO - Lead automation for Pine & Beckett Realtors
2. CLAUDE REMINDER SYSTEM - Autonomous reminders with iMessage
3. ZILOSS MEMORY SERVER - Persistent memory for Claude
4. CHS COACHING JOURNALS - Health tracking web app
5. GOHIGHLEVEL DND REVIEW - Contact cleanup project (PAUSED)
6. VOICE AGENTS - Riley and others for real estate
7. GHL AUDIT SYSTEM - Automated data quality audits for GoHighLevel
8. OKX TRADING BOT - BTC/USD spot trading with trend-following strategy
9. ZILOSS AI CRM - Full GoHighLevel replacement (IN PROGRESS)

================================================================================
================================================================================
##                                                                            ##
##                     1. ENABLEDSYNC PRO                                     ##
##                                                                            ##
================================================================================
================================================================================

PURPOSE: Automates cross-referencing Google Sheets leads with EnabledPlus CRM.
         Extracts appointment status, demo/sale flags, contract amounts.
         Runs 4x daily automatically on Railway.

CLIENT: Renewal by Andersen (Esler franchise locations)

--------------------------------------------------------------------------------
PRODUCTION DEPLOYMENT (RAILWAY)
--------------------------------------------------------------------------------

Project Name: "amusing-light"
Service Name: "grateful-fascination"
GitHub Repo:  https://github.com/jeremylerwick-max/enabledsync-railway
Railway URL:  https://railway.com/project/e12d8872-e3b4-4419-82e5-ded192fcd3c4

Schedule (Cron): 0 6,12,18,0 * * *
  - 12:00 AM UTC (5:00 PM MST)
  - 6:00 AM UTC  (11:00 PM MST)
  - 12:00 PM UTC (5:00 AM MST)
  - 6:00 PM UTC  (11:00 AM MST)

Environment Variables (Railway):
  - GOOGLE_CREDENTIALS_JSON  = (Base64 encoded service account)
  - GOOGLE_SHEET_ID          = 1ckdoAS7xci9ngmN_4fzEtDJ-LL5GUcjjtXCL9I-5UAY
  - ENABLEDPLUS_USERNAME     = BJupina
  - ENABLEDPLUS_PASSWORD     = !1234Bj

--------------------------------------------------------------------------------
LOCAL FILES
--------------------------------------------------------------------------------

Directory: /Users/mac/Desktop/enabledplus-automation/

  test_complete.py        - Local version (BLOCKED by default) - ACTIVE DEV
  backup_system.py        - Backup/restore utility
  google-credentials.json - Google service account
  PROJECT_INFO.md         - This section extracted

Railway Files: /Users/mac/Desktop/enabledplus-automation/railway/
  enabledsync_railway.py  - Production script (older column mapping)
  Dockerfile              - Chrome + Python container
  requirements.txt        - Dependencies
  railway.json            - Config with cron

Backups: /Users/mac/Desktop/enabledplus-automation/backups/
  hourly/   (keep 6)
  daily/    (keep 7)
  weekly/   (keep 4)
  monthly/  (keep 3)

--------------------------------------------------------------------------------
GOOGLE SHEET
--------------------------------------------------------------------------------

Sheet ID:  1ckdoAS7xci9ngmN_4fzEtDJ-LL5GUcjjtXCL9I-5UAY
Sheet URL: https://docs.google.com/spreadsheets/d/1ckdoAS7xci9ngmN_4fzEtDJ-LL5GUcjjtXCL9I-5UAY
Sheet Name: RBA - Esler - SMS - Appointment

Tabs:
  - CO - 945 Fill Forms      (Colorado)
  - PHI - 830 Fill Forms     (Philadelphia)
  - PHX - 720 Fill Forms     (Arizona)
  - SNE - 905 Fill Forms     (Southern New England)
  - CT - 706 Fills Forms     (Central Texas)
  - NT - 703 Fills Forms     (North Texas)
  - OK - 708 Fill Forms      (Oklahoma)
  - Maine & NH - 101         (Portland Maine)
  - 714 Nevada - Fill forms  (Nevada)

================================================================================
                         COLUMN MAPPINGS (test_complete.py)
================================================================================

INPUT COLUMNS (from GoHighLevel):
  Column | Index | Field            | Description
  -------|-------|------------------|----------------------------------
  A      | 1     | Submission Date  | When lead was submitted
  B      | 2     | First Name       | Customer first name
  C      | 3     | Last Name        | Customer last name
  E      | 5     | Address          | Street address
  F      | 6     | City             | City
  G      | 7     | State            | State abbreviation (CO, PA, TX)
  H      | 8     | Zip              | Zip code
  I      | 9     | Phone            | Phone number (search key)

OUTPUT COLUMNS (written by EnabledSync):
  Column | Index | Field              | Description
  -------|-------|--------------------|----------------------------------
  K      | 11    | Enabled Source     | Lead source from EnabledPlus
  L      | 12    | Enabled Breakdown  | Lead breakdown/list name
  M      | 13    | Appointment Date   | Scheduled appointment date
  N      | 14    | Enabled Results    | Date + Status (e.g., "12/15/2025 Demo No Sale")
  O      | 15    | NonBuyer Lead Demo | "1" if demo/sale AND NonBuyer
  P      | 16    | Buyer Lead Demo    | "1" if demo/sale AND Buyer
  Q      | 17    | Customer Type      | Buyer/NonBuyer (pre-populated, don't touch)
  R      | 18    | NonBuyer Sale      | "1" if sale AND NonBuyer
  S      | 19    | NonBuyer Net Sale  | Dollar amount if sale AND NonBuyer
  T      | 20    | Buyer Sale         | "1" if sale AND Buyer
  U      | 21    | Buyer Net Sale     | Dollar amount if sale AND Buyer

================================================================================
                              SKIP LOGIC
================================================================================

The script processes leads based on what's already in Column N (Enabled Results):

  Column N Status    | Action           | Reason
  -------------------|------------------|------------------------------------------
  "Sale"             | SKIP FOREVER     | Final outcome - customer purchased
  "Demo No Sale"     | SKIP FOREVER     | Final outcome - demo completed, no purchase
  (empty)            | PROCESS          | Never checked before
  "Not Found"        | RE-CHECK         | May now have results in EnabledPlus
  "CallBack"         | RE-CHECK         | Pending, may have updated
  "Answering Machine"| RE-CHECK         | Pending, may have updated
  "Left Message"     | RE-CHECK         | Pending, may have updated
  "Not Interested"   | RE-CHECK         | Could get additional follow-up
  Any other status   | RE-CHECK         | Reps update EnabledPlus daily

RATIONALE: Final outcomes (Sale, Demo No Sale) never change. Everything else
could be updated by sales reps daily, so we re-check to capture updates.

================================================================================
                           SEARCH STRATEGY
================================================================================

Cascading search in EnabledPlus until a match is found:

  Priority | Method    | Reliability | Notes
  ---------|-----------|-------------|------------------------------------------
  1        | Phone     | Highest     | Most reliable, exact match by digits
  2        | Last Name | Medium      | If phone fails, try name match
  3        | Address   | Lower       | If name fails, try address match
  4        | City      | Fallback    | Last resort, many potential matches

Each search method clicks into the lead detail page and extracts:
  - Source (Column K)
  - Breakdown (Column L)
  - History table for status, dates, appointment info

================================================================================
                         TAB-TO-REGION MAPPING
================================================================================

Each Google Sheet tab maps to an EnabledPlus region dropdown:

  Tab Prefix | EnabledPlus Region      | States Covered
  -----------|-------------------------|-----------------------------
  CO         | Colorado                | CO
  PHI        | Philly                  | PA, NJ, DE, MD
  PHX / AZ   | Arizona                 | AZ
  SNE        | Southern New England    | CT, RI, MA, NH, VT
  NT         | North Texas             | Dallas metro area (TX)
  CT         | Central Texas           | Austin, San Antonio (TX)
  OK         | Oklahoma                | OK
  NV / 714   | Nevada                  | NV, UT
  ME / MAINE | Portland Maine          | ME

TEXAS SPECIAL RULE: If tab doesn't specify, city determines region:
  - North Texas: Dallas, Fort Worth, Plano, Arlington, Frisco, McKinney, etc.
  - Central Texas: Austin, San Antonio, Waco, Temple, Round Rock, etc.

================================================================================
                        HISTORY TABLE LOOKBACK
================================================================================

- Only examines history entries from the LAST 20 DAYS
- Older entries are ignored (assumes already captured in previous runs)
- Cutoff calculated as: today - 20 days

================================================================================
                          RESULT STATUSES
================================================================================

The script tracks these statuses from EnabledPlus history:

  Status          | Is Demo? | Is Sale? | Column O/P | Column R/S/T/U
  ----------------|----------|----------|------------|----------------
  "Sale"          | No       | YES      | "1"        | "1" + amount
  "Demo No Sale"  | YES      | No       | "1"        | (none)
  "Not Interested"| No       | No       | (none)     | (none)
  "CallBack"      | No       | No       | (none)     | (none)
  "DNS"           | No       | No       | (none)     | (none)

================================================================================
                        BUYER vs NONBUYER LOGIC
================================================================================

Determines which columns get updated (O/P vs R/S/T/U):

  Type      | Definition                              | Demo Column | Sale Columns
  ----------|-----------------------------------------|-------------|-------------
  Buyer     | Has sale record BEFORE submission date  | P (16)      | T (20), U (21)
  NonBuyer  | No prior sales (first-time customer)    | O (15)      | R (18), S (19)

The check_if_buyer() function scans the history table for "Sale" entries
dated before the lead's submission date from Column A.

================================================================================
                    MULTIPLE MATCH DISAMBIGUATION
================================================================================

When multiple EnabledPlus records match the same search criteria:

  Priority | Method                | Description
  ---------|----------------------|--------------------------------------------
  1        | LeadCertain Sources  | Prioritize leads from Database Nurture Digital,
           |                      | Lead Certain, Nurture Digital sources
  2        | Address Matching     | Word overlap scoring between addresses
  3        | First Match          | If still tied, use first result

LeadCertain keywords: 'database nurture', 'database nurturing', 'lead nurturing',
                      'lead certain', 'nurture digital'

================================================================================
                         SESSION MANAGEMENT
================================================================================

- Verifies session every 15 LEADS to prevent mid-run expiration
- Auto-recovers if EnabledPlus session times out
- Up to 3 retry attempts per search before failing the lead
- recover_session() function handles re-login and navigation

================================================================================
                           SAFETY CHECKS
================================================================================

LOCAL EXECUTION:
  - BLOCKED by default to prevent Railway conflicts
  - Use --dry-run for testing without writes
  - Use --local-override for emergency (confirms Railway disabled)
  - Use --test to write to TEST OUTPUT sheet instead

FLAGS:
  --dry-run / -d        Test without writing to sheets
  --test / -t           Write to TEST OUTPUT sheet
  --local-override      Emergency: run locally (type 'YES' to confirm)
  --headless            Run without visible browser
  --yes / -y            Skip confirmation prompts

================================================================================
                        BUG FIXES LOG
================================================================================

2025-01-04: Column L Bug Fix - Production Verified
  - PROBLEM: Column L showed "Not Found" even when Column N had valid results
  - ROOT CAUSE: extract_source_breakdown() couldn't handle BOTH formats:
    * Same-line: "Source:Database Nurture Digital"
    * Next-line: "Source:" on line 1, value on line 2
  - FIX: Rewrote extraction with helper function that tries both patterns:
    1. Same-line regex: pattern + r'[:\s]+([^\n]+)'
    2. Next-line scan: find label line, return next non-empty line
  - PRODUCTION RUN: Full 9-tab run started ~1:45 PM MST
    * CO: 25/26 success (96%) - 1 test record expected fail
    * PHI: 58/59 success (98%)
    * PHX: 53/54 success (98%)
    * SNE, CT, NT, OK, Maine/NH, Nevada: In progress
  - ALSO FIXED: KeyError bugs (customer['address'] → customer.get('address', ''))

2025-01-03: Initial Bug Discovery
  - Identified Column L showing "Not Found" for valid leads
  - Investigation confirmed extraction function failing on certain page formats

================================================================================
                        DOCUMENTATION FILES
================================================================================

/Users/mac/Desktop/enabledplus-automation/BUSINESS_RULES.md
  - Complete business logic documentation
  - Skip logic, search strategy, column mappings
  - Buyer/NonBuyer determination
  - Multiple match disambiguation
  - All algorithm details

/Users/mac/Desktop/enabledplus-automation/CHANGELOG.md
  - Ongoing journal of changes and production runs
  - Bug fix history with root cause analysis
  - Production run statistics

================================================================================
                           CREDENTIALS
================================================================================

EnabledPlus:
  URL:      https://www.enabledplus.com
  Username: BJupina
  Password: !1234Bj

Google Service Account:
  File:     /Users/mac/Desktop/enabledplus-automation/google-credentials.json

================================================================================
                         BACKUP COMMANDS
================================================================================

export BACKUP_DIR=/Users/mac/Desktop/enabledplus-automation/backups

# Create backup
python3 backup_system.py backup hourly

# List backups
python3 backup_system.py list

# Restore (dry run)
python3 backup_system.py restore /path/to/backup

# Restore (actual)
python3 backup_system.py restore /path/to/backup --confirm

================================================================================
                         TROUBLESHOOTING
================================================================================

Check Railway: https://railway.com/project/e12d8872-e3b4-4419-82e5-ded192fcd3c4
  - Deployments tab for build status
  - Logs tab for runtime output
  - Cron tab for scheduled runs

Known Issues:
  - Barbara Bunn (PHX Row 24): #ERROR! in phone - fix in Sheet
  - Maine & NH: Test entries with fake phones - expected

Common Problems:
  - "Session lost": EnabledPlus timeout, auto-recovers
  - "Not Found": Lead may not exist in EnabledPlus or wrong region
  - "No results in 20 days": Lead exists but no recent activity

--------------------------------------------------------------------------------


================================================================================
================================================================================
##                                                                            ##
##                     2. CLAUDE REMINDER SYSTEM                              ##
##                                                                            ##
================================================================================
================================================================================

PURPOSE: Autonomous reminder system. Claude can schedule tasks, get texted 
         results, approve/reject actions via iMessage.

--------------------------------------------------------------------------------
DEPLOYMENT
--------------------------------------------------------------------------------

Railway API:    https://claude-reminders-production.up.railway.app
Railway Project: https://railway.com/project/de09e8ea-594a-4e1c-a35a-f8910b5ebbe7

Local Bridge:   /Users/mac/Desktop/claude-reminders/
  - imessage_bridge.py (runs on Mac, sends iMessages)
  - ngrok exposes port 8765

Your Phone: +19492300036

--------------------------------------------------------------------------------
START THE BRIDGE (Mac Terminal)
--------------------------------------------------------------------------------

cd /Users/mac/Desktop/claude-reminders
./start_bridge.sh

OR manually:
  ngrok http 8765 --domain=unlivable-venita-semiseriously.ngrok-free.dev
  python3 imessage_bridge.py

--------------------------------------------------------------------------------
API USAGE
--------------------------------------------------------------------------------

CREATE REMINDER:
curl -X POST https://claude-reminders-production.up.railway.app/reminders \
  -H "Content-Type: application/json" \
  -d '{
    "title": "My Task",
    "content": "Details",
    "action": "What to do",
    "trigger_time": "2025-12-17T12:00:00",
    "autonomy": "notify"
  }'

AUTONOMY LEVELS:
  🟢 auto    - Silent execution
  🟡 notify  - Texts you results
  🔴 approve - Asks permission first
  
  ⚠️ Actions with send/delete/deploy auto-upgrade to APPROVE

LIST REMINDERS:
  GET /reminders

PENDING APPROVALS:
  GET /approvals/pending

RESPOND TO APPROVAL:
  POST /approvals/respond

--------------------------------------------------------------------------------
FILES
--------------------------------------------------------------------------------

/Users/mac/Desktop/claude-reminders/
  main.py           - FastAPI server
  worker.py         - Cron job worker
  imessage_bridge.py - iMessage webhook
  mcp_server.py     - MCP tools
  start_bridge.sh   - Startup script

--------------------------------------------------------------------------------


================================================================================
================================================================================
##                                                                            ##
##                     3. ZILOSS MEMORY SERVER                                ##
##                                                                            ##
================================================================================
================================================================================

PURPOSE: Persistent memory for Claude across all conversations and devices.
         Stores conversation history, project context, decisions.

--------------------------------------------------------------------------------
DEPLOYMENT
--------------------------------------------------------------------------------

Railway URL:  https://ziloss-memory-server-production.up.railway.app
Database:     Encrypted SQLite with AES-256
Auth:         API key required

--------------------------------------------------------------------------------
MCP TOOLS (in Claude Desktop)
--------------------------------------------------------------------------------

memory:memory_search  - Search past memories
memory:memory_add     - Store new memory

Usage in Claude:
  "Search my memory for EnabledSync"
  "Save this to memory"

--------------------------------------------------------------------------------
MEMORY FALLBACK RULE
--------------------------------------------------------------------------------

If user references something not in context/built-in memory/past chats:
ALWAYS check memory:memory_search before saying "I don't know"

--------------------------------------------------------------------------------


================================================================================
================================================================================
##                                                                            ##
##                     4. CHS COACHING JOURNALS                               ##
##                                                                            ##
================================================================================
================================================================================

PURPOSE: Health tracking web app with 7 journal types for coaching clients.

--------------------------------------------------------------------------------
DEPLOYMENT
--------------------------------------------------------------------------------

Live URL:       https://web-production-be378.up.railway.app
Railway Project: https://railway.com/project/1eb95e57-d729-4511-9a3a-26e4b752bbad
Project Name:   "charming-manifestation"
Database:       Railway Postgres

--------------------------------------------------------------------------------
JOURNAL TYPES
--------------------------------------------------------------------------------

1. Daily Check-in
2. Gratitude
3. Food
4. Medications
5. Supplements
6. Exercise
7. Mood

--------------------------------------------------------------------------------
LOCAL FILES
--------------------------------------------------------------------------------

TODO: Add local file path when known

--------------------------------------------------------------------------------


================================================================================
================================================================================
##                                                                            ##
##                     5. GOHIGHLEVEL DND REVIEW                              ##
##                                                                            ##
================================================================================
================================================================================

PURPOSE: Review ~7,300 Esler CST contacts to find Inbound+SMS contacts 
         incorrectly marked DND (Do Not Disturb). Tag for manual review.

STATUS: ⏸️ PAUSED (switched to EnabledPlus automation)

--------------------------------------------------------------------------------
TARGET
--------------------------------------------------------------------------------

GoHighLevel Location: Esler CST
Location ID:          fRrP1e3LGLFewc5dQDhS

--------------------------------------------------------------------------------
SAFETY PROTOCOL (CRITICAL)
--------------------------------------------------------------------------------

MUST FOLLOW 3-PHASE PROCESS:

Phase 1: READ-ONLY analysis - fetch and report data, NO changes
Phase 2: TEST on 5 contacts only - requires explicit YES approval
Phase 3: FULL RUN - only after successful Phase 2 test

⚠️ Never deviate unless user explicitly says:
   "yes it is okay to deviate from phase 1 phase 2 plan"

--------------------------------------------------------------------------------
API REQUIREMENTS
--------------------------------------------------------------------------------

GoHighLevel API key needs scopes:
  - contacts.readonly
  - contacts.write
  - conversations.readonly
  - conversations.readonly.message

Create via: Settings > Private Integrations

--------------------------------------------------------------------------------
FINDINGS (Nov 13, 2025)
--------------------------------------------------------------------------------

- Fetched 10,100+ contacts successfully
- Contacts have NO "source" field (all null)
- Contacts have NO dndSettings data (all empty)
- All have tags: 'master list', 'okl'

BLOCKED: Need user to show how they filter in GHL UI

--------------------------------------------------------------------------------


================================================================================
================================================================================
##                                                                            ##
##                     6. VOICE AGENTS                                        ##
##                                                                            ##
================================================================================
================================================================================

PURPOSE: AI voice agents for Pine & Beckett Realtors lead handling.

--------------------------------------------------------------------------------
AGENTS
--------------------------------------------------------------------------------

Riley - Primary voice agent
  Platform: Vapi.ai / Retell AI
  Purpose:  Handle inbound calls, qualify leads

TODO: Add more details when available

--------------------------------------------------------------------------------
PLATFORMS
--------------------------------------------------------------------------------

Vapi.ai:   https://vapi.ai
Retell AI: https://retellai.com

--------------------------------------------------------------------------------


================================================================================
================================================================================
##                                                                            ##
##                     7. GHL AUDIT SYSTEM                                    ##
##                                                                            ##
================================================================================
================================================================================

PURPOSE: Automated data quality audit for GoHighLevel (GHL) CRM locations.
         Scans conversations and contacts to identify workflow conflicts,
         missed opportunities, and data inconsistencies.

STATUS: ✅ ACTIVE

--------------------------------------------------------------------------------
TARGET LOCATION
--------------------------------------------------------------------------------

Location:    Esler CST
Location ID: fRrP1e3LGLFewc5dQDhS
API Key:     pit-86cb2e15-313d-422c-89a3-4d91f69edd2b

Secondary:   Esler Email
Location ID: F4FDC6celS3tpC7iTP0z
API Key:     pit-ba752614-e64d-4f49-8be7-be1172c8a830

--------------------------------------------------------------------------------
ISSUES DETECTED
--------------------------------------------------------------------------------

1. STALE AI TAGS WITH DND (Medium)
   - Contacts have activate_dbr_ai but are DND
   - AI cannot send messages, creates workflow conflict
   - FIX: Remove AI tags when DND is applied

2. APPOINTMENT MISSING NOTIFICATION (High)
   - Appointment booked but no confirmation tag
   - Customer may not have received confirmation
   - FIX: Verify manually, fix workflow

3. APPOINTMENT WITH DND (Critical)
   - Booked appointment but marked DND
   - Won't receive reminders - likely no-show
   - FIX: Review case-by-case, call directly

4. REPLIED BUT NO OUTCOME (Medium)
   - Customer engaged but no resolution
   - Potential lost opportunity
   - FIX: Re-engage or mark outcome

5. UNREAD HIGH PRIORITY (Critical)
   - Unread messages with appointment intent
   - Hot leads going cold
   - FIX: Respond immediately

6. POTENTIAL MISSED BOOKINGS (High)
   - Positive response but no appointment tag
   - AI may have failed to book
   - FIX: Review and attempt rebooking

7. AI REPLYING STUCK (Low)
   - ai_replying tag but no activity 48+ hours
   - Creates reporting noise
   - FIX: Clean up stale tags

--------------------------------------------------------------------------------
AUDIT SCRIPTS
--------------------------------------------------------------------------------

Location: /Users/mac/Desktop/

ghl_comprehensive_audit.py  - Main audit (15K conversations)
ghl_audit_dnd_deep.py       - DND conflict deep dive
ghl_audit.py                - Legacy v1
ghl_audit_v2.py             - Legacy v2

--------------------------------------------------------------------------------
OUTPUT FILES
--------------------------------------------------------------------------------

/Users/mac/Desktop/GHL_COMPREHENSIVE_AUDIT.json  - Full audit results
/Users/mac/Desktop/GHL_DND_CONFLICT_DETAILS.json - DND details
/Users/mac/Desktop/GHL_AUDIT_SYSTEM.md           - Full documentation

--------------------------------------------------------------------------------
QUICK START
--------------------------------------------------------------------------------

# Run comprehensive audit
cd /Users/mac/Desktop && python3 ghl_comprehensive_audit.py

# Run DND deep dive
python3 ghl_audit_dnd_deep.py

--------------------------------------------------------------------------------
LATEST AUDIT RESULTS (Jan 4, 2026)
--------------------------------------------------------------------------------

Conversations Scanned:    15,000
Stale AI + DND:           19 contacts
Appt Missing Notification: 1 contact
Replied No Outcome:        1 contact

Key Finding: DND workflow doesn't remove activate_dbr_ai tags.
Recommendation: Update workflow to clean up AI tags on DND.

--------------------------------------------------------------------------------
RECOMMENDED SCHEDULE
--------------------------------------------------------------------------------

- Run comprehensive audit weekly (Monday AM)
- Review critical issues immediately
- Track trends over time

--------------------------------------------------------------------------------


================================================================================
================================================================================
##                                                                            ##
##                     GLOBAL RESOURCES                                       ##
##                                                                            ##
================================================================================
================================================================================

--------------------------------------------------------------------------------
RAILWAY PROJECTS SUMMARY
--------------------------------------------------------------------------------

| Project              | URL                                                    |
|----------------------|--------------------------------------------------------|
| EnabledSync Pro      | https://railway.com/project/e12d8872-e3b4-4419-82e5... |
| Claude Reminders     | https://railway.com/project/de09e8ea-594a-4e1c-a35a... |
| Memory Server        | https://ziloss-memory-server-production.up.railway.app |
| CHS Journals         | https://railway.com/project/1eb95e57-d729-4511-9a3a... |
| OKX Trading Bot      | https://railway.com/project/ce2d349a-47ab-43f9-9c31... |

--------------------------------------------------------------------------------
COMMON COMMANDS
--------------------------------------------------------------------------------

# Check all Railway projects
Open: https://railway.app/dashboard

# Git push to deploy (from project's railway folder)
git add -A && git commit -m "message" && git push

# Generate Base64 credentials
base64 -i /path/to/credentials.json | tr -d '\n'

--------------------------------------------------------------------------------
CONTACT
--------------------------------------------------------------------------------

Owner:   Jeremy Lerwick
Company: Ziloss Technologies
Phone:   +19492300036
Email:   (add email)

--------------------------------------------------------------------------------
HARDWARE
--------------------------------------------------------------------------------

Primary: 2017 iMac running macOS Ventura
GPUs:    6x RTX 6000 Ada (on order)

--------------------------------------------------------------------------------
CLAUDE MEMORY AGENT (Auto-Capture System)
--------------------------------------------------------------------------------

Location: ~/Desktop/claude-memory-agent/
Database: ~/Desktop/claude-memory-agent/claude_memory.db

HOW IT WORKS:
- Connects to Claude desktop app via Electron debug port
- Intercepts API responses (same data that renders on screen)
- Stores to local SQLite with full-text search
- Auto-tags messages by project (GetStacked, EnabledSync, etc.)
- Captures Claude's thinking/reasoning blocks

TO START CAPTURING:
```bash
# 1. Launch Claude with debug mode (kills existing, restarts with port 9223)
~/Desktop/claude-memory-agent/launch_claude_debug.sh

# 2. Run SQLite capture (in separate terminal)
python3 ~/Desktop/claude-memory-agent/capture_sqlite.py
```

SEARCH COMMANDS:
```bash
# Database stats (messages, projects, size)
python3 ~/Desktop/claude-memory-agent/search.py stats

# Full-text search
python3 ~/Desktop/claude-memory-agent/search.py search "keyword"

# Filter by project
python3 ~/Desktop/claude-memory-agent/search.py project GetStacked

# Recent messages
python3 ~/Desktop/claude-memory-agent/search.py recent 10

# Claude's reasoning/thinking
python3 ~/Desktop/claude-memory-agent/search.py thinking 5
```

SEMANTIC SEARCH (Vertex AI - understands meaning, not just keywords):
```bash
# Embed new messages to Vertex AI
python3 ~/Desktop/claude-memory-agent/vertex_embeddings.py embed

# Semantic search (finds related content even without exact words)
python3 ~/Desktop/claude-memory-agent/vertex_embeddings.py search "business strategy"

# Check embedding stats
python3 ~/Desktop/claude-memory-agent/vertex_embeddings.py stats
```

FILES:
- launch_claude_debug.sh  → Start Claude with debug port
- capture_sqlite.py       → Main capture script
- search.py               → Search utility
- db_setup.py            → Initialize database
- import_json.py         → Import old JSON captures
- claude_memory.db       → SQLite database

--------------------------------------------------------------------------------


################################################################################
#                           END OF MASTER PROJECT FILE                         #
################################################################################

================================================================================
VERTEX AI & MODEL ORCHESTRATION PLAN
================================================================================

Created: December 18, 2025
Status: PLANNED

--------------------------------------------------------------------------------
PHASE 1: VECTOR MEMORY SEARCH
--------------------------------------------------------------------------------

CURRENT SETUP:
  Claude Conversation
  "Let's price GetStacked at $99/month for the basic tier"
                    |
                    v
  capture_sqlite.py - Saves raw text to SQLite
                    |
                    v
  SQLite Database
  Text search only: "pricing" YES  "monetization" NO


WITH VERTEX AI EMBEDDINGS:
  Claude Conversation
  "Let's price GetStacked at $99/month for the basic tier"
                    |
                    v
  capture_sqlite.py - Saves raw text to SQLite
                    |
                    v
  Vertex AI Embedding API (text-embedding-004)
  Text -> [0.023, -0.456, 0.871, ...] (768 dimensions)
                    |
                    v
  Vertex AI Vector Search Index
  Query: "business model discussions"
  Finds: pricing convos, revenue talks, monetization chats
         (even without those exact words!)


SEARCH FLOW:
  You: "What did we decide about money stuff?"
                    |
                    v
  Query gets embedded: [0.019, -0.448, 0.892, ...]
                    |
                    v
  Vector math finds similar embeddings (cosine similarity)
                    |
                    v
  Returns: "$99/month pricing", "revenue model", "how to monetize"

--------------------------------------------------------------------------------
PHASE 2: FINE-TUNED ROUTER + WORKER MODELS  
--------------------------------------------------------------------------------

INCOMING REQUEST
"Check if EnabledSync has any errors in the last 24 hours"
                    |
                    v
ROUTER MODEL (Vertex AI - Fine-tuned Gemini Flash)
Trained on YOUR patterns:
- "EnabledSync" -> needs Railway logs
- "errors" -> simple check, not complex reasoning
- "24 hours" -> time-based query
DECISION: Route to cheap model, not Claude
                    |
          +---------+---------+
          |                   |
          v                   v
  SIMPLE TASKS           COMPLEX TASKS
  Gemini Flash           Claude Opus/Sonnet
  $0.00001/query         $0.015/query
  - Log checking         - Architecture
  - Status queries       - Complex code
  - Simple lookups       - Strategic planning

--------------------------------------------------------------------------------
PHASE 3: SPECIALIST WORKER MODELS (AI WORKFORCE)
--------------------------------------------------------------------------------

           SCOUT          ANALYST        BUILDER        ADVISOR
           |              |              |              |
           v              v              v              v
       Web search      Data work     Code tasks     Domain expert
       Research        Reports       Deploys        Q&A
       Monitoring      Summaries     Fixes          Health advice
           |              |              |              |
       Perplexity     Gemini Flash   Claude        Fine-tuned
       + Vertex                      Sonnet        Mistral
           |              |              |              |
       COST: $        COST: $        COST: $$$     COST: $$

                          |
                          v
              CLAUDE (ORCHESTRATOR)
              - Receives complex requests
              - Breaks into subtasks
              - Dispatches to specialists
              - Combines results
              - Makes final decisions

--------------------------------------------------------------------------------
EXAMPLE WORKFLOW
--------------------------------------------------------------------------------

YOU: "Prepare a weekly report on all my projects"

CLAUDE (Orchestrator):
|-- Dispatches to SCOUT:
|   "Check Railway for EnabledSync, Reminders"
|   "Check Google Cloud for CHS Journals"
|   Returns: service statuses, uptime, errors
|
|-- Dispatches to ANALYST:
|   "Summarize this week's conversations by project"
|   Uses: Vector search on memory DB
|   Returns: key decisions, blockers, progress
|
|-- Dispatches to SCOUT:
|   "Any relevant industry news for GetStacked?"
|   Returns: longevity/health supplement news
|
|-- CLAUDE combines everything:
    Generates formatted weekly report
    Highlights action items
    Sends to you

TOTAL COST: ~$0.05 (vs $0.50 if Claude did everything)

--------------------------------------------------------------------------------
VERTEX AI MODELS TO BUILD
--------------------------------------------------------------------------------

Model             | Purpose              | Base Model         | Training Data
------------------|----------------------|--------------------|------------------
Memory Embedder   | Vector search        | text-embedding-004 | None (pre-trained)
Router            | Task classification  | Gemini Flash       | 500 past convos
Log Analyzer      | Check services       | Gemini Flash       | Railway/GCP logs
GetStacked Advisor| Health Q&A           | Gemini Pro         | Longevity research

--------------------------------------------------------------------------------
COST ESTIMATE (Monthly)
--------------------------------------------------------------------------------

Component                         | Cost
----------------------------------|----------
Vector embeddings (memory search) | ~$5
Router model (fine-tuned)         | ~$10
Specialist queries (Gemini Flash) | ~$5-20
Claude (complex only)             | ~$20-50
TOTAL                             | ~$40-85

vs. Claude for everything: ~$200-400/mo

--------------------------------------------------------------------------------
IMPLEMENTATION STATUS
--------------------------------------------------------------------------------

[ ] Phase 1: Vector Memory Search
    [ ] Set up Vertex AI project
    [ ] Create embedding pipeline
    [ ] Connect to SQLite memory DB
    [ ] Build semantic search API

[ ] Phase 2: Router Model
    [ ] Export training data from conversations
    [ ] Fine-tune Gemini Flash
    [ ] Build routing logic

[ ] Phase 3: Specialist Workers
    [ ] Define specialist roles
    [ ] Fine-tune domain models
    [ ] Build dispatch system


================================================================================
================================================================================
##                                                                            ##
##                     8. OKX TRADING BOT                                     ##
##                                                                            ##
================================================================================
================================================================================

PURPOSE: Automated BTC/USD spot trading bot with conservative trend-following
         strategy. Uses SMA200 + RSI + ATR-based stops. Runs 24/7 with strict
         risk controls and circuit breakers.

STATUS: PAPER TRADING (Started January 6, 2026) - NOW ON RAILWAY
GOAL: Prove reliability + process, NOT maximize P&L during shakedown phase

--------------------------------------------------------------------------------
RAILWAY DEPLOYMENT (January 10, 2026)
--------------------------------------------------------------------------------

Project Name: okx-trading-bot
Project ID:   ce2d349a-47ab-43f9-9c31-fc0d9e726a73
Service ID:   3295565d-4cd8-4ea3-af45-d9fa222e643b
Railway URL:  https://railway.com/project/ce2d349a-47ab-43f9-9c31-fc0d9e726a73

Environment Variables (Railway):
  - OKX_KEY        = 97cb4be8-05b2-4bd0-92a8-bd2067dacf44
  - OKX_SECRET     = (configured)
  - OKX_PASSPHRASE = (configured)
  - PAPER_MODE     = true

Deployment Status: ✅ LIVE AND RUNNING
  - Connected to OKX US (https://us.okx.com)
  - Mode: PAPER trading
  - Symbol: BTC/USD
  - Checking every 60 seconds

--------------------------------------------------------------------------------
PROJECT FILES
--------------------------------------------------------------------------------

Directory: /Users/mac/Desktop/okx-trading-bot/

Core Bot Files:
  bot/
    __init__.py
    config.py         - All configuration parameters
    exchange.py       - OKX API wrapper with US domain fixes
    indicators.py     - SMA, RSI, ATR calculations
    state.py          - Position tracking, file locking, circuit breaker
    risk.py           - Position sizing, daily loss limits, order validation
    strategy.py       - Entry/exit logic (SMA200 + RSI + ATR)
    main.py           - Main orchestrator, startup checks, run loop
    utils.py          - Logging, telemetry helpers

Test Files:
  tests/
    test_indicators.py - SMA, RSI unit tests
    test_sizing.py     - Position sizing tests

Operational Scripts:
  flatten.py          - EMERGENCY: Sell all BTC, cancel orders, disable bot
  reconcile.py        - Verify local state matches exchange state
  boot_test.py        - Test bot boots cleanly 10x in a row
  test_atr.py         - Test ATR-based position sizing
  test_demo.py        - Raw API connectivity test
  test_order.py       - Test buy order execution
  test_sell.py        - Test sell order execution

Documentation:
  README.md              - Full documentation
  GO_LIVE_CHECKLIST.md   - Paper → Live checklist (must complete)
  AI_REVIEW_REQUEST.md   - Prompt for external AI review
  HANDOFF_DOCUMENT.md    - Complete technical spec for handoff

Runtime Files:
  state/state.json    - Bot state (position, trades, errors)
  logs/bot.log        - Full execution logs
  logs/telemetry.csv  - Trade telemetry for analysis
  .env                - API credentials (gitignored)

--------------------------------------------------------------------------------
OKX API CREDENTIALS (DEMO ACCOUNT)
--------------------------------------------------------------------------------

Base URL:    https://us.okx.com (CRITICAL: US domain required)
API Key:     97cb4be8-05b2-4bd0-92a8-bd2067dacf44
Secret:      2A1F8F7D15E605B724B45FD1349AEE78
Passphrase:  TradeBtc2026#Fun
Mode:        Demo (x-simulated-trading: 1 header)

Demo Account Balances (starting):
  USDT: ~$5M (paper money)
  BTC:  ~1.0 (paper money)
  USD:  ~$5K
  USDC: ~$5K

--------------------------------------------------------------------------------
CONFIGURATION (SHAKEDOWN v1 - Paper Trading)
--------------------------------------------------------------------------------

=== STRATEGY ===
SYMBOL            = "BTC/USD"
TIMEFRAME         = "1h"              # Changed from 15m (fewer, cleaner signals)
SMA_PERIOD        = 200
RSI_PERIOD        = 14
RSI_ENTRY_LOW     = 40.0              # Changed from 45 (guardrail, not gate)
RSI_ENTRY_HIGH    = 85.0              # Block extreme FOMO
MAX_SPREAD_PCT    = 0.10%

=== ATR-BASED STOP ===
ATR_PERIOD        = 14
ATR_MULTIPLIER    = 2.0               # Stop = entry - (2 × ATR)

=== RISK MANAGEMENT (PAPER) ===
SIMULATED_EQUITY  = $5,000            # Test with realistic capital, not $5M demo
RISK_PER_TRADE    = 0.5%              # $25 risk per trade
MAX_POSITION_PCT  = 25%               # $1,250 max position
MAX_TRADES_PER_DAY= 1
DAILY_MAX_LOSS    = 1%                # $50 max daily loss

=== CIRCUIT BREAKER ===
MAX_API_ERRORS    = 3
ERROR_WINDOW_MIN  = 5

=== LIVE CANARY CONFIG (after paper passes) ===
PAPER_MODE        = false
RISK_PER_TRADE    = 0.25%             # $12.50 risk per trade
MAX_POSITION_PCT  = 5%                # $250 max position
MAX_API_ERRORS    = 2

--------------------------------------------------------------------------------
STRATEGY LOGIC
--------------------------------------------------------------------------------

ENTRY CONDITIONS (ALL required):
  1. Close > SMA(200)     - Confirms uptrend
  2. RSI > 40             - Avoids weak markets
  3. RSI < 85             - Blocks extreme FOMO
  4. Spread < 0.1%        - Ensures liquidity
  5. Not in position      - Only one position at a time
  6. Trades today < 1     - Max 1 trade per day

EXIT CONDITIONS (ANY triggers):
  1. Close < SMA(200)     - Trend reversal
  2. Price <= stop_price  - ATR-based stop hit
  3. Price >= take_profit - TP hit (if enabled, currently disabled)

POSITION SIZING FORMULA:
  stop_distance = ATR × 2
  stop_price = entry - stop_distance
  risk_amount = equity × risk_pct
  position_btc = risk_amount / stop_distance
  position_usd = min(position_usd, equity × max_position_pct)

EXAMPLE ($5k equity, BTC @ $93k, ATR = $415):
  Stop Distance: $830 (ATR × 2)
  Stop Price:    $92,170
  Risk Amount:   $25 (0.5% of $5k)
  Position:      0.030 BTC ($25 / $830)
  Capped at:     $1,250 (25% of equity) → 0.0134 BTC

--------------------------------------------------------------------------------
GO-LIVE CHECKLIST STATUS
--------------------------------------------------------------------------------

A) SYSTEM RELIABILITY
  [✓] A1 - Boot test 10/10 passed
  [ ] A2 - Restart test (kill mid-run, restart, verify)
  [✓] A3 - State reconciliation passed
  [ ] A4 - Telemetry writes cleanly (monitoring)
  [ ] A5 - No repeated auth errors (monitoring)
  [ ] A6 - One action per candle enforced (monitoring)

B) EXECUTION CORRECTNESS (paper trading)
  [ ] B1 - 2 weeks OR 20 trades (started Jan 6, 2026)
  [ ] B2 - Every trade has entry recorded
  [ ] B3 - Every trade has stop recorded
  [ ] B4 - Every trade has exit + reason
  [ ] B5 - No trades when spread fails
  [ ] B6 - No trades after daily loss limit hit

C) RISK COMPLIANCE (paper trading)
  [ ] C1 - Max drawdown < 10%
  [ ] C2 - No single day > 1% loss
  [ ] C3 - Avg loss ≈ 1R (stop distance)

D) LIVE CANARY READINESS
  [ ] D1 - Rotate exposed API keys (IMPORTANT!)
  [ ] D2 - Live keys = trade only (no withdraw)
  [✓] D3 - FLATTEN script tested
  [ ] D4 - Commit: NO manual overrides for 30 days

--------------------------------------------------------------------------------
COMMANDS
--------------------------------------------------------------------------------

# Start bot
cd /Users/mac/Desktop/okx-trading-bot
python3 -m bot.main

# Check status
ps aux | grep bot.main
tail -20 logs/bot.log
tail -5 logs/telemetry.csv
cat state/state.json

# State reconciliation (run after any restart)
python3 reconcile.py

# Emergency flatten (sells all BTC, disables bot)
python3 flatten.py

# Boot test (10 iterations)
python3 boot_test.py

# Test ATR sizing
python3 test_atr.py

================================================================================
                         BUG FIXES & LESSONS LEARNED
================================================================================

--------------------------------------------------------------------------------
BUG #1: OKX Domain Mismatch (Error 50119)
--------------------------------------------------------------------------------

Date: January 5-6, 2026

SYMPTOM:
  {"msg":"APIKey does not match current environment.","code":"50101"}
  {"msg":"Domain verification failed","code":"50119"}

ROOT CAUSE:
  US OKX accounts MUST use us.okx.com domain.
  CCXT defaults to www.okx.com which fails for US accounts.

FIX:
  Created _override_to_us_domain() in exchange.py that forces:
    exchange.urls['api']['public'] = 'https://us.okx.com'
    exchange.urls['api']['private'] = 'https://us.okx.com'
  
  Applied to both public_exchange (unauthenticated) and exchange (authenticated)

LESSON:
  Always verify exchange domain requirements for different regions.
  US exchanges often have separate domains due to regulations.

--------------------------------------------------------------------------------
BUG #2: Demo Mode Header Not Applied (Error 50101)
--------------------------------------------------------------------------------

Date: January 6, 2026

SYMPTOM:
  API key validated but trades failed with "environment mismatch"

ROOT CAUSE:
  Demo trading requires x-simulated-trading: 1 header.
  CCXT doesn't apply this header to all endpoints consistently.

FIX:
  Created _raw_request() method that manually signs and sends HTTP requests
  with the demo header applied. Used for all private endpoints (balance, orders).
  
  Public endpoints (ticker, OHLCV) use CCXT without authentication.

LESSON:
  Exchange demo modes often have quirks. Test authentication AND trading
  separately. Don't assume library handles everything correctly.

--------------------------------------------------------------------------------
BUG #3: $100k Order Limit in Demo Mode (Error 51201)
--------------------------------------------------------------------------------

Date: January 6, 2026

SYMPTOM:
  "The value of a market order can't exceed 100000USDT"
  Bot tried to buy $1.5M worth of BTC (25% of $5M demo equity)

ROOT CAUSE:
  OKX demo accounts have $100k max order size.
  Position sizing calculated against $5M paper balance was too large.

FIX (v1 - not ideal):
  Added OKX_MAX_ORDER_USD = 95000 cap in risk.py

FIX (v2 - correct approach):
  Added SIMULATED_EQUITY = 5000 in config.
  Bot uses $5k for position sizing even though demo has $5M.
  This tests with REALISTIC sizing that matches live capital.

LESSON:
  Paper trading should mirror live conditions exactly.
  $5M paper money teaches nothing about $5k live trading.
  Always simulate with actual intended capital.

--------------------------------------------------------------------------------
BUG #4: RSI Band Too Tight (Missed Entries)
--------------------------------------------------------------------------------

Date: January 6, 2026

SYMPTOM:
  Good trend entries missed because RSI wasn't in exact 45-70 band.

ROOT CAUSE:
  Original strategy required RSI between 45-70.
  Too restrictive - missed valid trend entries.

FIX:
  Per ChatGPT review, changed to simpler RSI > 40 guardrail.
  Added RSI < 85 to block extreme FOMO.
  Now: RSI must be 40-85 (much wider, still protective)

LESSON:
  Don't over-optimize entry conditions before seeing data.
  Start simple, tighten later based on actual results.

--------------------------------------------------------------------------------
BUG #5: Fixed 1% Stop Too Tight (Random Stop-Outs)
--------------------------------------------------------------------------------

Date: January 6, 2026

SYMPTOM:
  (Anticipated) Fixed 1% stop doesn't adapt to volatility.
  Would cause random stop-outs during normal BTC price swings.

ROOT CAUSE:
  Fixed percentage stops ignore market conditions.
  1% on BTC can be hit in minutes during normal trading.

FIX:
  Implemented ATR-based stops: stop = entry - (2 × ATR(14))
  With current ATR ~$415, stop is ~$830 below entry (~0.9%)
  Adapts automatically as volatility changes.

LESSON:
  ATR-based stops are industry standard for trend systems.
  They adapt to market conditions and reduce noise-based exits.

--------------------------------------------------------------------------------
BUG #6: Boot Test Timeout (False Failures)
--------------------------------------------------------------------------------

Date: January 6, 2026

SYMPTOM:
  Boot test failed 7/10 with "timeout after 5 seconds"
  Bot was actually starting fine, just slowly.

ROOT CAUSE:
  Original boot_test.py used 10-second timeout.
  API calls to OKX US can take 3-5 seconds each.
  5 startup checks × 3 seconds = 15+ seconds total.

FIX:
  Increased timeout to 45 seconds.
  Used process groups (os.setsid) for clean termination.
  Added select() for non-blocking output reading.

LESSON:
  Network-dependent tests need generous timeouts.
  Always test against real APIs, not mocks, for integration tests.

--------------------------------------------------------------------------------
BUG #7: CRITICAL - Stop Loss Never Checked (Position Stuck 3 Days)
--------------------------------------------------------------------------------

Date: January 9-10, 2026

SYMPTOM:
  Bot entered position on Jan 6 @ $93,848.40 with stop @ $93,200.84
  BTC dropped to $90,584.90 by Jan 9 - STOP SHOULD HAVE TRIGGERED
  But position remained open for 3+ days, stop never executed

ROOT CAUSE:
  In bot/main.py run_cycle():
  ```python
  can_trade, reason = self.state_mgr.can_trade()
  if not can_trade:
      return  # <-- EXITS HERE, NEVER CHECKS STOP LOSS!
  ```
  
  When trades_today >= MAX_TRADES_PER_DAY, bot returns immediately
  WITHOUT ever fetching market data or checking stop loss condition.
  Stop loss logic was AFTER the can_trade check, unreachable.

FIX REQUIRED:
  1. Fetch market data BEFORE can_trade check
  2. ALWAYS check stop loss if in_position, regardless of trade limits
  3. Only use can_trade to block NEW entries, not stop management

STATUS: ⚠️ BUG STILL IN CODE - Deployed to Railway with bug
        Need to fix main.py and redeploy

LESSON:
  Stop loss management must be UNCONDITIONAL when in position.
  Trade limits should only gate entries, never exits.

--------------------------------------------------------------------------------
BUG #8: CRITICAL - Day Rollover Never Triggers (State Stuck)
--------------------------------------------------------------------------------

Date: January 9-10, 2026

SYMPTOM:
  state.json showed:
    current_date: "2026-01-06"
    trades_today: 1
  Even though it was January 9th - date never rolled over

ROOT CAUSE:
  State loaded ONCE at startup: self.state = self.state_mgr.state
  _check_day_rollover() exists but is only called during load()
  Main loop never calls load() again, so rollover never triggers
  Bot thought it was still Jan 6 with 1 trade used

FIX REQUIRED:
  Call self.state_mgr.load() at start of each run_cycle()
  This triggers _check_day_rollover() which resets trades_today

STATUS: ⚠️ BUG STILL IN CODE - Need to fix and redeploy

LESSON:
  State that depends on time must be refreshed each cycle.
  Don't cache state objects that have time-based logic.

================================================================================
                           MONITORING SCHEDULE
================================================================================

EVERY FEW HOURS:
  - Verify bot still running: ps aux | grep bot.main

DAILY:
  - Check telemetry: tail -20 logs/telemetry.csv
  - Check logs for errors: grep -i error logs/bot.log | tail -20
  - Verify state: cat state/state.json

AFTER ANY RESTART:
  - Run reconciliation: python3 reconcile.py

WEEKLY:
  - Review trade count and execution quality
  - Check for missed entries (compare signals to actual trades)
  - Verify no duplicate orders or orphan positions

AFTER 2 WEEKS OR 20 TRADES:
  - Complete Go-Live Checklist
  - Review expectancy and drawdown
  - If all pass → prepare live canary

================================================================================
                            EXTERNAL REVIEWS
================================================================================

AI_REVIEW_REQUEST.md was sent to ChatGPT for strategy review.

KEY RECOMMENDATIONS IMPLEMENTED:
  1. ✓ Switch to 1H timeframe (from 15m)
  2. ✓ Use ATR-based stops (from fixed 1%)
  3. ✓ Simplify RSI to guardrail (RSI > 40)
  4. ✓ Reduce max trades to 1/day
  5. ✓ Test with realistic $5k capital

RECOMMENDATIONS FOR LATER:
  - Trailing stops (after baseline established)
  - Volume confirmation (if entries too noisy)
  - Dynamic RSI thresholds (after enough data)
  - Backtesting with backtrader/vectorbt (before strategy changes)

================================================================================
                              PAPER TRADING LOG
================================================================================

Started: January 6, 2026, 01:00 AM MST
Target:  2 weeks OR 20 trades (whichever is longer)

DATE       | ACTION | PRICE    | SIZE      | STOP      | REASON
-----------|--------|----------|-----------|-----------|------------------
2026-01-06 | BUY    | $93,848  | ?         | $93,200   | SMA200 entry (Bug: size unknown)
2026-01-06 | -      | -        | -         | -         | RSI 27 at start, entry later
2026-01-09 | ⚠️ BUG | $90,584  | -         | -         | Stop should have hit, didn't
2026-01-09 | MANUAL | -        | -         | -         | Killed bot, reset state manually
2026-01-10 | DEPLOY | -        | -         | -         | Railway deployment successful
           |        |          |           |           |

NOTES:
- Jan 6: Bot entered position, stop set at $93,200
- Jan 7-9: BTC dropped below stop, but bug prevented exit
- Jan 9: Discovered critical bugs #7 and #8 (stop loss never checked)
- Jan 9: Manually killed bot (PID 10695), reset state.json
- Jan 10: Deployed to Railway, bot running but bugs still present
- CRITICAL: Must fix main.py before real trading resumes

================================================================================
                              REVISION HISTORY
================================================================================

2026-01-10: RAILWAY DEPLOYMENT + CRITICAL BUG DISCOVERY
  - Deployed bot to Railway (Project ID: ce2d349a-47ab-43f9-9c31-fc0d9e726a73)
  - Bot running in cloud, no longer dependent on local Mac
  - Environment variables configured for paper trading
  - Bot stable, checking every 60 seconds
  - Current signal: none (BTC $90,430 < SMA $91,278)
  - ⚠️ CRITICAL BUGS STILL IN CODE - need to fix before live trading

2026-01-09: CRITICAL BUGS DISCOVERED
  - Found bot stuck with position open since Jan 6
  - Entry @ $93,848, Stop @ $93,200, Current @ $90,584
  - Stop should have triggered but NEVER DID
  - ROOT CAUSE #1: Stop loss check after can_trade() return
  - ROOT CAUSE #2: Day rollover never called in main loop
  - Killed broken bot (PID 10695)
  - Manually reset state.json to clear stuck position
  - Documented as Bug #7 and Bug #8

2026-01-06: MAJOR UPDATE
  - Completed 8-phase bot development
  - Fixed OKX US domain issues
  - Fixed demo mode header issues
  - Implemented ATR-based stops
  - Changed to 1H timeframe
  - Changed RSI to 40-85 guardrail
  - Added $5k simulated equity for realistic testing
  - Created flatten.py, reconcile.py, boot_test.py
  - Boot test: 10/10 passed
  - Reconciliation: PASSED
  - Paper trading started

2026-01-05: INITIAL DEVELOPMENT
  - Created project structure
  - Implemented exchange wrapper
  - Hit domain/auth errors (see Bug Fixes)

================================================================================



================================================================================
                         FACEBOOK MCP SERVER - RESOLVED
================================================================================

Date: January 10-11, 2026
Status: ✅ FULLY OPERATIONAL

SERVER LOCATION: /Users/mac/Desktop/facebook-mcp-server/
PAGE: Deb Boler Realty (ID: 690963064110456, 80 fans)
META APP: "claude mcp"

TOOLS AVAILABLE (25 total):
  - Posts: create, get, delete, search, scheduled
  - Photos: upload, get
  - Videos: upload ✅ (FIXED - see below)
  - Comments: get, create, reply, delete, hide
  - Reactions: get, summary
  - Insights: page-level, post-level
  - Messaging: conversations, messages, send
  - Feed, Events, Tagged posts

CRITICAL DISCOVERY - VIDEO UPLOADS:
================================================================================
PROBLEM: "No permission to publish the video" error
WRONG ASSUMPTION: Need publish_video permission (couldn't find in Graph API Explorer)

ACTUAL SOLUTION: Use PAGE ACCESS TOKEN, not User Access Token!
  - publish_video permission is NOT required
  - pages_manage_posts + pages_read_engagement + pages_show_list is enough
  - The key is TOKEN TYPE, not permissions

HOW TO GET PAGE ACCESS TOKEN:
  1. Get User Token with: pages_manage_posts, pages_read_engagement, pages_show_list
  2. Call: GET /me/accounts?fields=access_token,name,id
  3. Response contains Page Access Token for each page you manage
  4. Use THAT token for video uploads

WORKING CURL COMMAND:
  curl -X POST "https://graph-video.facebook.com/v21.0/{PAGE_ID}/videos" \
    -F "source=@/path/to/video.mp4" \
    -F "title=Title Here" \
    -F "description=Description here" \
    -F "access_token={PAGE_ACCESS_TOKEN}"

FIRST SUCCESSFUL UPLOAD:
  - Video ID: 884290197388688
  - URL: https://www.facebook.com/reel/884290197388688/
  - File: /Users/mac/Desktop/video_for_facebook.mp4 (13MB, 2:05, 640x1138)

TOKEN INFO:
  - User tokens expire in ~1 hour (short-lived) or ~60 days (long-lived)
  - Page tokens from long-lived user tokens are PERMANENT (never expire)
  - To get permanent page token: extend user token first, then get page token

PERMISSIONS CURRENTLY GRANTED:
  ✅ pages_show_list
  ✅ business_management  
  ✅ pages_read_engagement
  ✅ pages_read_user_content
  ✅ pages_manage_posts
  ✅ pages_manage_engagement
  ✅ read_insights
  ✅ public_profile

CONFIG LOCATION: ~/Library/Application Support/Claude/claude_desktop_config.json
================================================================================



================================================================================
## BROWSER AUTOMATION ARCHITECTURE - January 11, 2026
================================================================================

CORE DECISION: Accessibility Tree First, Vision Fallback

DEFAULT MODE: Playwright browser_snapshot (Accessibility Tree)
- Returns structured JSON/YAML of all UI elements
- Each element has deterministic ref ID (e.g., ref="e145")
- Click commands use ref IDs - NEVER miss
- 10x faster, 10x cheaper than vision
- ~3,000 tokens vs ~15,000+ vision tokens

FALLBACK MODE: browser_take_screenshot (Vision)
- Only when accessibility tree insufficient
- Use cases: image content, canvas apps, broken accessibility, visual proof

STANDARD WORKFLOW:
1. browser_navigate(url)
2. browser_snapshot()  → Get accessibility tree with refs
3. browser_click(ref="eXXX", element="Button Name")
4. browser_snapshot()  → Verify new state

COST COMPARISON (1000 interactions/day):
- Accessibility: $9/day
- Vision: $150/day
- Savings: 94%

SKILL FILE: /Users/mac/Desktop/claude-skills/SKILL_BROWSER_AUTOMATION.md
APPLIES TO: Ziloss orchestrator, all browser automation agents
================================================================================



================================================================================
================================================================================
##                                                                            ##
##                     9. ZILOSS AI CRM                                       ##
##                                                                            ##
================================================================================
================================================================================

PURPOSE: Full GoHighLevel replacement - AI-native CRM platform
STATUS: Phase 5 of 7 in progress
STARTED: January 4, 2026
TARGET MVP: February 22, 2026

--------------------------------------------------------------------------------
PROJECT LOCATION
--------------------------------------------------------------------------------

Local Path: /Users/mac/Desktop/agent-orchestrator
Build Plan: /Users/mac/Desktop/ZILOSS_CRM_BUILD_PLAN.md (775 lines)
Project Tracker: /Users/mac/Desktop/ZILOSS_CRM_PROJECT.md

--------------------------------------------------------------------------------
PHASE STATUS (Updated January 11, 2026)
--------------------------------------------------------------------------------

| Phase | Status | Description |
|-------|--------|-------------|
| Phase 0: Foundation | ✅ COMPLETE | Docker, PostgreSQL, Redis, FastAPI |
| Phase 1: Data Model | ✅ COMPLETE | 7 CRM tables, full CRUD API |
| Phase 2: Communication | ✅ COMPLETE | Twilio SMS send/receive |
| Phase 3: AI Conversation | ✅ COMPLETE | Claude Haiku integration |
| Phase 4: Event Workflows | ✅ COMPLETE | 68 tests, TCPA compliance |
| Phase 5: NL Control | 🔄 IN PROGRESS | Natural language commands |
| Phase 6: UI | ⏳ PENDING | React dashboard |
| Phase 7: GHL Migration | ⏳ PENDING | Import contacts/data |

TIMELINE: 3-4 weeks ahead of schedule (5 phases in 1 week vs planned 5 weeks)

--------------------------------------------------------------------------------
INFRASTRUCTURE
--------------------------------------------------------------------------------

Docker Services:
  - PostgreSQL: localhost:5432 (postgres:postgres)
  - Redis: localhost:6379
  - Prometheus: localhost:9090
  - Jaeger: localhost:16686

Database: orchestrator (14 schema files)
API: http://localhost:8000

Twilio:
  - Account SID: [TWILIO_SID_REDACTED]
  - Phone: +1 (801) 212-9267

--------------------------------------------------------------------------------
MODULES BUILT
--------------------------------------------------------------------------------

modules/crm_core/          - Contacts, conversations, messages, appointments
modules/twilio_sms/        - SMS send/receive with Twilio
modules/ai_conversation/   - Claude Haiku for auto-responses
modules/compliance/        - TCPA: opt-out, quiet hours, DND enforcement
orchestrator/workers/      - Outbox worker for guaranteed delivery

--------------------------------------------------------------------------------
TEST COVERAGE
--------------------------------------------------------------------------------

Phase 4 Tests: 68 passing
  - test_compliance.py: 28 tests (opt-out, opt-in, DND, workflows)
  - test_outbox.py: 12 tests (enqueue, lifecycle, dead letter)
  - test_quiet_hours.py: 18 tests (timezone, area codes)
  - test_twilio_routing.py: 10 tests (phone normalization)

--------------------------------------------------------------------------------
COMMANDS
--------------------------------------------------------------------------------

# Start services
cd /Users/mac/Desktop/agent-orchestrator
docker-compose up -d

# Start API
source .venv/bin/activate
uvicorn orchestrator.api:app --host 0.0.0.0 --port 8000

# Run tests
PYTHONPATH=. pytest tests/phase4/ -v

# Database access
docker exec -it orchestrator-postgres psql -U postgres -d orchestrator

--------------------------------------------------------------------------------
NEXT STEPS (Phase 5)
--------------------------------------------------------------------------------

Claude Code Task: /Users/mac/Desktop/agent-orchestrator/CLAUDE_CODE_TASK_PHASE5.md

Building:
  - CRM intents (search, create, send, schedule)
  - Intent recognizer (keyword matching + LLM fallback)
  - Command executor (maps intents to CRM operations)
  - API endpoint (/nl/command)

Example commands to support:
  - "Show me leads from today"
  - "Find contacts tagged 'hot lead'"
  - "Text John saying hey there"
  - "Schedule a call with Jane tomorrow at 2pm"

================================================================================


================================================================================
SESSION: January 13, 2026 - Kapture + Stock Sentiment Research
================================================================================

KAPTURE MCP INSTALLATION
------------------------
- Successfully installed Kapture browser automation MCP
- Config: /Users/mac/Library/Application Support/Claude/claude_desktop_config.json
- Key fix: Use system npm (/usr/local/bin/npx) not Claude's bundled npm (v22 vs v25)
- GitHub Issue #2 documented the bundled npm version problem
- Chrome extension from Web Store: Kapture MCP Browser Automation
- Port: 61822, DevTools panel-based

COMPOSIO EVALUATION
-------------------
- 877 toolkits, 11,000+ tools via MCP or direct API
- Pricing: Free (20k calls), $29/mo (200k), $229/mo (2M), Enterprise (custom)
- Twitter toolkit: 72 tools - mostly engagement (post, DM, lists), not bulk research
- Twitter API limits the real bottleneck, not Composio

STOCK SENTIMENT PROJECT INTEREST
--------------------------------
Goal: Track ticker mentions on Twitter/X, identify what's being pushed/neg'd
Use case: Lead prospecting, monitoring mentions, buy signals

Twitter/X API Pricing:
- Free: 1,500 tweets/mo (useless)
- Basic: $100/mo, 10k tweets
- Pro: $5,000/mo, 1M tweets (target tier)
- Enterprise: $42,000+/mo

Alternative Tools Evaluated:
- Unusual Whales: $30-50/mo, options flow + social sentiment
- Quiver Quant: Free tier, WSB mentions, Congress trades
- Stocktwits: Free, dedicated stock social feed
- SwaggyStocks: Free, WSB/Reddit sentiment
- Finnhub API: Free tier, pre-processed sentiment per ticker
- LunarCrush/Santiment: Crypto-focused social metrics

Decision: If Twitter API needed, pay $5k/mo Pro tier - respects platform economics

NEXT: Build stock sentiment monitoring system



================================================================================
SESSION: January 13, 2026 (Evening) - Hedge Fund Research Continuation
================================================================================

MEMORY VERIFICATION
-------------------
- Confirmed all 5 memories from morning session found via mem0:search_memories
- All Desktop files verified: HEDGE_FUND_RESEARCH.md, HEDGE_FUND_HOWTO.md, HEDGE_FUND_HOLDINGS_Q3_2025.md
- MASTER_PROJECT_INFO.md confirmed with Jan 13 entry

COT/TFF DATA LAYER COMPLETED
----------------------------
Pulled CFTC Commitments of Traders / Traders in Financial Futures data
Report Date: January 6, 2026

KEY HEDGE FUND POSITIONING (Leveraged Funds):
- S&P 500: NET SHORT -415,354 contracts (MASSIVELY BEARISH)
- Nasdaq: NET SHORT -14,971 contracts
- VIX: NET SHORT -24,445 contracts (betting on calm)
- Ultra 10Y Treasury: NET SHORT -381,664 contracts
- Ultra Treasury Bond: NET SHORT -588,999 contracts
- Bitcoin: NET SHORT -23,342 contracts (combined standard + micro)
- Euro/USD: NET LONG +50,600 contracts
- Japanese Yen: NET SHORT -63,230 contracts

INTERPRETATION:
Hedge funds are positioned for:
1. Market decline (short S&P, short VIX = premium collection)
2. Rising rates / bond selloff (massively short Treasuries)
3. Stronger dollar (short JPY, CAD)
4. Bitcoin weakness

Files Created:
- /Users/mac/Desktop/COT_HEDGE_FUND_POSITIONING_JAN2026.md

Data Source: https://www.cftc.gov/dea/futures/financial_lf.htm

NEXT STEPS:
1. Build Q/Q change tracking (compare Q2 vs Q3 2025 13F data)
2. Create automated refresh script for COT data
3. Optional: Social sentiment layer (Twitter API $5k/mo)

CONTEXT MANAGEMENT:
- Estimated 30-35% context used at checkpoint
- Saved progress to mem0
- MCP Gateway routes to servers on-demand (no persistent connections)


================================================================================
SESSION: January 13, 2026 (Continued) - AI/Tech Holdings Deep Dive
================================================================================

CONTEXT CHECKPOINT: ~35-40% estimated

DATA SAVED:
- mem0: Q3 2025 hedge fund AI/tech positions
- File: /Users/mac/Desktop/HEDGE_FUND_AI_TECH_Q3_2025.md

KEY FINDINGS FROM LOCAL DATABASE:
Database: /Users/mac/Desktop/13f-database/sec_13f.db
- 120,426,118 total holdings records
- 333,771 submissions
- 43,170 COT TFF records
- Q3 2025 (30-SEP-2025): 9,953 filings

TOP AI/TECH POSITIONS (Q3 2025):
1. CITADEL - NVIDIA: $28.5B (!!!)
2. CITADEL - META: $15.8B
3. CITADEL - ALPHABET: $13.1B
4. CITADEL - MICROSOFT: $12.9B
5. MILLENNIUM - NVIDIA: $10.8B
6. CITADEL - AMAZON: $10.1B
7. CITADEL - PALANTIR: $7.2B
8. CITADEL - BROADCOM: $6.5B

ZILOSS VISION CONFIRMED:
- Local database ready for open-source LLM queries
- 6x RTX 6000 Ada GPUs on order
- Claude orchestrates → Local LLM runs SQL → FREE research

NEXT: Continue querying for Q/Q changes, more detailed breakdowns


---

YOY ANALYSIS COMPLETE:
- Saved: /Users/mac/Desktop/HEDGE_FUND_YOY_CHANGES_Q3_2024_2025.md
- Script: /Users/mac/Desktop/13f-database/yoy_compare.py

HEADLINE: CITADEL CUT NVIDIA BY $26.8B (-49%)
- Total AI/tech derisking: $155B -> $105B (-$50B, -32%)
- PALANTIR is the consensus BUY (+218% Citadel, new positions at Millennium/Two Sigma)
- Millennium/Berkshire/Point72 rotating IN as others exit

Context checkpoint: ~50% estimated


---

COMBINED ANALYSIS COMPLETE:
- File: /Users/mac/Desktop/HEDGE_FUND_COMBINED_ANALYSIS_JAN2026.md

KEY FINDING: Hedge funds are HEDGED - LONG individual AI stocks but SHORT S&P futures (-415K contracts)

SESSION FILES CREATED:
1. HEDGE_FUND_AI_TECH_Q3_2025.md
2. HEDGE_FUND_YOY_CHANGES_Q3_2024_2025.md  
3. PERPLEXITY_AI_RESEARCH_JAN2026.md
4. HEDGE_FUND_COMBINED_ANALYSIS_JAN2026.md
5. yoy_compare.py (reusable script)

Context estimate: ~65%


---

## OKX CRYPTO PAPER TRADING BOT
**Date:** Jan 13, 2026
**Status:** RUNNING ✅

### Location
`/Users/mac/Desktop/okx-trading-bot`

### Configuration
- **Exchange:** OKX
- **Mode:** PAPER (no real money)
- **Symbol:** BTC/USD
- **Strategy:** RSI-based (buy <30, sell >70)
- **Starting Equity:** $5,000 paper

### Commands
```bash
./start_bot.sh    # Start bot
./stop_bot.sh     # Stop bot
./monitor.sh      # Watch signals
tail -f logs/bot.log  # View logs
```

### Key Files
- `state/state.json` - Current position & P&L
- `logs/telemetry.csv` - Trade history
- `bot/strategy.py` - Trading logic
- `bot/config.py` - Settings

### Current Status (Jan 13, 2026 @ 9pm)
- **In Position:** YES
- **Entry:** $91,965.10
- **Current:** $94,390
- **Unrealized P&L:** +$33 (+2.6%)
- **RSI:** 85.77 (overbought - sell signal approaching)

### Notes
- Running since Jan 6, 2026
- Hourly candle checks
- Has stop-loss at $90,942
- Position sizing: ~25% of account per trade

