# ARCHITECTURE.md

# CuriousQuasar Data Platform - Architecture

> Living planning document for the multi-source ingest and analytics platform
> running on curiousquasar.com infrastructure.

**Status:** Draft v0.1
**Last updated:** 28 April 2026
**Owner:** Mandy

---

## 1. Goals

1. Ingest data from multiple personal data sources (Toggl, PurpleAir, more to come) into a central platform on curiousquasar.
2. Use Elastic as the primary analytics and dashboarding layer.
3. Keep raw data locally as a defensive measure and to enable future use cases beyond Elastic.
4. Build the platform incrementally, one source at a time, with each source following an established pattern.
5. Serve as practical Elastic research - exercise real ingest patterns, connectors, dashboards, and operational concerns.

## 2. Non-goals

- Not building a multi-tenant or production-grade SaaS platform.
- Not optimising for high write throughput (these are personal data volumes, not log fire-hoses).
- Not solving for high availability beyond reasonable backups.
- Not building custom dashboard UI - Kibana is the interface.

## 3. Architecture pattern

**Pattern: Postgres as landing zone, Elastic as workspace.**

Raw API responses land in Postgres as JSONB blobs with minimal transformation. Elastic ingest pipelines do the parsing, enrichment, and shaping. This means:

- Postgres stores the truth in its rawest available form. If a future use case needs different parsing, the original data is intact.
- Elastic does the modelling work where it is most natural - via ingest pipelines, runtime fields, and index templates.
- Schema decisions are deferred until we actually understand each data source.
- Switching, augmenting, or replacing the analytics layer later is low-risk.

```
+-------------+     +------------------+     +-----------------+
|  Toggl API  |---->|  Ingest scripts  |---->|    Postgres     |
+-------------+     |  (Python)        |     |  curiousquasar  |
                    |  systemd timers  |     |                 |
+-------------+     |                  |---->|  raw.toggl_*    |
| PurpleAir   |---->|                  |     |  raw.purpleair_*|
|    API      |     +------------------+     |  raw.<future>_* |
+-------------+                              +--------+--------+
                                                      |
                                            Elastic ingest
                                            (connector or
                                             Logstash JDBC)
                                                      |
                                                      v
                                            +------------------+
                                            |     Elastic      |
                                            |  (Cloud or self) |
                                            |                  |
                                            |  parsed indices  |
                                            |  + Kibana        |
                                            +------------------+
```

## 4. Component decisions

### 4.1 Postgres (landing zone)

- **Version:** PostgreSQL 16 (current stable on AlmaLinux/RHEL family).
- **Storage approach:** Per-source schema (e.g. `toggl`, `purpleair`). Each source gets one or more tables holding raw JSONB plus a small set of extracted columns for indexing and dedup (id, fetched_at, source-specific timestamp).
- **Why JSONB rather than parsed columns:** API shapes change. New fields appear. Storing the full payload means we never lose information we did not know we needed.
- **Indexing:** B-tree on extraction time and source-native id. GIN on the JSONB column where querying inside payloads is useful.
- **Backups:** `pg_dump` to off-server storage on a daily timer. Details TBD.

Example landing table shape:

```sql
CREATE SCHEMA IF NOT EXISTS toggl;

CREATE TABLE toggl.time_entries_raw (
    id              BIGINT PRIMARY KEY,        -- Toggl's entry id
    workspace_id    BIGINT NOT NULL,
    started_at      TIMESTAMPTZ NOT NULL,
    fetched_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    payload         JSONB NOT NULL,
    UNIQUE (id, workspace_id)
);

CREATE INDEX ON toggl.time_entries_raw (started_at);
CREATE INDEX ON toggl.time_entries_raw USING GIN (payload);
```

### 4.2 Ingest scripts

- **Language:** Python 3.12+.
- **Libraries:** `httpx` (HTTP client), `psycopg[binary]` (Postgres driver, v3), `pydantic` (response shape validation, optional but useful), `tenacity` (retries).
- **Style:** One package per data source, e.g. `ingest/toggl/`, `ingest/purpleair/`. Each package exposes a CLI entrypoint via `python -m ingest.toggl`.
- **Modes:** Each ingester supports `--backfill` (full historical pull) and `--incremental` (since last successful run, the default). State stored in a small `ingest_state` table in Postgres.
- **Idempotency:** All inserts are `INSERT ... ON CONFLICT (id) DO UPDATE` so reruns are safe.
- **Secrets:** Loaded from environment, sourced from a `.env` file readable only by the ingest user. No secrets in code or in repo.
- **Logging:** Structured JSON logs to stdout, captured by systemd journal.

### 4.3 Scheduling

- **Mechanism:** systemd timers (one timer + one service unit per ingester). Preferred over cron for journal integration, dependency management, and retry semantics.
- **Daily run:** Each source ingests on its own timer. Toggl daily, PurpleAir more frequent (TBD - depends on update cadence we want).
- **Failure handling:** systemd `OnFailure=` hook to a notification service (email or push - TBD).

### 4.4 Elastic

- **Hosting:** Deferred. Both options remain on the table:
    - **Elastic Cloud trial first.** Lowest-friction way to evaluate connectors, ingest pipelines, dashboards, and the rest of the platform features. 14-day trial.
    - **Self-hosted on curiousquasar.** Considered later if VPS resources allow and self-hosting becomes interesting from an ops-learning angle.
- **Sync from Postgres:** Two candidate paths, both worth trying:
    1. **Elastic's native Postgres connector** - cleanest, fits the "use Elastic as a real customer would" goal.
    2. **Logstash JDBC input** - more configurable, more moving parts, useful for understanding what the connector abstracts away.
- **Indices:** One index pattern per data source, e.g. `toggl-time-entries-*`, with date-based suffixes managed by ILM where appropriate.
- **Pipelines:** Per-source ingest pipelines that flatten the JSONB payload into typed fields, attach enrichments (e.g. project name lookups for Toggl), and set up timestamp fields.

### 4.5 Secrets management

- `.env` files on the server, gitignored.
- API tokens scoped per source (one Toggl token, one PurpleAir token).
- Postgres credentials per ingester (one role per source, with insert/update on its own schema only - principle of least privilege).
- Future option: move to a proper secret store (Bitwarden CLI, age-encrypted files, or systemd credentials) once there is more than one host.

### 4.6 Repository structure

```
curiousquasar/data-platform/
+-- docs/
|   +-- ARCHITECTURE.md         (this file)
|   +-- toggl-ingest.md         (per-source design notes)
|   +-- purpleair-ingest.md
|   +-- decisions/              (ADRs - one file per decision)
|       +-- 0001-postgres-as-landing-zone.md
|       +-- 0002-python-for-ingest.md
+-- ingest/
|   +-- common/                 (shared: logging, db, http, state)
|   +-- toggl/
|   +-- purpleair/
+-- infra/
|   +-- postgres/               (init scripts, schema migrations)
|   +-- systemd/                (timer and service unit files)
|   +-- elastic/                (index templates, ingest pipelines)
+-- pyproject.toml
+-- README.md
+-- .env.example
+-- .gitignore
```

## 5. Build phases

### Phase 0 - Foundations (server prep)

- [ ] Install PostgreSQL 16 on curiousquasar.
- [ ] Create database `curiousquasar_data`, an admin role, and per-source roles.
- [ ] Set up daily `pg_dump` backup to off-server location.
- [ ] Create the git repo, populate skeleton structure.
- [ ] Set up Python project (pyproject.toml, virtualenv, pre-commit, ruff).

### Phase 1 - Toggl ingest

- [ ] Investigate Toggl Track API v9 (auth, endpoints, pagination, rate limits).
- [ ] Design landing table schema.
- [ ] Build backfill script.
- [ ] Run backfill, validate row counts and date coverage.
- [ ] Build incremental script + systemd timer.
- [ ] Verify daily runs for one week.
- [ ] Document in `toggl-ingest.md`.

### Phase 2 - Elastic dashboarding (Toggl)

- [ ] Spin up Elastic Cloud trial.
- [ ] Connect Postgres -> Elastic via native connector.
- [ ] Build ingest pipeline to parse JSONB payload into typed fields.
- [ ] Build first Kibana dashboard (time-by-project, time-by-day, etc.).
- [ ] Document ingest pipeline and dashboard in repo.

### Phase 3 - PurpleAir ingest

- [ ] Investigate PurpleAir API (auth, sensor selection, polling cadence).
- [ ] Design landing table schema.
- [ ] Build ingest script + timer (likely more frequent than daily).
- [ ] Add to Elastic with its own ingest pipeline and dashboard.

### Phase 4 - Operational maturity

- [ ] Failure notifications via systemd `OnFailure=`.
- [ ] Health check endpoint or simple status page on observatory.curiousquasar.com.
- [ ] Backup verification (test restore from `pg_dump`).
- [ ] Decide on Elastic hosting long-term (Cloud vs self-hosted).

### Phase 5+ - Future sources

- [ ] (TBD) other sources to add - candidates: bank transactions, GitHub activity, ski/snowboard tracking, weather at Mt Hotham.

## 6. Open questions

These are explicitly unresolved and worth thinking about as we go:

- Backup destination - off-server cloud bucket, another server, or a local disk on a different machine?
- Off-Elastic dashboards - is there value in a Grafana or Metabase view as a control comparison? Useful for the research narrative.
- Schema migrations - Alembic, sqitch, or hand-rolled SQL files? At this scale, SQL files might be enough.
- Observability of the platform itself - do we ingest our own ingest logs into Elastic for meta-monitoring?
- PurpleAir polling cadence vs API rate limits - need to read the docs.

## 7. Decision log (ADR pointers)

Decisions captured as Architecture Decision Records under `docs/decisions/`. Each ADR is short, dated, and explains the context, decision, and consequences.

- **0001** - Postgres as landing zone (Pattern 3) over Elastic-only or Postgres-as-source-of-truth.
- **0002** - Python for ingest scripts.
- **0003** - systemd timers over cron.
- **0004** - JSONB payloads over upfront schema design.
- (future) **0005** - Elastic hosting choice.

## 8. Glossary

- **Landing zone** - a place where raw data is stored as-received, with minimal transformation, so downstream layers can be rebuilt without re-fetching.
- **Ingest pipeline** (Elastic) - a server-side processing chain that runs on documents at index time.
- **ILM** - Index Lifecycle Management; Elastic's feature for rolling, shrinking, and deleting indices on schedule.
- **ADR** - Architecture Decision Record; a short markdown file capturing why a decision was made.