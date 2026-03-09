# Ouroboros Import Report

> **What useful elements from `/Users/pratik/dev/ouroboros` could strengthen the agent guidelines in this repo?**
>
> **Last updated:** 2026-03-09 — ouroboros v0.20.0

## Executive Summary

Ouroboros is a specification-first AI workflow system built around an evolutionary loop: **Interview -> Seed -> Execute -> Evaluate -> Evolve**. Its agent guidelines focus on *what happens before and after code* — structured requirements elicitation, ontological analysis, multi-stage evaluation, and drift detection. This repo (ai-agent-things) focuses primarily on *what happens during code* — execution modes, verification, confidence calibration, and LLM failure mitigation.

The two are highly complementary. The biggest wins come from importing ouroboros' **pre-coding discipline** (Socratic interviewing, ambiguity scoring) and **cognitive mode personas** (thinking strategies that work in single-agent or multi-agent contexts).

### Changes Since Previous Report

Ouroboros has matured significantly. Key changes:

| Area | What Changed |
|------|-------------|
| **Version** | Now v0.20.0 with consistent versioning across plugin.json, pyproject.toml |
| **Skills** | Grew from 4 referenced skills to **13**: added `welcome`, `seed`, `run`, `evolve`, `ralph`, `setup`, `tutorial`, `help`, `update` |
| **Ralph loop** | New persistent self-referential execution loop with QA verdict integration, parallel execution, and git branch detection (see new Import #10 below) |
| **Plugin system** | Full `.claude-plugin/` packaging: `plugin.json`, `marketplace.json`, `.mcp.json` — marketplace-ready |
| **Hooks** | Added `SessionStart` hook (`session-start.py`) alongside existing keyword-detector and drift-monitor |
| **Git workflow detection** | CLAUDE.md now parses for git/PR workflows; `ralph` and `run` skills adapt behaviour for PR-based projects |
| **Agent SDK** | New dependency: `claude-agent-sdk>=0.1.0` (Anthropic's Agent SDK) |
| **Stagnation** | `evolve` skill now has explicit Wonder→Reflect stagnation loop breaking (validates Import #4) |
| **Ambiguity default** | Default ambiguity score lowered from threshold 0.2 to default 0.15, documented in seed skill |
| **Self-update** | `ooo update` command with PyPI version check and SSL fallback |
| **TUI monitoring** | Real-time Textual-based monitoring with alias support |

---

## High-Value Imports

### 1. Specialised Cognitive Modes (Agent Personas)

**What ouroboros has:** 10 agent roles (9 listed in CLAUDE.md + qa-judge), each embodying a distinct reasoning strategy — not just job titles but explicit thinking methodologies with prescribed questions, boundaries, and output formats.

**Why it matters:** ai-agent-things has multi-agent orchestration (worktrees, task coordination, integration verification) but no defined *cognitive strategies* for agents to adopt. The infrastructure is there; the thinking modes are missing.

**Most valuable personas to import:**

| Persona | Core Question | Value for This Repo |
|---------|--------------|---------------------|
| **Contrarian** | "What if the opposite is true?" | Forces examination of assumptions before committing to an approach. Directly addresses confirmation bias (currently S8.8) but as an active process, not just a warning |
| **Simplifier** | "What can we remove?" | Operationalises the "simplicity first" guideline (S8.2) with concrete heuristics: challenge every component, find minimum viable solution, target 50%+ reduction |
| **Researcher** | "What information are we missing?" | Strengthens "read before you write" (S8.1) into a full investigative protocol: define unknowns, gather evidence systematically, form hypothesis |
| **Architect** | "Are we fighting the architecture?" | Provides structured diagnosis for when surgical changes aren't enough — detect structural symptoms, map misalignment, propose minimal restructuring |
| **Ontologist** | "What IS this, really?" | Deepens "restate requirement" (S8.6) into four fundamental questions: essence, root cause, prerequisites, hidden assumptions |
| **Hacker** | "What constraints are actually required?" | Useful when blocked — question each constraint, look for bypasses, consider solving a simpler adjacent problem |

**Suggested integration:** Add these as a "Thinking Modes" section or appendix. When complexity triggers fire or confidence drops below 0.7, the agent could explicitly adopt a relevant mode rather than just "decomposing" generically. Could also be assigned as specialist roles in multi-agent Critical-mode tasks. The extra pattern worth copying from `skills/unstuck/SKILL.md` is the lightweight router: repeated failures -> `contrarian`, too many options -> `simplifier`, missing information -> `researcher`, structural issue -> `architect`.

**Source files:** `agents/*.md` plus `skills/unstuck/SKILL.md`

---

### 2. Structured Requirements Elicitation (Socratic Pre-work)

**What ouroboros has:** A formal interview phase before any implementation begins. The Socratic Interviewer agent asks probing questions, each response ending with a follow-up question, specifically targeting the biggest ambiguity. Strict boundaries: ONLY gathers information, never implements.

**Why it matters:** ai-agent-things' complexity trigger #3 says "Request is ambiguous -> Clarify" but provides no method for *how* to clarify. The current guidance amounts to "ask for clarification" — ouroboros provides a disciplined protocol for doing so.

**Key elements worth importing:**
- Always end clarification with a specific follow-up question (not just "let me know if you have questions")
- Target the single biggest ambiguity, not scatter-shot questions
- Maintain a strict information-gathering boundary — don't start solving while still clarifying
- Use ontological questions: "What are you assuming?" / "What must exist first?"
- When the question benefits from structured choices, present 2-3 likely answers instead of a blank prompt; ouroboros does this in `skills/interview/SKILL.md` and it maps well to Cursor's `AskQuestion` tool

**Suggested integration:** Enhance complexity trigger #3 and the "When Blocked" section with a structured clarification protocol. 3-5 bullet points would suffice; no need for the full Socratic Interviewer role. If the repo later adds project-specific workflows, this could become a dedicated interview skill rather than just a prose guideline.

---

### 3. Ambiguity Scoring (Pre-coding Gate)

**What ouroboros has:** A quantified measure of requirement clarity that determines readiness to proceed:

```
Ambiguity = 1 - Sum(clarity_i * weight_i)
```

**Dimensions:** Goal Clarity (40%), Constraint Clarity (30%), Success Criteria (30%). Threshold: Ambiguity <= 0.2 (i.e., 80% weighted clarity) before proceeding.

**Why it matters:** ai-agent-things has confidence calibration, but it measures confidence in the *solution*. Ambiguity scoring measures clarity of the *problem*. These are orthogonal — you can be confident in a solution to the wrong problem. Together they create two gates: "Do we understand what to build?" and "Did we build it correctly?"

**Suggested integration:** Add as a lightweight pre-check for Standard/Critical modes. Doesn't need the full mathematical formula — even a qualitative checklist would help:
- Is the goal specific and measurable? (not "make it better")
- Are constraints explicit? (performance, compatibility, scope)
- Are success criteria defined? (how do we know when it's done?)

If any answer is "no" for Standard/Critical, clarify before coding.

---

### 4. Stagnation & Oscillation Detection

**What ouroboros has:** Explicit detection of pathological loop patterns:
- **Stagnation:** Same approach tried 3+ times with no progress
- **Oscillation:** Solution N resembles solution N-2 (flip-flopping between two approaches)
- **Repetitive feedback:** 70%+ overlap in error messages across attempts
- **Hard cap:** 30 generation maximum as a safety valve

**Why it matters:** ai-agent-things has "max 4 retries, then escalate" (S5 Recovery Path) but doesn't detect *oscillation* — the common failure mode where an agent alternates between two broken approaches indefinitely, each "fixing" what the other broke. This is distinct from simple retry failure.

**Suggested integration:** Enhance the Recovery Path section:
- After 2 failed attempts, explicitly check: "Am I reverting changes from the previous attempt?" If yes, you're oscillating — step back and reframe the problem.
- After 3 failed attempts with similar error messages, adopt Researcher mode: stop coding, investigate root cause.
- The hard cap concept (absolute maximum attempts) is also worth adding.

---

### 5. Multi-Stage Evaluation Pipeline

**What ouroboros has:** Three escalating evaluation stages:
1. **Mechanical ($0):** Automated checks — lint, build, test, static analysis, coverage
2. **Semantic:** Does the output actually satisfy acceptance criteria? Score against evaluation principles (threshold: 0.8)
3. **Consensus (triggered):** Multi-model deliberation with Proposer, Devil's Advocate, and Synthesiser roles

Stage 3 only triggers when: Stage 2 < 0.8, high ambiguity detected, or manual request.

**Why it matters:** ai-agent-things' verification block is essentially Stage 1 only (tests pass/fail, types pass/fail, build pass/fail). The semantic evaluation layer — "does this actually do what was asked?" — is implicit in the confidence score but not structured. The escalating cost model (cheap checks first, expensive checks only when needed) is also elegant.

**Suggested integration:** Add a "Semantic Verification" step to the verification block for Standard/Critical modes:
- After mechanical checks pass, explicitly verify: does the change satisfy the original request? (Not just "does it compile?")
- For Critical mode, consider a structured self-review with devil's advocate questioning before marking complete.

---

### 6. Drift Monitoring via Claude Code Hooks

**What ouroboros has:** Three hook points configured in `.claude/settings.json` and `hooks/hooks.json`:
- `SessionStart` hook: runs `session-start.py` to initialise session state
- `UserPromptSubmit` hook: runs `keyword-detector.py` for `ooo` command routing (5s timeout)
- `PostToolUse` hook on `Write|Edit` matcher: runs `drift-monitor.py` to check whether file changes drift from the specification (3s timeout)

Each hook is a lightweight Python script with a hard timeout — they fire automatically and fail silently if they exceed the limit.

**Why it matters:** ai-agent-things has no hooks. This is a practical enforcement mechanism — rather than relying on the agent to self-police (which degrades over long sessions due to context rot), hooks provide external guardrails that fire automatically.

**Suggested integration:** Consider adding hooks for:
- Session start: log session context, load relevant memory files
- Post-edit check: "Does this change match the task described in tasks/todo.md?"
- Pre-commit check: "Did verification block pass?"
- These would be project-specific, so perhaps document the *pattern* in AGENTS.md and let AGENTS-PROJECT.md define specific hooks.

---

### Verified Additional Pattern: Command-Routed Skill Wrappers

**What ouroboros has:** `CLAUDE.md` acts as a command router (`ooo interview`, `ooo evaluate`, `ooo unstuck`, `ooo status`, etc.) and delegates each command to a focused `skills/*/SKILL.md` file. Those skill files define:
- trigger keywords
- when to prefer MCP vs. prompt-only fallback
- the exact agent or tool to use
- response structure and next-step guidance

**Why it matters:** ai-agent-things already has strong general guidance, but it is mostly passive documentation. Ouroboros turns guidance into *entry points* that are easier to invoke consistently. This is especially useful for recurring workflows like clarification, drift checks, and "I'm stuck" recovery.

**Suggested integration:** Don't copy the whole command surface, but do copy the pattern. A small set of named workflow wrappers for "clarify", "unstuck", and "evaluate" would make the current guidance easier to operationalize than keeping everything inside one giant handbook.

---

### NEW: 10. Self-Referential QA Loop (Ralph Pattern)

**What ouroboros has:** A persistent execution loop (`skills/ralph/SKILL.md`) that:
1. Executes a task
2. Runs the QA Judge agent against the output
3. If the QA verdict is below threshold, feeds the verdict back as input and re-executes
4. Supports parallel execution of independent sub-tasks
5. Detects git branch context and adapts (PR-based projects get branch-aware behaviour)
6. Has explicit rewind capability (`ralph-rewind.py`) for rolling back failed iterations

The QA Judge (`agents/qa-judge.md`) scores on 5 dimensions: Correctness, Completeness, Quality, Intent Alignment, and Domain-Specific criteria. Output is a structured verdict with score, differences, suggestions, and loop action (continue/stop/rewind).

**Why it matters:** ai-agent-things has verification (tests pass/fail) but no *iterative self-correction loop*. The Ralph pattern is essentially: "run, judge, fix, repeat" with structured feedback between iterations. This is more sophisticated than "retry on failure" — the QA verdict tells the next iteration *what specifically* to fix.

**Suggested integration:** This is heavier than the other imports — it requires a QA judge definition and a loop controller. Worth documenting as a pattern for Critical-mode tasks:
- After implementing, run a structured self-review with explicit scoring dimensions
- If score < threshold, iterate with the review as context (not just "try again")
- Cap iterations (ouroboros uses the 30-generation hard cap from the evolve skill)
- The 5-dimension scoring framework (Correctness, Completeness, Quality, Intent Alignment, Domain-Specific) could be added to the Semantic Verification step (Import #5)

**Source files:** `skills/ralph/SKILL.md`, `agents/qa-judge.md`, `scripts/ralph.py`, `scripts/ralph-rewind.py`

---

### NEW: 11. Git Workflow Detection

**What ouroboros has:** CLAUDE.md parsing logic that detects whether a project uses git/PR workflows, and adapts skill behaviour accordingly. The `ralph` and `run` skills check for branch context and adjust — e.g. creating feature branches, structuring commits for PR review.

**Why it matters:** ai-agent-things has multi-agent coordination via worktrees but doesn't auto-detect the project's git workflow conventions. Adding detection (main-only vs feature-branch vs PR-based) would let the guidelines adapt automatically.

**Suggested integration:** Low effort — add a note to AGENTS-PROJECT.md suggesting projects declare their git workflow, and let agents adapt commit/branch behaviour accordingly.

---

### NEW: 12. Skill-as-Plugin Distribution Pattern

**What ouroboros has:** A full plugin packaging system in `.claude-plugin/`:
- `plugin.json`: name, version, description, skill directory path, MCP server reference
- `marketplace.json`: marketplace schema compliance with tags and categories
- `.mcp.json`: simplified MCP server config for installed plugin context

Skills are self-contained in `skills/{name}/SKILL.md` with consistent structure: trigger keywords, MCP vs fallback mode, agent/tool to use, response format, next-step guidance.

**Why it matters:** ai-agent-things already has skills (clarify, unstuck, simplify) defined in AGENTS.md. Ouroboros shows how to package these for reuse across projects. The pattern is: each skill is a standalone file with a consistent interface, discovered by a router (CLAUDE.md), and distributable via a plugin manifest.

**Suggested integration:** Medium effort. If the skills in ai-agent-things grow beyond 5-6, consider extracting them into `skills/{name}/SKILL.md` files with a manifest. For now, documenting the pattern is sufficient — the current inline approach works at small scale.

---

## Medium-Value Imports

### 7. Result Type Pattern (Error Handling)

**What ouroboros has:** `Result[T, E]` return types instead of thrown exceptions for expected failures. Exceptions reserved for truly unexpected errors. Retry logic separated from Result conversion.

**Relevance:** ai-agent-things' S8.11 covers error handling well but doesn't prescribe this pattern. For TypeScript, discriminated unions already serve this purpose (`{ success: true; data: T } | { success: false; error: string }` is in AGENTS-REACT-TS.md). Worth reinforcing the connection — the discriminated union pattern IS the Result type pattern for TypeScript.

### 8. Dependency Layering Model

**What ouroboros has:** Strict directional dependency rules: CLI -> Application -> Domain -> Infrastructure. Lower layers never import upper layers. Domain phases never import each other directly.

**Relevance:** ai-agent-things has SOLID principles (S8.10) including Dependency Inversion, but no explicit layering model. For larger projects, a layering diagram would complement the existing guidelines. Worth mentioning as a pattern to adopt for projects that grow beyond a certain size.

### 9. PAL Router (Tiered Model Selection)

**What ouroboros has:** Three-tier model routing — Frugal (Haiku, 1x cost) -> Standard (Sonnet, 10x) -> Frontier (Opus, 30x) — with auto-escalation on failure and auto-downgrade on success.

**Relevance:** Maps conceptually to ai-agent-things' Quick/Standard/Critical modes. The interesting addition is the *automatic escalation* — if a Quick-mode approach fails, upgrade to Standard rather than just retrying. Currently, mode selection is manual. Could add guidance: "If Quick-mode attempt fails twice, consider upgrading to Standard."

---

## Verified Caveats

- ~~`ouroboros/CLAUDE.md` is directionally accurate, but at least one command reference appears stale: it mentions `skills/qa/SKILL.md`, while that file is not present in the checked-out repo.~~ **Update (v0.20.0):** CLAUDE.md has been overhauled. The stale `skills/qa/SKILL.md` reference appears to have been removed. The command routing table now maps to 13 verified skill files. Import the pattern, not the exact file list — but the list is now reliable.
- The repo contains both top-level `agents/*.md` files and mirrored `src/ouroboros/agents/*.md` files. For this report, I treated the top-level `agents/` files as the human-facing source of truth because that is what `CLAUDE.md` points to.
- **New caveat:** ouroboros now depends on `claude-agent-sdk>=0.1.0` (Anthropic's Agent SDK). This suggests the project may be moving towards programmatic agent orchestration beyond prompt-based workflows. The agent definitions in `agents/*.md` may increasingly serve as documentation for SDK-driven agents rather than prompt-only personas. Monitor this — if the SDK becomes the primary execution path, the personas become less directly importable as prompt instructions.

---

## Low-Value / Not Transferable

These are ouroboros-specific and don't generalise well:

| Element | Why Not Import |
|---------|---------------|
| Event Sourcing architecture | Domain-specific to ouroboros' evolutionary loop |
| Python 3.14+ / async-I/O rules | Wrong language stack |
| Plugin system / MCP server config | Now a full `.claude-plugin/` packaging system with marketplace.json — more mature than before, but still specific to ouroboros' distribution model. The *pattern* of skill packaging is worth noting (see Import #12) even if the implementation isn't transferable |
| Seed specification format (YAML) | Specific to ouroboros' interview-to-spec pipeline |
| Ontology convergence formula | Only relevant for evolutionary/iterative spec refinement |
| Double Diamond design process | Interesting but heavyweight; the useful parts (diverge then converge) are simpler to express as a guideline |
| `contextvars` / structlog patterns | Python-specific |

---

## What This Repo Already Does Better

For completeness — areas where ai-agent-things is stronger and ouroboros has nothing to offer:

| Area | ai-agent-things Strength |
|------|-------------------------|
| **RLM patterns** | Entire dedicated file (AGENTS-RLM.md) for handling large contexts. Ouroboros has nothing comparable |
| **LLM failure modes catalogue** | Comprehensive table with 10 documented failure modes and mitigations. Ouroboros doesn't address this |
| **Confidence calibration** | Research-backed scoring with cross-dependency penalties. Ouroboros uses ambiguity scoring for requirements but not for solution confidence |
| **Multi-agent coordination mechanics** | Worktrees, file edit conflicts with backoff, integration verification. Ouroboros mentions agents but doesn't address coordination |
| **Behavioural guidelines depth** | 15 detailed sections covering security, logging, documentation, arithmetic, framing bias, etc. Ouroboros focuses on architecture over behaviour |
| **Stack-specific guidance** | React/TS and Three.js patterns. Ouroboros is Python-only |
| **Surgical change discipline** | Explicit "touch only what you must" with concrete tests. Ouroboros is more about building new things than modifying existing code |

---

## Recommended Implementation Priority

If importing, I'd suggest this order:

1. **Cognitive Modes** (High impact, moderate effort) — Add a "Thinking Modes" section to AGENTS.md with the 6 personas listed above. These immediately improve how the agent approaches problems.

2. **Structured Clarification Protocol** (High impact, low effort) — 5 bullet points enhancing complexity trigger #3. Quick win.

3. **Ambiguity Scoring / Pre-coding Gate** (High impact, low effort) — 3-question checklist for Standard/Critical modes. Another quick win.

4. **Stagnation/Oscillation Detection** (Medium impact, low effort) — 3 bullet points in the Recovery Path section. *Note: ouroboros v0.20.0 now has explicit Wonder→Reflect stagnation breaking in the evolve skill, validating this recommendation.*

5. **Semantic Verification Step** (Medium impact, low effort) — One additional line in the verification block for Standard/Critical. Consider adding the QA Judge's 5-dimension scoring framework (Correctness, Completeness, Quality, Intent Alignment, Domain-Specific).

6. **Hooks Pattern** (Medium impact, moderate effort) — Document the pattern; actual hooks are project-specific. Ouroboros now has 3 concrete hook examples (session-start, keyword-detector, drift-monitor) with timeouts — good reference implementations.

7. **Named Workflow Wrappers** (Medium impact, moderate effort) — Optional after the core handbook changes. Add a small set of entry points for repeated flows like clarification, drift checks, and recovery when stuck. Ouroboros now has 13 skills showing the full pattern.

8. **Self-Referential QA Loop** (Medium impact, high effort) — For Critical-mode tasks, add an iterative self-correction pattern based on the Ralph loop: execute → judge → fix → repeat with structured feedback. Most valuable for complex, multi-file changes.

Items 1-5 could be done in a single editing pass to AGENTS.md. Items 6-8 are follow-on operational improvements. Total addition for the handbook-only changes: roughly 80-120 lines.

---

## Source File Reference

Key ouroboros files for reference during import (updated for v0.20.0):

### Agents (10 files in `agents/`)

| File | Content |
|------|---------|
| `agents/contrarian.md` | Assumption-challenging reasoning mode |
| `agents/simplifier.md` | Complexity reduction heuristics |
| `agents/researcher.md` | Evidence-based investigation protocol |
| `agents/architect.md` | Structural diagnosis and redesign |
| `agents/ontologist.md` | Fundamental nature analysis (4 questions: essence, root cause, prerequisites, hidden assumptions) |
| `agents/hacker.md` | Constraint questioning and bypass strategies |
| `agents/socratic-interviewer.md` | Requirements elicitation protocol (strict tool boundaries: CAN Read/Glob/Grep, CANNOT Write/Edit/Bash) |
| `agents/seed-architect.md` | Interview → Seed spec transformer (YAML output, ambiguity validation) |
| `agents/evaluator.md` | 3-stage evaluation pipeline (Mechanical → Semantic → Consensus) |
| `agents/qa-judge.md` | 5-dimension quality assessment (Correctness, Completeness, Quality, Intent Alignment, Domain-Specific) |

### Skills (13 files in `skills/*/SKILL.md`)

| File | Content |
|------|---------|
| `skills/welcome/SKILL.md` | First-touch onboarding, persona detection, MCP check |
| `skills/interview/SKILL.md` | Socratic interview with version check, MCP/fallback modes |
| `skills/seed/SKILL.md` | Generate Seed specs from interview, ambiguity validation, star question |
| `skills/run/SKILL.md` | Execute Seed through workflow engine, git workflow detection |
| `skills/evaluate/SKILL.md` | 3-stage evaluation wrapper, MCP tool invocation |
| `skills/evolve/SKILL.md` | Evolutionary loop with Wonder/Reflect, convergence detection, 30-gen cap |
| `skills/ralph/SKILL.md` | Persistent QA loop with parallel execution, git branch detection, rewind |
| `skills/unstuck/SKILL.md` | 5-persona lateral thinking router (hacker, researcher, simplifier, architect, contrarian) |
| `skills/status/SKILL.md` | Session status + drift measurement (0.3 threshold) |
| `skills/setup/SKILL.md` | MCP registration, CLAUDE.md integration, 6-step wizard |
| `skills/tutorial/SKILL.md` | Interactive hands-on learning, 6 phases, progressive disclosure |
| `skills/help/SKILL.md` | Command reference, natural language triggers, agent catalogue |
| `skills/update/SKILL.md` | PyPI version check, automatic plugin/package upgrade |

### Infrastructure

| File | Content |
|------|---------|
| `CLAUDE.md` | Command routing table, agent references, development mode docs (v0.20.0) |
| `README.md` | Ambiguity scoring formula, ontology convergence, 9-agent framework docs |
| `.claude/settings.json` | Hook configuration (UserPromptSubmit + PostToolUse with timeouts) |
| `hooks/hooks.json` | 3 hook definitions (SessionStart, UserPromptSubmit, PostToolUse) |
| `scripts/keyword-detector.py` | `ooo` command routing from user input |
| `scripts/drift-monitor.py` | Goal drift detection on Write/Edit operations |
| `scripts/session-start.py` | Session state initialisation |
| `scripts/ralph.py` | Ralph loop orchestration |
| `scripts/ralph-rewind.py` | Ralph iteration rollback |
| `.claude-plugin/plugin.json` | Plugin metadata and skill directory mapping |
| `.claude-plugin/marketplace.json` | Marketplace schema with tags and categories |
| `pyproject.toml` | Dependencies including `claude-agent-sdk>=0.1.0`, `litellm>=1.80.0` |
