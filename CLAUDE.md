## Hyperdrive — DCF Command Order

```
/generate-prd → /generate-architecture → /generate-modules → /generate-code → /setup-env → /deploy-module
```
POC loop (optional, after `/generate-architecture` — **requires `architecture/modules/` to be empty**; `/promote-poc` bootstraps modules from the POC):
```
/generate-poc → /modify-poc (repeat) → /sync-prd → /prepare-poc-promo → [human fills in] → /promote-poc → [human fills .env from CONFIG_GUIDE.md] → /setup-env
```

> `/generate-poc` hard-fails if `architecture/modules/` is non-empty. POC path is for greenfield work where stakeholder feedback will shape requirements; if you're in execution mode with stable requirements, use Path A.
>
> `/promote-poc` handles the code side: POC maturity classification, architecture merge, module bootstrap, per-module execution (`AS_IS` / `ADAPT` / `REWRITE`), all test gates, cross-cutting decision sweep. It outputs `CONFIG_GUIDE.md` and `POC_PROMOTION_REPORT.md`.
>
> `/setup-env` handles the environment side: validates `.env`, runs DB migrations, seeds reference data (production-safe only), verifies connectivity to every external service, runs a runtime smoke test against real services. Makes the code actually runnable.

Post-promotion change loop (on a feature git branch):
```
/modify -change|-new|-fix "<description>"
```
> `/modify` edits PRD.md, architecture, and src/ directly; runs L1 + smoke + L2 gates; appends to `tracking/change-tracking.md`.

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