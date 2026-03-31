---
description: Generate a structured Product Requirements Document from OVERVIEW.md with hierarchical REQ-IDs
model: claude-opus-4-6
---

## Purpose

Transform freeform requirements in OVERVIEW.md into a structured, numbered Product Requirements Document (PRD) with hierarchical REQ-IDs. This is the ROOT document for requirement traceability in DCF.

## Process

### Phase 1: Understanding

1. **Read OVERVIEW.md**
   - Identify all requirement sections (look for `[N]` numbering or logical groupings)
   - Note implicit requirements (mentioned features, expected behaviors)
   - Capture non-functional requirements (performance, security, usability)

2. **Read Referenced Documents**
   - Read any images (`.png`, `.jpg`) referenced in OVERVIEW.md
   - Read any linked `.md` or `.docx` files
   - Extract additional requirements from these sources

3. **Categorize Requirements**
   - Group related requirements into feature categories
   - Identify P1 (must-have), P2 (should-have), P3 (nice-to-have)
   - Note dependencies between requirement groups

### Phase 1.5: Requirement Classification

When extracting requirements, classify each by traceability:

| Type | Definition | Action |
|------|------------|--------|
| **Explicit** | Directly stated in OVERVIEW | Include with direct source citation |
| **Implicit** | Shown in examples/diagrams | Include with "[Implicit from section N]" |
| **Derived** | Logically necessary from stated reqs | Include with "[Derived from REQ-X]" |
| **Invented** | Not traceable to OVERVIEW | **REJECT - Do not include** |

**Source Tag Rules:**
- MUST reference actual OVERVIEW section numbers `[N]` or named sections
- "Architecture Decision" is INVALID (architecture comes AFTER PRD)
- "Technical Decision" is INVALID (technical details come AFTER PRD)
- If requirement comes from example/diagram, cite the section containing it
- If requirement is derived, cite the parent requirement it derives from

### Phase 2: PRD Generation

Create `PRD.md` at **ROOT level** (alongside OVERVIEW.md) with this structure:

```markdown
# Product Requirements Document

## Document Info
- **Source:** OVERVIEW.md
- **Version:** 1.0
- **Last Updated:** YYYY-MM-DD

---

## REQ-1: [Feature Category Name]
**Priority:** P1/P2/P3
**Source:** OVERVIEW section [N]

### REQ-1.1: [Specific Requirement]
[Clear, testable description of what the system must do]

### REQ-1.2: [Specific Requirement]
[Clear, testable description of what the system must do]

---

## REQ-2: [Feature Category Name]
**Priority:** P1/P2/P3
**Source:** OVERVIEW section [N]

### REQ-2.1: [Specific Requirement]
[Description]

---
```

**Numbering Rules:**
- Top-level: `REQ-1`, `REQ-2`, etc. (feature categories)
- Sub-requirements: `REQ-1.1`, `REQ-1.2`, etc.
- Deep nesting if needed: `REQ-1.1.1` (avoid more than 3 levels)
- IDs are permanent - never renumber after assignment

**Requirement Quality:**
- Each requirement must be **specific** and **testable**
- Avoid vague terms ("fast", "easy", "user-friendly") without metrics
- Include acceptance criteria where possible
- Cross-reference related requirements

### Phase 3: Validation

**Invoke traceability-validator-agent** to verify:
- All OVERVIEW.md sections have corresponding REQ-IDs
- No orphaned content (requirements mentioned but not captured)
- REQ-IDs follow naming convention
- Source mappings are accurate

**Fix any gaps identified:**
- Add missing requirements
- Clarify ambiguous requirements
- Ensure complete coverage of OVERVIEW.md

### Phase 3.5: Source Tag Validation

For each REQ in the generated PRD, verify its Source tag:

1. **Parse the Source reference** (e.g., "OVERVIEW section [2]")
2. **Read that OVERVIEW section** and confirm the requirement is stated or implied there
3. **REJECT and remove** any REQ if Source references:
   - "Architecture" or "Architecture Decision" (PRD precedes architecture)
   - "Technical Decision" or "Implementation" (PRD precedes technical design)
   - Non-existent section numbers
   - Sections that don't contain the requirement

**Validation Loop:**
```
FOR each REQ in PRD:
    source = parse_source_tag(REQ)
    IF source contains "Architecture" OR source contains "Technical":
        REJECT REQ - invalid source type
    IF source references section [N]:
        content = read_overview_section(N)
        IF REQ not stated or implied in content:
            REJECT REQ - source mismatch
```

**If any REQs rejected:** Remove them from PRD or find valid OVERVIEW source.

## Output

```
/PRD.md                    # Structured requirements with REQ-IDs (ROOT level)
```

## Quality Checklist

Before completion:
1. Every OVERVIEW.md feature has a corresponding REQ-ID?
2. All REQ-IDs are specific and testable?
3. Priority levels (P1/P2/P3) assigned to all categories?
4. Source mappings back to OVERVIEW.md sections?
5. No duplicate requirements?
6. Hierarchical structure is logical and consistent?

## Example PRD Output

```markdown
# Product Requirements Document

## Document Info
- **Source:** OVERVIEW.md
- **Version:** 1.0
- **Last Updated:** YYYY-MM-DD

---

## REQ-1: User Authentication
**Priority:** P1
**Source:** OVERVIEW section [2]

### REQ-1.1: Email/Password Login
Users must be able to create an account and log in using email and password. Passwords must meet minimum security requirements (8+ characters, mixed case, numbers).

### REQ-1.2: Session Management
User sessions must persist across browser refreshes. Sessions expire after 24 hours of inactivity.

### REQ-1.3: Password Recovery
Users must be able to reset forgotten passwords via email verification.

---

## REQ-2: Project Management
**Priority:** P1
**Source:** OVERVIEW section [3]

### REQ-2.1: Project Creation
Users can create named projects with optional descriptions and tags.

### REQ-2.2: Project Organization
Projects can be starred, archived, and organized into folders.

---

## REQ-3: Export/Import
**Priority:** P2
**Source:** OVERVIEW section [7]

### REQ-3.1: PDF Export
Users can export project content to PDF format with customizable layouts.

### REQ-3.2: Data Backup
Users can export all project data as JSON for backup purposes.

---
```

## CRITICAL CONSTRAINTS

- **ROOT LEVEL:** PRD.md must be at project root, not in architecture/
- **COMPLETE COVERAGE:** Every OVERVIEW.md requirement must have a REQ-ID
- **PERMANENT IDs:** Once assigned, REQ-IDs never change
- **TRACEABLE SOURCE:** Every REQ must map back to OVERVIEW.md section
- **NO INVENTION:** Do not add requirements not in OVERVIEW.md
