# POC Promotion Algorithm

Concise logic reference for `/promote-poc`. For full details, see [`.claude/commands/promote-poc.md`](.claude/commands/promote-poc.md).
Part of the [DCF workflow](DCF.md).

---

## Pipeline at a Glance

```
Phase 1 ─ Preflight ───────────────────────── inline (prereqs, maturity, resume)
Phase 2 ─ Architecture Merge & Validate ──── re-architect-agent + validators (combined)
Phase 3 ─ Code Promotion Plan ────────────── code-promotion-analyzer-agent + tracking init
Phase 4 ─ Implement & Test ──────────────── per-module execution + L2 + cross-cutting sweep
Phase 5 ─ Report & Finalize ──────────────── CONFIG_GUIDE.md + POC_PROMOTION_REPORT.md + close log
```

After `/promote-poc` completes, the human follows `CONFIG_GUIDE.md` to populate `.env`, then runs `/setup-env` to create the DB schema, seed reference data, and verify external service connectivity. `/setup-env` is a separate command — see `.claude/commands/setup-env.md`.

**Gated workflow assumption:**
`/generate-poc` hard-fails if `architecture/modules/` is non-empty. Therefore at promotion time, modules are always bootstrapped from POC — there is no existing module reconciliation scenario, and there is no pre-existing `src/` code to preserve.

**Authority chain (never violated):**
```
PRD.md (read-only) > architecture.md (modified) > modules/*.md (created)
```

---

## Phase 1: Preflight

```
PRD.md exists? ──── NO ──→ ERROR
       │ YES
poc/temp/poc_promotion/POC_PROMO_PREP.md exists? ── NO ──→ ERROR: run /prepare-poc-promo
       │ YES
Scan body for <!-- REQUIRED markers
  ├── Found → ERROR: list unfilled items
  └── None  → all decisions complete
       │
poc/architecture/ exists? ── NO ──→ ERROR
       │ YES
architecture/architecture.md + data-model.md exist? ── NO ──→ ERROR
       │ YES
architecture/modules/ empty?
  ├── NO  → ERROR: "/generate-poc gate should have prevented this. Remove
  │                 architecture/modules/*.md and rerun, or abandon POC path"
  └── YES → continue                                     ← gated workflow contract
       │
poc/poc-tracking/CHANGELOG.md has unsynced CL-entries?
  ├── YES (CL-IDs missing from PRD) → ERROR: run /sync-prd first
  └── NO  → continue
       │
Detect POC scope → frontend-only | backend-only | full-stack | undetermined
       │
Classify POC maturity:                                    ← NEW
  Scan poc/src/ for mock patterns, package.json for real-service SDKs,
  poc/.env.local for real credentials.
  → mock-heavy | hybrid | production-wired
  Report to user with 1-sentence evidence.
       │
poc/temp/poc_promotion/promotion.md exists?
  ├── YES → Resume: read completed phases, skip ahead
  └── NO  → Fresh run
       │
Create or update promotion.md (5-phase checklist + metadata + log)
       │
       ▼
Phase 2
```

**POC scope drives merge strategy.** POC maturity drives decision distribution priors passed to the analyzer in Phase 3.

---

## Phase 2: Architecture Merge & Validate

No backup needed — `/promote-poc` runs on a separate git branch. Main branch is the backup.

```
┌─────────────────────────────────────────────────────────────────┐
│  2a. re-architect-agent                                         │
│      • Merge POC architecture into main architecture.md          │
│      • Merge POC data patterns into data-model.md                │
│      • Bootstrap architecture/modules/*.md from POC module designs│
│      • Preserve main's technical depth for layers POC doesn't cover│
│      • Create/update Module Registry + Integration Matrix        │
│                                                                 │
│  2b. POC Incorporation Check (inline)                           │
│      For each POC screen: verify present in merged architecture  │
│      For each POC acceptance criterion: verify survived merge    │
│      For each CL-entry: verify change reflected                  │
│      Auto-fix flags; restore POC specificity                     │
│                                                                 │
│  2c. Validation loop (max 5 iterations, all run together):      │
│      • traceability-validator-agent (REQ coverage, refs)         │
│      • Cycle detection (DFS on Integration Matrix — BLOCKING)    │
│      • Sum test (modules exactly match architecture — BLOCKING)  │
│      • coherence-checker-agent (semantic consistency)            │
│                                                                 │
│      ALL PASS → break                                           │
│      Convergence: issue count not decreasing for 2 iters → break │
│      Auto-fix ORPHAN/DUPLICATE/INVENTED errors between iters     │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                    PASS / PARTIAL_PASS
                           │
                   FAIL → user choice: PROCEED with warnings or STOP
                           │
                           ▼
                        Phase 3
```

**Bootstrap module logic (always applied under gated workflow):**

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

---

## Phase 3: Code Promotion Plan

```
┌─────────────────────────────────────────────────────────────┐
│  3a. code-promotion-analyzer-agent                          │
│      Reads:                                                 │
│      • architecture/ (authoritative)                         │
│      • poc/architecture/ (supplementary)                     │
│      • poc/src/ (analysis target)                            │
│      • POC_PROMO_PREP.md (human decisions)                   │
│      Receives: POC maturity classification as prior          │
│                                                             │
│      Step 1: Inventory POC code                              │
│      Step 2: Map POC files → production modules              │
│      Step 3: Gap analysis per module                         │
│              poc_coverage = SATISFIED / total criteria       │
│      Step 4: Technical compatibility                         │
│              + mock classification: none | clean | embedded  │
│      Step 5: Surgical edit enumeration for AS_IS candidates  │
│      Step 6: Decision per module:                            │
│                                                             │
│        AS_IS ──── poc_coverage ≥ 0.80 AND                    │
│                   framework identical AND                    │
│                   (mock_strategy = none OR clean) AND        │
│                   gaps ≤ 5 surgical edits                    │
│                                                             │
│        REWRITE ── poc_coverage < 0.30 OR                     │
│                   mock_strategy = embedded OR                │
│                   framework mismatch OR low quality          │
│                                                             │
│        ADAPT ──── everything else (default)                  │
│                                                             │
│      Step 7: Consistency check vs. maturity prior            │
│              (production-wired → expect AS_IS-skew; etc.)    │
│      Returns: structured analysis (decisions, edits, gaps)   │
│                                                             │
│  3b. Initialize tracking/module-tracking.md                 │
│      All modules → Not Started, note decision in Notes col   │
│      On resume: read existing, skip completed                │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
                    Phase 4
```

---

## Phase 4: Implement & Test

```
Read Integration Matrix → topological sort → dependency order
       │
FOR EACH module (in dependency order):
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Decision = AS_IS?                                              │
│  ├── YES → /promote-poc copies POC files to target paths        │
│  │         Applies surgical edits from analyzer output           │
│  │         NO coding-agent invocation                            │
│  │         unit-test-generator-agent (max 5 tests)                │
│  │         unit-tester-agent (L1 gate)                            │
│  │         ├── PASS → smoke-test-agent                           │
│  │         └── FAIL → BLOCKED (decision was wrong;                │
│  │                    DO NOT enter repair loop — would            │
│  │                    contradict AS_IS. Escalate to user.)        │
│  │                                                              │
│  └── NO (ADAPT or REWRITE)                                      │
│           │                                                     │
│           ├── coding-agent (module spec + Retrofit: ADAPT/REWRITE)│
│           ├── unit-test-generator-agent (max 5 tests)            │
│           ├── unit-tester-agent (L1 gate, 60% coverage)           │
│           │     └── L1 repair loop (max 5 attempts)               │
│           └── smoke-test-agent (blocking)                         │
│                 └── FUNC_FAIL → fix dev-mock (max 2)               │
│                                                                 │
│  tracking-update-agent → update module status                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
       │
After all modules:
       │
┌─────────────────────────────────────────────────────────────────┐
│  Cross-Cutting Decision Sweep                                   │
│  Parse POC_PROMO_PREP.md decisions; classify each:              │
│    File-scoped (e.g., INT-2 across all API routes)               │
│    String-scoped (e.g., AUTH-2 across all .env.local references) │
│    Config-scoped (e.g., INT-1 AI_MODEL env var)                  │
│    Deployment-scoped (reported only, no code change)              │
│  For each code-impacting decision:                               │
│    Scan src/ for violations (grep/glob)                          │
│    Apply edit (Edit tool, file-by-file)                          │
│    Log: "[decision-id] Applied to N files: [list]"                │
│  Re-run affected L1 tests; max 3 fix attempts on regression       │
└──────────────────────┬──────────────────────────────────────────┘
                       │
l2-integration-agent (self-contained fix loop, max 5)
       │
tracking-update-agent → final status
       │
       ▼
Phase 5
```

---

## Phase 5: Report & Finalize

Two output files at project root — separation of concerns: one for config-as-instructions, one for promotion-as-history.

```
5a. Write CONFIG_GUIDE.md at project root  (production configuration walkthrough)
     ├── Quick Start (cp .env.example .env, fill in values, /setup-env)
     ├── Env var at-a-glance table (name → source dashboard → section)
     ├── §1..§N — one section per external service from POC_PROMO_PREP.md
     │     step-by-step provider dashboard flow, env var values, verification
     │     (mirrors poc/docs/config-guide.md style, adapted for prod tenants)
     ├── §N+1 — deployment-host configuration (per DEP-2/3/4)
     ├── After-editing-.env instructions (run /setup-env)
     └── Troubleshooting (per-service failure modes)

5b. Write POC_PROMOTION_REPORT.md at project root  (promotion metadata + history)
     ├── Pointer to CONFIG_GUIDE.md as next step
     ├── Promotion Summary (POC maturity, decision distribution, L2 status)
     ├── Per-Module Results (decision, L1 coverage, smoke, status)
     ├── Cross-Cutting Decision Sweep Summary (decision → files touched → result)
     ├── Human Decisions Applied (DP/AUTH/SEC/INT/DEP → where applied)
     ├── Blocked Modules (if any)
     ├── Generated Files (grouped by category)
     ├── Known Limitations (manual steps remaining)
     └── Next Steps (fill .env, run /setup-env, walk through primary flow)

5c. Finalize promotion.md → mark all phases [x], final summary
5d. Print summary to user → points at CONFIG_GUIDE.md + /setup-env
```

**Key design: CONFIG_GUIDE.md and POC_PROMOTION_REPORT.md are not duplicates.**
- `CONFIG_GUIDE.md` = how to configure (walkthrough, provider-by-provider, step-by-step)
- `POC_PROMOTION_REPORT.md` = what happened during promotion (metadata, decisions, outcomes)

`CONFIG_GUIDE.md` is the authoritative input for `/setup-env`.

---

## Validation Layer Map

Each layer catches a different class of error at a different trust boundary:

```
Phase 2 re-architect self-check ── "Did the merge agent do what it intended?"
         │
Phase 2 POC incorporation ────────── "Did POC features survive the merge?"
         │                            (only step that reads POC + merged docs)
Phase 2 structural validation ────── "Are REQ-IDs, DAG, and feature sets correct?"
         │                            (independent external validator)
Phase 2 semantic validation ──────── "Does the content actually make sense together?"
         │                            (POC-blind — validates docs on own merit)
Phase 3 code-level analysis ──────── "Is the POC code compatible with the design?"
         │                            (practical migration feasibility)
Phase 4 cross-cutting sweep ──────── "Did human decisions apply uniformly?"
                                      (catches per-module miss patterns)
```

---

## Safety Mechanisms

| Mechanism | Phase | Risk Mitigated |
|-----------|-------|----------------|
| Git branch isolation | All | Main branch preserved as backup |
| `/generate-poc` hard gate | (upstream) | Promoting into non-empty `architecture/modules/` |
| Prep completeness check | 1 | Running promotion with missing decisions |
| Empty-modules safety net | 1 | Promoting into non-empty modules (upstream gate failed) |
| Resume detection | 1 | Session failure loses progress |
| Promotion plan log | 1–5 | Audit trail + resume context |
| Sync guard (CL-entries) | 1 | Unsynced POC changes silently lost |
| POC maturity prior | 1 → 3 | Analyzer decisions inconsistent with POC reality |
| Hard stop on merge failure | 2 | Validating broken output |
| Content-level POC check | 2 | Silent feature drops in merge |
| Changelog cross-reference | 2 | POC changes not in PRD annotations |
| Convergence detection | 2 | Auto-fix ping-pong loops |
| Fresh-bootstrap awareness | 2 | False MAJOR flags on new modules |
| User choice on FAIL | 2 | Unrecoverable coherence issues |
| AS_IS no-repair | 4 | Silently overriding a wrong AS_IS decision |
| Tracking-based resume | 4 | Re-implementing already-completed modules |
| Cross-cutting sweep | 4 | Human decisions applied inconsistently across files |
| PRD + POC read-only | All | Source of truth never modified |

---

## Decision Values (canonical)

Only three valid decisions exist across the entire pipeline:

| Decision | Meaning | When |
|----------|---------|------|
| **AS_IS** | Copy POC files verbatim + apply surgical edits; no `coding-agent` | `poc_coverage ≥ 0.80`, framework identical, mock_strategy ∈ {none, clean}, ≤ 5 surgical edits per module |
| **ADAPT** | `coding-agent` copies POC as starting point, replaces mocks, fills production gaps | `poc_coverage ≥ 0.30`, mock_strategy = clean, framework compatible (default for most POCs) |
| **REWRITE** | `coding-agent` writes fresh from module spec; POC is reference only | `poc_coverage < 0.30` OR mock_strategy = embedded OR framework mismatch OR low quality |

Coverage percentages are **counted**, not eyeballed: `SATISFIED_criteria / total_criteria` from the module's `## Acceptance Criteria` list. `PARTIAL` (happy-path only) does not count as satisfied.

**Retired:** `WRITE NEW` (no POC code) and `KEEP EXISTING` (preserve production code) are not used — both are unreachable under the gated workflow where `architecture/modules/` and `src/` are empty at promotion time.

**POC maturity prior:** The analyzer receives a classification (`mock-heavy` / `hybrid` / `production-wired`) from Phase 1. A `production-wired` POC is expected to skew toward AS_IS; a `mock-heavy` POC toward ADAPT/REWRITE. Distributions that conflict with the prior require explicit justification.
