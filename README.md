# TechHire

[![CI](https://github.com/jayminsheladia/TechHire/actions/workflows/ci.yml/badge.svg)](https://github.com/jayminsheladia/TechHire/actions/workflows/ci.yml)

TechHire is a full-stack AI-powered job search platform for CS students and new-grad engineers. It aggregates listings from multiple sources, lets you filter by skill/salary/visa/work-mode, answers questions about the postings with cited sources (RAG), recommends similar roles, generates AI job summaries with a multi-model LangGraph ensemble, and scores your resume against any job description.

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

    PG -. pgvector + full-text .-> RAG[RAG retrieval]
    RAG --> API

    API --> M1[Groq · gpt-oss-120b]
    API --> M2[Groq · Qwen3.8-27B]
    API --> M3[Groq · gpt-oss-20b]
    M1 --> SYN[Synthesis pass]
    M2 --> SYN
    M3 --> SYN
    SYN --> API
```

The AI job-summary endpoint runs three Groq models concurrently, each prompted from a different angle (role/day-to-day, fit/growth, practical/requirements), then synthesizes the three outputs into one summary with a fourth call — reducing single-model bias instead of trusting one model's take. It's a [LangGraph](#langgraph-flows) graph, so a failed model is a routing decision rather than a failed request.

"Ask TechHire" and "similar roles" run on a [RAG index](#rag-ask-techhire--similar-roles) of the postings, stored in the same Postgres via pgvector.

**Job sourcing:** Greenhouse, Lever, and Ashby each publish a public, unauthenticated job-board API per company (`boards-api.greenhouse.io`, `api.lever.co`, `api.ashbyhq.com`) — explicitly meant for external consumption, no ToS issue. `python main.py` pulls real, live postings from ~27 real company boards (Stripe, OpenAI, Databricks, Palantir, Ramp, and others) through these three integrations with zero API key required. JSearch (RapidAPI) is wired in as a fourth, optional source for broader keyword-search coverage if you add a `RAPIDAPI_KEY` — TechHire deliberately does not scrape Indeed/Glassdoor/LinkedIn directly, since that would mean defeating their anti-bot measures to violate their Terms of Service.

---

## Features

- 🔍 Search and browse aggregated job listings
- 🎯 Filter by skills, salary, experience level, work mode, visa sponsorship, date posted
- 💬 "Ask TechHire": questions answered from real postings, every claim cited to a source passage
- 🧭 Similar roles at other companies, by embedding similarity
- 🤖 AI-generated job summaries (3-model LangGraph ensemble + synthesis, degrades gracefully if a model fails)
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
| Database | PostgreSQL + pgvector (HNSW) + full-text search |
| Cache / rate limiting | Redis |
| AI | Groq API, LangGraph (summary ensemble + grounded Q&A graph) |
| Embeddings | BAAI/bge-small-en-v1.5 via fastembed (local, ONNX) |
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
├── ai/                     # LangGraph flows: summary ensemble, grounded Q&A
├── rag/                    # Chunking, embeddings, indexing, retrieval
├── eval/                   # Frozen RAG eval set + latest results
├── tests/                  # Pytest suite
├── scripts/                 # Seed data, benchmark, build RAG index, RAG eval
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
python main.py                      # scrape, then update the RAG index
python scripts/build_rag_index.py   # or just (re)build the RAG index
```

The Postgres server needs the `pgvector` extension available. `docker compose` builds one (see [db/docker/Dockerfile](db/docker/Dockerfile)); for a local install, `brew install pgvector` / `apt install postgresql-15-pgvector`.

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
| GET | `/jobs` | Search/filter jobs (paginated) — trimmed fields, no description/responsibilities/qualifications/benefits | Cached in Redis, 45s TTL |
| GET | `/jobs/{id}` | Full job record, including description/responsibilities/qualifications/benefits | — |
| GET | `/jobs/{id}/summary` | AI job summary (3-model LangGraph ensemble); returns which models ran/failed | Cached in DB after first generation |
| GET | `/jobs/{id}/similar` | Nearest postings by embedding, other companies by default (`same_company=true` to include) | — |
| POST | `/rag/ask` | Grounded Q&A over postings: answer + numbered, cited sources | Rate-limited: 10 req / 10 min / IP |
| GET | `/rag/search` | Retrieval only, no LLM (`mode=dense\|lexical\|hybrid`) — for inspecting retrieval | — |
| POST | `/resume/extract-text` | Extract text from an uploaded PDF/TXT | — |
| POST | `/resume/analyze` | Score a resume against a job description | Rate-limited: 5 req / 10 min / IP |
| POST | `/scrape/refresh` | Trigger an incremental scrape | Admin key (if set) + 15 min cooldown |
| GET | `/scrape/status` | Check scraper status | — |
| GET | `/health` | Health check | — |

---

## Performance

An early benchmark at real-world data volume (1,037 real scraped postings) surfaced two concrete bottlenecks: every filter query did a full table scan (no index covered `title`/`company`/`required_skills`), and the list endpoint shipped each job's full description/responsibilities/qualifications/benefits even though the list UI never renders them. Both are fixed in the code; the numbers below are measured before/after on the **same 50,040-row dataset** (1,040 real Greenhouse/Lever/Ashby postings + 49,000 synthetic, `docker compose` on an Apple M-series laptop), not cherry-picked runs.

### 1. Indexing — the biggest win

`_build_query()`'s `search` and `skills` filters run as `ILIKE '%term%'`, which a plain B-tree index can't accelerate. Added trigram GIN indexes (`pg_trgm`) on `lower(title)`, `lower(company)`, and the skills expression, matching the exact predicates the query already uses — no query-behavior change, purely additive (`db/session.py`):

```sql
CREATE INDEX ix_job_listings_title_trgm   ON job_listings USING gin (lower(title) gin_trgm_ops);
CREATE INDEX ix_job_listings_company_trgm ON job_listings USING gin (lower(company) gin_trgm_ops);
CREATE INDEX ix_job_listings_skills_trgm  ON job_listings USING gin (immutable_array_to_string(required_skills, ',') gin_trgm_ops);
```

(`required_skills` is a Postgres array — indexing `array_to_string(...)` directly isn't allowed since Postgres marks that function `STABLE`, not `IMMUTABLE`; worked around it with a one-line SQL wrapper function declared `IMMUTABLE`, and pointed the app's query at that same wrapper so the planner matches it to the index.)

`EXPLAIN (ANALYZE, BUFFERS)`, identical queries, before vs. after — `Seq Scan` → `Bitmap Index Scan`:

| Query | Before (Seq Scan) | After (Index Scan) | Speedup |
|---|---|---|---|
| `company ILIKE '%stripe%'` (1,669 rows) | 19.6ms | 9.9ms | 2.0x |
| `title ILIKE '%security%'` (2,849 rows) | 15.5ms | 9.7ms | 1.6x |
| `skills ILIKE '%kubernetes%'` (7,446 rows) | 38.3ms | 25.6ms | 1.5x |
| **`company='databricks' AND skill='rust'` (256 rows)** | **47.7ms** | **5.6ms** | **8.5x** |

The single-filter cases are only 1.5–2x because the result sets are large (thousands of rows) — most of the time is spent fetching matching rows off disk (`Bitmap Heap Scan`), which an index can't shrink. The realistic case — a user stacking multiple filters, like the sidebar UI actually lets them do — is the one that matters, and it's the one where a wide table stops requiring a full scan: **8.5x faster** once the index narrows it to a bitmap of 256 rows before touching the heap.

### 2. Trimming the list payload

`GET /jobs` no longer returns `description`/`responsibilities`/`qualifications`/`benefits` — only `GET /jobs/{id}` (the detail view, which is what actually renders them) does. The frontend's `SlideOver` panel now fetches the full record on open instead of assuming the list item already had it.

A single full job record is ~4.3KB; a trimmed list page of 20 jobs is 11.5KB — vs. an estimated ~87KB if all 20 carried full detail. **~87% smaller list payload.**

### 3. Redis caching, at scale

Same cache design as before (45s TTL on `GET /jobs`), now measured against the indexed, trimmed, 50k-row table instead of a 1k-row one:

| | Cache MISS (Postgres) | Cache HIT (Redis) |
|---|---|---|
| Mean latency | 18.57ms | 1.25ms |
| p95 latency | 23.09ms | 1.63ms |

**14.9x faster (93% latency reduction)** — a much bigger win than the 2.1x measured at 1k rows, because the miss path now does real, non-trivial work (even with the index, it's still a bitmap scan + heap fetch over tens of thousands of rows) that the cache is actually saving. This is the more honest, more production-representative number.

### 4. Concurrency sweep

`python scripts/benchmark.py --sweep` fires increasing concurrent load at a warm cache:

| Concurrency | req/s | mean | p95 | p99 |
|---|---|---|---|---|
| 10 | 1,268 | 7.6ms | 15.1ms | 20.1ms |
| 50 | 1,043 | 45.1ms | 56.4ms | 60.0ms |
| 100 | 1,161 | 76.1ms | 95.2ms | 123.7ms |
| 150 | 757 | 174.7ms | 267.5ms | 368.1ms |
| 200 | 927 | 172.0ms | 399.8ms | 412.8ms |
| 300 | 978 | 204.3ms | 379.7ms | 385.8ms |

Throughput holds in the ~1,000–1,200 req/s range through 100 concurrent connections; latency then climbs sharply and throughput gets noticeably noisier beyond that. The most likely cause: `GET /jobs` is a synchronous (`def`, not `async def`) route handler, which FastAPI runs in a bounded worker thread pool (Starlette/anyio's default sync-endpoint limiter caps around 40 threads) — past that, requests queue behind the thread pool instead of the database or Redis, which is consistent with where the curve bends. Not fixed here (would mean an async DB driver, e.g. `asyncpg`), but this is exactly the kind of thing a load test is supposed to surface: not just "is it fast," but "where does it actually break, and why."

### Reproduce it

```bash
docker compose up -d
python main.py                                  # real postings via Greenhouse/Lever/Ashby
python scripts/seed_synthetic_jobs.py --count 49000   # scale up for a meaningful index benefit
python scripts/benchmark.py --sweep
```

---

## RAG: Ask TechHire + similar roles

**Pipeline:** postings → structure-aware chunks → local embeddings → Postgres (pgvector + full-text) → hybrid retrieval → LangGraph answer graph with citation checking.

### Chunking — [rag/chunking.py](rag/chunking.py)

Job postings have structure, so chunks follow it instead of a fixed token window:

- **Profile chunk** built from the structured columns (title, company, level, location, salary, visa, skills), so "remote senior Rust roles with visa sponsorship" can match facts that never appear in the prose.
- **Section split** on the posting's own headings ("What you'll do", "Minimum requirements"...), then paragraphs packed to ~180 words with 40-word overlap. Sentence/word splits only for paragraphs too long to fit.
- **Contextual header** on every chunk before embedding ("ML Engineer at Stripe — Responsibilities"), so a passage like "you'll own the ingestion pipeline" still carries which job it's from.
- **Boilerplate removal**, two ways: legal text (EEO, accommodations, background checks) by regex, and company template text *by frequency*. Any paragraph appearing verbatim in ≥10 postings is dropped. Without that, the same 650-word "Life at Palantir" section on ~190 Palantir postings made up the entire top-k for any culture-ish query. This cut the index from 8,697 to 5,928 chunks.

### Embeddings and storage

- **BAAI/bge-small-en-v1.5** via fastembed (ONNX, CPU): 384-dim, no API key, no per-token cost, and no PyTorch in the image. BGE is asymmetric: queries get an instruction prefix and passages don't, so the two go through different calls.
- **pgvector in the existing Postgres** rather than a separate vector DB: chunks live next to the rows they came from, deletes cascade, and it's one less service to run. HNSW index on cosine distance. At the current ~6k chunks the planner correctly chooses an exact scan; the index takes over as the table grows.
- **Incremental indexing:** each job's chunks are hashed with the chunker version and model name, and only changed jobs are re-embedded. The first build takes ~3.5 min on a laptop CPU; a re-run with no changes takes 0 s. `python main.py` runs it after every scrape.
- Synthetic load-test rows are not indexed, since their templated filler would crowd out real postings.

### Retrieval — [rag/retrieval.py](rag/retrieval.py)

- **Dense:** cosine similarity over chunk embeddings.
- **Lexical:** Postgres full-text search with terms OR-ed (a natural-language question never has all its words in one chunk), ranked with `ts_rank`. The first version used `ts_rank_cd` and took ~650 ms per query. I first patched that with a document-frequency cutoff on query terms, which helped. Profiling then showed the real cost was `ts_rank_cd` itself: cover-density ranking is expensive on a dozen OR-ed terms, and proximity is mostly noise for them. Switching to `ts_rank` was 17x faster *and* more accurate (MRR 0.37 → 0.58), and with it the cutoff measurably *hurt* quality, so I removed it.
- **HNSW recall:** pgvector's default `ef_search` (40), and even 100, lost ~4 points of Hit@1/MRR against exact search on the eval set. 400 matches exact search at ~20 ms, which is negligible next to the LLM call.
- **Hybrid:** Reciprocal Rank Fusion of both. RRF uses only ranks, so a cosine similarity and a `ts_rank` score never have to be put on the same scale.
- **Similar roles:** each job also gets one vector (the mean of its chunk vectors). The "other companies" filter uses pgvector 0.8's iterative index scans. Otherwise HNSW returns its ~40 nearest vectors *before* the filter runs, and for a company with many postings every one of them is removed, leaving nothing. Results default to *other* companies: every chunk header names the company, so same-company postings are always nearest, and "this role elsewhere" is the more useful recommendation. Reposts of one role in several cities are collapsed.

### Evaluation — [scripts/eval_rag.py](scripts/eval_rag.py)

Eval set ([eval/rag_eval_set.json](eval/rag_eval_set.json), frozen): 74 answerable questions and 10 unanswerable ones. For the answerable ones, I sampled real postings (≤4 per company, so OpenAI and Palantir can't dominate), picked one chunk each, and had gpt-oss-20b write the question a job seeker would type that the passage answers, without naming the company or title. Gold = that posting plus its reposts. The unanswerable questions are hand-written, about jobs the corpus doesn't have (nursing, trucking...).

**Retrieval** (`python scripts/eval_rag.py retrieval`, 74 questions, job-level, laptop CPU):

| | Hit@1 | Hit@5 | Hit@10 | MRR@10 | p50 latency |
|---|---|---|---|---|---|
| Dense (bge-small, pgvector) | 0.405 | 0.581 | 0.676 | 0.499 | 56 ms |
| Lexical (Postgres FTS) | **0.554** | 0.730 | 0.838 | 0.640 | 23 ms |
| **Hybrid (RRF)** — production | 0.514 | **0.824** | **0.919** | **0.647** | 85 ms |

Hybrid misses the gold job in its top 10 for 6 of 74 questions, versus 12 for lexical and 24 for dense. Lexical alone beats dense here, which I'd attribute partly to the eval-set bias below: generated questions can echo their passage's wording. Hybrid is the better bet on real queries either way, because it doesn't depend on which retriever happens to be stronger for a given question. Latency includes embedding the query on CPU (~30 ms).

**Chunking ablation** (`python scripts/eval_rag.py ablation`: dense retrieval, exact search, same model and questions, only the chunking differs):

| Chunking | Hit@1 | Hit@5 | Hit@10 | MRR@10 | Chunks |
|---|---|---|---|---|---|
| **Sections + contextual header** (production) | **0.405** | **0.581** | **0.676** | **0.499** | 5,928 |
| Sections, no header | 0.243 | 0.405 | 0.527 | 0.334 | 5,928 |
| Fixed 200-word windows | 0.176 | 0.284 | 0.392 | 0.222 | 5,784 |
| Whole posting, one vector | 0.189 | 0.351 | 0.473 | 0.256 | 1,040 |

The contextual header is worth about as much as the section-aware splitting itself. Without it, many chunks look alike ("5+ years of Python...") and nothing ties a chunk to its role.

**End-to-end** (`python scripts/eval_rag.py answers --n 15`: the real LangGraph answer graph and Groq, over 15 sampled answerable questions plus all 10 unanswerable ones):

| Metric | Result |
|---|---|
| Answerable → answered with valid citations | 15 / 15 |
| Answerable → wrongly abstained | 0 / 15 |
| Gold posting among the 8 retrieved sources | 13 / 15 (87%) |
| Gold posting actually cited | 12 / 15 (80%); 12 of the 13 where it was retrieved |
| Unanswerable → correctly abstained | 10 / 10 |
| Needed the citation-retry loop | 0 / 25 |

A small sample, so read it as "the mechanics work", not as a precise rate. Most misses are retrieval misses (the gold posting never reached the model), not generation errors. The retry loop didn't fire in this run: since citations are normalized first, the failure it guards against (invalid `[n]` references) is now rare. It's covered by unit tests with a scripted fake LLM instead.

**Caveats, stated plainly:** questions generated from one passage are narrower than real queries, and some are generic enough ("benefits for a senior SWE at a large tech company") that many postings fit but only one counts as gold. So read these as *relative* comparisons between configurations, not absolute quality. The choices above (ranker, HNSW `ef_search`, removing the cutoff) were all made by measuring on this same set. They're coarse, few in number and backed by clear mechanisms, but it is still tuning on the test data. A held-out split would be the next step. Citation checking verifies that every `[n]` points at a real source. It does **not** verify that the source supports the sentence; that would need an LLM judge and isn't measured.

---

## LangGraph flows

Both multi-step LLM flows are LangGraph graphs in [ai/](ai/), and both take the LLM as a function argument, so the tests drive every branch with a fake LLM and no API key.

### Summary ensemble — [ai/summary_graph.py](ai/summary_graph.py)

```text
        ┌─► draft (gpt-oss-120b) ─┐
START ──┼─► draft (qwen3.8-27b)  ─┼─► collect ─┬─► synthesize ─► END   (≥2 drafts ok)
        └─► draft (gpt-oss-20b)  ─┘            ├─► use_single ─► END   (1 ok)
                                               └─► fail ───────► END   (0 ok)
```

- **Fan-out with `Send`:** one branch per entry in `VARIANTS`, so adding a model is a list edit. The branches run concurrently (verified in a test: 3 parallel drafts plus synthesis take ~2 calls' time, not 4).
- **`operator.add` reducer** on `drafts`, so the parallel branches append to one list instead of overwriting each other.
- **Partial failure is routing, not an exception.** The previous `asyncio.gather` version failed the whole request if any one model failed, and Groq has retired models this project used before (404 on every call). Now each draft records its own error, and the graph synthesizes from whatever succeeded. If synthesis itself fails, it falls back to the best draft. The API response says which models ran, how long each took, and which path produced the summary.
- **This happened for real during testing:** Groq retired `qwen/qwen3.6-27b` (404 `model_not_found` on every call). The graph still returned a synthesized summary from the other two models in 1.8 s, and the response showed exactly which model failed and why. The `asyncio.gather` version would have failed the endpoint for every uncached job. The model is now `qwen/qwen3.8-27b`.
- **A LangGraph detail I got wrong first:** a conditional edge placed directly after `Send` branches is evaluated *per branch*, and each branch sees only its own draft. The first draft to finish routed to "single draft" with 1 of 3 results. The fix is the no-op `collect` node: a plain edge into it fires once, after all branches have written state.

### Grounded Q&A — [ai/answer_graph.py](ai/answer_graph.py)

```text
START ─► retrieve ─┬─► generate ─┬─► check ─┬─► finalize ─► END
                   │       ▲     │          │
                   │       └─────┼─ retry ──┘   invalid citations, attempts left
                   │             └─► finalize   LLM error (e.g. rate limit)
                   └─► finalize                 nothing retrieved
```

- **retrieve:** hybrid search, max 2 chunks per job, 8 sources total.
- **generate:** numbered sources, with every factual sentence required to cite `[n]`. If nothing is relevant, the model must reply `INSUFFICIENT`, so abstaining is a detectable state rather than a paraphrase to guess at. Sources are marked as untrusted data (they're scraped text) to blunt prompt injection.
- **check:** deterministic. Every `[n]` must exist, and a non-abstaining answer must cite something. On failure, the model gets its own answer back plus the exact problems, for one retry. This loop is the part that actually needs a graph.
- **finalize:** `answered` / `abstained` / `no_results` / `error` / `ungrounded` (still failing after the retry: returned, but flagged in the UI instead of presented as sourced).
- **Found in live testing:** gpt-oss sometimes cites in its native `【1†L3-L5】` style despite the prompt. Those answers were properly sourced but failed validation and ended up flagged ungrounded, so citations are normalized to `[n]` before checking. A two-part question whose sources covered only one part made the model abstain entirely, so the prompt now says to answer the covered part and state what's missing.

---

## Architecture Decisions

**Why Redis?** Two jobs: caching `GET /jobs` results (45s TTL, invalidated on refresh) to keep the filter UI snappy under load, and enforcing rate limits on the two expensive/abusable endpoints. The client degrades gracefully — if Redis is down or unset, caching and rate-limiting just turn off instead of crashing the app.

**Why an admin key on `/scrape/refresh`?** Scraping hits third-party APIs with their own quotas. Left fully open on a public deploy, anyone could trigger scrapes on a loop. The key is entered client-side into `localStorage` rather than baked into the frontend build, so it's never shipped in the public JS bundle.

**Why a 3-model ensemble for summaries instead of one call?** Different models surface different details from the same job posting. Running three in parallel with distinct prompt angles, then synthesizing, produces a more complete summary than any single model call — at the cost of ~4x the tokens, which is why the result is cached in Postgres after first generation.

**Why does every Groq call pass `reasoning_effort`/`reasoning_format`?** Groq's currently available chat models (`gpt-oss-*`, `qwen3.8-27b`) are reasoning models by default — left alone, they spend `max_tokens` on invisible chain-of-thought and can return empty content for a summarization task that doesn't need it. `_reasoning_kwargs()` in `api.py` pins each model family to minimal/no reasoning so `max_tokens` goes to the actual answer. Found this the hard way: the code originally targeted `llama-3.3-70b-versatile` and two other models Groq has since retired entirely (404 on every call) — worth knowing if you're maintaining this, since Groq's free-tier model lineup changes over time and whatever's hardcoded here may need updating again later.

---

## Testing

```bash
# Requires a running Postgres (docker compose up -d ddb, or point DATABASE_URL elsewhere)
export DATABASE_URL=postgresql://techhire:localdev@localhost:5433/techhire_db
pytest -v
```

Tests never touch that database directly — `tests/conftest.py` derives a dedicated `<dbname>_test` database (auto-created on first run) from `DATABASE_URL` and runs there instead, specifically so `pytest` can't wipe real local/scraped data by creating and dropping tables freely. Override with `TEST_DATABASE_URL` if you want it to point somewhere else.

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
- LLM-as-judge faithfulness check for RAG answers (citations are verified to exist, not yet to support their sentence)
- Cross-encoder reranking of the fused candidates before they reach the LLM
- Async DB driver (`asyncpg`) — the concurrency sweep in [Performance](#performance) points to the sync-endpoint thread pool as the next real bottleneck past ~100 concurrent requests

---

## License

This project is intended for educational and learning purposes.

---

## Author

**Jaymin Sheladia**
GitHub: https://github.com/jayminsheladia
