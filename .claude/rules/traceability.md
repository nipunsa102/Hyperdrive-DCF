---
paths:
  - "architecture/**/*.md"
  - "PRD.md"
---

# Requirement Traceability Rules

All requirements must be traceable from PRD.md through to implementation.

## REQ-ID Format
- Top-level: `REQ-1`, `REQ-2`, etc.
- Sub-requirements: `REQ-1.1`, `REQ-1.2`, etc.
- Maximum 3 levels: `REQ-1.1.1`

## PRD.md Requirements
- Every requirement has a unique REQ-ID
- REQ-IDs are permanent (never renumber)
- Each REQ maps back to OVERVIEW.md section

## architecture.md Requirements
- Every component section has an `**Implements:**` tag
- Tag lists REQ-IDs from PRD.md
- All PRD REQ-IDs covered by at least one component

## Module Spec Requirements
- First section is always `## Requirement Coverage`
- Table format: `| REQ ID | Requirement | Implementation |`
- All REQ-IDs from component's Implements tag must appear

## Validation
- The invoking context runs structural traceability validation to check coverage
- No orphan requirements allowed (REQ in PRD but not in modules)
- No invented requirements (REQ in module but not in PRD)
