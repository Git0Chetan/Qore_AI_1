# Qore AI — Local Setup Guide

AI-powered **Hiring** + **Learning & Career Buddy** platform. This guide walks you
through running the entire system on your own machine after cloning the repo.

> Already know the app? See [ARCHITECTURE.md](ARCHITECTURE.md) for the design and
> [DEPLOY.md](DEPLOY.md) for hosting it on Railway.

---

## What you'll be running

The app is a monorepo of **4 services** that run together:

| Service | Folder | Port (local) | What it does |
|---|---|---|---|
| **platform** | `platform/` | `3002` | Main app — auth, jobs, ATS, dashboards, learning, APIs |
| **quiz-frontend** | `quiz-frontend/` | `3001` | AI-proctored assessment (MCQ + coding) |
| **agent-ui** | `agent-starter-react/` | `3000` | LiveKit voice/video room UI (AI interview & Career Buddy) |
| **voice-agent** | `voice-agent/` | — | Python LiveKit worker (proctor / interviewer / career buddy) |

These talk to managed cloud services you'll need accounts for: **Supabase**,
**LiveKit Cloud**, **Groq**, and optionally **Anam** (avatar) and **YouTube Data API**.

---

## 1. Prerequisites

Install these first:

| Tool | Version | Install |
|---|---|---|
| **Node.js** | 20+ (tested on 25) | https://nodejs.org or `nvm install 20` |
| **pnpm** | 9+ | `npm install -g pnpm` |
| **Python** | 3.14+ | https://www.python.org/downloads/ |
| **uv** (Python pkg manager) | latest | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| **Git** | any | https://git-scm.com |

Verify:

```bash
node -v && pnpm -v && python3 --version && uv --version
```

> **Python version note:** `voice-agent/pyproject.toml` requires Python `>=3.14`.
> If you're on 3.13, either install 3.14, or open `voice-agent/pyproject.toml` and
> change `requires-python = ">=3.14"` to `">=3.13"`.

---

## 2. Clone the repo

```bash
git clone <your-repo-url> Qore_AI
cd Qore_AI
```

---

## 3. Create accounts & get API keys

You need keys from these providers (all have free tiers):

1. **Supabase** — https://supabase.com → create a project. From
   **Project Settings → API**, copy:
   - Project URL → `NEXT_PUBLIC_SUPABASE_URL`
   - `anon` `public` key → `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `service_role` key → `SUPABASE_SERVICE_ROLE_KEY` *(secret — server only)*
2. **LiveKit Cloud** — https://cloud.livekit.io → create a project →
   **Settings → Keys**: `LIVEKIT_URL` (`wss://...`), `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`.
3. **Groq** — https://console.groq.com → **API Keys** → create one → `GROQ_API_KEY`.
4. **Anam** *(optional, for the interviewer avatar video)* — https://anam.ai → `ANAM_API_KEY`.
   Without it, calls run voice-only.
5. **YouTube Data API** *(optional, for live course recommendations)* —
   Google Cloud Console → enable "YouTube Data API v3" → `YOUTUBE_API_KEY`.
   Without it, the learning module falls back to curated content.

Also invent two long random strings (any value, but keep them stable):
`ASSESSMENT_SHARED_SECRET` and `AGENT_SHARED_SECRET`.

---

## 4. Set up the Supabase database

In your Supabase project, open **SQL Editor** and run these files **in this exact
order** (copy-paste each file's contents and run):

```
platform/supabase/migrations/0001_init.sql
platform/supabase/migrations/0002_phase2.sql
platform/supabase/migrations/0003_interview_attempts.sql
platform/supabase/migrations/0004_learning.sql
platform/supabase/seed.sql           # creates the 4 demo login accounts
platform/supabase/seed_data.sql      # sample jobs + applications
platform/supabase/seed_learning.sql  # competencies, roles, courses, a sample skill report
```

Then:

1. **Authentication → Providers → Email**: turn **"Confirm email" OFF** so the demo
   logins work immediately.
2. **Authentication → URL Configuration**: set **Site URL** to `http://localhost:3002`
   and add `http://localhost:3002/**` to **Redirect URLs**.

The seed creates these logins (email / password):

| Role | Email | Password |
|---|---|---|
| Super Admin | `admin@qore.ai` | `Admin@12345` |
| HR / Admin | `hr@qore.ai` | `Hr@123456` |
| Internal Employee | `employee@qore.ai` | `Employee@123` |
| External Candidate | `candidate@qore.ai` | `Candidate@123` |

---

## 5. Configure environment variables

Each service reads its own env file. Copy the examples and fill in your keys.

### a) platform

```bash
cp platform/.env.example platform/.env.local
```

Edit `platform/.env.local` and fill in the Supabase, Groq, LiveKit values, your two
shared secrets, and (optionally) `YOUTUBE_API_KEY`. Leave the localhost URLs as-is.

### b) quiz-frontend

Create `quiz-frontend/.env.local`:

```
GROQ_API_KEY=your-groq-key
GROQ_MODEL=llama-3.3-70b-versatile
```

### c) agent-ui (agent-starter-react)

```bash
cp agent-starter-react/.env.example agent-starter-react/.env
```

For local dev you only need the quiz URL so the proctor popup links correctly. Add this
line to `agent-starter-react/.env`:

```
NEXT_PUBLIC_QUIZ_URL=http://localhost:3001
```

(LiveKit tokens are minted by the platform and passed in the URL, so LiveKit creds here
are optional locally.)

### d) voice-agent

Create `voice-agent/.env.local`:

```
LIVEKIT_URL=wss://your-project.livekit.cloud
LIVEKIT_API_KEY=your-livekit-key
LIVEKIT_API_SECRET=your-livekit-secret
GROQ_API_KEY=your-groq-key
GROQ_MODEL=llama-3.3-70b-versatile
GROQ_VISION_MODEL=meta-llama/llama-4-scout-17b-16e-instruct
ANAM_API_KEY=your-anam-key          # optional; omit for voice-only
PLATFORM_URL=http://localhost:3002
AGENT_SHARED_SECRET=must-match-platform   # same value as in platform/.env.local
```

> `AGENT_SHARED_SECRET` **must be identical** in `platform/.env.local` and
> `voice-agent/.env.local`, or the agent can't fetch context / post results.

---

## 6. Install dependencies

```bash
# Frontends (run each once)
cd platform            && pnpm install && cd ..
cd quiz-frontend       && pnpm install && cd ..
cd agent-starter-react && pnpm install && cd ..

# Python voice-agent (uv reads pyproject.toml)
cd voice-agent && uv sync && cd ..
```

The first `uv sync` also downloads model files (VAD / turn detection), so it can take a
few minutes.

---

## 7. Run everything

The repo includes a launcher that starts all 4 services together:

```bash
./start.sh
```

This brings up:

- platform → http://localhost:3002
- quiz-frontend → http://localhost:3001
- agent-ui → http://localhost:3000
- voice-agent → connects to LiveKit as a worker (no URL)

Press **Ctrl+C** to stop everything.

<details>
<summary>Prefer to run services individually (4 terminals)?</summary>

```bash
# Terminal 1
cd platform && pnpm dev

# Terminal 2
cd quiz-frontend && pnpm dev

# Terminal 3
cd agent-starter-react && pnpm dev

# Terminal 4
cd voice-agent && uv run agent.py dev
```
</details>

---

## 8. Try it out (smoke test)

1. Open **http://localhost:3002** and log in as `hr@qore.ai` / `Hr@123456`.
2. As HR, confirm sample **jobs** are listed (from `seed_data.sql`).
3. Log out, log in as `candidate@qore.ai` / `Candidate@123`, open a job and **apply
   with a PDF resume** → an **ATS score** should appear (this verifies Groq + Supabase
   storage).
4. Start the **assessment** → the quiz opens at `:3001` and the proctor connects via the
   voice-agent (check the voice-agent terminal for `registered worker` / job logs).
5. Pass the quiz → start the **AI interview**; you should join a LiveKit room at `:3000`
   and the agent should greet you.
6. Log in as `employee@qore.ai` / `Employee@123` → open **Learning** → see the skill
   readiness score, gaps, learning path, and try the **Career Buddy** (text/voice).

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| **Login fails / "Email not confirmed"** | In Supabase, turn **Auth → Email → Confirm email OFF**, then re-run `seed.sql`. |
| **ATS / quiz / Career Buddy: "trouble reaching AI service" or 401** | Your `GROQ_API_KEY` is wrong/expired. Generate a fresh one at console.groq.com and update **all** env files that use it. |
| **PDF resume "could not read text"** | Use a text-based PDF (not a scanned image). DOCX also works. |
| **Interview/Buddy call drops after a couple minutes** | The Anam avatar can drop on long calls; the agent auto-falls back to voice-only. To force voice-only, set `INTERVIEW_USE_AVATAR=0` / `BUDDY_USE_AVATAR=0` in `voice-agent/.env.local`. |
| **voice-agent won't install** | Ensure Python is **3.14+** (or lower `requires-python` in `pyproject.toml`). Re-run `uv sync`. |
| **"No career roles defined" in Learning** | You didn't run `seed_learning.sql`. Run it in the Supabase SQL editor. |
| **Agent connects but can't fetch context** | `AGENT_SHARED_SECRET` must match between `platform/.env.local` and `voice-agent/.env.local`, and `PLATFORM_URL` must be `http://localhost:3002`. |
| **Port already in use** | Stop whatever is on 3000/3001/3002, or change the port in that app's `package.json` `dev` script. |

---

## Project layout

```
Qore_AI/
├── platform/             # Next.js — main hub (jobs, ATS, dashboards, learning, APIs)
│   └── supabase/         # SQL migrations + seed files (run these in Supabase)
├── quiz-frontend/        # Next.js — AI assessment (MCQ + coding)
├── agent-starter-react/  # Next.js — LiveKit voice/video room UI
├── voice-agent/          # Python — LiveKit agent worker (proctor/interview/buddy)
├── start.sh              # launches all 4 services for local dev
├── ARCHITECTURE.md       # system design + diagrams
└── DEPLOY.md             # Railway deployment guide
```

---

## Notes

- Env files (`.env.local`, `.env`) are gitignored — never commit your real keys.
- `start.sh` is for **local dev only**; production runs each service separately
  (see [DEPLOY.md](DEPLOY.md)).
- Free tiers (Groq / LiveKit / Anam) have rate limits; heavy testing may hit them.
