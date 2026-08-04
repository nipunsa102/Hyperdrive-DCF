---
description: Generate Code - Module Implementation
model: claude-opus-4-8
---

**Switches**: `-module`, `-special`, `-max-attempts`, `-review`, `-skip-smoke`

**Switch Definitions**:
- `-module` → Specific module number to implement (e.g., -module 3). If omitted, processes ALL modules in dependency order.
- `-special` → Special implementation requirements or considerations
- `-max-attempts` → Maximum number of test-fix cycles (default: 5)
- `-review` → Enable optional code review (adds time but improves quality)
- `-skip-smoke` → Skip smoke tests (not recommended, use only for foundational modules)

> **Note:** Retrofit mode (POC-to-production promotion) has moved to `/promote-poc`, which handles architecture merge, code analysis, and implementation in a single command. If `POC_CODE_PROMOTE_PLAN.md` exists at project root, it is from a previous version — use `/promote-poc` instead.

## Purpose

Implements modules directly from architecture specifications — and delivers them into a **running, verified dev environment**. Reads module dependency order from the Integration Matrix in architecture.md, bootstraps the dev environment chosen in `DEPLOYMENT.md` (fully local / cloud / hybrid), implements each module with mandatory L1/smoke gates against that real environment, and finishes with the L2 integration gate plus a **blocking E2E functional gate** — headless-browser testing, mandatory for UI systems.

**This command is the orchestrator**: environment setup, implementation, testing, and every fix loop happen here. When it completes, the app is running in the chosen dev mode. There is no separate environment-setup step on this path; production deployment comes later via the future `/deploy-to-prod`.

## Core Principles

1. **Modules are the ONLY work unit** - No capabilities, no iterations
2. **One Module at a Time** - Complete focus until done
3. **Test per Module** - Each module has its own test suite
4. **Dependencies Respected** - Never implement before dependencies
5. **L2 Validates Integration** - Cross-module validation after all modules complete
6. **Real Environment From the Start** - Modules are verified against the bootstrapped dev environment (per `DEPLOYMENT.md`'s dev run mode), not mocks
7. **Seen Working, Not Assumed Working** - A blocking E2E gate drives the app the way a user would; headless-browser testing is mandatory when there is a UI

## Prerequisites

1. Architecture must exist (`/architecture/architecture.md`)
2. Module specifications must exist (`/architecture/modules/module-{N}-{name}.md`)
3. Integration Matrix must be available in architecture.md
4. **Normal mode only.** Retrofit mode (POC promotion) is handled by `/promote-poc`.
5. `DEPLOYMENT.md` must exist at project root with **no unfilled `<!-- REQUIRED` markers** (run `/plan-deployment` and fill in all decisions first — pre-filled decisions count as filled). If missing or incomplete, ERROR and STOP:

   ```
   ERROR: DEPLOYMENT.md is missing (or has unfilled decisions):
   - [Gap ID]: [question text]
   Code must be generated for a known target environment (identity source, data
   access model, configuration keys, process topology, build shape). Run
   /plan-deployment, fill in every <!-- REQUIRED: ... --> placeholder, then
   re-run /generate-code.
   ```

   (POC-path projects never hit this — they use `/promote-poc`, which reads `poc/temp/poc_promotion/POC_PROMO_PREP.md` instead.)

## Process

### Step 0: Read Architecture

1. **Read `/architecture/architecture.md`**
2. **Parse Integration Matrix** for module dependency order
3. **Get list of modules to implement:**
   - If `-module N` specified: Only module N
   - Otherwise: All modules in dependency order from Integration Matrix
4. **Read `DEPLOYMENT.md`** — build a **Deployment Context** summary passed to every coding-agent invocation:
   - Compute model & deployable topology (DEP-*) → process shape, host/port binding, static-serving strategy, entry point
   - Data platform & access model (DATA-*) → driver/dialect wiring, credential mechanism (password vs. platform-managed identity vs. developer-sign-in token), migration layout, initial-data source availability
   - Identity decisions (AUTH-*) → identity source per run mode, local-dev identity, deployed-environment guard, role-mapping injection via configuration
   - Configuration mechanism & Configuration Key Plan (SEC-*) → config module expectations and canonical key names (never invent alternate names for planned keys)
   - Integration tenancy (INT-*) and build/runtime expectations (OPS-*)

**Path Configuration:**
- Default: `/architecture/`, `/tracking/`

---

### Step 0.5: Environment Bootstrap (BLOCKING)

Makes the dev environment real BEFORE any module is implemented, so every gate below runs against it. Everything is driven by `DEPLOYMENT.md` — nothing here is vendor-hardcoded. Idempotent: on re-run (e.g., `-module N` on an already-bootstrapped project), verify instead of re-create and continue.

1. **Determine the dev run mode** from `DEPLOYMENT.md`'s DEP decisions:

   | Mode | App process | Services (DB, external) | In-loop verification target |
   |---|---|---|---|
   | Fully local | dev machine | local (embedded, local containers, local installs) | local URL |
   | Hybrid | dev machine | cloud dev resources | local URL against cloud services |
   | Cloud | cloud dev host | cloud dev resources | local URL during the loop; cloud dev URL re-verified at the E2E gate (Step 3.5) |

2. **Provision or verify dev services** per the Resource & Naming Plan in `DEPLOYMENT.md`:
   - Fully local: set up whatever local services `TECHSTACK.md` implies (embedded DB file, local container, local install)
   - Hybrid/cloud: verify each planned DEV resource exists; create the missing ones with the provider's CLI (must already be installed and authenticated — never store or prompt for raw credentials). Verify-before-create, idempotent, **dev-scoped ONLY**. NEVER create production resources — that is the future `/deploy-to-prod`.
   - If the CLI is missing/unauthenticated, or creation fails: STOP with the exact error and what the human must do.

3. **Write the dev configuration file(s)** in the project's convention (per `TECHSTACK.md` + `DEPLOYMENT.md`'s Configuration Key Plan):
   - Fill every derivable value (hosts, ports, resource names, group/tenant IDs — from provisioning outputs and the plan)
   - Identify human-owned values (third-party API keys, true secrets). If a required one is missing: STOP and list each missing key with where to obtain it (per the plan's "Source of value" column and the INT/SEC decisions). Never write secret values into any document or log.

4. **Verify connectivity** to every external service the architecture declares, using the project's own driver/SDK access patterns (per-service status lines; never log secrets; report all failures before stopping).

5. **Migrations & seed are NOT applied here** — they run inside the module loop, immediately after the schema-owning module passes L1 (Step 1.5), because the schema code must exist first. Later modules that add migrations re-apply idempotently.

6. **Log the bootstrap** to `tracking/env-setup.md` (same append-only `## Run:` block format `/setup-env` uses on the POC path).

If bootstrap cannot complete, STOP — do not implement modules against an environment that doesn't exist.

---

### Step 1: Module Implementation Loop

**FOR EACH module (in dependency order):**

#### 1.1 Read Module Spec
- **Read module specification** from `/architecture/modules/module-{N}-{name}.md`
- **Understand complete module scope** and all responsibilities
- **Check module dependencies** from Integration Matrix
- **Define success criteria**: What makes this module "complete"?

#### 1.2 Implementation Phase
- **MUST INVOKE `coding-agent`** with full module context:
  ```
  - Module ID: M{N}
  - Module Name: {Module Name}
  - Development Spec: /architecture/modules/module-{N}-{name}.md
  - Dependencies: [M1, M2, ...] (from Integration Matrix)
  - Coverage Target: 60%
  - Deployment Context: [the DEP/DATA/AUTH/SEC/INT/OPS decisions from DEPLOYMENT.md
    relevant to this module — dev run mode (fully local / hybrid / cloud), identity
    mode(s) and guard, config keys/mechanism, data access model, host binding, build shape]
  ```

- Coding-agent generates complete module implementation
- All components, services, and internal logic in one pass

#### 1.3 Test Generation Phase
- **MUST INVOKE `unit-test-generator-agent`** for complete module
  - Coverage Target: 60%
- Generate **maximum 5 tests** covering critical module functionality
- Focus on public API surface, not implementation details

#### 1.4 L1 Unit Test Execution Phase
- **MUST INVOKE `unit-tester-agent`** to execute module tests
  - Coverage Target: 60%

- **SUCCESS PATH (Tests Pass with 60%+ coverage)**:
  - Record test metrics (coverage %, pass rate)
  - Proceed to smoke test

- **FAILURE PATH (Tests Fail or Coverage < Coverage Target)**:
  - Enter **L1 Repair Loop** (Max 5 attempts):
    1. Analyze failure (missing logic, incorrect implementation, etc.)
    2. INVOKE `coding-agent` to fix the specific issues
    3. Re-run `unit-tester-agent`
    4. If still failing after 5 attempts:
       - Mark module as `BLOCKED`
       - Document blocking issue
       - **MUST INVOKE `tracking-update-agent`** to update status
       - EXIT (do not proceed)

**L1 Repair Loop Orchestration:**
The generate-code command (YOU) orchestrates this loop - NOT the individual agents:
```
FOR attempt = 1 to 5:
    1. Analyze failure output from unit-tester-agent
    2. Identify root cause (implementation bug vs test bug)
    3. INVOKE coding-agent with specific fix instructions
    4. INVOKE unit-tester-agent to re-run tests
    IF tests pass AND coverage >= Coverage Target:
        BREAK loop, proceed to smoke test
IF all 5 attempts fail:
    Mark module BLOCKED
    INVOKE tracking-update-agent
    EXIT
```

#### 1.5 Smoke Test (BLOCKING)
- **If this module owns the schema/migrations** (typically the data-foundation module): first apply migrations and the production-safe reference seed to the dev database (idempotent; data sources per `DEPLOYMENT.md`'s DATA decisions). A migration/seed failure blocks the module.
- **MUST INVOKE `smoke-test-agent`** unless `-skip-smoke` flag is set
- Verifies the application actually runs after module implementation — **against the bootstrapped dev environment** (real dev configuration and services per `DEPLOYMENT.md`'s run mode, not mocks)
- **Apply smoke test** to modules that produce runnable code (backend servers, frontend apps)
- **Skip smoke test** for foundational modules (data models, utilities, shared libraries) with no runnable entry points
- Determine classification by reading the module spec and checking for runnable endpoints or rendered UI

- **SUCCESS PATH (Smoke Test Passes)**:
  - Application starts without crashing
  - Basic functionality responds
  - Proceed to Step 1.5b

- **FAILURE PATH (Smoke Test Fails)**:
  - Application doesn't start or crashes
  - Mark module as `BLOCKED`
  - **MUST INVOKE `tracking-update-agent`** to update status
  - EXIT (do not proceed)

#### 1.5b Handle Functional Verification Result

**Skip this step** for foundational modules (no UI/API routes) or if `-skip-smoke` flag is set.

If the smoke-test-agent returns `FUNC_FAIL` (structural checks passed but functional checks failed):
1. This is a **data gap**, NOT a code bug — do NOT mark the module as BLOCKED
2. INVOKE `coding-agent` with instructions to fix the seed/reference data (or the seeding code) in the dev environment for the failing routes (details from the smoke-test-agent's `functional_verification` section)
3. Re-INVOKE `smoke-test-agent` to verify the fix
4. Maximum 2 fix attempts. If still `FUNC_FAIL` after 2 attempts, log a warning and proceed (non-blocking but reported)

If the smoke-test-agent returns `PASS`: Proceed to Step 1.5c.
If the smoke-test-agent returns `FAIL`: Mark module BLOCKED (existing behavior).

#### 1.5c Module E2E Check (UI modules — BLOCKING)

**Skip this step** for modules with no UI screens, or if `-skip-smoke` is set.

- **MUST INVOKE `e2e-test-agent`** with `Scope: module` — a headless-browser check that this module's screens render meaningful content and the primary interaction works, against the running app in the dev environment
- **SUCCESS:** proceed to tracking update
- **FAIL:** enter the module E2E fix loop (max 3 attempts):
  1. Read the agent's failure diagnostics (failing step, console/network errors, likely layer)
  2. INVOKE `coding-agent` to fix (or repair seed data when the failure is classified as a data gap)
  3. Re-INVOKE `e2e-test-agent` (`module` scope)
  4. If still failing after 3 attempts: mark module `BLOCKED`, INVOKE `tracking-update-agent`, EXIT

#### 1.6 Tracking Update
- **MUST INVOKE `tracking-update-agent`** to update:
  - `/tracking/module-tracking.md` - Update module status, L1 coverage

**→ Continue to next module in dependency order**

---

### Step 2: Code Review (OPTIONAL - requires `-review` flag)

**Only runs if `-review` flag is provided.** Skip this step by default for speed.

- **IF `-review` flag is set:**
  - INVOKE `code-review-agent` for all implemented modules
  - **IF** Code reviewer agent identifies any major issues:
    - **Issue Resolution Loop** (Maximum 5 iterations):
      - Identify problematic modules
      - INVOKE `coding-agent` to fix the identified issues
      - INVOKE `unit-tester-agent` to re-run unit tests for affected modules
      - If the Unit-test Gate does not PASS after 5 attempts:
        - Mark affected modules as `Failed`
        - MUST INVOKE `tracking-update-agent` to update status
        - **EXIT** without executing any further actions
  - **ELSE IF** no major issues identified → Proceed to Step 3

- **IF `-review` flag is NOT set:**
  - Skip code review entirely
  - Proceed directly to Step 3 (L2 Integration Testing)

---

### Step 3: L2 Integration Testing (After Code Review Completes)

**SINGLE INVOCATION**: `l2-integration-agent` is fully autonomous and handles everything internally.

1. **L2 Integration Gate:**
   - **MUST INVOKE `l2-integration-agent`** which performs ALL of the following autonomously:
     - Phase 0: Cross-module dependency analysis
     - Phase 1: Generate integration tests based on Integration Matrix
     - Phase 2: Execute all integration tests
     - Phase 3: Internal fix loop (up to 5 attempts) - fixes code and runs L1 tests directly

   - **SUCCESS PATH (l2-integration-agent returns SUCCESS)**:
     - All L2 tests pass
     - **Mark all modules as L2 PASS**
     - **MUST INVOKE `tracking-update-agent`** to update all tracking docs
     - **Implementation is COMPLETE**

   - **BLOCKED PATH (l2-integration-agent returns BLOCKED)**:
     - L2 tests failed after 5 fix attempts
     - Mark affected modules as `BLOCKED`
     - **MUST INVOKE `tracking-update-agent`** to update status
     - EXIT without executing any further actions

   **NOTE:** You do NOT orchestrate the L2 fix loop. The l2-integration-agent is fully self-contained.

---

### Step 3.5: E2E Functional Gate (BLOCKING — After L2 SUCCESS)

The application must be SEEN working in the chosen dev environment, driven the way a user would drive it. **Headless-browser testing is MANDATORY for systems with a UI** — HTTP-level checks are not a substitute. Non-UI systems (APIs, CLIs, workers) run equivalent end-to-end journeys through their real interface.

1. **MUST INVOKE `e2e-test-agent`** with:
   ```
   Scope: full
   Dev run mode: [fully local | hybrid | cloud] (from DEPLOYMENT.md)
   App under test: [local start command | dev URL]
   Verify: user journeys from architecture/architecture.md + module acceptance criteria
   Environment notes: [seeded-data expectations, dev identity mode, bootstrap-log flags]
   ```
   The agent persists up to 10 journey tests under `tests/e2e/` and returns SUCCESS or FAIL with per-failure diagnostics (failing step, console/network errors, screenshots, likely layer).

2. **SUCCESS:** proceed (cloud mode: see step 4 below; otherwise Step 4 Finalize).

3. **FAIL:** YOU orchestrate the E2E fix loop (max `-max-attempts`, default 5):
   ```
   FOR attempt = 1 to max-attempts:
       1. Read each failure's diagnostics and likely layer
          (frontend render / API handler / data-seed gap / config-environment)
       2. INVOKE coding-agent with specific fix instructions per failure
          (data-seed gaps: fix the seed source or seeding code, not app logic)
       3. Re-run affected module L1 tests (regression guard)
       4. Re-INVOKE e2e-test-agent (full scope)
       IF SUCCESS: BREAK
   IF all attempts fail:
       Mark the modules owning the failing journeys BLOCKED
       INVOKE tracking-update-agent
       EXIT
   ```

4. **Cloud dev mode only:** after the gate passes locally, deploy the dev build to the cloud dev host (per `DEPLOYMENT.md`'s DEP/OPS decisions — dev environment only, never production), then re-INVOKE `e2e-test-agent` (full scope) against the cloud dev URL. A cloud-only failure is an environment/config issue — fix configuration (max 2 attempts), not application code that already passed locally.

---

### Step 4: Finalize — Configuration Record & Handoff (After E2E SUCCESS)

Runs when ALL modules in `tracking/module-tracking.md` are `Complete`. On a partial `-module N` run that leaves other modules incomplete, skip with a note — it runs after the final module's pipeline completes. This is the direct-path analog of `/promote-poc` Phase 5a/5b, repurposed as a **record**: the dev environment is already bootstrapped, configured, and verified — this step documents it and prepares the production-facing walkthrough for the future `/deploy-to-prod`.

#### 4.1 Generate `CONFIG_GUIDE.md` at project root

Follow the document structure and generation rules of `/promote-poc` → Phase 5 → **§5a**, with these substitutions:
- The decision source is **`DEPLOYMENT.md`** (DEP / DATA / AUTH / SEC / INT / OPS decisions) instead of `POC_PROMO_PREP.md`
- Skip every POC-config-guide comparison/reuse rule (there is no POC on this path)
- The "Deployment-Host Configuration" section is generated from the DEP-* / OPS-* decisions plus the Resource & Naming Plan and Environments & Run Modes sections of `DEPLOYMENT.md`
- Write it **as-built**: record what the bootstrap actually provisioned/verified (resource names, environments, configuration keys in use) rather than setup instructions for things that already exist. Keep step-by-step instructions only for values a human supplied (third-party keys — where to rotate/re-obtain them) and for standing up FUTURE environments (production — consumed later by `/deploy-to-prod`)

Inputs: `DEPLOYMENT.md`, `TECHSTACK.md`, `architecture/architecture.md`, `architecture/data-model.md`, and the configuration keys actually read by `src/` (scan for env-var reads or the project's config-format lookups). Sections only for services the code actually uses. Never include secret values.

#### 4.2 Generate configuration template file(s) at project root

Follow `/promote-poc` → Phase 5 → **§5b**, with the config convention detected from `TECHSTACK.md` + `src/` scan + the Configuration Key Plan in `DEPLOYMENT.md` (no POC detection). The real dev configuration file was already written by Step 0.5 — the template is the **secret-free reference for future environments**. Every key the code reads appears with a placeholder value and an inline comment naming its `CONFIG_GUIDE.md` section. If the code reads zero configuration values, skip with a note.

**Cross-check:** every key in `DEPLOYMENT.md`'s Configuration Key Plan must appear both in the template and in `src/` config reads — report any drift (a planned key the code never reads, or a code-read key missing from the plan).

#### 4.3 Report

```
GENERATE-CODE COMPLETE
======================
Modules: [N] complete ([list]) | Blocked: [N] [list if any]
L1: all pass (avg [N]% coverage) | L2: PASS
E2E: PASS ([N] journeys — headless browser: [tool] / [n/a: no UI])
Dev environment ([fully local | hybrid | cloud]): RUNNING
  Start / reach it: [dev command or URL]
  Bootstrap log: tracking/env-setup.md

Outputs:
- src/, tests/ (unit, integration, e2e), tracking/module-tracking.md
- CONFIG_GUIDE.md            ← as-built environment record + production walkthrough
- [config template file(s)]  ← secret-free reference for future environments

Next: production deployment via the future /deploy-to-prod command (not yet implemented).
```

---

## Implementation Flow (SPEED-OPTIMIZED)

```
START → Read architecture.md + DEPLOYMENT.md → Parse Integration Matrix
           ↓
    ┌────────────────────────────────────────────────┐
    │ Environment Bootstrap (BLOCKING)                │
    │  • dev run mode: fully local / hybrid / cloud   │
    │  • provision-or-verify dev services             │
    │  • write dev config (STOP if human values miss) │
    │  • connectivity checks → tracking/env-setup.md  │
    └────────────────────────────────────────────────┘
           ↓
    FOR EACH Module (in dependency order):
    ┌────────────────────────────────────────────────┐
    │  1. Read module spec                            │
    │  2. coding-agent (module + deployment context)  │
    │  3. unit-test-generator-agent (MAX 5 tests)     │
    │  4. unit-tester-agent (L1 - Coverage Target)    │
    │     └─ Fix loop (max 5) if fails                │
    │  5. schema module? → migrate + seed dev DB      │
    │     smoke-test-agent vs REAL dev env (BLOCKING) │
    │     ├─ PASS → proceed                           │
    │     ├─ FUNC_FAIL → fix seed data (max 2)        │
    │     └─ FAIL → BLOCKED                           │
    │  6. UI module? e2e-test-agent [module scope]    │
    │     headless render + interaction (fix loop ×3) │
    │  7. tracking-update-agent                       │
    └────────────────────────────────────────────────┘
           ↓
    All Modules L1 + Smoke + Module-E2E Complete?
           ↓
    ┌──────────────────────────────────────────────┐
    │ Code Review (OPTIONAL - only with -review)    │
    │  └─ Skipped by default for speed              │
    └──────────────────────────────────────────────┘
           ↓
    ┌──────────────────────────────────────────────┐
    │ L2 Integration (l2-integration-agent)         │
    │  • Cross-module integration (10 tests max)    │
    │  • Integration Matrix verification            │
    │  • Self-contained fix loop (max 5)            │
    └──────────────────────────────────────────────┘
           ↓
    ┌──────────────────────────────────────────────┐
    │ E2E Functional Gate (BLOCKING)                │
    │  • e2e-test-agent [full scope]                │
    │  • headless browser MANDATORY for UI          │
    │  • ≤10 user journeys vs running dev env       │
    │  • YOU orchestrate the fix loop (max 5)       │
    │  • cloud mode: deploy dev build + re-verify   │
    └──────────────────────────────────────────────┘
           ↓
    SUCCESS → tracking-update-agent
           ↓
    ┌──────────────────────────────────────────────┐
    │ Finalize (all modules complete)               │
    │  • CONFIG_GUIDE.md as-built record            │
    │  • config template file(s) at project root    │
    └──────────────────────────────────────────────┘
           ↓
    IMPLEMENTATION COMPLETE — dev environment RUNNING
```

---

## Output Requirements

### Per Module
- Source code implementation
- Unit tests with 60% coverage minimum (default, configurable) (max 5 tests)
- Test execution scripts
- Test reports

### After All Modules Complete
- Complete integrated source code for all modules
- Integration tests validating Integration Matrix patterns
- E2E journey suite under `tests/e2e/` (reports in `tests/reports/e2e/`)
- Test reports
- Module integration verified per Integration Matrix
- Running dev environment (bootstrapped per `DEPLOYMENT.md`, logged in `tracking/env-setup.md`)
- `CONFIG_GUIDE.md` + configuration template file(s) at project root (Step 4 — as-built record + production walkthrough for the future `/deploy-to-prod`)

## Test Gate Summary

| Gate | Scope | Coverage/Tests | Blocking | When |
|------|-------|----------------|----------|------|
| Environment Bootstrap | Dev environment | services + config + connectivity | YES | Before first module |
| L1 Unit Tests | Per module | 60% min (default), 5 tests max | YES | After each module |
| Smoke Test | App vs real dev env | App runs | YES | After L1 (runnable modules) |
| Module E2E (headless browser) | UI modules | ≤2 checks per screen | YES | After smoke |
| Code Review | All modules | N/A | NO (optional) | After all L1 + smoke |
| L2 Integration | Cross-module | 10 tests max | YES | After all modules complete |
| E2E Functional (headless browser for UI) | Whole app vs running dev env | 10 journeys max | YES | After L2 |

## Dependency Management

### Dependency Resolution Algorithm
```
1. Read Integration Matrix from architecture.md
2. Build Module Dependency Graph
3. Perform topological sort
4. Implementation Order = sorted modules (dependencies first)
```

### Example
```
Integration Matrix shows:
- M1: No dependencies (Layer 0)
- M2: Depends on M1 (Layer 1)
- M3: Depends on M1, M2 (Layer 2)
- M4: Depends on M2 (Layer 2)
- M5: Depends on M3, M4 (Layer 3)

Implementation Order: M1 → M2 → M3 → M4 → M5
```

## Manual Intervention Rules

- **Module Blocking**: If ANY module is BLOCKED, STOP all implementation
- **Dependency Violation**: Never implement a module before its dependencies
- **Bootstrap Failure**: If the dev environment cannot be bootstrapped (Step 0.5), STOP before any module
- **L1 Test Failure Threshold**: After 5 repair attempts, mark module as BLOCKED
- **Module E2E Failure Threshold**: After 3 fix attempts, mark module as BLOCKED
- **L2 Test Failure Threshold**: After 5 fix attempts, mark as BLOCKED
- **E2E Gate Failure Threshold**: After `-max-attempts` fix cycles (default 5), mark owning modules as BLOCKED

## Important Notes

- **Module-Only**: No capabilities, no iterations - just modules
- **Direct from Architecture**: Reads module specs directly
- **Orchestrator Role**: Environment bootstrap, implementation, all test gates (including the headless-browser E2E gate), and every fix loop are owned by THIS command — no separate environment-setup step follows on this path
- **Strict Gates**: Cannot skip or override test failures
- **L2 Handles Integration**: Cross-module validation at the end
- **Dev Only**: Provisions and deploys to the DEV environment only — production is the future `/deploy-to-prod`
- **Tracking**: Automatically updated via tracking-update-agent

## Success Criteria

### Per Module
- Implementation complete
- Unit tests passing with 60%+ coverage (default, configurable) (max 5 tests)
- Smoke test passes against the bootstrapped dev environment (app runs without crashing)
- Data-driven pages display real/seeded data from the dev services (not empty states)
- API endpoints return structured responses with data
- UI modules: headless-browser module check passes (screens render + primary interaction works)
- Dependencies properly integrated
- Status updated in tracking

### Overall
- All modules implemented
- All L1 unit tests passing
- L2 integration tests passing
- E2E functional gate passing — up to 10 user journeys driven against the running dev environment (headless browser, mandatory for UI systems)
- Module integrations verified per Integration Matrix
- The dev environment (fully local / hybrid / cloud per `DEPLOYMENT.md`) is bootstrapped, seeded, and RUNNING — the app is navigatable with real/seeded data on all data-driven pages
- Code reflects the `DEPLOYMENT.md` decisions (identity source + guard, data access model, canonical config keys, process topology, build shape) — no environment guessing
- `CONFIG_GUIDE.md` (as-built record + production walkthrough) + configuration template generated (Step 4)
- Complete documentation generated

## Core Requirements

- **MUST** read architecture.md and parse Integration Matrix first
- **MUST** validate `DEPLOYMENT.md` is complete (no `<!-- REQUIRED` markers) and pass its Deployment Context into every coding-agent invocation
- **MUST** bootstrap the dev environment (Step 0.5) before implementing any module — and STOP if it cannot be bootstrapped
- **MUST** implement modules in dependency order
- **MUST** implement exactly what's specified in the module development specification
- **MUST** respect module dependencies from Integration Matrix
- **MUST** ensure all test gates pass as defined above — including the E2E functional gate (headless browser mandatory for UI systems)
- **MUST NOT** create production resources or deploy to production (dev environment only — production is the future `/deploy-to-prod`)
- **MUST** invoke all specified agents in the correct sequence:

  **Per Module:**
  - `coding-agent` (full module) → `unit-test-generator-agent` → `unit-tester-agent` (L1)
  - `smoke-test-agent` (for modules with runnable code, unless `-skip-smoke`)
  - `e2e-test-agent` [module scope] (for UI modules)
  - `tracking-update-agent`

  **After ALL Modules Complete:**
  - `code-review-agent` (only if `-review` flag) → `l2-integration-agent` → `e2e-test-agent` [full scope] → `tracking-update-agent`
  - Finalize (Step 4): `CONFIG_GUIDE.md` as-built record + config template from `DEPLOYMENT.md`

- Use clean, readable code following standard best practices
- **MUST** invoke `tracking-update-agent` after each module and final completion
