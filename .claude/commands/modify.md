---
description: Implement stakeholder-requested changes to promoted production code with full tracking and test gates
model: claude-opus-4-6
---

**Switches**: `-change`, `-new`, `-fix`

**Switch Definitions**:
- `-change` → Description of a modification to existing production functionality, in the product manager's own words
- `-new` → Description of a new feature to add that did not exist in the current PRD, in the product manager's own words
- `-fix` → Description of something broken in production that needs fixing (e.g., "stats endpoint returns 500 when user has no flicks", "search bar doesn't filter results")

**Switch Rules:** Exactly one switch must be provided. If none is provided, or more than one is provided, ERROR.

## Purpose

Implements stakeholder-requested changes to promoted production code and tracks all modifications. After a POC has been promoted (via `/promote-poc`) and the codebase is under active development, the product manager captures stakeholder feedback and uses this command to (1) update `PRD.md`, `architecture/`, and `src/` code, (2) run all test gates, and (3) maintain a single tracking log for traceability.

**Three modes:**
- **`-change`**: Modifies existing production functionality (e.g., "change the stats card order", "show vocabulary categories on flick cards")
- **`-new`**: Adds a feature that does not exist in the current PRD (e.g., "add a notifications inbox", "add a weekly digest email"). New features require PRD, architecture, and module spec updates.
- **`-fix`**: Fixes something broken in production (e.g., "stats endpoint crashes for new users", "publish button navigates to wrong page"). Bug fixes update code and tests only — PRD acceptance criteria may be refined to clarify correct behavior.

**What this command does:**
- Implements the requested change, new feature, or bug fix across `src/`, `architecture/`, and `PRD.md` as needed
- Updates `PRD.md` directly in place (git branch is the backup — no deprecation file is written)
- Updates `architecture/architecture.md` and `architecture/modules/module-{N}-{name}.md` when scope demands it
- Runs the full DCF test stack: L1 unit tests, smoke, and L2 integration (when cross-module)
- Updates `tracking/module-tracking.md` via `tracking-update-agent`
- Appends a raw log entry to `tracking/change-tracking.md` (product manager reference)

**What this command does NOT do:**
- Modify anything under `poc/` (never reads POC code — POC is historical at this point)
- Create a root-level `CHANGELOG.md` — every change runs on its own git branch, so the branch history is the change record. `tracking/change-tracking.md` is a PM-facing log, not a technical changelog
- Write a deprecated PRD file — edits are applied in place, and git provides the backup
- Deploy or publish anything

## Prerequisites

1. `/promote-poc` must have been run at least once (i.e., `src/` contains promoted code and `tracking/module-tracking.md` exists)
2. `PRD.md` must exist at project root
3. `architecture/architecture.md` and `architecture/modules/` must exist with at least one module spec

If any prerequisite fails, ERROR with a clear message and stop:

- Missing `tracking/module-tracking.md` → `ERROR: No module tracking found. Run /promote-poc first — /modify is for post-promotion code changes.`
- Missing `PRD.md` → `ERROR: PRD.md not found at project root. Run /generate-prd first.`
- Missing `architecture/architecture.md` → `ERROR: architecture/architecture.md not found. Run /generate-architecture first.`

## Tracking File

This command maintains one tracking file at `tracking/change-tracking.md`:

### `tracking/change-tracking.md` — Product Manager Log (append-only)

**Audience:** Product manager (non-technical)
**Purpose:** Simple append-only log of what was requested, in the PM's own words
**Not consumed by any command** — purely for human reference

**Format:**
```markdown
# Change Tracking

## CT-001: {YYYY-MM-DD} [CHANGE]
{Raw description from the -change switch, exactly as the PM wrote it}

## CT-002: {YYYY-MM-DD} [NEW]
{Raw description from the -new switch, exactly as the PM wrote it}

## CT-003: {YYYY-MM-DD} [FIX]
{Raw description from the -fix switch, exactly as the PM wrote it}
```

**Rules:**
- Append-only — never edit or remove previous entries
- CT numbers are sequential (CT-001, CT-002, CT-003, ...) and never reused
- Date is the date the change was implemented (today)
- Tag is `[CHANGE]`, `[NEW]`, or `[FIX]` matching the switch used
- Content is the raw switch text, unmodified

**Note on technical detail:** There is no `CHANGELOG.md` at root. The PRD is updated in place and the git branch serves as the technical changelog. `tracking/module-tracking.md` (maintained by `tracking-update-agent`) records per-module test and status outcomes.

## Process

### Phase 1: Understanding

1. **Read request** from the provided switch
   - If no switch is provided, ERROR: `A switch is required. Usage: /modify -change "description" OR /modify -new "description" OR /modify -fix "description"`
   - If more than one switch is provided, ERROR: `Use only one switch at a time. Use -change for modifications, -new for new features, or -fix for bug fixes.`
   - Record the **mode** for use in later phases: `CHANGE`, `NEW`, or `FIX`

2. **Read existing tracking files**
   - Read `tracking/change-tracking.md` (if it exists) to determine next CT number
   - Read `tracking/module-tracking.md` to understand current module status and coverage

3. **Read main architecture and module specs**
   - Read `architecture/architecture.md` for current Module Registry, Integration Matrix, and components
   - Read `architecture/data-model.md` if it exists (entity/relationship context)
   - Read module specs under `architecture/modules/` that are likely affected (scan descriptions for keywords from the request)

4. **Read `PRD.md`** — parse the full document structure:
   - Document Info (version, status, date)
   - All sections and REQ-ID tables
   - Identify candidate REQ-IDs affected by this change
   - For `-new`, also note the section naming conventions so a new REQ-ID can be chosen correctly

### Phase 2: Impact Analysis

1. **Identify affected areas:**
   - Which PRD REQ-IDs are affected (or which section gets a new REQ-ID for `-new`)?
   - Which architecture components are affected?
   - Which module specs need updating?
   - Which `src/` modules need code changes?
   - Does the change touch integrations between modules (triggering L2)?

2. **Determine scope based on mode:**
   - **`-change` mode:** Always updates PRD acceptance criteria (or user story). May also update architecture and module specs if the change affects component boundaries, integration, or data model.
   - **`-new` mode:** MUST update PRD (new user story + acceptance criteria), architecture (new or expanded component), and at least one module spec (Requirement Coverage table + acceptance criteria). If the feature warrants a new module, create `architecture/modules/module-{N}-{name}.md` and add a row to the Module Registry and Integration Matrix in `architecture/architecture.md`.
   - **`-fix` mode:** Code + tests only by default. PRD acceptance criteria MAY be refined to clarify correct behavior if the original wording was ambiguous (matching `/sync-prd` treatment of `[FIX]` entries). Architecture and module specs are NOT updated — the design was correct, the code wasn't.

3. **Build affected-module list** in dependency order (topological sort on the Integration Matrix, same as `/promote-poc` Phase 4). This determines the order in which `coding-agent` runs.

### Phase 3: Implementation

1. **Update `PRD.md`** (in place — no deprecation file)
   - **`-change` mode:** Edit the affected user story and/or acceptance criteria in their existing section. Append `(Updated via /modify — CT-XXX)` to each modified item so changes remain traceable.
   - **`-new` mode:** Add a new user story row to the appropriate section's user story table and add new acceptance criteria items. Pick the next available REQ-ID in that section (never reuse retired IDs). Annotate each addition with `(Added via /modify — CT-XXX)`.
   - **`-fix` mode:** Update acceptance criteria only if the fix clarifies ambiguously specified behavior. Annotate with `(Refined via /modify — CT-XXX)`. Otherwise skip — a pure bug fix may require no PRD change.
   - **All modes:** Increment `Document Info → Version` by the patch digit (e.g., 1.8 → 1.8.1). Update `Last Updated` to today. Leave `Status` unchanged (it was last set by `/sync-prd`).
   - Do NOT add a "What Changed" section. That section is owned by `/sync-prd`. Traceability here flows through CT-XXX annotations and git history.

2. **Update `architecture/architecture.md`** (REQUIRED for `-new`, sometimes for `-change`, SKIP for `-fix`)
   - **`-new` mode:** Add or expand the affected component. Update `**Implements:**` tags to include new REQ-IDs. Update the Module Registry if a new module is introduced. Update the Integration Matrix if new edges between modules are introduced.
   - **`-change` mode:** Update only sections whose responsibilities, boundaries, or integration contracts change. If the change does not cross component boundaries, architecture may not need updating.
   - **`-fix` mode:** SKIP.

3. **Update module specs** (REQUIRED for `-new`, sometimes for `-change`, SKIP for `-fix`)
   - **`-new` mode:** Update or create `architecture/modules/module-{N}-{name}.md`. Add new REQ-IDs to the Requirement Coverage table. Add acceptance criteria. If a new module is introduced, write the full module spec from scratch.
   - **`-change` mode:** Update affected specs if the change impacts responsibilities, acceptance criteria, data inputs/outputs, or dependencies.
   - **`-fix` mode:** SKIP — the spec already describes the correct behavior.

4. **Investigate and locate the issue** (`-fix` mode only)
   - The `-fix` switch accepts a plain natural language description (e.g., "stats endpoint returns 500 when user has no flicks") — no REQ-IDs, module numbers, or file paths required from the user
   - Read the description and determine which modules and source files are involved
   - Read the relevant source files under `src/` to locate the bug
   - Identify root cause before invoking `coding-agent`

5. **Implement code changes — per affected module, in dependency order**

   **FOR EACH affected module:**

   a. **Read module spec** from `architecture/modules/module-{N}-{name}.md` (post-update)

   b. **INVOKE `coding-agent`** with context:

      ```
      MODIFY MODE — MODIFY EXISTING PRODUCTION CODE:
      - Module ID: M{N}
      - Module Name: {Module Name}
      - Development Spec: architecture/modules/module-{N}-{name}.md
      - Output Path: src/{module_location}/ (main src/, NOT poc/)
      - Coverage Target: 60% (default per test-gates rule)
      (No Retrofit context — this is Normal mode operating against existing production code, not a POC promotion)

      CHANGE REQUEST: {description of what needs to change in this module, mapped to the user's switch text}
      CT Reference: CT-{NNN}
      EXISTING CODE: Read existing files under src/{module_location}/ before writing. This is a modification of live production code, not a fresh build. Preserve unrelated logic, tests, and public interfaces unless the spec explicitly changes them.
      MODE: {CHANGE|NEW|FIX}
      ```

      For `-new` that introduces a new module, use Normal mode with the module spec as the only source of truth — there is no POC reference at modify time.

   c. **INVOKE `unit-test-generator-agent`** — only when test updates are needed:
      - **`-new` mode:** ALWAYS invoke — new code needs new tests (max 5 per module)
      - **`-change` mode:** Invoke if the public API surface changed or existing tests no longer reflect the spec
      - **`-fix` mode:** Invoke to add one regression test that reproduces the bug (budget permitting — respect the 5-test-per-module hard limit from `test-limits.md`; if the module is already at 5, update an existing test instead of adding a new one)

   d. **INVOKE `unit-tester-agent`** — L1 gate (60% coverage)
      - **PASS:** proceed to smoke
      - **FAIL:** enter L1 Repair Loop (max 5 attempts):
        1. Analyze failure
        2. INVOKE `coding-agent` to fix
        3. INVOKE `unit-tester-agent` to re-run
        4. If still failing after 5 attempts: mark module `Blocked`, INVOKE `tracking-update-agent` with `module_l1_fail`, STOP and report

   e. **INVOKE `smoke-test-agent`** (blocking unless the affected module is foundational with no runnable entry point)
      - **PASS:** proceed
      - **FUNC_FAIL:** INVOKE `coding-agent` to fix dev-mock data (max 2 attempts), re-run smoke
      - **FAIL:** mark module `Blocked`, INVOKE `tracking-update-agent`, STOP and report

   f. **INVOKE `tracking-update-agent`** with `module_l1_pass` (coverage, test count) after L1+smoke both pass

6. **L2 Integration Gate** — INVOKE `l2-integration-agent` **only if** the change touches more than one module OR the Integration Matrix was modified in Phase 3 step 2
   - Skip for single-module `-change` or `-fix` where no integration edges were affected
   - **SUCCESS:** INVOKE `tracking-update-agent` with `l2_pass` → affected modules → `Complete`
   - **BLOCKED:** INVOKE `tracking-update-agent` with `l2_fail` → affected modules → `Blocked`, STOP and report

### Phase 4: Tracking Updates

After successful implementation and all test gates pass:

**Append to `tracking/change-tracking.md`** (append-only, PM log)

1. Determine the next CT number:
   - If the file does not exist, start with CT-001 and create the file with header `# Change Tracking`
   - Otherwise, increment from the last CT number
2. Append a new entry with today's date, the mode tag (`[CHANGE]`, `[NEW]`, or `[FIX]`), and the raw switch text — exactly as the PM wrote it

**Module tracking** is already updated in Phase 3 via `tracking-update-agent` — no additional action here.

## Implementation Flow

```
START → Read switch (exactly one of -change, -new, -fix REQUIRED)
          ↓
   Validate: switch provided? Only one? → ERROR if not
          ↓
   Validate prerequisites (promote-poc run? PRD.md? architecture?) → ERROR if missing
          ↓
   Set mode: CHANGE, NEW, or FIX
          ↓
   Phase 1: Understanding
   ┌──────────────────────────────────────────────┐
   │  1. Read tracking/change-tracking.md         │
   │  2. Read tracking/module-tracking.md         │
   │  3. Read architecture/architecture.md        │
   │  4. Read architecture/modules/* (affected)   │
   │  5. Read PRD.md — parse full structure       │
   └──────────────────────────────────────────────┘
          ↓
   Phase 2: Impact Analysis
   ┌──────────────────────────────────────────────┐
   │  1. Identify affected REQ-IDs / components / │
   │     modules / src files                       │
   │  2. Determine scope:                         │
   │     -change → PRD always + optional arch/spec│
   │     -new    → PRD + arch + spec + code       │
   │     -fix    → code/tests (optional PRD AC)   │
   │  3. Build dependency-ordered module list     │
   └──────────────────────────────────────────────┘
          ↓
   Phase 3: Implementation
   ┌──────────────────────────────────────────────┐
   │  1. Update PRD.md in place (annotated)       │
   │  2. Update architecture.md (if affected)     │
   │  3. Update module specs (if affected)        │
   │  4. Investigate bug (-fix: read source)      │
   │  5. Per module, in dependency order:         │
   │      a. coding-agent                          │
   │      b. unit-test-generator-agent (as needed)│
   │      c. unit-tester-agent (L1 gate)          │
   │         └─ L1 Repair Loop (max 5) on fail    │
   │      d. smoke-test-agent                      │
   │         └─ func fix loop (max 2) on FUNC_FAIL│
   │      e. tracking-update-agent (module_l1_pass│
   │  6. l2-integration-agent (if cross-module)   │
   │      └─ tracking-update-agent (l2_pass|fail) │
   └──────────────────────────────────────────────┘
          ↓
   Phase 4: Tracking Update
   ┌──────────────────────────────────────────────┐
   │  Append CT-{NNN} entry to                    │
   │  tracking/change-tracking.md                 │
   └──────────────────────────────────────────────┘
          ↓
       DONE — Report changes to user
```

## Outputs

Files created or modified:
- `PRD.md` — updated in place with CT-XXX annotations (modes: `-change`, `-new`, and `-fix` when AC is refined); `Document Info → Version` patch-incremented; `Last Updated` set to today
- `architecture/architecture.md` — updated when component boundaries or Integration Matrix change (modes: `-new` always; `-change` sometimes; `-fix` never)
- `architecture/modules/module-{N}-{name}.md` — updated for modules whose specs are affected (modes: `-new` always; `-change` when spec is impacted; `-fix` never)
- `src/{module_location}/**` — code changes scoped to the affected modules; unrelated logic and interfaces preserved
- `tests/unit/{module_name}/**` — test updates generated or refreshed by the test generation step
- `tests/integration/integration.test.{ext}` — integration tests updated by the L2 gate when the change crosses module boundaries

Tracking files updated:
- `tracking/module-tracking.md` — status, coverage, and L2 columns refreshed by the tracking-update step for every touched module
- `tracking/change-tracking.md` — new append-only CT-XXX entry with the PM's raw switch text

Side effects:
- None on external services. `/modify` does not call external APIs, modify databases, deploy, or mutate infrastructure.

Files NEVER touched:
- Anything under `poc/` (POC is historical after promotion)
- No deprecated PRD file (git branch is the backup)
- No root-level `CHANGELOG.md`

## CRITICAL CONSTRAINTS

- **POST-PROMOTION ONLY** — requires `/promote-poc` to have run; refuses to operate on a pre-promotion project
- **Never reads or writes `poc/`** — POC is historical after promotion
- **PRD updates are in place** — no deprecation file is written. Git branch history is the backup (this command is expected to run on a feature branch)
- **No root `CHANGELOG.md`** — the git branch is the technical changelog; `tracking/change-tracking.md` is only the PM log
- **Module tracking is owned by `tracking-update-agent`** — never edit `tracking/module-tracking.md` directly
- **All test gates apply** — L1 (60% coverage, 5-test max), smoke, and L2 when cross-module. Failures STOP the command with a clear error
- **Annotate every PRD edit** with `(Updated via /modify — CT-XXX)`, `(Added via /modify — CT-XXX)`, or `(Refined via /modify — CT-XXX)`
- **Respect hard test limits** from `test-limits.md` — 5 tests per module, 10 integration tests total
- **Preserve untouched code** — `coding-agent` is invoked with `EXISTING CODE: Read existing files before writing`, instructing it to preserve unrelated logic, tests, and public interfaces

## Agents Used

| Agent | Purpose | Invocation |
|-------|---------|------------|
| `coding-agent` | Modify production code for each affected module | Phase 3 — per module, in dependency order |
| `unit-test-generator-agent` | Generate/update L1 tests | Phase 3 — as needed per mode |
| `unit-tester-agent` | L1 coverage gate (60%, max 5 tests) | Phase 3 — after each code change |
| `smoke-test-agent` | Verify app still starts and routes respond | Phase 3 — after L1 passes |
| `l2-integration-agent` | Cross-module integration gate | Phase 3 — only when change spans modules |
| `tracking-update-agent` | Update `tracking/module-tracking.md` | Phase 3 — after L1+smoke, and after L2 |

**Agents NOT used:**
- `re-architect-agent` — the command edits architecture and specs directly; no POC-to-main merge is happening
- `code-promotion-analyzer-agent` — POC is not in scope
- `poc-gap-analyzer-agent` — POC is not in scope
- `coherence-checker-agent` — heavy coherence analysis is owned by `/promote-poc`; `/modify` stays narrow to its requested scope
- `traceability-validator-agent` — not invoked by default (small-scope edits don't need a global traceability sweep). A future flag could add this if it becomes valuable
- `deploy-config-agent` — deployment is out of scope

## Core Requirements

- **MUST** verify prerequisites before any work (promotion done, PRD exists, architecture exists)
- **MUST** have exactly one switch (`-change`, `-new`, or `-fix`) — error if none or multiple provided
- **MUST** read existing `src/` files via `coding-agent` before modifying them (preserve unrelated logic)
- **MUST** update `PRD.md` in place for every `-change` and `-new`, and for `-fix` when clarification is warranted
- **MUST** update `architecture/` and module specs when `-new` is used
- **MUST** run L1 gate for every touched module (60% coverage, max 5 tests)
- **MUST** run smoke-test-agent after L1 passes for every runnable module
- **MUST** run l2-integration-agent when the change spans more than one module or alters the Integration Matrix
- **MUST** INVOKE `tracking-update-agent` rather than writing `tracking/module-tracking.md` directly
- **MUST** append a CT-XXX entry to `tracking/change-tracking.md` only after all gates pass
- **MUST** investigate and locate bugs from natural language descriptions when `-fix` is used — no REQ-IDs, module numbers, or file paths required from the user
- **MUST NOT** read, modify, or reference anything under `poc/`
- **MUST NOT** create a deprecated PRD file or a root-level CHANGELOG.md

## Success Criteria

- [ ] Exactly one switch provided and validated
- [ ] Prerequisites verified (post-promotion state)
- [ ] `PRD.md` updated in place with CT-XXX annotations (for `-change`, `-new`, and `-fix` when AC is refined)
- [ ] `architecture/architecture.md` updated (for `-new`, and `-change` when component boundaries shift)
- [ ] Affected module specs updated (for `-new`, and `-change` when the spec is impacted)
- [ ] Code changes applied under `src/` only (never under `poc/`)
- [ ] L1 passes for every touched module at the 60% coverage target
- [ ] Smoke test passes for every runnable module
- [ ] L2 integration passes when the change crossed module boundaries
- [ ] `tracking/module-tracking.md` updated via `tracking-update-agent` for every module
- [ ] `tracking/change-tracking.md` has a new CT-XXX entry with the PM's raw request
- [ ] PRD `Document Info → Version` incremented and `Last Updated` set to today
