# Systematic Literature Review Tool — Phased Development Plan

> Project: 278-systematic-literature-review-tool · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and `data-model-suggestion-3.md` (Hybrid Relational + JSONB, the chosen base schema, augmented with the RoB 2 / GRADE / PRISMA structures from `data-model-suggestion-1.md`). All four research foundations are present, so the plan covers competitive feature scope, methodological standards (PRISMA 2020, Cochrane RoB 2, GRADE, AMSTAR-2), citation formats (RIS, BibTeX, NBIB), and bibliographic database APIs (PubMed E-utilities, OpenAlex, Crossref, Semantic Scholar).

---

## Product Summary

An AI-native, open-source, browser-based platform for conducting PRISMA 2020-compliant systematic literature reviews. It covers the full evidence-synthesis pipeline: protocol/PICO definition, multi-database search, citation import and semantic deduplication, dual-phase screening (title/abstract → full text) with active-learning AI prioritisation, customisable AI-assisted data extraction from PDFs, Cochrane RoB 2 assessment, GRADE evidence profiling, meta-analysis, automated PRISMA flow-diagram and checklist generation, and living-review monitoring. The differentiator versus Covidence/Rayyan/DistillerSR is that AI augmentation (active-learning screening, structured PDF extraction, semantic dedup, evidence-gap mapping) is built into the core rather than bolted on, and the entire workflow is reproducible and auditable.

**Primary personas:** clinical/health researchers (Cochrane-style reviews), HTA teams (NICE, CADTH, IQWiG), environmental/policy evidence-synthesis researchers, pharma/medtech regulatory affairs, academics fulfilling journal requirements.

**Deployment model:** Self-hostable via Docker Compose (single-tenant) and multi-tenant SaaS from the same codebase. API-first (OpenAPI 3.1), with an optional MCP server exposing search/screening/extraction as agent tools.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12 | The product is LLM- and ML-heavy (active-learning screening, PDF extraction, semantic dedup, meta-analysis stats). Python has the dominant ecosystem: scikit-learn, sentence-transformers, PyMuPDF, statsmodels, Biopython (Entrez). ASReview — the closest open-source reference — is Python. |
| API framework | FastAPI | Native async (needed for concurrent calls to PubMed/OpenAlex/Crossref and LLM providers), Pydantic v2 request/response validation, and automatic OpenAPI 3.1 generation (a `standards.md` requirement). |
| ASGI server | Uvicorn (dev) / Gunicorn+Uvicorn workers (prod) | Standard FastAPI deployment. |
| Database | PostgreSQL 16 | The chosen Hybrid Relational + JSONB model (data-model-suggestion-3) relies on JSONB + GIN indexes for flexible extraction/RoB/GRADE templates while keeping core entities relational. `pgvector` extension provides embedding storage for semantic dedup and screening features. Full-text search via `tsvector`. |
| Vector storage | pgvector extension | Avoids a second datastore; sufficient for per-review corpora (typically <100k records). Stores sentence-transformer embeddings for semantic dedup and active-learning features. |
| ORM / migrations | SQLAlchemy 2.0 (async) + Alembic | Mature async ORM; Alembic gives versioned, reviewable migrations (one per phase). |
| Task queue | Celery + Redis | Long-running async work (PDF text extraction, embedding generation, batch LLM extraction, database harvesting, living-review cron) must run off the request path. Redis doubles as broker and result backend. |
| Scheduler | Celery Beat | Drives living-review periodic searches. |
| LLM access | LiteLLM gateway | Provider-agnostic abstraction over OpenAI/Anthropic/local models for extraction and screening assistance. Configurable per deployment; self-hosters can point at a local model. |
| Embeddings | sentence-transformers (`all-MiniLM-L6-v2` default) | Local, no per-call cost, deterministic — important for reproducible semantic dedup. Pluggable for higher-quality models. |
| Active-learning classifier | scikit-learn (TF-IDF + logistic regression / SVM), pluggable | Mirrors ASReview's proven default; fast to retrain per decision; transparent and auditable. Embedding-based features optional. |
| PDF parsing | PyMuPDF (text + layout) + pdfplumber (tables) | PyMuPDF for fast text/coordinates (annotation anchoring); pdfplumber for table extraction feeding AI extraction. |
| Meta-analysis stats | statsmodels + NumPy/SciPy | Fixed/random-effects pooling, I², τ², forest/funnel plot data. No need for an R bridge for MVP scope. |
| Citation parsing | `rispy` (RIS), `bibtexparser` (BibTeX), custom NBIB parser | Per `standards.md`, RIS/BibTeX/NBIB import-export is mandatory. |
| Auth | Authlib (OAuth 2.0 / OIDC) + local password (Argon2 via `passlib`) | `standards.md` requires OAuth 2.0 (RFC 6749) and OIDC for institutional SSO and database-subscription delegation; local auth for self-hosters. |
| Frontend | React 18 + TypeScript + Vite | Browser-based collaborative workflows are a must-have differentiator; screening and PDF annotation need a rich SPA. |
| UI components | Tailwind CSS + shadcn/ui | Fast, accessible, consistent component layer. |
| PDF viewer (frontend) | PDF.js (react-pdf) | Renders PDFs with a text/overlay layer for highlight annotations linked to extraction fields. |
| Charts | Recharts | Forest plots, funnel plots, evidence-gap heatmaps, PRISMA diagram rendering. |
| Realtime | WebSockets (FastAPI) + Redis pub/sub | Collaborative screening/extraction presence and live counts. |
| Object storage | S3-compatible (MinIO self-host / AWS S3 SaaS) via `boto3` | PDF and export artefact storage; keyed by content hash. |
| Containerisation | Docker + Docker Compose | Single-command self-host (`docker compose up`): api, worker, beat, postgres+pgvector, redis, minio, web. |
| Testing | pytest + pytest-asyncio + httpx + factory_boy + Playwright | Unit/integration for backend; Playwright for frontend E2E of screening and extraction flows. |
| Code quality | Ruff (lint+format), mypy (types) — Python; ESLint + Prettier + tsc — frontend | Enforced in CI and in each phase's Definition of Done. |
| Package managers | uv (Python), pnpm (frontend) | Fast, reproducible lockfiles. |
| CI | GitHub Actions | Lint, type-check, test matrix, Docker build per PR. |

### Project Structure

```
slr-tool/
├── README.md
├── docker-compose.yml
├── docker-compose.dev.yml
├── .github/workflows/ci.yml
├── backend/
│   ├── pyproject.toml
│   ├── uv.lock
│   ├── alembic.ini
│   ├── Dockerfile
│   ├── alembic/
│   │   └── versions/                  # one migration per phase
│   ├── src/slr/
│   │   ├── main.py                    # FastAPI app factory, router registration
│   │   ├── config.py                  # Pydantic Settings (env-driven)
│   │   ├── db.py                      # async engine, session factory
│   │   ├── deps.py                    # FastAPI dependencies (db session, current user)
│   │   ├── security.py                # password hashing, JWT, OAuth/OIDC
│   │   ├── models/                    # SQLAlchemy ORM models (one module per domain)
│   │   │   ├── base.py
│   │   │   ├── org.py  user.py  review.py  reference.py
│   │   │   ├── screening.py  document.py  extraction.py
│   │   │   ├── rob.py  grade.py  meta.py  prisma.py  living.py  audit.py
│   │   ├── schemas/                   # Pydantic request/response models
│   │   ├── api/                       # FastAPI routers (one per resource)
│   │   │   ├── auth.py  reviews.py  references.py  imports.py
│   │   │   ├── screening.py  documents.py  extraction.py
│   │   │   ├── rob.py  grade.py  meta.py  prisma.py  exports.py  living.py  ws.py
│   │   ├── services/                  # business logic (testable, framework-free)
│   │   │   ├── dedup.py  active_learning.py  pdf_extract.py
│   │   │   ├── ai_extract.py  prisma.py  meta_analysis.py  exports.py
│   │   ├── integrations/              # external bibliographic DB clients
│   │   │   ├── base.py  pubmed.py  openalex.py  crossref.py  semantic_scholar.py  arxiv.py
│   │   ├── parsers/                   # citation format parsers
│   │   │   ├── ris.py  bibtex.py  nbib.py  detect.py
│   │   ├── llm.py                     # LiteLLM wrapper, prompt templates
│   │   ├── tasks/                     # Celery tasks
│   │   │   ├── celery_app.py  ingest.py  embeddings.py  extraction.py  living.py
│   │   ├── mcp/                       # MCP server exposing tools (final phase)
│   │   └── audit.py                   # audit-log helper
│   └── tests/
│       ├── conftest.py  factories.py
│       ├── unit/  integration/  fixtures/   # sample RIS/NBIB/PDF files
├── frontend/
│   ├── package.json  pnpm-lock.yaml  vite.config.ts  Dockerfile
│   ├── src/
│   │   ├── main.tsx  App.tsx  api/client.ts (generated from OpenAPI)
│   │   ├── pages/  components/  hooks/  stores/
│   │   └── features/{auth,reviews,search,screening,extraction,rob,grade,synthesis,prisma}/
│   └── tests/e2e/                     # Playwright specs
└── docs/
    └── openapi.json                   # exported spec, regenerated in CI
```

The backend separates **services** (pure business logic, fully unit-testable) from **api** (thin HTTP routers) and **integrations/parsers** (I/O adapters). Each phase adds modules without restructuring.

---

## Phase 1: Foundation — Project Skeleton, Auth, Tenancy

### Purpose
Establish a running, testable, deployable skeleton: containerised FastAPI + PostgreSQL + Redis + React, configuration management, the database session layer, migrations, authentication (local + OIDC-ready), and multi-tenant organisation/user/review-membership models. After this phase a user can register, log in, create an organisation, and create an empty review — and CI runs lint, types, and tests on every PR.

### Tasks

#### 1.1 — Repository scaffolding and tooling

**What**: Create the monorepo structure, dependency manifests, Docker Compose stack, and CI pipeline.

**Design**:
- `backend/pyproject.toml` declares deps (fastapi, uvicorn, sqlalchemy[asyncio], asyncpg, alembic, pydantic-settings, authlib, passlib[argon2], celery, redis, pytest, ruff, mypy).
- `config.py` using `pydantic_settings.BaseSettings`:
  ```python
  class Settings(BaseSettings):
      database_url: str
      redis_url: str = "redis://redis:6379/0"
      jwt_secret: str
      jwt_algorithm: str = "HS256"
      access_token_ttl_min: int = 60
      refresh_token_ttl_days: int = 30
      s3_endpoint: str | None = None
      s3_bucket: str = "slr-documents"
      llm_provider: str = "openai"          # passed to LiteLLM
      llm_model: str = "gpt-4o-mini"
      embedding_model: str = "all-MiniLM-L6-v2"
      single_tenant: bool = False           # self-host mode auto-creates default org
      model_config = SettingsConfigDict(env_prefix="SLR_")
  ```
- `docker-compose.yml` services: `postgres` (image `pgvector/pgvector:pg16`), `redis`, `minio`, `api`, `worker`, `beat`, `web`. Healthchecks on postgres/redis; api waits for healthy db.
- `db.py`: `create_async_engine`, `async_sessionmaker`, `get_session` dependency.
- `.github/workflows/ci.yml`: jobs `lint` (ruff check + format --check), `types` (mypy), `test` (pytest with a Postgres service container), `docker-build`.

**Testing**:
- `Unit: Settings loads from env with SLR_ prefix → correct typed fields, defaults applied`
- `Unit: GET /health → 200 {"status":"ok","db":"ok","redis":"ok"}` (checks connectivity)
- `Integration: docker compose config validates` (CI lints the compose file)
- `CI: ruff, mypy, pytest all green on empty skeleton`

#### 1.2 — Core ORM base, migrations, audit log

**What**: SQLAlchemy declarative base with UUID PKs and timestamp mixins, the initial Alembic migration, and an append-only audit-log table.

**Design**:
- `models/base.py`:
  ```python
  class Base(DeclarativeBase): ...
  class UUIDPK:
      id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
  class Timestamps:
      created_at: Mapped[datetime] = mapped_column(server_default=func.now())
      updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())
  ```
- `models/audit.py` mirrors `audit_log` from data-model-suggestion-1 (review_id, user_id, action, entity_type, entity_id, old_value, new_value JSONB, ip_address, created_at). Helper `audit.record(session, *, action, entity_type, entity_id, old, new, actor, review_id)`.
- Alembic env configured for async engine; first migration enables `pgvector` and `pg_trgm` extensions and creates `audit_log`.

**Testing**:
- `Unit: audit.record persists row with serialised old/new JSON`
- `Integration: alembic upgrade head then downgrade base runs cleanly against test DB`
- `Integration: pgvector and pg_trgm extensions present after migration`

#### 1.3 — Organisation, user, membership models and auth

**What**: Implement `organisation`, `app_user`, `organisation_member` (from data-model-suggestion-1) plus registration, login, token refresh, and OIDC login.

**Design**:
- ORM models per the suggestion-1 DDL (org with `subscription_tier`; user with `orcid_id`, `auth_provider`, `auth_subject`, `password_hash`; membership with `role` enum owner/admin/member, UNIQUE(org,user)).
- `security.py`: Argon2 hashing; JWT access+refresh issuance/verification; `get_current_user` dependency decoding the bearer token; `require_role(review_id, min_role)` dependency.
- Auth endpoints:
  ```
  POST /auth/register   {email, password, display_name} → 201 {user}
  POST /auth/login      {email, password} → 200 {access_token, refresh_token}
  POST /auth/refresh    {refresh_token} → 200 {access_token}
  GET  /auth/me         → 200 {user, organisations[]}
  GET  /auth/oidc/login?provider=… → 302 to IdP   (Authlib)
  GET  /auth/oidc/callback → 200 {access_token,…}  (creates/links user by auth_subject)
  POST /orgs            {name, slug} → 201 {org}   (creator becomes owner)
  POST /orgs/{id}/members {email, role} → 201
  ```
- Single-tenant mode: on first boot, auto-create a `default` org and assign the first registered user as owner.

**Testing**:
- `Unit: password hash/verify round-trip; wrong password → False`
- `Unit: JWT issue/decode round-trip; expired token → 401`
- `Integration: register → login → /auth/me returns user with org`
- `Integration: duplicate email register → 409`
- `Integration: login wrong password → 401`
- `Integration (mocked IdP): oidc callback with valid id_token → user created, linked by auth_subject`
- `Integration: non-member requests org resource → 403`

#### 1.4 — Review entity and membership

**What**: `review`, `review_team_member`, `review_protocol`, and `review_question` (PICO) models with CRUD endpoints.

**Design**:
- Models per suggestion-1: review (`review_type`, `status` lifecycle enum `protocol|searching|screening|extraction|synthesis|reporting|published|archived`), team member roles (`lead|reviewer|screener|extractor|statistician|observer`), protocol (PROSPERO id, eligibility, search strategy text), review_question with explicit P/I/C/O columns.
- Endpoints:
  ```
  POST   /reviews                         {title, review_type} → 201
  GET    /reviews                         → list (scoped to user's orgs)
  GET    /reviews/{id}                    → detail (requires membership)
  PATCH  /reviews/{id}                    {title?, status?, description?}
  POST   /reviews/{id}/members            {user_id|email, role}
  PUT    /reviews/{id}/protocol           {prospero_id?, eligibility_criteria?, …}
  POST   /reviews/{id}/questions          {question_text, population, intervention, comparator, outcome}
  GET    /reviews/{id}/questions
  ```
- Status transitions validated by a small state machine (only forward transitions plus archive; raises 422 on illegal move).

**Testing**:
- `Unit: status transition protocol→searching allowed; published→protocol rejected`
- `Integration: create review → creator added as lead team member`
- `Integration: add PICO question → retrievable with P/I/C/O fields`
- `Integration: user not on review team → GET /reviews/{id} → 403`

---

## Phase 2: References, Citation Import & Bibliographic Database Search

### Purpose
Give a review a corpus to work on. Implement the `reference` model and supporting tables, robust import of RIS/BibTeX/NBIB/CSV files (the de-facto exchange standards from `standards.md`), and live search against free bibliographic APIs (PubMed E-utilities, OpenAlex, Crossref, Semantic Scholar, arXiv). After this phase, a reviewer can run a documented search, import thousands of records, and see them listed — with full search-strategy provenance for PRISMA.

### Tasks

#### 2.1 — Reference and supporting models

**What**: Implement `reference`, `reference_author`, `reference_keyword`, `search_source`, `search_execution`, `citation_import`.

**Design**:
- `reference` per suggestion-1/3: identifier columns (doi, pmid, pmcid, openalex_id, semantic_scholar_id, arxiv_id), bibliographic fields, `import_source_id`, `import_batch_id`, dedup columns (`dedup_cluster_id`, `is_duplicate`, `master_ref_id`), and a `search_vector tsvector` GENERATED column over title+abstract for full-text search (GIN indexed). Add an `embedding vector(384)` column (nullable, populated in Phase 4).
- Indexes: review_id, partial unique-ish indexes on doi/pmid, GIN on search_vector, GIN trgm on title for fuzzy match.
- `search_execution` records query string, source, results_count, executed_by, executed_at — this is the PRISMA "Identification: records from databases/registers" provenance.

**Testing**:
- `Unit: reference search_vector populated from title+abstract on insert`
- `Integration: create reference, full-text query 'cognitive AND therapy' matches`
- `Integration: cascade delete review removes its references and authors`

#### 2.2 — Citation file parsers (RIS, BibTeX, NBIB, CSV)

**What**: Format-detecting parsers that normalise heterogeneous citation files into reference rows.

**Design**:
- `parsers/detect.py: detect_format(content: bytes, filename: str) -> Literal["ris","bibtex","nbib","csv"]` using extension + content sniffing (`TY  -` for RIS, `@` for BibTeX, `PMID-`/`FAU -` for NBIB).
- Each parser exposes `parse(content: str) -> list[ParsedReference]` where:
  ```python
  @dataclass
  class ParsedReference:
      title: str; abstract: str | None; authors: list[ParsedAuthor]
      journal: str | None; volume: str | None; issue: str | None
      pages: str | None; publication_year: int | None
      doi: str | None; pmid: str | None; document_type: str | None
      language_code: str | None; keywords: list[str]; raw: dict
  ```
- NBIB parser maps NCBI two-letter tags (TI, AB, FAU, AU, DP, LID/AID for DOI, PMID).
- Mapping tables for RIS `TY`/NBIB `PT` → internal `document_type`.

**Testing**:
- `Unit: detect_format on sample RIS/BibTeX/NBIB/CSV fixtures → correct label`
- `Fixture: parse pubmed_50.nbib → 50 ParsedReference, DOIs and PMIDs populated`
- `Fixture: parse scopus_export.ris → authors split into family/given`
- `Unit: malformed RIS entry → skipped, error collected, others still parsed`
- `Unit: BibTeX with LaTeX-escaped chars → unescaped in title`

#### 2.3 — Import endpoint and batch tracking

**What**: Upload-and-import endpoint producing a `citation_import` batch and persisting references.

**Design**:
```
POST /reviews/{id}/imports   (multipart: file, source_id?)
  → 202 {import_batch_id}  (parsing >N records dispatched to Celery)
GET  /reviews/{id}/imports/{batch_id}  → {records_total, records_imported, records_duplicate, status}
GET  /reviews/{id}/references?page=&size=&q=&status=  → paginated list
```
- Small files parse synchronously; large files dispatch `tasks/ingest.import_citations`. Within-batch exact-duplicate suppression by (doi) then (pmid) then normalised-title hash. Cross-batch semantic dedup is Phase 4.
- Records linked to `search_execution` when `source_id` provided; PRISMA identification counts derive from these.

**Testing**:
- `Integration: POST 50-record NBIB → import batch, 50 references, records_imported=50`
- `Integration: re-import same file → records_duplicate counts exact DOI matches, no new rows`
- `Integration (mocked Celery eager): 5000-record file → 202, batch completes, count correct`
- `Integration: list references paginated, q filter narrows results`

#### 2.4 — Bibliographic database integration clients

**What**: Async clients for PubMed E-utilities, OpenAlex, Crossref, Semantic Scholar, arXiv behind a common interface, plus a search endpoint.

**Design**:
- `integrations/base.py`:
  ```python
  class BiblioClient(Protocol):
      name: str
      async def search(self, query: str, *, limit: int, offset: int) -> SearchPage: ...
  @dataclass
  class SearchPage:
      total: int; results: list[ParsedReference]; next_offset: int | None
  ```
- `pubmed.py` uses esearch→efetch (Biopython or raw httpx); respects 3 req/s (10 with key). `openalex.py` adds polite-pool `mailto`. `crossref.py` polite pool. `semantic_scholar.py` optional API key. `arxiv.py` Atom XML.
- All clients implement retry with exponential backoff and per-source rate limiting (token bucket in Redis).
- Endpoint:
  ```
  POST /reviews/{id}/search   {source, query, limit} → {total, preview: ParsedReference[]}
  POST /reviews/{id}/search/import {source, query, max_records} → 202 {import_batch_id}
        (harvests all pages via Celery, records a search_execution)
  ```

**Testing**:
- `Integration (mocked HTTP via respx): pubmed search → SearchPage with parsed refs`
- `Integration (mocked): openalex pagination cursor followed to completion`
- `Unit: rate limiter blocks 4th PubMed call within 1s when no key configured`
- `Integration (mocked): search/import harvests 3 pages → search_execution.results_count correct`
- `Integration (real, marked optional/network): live OpenAlex query returns ≥1 result`

---

## Phase 3: Screening Workflow (Manual)

### Purpose
Deliver the methodological heart of a systematic review: dual-phase screening with multiple independent reviewers, conflict detection and resolution, exclusion-reason tracking, and assignment. This phase ships the manual workflow end-to-end (AI prioritisation layers on in Phase 5) so reviewers can already complete a full PRISMA-compliant screen.

### Tasks

#### 3.1 — Screening models and decision recording

**What**: `screening_phase`, `screening_decision`, `exclusion_reason`, `screening_conflict` with decision recording and automatic conflict detection.

**Design**:
- Models per suggestion-1. `screening_phase` has `phase_type` (`title_abstract|full_text`), `required_reviewers` (default 2), `conflict_resolution` (`consensus|third_reviewer|lead_decides`).
- `screening_decision` UNIQUE(phase, reference, reviewer); `decision` ∈ `include|exclude|maybe`; optional `exclusion_reason_id` and notes.
- On each decision, a service `record_decision` recomputes reference state:
  - states: `pending` (fewer than required decisions) → `included` (all agree include) → `excluded` (all agree exclude) → `conflicted` (disagreement among required reviewers).
  - On conflict, upsert a `screening_conflict(status=unresolved)`.
- Endpoints:
  ```
  POST /reviews/{id}/screening/phases   {phase_type, required_reviewers, conflict_resolution}
  POST /reviews/{id}/screening/{phase_id}/decisions
        {reference_id, decision, exclusion_reason_id?, notes?} → {reference_state}
  GET  /reviews/{id}/screening/{phase_id}/queue?assigned_to=me  → next undecided references for the reviewer
  GET  /reviews/{id}/screening/{phase_id}/conflicts
  POST /reviews/{id}/screening/{phase_id}/conflicts/{ref_id}/resolve {resolution, notes}
  ```

**Testing**:
- `Unit: two 'include' decisions (required=2) → state included`
- `Unit: include + exclude → state conflicted, conflict row created`
- `Unit: full_text exclude requires exclusion_reason → 422 if missing`
- `Integration: resolve conflict → state set, conflict marked resolved, audit logged`
- `Integration: reviewer cannot submit two decisions on same reference (unique violation → 409)`

#### 3.2 — Exclusion reasons and per-review config

**What**: Per-review exclusion-reason catalogue with sensible PRISMA defaults.

**Design**:
- `exclusion_reason(review_id, code, label, phase_type, sort_order)`; seeded defaults on review creation (`wrong_population`, `wrong_intervention`, `wrong_comparator`, `wrong_outcome`, `wrong_study_design`, `duplicate`, `full_text_unavailable`).
- CRUD endpoints under `/reviews/{id}/exclusion-reasons`. Full-text-phase exclusions feed the PRISMA "reports excluded, with reasons" box.

**Testing**:
- `Integration: new review seeded with default exclusion reasons`
- `Integration: add custom reason → usable in a full_text exclusion`

#### 3.3 — Screening assignment and progress

**What**: Assign references to reviewers and report progress.

**Design**:
- Assignment strategy `auto` (round-robin undecided refs across screeners ensuring `required_reviewers` distinct reviewers per ref) or `manual`.
- `GET /reviews/{id}/screening/{phase_id}/progress` → `{total, decided, pending, conflicted, per_reviewer:{user_id:count}}`.

**Testing**:
- `Unit: round-robin assigns each reference to exactly required_reviewers distinct screeners`
- `Integration: progress endpoint counts match decisions made`

#### 3.4 — Screening UI (frontend)

**What**: A keyboard-driven screening interface and conflict-resolution view.

**Design**:
- `features/screening`: queue view showing one reference at a time (title, abstract, journal, year) with Include / Exclude / Maybe buttons and keyboard shortcuts (`i`/`e`/`m`); exclusion-reason picker on full-text exclude; progress bar; conflict inbox listing conflicted refs with both reviewers' decisions and a resolve action.
- API client generated from OpenAPI.

**Testing**:
- `E2E (Playwright): screen 5 refs with keyboard → decisions persisted, queue advances`
- `E2E: full-text exclude prompts for reason; cannot submit without one`
- `E2E: conflict appears in inbox; lead resolves → removed from inbox`

---

## Phase 4: Deduplication & Document Management

### Purpose
Remove duplicate records arriving from multiple databases (a core PRISMA identification step and a stated AI-native differentiator) and let reviewers upload, store, and annotate full-text PDFs ahead of extraction. This phase introduces embeddings (pgvector), enabling semantic dedup now and active-learning features in Phase 5.

### Tasks

#### 4.1 — Embedding generation pipeline

**What**: Generate and store sentence-transformer embeddings for every reference.

**Design**:
- `tasks/embeddings.embed_references(reference_ids)` loads `all-MiniLM-L6-v2`, encodes `title + " " + abstract`, writes to `reference.embedding`. Triggered after each import batch.
- Batched (default 256) for throughput; idempotent (skips refs already embedded with the same model version, tracked in a small `embedding_meta` column or table).

**Testing**:
- `Unit: embed text → 384-dim vector`
- `Integration: import → embeddings task enqueued → references have embeddings`
- `Integration: re-running embed task is idempotent`

#### 4.2 — Deduplication service

**What**: Multi-strategy dedup: exact identifier match, then fuzzy title/author, then semantic similarity; cluster and flag duplicates.

**Design**:
- `services/dedup.find_duplicates(review_id) -> list[DuplicateCluster]`:
  1. Exact: group by normalised DOI, then PMID.
  2. Fuzzy: `pg_trgm` similarity on normalised title ≥ 0.85 AND year match AND first-author surname match.
  3. Semantic: pgvector cosine distance < 0.10 among remaining, as a candidate flag for human review.
  - Assign `dedup_cluster_id`; choose master (richest metadata: prefers record with DOI+abstract); mark others `is_duplicate=true`, set `master_ref_id`.
- Endpoints:
  ```
  POST /reviews/{id}/dedup/run            → 202 {cluster_count, auto_marked}
  GET  /reviews/{id}/dedup/clusters?status=pending
  POST /reviews/{id}/dedup/clusters/{cid}/decide {master_ref_id, duplicates:[ref_id]}
  ```
- Semantic matches require human confirmation by default (`auto_screen=false`); exact/strong-fuzzy auto-marked.

**Testing**:
- `Unit: identical DOI different formatting → same cluster`
- `Unit: title 'CBT for anxiety' vs 'C.B.T. for anxiety:' same year/author → fuzzy match`
- `Unit: semantically near but different studies → flagged, not auto-marked`
- `Integration: dedup run sets is_duplicate and master_ref_id, updates PRISMA duplicates_removed`
- `Integration: manual decide overrides automatic clustering`

#### 4.3 — Document storage and PDF text/table extraction

**What**: `document` model, PDF upload to object storage, and background text+table extraction.

**Design**:
- `document` per suggestion-1 (reference_id, file_type, storage_path, content_hash SHA-256, full_text, page-level data). Upload streams to S3/MinIO keyed by content hash (dedupes identical PDFs).
- `tasks/ingest.extract_pdf(document_id)`: PyMuPDF → per-page text + word bounding boxes (stored as JSONB for annotation anchoring); pdfplumber → table candidates (stored JSONB for AI extraction in Phase 6).
- Endpoints:
  ```
  POST /references/{id}/documents  (multipart) → 202 {document_id}
  GET  /documents/{id}             → metadata + signed download URL
  GET  /documents/{id}/text        → extracted text + page map
  ```

**Testing**:
- `Integration: upload PDF → stored at content-hash key, extract task enqueued`
- `Integration: re-upload identical PDF → same storage key, no duplicate object`
- `Fixture: extract sample_trial.pdf → full_text non-empty, page count correct, ≥1 table candidate`

#### 4.4 — PDF annotation

**What**: `document_annotation` model and highlight/note CRUD anchored to page coordinates.

**Design**:
- Model per suggestion-1 (page_number, x/y/width/height, annotation_type `highlight|note|extraction_link`, colour, content, `linked_extraction_field` for Phase 6).
- Endpoints under `/documents/{id}/annotations` (list, create, update, delete). Coordinates normalised to PDF point space for renderer-independence.

**Testing**:
- `Integration: create highlight annotation → persisted with coordinates`
- `Integration: list annotations for document returns user-visible set`
- `Unit: annotation with out-of-range page → 422`

---

## Phase 5: AI-Powered Active-Learning Screening

### Purpose
Deliver the flagship AI-native differentiator: an active-learning engine that ranks unscreened records by predicted relevance and continuously retrains as reviewers decide, so confident stopping is reached with far fewer human screens than random order. This builds directly on Phase 3 decisions and Phase 4 embeddings; it augments — never replaces — human decisions.

### Tasks

#### 5.1 — Active-learning model and prediction store

**What**: `screening_prediction` model and a pluggable classifier producing per-reference relevance scores and uncertainty.

**Design**:
- `screening_prediction` per suggestion-1 (phase, reference, model_version, relevance_score 0–1, predicted_at, training_labels_count).
- `services/active_learning`:
  ```python
  class ScreeningModel(Protocol):
      def fit(self, X, y) -> None: ...
      def predict_proba(self, X) -> np.ndarray: ...
  # default: TfidfVectorizer + LogisticRegression; embedding-feature variant pluggable
  def train_and_predict(phase_id) -> PredictionBatch:
      # labels = decided references (include=1, exclude=0)
      # features = TF-IDF of title+abstract (or stored embeddings)
      # returns relevance_score + uncertainty = 1 - |p - 0.5|*2 for each undecided ref
  ```
- Retrain trigger: after every N (default 10) new decisions, dispatch `tasks/extraction`-style Celery job (debounced) to retrain and write predictions. Model artefacts and `model_version` recorded for reproducibility.

**Testing**:
- `Unit: fit on labelled toy set → relevant-topic refs score higher than off-topic`
- `Unit: uncertainty maximal at p=0.5, minimal at p∈{0,1}`
- `Integration: 10 decisions → retrain task enqueued, predictions written with incremented training_labels_count`

#### 5.2 — Relevance-ranked queue and stopping criteria

**What**: Order the screening queue by predicted relevance and surface a data-driven stopping recommendation.

**Design**:
- `GET .../queue?order=relevance` returns undecided refs sorted by `relevance_score DESC` (certainty sampling) with an `?order=uncertainty` option (active-learning sampling).
- Stopping heuristic: track consecutive irrelevant records in relevance order; surface "X consecutive excludes; estimated recall ≥ 95%" indicator (ASReview-style). Endpoint `GET .../screening/{phase_id}/stopping → {screened, consecutive_irrelevant, estimated_recall, recommend_stop}`.

**Testing**:
- `Unit: queue ordered by relevance desc`
- `Unit: stopping recommends stop after threshold consecutive excludes`
- `Integration: as includes are found, ranking surfaces likely-relevant refs earlier than random baseline (recall@k higher)`

#### 5.3 — Screening UI: AI assistance

**What**: Show AI relevance scores, prediction confidence, and stopping indicator in the screening interface.

**Design**:
- Queue ordered by AI relevance with a per-card score badge; toggle between relevance/uncertainty/import order; stopping panel with recall estimate and a "mark remaining as excluded" bulk action (audited, requires confirmation).

**Testing**:
- `E2E: enable AI ordering → highest-scored refs appear first`
- `E2E: stopping panel appears after threshold; bulk-exclude requires confirm and logs audit`

---

## Phase 6: AI-Assisted Data Extraction

### Purpose
Implement customisable extraction templates and LLM-assisted structured extraction from full-text PDFs (mapped to PICO), the second flagship differentiator. Reviewers define a template once; AI pre-fills values with confidence, source page, and supporting quote; humans verify. Built on Phase 4 documents/tables.

### Tasks

#### 6.1 — Extraction templates (JSONB)

**What**: `extraction_template` and `extraction_record` per data-model-suggestion-3 (JSONB schema and values).

**Design**:
- `extraction_template.schema` JSONB = `{sections:[{name, fields:[{name,label,type,options?,pico?,required?,help?}]}]}`; types `text|number|date|select|multi_select|boolean|textarea`.
- `extraction_record.values` JSONB keyed by field name → `{value, ai_extracted, ai_confidence?, verified, source_page?, source_quote?}`. UNIQUE(review, reference, template, extractor). GIN index on `values`.
- Pydantic models validate template schema and value payloads against the active template (type checks, required, select-option membership).
- Endpoints:
  ```
  POST /reviews/{id}/extraction/templates {name, schema} → validates schema
  GET  /reviews/{id}/extraction/templates
  POST /reviews/{id}/extraction/records   {reference_id, template_id}
  PUT  /extraction/records/{id}/values    {field_name: value, …}  (partial, validated)
  ```

**Testing**:
- `Unit: schema with duplicate field name → 422`
- `Unit: value for select field not in options → 422`
- `Unit: missing required field on complete → 422`
- `Integration: save partial values, retrieve, type-correct JSONB round-trip`

#### 6.2 — LLM extraction service

**What**: AI pre-extraction of template fields from a reference's PDF text/tables with citations.

**Design**:
- `services/ai_extract.extract(record_id) -> dict[field, AIValue]`:
  - Builds prompt from the template field definitions + document full_text (chunked) + table candidates.
  - **System prompt** (templated): "You are a data-extraction assistant for systematic reviews. Extract only values explicitly stated in the provided study text. For each requested field return JSON: value, source_page, source_quote (verbatim), confidence (0–1). If the value is not present, return null with confidence 0. Never infer or fabricate."
  - **User prompt**: field schema (name/label/type/options) + numbered page text.
  - Calls LiteLLM with `response_format=json`; validates against template types; clamps select values to options.
  - Writes results into `extraction_record.values` with `ai_extracted=true, verified=false`.
- Dispatched via `tasks/extraction.ai_extract_record`. Endpoint `POST /extraction/records/{id}/ai-extract → 202`.

**Testing**:
- `Unit (mocked LLM): well-formed JSON response → values mapped, confidence stored`
- `Unit (mocked LLM): hallucinated select option → clamped/rejected, marked low confidence`
- `Unit (mocked LLM): field not in text → null value, confidence 0`
- `Integration: ai-extract a fixture record → values present, verified=false, source_page set`

#### 6.3 — Extraction UI with PDF linkage

**What**: Side-by-side PDF viewer and extraction form; AI suggestions highlighted; click-to-verify and source-quote jump.

**Design**:
- `features/extraction`: PDF.js viewer (left) + dynamic form from template (right). AI-filled fields show a confidence chip and "view source" that scrolls the PDF to `source_page` and highlights `source_quote`. Verifying flips `verified=true`. Dual extraction (two extractors) shows a reconciliation view.

**Testing**:
- `E2E: open record → AI suggestions shown with confidence; verify a field → persisted verified`
- `E2E: click 'view source' → PDF scrolls to page and highlights quote`

---

## Phase 7: Risk of Bias (RoB 2) & Quality Assessment

### Purpose
Implement Cochrane RoB 2 (and ROBINS-I / CEE templates via JSONB), including signalling questions, domain judgements, overall judgement, and AI-assisted answer suggestions — required for Cochrane-compatible clinical reviews.

### Tasks

#### 7.1 — RoB templates and assessment models

**What**: `rob_template` (JSONB schema, seeded RoB 2) and `rob_assessment` (JSONB domain answers/judgements) per data-model-suggestion-3, with the RoB 2 overall-judgement algorithm from suggestion-1.

**Design**:
- `rob_template.schema` = `{domains:[{number,name,questions:[{id,text,help}]}]}`. Seed RoB 2 (5 domains) and provide ROBINS-I, CEE CAT as additional seeds.
- `rob_assessment` stores `answers` JSONB (`{question_id: {answer, support_text}}`, answers ∈ `yes|probably_yes|no|probably_no|no_information`) and `domain_judgements` JSONB (`{domain: low|some_concerns|high}`) plus `overall_judgement`.
- `services` implements RoB 2 algorithm mapping signalling answers → domain judgement, and domains → overall (`high` if any high; `some_concerns` if any some-concerns; else `low`).
- Endpoints under `/reviews/{id}/rob` (templates, create assessment, save answers, compute judgement).

**Testing**:
- `Unit: all 'yes/probably_yes' favourable answers → domain low`
- `Unit: one high-risk domain → overall high`
- `Unit: some-concerns domains only → overall some_concerns`
- `Integration: save answers, recompute, overall persisted`

#### 7.2 — AI-assisted RoB suggestions

**What**: LLM proposes signalling-question answers with supporting quotes from the PDF.

**Design**:
- Reuses `ai_extract` prompt pattern: per signalling question, prompt the model for `{answer, support_text, confidence}` grounded in document text; stored as suggestions the assessor confirms (never auto-finalised).

**Testing**:
- `Unit (mocked LLM): returns valid answer enum + quote → stored as suggestion`
- `Integration: assessor accepts suggestion → answer recorded, audit logged`

#### 7.3 — RoB UI and traffic-light summary

**What**: Domain-by-domain assessment form and a traffic-light (green/amber/red) summary table across studies.

**Design**: `features/rob` form per domain with signalling questions and judgement; summary matrix (studies × domains) with colour cells, exportable as image/CSV.

**Testing**:
- `E2E: complete a RoB 2 assessment → overall computed and shown`
- `E2E: summary matrix renders correct colours for judgements`

---

## Phase 8: Evidence Synthesis — GRADE, Meta-Analysis & Evidence-Gap Mapping

### Purpose
Turn extracted data into synthesised evidence: GRADE Summary of Findings tables, pairwise meta-analysis with forest/funnel plots, and PICO-based evidence-gap maps (a differentiator). Statistician-facing, building on Phases 6–7.

### Tasks

#### 8.1 — GRADE assessment

**What**: `grade_outcome` and `grade_assessment` per suggestion-1; certainty computed from six domains.

**Design**:
- Models with downgrade domains (risk_of_bias, inconsistency, indirectness, imprecision, publication_bias) and upgrade factors; `services.grade.compute_certainty(start='high', downgrades, upgrades) -> high|moderate|low|very_low`. Effect-estimate fields for Summary of Findings.
- Endpoints under `/reviews/{id}/grade`; JSON-LD export of SoF tables (GRADEpro interop per `standards.md`).

**Testing**:
- `Unit: two serious downgrades from high → low`
- `Unit: observational start low + large effect upgrade → moderate`
- `Integration: SoF export validates against JSON-LD structure`

#### 8.2 — Meta-analysis engine

**What**: `meta_analysis` and `meta_analysis_study` per suggestion-1; fixed/random-effects pooling.

**Design**:
- `services/meta_analysis`: compute per-study effect (OR/RR/MD/SMD) and SE from raw 2×2 or mean/SD data; inverse-variance pooling (fixed) and DerSimonian-Laird (random); heterogeneity I², τ², Q. Returns pooled estimate, CI, and per-study weights for forest plots; funnel-plot coordinates and Egger's test for publication bias.
- Endpoints under `/reviews/{id}/meta-analyses` (create, add studies, run → results JSON).

**Testing**:
- `Unit: known 2×2 dataset → OR matches hand-computed value`
- `Unit: random-effects pooled estimate and I² match a reference (validated against statsmodels/known example)`
- `Integration: create MA from extracted values, run, forest-plot data returned`

#### 8.3 — Evidence-gap map and synthesis UI

**What**: PICO-dimension evidence-gap heatmap plus forest/funnel plot rendering.

**Design**:
- `services` aggregates included studies across Population × Intervention × Outcome (from extraction PICO fields) into a matrix of study counts. `features/synthesis` renders the heatmap (Recharts), forest plots, funnel plots, and the SoF table.

**Testing**:
- `Unit: gap-map aggregation counts studies per PICO cell correctly`
- `E2E: run meta-analysis → forest plot renders with correct number of studies`
- `E2E: evidence-gap heatmap highlights empty cells`

---

## Phase 9: PRISMA Reporting & Exports

### Purpose
Auto-generate the PRISMA 2020 flow diagram and 27-item checklist from live review data, and export the full review in standard formats (RIS/BibTeX, CSV, PRISMA diagram, GRADE JSON-LD, PDF report). This is the deliverable researchers ultimately need.

### Tasks

#### 9.1 — PRISMA flow diagram (auto-calculated)

**What**: `prisma_flowchart` and `prisma_checklist_item` per suggestion-1; counts derived from real data.

**Design**:
- `services/prisma.recalculate(review_id)` computes: records from databases/registers/other (from `search_execution` + import sources), duplicates removed (from dedup), records screened/excluded (title/abstract phase), reports sought/assessed/excluded-with-reasons (full-text phase, grouped by exclusion reason), studies/reports included. Stores in `prisma_flowchart` with `auto_calculated=true`; manual override supported.
- 27-item checklist seeded per review; items auto-marked complete where data exists (e.g., item for "information sources" complete when search_executions present).
- Endpoints: `GET /reviews/{id}/prisma/flow`, `POST .../recalculate`, `GET .../checklist`, `GET .../flow.svg` (rendered diagram).

**Testing**:
- `Unit: flow arithmetic — identification − duplicates = screened; screened − excluded = assessed (consistency assertions)`
- `Integration: full pipeline fixture → flow counts match decisions and dedup`
- `Integration: exclusion-reason breakdown in 'excluded with reasons' box matches full-text exclusions`

#### 9.2 — Exports

**What**: Multi-format export of references, screening decisions, extraction data, RoB, GRADE, and a full PDF report.

**Design**:
- `services/exports`: RIS/BibTeX (included references), CSV (extraction values flattened from JSONB; screening decisions; RoB matrix), PRISMA diagram SVG/PNG, GRADE SoF JSON-LD, and a composed PDF report (protocol + PRISMA + included-studies table + RoB summary + SoF). Large exports run via Celery, delivered as signed S3 URLs.
- `GET /reviews/{id}/export?format=ris|bibtex|csv|prisma_svg|grade_jsonld|report_pdf`.

**Testing**:
- `Unit: extraction JSONB flattened to CSV with one column per field`
- `Integration: RIS export of included refs re-imports to identical reference set (round-trip)`
- `Integration: report_pdf generated, non-empty, contains PRISMA section`

---

## Phase 10: Collaboration, Living Review, Realtime & MCP

### Purpose
Layer the collaboration and automation features that differentiate the platform for teams: realtime presence/live counts, scheduled living-review monitoring with auto-screening of new records, and an MCP server exposing the workflow to AI agents (the `standards.md` MCP opportunity).

### Tasks

#### 10.1 — Realtime collaboration

**What**: WebSocket presence and live updates for screening/extraction.

**Design**:
- `api/ws.py`: per-review WebSocket channel backed by Redis pub/sub. Broadcasts: decision made, conflict raised/resolved, progress counts, who's-online presence. Frontend subscribes and updates queues/progress live; soft-lock indicator when another reviewer opens the same reference.

**Testing**:
- `Integration: two WS clients on a review; one posts a decision → other receives progress update`
- `Integration: presence join/leave broadcast`

#### 10.2 — Living review monitoring

**What**: `living_review_config` and `living_review_run` per suggestion-1; scheduled re-search and alerting.

**Design**:
- `living_review_config` (search_interval_days, next_run_at, notify_on_new_records, auto_screen). Celery Beat dispatches `tasks/living.run_living_review` per due config: re-executes stored search_executions against the source APIs, dedups against existing references, optionally auto-screens new records with the trained active-learning model (threshold-gated), records a `living_review_run`, and notifies the team.
- Endpoints under `/reviews/{id}/living` (configure, run-now, list runs).

**Testing**:
- `Integration (mocked APIs + eager Celery): run finds 3 new non-duplicate records → run row records counts`
- `Unit: only configs with next_run_at ≤ now are dispatched`
- `Integration: auto_screen on → new records get AI predictions and provisional decisions above threshold`

#### 10.3 — MCP server

**What**: Expose search, screening, extraction, and export as MCP tools for agentic automation.

**Design**:
- `mcp/server.py` registers tools: `search_database(source, query)`, `import_references(review_id, source, query)`, `screen_reference(review_id, reference_id, decision)`, `extract_reference(review_id, reference_id, template_id)`, `get_prisma_flow(review_id)`. Tools call the same service layer used by the REST API; auth via API key scoped to a user/review. Optional component (`SLR_ENABLE_MCP`).

**Testing**:
- `Integration: MCP search tool returns results; screen tool records a decision via service layer`
- `Unit: MCP tool auth rejects key without review access`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (auth, tenancy, review)        ── required by everything
    │
Phase 2: References, Import & Search               ── requires P1
    │
Phase 3: Manual Screening                          ── requires P2
    │
    ├── Phase 4: Dedup & Documents (embeddings)    ── requires P2; can parallel with P3
    │       │
    │       └── Phase 6: AI Data Extraction        ── requires P4 (docs) + P3
    │       └── Phase 7: Risk of Bias              ── requires P4 (docs); can parallel with P6
    │
    └── Phase 5: AI Active-Learning Screening      ── requires P3 (decisions) + P4 (embeddings)
            │
Phase 8: Synthesis (GRADE, meta-analysis, gap map) ── requires P6 + P7
    │
Phase 9: PRISMA Reporting & Exports                ── requires P3, P4, P6, P7, P8 (consumes all)
    │
Phase 10: Collaboration, Living Review, MCP        ── requires P2/P3/P5; MCP requires service layers from P2–P6
```

**Parallelism opportunities:**
- Phase 4 (Dedup & Documents) can be developed concurrently with Phase 3 (Manual Screening) once Phase 2 is done.
- Phase 6 (AI Extraction) and Phase 7 (Risk of Bias) can be developed concurrently once Phase 4 is done.
- Phase 5 (AI Screening) can proceed in parallel with Phases 6/7 once Phases 3 and 4 are complete.
- Frontend feature work for each phase can lag one phase behind backend once the OpenAPI client is generated.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass (`pytest`), including the named scenarios in each task.
3. Frontend E2E tests (Playwright) for the phase pass where the phase has UI tasks.
4. Linting and formatting pass: `ruff check`, `ruff format --check` (backend); `eslint`, `prettier --check` (frontend).
5. Type checking passes: `mypy src/slr` (backend); `tsc --noEmit` (frontend).
6. `docker compose up` builds and starts all services; `/health` returns ok.
7. The phase's primary capability works end-to-end against a running stack (manually verified once, then covered by an E2E or integration test).
8. New configuration options are documented in `README.md` and `config.py` with defaults.
9. New API endpoints appear in the auto-generated OpenAPI 3.1 spec (`docs/openapi.json` regenerated and committed).
10. An Alembic migration exists for any schema change and `alembic upgrade head` / `downgrade` run cleanly on a fresh database.
11. Significant state-changing actions write to the `audit_log` (reproducibility requirement for clinical/regulatory reviews).
