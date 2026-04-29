# Remotion MotionPNGTuber

Codex and Claude Code plugin for adding MotionPNGTuber / MotionPNGTuber_UI
style talking characters and Japanese TTS narration to Remotion videos.

The bundled skill covers:

- Rendering MotionPNGTuber mouth sprites as a Remotion frame-driven canvas
  overlay from `mouth_track.json`.
- Preserving animated mouthless body video or synchronized extracted frames.
- Generating VOICEVOX or AivisSpeech narration through a local
  VOICEVOX-compatible HTTP API.
- Validating visual sync, transparent compositing, and final MP4 output.

## Install in Codex

Add this repository as a Codex plugin marketplace:

```bash
codex plugin marketplace add https://github.com/tegnike/remotion-motionpngtuber.git
```

Then restart Codex, open the plugin directory, select the
`Remotion MotionPNGTuber` marketplace, and install the plugin.

For local development from this folder:

```bash
codex plugin marketplace add /absolute/path/to/remotion-motionpngtuber
```

## Install in Claude Code

Add this repository as a Claude Code plugin marketplace:

```bash
claude plugin marketplace add tegnike/remotion-motionpngtuber
```

Then install the plugin:

```bash
claude plugin install remotion-motionpngtuber@remotion-motionpngtuber
```

For local development from this folder:

```bash
claude --plugin-dir /absolute/path/to/remotion-motionpngtuber
```

## Use in Codex

Invoke the skill explicitly in Codex:

```text
$remotion-motionpngtuber Add this MotionPNGTuber character to my Remotion composition using the provided asset directory and VOICEVOX engine.
```

## Use in Claude Code

Invoke the plugin skill explicitly in Claude Code:

```text
/remotion-motionpngtuber:remotion-motionpngtuber Add this MotionPNGTuber character to my Remotion composition using the provided asset directory and VOICEVOX engine.
```

The request should include the MotionPNGTuber asset directory, TTS engine URL,
speaker/model/style selection, and dialogue lines.
