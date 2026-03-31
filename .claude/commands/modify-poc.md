---
description: Implement stakeholder-requested changes to the POC with full tracking
model: claude-opus-4-6
---

**Switches**: `-change`, `-new`, `-fix`

**Switch Definitions**:
- `-change` → Description of a modification to existing POC functionality, in the product manager's own words
- `-new` → Description of a new feature to add that did not exist in the original POC, in the product manager's own words
- `-fix` → Description of something broken in the POC that needs fixing (e.g., "calendar page crashes when clicking next month", "search bar doesn't filter results")

**Switch Rules:** Exactly one switch must be provided. If none is provided, or more than one is provided, ERROR.

## Purpose

Implements stakeholder-requested changes to the POC and tracks all modifications. After stakeholders review the POC, the product manager captures their feedback and uses this command to (1) implement those changes in POC code, architecture, and module specs and (2) maintain two tracking files for traceability.

**Three modes:**
- **`-change`**: Modifies existing POC functionality (e.g., "make the calendar show week numbers", "change the table sort order")
- **`-new`**: Adds a feature that did not exist in the original POC (e.g., "add an asset comparison view", "add a dashboard widget for overdue counts"). New features require POC architecture and module spec updates.
- **`-fix`**: Fixes something broken in the POC (e.g., "calendar crashes when clicking next month", "search doesn't work"). Bug fixes — no architecture or spec changes, lightweight tracking only.

**What this command does:**
- Implements the requested change, new feature, or bug fix across POC code, architecture, and module specs as needed
- Appends a raw log entry to `poc/change-tracking.md` (product manager reference)
- Creates or updates a detailed technical entry in `poc/changelog.md` (future `/sync-prd` consumption)
- Verifies the POC still works after changes

**What this command does NOT do:**
- Modify anything outside `poc/` (never touches main `src/`, `architecture/`, `tracking/`)
- Modify `PRD.md` or main architecture docs (reads them for context only)
- Propagate changes back to PRD (separate future `/sync-prd` command)

## Prerequisites

1. POC must exist (`poc/` folder with working application — run `/generate-poc` first)
2. `PRD.md` must exist (for requirement context)

## Tracking Files

This command maintains two tracking files in `poc/`:

### `poc/change-tracking.md` — Product Manager Log

**Audience:** Product manager (non-technical)
**Purpose:** Simple append-only log of what was requested, in the PM's own words
**Not consumed by any command** — purely for human reference

**Format:**
```markdown
# POC Change Tracking

## CT-001: {YYYY-MM-DD} [CHANGE]
{Raw description from the -change switch, exactly as the PM wrote it}

## CT-002: {YYYY-MM-DD} [NEW]
{Raw description from the -new switch, exactly as the PM wrote it}

## CT-003: {YYYY-MM-DD} [FIX]
{Raw description from the -fix switch, exactly as the PM wrote it}
```

**Rules:**
- Append-only — never edit or remove previous entries
- CT numbers are sequential (CT-001, CT-002, CT-003, ...)
- Date is the date the change was implemented
- Tag is `[CHANGE]`, `[NEW]`, or `[FIX]` matching the switch used
- Content is the raw switch text, unmodified

### `poc/changelog.md` — Technical Living Document

**Audience:** Technical team / future commands (especially `/sync-prd`)
**Purpose:** Detailed breakdown of what changed across code, POC architecture, and POC module specs

**Format:**
```markdown
# POC Changelog

### CL-001: {Short Title} [CHANGE|NEW|FIX]
**PRD Sections Affected:** {REQ-IDs from PRD.md, e.g., REQ-2.1, REQ-3.3} (or "None — new feature not in original PRD" for `-new`)
**Date First Changed:** {YYYY-MM-DD}
**Date Last Updated:** {YYYY-MM-DD}
**Description:** {What changed and why}
**Current State vs Original:** {How the POC now differs from the original PRD/architecture}
**Architecture Changes:**
- `poc/architecture/architecture.md` — {What was updated} (or "None")
**Module Spec Changes:**
- `poc/architecture/modules/module-{N}-{name}.md` — {What was updated} (or "None")
**Code Changes:**
- `poc/src/path/to/file.tsx` — {What was added/modified}
```

**Rules:**
- Living document — entries for the same area are merged/updated (not duplicated)
- When a new change affects the same area as a prior CL entry, UPDATE that entry:
  - Update "Date Last Updated"
  - Update "Description" and "Current State vs Original" to reflect cumulative state
  - Add new files to the changes lists
- When a change affects a NEW area, create a new CL-{NNN} entry
- CL numbers are sequential and never reused

## Process

### Phase 1: Understanding

1. **Read request** from the provided switch
   - If no switch is provided, ERROR: "A switch is required. Usage: /modify-poc -change \"description\" OR /modify-poc -new \"description\" OR /modify-poc -fix \"description\""
   - If more than one switch is provided, ERROR: "Use only one switch at a time. Use -change for modifications, -new for new features, or -fix for bug fixes."
   - Record the **mode** for use in later phases: `CHANGE`, `NEW`, or `FIX`

2. **Read existing tracking files** (if they exist)
   - Read `poc/changelog.md` to understand prior changes and current state
   - Read `poc/change-tracking.md` to see request history and determine next CT number

3. **Read POC architecture and module specs**
   - Read `poc/architecture/architecture.md` for current POC structure
   - Read relevant POC module specs under `poc/architecture/modules/`

4. **Read PRD.md** for requirement context (read-only — do NOT modify)
   - Identify which REQ-IDs are affected by the change request

### Phase 2: Impact Analysis

1. **Identify affected areas:**
   - Which POC modules/screens are affected?
   - Which POC source files need changes?
   - Does the POC architecture doc need updating?
   - Do any POC module specs need updating?

2. **Check changelog for prior changes to the same areas:**
   - If a prior CL entry covers the same area, plan to UPDATE that entry (not create a new one)
   - If this is a new area, plan to create a new CL entry

3. **Determine scope based on mode:**
   - **`-change` mode:** May be code only (visual/behavioral tweak) OR architecture + specs + code (if the modification affects structure). If the change is structurally significant enough to warrant a new module, a new module spec may be created.
   - **`-new` mode:** Always requires architecture + module specs + code — new features MUST be reflected in POC architecture and at least one module spec before code is written
   - **`-fix` mode:** Code only — bug fixes do NOT update architecture or module specs (the specs describe intended behavior, which hasn't changed)

### Phase 3: Implementation

1. **Update POC architecture** (REQUIRED for `-new`, if affected for `-change`, SKIP for `-fix`)
   - **`-new` mode:** MUST update `poc/architecture/architecture.md` — add the new feature to the screen inventory, navigation flow, and component list
   - **`-change` mode:** Update `poc/architecture/architecture.md` only if the change affects structure, navigation, or data model
   - **`-fix` mode:** SKIP — bug fixes don't change architecture (the design was correct, the code wasn't)
   - Only update sections relevant to the change

2. **Update POC module specs** (REQUIRED for `-new`, if affected for `-change`, SKIP for `-fix`)
   - **`-new` mode:** MUST update or create module specs in `poc/architecture/modules/` — add new screens, acceptance criteria, and mock data requirements. If the feature fits an existing module, update that spec. If it requires a new module, create `poc/architecture/modules/module-{N}-{name}.md`
   - **`-change` mode:** Update affected specs if the change impacts screen descriptions, acceptance criteria, or mock data requirements. If the modification is structurally significant enough to warrant a new module (e.g., splitting a screen into separate concerns), create `poc/architecture/modules/module-{N}-{name}.md`
   - **`-fix` mode:** SKIP — the specs already describe the correct behavior

3. **Investigate and locate the issue** (`-fix` mode only)
   - The `-fix` switch accepts a plain natural language description (e.g., "the calendar crashes when I click next month") — no PRD IDs, module numbers, or file paths required
   - Read the description and determine which POC screens/components are involved
   - Read the relevant source files under `poc/src/` to locate the bug
   - Identify root cause before invoking coding-agent

4. **Implement code changes**
   - **MUST INVOKE `coding-agent`** for each affected POC module with the following context:

   ```
   POC MODE — MODIFY-POC CHANGE IMPLEMENTATION:
   - Module ID: POC-M{N}
   - Module Name: {Module Name}
   - Development Spec: poc/architecture/modules/module-{N}-{name}.md
   - Output Path: ALL source code goes under poc/src/ (NOT the main src/)
   - Coverage Target: 0% (no unit test coverage required for POC)
   - This is a POC — ALL data is mocked, ALL integrations are mocked
   - No real API calls, no real database, no real authentication
   - Mock data comes from local files or hardcoded constants in poc/src/mocks/
   -
   - CHANGE REQUEST: {description of what needs to change in this module}
   - EXISTING CODE: Read existing files before modifying — this is a modification, not a fresh build
   ```

5. **Verify POC still works**
   - **MUST INVOKE `smoke-test-agent`** to verify:
     - POC starts without errors (e.g., `cd poc && npm install && npm run dev`)
     - All routes still render
     - No console errors on navigation

6. **Fix loop** (if smoke test fails):
   - **INVOKE `coding-agent`** to fix the issue
   - **INVOKE `smoke-test-agent`** again
   - Maximum 3 fix attempts
   - If still failing after 3 attempts: STOP and report the failure to the user

### Phase 4: Tracking Updates

After successful implementation and smoke test:

**A) Update `poc/change-tracking.md`** (append-only)

1. Determine the next CT number:
   - If file doesn't exist, start with CT-001
   - Otherwise, increment from the last CT number
2. Append a new entry with today's date, the mode tag (`[CHANGE]` or `[NEW]`), and the raw switch text
3. If the file doesn't exist, create it with the header `# POC Change Tracking` followed by the first entry

**B) Update `poc/changelog.md`** (living document — for `-change` and `-new` modes only)

1. **`-fix` mode:** Create a lightweight `[FIX]` entry — code changes only, no architecture or module spec changes. Description should note what was broken and what was fixed.
2. **`-change` / `-new` mode:** Determine if this change should UPDATE an existing CL entry or CREATE a new one:
   - If a prior CL entry covers the same functional area or PRD sections → UPDATE it
   - If this is a new area → CREATE a new CL entry with the next CL number
3. For new entries: Include all fields (PRD sections, dates, description, current state vs original, architecture/module/code changes)
4. For updated entries: Update "Date Last Updated", revise "Description" and "Current State vs Original" to reflect cumulative state, add new files to change lists
5. If the file doesn't exist, create it with the header `# POC Changelog` followed by the first entry

## Implementation Flow

```
START → Read switch (exactly one of -change, -new, -fix REQUIRED)
          ↓
   Validate: switch provided? Only one? → ERROR if not
          ↓
   Set mode: CHANGE, NEW, or FIX
          ↓
   Phase 1: Understanding
   ┌──────────────────────────────────────────────┐
   │  1. Read poc/changelog.md (prior changes)    │
   │  2. Read poc/change-tracking.md (CT numbers) │
   │  3. Read poc/architecture/architecture.md    │
   │  4. Read poc/architecture/modules/*          │
   │  5. Read PRD.md (context only, NOT modified) │
   └──────────────────────────────────────────────┘
          ↓
   Phase 2: Impact Analysis
   ┌──────────────────────────────────────────────┐
   │  1. Identify affected modules/files/docs     │
   │  2. Check changelog for prior same-area edits│
   │  3. Determine scope:                         │
   │     -change → code only OR arch+specs+code   │
   │     -new    → ALWAYS arch+specs+code         │
   │     -fix    → code only (investigate bug)    │
   └──────────────────────────────────────────────┘
          ↓
   Phase 3: Implementation
   ┌──────────────────────────────────────────────┐
   │                                               │
   │  1. Update poc/architecture/ (MUST for -new)  │
   │  2. Update poc module specs (MUST for -new)   │
   │  3. Investigate bug (-fix: read source code)  │
   │  4. coding-agent (per affected module)        │
   │  5. smoke-test-agent (verify POC works)       │
   │     └─ Fix loop (max 3) if fails              │
   │                                               │
   └──────────────────────────────────────────────┘
          ↓
   Phase 4: Tracking Updates
   ┌──────────────────────────────────────────────┐
   │  A. change-tracking.md — Append with [TAG]   │
   │  B. changelog.md:                            │
   │     -change/-new → full or merged CL entry   │
   │     -fix → lightweight [FIX] CL entry        │
   └──────────────────────────────────────────────┘
          ↓
       DONE — Report changes to user
```

## CRITICAL CONSTRAINTS

- **ALL changes stay within `poc/` folder** — never touch main `src/`, `architecture/`, or `tracking/`
- **Follows POC mode rules** — mock data, no real backends, frontend only
- **Reads but does NOT modify** `PRD.md` or main `architecture/architecture.md`
- **`poc/change-tracking.md` is append-only** — never edit previous entries
- **`poc/changelog.md` is a living document** — merge/update entries for the same area
- **The command does NOT propagate changes to PRD** — that is a separate future command

## Success Criteria

- [ ] Requested change is implemented in POC code
- [ ] POC architecture and module specs updated (if the change warrants it)
- [ ] POC still starts and runs (e.g., `cd poc && npm install && npm run dev`)
- [ ] Smoke test passes
- [ ] `poc/change-tracking.md` has new log entry with PM's raw request
- [ ] `poc/changelog.md` has detailed technical entry (new or merged with existing)

## Agents Used

| Agent | Purpose | Invocation |
|-------|---------|------------|
| `coding-agent` | Implement POC code changes (POC mode) | Phase 3 — per affected module |
| `smoke-test-agent` | Verify POC still starts and works | Phase 3 — after all code changes |

**Agents NOT used in modify-poc:**
- `unit-test-generator-agent` — No unit tests for POC
- `unit-tester-agent` — No L1 gate for POC
- `l2-integration-agent` — No integration tests for POC
- `traceability-validator-agent` — No traceability validation for POC
- `tracking-update-agent` — No main tracking updates for POC changes
- `deploy-config-agent` — No deployment for POC
- `code-review-agent` — No code review for POC

## Core Requirements

- **MUST** read POC architecture and changelog before making changes
- **MUST** read PRD.md for requirement context (but never modify it)
- **MUST** invoke `coding-agent` for code changes with POC mode context
- **MUST** invoke `smoke-test-agent` after all code changes
- **MUST** update both tracking files after successful implementation
- **MUST** keep all changes within `poc/` folder
- **MUST** have exactly one switch (`-change`, `-new`, or `-fix`) — error if none or multiple provided
- **MUST** update POC architecture and module specs when `-new` is used (new features always need documentation)
- **MUST** investigate and locate bugs from natural language descriptions when `-fix` is used — no PRD IDs, module numbers, or file paths required from the user
