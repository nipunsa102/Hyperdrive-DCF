## Hyperdrive — DCF Command Order

```
/generate-prd → /generate-architecture → /generate-modules → /generate-code → /deploy-module
```
POC loop (optional, after /generate-architecture — replaces /generate-modules):
```
/generate-poc → /modify-poc (repeat) → /sync-prd → /promote-poc-design → /generate-code -retrofit
```

## Core Rules

**Working Guidelines:**
- Do exactly what's asked; nothing more, nothing less

 **Simplicity (KISS):**
 - Simplest working solution wins
 - YAGNI - no hypothetical future features

 **Security (OWASP):**
 - Never hardcode secrets - use configuration options
 - Never concatenate SQL/commands - use parameterized APIs
 - Never log PII/passwords/tokens

 **Code Quality:**
 - Single Responsibility per module/class/function
 - Explicit names, no magic numbers
 - Fail fast, validate early
 - Immutable by default, pure functions preferred

 **Error Handling:**
 - Friendly user messages, detailed logs
 - Never swallow exceptions
 - Timeout all external calls