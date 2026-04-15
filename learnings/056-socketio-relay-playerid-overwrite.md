# Learning 056: Don't Overwrite playerId in Socket.IO Relay

**Severity:** Critical
**Sources:** Tempest 2026 — phone controller tilt moving player off-tube
**Category:** Socket.IO, VGF, Multiplayer

## Principle

When relaying direct Socket.IO events (outside VGF) from controllers to the display, do NOT overwrite the `playerId` field with `socket.id`. The controller sends its VGF `userId` as `playerId`, which must match `state.players[].playerId` on the display for the player to be rendered correctly.

## The Mistake

The server relay overwrote `playerId`:

```typescript
// WRONG — replaces controller's userId with a random Socket.IO connection ID
socket.on("player-input", (data) => {
    socket.to(sessionId).emit("player-input", {
        ...data,
        playerId: socket.id,  // "abc123xyz" — doesn't match VGF state
    })
})
```

The display stored the input under `socket.id`, but looked up player colour/slot via `state.players.find(p => p.playerId === playerId)`. Since `socket.id` never matched any VGF player entry, the blaster rendered with fallback colour and the camera didn't follow it correctly.

Additionally, a "D-pad ghost player" was always present at segment 0 (created by `usePlayerInput` for keyboard fallback). With two players in the input map, the camera followed the D-pad player instead of the phone player.

## The Correct Process

```typescript
// CORRECT — relay the controller's own playerId as-is
socket.on("player-input", (data) => {
    socket.to(sessionId).emit("player-input", data)
})
```

And remove the D-pad player from the input map when a phone controller connects:

```typescript
// In the display's Socket.IO listener
if (data.playerId !== DPAD_PLAYER_ID && inputsRef.current.has(DPAD_PLAYER_ID)) {
    inputsRef.current.delete(DPAD_PLAYER_ID)
}
```

## Red Flags to Watch For

- `socket.id` used as a player identity — it's a connection ID that changes on reconnect
- Multiple entries in a player input map with unexpected keys
- Camera following the wrong player
- "Player moves off the tube" — likely wrong segment lookup due to mismatched playerId

## Prevention

1. **Never overwrite identity fields** in relay handlers — pass data through unchanged
2. **Use VGF `userId`** as the canonical player identity, not Socket.IO `socket.id`
3. **Remove fallback/ghost players** when a real player connects
