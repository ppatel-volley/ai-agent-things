# Learning 059: PlatformAudio Lifecycle Races — Generation Counter, Listener Stacking, Seek Reset

**Severity:** Critical
**Sources:** Tempest 2026 — multiple music tracks playing simultaneously
**Category:** Audio, React Patterns, PlatformAudio

## Principle

When reusing PlatformAudio instances across game phase transitions (e.g. preloaded music tracks), three interacting bugs cause overlapping playback: stale async callbacks, stacked event listeners on reused instances, and missing seek resets. All three must be addressed together — fixing only one or two leaves the bug present.

## The Mistake

The music system had three interacting bugs:

### Bug 1: Stale Async Callbacks

`startLobbyMusic()` called `audio.load().then(() => audio.play())`. If a phase transition called `stopAll()` before the Promise resolved, the `.then()` callback still fired and played the orphaned audio instance — `stopAll()` had destroyed the module-level reference but the callback captured the local variable.

```typescript
// BROKEN — stale callback plays after stopAll()
export function startLobbyMusic(): void {
    stopAll()
    const audio = new PlatformAudio({ src: TRACK_LOBBY, ... })
    lobbyAudio = audio
    audio.load().then(() => {
        audio.play()  // ← fires even after stopAll() destroyed lobbyAudio
    })
}
```

### Bug 2: Event Listener Stacking on Reused Instances

Preloaded tracks were reused across game cycles. Each call to `playGameplayTrack()` registered a `.once("end", handler)`. When the track was paused mid-playback (via `stopAll()` during a phase transition), the `.once("end")` handler stayed registered — it only removes itself when the event actually fires. Next game cycle added another handler. After N cycles, N handlers were stacked on the same instance. When the track finally played to completion, ALL handlers fired simultaneously, each calling `playGameplayTrack()` with a different index — starting multiple tracks at once.

```typescript
// BROKEN — listeners accumulate across game cycles
function playGameplayTrack(idx: number): void {
    const audio = preloadedTracks[trackIdx]
    audio.once("end", () => {        // ← stacks on previous unfired handlers
        gameplayIndex++
        playGameplayTrack(gameplayIndex)
    })
    audio.pause()
    audio.play()                     // ← no seek(0), resumes from last position
}
```

**Timeline of the stacking bug:**
1. Level 1: `playGameplayTrack(0)` → handler A registered on `preloadedTracks[0]`
2. Level complete: `stopAll()` pauses track 0 → handler A stays (track paused, not ended)
3. Level 2: `playGameplayTrack(0)` → handler B registered on same `preloadedTracks[0]`
4. Track 0 plays to end → both A and B fire → two new tracks start simultaneously
5. Each subsequent level adds more handlers — problem compounds

### Bug 3: Incomplete `stopAll()` Cleanup

`stopAll()` only paused whichever track `gameplayAudio` pointed to, but didn't touch other preloaded track instances that might have accumulated listeners.

```typescript
// BROKEN — only cleans up the current gameplayAudio reference
function stopAll(): void {
    if (gameplayAudio) {
        gameplayAudio.pause()
        gameplayAudio = null
    }
    // preloadedTracks[0] and [1] may still have stacked listeners!
}
```

## The Correct Process

### Fix 1: Generation Counter for Async Guards

```typescript
let generation = 0

function stopAll(): void {
    generation++  // invalidates all pending callbacks
    // ... cleanup
}

export function startLobbyMusic(): void {
    stopAll()
    const gen = generation  // capture AFTER stopAll increments
    const audio = new PlatformAudio({ ... })
    lobbyAudio = audio
    audio.load().then(() => {
        if (gen !== generation) return  // stale — a new stopAll() fired
        audio.play()
    })
}
```

### Fix 2: Clear Listeners Before Re-registering

PlatformAudio doesn't expose `.off()` — access the underlying Howl via `audio.howl` (public readonly property on PlatformAudioCore):

```typescript
function playGameplayTrack(idx: number): void {
    const gen = generation
    const audio = preloadedTracks[trackIdx]
    gameplayAudio = audio
    audio.howl.off("end")        // clear ALL stacked handlers
    audio.once("end", () => {
        if (gen !== generation) return
        gameplayIndex++
        playGameplayTrack(gameplayIndex)
    })
    audio.pause()
    audio.howl.seek(0)           // reset position to start
    audio.play()
}
```

### Fix 3: `stopAll()` Clears ALL Preloaded Tracks

```typescript
function stopAll(): void {
    generation++
    if (lobbyAudio) { lobbyAudio.pause(); lobbyAudio.destroy(); lobbyAudio = null }
    if (gameplayAudio) { gameplayAudio.pause(); gameplayAudio = null }
    // Clear ALL preloaded tracks — not just the current gameplayAudio
    for (const track of preloadedTracks) {
        track.howl.off("end")
        track.pause()
    }
}
```

### Fix 4: Centralise Music Lifecycle in a React Hook

Instead of scattering imperative `startLobbyMusic()` calls across scene components (LobbyScene, GameOverScene), use a single `useEffect` keyed on phase:

```typescript
export function useMusic(phase: string, _level: number): void {
    useEffect(() => {
        if (phase === "lobby" || phase === "gameOver") {
            startLobbyMusic()
        }
    }, [phase])

    useEffect(() => {
        return () => stopAll()  // cleanup on unmount
    }, [])
}
```

**Exception:** Gameplay music start stays in the LobbyScene keyboard handler (user gesture call stack for Web Audio activation on some browsers).

## PlatformAudio API Limitations

PlatformAudio wraps Howler.js but only exposes a subset of methods:

| Available on PlatformAudio | Must access via `audio.howl` |
|---|---|
| `play()`, `pause()`, `load()`, `destroy()` | `off()`, `seek()`, `stop()` (instance) |
| `on()`, `once()` | `duration()`, `playing()`, `state()` |
| Static: `mute()`, `volume()`, `stop()`, `unload()` | All other Howl methods |

The `howl` property is `public readonly` on PlatformAudioCore.

## Red Flags to Watch For

- **`.once("end", ...)` on a reused PlatformAudio instance** — if the instance can be paused before ending, the listener accumulates. Always call `.howl.off("end")` first.
- **`.load().then(() => audio.play())`** without a generation/cancellation guard — stale callbacks will play orphaned audio after phase transitions.
- **Reusing a preloaded track without `.howl.seek(0)`** — it resumes from where it was last paused, not from the beginning.
- **`stopAll()` that only cleans up named references** (e.g. `gameplayAudio`) — preloaded track arrays may hold additional instances with stacked listeners.
- **Imperative music calls scattered across multiple scene components** — leads to double-calls during rapid phase transitions (e.g. GameOverScene + LobbyScene both calling `startLobbyMusic()` during Play Again flow).

## Prevention

1. **Always use a generation counter** when mixing async operations (Promise callbacks, event listeners) with imperative stop/cleanup. The pattern: increment on stop, capture before async, check before executing.
2. **Always `.howl.off(eventName)` before `.once(eventName, ...)`** on reused PlatformAudio instances.
3. **Always `.howl.seek(0)` before `.play()`** on reused tracks.
4. **Centralise audio lifecycle in a single React hook** keyed on game phase — don't rely on individual scene components to coordinate stop/start.
5. **Test the "Play Again" flow specifically** — it's the highest-risk path because it chains: gameOver → lobby → levelIntro → playing, with each transition potentially triggering music changes.
