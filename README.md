# codeVerse-hackathon---campusFix

# CampusVoice

**Agentic-AI campus grievance redressal platform** (CodeVerse National Hackathon).
Students raise complaints (text, voice, photos, anonymous). AI agents triage, translate, cluster duplicates, rank by distress, and route to the right authority. An SLA watchdog escalates automatically when deadlines are missed. An Emergency SOS reaches responders in seconds.

Docs live in [`docs/`](docs/):

| Doc | Purpose |
|---|---|
| [PROJECT_OVERVIEW](docs/PROJECT_OVERVIEW.md) | Scope, roles, constants (source of truth) |
| [ARCHITECTURE](docs/ARCHITECTURE.md) | Services, flows, security, diagrams |
| [DATABASE_SCHEMA](docs/DATABASE_SCHEMA.md) | Tables, indexes, ER diagram |
| [API_SPEC](docs/API_SPEC.md) | REST contract, RBAC, examples |
| [AGENT_ARCHITECTURE](docs/AGENT_ARCHITECTURE.md) | LangGraph, schemas, prompts, fallbacks |
| [FOLDER_STRUCTURE](docs/FOLDER_STRUCTURE.md) | Repo layout |
| [MVP_ROADMAP](docs/MVP_ROADMAP.md) | 36-hour plan |
| [DEMO_SCRIPT](docs/DEMO_SCRIPT.md) | 5-minute demo |

## 1. Stack

Next.js 14 + Tailwind + shadcn/ui, FastAPI, PostgreSQL 16 + pgvector, Redis + Celery, LangGraph, Anthropic (text and vision), OpenAI (Whisper and embeddings), Mailpit (dev email).

## 2. Prerequisites

- Docker and Docker Compose v2
- API keys: `ANTHROPIC_API_KEY` and `OPENAI_API_KEY` (not needed if you run with `LLM_MODE=mock`)
- Optional for local (non-Docker) dev: Python 3.11, Node 20, and `libmagic` installed on your system

## 3. Quick start (Docker)

```bash
git clone <your-repo-url> campusvoice && cd campusvoice
cp .env.example .env              # then edit keys, or set LLM_MODE=mock
docker compose up -d --build
docker compose exec api alembic upgrade head
docker compose exec api python -m app.seeds.run --profile demo
```

Or simply: `make up migrate seed`.

| URL | What |
|---|---|
| http://localhost:3000 | Web app (log in with the persona switcher) |
| http://localhost:8000/docs | API docs (Swagger) |
| http://localhost:8025 | Mailpit (all outgoing email) |

Check health: `curl http://localhost:8000/ready`

## 4. Environment variables

Copy `.env.example` to `.env`. All variables are read in `apps/api/app/core/config.py` (web variables are prefixed `NEXT_PUBLIC_`).

| Variable | Default | Description |
|---|---|---|
| `APP_ENV` | `dev` | `dev` or `prod` |
| `SECRET_KEY` | (change me) | JWT signing key, 32+ random chars |
| `JWT_ALGORITHM` | `HS256` | |
| `ACCESS_TOKEN_MINUTES` | `60` | |
| `REFRESH_TOKEN_DAYS` | `7` | |
| `ALLOWED_EMAIL_DOMAINS` | `campus.edu` | Comma-separated domains allowed to register |
| `CORS_ORIGINS` | `http://localhost:3000` | Comma-separated |
| `DATABASE_URL` | `postgresql+psycopg://campus:campus@postgres:5432/campusvoice` | |
| `REDIS_URL` | `redis://redis:6379/0` | |
| `LLM_MODE` | `live` | `live` or `mock` (fixture replay, works offline) |
| `LLM_PROVIDER` | `anthropic` | |
| `ANTHROPIC_API_KEY` | | Required when `LLM_MODE=live` |
| `LLM_MODEL_HEAVY` | `claude-sonnet-5-5` | Vision and Triage (and Chat, Insights) |
| `LLM_MODEL_FAST` | `claude-haiku-4-5-20251001` | Translate, Sentiment, Cluster text, Assist |
| `LLM_TIMEOUT_S` | `25` | Per-call timeout |
| `LLM_MAX_RETRIES` | `2` | Before fallback |
| `OPENAI_API_KEY` | | Whisper and embeddings (live mode) |
| `STT_MODEL` | `whisper-1` | |
| `EMBEDDING_MODEL` | `text-embedding-3-small` | |
| `EMBEDDING_DIM` | `1536` | Must match the `vector(1536)` column. Changing it needs a migration |
| `CLUSTER_SIMILARITY_THRESHOLD` | `0.82` | Initial value for `clustering.threshold` setting |
| `UPLOAD_DIR` | `/data/uploads` | Docker volume |
| `MAX_IMAGE_MB` | `5` | |
| `MAX_AUDIO_SECONDS` | `60` | |
| `SMTP_HOST` / `SMTP_PORT` | `mailpit` / `1025` | |
| `EMAIL_FROM` | `campusvoice@campus.edu` | |
| `SMS_PROVIDER` | `mock` | `mock` or `twilio` |
| `TWILIO_ACCOUNT_SID` / `TWILIO_AUTH_TOKEN` / `TWILIO_FROM` | | Only if `SMS_PROVIDER=twilio` |
| `WELLBEING_HELPLINE_TEXT` | helpline text (for example Tele-MANAS 14416) | Initial value for `wellbeing.helpline_text` |
| `DEMO_MODE` | `true` | Enables persona switcher, time-skip, and reset endpoints. **Set `false` in any shared deployment** |
| `NEXT_PUBLIC_API_URL` | `http://localhost:8000` | Browser to API base URL |

## 5. Database

- Image `pgvector/pgvector:pg16`. Extensions (`vector`, `pg_trgm`, `citext`, `pgcrypto`) are created by the first Alembic migration.
- Migrate: `make migrate` (`alembic upgrade head`).
- Reset everything (destroys data): `docker compose down -v && make up migrate seed`.
- Inspect: `make psql`.

## 6. Seed data and demo accounts

`make seed` runs `python -m app.seeds.run --profile demo` (idempotent). Use `--profile minimal` for just the organization and settings.

All demo passwords: **`Demo@1234`**

| Role | Email | Notes |
|---|---|---|
| Student | `aarav.sharma@campus.edu` | Hostel Block B, Room 214, two prior fan complaints |
| Student | `priya.nair@campus.edu`, `rohan.mehta@campus.edu`, `sneha.iyer@campus.edu` | In the seeded Mess B cluster |
| Student | `karan.verma@campus.edu`, `ananya.das@campus.edu` | Used live for clustering and Hindi voice |
| Authority L1 | `warden.b@campus.edu` | Hostel Block B |
| Authority L1 | `mess.supervisor@campus.edu` | Mess B |
| Authority L1 | `lab.assistant@campus.edu` | CS Lab 1 |
| Authority L1 | `helpdesk@campus.edu` | General fallback |
| Authority L1 | `medical.officer@campus.edu` | Medical, also SOS responder (tier 1) |
| Authority L2 | `chief.warden@campus.edu` | Hostel, also SOS responder (tier 1) |
| Authority L2 | `mess.head@campus.edu`, `hod.cse@campus.edu`, `chief.medical@campus.edu` | |
| Authority L3 | `dean.welfare@campus.edu` | Hostel, Mess, Medical (L3); General (L2). SOS tier 2 |
| Authority L3 | `dean.academics@campus.edu` | Lab |
| Authority L4 | `director@campus.edu` | All domains |
| SOS responders | `security.chief@campus.edu` (tier 1), `registrar@campus.edu` (tier 2, also General L3) | |
| Counsellor | `counsellor@campus.edu` | Wellbeing alerts |
| Admin | `admin@campus.edu` | Full admin and demo controls |
| Admin + committee | `committee@campus.edu` | Has `can_view_anonymous_identity` (ICC and anti-ragging committee) |

Seed demo state: an open **Mess B cluster of 3** (2 days old, at L1), two closed Hostel B fan complaints for Aarav, about 80 historical closed complaints, and pre-computed embeddings for offline clustering.

## 7. Running locally without Docker (optional)

```bash
# infra only
docker compose up -d postgres redis mailpit

# backend
cd apps/api
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
export DATABASE_URL=postgresql+psycopg://campus:campus@localhost:5432/campusvoice
export REDIS_URL=redis://localhost:6379/0 SMTP_HOST=localhost
alembic upgrade head
python -m app.seeds.run --profile demo
uvicorn app.main:app --reload --port 8000
# in two more terminals:
celery -A app.tasks.celery_app worker -Q intake,sos,notify -l info
celery -A app.tasks.celery_app beat -l info

# frontend
cd apps/web
npm ci
npm run gen:api      # generate TS types from http://localhost:8000/openapi.json
npm run dev          # http://localhost:3000
```

## 8. Makefile targets

| Target | Action |
|---|---|
| `make up` / `make down` | Start or stop the stack |
| `make logs` | Follow logs for all services |
| `make migrate` | Apply Alembic migrations |
| `make seed` | Seed the demo profile |
| `make reset-demo` | Re-seed demo state and reset the simulated clock |
| `make test` | Run backend tests (`pytest`) |
| `make fixtures-record` | Run the demo scenarios against live LLMs and save responses to `demo/fixtures/` |
| `make fixtures-load` | Copy `demo/fixtures/` into the API's fixture directory (used by `LLM_MODE=mock`) |
| `make gen-api` | Regenerate frontend API types from OpenAPI |
| `make psql` / `make shell-api` | DB shell / API container shell |

## 9. Offline and mock mode (venue Wi-Fi insurance)

1. While online: set `LLM_MODE=live`, run `make fixtures-record` (executes the demo scenarios from `demo/scenarios.md` and stores every agent response, embedding, and transcript).
2. Before the demo: `make fixtures-load`, set `LLM_MODE=mock` in `.env`, then `docker compose up -d`.
3. In mock mode the agents replay the recorded outputs by input hash, so the demo behaves identically without internet. Unknown inputs fall back to the rules-based fallbacks.

## 10. Demo instructions

1. `make reset-demo` and confirm the clock offset is 0 on the admin Demo page.
2. Open the four windows and the persona switcher as described in [DEMO_SCRIPT](docs/DEMO_SCRIPT.md).
3. Follow the script: clustering and Hindi voice, photo evidence and priority breakdown, SOS, **Simulate +6 days**, admin analytics.
4. Reset between rehearsals with `make reset-demo` (about 20 s).

Demo assets go in `demo/assets/`: `broken_fan.jpg`, `mess_hindi.webm`, `fan_after.jpg`, `rice_1.jpg`. The exact texts used are in `demo/scenarios.md`.

## 11. Testing

```bash
make test                 # unit, agent (mock LLM), and API tests
```

Key tests: priority golden test (demo complaint = 60.5, band HOT), routing conflict skip, state transitions, safety rules (forced CRITICAL, wellbeing alert), fallbacks (force LLM errors, complaint still routes), RBAC negative cases.

## 12. Troubleshooting

| Symptom | Fix |
|---|---|
| `api` fails with extension `vector` not found | Use the `pgvector/pgvector:pg16` image, not plain `postgres` |
| Complaint stuck in `PROCESSING` | Check `docker compose logs worker`. Usually missing API keys (use `LLM_MODE=mock`) or Redis unreachable |
| SSE events not arriving | Check that the browser reaches `NEXT_PUBLIC_API_URL` and that no proxy buffers `text/event-stream` |
| Time-skip has no effect | Confirm `DEMO_MODE=true`, you are logged in as `admin`, and `beat`/`worker` are running (the endpoint also triggers the watchdog directly) |
| Voice upload rejected | Audio must be <= 60 s, <= 5 MB, and one of webm/ogg/mp3/wav/m4a |
| Hindi voice is mis-transcribed | Speak slowly and use the fixture clip `demo/assets/mess_hindi.webm` |
| Clustering merges too much or too little | Tune `clustering.threshold` in Admin, Priority and Clustering settings (default 0.82) |
| Emails not visible | Open Mailpit at http://localhost:8025 (nothing is sent to real inboxes) |

## 13. Security notes

- Never commit `.env`. Rotate any key that was shared in a chat or a screenshot.
- `DEMO_MODE=false` removes the persona switcher and all `/admin/demo/*` endpoints. Do this for any deployment other than the demo laptop.
- The default `SECRET_KEY` and demo passwords are for local use only.

## 14. Team

| Role | Owner | Area |
|---|---|---|
| A | | Frontend |
| B | | Backend core |
| C | | AI and agents |
| D | | Data, demo, DevOps |


