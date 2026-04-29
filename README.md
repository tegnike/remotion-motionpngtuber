# MotionPNGTuber for Remotion and HyperFrames

Codex and Claude Code plugin for adding MotionPNGTuber / MotionPNGTuber_UI
style talking characters and Japanese TTS narration to Remotion or HyperFrames
videos.

The bundled skill covers:

- Rendering MotionPNGTuber mouth sprites as a frame-driven canvas overlay from
  `mouth_track.json` in either Remotion or HyperFrames.
- Preserving animated mouthless body video or synchronized extracted frames.
- Falling back to the bundled default PNGTuber model in
  `assets/default-pngtuber/nike_loop_fix` when no model is specified.
- Generating VOICEVOX or AivisSpeech narration through a local
  VOICEVOX-compatible HTTP API.
- Validating visual sync, transparent compositing, and final MP4 output.

## Install in Codex

Add this repository as a Codex plugin marketplace:

```bash
codex plugin marketplace add https://github.com/tegnike/remotion-motionpngtuber.git
```

Then restart Codex, open the plugin directory, select the
`MotionPNGTuber for Remotion and HyperFrames` marketplace, and install the
plugin.

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
$remotion-motionpngtuber Add this MotionPNGTuber character to my video using HyperFrames, the provided asset directory, and VOICEVOX engine.
```

or:

```text
$remotion-motionpngtuber Add this MotionPNGTuber character to my Remotion composition using the provided asset directory and VOICEVOX engine.
```

## Use in Claude Code

Invoke the plugin skill explicitly in Claude Code:

```text
/remotion-motionpngtuber:remotion-motionpngtuber Add this MotionPNGTuber character to my video using Remotion or HyperFrames, the provided asset directory, and VOICEVOX engine.
```

The request should include the TTS engine URL, speaker/model/style selection,
dialogue lines, and preferred runtime when it matters. If no runtime is
specified, choose the one that matches the target project already present in the
workspace. If no MotionPNGTuber asset directory is provided, the skill uses the
bundled default model.
