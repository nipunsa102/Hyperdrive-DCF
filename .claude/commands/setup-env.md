---
description: Initialize the runtime environment after /promote-poc — validate configuration, apply backend schema/migrations, seed reference data, verify external service connectivity, and confirm the app is ready to run.
model: claude-opus-4-6
---

**Switches**: `-skip-smoke`, `-verify-only`

**Switch Definitions**:
- `-skip-smoke` → Skip the final runtime smoke test (useful if you want to set up backends but not start the app)
- `-verify-only` → Run only the validation + connectivity checks; do not apply migrations or seed data (dry-run for existing environments)

## Purpose

Makes the promoted code actually runnable against a configured backend. After `/promote-poc` finishes, the `src/` tree is in place but backend stores have no schema, reference tables aren't seeded, and no external service has been confirmed reachable. This command closes that gap — it reads the project's configuration file(s), `CONFIG_GUIDE.md`, the promoted code, and project architecture; detects whatever technology stack the project actually uses; and takes every safe one-time setup action so the next run of the project's dev command works end-to-end.

**This command is framework-agnostic.** It makes no assumptions about which database, ORM, authentication provider, AI/ML provider, cache, queue, storage backend, or any other external service the project uses — and no assumptions about what configuration file format or location the project uses. Everything is detected from the project's own files.

**Position in flow:** After `/promote-poc` AND after the human has filled in the project's configuration file(s) per `CONFIG_GUIDE.md`. Before `/deploy-module` (when implemented).

**What this command does:**
- Detects the project's configuration convention from its POC + `TECHSTACK.md` + source code
- Validates that the production configuration file(s) at project root contain every required value listed in `CONFIG_GUIDE.md`
- Detects the project's migration / schema-management tooling from on-disk signals
- Applies initial migrations (creates schemas, tables, indexes, constraints — whatever the detected tool manages)
- Runs a **production-safe reference seed** if the project has one (taxonomy / enums / lookup tables only — NEVER sample user content)
- Inventories every external service declared in `architecture/architecture.md` and verifies connectivity to each
- Runs a runtime smoke test against real services via `smoke-test-agent`
- Writes `tracking/env-setup.md` recording what ran successfully and what was skipped or failed

**What this command does NOT do:**
- Provision cloud infrastructure (creating accounts, projects, tenants, buckets, databases, DNS entries — those are human steps in `CONFIG_GUIDE.md`)
- Seed sample user content, demo data, or any non-reference data
- Deploy the app to any environment (that's future `/deploy-module`)
- Rotate or generate real secrets (the human populates configuration from `CONFIG_GUIDE.md`)
- Assume any particular vendor, provider, technology, or configuration file format — all detection is project-driven
- Touch anything under `poc/`

## Prerequisites

1. `/promote-poc` must have completed successfully. Verify:
   - `CONFIG_GUIDE.md` exists at project root
   - `src/` exists with promoted code
   - `tracking/module-tracking.md` exists with modules marked `Complete` or `L1 Pass` (not all `Blocked`)
2. The project's production configuration file(s) must exist at project root in whatever format the project uses (detected in Phase 1 below). If missing, ERROR pointing the user to `CONFIG_GUIDE.md` and the config template produced by `/promote-poc`.
3. `architecture/architecture.md` and `architecture/data-model.md` must exist (for service inventory and schema reference).
4. `TECHSTACK.md` must exist (for tooling detection). If missing, ERROR asking the user to ensure the project's tech stack is documented.

If any prerequisite fails, ERROR with a clear message and stop.

## Process

### Phase 1: Configuration Detection & Validation

This phase is deliberately config-file-agnostic. Different projects use different configuration conventions (dotenv, YAML, TOML, JSON, platform-injected env vars, etc.). The framework detects whichever the project uses rather than hardcoding one.

**Scope rule:** discovery is restricted to the project root. `poc/` is NEVER scanned — POC is historical after promotion, and production config lives at root only.

1. **Detect the configuration format** by cross-referencing these two sources only:

   - **`TECHSTACK.md`** — if it names a specific configuration mechanism (e.g., "uses dotenv for local config", "config via YAML at `config/*.yaml`"), that is authoritative.
   - **Promoted source code** under `src/` — scan for how configuration is actually read at runtime (examples: `process.env.X` implies dotenv-style env vars; `fs.readFileSync('config.yaml')` implies YAML; `toml.load()` implies TOML; `import settings from './config'` implies a module import). The reader's expectations define the ground truth.

   From these signals, determine:
   - The **format** (dotenv KEY=VALUE, YAML, TOML, JSON, etc.)
   - The **filename pattern(s)** that real config files of this format typically use at project root (e.g., for dotenv: `.env`, `.env.local`, `.env.production`; for YAML: `config.yaml`, `config.yml`; for TOML: `config.toml`)
   - The **required value list** (sourced from `CONFIG_GUIDE.md`'s configuration-values table)

2. **Discover config files at project root only.** Enumerate files at the project root (non-recursive — do NOT descend into subdirectories, do NOT look in `poc/`) that match the detected format's filename patterns.

   **Exclude template files** — any file whose name contains `.example`, `.sample`, `.template`, `example`, `sample`, or `template` as a segment or suffix (e.g., `.env.example`, `config.sample.yaml`, `settings.template.toml`). Templates are produced by `/promote-poc` as scaffolding and are not real configuration.

   After filtering templates, handle cardinality as follows:

   - **Zero real config files found** → ERROR and STOP:
     ```
     ERROR: No production configuration file found at project root.
     Detected format (from TECHSTACK.md + src/): [format]
     Expected filename pattern(s) at root: [patterns]

     A template was produced by /promote-poc — look for [detected template name]
     at project root, copy it to one of the expected paths, and fill in values
     per CONFIG_GUIDE.md. Then re-run /setup-env.
     ```

   - **Exactly one real config file found** → use it. Report:
     ```
     Configuration file detected: [path] ([format])
     ```

   - **Multiple real config files found** → ASK the user which to use. Do NOT silently merge or validate all. Present the list and wait for selection:
     ```
     Multiple configuration files found at project root:
       (1) [path-1]
       (2) [path-2]
       ...
     Which should /setup-env use? Reply with the number or the path.
     ```
     STOP until the user responds. Use only the selected file for the rest of this run.

3. **Parse the selected config file** using the appropriate parser for the format (dotenv line parser, YAML loader, TOML loader, JSON parser). Never log parsed values.

4. **Validate against `CONFIG_GUIDE.md`:**
   - Every value listed in the guide's configuration table is present in the selected config file
   - Values are not empty / null
   - Values are not obvious placeholders: anything matching `YOUR_`, `<your-`, `<...>`, `TODO`, `CHANGE_ME`, `PLACEHOLDER`, `xxx...x` patterns, or the literal text `example.com`

5. **On failure, ERROR with a specific list** (variable/key names only — never values):
   ```
   ERROR: Configuration is missing or has placeholder values in [selected file]:
   - [KEY_1] (missing)
   - [KEY_2] (still has placeholder "YOUR_...")
   - [KEY_3] (missing)
   See CONFIG_GUIDE.md for how to obtain each value.
   ```

6. **Report pass:**
   ```
   Configuration validation: PASS
   - Format: [detected format]
   - File: [selected path]
   - Values: all N required keys present and non-placeholder.
   ```

If `-verify-only` was passed, skip straight to Phase 4 (connectivity) after this.

### Phase 2: Backend Schema / Migrations

This phase is framework-agnostic. It detects whatever migration or schema-management tooling the project actually uses by inspecting on-disk signals, reads `TECHSTACK.md` to confirm, and invokes the appropriate command.

1. **Detect the migration tool** by looking for on-disk signals in this priority order. Do NOT hardcode a specific tool — read `TECHSTACK.md` first and let it specify the tool; fall back to file-signal detection only if `TECHSTACK.md` is silent on schema management.

   Examples of signals and what they imply (non-exhaustive — expand as needed):

   | Signal observed | Likely tool | Command pattern |
   |---|---|---|
   | `*.prisma` schema file | Prisma | migrate / deploy |
   | `drizzle.config.*` | Drizzle | push |
   | `alembic.ini` | Alembic | upgrade head |
   | `knexfile.*` or `migrations/*.sql` with knex in package.json | Knex | migrate:latest |
   | `migrations/*.sql` with no ORM | golang-migrate, sqlx, flyway, etc. | tool-specific |
   | Rails project structure | ActiveRecord | db:migrate |
   | Django project structure | Django migrations | migrate |
   | Liquibase `changelog.xml` | Liquibase | update |
   | Flyway `flyway.conf` or `db/migration/` | Flyway | migrate |
   | No migration tool detected, but data model implies persistence | — | Report and ask user |

2. **Cross-check with `TECHSTACK.md`** — if it names a tool, use the command that tool prescribes. If the detected signal contradicts `TECHSTACK.md`, report the conflict and ask the user to clarify before proceeding.

3. **Run the migration command** via Bash. Stream output to the user (so slow migrations show progress).

4. **If no migration tool is detected AND the data model implies persistence:**
   - Inspect `architecture/data-model.md` — are there entities that need storage?
   - If yes, ERROR:
     ```
     ERROR: architecture/data-model.md declares N entities, but no migration tool
     was detected in the project and TECHSTACK.md does not name one.
     Options:
       (a) Add the chosen migration tool and its config to the project, then re-run.
       (b) If schema is managed outside this repo (e.g., by DBA), document that
           in TECHSTACK.md and re-run (Phase 2 will then skip gracefully).
     ```
   - If the data model is empty or describes only in-memory state, skip Phase 2 with a note.

5. **If migration fails** at runtime:
   - Capture the tool's error output
   - Classify the failure from common patterns:
     - "connection refused" / "cannot connect" → env/config issue → point at `CONFIG_GUIDE.md` database section
     - "authentication failed" / "password authentication failed" → wrong credentials → point at `CONFIG_GUIDE.md`
     - "database does not exist" / "schema does not exist" → pre-migration provisioning missing → ERROR with instructions (some providers require manual DB/schema creation before migration)
     - Tool-specific errors (conflicting schema, drift, lock held) → surface the tool's error verbatim and suggest the tool's own diagnostic command
   - Do NOT retry blindly. Mark Phase 2 `FAIL` and STOP.

6. **On success:** report a high-level summary (e.g., number of migrations applied, number of tables/collections created — parse from tool output where possible).

If `-verify-only` was passed, skip this phase (no mutation).

### Phase 3: Reference Data Seeding

Goal: seed the backend with reference / lookup / taxonomy data that is part of the application's correctness contract, WITHOUT seeding sample user content, demo data, or test records that only belong in development databases.

1. **Detect the seed mechanism** by looking for conventional file locations (examples, non-exhaustive):
   - A seed script configured in the project's migration tool (e.g., `db:seed` script in `package.json`, `alembic` data migration, `knex seed:run`)
   - A standalone seed file at a conventional location (`scripts/seed*.{ts,js,py,rb,go}`, `db/seeds.*`, `prisma/seed.*`, etc.)
   - A dedicated production seed (file name includes `prod`, `production`, `reference`, `taxonomy` — e.g., `seed.prod.ts`, `scripts/seed-reference.ts`)

2. **Classify the seed script before running it.** Read the file contents and look for:
   - **Production-safe markers** (safe to run in production):
     - Inserts into taxonomy / lookup / enum / category / role-definition tables
     - Uses `upsert` or similar idempotent patterns
     - Data is clearly pre-defined and part of the application's correctness (e.g., valid subject list, permissions taxonomy, status enums)
   - **Production-UNSAFE markers** (must NOT run in production):
     - Creates user accounts with realistic-looking names/emails
     - Inserts sample content records (posts, flicks, articles, orders) with fabricated text
     - Creates admin accounts with known passwords
     - Inserts data for testing purposes (e.g., "test user", "demo account")

3. **Decision tree:**
   - If a **dedicated production seed** exists (`seed.prod.*`, `seed-reference.*`, etc.) → run it.
   - If the only seed script is a **dev seed with mixed content** → DO NOT run it. Report:
     ```
     SKIPPED: No production-safe seed script found.
     The project's seed script ([path]) inserts sample user/content data which
     should not go into production. To seed reference data:
       1. Create a dedicated production seed that inserts only taxonomy/enums/
          lookup tables (name it with a "prod", "production", or "reference"
          marker so /setup-env detects it)
       2. Re-run /setup-env
     Alternatively, seed manually via the backend's admin console/CLI using the
     reference-data section of [dev seed path] as the source of truth.
     ```
     This is a warning, not an error — continue to Phase 4.
   - If **no seed script exists at all** → report "No seed step detected — skipping" and continue. This is normal for projects whose reference data lives in migration files.

4. **Run the identified prod-safe seed.**

5. **On failure:** surface the error and mark Phase 3 `FAIL`. Do NOT stop — Phase 4 connectivity checks are still valuable diagnostic info.

If `-verify-only` was passed, skip this phase.

### Phase 4: External Service Connectivity

This is the phase most prone to vendor-specific assumptions. It is deliberately built to be generic: the project tells `/setup-env` what to check, not the other way around.

1. **Inventory external services** — read `architecture/architecture.md` (especially the Integration Matrix and any "External Dependencies" / "Integrations" section) and `TECHSTACK.md`. Classify each external service into a generic **category** based on its role in the architecture, not its vendor:

   | Generic category | What this is | Typical verification pattern |
   |---|---|---|
   | Relational database | Transactional DB | Open connection + run a trivial query (e.g., `SELECT 1`, driver ping) |
   | Document/NoSQL database | Document, KV, wide-column, graph, or time-series store | Vendor-agnostic ping/health call using the project's driver |
   | Cache | In-memory cache (Redis-like) | Driver ping or SET/GET on a scratch key |
   | Queue / broker | Message broker | Connect + list queues/topics or read a health endpoint |
   | Identity / auth provider | Session / OIDC / OAuth provider | Call the provider's discovery endpoint (e.g., `/.well-known/openid-configuration`) or a documented health endpoint |
   | AI / ML provider | LLM, embedding, or model-inference API | Lightweight list-models / auth-check call via the configured SDK or REST endpoint |
   | Object storage | Blob / file storage | Connection-test call — list or head a known bucket/container |
   | Email / notification | Transactional mail / push / SMS | Credential validation call (not a real send) |
   | Search / analytics | Search index, analytics tracker | Cluster health / ping endpoint |
   | Other external API | Any remaining external HTTP API declared in architecture | Hit a documented health/auth endpoint, or a no-op GET |

   If the architecture declares a service whose category is unclear, ask the user how to verify it and add the pattern to `tracking/env-setup.md` for next time.

2. **Derive the verification call for each service from the project itself, not from hardcoded assumptions:**
   - Read the project's own client / SDK initialization code under `src/` to see how the service is accessed
   - Read `TECHSTACK.md` for the chosen SDK / driver / library
   - Use the project's own configured access pattern to make the verification call (so the check uses the same connection, credentials, and configuration the runtime will use)
   - If the project uses a specific SDK, prefer its built-in health/ping method over a raw HTTP call
   - Only fall back to a raw HTTP probe (e.g., `curl` to a well-known endpoint) if the project's SDK has no health primitive

3. **For each service, emit a status line** (never the secret value — only the service name, a derived identifier like hostname, and the outcome):
   ```
   [PASS] [SERVICE_NAME]  → reached [host/endpoint derived from env]
   [FAIL] [SERVICE_NAME]  → HTTP 401 (authentication failed) — see CONFIG_GUIDE.md §[N]
   [SKIP] [SERVICE_NAME]  → no verification pattern defined; manual check required
   ```

4. **Fail strategy:** a single-service failure does NOT abort the command — report and continue, so the user sees the full picture. At the end, if any service failed, exit with non-zero status.

5. **Never log actual secret values.** Log service names, hostnames / identifiers derived from env vars (but not the secrets themselves), HTTP status codes, and derived success/failure only.

### Phase 5: Runtime Smoke Test

Skip this phase if `-skip-smoke` or `-verify-only` was passed.

1. **INVOKE `smoke-test-agent`** with:
   ```
   SETUP-ENV SMOKE TEST:
   - Build and start the app against the real configured services (not mocks)
   - Exercise ONE minimal happy-path flow appropriate to the project type:
     * For HTTP services: hit a documented health or identity endpoint
     * For CLI or library projects: run the tool's own self-test if it has one
     * For background workers: verify the worker starts and registers
   - Confirm one data-read operation returns a structured response
     (even if empty — proves the round-trip to the backend works)
   - Shutdown cleanly after verification
   ```

2. **Interpret result:**
   - **PASS** → Phase 5 done
   - **FUNC_FAIL** → data gap (schema exists but no reference data, or reference data incomplete) — NOT a code bug. Report; do not block overall success
   - **FAIL** → runtime error (app crashes on start, handler throws, backend connection fails at app-level despite Phase 4 passing) — this is the signal that Phase 4 connectivity check missed something. Report the error verbatim.

### Phase 6: Write env-setup log

Write `tracking/env-setup.md`:

```markdown
# Environment Setup Log

## Run: [timestamp]

| Phase | Status | Notes |
|---|---|---|
| 1. Configuration validation | PASS | [format] at [path(s)] — all N required values present |
| 2. Schema / migrations | PASS | [tool-detected] applied N migrations |
| 3. Reference seed | SKIPPED | [reason — e.g., no prod-safe seed found; see detail below] |
| 4. Service connectivity | PARTIAL | M of N services reachable |
| 5. Runtime smoke | SKIPPED | -skip-smoke flag |

## Service Connectivity

| Service | Category | Endpoint / Identifier | Result |
|---|---|---|---|
| [SERVICE_1] | [category] | [derived from env, no secret] | PASS |
| [SERVICE_2] | [category] | [derived from env, no secret] | FAIL — [reason] |
| ... | ... | ... | ... |

## Remaining Actions

- [Specific fix for each FAIL row, pointing at CONFIG_GUIDE.md sections]
- [Any SKIPPED items that need human follow-up]
```

This file is append-only history — previous runs are preserved as numbered `## Run:` blocks.

### Phase 7: Final Summary

Print to user:

```
SETUP-ENV COMPLETE
==================
Configuration validation: PASS
Schema / migrations: [PASS/FAIL/SKIPPED — tool: {detected}]
Reference seed: [PASS/FAIL/SKIPPED]
Service connectivity: [M of N services reachable]
Runtime smoke: [PASS/FUNC_FAIL/FAIL/SKIPPED]

Overall: [READY / NEEDS ATTENTION]

[If NEEDS ATTENTION:]
Fix the items flagged above (details in tracking/env-setup.md), then:
  /setup-env -verify-only     (to re-check without re-running migrations/seeds)

[If READY:]
Start the app using your project's dev command (see CONFIG_GUIDE.md Quick Start).
```

Exit code: 0 if overall READY; non-zero if any hard failure in Phase 1, 2, or an unrecovered FAIL in Phase 5.

## Outputs

Files created or modified:
- `tracking/env-setup.md` — run log with per-phase status, service connectivity results, and remaining actions (append-only history; each run adds a new `## Run: [timestamp]` block)

Side effects on the project's configured backends (this is where `/setup-env` differs from every other command — it intentionally mutates external state):
- **Database schema** — migrations applied via the detected migration tool (tables/collections/indexes/constraints created or altered per the project's migration history)
- **Reference data** — production-safe seed rows inserted (taxonomy / enums / lookup tables only — never user/content data)
- **External service verification** — authenticated read-only calls to each declared external service (no writes — health/ping/list-style calls only)
- **Runtime smoke test** — starts and shuts down the app/service for a minimal happy-path probe (no persistent state change beyond what's already been applied)

What `-verify-only` restricts the above to:
- NO migrations applied, NO seed rows inserted
- Only configuration validation and connectivity checks
- Safe to run repeatedly against a live environment for health diagnostics

Files NEVER touched:
- Source code (`src/`), architecture (`architecture/`), module tracking (`tracking/module-tracking.md`)
- Anything under `poc/`
- Project configuration files at root (reads `.env` or equivalent, never writes)

## Rules

1. **Framework-agnostic** — Make no assumptions about which database, ORM, auth provider, AI provider, cache, queue, storage backend, or any external service the project uses. All detection comes from the project's own files (`TECHSTACK.md`, `architecture/architecture.md`, `CONFIG_GUIDE.md`, on-disk signals under `src/`).
2. **Never logs secret values** — only variable names, service names, hostnames / identifiers derived from env vars, HTTP status codes, and derived outcomes.
3. **Production-safe seed discipline** — refuses to run dev seeds that insert sample user content. If the project has only a dev seed, skips with a clear explanation.
4. **Idempotent where possible** — re-running `/setup-env` after a partial failure should be safe. Migrations are tool-managed (the project's chosen tool tracks applied migrations); reference seeds should use `upsert` patterns by convention.
5. **Separation of concerns** — `/setup-env` does runtime setup; does not modify code or architecture. Code issues surface from `/promote-poc`; infrastructure provisioning is a human step per `CONFIG_GUIDE.md`; deployment is future `/deploy-module`.
6. **`-verify-only` is non-mutating** — runs only validation + connectivity, skips migrations and seeds. Safe to run against a live production environment for health checks.
7. **Service-failure tolerance** — a single external service failure does not abort the whole run; the report summarizes all failures so the user can fix them in batch.
8. **Never touches `poc/`** — POC is historical after promotion.
9. **Never creates cloud infrastructure** — all provider account / project / tenant / bucket / database creation is delegated to the human via `CONFIG_GUIDE.md`.
10. **Tool detection is declarative, not prescriptive** — `TECHSTACK.md` is the authority. On-disk signals are fallback evidence when `TECHSTACK.md` is silent. If signals and `TECHSTACK.md` conflict, surface the conflict and ask the user.

## Agents Used

| Agent | Purpose | Invocation |
|-------|---------|------------|
| `smoke-test-agent` | Runtime smoke test against real services | Phase 5 — unless `-skip-smoke` or `-verify-only` |

**Agents NOT used:**
- `coding-agent` — does not modify code
- `code-promotion-analyzer-agent` — out of scope (POC promotion is done)
- `unit-tester-agent`, `l2-integration-agent` — test gates are owned by `/promote-poc` and `/modify`
- `tracking-update-agent` — module tracking is immutable post-promotion; `/setup-env` writes its own `tracking/env-setup.md`

## Core Requirements

- **MUST** refuse to proceed without valid production configuration file(s) at project root (format and paths detected per the project's convention)
- **MUST** detect and use the project's own declared tooling (from `TECHSTACK.md`) in preference to any built-in defaults
- **MUST** fall back gracefully when a required tool is not detected (ask, do not guess)
- **MUST** distinguish production-safe seeds from dev seeds and skip the latter
- **MUST** verify connectivity to every external service declared in `architecture/architecture.md`
- **MUST** derive verification calls from the project's own SDK / client setup, not from hardcoded provider knowledge
- **MUST** log outcomes without leaking secrets
- **MUST** write a complete run log to `tracking/env-setup.md`
- **MUST NOT** provision cloud infrastructure (provider accounts, DBs, buckets, tenants)
- **MUST NOT** modify code, architecture, `tracking/module-tracking.md`, or anything under `poc/`
- **MUST NOT** deploy the application
- **MUST NOT** embed vendor-specific names, commands, endpoints, or assumptions in its own logic — every specific is looked up from the project's declared stack and architecture
