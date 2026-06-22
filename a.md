# Qore AI — End-to-End Architecture

AI-powered **Hiring** + **Learning & Career Buddy** platform. Four deployable
services on Railway, backed by Supabase, LiveKit Cloud, Groq, Anam, and YouTube.

---

## 1. System overview (components, cloud services, AI)

```mermaid
flowchart TB
  subgraph Clients["👤 Clients (browser)"]
    SA["Super Admin"]
    HR["HR / Manager"]
    EMP["Internal Employee"]
    CAND["External Candidate"]
  end

  subgraph Railway["☁️ Railway (our services)"]
    PF["platform (Next.js 15)\nauth · jobs · ATS · dashboards · learning · APIs"]
    QF["quiz-frontend (Next.js)\nAI assessment: MCQ + coding"]
    AUI["agent-ui / agent-starter-react (Next.js)\nLiveKit voice + video room UI"]
    VA["voice-agent (Python worker)\nLiveKit Agents: Proctor / Interview / Buddy"]
  end

  subgraph Managed["🔌 Managed cloud services"]
    SB["Supabase\nPostgres + RLS · Auth · Storage"]
    LK["LiveKit Cloud\nrealtime rooms · agent dispatch · STT/TTS gateway"]
    Ollama["LLM + gemma for vision (OpenAI-compatible)"]
    ANAM["Anam\navatar video persona"]
    YT["YouTube Data API"]
    RESEND["Resend (email, optional)"]
    LIPER["LinkedIn / Percipio (stubbed adapters)"]
  end

  SA --> PF
  HR --> PF
  EMP --> PF
  CAND --> PF

  PF -->|"REST + RLS"| SB
  PF -->|"chat / JSON / vision"| GROQ
  PF -->|"mint LiveKit token"| LK
  PF -->|"content search"| YT
  PF -->|"notifications"| RESEND
  PF -. "adapter (curated)" .-> LIPER

  PF -->|"handoff ?token (HMAC)"| QF
  PF -->|"handoff ?lkToken + ?returnUrl"| AUI
  QF -->|"questions / grade"| GROQ
  QF -->|"context + results (CORS)"| PF

  AUI <-->|"WebRTC audio/video"| LK
  VA <-->|"join room, publish A/V"| LK
  VA -->|"avatar video"| ANAM
  VA -->|"vision + chat"| GROQ
  VA <-->|"context + results (x-agent-secret)"| PF

  LK -. "STT/TTS/VAD inference" .-> VA
```

**Legend:** solid = request/data flow, dotted = inference/optional/stubbed.

---

## 2. AI models, agents & APIs

| Capability | Model / Service | Where it runs | Used for |
|---|---|---|---|
| Text LLM | **Ollama** | platform, quiz, voice-agent | ATS scoring, resume parse, quiz gen, grading, skill-gap, learning path, Career Buddy text, doc-parser transforms, interview scoring |
| Vision LLM | **Gemma** | voice-agent | Screen monitoring (tab-switch), webcam proctoring (faces/gaze/phone) |
| Speech-to-Text | **AssemblyAI `universal-streaming`** (via LiveKit inference) | LiveKit | Candidate/employee speech in interviews & Buddy |
| Text-to-Speech | **Cartesia `sonic-3`** (via LiveKit inference) | LiveKit | Agent voice |
| Voice activity / turns | **Silero VAD** + LiveKit **MultilingualModel** | voice-agent | Turn detection |
| Avatar video | **Anam** persona | voice-agent → LiveKit | Video coaching / interviewer face |
| Learning content | **YouTube Data API** (live) + LinkedIn/Percipio (stub) + internal LMS (DB) | platform | Course recommendations |
| In-browser narration | **Web Speech API** | client | Doc-parser "audio" output |
| Diagram render | **Mermaid** | client | Doc-parser "flowchart" output |

**LiveKit agents (Python, dispatched by room-name prefix):**

| Agent | Room prefix | Responsibility |
|---|---|---|
| `ProctorAgent` | `assessment_<appId>` | Screen + webcam proctoring during the quiz; reports integrity + violations |
| `InterviewAgent` | `interview_<appId>` | JD-driven voice/video interview → scores + hire recommendation |
| `CareerBuddyAgent` | `buddy_<employeeId>` | Voice/video career coaching grounded in learning context |

---

## 3. Hiring pipeline — data flow

```mermaid
sequenceDiagram
  participant C as Candidate (browser)
  participant PF as platform
  participant SB as Supabase
  participant G as Ollama/Gemma
  participant QF as quiz-frontend
  participant VA as voice-agent
  participant LK as LiveKit

  C->>PF: Apply + upload resume
  PF->>SB: store resume (Storage) + application (DB)
  PF->>G: parse resume + ATS score vs JD
  G-->>PF: ats_score + breakdown + reasoning
  PF->>SB: update application - gate on ATS threshold ;
  Note over PF: pass → assessment_assigned · fail → ats_rejected

  C->>PF: Start assessment (identity check)
  PF->>QF: redirect ?token=HMAC&platform
  QF->>PF: GET /api/assessment/context (token)
  QF->>G: generate JD questions + grade coding
  Note over C,QF: ProctorAgent watches via LiveKit room
  QF->>PF: POST /api/assessment/result (token)
  PF->>SB: assessment score; pass/fail vs test threshold

  C->>PF: Start AI interview
  PF->>LK: mint token (room interview_<id>)
  PF->>C: redirect to agent-ui ?lkToken
  C<<->>LK: WebRTC A/V
  VA->>LK: dispatched to interview_ room
  VA->>PF: GET /api/agent/context
  VA->>PF: POST /api/agent/result (score + recommend)
  PF->>SB: decision → hr_review or rejected
```

---

## 4. Realtime voice/video (interview & Career Buddy)

```mermaid
sequenceDiagram
  participant U as User (browser)
  participant PF as platform
  participant AUI as agent-ui
  participant LK as LiveKit Cloud
  participant VA as voice-agent
  participant ANAM as Anam
  participant G as Ollama/Gemma

  U->>PF: Start voice/video session
  PF->>PF: verify role/stage, mint LiveKit token
  PF->>AUI: redirect ?lkUrl&lkToken&returnUrl
  AUI->>LK: connect with literal token
  LK->>VA: dispatch agent (room prefix)
  VA->>PF: fetch context (x-agent-secret)
  VA->>ANAM: start avatar (video)
  loop conversation
    U->>LK: speech
    LK->>VA: STT transcript (AssemblyAI)
    VA->>G: LLM reply (+ vision proctoring)
    VA->>LK: TTS audio (Cartesia) + avatar video
    LK->>U: agent A/V
  end
  Note over VA: avatar drop → watchdog falls back to voice-only
  VA->>PF: post result/summary (x-agent-secret)
  VA->>LK: end → room closed → AUI redirects to returnUrl
```

---

## 5. Learning & Career Buddy module

```mermaid
flowchart LR
  EMP["Internal Employee"] --> DASH["/learn dashboard"]
  DASH --> SKILL["Skill-gap analysis"]
  SKILL -->|"resume + assessments + role + competency framework"| G1["Groq LLM"]
  G1 --> RS["Readiness Score + gaps"]
  RS --> PATH["Personalized learning path"]
  PATH --> REC["Content recommendations"]
  REC --> YT["YouTube (live)"]
  REC --> LMS["Internal LMS (DB)"]
  REC -. stub .-> LP["LinkedIn / Percipio"]
  DASH --> BUDDY["Career Buddy"]
  BUDDY -->|text| G2["Groq LLM"]
  BUDDY -->|voice/video| LKB["LiveKit + CareerBuddyAgent"]
  DASH --> DOC["Intelligent Doc Parser"]
  DOC -->|"PDF/DOCX → text"| EX["extract (pdf-parse / mammoth)"]
  EX --> G3["Groq LLM"]
  G3 --> OUT["summary · key points · flowchart (Mermaid) · flashcards · audio (Web Speech)"]
  EMP --> REQ["Training nominations"] --> MGR["HR/Manager approves"]
  MGR --> HEAT["Team skill heatmap + readiness"]
```

---

## 6. Data model (Supabase Postgres, grouped)

```mermaid
flowchart TB
  subgraph Identity
    org["organizations"]
    prof["profiles (role, manager_id)"]
  end
  subgraph Hiring
    jobs["jobs"]
    apps["applications (ats_score, status)"]
    assess["assessments"]
    ev["application_events"]
    iv["interviews"]
    off["offers"]
  end
  subgraph Learning
    comp["competencies"]
    roles["career_roles + role_competencies"]
    eskill["employee_skills"]
    gap["skill_gap_reports"]
    lp["learning_paths + path_items"]
    crs["courses + enrollments"]
    tr["training_requests"]
    buddy["buddy_conversations + messages"]
  end
  subgraph Storage["Storage buckets"]
    rb["resumes"]
    rec["recordings"]
  end
  subgraph Audit
    al["audit_logs"]
  end

  org --> prof
  jobs --> apps --> assess
  apps --> ev
  apps --> iv
  apps --> off
  prof --> eskill --> comp
  roles --> gap --> lp --> crs
  prof --> tr
  prof --> buddy
```

All tables enforce **Row Level Security**: owners read/write their own rows;
`is_hr()` / `is_super_admin()` widen access; trusted server writes use the
**service-role** key.

---

## 7. Deployment topology (Railway)

```mermaid
flowchart TB
  subgraph RW["Railway project"]
    direction LR
    s1["platform (web :PORT)"]
    s2["quiz-frontend (web :PORT)"]
    s3["agent-ui (web :PORT)"]
    s4["voice-agent (Docker worker, no port)"]
  end
  subgraph EXT["External managed"]
    sb["Supabase"]
    lk["LiveKit Cloud"]
    gq["ollama / Gemma "]
    an["Anam"]
    yt["YouTube API"]
  end
  s1 --- sb
  s1 --- gq
  s1 --- lk
  s1 --- yt
  s2 --- gq
  s3 --- lk
  s4 --- lk
  s4 --- gq
  s4 --- an
  s4 --- s1
```

- 3 Next.js apps build with Nixpacks; voice-agent builds from its **Dockerfile**.
- Cross-service URLs are wired via env (`NEXT_PUBLIC_APP_URL`, `NEXT_PUBLIC_QUIZ_URL`,
  `NEXT_PUBLIC_AGENT_UI_URL`, `PLATFORM_URL`). See [DEPLOY.md](DEPLOY.md).

---

## 8. Security & trust boundaries

| Boundary | Mechanism |
|---|---|
| Browser ↔ DB | Supabase **RLS** (anon key); never the service role |
| Trusted server writes (ATS, results) | Supabase **service-role** key (server-only) |
| platform ↔ quiz-frontend | short-lived **HMAC assessment token** binds session to an application |
| platform ↔ voice-agent | **`AGENT_SHARED_SECRET`** header (server-to-server) |
| LiveKit room access | platform-minted **JWT**, gated by role + application stage |
| Cross-origin assessment APIs | **CORS** locked to the quiz origin |

---

## 9. Tech stack summary

- **Frontend:** Next.js 15 (App Router, React 19), TypeScript, custom dark design system, Mermaid.
- **Backend:** Next.js Route Handlers + Server Actions; Supabase Postgres/Auth/Storage.
- **Realtime:** LiveKit Cloud + LiveKit Agents (Python, `uv`).
- **AI:** Groq (text + vision), AssemblyAI STT, Cartesia TTS, Silero VAD, Anam avatar.
- **Docs/parsing:** pdf-parse, mammoth.
- **Infra:** Railway (4 services), Docker (worker).
```
