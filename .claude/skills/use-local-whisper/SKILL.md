---
name: use-local-whisper
description: Use when the user wants local voice transcription instead of OpenAI Whisper API. Switches to whisper.cpp running on Apple Silicon. WhatsApp only for now. Requires voice-transcription skill to be applied first.
---

# Use Local Whisper

Switches voice transcription from OpenAI's Whisper API to local whisper.cpp. Runs entirely on-device — no API key, no network, no cost.

**Channel support:** Currently WhatsApp only. The transcription module (`src/transcription.ts`) uses Baileys types for audio download. Other channels (Telegram, Discord, etc.) would need their own audio-download logic before this skill can serve them.

**Note:** The Homebrew package is `whisper-cpp`, but the CLI binary it installs is `whisper-cli`.

## Prerequisites

- `voice-transcription` skill must be applied first (WhatsApp channel)
- macOS with Apple Silicon (M1+) recommended
- `whisper-cpp` installed: `brew install whisper-cpp` (provides the `whisper-cli` binary)
- `ffmpeg` installed: `brew install ffmpeg`
- A GGML model file downloaded to `data/models/`

## Phase 1: Pre-flight

### Check if already applied

Check if `src/transcription.ts` already uses `whisper-cli`:

```bash
grep 'whisper-cli' src/transcription.ts && echo "Already applied" || echo "Not applied"
```

If already applied, skip to Phase 3 (Verify).

### Check dependencies are installed

```bash
whisper-cli --help >/dev/null 2>&1 && echo "WHISPER_OK" || echo "WHISPER_MISSING"
ffmpeg -version >/dev/null 2>&1 && echo "FFMPEG_OK" || echo "FFMPEG_MISSING"
```

If missing, install via your package manager (e.g., `pacman -S whisper.cpp ffmpeg` on Arch/Manjaro, or build from source).

### Check for model file

```bash
ls data/models/ggml-*.bin 2>/dev/null || echo "NO_MODEL"
```

If no model exists, download the base model (148MB, good balance of speed and accuracy):

```bash
mkdir -p data/models
curl -L -o data/models/ggml-base.bin "https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.bin"
```

For better accuracy at the cost of speed, use `ggml-small.bin` (466MB) or `ggml-medium.bin` (1.5GB).

## Phase 2: Apply Code Changes

### Ensure WhatsApp fork remote

```bash
git remote -v
```

If `whatsapp` is missing, add it:

```bash
git remote add whatsapp https://github.com/qwibitai/nanoclaw-whatsapp.git
```

### Merge the skill branch

```bash
git fetch whatsapp skill/local-whisper
git merge whatsapp/skill/local-whisper || {
  git checkout --theirs package-lock.json
  git add package-lock.json
  git merge --continue
}
```

This modifies `src/transcription.ts` to use the `whisper-cli` binary instead of the OpenAI API.

### Validate

```bash
npm run build
```

## Phase 3: Verify

### Ensure PATH includes whisper-cli and ffmpeg

The NanoClaw systemd service may run with a restricted PATH. Ensure `whisper-cli` and `ffmpeg` are accessible.

Check by running:

```bash
which whisper-cli ffmpeg
```

If the binaries are in a non-standard location, add the directory to the `Environment=PATH=...` in the systemd unit file, then reload:

```bash
systemctl --user daemon-reload
systemctl --user restart nanoclaw
```

### Build and restart

```bash
npm run build
systemctl --user restart nanoclaw
```

### Test

Send a voice note in any registered group. The agent should receive it as `[Voice: <transcript>]`.

### Check logs

```bash
tail -f logs/nanoclaw.log | grep -i -E "voice|transcri|whisper"
```

Look for:

- `Transcribed voice message` — successful transcription
- `whisper.cpp transcription failed` — check model path, ffmpeg, or PATH

## Configuration

Environment variables (optional, set in `.env`):

| Variable        | Default                     | Description                |
| --------------- | --------------------------- | -------------------------- |
| `WHISPER_BIN`   | `whisper-cli`               | Path to whisper.cpp binary |
| `WHISPER_MODEL` | `data/models/ggml-base.bin` | Path to GGML model file    |

## Troubleshooting

**"whisper.cpp transcription failed"**: Ensure both `whisper-cli` and `ffmpeg` are in PATH. The systemd service may use a restricted PATH — see Phase 3 above. Test manually:

```bash
ffmpeg -f lavfi -i anullsrc=r=16000:cl=mono -t 1 -f wav /tmp/test.wav -y
whisper-cli -m data/models/ggml-base.bin -f /tmp/test.wav --no-timestamps -nt
```

**Transcription works in dev but not as service**: The systemd service PATH may not include the directory containing `whisper-cli`. See "Ensure PATH includes whisper-cli and ffmpeg" in Phase 3.

**Slow transcription**: Check CPU usage — another process may be competing.

**Wrong language**: whisper.cpp auto-detects language. To force a language, you can set `WHISPER_LANG` and modify `src/transcription.ts` to pass `-l $WHISPER_LANG`.
