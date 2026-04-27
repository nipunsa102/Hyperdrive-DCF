---
description: Generate architecture documentation following DCF methodology.
model: claude-opus-4-6
---

**Switches**: `-special`

**Switch Definitions**:
- `-special` — Special requirements for planning

## Purpose

Generates the main architecture documentation for the project: `architecture/architecture.md` (system-level design, components, screens, integration matrix, requirement coverage) and `architecture/data-model.md` (logical data model — entities, relationships, constraints). Derives everything from `PRD.md` so that every REQ-ID has architectural coverage.

## Prerequisites

1. `PRD.md` must exist at project root. If missing, ERROR: `"PRD.md not found. Run /generate-prd first."`

## Process

### Phase 1: Understanding

1. **Read `OVERVIEW.md` for project context**

2. **Read `PRD.md`** for project requirements**
   - If PRD.md does not exist, ERROR: "Run /generate-prd first to create PRD.md"
   - Extract all REQ-IDs and their descriptions
   - Understand requirement categories and priorities

3. **Read Supporting Documents**
   - Read `architecture.png` if available
   - Read `DESIGNGUIDE.md` for design requirements (UI/UX, database, code style, etc.) and follow any referenced external files within it (e.g., images, docs). Screen layouts, databas sesign, security requirements, component behaviours etc. in the architecture MUST reflect these design constraints.
   - Read any referenced images, .md or .docx files in OVERVIEW.md
   - Read `TECHSTACK.md` — if a tech stack is specified, use those technology choices to inform architectural decisions. If TECHSTACK.md is empty or only contains template placeholders, keep the architecture technology-agnostic.

### Phase 2: Architecture Generation (HIGH-LEVEL ONLY)

#### Step 1: Create architecture/architecture.md

Generate the main architecture file as a **conceptual overview** that anyone can understand:

**MUST Include:**
- System overview (2-3 paragraphs explaining what the system does)
- High-level component diagram (ASCII boxes showing major parts)
- **Screen layout diagrams** (for UI-heavy applications):
  - ASCII box diagrams showing major screen areas and their relationships
  - Header, footer, main content areas with percentage widths
  - Key UI elements (tiles, panels, tabs, cards) with annotations
  - Interactive element notes (click actions, navigation flows)
- **High-level flow diagrams** (for process-heavy applications):
  - User journey flows showing major steps
  - Decision points at a conceptual level
  - NOT sequence diagrams (those belong in module specs)
- User journeys (what users do, step by step, in plain language)
- Data concepts (what information exists, NOT how it's stored)
- How components connect (conceptual, NOT protocols/APIs)
- **"Implements:" tags** on every component (see below)

**MUST NOT Include:**
- Module Registry table (created by /generate-modules)
- Integration Matrix (created by /generate-modules)
- Individual module specifications
- API endpoint specifications
- File format specifications
- Technical protocols (REST, JSON, HTTP)
- Pseudo-code or function signatures
- Component prop definitions
- Implementation details

**Requirement Traceability - "Implements:" Tags:**

Every component section MUST include an `**Implements:**` tag listing the REQ-IDs it addresses:

```markdown
## System Components

### Authentication System
**Implements:** REQ-1

The authentication system handles user identity...

### Dashboard
**Implements:** REQ-2, REQ-3

The dashboard provides the main workspace...

### Task Management
**Implements:** REQ-4, REQ-4.1, REQ-4.2

Tasks are the core work items...

### Export System
**Implements:** REQ-5

Export functionality allows users to...
```

**Rules for "Implements:" Tags:**
- List REQ-IDs from PRD.md that this component addresses
- A REQ-ID can appear in multiple components if shared
- Use top-level REQ-IDs (REQ-1) for broad coverage
- Use sub-requirements (REQ-1.1) for specific functionality
- Every REQ-ID in PRD.md must appear in at least one component

**Requirement Coverage Matrix:**

After generating all component sections, include a coverage summary matrix:

```markdown
## Requirement Coverage Matrix

| REQ-ID | Requirement | Component(s) |
|--------|-------------|--------------|
| REQ-1 | User Authentication | Authentication System |
| REQ-1.1 | Email/Password Login | Authentication System |
| REQ-1.2 | Session Management | Authentication System |
| REQ-2 | Project Management | Dashboard, Project Manager |
| REQ-2.1 | Project Creation | Project Manager |
| ... | ... | ... |
```

**Matrix Rules:**
- List EVERY REQ-ID from PRD.md (including sub-requirements)
- Show which component(s) implement each
- Empty "Component(s)" column = ORPHAN requirement (must fix)
- This matrix enables quick validation that no requirement is missed

**Data Model:**

The architecture MUST include a **"Data Model"** section in `architecture/architecture.md` that gives a one-paragraph conceptual summary of the system's data and links to `architecture/data-model.md`.

Additionally, produce `architecture/data-model.md` alongside `architecture/architecture.md`. This file contains the **logical data model** at architecture depth:

- Logical entities with fields, types, nullability, enums
- Relationships (1:1, 1:N, N:M) with cardinality and FK direction
- Indexes and uniqueness constraints
- Derived/computed fields and their source rule
- Seed / reference data
- REQ-ID back-references on every entity (traceability)
- Notes on storage technology **intent** (e.g., "relational, transactional") but NOT specific vendor choice — vendor is decided later (e.g., during `/prepare-poc-promo` or by TECHSTACK.md)

Use logical-model depth (types, relationships, constraints, enums) but defer vendor-specific choices (no SQL, no Prisma schema, no migration files).

**Example of correct abstraction (in architecture.md Data Concepts section):**
```
## Data Concepts

The system manages:
- Projects: Named workspaces containing tasks, ideas, notes, and files
- Tasks: To-do items with status, priority, and due dates
- Ideas: Visual brainstorming boards (drawings and diagrams)
- Notes: Rich text documentation
- Artifacts: External files (PDFs, documents, images)

See [Data Model](data-model.md) for the full logical schema.
```

NOT:
```
CREATE TABLE tasks (
  id TEXT PRIMARY KEY,
  status TEXT NOT NULL...
)
```

### Visual Documentation Guidelines

**Screen Layouts (UI Applications):**

When the application has significant UI, include ASCII diagrams for each major screen:

```
### [Screen Name] Layout
**Implements:** REQ-X.X, REQ-Y.Y

+------------------------------------------------------------------+
|                           HEADER                                  |
|  [Logo/Home]                                    [Settings] [User] |
+------------------------------------------------------------------+
|                                                                   |
|  +-------------------+  +--------------------------------------+  |
|  |   LEFT PANEL      |  |          MAIN CONTENT               |  |
|  |   (35% width)     |  |          (65% width)                |  |
|  |                   |  |                                      |  |
|  | Key elements:     |  |  Key elements:                      |  |
|  | - List items      |  |  - Cards/Tiles                      |  |
|  | - Navigation      |  |  - Data display                     |  |
|  +-------------------+  +--------------------------------------+  |
|                                                                   |
+------------------------------------------------------------------+
|                           FOOTER                                  |
+------------------------------------------------------------------+

**Key Elements:**
- Description of each major UI region
- Interactive behaviors (click, drag, etc.)
```

**Include screen layouts when:**
- Application has 2+ distinct screens/views
- UI layout is critical to understanding the system
- Requirements specify specific UI arrangements

**High-Level Flow Diagrams:**

For systems with significant workflows, include conceptual flow diagrams:

```
### [Process Name] Flow
**Implements:** REQ-X.X

User Action → System Response → Next State
     │              │              │
     │              ▼              │
     │         Decision?           │
     │          /    \             │
     │        Yes    No            │
     │         ▼      ▼            │
     └──── Path A   Path B ────────┘
```

**Include flow diagrams when:**
- Multi-step user journeys exist
- System has decision-based routing
- Understanding the "happy path" is essential

**DO NOT include in architecture.md:**
- Sequence diagrams (too detailed - belongs in modules)
- Vendor-specific database schemas or SQL (belongs in modules; logical data model goes in `architecture/data-model.md`)
- API specifications (belongs in modules)
- Pseudo-code (belongs in modules)

### Phase 3: Traceability Validation

**Invoke traceability-validator-agent** to verify:
- All REQ-IDs in PRD.md are referenced by at least one component's "Implements:" tag
- No invalid REQ-IDs (references to IDs not in PRD.md)
- Component descriptions align with their assigned requirements

**Fix any gaps identified:**
- Add missing "Implements:" tags
- Add missing REQ-IDs to appropriate components
- Ensure 100% requirement coverage before completion

## Output Structure
```
architecture/
├── architecture.md          # High-level conceptual overview with Implements: tags
└── data-model.md            # Logical data model (entities, relationships, constraints)
```

**Note:** Module specifications are created separately by `/generate-modules` command.

## CRITICAL CONSTRAINTS

**Abstraction Levels:**
- **PRD.md** = "What must the system do?" (requirements with REQ-IDs)
- **architecture.md** = "What is the system?" (conceptual, any stakeholder can read)
- **Module specs** = Created later by /generate-modules

**Traceability Rules:**
- **Every component** must have an "Implements:" tag
- **Every REQ-ID** in PRD.md must be covered by at least one component
- **No invented requirements** - only reference REQ-IDs from PRD.md

**Content Rules:**
- NO module extraction (done by /generate-modules)
- NO Integration Matrix (done by /generate-modules)
- NO pseudo-code or technical specs
- YES high-level component descriptions
- YES user perspective
- YES plain language

**Quality Rules:**
- KEEP IT SIMPLE - Minimum viable architecture
- COMPLETE TRACEABILITY - All REQ-IDs mapped to components
- NO GAPS - traceability-validator-agent must return PASS

## Quality Checklist
Before completion:
1. Can a non-developer understand architecture.md?
2. Does every component have an "Implements:" tag?
3. Is every REQ-ID from PRD.md covered by at least one component?
4. Are component descriptions in plain English?
5. Does traceability-validator-agent return PASS status?
6. Is there NO Module Registry or Integration Matrix (saved for /generate-modules)?
7. For UI-heavy applications: Are screen layout diagrams included?
8. For process-heavy applications: Are high-level flow diagrams included?
9. Do all diagrams have "Implements:" tags linking to requirements?
10. Does `architecture/data-model.md` exist with logical entities, relationships, and REQ-ID traceability?
11. Does `architecture/architecture.md` contain a "Data Model" section linking to `data-model.md`?

## Next Step

After this command completes, run `/generate-modules` to:
- Extract modules from architecture components
- Create module specifications with Requirement Coverage tables
- Generate Module Registry and Integration Matrix
