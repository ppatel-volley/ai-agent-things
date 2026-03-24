# Learning 052: Touch + Mouse Event Double-Fire on Mobile

**Severity:** High
**Sources:** TokenRaider (TR) — Recognition Service integration
**Category:** UI, Cross-Platform

## Principle

When a button has both `onTouchStart`/`onTouchEnd` AND `onMouseDown`/`onMouseUp` handlers, touch devices will fire BOTH — the touch events first, then synthesised mouse events. This causes the handler to execute twice. Always call `event.preventDefault()` in touch handlers to suppress the synthesised mouse events.

## The Mistake

A push-to-talk mic button registered all four event handlers for cross-device support:

```tsx
// WRONG — double-fires on touch devices
<button
    onTouchStart={startRecording}
    onTouchEnd={stopRecording}
    onMouseDown={startRecording}
    onMouseUp={stopRecording}
>
```

On a phone, pressing the button fired `touchstart` → `startRecording()`, then the browser synthesised `mousedown` → `startRecording()` again. The guard `if (clientRef.current) return` prevented a double-connect, but `stopRecording()` fired twice on release — and the second call was a wasted no-op that could cause timing issues.

## The Correct Process

```tsx
// CORRECT — preventDefault suppresses synthesised mouse events
const handleTouchStart = useCallback((e: React.TouchEvent) => {
    e.preventDefault();
    startRecording();
}, [startRecording]);

const handleTouchEnd = useCallback((e: React.TouchEvent) => {
    e.preventDefault();
    stopRecording();
}, [stopRecording]);

<button
    onTouchStart={handleTouchStart}
    onTouchEnd={handleTouchEnd}
    onMouseDown={startRecording}
    onMouseUp={stopRecording}
>
```

`preventDefault()` on touch events tells the browser NOT to synthesise the corresponding mouse events. Desktop users (no touch) still get mouse events normally.

## Red Flags to Watch For

- Any element with both `onTouch*` and `onMouse*` handlers — guaranteed double-fire on mobile
- Push-to-talk / press-and-hold interactions — especially dangerous because the handler runs expensive async operations (mic access, WebSocket connections)
- "Works on desktop, weird on mobile" — classic symptom of touch event synthesis

## Prevention

1. **Always `preventDefault()` in touch handlers** when mouse handlers exist on the same element.
2. **Alternative:** Use only `onPointerDown`/`onPointerUp` (Pointer Events API) which unifies touch and mouse into a single event type. However, this may have different behaviour on older browsers/WebViews.
3. **Test on actual mobile devices** — desktop browser device emulation does NOT accurately reproduce touch event synthesis behaviour.
