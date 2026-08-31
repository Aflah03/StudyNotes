# DemoDrop — Project Explanation

> An interactive drag-and-drop assessment platform that evaluates candidates' decision-making, prioritization, and problem-solving skills through six types of interactive activities — all secured by a server-authoritative architecture where the browser never sees the correct answers.

---

## What Is DemoDrop?

DemoDrop is a **full-stack assessment platform** where candidates take interactive tests by dragging and dropping items — categorizing, matching, sequencing, ranking, building process flows, and filling in blanks. Admins can author tests, curate a reusable question bank, and review candidate performance through analytics dashboards.

Think of it like Google Forms, but instead of multiple-choice questions, every question is a hands-on drag-and-drop puzzle — and all grading happens securely on the server.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend Framework** | React 18 |
| **Build Tool** | Vite 5 |
| **Styling** | Tailwind CSS v4 |
| **Drag & Drop** | @dnd-kit (core + sortable) |
| **Animations** | Framer Motion |
| **Data Visualisation** | Recharts |
| **Routing** | React Router DOM v6 |
| **Backend Framework** | Spring Boot 4.1 (Java 21) |
| **ORM** | Spring Data JPA / Hibernate |
| **Database** | PostgreSQL (prod) · H2 (tests) |
| **Migrations** | Flyway |
| **Testing** | JUnit 5 + MockMvc integration tests |

---

## Core Architecture — The "Dumb Terminal" Pattern

This is the single most important design decision in the project and a great talking point in interviews.

```
┌──────────────────────────────────────────────────────┐
│                   REACT FRONTEND                     │
│                                                      │
│  • Renders questions (items, zones, blanks)           │
│  • Captures drag interactions & telemetry             │
│  • Sends raw placements to server                    │
│  • NEVER receives answer keys or scoring logic        │
│                                                      │
│            ▼  HTTP REST (JSON)  ▼                     │
├──────────────────────────────────────────────────────┤
│                 SPRING BOOT BACKEND                  │
│                                                      │
│  • Owns all answer keys (stored in DB, never sent    │
│    to candidate endpoints)                           │
│  • Computes scores, accuracy, and penalties           │
│  • Replays event streams to detect incorrect moves    │
│  • Autosaves drafts for mid-test resume               │
│  • Applies deterministic per-attempt item shuffling   │
│                                                      │
│            ▼  JPA / Flyway  ▼                         │
├──────────────────────────────────────────────────────┤
│                    POSTGRESQL                        │
│                                                      │
│  assessments, activities, attempts,                  │
│  activity_drafts, activity_submissions,              │
│  bank_questions                                      │
└──────────────────────────────────────────────────────┘
```

**Why this matters:**
- A candidate cannot open DevTools and find the correct answers — they simply aren't there.
- All scoring runs server-side, so results can't be tampered with.
- The browser is just an input collector, like a "dumb terminal."

---

## The Six Activity Types

Each activity type uses `@dnd-kit` differently and has its own scoring branch on the backend:

| Type | What the Candidate Does | Example |
|------|------------------------|---------|
| **Categorize** | Drag items from an unsorted tray into labelled buckets | *Sort these tasks into "Urgent" vs "Can Wait"* |
| **Match** | Pair items in column A with slots in column B (1-to-1) | *Match each design pattern to its description* |
| **Sequence** | Arrange steps in chronological order | *Order the steps to deploy a web application* |
| **Rank** | Prioritise items from most to least important | *Rank these project risks by severity* |
| **Flow Builder** | Build a process pipeline between Start and End caps | *Arrange the incident response steps into a flowchart* |
| **Fill-in-the-Blank** | Drag words into inline blanks within a sentence | *Complete: "The ___ pattern separates ___ from ___"* |

---

## How a Test Attempt Works (End-to-End Flow)

This is the core user journey and the most complex data flow in the app:

```
Candidate clicks "Start Test"
        │
        ▼
POST /api/attempts  ──────────────►  Backend creates an Attempt row
        │                              with status = "in_progress"
        ▼
GET /api/attempts/{id}/assessment ──►  Backend loads activities,
        │                              STRIPS answer keys,
        │                              SHUFFLES items deterministically
        │                              (seeded by attemptId:activityId)
        ▼
Candidate drags items around
        │
        ├── Every move ──► useAssessmentTelemetry records
        │                   timestamp, drag count, event log
        │
        ├── 700ms debounce ──► PUT /api/attempts/{id}/activities/{id}/autosave
        │                       Backend upserts into activity_draft table
        │
        ├── "Mark for Review" ──► Saved in draft alongside answers
        │
        ▼
Candidate clicks "Submit Test"
        │
        ▼
POST /api/attempts/{id}/complete
        │
        ▼
Backend for each activity:
  1. Loads draft (or submitted answer)
  2. Loads secret answer key from activity table
  3. ScoringService evaluates:
     ├── Mapping types → set intersection (correct placements)
     ├── Ordering types → positional comparison
     ├── Replays event log → counts incorrect placements
     └── Applies capped drag-efficiency penalty
  4. Saves ActivitySubmission with breakdown
  5. Aggregates total score & accuracy
  6. Generates feedback band ("Excellent" / "Good" / "Needs Improvement")
        │
        ▼
Returns AssessmentResultDto to frontend
        │
        ▼
ResultsPage renders:
  • Score badges & accuracy metrics
  • Recharts bar/line charts (time, attempts, accuracy per activity)
  • Answer review table (your answer vs correct answer)
```

---

## Key Features Worth Highlighting

### 1. Real-Time Autosave & Mid-Test Resume
Every candidate interaction is debounce-saved (700ms) to an `activity_draft` table. If the browser tab is closed or refreshed, the candidate can resume exactly where they left off — their placements, review marks, and progress are all restored from the server.

### 2. Anti-Cheat Item Shuffling
Items are shuffled using a **deterministic Fisher–Yates algorithm** seeded with `attemptId:activityId`. This means:
- Every candidate sees a different item order (prevents copying).
- Refreshing the page gives the **same** shuffle (no database storage needed).

### 3. Behavioural Telemetry & Event Replay
The frontend tracks every drag event (`place`, `move`, `remove`, `reorder`) with timestamps. On submission, the backend **replays** this event stream against the secret answer key to count how many incorrect placements the candidate made — even ones they later corrected. Excessive dragging incurs a capped penalty.

### 4. Non-Linear Question Navigation
A visual question palette lets candidates jump to any question, see which are answered/unanswered/marked-for-review, and get a warning modal if they try to submit with unanswered questions.

### 5. Admin Authoring with Lock-on-Publish
Admins can create tests in "draft" mode, add/edit activities with a visual editor, and publish when ready. Published tests are **locked** — they cannot be edited until unpublished. This prevents mid-test content changes.

### 6. Question Bank & Auto-Generation
A pool of 212 reusable questions (tagged by category and type) lets admins generate new tests instantly — either balanced across activity types or by total count.

---

## Database Schema

Seven Flyway migrations build up this schema:

```
┌──────────────┐       ┌──────────────────┐
│  assessment   │──────<│     activity      │
│              │  1:N   │                  │
│  id (PK)     │       │  id (PK)         │
│  title       │       │  assessment_id(FK)│
│  description │       │  type            │
│  status      │       │  prompt          │
│  duration_min│       │  items (JSON)    │
│              │       │  zones (JSON)    │
│              │       │  answer_key(JSON)│  ◄── NEVER sent to candidates
│              │       │  max_score       │
│              │       │  position        │
│              │       │  suffix          │
└──────────────┘       └──────────────────┘
       │
       │ 1:N
       ▼
┌──────────────┐       ┌────────────────────┐
│   attempt     │──────<│ activity_submission │
│              │  1:N   │                    │
│  id (PK)     │       │  id (PK)           │
│  assessment_id│       │  attempt_id (FK)   │
│  candidate_*  │       │  activity_id (FK)  │
│  status      │       │  answer (JSON)     │
│  started_at  │       │  score, correct_cnt│
│  total_score │       │  incorrect_placements│
│  accuracy    │       │  UNIQUE(attempt,act)│
│  feedback    │       └────────────────────┘
└──────────────┘
       │
       │ 1:N
       ▼
┌──────────────────┐       ┌──────────────────┐
│  activity_draft   │       │  bank_question    │
│                  │       │  (standalone)     │
│  id (PK)         │       │                  │
│  attempt_id (FK) │       │  id (PK)         │
│  activity_id(FK) │       │  type            │
│  answer (JSON)   │       │  category        │
│  marked_review   │       │  prompt, items,  │
│  UNIQUE(att,act) │       │  zones, answer_key│
└──────────────────┘       └──────────────────┘
```

---

## API Design

The API is split into two groups, each with its own security boundary:

### Candidate APIs (`/api/**`) — No auth header required
| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/assessments` | List published tests |
| GET | `/api/assessments/{id}` | Test details (no answer keys) |
| POST | `/api/attempts` | Start a new attempt |
| GET | `/api/attempts/{id}` | Get attempt state (for resume) |
| GET | `/api/attempts/{id}/assessment` | Get shuffled activities |
| PUT | `/api/attempts/{id}/activities/{id}/autosave` | Save draft |
| POST | `/api/attempts/{id}/activities/{id}/submit` | Submit & grade one activity |
| POST | `/api/attempts/{id}/complete` | Finish test, grade everything |
| GET | `/api/attempts/{id}/result` | Get scores |
| GET | `/api/attempts/{id}/review` | Get answer comparison |
| GET | `/api/attempts/history?candidateEmail=...` | Past attempts |

### Admin APIs (`/api/admin/**`) — Requires `X-Role: admin` header
| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET/POST | `/api/admin/assessments` | List all / Create new test |
| PUT/DELETE | `/api/admin/assessments/{id}` | Edit / Delete test |
| POST | `/api/admin/assessments/{id}/publish` | Validate & publish |
| POST | `/api/admin/assessments/{id}/unpublish` | Revert to draft |
| CRUD | `/api/admin/bank/questions/**` | Question bank management |
| POST | `/api/admin/bank/generate` | Auto-generate test from bank |
| GET/DELETE | `/api/admin/attempts/**` | Review / delete attempts |

The admin header check is implemented via a Spring `HandlerInterceptor` — it's intentionally a placeholder for a future JWT/Spring Security integration.

---

## Frontend Architecture

```
main.jsx
  └── BrowserRouter
        └── AuthProvider (React Context — stores name, email, role in localStorage)
              └── AttemptProvider (useReducer — tracks active attempt, current question, results)
                    └── App.jsx (routes + animated page transitions via Framer Motion)
                          ├── LoginPage
                          ├── DashboardPage (candidate)
                          ├── AssessmentRunnerPage (test execution engine)
                          │     ├── AssessmentShell (header, stopwatch, palette, nav)
                          │     ├── ActivityRenderer (dispatches to the correct activity component)
                          │     │     ├── CategorizeActivity
                          │     │     ├── MatchActivity
                          │     │     ├── SequenceActivity
                          │     │     ├── RankActivity
                          │     │     ├── FlowBuilderActivity
                          │     │     └── FillBlankActivity
                          │     └── useAssessmentTelemetry (hook for tracking events)
                          ├── ResultsPage (scores + charts + review)
                          ├── HistoryPage
                          └── Admin Area (guarded by RequireAdmin)
                                ├── AdminDashboardPage
                                ├── AssessmentEditorPage
                                ├── QuestionBankPage
                                └── AdminAttemptsPage
```

**State management** uses React Context + `useReducer` (no Redux). Two API client modules (`client.js` for candidates, `adminClient.js` for admins) handle all HTTP communication.

---

## Scoring Algorithm (Backend)

The `ScoringService` is the brain of the application. Here's how it works:

```
For each activity in the attempt:
  │
  ├── Load the candidate's answer (from submission or draft)
  ├── Load the secret answer_key from the activity table
  │
  ├── IF type is mapping-based (categorize, match, fill-blank):
  │     Count correct placements via set intersection
  │     Score = (correct / total) × maxScore
  │
  ├── IF type is ordering-based (sequence, rank, flow):
  │     Compare candidate order vs correct order position-by-position
  │     Score = (matching positions / total) × maxScore
  │
  ├── Replay the event log:
  │     Walk through [place, move, remove, reorder] events
  │     Count placements that don't match the key → incorrectPlacements
  │
  ├── Apply drag-efficiency penalty:
  │     If dragCount > (2 × itemCount):
  │       penalty = min(0.1 × maxScore, excess × 0.5)
  │       score = max(0, score − penalty)
  │
  └── Save ActivitySubmission with score, correctCount, incorrectPlacements

Aggregate across all activities:
  totalScore = sum of activity scores
  accuracy = (totalCorrect / totalPossible) × 100
  feedback = band(accuracy)  →  "Excellent" / "Good" / "Needs Improvement"
```

---

## How to Talk About This in an Interview

### "Tell me about this project"
> "DemoDrop is a full-stack assessment platform I built with React and Spring Boot. Instead of traditional multiple-choice tests, candidates interact with drag-and-drop puzzles — categorizing, matching, sequencing, ranking, building flows, and filling in blanks. The key architectural decision was a server-authoritative 'dumb terminal' pattern: the browser never receives answer keys, all scoring happens on the backend, and every interaction is autosaved for mid-test resume."

### "What was the most challenging part?"
> "Implementing the real-time autosave with draft rehydration. Every drag interaction is debounce-saved to the server, and if the candidate refreshes or closes the tab, they resume exactly where they left off — including their item placements, review marks, and progress. The tricky part was ensuring the deterministic item shuffle stayed consistent across page reloads without needing to store the shuffled order in the database."

### "How do you prevent cheating?"
> "Three mechanisms: First, answer keys never leave the server — the candidate endpoint strips them before responding. Second, items are shuffled with a deterministic seed per attempt, so every candidate sees a different order. Third, the backend replays the full event stream to detect and penalise excessive incorrect placements, even ones the candidate later corrected."

### "Why this tech stack?"
> "React with @dnd-kit gives excellent accessible drag-and-drop (keyboard navigation, screen reader announcements). Spring Boot with JPA provides a clean layered architecture with compile-time type safety. Flyway ensures database migrations are versioned and repeatable. The H2 test profile lets integration tests run without a real PostgreSQL instance."

---

## Running the Project

```bash
# Backend (requires Java 21 + PostgreSQL)
cd backend
./gradlew bootRun

# Frontend (requires Node.js)
npm install
npm run dev
```

Environment variables (see `.env.example`):
```
VITE_API_BASE_URL=http://localhost:8080/api
```

---

## Test Suite

The backend has a comprehensive automated test suite:

| Test Class | Type | What It Covers |
|------------|------|---------------|
| `ScoringServiceTest` | Unit | Scoring calculations, event replay, drag penalties, feedback bands |
| `ApiIntegrationTest` | Integration (H2) | Assessment retrieval, item shuffling, autosave, submissions, key secrecy |
| `AdminApiIntegrationTest` | Integration (H2) | Admin auth guards, publish validation, lock-on-publish conflicts |
| `AdminBankApiIntegrationTest` | Integration (H2) | Question bank CRUD, balanced test generation |
| `HistoryReviewApiIntegrationTest` | Integration (H2) | Candidate history scoping, post-completion answer reviews |

Run with:
```bash
cd backend
./gradlew test
```
