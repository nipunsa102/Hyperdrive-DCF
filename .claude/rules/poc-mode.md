---
paths:
  - "poc/**/*"
---

# POC Mode Rules

## Detection
POC mode is active when the invoking command passes:
- `Module ID: POC-M{N}` (module IDs prefixed with `POC-`)
- `Coverage Target: 0%`

## Implementation Rules

1. **Mock everything:** No real APIs, databases, or auth. All data comes from mock files
2. **Mock the layers you're not proving:** Build the layer under validation; mock everything else.
   - Frontend-focused POC: No real backend server. Direct imports from mock data files.
   - Backend-focused POC: No real frontend. Use mock API endpoints and test harnesses.
   - Full-stack POC: Minimize external integrations. Mock third-party services and auth providers.
3. **Navigation:** Every screen must be reachable by clicking through the UI
4. **No real integrations:** Dependencies are mocked, not connected to live services
5. **Standard quality rules still apply:** No scope creep, KISS, clean code

## Real-Mode Override (`-special`)

A POC generated with a `-special` directive MAY be a real system. The authority is the deviation table in `poc/architecture/architecture.md`: where it declares a real integration (database, auth, storage, external service, deployment), that layer is REAL — the mock rules above apply only to layers it does not cover. Everything else stands: KISS, clean code, navigable screens, smoke gate. Agents invoked with `Module ID: POC-M{N}` must check the POC architecture's deviation table before assuming mocks.

## Workflow Overrides
In POC mode, the standard implementation workflow is modified:

- **Skip integration sanity check:** No real dependencies to verify
- **Skip test readiness:** No unit tests or test fixtures required
- **Runtime gate:** Smoke test (app starts and screens render). The orchestrating command also runs the architecture-alignment gate (docs vs code) — see .claude/rules/architecture-doc-standard.md.

For output locations and folder structure, see `project-structure.md` (poc/ section).
