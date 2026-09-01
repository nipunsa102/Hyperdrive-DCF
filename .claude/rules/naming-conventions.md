---
paths:
  - "src/**/*"
  - "tests/**/*"
  - "poc/**/*"
---

# Naming Conventions

Common patterns by language/framework — adapt to your stack.

## Files

| Type | Pattern | Example |
|------|---------|---------|
| Python module | `snake_case.py` | `user_service.py` |
| TypeScript module | `kebab-case.ts` | `user-service.ts` |
| Component (React/Vue/etc.) | `PascalCase.{ext}` | `UserDashboard.tsx` |
| Hook (if applicable) | `use{Name}.{ext}` | `useAuth.ts` |
| Stylesheet | `kebab-case.css` | `globals.css` |
| Test (Unit) | `{name}.test.{ext}` | `user-service.test.ts`, `test_user_service.py` |
| Test (Integration) | `integration.test.{ext}` | `tests/integration/integration.test.ts` (single file per project — see test-gates.md) |
| Config | `{tool}.config.{ext}` | `vite.config.ts`, `pytest.ini` |
| Docs | `kebab-case.md` | `naming-conventions.md` |

## Code Identifiers

| Language | Element | Pattern | Example |
|----------|---------|---------|---------|
| Python | Class | `PascalCase` | `UserService` |
| Python | Function/Variable | `snake_case` | `get_user_by_id` |
| Python | Constant | `UPPER_SNAKE_CASE` | `MAX_RETRIES` |
| Python | Private | `_leading_underscore` | `_validate` |
| TypeScript | Class/Interface/Type | `PascalCase` | `UserService` |
| TypeScript | Function/Variable | `camelCase` | `getUserById` |
| TypeScript | Constant | `UPPER_SNAKE_CASE` | `MAX_RETRIES` |
| Component Framework (if applicable) | Component | `PascalCase` | `UserDashboard` |
| Component Framework (if applicable) | Props/Params | `{Component}Props` | `UserDashboardProps` |
| Component Framework (if applicable) | Handler | `handle{Event}` | `handleSubmit` |

## API & Database

| Element | Pattern | Example |
|---------|---------|---------|
| REST endpoint | `/api/{resources}` | `/api/users`, `/api/users/:id` |
| Query params | `camelCase` | `?sortBy=createdAt` |
| DB table | `snake_case` (plural) | `user_tasks` |
| DB column | `snake_case` | `created_at` |

## Git

| Type | Pattern | Example |
|------|---------|---------|
| Feature branch | `feature/{ticket}-{desc}` | `feature/PROJ-123-auth` |
| Bugfix branch | `bugfix/{ticket}-{desc}` | `bugfix/PROJ-456-login` |
| Commit | `{type}({scope}): {msg}` | `feat(auth): add login` |

Commit types: `feat`, `fix`, `docs`, `test`, `refactor`, `chore`
