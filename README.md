# HireIntOS - AI Multi-Persona Interview Platform

> Practice interviews with AI agents that read your resume, adapt in real-time, give hints not answers, and score you on professional rubrics.


**Live Demo:** https://hireintos-frontend-856672744274.us-central1.run.app  
**Backend API:** https://hireintos-backend-856672744274.us-central1.run.app  
**Demo Video:** https://youtu.be/iyTSmtlEOZE

---

## What It Does

HireIntOS simulates a real multi-round hiring process with 4 specialized AI interviewer agents. Sign up, upload your resume and target job description - the agents generate personalized questions, remember your past sessions, and adapt difficulty based on your performance.

### The 4 AI Interviewers

| Agent | What It Does |
|-------|-------------|
| 🤝 **HR Agent** | Behavioral & cultural fit questions using STAR method |
| 📋 **Manager Agent** | Situational scenarios about leadership & prioritization |
| 🧠 **Technical Agent** | System design & CS concepts, adapts to your level |
| 💻 **Coding Agent** | Live coding with progressive hints — never gives the answer |

### Key Features

- **User authentication** - Sign up / login with email + password, all sessions linked to your account
- **Resume-aware** - Agents generate questions from your resume + job description
- **Voice-first interview UI** - Circular waveform animation, auto-listen after agent speaks
- **Webcam + proctoring** - MediaPipe face mesh eye tracking detects when you look away, beeps as warning
- **Hints-only coding round** - AI guides you with progressive hints, never reveals solutions
- **Monaco code editor** - Full IDE experience for the coding round
- **Multi-dimensional scorecard** - Rubric-based evaluation per round
- **Dashboard** - Session history, score trends, strengths/weaknesses, progress over time
- **Candidate memory** - Remembers past sessions, tracks growth across interviews
- **Adaptive difficulty** - Adjusts question difficulty based on past performance
- **PDF upload** - Upload resume and JD as PDFs
- **Session timer** - Tracks interview duration

---

## MongoDB MCP Server Integration

This project uses the [MongoDB MCP Server](https://github.com/mongodb-js/mongodb-mcp-server) as the primary tool interface between AI agents and the database - making MongoDB Atlas the **persistent memory layer** for all agentic workloads.

### Why MongoDB is Essential (Not Just Storage)

Unlike a simple CRUD backend, our agents **autonomously query and write to MongoDB during conversations** via the MCP Server. MongoDB Atlas serves as the unified operational foundation where:

1. **Agents reason over historical data** - Before asking questions, agents query past evaluations to understand candidate strengths/weaknesses
2. **Agents adapt in real-time** - Aggregation pipelines calculate average scores to dynamically adjust difficulty
3. **Agents deduplicate questions** - Queries prevent repeating problems across sessions
4. **Agents persist their own evaluations** - At round end, agents write structured rubric scores directly to MongoDB
5. **Cross-session memory** - Agents reference performance from previous sessions, noting improvement trends

### MCP Tools Used by Agents

| MCP Tool | Agent Usage |
|----------|-------------|
| `find` | Fetch coding questions by difficulty, lookup user sessions, check candidate history |
| `aggregate` | Calculate avg scores across sessions, identify weak dimensions, compute adaptive difficulty |
| `insertMany` | Save rubric evaluations, create session records |
| `updateMany` | Append conversation messages, save proctoring data, update session status |
| `count` | Check how many problems candidate has solved, verify evaluation existence |
| `listCollections` | Agent discovery of available data collections |

### Data Model (4 Collections)

| Collection | Purpose | Indexed Fields |
|-----------|---------|----------------|
| `users` | Authentication + session linkage | `email` (unique) |
| `sessions` | Full interview state — rounds, resume, JD, messages, proctoring | `session_id` (unique), `email`, `created_at` |
| `evaluations` | Per-round rubric scores written by agents | `session_id`, `candidate_name`, `(session_id, round_type)`, `(candidate_name, round_type)` |
| `questions` | Coding problem bank (seeded) | `(round_type, difficulty)` |

### Aggregation Pipeline Examples

Agents execute these aggregations via MCP at runtime:

```javascript
// Calculate adaptive difficulty (agent runs this before picking a question)
db.evaluations.aggregate([
  { $match: { candidate_name: "Sreeram", round_type: "coding" } },
  { $group: { _id: null, avgScore: { $avg: "$overall_score" } } }
])
// Result: avgScore >= 4 → hard, >= 2.5 → medium, else → easy

// Identify weak dimensions for targeted questioning
db.evaluations.aggregate([
  { $match: { candidate_name: "Sreeram" } },
  { $unwind: { path: "$scores" } },
  { $group: { _id: "$scores.dimension", avg: { $avg: "$scores.value" } } },
  { $sort: { avg: 1 } },
  { $limit: 3 }
])

// Progress tracking across sessions
db.evaluations.aggregate([
  { $match: { session_id: { $in: userSessionIds } } },
  { $sort: { created_at: 1 } },
  { $group: { _id: "$round_type", scores: { $push: "$overall_score" } } }
])
```

### Document Structure Examples

```javascript
// Evaluation document (written by agent via MCP)
{
  session_id: "uuid-123",
  round_type: "technical",
  candidate_name: "Sreeram",
  scores: {
    depth_of_knowledge: 4,
    system_design: 3,
    problem_decomposition: 4,
    tradeoff_analysis: 3,
    communication: 5
  },
  feedback: "Strong communicator with good system design intuition...",
  overall_score: 3.8,
  topics_covered: ["distributed systems", "caching", "load balancing"],
  difficulty_level: "medium",
  proctoring_warnings: 2,
  created_at: ISODate("2026-06-11T07:00:00Z")
}

// Session document (persistent memory)
{
  session_id: "uuid-123",
  candidate_name: "Sreeram",
  email: "sreeram@example.com",
  rounds: ["hr", "technical", "coding"],
  current_round_index: 1,
  resume_text: "...",
  job_description: "...",
  messages: [
    { round: "hr", user: "...", agent: "...", timestamp: ISODate() }
  ],
  proctoring: { hr: { warning_count: 0 }, technical: { warning_count: 2 } },
  created_at: ISODate("2026-06-11T06:30:00Z")
}
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        FRONTEND                                      │
│                 Next.js 15 + React 19 + Tailwind v4                 │
│                                                                      │
│  ┌──────────┐  ┌──────┐  ┌──────────────┐  ┌───────────┐          │
│  │ Landing  │→ │ Auth │→ │Persona Select│→ │ Interview │          │
│  │  Page    │  │ Page │  │ + PDF Upload │  │   Page    │          │
│  └──────────┘  └──────┘  └──────────────┘  └─────┬─────┘          │
│                                                    │                 │
│                              ┌─────────────────────┼──────────┐     │
│                              │                     │          │     │
│                        ┌─────▼────┐  ┌──────▼─────┐  ┌──────▼──┐  │
│                        │ Circular │  │   Monaco   │  │ Webcam  │  │
│                        │ Waveform │  │   Editor   │  │+Proctor │  │
│                        │  + Voice │  │ (Coding)   │  │MediaPipe│  │
│                        └──────────┘  └────────────┘  └─────────┘  │
│                                                                      │
│  ┌──────────────┐  ┌───────────┐                                    │
│  │  Dashboard   │  │ Scorecard │                                    │
│  │ History/Stats│  │  Results  │                                    │
│  └──────────────┘  └───────────┘                                    │
└────────────────────────────┬────────────────────────────────────────┘
                             │ REST API (JSON)
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        BACKEND                                       │
│                   Python + FastAPI                                    │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                    API Routes                                   │ │
│  │  /api/auth/signup|login  → User authentication                 │ │
│  │  /api/sessions/start     → Create session + store resume/JD    │ │
│  │  /api/interview/chat     → Send message to current agent       │ │
│  │  /api/interview/next     → Advance to next round               │ │
│  │  /api/dashboard/history  → User session history                │ │
│  │  /api/dashboard/progress → Score trends + analytics            │ │
│  │  /api/upload/parse-pdf   → Extract text from PDF               │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                             │                                        │
│  ┌──────────────────────────▼─────────────────────────────────────┐ │
│  │              Google ADK (Agent Development Kit)                  │ │
│  │                                                                  │ │
│  │  ┌─────────────────────────────────────────────────────────┐   │ │
│  │  │                 Orchestrator Agent                        │   │ │
│  │  │         Routes to correct persona based on round         │   │ │
│  │  └──────┬──────────┬──────────────┬──────────────┬─────────┘   │ │
│  │         │          │              │              │              │ │
│  │  ┌──────▼───┐ ┌────▼─────┐ ┌─────▼──────┐ ┌────▼──────┐     │ │
│  │  │ HR Agent │ │ Manager  │ │ Technical  │ │  Coding   │     │ │
│  │  │ STAR     │ │ Scenario │ │ Sys Design │ │ Hints Only│     │ │
│  │  │ Method   │ │ Based    │ │ Adaptive   │ │ Progressive│    │ │
│  │  └──────┬───┘ └────┬─────┘ └─────┬──────┘ └────┬──────┘     │ │
│  │         └──────────┴──────────────┴──────────────┘              │ │
│  │                             │                                    │ │
│  │                    MongoDB MCP Toolset                           │ │
│  └─────────────────────────────┬──────────────────────────────────┘ │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ stdio (npx mongodb-mcp-server)
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    MongoDB MCP Server                                 │
│              (Official: mongodb-js/mongodb-mcp-server)                │
│                                                                      │
│  Tools exposed to agents:                                            │
│  ┌────────────┐ ┌────────────┐ ┌──────────────┐ ┌──────────────┐  │
│  │   find     │ │ aggregate  │ │ insert-many  │ │ update-many  │  │
│  └────────────┘ └────────────┘ └──────────────┘ └──────────────┘  │
│  ┌────────────┐ ┌────────────┐ ┌──────────────┐ ┌──────────────┐  │
│  │   count    │ │list-colls  │ │ coll-schema  │ │ delete-many  │  │
│  └────────────┘ └────────────┘ └──────────────┘ └──────────────┘  │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ MongoDB Wire Protocol (TLS)
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      MongoDB Atlas (Cloud)                            │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │    users          │  │    sessions       │  │   evaluations    │  │
│  │ - name            │  │ - session_id      │  │ - session_id     │  │
│  │ - email (unique)  │  │ - candidate_name  │  │ - round_type     │  │
│  │ - password (hash) │  │ - email           │  │ - scores{}       │  │
│  │ - sessions[]      │  │ - rounds[]        │  │ - feedback       │  │
│  │ - created_at      │  │ - resume_text     │  │ - overall_score  │  │
│  │                    │  │ - job_description │  │ - hint_count     │  │
│  │  ┌────────────┐   │  │ - messages[]      │  │ - proctoring     │  │
│  │  │ questions   │   │  │ - proctoring{}    │  │ - created_at     │  │
│  │  │ - title     │   │  │ - created_at      │  │                  │  │
│  │  │ - difficulty│   │  └──────────────────┘  └──────────────────┘  │
│  │  │ - hints[]   │   │                                              │
│  │  │ - examples[]│   │                                              │
│  │  └────────────┘   │                                              │
│  └──────────────────┘                                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Start

### Prerequisites

- Python 3.12+
- Node.js 20+
- MongoDB Atlas cluster
- Google Gemini API key

### Backend

```bash
cd backend
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env  # Add your credentials
python -m app.seed_data  # Seed coding questions
uvicorn app.main:app --reload --port 8000
```

### Frontend

```bash
cd frontend
npm install
npm run dev
```

Open http://localhost:3000

### Environment Variables

```env
GOOGLE_API_KEY=your_gemini_api_key
MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/
MONGODB_DATABASE=hireintos
```

---

## Deployment (Google Cloud Run)

Both services are deployed on Cloud Run:

```bash
# Build and push backend
gcloud builds submit --config=cloudbuild-backend.yaml .

# Deploy backend
gcloud run deploy hireintos-backend \
  --image us-central1-docker.pkg.dev/PROJECT_ID/hireintos/backend:latest \
  --region us-central1 --allow-unauthenticated \
  --set-env-vars "GOOGLE_API_KEY=...,MONGODB_URI=...,MONGODB_DATABASE=hireintos" \
  --memory 1Gi --timeout 300 --port 8000

# Build and push frontend
gcloud builds submit --config=cloudbuild-frontend.yaml .

# Deploy frontend
gcloud run deploy hireintos-frontend \
  --image us-central1-docker.pkg.dev/PROJECT_ID/hireintos/frontend:latest \
  --region us-central1 --allow-unauthenticated --port 3000
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| LLM | Google Gemini 2.5 Flash |
| Agent Framework | Google ADK (Agent Development Kit) |
| Database Tool | MongoDB MCP Server (npx mongodb-mcp-server) |
| Backend | Python, FastAPI, Motor (async MongoDB) |
| Frontend | Next.js 15, React 19, TypeScript, Tailwind CSS v4 |
| Code Editor | Monaco Editor |
| Voice | Web Speech API (TTS + STT) |
| Proctoring | MediaPipe Face Mesh (eye tracking) |
| Auth | Email + password with salted SHA-256 hashing |
| Database | MongoDB Atlas |
| Deployment | Google Cloud Run + Cloud Build + Artifact Registry |

---

## How This Maps to Judging Criteria

| Criteria | How We Address It |
|----------|-------------------|
| **Technological Implementation** | Multi-agent orchestration via Google ADK, MongoDB MCP Server for autonomous agent-DB interaction, MediaPipe face tracking, Web Speech API, deployed on Cloud Run with CI/CD |
| **Design** | Voice-first circular waveform UI for Q&A rounds, Monaco editor for coding, responsive glass-morphism design, session timer, proctoring indicators |
| **Potential Impact** | Democratizes interview prep — candidates get unlimited AI-powered practice with personalized feedback, tracks growth over time, applicable to students and job seekers globally |
| **Quality of the Idea** | Unique combination: multi-persona agents that remember you, hints-only coding (never gives answers), eye-tracking proctoring, adaptive difficulty from MongoDB history |

---

## Demo

### Screenshots

| Landing Page | Interview Persona Selection |
|:---:|:---:|
| ![Landing](IMAGES/1.png) | ![Auth](IMAGES/2.png) |

| Live Interview | Interview Results |
|:---:|:---:|
| ![Select](IMAGES/3.png) | ![Interview](IMAGES/4.png) |

| Coding Round | Dashboard |
|:---:|:---:|
| ![Coding](IMAGES/5.png) | ![Dashboard](IMAGES/6.png) |

### Platform Flow

1. **Sign Up / Login** → Create account with email + password
2. **Select Interviewers** → Choose rounds + upload resume/JD (PDF)
3. **Interview** → Voice-first circular waveform UI (HR/Manager/Technical) or Chat + Editor (Coding)
4. **End Round** → Agent evaluates and saves scores to MongoDB
5. **Dashboard** → View history, score trends, strengths/weaknesses
6. **Scorecard** → Detailed per-round rubric scores and feedback

---

Built by [sreeram2001](https://github.com/sreeram2001) and [charusnehalr](https://github.com/charusnehalr)

## License

MIT
