# VGF 4.8.0 Phase Transition Rules

**Severity:** Critical
**Sources:** emoji-multiplatform/038
**Category:** VGF, Phases, Migration

## Principle

VGF 4.8.0 throws `PhaseModificationError` if any reducer directly modifies `state.phase`. Phase transitions must use the `nextPhase` + `SET_NEXT_PHASE` reducer + `TRANSITION_TO_PHASE` thunk + `endIf`/`next` pattern. `WGFServer` does NOT call `onConnect`/`onDisconnect` lifecycle hooks — handle setup via client-initiated thunks.

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
  onEnd: (ctx: IOnEndContext): GameState => {
    // Clear the signal
    return ctx.reducerDispatcher("SET_NEXT_PHASE", { phase: null });
  },
};
```

### Dispatch context table

| Dispatch origin | Goes through GameRunner? | endIf evaluated? | Phase transition works? |
|----------------|------------------------|-------------------|----------------------|
| Client dispatch | Yes | Yes | Yes |
| Thunk (`ctx.dispatch`) | Yes | Yes (4.8.0 fix) | Yes |
| `onConnect` | No | No | No |
| `onDisconnect` | No | No | No |
| Scheduler thunk | Yes (via `dispatchThunk`) | Yes (4.8.0 fix) | Yes |

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

<details>
<summary>EM-038 Context</summary>

Upgrading from VGF 4.x to 4.8.0 broke the entire emoji quiz game. The lobby phase never transitioned to playing because `SET_PHASE` directly modified `state.phase`, which now threw `PhaseModificationError`. Additionally, `onConnect` was no longer called by `WGFServer`, so player registration and dev auto-start both failed silently. The initial `broadcastStateUpdate` sent `{}` because it ran before any setup logic. The fix required three coordinated changes: the `nextPhase` pattern for transitions, moving `onConnect` logic to client-initiated thunks, and making dev auto-start client-triggered.

</details>
