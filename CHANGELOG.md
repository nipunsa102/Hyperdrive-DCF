# Changelog

All notable changes to the Hyperdrive DCF framework will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- `/plan-deployment` command — direct-path analog of `/prepare-poc-promo`. Generates `DEPLOYMENT.md` at project root: a vendor-agnostic deployment & environment decision template (`DEP`/`DATA`/`AUTH`/`SEC`/`INT`/`OPS` gaps with `<!-- REQUIRED -->` placeholders, pre-filled where project documents already decide, optional read-only platform verification). Always surfaces the **dev run mode** decision: fully local / cloud / hybrid. Runs after `/generate-architecture`, before `/generate-modules`; hard-fails on POC-path projects. Initially authored as `/prepare-deployment`; renamed before release — the step plans deployment upfront, before modules exist, rather than preparing an imminent deploy.
- `deployment-gap-analyzer-agent` — scans PRD/architecture/data-model/TECHSTACK for unanswered environment-coupled decisions (no POC required).
- `e2e-test-agent` — headless-browser E2E verification (mandatory for UI systems; API/CLI journeys otherwise) against the running dev environment. Module scope (render + primary interaction per screen) and full scope (≤10 user journeys persisted under `tests/e2e/`).
- `/generate-code` Step 0.5 "Environment Bootstrap" — provisions/verifies the dev environment per `DEPLOYMENT.md`'s dev run mode (dev-scoped only, idempotent), writes dev config, checks connectivity, and logs to `tracking/env-setup.md`.
- `/generate-code` Step 3.5 "E2E Functional Gate" (blocking) — headless-browser journeys against the running dev app with a command-orchestrated fix loop; cloud mode additionally deploys the dev build and re-verifies.

### Changed
- `/generate-code` is now the direct path's orchestrator: environment bootstrap → per-module implementation with L1/smoke (+ per-module headless check for UI modules) against the real dev environment → L2 → blocking E2E gate → finalize. Ends with the app RUNNING in the chosen dev mode; `CONFIG_GUIDE.md` becomes the as-built record + production walkthrough.
- `/generate-modules` and `/generate-code` require a completed `DEPLOYMENT.md` (no unfilled `<!-- REQUIRED` markers) and consume its decisions — module specs and code are generated for a known target environment instead of environment-blind.
- `/setup-env` is POC-path-only again — the direct path has no separate environment-setup step.
- Test gates: E2E functional gate added as blocking on the direct path (max 10 journeys, `tests/e2e/`); smoke runs against the bootstrapped dev environment.

### Removed
- `/deploy-module` placeholder command and `deploy-config-agent` placeholder — superseded by `DEPLOYMENT.md` + `/generate-code`'s dev bootstrap; production deployment will be a future `/deploy-to-prod` command.

## [0.1.0] - 2026-02-24

### Added
- Core DCF workflow: `/generate-prd`, `/generate-architecture`, `/generate-modules`, `/generate-code`, `/deploy-module`
- POC workflow: `/generate-poc`, `/modify-poc`, `/sync-prd`
- Maintenance command: `/update-tracking`
- Agent configurations: code-review, deploy-config, L1/L2 test, smoke-test, and more
- Project rules for structure enforcement
- Template files for `OVERVIEW.md`, `PRD.md`, and `TECHSTACK.md`
- MIT license
- Contributing guidelines
