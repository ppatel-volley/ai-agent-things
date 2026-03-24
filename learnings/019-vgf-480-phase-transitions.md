# VGF 4.8.0+ Phase Transition Rules

**Severity:** Critical
**Sources:** emoji-multiplatform/038, emoji-multiplatform/042
**Category:** VGF, Phases, Migration

## Principle

VGF 4.8.0+ throws `PhaseModificationError` if any reducer directly modifies `state.phase`. Phase transitions must use the `nextPhase` + `SET_NEXT_PHASE` reducer + `TRANSITION_TO_PHASE` thunk + `endIf`/`next` pattern. `WGFServer` does NOT call `onConnect`/`onDisconnect` lifecycle hooks — handle setup via client-initiated thunks.

> **VGF 4.9.0–4.12.1 updates:**
> - **Actions removed (v4.9.0):** `GameRuleset.actions` and `Phase.actions` are no longer processed. Use reducers and thunks only.
> - **Cascade depth limiting (v4.10.0):** The stale `nextPhase` infinite oscillation bug (EM-042) would now be caught by `maxTransitionDepth` before reaching OOM — but the root cause (stale `nextPhase`) should still be fixed properly.
> - **Async dispatch (v4.11.0):** `reducerDispatcher` returns `Promise<void>`. Can `await` for fresh state.
> - **Lifecycle void return (v4.11.0):** `onBegin`/`onEnd` can return `void`. No longer required to return `GameState`.
> - **Dispatch buffering (v4.12.0):** Dispatches during lifecycle hooks are buffered and drained after transition. This may improve the reliability of dispatches from `onConnect`.

## Details

### The breaking change

In VGF 4.8.0, reducers are no longer allowed to modify `state.phase` directly. Any reducer that sets `state.phase` throws a `PhaseModificationError` at runtime:

```ts
// BAD — throws PhaseModificationError in 4.8.0
const setPhase = (state, payload) => ({
  ...state,
  phase: payload.phase,  // PhaseModificationError!
});
```

### The correct pattern

Phase transitions in 4.8.0 require four coordinated pieces:

**1. State field:** Add a `nextPhase` field to your game state:

```ts
interface GameState {
  phase: string;
  nextPhase: string | null;  // new — signals desired transition
  // ... other fields
}
```

**2. Reducer:** `SET_NEXT_PHASE` sets the signal without touching `phase`:

```ts
const SET_NEXT_PHASE = (state: GameState, payload: { phase: string }): GameState => ({
  ...state,
  nextPhase: payload.phase,
});
```

**3. Thunk:** `TRANSITION_TO_PHASE` dispatches the reducer:

```ts
const TRANSITION_TO_PHASE = (ctx: IThunkContext, phase: string) => {
  ctx.dispatch("SET_NEXT_PHASE", { phase });
};
```

**4. Phase config:** `endIf` checks the signal, `next` returns the target:

```ts
const lobbyPhase = {
  endIf: (ctx: IGameActionContext): boolean => {
    const state = ctx.session.state;
    return state.nextPhase !== null && state.nextPhase !== state.phase;
  },
  next: (ctx: IGameActionContext): string => {
    return ctx.session.state.nextPhase;
  },
  onEnd: async (ctx: IOnEndContext): Promise<void> => {
    // Clear the signal (v4.11.0+: void return preferred)
    await ctx.reducerDispatcher("SET_NEXT_PHASE", { phase: null });
  },
};
```

### Dispatch context table

| Dispatch origin | Goes through GameRunner? | endIf evaluated? | Phase transition works? |
|----------------|------------------------|-------------------|----------------------|
| Client dispatch | Yes | Yes | Yes |
| Thunk (`ctx.dispatch`) | Yes | Yes (4.8.0 fix) | Yes |
| `onConnect` | No | No — use `SET_NEXT_PHASE` | No |
| `onDisconnect` | No | No — use `SET_NEXT_PHASE` | No |
| Scheduler thunk | Yes (via `dispatchThunk`) | **Unreliable** — see [016](./016-vgf-endif-cascade-limitations.md) Scenario 3. Use defensive `SET_NEXT_PHASE`. | Only with explicit `SET_NEXT_PHASE` |

> **Note:** The scheduler row was previously marked "Yes (4.8.0 fix)" but learning 016 documents an observed bug (EM-024) where scheduler-triggered thunks did NOT evaluate `endIf`, causing the game to hang. Always pair scheduler-triggered state changes with `SET_NEXT_PHASE` dispatches — NOT `SET_PHASE`, which throws `PhaseModificationError` in 4.8.0+.

### WGFServer lifecycle limitations

`WGFServer` does NOT call `onConnect` or `onDisconnect` lifecycle hooks. Any setup logic that was previously in `onConnect` must be moved to a client-initiated thunk:

```ts
// BAD — onConnect is never called by WGFServer
onConnect: (ctx) => {
  ctx.reducerDispatcher("addPlayer", { id: ctx.userId });
},

// GOOD — client sends a thunk after connecting
// Client side:
transport.dispatchThunk("joinGame", { userId });

// Server side:
const joinGame = (ctx: IThunkContext, payload: { userId: string }) => {
  ctx.dispatch("addPlayer", { id: payload.userId });
};
```

### Dev auto-start must be client-initiated

In 4.8.0, `broadcastStateUpdate` runs BEFORE `onConnect`. If dev auto-start logic runs in `onConnect`, the initial broadcast sends `{}` (empty state) to the client. The client must initiate the auto-start sequence:

```ts
// BAD — broadcastStateUpdate sends {} before onConnect runs
onConnect: (ctx) => {
  if (isDev) setupDevGame(ctx);
},

// GOOD — client triggers dev setup after connection
// Client:
useEffect(() => {
  if (isDev) {
    transport.dispatchThunk("devSetup", {});
  }
}, [connected]);
```

### CRITICAL: Stale nextPhase causes infinite oscillation (OOM) (EM-042)

The `nextPhase` pattern has a critical gotcha: `nextPhase` is a one-shot signal that must be cleared after consumption. If it persists, it re-triggers `endIf` from subsequent phases, causing an infinite cascade.

**The bug chain:**
1. `SET_NEXT_PHASE("categorySelect")` dispatched from `onConnect`
2. Phase runner: lobby → categorySelect. `nextPhase` stays `"categorySelect"`.
3. User selects category → `categorySelect.endIf` = true → transitions to `difficultySelect`
4. `difficultySelect.endIf` checks `hasNextPhase("categorySelect" !== "difficultySelect")` → **TRUE!**
5. `difficultySelect.next()` returns stale `"categorySelect"`
6. Back to categorySelect → `category !== null` → endIf true → back to difficultySelect
7. **Infinite loop** — each cycle allocates state objects until 4GB OOM

**The fix:** Every phase's `onBegin` must clear `nextPhase` as its first action:

```ts
// Add CLEAR_NEXT_PHASE reducer
const CLEAR_NEXT_PHASE = (state: GameState): GameState => ({
  ...state,
  nextPhase: null,
});

// Every phase's onBegin:
onBegin: (ctx) => {
  ctx.reducerDispatcher("CLEAR_NEXT_PHASE", {});
  // ... rest of onBegin
},
```

**Alternatively**, clear `nextPhase` in the source phase's `onEnd` (as shown in the phase config example above). The critical point is that `nextPhase` must be nullified before any subsequent `endIf` evaluation.

**Red flags for stale nextPhase:**
- One-shot state fields (`nextPhase`, `pendingAction`, etc.) without a corresponding clear mechanism
- `endIf` conditions that combine domain logic (`category !== null`) with transition signals (`hasNextPhase`) — signals can re-fire from unexpected phases
- Phase transitions that work in isolation but fail when chained (A→B works, B→C→??? loops)
- Server OOM within seconds of a user action — almost always an infinite loop

### Red flags

If you see any of these, you've hit the 4.8.0 phase transition issue:

- `PhaseModificationError` in server logs
- Game stuck in lobby despite meeting start conditions
- `broadcastStateUpdate` sending `{}` to clients
- `onConnect` logic not executing

## Prevention

1. **Search and replace:** Before upgrading to 4.8.0, grep for any reducer that assigns to `state.phase` and refactor to the `nextPhase` pattern.
2. **Remove onConnect dependencies:** Audit all `onConnect` handlers and move critical logic to client-initiated thunks.
3. **Dev mode smoke test:** After upgrading, verify that dev auto-start still works — this is the most likely regression.
4. **Type guard:** Make `phase` a `readonly` field in your state interface to catch direct assignments at compile time.
5. **Clear nextPhase:** Every phase's `onBegin` must dispatch `CLEAR_NEXT_PHASE` as its first action. Any "signal" field in game state must have a corresponding clear mechanism.
6. **Test phase chains:** Test multi-phase transition chains end-to-end (A→B→C), not just individual transitions (A→B).

<details>
<summary>EM-038 Context</summary>

Upgrading from VGF 4.x to 4.8.0 broke the entire emoji quiz game. The lobby phase never transitioned to playing because `SET_PHASE` directly modified `state.phase`, which now threw `PhaseModificationError`. Additionally, `onConnect` was no longer called by `WGFServer`, so player registration and dev auto-start both failed silently. The initial `broadcastStateUpdate` sent `{}` because it ran before any setup logic. The fix required three coordinated changes: the `nextPhase` pattern for transitions, moving `onConnect` logic to client-initiated thunks, and making dev auto-start client-triggered.

</details>
