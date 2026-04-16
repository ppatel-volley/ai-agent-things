# Voice Input TV Audio Feedback Loop

**Severity:** Critical
**Sources:** discworld-trivia Fire TV deployment (2026-04-15)
**Category:** Voice Input, Recognition Service, TV Games

## Principle

In voice-input TV games, the phone microphone picks up audio from the TV speakers. If the mic stays active after submitting an answer, the TV's response ("Correct!", next question text, sound effects) is transcribed by the Recognition Service and dispatched as a new answer — creating a feedback loop that cycles through all questions in seconds with a score of 0.

## Details

### The feedback loop

```
1. Player holds mic, says "Rincewind"
2. Recognition Service sends finalTranscript → PROCESS_TRANSCRIPTION dispatched
3. Server advances to Q2, broadcasts new state
4. TV display says "Correct!" and shows Q2 text
5. Phone mic is STILL ACTIVE — picks up TV audio
6. Recognition Service transcribes TV audio → PROCESS_TRANSCRIPTION dispatched again
7. Server advances to Q3 (answer was wrong — score stays 0)
8. TV shows Q3 → mic picks it up → repeat
9. All 5 questions answered in ~2 seconds, final score: 0
```

### Why it only happens on real devices

- **Local dev:** display and controller are on the same computer — no speaker/mic path
- **E2E tests:** Playwright has no audio — Recognition SDK falls back to text input
- **Fire TV + phone:** TV speakers are 2-3 feet from phone mic — audio coupling is strong

### The root cause in code

The `onTranscript` callback dispatched `PROCESS_TRANSCRIPTION` but never stopped recording:

```typescript
// BAD — mic stays open after answer, picks up TV audio
.onTranscript((result) => {
    if (!result.finalTranscript) return
    dispatchThunk("PROCESS_TRANSCRIPTION", { text: result.finalTranscript })
    // mic is still active! TV audio → next transcript → next answer...
})
```

### The fix

Auto-stop recording after receiving a final transcript, with a cooldown before re-activation:

```typescript
// GOOD — stop mic after answer, cooldown prevents re-activation
.onTranscript((result) => {
    if (!result.finalTranscript) return
    dispatchThunk("PROCESS_TRANSCRIPTION", { text: result.finalTranscript })

    // Stop recording immediately
    const res = resourcesRef.current
    if (res?.client) {
        try { res.client.stopRecording() } catch {}
    }
    cleanup()

    // 2-second cooldown before mic can be re-activated
    setAnswerCooldown(true)
    setTimeout(() => setAnswerCooldown(false), 2000)
})

// Guard startRecording
const startRecording = useCallback(async () => {
    if (!recognitionSdk || isRecording || answerCooldown) return
    // ...
}, [isRecording, answerCooldown, ...])
```

### Why 2 seconds?

The cooldown must be long enough for:
- VGF state broadcast round-trip (~500ms)
- TV display re-render with new question (~200ms)
- TV text-to-speech or sound effects to finish (~1-2s)
- User to read the feedback ("Correct!") before re-engaging

2 seconds covers these. Shorter cooldowns risk the tail end of TV audio being picked up.

## Red flags

- Game works perfectly in local dev and E2E tests but "goes crazy" on Fire TV
- All questions answered instantly with score 0
- Player says "I only answered once"
- Recognition Service logs show rapid-fire finalTranscript events with TV dialogue content

## Prevention

1. **Always auto-stop after answer:** The `onTranscript` handler must stop recording after dispatching. Never leave the mic open across question boundaries.
2. **Cooldown timer:** Prevent re-activation for 2+ seconds after each answer.
3. **Visual feedback:** Show "Processing..." on the button during cooldown so the user knows the mic is intentionally disabled.
4. **Test on real hardware:** This bug is invisible in local dev, E2E tests, and simulators. Only real TV + phone with speaker/mic coupling exposes it.
5. **Consider push-to-talk only:** The hold-to-speak pattern (mic active only while button held) naturally prevents this — the user releases the button after speaking. The bug occurs if `onTranscript` fires after `stopRecording` was already called by the button release handler but before the Recognition Service has fully stopped.
