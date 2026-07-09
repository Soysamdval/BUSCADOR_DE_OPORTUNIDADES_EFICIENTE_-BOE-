# BOE (Buscador de Oportunidades Eficiente) — Opportunity Intelligence Platform

> An end-to-end scraping + ETL + AI-scoring platform that replaced a fragile n8n/Tavily
> automation with a production-grade system: it monitors public and authenticated
> procurement sources (RFx, tenders, grants), normalizes every finding into a single
> lead schema, scores each lead against the firm's service catalog using RAG + LLMs,
> and surfaces the results in a web dashboard with Microsoft Teams notifications.

**Client:** RSM Colombia (audit, tax & consulting firm) · **Role:** Full-stack / data engineering · **Status:** In production

<p>
  <img alt="Python" src="https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white">
  <img alt="FastAPI" src="https://img.shields.io/badge/FastAPI-0.115-009688?logo=fastapi&logoColor=white">
  <img alt="React" src="https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=black">
  <img alt="TypeScript" src="https://img.shields.io/badge/TypeScript-5-3178C6?logo=typescript&logoColor=white">
  <img alt="PostgreSQL" src="https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white">
  <img alt="Celery" src="https://img.shields.io/badge/Celery-5.4-37814A?logo=celery&logoColor=white">
  <img alt="Docker" src="https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white">
  <img alt="Playwright" src="https://img.shields.io/badge/Playwright-1.49-2EAD33?logo=playwright&logoColor=white">
  <img alt="OpenAI" src="https://img.shields.io/badge/OpenAI-RAG-412991?logo=openai&logoColor=white">
</p>

---

## The problem

The firm's business-development team was discovering procurement opportunities through
a hand-wired n8n + Tavily flow. It was expensive per query, brittle when a source
changed its markup, impossible to authenticate against gated portals (SAP Ariba, Cámara
de Comercio), and produced a raw firehose with no notion of *"is this actually relevant
to us?"*. Analysts spent hours triaging noise.

## The solution

A self-hosted platform that owns the full pipeline — from raw HTML to a ranked,
deduplicated, business-relevant lead sitting in a dashboard — with zero per-query API
cost for scraping and a pluggable source registry that a non-developer can extend from
the admin panel.

```
External sources ─► Scrapers (registry) ─► RawLead ─► Transformer ─► Loader (dedup) ─► PostgreSQL
  RSS · HTML · JSON                          (normalize, classify sector)  (upsert on url_hash)      │
  browser · Ariba API · CCB                                                                          ▼
                                                                                          Affinity Scorer
      MS Teams ◄── notify_tasks (Adaptive Cards) ◄── Celery Beat (cron) ──► scrape_tasks    (RAG + GPT-4o-mini)
                                                                                                    │
      React SPA (Nginx) ─────────────► FastAPI (/api/v1) ─────────────► PostgreSQL ◄───────────────┘
```

---

## Highlights (what makes this interesting)

| Area | What was built | Why it matters |
|------|----------------|----------------|
| **Pluggable scraping** | A `BaseScraper` ABC + a `source_type → class` registry. New sources are configured as data (CSS selectors / field maps) — no redeploy. | A business user adds a source from the UI; the engineer never touches code for a new RSS/HTML/JSON feed. |
| **Authenticated portals** | Playwright + Chromium with an assisted-login flow (Xvfb + CDP + noVNC in a sidecar container) and Redis-persisted sessions. | Handles gated sources that a plain HTTP scraper can't reach. |
| **Anti-bot bypass (SAP Ariba)** | When headless login was blocked by anti-bot, pivoted to replaying Ariba's private JSON API using cookies + CSRF token imported via "Copy as cURL". | Pragmatic engineering: shipped a working integration instead of fighting an unwinnable browser war. |
| **AI affinity scoring** | RAG over the firm's service catalog (`text-embedding-3-small`) + `gpt-4o-mini` in JSON mode, with weighted criteria and "killer constraints". | Turns a raw feed into a *ranked* feed — the analyst sees the best leads first. |
| **Deterministic dedup** | `url_hash` = SHA-256 of the URL, unique index, `INSERT … ON CONFLICT DO NOTHING`. | Idempotent ingestion across overlapping sources and repeated runs. |
| **Scheduled digests** | DB-editable schedules; Beat ticks every minute and dispatches Teams summaries when due, with Redis-based dedup. | Ops configures notification times without touching cron/code. |
| **Security by default** | JWT access+refresh, bcrypt passwords, Fernet-encrypted source credentials, SSRF guard on outbound scraper URLs, Teams webhooks locked to Microsoft domains. | Handles third-party credentials and untrusted URLs responsibly. |

---

## Architecture

**Backend** — FastAPI 0.115 (fully async), SQLAlchemy 2.0 async + asyncpg, Alembic
migrations, Pydantic v2 / pydantic-settings, structlog structured logging.

**Queue / scheduler** — Celery 5.4 + Beat with Redis 7 as broker & backend, timezone-aware
(`America/Bogota`).

**Scraping** — httpx (RSS/HTML/JSON), BeautifulSoup4 + lxml, feedparser, Playwright +
Chromium for dynamic SPAs and authenticated portals.

**Scoring** — OpenAI embeddings + `gpt-4o-mini`, RAG over a curated service catalog.

**Frontend** — React 18 + TypeScript (strict), Vite 5, Tailwind 3, TanStack Query,
React Router 6, Zustand, Recharts. Axios client with a JWT interceptor that transparently
refreshes on 401; authenticated file downloads via blob.

**Infrastructure** — Docker Compose (postgres, redis, api, worker, beat, frontend, nginx,
browser sidecar), Nginx reverse proxy with an internal resolver so the stack stays up even
when the optional browser container is down.

### Source types

| Type | Scraper | Use case |
|------|---------|----------|
| `rss` | `RSSFeedScraper` | RSS/Atom feeds |
| `html` | `HTMLScraper` | static HTML, CSS selectors as config |
| `json` | `JSONAPIScraper` | JSON APIs, field mapping as config (templated URLs, fallbacks) |
| `browser` | `BrowserScraper` | dynamic SPAs (Playwright) |
| `auth_browser` | `AuthBrowserScraper` | login-gated sites (session in Redis) |
| `ariba` / `ariba_api` | `AribaScraper` / `AribaAPIScraper` | SAP Ariba — automated login, and the JSON-API replay fallback |
| `ccb` | `CCBScraper` | Cámara de Comercio de Bogotá (SSO) |

---

## Engineering decisions worth calling out

- **Sources seeded via migration**, not fixtures — they survive redeploys, and the API
  runs `alembic upgrade head` on boot.
- **Admin seeded in the app lifespan** from environment variables, so no secret is ever
  hardcoded in a migration.
- **Isolated async engine per Celery task** to avoid `asyncpg` `InterfaceError` from
  sharing connections across the event loop boundary — a subtle bug that only shows up
  under concurrency.
- **ORM-level `pg_insert`** for the leads upsert so JSONB columns are handled correctly on
  conflict.
- **Nginx internal resolver** for the optional VNC upstream, so a missing sidecar never
  takes down the reverse proxy.

---

## Tech stack at a glance

`Python` · `FastAPI` · `SQLAlchemy 2.0 async` · `asyncpg` · `Alembic` · `Pydantic v2` ·
`Celery + Beat` · `Redis` · `Playwright` · `BeautifulSoup4` · `feedparser` · `httpx` ·
`OpenAI (RAG + LLM)` · `PostgreSQL 16` · `React 18` · `TypeScript` · `Vite` · `Tailwind` ·
`TanStack Query` · `Zustand` · `Recharts` · `Docker Compose` · `Nginx` · `JWT` · `Fernet`

---

## Running it locally

```bash
git clone <repo-url> boe
cd boe
cp .env.example .env          # then fill SECRET_KEY, ENCRYPTION_KEY, POSTGRES_PASSWORD
docker compose up --build -d  # api runs `alembic upgrade head` before uvicorn
```

- **Dashboard:** http://localhost
- **API docs (OpenAPI):** http://localhost/api/docs

Generate the required keys:

```bash
python3 -c "import secrets; print(secrets.token_hex(32))"                       # SECRET_KEY
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"  # ENCRYPTION_KEY
```

> A full technical setup, source-configuration guide, and operations runbook live in
> [`README.md`](./README.md) and `docs/`.

---

## What I'd highlight in an interview

- Designed a **plugin architecture** (registry + ABC) so the system grows by configuration,
  not code — a real "build once, extend forever" win for a non-technical BD team.
- Shipped a **working SAP Ariba integration** by reverse-engineering its private API when
  the browser path was blocked — chose the pragmatic path over the elegant-but-doomed one.
- Layered **RAG-based relevance scoring** on top of raw ingestion to convert a noisy
  firehose into a ranked, decision-ready queue.
- Built the whole thing **async end-to-end** and containerized it into a reproducible
  Docker Compose stack with migrations-on-boot and secure-by-default handling of
  third-party credentials.


