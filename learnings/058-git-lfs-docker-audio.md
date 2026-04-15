# Learning 058: Git LFS + Docker = Silent Audio Failure

**Severity:** Critical
**Sources:** Tempest 2026 — no audio on Fire TV
**Category:** Git, Docker, Deployment

## Principle

Git LFS stores pointer files (130 bytes of text) in the repository, with the actual binary content in a separate LFS server. Docker's `COPY . .` command copies the pointer files, NOT the actual binaries. If the Docker build doesn't run `git lfs pull`, the deployed server serves 130-byte text files instead of real MP3/WAV/image files. Audio and image loading fails silently — PlatformAudio/Howler reports "failed to load" with no clear indication that the file content is wrong.

**STATUS (2026-04-15):** Bifrost LFS support was added on 2026-04-09 but **reverted on 2026-04-15** by Cole — it broke the cluster buildpacks setup. A better implementation is planned. Until then, **do NOT use Git LFS for any files served at runtime** (audio, images, etc.). Store them as regular git objects. The Tempest 2026 repo was de-LFS'd in PR #4.

## The Mistake

Audio files were tracked by Git LFS:
```
*.mp3 filter=lfs diff=lfs merge=lfs -text
```

The Dockerfile used `COPY . .` which copied LFS pointer files:
```
version https://git-lfs.github.com/spec/v1
oid sha256:8967413425b08237d9eab58cec2a5614971f93feca09f93056a826e3ec0b5684
size 28800
```

The production server served this text at `/laser.mp3`. PlatformAudio tried to decode it as audio, failed, and reported "failed to load" — but the HTTP response was 200 OK with 130 bytes.

## How to Diagnose

```bash
# Check if the deployed file is an LFS pointer
curl -s "https://your-game.volley-services.net/laser.mp3" | head -1
# If it starts with "version https://git-lfs.github.com/spec/v1" — it's a pointer

# Check file size — real MP3 will be >1KB
curl -s -o /dev/null -w "%{size_download}" "https://your-game.volley-services.net/laser.mp3"
# 130 = pointer, 28800 = real file
```

## The Fix (if Bifrost LFS support is unavailable)

Remove the file type from LFS tracking and re-add files as regular git objects:

```bash
git lfs untrack "*.mp3"
git rm --cached path/to/*.mp3
git add path/to/*.mp3
git commit -m "fix: remove MP3 from LFS"
```

## Additional Audio Lessons from This Investigation

1. **Use simple filenames** — spaces and UTF-8 characters (smart quotes) in audio paths fail on some platforms. Use `thermal-resolution.mp3` not `01. Thermal Resolution.mp3`
2. **Put audio in public/ root** — Song Quiz puts all audio directly in `public/` with bare filenames (`CorrectAnswer.mp3`). Nested paths can fail with PlatformAudio.
3. **Preload during lobby** — creating PlatformAudio on first play adds 1-2s load delay. Call `audio.load()` during the lobby phase.
4. **Lazy-init SFX** — don't create PlatformAudio at module level (before PlatformProvider is ready). Create on first use or during lobby mount.

## Red Flags to Watch For

- Audio "works locally but not deployed" — check for LFS pointers
- `audio.load()` Promise rejecting with no network error — file might be a text pointer
- HTTP 200 response with tiny file size for a media asset
- `.gitattributes` with `filter=lfs` for any asset type used at runtime
