---
name: remotion-motionpngtuber
description: Add MotionPNGTuber / MotionPNGTuber_UI style talking characters and Japanese TTS narration to Remotion videos. Use when Codex needs to generate dialogue audio with VOICEVOX or AivisSpeech, place the audio on a Remotion timeline, and render a PNGTuber character using a mouthless video or frame sequence, mouth_track.json, and mouth sprites; fix mouth alignment, green-screened assets, lip-sync timing, or render issues involving MotionPNGTuber in Remotion.
---

# Remotion MotionPNGTuber

## Core Rule

Implement MotionPNGTuber in Remotion as a frame-driven canvas overlay, not as pre-baked mouth overlay images unless the user explicitly asks for baked assets.

The MotionPNGTuber_UI browser player depends on runtime `HTMLVideoElement.currentTime`, `requestVideoFrameCallback`, WebAudio volume analysis, and DOM resize state. Do not paste that class directly into Remotion. Port the important rendering model instead:

1. Render the mouthless character body.
2. Overlay a `<canvas>` with the same source coordinate system as `mouth_track.json`.
3. Load mouth sprites once with `delayRender`.
4. On each Remotion frame, compute the track frame from `useCurrentFrame()`.
5. Draw the active mouth sprite into the tracked quad using the same two-triangle affine warp as MotionPNGTuber_UI.

If the body source is an animated mouthless video, the Remotion render must keep that body animation. Do not replace it with a single still frame as an optimization. If direct `<Video>` rendering is too slow or unstable, extract the video into a frame sequence and render the frame matching the same `trackFrameIndex` used for the mouth canvas.

Also handle narration audio generation when the user provides a VOICEVOX-compatible engine. VOICEVOX and AivisSpeech should be treated as local HTTP TTS engines with the same basic flow: inspect `/speakers`, create an audio query with `/audio_query`, then synthesize WAV with `/synthesis`.

## Workflow

1. Confirm or extract required inputs:
   - MotionPNGTuber asset directory containing `mouth_track.json`, `mouth/*.png`, and a mouthless body video or frame sequence.
   - TTS engine type: `voicevox` or `aivisspeech`.
   - TTS base URL, such as `http://localhost:50021` or `http://localhost:10101`.
   - TTS model/speaker/style selection. Do not guess; inspect `/speakers` when the user provides only a model name or says there is one model.
   - Dialogue lines, intended order, and any requested speech parameters such as `speedScale`.

2. Inspect the MotionPNGTuber assets:
   - `mouth_track.json`: note `fps`, `width`, `height`, `frames[].quad`, `calibration`, and `calibrationApplied`.
   - mouth sprites: at least `mouth/closed.png` and `mouth/open.png`; use `half.png` if present.
   - body source: mouthless video, alpha video, or extracted transparent frame sequence.
   - If the body source has a green-screen or solid-color background, treat it as not yet compositing-ready. Confirm whether alpha is present; if not, extract frames, chroma-key the background to alpha, and keep the frame dimensions unchanged.

3. Generate TTS audio:
   - Use the VOICEVOX-compatible flow in `references/tts-generation.md`.
   - Save one WAV per dialogue cue with stable names such as `voice-001.wav`.
   - Record durations with `ffprobe` or Remotion/Mediabunny and convert them to frame counts.
   - If a screen label and TTS reading differ, keep display text separate from synthesis text.

4. Preserve coordinate systems:
   - Canvas `width` and `height` must match `mouth_track.json` source dimensions.
   - Style the canvas with the exact same `left`, `top`, `width`, `height`, scale, clip, and crop as the body source.
   - If the character body is preprocessed from green screen, keep frame dimensions unchanged so the track coordinates still match.
   - If chroma-keying leaves dark or green edge pixels at the frame border, apply the same small `clipPath: inset(...)` or equivalent crop to both the body and the canvas.

5. Drive synchronization from Remotion frames:
   - Use composition FPS for timeline position.
   - Convert to track FPS: `trackFrameIndex = Math.floor((loopFrame / compositionFps) * track.fps) % track.frames.length`.
   - Use the same `trackFrameIndex` for the body frame sequence when body frames are extracted from the same source.
   - A body frame sequence extracted from a source video must loop over `mouthTrack.frames.length`, not over the composition duration. This preserves the original MotionPNGTuber motion and keeps mouth tracking aligned.

6. Choose mouth state deterministically:
   - Use generated voice cue windows by default.
   - Optionally derive rough amplitude windows offline from generated WAV files.
   - Avoid WebAudio realtime analysis inside Remotion renders.
   - Keep `closed`, `half`, `open` fallback order.

7. Validate visually and aurally:
   - Render at least one still where the character is speaking and one where the mouth should be closed.
   - Render or compare at least two stills several frames apart; verify the body changes, not only the mouth.
   - Check that green-screen or solid-color source backgrounds are transparent in the final composition.
   - Confirm generated audio is audible and aligned with subtitles/dialogue.
   - Check the generated mp4 frame, not only the Studio preview.
   - If the mouth is invisible, verify sprite loading, canvas dimensions, alpha, z-index, and whether canvas styling matches the body.

## Reference

For reusable implementation details, read only what is needed:

- `references/canvas-overlay-pattern.md`
- `references/tts-generation.md`

## Avoid

- Do not make a separate mouth video and paste it onto the face.
- Do not replace MotionPNGTuber behavior with static mouth PNGs positioned by hand.
- Do not replace an animated body source with a single still frame. If optimization is needed, use a synchronized transparent frame sequence.
- Do not leave a green-screen body source as-is in the composition. Key it to alpha or otherwise remove the background before final render.
- Do not use SVG `<image>` clipping as the primary implementation; it can fail in Remotion output depending on asset loading and alpha handling.
- Do not pre-bake `mouth-frames` unless explicitly requested or needed as an optimization after the canvas version is correct.
- Do not infer the TTS speaker/model silently. Use `/speakers` and the user's stated model/style.
