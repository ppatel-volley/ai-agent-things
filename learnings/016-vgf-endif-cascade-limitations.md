# VGF endIf Cascade Limitations

**Severity:** Critical
**Sources:** emoji-multiplatform/015, emoji-multiplatform/020, emoji-multiplatform/024
**Category:** VGF, Phases, Server

## Principle

VGF's `endIf` is only reliably evaluated after client-originated dispatches. It is NOT evaluated after `onConnect`, scheduler-triggered thunks, or `onDisconnect`. When `endIf` does cascade, the `onBegin` context may have a different shape than expected. For critical transitions, always use explicit phase transitions instead of relying on `endIf`.

## Details

Three separate scenarios where `endIf` failed to trigger, each requiring a different workaround but all pointing to the same root cause: `endIf` evaluation is tightly coupled to the client dispatch pipeline.

### Scenario 1: endIf not evaluated after onConnect (EM-015)

When a player connects and a dispatch happens inside `onConnect`, `endIf` is not re-evaluated. The phase transition condition was met but never checked.

```ts
// BAD — endIf won't fire after onConnect dispatch
onConnect: (ctx) => {
  ctx.reducerDispatcher("addPlayer", { playerId: ctx.userId });
  // endIf checks playerCount >= 2... but never runs
},

// GOOD — force the transition explicitly
onConnect: (ctx) => {
  const state = ctx.reducerDispatcher("addPlayer", { playerId: ctx.userId });
  if (state.players.length >= 2) {
    ctx.reducerDispatcher("SET_PHASE", { phase: "playing" });
  }
},
```

### Scenario 2: endIf cascade passes wrong context to onBegin (EM-020)

When `endIf` does cascade (e.g. after a client dispatch that ends multiple phases in sequence), the `onBegin` of the next phase receives a context with an unexpected shape.

```ts
// In the cascaded onBegin, this crashed:
const onBegin = (ctx) => {
  const sessionId = ctx.getSessionId();  // TypeError: getSessionId is not a function
};
```

The cascaded context is an `IGameActionContext`, not the full `IOnBeginContext`. Methods like `getSessionId()` do not exist on it.

**Fix:** Use a thunk with explicit `SET_PHASE` instead of relying on the cascade:

```ts
// Instead of endIf cascade, use explicit transition
const checkAndTransition = (ctx: IThunkContext) => {
  const state = ctx.getState();
  if (state.roundComplete) {
    ctx.dispatch("SET_PHASE", { phase: "nextRound" });
  }
};
```

### Scenario 3: endIf not evaluated after scheduler-triggered thunks (EM-024)

A scheduler fires a thunk that sets `state.status = "QUIZ_OVER"`. The `endIf` for the current phase checks for `status === "QUIZ_OVER"` but never runs. The game gets stuck on "Time's Up!" indefinitely.

```ts
// The scheduler thunk correctly updates state...
const onTimeout = (ctx: IThunkContext) => {
  ctx.dispatch("setQuizOver", {});
  // endIf should fire here... but it doesn't
};

// Fix: dispatch the phase transition explicitly
const onTimeout = (ctx: IThunkContext) => {
  ctx.dispatch("setQuizOver", {});
  ctx.dispatch("SET_PHASE", { phase: "gameOver" });
};
```

### Summary table

| Trigger | endIf evaluated? | Workaround |
|---------|-----------------|------------|
| Client dispatch | Yes | None needed |
| `onConnect` dispatch | No | Explicit `SET_PHASE` |
| Scheduler thunk | No | Explicit `SET_PHASE` |
| `onDisconnect` dispatch | No | Explicit `SET_PHASE` |
| `onBegin` cascade | Partial (wrong context) | Thunk with `SET_PHASE` |

## Prevention

1. **Rule of thumb:** Never rely on `endIf` for critical transitions. Always pair it with an explicit `SET_PHASE` dispatch as a fallback.
2. **Defensive thunks:** Wrap phase-ending logic in thunks that check the condition and dispatch `SET_PHASE` directly.
3. **Integration test:** For each phase, test that the transition fires from both client dispatches and server-side triggers (scheduler, onConnect).
4. **Timeout safety net:** Add a scheduler-based timeout that checks if the phase should have ended and forces the transition.

<details>
<summary>EM-015 Context</summary>

Players joining the lobby triggered `onConnect` which dispatched `addPlayer`. The `endIf` condition (`playerCount >= minPlayers`) was met but never evaluated. The game remained stuck in the lobby phase. The fix was adding an explicit phase check and `SET_PHASE` dispatch inside `onConnect`.

</details>

<details>
<summary>EM-020 Context</summary>

A client dispatch caused a phase to end, which cascaded into the next phase's `onBegin`. Inside that `onBegin`, `ctx.getSessionId()` threw "getSessionId is not a function" because the cascaded context was an `IGameActionContext`, not the expected `IOnBeginContext`. The fix replaced the `endIf` cascade with an explicit thunk that dispatched `SET_PHASE`.

</details>

<details>
<summary>EM-024 Context</summary>

The quiz timer expired, triggering a scheduler thunk that set `state.status = "QUIZ_OVER"`. The `endIf` for the playing phase was supposed to detect this and transition to `gameOver`, but it never ran. The game displayed "Time's Up!" indefinitely. The fix added an explicit `SET_PHASE` dispatch to the timeout thunk.

</details>
