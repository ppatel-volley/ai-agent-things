# Learning 051: useEffect Unmount Cleanup — The cleanupRef Pattern

**Severity:** Critical
**Sources:** TokenRaider (TR) — Recognition Service integration
**Category:** React Patterns

## Principle

A `useEffect(() => { return () => cleanup(); }, [])` with an empty dependency array captures `cleanup` at mount time. If `cleanup` is a `useCallback` that gets recreated (because its dependencies change), the unmount will call the **stale** version — which may reference outdated refs and fail to clean up resources. Never eslint-disable the exhaustive-deps warning to paper over this; use the `cleanupRef` pattern instead.

## The Mistake

A voice input hook had a `cleanup` function created via `useCallback` that depended on `releaseAudioResources`. The unmount effect suppressed the lint warning:

```typescript
// WRONG — cleanup is stale at unmount time
const cleanup = useCallback(() => {
    clientRef.current?.stopAbnormally();
    releaseAudioResources();
}, [releaseAudioResources]);

useEffect(() => {
    return () => { cleanup(); };
    // eslint-disable-next-line react-hooks/exhaustive-deps
}, []);
```

When the component re-rendered and `cleanup` was recreated, the unmount effect still held a reference to the original version. On unmount, it called the stale cleanup which could reference outdated refs.

## Why This Is Wrong

- `useCallback` with dependencies creates a NEW function reference when deps change
- `useEffect` with `[]` captures the function reference at mount time and never updates
- Suppressing the eslint warning hides a real bug — the warning exists for exactly this reason
- Resources (AudioContext, MediaStream, WebSocket clients) leak on unmount

## The Correct Process

Use a ref to always hold the latest cleanup function:

```typescript
// CORRECT — cleanupRef always points to the latest cleanup
const cleanup = useCallback(() => {
    clientRef.current?.stopAbnormally();
    releaseAudioResources();
}, [releaseAudioResources]);

const cleanupRef = useRef(cleanup);
cleanupRef.current = cleanup;  // Updated on every render

useEffect(() => {
    return () => { cleanupRef.current(); };
}, []);
```

This gives you both:
- Stable effect (never re-registers, no dependency array issues)
- Current cleanup (always calls the latest version via the ref)

## Red Flags to Watch For

- `eslint-disable-next-line react-hooks/exhaustive-deps` on a cleanup effect — almost always wrong
- `useEffect` with `[]` that calls a `useCallback` function — check if the callback has dependencies
- Resource cleanup (AudioContext, WebSocket, MediaStream) in unmount effects — these MUST use current refs
- "Works in dev but leaks in production" — React StrictMode double-mounts mask some stale closure bugs

## Prevention

1. **Never suppress exhaustive-deps for cleanup effects.** If the lint complains, the closure IS stale.
2. **Default to the cleanupRef pattern** for any unmount cleanup that involves mutable resources.
3. **Code review rule:** Any `useEffect` return function that calls a function defined outside the effect must either include that function in deps OR use a ref.
