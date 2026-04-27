---
description: Generate Code - Module Implementation
model: claude-opus-4-6
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

Implements modules directly from architecture specifications. Reads module dependency order from the Integration Matrix in architecture.md and implements each module with mandatory L1/L2 test gates.

## Core Principles

1. **Modules are the ONLY work unit** - No capabilities, no iterations
2. **One Module at a Time** - Complete focus until done
3. **Test per Module** - Each module has its own test suite
4. **Dependencies Respected** - Never implement before dependencies
5. **L2 Validates Integration** - Cross-module validation after all modules complete

## Prerequisites

1. Architecture must exist (`/architecture/architecture.md`)
2. Module specifications must exist (`/architecture/modules/module-{N}-{name}.md`)
3. Integration Matrix must be available in architecture.md
4. **Normal mode only.** Retrofit mode (POC promotion) is handled by `/promote-poc`.

## Process

### Step 0: Read Architecture

1. **Read `/architecture/architecture.md`**
2. **Parse Integration Matrix** for module dependency order
3. **Get list of modules to implement:**
   - If `-module N` specified: Only module N
   - Otherwise: All modules in dependency order from Integration Matrix

**Path Configuration:**
- Default: `/architecture/`, `/tracking/`

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
- **MUST INVOKE `smoke-test-agent`** unless `-skip-smoke` flag is set
- Verifies the application actually runs after module implementation
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
2. INVOKE `coding-agent` with instructions to fix the dev-mock data or seed data for the failing routes (details from the smoke-test-agent's `functional_verification` section)
3. Re-INVOKE `smoke-test-agent` to verify the fix
4. Maximum 2 fix attempts. If still `FUNC_FAIL` after 2 attempts, log a warning and proceed (non-blocking but reported)

If the smoke-test-agent returns `PASS`: Proceed directly to tracking update.
If the smoke-test-agent returns `FAIL`: Mark module BLOCKED (existing behavior).

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

## Implementation Flow (SPEED-OPTIMIZED)

```
START → Read architecture.md → Parse Integration Matrix
           ↓
    Get Module Dependency Order
           ↓
    FOR EACH Module (in dependency order):
    ┌────────────────────────────────────────────────┐
    │                                                 │
    │  1. Read module spec                            │
    │  2. coding-agent (full module)                  │
    │  3. unit-test-generator-agent (MAX 5 tests)     │
    │  4. unit-tester-agent (L1 - Coverage Target)    │
    │     └─ Fix loop (max 5) if fails                │
    │  5. smoke-test-agent (BLOCKING)                 │
    │     ├─ PASS → proceed                           │
    │     ├─ FUNC_FAIL → fix dev-mock (max 2)         │
    │     └─ FAIL → BLOCKED                           │
    │  6. tracking-update-agent                       │
    │                                                 │
    └────────────────────────────────────────────────┘
           ↓
    All Modules L1 + Smoke Complete?
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
    SUCCESS → tracking-update-agent
           ↓
         IMPLEMENTATION COMPLETE
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
- Test reports
- Module integration verified per Integration Matrix

## Test Gate Summary

| Gate | Scope | Coverage/Tests | Blocking | When |
|------|-------|----------------|----------|------|
| L1 Unit Tests | Per module | 60% min (default), 5 tests max | YES | After each module |
| Smoke Test | Application | App runs | YES | After L1 (runnable modules) |
| Code Review | All modules | N/A | NO (optional) | After all L1 + smoke |
| L2 Integration | Cross-module | 10 tests max | YES | After all modules complete |

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
- **L1 Test Failure Threshold**: After 5 repair attempts, mark module as BLOCKED
- **L2 Test Failure Threshold**: After 5 fix attempts, mark as BLOCKED

## Important Notes

- **Module-Only**: No capabilities, no iterations - just modules
- **Direct from Architecture**: Reads module specs directly
- **Strict Gates**: Cannot skip or override test failures
- **L2 Handles Integration**: Cross-module validation at the end
- **Tracking**: Automatically updated via tracking-update-agent

## Success Criteria

### Per Module
- Implementation complete
- Unit tests passing with 60%+ coverage (default, configurable) (max 5 tests)
- Smoke test passes (app runs without crashing)
- Dev-mode pages display sample data (not empty states)
- Dev-mode API endpoints return structured responses with data
- Dependencies properly integrated
- Status updated in tracking

### Overall
- All modules implemented
- All L1 unit tests passing
- L2 integration tests passing
- Module integrations verified per Integration Matrix
- Application is navigatable in dev mode with sample data visible on all data-driven pages
- Complete documentation generated

## Core Requirements

- **MUST** read architecture.md and parse Integration Matrix first
- **MUST** implement modules in dependency order
- **MUST** implement exactly what's specified in the module development specification
- **MUST** respect module dependencies from Integration Matrix
- **MUST** ensure all Unit/Integration test Gates pass as defined above
- **MUST** invoke all specified agents in the correct sequence:

  **Per Module:**
  - `coding-agent` (full module) → `unit-test-generator-agent` → `unit-tester-agent` (L1)
  - `smoke-test-agent` (for modules with runnable code, unless `-skip-smoke`)
  - `tracking-update-agent`

  **After ALL Modules Complete:**
  - `code-review-agent` (only if `-review` flag) → `l2-integration-agent` → `tracking-update-agent`

- Use clean, readable code following standard best practices
- **MUST** invoke `tracking-update-agent` after each module and final completion
