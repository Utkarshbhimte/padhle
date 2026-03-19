# 🦜 Padhle (पढ़ले)

**"Padhle beta!"** — Every Indian parent, ever.

An AI-powered reading comprehension platform that turns textbook pages into interactive quizzes. Upload a screenshot, get a quiz. That simple.

---

## The Problem

Parents want to help their kids practice reading comprehension but:
- Making quizzes by hand is tedious
- Evaluating answers fairly takes effort  
- Tracking progress over time? Nobody does it
- Kids find worksheets boring

## The Solution

**Padhle** uses AI at two key moments — generating quizzes from textbook screenshots, and evaluating answers with gentle, age-appropriate feedback. Everything in between is fast, fun, and powered by a database (not AI).

### How It Works

```
📸 Parent uploads textbook screenshot
     ↓
🤖 AI extracts passage + generates questions
     ↓
✏️ Parent reviews & publishes quiz
     ↓
🎮 Child takes quiz (Duolingo-style UI)
     ↓
🤖 AI evaluates all answers together
     ↓
🎉 Report card with score + gentle feedback
```

## Features

- **📸 Screenshot to Quiz** — Upload a textbook page, AI does the rest
- **🎮 Dual-Pane Quiz** — Passage always visible, questions one at a time
- **🧠 Adaptive Difficulty** — Uses past results to target weak areas (70-20-10 rule)
- **📊 Progress Tracking** — XP, streaks, badges, per-chapter breakdown
- **👨‍👩‍👧‍👦 Netflix-Style Profiles** — Multiple kids on one parent account
- **📝 Gentle Grammar Feedback** — Max 2 tips per quiz, praise first
- **📚 Book & Chapter Linked** — Track progress per textbook
- **🇮🇳 Built for ICSE** — Indian syllabus first, more boards coming

## Design Philosophy

Inspired by **Duolingo** — learning should feel like a game, not a chore.

- Bright, playful UI with an Indian parrot mascot 🦜
- Large touch targets for little hands
- Timer that doesn't stress kids out
- Confetti on good scores
- No going back on questions (prevents overthinking)

## Tech Stack

| Layer | Tech |
|-------|------|
| Framework | Next.js 14 (App Router) |
| Language | TypeScript (strict) |
| Styling | Tailwind CSS |
| State | Zustand |
| ORM | Prisma |
| Database | PostgreSQL |
| Auth | NextAuth.js (Phone OTP) |
| AI | OpenAI / Anthropic (vision + text) |
| Hosting | Vercel |

## Getting Started

```bash
# Clone
git clone https://github.com/Utkarshbhimte/padhle.git
cd padhle

# Install
npm install

# Setup database
cp .env.example .env.local
# Add your DATABASE_URL and AI API keys
npx prisma db push

# Run
npm run dev
```

Open [http://localhost:3000](http://localhost:3000)

## AI Usage (Intentionally Minimal)

| When | What AI Does | Cost |
|------|-------------|------|
| Quiz creation | Extract passage from image + generate questions | ~$0.05 |
| Evaluation | Grade all answers + write report card | ~$0.03 |
| Student insights | Identify weak areas after 5+ attempts | ~$0.02 |

**During the quiz itself?** Zero AI. Questions are served from the database. Fast, cheap, reliable.

## Pedagogy

Based on research from Reading Rockets, r/ELATeachers, and testing effect studies:

- **4 question types**: Right There → Think and Search → Author and You → On Your Own
- **70-20-10 rule**: 70% comfort, 20% stretch, 10% challenge
- **Testing effect**: Active recall > passive re-reading
- **Grammar feedback**: Max 2 tips, age-appropriate, praise-first

## License

MIT

---

*Built with ❤️ for every kid who's heard "padhle beta" one too many times.*
