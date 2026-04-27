---
paths:
  - "src/**/*"
  - "tests/**/*"
---

# Retrofit Mode Rules

## Detection
Retrofit mode is active when the invoking context passes one of:
- `Retrofit: AS_IS` — POC file is production-quality; copy verbatim and apply a short list of surgical edits. No code regeneration is required in this mode.
- `Retrofit: ADAPT` — POC structure is sound; preserve POC code as a starting point, replace the mock/fake layer with real implementations, and fill any production gaps from the module spec.
- `Retrofit: REWRITE` — POC code is reference only. Write fresh production code from the module spec, preserving visual/behavioral patterns observed in POC where they align.

## Source of Truth
The module spec (`architecture/modules/module-{N}-{name}.md`) defines WHAT to build.
The retrofit context (passed by the invoking context, derived from an upstream code promotion analysis) defines HOW to start.
**The module spec always wins** if it conflicts with POC code.

## Retrofit: AS_IS

The POC code satisfies the module spec as-is (upstream analysis reported `poc_coverage ≥ 0.80`, framework identical to production, clean or no mocks). In this mode, no code generation is performed — the invoking context handles promotion directly:

**What the invoking context does:**
- Copies each POC file listed in the retrofit context to its target production path
- Applies the short list of surgical edits named in the retrofit context (typical edit types: swap a hardcoded value for an env-var read, replace POC-specific file-path references in error messages, rename a domain term that changed during POC → PRD sync, update an import path for the production directory layout)
- Does NOT reformat, restructure, or regenerate the file

**What still runs (unchanged from other modes, but orchestrated by the invoking context — not referenced here):**
- L1 test generation
- L1 unit test gate (coverage target)
- Smoke test
- Module tracking update

**Defense-in-depth:** If this rule is consulted by a code-generation consumer in AS_IS mode (orchestration bug), the consumer MUST return immediately with status `"AS_IS — no implementation required; files are handled by the invoking context"` and touch **no files**.

**Key rule:** AS_IS is the fast path for production-quality POCs. If the upstream analysis misclassified and a fresh test failure reveals the POC code does not actually satisfy the spec, the module is marked `Blocked` — do not silently upgrade to ADAPT. Escalate so the upstream analysis can re-run or the code can be fixed manually.

## Retrofit: ADAPT

The retrofit context provides: POC file list with actions, gaps list, and migration steps.

1. **Read the POC files** listed in the retrofit context
2. **Copy POC code** as the starting point for each production file
3. **Adapt to production framework:** Replace framework-specific code (e.g., POC routing to production routing conventions, POC bundler to production bundler, client-side rendering to server-side rendering where appropriate)
4. **Replace all mocks:** Every mock data import, fake API call, simulated timer, and hardcoded response must be replaced with real implementations per the module spec
5. **Fill gaps:** Implement everything listed in the gaps section — these are production requirements the POC did not cover
6. **Follow migration steps** in the order provided
7. **Validate against module spec:** The final implementation must satisfy ALL requirements in the module spec, not just what the POC covered

**Key rule:** POC code is a starting accelerator, not a constraint. If the POC approach conflicts with the module spec, the module spec wins.

## Retrofit: REWRITE

The retrofit context provides: POC file list (reference only), behavioral patterns to preserve, and gaps list.

1. **Read the POC files** listed in the retrofit context for behavioral/visual reference
2. **Do NOT copy POC code** — write fresh production code from the module spec
3. **Preserve visual and behavioral patterns** observed in the POC where they align with the module spec (e.g., layout structure, interaction flows, component hierarchy)
4. **Extract reusable logic** from POC files when explicitly noted in the retrofit context (e.g., "Extract `renderHighlightedText` algorithm")
5. **Implement the full module spec** including all gaps

**Key rule:** The POC is a reference, not a template. The implementation is driven entirely by the module spec.

## Workflow Overrides
In Retrofit mode, the standard implementation workflow is NOT modified:

- **All test gates apply:** L1 unit tests (60% coverage), smoke tests, L2 integration tests
- **All fix loops apply:** Same max attempts, same agent invocations
- **Dependency ordering:** Same as Normal mode (from Integration Matrix)
- **Output location:** All code goes to `src/` (NOT `poc/src/`)
- **Coverage target:** As specified by the invoking context (default 60%)

For test gate details, see `test-gates.md`.
For test limit details, see `test-limits.md`.
For output locations, see `project-structure.md` (src/ section).
