# VGF Session Auto-Creation for ProtoHub Launch

**Severity:** Critical
**Sources:** discworld-trivia E2E deployment (2026-04-15)
**Category:** VGF, ProtoHub, Session Management

## Principle

VGF's WGFServer does not auto-create sessions when a client connects with an unknown `sessionId`. ProtoHub passes a hub-generated UUID as the `sessionId` query parameter when launching games in an iframe, but this session doesn't exist on the game server. The display client must create a session via the WGFServer session API (`POST /api/session`) before establishing the Socket.IO connection, then connect with the **server-assigned** session ID — not the ProtoHub UUID.

## Details

### The launch flow

1. User selects a game in ProtoHub on Fire TV
2. ProtoHub generates a UUID (`bba297e8-5b8e-46ff-a4ce-e1292163729b`)
3. ProtoHub loads the game in an iframe: `https://game.volley-services.net/?sessionId=bba297e8-...&volley_hub_session_id=bba297e8-...`
4. Game's display client connects to VGF server with `sessionId=bba297e8-...`
5. **VGF server has no session with that ID** — returns empty state `{}`
6. `useStateSync()` returns `{}`, SceneRouter shows "Connecting..." forever

### Why dev mode works

`dev.ts` pre-creates a `dev-test` session on startup and re-creates it every 2 seconds. When testing locally with `?sessionId=dev-test`, the session always exists. Production has no such mechanism.

### The fix

The display provider must ensure a session exists before connecting:

```typescript
// 1. Check if ProtoHub's sessionId exists on the server
const headRes = await fetch(`${url}/api/session/${requestedSessionId}`, { method: "HEAD" })
if (headRes.ok) {
    // Session exists — use it directly
    setResolvedSessionId(requestedSessionId)
    return
}

// 2. Session doesn't exist — create a new one
const postRes = await fetch(`${url}/api/session`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
})
const data = await postRes.json()

// 3. Update URL with server-assigned ID (so controller can join)
const newUrl = new URL(window.location.href)
newUrl.searchParams.set("sessionId", data.sessionId)
window.history.replaceState({}, "", newUrl.toString())

// 4. Connect transport with the VALID session ID
setResolvedSessionId(data.sessionId)
```

**Critical:** The transport must NOT be created until `resolvedSessionId` is available. Use a `null` initial state and conditionally render a loading screen.

### Common mistakes

| Mistake | Result |
|---------|--------|
| Connect with ProtoHub's UUID without checking | `{}` state, blank screen |
| `POST /api/session` but still connect with original UUID | New session created but unused, still blank |
| Create transport in `useMemo` before `ensureSession` resolves | Race condition — transport connects with stale ID |
| Assume `POST /api/session` accepts a custom sessionId | WGFServer generates its own IDs — you must use the returned one |

### Red flags

- Game works locally (`?sessionId=dev-test`) but shows blank blue/dark screen on Fire TV via ProtoHub
- `useStateSync()` returns `{}` in production but works in dev
- No errors in console — VGF silently accepts connections to non-existent sessions
- Game works when accessed directly but not through ProtoHub's iframe

## Prevention

1. **Template default:** The hello-weekend template's `VGFDisplayProvider` should include session auto-creation out of the box.
2. **Integration test:** Add an E2E test that uses a random UUID sessionId (not `dev-test`) to catch this before deployment.
3. **Server logging:** Add a log line in WGFServer when a client connects to a non-existent session — the current silent failure makes debugging extremely difficult.
4. **ProtoHub feedback:** If the game iframe doesn't send a "ready" message within N seconds, ProtoHub should show an error rather than leaving the user on a blank screen.
