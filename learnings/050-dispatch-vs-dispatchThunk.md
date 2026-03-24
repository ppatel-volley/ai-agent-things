# Learning 050: ctx.dispatch vs ctx.dispatchThunk — Silent No-Op

**Severity:** Critical
**Sources:** TokenRaider (TR) — Recognition Service integration
**Category:** State Management, VGF Framework

## Principle

In VGF thunks, `ctx.dispatch()` dispatches to **reducers** and `ctx.dispatchThunk()` dispatches to **thunks**. Using the wrong one causes a silent no-op — no error, no warning, nothing happens. When building a data-driven dispatch system (like a voice command registry), verify whether each action name is a reducer or a thunk, and call the correct method.

## The Mistake

A voice command registry stored action names (`SET_HEADING`, `LAY_ANCHOR`, `SEEK_RESOURCES`) that were **thunk** names. The `PROCESS_TRANSCRIPTION` thunk dispatched them via `ctx.dispatch()`:

```typescript
// WRONG — dispatch() targets reducers, not thunks
for (const command of matchedCommands) {
    ctx.dispatch(command.action, command.payload);
}
```

`SET_HEADING` and `SEEK_RESOURCES` are thunks (they start the ship tick loop and have side effects). Dispatching them as reducers did nothing — no error, no crash, just silence. The ship never moved.

## Why This Is Wrong

The `IThunkContext` interface has two distinct dispatch methods:
- `dispatch(reducerName, ...args)` — invokes a **reducer** (pure state transform, no side effects). Returns `void | Promise<void>` — **can be awaited** (v4.11.0+) for fresh state via `getState()`.
- `dispatchThunk(thunkName, ...args)` — invokes a **thunk** (async, can have side effects like timers). Returns `Promise<void>`.

These are NOT interchangeable. Calling `dispatch("SET_HEADING", "north")` looks for a reducer named `SET_HEADING`. The actual reducer is `SET_SHIP_HEADING` — so even if dispatch-to-reducer were correct, the name wouldn't match.

## The Correct Process

```typescript
// CORRECT — dispatchThunk() for thunk actions
for (const command of matchedCommands) {
    await ctx.dispatchThunk(command.action, command.payload);
}
```

Additionally, the **payload shape** must match the thunk's signature:

```typescript
// WRONG — thunk expects bare string, not object
{ action: "SET_HEADING", payload: { heading: "north" } }

// CORRECT — thunk signature is (ctx, heading: ShipHeading)
{ action: "SET_HEADING", payload: "north" }
```

## Red Flags to Watch For

- A data-driven dispatch table where action names are strings — verify each one is a reducer or thunk
- `ctx.dispatch()` with an action name that doesn't appear in `globalReducers`
- Voice/transcript processing that dispatches game actions — these are usually thunks with side effects
- Any dispatch that silently does nothing — check reducer vs thunk first

## Prevention

1. **Read the thunk/reducer signatures** before wiring a dispatch table. Don't assume.
2. **Consider adding a `dispatchType` field** to command registries: `"reducer" | "thunk"` to make the intent explicit.
3. **Test the dispatch path end-to-end** — a unit test that mocks `ctx.dispatchThunk` and verifies it's called (not `ctx.dispatch`) would have caught this immediately.
4. **Type-safe action names** — use `keyof typeof thunks | keyof typeof reducers` union types instead of bare `string`.
