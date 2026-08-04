# Hyperdrive DCF

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

**Design Cascading Framework** -- a top-down methodology for rapid POC generation and idea validation, with every change tracked and synced back to the requirements spec with precision. Validated requirements then cascade through architecture and module specs to generate fully traced, test-gated production code. Powered by Claude Code.

## Why DCF?

Most AI-assisted development starts with code and hopes a coherent architecture emerges. DCF flips this:

- **Architecture-first**: Design decisions are made upfront, not discovered during debugging
- **Full traceability**: Every line of code traces back to a requirement via REQ-IDs
- **No scope creep**: The Sum Test guarantees modules cover the architecture exactly -- nothing more, nothing less
- **Repeatable**: The same workflow works for any project, any tech stack, any team size
- **Rapid prototyping**: Generate a navigatable POC, validate with stakeholders, iterate with change tracking, and sync validated changes back to the PRD -- all before engineering begins

```
                                        Main Path (direct)
Requirements ──→ PRD ──→ Architecture ──→ Deployment Plan ──→ Module Specs ──→ Code ──→ Deployment
                  ↑           ↑           (human decisions)        ↑            ↑        (future)
                  │           │                                    │            │
                  │           └─────────── /promote-poc ───────────┴────────────┘
                  │             (merge + validate + implement + test)
                  │                             ↑
                  │                       Prepare Promo
                  │                      (gap decisions)
                  │                             ↑
               Sync PRD ←── Modify POC ←── Generate POC
              (changelog)  (stakeholders)   (mock data)
```

On the **direct path**, `/plan-deployment` inserts a human decision gate between architecture and module design: it generates `DEPLOYMENT.md` (deployment & environment decisions), and `/generate-modules` / `/generate-code` refuse to run until every decision in it is filled — see [Deployment Planning](#deployment-planning-direct-path). The pipeline ends with the app **running in a bootstrapped dev environment**; production deployment is the future `/deploy-to-prod` (not yet implemented).

Every artifact is **derived** from the one above it. Modules are extracted from architecture, not invented. The Sum Test guarantees completeness:

```
Module 1 + Module 2 + ... + Module N = architecture.md (exactly)
```

### DCF vs Traditional Development

| Traditional | DCF |
|---|---|
| Start coding, design emerges | Design first, code follows |
| Features scattered across codebase | Each feature in exactly ONE module |
| Integration issues discovered late | Integration Matrix defined upfront |
| Scope creep during development | Modules sum to architecture EXACTLY |
| Stakeholder feedback after build | Stakeholder validation BEFORE engineering |

## POC Workflow -- Validate Before You Build

The POC workflow lets product managers and non-technical stakeholders validate ideas with a working prototype before committing to production development. PMs capture stakeholder feedback, iterate on the POC, and sync validated changes back to the PRD through a change-tracked process where nothing gets dropped or forgotten.

| Command | What it does |
|---------|--------------|
| `/generate-poc` | Generates a navigatable proof-of-concept with mock data and mock integrations. Self-contained in `poc/` directory |
| `/modify-poc` | Implements stakeholder-requested changes to the POC with full tracking |
| `/sync-prd` | Merges validated POC changelog entries back into a new PRD document |
| `/prepare-poc-promo` | Generates a gap analysis template at `poc/temp/poc_promotion/POC_PROMO_PREP.md` for human decisions on data, auth, secrets, integrations, and deployment |
| `/promote-poc` | Merges POC architecture, implements production code via coding-agent with L1/L2 test gates, and produces `POC_PROMOTION_REPORT.md`. See [Design Promotion](#design-promotion--from-poc-to-production) |

### How change tracking works

Every `/modify-poc` cycle maintains two files:

- **`poc/poc-tracking/change-tracking.md`** -- simple append-only log in the PM's own words. Human-readable, not consumed by any command. Purely a record of what was requested.
- **`poc/poc-tracking/CHANGELOG.md`** -- intelligent living document. Each entry maps to PRD requirement numbers (REQ-IDs). When the same feature is modified multiple times across feedback rounds, the changelog **merges updates into one entry** with the latest state -- no duplicate logs. This is what `/sync-prd` reads.

Because the changelog maintains the latest state per feature rather than accumulating redundant entries, syncing back to the PRD is clean -- the AI compares the current state of each entry against the existing PRD and applies updates without confusion.

### How sync works

When stakeholders are satisfied and the POC is validated:

1. `/sync-prd` reads `poc/poc-tracking/CHANGELOG.md` (not `change-tracking.md`)
2. Compares each entry against existing PRD requirements
3. Updates existing user stories and acceptance criteria for `[CHANGE]` and `[FIX]` entries
4. Adds new user stories for `[NEW]` entries (features that didn't exist in the original PRD)
5. Deprecates the old PRD (preserved for audit) and writes the new PRD as the active document
6. Archives the changelog with a matching timestamp

### Who does what

```
Product Manager    writes OVERVIEW.md, runs /generate-prd, demos POC,
                   captures stakeholder feedback via /modify-poc

Engineering        decides tech stack (TECHSTACK.md), takes over once the
                   PRD is validated -- fills the decision template
                   (POC_PROMO_PREP.md here; DEPLOYMENT.md on the direct
                   path) and runs the promotion / code-generation pipeline

Stakeholders       review POC demos, provide feedback -- never touch
                   the codebase
```

The engineering team is needed at two points: initially to define the tech stack, and later once the POC is validated and the PRD is updated. Everything in between is driven by the product manager.

## Design Promotion -- From POC to Production

After `/sync-prd` updates the PRD with validated POC changes, the human fills in production decisions via `/prepare-poc-promo`, and then `/promote-poc` runs the full promotion pipeline: it classifies POC maturity, merges the POC's architecture into the main architecture, runs multi-layered validation, analyzes the POC code for per-module promotion strategy (AS_IS / ADAPT / REWRITE), implements all modules as production code, runs mandatory L1/L2 test gates, applies cross-cutting configuration decisions uniformly across the codebase, and produces `CONFIG_GUIDE.md` + `POC_PROMOTION_REPORT.md` + a production configuration template. The command runs on a separate git branch -- the main branch is the backup.

The pipeline runs through 5 phases: preflight (prerequisites + POC maturity classification + resume detection), architecture merge and validation (POC architecture merged into main, bootstraps production modules, structural and semantic validation), code promotion planning (per-module decisions based on the POC's actual maturity), implementation and test gates (module-by-module execution, cross-cutting decision sweep, L2 integration tests), and report generation with config guide and configuration template. Each validation layer catches a different class of error -- from silent feature drops to semantic inconsistencies -- providing defense-in-depth before code is written. A running log at `poc/temp/poc_promotion/promotion.md` enables resume on session failure.

> For the full phase-by-phase reference, see [`POC_PROMOTION_ALGORITHM.md`](POC_PROMOTION_ALGORITHM.md).

### Three-Tier Decision Model

For each module, the promotion analyzer assigns exactly one of:

- **`AS_IS`** -- POC code already satisfies the spec (poc_coverage ≥ 0.80, framework identical to production, clean or no mocks). The invoking context copies the POC files verbatim and applies a short list of surgical edits. No code regeneration.
- **`ADAPT`** -- POC structure is sound but needs real-service wiring, auth/error handling, or framework adaptation. The promoted code starts from the POC and adapts.
- **`REWRITE`** -- POC is reference only (poc_coverage < 0.30, embedded mocks, or framework mismatch). Fresh code is written from the module spec, preserving POC's visual/behavioral patterns where they align.

The analyzer is passed a **POC maturity prior** (`mock-heavy` / `hybrid` / `production-wired`) so its decision distribution can be sanity-checked against the POC's actual nature.

### Gated Workflow

`/generate-poc` hard-fails if `architecture/modules/` is non-empty -- the POC path is for greenfield work where `/promote-poc` bootstraps modules from the POC. If production modules already exist, use Path A (`/generate-modules -> /generate-code`) instead. This separation keeps promotion's code path simple: at promotion time, `src/` and `architecture/modules/` are always empty, so the three-tier decision model is sufficient (no KEEP EXISTING / WRITE NEW tiers needed).

### After Promotion -- Making It Runnable

`/promote-poc` produces the code and a config template, but does not touch real credentials or run migrations. After the human fills in production configuration (per `CONFIG_GUIDE.md`), `/setup-env` detects the project's migration tool, applies schema migrations, seeds reference data (taxonomy / enums only -- never sample user content), verifies connectivity to every declared external service, and runs a runtime smoke test against real backends.

## Deployment Planning (Direct Path)

The POC path captures production environment decisions in `POC_PROMO_PREP.md` via `/prepare-poc-promo`. The **direct path** has its own decision gate: after `/generate-architecture`, run `/plan-deployment`. It invokes the `deployment-gap-analyzer-agent` to scan the PRD, architecture, and data model for every environment-coupled decision the design demands but no project document answers, then generates `DEPLOYMENT.md` at project root -- a vendor-agnostic template covering deployment target & compute (DEP), data & storage (DATA), identity & access (AUTH), secrets & configuration (SEC), external integrations (INT), and build/delivery/operations (OPS), always including the **dev run mode** decision (fully local / cloud / hybrid).

```
/generate-architecture
         │
         ▼
/plan-deployment ─────→ DEPLOYMENT.md  (template with <!-- REQUIRED --> placeholders)
         │
         ▼   [you fill in every decision -- re-run to enrich & read-only-verify the resource plan]
         │
/generate-modules ──→ /generate-code   (both REFUSE to run while any decision is unfilled)
```

- **The document is the questionnaire.** Decisions your project documents already answer (`TECHSTACK.md`, `OVERVIEW.md`, ...) are pre-filled with the source cited; everything else is a `<!-- REQUIRED -->` placeholder for you. No interactive Q&A.
- **Downstream commands hard-block on it.** `/generate-modules` and `/generate-code` scan `DEPLOYMENT.md` and refuse to run while the file is missing or any placeholder is unfilled -- so module specs and code are always generated for a known target environment, never environment-blind.
- **Direct path only.** `/plan-deployment` hard-fails if `poc/src/` exists; on the POC path the same decisions flow through `/prepare-poc-promo` → `/promote-poc` instead. The two flows never mix.
- **Read-only toward your platform.** Re-run after choosing a provider and it derives the Resource & Naming Plan and verifies names, regions, and conventions via read-only CLI calls only. Dev resources get created later by `/generate-code`'s environment bootstrap; production resources wait for the future `/deploy-to-prod`.

## Installation

1. Copy the `.claude/` directory into the root of your project:

   ```bash
   cp -r path/to/hyperdrive-dcf/.claude/ your-project/.claude/
   ```

2. Create the required input files at your project root:
   - `OVERVIEW.md` -- your freeform requirements (see template)
   - `TECHSTACK.md` -- your technology choices (see template)

3. Add `CLAUDE.md` to your project root (copy from this repo and customize as needed).

4. Start a Claude Code session and run the workflow commands in order.

5. (Optional) Create `.claude/settings.local.json` for your local environment settings (permissions, allowed tools, etc.). This file is gitignored and won't be committed.

### Prerequisites

- [Claude Code CLI](https://code.claude.com/docs/en/overview) installed and configured
- A project with an `OVERVIEW.md` describing your requirements
- A `TECHSTACK.md` defining your technology choices

## Command Reference

All commands are run as slash commands inside a Claude Code session.

### DCF Workflow (run in order)

| Step | Command | What it does |
|------|---------|--------------|
| 1 | `/generate-prd` | Transforms `OVERVIEW.md` into a structured `PRD.md` with hierarchical REQ-IDs for full traceability |
| 2 | `/generate-architecture` | Generates `architecture/architecture.md` with high-level components, data flows, and user journeys. Each component maps back to REQ-IDs via `Implements:` tags |
| 2a | `/generate-poc` | *(Optional)* Generates a navigatable POC with mock data for stakeholder validation |
| 2b | `/modify-poc` | *(Optional, repeat)* Implements stakeholder-requested changes with full tracking |
| 2c | `/sync-prd` | *(Optional)* Merges validated POC changes back into the PRD |
| 2d | `/prepare-poc-promo` | *(Optional)* Generates gap analysis template for human production decisions |
| 2e | `/promote-poc` | *(Optional)* Merges POC architecture, analyzes code, implements all modules with test gates, produces `POC_PROMOTION_REPORT.md`. **If you use the POC path (2a-2e), skip Steps 3-5** -- `/promote-poc` creates module specs, implements code, and runs test gates |
| 3 | `/plan-deployment` | *(Direct path only — skipped on the POC path, where `POC_PROMO_PREP.md` plays this role)* Generates `DEPLOYMENT.md`: a vendor-agnostic deployment & environment decision template (hosting/compute model, environments, identity wiring, data platform access, secrets/config, integration tenancy) with `<!-- REQUIRED -->` placeholders. You fill in the decisions; Steps 4-5 refuse to run until complete. Re-run after choosing a provider to derive and read-only-verify the resource naming plan |
| 4 | `/generate-modules` | Extracts module specs from architecture into `architecture/modules/`, consuming the `DEPLOYMENT.md` decisions where they shape module design. Skipped if you used the POC path (Steps 2a-2e). Each module includes a Requirement Coverage table, user stories, acceptance criteria, and optional technical details (pseudo-code, schemas, API contracts) where they add clarity. Also generates the Module Registry and Integration Matrix in `architecture.md` |
| 5 | `/generate-code` | The direct path's orchestrator. Bootstraps the dev environment declared in `DEPLOYMENT.md` (fully local / cloud / hybrid), implements modules with mandatory **L1 (unit, 60% coverage)** and **smoke** gates against that real environment, then gates on **L2 (integration)** and a blocking **E2E functional pass — headless-browser testing, mandatory for UI systems** — orchestrating its own fix loops. Ends with the app running in dev and `CONFIG_GUIDE.md` as the as-built record. Production deployment comes later via the future `/deploy-to-prod` |
| 6 | `/setup-env` | *(POC path only — after `/promote-poc` and filling `.env` per `CONFIG_GUIDE.md`.)* Validates `.env`, runs DB migrations, seeds reference data, verifies connectivity to every external service, and runs a runtime smoke test against real services. On the direct path this step doesn't exist — `/generate-code` bootstraps and verifies the environment itself |

#### `/generate-code` options

| Switch | Description |
|--------|-------------|
| `-module N` | Implement a specific module (e.g., `-module 3`). Omit to process all modules in dependency order |
| `-special "..."` | Special implementation requirements |
| `-max-attempts N` | Max test-fix cycles before giving up (default: 5) |
| `-review` | Enable optional code review after all modules pass L1 + smoke |
| `-skip-smoke` | Skip smoke tests (not recommended, use only for foundational modules) |

> **Note:** POC-to-production retrofit mode is handled by `/promote-poc`, not `/generate-code`. See the POC workflow (Steps 2a-2e) above.

#### Other command options

**`/generate-architecture`**

| Switch | Description |
|--------|-------------|
| `-special "..."` | Special requirements or constraints for architecture planning |

**`/generate-poc`**

| Switch | Description |
|--------|-------------|
| `-special "..."` | Special requirements or focus areas for the POC |

**`/modify-poc`** -- *exactly one of the following is required*

| Switch | Description |
|--------|-------------|
| `-change "..."` | Description of a modification to existing POC functionality, in the product manager's own words |
| `-new "..."` | Description of a new feature to add that did not exist in the original POC |
| `-fix "..."` | Description of something broken in the POC that needs fixing (e.g., "calendar page crashes when clicking next month") |

If zero or multiple switches are passed, the command errors out.

### Post-Promotion Changes

Run on a feature git branch — the branch history is the technical changelog.

| Command | What it does |
|---------|--------------|
| `/modify` | Implements stakeholder-requested changes to promoted production code. Edits `PRD.md`, `architecture/`, and `src/` directly; runs L1 + smoke + L2 gates; appends a CT-XXX entry to `tracking/change-tracking.md` |

`/modify` requires *exactly one of* the following switches:

| Switch | Description |
|--------|-------------|
| `-change "..."` | Modify existing production functionality |
| `-new "..."` | Add a feature absent from the current PRD |
| `-fix "..."` | Repair broken production behavior |

### Maintenance

| Command | What it does |
|---------|--------------|
| `/update-tracking` | Update module status in `tracking/module-tracking.md` |

Options: `-module M1` (specific module), `-status <status>` (one of `not_started`, `in_progress`, `l1_pass`, `blocked`, `complete`, `deployed`)

## Agents

Commands orchestrate specialized agents defined in `.claude/agents/` — agents are never invoked directly by the user. See the [Agent Map in DCF.md](DCF.md#agent-map) for exactly which command invokes which.

| Agent | Role |
|-------|------|
| `coding-agent` | Implements module code (production and POC modes) |
| `unit-test-generator-agent` | Generates L1 unit tests for a completed module |
| `unit-tester-agent` | Runs L1 tests and enforces the 60% coverage gate |
| `smoke-test-agent` | Verifies the app builds, starts, and responds — against the bootstrapped dev environment |
| `e2e-test-agent` | End-to-end functional verification against the running dev environment: headless-browser journeys (mandatory for UI systems) or API/CLI journeys |
| `l2-integration-agent` | Generates, runs, and fixes cross-module integration tests per the Integration Matrix |
| `code-review-agent` | Optional code quality review (`/generate-code -review`) |
| `traceability-validator-agent` | Validates REQ-ID coverage and the Sum Test across DCF documents |
| `coherence-checker-agent` | Deep semantic-consistency analysis of merged architecture, modules, and PRD |
| `deployment-gap-analyzer-agent` | Scans design documents for unanswered deployment & environment decisions (`/plan-deployment`) |
| `poc-gap-analyzer-agent` | Scans POC code vs architecture for POC-to-production gaps (`/prepare-poc-promo`) |
| `code-promotion-analyzer-agent` | Assigns each module its AS_IS / ADAPT / REWRITE promotion strategy (`/promote-poc`) |
| `re-architect-agent` | Merges POC architecture into the main architecture and bootstraps production modules (`/promote-poc`) |
| `tracking-update-agent` | Maintains module status in `tracking/module-tracking.md` |

## Project Structure

```
project-root/
├── .claude/                   # Hyperdrive DCF Framework (IMMUTABLE)
│   ├── commands/              # Slash command definitions
│   ├── agents/                # Agent configurations
│   ├── skills/                # Skill definitions (optional examples)
│   └── rules/                 # Project rules (auto-loaded)
├── architecture/              # Generated architecture docs
│   ├── architecture.md        # Master architecture
│   ├── data-model.md          # Logical data model (entities, relationships, constraints)
│   └── modules/               # Module specifications
├── tracking/                  # Project tracking
│   ├── module-tracking.md     # Source of truth for module status
│   ├── change-tracking.md     # Append-only PM log of post-promotion change requests
│   └── env-setup.md           # Runtime environment setup log
├── src/                       # All source code
│   ├── app/                   # Frontend (build configs colocated here)
│   ├── server/                # Backend
│   └── shared/                # Shared code
├── tests/
│   ├── unit/                  # L1 tests (60% coverage gate)
│   ├── integration/           # L2 tests (blocking gate)
│   └── e2e/                   # E2E journeys (headless browser for UI; blocking gate in /generate-code)
├── poc/                       # POC output (self-contained, isolated)
│   ├── architecture/          # POC-scoped architecture docs
│   ├── src/                   # POC source (UI-focused, all mocks)
│   ├── poc-tracking/          # POC change tracking (CHANGELOG.md, change-tracking.md)
│   └── temp/poc_promotion/    # POC_PROMO_PREP.md + promotion.md (during /promote-poc)
├── infra/                     # Infrastructure configs
├── OVERVIEW.md                # Your requirements (input)
├── TECHSTACK.md               # Technology stack (input)
├── PRD.md                     # Generated structured requirements
├── DEPLOYMENT.md              # Deployment & environment decisions (direct path — /plan-deployment template, you complete it)
└── CONFIG_GUIDE.md            # As-built configuration record + production walkthrough (generated)
```

## Test Gates

DCF enforces blocking test gates during code generation:

- **L1 (Unit)** -- 60% code coverage minimum per module
- **Smoke Test** -- Application starts and basic functionality responds against the bootstrapped dev environment (applies to runnable modules; skipped for foundational modules like data models or utilities)
- **L2 (Integration)** -- Cross-module validation per the Integration Matrix
- **E2E Functional Gate** *(direct path)* -- Up to 10 user journeys driven against the running dev environment; **headless-browser testing is mandatory for UI systems**. UI modules also get a per-module headless render + interaction check after smoke

Code generation will not proceed if these gates fail. Human-authored e2e tests beyond the gate remain non-blocking.

## Key Documents

| Document | Role |
|----------|------|
| `OVERVIEW.md` | Freeform requirements (you write this) |
| `TECHSTACK.md` | Technology choices (you write this) |
| `PRD.md` | Structured requirements with REQ-IDs (generated) |
| `DEPLOYMENT.md` | Deployment & environment decisions — vendor-agnostic template generated by `/plan-deployment`, completed by you (direct path only) |
| `architecture/architecture.md` | High-level design with components and requirement traceability (generated) |
| `architecture/data-model.md` | Logical data model with entities, relationships, and constraints (generated by `/generate-architecture`) |
| `architecture/modules/*.md` | Module specs with pseudo-code and API contracts (generated) |
| `tracking/module-tracking.md` | Module implementation status (auto-updated) |
| `tracking/change-tracking.md` | Append-only PM log of post-promotion change requests (maintained by `/modify`) |
| `tracking/env-setup.md` | Runtime environment setup log (generated by `/setup-env` on the POC path, or `/generate-code`'s environment bootstrap on the direct path) |
| `DESIGNGUIDE.md` | UI/UX and design constraints (you write this, optional) |
| `CONFIG_GUIDE.md` | Production configuration walkthrough — env vars, provider dashboards, verification (generated by `/promote-poc` or `/generate-code`) |
| `POC_PROMOTION_REPORT.md` | Promotion metadata + per-module summary + cross-cutting decisions (generated by `/promote-poc`) |
| `DCF.md` | Technical reference -- command map, agent map, artifact map, authority chain, retrofit mode |
| `POC_PROMOTION_ALGORITHM.md` | Phase-by-phase reference for the `/promote-poc` pipeline |

## Skills

The `.claude/skills/` directory can hold optional [Skills](https://code.claude.com/docs/en/skills#extend-claude-with-skills) that extend Claude Code with specialized capabilities. Skills are not required for the core DCF workflow but can provide domain-specific knowledge for implementation tasks (e.g., editor integrations, UI component patterns). Add your own by placing `.md` files in `.claude/skills/`.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on extending the framework.

## License

[MIT](LICENSE)
