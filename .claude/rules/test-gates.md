---
paths:
  - "src/**/*"
  - "tests/**/*"
---

# Test Gate Enforcement

All code changes must pass test gates before being considered complete.

## L1 Gate (Unit Tests)
- **Minimum 60% coverage** per module (default, configurable)
- **Maximum 5 tests** per module
- Run with the project's test runner (e.g., `npx vitest`, `pytest`, `go test`)
- Tests must be in `{name}.test.{ext}` format (e.g., `.ts`, `.py`, `.go`)
- CANNOT proceed to Smoke Test without passing L1

## Coverage Parameterization
The default coverage target is 60%. The invoking context passes the actual coverage target to the work it orchestrates; consumers of this rule use the value provided.

## Smoke Test Gate (BLOCKING)
- Application must start without crashing
- Basic functionality must respond with meaningful content:
  - Pages return rendered HTML with visible UI content (not just the mount point)
  - API endpoints return properly structured responses
  - In dev mode, data-driven pages display sample data (not empty/error states)
  - The smoke test step performs both structural checks (build, routes) and functional checks (content, data)
- BLOCKING - module cannot proceed if smoke fails
- **Apply smoke test** to modules that produce runnable code (backend servers, frontend apps)
- **Skip smoke test** for foundational modules (data models, utilities, shared libraries) with no runnable entry points
- Determine classification by reading the module spec and checking for runnable endpoints or rendered UI

> **Foundational modules** (data types, schemas, utilities with no UI/API routes) skip functional checks. Only structural checks (build, startup) apply.

> **Functional check failures (`FUNC_FAIL`) are non-blocking:** If the app starts and routes respond (structural pass) but pages render empty or API data is missing (functional fail), the smoke test returns `FUNC_FAIL`. The invoking context should treat this as a data gap and trigger a targeted fix of the sample data (dev-mock data on the POC path; seed/reference data in the bootstrapped dev environment on the direct path — max 2 attempts) before proceeding. Only structural failures (`FAIL` — app won't start, routes crash) are blocking.

## L2 Gate (Integration Tests)
- Cross-module validation for critical flows
- **Maximum 10 integration tests** total
- Tests must be in `integration.test.{ext}` format
- Run after all modules pass L1 + Smoke

## E2E Functional Gate (BLOCKING — `/generate-code`, direct path)
- Drives the app the way a user would, against the **running dev environment** declared in `DEPLOYMENT.md` (fully local / cloud / hybrid)
- **Headless-browser testing is MANDATORY for systems with a UI** — HTTP-level checks are not a substitute; non-UI systems run equivalent journeys through their real interface (API/CLI)
- **Maximum 10 journey tests**, derived from architecture user journeys + module acceptance criteria, persisted under `tests/e2e/`
- Per-module: UI modules also get a lightweight headless render + primary-interaction check (≤2 checks per screen) after smoke
- Failures feed the invoking command's fix loop (`coding-agent` fixes, re-run) — the gate blocks completion until it passes
- POC path is unaffected (POC has only the smoke gate; promoted code is verified by `/setup-env`'s runtime smoke)

## Code Review (OPTIONAL)
- Only runs with `-review` flag
- Skipped by default for speed
- Use when quality is more important than speed

## Enforcement
- After implementing code, ALWAYS run the relevant test suite
- Report coverage percentage
- If below 60% (default), add more tests before marking module complete
- Record test status in the project's tracking log

## Gate Summary

| Gate | Coverage/Tests | Blocking | When |
|------|----------------|----------|------|
| L1 Unit Tests | 60% min (default), 5 max | YES | After each module |
| Smoke Test | App runs | YES | After L1 (runnable modules) |
| Module E2E (headless browser, UI modules — direct path) | ≤2 checks per screen | YES | After smoke |
| Code Review | N/A | NO | Optional (-review) |
| L2 Integration | 10 max | YES | After all modules |
| E2E Functional (headless browser for UI — direct path) | 10 journeys max | YES | After L2 |

## Test Artifact Paths

Tests and test-generated reports live at predictable paths so that downstream commands and humans can locate them without inference:

| Artifact | Path | Produced by |
|----------|------|-------------|
| L1 unit test source files | `tests/unit/{module_name}/{name}.test.{ext}` | L1 test generation step |
| L2 integration test source file | `tests/integration/integration.test.{ext}` | L2 integration test step |
| E2E journey tests (blocking gate in `/generate-code`, direct path) | `tests/e2e/**` | `e2e-test-agent` (full scope); humans may add more |
| E2E run reports (screenshots, failure diagnostics) | `tests/reports/e2e/` | `e2e-test-agent` |
| Test run reports (coverage, pass/fail JSON) | `tests/reports/unit/{module_name}/`, `tests/reports/integration/` | Test runner / test gate step |

The `tests/reports/` subtree is ephemeral and should be gitignored. The `tests/unit/` and `tests/integration/` subtrees are part of the committed codebase.
