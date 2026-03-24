# VGF Phase Callback Contexts

**Severity:** Critical
**Sources:** weekend-poker/009, emoji-multiplatform/016
**Category:** VGF, Server, Phases

## Principle

VGF provides different context objects to phase lifecycle callbacks versus thunks. Using the wrong API on the wrong context causes runtime crashes.

> **VGF 4.11.0+ update:** `onBegin` and `onEnd` now support returning `void` — returning nothing is the **preferred** pattern. Dispatch reducers instead of returning state. The v4.8.0 crash from returning `undefined` is fixed. The `reducerDispatcher` method now returns `void | Promise<void>` (not `GameState`), so `return ctx.reducerDispatcher(...)` returns void, not state.

## Details

### Context type reference

| Callback | Context type | State access | Dispatch method | Has `thunkDispatcher` | Has `scheduler` |
|----------|-------------|-------------|----------------|----------------------|-----------------|
| `onBegin` | `IOnBeginContext` | `getState()` | `reducerDispatcher` | Yes | Yes |
| `onEnd` | `IOnEndContext` | `getState()` | `reducerDispatcher` | Yes | Yes |
| `endIf` | `IGameActionContext` | `ctx.session.state` | N/A (read-only) | No | No |
| `next` | `IGameActionContext` | `ctx.session.state` | N/A (read-only) | No | No |
| Thunks | `IThunkContext` | `getState()` | `ctx.dispatch` | `dispatchThunk` | Yes |

### Key differences

```ts
// onBegin — uses reducerDispatcher, can return void (preferred in 4.11.0+)
const onBegin = async (ctx: IOnBeginContext): Promise<void> => {
  // Use reducerDispatcher to modify state — await for fresh state
  await ctx.reducerDispatcher("initRound", { round: 1 });
  // No return needed — void is preferred
};

// endIf — uses ctx.session.state, read-only check
const endIf = (ctx: IGameActionContext): boolean => {
  const state = ctx.session.state;  // NOT getState()
  return state.roundComplete === true;
};

// Thunk — uses ctx.dispatch, does NOT return state
const myThunk = (ctx: IThunkContext) => {
  const state = ctx.getState();
  ctx.dispatch("updateScore", { score: 10 });
  ctx.dispatchThunk(anotherThunk);
};
```

### reducerDispatcher behaviour (WP-009)

- `reducerDispatcher` is an arrow function — safe to extract via object destructuring or getters.
- Root reducers (defined at the game level, not phase level) are available in ALL phases.
- Phase cascade is recursive and atomic — if `endIf` triggers during `onBegin`, the cascade completes before control returns.
- Server-side `reducerDispatcher` throws immediately when given an unrecognised reducer name. Client-side dispatch silently times out instead.

```ts
// Server — throws immediately
ctx.reducerDispatcher("typoName", {});
// Error: Reducer "typoName" not found

// Client — silent timeout
dispatch("typoName", {});
// ... 30 seconds later: DispatchTimeoutError
```

### onBegin return value (EM-016)

> **VGF 4.11.0+ update:** This section describes a bug that was fixed in v4.11.0. Returning void from `onBegin`/`onEnd` is now explicitly supported. The preferred pattern is to dispatch reducers and NOT return state.

~~Returning `undefined` from `onBegin` crashes VGF deep inside its phase evaluation logic (`didPhaseEnd`).~~ Fixed in v4.11.0. The type is now `GameState | void | Promise<GameState | void>`.

```ts
// VGF 4.11.0+ — preferred pattern: dispatch reducers, return nothing
const onBegin = async (ctx) => {
  await ctx.reducerDispatcher("setup", {});
  // void return is fine — preferred over returning state
};

// Also valid: return state explicitly (backwards compatible)
const onBegin = (ctx) => {
  return { ...ctx.getState(), ready: true };
};
```

## Prevention

1. **Type enforcement:** Use VGF's own type definitions for phase callbacks, not custom interfaces. If the project has a local `Phase` type, verify it matches VGF's declarations.
2. **Prefer void return:** In VGF 4.11.0+, dispatch reducers in `onBegin`/`onEnd` and return nothing. Avoid returning state objects — they can overwrite inline dispatches.
3. **Context cheat sheet:** Keep the callback-context table above visible during development. Print it out if necessary.
4. **Integration test:** For each phase, verify that `endIf` accesses state via `ctx.session.state` and that `onBegin` dispatches produce expected state.

<details>
<summary>WP-009 Context</summary>

In Weekend Poker, confusion between `reducerDispatcher` and `ctx.dispatch` caused crashes in phase lifecycle hooks. Investigation revealed that `reducerDispatcher` is an arrow function (safe to destructure), root reducers are globally available across phases, and the server throws immediately on unrecognised names — unlike the client which silently times out. The phase cascade being recursive and atomic was discovered when an `endIf` triggered mid-`onBegin`, causing unexpected but correct behaviour.

</details>

<details>
<summary>EM-016 Context</summary>

In the emoji-multiplatform project, `onBegin` returned `undefined` because the local `Phase` interface typed it as `void`. VGF crashed inside `didPhaseEnd` with an opaque error. The original fix was returning the result of `reducerDispatcher` from `onBegin`. **VGF 4.11.0+ note:** This crash is now fixed — void return is supported. The preferred pattern is to dispatch reducers and return nothing.

</details>
