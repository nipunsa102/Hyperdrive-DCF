---
paths:
  - "architecture/**/*"
  - "poc/architecture/**/*"
  - "src/**/*"
  - "poc/src/**/*"
---

# Architecture Documentation Standard

The architecture documents are a **statement of present truth** about the system — not a history
of changes (that is the changelog's job) and not a substitute for reading code. Their purpose is
to stop design drift: a future agent that "simplifies" a three-call pipeline back to one call, or
"optimizes" PNG tiles to JPEG, does so only because nothing told it why things are the way they
are.

## The Two Altitudes

| File | Altitude | Audience test |
|---|---|---|
| `architecture/architecture.md` (and `poc/architecture/architecture.md`) | **The full picture, visually presented.** Box diagrams, UML-style sequence diagrams, budgets & limits tables, screen layouts, flow diagrams. | A human understands the whole system — and an AI gets the overarching idea — **without opening a module spec**. |
| `architecture/modules/*.md` (and the poc equivalents) | **The implementation contract.** API specifications, data contracts, pseudocode, state machines, decision-evidence tables, edge semantics. | An implementer can build the module from it. |

**The litmus test for placement:** *understanding-oriented → `architecture.md`;
implementation-oriented → the owning module spec.* This is decided by **purpose, not diagram
type** — a sequence diagram that explains how the system fundamentally behaves (e.g. a
fire-and-poll flow) belongs in `architecture.md`; a sequence diagram that specifies retry counts
and header names belongs in the module. Some overlap is fine; unexplained divergence is not.

## What MUST Be Documented

Whenever the system contains one of these, the docs must carry it at both altitudes (picture in
`architecture.md`, contract in the owning module spec):

1. **Pipelines and algorithms** — anything with stages, ordering, or non-obvious logic.
   Picture: box diagram of the stages and what flows between them.
   Contract: pseudocode + the reply/data contracts.
2. **Multi-actor flows** — any behavior spanning browser / API / worker / external service.
   Picture: UML-style sequence diagram.
3. **Budgets and limits** — timeouts, memory, payload caps, token budgets, rate/count limits,
   polling intervals. These are architecture, not trivia: they shape designs and their violation
   is how systems fail silently. Picture: a limits table with the *consequence* of each limit.
4. **Failure and recovery semantics** — claims, staleness windows, idempotency, ownership guards,
   what happens when a process is killed. Contract: a state-machine diagram in the module spec.
5. **Decision rationale with evidence** — every non-obvious choice gets a row in a
   *Decisions and their evidence* table in the owning module spec: the decision, why, the
   measurement or incident that proves it, and the REQ/CT ref. The WHY is the part code
   cannot carry.

## Format

- ASCII box, sequence, and state diagrams inside fenced code blocks. **No mermaid.** ASCII is
  diffable in git, renders in every terminal and viewer, and never needs a toolchain.
- Diagrams carry real names (component, endpoint, column, testid) — never `Foo`/`Bar`.
- Tables for coverage matrices, limits, and decision evidence.

## Placement Across POC and Root

- A decision made **during POC creation or modification** is documented in detail in
  `poc/architecture/` at both altitudes. Root `architecture/` is not touched by POC work.
- At promotion, `re-architect-agent` merges the POC's decision documentation **meaningfully** into
  the root docs — diagrams, evidence tables, and limits come across, not just requirement rows.
- Derived POC docs do **not** duplicate root diagrams for parts the POC did not change — a pointer
  is enough. If the POC changed the mechanism, the POC doc carries the changed diagram.

## Sync Discipline

- **Code is ground truth for WHAT; docs add WHY.** On a mismatch, do not silently rewrite either:
  a doc-vs-code conflict may be a code bug. Update the doc when the code is the accepted reality
  (e.g. a shipped, verified change); flag to the human when the code appears to have drifted from
  an approved design.
- The changelog records *what changed and why it changed*; the architecture records *what is and
  why it is*. Never point a reader at the changelog to learn the current design.
- Enforcement is the `architecture-alignment-agent` (see its charter), invoked by the generate,
  modify, and promote flows — including `-fix` mode, because a fix that changes call topology,
  algorithms, limits, or failure semantics **is a design change wearing a fix's clothes**.
- Any agent that implements code and knowingly departs from the module spec or architecture MUST
  list that departure in its final report ("Deviation Report") so the alignment agent can
  reconcile the docs.
