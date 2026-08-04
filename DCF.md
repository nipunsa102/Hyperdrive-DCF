# DCF - Design Cascading Framework

A top-down methodology where design **cascades** from requirements through architecture to implementation. Every artifact is **derived** from its parent — no feature invention, no scope expansion.

---

## The Cascade

```
OVERVIEW.md                     (freeform requirements)
     |
     |  /generate-prd
     v
PRD.md                          (structured requirements with REQ-IDs)
     |
     |  /generate-architecture
     v
architecture/architecture.md    (high-level design with Implements: tags)
  + architecture/data-model.md  (logical data model)
     |
     |  /plan-deployment  →  [human fills in]
     v
DEPLOYMENT.md                   (deployment & environment decisions — direct path)
     |
     |  /generate-modules
     v
architecture/modules/*.md       (low-level design per module)
     |
     |  /generate-code
     v
src/ + running dev environment  (env bootstrap per DEPLOYMENT.md's dev run mode,
                                 L1/L2 gates, blocking headless-browser E2E gate,
                                 `tracking/env-setup.md`)
     |
     |  /deploy-to-prod [future]
     v
Deployed System (production)
```

---

## POC Loop (optional, before implementation)

The POC loop validates requirements with stakeholders before committing to production code.

```
                  PRD.md + architecture.md
                           |
                           |  /generate-poc
                           v
                     poc/  (navigatable UI, all mocks)
                           |
              .------------+------------.
              |                         |
              |  /modify-poc            | stakeholder
              |  -change / -new / -fix  | feedback
              |                         |
              '------------+------------'
                           |  (repeat until stable)
                           v
             poc/poc-tracking/CHANGELOG.md
                           |
                           |  /sync-prd
                           v
                    PRD.md  (updated, old PRD deprecated)
                           |
                           |  /prepare-poc-promo  →  [human fills in]
                           v
              poc/temp/poc_promotion/POC_PROMO_PREP.md  (gap decisions)
                           |
                           |  /promote-poc
                           v
              architecture/ + src/  (merged, implemented, tested)
                           |
                           |  [human fills .env from CONFIG_GUIDE.md]
                           |  /setup-env
                           v
              configured runtime  (DB schema, reference data, verified connectivity)
```

---

## Deployment Planning (Direct Path)

The POC path resolves environment decisions through `/prepare-poc-promo` → `POC_PROMO_PREP.md` → `/promote-poc` (whose architecture merge may bake hosting detail into the merged design). The direct path has no POC to analyze — so `/plan-deployment` fills the same role from the design documents alone, **before** modules are extracted.

```
/generate-architecture
        |
        |  /plan-deployment           (direct path only; hard-fails if poc/src/ exists)
        v
DEPLOYMENT.md   ← template with <!-- REQUIRED: ... --> placeholders
        |          categories: DEP (target & compute) / DATA / AUTH / SEC / INT / OPS
        |          pre-filled where TECHSTACK.md / OVERVIEW.md already decide (source cited)
        |          optional read-only platform verification (naming, regions, availability)
        |
        |  [human fills decisions — re-run to enrich resource plan once provider is chosen]
        v
/generate-modules → /generate-code       (both REFUSE to run with unfilled decisions)
        |
        v
running dev environment                  (bootstrapped per the dev run mode — fully local /
  + CONFIG_GUIDE.md as-built record        cloud / hybrid — and verified by the blocking
  + config template                        headless-browser E2E gate)
        |
        v
/deploy-to-prod [future]                 (production deployment)
```

**Division of responsibility** — why a separate document:

| Document | Owns | Stays |
|---|---|---|
| `TECHSTACK.md` | *what technologies* (languages, frameworks, libraries) | portable |
| `architecture/architecture.md` | *what the system is* | platform-agnostic (direct path) |
| `DEPLOYMENT.md` | *where & how it runs* (provider, compute model, environments, identity wiring, data access, secrets/config, integration tenancy, delivery) | the single home for platform specifics |

Module specs and generated code consume `DEPLOYMENT.md` decisions wherever they change design (identity source + dev-mode guard, data-access credential model, config module + canonical keys, process topology, build shape). DEV resources are verified-or-created by `/generate-code`'s environment bootstrap (dev-scoped, idempotent, via an authenticated provider CLI); production provisioning and deployment wait for the future `/deploy-to-prod`.

---

## Command Map

| Step | Command | Input | Output |
|------|---------|-------|--------|
| 0 | manual | -- | `OVERVIEW.md` |
| 1 | `/generate-prd` | `OVERVIEW.md` | `PRD.md` |
| 2 | `/generate-architecture` | `PRD.md` | `architecture/architecture.md` + `architecture/data-model.md` |
| 3 | `/plan-deployment` | architecture + data model + `TECHSTACK.md` | `DEPLOYMENT.md` (decision template; human completes — direct path only) |
| 4 | `/generate-modules` | `architecture.md` + `PRD.md` + `DEPLOYMENT.md` | `architecture/modules/*.md` |
| 5 | `/generate-code` | module specs + `DEPLOYMENT.md` | `src/` + tests + **running dev environment** (bootstrapped, E2E-verified) + `CONFIG_GUIDE.md` + config template |
| 6 | `/deploy-to-prod` *(future)* | `DEPLOYMENT.md` + verified modules | production deployment *(not yet implemented)* |
| -- | `/generate-poc` | `PRD.md` + architecture | `poc/` |
| -- | `/modify-poc` | stakeholder feedback | updated `poc/` + changelog |
| -- | `/sync-prd` | `poc/poc-tracking/CHANGELOG.md` | updated `PRD.md` |
| -- | `/prepare-poc-promo` | POC + main architecture + data model | `poc/temp/poc_promotion/POC_PROMO_PREP.md` (gap template for human decisions) |
| -- | `/promote-poc` | POC + main architecture + completed prep | merged architecture + modules + production code + `CONFIG_GUIDE.md` + `POC_PROMOTION_REPORT.md` ([algorithm](POC_PROMOTION_ALGORITHM.md)) |
| -- | `/setup-env` *(POC path)* | `.env` + `CONFIG_GUIDE.md` + architecture | DB schema + reference data + verified connectivity + `tracking/env-setup.md` |
| -- | `/modify` | `-change`/`-new`/`-fix` request + post-promotion `src/` | updated `PRD.md` + `architecture/` + `src/` + new `tracking/change-tracking.md` entry (post-promotion change loop) |
| -- | `/update-tracking` | module status | `tracking/module-tracking.md` |

---

## Agent Map

Agents are invoked by commands — never called directly by the user.

```
/generate-prd
  '-- traceability-validator-agent  REQ-ID coverage validation

/generate-architecture
  '-- traceability-validator-agent  REQ-ID coverage validation

/plan-deployment
  '-- deployment-gap-analyzer-agent  unanswered environment-decision scan

/generate-modules
  '-- traceability-validator-agent  REQ-ID coverage + sum test validation

/generate-code
  |-- coding-agent              writes implementation code
  |-- unit-test-generator-agent generates L1 unit tests
  |-- unit-tester-agent         runs and fixes L1 tests
  |-- smoke-test-agent          build/start validation vs the bootstrapped dev env
  |-- e2e-test-agent            headless-browser E2E verification (module + full scope)
  |-- l2-integration-agent      cross-module integration tests
  |-- code-review-agent         optional quality review
  '-- tracking-update-agent     updates module-tracking.md

/generate-poc
  |-- coding-agent              implements POC modules (POC mode)
  '-- smoke-test-agent          verifies POC starts and renders

/modify-poc
  |-- coding-agent              modifies POC code (POC mode)
  '-- smoke-test-agent          verifies POC still works

/promote-poc
  |-- re-architect-agent             merges POC + main architecture, bootstraps modules
  |-- traceability-validator-agent   REQ-ID coverage, valid refs
  |-- coherence-checker-agent        semantic consistency analysis
  |-- code-promotion-analyzer-agent  AS_IS/ADAPT/REWRITE decisions
  |-- coding-agent                   implements production code per module
  |-- unit-test-generator-agent      generates L1 unit tests per module
  |-- unit-tester-agent              runs and fixes L1 tests per module
  |-- smoke-test-agent               quick build/start validation per module
  |-- l2-integration-agent           cross-module integration tests
  '-- tracking-update-agent          updates module-tracking.md

/prepare-poc-promo
  '-- poc-gap-analyzer-agent         scans POC vs architecture for gaps

/setup-env (POC path)
  '-- smoke-test-agent               runtime smoke test against real services

/modify
  |-- coding-agent              modifies production code per affected module
  |-- unit-test-generator-agent generates/updates L1 tests
  |-- unit-tester-agent         L1 gate (60% coverage)
  |-- smoke-test-agent          verifies app still starts and routes respond
  |-- l2-integration-agent      cross-module integration (when change spans modules)
  '-- tracking-update-agent     updates module-tracking.md + change-tracking.md
```

---

## Artifact Map

```
project-root/
  OVERVIEW.md .................. freeform requirements (manual)
  PRD.md ....................... structured requirements with REQ-IDs
  DEPLOYMENT.md ................ deployment & environment decisions (template from /plan-deployment, human-completed — direct path)
  CONFIG_GUIDE.md .............. production configuration walkthrough (from /promote-poc or /generate-code)
  POC_PROMOTION_REPORT.md ..... promotion metadata + summary (from /promote-poc)
  TECHSTACK.md ................. technology choices
  DESIGNGUIDE.md ............... UI/UX and design constraints
  architecture/
    architecture.md ............ high-level design (components, flows, integrations)
    data-model.md .............. logical data model (entities, relationships, constraints)
    modules/
      module-{N}-{name}.md ..... low-level design per module
  tracking/
    module-tracking.md ......... module status (not_started → complete → deployed)
    env-setup.md ............... runtime-env setup log (from /setup-env on the POC path, or /generate-code's env bootstrap)
  poc/
    architecture/
      architecture.md .......... POC-scoped architecture
      modules/ ................. POC-scoped module designs
    temp/
      poc_promotion/
        POC_PROMO_PREP.md ...... human questionnaire (by /prepare-poc-promo)
        promotion.md ........... running promotion log (by /promote-poc)
    src/ ....................... POC source code (mocks, UI-only)
    poc-tracking/
      CHANGELOG.md ............. structured change log (consumed by /sync-prd)
      change-tracking.md ....... PM-facing change log (human reference)
      changelog-Merged-<dt>.md . archived changelogs after /sync-prd
  src/ ......................... production source code
  tests/ ....................... test suites (unit, integration, e2e)
```

---

## Traceability Chain

Every production line traces back to a requirement.

```
REQ-ID (PRD.md)
  --> Implements: tag (architecture.md component)
    --> Requirement Coverage table (module spec)
      --> Implementation code (src/)
        --> Test assertions (tests/)
```

**The Sum Test** — modules must exactly cover the architecture:
```
Module 1 + Module 2 + ... + Module N = architecture.md
```
No orphans. No duplicates. No invented features.

---

## Code Promotion Strategy

During Phase 3, `/promote-poc` analyzes POC source code against the merged architecture and assigns each module a decision (computed internally — no intermediate file produced):

```
 AS_IS ──── POC code already satisfies the spec; copy verbatim + apply a
            short list of surgical edits. No coding-agent invocation.
            (poc_coverage >= 0.80, no/clean mocks, framework identical)
 ADAPT ──── POC structure is sound; coding-agent copies POC as starting
            point, replaces mock layer with real impl, fills production gaps.
            (poc_coverage >= 0.30, clean mocks, framework compatible)
 REWRITE ── POC is reference only; coding-agent writes fresh from spec.
            (poc_coverage < 0.30, embedded mocks, or framework mismatch)
```

Coverage percentages are counted, not estimated — `SATISFIED_criteria / total_criteria` from each module's acceptance criteria list. A **POC maturity classification** (`mock-heavy` / `hybrid` / `production-wired`) is passed to the analyzer as a prior: a `production-wired` POC is expected to skew toward `AS_IS`; a `mock-heavy` POC is expected to skew toward `ADAPT`/`REWRITE`.

Under the gated workflow, `/generate-poc` only runs when `architecture/modules/` is empty. This means at promotion time there is never pre-existing production code to reconcile — so the decision tiers `WRITE NEW` (no POC code) and `KEEP EXISTING` (preserve existing src/) do not apply and are not part of the model.

## Retrofit Mode

`/promote-poc` dispatches per module based on the assigned decision:

| Decision | Behavior |
|---|---|
| `AS_IS` | `/promote-poc` copies POC files verbatim to production paths and applies the surgical edit list from the analyzer output. `coding-agent` is NOT invoked. On L1/smoke failure, module is marked `Blocked` (decision was wrong — do not silently upgrade to ADAPT). |
| `ADAPT` | `coding-agent` copies POC files as starting point, replaces mocks, fills production gaps, follows migration steps |
| `REWRITE` | `coding-agent` writes fresh from module spec, references POC for visual/behavioral patterns only |

After all per-module implementation completes, `/promote-poc` runs a **cross-cutting decision sweep**: applies `POC_PROMO_PREP.md` decisions (INT-2 generic error messages, AUTH-2 string references, etc.) systematically across every affected file in `src/`. This ensures decisions apply uniformly rather than per-module.

```
/promote-poc (single command)
        |
  classifies POC maturity (mock-heavy / hybrid / production-wired)
  merges architecture + bootstraps modules from POC
  analyzes POC per-module → AS_IS / ADAPT / REWRITE
  executes per-module (copy+edits / coding-agent ADAPT / coding-agent REWRITE)
  runs all test gates (L1 per module, smoke, L2)
  applies cross-cutting decisions uniformly across src/
  produces POC_PROMOTION_REPORT.md
```

`/generate-code` (normal mode, direct path) bootstraps the dev environment per `DEPLOYMENT.md`'s dev run mode (fully local / cloud / hybrid), implements from module specs + `DEPLOYMENT.md`, gates on headless-browser E2E verification, and finalizes `CONFIG_GUIDE.md` as the as-built record. For POC-to-production promotion, use `/promote-poc`.

---

## Post-Promotion Changes

After `/promote-poc` (or `/generate-code`) has produced live `src/`, ongoing stakeholder requests are handled by `/modify` — run on a feature git branch so the branch history is the technical changelog.

```
/modify -change|-new|-fix "<description>"
        |
  Phase 1  read PRD.md + architecture + module specs + existing tracking
  Phase 2  identify affected REQ-IDs / components / modules / src files
  Phase 3  edit PRD.md + architecture/ + module specs (in place, annotated)
           per affected module: coding-agent → L1 → smoke
           cross-module change → l2-integration-agent
  Phase 4  append CT-XXX entry to tracking/change-tracking.md
```

| Switch | Use for |
|---|---|
| `-change` | Modify existing production functionality. Updates PRD acceptance criteria; may update architecture/module specs if component boundaries shift. |
| `-new` | Add a feature absent from the current PRD. Always updates PRD + architecture + module specs + code. |
| `-fix` | Repair broken production behavior. Updates code/tests; PRD acceptance criteria refined only when the original wording was ambiguous. |

**Constraints:**
- Requires `/promote-poc` or `/generate-code` to have run (post-promotion only — refuses pre-promotion projects).
- Never reads or writes `poc/` (POC is historical at this point).
- Edits `PRD.md` in place — git branch is the backup; no deprecated PRD file is written.
- No root-level `CHANGELOG.md` — the git branch is the technical changelog; `tracking/change-tracking.md` is the PM-facing log.
- All test gates apply (L1 60%, smoke, L2 when cross-module). Failures STOP the command.
- PRD edits are annotated with `(Updated via /modify — CT-XXX)`, `(Added via /modify — CT-XXX)`, or `(Refined via /modify — CT-XXX)`.

---

## Test Gates

Every module passes through mandatory gates during `/generate-code`:

```
Module implementation
     |
     v
L1: Unit Tests (60% coverage gate)
     |  FAIL --> fix loop (max attempts)
     v
Smoke Test (build + start vs the bootstrapped dev environment)
     |  FAIL --> fix loop
     v
Module E2E (headless browser — UI modules, direct path)
     |  FAIL --> fix loop
     v
L2: Integration Tests (cross-module, after all modules)
     |  FAIL --> fix loop
     v
E2E Functional Gate (whole app vs the running dev environment;
headless browser MANDATORY for UI systems — direct path)
     |  FAIL --> fix loop (orchestrated by /generate-code)
     v
Done — dev environment running
```

- L1, L2, and the E2E functional gate are **blocking** — the pipeline cannot complete without passing
- Headless-browser testing is **mandatory for UI systems** on the direct path; non-UI systems run API/CLI journeys instead
- Additional human-authored e2e tests beyond the gate remain non-blocking

---

## Authority Chain

Never violated — reads flow down, modifications flow up through commands only.

```
PRD.md (source of truth, read-only except by /sync-prd)
  |
  v
architecture.md (derived from PRD, modified by /generate-architecture and /promote-poc)
  |
  v
module specs (derived from architecture, modified by /generate-modules and /promote-poc)
  |
  v
src/ (derived from module specs, generated by /generate-code or /promote-poc)
```
