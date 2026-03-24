# Learning 053: Multi-Agent Worktree Merge Requires Shared Package Rebuild

**Severity:** High
**Sources:** TokenRaider (TR) — Recognition Service integration
**Category:** Multi-Agent, Build Pipeline

## Principle

When merging files from multiple git worktree agents into the main working directory, packages that use TypeScript project references (e.g., `"references": [{ "path": "../../packages/shared" }]`) will fail typecheck unless the shared package is **rebuilt** after merging. Each worktree agent has its own build artefacts; the main directory's build cache is stale.

## The Mistake

Three agents worked in isolated worktrees (server, controller, display). All three passed typecheck in their own worktrees. After merging their files into the main directory, the server failed typecheck with dozens of "has no exported member" errors — even though the shared package source was correct.

```
src/reducers.ts(5,3): error TS2305: Module '"@project/shared"' has no exported member 'ShipNavigation'.
src/reducers.ts(6,3): error TS2305: Module '"@project/shared"' has no exported member 'ShipHeading'.
// ... 40+ similar errors
```

## Why This Happened

TypeScript project references use a **build cache** (`dist/` or `tsconfig.tsbuildinfo`). The main directory's shared package build cache was from BEFORE the shared types were modified. Each worktree had its own fresh build. After copying files from worktrees to main, the server tried to resolve imports against the stale main build cache — which didn't include the new exports.

## The Correct Process

After merging worktree agent outputs into the main directory:

```bash
# 1. Rebuild the shared/foundation packages FIRST
pnpm --filter @yourproject/shared build

# 2. THEN run the full typecheck
pnpm typecheck
```

## Red Flags to Watch For

- "has no exported member" errors for types that clearly exist in the source
- Errors only in packages that depend on a shared package via project references
- Typecheck passes per-package (`--filter shared typecheck`) but fails when dependent packages check
- Post-merge typecheck failures that don't appear in any individual worktree

## Prevention

1. **Always rebuild foundation packages** after merging multi-agent worktree outputs. The order is: shared → server/client packages.
2. **Add a post-merge verification script** that does `pnpm -r build && pnpm typecheck` rather than just `pnpm typecheck`.
3. **Consider using `tsc --build --force`** to ignore stale build info after a multi-agent merge.
4. **In agent prompts**, mandate that the lead agent runs the full build chain (not just typecheck) after merging worktree branches.
