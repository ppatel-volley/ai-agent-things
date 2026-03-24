# VGF endIf Cascade Limitations

**Severity:** Critical
**Sources:** emoji-multiplatform/015, emoji-multiplatform/020, emoji-multiplatform/024, weekend-poker/021
**Category:** VGF, Phases, Server

## Principle

VGF's `endIf` is only reliably evaluated after client-originated dispatches. It is NOT evaluated after `onConnect`, scheduler-triggered thunks, or `onDisconnect`. When `endIf` does cascade, the `onBegin` context may have a different shape than expected. For critical transitions, always use explicit phase transitions instead of relying on `endIf`.

> **VGF 4.10.0–4.12.1 changes (unverified — apply all workarounds until confirmed in your project):**
> - **Cascade depth limiting (v4.10.0):** Configurable `maxTransitionDepth` prevents infinite cascades. The OOM in Scenario 4 would now error instead of crashing — but fix the root cause (stale flags) regardless.
> - **Engine state machine + dispatch buffering (v4.12.0):** Dispatches during lifecycle hooks are now buffered and drained after transitions. This *may* change which triggers evaluate `endIf`, but the VGF changelog does not explicitly confirm that onConnect/scheduler dispatches now trigger endIf.
>
> **Operational rule:** Continue applying all workarounds below (explicit `SET_NEXT_PHASE`, `resetPhaseFlags`) until you have verified the specific scenario works in your installed VGF version with a test. "The changelog mentions buffering" is not the same as "endIf now fires after onConnect."

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

// GOOD — force the transition explicitly via SET_NEXT_PHASE (4.8.0+)
onConnect: async (ctx) => {
  await ctx.reducerDispatcher("addPlayer", { playerId: ctx.userId });
  const state = ctx.getState();  // v4.11.0+: getState() is fresh after await
  if (state.players.length >= 2) {
    await ctx.reducerDispatcher("SET_NEXT_PHASE", { phase: "playing" });
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

**Fix:** Use a thunk with explicit `SET_NEXT_PHASE` instead of relying on the cascade:

```ts
// Instead of endIf cascade, use explicit transition via SET_NEXT_PHASE
const checkAndTransition = async (ctx: IThunkContext) => {
  const state = ctx.getState();
  if (state.roundComplete) {
    await ctx.dispatch("SET_NEXT_PHASE", { phase: "nextRound" });
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

// Fix: dispatch the phase transition explicitly via SET_NEXT_PHASE
const onTimeout = async (ctx: IThunkContext) => {
  await ctx.dispatch("setQuizOver", {});
  await ctx.dispatch("SET_NEXT_PHASE", { phase: "gameOver" });
};
```

### Scenario 4: PhaseRunner2 checks endIf BEFORE onBegin on re-entry (WP-021)

When a phase loop completes (e.g., `BJ_HAND_COMPLETE` → back to `BJ_PLACE_BETS`), `PhaseRunner2.performSingleTransitionCheck()` checks `endIf` **before** running `onBegin`. If per-round completion flags (`allBetsPlaced`, `dealComplete`, etc.) are still `true` from the previous round, `endIf` immediately triggers the next transition without `onBegin` ever resetting them. This creates an infinite cascade through all phases until OOM (3.9GB).

```ts
// PhaseRunner2 loop:
// 1. Check endIf — if true, set phase to next, loop again
// 2. Check if phase changed — if so, run onEnd/onBegin, loop again
// On loop-back, step 1 fires BEFORE step 2 ever runs onBegin
```

**The fix:** Add a `resetPhaseFlags` reducer that clears ALL per-phase completion flags. Call it in the loop-back phase's `onBegin` before setting any new flags:

```ts
// In the round-complete phase's onBegin:
ctx.reducerDispatcher('resetPhaseFlags')      // Clear stale flags first
ctx.reducerDispatcher('setRoundReady', true)   // Then mark round complete
```

The reset reducer must clear EVERY flag that any `endIf` checks:

```ts
resetPhaseFlags: (state) => ({
  ...state,
  allBetsPlaced: false,
  dealComplete: false,
  insuranceComplete: false,
  playerTurnsComplete: false,
  dealerTurnComplete: false,
  settlementComplete: false,
  roundCompleteReady: false,
})
```

**Affected all 5 Weekend Casino games** — Blackjack Classic, Blackjack Competitive, Three Card Poker, Roulette, and Craps all needed reset reducers. 5-Card Draw was safe (used `playablePlayers.length < 2` guard instead of flag checks).

**Red flags:**
- Any phase whose `next` loops back to an earlier phase in the flow
- Any `endIf` that checks a flag set in `onBegin` — those flags persist across loops
- Missing flags in the reset reducer (add every new `endIf` flag to the reset)

### Summary table

| Trigger | endIf evaluated? | Workaround (still required) |
|---------|-----------------|------------|
| Client dispatch | Yes | None needed |
| `onConnect` dispatch | No (unverified in 4.12.1) | Dispatch `SET_NEXT_PHASE` (see [019](./019-vgf-480-phase-transitions.md)) |
| Scheduler thunk | No (unverified in 4.12.1) | Dispatch `SET_NEXT_PHASE` from the thunk |
| `onDisconnect` dispatch | No (unverified in 4.12.1) | Dispatch `SET_NEXT_PHASE` |
| `onBegin` cascade | Partial (unverified in 4.12.1) | Thunk with `SET_NEXT_PHASE` |
| Phase re-entry (loop-back) | Yes, BEFORE onBegin | `resetPhaseFlags` reducer (+ `maxTransitionDepth` as safety net) |

> **Important:** VGF 4.8.0+ throws `PhaseModificationError` if a reducer modifies `state.phase` directly. The workaround is `SET_NEXT_PHASE` + `endIf`/`next` — see [learning 019](./019-vgf-480-phase-transitions.md) for the full pattern.

## Prevention

1. **Rule of thumb:** Never rely on `endIf` for critical transitions. Always pair it with an explicit `SET_NEXT_PHASE` dispatch as a fallback (see [learning 019](./019-vgf-480-phase-transitions.md) for the full pattern).
2. **Defensive thunks:** Wrap phase-ending logic in thunks that check the condition and dispatch `SET_NEXT_PHASE` directly.
3. **Integration test:** For each phase, test that the transition fires from both client dispatches and server-side triggers (scheduler, onConnect).
4. **Timeout safety net:** Add a scheduler-based timeout that checks if the phase should have ended and forces the transition.
5. **Reset flags on loop-back:** When any phase loops back to an earlier phase, add a `resetPhaseFlags` reducer that clears all per-round completion flags. Call it as the first action in `onBegin`.
6. **Flag audit:** When adding a new `endIf` flag to any game, add it to the reset reducer too.

<details>
<summary>EM-015 Context</summary>

Players joining the lobby triggered `onConnect` which dispatched `addPlayer`. The `endIf` condition (`playerCount >= minPlayers`) was met but never evaluated. The game remained stuck in the lobby phase. The fix was adding an explicit phase check and `SET_NEXT_PHASE` dispatch inside `onConnect`.

</details>

<details>
<summary>EM-020 Context</summary>

A client dispatch caused a phase to end, which cascaded into the next phase's `onBegin`. Inside that `onBegin`, `ctx.getSessionId()` threw "getSessionId is not a function" because the cascaded context was an `IGameActionContext`, not the expected `IOnBeginContext`. The fix replaced the `endIf` cascade with an explicit thunk that dispatched `SET_NEXT_PHASE`.

</details>

<details>
<summary>EM-024 Context</summary>

The quiz timer expired, triggering a scheduler thunk that set `state.status = "QUIZ_OVER"`. The `endIf` for the playing phase was supposed to detect this and transition to `gameOver`, but it never ran. The game displayed "Time's Up!" indefinitely. The fix added an explicit `SET_NEXT_PHASE` dispatch to the timeout thunk.

</details>
