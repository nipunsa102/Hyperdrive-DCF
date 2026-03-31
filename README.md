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
                                    Main Path
Requirements ──→ PRD ──→ Architecture ──→ Module Specs ──→ Code ──→ Deployment
                  ↑            ↑               ↑
                  │            └─── Promote ────┘
                  │             POC Design
                  │           (merge + validate
                  │            + promotion plan)
                  │                 ↑
               Sync PRD ←── Modify POC ←── Generate POC
              (changelog)  (stakeholders)   (mock data)
```

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
| `/promote-poc-design` | Merges POC architecture into main architecture, validates coherence, and produces a code promotion plan. See [Design Promotion](#design-promotion--from-poc-to-production) |

### How change tracking works

Every `/modify-poc` cycle maintains two files:

- **`poc/change-tracking.md`** -- simple append-only log in the PM's own words. Human-readable, not consumed by any command. Purely a record of what was requested.
- **`poc/changelog.md`** -- intelligent living document. Each entry maps to PRD requirement numbers (REQ-IDs). When the same feature is modified multiple times across feedback rounds, the changelog **merges updates into one entry** with the latest state -- no duplicate logs. This is what `/sync-prd` reads.

Because the changelog maintains the latest state per feature rather than accumulating redundant entries, syncing back to the PRD is clean -- the AI compares the current state of each entry against the existing PRD and applies updates without confusion.

### How sync works

When stakeholders are satisfied and the POC is validated:

1. `/sync-prd` reads `poc/changelog.md` (not change-tracking.md)
2. Compares each entry against existing PRD requirements
3. Updates existing user stories and acceptance criteria for `[CHANGE]` and `[FIX]` entries
4. Adds new user stories for `[NEW]` entries (features that didn't exist in the original PRD)
5. Deprecates the old PRD (preserved for audit) and writes the new PRD as the active document
6. Archives the changelog with a matching timestamp

### Who does what

```
Product Manager    writes OVERVIEW.md, runs /generate-prd, demos POC,
                   captures stakeholder feedback via /modify-poc

Engineering        decides tech stack (TECHSTACK.md), takes over after
                   PRD is validated, runs /generate-code

Stakeholders       review POC demos, provide feedback -- never touch
                   the codebase
```

The engineering team is needed at two points: initially to define the tech stack, and later once the POC is validated and the PRD is updated. Everything in between is driven by the product manager.

## Design Promotion -- From POC to Production

After `/sync-prd` updates the PRD with validated POC changes, the main architecture and module specs are out of date. `/promote-poc-design` reconciles them: it merges the POC's architecture into the main architecture, runs multi-layered validation to ensure nothing is lost or inconsistent, and produces a code migration plan -- all before a single line of production code is written.

The pipeline runs through 8 phases: prerequisites + sync guard + POC scope detection (frontend-only, backend-only, full-stack, or undetermined), backup, architecture & module merge via the `re-architect-agent`, POC incorporation check, structural validation (REQ-ID traceability, cycle detection, sum test), semantic validation via the `coherence-checker-agent`, code promotion plan via the `code-promotion-analyzer-agent`, and a summary report. Each validation layer catches a different class of error -- from silent feature drops to semantic inconsistencies -- providing defense-in-depth at design time rather than implementation time.

> For the full phase-by-phase reference including sub-agent internals, see [`POC_PROMOTION_ALGORITHM.md`](POC_PROMOTION_ALGORITHM.md).

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
| 2b | `/sync-prd` | *(Optional)* Merges validated POC changes back into the PRD |
| 2c | `/promote-poc-design` | *(Optional)* Merges POC architecture into main architecture, validates coherence, produces code promotion plan. **If you use the POC path (2a-2c), skip Step 3** -- `/promote-poc-design` creates the module specs via merge, and running `/generate-modules` afterward would overwrite them |
| 3 | `/generate-modules` | Extracts module specs from architecture into `architecture/modules/`. Skipped if you used the POC path (Steps 2a-2c). Each module includes a Requirement Coverage table, user stories, acceptance criteria, and optional technical details (pseudo-code, schemas, API contracts) where they add clarity. Also generates the Module Registry and Integration Matrix in `architecture.md` |
| 4 | `/generate-code` | Implements modules as production code with mandatory **L1 (unit, 60% coverage)** and **L2 (integration)** test gates. Fails the pipeline if gates are not met |
| 5 | `/deploy-module` | Generates deployment configuration and deploys a module to cloud infrastructure |

#### `/generate-code` options

| Switch | Description |
|--------|-------------|
| `-module N` | Implement a specific module (e.g., `-module 3`). Omit to process all modules in dependency order |
| `-special "..."` | Special implementation requirements |
| `-max-attempts N` | Max test-fix cycles before giving up (default: 5) |
| `-review` | Enable optional code review after all modules pass L1 + smoke |
| `-skip-smoke` | Skip smoke tests (not recommended, use only for foundational modules) |
| `-retrofit` | Enable retrofit mode using `POC_CODE_PROMOTE_PLAN.md` from `/promote-poc-design`. Passes per-module POC context (REFACTOR/REWRITE/WRITE NEW decision, files, gaps, migration steps) to the coding agent. Falls back to normal mode if the plan file is missing |

#### `/deploy-module` options *(coming soon)*

| Switch | Description |
|--------|-------------|
| `-module M1` | Module ID to deploy (required) |
| `-provider aws\|azure\|gcp\|local` | Target cloud provider (default: `local`) |
| `-environment dev\|staging\|prod` | Target environment (default: `dev`) |
| `-dry-run` | Generate config only, skip actual deployment |

### Maintenance

| Command | What it does |
|---------|--------------|
| `/update-tracking` | Update module status in `tracking/module-tracking.md` |

Options: `-module M1` (specific module), `-status <status>` (one of `not_started`, `in_progress`, `l1_pass`, `blocked`, `complete`, `deployed`)

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
│   └── modules/               # Module specifications
├── tracking/                  # Module status tracking
│   └── module-tracking.md     # Source of truth for module status
├── src/                       # All source code
│   ├── app/                   # Frontend (build configs colocated here)
│   ├── server/                # Backend
│   └── shared/                # Shared code
├── tests/
│   ├── unit/                  # L1 tests (60% coverage gate)
│   ├── integration/           # L2 tests (blocking gate)
│   └── e2e/                   # L3 tests (non-blocking)
├── poc/                       # POC output (self-contained, isolated)
├── infra/                     # Infrastructure configs
├── OVERVIEW.md                # Your requirements (input)
├── TECHSTACK.md               # Technology stack (input)
└── PRD.md                     # Generated structured requirements
```

## Test Gates

DCF enforces three blocking test gates during code generation:

- **L1 (Unit)** -- 60% code coverage minimum per module
- **Smoke Test** -- Application starts and basic functionality responds (applies to runnable modules; skipped for foundational modules like data models or utilities)
- **L2 (Integration)** -- Cross-module validation per the Integration Matrix

Code generation will not proceed if these gates fail. L3 (E2E) tests are non-blocking.

## Key Documents

| Document | Role |
|----------|------|
| `OVERVIEW.md` | Freeform requirements (you write this) |
| `TECHSTACK.md` | Technology choices (you write this) |
| `PRD.md` | Structured requirements with REQ-IDs (generated) |
| `architecture/architecture.md` | High-level design with Integration Matrix (generated) |
| `architecture/modules/*.md` | Module specs with pseudo-code and API contracts (generated) |
| `tracking/module-tracking.md` | Module implementation status (auto-updated) |
| `DESIGNGUIDE.md` | UI/UX and design constraints (you write this, optional) |
| `POC_CODE_PROMOTE_PLAN.md` | Per-module REFACTOR/REWRITE/WRITE NEW migration plan (generated by `/promote-poc-design`) |
| `DCF.md` | Technical reference -- command map, agent map, artifact map, authority chain, retrofit mode |
| `POC_PROMOTION_ALGORITHM.md` | Phase-by-phase reference for the `/promote-poc-design` pipeline |

## Skills

The `.claude/skills/` directory can hold optional [Skills](https://code.claude.com/docs/en/skills#extend-claude-with-skills) that extend Claude Code with specialized capabilities. Skills are not required for the core DCF workflow but can provide domain-specific knowledge for implementation tasks (e.g., editor integrations, UI component patterns). Add your own by placing `.md` files in `.claude/skills/`.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on extending the framework.

## License

[MIT](LICENSE)
