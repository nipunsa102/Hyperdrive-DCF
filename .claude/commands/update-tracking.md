---
description: Update Tracking
model: claude-opus-4-6
---

**Switches**: `-module`, `-status`

**Switch Definitions**:
- `-module` — Specific module to update (e.g., `M1`, `M2`, `M3`)
- `-status` — New status. One of: `Not Started`, `In Progress`, `L1 Pass`, `Blocked`, `Complete`, `Deployed`

## Purpose

Updates module status in the tracking document at `tracking/module-tracking.md`. Ensures the status table and summary counts stay accurate. Typically invoked programmatically from other workflow commands (via the tracking-update agent), but can also be run manually with switches to correct a specific module's status.

## Prerequisites

1. `tracking/module-tracking.md` must exist. If missing, ERROR: `"tracking/module-tracking.md not found. Run /promote-poc or /generate-code first to initialize tracking."`
2. `architecture/architecture.md` must exist (the module registry is the source of truth for what modules exist). If missing, ERROR: `"architecture/architecture.md not found. Run /generate-architecture first."`

If any prerequisite fails, ERROR with a clear message and stop.

## Process

### Phase 1: Read current state

1. **Read `tracking/module-tracking.md`** and parse:
   - The Summary block (total, counts per status)
   - The Module Status table (per-module rows: ID, Name, Status, L1 Coverage, L2 Status, Dependencies, Notes)
   - The Blockers section (if present)
   - The History section (append-only log)

2. **Read `architecture/architecture.md` Module Registry** to confirm the set of modules that should appear in the tracking table. If a module in the registry is missing from the tracking table, note it as a recovery action in Phase 2.

### Phase 2: Apply update

Two modes of operation:

- **Targeted update (both `-module` and `-status` provided):** update just the named module's Status column. If `-status` is `L1 Pass` or `Complete`, the invoking context is expected to also pass `L1 Coverage` and, where relevant, `L2 Status` as part of the invocation payload — write those if provided.
- **Sync mode (no switches):** do not change any module's status; only recalculate Summary counts from the current Module Status table. This is the default automatic-invocation behavior when another workflow step wants to refresh counts after a batch of changes.

Validation rules:
- Reject an unknown `-status` value with a clear error (list allowed values).
- Reject `-module` values not present in the Module Registry with a clear error.
- If targeting a module that is currently `Blocked`, require the new status to be explicit — do not silently clear a blocker.

### Phase 3: Recalculate summary and write back

1. **Recompute Summary counts** from the (possibly updated) Module Status table:
   - Total Modules = row count
   - Complete = rows with Status `Complete`
   - L1 Pass = rows with Status `L1 Pass`
   - In Progress = rows with Status `In Progress`
   - Blocked = rows with Status `Blocked`
   - Not Started = rows with Status `Not Started`
   - Deployed = rows with Status `Deployed`

2. **Append a History entry** for any status transition made this run (timestamp + `M{N}: {old} -> {new}`).

3. **Rewrite `tracking/module-tracking.md`** with updated Summary block, updated Module Status table, updated Blockers section (if a module transitioned to/from Blocked), and the appended History entry. Preserve everything else verbatim.

4. **Report** a concise summary to the caller:
   ```
   Updated tracking: M{N} -> {new status}
   Summary: {N} Complete, {N} L1 Pass, {N} In Progress, {N} Blocked, {N} Not Started [, {N} Deployed]
   ```

## Outputs

Files modified:
- `tracking/module-tracking.md` — Summary counts, Module Status row(s), Blockers section (if relevant), History section (append-only)

No side effects outside the tracking file. This command does not touch source code, architecture, or any external services.

## Rules

1. **Tracking is append-only for history** — never rewrite or remove History entries, only add.
2. **Summary must match the table** — counts are derived from the table, not stored independently.
3. **Blockers section stays in sync** — if a module transitions to `Blocked`, add it to Blockers with the reason; if it transitions away from `Blocked`, remove the entry.
4. **Never silently clear a blocker** — targeted updates from `Blocked` to another status require the caller to pass the new status explicitly.
5. **Registry is source of truth for membership** — modules are added to the tracking table only when they appear in the architecture's Module Registry.

## Agents Used

None. This command is a file-update utility and does not invoke any agents.

## Core Requirements

- **MUST** refuse to operate if prerequisites are missing
- **MUST** preserve History (append-only)
- **MUST** recalculate Summary counts on every run (targeted or sync mode)
- **MUST** keep Blockers section consistent with Module Status table
- **MUST NOT** modify any file outside `tracking/module-tracking.md`
- **MUST NOT** invoke agents

## Module Tracking Document Format (reference)

```markdown
# Module Tracking

## Summary
- Total Modules: 5
- Complete: 2
- L1 Pass: 1
- In Progress: 1
- Blocked: 0
- Not Started: 1

## Module Status

| Module | Name | Status | L1 Coverage | L2 Status | Dependencies | Notes |
|--------|------|--------|-------------|-----------|--------------|-------|
| M1 | Data Layer | Complete | 85% | Pass | None | All tests pass |
| M2 | API Layer | Complete | 82% | Pass | M1 | All tests pass |
| M3 | Core UI | L1 Pass | 75% | Pending | M1, M2 | Awaiting L2 |
| M4 | Content Mgmt | In Progress | - | - | M2 | Implementation ongoing |
| M5 | Editors | Not Started | - | - | M3, M4 | |

## Blockers

(List any blocked modules with reasons)

## History

### 2026-04-22
- M1: Complete (85% coverage, L2 pass)
- M2: Complete (82% coverage, L2 pass)
- M3: L1 Pass (75% coverage)
```

## Status Definitions

| Status | Description |
|--------|-------------|
| `Not Started` | Module implementation not yet begun |
| `In Progress` | Module currently being implemented |
| `L1 Pass` | Module passed L1 unit tests (meets coverage target); awaiting L2 |
| `Blocked` | Module blocked by test failure, dependency, or promotion decision |
| `Complete` | Module passed both L1 and L2 gates |
| `Deployed` | Module deployed to a target environment |
