# AI Agent Guidelines Framework

A battle-tested, reusable framework for configuring AI coding agents — informed by [LLM reasoning failure research](https://arxiv.org/abs/2602.06176) and practical experience shipping code with Claude, Claude Code, and Cursor in production workflows.

The core idea is simple: **AI agents are powerful first-draft generators, but they have systematic failure modes.** This framework encodes mitigations for those failure modes directly into the agent's instructions, so you get better outputs without having to remember the workarounds yourself.

---

## What This Solves

AI coding agents tend to:

- **Over-engineer** — adding abstractions, error handling, and flexibility nobody asked for
- **Skip verification** — saying "done" without proving the code actually works
- **Degrade in long sessions** — earlier context pollutes reasoning about newer tasks
- **Fail at composition** — handling individual sub-problems but botching multi-step reasoning
- **Confirm your biases** — agreeing with whatever framing you present rather than challenging it
- **Hallucinate arithmetic** — producing plausible-looking but wrong calculations

This framework addresses each of these with specific, actionable rules the agent follows automatically.

---

## Quick Start

1. Copy the framework files into your project root
2. Customise `AGENTS-PROJECT.md` with your commands, test locations, and domain keywords
3. Swap `AGENTS-REACT-TS.md` for your tech stack (or keep it if you're using React/TypeScript)
4. Start working with your AI agent — it picks up the guidelines automatically

That's it. No build step, no dependencies, no configuration beyond the markdown files.

---

## Tool Compatibility

| Tool | Auto-read file | Setup |
|------|---------------|-------|
| **Claude Code** | `CLAUDE.md` | Automatic — no setup needed |
| **Cursor** | `.cursorrules` | Included in this repo (mirrors CLAUDE.md) |
| **Windsurf** | `.windsurfrules` | Copy `.cursorrules` to `.windsurfrules` |

The entry point (`CLAUDE.md` / `.cursorrules`) directs the agent to read the framework files in the correct order.

---

## Framework Structure

```
.
├── CLAUDE.md              # Entry point — tool compatibility + reading order
├── .cursorrules           # Cursor mirror of CLAUDE.md
├── AGENTS.md              # Core guidelines (start here)
├── AGENTS-PROJECT.md      # Project-specific config (customise this)
├── AGENTS-REACT-TS.md     # Language/framework patterns (swap for your stack)
├── AGENTS-RLM.md          # Large context handling (>100K tokens)
└── skills/                # Reusable workflow scripts (invoked by command)
    ├── clarify/SKILL.md   # Structured requirements interview
    └── unstuck/SKILL.md   # Lateral thinking modes for stagnation
```

### Reading Order

The agent reads these in sequence. The first three sections of `AGENTS.md` are mandatory before any code gets written.

1. **[AGENTS.md](./AGENTS.md)** — Core behavioural guidelines, verification requirements, complexity triggers
2. **[AGENTS-PROJECT.md](./AGENTS-PROJECT.md)** — Your project's commands, test locations, keyword triggers
3. **[AGENTS-REACT-TS.md](./AGENTS-REACT-TS.md)** — Language and framework patterns
4. **[AGENTS-RLM.md](./AGENTS-RLM.md)** — Strategies for processing large contexts

---

## What's Inside

### AGENTS.md — The Core Framework

The heart of the system. Eleven sections covering everything from mandatory verification to workflow orchestration.

**Sections 1–3: Mandatory Requirements**

- **Verification Blocks** — Every code change must end with a structured block reporting: tests passed/failed, types checked, new tests added, and a confidence score (0.0–1.0). No more "I've updated the file" without proof.
- **Complexity Triggers** — Before starting any task, the agent checks whether the work touches 3+ files, requires architectural decisions, is ambiguous, or involves risk. If any trigger fires, it must decompose the problem before writing code.
- **Ambiguity Pre-check** — For Standard/Critical tasks, three questions must pass before coding: is the goal specific? Are constraints explicit? Are success criteria defined? If any answer is "no", clarify first.
- **Testing Requirements** — Tests are the source of truth. If a test fails, the *code* is wrong, not the test. Bug fixes require regression tests. New features require both unit and integration tests.

**Sections 4–7: Quality Controls**

- **Pre-Completion Checklist** — Eight items the agent must verify before claiming work is done, including "Would a staff engineer approve this?"
- **Confidence Calibration** — A defined scale from 0.0 to 1.0 with explicit actions at each level. Below 0.5 means stop and ask for clarification.
- **Meta-Cognitive Process** — For complex tasks: Decompose, Solve, Verify, Synthesise, Reflect.
- **Thinking Modes** — Six cognitive lenses (Contrarian, Simplifier, Researcher, Architect, Ontologist, Hacker) for when standard decomposition isn't cutting it. Each reframes the problem through a different perspective, with a routing heuristic to pick the right one.
- **Communication Style** — Concise, no filler, explicit about assumptions and risks.

**Section 8: Behavioural Guidelines (LLM Failure Mitigations)**

Fifteen subsections targeting specific, research-backed failure modes:

| Subsection | What It Prevents |
|---|---|
| 8.1 Think Before Coding | Hidden assumptions, silent interpretation choices |
| 8.2 Simplicity First | Over-engineering, premature abstraction |
| 8.3 Surgical Changes | Scope creep, unnecessary refactoring |
| 8.4 Goal-Driven Execution | Vague "it works" without verifiable criteria |
| 8.5 Session & Context Hygiene | Proactive interference from stale context |
| 8.6 Problem Framing Awareness | Anchoring bias, framing effects, reversal curse |
| 8.7 Never Trust Arithmetic | Mental maths errors in calculations |
| 8.8 Confirmation Bias | Rubber-stamping the user's assumptions |
| 8.9 Robustness & Consistency | Blind retries instead of restructuring |
| 8.10 DRY & SOLID Principles | Dogmatic application of patterns without pragmatism |
| 8.11 Error Handling | Swallowed errors, missing context, wrong catch level |
| 8.12 Security Practices | Hardcoded secrets, unsanitised input, SQL injection |
| 8.13 Documentation & Comments | Stale comments, TODO without tickets, commenting the "what" |
| 8.14 Logging & Observability | Free-text logs, logged secrets, wrong log levels |
| 8.15 Dependency & Type Awareness | Hallucinated APIs, unchecked method signatures |

**Sections 9–11: Workflow, Tasks & Skills**

- **Workflow Orchestration** — When to use plan mode, subagent strategies, autonomous execution rules, multi-agent coordination with human review gates.
- **Task Management** — Structured planning, progress tracking, and a self-improvement loop that captures lessons from mistakes.
- **Skills** — Reusable workflow scripts invoked by explicit command (`/unstuck`, `/clarify`). See below.

**Appendices**

- **A: RLM Patterns** — Quick reference for large context handling
- **B: Quick Reference** — Decision flow chart and a Known LLM Failure Modes table
- **C: Learnings System** — Institutional memory for documented mistakes
- **D: Project Documentation** — Guidelines for writing project knowledge docs

---

### Skills — Reusable Workflow Scripts

Skills are invoked by explicit command. They provide structured workflows for situations where general guidelines aren't enough.

| Command | What It Does |
|---------|-------------|
| `/clarify` | Runs a structured requirements interview when the task is ambiguous. The agent adopts an interviewer role, targets the biggest ambiguity, and presents options rather than open-ended questions. Ends with a bullet-point summary before proceeding. |
| `/unstuck` | Routes into a Thinking Mode based on the type of block. Repeated failures → Contrarian, too many options → Simplifier, missing info → Researcher, structural issues → Architect, analysis paralysis → Hacker, vague requirements → Ontologist. |

Skills live in `skills/*/SKILL.md` and are referenced from `AGENTS.md` §11. Add your own by creating a new directory under `skills/` with a `SKILL.md` file.

---

### AGENTS-PROJECT.md — Project Configuration

A template you fill in with your project's specifics:

- **Commands** — test, typecheck, build, and dev commands
- **Test locations** — where tests live in your directory structure
- **Keyword triggers** — domain keywords (e.g., "auth", "database", "api") mapped to relevant learnings
- **Task type categories** — quick lookup from task type to relevant past lessons
- **Verification block template** — pre-filled with your project's actual commands

---

### AGENTS-REACT-TS.md — Language Patterns

React, TypeScript, and Three.js patterns including:

- **Vitest testing** — assertions, mocking, async patterns, React Testing Library
- **React patterns** — component structure, hooks, state management (reducer/context)
- **Three.js / R3F** — shader development rules, performance guidelines, common mistakes
- **TypeScript** — strict null handling, type guards, discriminated unions
- **ESLint/Prettier** — common fixes

Swap this file for your stack. The framework is language-agnostic at its core.

---

### AGENTS-RLM.md — Large Context Handling

When context exceeds ~100K tokens, standard prompting degrades. This file provides structured strategies:

- **Probe, Filter, Process** — examine structure, filter programmatically, then reason
- **Chunk, Map, Reduce** — split into manageable pieces, process each, aggregate
- **Iterative Refinement** — build answers progressively with running buffers
- **Structural Decomposition** — split by document structure, summarise sections independently

Also covers context rot (why quality degrades with length), task complexity classification (O(1) vs O(N) vs O(N²)), cost-performance trade-offs, and common anti-patterns.

---

## Customising for Your Project

### Step 1: Fill in AGENTS-PROJECT.md

Replace the template values with your actual commands:

```markdown
## Project Commands
- **Test**: `npm test`
- **Typecheck**: `npx tsc --noEmit`
- **Build**: `npm run build`
```

Add your test locations, keyword triggers, and any project-specific rules.

### Step 2: Swap the Language File

If you're not using React/TypeScript, replace `AGENTS-REACT-TS.md` with a file for your stack:

- `AGENTS-CSHARP.md` for C# / .NET / Unity
- `AGENTS-PYTHON.md` for Python / Django / FastAPI
- `AGENTS-CPP.md` for C++ / Unreal

Update the include reference in `AGENTS.md` accordingly.

### Step 3: Build Your Learnings (Optional)

Create a `learnings/` directory with an `INDEX.md` file. As the agent makes mistakes and you correct them, document the patterns. The framework automatically checks these before starting new work, creating a feedback loop that improves over time.

---

## The Learnings System

One of the more useful bits of the framework. When the agent makes a mistake:

1. The mistake gets documented in `learnings/XXX-topic.md` with: what went wrong, why it's wrong, the correct process, red flags to watch for, and prevention steps
2. `learnings/INDEX.md` indexes these by task type and keyword
3. Before starting any new task, the agent checks the index for relevant past mistakes

This creates institutional memory. The agent stops making the same mistake twice — which, if you've worked with AI agents for any length of time, you'll appreciate is not their natural inclination.

---

## Known LLM Failure Modes

The framework is built around mitigating these documented failure patterns:

| Failure Mode | Risk Level | Framework Mitigation |
|---|---|---|
| Working memory overload | High in long sessions | Fresh conversations per topic (§8.5) |
| Anchoring / confirmation bias | High for decisions | Lead with problem, request counterarguments (§8.6, §8.8) |
| Compositional reasoning | High for multi-step tasks | Decompose into sequential steps (§2, §6) |
| Arithmetic errors | High for calculations | Always use code, never mental maths (§8.7) |
| Framing sensitivity | Medium | Restructure approach if output is off (§8.6) |
| Reversal curse | Medium in complex context | State relationships bidirectionally (§8.6) |
| Multi-agent coordination | High if parallelising | Human review gates, no auto-merge (§9) |

---

## References

- Song, Han, Goodman — ["Large Language Model Reasoning Failures"](https://arxiv.org/abs/2602.06176) (ICML 2025)

---

## Licence

Use it however you like. No attribution required, though it's appreciated.
