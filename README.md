# TechHire

[![CI](https://github.com/jayminsheladia/TechHire/actions/workflows/ci.yml/badge.svg)](https://github.com/jayminsheladia/TechHire/actions/workflows/ci.yml)

TechHire is a full-stack AI-powered job search platform for CS students and new-grad engineers. It aggregates listings from multiple sources, lets you filter by skill/salary/visa/work-mode, generates AI job summaries with a multi-model ensemble, and scores your resume against any job description.

**Live demo:** _add your deployed URL here after following [DEPLOYMENT.md](DEPLOYMENT.md)_

---

## Screenshots

> _Add screenshots/GIF here — run `npm run dev` in `frontend/`, then capture: the jobs grid with filters open, a job's AI summary panel, and the resume checker results. A picture here does more for a recruiter skimming this page than any bullet list below._

---

## Architecture

```mermaid
flowchart LR
    subgraph Sources["Job sources"]
        GH[Greenhouse API]
        LVR[Lever API]
        ASH[Ashby API]
        JS[JSearch aggregator]
    end

    GH --> PG[(PostgreSQL)]
    LVR --> PG
    ASH --> PG
    JS --> PG

    PG <--> API[FastAPI backend]
    API <--> FE[React frontend]
    API <--> RDS[(Redis)]
    RDS -. query cache + rate limits .-> API

    API --> M1[Groq · gpt-oss-120b]
    API --> M2[Groq · Qwen3.6-27B]
    API --> M3[Groq · gpt-oss-20b]
    M1 --> SYN[Synthesis pass]
    M2 --> SYN
    M3 --> SYN
    SYN --> API
```

The AI job-summary endpoint runs three Groq models concurrently, each prompted from a different angle (role/day-to-day, fit/growth, practical/requirements), then synthesizes the three outputs into one summary with a fourth call — reducing single-model bias instead of trusting one model's take.

**Job sourcing:** Greenhouse, Lever, and Ashby each publish a public, unauthenticated job-board API per company (`boards-api.greenhouse.io`, `api.lever.co`, `api.ashbyhq.com`) — explicitly meant for external consumption, no ToS issue. `python main.py` pulls real, live postings from ~27 real company boards (Stripe, OpenAI, Databricks, Palantir, Ramp, and others) through these three integrations with zero API key required. JSearch (RapidAPI) is wired in as a fourth, optional source for broader keyword-search coverage if you add a `RAPIDAPI_KEY` — TechHire deliberately does not scrape Indeed/Glassdoor/LinkedIn directly, since that would mean defeating their anti-bot measures to violate their Terms of Service.

---

## Features

- 🔍 Search and browse aggregated job listings
- 🎯 Filter by skills, salary, experience level, work mode, visa sponsorship, date posted
- 🤖 AI-generated job summaries (3-model ensemble + synthesis)
- 📄 Resume upload (PDF/TXT) with text extraction
- 📊 AI resume-to-job-description scoring with section-by-section rewrite suggestions
- 🔄 Admin-gated, rate-limited job refresh via web scrapers
- ⚡ Redis-backed query caching and abuse-resistant rate limiting
- 🐳 One-command full-stack Docker Compose setup
- ✅ Pytest suite + GitHub Actions CI

---

## Tech Stack

| Layer | Tech |
|---|---|
| Frontend | React, Vite, Tailwind CSS, React Query |
| Backend | FastAPI, SQLAlchemy 2.0 |
| Database | PostgreSQL |
| Cache / rate limiting | Redis |
| AI | Groq API (multi-model ensemble) |
| Job sourcing | Greenhouse, Lever, Ashby APIs + JSearch aggregator |
| Infra | Docker Compose, GitHub Actions |
| Tests | Pytest |

---

## Project Structure

```text
TechHire/
│
├── frontend/              # React frontend (Dockerfile, nginx.conf)
├── scraper/                # Job scraping modules
├── db/                     # Database models, session, save/dedup logic
├── core/                   # Redis client, caching, rate limiting
├── tests/                  # Pytest suite
├── scripts/                 # Seed demo/synthetic data, benchmark
├── .github/workflows/      # CI
├── api.py                  # FastAPI application
├── main.py                 # Scraper runner (CLI)
├── Dockerfile               # Backend image
├── docker-compose.yml       # Full stack: db, redis, api, frontend
├── .env.example
├── DEPLOYMENT.md            # Guide to deploying a live demo
└── README.md
```

---

## Quick Start (Docker — recommended)

```bash
git clone https://github.com/jayminsheladia/TechHire.git
cd TechHire
cp .env.example .env   # then fill in GROQ_API_KEY at minimum
docker compose up --build
```

- Frontend: http://localhost:5173
- Backend: http://localhost:8000 (docs at `/docs`)

This brings up Postgres, Redis, the API, and the frontend together — no local Python/Node setup required.

The database starts empty — no scraper credentials needed to see the app populated:

```bash
python scripts/seed_demo_jobs.py         # 22 realistic hand-written jobs, instant
python scripts/seed_synthetic_jobs.py --count 5000   # for load/benchmark testing
```

---

## Manual Setup (without Docker)

### Prerequisites
- Python 3.10+
- Node.js 18+
- PostgreSQL
- Redis (optional — the app runs without it, just without caching/rate-limiting)

### 1. Backend

```bash
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env            # fill in DATABASE_URL, GROQ_API_KEY, etc.
uvicorn api:app --reload
```

Backend: http://127.0.0.1:8000 · Swagger docs: http://127.0.0.1:8000/docs

### 2. Frontend

```bash
cd frontend
npm install
npm run dev
```

Frontend: http://localhost:5173

### 3. Refresh job listings (optional)

```bash
python main.py
```

---

## Environment Variables

See [.env.example](.env.example) for the full template.

| Variable | Required | Description |
|---|---|---|
| `DATABASE_URL` | Yes | Postgres connection string |
| `GROQ_API_KEY` | For AI features | Free key at [console.groq.com](https://console.groq.com) |
| `RAPIDAPI_KEY` | No | Optional JSearch key for the Indeed/Glassdoor/Handshake keyword-search sources. Not required — `python main.py` already pulls real postings from Greenhouse/Lever/Ashby without it |
| `REDIS_URL` | No | Defaults to `redis://localhost:6379/0`. If unreachable, caching/rate-limiting silently disable — the app still runs |
| `ADMIN_API_KEY` | No | When set, `/scrape/refresh` requires header `X-Admin-Key: <value>`. Unset = open (fine for local dev) |
| `ALLOWED_ORIGINS` | No | Comma-separated CORS allowlist. Add your deployed frontend URL here |

---

## API Endpoints

| Method | Endpoint | Description | Protection |
|---------|----------|-------------|---|
| GET | `/jobs` | Search/filter jobs (paginated) | Cached in Redis, 45s TTL |
| GET | `/jobs/{id}` | Get job details | — |
| GET | `/jobs/{id}/summary` | AI job summary (3-model ensemble) | Cached in DB after first generation |
| POST | `/resume/extract-text` | Extract text from an uploaded PDF/TXT | — |
| POST | `/resume/analyze` | Score a resume against a job description | Rate-limited: 5 req / 10 min / IP |
| POST | `/scrape/refresh` | Trigger an incremental scrape | Admin key (if set) + 15 min cooldown |
| GET | `/scrape/status` | Check scraper status | — |
| GET | `/health` | Health check | — |

---

## Performance

`scripts/benchmark.py` measures `GET /jobs` latency (cache-cold vs cache-warm) and throughput under concurrent load against a running instance:

```bash
docker compose up -d
python main.py                    # real postings via Greenhouse/Lever/Ashby (see below)
python scripts/benchmark.py
```

Three runs, same machine (Docker Compose, Apple M-series laptop), increasing realism:

| Dataset | Cache MISS mean | Cache HIT mean | Speedup | Throughput @ concurrency=40 |
|---|---|---|---|---|
| 22 hand-written demo jobs | 3.40ms | 0.93ms | 3.7x | ~684 req/s |
| 5,022 synthetic jobs | 4.08ms | 1.44ms | 2.8x | ~910 req/s |
| **1,037 real postings** (Greenhouse/Lever/Ashby) | **5.81ms** | **2.81ms** | **2.1x** | **~498 req/s** |
| 1,547 real postings — independent re-run, later date | 5.71ms | 2.65ms | 2.2x | ~417 req/s |

The last row is a reproduction run on a separately scraped dataset (the ATS boards return
different postings on different days), kept here because it is the only evidence that the row
above isn't a one-off. Latency landed within 0.2ms on both measures despite 50% more rows.
Throughput dropped ~16%, consistent with the per-request serialization explanation below —
more rows of multi-KB job descriptions, same fixed cost per response.

The real-data run is the one that matters, and it tells an honest, slightly less flattering story than the synthetic one: throughput went *down* even though row count went *down* too, because real job descriptions are large (full text, arrays of qualifications/responsibilities/benefits — multiple KB per row), so both the Postgres row size and the JSON (de)serialization cost per request are meaningfully higher than synthetic filler data. The cache's relative win also shrank (2.1x vs 2.8x) for the same reason — caching helps less when the *fixed* per-request serialization cost dominates over the *variable* query-execution cost it's actually saving.

Two real next steps this points to, not implemented here: an index (`pg_trgm`/GIN) once row count grows further, and trimming `GET /jobs`'s list-view payload to exclude full `description`/`responsibilities`/`qualifications` (only needed on the single-job detail view) to cut the per-request serialization cost that's now the larger factor.

---

## Architecture Decisions

**Why Redis?** Two jobs: caching `GET /jobs` results (45s TTL, invalidated on refresh) to keep the filter UI snappy under load, and enforcing rate limits on the two expensive/abusable endpoints. The client degrades gracefully — if Redis is down or unset, caching and rate-limiting just turn off instead of crashing the app.

**Why an admin key on `/scrape/refresh`?** Scraping hits third-party APIs with their own quotas. Left fully open on a public deploy, anyone could trigger scrapes on a loop. The key is entered client-side into `localStorage` rather than baked into the frontend build, so it's never shipped in the public JS bundle.

**Why a 3-model ensemble for summaries instead of one call?** Different models surface different details from the same job posting. Running three in parallel with distinct prompt angles, then synthesizing, produces a more complete summary than any single model call — at the cost of ~4x the tokens, which is why the result is cached in Postgres after first generation.

---

## Testing

```bash
# Requires a running Postgres (docker compose up -d ddb, or point DATABASE_URL elsewhere)
export DATABASE_URL=postgresql://techhire:localdev@localhost:5433/techhire_db
pytest -v
```

CI runs this same suite against a fresh Postgres service container on every push/PR — see [.github/workflows/ci.yml](.github/workflows/ci.yml).

---

## Deployment

See [DEPLOYMENT.md](DEPLOYMENT.md) for a step-by-step guide to deploying the frontend (Vercel), backend (Render/Railway), Postgres (Neon), and Redis (Upstash) for free.

---

## Future Improvements

- User authentication and saved jobs
- Resume scoring history / trend tracking
- Job recommendation engine
- Email notifications for new matching listings
- Frontend test coverage

---

## License

This project is intended for educational and learning purposes.

---

## Author

**Jaymin Sheladia**
GitHub: https://github.com/jayminsheladia
