# PRD: Reading Quiz Platform

> Version: 1.0 | Author: selmon for bhaisaab | Date: March 19, 2026

---

## 1. Overview

A web app where parents upload book pages / textbook screenshots, AI generates reading comprehension quizzes, children take them in a clean UI, and AI evaluates answers at the end — with progress tracking over time.

**Core insight:** AI is expensive and slow. Use it only where humans can't do the job (extracting passages from images, generating good questions, writing nuanced evaluations). Everything else — storing questions, serving them, tracking time, showing results — is plain database + UI work.

---

## 2. Users

### 2.1 Parent (Admin User)
- Creates account, adds children
- Uploads book screenshots → triggers AI quiz generation
- Reviews generated quiz before publishing (edit/approve flow)
- Views child's results, progress over time
- Sets preferences: syllabus, grade level, language, time limits
- Gets notified when child completes a quiz

### 2.2 Child (Student User)
- Linked to a parent account (no independent signup)
- Sees assigned quizzes in their dashboard
- Takes quizzes one question at a time
- Sees report card after completing a quiz
- Can retake quizzes (each attempt tracked separately)
- No access to answers or expected responses

### 2.3 Access Model
```
Parent (1) ──→ has many ──→ Children (N)
Parent (1) ──→ creates ──→ Quizzes (N)
Child (1)  ──→ takes ──→ Quiz Attempts (N per quiz)
```

A parent can have multiple children. Each child sees only quizzes assigned to them. Parents see all their children's results.

---

## 3. Where AI Lives (and Where It Doesn't)

### AI-Powered (async, queued jobs)
| Step | What AI Does | When It Runs |
|------|-------------|--------------|
| **Quiz Generation** | Extracts passage from screenshot(s), generates age-appropriate questions with expected answers + key points + marking scheme | On image upload, before parent approval |
| **Evaluation** | Reads ALL answers together, grades content, assesses grammar patterns, writes a holistic report card with encouragement | After child submits final answer |
| **Student Profiling** | Over time, identifies weak areas (comprehension vs grammar vs vocabulary), adjusts question difficulty | Background job after N attempts |

### NOT AI (database + application logic)
| Step | How It Works |
|------|-------------|
| **Storing quizzes** | Questions, passages, expected answers → all in DB after AI generates + parent approves |
| **Serving questions** | Read from DB, render in UI, one at a time |
| **Time tracking** | Client-side timer + server timestamps per question |
| **Attempt logging** | Each answer saved to DB as student types/submits |
| **Progress dashboard** | SQL queries on attempts table, rendered in charts |
| **Notifications** | DB triggers / webhooks, no AI needed |

### Why This Split Matters
- **Cost:** AI calls only at quiz creation (1x) and evaluation (1x per attempt). Not per question.
- **Speed:** Questions served from DB in milliseconds. No AI latency during the quiz.
- **Reliability:** If AI is down, existing quizzes still work. Only new quiz creation and grading are blocked.
- **Auditability:** Every question and expected answer is in the DB. Parents can review/edit before the child sees it.

---

## 4. Features

### 4.1 Quiz Creation Flow

```
Parent uploads screenshot(s)
    ↓
AI extracts passage text (vision model)
    ↓
AI generates questions + expected answers + key points + marks
    ↓
Draft saved to DB (status: "draft")
    ↓
Parent reviews in editor UI:
  - Edit passage text (OCR corrections)
  - Edit/add/remove questions
  - Adjust marks, time limits
  - Assign to child(ren)
  - Set syllabus tag + grade level
    ↓
Parent approves → status: "published"
    ↓
Shows up in child's quiz dashboard
```

**AI prompt context for generation:**
- Child's grade level
- Syllabus (CBSE, ICSE, State Board, etc.)
- Parent preferences (focus areas: comprehension, inference, vocabulary)
- Screenshot image(s)

### 4.2 Quiz Taking Flow (The Dual-Window Experience)

**Layout: Split-screen / dual-pane**

```
┌─────────────────────────────┬──────────────────────────────┐
│                             │                              │
│   📖 PASSAGE PANE           │   ✍️ QUESTION PANE            │
│                             │                              │
│   Full passage text         │   Question 3 of 5            │
│   (scrollable, always       │                              │
│   visible while answering)  │   "Why was everyone happy    │
│                             │    even though the team       │
│   Highlighted sections      │    lost the match?"          │
│   (optional: highlight      │                              │
│   relevant paragraph for    │   ┌────────────────────┐     │
│   current question)         │   │ Type your answer   │     │
│                             │   │ here...            │     │
│                             │   │                    │     │
│                             │   └────────────────────┘     │
│                             │                              │
│                             │   ⏱️ 12:34 remaining          │
│                             │                              │
│                             │   [Next Question →]          │
│                             │                              │
└─────────────────────────────┴──────────────────────────────┘
```

**Key behaviors:**
- Passage always visible on left (no flipping back and forth)
- One question at a time on right
- Timer per question (visible but not scary — no red flashing)
- "Next" button moves forward. No going back (prevents overthinking)
- On mobile: stacked layout (passage on top, question below, both scrollable)
- Auto-save answer on every keystroke (no lost work)
- Final "Submit Quiz" button after last question

**What gets saved to DB per question:**
- `question_id`
- `attempt_id`
- `answer_text` (verbatim)
- `started_at` (timestamp)
- `submitted_at` (timestamp)
- `time_taken_seconds` (computed)

### 4.3 Evaluation Flow (AI-Powered)

```
Child submits last answer
    ↓
All answers pulled from DB
    ↓
AI receives: passage + all questions + expected answers + key points + child's answers + child's grade level
    ↓
AI generates:
  - Per-question marks + content feedback
  - Overall grammar assessment (1-2 tips, NOT per question)
  - Encouragement message
  - One specific thing to practice
  - Difficulty rating (was this quiz appropriate for the level?)
    ↓
Evaluation saved to DB
    ↓
Report card shown to child
Parent notified
```

**Evaluation principles (baked into AI prompt):**
- Max 2 grammar tips per quiz (not per question)
- Praise first, correct second
- Age-appropriate language
- Frame corrections as tips, not errors
- Mention what they did well before what to improve
- For younger kids (Class 1-5): emoji, simple language, extra encouragement
- For older kids (Class 6-10): more detailed feedback, vocabulary suggestions

### 4.4 Progress Dashboard (Parent View)

- Score trend chart (line graph over attempts)
- Per-skill breakdown: comprehension, inference, vocabulary, grammar
- Time trend (are they getting faster?)
- Weak areas flagged (AI-generated insights after 5+ attempts)
- Quiz history table with scores, dates, time taken
- Compare across children (if multiple)

### 4.5 Syllabus Support

**Default syllabus: ICSE**

Quizzes are linked to:
- **Board:** ICSE (default), CBSE, State Board, IB, Custom
- **Grade:** Class 1-12
- **Book:** e.g. "New Oxford Modern English 5", "Treasure Trove 8" (the actual textbook)
- **Subject:** English, Hindi, EVS, Science, Social Studies
- **Chapters/Topics:** Array — a quiz can span multiple chapters/topics
  - e.g. `["Chapter 3: The White Wolf", "Comprehension Practice"]`
  - Allows filtering by chapter and tracking progress per chapter

### Book Registry
Books are first-class entities — quizzes link to a book, and books contain chapters:

```
Book: "New Oxford Modern English 5"
Board: ICSE
Grade: Class 5
Subject: English
Chapters:
  - Chapter 1: ...
  - Chapter 2: ...
  - Chapter 3: The White Wolf
```

This enables:
- "Show me all quizzes from Chapter 3"
- "How is Naveer doing on this book overall?"
- AI knows which chapters have been covered vs. pending
- Progress tracking per book/chapter, not just per quiz
- ICSE question style: inference-heavy, opinion-based, text-evidence required

---

## 5. Database Schema

### Core Tables

```sql
-- Users (both parents and children)
CREATE TABLE users (
    id              UUID PRIMARY KEY,
    email           VARCHAR(255) UNIQUE,        -- parents only
    phone           VARCHAR(20),
    name            VARCHAR(255) NOT NULL,
    role            ENUM('parent', 'child') NOT NULL,
    parent_id       UUID REFERENCES users(id),  -- NULL for parents, set for children
    grade_level     INTEGER,                     -- child's current class
    syllabus        VARCHAR(50),                 -- CBSE, ICSE, etc.
    language_pref   VARCHAR(20) DEFAULT 'English',
    preferences     JSONB,                       -- AI-relevant prefs (focus areas, difficulty, etc.)
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Books (textbooks that quizzes are linked to)
CREATE TABLE books (
    id              UUID PRIMARY KEY,
    title           VARCHAR(255) NOT NULL,       -- "New Oxford Modern English 5"
    board           VARCHAR(50) DEFAULT 'ICSE',
    grade_level     INTEGER NOT NULL,
    subject         VARCHAR(50) NOT NULL,
    chapters        JSONB NOT NULL DEFAULT '[]', -- ["Ch 1: Title", "Ch 2: Title", ...]
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Quizzes (the reusable quiz document)
CREATE TABLE quizzes (
    id              UUID PRIMARY KEY,
    created_by      UUID REFERENCES users(id) NOT NULL,  -- parent
    title           VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) UNIQUE,
    passage_text    TEXT NOT NULL,
    source_images   TEXT[],                     -- S3 URLs of uploaded screenshots
    book_id         UUID REFERENCES books(id),  -- linked to a book
    chapters        TEXT[],                     -- array: ["Chapter 3: The White Wolf", "Comprehension"]
    board           VARCHAR(50) DEFAULT 'ICSE',
    grade_level     INTEGER,
    subject         VARCHAR(50),
    time_limit_mins INTEGER DEFAULT 20,         -- per question
    total_marks     INTEGER,
    status          ENUM('draft', 'published', 'archived') DEFAULT 'draft',
    ai_model_used   VARCHAR(100),               -- which model generated it
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Questions (belong to a quiz, served from DB — no AI at read time)
CREATE TABLE questions (
    id              UUID PRIMARY KEY,
    quiz_id         UUID REFERENCES quizzes(id) ON DELETE CASCADE,
    order_num       INTEGER NOT NULL,           -- 1, 2, 3...
    question_text   TEXT NOT NULL,
    expected_answer TEXT NOT NULL,
    key_points      TEXT[],                     -- array of must-mention concepts
    marks           INTEGER NOT NULL,
    question_type   ENUM('comprehension', 'inference', 'vocabulary', 'grammar', 'opinion') DEFAULT 'comprehension',
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Quiz Assignments (which child gets which quiz)
CREATE TABLE assignments (
    id              UUID PRIMARY KEY,
    quiz_id         UUID REFERENCES quizzes(id),
    child_id        UUID REFERENCES users(id),
    assigned_by     UUID REFERENCES users(id),  -- parent
    due_date        TIMESTAMPTZ,
    max_attempts    INTEGER DEFAULT 3,
    assigned_at     TIMESTAMPTZ DEFAULT NOW()
);

-- Attempts (one per quiz-take)
CREATE TABLE attempts (
    id              UUID PRIMARY KEY,
    quiz_id         UUID REFERENCES quizzes(id),
    child_id        UUID REFERENCES users(id),
    attempt_number  INTEGER NOT NULL,           -- 1st, 2nd, 3rd try
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    total_duration  INTEGER,                    -- seconds
    status          ENUM('in_progress', 'completed', 'timeout', 'abandoned') DEFAULT 'in_progress',
    total_marks     INTEGER,                    -- filled after AI evaluation
    total_possible  INTEGER,
    percentage      DECIMAL(5,2),
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Answers (one per question per attempt — the raw student responses)
CREATE TABLE answers (
    id              UUID PRIMARY KEY,
    attempt_id      UUID REFERENCES attempts(id) ON DELETE CASCADE,
    question_id     UUID REFERENCES questions(id),
    answer_text     TEXT,                       -- verbatim student answer
    started_at      TIMESTAMPTZ,               -- when question was shown
    submitted_at    TIMESTAMPTZ,               -- when student hit next
    time_taken_secs INTEGER,                   -- computed
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Evaluations (AI-generated, one per attempt — NOT per question)
CREATE TABLE evaluations (
    id              UUID PRIMARY KEY,
    attempt_id      UUID REFERENCES attempts(id) UNIQUE,
    ai_model_used   VARCHAR(100),
    -- per-question breakdown stored as JSONB array
    question_grades JSONB NOT NULL,
    /*
      [
        {
          "question_id": "uuid",
          "marks_awarded": 8,
          "marks_possible": 10,
          "content_feedback": "Good understanding of...",
          "missed_points": ["key point X"]
        }
      ]
    */
    grammar_summary     TEXT,                   -- 1-2 tips, overall
    content_summary     TEXT,                   -- overall comprehension assessment
    encouragement       TEXT,                   -- positive closing message
    next_steps          TEXT,                   -- one thing to practice
    difficulty_rating   ENUM('too_easy', 'appropriate', 'too_hard'),
    created_at          TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_users_parent ON users(parent_id);
CREATE INDEX idx_books_grade ON books(grade_level);
CREATE INDEX idx_quizzes_created_by ON quizzes(created_by);
CREATE INDEX idx_quizzes_book ON quizzes(book_id);
CREATE INDEX idx_questions_quiz ON questions(quiz_id);
CREATE INDEX idx_assignments_child ON assignments(child_id);
CREATE INDEX idx_attempts_child ON attempts(child_id);
CREATE INDEX idx_attempts_quiz ON attempts(quiz_id);
CREATE INDEX idx_answers_attempt ON answers(attempt_id);
```

### Data Flow Through Tables

```
Parent uploads image
    → quizzes (draft) + questions (AI-generated)
    
Parent approves
    → quizzes (published) + assignments (linked to child)
    
Child starts quiz
    → attempts (in_progress)
    
Child answers each question
    → answers (one row per question, timestamps tracked)
    
Child submits
    → attempts (completed) → AI runs → evaluations (report card)
    
Parent views dashboard
    → JOIN attempts + evaluations + questions + answers
    → Aggregated by child, quiz, date
```

---

## 6. AI Integration Points

### 6.1 Quiz Generation API

**Trigger:** Parent uploads screenshot(s) and hits "Generate Quiz"

**Input:**
```json
{
    "images": ["s3://bucket/screenshot1.jpg"],
    "child_grade": 5,
    "syllabus": "CBSE",
    "subject": "English",
    "num_questions": 4,
    "focus_areas": ["comprehension", "inference"],
    "language": "English"
}
```

**AI does:**
1. Vision model extracts passage text from image(s)
2. Generates N questions with expected answers, key points, marks, question type
3. Tags each question by type (comprehension/inference/vocabulary/etc.)

**Output → saved to DB:**
- `quizzes` row (passage_text, metadata)
- `questions` rows (one per question)
- Status: `draft` (parent must approve)

**Model:** Vision-capable (GPT-4o, Claude, Gemini) — configurable per deployment

### 6.2 Evaluation API

**Trigger:** Child submits last answer (attempt status → completed)

**Input (from DB, not from client):**
```json
{
    "passage": "...",
    "child_grade": 5,
    "child_name": "Naveer",
    "questions": [
        {
            "question": "...",
            "expected_answer": "...",
            "key_points": ["courage", "spirit"],
            "marks": 10,
            "student_answer": "...",
            "time_taken_secs": 340
        }
    ]
}
```

**AI does:**
1. Grades each answer (marks + content feedback)
2. Assesses grammar across ALL answers (not per-question)
3. Writes encouragement + next steps
4. Rates difficulty appropriateness

**Output → saved to DB:**
- `evaluations` row (linked to attempt)
- `attempts` row updated (total_marks, percentage)

**Prompt includes:** Grammar guide rules (max 2 tips, praise first, age-appropriate)

### 6.3 Adaptive Quiz Generation (Uses Past Reports)

**Trigger:** Every new quiz generation for a student who has past attempts

**Input:** Last 3-5 evaluations for the child + new screenshot

**AI does:**
1. Reviews past evaluations — which question types were weak? (factual/inference/synthesis/opinion)
2. Checks recurring grammar patterns
3. Notes score and time trends
4. Generates new quiz using 70-20-10 rule:
   - 70% comfort zone (question types they handle well)
   - 20% stretch zone (target their weak types)
   - 10% challenge (aspirational, OK to fail)
5. Tags each question by type for future tracking

**Output:** Quiz draft with adapted difficulty, parent can review before publishing

**Research backing:** Testing effect (active recall strengthens retention), spaced repetition on weak areas, tiered questioning (Right There → Think and Search → Author and You → On Your Own)

### 6.4 Student Insights (Background, After 5+ Attempts)

**Trigger:** Cron or on-demand by parent

**Input:** Last N evaluations for a child

**AI does:**
1. Identifies weak areas (comprehension vs grammar vs vocabulary)
2. Tracks question type performance over time
3. Suggests difficulty adjustment
4. Recommends focus areas for next quiz
5. Flags red flags (dropping scores, rushing, same grammar mistakes 3+ times)

**Output:** Updated `users.preferences` JSONB for the child

---

## 7. Tech Stack

See Section 14 for final tech stack decisions.

---

## 8. Pages / Screens

### Pre-Auth
1. **Landing** — Duolingo-style playful landing, "Padhle beta!" tagline
2. **Login** — Phone OTP (Indian parents), email fallback

### Profile Picker (Netflix-style)
3. **Who's Reading?** — Avatar grid. Parent profile + child profiles. Tap to enter. "Add Child" button. Colorful avatars, animal mascots.

### Parent View
4. **Dashboard** — children cards, recent activity, quick stats, streak counter
5. **Create Quiz** — upload screenshots, preview AI-generated quiz, edit, publish, link to book/chapter
6. **Quiz Library** — all quizzes (filter by book, chapter, grade)
7. **Book Library** — registered books with chapters, coverage progress
8. **Child Profile** — Duolingo-style progress (XP, streaks, level), attempt history, weak areas chart
9. **Results Detail** — specific attempt report card, per-question breakdown

### Child View (Duolingo-style)
10. **Home** — fun dashboard with mascot, streak counter, XP, "Today's Quiz" CTA
11. **My Quizzes** — card grid (new ✨, in-progress 📝, completed ✅), sorted by due date
12. **Take Quiz** — dual-pane experience (passage left, question right)
13. **Report Card** — post-quiz celebration screen (confetti if good score), then detailed breakdown
14. **My Progress** — XP graph, streak calendar, badges earned, level progress bar

---

## 9. Mobile Considerations

- Dual-pane becomes stacked on mobile (passage top, question bottom)
- Passage collapses to a "tap to expand" accordion on small screens
- Timer stays sticky at bottom
- Large touch targets for young kids
- Consider PWA for app-like experience without app store

---

## 10. Future Scope (v2+)

- **Pre-built quiz banks** per CBSE/ICSE chapter (community or curated)
- **Audio passage** — AI reads the passage aloud (for younger kids)
- **Handwriting input** — camera captures handwritten answers
- **Multiplayer mode** — siblings or classmates take same quiz, leaderboard
- **Teacher role** — assign quizzes to a whole class
- **WhatsApp/Telegram bot** — take quizzes via chat (current skill behavior, productized)
- **Adaptive difficulty** — AI adjusts question complexity based on past performance
- **Parent co-pilot** — AI suggests "Naveer struggles with inference questions, try these exercises"
- **Offline mode** — download quiz for areas with poor connectivity
- **Multi-language** — Hindi, regional languages (passage + questions)

---

## 11. MVP Scope

Ship the smallest thing that's useful:

1. Parent signup + add one child
2. Upload screenshot → AI generates quiz (draft)
3. Parent reviews + publishes
4. Child takes quiz (dual-pane UI)
5. AI evaluates → report card
6. Parent sees results

**Cut from MVP:** Multiple syllabus filtering, student insights, progress charts, adaptive difficulty, mobile PWA

**Timeline estimate:** 3-4 weeks for a solo dev, 2 weeks with a pair

---

## 12. Decisions (Resolved)

1. **Auth:** Netflix-style profile picker. Parent logs in (email/phone), sees child profiles on home screen. Child taps their avatar to enter. No PIN, no separate login. Multiple parents supported.
2. **Pricing:** Per-child pricing. Not implementing in MVP — focus on product first.
3. **AI cost management:** Limit quiz generation. Support two quiz types:
   - **Personalized quizzes** — AI-generated per student, uses past reports for adaptation
   - **Generic/topic quizzes** — generated once, reusable across multiple students (cheaper, scalable)
4. **Content moderation:** Trust parents for MVP. Future: we upload and index standard ICSE textbooks ourselves (OCR'd, chapter-indexed, ready for quiz generation).
5. **Data/Privacy:** India-focused. Comply with India's DPDP Act 2023 (Digital Personal Data Protection):
   - Parental consent required for children under 18
   - Data minimization — only collect what's needed
   - Parent can request data deletion
   - No behavioral tracking/advertising on children's data
   - Data stored in India-region servers

## 13. Branding

**Name: Padhle** (पढ़ले)
- Means "go read/study" in Hindi
- Every Indian kid has heard "padhle beta" from their parents
- Memorable, fun, culturally resonant
- Works as: padhle.app, padhle.in

## 14. Tech Stack (Final)

| Layer | Tech |
|-------|------|
| Framework | Next.js 14 (App Router, TypeScript) |
| Styling | Tailwind CSS |
| UI | Duolingo-inspired, playful, colorful |
| State | Zustand |
| ORM | Prisma |
| Database | PostgreSQL (Supabase) |
| Auth | NextAuth.js (phone OTP for Indian parents) |
| AI | OpenAI / Anthropic (vision + text) |
| File Storage | Supabase Storage |
| Hosting | Vercel (via GitHub repo) |
| Language | TypeScript (strict) |
