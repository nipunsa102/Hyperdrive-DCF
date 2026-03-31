# POC Promotion Algorithm

Concise logic reference for `/promote-poc-design`. For full details, see [`.claude/commands/promote-poc-design.md`](.claude/commands/promote-poc-design.md).
Part of the [DCF workflow](DCF.md).

---

## Pipeline at a Glance

```
Phase 1 ─ Prerequisites + Sync Guard ─── inline
Phase 2 ─ Backup ─────────────────────── inline
Phase 3 ─ Merge ──────────────────────── re-architect-agent
Phase 4 ─ POC Incorporation Check ────── inline
Phase 5 ─ Structural Validation ───────── traceability-validator + inline algorithms
Phase 6 ─ Semantic Validation ─────────── coherence-checker-agent
Phase 7 ─ Code Promotion Plan ─────────── code-promotion-analyzer-agent
Phase 8 ─ Report ─────────────────────── inline
```

**Authority chain (never violated):**
```
PRD.md (read-only) > architecture.md (modified) > modules/*.md (modified)
```

---

## Phase 1: Prerequisites + Sync Guard

```
PRD.md exists? ──── NO ──→ ERROR: run /generate-prd
       │ YES
poc/architecture/ exists? ── NO ──→ ERROR: run /generate-poc
       │ YES
architecture/architecture.md exists? ── NO ──→ ERROR: run /generate-architecture
       │ YES
architecture/modules/ has .md files?
  ├── YES → Scenario A (merge)
  └── NO  → Scenario B (bootstrap)    ← PRIMARY PATH
       │
poc/changelog.md has unsynced CL-entries?
  ├── YES (CL-IDs missing from PRD) → ERROR: run /sync-prd first     ← C1 GUARD
  └── NO  → continue
       │
Detect POC scope → frontend-only | backend-only | full-stack | undetermined
       │
       ▼
Phase 2
```

**POC scope drives merge strategy** — which layers to preserve vs. integrate.

Backend detection uses **recursive** glob (`**/server/`, `**/api/`, etc.) to catch nested structures.

---

## Phase 2: Backup

```
Generate timestamp YYYY-MM-DD_HH-MM
       │
Copy architecture/ → architecture/Deprecated-<ts>/
  ├── architecture.md
  └── modules/ (if Scenario A)
       │
Verify backup → FAIL → ERROR
       │ OK
Log counts (REQ-IDs, components, modules)
       │
       ▼
Phase 3
```

---

## Phase 3: Architecture & Module Merge (re-architect-agent)

```
┌─────────────────────────────────────────────────────────────┐
│  Step 1: Read all sources (PRD, both archs, all modules)    │
│  Step 2: Confirm scenario A or B                            │
│  Step 3: Diff analysis (new/modified/removed components)    │
│  Step 4: Merge architecture.md                              │
│     • Preserve main structure + technical depth             │
│     • Integrate POC changes                                 │
│     • Create or update Module Registry                      │
│     • Create or update Integration Matrix (all 5 cols)      │
│  Step 5: Module merge                                       │
│     • Scenario A: merge POC changes into existing modules   │
│     • Scenario B: copy POC modules, enhance for production  │
│  Step 6: Self-validation (arch↔PRD, modules↔arch)           │
│  Step 7: Cleanup (remove merge markers)                     │
└──────────────────────────┬──────────────────────────────────┘
                           │
                   PASS? ──┤── NO → ERROR + STOP
                           │
                           ▼
                        Phase 4
```

### Scenario B Logic (primary path)

```
POC modules ──→ Production modules
                 │
                 ├── Preserve POC number + name
                 ├── Add Requirement Coverage table (first section)
                 ├── Expand to full stack (UI → add backend; backend → add UI)
                 ├── Add Data Needs, Interactions, Implements: tags
                 ├── Add Technical Details (lighter depth OK)
                 │
                 ├── Create new modules for arch components with no POC match
                 └── Create/update Registry, Integration Matrix, Component Mapping
```

**No boundary re-derivation** — POC boundaries are preserved and enhanced.

---

## Phase 4: POC Incorporation Check

```
For each POC screen:
  Compare behavior + layout + navigation → merged architecture
  FLAG if MISSING or BEHAVIOR_ALTERED

For each POC module's acceptance criteria:
  Diff against merged module criteria
  FLAG if CRITERIA_DROPPED | CRITERIA_WEAKENED | FEATURE_SIMPLIFIED

For each PRD CL-annotation:
  Verify intent present in merged architecture

For each CL-entry in poc/changelog.md:          ← C2 CROSS-REFERENCE
  Verify described change in merged architecture
  FLAG + auto-fix if missing

Auto-fix all flags → restore POC specificity while keeping merge depth
       │
       ▼
Phase 5
```

---

## Phase 5: Structural Validation (loop)

```
┌─────────────────────────────────────────┐
│  REPEAT (max 5 iterations):             │
│                                         │
│  5a. traceability-validator-agent        │
│      → REQ-ID coverage, valid refs      │
│                                         │
│  5b. Cycle detection (inline DFS)       │
│      → Integration Matrix is a DAG      │
│      → BLOCKING if cycle found          │
│                                         │
│  5c. Sum test (inline set algebra)      │
│      → No ORPHAN/DUPLICATE/INVENTED/    │
│        SPLIT features                   │
│                                         │
│  ALL PASS? → break                      │
│                                         │
│  Convergence check:                     │
│    issue_count not decreasing            │
│    for 2 iterations? → break early      │
│                                         │
│  Auto-fix → next iteration              │
└─────────────┬───────────────────────────┘
              │
              ▼
           Phase 6
```

---

## Phase 6: Semantic Validation (coherence-checker-agent)

```
┌─────────────────────────────────────────────────────────┐
│  Context awareness:                                      │
│    Freshly bootstrapped modules?                         │
│    → SHALLOW_COVERAGE = MINOR (not MAJOR)                │
│    → Focus fixes on CRITICAL + MAJOR semantic issues     │
│                                                          │
│  Phase 1: Semantic coherence                             │
│    PRD ↔ Architecture (intent, completeness, contradictions)
│    Architecture ↔ Modules (data models, APIs, flows)     │
│    Cross-module (shared entities, handoffs, state)        │
│                                                          │
│  Phase 2: Requirement depth                              │
│    User stories address REQ-IDs? Implementation detail?  │
│    Acceptance criteria complete?                          │
│                                                          │
│  Phase 3: Integration Matrix depth                       │
│    Completeness, phantoms, interface accuracy,            │
│    error strategies, bidirectional consistency            │
│                                                          │
│  Fix loop (max 3): CRITICAL → MAJOR → MINOR             │
│                                                          │
│  CANNOT read poc/ — validates merged docs on own merit   │
└──────────────────────┬──────────────────────────────────┘
                       │
              ┌────────┴────────┐
              │                 │
           PASS /            FAIL
         PARTIAL_PASS          │
              │         User choice:
              │         PROCEED or ROLLBACK
              │                 │
              ▼                 ▼
           Phase 7         (restore backup + STOP)
```

---

## Phase 7: Code Promotion Analysis (code-promotion-analyzer-agent)

```
┌─────────────────────────────────────────────────────────┐
│  Reads: main architecture/ (authoritative)               │
│         poc/architecture/ (supplementary context)         │
│         poc/src/ (analysis target)                        │
│         src/ (existing code check)                        │
│                                                          │
│  Step 1: Inventory POC code (files, LOC, quality)        │
│  Step 2: Map POC files → production modules              │
│  Step 3: Gap analysis per module                         │
│  Step 4: Technical compatibility (7 dimensions)          │
│  Step 5: Existing production code assessment             │
│  Step 6: Decision per module:                            │
│                                                          │
│    REFACTOR ── POC code worth preserving, ≥30% coverage  │
│                (>60% + <30% changes = "minimal adapt")   │
│    REWRITE ─── POC as reference only (mocks, mismatch)   │
│    WRITE NEW ─ No POC code for this module               │
│                                                          │
│  Step 7: Write POC_CODE_PROMOTE_PLAN.md                  │
│          (includes dev-mode data reqs, migration steps)  │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
                    Phase 8 → Summary report → DONE
```

---

## Validation Layer Map

Each layer catches a different class of error at a different trust boundary:

```
Phase 3 self-check ──── "Did the merge agent do what it intended?"
         │
Phase 4 POC check ───── "Did POC features survive the merge?"
         │                (only phase that reads POC + merged docs)
Phase 5 structural ──── "Are REQ-IDs, DAG, and feature sets correct?"
         │                (independent external validator)
Phase 6 semantic ─────── "Does the content actually make sense together?"
         │                (POC-blind — validates docs on own merit)
Phase 7 code-level ──── "Is the POC code compatible with the design?"
                          (practical migration feasibility)
```

---

## Safety Mechanisms

| Mechanism | Phase | Risk Mitigated |
|-----------|-------|----------------|
| Timestamped backup | 2 | Worst-case recovery |
| Sync guard (CL-entries) | 1 | Unsynced POC changes silently lost |
| Hard stop on merge failure | 3 | Validating broken output |
| Content-level POC check | 4 | Silent feature drops in merge |
| Changelog cross-reference | 4 | POC changes not in PRD annotations |
| Convergence detection | 5 | Auto-fix ping-pong loops |
| Fresh-bootstrap awareness | 6 | False MAJOR flags on new modules |
| User choice on FAIL | 6 | Unrecoverable coherence issues |
| User consent flag | 7 | Overwriting existing production code |
| PRD + POC read-only | All | Source of truth never modified |

---

## Decision Values (canonical)

Only three valid decisions exist across the entire pipeline:

| Decision | Meaning | When |
|----------|---------|------|
| **REFACTOR** | Copy POC code, adapt for production | Structure worth preserving, ≥30% coverage |
| **REWRITE** | Fresh code, POC as visual/behavioral reference | Mocks, framework mismatch, poor quality, <30% |
| **WRITE NEW** | Implement from module spec, no POC code | Backend/infra/data modules without POC equivalent |

PORT, ADAPT, MIGRATE, COPY are **never valid**. High-quality POC code uses REFACTOR with "minimal adaptation" note.
