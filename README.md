# CLAUDE.md — Closira Full-Stack Engineering Intern Assignment

> This file is the single source of truth for Claude Code to build the complete Closira project.
> Read every section carefully before writing any code. Do not skip sections.
> Build **both** the Backend and Frontend assignments in a single repository.

---

## Repository Structure

Create this exact folder layout before writing any code:

```
closira/
├── CLAUDE.md                  ← this file (keep at root)
├── README.md                  ← combined project README (write last)
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py            ← FastAPI app entry point
│   │   ├── database.py        ← SQLAlchemy engine + session
│   │   ├── models.py          ← ORM models
│   │   ├── schemas.py         ← Pydantic request/response schemas
│   │   ├── crud.py            ← DB read/write helpers
│   │   ├── worker.py          ← background task logic (SOP matching)
│   │   ├── logging_config.py  ← structured JSON logging setup
│   │   └── routers/
│   │       ├── __init__.py
│   │       ├── enquiry.py     ← /enquiry routes
│   │       └── health.py      ← /health route
│   ├── tests/
│   │   └── test_api.http      ← HTTP test file for all endpoints
│   ├── requirements.txt
│   ├── .env.example
│   └── README.md              ← backend-specific README
└── frontend/
    ├── src/
    │   ├── navigation/
    │   │   └── AppNavigator.tsx
    │   ├── screens/
    │   │   ├── DashboardScreen.tsx
    │   │   ├── LeadsScreen.tsx
    │   │   ├── EscalationsScreen.tsx
    │   │   ├── FollowUpsScreen.tsx
    │   │   └── ConversationDetailScreen.tsx
    │   ├── components/
    │   │   ├── StatCard.tsx
    │   │   ├── ActivityFeedItem.tsx
    │   │   ├── LeadCard.tsx
    │   │   ├── EscalationCard.tsx
    │   │   ├── FollowUpCard.tsx
    │   │   ├── ChannelBadge.tsx
    │   │   ├── StatusBadge.tsx
    │   │   ├── MessageBubble.tsx
    │   │   ├── TimelineEvent.tsx
    │   │   └── EmptyState.tsx
    │   ├── mock/
    │   │   ├── enquiries.json
    │   │   ├── escalations.json
    │   │   ├── followups.json
    │   │   └── dashboard.json
    │   └── types/
    │       └── index.ts
    ├── App.tsx
    ├── package.json
    ├── tsconfig.json
    └── README.md              ← frontend-specific README
```

---

## PART 1 — BACKEND

### Tech Stack

| Layer | Choice | Reason to state in README |
|---|---|---|
| Framework | FastAPI | Async-native, auto-docs via OpenAPI |
| Database | SQLite (file: `closira.db`) | Zero-config, portable; swap PostgreSQL in prod |
| Background Tasks | FastAPI `BackgroundTasks` | No broker needed for internship scope; Celery adds ops overhead without benefit at this scale |
| ORM | SQLAlchemy 2.x (sync) | Industry standard, easy migration path |
| Logging | `python-json-logger` | Structured JSON logs; machine-parseable |
| Validation | Pydantic v2 | Ships with FastAPI |

### 1.1 Database Schema

Create these tables in `models.py` using SQLAlchemy declarative base:

#### `enquiries` table

| Column | Type | Notes |
|---|---|---|
| `id` | String (UUID4) | Primary key, generated on insert |
| `channel` | String | `whatsapp`, `email`, or `call` |
| `customer_name` | String | Not null |
| `message` | Text | Original inbound message |
| `status` | String | `pending`, `processing`, `sop_matched`, `escalated`, `resolved` |
| `matched_sop` | String | Nullable; set by background task |
| `suggested_response` | Text | Nullable; set by background task |
| `created_at` | DateTime | UTC, auto-set on insert |
| `updated_at` | DateTime | UTC, auto-updated |

#### `events` table  ← status timeline + conversation history

| Column | Type | Notes |
|---|---|---|
| `id` | Integer | Auto-increment PK |
| `enquiry_id` | String | FK → `enquiries.id`, cascade delete |
| `event_type` | String | `enquiry_created`, `sop_matched`, `escalated`, `followup_scheduled`, `response_sent` |
| `description` | Text | Human-readable detail |
| `metadata` | JSON | Arbitrary extra data (SOP name, delay, reason, etc.) |
| `created_at` | DateTime | UTC |

#### `followups` table

| Column | Type | Notes |
|---|---|---|
| `id` | Integer | Auto-increment PK |
| `enquiry_id` | String | FK → `enquiries.id` |
| `delay_minutes` | Integer | Scheduled delay |
| `message_template` | Text | Nullable |
| `scheduled_at` | DateTime | `created_at + delay_minutes` |
| `done` | Boolean | Default False |
| `created_at` | DateTime | UTC |

### 1.2 SOP Matching Logic

Implement in `worker.py`. The background task receives the `enquiry_id` and does:

1. Load the enquiry from DB.
2. Lowercase the message and run keyword matching against these 5 SOPs:

```python
SOPS = [
    {
        "name": "Booking Enquiry",
        "keywords": ["book", "appointment", "schedule", "reserve", "slot", "availability"],
        "response": "Thank you for reaching out! We'd love to help you book an appointment. Please share your preferred date and time, and we'll confirm availability shortly."
    },
    {
        "name": "Pricing Question",
        "keywords": ["price", "cost", "rate", "fee", "charge", "quote", "how much", "pricing"],
        "response": "Great question! Our pricing varies based on your specific requirements. Could you share more details about what you're looking for? We'll send you a tailored quote within 24 hours."
    },
    {
        "name": "Complaint",
        "keywords": ["complaint", "unhappy", "dissatisfied", "issue", "problem", "bad", "terrible", "disappointed", "refund"],
        "response": "We're sorry to hear about your experience. Your concern has been noted and a team member will reach out within 2 hours to resolve this for you."
    },
    {
        "name": "After-Hours Message",
        "keywords": ["tonight", "weekend", "sunday", "saturday", "after hours", "closed", "office hours"],
        "response": "Thanks for your message! Our team is currently unavailable, but we'll get back to you first thing on the next business day."
    },
    {
        "name": "General Information",
        "keywords": ["info", "information", "details", "tell me", "what do you", "how does", "explain"],
        "response": "Thanks for your interest! We'd be happy to share more information. A team member will follow up with detailed information shortly."
    },
]
```

3. Return the FIRST matching SOP (order matters). Update `enquiries.matched_sop`, `enquiries.suggested_response`, `enquiries.status = "sop_matched"`.
4. Log event in `events` table: `event_type="sop_matched"`, metadata includes SOP name.
5. If NO SOP matches: set `status = "escalated"`, log `event_type="escalated"`, metadata `{"reason": "No SOP matched for inbound message"}`.
6. Log every step as structured JSON.

### 1.3 API Endpoints — Full Specification

#### `POST /enquiry`

**Request body:**
```json
{
  "channel": "whatsapp",
  "customer_name": "Priya Sharma",
  "message": "Hi, I'd like to know the pricing for your monthly plan"
}
```

**Validation:**
- `channel` must be one of `["whatsapp", "email", "call"]` — return 422 with clear message otherwise.
- `customer_name` and `message` must be non-empty strings.

**Response (202 Accepted):**
```json
{
  "job_id": "uuid-here",
  "status": "pending",
  "message": "Enquiry received and is being processed."
}
```

**Side effects:**
- Insert row into `enquiries`.
- Insert event `enquiry_created` into `events`.
- Enqueue background task to run SOP matching.
- Log structured event: `{"event": "enquiry_created", "enquiry_id": "...", "channel": "...", "customer_name": "..."}`.

---

#### `POST /enquiry/{id}/followup`

**Request body:**
```json
{
  "delay_minutes": 30,
  "message_template": "Hi {customer_name}, just following up on your enquiry!"
}
```

**Validation:**
- `delay_minutes` must be a positive integer (≥ 1).
- Enquiry must exist; return 404 if not.
- Enquiry must not be in `escalated` or `resolved` status — return 400 with reason.

**Response (200):**
```json
{
  "followup_id": 1,
  "enquiry_id": "uuid",
  "scheduled_at": "2025-05-20T10:44:00Z",
  "message": "Follow-up scheduled successfully."
}
```

**Side effects:** Insert into `followups`, insert event `followup_scheduled` into `events`.

---

#### `POST /enquiry/{id}/escalate`

**Request body:**
```json
{
  "reason": "Customer is requesting a refund and is very upset"
}
```

**Validation:**
- Enquiry must exist; return 404 if not.
- `reason` must be non-empty.
- If already `escalated`, return 400: `{"detail": "Enquiry is already escalated."}`.

**Response (200):**
```json
{
  "enquiry_id": "uuid",
  "status": "escalated",
  "reason": "Customer is requesting a refund and is very upset",
  "message": "Enquiry has been escalated to a human agent."
}
```

**Side effects:** Update `enquiries.status`, insert event `escalated` into `events`, structured log.

---

#### `GET /enquiry/{id}/history`

**Response (200):**
```json
{
  "enquiry": {
    "id": "uuid",
    "channel": "whatsapp",
    "customer_name": "Priya Sharma",
    "message": "Hi, I'd like to know the pricing...",
    "status": "sop_matched",
    "matched_sop": "Pricing Question",
    "suggested_response": "Great question! Our pricing varies...",
    "created_at": "2025-05-20T09:14:00Z",
    "updated_at": "2025-05-20T09:14:05Z"
  },
  "timeline": [
    {
      "event_type": "enquiry_created",
      "description": "Enquiry received via whatsapp",
      "metadata": {},
      "created_at": "2025-05-20T09:14:00Z"
    },
    {
      "event_type": "sop_matched",
      "description": "Matched SOP: Pricing Question",
      "metadata": {"sop_name": "Pricing Question"},
      "created_at": "2025-05-20T09:14:02Z"
    }
  ],
  "followups": []
}
```

Return 404 if enquiry not found.

---

#### `GET /health`

**Response (200):**
```json
{
  "status": "ok",
  "database": "connected",
  "timestamp": "2025-05-20T09:14:00Z"
}
```

If DB is unreachable, return `"database": "unreachable"` but still 200 (health checks shouldn't 500).

---

### 1.4 Structured Logging

In `logging_config.py`, set up `python-json-logger`. Every log entry must include:
- `timestamp` (ISO 8601 UTC)
- `level`
- `event` (a human-readable key like `"enquiry_created"`)
- Relevant IDs and context

Example output:
```json
{"timestamp": "2025-05-20T09:14:00Z", "level": "INFO", "event": "enquiry_created", "enquiry_id": "abc123", "channel": "whatsapp", "customer_name": "Priya Sharma"}
{"timestamp": "2025-05-20T09:14:02Z", "level": "INFO", "event": "sop_matched", "enquiry_id": "abc123", "sop_name": "Pricing Question"}
{"timestamp": "2025-05-20T09:14:10Z", "level": "WARNING", "event": "escalation_triggered", "enquiry_id": "xyz789", "reason": "No SOP matched"}
```

### 1.5 Error Handling Rules

- Never let an unhandled exception propagate to the client. Wrap all routes in try/except.
- 404 → `{"detail": "Enquiry not found."}` 
- 422 → FastAPI default Pydantic validation errors are fine; make sure field descriptions in schemas are clear.
- 400 → Business rule violations (already escalated, followup on closed enquiry, etc.).
- 500 → Only if truly unexpected; log the full traceback server-side, return generic `{"detail": "Internal server error."}`.

### 1.6 FastAPI /docs Requirements

Every endpoint must have:
- `summary` and `description` in the router decorator.
- `response_model` set explicitly.
- At least one `openapi_extra` or Pydantic `Field(description=..., example=...)` so the Swagger UI shows example payloads.

### 1.7 `requirements.txt`

```
fastapi>=0.111.0
uvicorn[standard]>=0.29.0
sqlalchemy>=2.0.0
pydantic>=2.0.0
python-json-logger>=2.0.7
python-dotenv>=1.0.0
httpx>=0.27.0          # for async test client if needed
```

### 1.8 `.env.example`

```
DATABASE_URL=sqlite:///./closira.db
LOG_LEVEL=INFO
```

### 1.9 API Test File (`tests/test_api.http`)

Write a `.http` file (compatible with VS Code REST Client and JetBrains HTTP Client) covering:

1. POST `/enquiry` — valid whatsapp message matching "Pricing Question" SOP
2. POST `/enquiry` — valid email message matching "Booking Enquiry" SOP
3. POST `/enquiry` — valid call message that matches NO SOP (should trigger auto-escalation)
4. POST `/enquiry/{id}/followup` — schedule follow-up on enquiry from #1
5. POST `/enquiry/{id}/escalate` — escalate enquiry from #2
6. GET `/enquiry/{id}/history` — get history for enquiry from #1
7. GET `/health`
8. POST `/enquiry` — invalid channel (expect 422)
9. GET `/enquiry/nonexistent-id/history` — expect 404

Use `@baseUrl = http://localhost:8000` at the top. Use `###` separators between requests. Add comments explaining what each request tests.

### 1.10 Backend `README.md`

Write a clear README with these exact sections:

```markdown
# Closira Backend

## Setup & Run
(step-by-step: clone, venv, pip install, uvicorn command)

## Database Schema & Reasoning
(explain the 3 tables, why SQLite, how to swap to PostgreSQL)

## Celery vs BackgroundTasks Decision
(explain the choice clearly: no broker, intern scope, trade-offs)

## SOP Matching Logic
(briefly explain keyword approach, list the 5 SOPs)

## API Endpoints
(table: method, path, description)

## Logging
(explain JSON structured logging, show sample output)

## Trade-offs & Known Limitations
(be honest: SQLite not prod-ready, no auth, no pagination, SOP matching is naive keyword logic)
```

---

## PART 2 — FRONTEND

### Tech Stack

| Layer | Choice | Reason to state in README |
|---|---|---|
| Framework | React Native (Expo) | Fastest mobile setup, no native toolchain required |
| Navigation | React Navigation v6 (Bottom Tabs + Stack) | Industry standard |
| Styling | NativeWind (Tailwind for RN) | Consistent design tokens, utility-first, no StyleSheet sprawl |
| Language | TypeScript | Type safety, better IDE support |
| Data | Hardcoded JSON in `/mock` | Assignment requirement; structured as if from API |

### 2.1 Type Definitions (`src/types/index.ts`)

```typescript
export type Channel = 'whatsapp' | 'email' | 'call';
export type EnquiryStatus = 'new' | 'qualified' | 'escalated' | 'resolved';
export type UrgencyLevel = 'high' | 'medium' | 'low';

export interface Enquiry {
  id: string;
  customer: string;
  channel: Channel;
  status: EnquiryStatus;
  message: string;
  receivedAt: string; // ISO 8601
  summary?: string;
  matchedSop?: string;
  suggestedResponse?: string;
}

export interface Escalation extends Enquiry {
  reason: string;
  urgency: UrgencyLevel;
  resolved: boolean;
}

export interface FollowUp {
  id: string;
  enquiryId: string;
  customer: string;
  channel: Channel;
  dueAt: string; // ISO 8601
  messagePreview: string;
  done: boolean;
}

export interface TimelineEvent {
  eventType: string;
  description: string;
  createdAt: string;
}

export interface ConversationDetail extends Enquiry {
  timeline: TimelineEvent[];
  followUps: FollowUp[];
}

export interface DashboardStats {
  totalLeadsToday: number;
  missedEnquiries: number;
  openEscalations: number;
  followUpsDue: number;
}
```

### 2.2 Mock Data

Create realistic, varied mock data. All files go in `src/mock/`.

#### `src/mock/dashboard.json`
```json
{
  "stats": {
    "totalLeadsToday": 24,
    "missedEnquiries": 3,
    "openEscalations": 5,
    "followUpsDue": 8
  },
  "activityFeed": [
    {
      "id": "enq_001",
      "customer": "Sarah M.",
      "channel": "whatsapp",
      "status": "escalated",
      "message": "I am very unhappy with the service I received last week.",
      "receivedAt": "2025-05-20T09:14:00Z",
      "summary": "Customer unhappy with quote, requested manager."
    },
    {
      "id": "enq_002",
      "customer": "Raj Patel",
      "channel": "email",
      "status": "qualified",
      "message": "I'd like to book an appointment for next Tuesday if possible.",
      "receivedAt": "2025-05-20T08:52:00Z",
      "summary": "Booking request for next week."
    },
    {
      "id": "enq_003",
      "customer": "Amina K.",
      "channel": "call",
      "status": "new",
      "message": "What are your pricing options for the premium plan?",
      "receivedAt": "2025-05-20T08:30:00Z",
      "summary": "Pricing enquiry for premium tier."
    }
  ]
}
```

#### `src/mock/enquiries.json`
Create at least 8 enquiries covering all channels (whatsapp, email, call) and all statuses (new, qualified, escalated, resolved). Use realistic Indian and international customer names. Vary the messages realistically.

#### `src/mock/escalations.json`
Create at least 5 escalations with varied urgency (high/medium), reasons, and channels. Include a mix of resolved and unresolved.

#### `src/mock/followups.json`
Create at least 6 follow-ups with varied due times (some overdue, some upcoming), channels, and done/not-done states.

### 2.3 Component Specifications

#### `ChannelBadge.tsx`
- Props: `channel: Channel`
- WhatsApp → green background (`#25D366`), white text, "WhatsApp" label
- Email → blue background (`#3B82F6`), white text, "Email" label
- Call → amber background (`#F59E0B`), white text, "Call" label
- Small pill shape, consistent padding

#### `StatusBadge.tsx`
- Props: `status: EnquiryStatus`
- New → blue (`#3B82F6`)
- Qualified → green (`#10B981`)
- Escalated → red (`#EF4444`)
- Resolved → gray (`#6B7280`)
- Pill shape, consistent with ChannelBadge

#### `StatCard.tsx`
- Props: `label: string`, `value: number`, `color?: string`
- Used in Dashboard for the 4 summary stats
- Clean card with large number, smaller label below

#### `LeadCard.tsx`
- Props: `enquiry: Enquiry`, `onPress: () => void`
- Shows: customer name, ChannelBadge, StatusBadge, time received (relative: "2h ago"), message preview (truncated to 1 line)
- Full-width tappable card with subtle border/shadow

#### `EscalationCard.tsx`
- Props: `escalation: Escalation`, `onResolve: (id: string) => void`
- Shows: customer name, ChannelBadge, urgency indicator (🔴 High / 🟡 Medium), reason, time
- Resolve button at bottom right
- High urgency cards have a red left border accent

#### `FollowUpCard.tsx`
- Props: `followUp: FollowUp`, `onMarkDone: (id: string) => void`
- Shows: customer name, ChannelBadge, due time (show "OVERDUE" in red if past due), message preview
- Mark as Done button; when done, card shows strikethrough and grayed out state

#### `ActivityFeedItem.tsx`
- Props: `enquiry: Enquiry`
- Compact row: channel icon dot (colored), customer name, message preview, time
- Tappable, goes to ConversationDetail

#### `MessageBubble.tsx`
- Props: `text: string`, `isAI?: boolean`, `timestamp: string`
- Customer messages: left-aligned, light gray background
- AI/system messages: right-aligned, brand color background
- Shows timestamp below

#### `TimelineEvent.tsx`
- Props: `event: TimelineEvent`
- Vertical timeline dot with connecting line
- Event type label (bold), description text, timestamp

#### `EmptyState.tsx`
- Props: `message: string`, `icon?: string`
- Centered illustration placeholder, message text
- Used in all list screens when data is empty

### 2.4 Screen Specifications

#### `DashboardScreen.tsx`
Layout (top to bottom):
1. Header: "Good morning 👋" + date
2. 2×2 grid of `StatCard` components (total leads, missed, escalations, follow-ups due)
3. Quick action buttons row: "New Enquiry" / "View Escalations" / "Schedule Follow-up" (visual only, navigate to correct tab)
4. Section header: "Recent Activity"
5. `FlatList` of `ActivityFeedItem` (last 10 from dashboard mock)

#### `LeadsScreen.tsx`
Layout:
1. Search bar (filter by customer name, local filter on mock data)
2. Filter chips row: All / WhatsApp / Email / Call (filter by channel)
3. `FlatList` of `LeadCard` sorted by `receivedAt` descending
4. `EmptyState` when no results

Tapping a LeadCard navigates to `ConversationDetail` with the enquiry data.

#### `EscalationsScreen.tsx`
Layout:
1. Active escalations count badge in header
2. Filter chips: All / High / Medium urgency
3. `FlatList` of `EscalationCard`
4. Resolve action updates local state (remove from active list or gray out)
5. `EmptyState` if all resolved

Tapping card (not button) navigates to `ConversationDetail`.

#### `FollowUpsScreen.tsx`
Layout:
1. Two sections: "Overdue" (red section header) and "Upcoming" (normal header)
2. `SectionList` of `FollowUpCard`
3. Mark as Done updates local state
4. `EmptyState` if no follow-ups

#### `ConversationDetailScreen.tsx`
Receives navigation params: `enquiryId` and full enquiry object.

Layout:
1. Header: customer name + back button
2. Info row: ChannelBadge + StatusBadge + received time
3. Section: "AI Summary" — gray card with summary text and matched SOP label (pill badge)
4. Section: "Conversation" — FlatList of `MessageBubble` (original message as customer, suggested response as AI)
5. Section: "Status Timeline" — list of `TimelineEvent` components
6. If escalated: show red banner "Escalated to Human Agent" with reason

### 2.5 Navigation Structure

```typescript
// Bottom Tab Navigator tabs:
// Home (icon: home), Leads (icon: users), Escalations (icon: alert-triangle), Follow-ups (icon: clock)

// Stack Navigator (wrapping tabs or per-tab):
// Each tab has its own stack so ConversationDetail can be pushed from Leads or Escalations
```

Use `@react-navigation/bottom-tabs` and `@react-navigation/stack` (or native-stack).

Pass the full enquiry/escalation object as a navigation param to `ConversationDetailScreen` so no separate data lookup is needed.

### 2.6 Design System

Apply consistently across all screens:

| Token | Value |
|---|---|
| Primary color | `#6366F1` (indigo) |
| Background | `#F9FAFB` |
| Card background | `#FFFFFF` |
| Text primary | `#111827` |
| Text secondary | `#6B7280` |
| Border | `#E5E7EB` |
| Font scale | 12/14/16/18/24 (small/body/subheading/heading/title) |
| Card border radius | 12px |
| Card shadow | `elevation: 2` / iOS shadow |
| Screen horizontal padding | 16px |
| Card padding | 16px |
| Stack spacing | 8px / 12px / 16px |

### 2.7 `package.json` dependencies

```json
{
  "dependencies": {
    "expo": "~51.0.0",
    "react": "18.2.0",
    "react-native": "0.74.0",
    "@react-navigation/native": "^6.1.17",
    "@react-navigation/bottom-tabs": "^6.5.20",
    "@react-navigation/native-stack": "^6.9.26",
    "react-native-safe-area-context": "4.10.1",
    "react-native-screens": "3.31.1",
    "nativewind": "^2.0.11",
    "tailwindcss": "^3.4.0",
    "react-native-vector-icons": "^10.0.3"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "@types/react": "~18.2.0",
    "@types/react-native": "~0.73.0",
    "babel-plugin-module-resolver": "^5.0.0"
  }
}
```

### 2.8 Frontend `README.md`

```markdown
# Closira Frontend

## Setup & Run
(Expo Go instructions: expo start, scan QR)

## Styling Choice: NativeWind
(why: design tokens, Tailwind familiarity, no StyleSheet sprawl, trade-offs: build step, limited RN support for some utilities)

## Screen Descriptions
(table: screen name, what it shows, navigation)

## Mock Data Structure
(explain the /mock folder, API-readiness thinking)

## Known Limitations
(no real backend, local state only, no persistence, no auth)
```

---

## PART 3 — COMBINED README (root `README.md`)

Write a polished root README with these sections:

```markdown
# Closira — Engineering Intern Assignment

## What's Built
Brief description of both assignments and what Closira does.

## Repository Structure
(folder tree)

## Backend — Quick Start
## Frontend — Quick Start
## Architecture Decisions
(SQLite vs PostgreSQL, BackgroundTasks vs Celery, NativeWind vs StyleSheet)

## Trade-offs & Known Limitations
## What I Would Do With More Time
(honest reflection: auth, real DB, Celery, pagination, tests, CI)
```

---

## Build Order

Execute in this exact order to avoid dependency issues:

1. Create full folder structure
2. Write all TypeScript types (`src/types/index.ts`)
3. Write all mock JSON data files (`src/mock/`)
4. Write all React Native components (smallest to largest)
5. Write all screens
6. Write navigation
7. Write `App.tsx`
8. Write `package.json` and `tsconfig.json`
9. Write frontend README
10. Write all SQLAlchemy models (`backend/app/models.py`)
11. Write Pydantic schemas (`backend/app/schemas.py`)
12. Write database setup (`backend/app/database.py`)
13. Write CRUD helpers (`backend/app/crud.py`)
14. Write logging config (`backend/app/logging_config.py`)
15. Write background worker (`backend/app/worker.py`)
16. Write all routers (`backend/app/routers/`)
17. Write `backend/app/main.py` (mount routers, lifespan, startup DB creation)
18. Write `requirements.txt` and `.env.example`
19. Write API test file (`backend/tests/test_api.http`)
20. Write backend README
21. Write combined root README

---

## Quality Checklist

Before finishing, verify every item:

### Backend
- [ ] `uvicorn app.main:app --reload` starts without errors
- [ ] `/docs` loads with all 5 endpoints, descriptions, and example payloads
- [ ] POST `/enquiry` returns 202 immediately (background task runs after)
- [ ] Background task updates DB with matched SOP within ~1 second
- [ ] No-match message triggers `status = "escalated"` automatically
- [ ] All error paths return correct HTTP status codes
- [ ] Structured JSON logs appear in terminal for every key event
- [ ] `/health` returns DB connectivity status
- [ ] 404 on unknown enquiry ID (not 500)
- [ ] 422 on invalid channel value
- [ ] All `.http` test requests have example payloads and comments

### Frontend
- [ ] App loads in Expo Go without errors
- [ ] Bottom tab navigation works (4 tabs)
- [ ] Tapping a LeadCard opens ConversationDetail
- [ ] Tapping an EscalationCard opens ConversationDetail
- [ ] Resolve button on EscalationCard updates UI
- [ ] Mark as Done on FollowUpCard updates UI (strikethrough)
- [ ] Channel badges are correct colors on all screens
- [ ] Status badges are correct colors on all screens
- [ ] Empty states show for empty lists (not blank screens)
- [ ] Overdue follow-ups show "OVERDUE" in red
- [ ] ConversationDetail shows: summary, SOP match, message thread, timeline
- [ ] No TypeScript errors (`tsc --noEmit`)
- [ ] All components are in separate files (no monolithic screens)

---

## Important Notes for Claude Code

- **Do not use placeholder comments like `// TODO: implement this`.** Write complete, working implementations.
- **Do not truncate files.** Every file must be complete.
- **State management in frontend:** Use React `useState` for local state mutations (resolve, mark done, search filter). No Redux or Zustand needed.
- **Backend imports:** Make sure all imports are correct and circular imports are avoided. `main.py` imports routers; routers import crud; crud imports models and database. Worker imports crud and models.
- **Database initialization:** Call `Base.metadata.create_all(bind=engine)` in a FastAPI `lifespan` event (not deprecated `@app.on_event`).
- **Background task timing:** After returning 202, the BackgroundTask runs. In SQLite this is fine. The `GET /enquiry/{id}/history` call may briefly show `status: "pending"` if called immediately — this is expected behavior; note it in the README.
- **NativeWind setup:** Ensure `babel.config.js` includes `nativewind/babel` plugin and `tailwind.config.js` points to `src/**/*.{ts,tsx}`.
- **Expo entry point:** `App.tsx` at root, not inside `src/`.
- **No authentication required** in either assignment. The evaluators know this is an intern project.
- **Clean git commits:** If creating commits, use meaningful messages: `feat(backend): add enquiry router`, `feat(frontend): add LeadCard component`, etc.