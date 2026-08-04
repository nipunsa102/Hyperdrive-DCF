---
paths:
  - "tests/**/*"
---

# Test Case Limits

HARD LIMITS on test case generation. These are NOT suggestions - they are enforced.

## L1 Unit Tests

**Maximum 5 test cases per module.** (HARD LIMIT - NO EXCEPTIONS)

- 5 is the absolute maximum
- Combine related tests into single test cases
- Test PUBLIC API only, not implementation
- Default coverage target: 60%. Actual target is passed by the invoking command.

| # | Test Type | Purpose |
|---|-----------|---------|
| 1 | Happy Path | Main function works |
| 2 | Happy Path 2 | Secondary function works |
| 3 | Invalid Input | Rejects bad data |
| 4 | Error Case | Handles failures |
| 5 | Edge Case | Boundary (optional) |

## L2 Integration Tests

**Maximum 10 integration tests total.** (HARD LIMIT)

| Category | Max Tests |
|----------|-----------|
| Critical user flows | 5 |
| API integration | 3 |
| Error handling | 2 |
| **Total** | **10** |

## E2E Journey Tests (direct path, `/generate-code` E2E gate)

**Maximum 10 journey tests total.** (HARD LIMIT)

- Headless browser for UI systems (mandatory); API/CLI journeys otherwise
- Derived from architecture user journeys + module acceptance criteria, prioritized by requirement criticality
- Each journey tagged with the REQ-IDs it exercises
- Module-scope checks (during the per-module loop): maximum 2 checks per screen, ephemeral (not persisted)

## Why These Limits?

1. **Quality over quantity** - More tests ≠ better tests
2. **Maintainability** - Fewer tests are easier to maintain
3. **Speed** - Large test suites slow down CI/CD
4. **Focus** - Forces prioritization of critical paths
5. **Token efficiency** - Reduces generation costs

## What To Do Instead

- Combine related assertions into single test cases
- Use parameterized tests for similar scenarios
- Test the contract, not implementation details
- Skip trivial getters/setters
- Focus on business logic and error paths

## Enforcement

If you find yourself exceeding these limits:
1. STOP and review your test strategy
2. Identify tests that can be combined
3. Remove low-value tests (trivial cases)
4. Ensure you're testing behavior, not implementation
