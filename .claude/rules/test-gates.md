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
The default coverage target is 60%. Invoking commands (e.g., /generate-code, /generate-poc)
pass the actual coverage target to agents. Agents use the value provided by the invoking command.

## Smoke Test Gate (BLOCKING)
- Application must start without crashing
- Basic functionality must respond with meaningful content:
  - Pages return rendered HTML with visible UI content (not just the mount point)
  - API endpoints return properly structured responses
  - In dev mode, data-driven pages display sample data (not empty/error states)
  - The smoke-test-agent performs both structural checks (build, routes) and functional checks (content, data)
- BLOCKING - module cannot proceed if smoke fails
- **Apply smoke test** to modules that produce runnable code (backend servers, frontend apps)
- **Skip smoke test** for foundational modules (data models, utilities, shared libraries) with no runnable entry points
- Determine classification by reading the module spec and checking for runnable endpoints or rendered UI

> **Foundational modules** (data types, schemas, utilities with no UI/API routes) skip functional checks. Only structural checks (build, startup) apply.

> **Functional check failures (`FUNC_FAIL`) are non-blocking:** If the app starts and routes respond (structural pass) but pages render empty or API data is missing (functional fail), the smoke-test-agent returns `FUNC_FAIL`. The generate-code command treats this as a data gap and invokes coding-agent to fix dev-mock data (max 2 attempts) before proceeding. Only structural failures (`FAIL` — app won't start, routes crash) are blocking.

## L2 Gate (Integration Tests)
- Cross-module validation for critical flows
- **Maximum 10 integration tests** total
- Tests must be in `integration.test.{ext}` format
- Run after all modules pass L1 + Smoke

## Code Review (OPTIONAL)
- Only runs with `-review` flag
- Skipped by default for speed
- Use when quality is more important than speed

## Enforcement
- After implementing code, ALWAYS run the relevant test suite
- Report coverage percentage
- If below 60% (default), add more tests before marking module complete
- Use `/update-tracking` to record test status

## Gate Summary

| Gate | Coverage/Tests | Blocking | When |
|------|----------------|----------|------|
| L1 Unit Tests | 60% min (default), 5 max | YES | After each module |
| Smoke Test | App runs | YES | After L1 (runnable modules) |
| Code Review | N/A | NO | Optional (-review) |
| L2 Integration | 10 max | YES | After all modules |
