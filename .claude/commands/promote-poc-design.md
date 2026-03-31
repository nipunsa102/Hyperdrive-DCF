---
description: Promote POC architecture and module designs to main architecture with intelligent merge and review
model: claude-opus-4-6
---

## Purpose

Promotes POC design artifacts (architecture and module designs) to the main production architecture. After `/sync-prd` has merged POC changes back into the PRD, this command reconciles the POC's architecture and module designs with the main architecture and module designs, ensures everything aligns with the updated PRD, performs deep coherence analysis, and produces a code promotion plan.

**What this command does:**
- Backs up the existing main architecture and module designs to `architecture/Deprecated-<datetime>/` before making changes
- Intelligently merges POC architecture (`poc/architecture/architecture.md`) into main architecture (`architecture/architecture.md`)
- Intelligently merges POC module designs (`poc/architecture/modules/`) into main module designs (`architecture/modules/`)
- Ensures the merged result perfectly aligns with `PRD.md` (the source of truth)
- Updates Requirement Coverage tables, Integration Matrix, Module Registry, and Component-to-Module Mapping
- Validates the merge via traceability checks, cycle detection, and sum test
- Performs deep coherence analysis across PRD, architecture, and modules (semantic consistency, requirement depth, integration completeness)
- Analyzes POC source code against the merged design and produces `POC_CODE_PROMOTE_PLAN.md` with REFACTOR/REWRITE/WRITE NEW decisions per module

**What this command does NOT do:**
- Generate or modify source code (no changes to `src/` or `poc/src/`)
- Modify POC files (POC architecture is read-only reference; POC source code is read-only for analysis)
- Modify `PRD.md` (reads it as source of truth, never writes to it)
- Run any test gates (this is a design operation, not implementation)
- Copy or migrate POC code (it only creates a plan for that — actual migration is a separate step)

## Prerequisites

1. `PRD.md` must exist at project root (ideally post-`/sync-prd` with POC changes merged in)
2. `poc/architecture/architecture.md` must exist
3. `poc/architecture/modules/` must exist with at least one module file
4. `architecture/architecture.md` must exist (run `/generate-architecture` first if missing)

If any prerequisite fails, ERROR with a clear message and stop.

## Process

### Phase 1: Prerequisites Check

1. Verify `PRD.md` exists — if not, ERROR: "`PRD.md` not found. Run `/generate-prd` first."
2. Verify `poc/architecture/architecture.md` exists — if not, ERROR: "POC architecture not found. Run `/generate-poc` first."
3. Verify `poc/architecture/modules/` has at least one `.md` file — if not, ERROR: "POC module designs not found. Run `/generate-poc` first."
4. Verify `architecture/architecture.md` exists — if not, ERROR: "Main architecture not found. Run `/generate-architecture` first."
5. Check whether `architecture/modules/` has any `.md` files — this determines the merge scenario:
   - **Has modules**: Scenario A (merge both architecture and modules)
   - **No modules**: Scenario B (merge architecture, bootstrap modules from POC)

6. **Check POC changelog sync status:**
   - If `poc/changelog.md` exists AND contains CL-entries:
     - Scan PRD.md for references to those CL-IDs (e.g., "CL-001", "CL-002")
     - If ANY CL-entry has no corresponding reference in PRD.md:
       ERROR: "POC changelog contains unsynced changes (CL-XXX, ...).
       Run `/sync-prd` first to merge POC changes into PRD.md."
   - If `poc/changelog.md` does not exist or has no CL-entries: proceed normally

7. **Detect POC Scope** — Examine the POC's file structure and tech stack to classify its nature:

   Detect POC scope by examining file structure and dependencies:
   - **Frontend indicators**: UI files (`.tsx`, `.jsx`, `.vue`, `.svelte`, `.html`) or frontend frameworks in `poc/package.json` (`react`, `vue`, `angular`, `svelte`, `next`, `nuxt`)
   - **Backend indicators**: Server files (`.py`, `.go`, `.java`, `.rs`, `.rb`), server directories (`server/`, `api/`, `routes/`, `controllers/`), backend manifests (`requirements.txt`, `go.mod`, `Cargo.toml`), or server frameworks in `poc/package.json` (`express`, `fastify`, `nest`, `koa`, `hono`)
   - **Mock indicators**: Directories named `mocks/`, `mock/`, `__mocks__/`, `fixtures/`, or files with "mock", "fake", "stub", "dummy" in filenames

   Classification:
   - `full-stack` = frontend + backend, no mocks
   - `frontend-only` = frontend + (no backend OR uses mocks)
   - `backend-only` = backend + no frontend
   - `undetermined` = none of the above (warn user)

   Store the detected `poc_scope` for use in all downstream phases.

Report the detected scenario and POC scope to the user before proceeding:
```
Detected:
- Merge scenario: [A (merge) | B (bootstrap)]
- POC scope: [frontend-only | backend-only | full-stack | undetermined]
  [If frontend-only]: POC is UI-focused with mock data — main architecture's backend/DB/API depth will be preserved
  [If backend-only]: POC is API/backend-focused — main architecture's UI/frontend depth will be preserved
  [If full-stack]: POC covers both frontend and backend — merge will reconcile both layers
  [If undetermined]: Could not determine POC scope — proceeding with generic merge strategy
```

### Phase 2: Backup & Pre-Merge Snapshot

Before any modifications, back up the existing architecture and log the current state.

1. **Generate timestamp** in format `YYYY-MM-DD_HH-MM` (e.g., `2026-03-03_14-30`). Use this same timestamp throughout the command.

2. **Back up existing architecture** to `architecture/Deprecated-<datetime>/`:
   - Create directory `architecture/Deprecated-<datetime>/`
   - Copy `architecture/architecture.md` → `architecture/Deprecated-<datetime>/architecture.md`
   - If `architecture/modules/` has files (Scenario A):
     - Create directory `architecture/Deprecated-<datetime>/modules/`
     - Copy all `architecture/modules/*.md` → `architecture/Deprecated-<datetime>/modules/`
   - Verify the backup was created successfully before proceeding

3. **Read and log the current state:**
   - Count main architecture components (sections with `**Implements:**` tags)
   - Count main modules (files in `architecture/modules/`)
   - Count POC modules (files in `poc/architecture/modules/`)
   - Count total REQ-IDs in PRD.md

4. **Report to user:**
   ```
   Pre-Merge State:
   - PRD REQ-IDs: [N]
   - Main architecture components: [N]
   - Main modules: [N] (Scenario A) | 0 (Scenario B)
   - POC modules: [N]
   - Merge scenario: A (merge) | B (bootstrap)
   - POC scope: [frontend-only | backend-only | full-stack | undetermined]
   - Backup: architecture/Deprecated-<datetime>/
   ```

### Phase 3: Architecture & Module Merge

**INVOKE `re-architect-agent`** with the following context:

```
PROMOTE-POC-DESIGN — ARCHITECTURE MERGE:

Merge Scenario: [A or B]
POC Scope: [frontend-only | backend-only | full-stack | undetermined]

Source of Truth: PRD.md (post-sync-prd)
POC Reference: poc/architecture/ (read-only)
Merge Target: architecture/ (write here)

Scope-specific merge strategy:
[If frontend-only]: POC is UI-focused — preserve main architecture's backend/DB/API depth
[If backend-only]: POC is API/backend-focused — preserve main architecture's frontend/UI depth
[If full-stack]: Reconcile both layers, preserving depth from both sources
[If undetermined]: Preserve all main architecture depth, only add clear POC contributions
```

**Wait for the re-architect-agent to complete.** Capture its report.

If the re-architect-agent reports FAILED status:
- Display the failure details to the user
- ERROR: "Architecture merge failed. Review the errors above and fix manually, or re-run `/promote-poc-design`."
- STOP — do not proceed to review.

### Phase 4: POC Incorporation Check

Before structural validation, verify that POC changes were actually incorporated into the merge. This check is specific to the promote-poc-design context and performs **content-level comparison**, not just name-matching.

**Perform the following checks directly (no agent needed):**

1. **Screen/Feature Coverage (Content-Level):**
   - Read `poc/architecture/architecture.md` Screen Inventory (or equivalent section)
   - For each POC screen:
     - Verify a corresponding screen layout or description exists in the merged `architecture/architecture.md`
     - **Content check**: Compare the POC screen's described behavior, layout elements, and user interactions against the merged version. A screen that exists by name but has significantly different behavior counts as a partial miss.
     - **Navigation flow check**: Verify that navigation paths to/from this screen described in the POC are preserved in the merged architecture's User Journeys section
   - If a POC screen is missing or has altered behavior, FLAG with specifics:
     ```
     FLAG: Screen "[Name]" — [MISSING | BEHAVIOR_ALTERED]
       POC describes: [summary of POC behavior]
       Merged has: [summary of merged behavior, or "not found"]
       Delta: [what was lost or changed]
     ```

2. **POC Module Functionality (Acceptance Criteria Diff):**
   - For each POC module in `poc/architecture/modules/`:
     - Verify every feature/screen is covered by a main module in `architecture/modules/`
     - **Acceptance criteria comparison**: For each user story in the POC module, compare its acceptance criteria text against the corresponding main module's acceptance criteria. Check for:
       - Criteria that were dropped entirely during merge
       - Criteria that were simplified (lost specificity or edge cases)
       - Criteria whose intent was altered (e.g., "real-time updates" became "periodic refresh")
     - **Feature completeness**: For each feature described in the POC module's overview, verify the main module's overview includes equivalent functionality — not just the same name, but the same scope
   - If features or criteria are missing or weakened, FLAG with specifics:
     ```
     FLAG: Module "[POC Module Name]" → "[Main Module Name]" — [CRITERIA_DROPPED | CRITERIA_WEAKENED | FEATURE_SIMPLIFIED]
       POC states: "[exact POC text]"
       Merged states: "[exact merged text, or 'not found']"
       Impact: [what user-facing behavior is lost]
     ```

3. **PRD Change Annotations:**
   - If `PRD.md` contains annotations like `(Updated from POC validation — CL-XXX)` or `(Added from POC validation — CL-XXX)`:
     - Verify each annotated change is reflected in the merged architecture
     - **Content check**: The annotation's intent must be present in the merged architecture, not just a related component name
   - If not, FLAG and report

4. **POC Changelog Cross-Reference:**
   - Read `poc/changelog.md` (if it exists)
   - For each CL-entry, verify the described change is reflected in the merged
     architecture (check component descriptions, screen layouts, module acceptance criteria)
   - If not reflected, FLAG and auto-fix (add missing feature/behavior)
   - This check is independent of PRD annotations — uses the changelog directly

If any POC incorporation issues are found:
- Apply fixes directly (add missing screens, features, criteria to the merged architecture/modules)
- When fixing CRITERIA_WEAKENED or FEATURE_SIMPLIFIED issues, restore the POC's original specificity while preserving any technical depth added by the merge
- Log every fix with before/after content for the final report

### Phase 5: Traceability & Structural Validation

Three validation checks must all PASS. Run them in a combined loop with auto-fixes.

#### 5a. Traceability Validation

**Invoke traceability-validator-agent** to verify:
- Every REQ-ID in PRD.md appears in at least one module's Requirement Coverage table
- No invalid REQ-IDs (references that don't exist in PRD.md)
- Every module has a Requirement Coverage section
- Architecture components' "Implements:" tags are fully covered by modules

**Auto-Fix for traceability errors:**
For each orphan requirement identified:
1. Determine the most appropriate module based on requirement description
2. Add the REQ-ID to that module's Requirement Coverage table
3. Fill in Implementation column with planned approach
For each missing coverage table:
1. Add "## Requirement Coverage" section to the module (must be first section after title)
For each invalid REQ-ID:
1. Remove from the module or architecture component that references it

#### 5b. Integration Matrix Cycle Detection (BLOCKING)

Verify the Integration Matrix has no circular dependencies using DFS cycle detection on the directed graph formed by (From Module → To Module) edges.

**If Cycle Detected:**
1. Report the exact cycle path (e.g., "M3 → M4 → M5 → M3")
2. **REJECT** the design — do not proceed to Sum Test
3. Suggest fixes: extract shared module, use event decoupling, merge coupled modules, or add abstraction layer

**This check is BLOCKING** — cyclic dependencies cause infinite implementation loops.

#### 5c. Sum Test Validation (BLOCKING)

The Sum Test ensures modules exactly match architecture — no more, no less.

**Validation logic:**
1. Extract features from architecture components (via `Implements:` tags)
2. Extract features from module Requirement Coverage tables
3. Verify every architecture feature is in exactly one module
4. Verify no module claims features not in architecture
5. Verify no architecture component is split across multiple modules

**Error types and fixes:**

| Error | Meaning | Fix |
|-------|---------|-----|
| ORPHAN | Feature in architecture but no module covers it | Add to appropriate module's Requirement Coverage |
| DUPLICATE | Feature covered by multiple modules | Remove from all but one module |
| INVENTED | Module claims feature not in architecture | Remove from module OR add to architecture.md |
| SPLIT | Component divided across modules | Consolidate into single module |

**Auto-fix:** Fix ORPHAN (assign to best module) and DUPLICATE (keep primary owner) automatically. INVENTED and SPLIT require manual review — do not auto-add to architecture.

#### 5d. Combined Validation Loop

Run 5a + 5b + 5c in a loop (max 5 iterations). After each iteration:
- If all PASS → break
- Apply auto-fixes for traceability (5a) and sum test (5c, except INVENTED/SPLIT)
- **Convergence detection:** If total issue count hasn't decreased for 2 consecutive iterations, break early (auto-fixes may be causing a ping-pong effect — manual review required)
- If iterations exhausted with failures remaining → warn user and list remaining errors

### Phase 6: Deep Coherence Analysis

After structural validation passes (or completes with warnings), perform deep semantic coherence analysis on the merged design. This catches inconsistencies that structural checks cannot detect.

**INVOKE `coherence-checker-agent`** with the following context:

```
PROMOTE-POC-DESIGN — DEEP COHERENCE ANALYSIS:

Architecture merge and structural validation complete. Perform deep semantic
coherence analysis on the merged design documents.

Source of Truth: PRD.md (read-only)
Design Documents: architecture/ (read and fix)
EXCLUDED: poc/ (do not read or reference)
```

**Wait for the coherence-checker-agent to complete.** Capture its report.

**Interpret results:**
- **PASS**: All coherence checks passed or all issues were fixed. Proceed to Phase 7.
- **PARTIAL_PASS**: No CRITICAL issues remain, but some MAJOR issues couldn't be auto-fixed. Log warnings and proceed to Phase 7.
- **FAIL**: CRITICAL issues remain after 3 fix iterations. Handle as follows:
  1. Display the remaining CRITICAL issues to the user
  2. **Offer the user a choice:**
     ```
     Coherence analysis FAILED — CRITICAL issues remain after 3 fix iterations.

     Remaining CRITICAL issues:
     - [list each CRITICAL issue with description]

     Options:
     1. PROCEED — Continue to code promotion analysis (Phase 7).
        The code promotion plan will be based on the current (inconsistent) architecture.
        You can fix the design issues manually afterward.

     2. ROLLBACK — Restore the backup from architecture/Deprecated-<datetime>/ and stop.
        This reverts all changes made by Phases 3-6.
        You can then fix the underlying issues and re-run /promote-poc-design.
     ```
  3. **If user chooses ROLLBACK:**
     - Copy `architecture/Deprecated-<datetime>/architecture.md` back to `architecture/architecture.md`
     - If `architecture/Deprecated-<datetime>/modules/` exists, copy all files back to `architecture/modules/` (replacing current files)
     - Delete any module files in `architecture/modules/` that did NOT exist in the backup (new modules created during merge)
     - Report: "Rollback complete. Architecture restored to pre-merge state from architecture/Deprecated-<datetime>/."
     - STOP — do not proceed to Phase 7 or Phase 8
  4. **If user chooses PROCEED (or no response within reasonable time):**
     - WARN: "Proceeding with CRITICAL coherence issues. Manual review of the flagged issues is strongly recommended before running /generate-code."
     - Proceed to Phase 7

### Phase 7: Code Promotion Analysis

After the design is merged and validated, analyze the POC source code against the finalized architecture and module designs to determine the optimal code promotion strategy.

**INVOKE `code-promotion-analyzer-agent`** with the following context:

```
PROMOTE-POC-DESIGN — CODE PROMOTION ANALYSIS:

POC Scope: [frontend-only | backend-only | full-stack | undetermined]

Design Reference: architecture/ (authoritative)
POC Code: poc/src/ and poc/ root configs (read-only analysis)
Existing Code: src/ (read-only check)
Output: POC_CODE_PROMOTE_PLAN.md at project root
EXCLUDED: poc/architecture/, architecture/Deprecated-*/
```

**Wait for the code-promotion-analyzer-agent to complete.**

**Post-analysis check:**
- Verify `POC_CODE_PROMOTE_PLAN.md` was created at project root
- If `src/` has existing code, verify the plan prominently flags the user consent requirement
- Log the plan summary for the final report

### Phase 8: Finalize & Report

1. **Post-Merge Summary:**

```
PROMOTE-POC-DESIGN COMPLETE
=============================

Merge Scenario: [A or B]

BACKUP:
- Previous architecture preserved at: architecture/Deprecated-<datetime>/
  |-- architecture.md
  +-- modules/ (if Scenario A)

CHANGES APPLIED:
- Architecture sections added: [N]
- Architecture sections modified: [N]
- Modules created: [N] ([list names])
- Modules modified: [N] ([list names])
- Modules unchanged: [N] ([list names])

STRUCTURAL VALIDATION (Phase 5):
- Traceability (REQ-ID coverage): [PASS/FAIL]
- Cycle Detection (DAG): [PASS/FAIL]
- Sum Test: [PASS/FAIL]
- POC Incorporation: [PASS/FAIL]
- Auto-fix iterations: [N] of 5

COHERENCE ANALYSIS (Phase 6):
- Status: [PASS/PARTIAL_PASS/FAIL]
- Issues found: [N] (CRITICAL: [N], MAJOR: [N], MINOR: [N])
- Issues fixed: [N]
- Issues remaining: [N]
- Fix iterations: [N] of 3

CODE PROMOTION PLAN (Phase 7):
- Plan created: POC_CODE_PROMOTE_PLAN.md
- Modules to REFACTOR: [N] ([list])
- Modules to REWRITE: [N] ([list])
- Modules to WRITE NEW: [N] ([list])
[If src/ has existing code:]
- WARNING: Existing code in src/ — user consent required before changes

[If warnings remain:]
REMAINING WARNINGS:
- [list]

OUTPUT FILES:
- architecture/architecture.md (updated)
- architecture/modules/module-{N}-{name}.md (updated/created) x [N]
- POC_CODE_PROMOTE_PLAN.md (created)

NEXT STEPS:
- Review the merged architecture at architecture/architecture.md
- Review module designs in architecture/modules/
- Review the code promotion plan at POC_CODE_PROMOTE_PLAN.md
[If coherence issues remain:]
- Address remaining coherence issues flagged above
- To revert: restore from architecture/Deprecated-<datetime>/
- When ready: follow POC_CODE_PROMOTE_PLAN.md to set up src/, then run /generate-code for each module
```

## Rules

1. **Back up before modifying** — Copy existing architecture to `architecture/Deprecated-<datetime>/` BEFORE merge. Verify backup exists before invoking re-architect-agent. Use same timestamp throughout.
2. **Design-only** — Never modify source code (`src/`, `poc/src/`). Only design documents and `POC_CODE_PROMOTE_PLAN.md`.
3. **PRD is source of truth** — Every REQ-ID must be covered in both architecture and modules. Never modify `PRD.md`.
4. **POC files are read-only** — Never modify anything under `poc/`.
5. **Preserve technical depth** — Main architecture's backend/DB/API details (or frontend/UI details for backend POCs) must survive the merge.
6. **No invention** — Only merge what exists in PRD and POC designs. No new requirements, components, or features.
7. **User consent for existing code** — If `src/` has code, `POC_CODE_PROMOTE_PLAN.md` must prominently flag that consent is required before deletion/overwriting.
8. **Structural completeness** — Every component needs `Implements:` tags. Every module starts with Requirement Coverage table. Module Registry, Component-to-Module Mapping, and Integration Matrix (all 5 columns) must be complete.
9. **All validations must pass** — Traceability (PASS), cycle detection (no cycles), sum test (PASS), coherence (PASS or PARTIAL_PASS).
10. **Detect and report** — Detect merge scenario (A/B) and POC scope before merging. Report full summary upon completion.
