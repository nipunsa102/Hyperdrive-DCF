---
paths:
  - "**/*"
---

# Project Structure

## Root-Level Rules

**Allowed at root:**
- Project manifest and config files (e.g., `package.json`, `pyproject.toml`, `go.mod`, `tsconfig.json`)
- `.gitignore`, `.env*`
- Documentation: `CLAUDE.md`, `OVERVIEW.md`, `TECHSTACK.md`, `README.md`, `PRD.md`
- Generated DCF artifacts: `CONFIG_GUIDE.md`, `POC_PROMOTION_REPORT.md`

**NOT allowed at root** (must be in `src/`):
- Source code files (e.g., `.ts`, `.py`, `.go`, `.js`)
- Build/bundler configs (e.g., `vite.config.ts`, `webpack.config.js`)
- Test runner configs (e.g., `vitest.config.ts`, `pytest.ini`)

## Folder Structure

```
project-root/
├── .claude/                   # DCF Framework (IMMUTABLE)
│   ├── commands/
│   ├── agents/
│   ├── skills/
│   └── rules/                 # Project rules (auto-loaded)
├── architecture/              # Architecture docs
│   ├── architecture.md
│   ├── data-model.md          # Logical data model (entities, relationships)
│   └── modules/
├── tracking/                  # Project tracking
│   ├── module-tracking.md     # Module status log (maintained during module implementation)
│   ├── change-tracking.md     # Post-promotion PM change log (append-only)
│   └── env-setup.md           # Runtime environment setup log
├── src/                       # All source code
│   ├── app/                   # Frontend (configs colocated here)
│   ├── server/                # Backend
│   └── shared/                # Shared code
├── tests/
│   ├── unit/                  # L1: Coverage gate
│   ├── integration/           # L2: Blocking gate
│   ├── e2e/                   # L3: Non-blocking
│   └── reports/               # Generated test reports (gitignored)
├── poc/                       # POC output directory
│   ├── dependencies & configs # Self-contained (e.g., package.json, requirements.txt)
│   ├── entry point            # e.g., index.html, main.py
│   ├── architecture/          # Lightweight POC architecture
│   │   ├── architecture.md
│   │   └── modules/
│   ├── poc-tracking/          # POC change tracking (updated during POC modification, consumed during PRD sync)
│   │   ├── CHANGELOG.md                       # Structured technical changelog (consumed during PRD sync)
│   │   ├── change-tracking.md                 # PM-facing raw log (human reference)
│   │   └── changelog-Merged-<datetime>.md     # Archived changelogs after PRD sync
│   ├── temp/                  # Promotion temp files (writable during POC promotion)
│   │   └── poc_promotion/
│   │       ├── POC_PROMO_PREP.md   # Human questionnaire (produced during promotion preparation)
│   │       └── promotion.md        # Running promotion log (maintained during POC promotion)
│   └── src/                   # POC source (UI-focused, all mocks)
│       ├── mocks/             # Mock data files
│       ├── types/             # Shared types/models
│       ├── components/        # UI components
│       └── pages/             # Screen/page components
├── infra/                     # Infrastructure configs
└── scripts/                   # Build/deploy scripts
```

## Key Rules

1. **All code in `src/`** - No source files at root
2. **Colocate configs** - Build configs live with their target (`src/app/`, `src/server/`)
3. **IMMUTABLE** - Never modify `.claude/commands/`, `.claude/agents/`, `.claude/skills/`
4. **Source of truth** - `tracking/module-tracking.md` for module status; `tracking/change-tracking.md` is an append-only PM log of post-promotion change requests (no root-level `CHANGELOG.md` — the git branch is the technical changelog)
5. **POC isolation** - `poc/` is self-contained. POC code never imports from `src/`. Main `src/` never imports from `poc/`. POC has its own dependencies, configs, and architecture docs.
