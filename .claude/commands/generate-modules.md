---
description: Extract modules from architecture with requirement traceability
model: claude-opus-4-6
---

## Purpose

Extract modules from architecture.md components and create module specifications with full requirement traceability. Each module includes a Requirement Coverage table mapping REQ-IDs to implementations.

## Prerequisites

- `PRD.md` must exist (run `/generate-prd` first)
- `architecture/architecture.md` must exist (run `/generate-architecture` first)
- Architecture components must have "Implements:" tags

## Process

### Phase 1: Understanding

1. **Read PRD.md**
   - Extract all REQ-IDs and their descriptions
   - Build a checklist of requirements to trace

2. **Read architecture/architecture.md**
   - Identify all components and their "Implements:" tags
   - Understand component relationships and data flows
   - Note which REQ-IDs each component claims to implement

3. **Check for module-guide.md**
   - If exists, follow grouping instructions
   - If not, determine logical module boundaries based on architecture

### Phase 2: Module Extraction

For each module, create `architecture/modules/module-{N}-{name}.md`:

**Naming Rules:**
- Format: `module-{N}-{name}.md`
- N = sequential number starting at 1
- name = lowercase-kebab-case descriptive name
- Order modules by implementation dependency (foundations first)

**Module Structure (MANDATORY sections in order):**

```markdown
# Module {N}: {Name}

## Requirement Coverage

| REQ ID | Requirement | Implementation |
|--------|-------------|----------------|
| REQ-1.1 | Email/Password Login | AuthService.login() |
| REQ-1.3 | Session Management | SessionManager class |

## Overview
2-3 sentences: What is this module? Why does it exist?

## User Stories
- As a user, I can...
- As a user, I can...

## What Users See
### [Screen/View Name]
Description of UI elements and interactions...

## Key Concepts
- Concept 1: explanation
- Concept 2: explanation

## Data Needs
What information this module stores/displays (conceptual)

## Interactions
- Depends on: Module X, Module Y
- Used by: Module Z

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2
```

**Section-Level Requirement Mapping:**

Every descriptive section (User Stories, What Users See, Key Concepts, etc.) should include inline `Implements:` tags where applicable:

```markdown
## What Users See

### Login Screen
**Implements:** REQ-1.1

A centered card with email and password fields...

### Dashboard View
**Implements:** REQ-2.1, REQ-2.3

The main workspace showing project tiles...
```

**Mapping Rules:**
- Each sub-section (###) should have `**Implements:**` if it directly addresses a requirement
- User Stories can reference requirements inline: "As a user, I can log in (REQ-1.1)"
- Acceptance Criteria should map to requirements: "- [ ] Can log in (REQ-1.1)"

**Requirement Coverage Table Rules:**
- MUST be the FIRST section after the module title
- List every REQ-ID this module implements
- "Requirement" column = brief description from PRD.md
- "Implementation" column = how it will be implemented (class, function, component)
- A REQ-ID can appear in multiple modules if shared
- Every REQ-ID in the component's "Implements:" tag must appear here

**Coherence Rules (MANDATORY):**

1. **1:1 Component-to-Module Mapping**
   - Each architectural component belongs to EXACTLY one module
   - No component split across multiple modules
   - No module combines unrelated architectural components

2. **Feature Containment**
   - All features of a component stay in the same module
   - Cross-cutting concerns (logging, auth checks) handled via Integration Matrix, not duplication

3. **Coherence Validation Checklist:**
   - [ ] Every architecture.md component appears in exactly one module
   - [ ] No module references components from unrelated architectural areas
   - [ ] Related features grouped together (high cohesion)
   - [ ] Module boundaries align with architectural boundaries

**Technical Detail Sections (include when applicable):**

| Detail Type | Include When | Skip When | Example |
|-------------|--------------|-----------|---------|
| **Sequence Diagram** | Multi-step async flows, complex interactions between components | Simple request-response, CRUD operations | User login with OAuth, file upload with processing |
| **ER Diagram** | Module owns data with 2+ related entities | Single entity, no relationships | Customer-Order-LineItem relationships |
| **Pseudo-code** | Complex algorithms, state machines, tricky business logic | Simple CRUD, straightforward UI rendering | Search ranking algorithm, retry logic with backoff |
| **State Diagram** | Components with multiple states and transitions | Stateless operations | Order workflow (pending→processing→shipped→delivered) |
| **API Contract** | Module exposes external interface | Internal-only module | REST endpoints consumed by frontend |

**Important:** These are OPTIONAL - only include when they add clarity. Simple modules need none of these.

### Detailed Diagram Guidelines

**Sequence Diagrams:**

Use for multi-step interactions involving 2+ components:

```
### [Operation Name] Sequence
Implements: REQ-X.X

┌─────────┐     ┌─────────┐     ┌─────────┐
│  User   │     │ Frontend│     │ Backend │
└────┬────┘     └────┬────┘     └────┬────┘
     │               │               │
     │  1. Action    │               │
     │──────────────>│               │
     │               │  2. API Call  │
     │               │──────────────>│
     │               │               │
     │               │  3. Response  │
     │               │<──────────────│
     │  4. Update    │               │
     │<──────────────│               │
     │               │               │
```

**ER Diagrams:**

Use when module manages related data entities:

```
### Data Model
Implements: REQ-X.X

┌──────────────┐       ┌──────────────┐
│   Customer   │       │    Order     │
├──────────────┤       ├──────────────┤
│ id (PK)      │───┐   │ id (PK)      │
│ name         │   │   │ customer_id(FK)│──┐
│ email        │   └──>│ total        │  │
│ created_at   │       │ status       │  │
└──────────────┘       │ created_at   │  │
                       └──────────────┘  │
                              ▲          │
                              │          │
┌──────────────┐              │          │
│  OrderItem   │──────────────┘          │
├──────────────┤                         │
│ id (PK)      │                         │
│ order_id (FK)│<────────────────────────┘
│ product_name │
│ quantity     │
└──────────────┘
```

**Pseudo-code:**

Use ONLY for complex logic that isn't obvious:

```
### [Algorithm Name]
Implements: REQ-X.X

FUNCTION calculate_discount(order, customer):
    base_discount = 0

    # Loyalty tier discount
    tier_discounts = {bronze: 0.05, silver: 0.10, gold: 0.15}
    base_discount = tier_discounts[customer.tier]

    # Volume discount (stacks with loyalty)
    IF order.total > 500:
        base_discount += 0.10
    ELIF order.total > 200:
        base_discount += 0.05

    # Cap at maximum discount
    max_discount = 0.25
    final_discount = MIN(base_discount, max_discount)

    RETURN order.total * final_discount
```

**DO NOT write pseudo-code for:**
- Simple CRUD operations
- Standard UI event handlers
- Basic data transformations
- Anything that's obvious from the description

**State Diagrams:**

Use when a component has multiple states:

```
### [Entity] State Machine
Implements: REQ-X.X

    ┌─────────┐
    │ PENDING │
    └────┬────┘
         │ confirm
         ▼
    ┌─────────┐     cancel
    │PROCESSING│─────────────┐
    └────┬────┘              │
         │ ship              │
         ▼                   ▼
    ┌─────────┐       ┌─────────┐
    │ SHIPPED │       │CANCELLED│
    └────┬────┘       └─────────┘
         │ deliver
         ▼
    ┌─────────┐
    │DELIVERED│
    └─────────┘
```

### Phase 3: Matrix Creation

Add to `architecture/architecture.md`:

**Module Registry:**
```markdown
## Module Registry

| Module # | Name | Purpose | Dependencies | Implements |
|----------|------|---------|--------------|------------|
| 1 | Foundation | Core utilities and storage | None | REQ-0 |
| 2 | Authentication | User identity management | 1 | REQ-1 |
| 3 | Dashboard | Main workspace | 1, 2 | REQ-2, REQ-3 |
```

**Integration Matrix:**

**Schema (MANDATORY columns):**

| Column | Required | Valid Values | Purpose |
|--------|----------|--------------|---------|
| From Module | Yes | Module ID (M1, M2...) | Caller/initiator |
| To Module | Yes | Module ID | Callee/dependency |
| Type | Yes | `Uses`, `Events`, `Reads` | Integration pattern |
| Interface | Yes | Method/endpoint name | Specific contract |
| Error Strategy | Yes | `Retry`, `Fallback`, `DLQ`, `Propagate` | Failure handling |

**Type Definitions:**
- `Uses`: Synchronous call (request-response)
- `Events`: Asynchronous messaging (fire-and-forget or pub-sub)
- `Reads`: Read-only data access (no mutations)

```markdown
## Integration Matrix

| From Module | To Module | Type | Interface | Error Strategy |
|-------------|-----------|------|-----------|----------------|
| M2-Auth | M1-Foundation | Uses | StorageAPI.save() | Retry(3) |
| M3-Dashboard | M2-Auth | Uses | AuthService.getCurrentUser() | Propagate |
| M4-Export | M3-Dashboard | Reads | DashboardAPI.getProjects() | Fallback(empty) |
| M3-Dashboard | M5-Notifications | Events | queue:project-updated | DLQ |
```

### Phase 4: Traceability Validation

**Invoke traceability-validator-agent** to verify:
- Every REQ-ID in PRD.md appears in at least one module's Requirement Coverage table
- No invalid REQ-IDs (references that don't exist in PRD.md)
- Every module has a Requirement Coverage section
- Architecture components' "Implements:" tags are fully covered by modules

**Auto-Fix Process:**
For each orphan requirement identified:
1. Determine the most appropriate module based on requirement description
2. Add the REQ-ID to that module's Requirement Coverage table
3. Fill in Implementation column with planned approach
4. Re-run validation until PASS

**Validation Loop:**
```
REPEAT:
  Run traceability-validator-agent
  IF status == PASS: BREAK
  FOR each orphan requirement:
    Identify target module
    Add to Requirement Coverage table
  FOR each missing coverage table:
    Add section to module
```

### Phase 4.5: Integration Matrix Cycle Detection (BLOCKING)

Before proceeding to Sum Test, verify the Integration Matrix has no circular dependencies:

**Cycle Detection Algorithm:**
```
FUNCTION detect_cycles(integration_matrix):
    # Build directed graph from Integration Matrix
    graph = {}
    FOR each row in integration_matrix:
        from_module = row["From Module"]
        to_module = row["To Module"]
        IF from_module not in graph:
            graph[from_module] = []
        graph[from_module].append(to_module)

    # DFS cycle detection
    visited = set()
    rec_stack = set()

    FUNCTION dfs(node, path):
        visited.add(node)
        rec_stack.add(node)
        path.append(node)

        FOR neighbor in graph.get(node, []):
            IF neighbor not in visited:
                cycle = dfs(neighbor, path)
                IF cycle: RETURN cycle
            ELIF neighbor in rec_stack:
                # Cycle found! Return the cycle path
                cycle_start = path.index(neighbor)
                RETURN path[cycle_start:] + [neighbor]

        rec_stack.remove(node)
        path.pop()
        RETURN None

    FOR node in graph:
        IF node not in visited:
            cycle = dfs(node, [])
            IF cycle:
                RETURN cycle
    RETURN None

# Execute
cycle = detect_cycles(integration_matrix)
IF cycle:
    PRINT f"CYCLE DETECTED: {' -> '.join(cycle)}"
    PRINT "Module design REJECTED - fix circular dependency"
    EXIT with failure
```

**If Cycle Detected:**
1. Report the exact cycle path (e.g., "M3 → M4 → M5 → M3")
2. **REJECT** the design - do not proceed to Sum Test
3. Require restructuring using one of these solutions:
   - Extract shared functionality to a new lower-level module
   - Use event-based decoupling instead of direct dependency
   - Merge tightly-coupled modules into one
   - Introduce an interface/abstraction layer

**This check is BLOCKING** - cyclic dependencies cause infinite implementation loops.

### Phase 5: Sum Test Validation (BLOCKING)

The Sum Test ensures modules exactly match architecture - no more, no less.

**Sum Test Algorithm:**

```
FUNCTION validate_sum_test():
    # Step 1: Extract architecture features
    arch_components = parse_architecture_md()
    arch_features = []
    FOR each component in arch_components:
        features = extract_features(component)  # from "Implements:" tags
        arch_features.extend(features)

    # Step 2: Extract module features
    module_features = {}
    FOR each module_file in architecture/modules/:
        module_id = extract_module_id(module_file)
        features = extract_requirement_coverage(module_file)
        module_features[module_id] = features

    # Step 3: Validate - every arch feature in EXACTLY one module
    errors = []
    FOR each feature in arch_features:
        modules_with_feature = [m for m, feats in module_features.items()
                                if feature in feats]
        IF len(modules_with_feature) == 0:
            errors.append(f"ORPHAN: {feature} not in any module")
        ELIF len(modules_with_feature) > 1:
            errors.append(f"DUPLICATE: {feature} in {modules_with_feature}")

    # Step 4: Validate - no invented features
    all_module_features = union(module_features.values())
    invented = all_module_features - arch_features
    IF invented:
        errors.append(f"INVENTED: {invented} not in architecture")

    # Step 5: Validate - component coherence
    FOR each component in arch_components:
        modules_claiming = [m for m, feats in module_features.items()
                           if any(f in feats for f in component.features)]
        IF len(modules_claiming) > 1:
            errors.append(f"SPLIT: Component '{component.name}' split across {modules_claiming}")

    RETURN len(errors) == 0, errors

# Execution
status, errors = validate_sum_test()
IF NOT status:
    PRINT "Sum Test FAILED:"
    FOR error in errors:
        PRINT f"  - {error}"
    EXIT with failure
ELSE:
    PRINT "Sum Test PASSED: Modules exactly match architecture"
```

**Sum Test Error Types:**

| Error | Meaning | Fix |
|-------|---------|-----|
| ORPHAN | Feature in architecture but no module covers it | Add to appropriate module's Requirement Coverage |
| DUPLICATE | Feature covered by multiple modules | Remove from all but one module |
| INVENTED | Module claims feature not in architecture | Remove from module OR add to architecture.md |
| SPLIT | Component divided across modules | Consolidate into single module |

**Auto-Fix Process:**
```
IF errors contain ORPHAN:
    FOR each orphan feature:
        target_module = find_best_module(feature)  # by component alignment
        ADD feature to target_module Requirement Coverage

IF errors contain DUPLICATE:
    FOR each duplicate feature:
        primary_module = determine_primary_owner(feature)
        REMOVE feature from other modules

IF errors contain INVENTED:
    WARN: "Review required - feature may need architecture update"
    # Do NOT auto-add to architecture (maintains architecture authority)

IF errors contain SPLIT:
    WARN: "Review required - consolidate component into single module"
```

**Combined Validation Loop (Phase 4 + Phase 5):**
```
REPEAT (max 5 iterations):
    Run traceability-validator-agent  # REQ-ID coverage
    Run sum_test_validation           # Feature coverage + coherence

    IF both PASS: BREAK

    Apply auto-fixes for traceability errors
    Apply auto-fixes for sum test errors (except INVENTED/SPLIT)

IF iterations exhausted AND still failing:
    PRINT "Manual review required for:"
    PRINT remaining errors
    EXIT with failure
```

## Output Structure

```
architecture/
├── architecture.md              # Updated with Module Registry + Integration Matrix
└── modules/
    ├── module-1-foundation.md   # With Requirement Coverage table
    ├── module-2-auth.md         # With Requirement Coverage table
    ├── module-3-dashboard.md    # With Requirement Coverage table
    └── ...
```

## CRITICAL CONSTRAINTS

**Traceability Rules:**
- **EVERY module** starts with Requirement Coverage table
- **EVERY REQ-ID** must be in at least one module's coverage table
- **NO orphans** - traceability-validator-agent must return PASS
- **NO invented requirements** - only reference REQ-IDs from PRD.md

**Sum Test (MANDATORY):**
```
Sum(Module 1 + Module 2 + ... + Module N) == architecture.md
```
- Every architecture.md feature in exactly ONE module
- No feature missing or duplicated
- All Integration Matrix connections maintained
- No expanded responsibilities beyond architecture.md

**Content Rules:**
- Requirement Coverage table is ALWAYS first section
- Human-readable design docs, not technical specs
- Technical details only where crucial (see table above)
- Plain English for most content

## Quality Checklist

**Traceability Checks (Phase 4):**
1. [ ] Does every module have a Requirement Coverage table as first section?
2. [ ] Is every REQ-ID from PRD.md covered by at least one module?
3. [ ] Does traceability-validator-agent return PASS status?
4. [ ] No invalid REQ-ID references (all exist in PRD.md)?

**Sum Test Checks (Phase 5 - BLOCKING):**
5. [ ] Sum Test algorithm executed successfully?
6. [ ] No ORPHAN errors (all architecture features covered by modules)?
7. [ ] No DUPLICATE errors (each feature in exactly one module)?
8. [ ] No INVENTED errors (no features beyond architecture)?
9. [ ] No SPLIT errors (components not divided across modules)?

**Coherence Checks:**
10. [ ] Each architecture component maps to exactly one module?
11. [ ] Modules have high cohesion (related features grouped)?
12. [ ] Module boundaries align with architectural boundaries?

**Structure Checks:**
13. [ ] Are modules ordered by implementation dependency?
14. [ ] Is Module Registry added to architecture.md?
15. [ ] Is Integration Matrix added with all required columns (Type, Interface, Error Strategy)?
16. [ ] Does Integration Matrix form valid DAG (no circular dependencies)?

**Diagram Quality Checks:**
17. [ ] Sequence diagrams included for complex multi-step operations?
18. [ ] ER diagrams included for modules with 2+ related entities?
19. [ ] Pseudo-code included ONLY for complex/non-obvious logic?
20. [ ] All diagrams have "Implements:" tags?
21. [ ] No unnecessary diagrams for simple CRUD operations?

**All checks must PASS before /generate-modules completes.**

## Example Module with Requirement Coverage

```markdown
# Module 2: Authentication

## Requirement Coverage

| REQ ID | Requirement | Implementation |
|--------|-------------|----------------|
| REQ-1.1 | Email/Password Login | LoginForm component + AuthService.login() |
| REQ-1.2 | Session Management | SessionManager with localStorage |
| REQ-1.3 | Password Recovery | ForgotPasswordForm + email verification flow |
| REQ-1.4 | Logout | AuthService.logout() + session cleanup |

## Overview

The Authentication module handles user identity management including login, logout, session persistence, and password recovery. It provides the security foundation for all user-specific features.

## User Stories

- As a user, I can create an account with email and password
- As a user, I can log in to access my projects
- As a user, I can stay logged in across browser sessions
- As a user, I can reset my password if forgotten
- As a user, I can log out from any device

## What Users See

### Login Screen
A centered card with email and password fields, "Remember me" checkbox, and "Forgot password?" link. Shows validation errors inline.

### Registration Screen
Similar to login with additional "Confirm password" field. Shows password strength indicator.

## Key Concepts

- User: An authenticated identity with email and encrypted password
- Session: A time-limited authentication token stored locally
- Password Reset Token: A one-time code sent via email

## Data Needs

- User credentials (email, hashed password)
- Session tokens with expiration
- Password reset tokens with expiration
- Login attempt tracking for rate limiting

## Interactions

- Depends on: Module 1 (Foundation) for storage
- Used by: All modules requiring user context

## Acceptance Criteria

- [ ] Can create account with email/password
- [ ] Can log in with valid credentials
- [ ] Session persists across browser refresh
- [ ] Can reset password via email
- [ ] Invalid credentials show appropriate error
- [ ] Rate limiting prevents brute force attempts
```

## Traceability Queries

With this structure, users can easily answer:

| Query | How |
|-------|-----|
| "Which modules implement REQ-3?" | `grep -r "REQ-3" architecture/modules/` |
| "What requirements does Module 4 cover?" | Read module-4-*.md → Requirement Coverage table |
| "Is REQ-2.3 implemented?" | Search for REQ-2.3 in all module specs |
| "What changed for REQ-5?" | Git diff on files containing REQ-5 |
