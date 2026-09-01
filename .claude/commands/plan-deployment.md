---
description: Generate a deployment & environment decision template (DEPLOYMENT.md) for human input before modules and code are generated on the direct path
model: fable
---

## Purpose

The direct-path analog of `/prepare-poc-promo`. On the POC path, environment decisions (hosting, identity wiring, data platform access, secrets, integrations) are captured in `poc/temp/poc_promotion/POC_PROMO_PREP.md` and applied by `/promote-poc`. On the direct path (`/generate-modules` → `/generate-code`) no equivalent step existed — code generated blind to its target environment, and the initial DEV environment, hosting approach, and constraints had to be reverse-engineered afterwards.

This command closes that gap. It analyzes the design documents, identifies every environment-coupled decision the design demands but no document answers, and generates **`DEPLOYMENT.md` at project root** as a **template with `<!-- REQUIRED: ... -->` placeholders**. The human fills in the placeholders at their own pace, then runs `/generate-modules`. Once the document is complete, the rest of the pipeline (`/generate-modules` → `/generate-code`) can produce production-ready code AND a running, verified dev environment — `/generate-code` bootstraps the chosen dev setup and gates on end-to-end verification (headless browser for UI systems) — without ever going back to ask what cloud resources or hosting approach to take.

**Position in flow:** After `/generate-architecture`, before `/generate-modules`. **Direct path only** — never part of the POC loop.

**Division of responsibility** (why this is a separate document):
- `TECHSTACK.md` — *what technologies* (languages, frameworks, libraries). Stays portable.
- `architecture/architecture.md` — *what the system is*. Stays platform-agnostic on the direct path; this command never modifies it.
- `DEPLOYMENT.md` — *where and how it runs*: provider/platform, compute model, environments, identity wiring, data platform access, secrets/config mechanism, integration tenancy, delivery approach. The single home for platform specifics.

**What this command does:**
- Invokes `deployment-gap-analyzer-agent` to scan the design documents for unanswered environment decisions, categorized as `DEP` / `DATA` / `AUTH` / `SEC` / `INT` / `OPS`
- Always surfaces the **development run mode** decision — fully local / cloud / hybrid (local app against cloud dev services) — unless a project document already decides it; `/generate-code`'s environment bootstrap and E2E verification depend on it
- Pre-fills every decision a project document already answers (with the source cited) — the human reviews those instead of answering them
- Optionally verifies platform context **read-only** (naming conventions, name availability, region precedents) when a provider is already decided and its CLI is available
- Generates `DEPLOYMENT.md` — the document IS the questionnaire; no interactive Q&A during the command

**What this command does NOT do:**
- Create, modify, or delete any cloud resource (verification is strictly read-only)
- Collect real secrets, API keys, or credentials (strategy decisions only)
- Modify `architecture/`, `src/`, or any other project file — only `DEPLOYMENT.md` is written
- Apply to the POC path (that path's decisions flow through `/prepare-poc-promo` + `/promote-poc`)
- Hardcode any cloud vendor — every provider/product name is either quoted from a project document or offered as an example

## Prerequisites

1. `PRD.md` must exist (run `/generate-prd` first)
2. `architecture/architecture.md` must exist (run `/generate-architecture` first)
3. `architecture/data-model.md` must exist (run `/generate-architecture` first)
4. **POC-path guard:** `poc/src/` must NOT exist. If it does, ERROR and STOP:
   ```
   ERROR: This project is on the POC path (poc/src/ exists).
   Deployment and environment decisions for the POC path are captured by
   /prepare-poc-promo (poc/temp/poc_promotion/POC_PROMO_PREP.md) and applied
   by /promote-poc. /plan-deployment is for the direct path only.
   ```
5. If `architecture/modules/` already contains module files AND `DEPLOYMENT.md` does not exist yet, WARN (do not stop):
   ```
   WARNING: architecture/modules/ contains {N} module specs that were generated
   without deployment context. After filling in DEPLOYMENT.md, review whether the
   module specs need regeneration (/generate-modules) before /generate-code.
   ```

If any prerequisite fails, ERROR with a clear message and stop.

## Process

### Phase 1: Context Gathering

Read the following files:
1. `PRD.md`
2. `architecture/architecture.md`
3. `architecture/data-model.md`
4. `architecture/modules/*.md` (if they exist)
5. `TECHSTACK.md` (if exists — authoritative where it speaks)
6. `OVERVIEW.md` (stakeholder hosting/constraint direction often lives here)
7. `DESIGNGUIDE.md` (if exists)
8. `infra/` contents (if any)
9. `DEPLOYMENT.md` (if exists — this is a re-run; see Phase 5)

### Phase 2: Gap Analysis

**INVOKE `deployment-gap-analyzer-agent`** with the following context:

```
PLAN-DEPLOYMENT — ENVIRONMENT GAP ANALYSIS:

Analyze the design documents and identify every environment-coupled decision
the design demands but no project document answers. Use ONLY these categories:

1. Deployment Target & Compute (DEP) — provider/platform, compute model
   (managed app platform / containers / serverless functions / static hosting +
   API / VMs / on-prem / local-only — all examples), deployable topology,
   environments and which is built first, release workflow, region, naming
   conventions, first-environment tier guidance, and the DEVELOPMENT RUN MODE
   (fully local / cloud / hybrid: local app process against cloud dev services).
   The dev-run-mode question is ALWAYS a gap unless a project document
   explicitly decides it.
2. Data & Storage Environment (DATA) — where each persistent store runs per
   environment, dev-database strategy, application-to-database authentication
   model, network path, migration execution context, initial/reference data
   source and where it is available
3. Identity & Access (AUTH) — identity integration model, per-environment
   identity source, LOCAL DEV identity strategy and deployed-environment guard,
   role source and mapping injection
4. Secrets & Configuration (SEC) — configuration mechanism per environment,
   local config conventions, fail-fast validation expectation, local developer
   credentials for shared services, where true secrets live
5. External Integrations (INT) — per architecture-declared service: tenancy per
   environment, provisioning ownership, authentication, network constraints
6. Build, Delivery & Operations (OPS) — build artifact shape, runtime entry
   expectation, CI/CD, IaC tool and infra/ layout, telemetry sink (ADVISORY-leaning)

OUT OF SCOPE: observability stack design, error-handling/retry defaults,
performance targets, application feature gaps.

For each gap provide: Gap ID, evidence (document sections or explicit silences),
severity (BLOCKING / ADVISORY), resolution (HUMAN / PREFILL), suggested
placeholder question, suggested options with a recommended default.
Also return: Pre-Fillable Decisions (with sources) and Already Determined items.

Input files: PRD.md, architecture/architecture.md, architecture/data-model.md,
architecture/modules/ (if exists), TECHSTACK.md (if exists), OVERVIEW.md,
DESIGNGUIDE.md (if exists), infra/, DEPLOYMENT.md (if exists — treat filled
decisions as answered).

Vendor-agnostic rule: name a provider/product only when a project document
names it, and cite the source. Otherwise providers appear as examples only.
```

**Wait for the agent to complete.** Capture its structured gap list, pre-fillable decisions, and already-determined items.

### Phase 3: Read-Only Platform Verification (conditional)

Runs only when BOTH hold:
- A target provider/platform is already decided — by `TECHSTACK.md`/`OVERVIEW.md`, or by a filled `DEP` decision in an existing `DEPLOYMENT.md` (re-run)
- That provider's CLI is installed AND already authenticated (e.g., `az`, `aws`, `gcloud`, `flyctl`, `vercel` — whichever matches the decided provider; never install or prompt a login)

If it runs, perform **strictly read-only** exploration and record findings for the "Verified Platform Context" section:
- Account/subscription/tenant context currently active
- Region conventions (where existing workloads live)
- Existing resource naming precedents relevant to the proposed names
- Name-availability checks for proposed resource names (read-only availability APIs only)
- Relevant provider feature/service registration status (read-only)

**Hard rules:** No create/update/delete subcommands. No writes of any kind. Never echo credentials or tokens into the document. If the CLI is unavailable, not authenticated, or the provider is undecided: skip this phase and mark the section `Not verified — [reason]. Re-run /plan-deployment after deciding DEP-1 to verify names and conventions read-only.`

### Phase 4: Generate Template

Create `DEPLOYMENT.md` at project root.

**Document structure:**

```markdown
# Deployment & Environment Preparation

Generated by `/plan-deployment` on [date]

**Scope:** environment-coupled decisions the design demands but no project document
answers — analyzed from PRD.md, architecture/architecture.md, architecture/data-model.md,
and TECHSTACK.md. Decisions already made elsewhere are pre-filled with their source.

## How This Document Is Used

- `/generate-modules` and `/generate-code` REFUSE to run while any `<!-- REQUIRED -->`
  placeholder remains. Pre-filled decisions count as filled.
- Module specs and generated code consume these decisions; `architecture/architecture.md`
  stays platform-agnostic.
- `/generate-code` bootstraps the DEV environment declared here (per the dev run mode:
  fully local / cloud / hybrid), verifies it end-to-end (headless browser for UI systems),
  and derives `CONFIG_GUIDE.md` + the configuration template as the as-built record.
- Production deployment consumes the DEP/OPS decisions later, via the future
  `/deploy-to-prod` command.

## What You Need To Fill In

| # | Gap ID | Question | Section |
|---|--------|----------|---------|
| 1 | DEP-1 | [question from gap analysis] | [DEP-1](#dep-1-title) |
| ... | ... | ... | ... |

Replace each `<!-- REQUIRED: ... -->` placeholder in the sections below with your decision.
When all placeholders are filled, run `/generate-modules`.

## Pre-Filled Decisions

Already answered by your project documents or unambiguous convention — review and edit
before `/generate-modules` only if you disagree:

| Gap ID | Decision (summary) | Source |
|---|---|---|
| [ID] | [decision] | [TECHSTACK.md §..., OVERVIEW.md, ...] |

## Verified Platform Context (read-only, [date])

[Findings table: account context, region conventions, naming precedents,
name-availability results — OR "Not verified — [reason]"]

## Environments & Run Modes

| Environment | Purpose | Exists first? |
|---|---|---|
| [e.g., dev (acts as UAT)] | [purpose] | [yes/no] |

Derived from the DEP dev-run-mode decision (fully local / cloud / hybrid):

| Mode | Runs | Identity | Data |
|---|---|---|---|
| [1] Local dev | [developer machine, dev command] | [per AUTH decisions] | [per DATA decisions] |
| [2] [Deployed env] | [platform per DEP decisions] | [per AUTH decisions] | [per DATA decisions] |

## Resource & Naming Plan

DEV-environment resources — verified-or-created by `/generate-code`'s environment
bootstrap (idempotent, dev-scoped only; requires an authenticated provider CLI for
cloud/hybrid modes) or by a human beforehand. Production resources are NEVER created
here — they wait for the future `/deploy-to-prod`. Derived once DEP decisions are
filled (re-run `/plan-deployment` to enrich and verify names read-only).

| Resource | Proposed name | Notes |
|---|---|---|
| [resource] | [name or "pending DEP-1"] | [availability-checked? convention source?] |

## Configuration Key Plan

Canonical configuration keys generated code MUST use (so code, the bootstrapped dev
config, and `CONFIG_GUIDE.md` stay consistent). `/generate-code` may add keys but must
not rename these.

| Key | Purpose | Source of value | Environments |
|---|---|---|---|
| [KEY] | [purpose] | [where the value comes from — never the value itself] | [local / deployed / all] |

---

## Deployment Target & Compute

### DEP-1: [Title]
**Severity:** [BLOCKING/ADVISORY]
**Evidence:** [document sections or silences, from gap analysis]
**Options:** [option 1 (recommended)], [option 2], [option 3], Other
**Decision:** <!-- REQUIRED: [question] -->

## Data & Storage Environment

### DATA-1: [Title]
**Severity:** ...
**Evidence:** ...
**Options:** ...
**Decision:** <!-- REQUIRED: [question] -->

## Identity & Access

### AUTH-1: ...

## Secrets & Configuration

### SEC-1: ...

## External Integrations

### INT-1: ...

## Build, Delivery & Operations

### OPS-1: ...

---

## Already Determined — no action needed

Verified by the gap analysis; recorded so downstream commands don't re-litigate:

- [decision + source, one bullet per item]

## Validation Rules

`/generate-modules` and `/generate-code` will scan this document for remaining
`<!-- REQUIRED` markers. If any are found, they will refuse to proceed and list the
unfilled items. All `<!-- REQUIRED: ... -->` placeholders must be replaced with your
decision text. Pre-filled decisions count as filled — review them and edit freely.
```

**Rules for generating the template:**
- Only include gaps the agent identified — don't invent categories with no gaps. A category with zero gaps is omitted entirely.
- If `TECHSTACK.md` or another project document already answers a question, it goes in **Pre-Filled Decisions** (with source) or **Already Determined** — never as a `<!-- REQUIRED -->` question.
- `PREFILL`-resolution gaps get their resolved decision text written in place (marked *"(Pre-filled: [source] — review before `/generate-modules`.)"*), not a placeholder.
- The "What You Need To Fill In" table is a **read-only index** listing only the gaps that still need a human answer.
- Each `HUMAN` gap gets exactly one `<!-- REQUIRED: ... -->` placeholder.
- Options lists put the recommended choice first, marked "(recommended)", and always end with "Other".
- Never include placeholders for actual secret values — only strategy decisions and key *names*.
- Provider/product names appear only as quoted project decisions or as examples ("e.g., ...").
- "Local-only / no hosted environment" is a legitimate DEP answer — a project with no cloud target still completes this step (quickly), so downstream gates always have a document to read.

### Phase 5: Idempotency (re-run behavior)

If `DEPLOYMENT.md` already exists:
1. Read the existing file
2. **Preserve every filled-in decision** — never overwrite a human's answer
3. Append any newly found gaps to the relevant category section with new `<!-- REQUIRED -->` placeholders
4. **Enrich derived sections** now that more decisions are filled: complete the Resource & Naming Plan and Configuration Key Plan, and (re)run Phase 3 verification against the now-decided provider
5. Report: "Added N new gaps. M existing decisions preserved. Resource plan [derived/updated]. Platform context [verified/not verified]."

This makes the recommended workflow: run once → fill in the `DEP` decisions (at minimum) → optionally re-run to derive and read-only-verify the resource plan → fill remaining decisions → `/generate-modules`.

## Outputs

- `DEPLOYMENT.md` at project root (created or updated)

## Rules

- **Never write secrets** — only key names, sources, and strategy decisions
- **Read-only toward the platform** — never create/modify/delete any cloud resource; verification uses read-only calls only, and only when a CLI is already installed and authenticated
- **Never modify** `architecture/`, `src/`, `poc/`, or any file other than `DEPLOYMENT.md`
- **Architecture stays platform-agnostic** — platform specifics live in `DEPLOYMENT.md`, surfacing in module specs and code only where they change design
- **Skip questions the project already answers** — pre-fill with source instead
- **Vendor-agnostic** — no provider defaults; concrete names must trace to project documents or human answers
- **Idempotent** — re-running preserves human decisions, adds new gaps, enriches derived sections
- **Do not conflate with the POC path** — hard-stop if `poc/src/` exists

## Agents Used

| Agent | Purpose | Invocation |
|-------|---------|------------|
| `deployment-gap-analyzer-agent` | Non-interactive environment gap analysis | Phase 2 — produces structured gap list, pre-fills, already-determined items |

## Next Step

After filling in all `<!-- REQUIRED: ... -->` placeholders (optionally re-running `/plan-deployment` once the provider is decided, to derive and verify the resource plan read-only), run `/generate-modules`. The decisions in `DEPLOYMENT.md` then flow through module specs into `/generate-code`, which bootstraps the chosen dev environment (fully local / cloud / hybrid), implements and verifies every module against it — headless-browser E2E testing is mandatory for UI systems — and finalizes `CONFIG_GUIDE.md` as the as-built record. Production deployment follows later via the future `/deploy-to-prod`.
