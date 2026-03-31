# Retrofit Mode Rules

## Detection
Retrofit mode is active when the invoking command passes:
- `Retrofit: REFACTOR` — POC code is the starting point, adapted for production
- `Retrofit: REWRITE` — POC code is a reference only, fresh production code written
- `Retrofit: WRITE NEW` — No POC code relevant, identical to Normal mode

## Source of Truth
The module spec (`architecture/modules/module-{N}-{name}.md`) defines WHAT to build.
The retrofit context (from `POC_CODE_PROMOTE_PLAN.md`) defines HOW to start.
**The module spec always wins** if it conflicts with POC code.

## Retrofit: REFACTOR

The invoking command provides: POC file list with actions, gaps list, and migration steps.

1. **Read the POC files** listed in the retrofit context
2. **Copy POC code** as the starting point for each production file
3. **Adapt to production framework:** Replace framework-specific code (e.g., POC routing to production routing conventions, POC bundler to production bundler, client-side rendering to server-side rendering where appropriate)
4. **Replace all mocks:** Every mock data import, fake API call, simulated timer, and hardcoded response must be replaced with real implementations per the module spec
5. **Fill gaps:** Implement everything listed in the gaps section — these are production requirements the POC did not cover
6. **Follow migration steps** in the order provided
7. **Validate against module spec:** The final implementation must satisfy ALL requirements in the module spec, not just what the POC covered

**Key rule:** POC code is a starting accelerator, not a constraint. If the POC approach conflicts with the module spec, the module spec wins.

## Retrofit: REWRITE

The invoking command provides: POC file list (reference only), behavioral patterns to preserve, and gaps list.

1. **Read the POC files** listed in the retrofit context for behavioral/visual reference
2. **Do NOT copy POC code** — write fresh production code from the module spec
3. **Preserve visual and behavioral patterns** observed in the POC where they align with the module spec (e.g., layout structure, interaction flows, component hierarchy)
4. **Extract reusable logic** from POC files when explicitly noted in the retrofit context (e.g., "Extract `renderHighlightedText` algorithm")
5. **Implement the full module spec** including all gaps

**Key rule:** The POC is a reference, not a template. The implementation is driven entirely by the module spec.

## Retrofit: WRITE NEW

No retrofit-specific context is provided beyond the mode indicator. This is identical to Normal mode — implement from the module spec with no POC reference.

## Workflow Overrides
In Retrofit mode, the standard implementation workflow is NOT modified:

- **All test gates apply:** L1 unit tests (60% coverage), smoke tests, L2 integration tests
- **All fix loops apply:** Same max attempts, same agent invocations
- **Dependency ordering:** Same as Normal mode (from Integration Matrix)
- **Output location:** All code goes to `src/` (NOT `poc/src/`)
- **Coverage target:** As specified by invoking command (default 60%)

For test gate details, see `test-gates.md`.
For test limit details, see `test-limits.md`.
For output locations, see `project-structure.md` (src/ section).
