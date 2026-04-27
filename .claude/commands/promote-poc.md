---
description: Full POC promotion — merges architecture, implements production code, runs test gates, and produces a configuration guide
model: claude-opus-4-6
---

## Purpose

One-stop command that promotes a validated POC to production. Merges POC architecture into main, bootstraps modules, analyzes POC code maturity, implements production code (verbatim copy / adapted / rewritten per module), runs all test gates, applies cross-cutting decisions systematically, and produces a configuration guide.

**This command runs on a separate git branch.** The main branch is the backup — no separate backup/snapshot phase needed.

**What this command does:**
- Validates that all human decisions in `poc/temp/poc_promotion/POC_PROMO_PREP.md` are complete
- **Classifies POC maturity** (`mock-heavy` / `hybrid` / `production-wired`) as a prior for expected effort
- Merges POC architecture and module designs into main architecture (invoke `re-architect-agent`)
- Validates the merge (traceability, cycle detection, sum test, coherence)
- Analyzes POC source code and determines per-module promotion strategy (invoke `code-promotion-analyzer-agent`) — decisions are `AS_IS`, `ADAPT`, or `REWRITE`
- Creates `tracking/module-tracking.md` from the merged Module Registry
- Implements all modules per their assigned decision:
  - `AS_IS` → direct file copy + apply listed surgical edits (no `coding-agent`)
  - `ADAPT` → `coding-agent` in ADAPT mode (preserve POC structure, swap mocks, fill gaps)
  - `REWRITE` → `coding-agent` in REWRITE mode (POC as reference only, fresh from spec)
- Runs all test gates: L1 unit tests, smoke tests, L2 integration tests
- **Applies cross-cutting decisions from `POC_PROMO_PREP.md` uniformly** across all src/ files (INT-2 error messages, AUTH-2 string references, etc.)
- Produces `POC_PROMOTION_REPORT.md` with config guide and promotion summary
- Maintains `poc/temp/poc_promotion/promotion.md` as a running log for resume capability

**What this command does NOT do:**
- Modify POC files (except `poc/temp/poc_promotion/` for the running log)
- Modify `PRD.md` (reads it as source of truth)
- Deploy to any environment

## Prerequisites

1. `PRD.md` must exist at project root (post-`/sync-prd`)
2. `poc/temp/poc_promotion/POC_PROMO_PREP.md` must exist — if not, ERROR: "Run `/prepare-poc-promo` first."
3. `poc/architecture/architecture.md` must exist
4. `poc/architecture/modules/` must exist with at least one module file
5. `architecture/architecture.md` must exist (run `/generate-architecture` first if missing)
6. `architecture/data-model.md` must exist (run `/generate-architecture` first if missing)
7. `architecture/modules/` must be empty. If it contains any `.md` files, ERROR: `"architecture/modules/ is non-empty (N module files). The /generate-poc gate should have prevented this state. Either you bypassed the gate manually, or modules were added after /generate-poc ran. Remove architecture/modules/*.md and rerun, or abandon the POC path."` This is a safety net — the `/generate-poc` gate is the primary defense.

If any prerequisite fails, ERROR with a clear message and stop.

## Process

### Phase 1: Preflight

1. **Verify prerequisites** above, including the `architecture/modules/` empty check.

2. **Validate POC_PROMO_PREP.md completeness:**
   - Read `poc/temp/poc_promotion/POC_PROMO_PREP.md`
   - Scan the body for any remaining `<!-- REQUIRED` markers
   - If ANY unfilled placeholders remain:
     ```
     ERROR: POC_PROMO_PREP.md has unfilled decisions:
     - [Gap ID]: [question text]
     - [Gap ID]: [question text]
     Fill in all <!-- REQUIRED: ... --> placeholders and re-run /promote-poc.
     ```
     STOP — do not proceed.

3. **Check POC changelog sync status:**
   - If `poc/poc-tracking/CHANGELOG.md` exists with CL-entries, verify all are referenced in PRD.md
   - If unsynced entries found, ERROR: "Run `/sync-prd` first."

4. **Detect POC Scope** — classify as `frontend-only`, `backend-only`, `full-stack`, or `undetermined` by scanning POC file extensions, frameworks, and mock directories.

5. **Classify POC Maturity (NEW):**

   The goal is to classify the POC as `mock-heavy`, `hybrid`, or `production-wired` based on evidence. Use whichever of the following signal sources apply to this project's stack — the signals listed are **examples, non-exhaustive**. Adapt the search patterns to the project's actual language, package manager, and config conventions.

   **Signal sources (all examples — use whatever applies to this project):**
   - **Source code under `poc/src/`** — look for mock patterns. Example patterns (adapt to language):
     - Hardcoded data arrays used as fake data (any language)
     - Fake-async primitives (e.g., `setTimeout(...)` in JS/TS, `time.sleep(...)` with canned returns in Python)
     - Imports from a mock/fixture directory
     - In-memory repository / store scaffolds (e.g., `InMemory*Repository`, `MockService`, `FakeClient` — name varies)
   - **POC dependency manifest** — whatever manifest the project uses (e.g., `poc/package.json` for Node, `poc/pyproject.toml` or `poc/requirements.txt` for Python, `poc/Cargo.toml` for Rust, `poc/go.mod` for Go, `poc/Gemfile` for Ruby, etc.). Look for production-grade dependencies in whatever domain the project needs:
     - Real database drivers / ORMs (e.g., `Prisma`, `TypeORM`, `pg`, `mongodb`, `SQLAlchemy`, `Django ORM`, `sqlx`, `gorm`, `ActiveRecord` — the specific package depends on the stack)
     - Real identity/auth SDKs (e.g., `@auth0/*`, `@clerk/*`, `next-auth`, `python-social-auth`, `devise`, `goth` — the specific package depends on the stack)
     - Real external-service SDKs for whatever integrations the architecture declares
   - **POC configuration / credential files** — whatever format the project uses (e.g., `poc/.env*`, `poc/config.*`, `poc/secrets.*`, `poc/settings.*`, `poc/config/*.{yaml,toml,json}` — whichever is actually present). Look for real (non-placeholder) credential patterns: DB connection strings with real hosts, API keys that don't match obvious placeholders like `YOUR_...` or `<replace-me>`.

   **Classification (choose exactly one):**
   - **`mock-heavy`** — mock patterns dominate; few or no real-service integrations
   - **`hybrid`** — mix of real and mock (e.g., real DB but mock external services, or vice versa)
   - **`production-wired`** — real DB, real auth, real external services; mocks minimal or absent

   Report the classification to the user with a 1-sentence justification citing the actual signals found (not the example signals above):
   `"POC classified as [classification] — [1-sentence evidence citing what was actually found]. Expect the analyzer to emit mostly [expected decision mix]."`

6. **Resume detection:**
   - Check if `poc/temp/poc_promotion/promotion.md` exists from a previous run
   - If yes: read it to determine which phases are already complete
   - **Resume-integrity check (NEW):** before trusting the phase checklist, verify that the on-disk outputs of completed phases still exist. Run the following cross-checks:
     - If Phase 2 is `[x]` but `architecture/modules/` is empty → promotion log is stale
     - If Phase 3 is `[x]` but `tracking/module-tracking.md` is missing → promotion log is stale
     - If Phase 4 is `[x]` but `src/` is empty or missing → promotion log is stale
     - If Phase 5 is `[x]` but `CONFIG_GUIDE.md` is missing → promotion log is stale
   - If any cross-check fails, treat the log as stale and ERROR:
     ```
     ERROR: promotion.md claims Phase N completed, but its expected outputs are missing
     ([list which outputs]). This typically happens when the promoted code was
     deleted after a previous run.

     To restart the promotion cleanly:
       rm poc/temp/poc_promotion/promotion.md

     Then re-run /promote-poc. The POC and all human decisions in POC_PROMO_PREP.md
     are preserved.
     ```
     STOP — do not silently proceed with a corrupt resume state.
   - If all cross-checks pass: skip ahead to the first incomplete phase
   - Report: `"Resuming from Phase N — previous run completed through Phase M."`

7. **Create or resume `poc/temp/poc_promotion/promotion.md`:**

   **Initial creation:**
   ```markdown
   # POC Promotion Log

   Started: [timestamp]
   Branch: [current git branch]
   POC Scope: [frontend-only/backend-only/full-stack/undetermined]
   POC Maturity: [mock-heavy/hybrid/production-wired]

   ## Promotion Plan

   - [ ] Phase 1: Preflight
   - [ ] Phase 2: Architecture Merge & Validate
   - [ ] Phase 3: Code Promotion Plan
   - [ ] Phase 4: Implement & Test
   - [ ] Phase 5: Report & Finalize

   ## Log
   ```

   **After each phase completes:** Mark it `[x]` and append a concise log entry with timestamp, what happened, and key highlights.

### Phase 2: Architecture Merge & Validate

**No backup needed** — this command runs on a separate git branch. The main branch is the backup.

**2a. INVOKE `re-architect-agent`** with context:

```
PROMOTE-POC — ARCHITECTURE MERGE:

POC Scope: [frontend-only | backend-only | full-stack | undetermined]
POC Maturity: [mock-heavy | hybrid | production-wired]

Source of Truth: PRD.md (post-sync-prd)
POC Reference: poc/architecture/ (read-only)
Merge Target: architecture/ (write here)
Promotion Prep: poc/temp/poc_promotion/POC_PROMO_PREP.md (human decisions for data, auth, integrations, deployment)
Data Model: architecture/data-model.md (merge POC data patterns into this)

Scope-specific merge strategy:
[If frontend-only]: POC is UI-focused — preserve main architecture's backend/DB/API depth
[If backend-only]: POC is API/backend-focused — preserve main architecture's frontend/UI depth
[If full-stack]: Reconcile both layers, preserving depth from both sources
[If undetermined]: Preserve all main architecture depth, only add clear POC contributions

Note: This is a bootstrap merge. architecture/modules/ is empty by contract.
```

**Wait for completion.** If FAILED, log to promotion.md and STOP.

**2b. POC Incorporation Check** — lightweight verification that POC changes were incorporated into the merge (screen/feature coverage, acceptance criteria comparison, PRD change annotations, POC changelog cross-reference). If issues found: apply fixes directly, log fixes to promotion.md.

**2c. Structural and Coherence Validation** (combined loop, max 5 iterations):

Run all of the following in one pass; repeat until all pass or convergence stalls (no issues fixed for 2 consecutive iterations):

- **Traceability Validation** — INVOKE `traceability-validator-agent`. Auto-fix ORPHAN and DUPLICATE errors. INVENTED and SPLIT require manual review.
- **Integration Matrix Cycle Detection** (BLOCKING) — DFS on directed graph. No cycles permitted.
- **Sum Test Validation** (BLOCKING) — modules exactly match architecture.
- **Deep Coherence Analysis** — INVOKE `coherence-checker-agent`:
  ```
  PROMOTE-POC — DEEP COHERENCE ANALYSIS:

  Architecture merge and structural validation complete. Perform deep semantic
  coherence analysis on the merged design documents.

  Source of Truth: PRD.md (read-only)
  Design Documents: architecture/ (read and fix)
  Data Model: architecture/data-model.md (validate entity consistency across modules)
  EXCLUDED: poc/ (do not read or reference)
  ```

**Interpret combined results:**
- **PASS**: proceed to Phase 3.
- **PARTIAL_PASS**: proceed with warnings logged to promotion.md.
- **FAIL**: offer the user a choice — PROCEED (log warning) or STOP (user can switch back to main branch and fix issues).

Update promotion.md with validation results, iterations used, remaining warnings.

### Phase 3: Code Promotion Plan

**INVOKE `code-promotion-analyzer-agent`** with context:

```
PROMOTE-POC — CODE PROMOTION ANALYSIS:

POC Scope: [frontend-only | backend-only | full-stack | undetermined]
POC Maturity: [mock-heavy | hybrid | production-wired]  ← prior for expected decision distribution

Design Reference: architecture/ (authoritative)
POC Code: poc/src/ and poc/ root configs (read-only analysis)
Promotion Prep: poc/temp/poc_promotion/POC_PROMO_PREP.md (human decisions)
EXCLUDED: architecture/Deprecated-*/

IMPORTANT: Return your full analysis as a structured report with per-module
decisions (AS_IS, ADAPT, or REWRITE). For AS_IS modules, include the surgical
edit list. For ADAPT/REWRITE modules, include the migration steps and gaps.
Do NOT write a plan file — the invoking command will use your analysis directly.
```

**Wait for completion.** The agent returns:
- Per-module decisions (`AS_IS` / `ADAPT` / `REWRITE`)
- For AS_IS: POC files to copy verbatim + surgical edits list per module
- For ADAPT: POC file mappings, gaps, migration steps, patterns to preserve
- For REWRITE: POC files (reference only), patterns to preserve, migration steps
- Mock strategy classification, coverage percentages, dev-mode data requirements

**Store decisions internally** for use in Phase 4. Do NOT write to a file.

**Sanity-check distribution against POC maturity** (from Phase 1 classification):
- `production-wired` → expect mostly `AS_IS`, some `ADAPT`; any `REWRITE` with specific justification
- `hybrid` → expect mix of `AS_IS` and `ADAPT`
- `mock-heavy` → expect mostly `ADAPT` and `REWRITE`

If the distribution conflicts with the prior and the analyzer did not justify the mismatch, log a warning to promotion.md and prompt the user to confirm before proceeding.

**Log decision summary to user:**
```
POC Maturity: [classification]
Code Promotion Decisions:
- AS_IS:   [count] modules — [list]  (verbatim copy + surgical edits)
- ADAPT:   [count] modules — [list]  (coding-agent preserves POC, swaps mocks)
- REWRITE: [count] modules — [list]  (coding-agent writes fresh from spec)
```

**Initialize `tracking/module-tracking.md`** from the merged Module Registry in `architecture/architecture.md`:

```markdown
# Module Tracking

## Summary
- Total Modules: [N]
- Complete: 0
- In Progress: 0
- Blocked: 0
- Not Started: [N]

## Module Status

| Module | Name | Status | L1 Coverage | L2 Status | Dependencies | Notes |
|--------|------|--------|-------------|-----------|--------------|-------|
| M1 | [name] | Not Started | - | - | [deps] | [decision: AS_IS/ADAPT/REWRITE] |
| M2 | [name] | Not Started | - | - | [deps] | [decision] |
...

## Blockers

(none)

## History

### [date]
- Tracking initialized by /promote-poc ([maturity] POC, [N] modules: [X] AS_IS, [Y] ADAPT, [Z] REWRITE)
```

**On resume:** If tracking document already exists, read current state. Skip modules already marked complete.

Update promotion.md: maturity, distribution, module count, tracking initialized.

### Phase 4: Implement & Test

Read Integration Matrix from merged `architecture/architecture.md`. Build module dependency order via topological sort.

**On resume:** Check `tracking/module-tracking.md` — skip modules already marked complete.

**FOR EACH module in dependency order:**

Update promotion.md: `Starting M{N}: {name} — {DECISION}`

#### If decision is AS_IS:

1. **Read the POC files** listed in the analyzer output for this module.
2. **Copy each POC file** to its target production path (as specified by the analyzer's source → target mapping). Use the Write tool to create the production file with the POC file's contents. Do NOT reformat.
3. **Apply surgical edits** from the analyzer output, one by one. Each edit is a narrow before→after change on a specific file. Use the Edit tool with `replace_all: false` (or `true` where the analyzer explicitly marks an edit as file-wide).
4. **INVOKE `unit-test-generator-agent`** — max 5 tests, 60% coverage target. POC has no tests, so tests are always generated for AS_IS modules.
5. **INVOKE `unit-tester-agent`** — L1 gate.
   - **PASS (60%+ coverage):** proceed to smoke test.
   - **FAIL:** This is a signal that the AS_IS decision was wrong for this module (the copied code does not satisfy the spec's tested behavior). **Do NOT invoke `coding-agent` to fix** — that would contradict the AS_IS decision. Mark module `Blocked` with note `"AS_IS failed L1 gate — POC code does not satisfy spec. Re-run /promote-poc (analyzer will reclassify) or fix manually."`. INVOKE `tracking-update-agent`. EXIT this module; continue to the next.
6. **INVOKE `smoke-test-agent`** (blocking, unless foundational module with no runnable entry point).
   - **PASS:** proceed.
   - **FUNC_FAIL:** INVOKE `coding-agent` to fix dev-mock data (max 2 attempts), re-run smoke. Note: dev-mock data fixes are acceptable under AS_IS because they don't modify the module's production code.
   - **FAIL:** mark `Blocked` (as in step 5), INVOKE `tracking-update-agent`, EXIT this module.
7. **INVOKE `tracking-update-agent`** — update module status, L1 coverage.

#### If decision is ADAPT or REWRITE:

1. **Read module specification** from `architecture/modules/module-{N}-{name}.md`.
2. **INVOKE `coding-agent`** with full context:
   ```
   Module ID: M{N}
   Module Name: {name}
   Development Spec: architecture/modules/module-{N}-{name}.md
   Dependencies: [from Integration Matrix]
   Coverage Target: 60%
   Retrofit: {ADAPT | REWRITE}
   ```
   For ADAPT:
   - POC Files table (file paths and actions: copy-and-adapt)
   - What POC Covers (satisfied criteria)
   - Gaps (must build)
   - Migration Steps (ordered)

   For REWRITE:
   - POC Files (reference only)
   - Patterns to Preserve
   - Gaps (full list)
   - Migration Steps (advisory)

3. **INVOKE `unit-test-generator-agent`** — max 5 tests, 60% coverage target.

4. **INVOKE `unit-tester-agent`** — L1 gate.
   - **PASS (60%+ coverage):** proceed to smoke test.
   - **FAIL:** enter L1 Repair Loop (max 5 attempts):
     1. Analyze failure
     2. INVOKE `coding-agent` to fix
     3. INVOKE `unit-tester-agent` to re-run
     4. If still failing after 5 attempts: mark module `Blocked`, INVOKE `tracking-update-agent`, EXIT.

5. **INVOKE `smoke-test-agent`** (blocking, unless foundational module with no runnable entry point).
   - **PASS:** proceed.
   - **FUNC_FAIL:** INVOKE `coding-agent` to fix dev-mock data (max 2 attempts), re-run smoke.
   - **FAIL:** mark `Blocked`, INVOKE `tracking-update-agent`, EXIT.

6. **INVOKE `tracking-update-agent`** — update module status, L1 coverage.

Update promotion.md: module result (PASS/BLOCKED), coverage, key notes.

**→ Continue to next module in dependency order**

---

**After ALL modules complete:**

#### Cross-Cutting Decision Sweep (NEW)

Apply decisions from `POC_PROMO_PREP.md` systematically across all `src/` files. This catches decisions that the analyzer may have assigned to only a subset of modules (e.g., `INT-2` generic error messages must apply to every API route, not just the ones the analyzer flagged).

1. **Parse `POC_PROMO_PREP.md`** — extract each filled-in decision and classify it:
   - **File-scoped** (applies to all files of a certain type): e.g., an `INT-*` decision that says "generic error messages across all routes" applies to every API route, not just the ones the analyzer flagged; an `AUTH-*` decision about fallback UI messaging applies to every page component referencing the auth provider's config
   - **String-scoped** (specific string patterns to remove/replace across the codebase): e.g., `.env.local` references in error messages or UI text
   - **Config-scoped** (env vars, dependency-manifest declarations — whatever form the project uses for config): e.g., an `INT-*` decision that introduces a new env var like `AI_MODEL`, or an `INT-*` decision that adds a required dependency to the project's manifest
   - **Deployment-scoped** (documented only in `POC_PROMOTION_REPORT.md`, no code change): `DEP-2`, `DEP-3`, `DEP-4`

2. **For each code-impacting decision, scan `src/` and apply fixes:**
   - Build a grep/glob query that finds all violations
   - Apply the fix via Edit tool (one decision at a time, one file at a time)
   - Log each fix: `"[decision-id] Applied to N files: [list]"`

3. **Re-run affected module L1 tests** after the sweep. If any L1 regresses:
   - Enter a targeted fix loop (max 3 attempts) using `coding-agent`
   - If still failing, mark affected module `Blocked` with note `"Cross-cutting sweep regressed L1 — manual fix required"`

Report sweep results to promotion.md and include a summary in `POC_PROMOTION_REPORT.md`.

#### L2 Integration Gate

**INVOKE `l2-integration-agent`** — fully autonomous:
- Generates integration tests from Integration Matrix
- Executes tests
- Internal fix loop (max 5 attempts)
- Returns SUCCESS or BLOCKED

**If SUCCESS:** INVOKE `tracking-update-agent` with `l2_pass` — all modules → Complete.
**If BLOCKED:** INVOKE `tracking-update-agent` with `l2_fail` — affected modules → Blocked.

Update promotion.md: L2 results.

### Phase 5: Report & Finalize

This phase produces **three separate output artifacts** at project root:

- **`CONFIG_GUIDE.md`** — the human-facing configuration guide (provider setup, config values, what goes where). This is the sibling of the POC's config guide (if any) on the production side, and the authoritative input for `/setup-env`.
- **Configuration template file(s)** — a template in whatever format the POC uses for configuration (dotenv, YAML, TOML, JSON, etc.), placed at project root so the human can copy it to the real configuration file path and fill in values.
- **`POC_PROMOTION_REPORT.md`** — promotion metadata (what happened, which decisions applied, per-module outcomes, blocked modules, known limitations). Points to `CONFIG_GUIDE.md` for configuration steps.

---

#### 5a. Generate `CONFIG_GUIDE.md` at project root

This file is the **production analog of the POC's config guide** — one section per external service the promoted code actually uses, with step-by-step provider-dashboard instructions for obtaining the credentials and wiring them into `.env`.

**Input sources** (read all of these; `CONFIG_GUIDE.md` is a synthesis, not a template fill-in):
- `poc/temp/poc_promotion/POC_PROMO_PREP.md` — human decisions (DP / AUTH / SEC / INT / DEP) including which providers, environments, regions were chosen for production
- `TECHSTACK.md` — authoritative stack choices (ORM, drivers, SDKs, external service vendors)
- `architecture/architecture.md` — external-service component list and integration matrix (what the code actually talks to)
- `architecture/data-model.md` — entity list (informs whether migration / seeding sections are needed)
- `src/` imports — which SDK / client libraries are actually wired into the promoted code
- **`poc/docs/config-guide.md` (or equivalent POC config guide) if present** — structural reference for style, depth, and section layout

**Generation is project-driven, not template-driven.** The sections, headings, env var names, provider names, and setup steps must all come from the actual project's stack as declared in the sources above. The framework does not know ahead of time whether the project uses Postgres vs MongoDB, Auth0 vs Clerk vs NextAuth, Bedrock vs OpenAI vs a local model, S3 vs R2 vs GCS, etc. — all of that comes from the project.

**Document structure** (use the POC config guide as the style reference; the skeleton below is generic and must be filled with project-specific content):

```markdown
# [Project Name from README or PRD] — Production Configuration Guide

This guide walks through configuring the production environment after `/promote-poc` has run. Fill in the project's production configuration file(s) at project root with the values listed, then run `/setup-env` to apply schema/migrations, seed reference data, and verify connectivity to every external service.

> The configuration file path, name, and format are specific to this project — see the Quick Start below.

> For the POC (pre-promotion) configuration, see the POC config guide referenced at the bottom of this file.

## Quick Start

This project uses the following configuration convention (detected during promotion):

- **Format:** [dotenv | YAML | TOML | JSON | other]
- **Production file path(s):** [e.g., `.env` at project root, or `config/production.yaml`, or whatever the project uses]
- **Template(s) generated by `/promote-poc`:** [e.g., `.env.example`, `config.example.yaml`]

After completing this guide:
1. Copy the template file(s) to the production path(s) named above (or create the production file(s) manually)
2. Fill in the values per the sections below
3. Run `/setup-env` to initialize the runtime environment (applies migrations, seeds reference data, verifies services)
4. Start the app using the project's dev command (see the project README or its manifest file — `package.json` / `pyproject.toml` / `Cargo.toml` / `go.mod` / etc. — for the exact command)

## Configuration Values at a Glance

One row per value required by the promoted code (keys derived from scanning `src/` for env-var reads, YAML/TOML key paths, or whatever the detected format uses). The "Source" column says where the human obtains each value. The "Section" column links to the detailed walkthrough.

| Key | Purpose | Source | Section |
|---|---|---|---|
| [...one row per required config key...] | [...] | [...] | [...] |

## Section 1..N: One section per external service the promoted code actually uses

For each external service declared in `architecture/architecture.md` AND whose production vendor was decided in `POC_PROMO_PREP.md`, emit a section. Section order: data stores → identity/auth → compute/AI → storage → messaging → observability → other, but follow whatever order the POC config guide used if present (consistency with POC).

Each section should contain:

- **What it does** — one-line description of what this service provides to the app
- **Step-by-step provider setup** — exact clicks in the provider's dashboard (create account / project / tenant / bucket / database, configure required options per `POC_PROMO_PREP.md` decisions, etc.)
- **Required configuration values** — the exact keys the code reads (derived from scanning `src/` for env-var reads or the format-appropriate config lookups)
- **Verification step** — how the human can tell the service is ready (often a link to provider dashboard or a CLI command)

Production-vs-POC adaptation required per section:
- Where `POC_PROMO_PREP.md` chose separate production resources (separate tenant / separate project / isolated DB / different region), write the production-specific steps, not the POC steps
- Skip POC-only guidance such as free-tier notes that do not apply at production scale
- Where a service requires provider-side configuration beyond the config file (e.g., an action, a trigger, a policy, a callback URL list) derived from the promoted code's expectations — include the exact configuration needed, copied or adapted from the code itself

## Section N+1: Deployment-Host Configuration

Generated from DEP-* decisions in `POC_PROMO_PREP.md`:
- Number of environments the human committed to (e.g., preview + production, or just production)
- Per-environment configuration scoping approach (if the hosting platform supports this)
- Domain / SSL / DNS guidance at a high level (specific provider clicks only if DEP-* explicitly chose a hosting provider)

## After Editing Configuration

Run `/setup-env` to apply migrations, seed reference data (taxonomy / enums / lookup tables only — never sample users or content), and verify connectivity to every external service.

If `/setup-env` reports a failure, the error message names the service and points back to the section of this guide to revisit.

## Troubleshooting

One entry per common failure mode per service. Mirror the structure of the POC config guide's troubleshooting section (if the POC has one) and adapt for production-specific issues (e.g., config scoping on the chosen host, production tenant domain mismatches, region-specific model availability).

## POC Config Guide Reference

If a POC config guide exists (e.g., `poc/docs/config-guide.md` or the project's equivalent), link to it here. Readers who are debugging a promoted-code issue by cross-referencing the POC behavior will find it useful.
```

**Rules for CONFIG_GUIDE.md generation:**
- Only include sections for services the promoted code actually uses. Scan `src/` imports and `architecture/architecture.md` — do not invent services the project does not use, and do not copy POC sections for services that were dropped during promotion.
- Never include actual secret values in the guide — only variable names, provider-dashboard navigation paths, and "where to copy it from" instructions.
- Use real section numbers based on actual service count — the skeleton's "Section 1..N" is a placeholder; the generated file must have concrete numbered sections.
- Commands and URLs must be copy-pasteable — verify each command references a tool the project actually uses (per `TECHSTACK.md`).
- If the promoted code requires non-env-var provider configuration (callback URLs, roles, actions, policies, triggers, bucket policies, IAM permissions, etc.), include it in the relevant section with the exact configuration derived from the code.
- No framework-level vendor assumptions. If you find yourself writing "Auth0" or "Bedrock" or any other provider name, verify it comes from the project's own `TECHSTACK.md` / `POC_PROMO_PREP.md` / architecture — not from framework defaults.
- Assume zero provider-dashboard familiarity. For every step that references a provider UI, state the exact navigation path (tenant/workspace → section → page → field) and name the UI element the user is expected to interact with. Never write instructions that require the reader to already know how to switch tenants, accounts, projects, workspaces, or environments within the provider. If an action must happen inside a specific tenant/workspace/project, state that explicitly before the step (e.g., "Switch to the `wisflicks-production` tenant via the top-left dropdown, then ..."). When a concept is easy to confuse with a similar one in the same provider (e.g., session-encryption secret vs. OAuth client secret, provider-level OAuth credential vs. per-tenant connection), disambiguate in-line at the step where the confusion would occur — not in a separate glossary.
- **Compare against, and build on, the POC config guide — don't repeat it.** If a POC config guide exists (e.g., `poc/docs/config-guide.md` or any config guide detected under `poc/`), read it in full before generating the production guide. The production guide must:
  - **Reuse the POC guide's wording and structure** where the step is identical (e.g., Deepgram signup URL, the `openssl rand -hex 32` command). Do not paraphrase for the sake of paraphrase — conformity reduces reader burden.
  - **Add only what's different for production** — separate tenants/projects/accounts, production-grade credentials (not shared dev keys), callback URLs on the prod domain, least-privilege IAM policies, production model IDs, etc. Start each section by stating *what changes vs. the POC*.
  - **Never re-suggest identical URLs or steps** as if they were new — e.g., if the POC guide already explains how to create a Deepgram account, the prod guide says "Create a production-dedicated API key in the same Deepgram account (signup already covered in the POC guide)" rather than repeating the signup flow.
  - **Maintain cross-reference conformity** — env var names, section ordering, and terminology must match the POC guide wherever the underlying concept is the same. If the POC guide calls a key `DEEPGRAM_API_KEY`, do not rename it `DG_KEY` in prod. If the POC guide uses "Section 1: Neon," do not reshuffle to "Section 1: Auth0" in prod without reason.
  - **Explicitly note divergence** when the prod setup *must* differ (e.g., "POC uses a single shared Auth0 tenant; prod uses a dedicated `<project>-production` tenant per DEP-3"). Reference the relevant `POC_PROMO_PREP.md` decision ID so the reason is auditable.
  - **Link to the POC config guide at the top** of the prod guide, not only at the bottom, so readers can context-switch quickly when debugging.

---

#### 5b. Generate configuration template file(s) at project root

Produces a template configuration file (or files) at project root that the human copies to the real config path and fills in. The format mirrors whatever the POC uses so the consumer code works identically.

**Detection:**
- Inventory every configuration-like file under `poc/` (examples of what to look for: `poc/.env*`, `poc/config.*`, `poc/config/**/*.{yaml,toml,json}`, `poc/secrets.*`, `poc/settings.*`, `poc/app.config.*`). Do not assume any particular convention — detect from what is actually present.
- For each POC config file, determine the **production counterpart path**: same filename at project root, OR the conventional production name (e.g., if POC uses `poc/.env.local` for local dev, production typically uses `.env` at root; if POC uses `poc/config.yaml`, production uses `config.yaml` at root).
- If `TECHSTACK.md` declares a different config convention than the POC is using, prefer `TECHSTACK.md` and call the conflict out in `POC_PROMOTION_REPORT.md`.

**Template generation:**
For each production config file path, write a corresponding **template file** at project root, using a `.example` or `.template` suffix before the extension (project convention permitting). Examples:
- POC has `poc/.env.local` → production expects `.env`; template is `.env.example`
- POC has `poc/config.yaml` → production expects `config.yaml`; template is `config.example.yaml` or `config.yaml.example` (match what is idiomatic for the format)
- POC has `poc/config/secrets.toml` → production expects `config/secrets.toml`; template is `config/secrets.example.toml`

**Template content:**
- Every configuration key the promoted code reads, derived by scanning `src/` (`process.env.X`, `os.environ['X']`, `ENV['X']`, YAML key paths, TOML key paths, etc., matching the detected format)
- For each key:
  - The key name
  - An inline comment (format-appropriate) naming the source section in `CONFIG_GUIDE.md`
  - A placeholder value (`YOUR_XXX_HERE`, `<replace-me>`, or format-appropriate)
- No actual secret values. Ever.

**If the promoted code reads zero configuration values** (e.g., static-site-only project, no external services, no env vars): skip Phase 5b with a note in `POC_PROMOTION_REPORT.md` that no configuration template was needed.

**If the POC has no configuration files** (mock-heavy POC with inline fakes, no config at all): determine the production config convention from `TECHSTACK.md` + `src/` scan, and generate the template from scratch based on what the promoted code reads.

Log to promotion.md: generated template path(s), total key count, any conventions the promotion surfaced to the user.

---

#### 5c. Generate `POC_PROMOTION_REPORT.md` at project root

Read:
- `poc/temp/poc_promotion/POC_PROMO_PREP.md` — for decisions-applied summary
- `tracking/module-tracking.md` — for final module status
- `poc/temp/poc_promotion/promotion.md` — for promotion history

```markdown
# POC Promotion Report

Generated by `/promote-poc` on [date]

> **Next step:** See `CONFIG_GUIDE.md` for environment configuration, then run `/setup-env` to initialize the runtime environment.

## Promotion Summary

| Metric | Value |
|--------|-------|
| POC Maturity | [mock-heavy / hybrid / production-wired] |
| Modules Promoted | [N] |
| AS_IS | [N] ([list]) |
| ADAPT | [N] ([list]) |
| REWRITE | [N] ([list]) |
| Architecture Sections Added | [N] |
| Architecture Sections Modified | [N] |
| L2 Integration Status | [PASS/BLOCKED] |

### Per-Module Results

| Module | Decision | L1 Coverage | Smoke | Status |
|--------|----------|-------------|-------|--------|
| M1 | AS_IS | 75% | PASS | Complete |
| M2 | ADAPT | 68% | PASS | Complete |
| ... | ... | ... | ... | ... |

## Cross-Cutting Decision Sweep

| Decision | Scope | Files Touched | Result |
|----------|-------|---------------|--------|
| INT-2 (generic error messages) | All API routes | 17 | Applied |
| AUTH-2 (.env.local references) | Client components | 2 | Applied |
| ... | ... | ... | ... |

## Human Decisions Applied

| ID | Decision | Applied In |
|----|----------|------------|
| DP-2 | [decision text] | [module / file] |
| AUTH-2 | [decision text] | [module / file] |
| INT-1 | [decision text] | [module / file] |
| ... | ... | ... |

[If any BLOCKED:]
## Blocked Modules

- **M{N}**: [reason] — [recommended action]

## Generated Files

Grouped by category: architecture, tracking, source, tests, configs.

## Known Limitations

Things the promotion could not automate — manual steps remaining
(e.g., provider-side configuration that can only be done in a vendor dashboard,
such as identity-provider actions/triggers, storage-bucket creation, or other
manual setup captured under DEP-scoped decisions in POC_PROMO_PREP.md).

## POC Retention After Promotion

The `poc/` directory is **preserved**, not deleted, by `/promote-poc`. After promotion succeeds, treat `poc/` as a historical reference — its architecture, code, and change log remain useful when debugging a promoted-code issue against the validated POC behavior.

Options for what to do with `poc/` after promotion:
- **Recommended: leave it in place.** It's self-contained and gitignored paths are already excluded. A frozen POC serves as a living reference for the decisions that led to production.
- **Archive it.** Move `poc/` into `poc-archive/{timestamp}/` if the repo is getting large or if multiple POC generations are expected over time.
- **Delete it.** Only recommended when the git history is the reference of record and the project explicitly wants a clean working tree. The git branch still preserves the pre-promotion state.

`/promote-poc` does NOT make this decision for you — no automatic archive or delete step. Leaving `poc/` in place is the safe default.

## Next Steps

1. Review `CONFIG_GUIDE.md` and follow each section to obtain credentials and populate `.env`
2. Run `/setup-env` to run migrations, seed reference data, and verify external service connectivity
3. Start the app using the project's dev command (e.g., `npm run dev`, `python manage.py runserver`, `go run .`, `bundle exec rails server` — whatever the project README or manifest documents) and walk through a primary user flow
4. When ready for deployment: `/deploy-module` (when implemented) or deploy manually per `CONFIG_GUIDE.md` §5
```

The report is **metadata and history** — it does not duplicate the configuration steps from `CONFIG_GUIDE.md`.

---

#### 5d. Finalize promotion log

Update `poc/temp/poc_promotion/promotion.md` — mark all phases `[x]`, append final summary.

#### 5e. Print summary to user

```
PROMOTE-POC COMPLETE
====================
POC Maturity: [classification]
Modules: [N] total ([X] AS_IS, [Y] ADAPT, [Z] REWRITE)
L1: All pass (avg [N]% coverage)
L2: [PASS/BLOCKED]
Cross-cutting sweep: [M] decisions applied across [K] files
Blocked: [N] modules [list if any]

Outputs:
- src/, tests/, architecture/modules/, tracking/module-tracking.md
- CONFIG_GUIDE.md              ← configuration walkthrough
- POC_PROMOTION_REPORT.md      ← promotion metadata
- [config template file(s)]    ← template(s) in the project's config format (detected per 5b)

Next: Read CONFIG_GUIDE.md, fill in the project's production config file(s), then run /setup-env to initialize the runtime.
```

## Rules

1. **Git branch** — This command runs on a separate branch. Main branch is the backup. No `architecture/Deprecated-<datetime>/` needed.
2. **POC is read-only** — Never modify `poc/` except `poc/temp/poc_promotion/` (running log only).
3. **PRD is source of truth** — Every REQ-ID must be covered. Never modify `PRD.md`.
4. **Direct agent invocation** — All agents are invoked directly. Commands cannot invoke other commands.
5. **Gated workflow** — `architecture/modules/` MUST be empty at promotion time. The `/generate-poc` gate is the primary defense; the preflight check in Phase 1 is the safety net.
6. **Three-tier decision model** — Every module gets exactly one of `AS_IS`, `ADAPT`, or `REWRITE`. `WRITE NEW` and `KEEP EXISTING` are not used (unreachable under the gated workflow).
7. **AS_IS skips `coding-agent`** — `/promote-poc` performs the file copy and applies surgical edits directly. If `coding-agent` is invoked for an AS_IS module (orchestration bug), defense-in-depth rules in `retrofit-mode.md` prevent file modification.
8. **AS_IS failure is not auto-repaired** — If an AS_IS module fails L1 or smoke, the decision was wrong. Mark `Blocked` and escalate; do not enter the repair loop (which would invoke `coding-agent` and contradict AS_IS).
9. **Structural completeness** — Every component needs `Implements:` tags. Every module starts with Requirement Coverage table. Module Registry, Integration Matrix (all 5 columns) must be complete.
10. **All validations must pass** — Traceability (PASS), cycle detection (no cycles), sum test (PASS), coherence (PASS or PARTIAL_PASS).
11. **All test gates apply** — L1 unit tests (60% coverage), smoke tests, L2 integration tests. Applies to all three tiers (AS_IS, ADAPT, REWRITE).
12. **Cross-cutting decisions apply systematically** — After per-module implementation, sweep `src/` to apply `POC_PROMO_PREP.md` decisions (INT-2 error messages, AUTH-2 string references, etc.) uniformly. The sweep runs before L2.
13. **Resume capability** — On re-run, check `poc/temp/poc_promotion/promotion.md` and `tracking/module-tracking.md` to skip completed work.
14. **No secrets in output** — `POC_PROMOTION_REPORT.md` lists what secrets are needed and where to get them, never actual values.
15. **Code review is intentionally not part of promotion** — unlike `/generate-code`, this command does not offer a `-review` flag. Quality assurance on promoted code relies on: (a) the POC-maturity classification (Phase 1) calibrating expectations, (b) the three-tier decision model (Phase 3) choosing the right adaptation strategy per module, (c) L1/smoke/L2 gates (Phase 4) enforcing functional correctness, and (d) the cross-cutting decision sweep (Phase 4) applying human decisions systematically. Adding a review pass on top would duplicate these checks without commensurate value. For explicit review of production code changes after promotion, use `/modify` on a feature branch — it runs the full gate cycle including L2.
