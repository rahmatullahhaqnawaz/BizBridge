# CLAUDE.md

This file tells Claude (and any new contributor) exactly how BizBridge is structured,
how to run it, and how to contribute without breaking things.

---

## Project overview

**BizBridge** is an AI-powered developer task generator. A Product Owner types a plain-English
feature request. BizBridge fetches the real codebase from GitHub, sends it to GPT-4o alongside
the requirement, and returns a structured developer task — with affected files, what to build,
why, and acceptance criteria — in under 10 seconds.

Built for the **Finanz Informatik Hackathon — AI as a Collaboration Booster** track.

**The problem it solves:** Business and engineering teams lose days to requirement translation.
POs write vague tickets. Devs ask 10 clarification questions. BizBridge eliminates that
back-and-forth by grounding every task in the actual codebase.

**What it is not:** A code generator. A chatbot. A project management tool.
It generates one thing: a precise, code-aware developer task from a business requirement.

---

## Stack

| Layer | Technology | Why |
|---|---|---|
| Frontend | React + Vite + Tailwind | Fast scaffold, no SSR needed, Tailwind keeps styling fast |
| Backend | FastAPI (Python 3.11) | Unified language with AI layer, async, auto docs |
| AI | GPT-4o via OpenAI API | JSON mode, 128k context, best code understanding |
| Database | Google Firestore | Serverless, schema-free, native GCP auth from Cloud Run |
| Hosting (frontend) | Firebase Hosting | One command deploy, free tier, same GCP ecosystem |
| Hosting (backend) | Google Cloud Run | Serverless containers, min-instances=1 prevents cold start |
| Secrets | Google Secret Manager | API keys never in env files in production |
| Repo source | GitHub REST API | Fetches real files on every request — no stale cache |

---

## Folder structure

```
bizbridge/
│
├── backend/
│   ├── main.py                  # FastAPI app — POST /generate, GET /tasks, GET /health
│   ├── repo_fetcher.py          # GitHub REST API — file tree + content fetch
│   ├── prompt_builder.py        # Assembles system + user prompt for GPT-4o
│   ├── codex_client.py          # OpenAI API call, JSON validation, error handling
│   ├── firestore_client.py      # save_task() and get_tasks() — nothing else
│   ├── requirements.txt         # Python dependencies
│   ├── Dockerfile               # python:3.11-slim, uvicorn on port 8080
│   └── .env.example             # Copy to .env and fill in values
│
├── frontend/
│   └── bizbridge-ui/
│       ├── src/
│       │   ├── App.jsx          # Root component — two-panel layout
│       │   ├── components/
│       │   │   ├── RequirementForm.jsx   # Left panel — textarea + submit
│       │   │   ├── TaskCard.jsx          # Right panel — renders 6-field task
│       │   │   └── TaskHistory.jsx       # Sidebar — GET /tasks list
│       │   └── main.jsx
│       ├── .env.example         # Copy to .env and fill in VITE_API_URL
│       ├── index.html
│       ├── vite.config.js
│       ├── tailwind.config.js
│       └── package.json
│
├── CLAUDE.md                    # This file
├── README.md                    # Full code + setup guide
└── ROADMAP.md                   # Priority tiers, milestones, team split
```

---

## How to run locally

### Prerequisites

- Python 3.11+
- Node 18+
- A Google Cloud project with Firestore enabled (Native mode)
- An OpenAI API key
- A GitHub personal access token (classic, `repo` scope)

### Backend

```bash
cd backend

# Create virtual environment
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
cp .env.example .env
# Fill in .env (see Environment variables section below)

# Run the server
uvicorn main:app --reload --port 8000
```

API docs: http://localhost:8000/docs
Health check: http://localhost:8000/health

### Frontend

```bash
cd frontend/bizbridge-ui

# Install dependencies
npm install

# Set up environment variables
cp .env.example .env
# Set VITE_API_URL=http://localhost:8000

# Run dev server
npm run dev
```

App: http://localhost:5173

### Test the full flow with curl

```bash
curl -X POST http://localhost:8000/generate \
  -H "Content-Type: application/json" \
  -d '{
    "requirement": "Add a feature that calculates how long a loan takes to repay",
    "repo_owner": "your-repo-owner",
    "repo_name": "loan-calculator"
  }'
```

Expected: JSON object with `id` and `task` containing all 6 fields.

---

## Environment variables

### Backend — `backend/.env`

```env
# OpenAI
OPENAI_API_KEY=sk-...

# GitHub — classic personal access token, repo scope only
GITHUB_TOKEN=ghp_...

# Google Cloud
GCP_PROJECT_ID=your-gcp-project-id

# Default repo to analyze (can be overridden per request)
REPO_OWNER=target-github-username
REPO_NAME=loan-calculator
```

### Frontend — `frontend/bizbridge-ui/.env`

```env
# Point to your local backend or Cloud Run URL
VITE_API_URL=http://localhost:8000
```

### Production (Cloud Run)

In production, `OPENAI_API_KEY` and `GITHUB_TOKEN` are mounted from Google Secret Manager —
never set as plain env vars. `GCP_PROJECT_ID`, `REPO_OWNER`, and `REPO_NAME` are set as
plain Cloud Run env vars (not sensitive).

```bash
gcloud run deploy bizbridge-backend \
  --set-secrets="OPENAI_API_KEY=openai-api-key:latest,GITHUB_TOKEN=github-token:latest" \
  --set-env-vars="GCP_PROJECT_ID=your-id,REPO_OWNER=owner,REPO_NAME=loan-calculator"
```

---

## Key constraints

- **No streaming.** GPT-4o is called synchronously. A loading spinner for ~10 seconds is
  intentional and acceptable. Do not add streaming — it adds complexity for no demo value.

- **No auth.** CORS is open (`allow_origins=["*"]`). This is a hackathon build. Do not add
  login, JWT, or API key gating during the event.

- **Hard-coded repo.** The loan calculator repo is hard-coded as the default target.
  A repo picker UI is explicitly cut — one repo, one focused demo.

- **Top 10 files only.** `repo_fetcher.py` fetches a maximum of 10 source files per request
  to stay within GPT-4o context limits and keep latency under 10 seconds. Do not increase
  this without profiling token usage first.

- **6 required output fields.** The JSON output from GPT-4o must contain exactly:
  `task_title`, `affected_files`, `what_to_build`, `why`, `business_reason`,
  `acceptance_criteria`. Any response missing a field is rejected with a `500` error —
  never let partial output reach the frontend.

- **`--min-instances=1` on Cloud Run.** Non-negotiable. Cold start is ~8 seconds and will
  kill the live demo. This setting keeps one instance warm at all times.

---

## API reference

### `POST /generate`

Generates a structured developer task from a plain-English requirement.

**Request body:**
```json
{
  "requirement": "string — plain English feature request",
  "repo_owner": "string — GitHub username or org",
  "repo_name": "string — repository name"
}
```

**Response:**
```json
{
  "id": "firestore-document-id",
  "task": {
    "task_title": "string",
    "affected_files": ["list", "of", "filenames"],
    "what_to_build": "string — exact function/class/endpoint",
    "why": "string — technical reasoning from the codebase",
    "business_reason": "string — plain English for non-technical readers",
    "acceptance_criteria": ["testable", "conditions"]
  }
}
```

**Error responses:**
- `400` — empty requirement
- `404` — no source files found in repo
- `502` — GitHub API fetch failed
- `500` — GPT-4o returned invalid/incomplete JSON, or Firestore write failed

### `GET /tasks`

Returns the 10 most recently generated tasks from Firestore, newest first.

### `GET /health`

Returns `{"status": "ok"}`. Use before the demo to confirm Cloud Run is live.

---

## Module responsibilities

Each module has one job. Do not expand these responsibilities.

| Module | Job | Do not add |
|---|---|---|
| `repo_fetcher.py` | Fetch + decode GitHub files | Caching, file scoring, semantic search |
| `prompt_builder.py` | Assemble prompts | Calling OpenAI, touching Firestore |
| `codex_client.py` | Call OpenAI, validate JSON | Business logic, prompt construction |
| `firestore_client.py` | Read/write tasks | Any other collection or query logic |
| `main.py` | HTTP routing + orchestration | Business logic (delegate to modules) |

---

## Contribution notes

### Branch strategy

```
main          — stable, demo-ready at all times
dev           — integration branch, merge feature branches here first
feature/xxx   — individual feature branches (e.g. feature/history-sidebar)
```

Never push directly to `main` during the hackathon. PR into `dev`, confirm it works,
then merge `dev` → `main` only before a milestone checkpoint.

### Commit message format

```
[module] short description of what changed

Examples:
[prompt] tighten JSON schema to enforce affected_files as list
[backend] add 502 error handling for GitHub rate limit
[frontend] wire TaskHistory to GET /tasks endpoint
[infra] set min-instances=1 in Cloud Run deploy command
```

### Before opening a PR

- [ ] `POST /generate` still returns valid JSON with all 6 fields
- [ ] `GET /health` returns 200
- [ ] No hardcoded API keys anywhere in the diff
- [ ] Docker build passes locally (`docker build -t bizbridge-backend .`)
- [ ] Frontend renders task card without console errors

### Known gotchas

1. **Firestore `order_by` index error** — First time you run `GET /tasks` in a fresh
   Firestore database, you'll get an index error with a direct link to create it.
   Click the link. The index takes ~2 minutes to build. This is expected.

2. **Cloud Run IAM** — If Cloud Run can't read secrets, the service account needs
   `roles/secretmanager.secretAccessor`. Run:
   ```bash
   gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
     --member="serviceAccount:YOUR_SA@developer.gserviceaccount.com" \
     --role="roles/secretmanager.secretAccessor"
   ```

3. **GPT-4o JSON mode requirement** — When using `response_format: json_object`,
   the word "JSON" must appear somewhere in the prompt. `prompt_builder.py` handles
   this — do not remove the word "JSON" from the system prompt or the API will error.

4. **GitHub base64 content** — The GitHub Contents API returns file content as
   base64 with newlines embedded in the string. Always decode with:
   ```python
   base64.b64decode(data["content"].replace("\n", "")).decode("utf-8")
   ```

---

## Team

| Person | Role | Owns |
|---|---|---|
| P1 | Prompt engineering + AI integration | `prompt_builder.py`, `codex_client.py` |
| P2 | Backend + Cloud Run | `main.py`, `Dockerfile`, deployment |
| P3 | Frontend | React UI, Firebase Hosting |
| P4 | Data + GitHub API | `repo_fetcher.py`, `firestore_client.py` |
| P5 | Demo + QA + slides | Pitch deck, test scenarios, backup video |

---

*Still learning. Still building. Let's grow together.*
