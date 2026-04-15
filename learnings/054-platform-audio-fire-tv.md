# Learning 054: PlatformAudio for Fire TV — No Raw HTMLAudioElement

**Severity:** Critical
**Sources:** Tempest 2026 — audio playback on Fire TV
**Category:** Audio, Fire TV, Platform SDK

## Principle

On Fire TV (Chrome 68 in VWR iframe), raw `HTMLAudioElement` and direct `AudioContext` usage fail silently. Always use `PlatformAudio` from `@volley/platform-sdk/lib` for audio playback in Volley TV games. This wraps Howler.js with Fire TV-specific fixes.

## The Mistake

Multiple approaches were tried to play audio on Fire TV, all of which worked in desktop Chrome but failed silently on the TV:

1. **HTMLAudioElement with `new Audio(src)`** — autoplay policy blocked play(), `.catch(() => {})` swallowed the error
2. **Single Audio element with `src` swapping** — `audio.load()` resets the browser's "user-gesture-activated" flag, so subsequent `play()` calls from React `useEffect` (not in gesture call stack) were blocked
3. **Pre-created Audio elements at module level** — HMR recreated them, losing gesture activation state
4. **Web Audio API with `AudioContext`** — `ctx.resume()` needs to be in a user gesture call stack; async `fetch()` + `decodeAudioData()` broke the gesture chain even when `resume()` was called synchronously
5. **User gesture retry listeners** — unreliable timing, especially during React phase transitions

All of these produced working audio on desktop and complete silence on Fire TV.

## The Correct Process

```typescript
import { PlatformAudio } from "@volley/platform-sdk/lib"

const audio = new PlatformAudio({
    src: "assets/audio/effects/laser.mp3",  // relative to public/
    preload: true,
    autoplay: false,
    html5: false,     // CRITICAL: forces Web Audio API mode
    volume: 0.7,
    loop: false,
})

await audio.load()
audio.play()

// Events
audio.once("end", () => { /* track finished */ })
audio.once("loaderror", () => { /* failed to load */ })

// Cleanup
audio.pause()
audio.destroy()
```

**Key settings:**
- `html5: false` — forces Web Audio API mode which works on Chrome 68. Without this, Howler falls back to HTML5 Audio which has the same autoplay issues.
- `preload: true` — begins loading during construction
- Call `audio.load()` before `audio.play()` for reliability

**For SFX pools (overlapping sounds like rapid-fire laser):**
```typescript
const pool = Array.from({ length: 4 }, () => new PlatformAudio({
    src: "assets/audio/effects/laser.mp3",
    preload: true, html5: false, volume: 0.15,
}))
pool.forEach(a => a.load().catch(() => {}))

let index = 0
function playLaser() {
    const audio = pool[index++ % pool.length]
    audio.pause()  // Note: no instance .stop() method
    audio.play()
}
```

**For music playlists (sequential tracks):**
```typescript
function playTrack(idx: number) {
    const audio = new PlatformAudio({ src: tracks[idx], html5: false, volume: 0.6 })
    audio.load().then(() => {
        audio.once("end", () => { audio.destroy(); playTrack(idx + 1) })
        audio.play()
    })
}
```

## Red Flags to Watch For

- Any `new Audio()` or `new AudioContext()` in a display app — should be `new PlatformAudio()` instead
- `audio.play().catch(() => {})` — silently swallowed autoplay rejection, gives false confidence in desktop testing
- `audio.load()` on an HTMLAudioElement before `.play()` — resets gesture activation
- Audio that "works on localhost but not on Fire TV" — almost certainly a PlatformAudio vs raw Audio issue
- `html5: true` or omitting the `html5` option — Howler defaults to HTML5 Audio which has the same problems

## Prevention

1. **Always use `PlatformAudio`** for any audio in a display (TV) app. No exceptions.
2. **Reference Song Quiz and Wheel of Fortune** for audio patterns — both use `PlatformAudio` exclusively.
3. **Test on actual Fire TV hardware** — desktop Chrome does not reproduce Fire TV audio policy behaviour.
4. **Set `html5: false`** on every PlatformAudio instance to force Web Audio API mode.
5. **No instance `.stop()` method** exists on PlatformAudio — use `.pause()` instead. `.stop()` is a static method.

## Update (2026-04-09): Bifrost now supports Git LFS

Cole updated Bifrost to resolve Git LFS pointers during Docker builds. Previously, `COPY . .` in the Dockerfile would copy 130-byte LFS pointer text files instead of actual binaries. This has been fixed — LFS-tracked files should now work correctly in deployed builds. However, the other fixes in this learning (PlatformAudio, simple filenames, lazy init) are still required regardless of LFS support.
