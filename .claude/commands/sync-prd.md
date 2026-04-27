---
description: Merge validated POC changelog entries into PRD.md (deprecates old PRD)
model: claude-opus-4-6
---

## Purpose

Closes the POC feedback loop by merging validated changes from `poc/poc-tracking/CHANGELOG.md` back into the PRD. Deprecates the old `PRD.md` and replaces it with the merged version.

**What this command does:**
- Reads `PRD.md` (merge base) and `poc/poc-tracking/CHANGELOG.md` (validated changes)
- Classifies each CL entry (`[CHANGE]`, `[NEW]`, `[FIX]`) and maps it to PRD sections
- Deprecates the existing `PRD.md` by renaming it to `PRD-Deprecated-<datetime>.md`
- Writes the new merged PRD as `PRD.md` (so it is immediately the active PRD)
- Archives the changelog as `poc/poc-tracking/changelog-Merged-<datetime>.md`

**What this command does NOT do:**
- Read `poc/poc-tracking/change-tracking.md` — it is a PM log that may introduce noise
- Invent requirements — only merges what is explicitly described in CL entries

## Prerequisites

1. `PRD.md` must exist at project root
2. `poc/poc-tracking/CHANGELOG.md` must exist and contain at least one CL entry

If either prerequisite fails, ERROR with a clear message and stop.

## Process

### Phase 1: Prerequisites Check

1. Verify `PRD.md` exists at project root — if not, ERROR: "`PRD.md` not found at project root. Run `/generate-prd` first."
2. Verify `poc/poc-tracking/CHANGELOG.md` exists and is non-empty — if not, ERROR: "`poc/poc-tracking/CHANGELOG.md` not found or empty. Run `/modify-poc` to create changelog entries first."

### Phase 2: Understanding

1. **Read `PRD.md`** — parse the full document structure:
   - Document Info (version, status, date)
   - All sections and subsections
   - All REQ-ID tables (parsed dynamically from PRD section structure)
   - All acceptance criteria lists
   - Any deferred or out-of-scope sections

2. **Read `poc/poc-tracking/CHANGELOG.md`** — parse all CL entries:
   - Extract CL number, title, type tag (`[CHANGE]`, `[NEW]`, `[FIX]`)
   - Extract "PRD Sections Affected" (REQ-IDs or "None — new feature not in original PRD")
   - Extract "Description" and "Current State vs Original"

3. **Classify each CL entry** by its type tag and map "PRD Sections Affected" to actual PRD sections.

4. **Do NOT read `poc/poc-tracking/change-tracking.md`** — it is excluded from this process.

### Phase 3: Merge Planning

For each CL entry, determine the merge action:

**`[CHANGE]` entries:**
- Locate the affected REQ-IDs in PRD (e.g., REQ-2.3)
- Plan updates to the user story description and/or acceptance criteria
- Use the "Current State vs Original" field to understand the new behavior

**`[FIX]` entries:**
- Locate the affected REQ-IDs in PRD
- Plan updates to acceptance criteria to clarify correct behavior
- Fixes typically refine existing requirements rather than changing them

**`[NEW]` entries (PRD Sections = "None — new feature not in original PRD"):**
- Determine the most appropriate existing PRD section based on the feature area
- Plan a new user story with the next available ID in that section's naming convention
- If the feature genuinely doesn't fit any existing section, plan a new subsection

### Phase 4: PRD Generation

1. **Generate timestamp** in format `YYYY-MM-DD_HH-MM` (e.g., `2026-02-16_14-30`)

2. **Copy full PRD structure** as the base for the merged document

3. **Insert a "What Changed (POC Validation)" section** immediately after the document info block and before the first major section. This section provides a user-friendly, plain-language summary of all changes grouped by affected PRD section.

   **Rules for generating this section:**
   - Write in plain language — do NOT mirror the raw changelog table
   - Derive section groupings from affected REQ-IDs (group by prefix or parent ID to map to the PRD sections they belong to)
   - Each bullet describes the change from the user's perspective
   - `[FIX]` entries → phrase as refinements, `[CHANGE]` entries → updated behaviors, `[NEW]` entries → newly added capabilities
   - End each bullet with a parenthetical: "(acceptance criteria refined)", "(user story updated)", or "(new user story added)"
   - Omit CL entry IDs and type tags from the visible summary
   - Include compact merge metadata (date + count) as a single italic line at the top

   Format:
   ```markdown
   ---

   ## What Changed (POC Validation)

   *Merged from POC on <date> — <N> updates applied.*

   ### Feature Area A
   - Updated behavior description from the user's perspective (acceptance criteria refined)

   ### Feature Area B
   - Newly added capability description from the user's perspective (new user story added)

   ---
   ```

4. **Apply merge actions** from Phase 3:

   - **For `[CHANGE]` and `[FIX]` entries:** Update the affected user stories and/or acceptance criteria in-place within their existing section. Append `(Updated from POC validation — CL-XXX)` annotation to each modified item so changes are traceable.

   - **For `[NEW]` entries:** Add new user story rows to the appropriate section's user story table. Add new acceptance criteria to the section's acceptance criteria list. Mark each addition with `(Added from POC validation — CL-XXX)`.

5. **Update Document Info:**
   - Version: increment minor version (e.g., 1.0 → 1.1)
   - Status: "Merged from POC"
   - Last Updated: today's date (YYYY-MM-DD)

6. **Deprecate the existing PRD** by renaming `PRD.md` → `PRD-Deprecated-<datetime>.md` (same timestamp used throughout)
7. **Write the merged PRD** as `PRD.md` at project root (this is now the active PRD)

### Phase 5: Archive & Finalize

1. **Verify** the new `PRD.md` was written successfully (read it back to confirm)
2. **Verify** `PRD-Deprecated-<datetime>.md` exists (the old PRD was preserved)
3. **Rename** `poc/poc-tracking/CHANGELOG.md` → `poc/poc-tracking/changelog-Merged-<datetime>.md` (same timestamp as the deprecated PRD)
4. **Report summary** to the user:
   - Number of CL entries merged
   - Breakdown by type (CHANGE / FIX / NEW)
   - List of affected PRD sections
   - Output file paths: deprecated PRD, new `PRD.md`, and archived changelog
   - Note that the old PRD was preserved as `PRD-Deprecated-<datetime>.md`

## Merge Rules (Critical)

1. **Always deprecate before replacing** — rename existing `PRD.md` to `PRD-Deprecated-<datetime>.md` BEFORE writing the new `PRD.md`
2. **Never read `poc/poc-tracking/change-tracking.md`** — it is a PM log that may introduce noise into the merge
3. **Preserve PRD structure** — the merged PRD must maintain the same section layout, table formats, markdown conventions, and REQ-ID naming patterns as the original
4. **Annotate all changes** — every modification must be traceable to a CL entry via `(Updated from POC validation — CL-XXX)` or `(Added from POC validation — CL-XXX)`
5. **No invention** — only merge what is explicitly described in changelog entries; do not add requirements, acceptance criteria, or user stories beyond what CL entries describe
6. **Timestamp consistency** — all output files (`PRD-Deprecated-<datetime>.md` and `poc/poc-tracking/changelog-Merged-<datetime>.md`) use the exact same datetime string

## Implementation Flow

```
START → Phase 1: Prerequisites Check
          ↓
   PRD.md exists? → NO → ERROR: "Run /generate-prd first"
          ↓ YES
   poc/poc-tracking/CHANGELOG.md exists and non-empty? → NO → ERROR: "Run /modify-poc first"
          ↓ YES
   Phase 2: Understanding
   ┌──────────────────────────────────────────────┐
   │  1. Read PRD.md — parse full structure       │
   │  2. Read CHANGELOG.md — parse CL entries     │
   │  3. Classify entries by type                 │
   │  4. Map entries to PRD sections              │
   │  5. Do NOT read change-tracking.md           │
   └──────────────────────────────────────────────┘
          ↓
   Phase 3: Merge Planning
   ┌──────────────────────────────────────────────┐
   │  For each CL entry:                          │
   │  - [CHANGE] → update user story / AC         │
   │  - [FIX]    → refine acceptance criteria     │
   │  - [NEW]    → add user story + AC to section │
   └──────────────────────────────────────────────┘
          ↓
   Phase 4: PRD Generation
   ┌──────────────────────────────────────────────┐
   │  1. Generate timestamp YYYY-MM-DD_HH-MM      │
   │  2. Copy full PRD as base                    │
   │  3. Insert What Changed section               │
   │  4. Apply merge actions with annotations     │
   │  5. Update Document Info (version, status)   │
   │  6. Rename PRD.md → PRD-Deprecated-<dt>.md   │
   │  7. Write new merged PRD as PRD.md           │
   └──────────────────────────────────────────────┘
          ↓
   Phase 5: Archive & Finalize
   ┌──────────────────────────────────────────────┐
   │  1. Verify new PRD.md written successfully   │
   │  2. Verify PRD-Deprecated-<dt>.md exists     │
   │  3. Rename CHANGELOG.md → archived           │
   │  4. Report summary to user                   │
   └──────────────────────────────────────────────┘
          ↓
       DONE — Report to user
```

## No Agents Required

This is a document transformation task. The command instructs Claude directly to read, analyze, and generate the merged PRD. No coding-agent, smoke-test-agent, or other agents are needed.

## Success Criteria

- [ ] Existing `PRD.md` deprecated to `PRD-Deprecated-<datetime>.md`
- [ ] New merged PRD written as `PRD.md` at project root
- [ ] What Changed section present with user-friendly summary of all changes grouped by PRD section
- [ ] All `[CHANGE]` and `[FIX]` entries reflected as updates to existing user stories / acceptance criteria
- [ ] All `[NEW]` entries added as new user stories in appropriate sections
- [ ] Every modification annotated with `(Updated from POC validation — CL-XXX)` or `(Added from POC validation — CL-XXX)`
- [ ] Document Info updated (version incremented, status "Merged from POC", date updated)
- [ ] `poc/poc-tracking/CHANGELOG.md` renamed to `poc/poc-tracking/changelog-Merged-<datetime>.md`
- [ ] All output files use the same datetime string

## Core Requirements

- **MUST** verify prerequisites before proceeding
- **MUST** read `PRD.md` for full structure before deprecating it
- **MUST** read `poc/poc-tracking/CHANGELOG.md` for CL entries
- **MUST NOT** read `poc/poc-tracking/change-tracking.md`
- **MUST** deprecate existing `PRD.md` to `PRD-Deprecated-<datetime>.md` BEFORE writing the new `PRD.md`
- **MUST** write the merged PRD as `PRD.md` (not a separate file)
- **MUST** preserve PRD structure, table formats, and REQ-ID conventions
- **MUST** annotate every change with its source CL entry
- **MUST** generate a user-friendly "What Changed" summary section in the merged PRD (not a raw copy of changelog entries)
- **MUST** archive the changelog with matching timestamp
- **MUST** report results to the user upon completion

## Next Step

After `/sync-prd` completes, the PRD is authoritative but the production design is not yet updated to reflect the merged changes. Run `/prepare-poc-promo` next to generate the gap-analysis questionnaire; after filling that in, run `/promote-poc` to merge architecture, bootstrap modules, and implement production code.

Flow position:

```
... → /modify-poc (repeat) → /sync-prd → /prepare-poc-promo → [human fills in] → /promote-poc → /setup-env
                                          ↑ you are here
```
