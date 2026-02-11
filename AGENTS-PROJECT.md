# Project Configuration

> Project-specific configuration for the AI Agent Guidelines.
> Referenced by `AGENTS.md`. Replace placeholder values with your project's actual commands, paths, and keywords.

---

## Project Commands

```bash
# Run all tests
pnpm test -- --run

# Run specific package tests (monorepo)
pnpm --filter @yourproject/shared test
pnpm --filter @yourproject/server test

# Type checking
pnpm typecheck

# Production build
pnpm build

# Development mode
pnpm dev
```

---

## Test Locations

| Path | Purpose |
|------|---------|
| `packages/shared/src/__tests__/` | Shared types and utilities |
| `apps/server/src/__tests__/` | Server logic and API handlers |
| `apps/client/src/__tests__/` | Client components and hooks |

---

## Existing Test Files

| File | Purpose |
|------|---------|
| `example.test.ts` | Example unit tests |
| `example.integration.test.ts` | Example integration tests |

---

## Keyword Triggers

Map task keywords to learning document numbers from your `learnings/` folder. See AGENTS.md Appendix C.

| Keywords | Learnings |
|----------|-----------|
| auth, login, session, token | 001, 002 |
| database, migration, schema | 003 |
| api, endpoint, route | 004, 005 |
| test, spec, expect | 006 |

---

## Task Type Categories

| Task | Learnings |
|------|-----------|
| Authentication | 001, 002 |
| Database | 003 |
| API Development | 004, 005 |
| Testing | 006 |

---

## Learnings Summary

Current count: **0 documented learnings**

See [`learnings/INDEX.md`](./learnings/INDEX.md) for the complete categorized list.

> Create a `learnings/` directory with an `INDEX.md` inside it. Add learnings as you discover project-specific gotchas — see AGENTS.md Appendix C for the format.
