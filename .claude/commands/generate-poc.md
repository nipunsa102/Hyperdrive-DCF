---
description: Generate a navigatable POC with mock data and mock integrations
model: claude-opus-4-6
---

**Switches**: `-special`

**Switch Definitions**:
- `-special` → Special requirements or focus areas for the POC

## Purpose

Generates a complete Proof of Concept (POC) in a `/poc` folder. The POC has a fully navigatable UI with all screens, flows, and interactions — but ALL integrations and data are mocked. No real backends, no real databases, no real AI calls. The goal is to give stakeholders a feel of the final product so a product manager can walk them through the experience.

**What the POC IS:**
- A working, navigatable UI prototype
- All screens from the architecture rendered and interactive
- Mock data that looks realistic (hardcoded JSON, local files, or in-memory)
- Click-through flows that demonstrate user journeys end-to-end
- Status indicators, filters, sorts, and views that work with mock data

**What the POC is NOT:**
- A production system
- Connected to real APIs, databases, or AI services
- Covered by full test suites
- Deployed infrastructure

## Prerequisites

1. `PRD.md` must exist (run `/generate-prd` first)
2. `architecture/architecture.md` must exist (run `/generate-architecture` first)
3. `architecture/modules/` must be empty (or not exist). If ANY module `.md` file exists under `architecture/modules/`, ERROR and STOP:

   ```
   ERROR: POC path requires a pre-module architecture.
   architecture/modules/ contains {N} module files ({list}), which would conflict
   with POC promotion — /promote-poc bootstraps modules from the POC, and cannot
   reconcile pre-existing module specs.

   Choose one:
   (a) Delete architecture/modules/*.md and rerun /generate-poc. This is correct
       if the POC path is what you want (recommended for early-stage projects
       where stakeholder feedback will shape requirements).
   (b) Skip the POC path entirely and use Path A: /generate-modules → /generate-code.
       This is correct if requirements are stable and you are in execution mode.

   The full workflow rationale is in DCF.md: Architecture → POC → stakeholder
   feedback → sync-prd → promote-poc (bootstraps modules) → ready for production.
   POC path validates UX early and reduces module rework risk for greenfield work.
   ```

   This is a hard gate. There is no interactive override.

## Process

### Phase 1: Understanding

1. **Read PRD.md** (REQUIRED)
   - If `PRD.md` does not exist, ERROR: "Run /generate-prd first to create PRD.md"
   - Understand all user stories and acceptance criteria
   - Understand all feature areas and user journeys

2. **Read Supporting Documents**
   - Read `OVERVIEW.md` for full product context
   - Read `PRD.md` for project requirements
   - Read `architecture/architecture.md` if it exists — reuse screen layouts, and component structure
   - Read `TECHSTACK.md` if it exists — use the specified tech stack. if not **MUST create a techstack that is most suitable for the project based on the architecture and PRD**
   - Read `architecture/modules/` if they exist - When POC modules are created this module structure must be followed

3. **Determine UI Stack**
   - If `TECHSTACK.md` specifies a technology stack, use it.
   - If not specified, pick a stack idiomatic for the project's domain (e.g., for a web SPA: React + TypeScript + Vite + Tailwind CSS; for a desktop GUI: Tauri / Electron / native toolkit; for a CLI prototype: the project's primary language with a lightweight TUI library). Choose based on what the PRD describes, not a one-size-fits-all default.
   - The POC is UI-focused — no real backend server required

### Phase 2: POC Architecture Generation

Generate a lightweight architecture document at `poc/architecture/architecture.md`:

**MUST Include:**
- System overview (2-3 paragraphs — what this POC demonstrates)
- Screen inventory (every screen/view with its purpose)
- Navigation flow (how screens connect — what clicks go where)
- Mock data strategy (what data is mocked and where it lives)
- Component list (major UI components needed)

**MUST NOT Include:**
- Backend architecture, API specs, database schemas
- Integration Matrix, Module Registry
- Deployment or infrastructure details
- Security or authentication implementation details

**If `architecture/architecture.md` already exists:**
- Derive the POC architecture from it — reuse screen layouts, user journeys, and component descriptions
- Simplify: remove all backend components, API layers, and infrastructure
- Keep: all UI screens, navigation flows, and user-facing features

**If NO architecture exists yet:**
- Derive screens and flows directly from PRD.md user stories
- Group user stories into logical screens
- Define navigation between screens

### Phase 3: POC Module Extraction

Generate lightweight module specs in `poc/architecture/modules/`:

**Module Strategy:**
- One module per major screen or feature area (matching PRD feature areas)
- Modules are simpler than production modules — focused on UI and mock data only
- Each module spec file: `poc/architecture/modules/module-{N}-{name}.md`

**Module Spec Structure (simplified):**

```markdown
# POC Module {N}: {Name}

## Overview
What this module demonstrates in the POC.

## Screens
### {Screen Name}
- Layout description (what the user sees)
- Interactive elements (buttons, forms, filters, sorts)
- Mock data displayed
- Navigation targets (where clicks go)

## Mock Data
- What data this module needs
- Sample data structure (JSON)
- Where mock data lives (e.g., `src/mocks/{name}.{ext}`)

## User Flow
Step-by-step walkthrough of what the user does in this module.

## Acceptance Criteria (POC-level)
- [ ] Screen renders with mock data
- [ ] Navigation to/from this screen works
- [ ] Interactive elements respond (filters, sorts, status changes)
- [ ] Mock data looks realistic
```

### Phase 4: Implementation

**MUST INVOKE `coding-agent`** for each POC module with the following context:

```
POC MODE — IMPORTANT INSTRUCTIONS:
- Module ID: POC-M{N}
- Module Name: {Module Name}
- Development Spec: poc/architecture/modules/module-{N}-{name}.md
- Output Path: ALL source code goes under poc/src/ (NOT the main src/)
- Coverage Target: 0% (no unit test coverage required for POC)
- This is a POC — ALL data is mocked, ALL integrations are mocked
- No real API calls, no real database, no real authentication
- Mock data comes from local files or hardcoded constants in poc/src/mocks/
- Every screen must be navigatable and interactive with mock data
- Use the frontend stack from TECHSTACK.md; if TECHSTACK.md is silent, pick a stack idiomatic for the project's domain (e.g., React + TS + Vite + Tailwind for a web SPA — pick something else if the project isn't a web SPA)
```

**Implementation Rules for coding-agent:**
- ALL code goes under `poc/src/` — never touch the main `src/` folder
- Build/test configs live in `poc/` (e.g., `poc/package.json`, `poc/vite.config.ts`, `poc/tsconfig.json` — as appropriate for the chosen stack)
- Mock data files in `poc/src/mocks/`
- Components in `poc/src/components/` or `poc/src/app/`
- Shared types in `poc/src/types/`
- Use realistic-looking mock data (real-sounding names, dates, serial numbers)
- All navigation must work (client-side routing, e.g., React Router, Vue Router, or equivalent)
- All interactive elements must function with mock data (filters filter, sorts sort, status changes update UI)
- Forms should accept input and show confirmation (even if data goes nowhere)

**Module Implementation Order:**
1. Shared layout / navigation shell (header, sidebar, routing)
2. Feature modules in logical order (following PRD feature area order)

**FOR EACH POC module:**
1. Read the POC module spec
2. **INVOKE `coding-agent`** with POC mode context (as above)
3. Verify the module renders — **INVOKE `smoke-test-agent`** (frontend check only)
4. If smoke test fails, **INVOKE `coding-agent`** to fix (max 3 attempts)
5. Move to next module

### Phase 5: Final Smoke Test

After all modules are implemented:

1. **INVOKE `smoke-test-agent`** for the complete POC application
   - Frontend starts without errors
   - All routes render
   - No console errors on navigation

2. **If smoke test fails:**
   - **INVOKE `coding-agent`** to fix issues (max 3 attempts)
   - Re-run smoke test after each fix

3. **Final verification checklist:**
   - [ ] POC starts (e.g., `cd poc && npm install && npm run dev`)
   - [ ] Every screen from the POC architecture is accessible
   - [ ] Navigation between all screens works
   - [ ] Mock data displays correctly on all views
   - [ ] Interactive elements (filters, sorts, search, status changes) work
   - [ ] Forms accept input and provide feedback

## POC Folder Structure

```
poc/
├── dependencies & configs     # e.g., package.json, tsconfig.json, vite.config.ts
│                              # (as appropriate for chosen stack)
├── entry point                # e.g., index.html, main.py
├── architecture/
│   ├── architecture.md        # POC architecture (lightweight)
│   └── modules/
│       ├── module-1-{name}.md # POC module specs
│       ├── module-2-{name}.md
│       └── ...
└── src/
    ├── main.{ext}             # App entry point
    ├── App.{ext}              # Root component with routing (if applicable)
    ├── mocks/                 # All mock data
    │   ├── assets.{ext}       # Mock asset data
    │   ├── {domain-entities}.{ext} # Mock domain entity data
    │   └── ...
    ├── types/                 # Shared types/models
    │   └── index.{ext}
    ├── components/            # Shared UI components
    │   ├── layout/            # Header, sidebar, footer
    │   └── common/            # Buttons, cards, tables, etc.
    └── pages/                 # Screen/page components
        ├── {ScreenName}.{ext}
        └── ...
```

## Implementation Flow

```
START → Read PRD.md + OVERVIEW.md + existing architecture (if any)
          ↓
   Phase 2: Generate poc/architecture/architecture.md
          ↓
   Phase 3: Generate poc/architecture/modules/
          ↓
   Phase 4: FOR EACH POC module:
   ┌──────────────────────────────────────────────┐
   │                                               │
   │  1. Read POC module spec                      │
   │  2. coding-agent (POC mode — mock everything) │
   │  3. smoke-test-agent (frontend renders?)      │
   │     └─ Fix loop (max 3) if fails              │
   │                                               │
   └──────────────────────────────────────────────┘
          ↓
   Phase 5: Final smoke test (full app)
          ↓
        POC COMPLETE
```

## What Gets Mocked

| Real System Component | POC Mock Strategy |
|----------------------|-------------------|
| Authentication service | Hardcoded user object, no login screen (or simple fake login) |
| External API calls | Pre-generated mock responses in JSON |
| Database queries | In-memory arrays or JSON files in `poc/src/mocks/` |
| API endpoints | Direct imports from mock data files (no HTTP calls) |
| File uploads / storage | Mock file list, no real upload |
| Notifications | Toast/snackbar UI components (no real delivery) |

## Data Model Conformance

If `architecture/data-model.md` exists, read it before generating mocks. Mock data MUST conform to the entity shapes defined there:
- Mock files in `poc/src/mocks/` use the **same field names and types** as the data model
- Relationships between entities must be consistent with the data model's cardinality
- Enum values must match those defined in the data model
- This removes drift between POC mocks and the real schema — a huge quality win during promotion

If `architecture/data-model.md` does not exist, derive mock shapes from the architecture and PRD as before.

## Mock Data Guidelines

Mock data must look realistic to stakeholders:

- **Entities:** Use realistic-sounding names relevant to your domain
- **Dates:** Use dates relative to today (some past, some future, some overdue)
- **Categories:** Use realistic groupings that match your domain
- **Status variety:** Mix of different status values to demonstrate filtering and sorting

## Test Strategy (Minimal)

The POC uses a minimal test approach — just enough to verify the UI works:

| Gate | Scope | Blocking | When |
|------|-------|----------|------|
| Smoke Test | App starts, screens render | YES | After each module + final |

**No L1 unit tests. No L2 integration tests. No code review.**
The only gate is: does it start and can you click through it?

## CRITICAL CONSTRAINTS

- **ALL output in `poc/` folder** — never modify main `src/`, `architecture/`, or `tracking/`
- **UI-focused** — no real backend server, no database, no real API calls
- **Mock everything** — every data source is a local mock
- **Navigatable** — every screen must be reachable by clicking
- **Interactive** — filters, sorts, search, status changes must work with mock data
- **Realistic data** — mock data must look convincing to stakeholders
- **Self-contained** — install dependencies and start dev server from `poc/` must work (e.g., `cd poc && npm install && npm run dev`)

## Success Criteria

- [ ] POC starts (e.g., `cd poc && npm install && npm run dev`)
- [ ] All screens from PRD feature areas are present and navigatable
- [ ] Mock data displays realistically on all views
- [ ] List/table view supports sort, filter, and search with mock data
- [ ] Detail view shows full entity information
- [ ] Status tracking works in UI
- [ ] Notes/comments can be added (stored in-memory only)
- [ ] Primary workflow is walkable end-to-end (even with mock data)
- [ ] A product manager could demo this to stakeholders and they'd understand the product

## Agents Used

| Agent | Purpose in POC | Invocation |
|-------|---------------|------------|
| `coding-agent` | Implement each POC module (frontend + mocks) | Per module, with POC mode context |
| `smoke-test-agent` | Verify screens render and app starts | After each module + final |

**Agents NOT used in POC:**
- `unit-test-generator-agent` — No unit tests for POC
- `unit-tester-agent` — No L1 gate for POC
- `l2-integration-agent` — No integration tests for POC
- `traceability-validator-agent` — No traceability validation for POC
- `tracking-update-agent` — No tracking updates for POC
- `deploy-config-agent` — No deployment for POC
- `code-review-agent` — No code review for POC
