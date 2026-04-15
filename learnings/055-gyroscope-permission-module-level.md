# Learning 055: Gyroscope Permission Must Persist Across Component Remounts

**Severity:** Critical
**Sources:** Tempest 2026 — iOS tilt controls
**Category:** React, Device APIs, iOS

## Principle

On iOS 13+, `DeviceOrientationEvent.requestPermission()` must be called from a user gesture, and the granted permission is tied to the page, not the component. If the permission state is stored in React component state (`useState`), it is lost when the component unmounts. When a new component mounts and creates a fresh `useGyroscope` hook instance, `permissionGranted` starts as `false` and the `deviceorientation` listener is never registered.

## The Mistake

The gyroscope hook stored permission state in `useState`:

```typescript
// WRONG — permission lost on component unmount
export function useGyroscope() {
    const [permissionGranted, setPermissionGranted] = useState(false)
    // ...
}
```

The lobby component called `requestPermission()` and got `permissionGranted = true`. When the game transitioned to the playing phase, the lobby unmounted and the playing component mounted a NEW `useGyroscope` instance with `permissionGranted = false`. The `useEffect` that registers the `deviceorientation` listener checked `if (!permissionGranted) return` — so it never ran.

Result: tilt data showed `TILT: 0.500` permanently — gyroscope events were never received during gameplay.

## The Correct Process

Store permission state at **module level** so it survives component mount/unmount cycles:

```typescript
// CORRECT — persists across component lifecycle
let globalPermissionGranted = false

export function useGyroscope() {
    const [permissionGranted, setPermissionGranted] = useState(globalPermissionGranted)

    useEffect(() => {
        if (globalPermissionGranted) setPermissionGranted(true)
    }, [])

    const requestPermission = useCallback(async () => {
        // ... iOS permission flow ...
        globalPermissionGranted = true
        setPermissionGranted(true)
    }, [])
}
```

## Red Flags to Watch For

- Any `useState` for device permission that's checked in a `useEffect` controlling event listeners
- Permission requested in one component (lobby) but consumed in another (gameplay)
- VGF phase transitions that unmount/remount display components — any per-component state is lost
- "Works in lobby but not in gameplay" — classic symptom of state lost on remount

## Prevention

1. **Store device permission state at module level**, not in React state
2. **Test the full lobby → gameplay transition** on a real iOS device — not just the lobby in isolation
3. **For any hook that spans multiple VGF phases**, consider whether its state survives the phase transition component swap
