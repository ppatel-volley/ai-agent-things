# AI Agent Instructions

> This file is automatically read by **Claude Code** at startup.
> For other tools, see the compatibility table below.

## Tool Compatibility

| Tool | Auto-read file | Setup |
|------|---------------|-------|
| Claude Code | `CLAUDE.md` (this file) | Automatic — no setup needed |
| Cursor | `.cursorrules` | Included in this repo (mirrors this file) |
| Windsurf | `.windsurfrules` | Copy `.cursorrules` to `.windsurfrules` |

## Instructions

Read and follow the guidelines in these files before starting any task:

1. **[AGENTS.md](./AGENTS.md)** — Core behavioral guidelines, verification requirements, and complexity triggers *(start here — sections 1–3 are mandatory, section 11 defines available skills)*
2. **[AGENTS-PROJECT.md](./AGENTS-PROJECT.md)** — Project-specific commands, test locations, and keyword triggers
3. **[AGENTS-REACT-TS.md](./AGENTS-REACT-TS.md)** — Language and framework patterns (swap this file for your stack)

These files contain mandatory requirements for code changes, testing, and verification. Read sections 1–3 and section 11 (Skills) of AGENTS.md before writing any code. Skills (`/unstuck`, `/clarify`, `/review`) are reusable workflows in `skills/*/SKILL.md` — read §11 so you know when to invoke them.

**Conditional reading (load only when needed):**
- **[AGENTS-THREEJS.md](./AGENTS-THREEJS.md)** — Read ONLY for 3D/WebGL tasks using Three.js, shaders, or React Three Fiber.
- **[AGENTS-RLM.md](./AGENTS-RLM.md)** — Read ONLY when context exceeds ~100K tokens or task requires processing most/all of a large input. Not needed for typical tasks.

## Maintenance

**README sync:** When adding or removing features from any agent file (`AGENTS.md`, `AGENTS-PROJECT.md`, skills, etc.), update `README.md` to reflect the change. The README is the public-facing documentation — it must stay current with the framework's actual capabilities.
