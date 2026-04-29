# Remotion MotionPNGTuber

Codex plugin for adding MotionPNGTuber / MotionPNGTuber_UI style talking
characters and Japanese TTS narration to Remotion videos.

The bundled skill covers:

- Rendering MotionPNGTuber mouth sprites as a Remotion frame-driven canvas
  overlay from `mouth_track.json`.
- Preserving animated mouthless body video or synchronized extracted frames.
- Generating VOICEVOX or AivisSpeech narration through a local
  VOICEVOX-compatible HTTP API.
- Validating visual sync, transparent compositing, and final MP4 output.

## Install

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

## Use

Invoke the skill explicitly in Codex:

```text
$remotion-motionpngtuber Add this MotionPNGTuber character to my Remotion composition using the provided asset directory and VOICEVOX engine.
```

The request should include the MotionPNGTuber asset directory, TTS engine URL,
speaker/model/style selection, and dialogue lines.
