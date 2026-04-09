# BizBridge
# BizBridge — Roadmap & Priority Order

> **Hackathon:** Finanz Informatik — AI as a Collaboration Booster
> **Team:** 5 people · **Time:** 20 hours · **Feature freeze:** Hour 14

---

## Priority Tiers

### P0 — Must Have (Hour 0–8)
> **Goal:** Working end-to-end flow — type a requirement, see a structured dev task generated from the real repo.

| Task | Module | Owner | Status |
|---|---|---|---|
| GPT-4o system prompt + JSON schema | `prompt_builder.py` | P1 | TODO |
| User prompt assembler (requirement + repo files) | `prompt_builder.py` | P1 | TODO |
| OpenAI API call with `response_format: json_object` | `codex_client.py` | P1 | TODO |
| JSON field validation (6 required fields) | `codex_client.py` | P1 | TODO |
| GitHub file tree fetch (`GET /git/trees`) | `repo_fetcher.py` | P4 | TODO |
| File content fetch + base64 decode | `repo_fetcher.py` | P4 | TODO |
| `POST /generate` FastAPI route | `main.py` | P2 | TODO |
| `GET /tasks` FastAPI route | `main.py` | P2 | TODO |
| CORS middleware (`allow_origins=["*"]`) | `main.py` | P2 | TODO |
| Firestore `save_task()` | `firestore_client.py` | P4 | TODO |
| Firestore `get_tasks()` | `firestore_client.py` | P4 | TODO |
| React two-panel layout (input left, output right) | Frontend | P3 | TODO |
| Requirement textarea + submit button + spinner | Frontend | P3 | TODO |
| Task card renderer (all 6 JSON fields) | Frontend | P3 | TODO |
| Dockerfile + local Docker build passing | `Dockerfile` | P2 | TODO |

---

### P1 — Should Have (Hour 8–14)
> **Goal:** Full integration live on Cloud Run — real URL, real repo files, real Firestore history.

| Task | Module | Owner | Status |
|---|---|---|---|
| Cloud Run deployment (`gcloud run deploy`) | Infra | P2 | TODO |
| `--min-instances=1` set (prevents cold start) | Infra | P2 | TODO |
| Secret Manager env vars mounted in Cloud Run | Infra | P2 | TODO |
| M1→M2→M3 pipeline connected end-to-end | Integration | P1 + P4 | TODO |
| M3→M4 Firestore save wired after every generation | Integration | P1 + P4 | TODO |
| Frontend `VITE_API_URL` pointed at Cloud Run URL | Frontend | P3 | TODO |
| Task history sidebar (`GET /tasks`) | Frontend | P3 | TODO |
| Firebase Hosting deploy (`firebase deploy`) | Frontend | P3 | TODO |
| Full end-to-end test — all 3 demo scenarios pass | QA | P5 | TODO |
| Firestore composite index created (order_by fix) | Infra | P4 | TODO |

---

### P2 — Nice to Have (Hour 14–18)
> **Goal:** Polish + demo confidence = winning presentation.

| Task | Module | Owner | Status |
|---|---|---|---|
| "Copy as JSON" button on task card | Frontend | P3 | TODO |
| UI spacing + typography cleanup | Frontend | P3 | TODO |
| Task card shows timestamp + repo source | Frontend | P3 | TODO |
| Prompt fine-tuning pass (consistency check) | `prompt_builder.py` | P1 | TODO |
| `/health` endpoint confirmed live | `main.py` | P2 | TODO |
| Cloud Run memory set to 512Mi | Infra | P2 | TODO |
| Slide deck finalised with real app screenshots | Slides | P5 | TODO |
| Live demo rehearsed twice, timed under 90s | Demo | P5 | TODO |
| Backup screen recording saved | Demo | P5 | TODO |

---

### P3 — Cut It (Do not build)
> These add zero value to a hackathon demo. Do not touch.

| Task | Why cut |
|---|---|
| User auth / login | 3 hours cost, zero judge value |
| Multiple repo picker UI | Hard-code the loan calculator repo |
| Streaming / real-time output | Spinner for 10s is fine |
| Rate limiting / API keys per user | Not needed for demo |
| Unit tests | Write after the hackathon |
| CI/CD pipeline | Manual deploy is fast enough |

---

## Work Split

- **P1** — Own `prompt_builder.py` + `codex_client.py`. Test prompt with hardcoded repo files before M1 is ready. The prompt quality is the product — nothing else matters until this is reliable.
- **P2** — Own `main.py` + `Dockerfile` + Cloud Run deployment. Get a live URL by hour 10. Set `--min-instances=1` — non-negotiable.
- **P3** — Own all frontend. Start with a task card that renders from hardcoded JSON. Wire to real API once Cloud Run URL is live.
- **P4** — Own `repo_fetcher.py` + `firestore_client.py`. Test both in isolation with `print()` statements before integrating with the pipeline.
- **P5** — Own demo + slides + QA. Run all 3 demo scenarios every time a phase completes. Log every bug. Record backup video by hour 16.

---

## Key Milestones

1. **Milestone 1 — Hour 2:** GPT-4o returns valid 6-field JSON from hardcoded requirement + hardcoded repo files. Test with `python codex_client.py` directly.
2. **Milestone 2 — Hour 5:** `repo_fetcher.py` returns real files from the loan calculator GitHub repo. Test with `python repo_fetcher.py`.
3. **Milestone 3 — Hour 8:** `POST /generate` works end-to-end locally — requirement in, task JSON out, saved to Firestore. Test with curl or Postman.
4. **Milestone 4 — Hour 10:** Full flow live on Cloud Run URL — frontend on Firebase Hosting calls real backend, real task card renders in browser.
5. **Milestone 5 — Hour 14:** All 3 demo scenarios pass cleanly on the live URL. P5 signs off. Feature freeze begins.

---

## Demo Scenarios (for judges)

Run these in order. Type them exactly as written.

```
1. "Add a feature that calculates how long a loan takes to repay"
2. "Show the user a breakdown of how much of each payment goes to interest vs principal"
3. "Add input validation so the loan amount cannot be negative or zero"
```

Expected output for each: task card with correct file names from the loan calculator repo, plain-English business reason, and testable acceptance criteria.

---

## High-Risk Steps (flag before starting)

| Risk | When | Mitigation |
|---|---|---|
| GPT-4o prompt hallucinating file names | Hour 2–8 | Test with 5+ requirements before integrating. Fail fast. |
| Cloud Run can't access Secret Manager (IAM) | Hour 8–10 | Run `gcloud projects add-iam-policy-binding` with `roles/secretmanager.secretAccessor` |
| Firestore `order_by` throws index error | Hour 10 | Click the auto-generated index link in the error. Takes 2 min to build. |
| Cold start kills demo (8s delay) | Hour 14+ | Set `--min-instances=1` at deploy time. Non-negotiable. |

---

## Hard Rules

- **Feature freeze at Hour 14.** No new code after this point. Stability beats features.
- **Deploy to Cloud Run by Hour 10.** localhost is not a demo.
- **P5 runs QA after every phase.** Don't wait until the end to find out something is broken.
- **Backup video recorded by Hour 16.** If the live demo crashes, you switch to the video without missing a beat.

---

*Still learning. Still building. Let's grow together.*
