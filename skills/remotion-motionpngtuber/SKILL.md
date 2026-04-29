---
name: remotion-motionpngtuber
description: Add MotionPNGTuber / MotionPNGTuber_UI style talking characters and Japanese TTS narration to Remotion or HyperFrames videos. Use when Codex needs to generate dialogue audio with VOICEVOX or AivisSpeech, place the audio on a Remotion or HyperFrames timeline, and render a PNGTuber character using a mouthless video or frame sequence, mouth_track.json, and mouth sprites; fix mouth alignment, green-screened assets, lip-sync timing, or render issues involving MotionPNGTuber in Remotion or HyperFrames.
---

# MotionPNGTuber for Remotion and HyperFrames

## Core Rule

Implement MotionPNGTuber as a frame-driven canvas overlay, not as pre-baked mouth overlay images unless the user explicitly asks for baked assets.

The MotionPNGTuber_UI browser player depends on runtime `HTMLVideoElement.currentTime`, `requestVideoFrameCallback`, WebAudio volume analysis, and DOM resize state. Do not paste that class directly into Remotion or HyperFrames. Port the important rendering model instead:

1. Render the mouthless character body.
2. Overlay a `<canvas>` with the same source coordinate system as `mouth_track.json`.
3. Load mouth sprites once before rendering the final frames.
4. Compute the track frame from Remotion's `useCurrentFrame()` or HyperFrames' seeked GSAP timeline time.
5. Draw the active mouth sprite into the tracked quad using the same two-triangle affine warp as MotionPNGTuber_UI.

Choose the runtime from the target project or the user's request:

- Use Remotion when the project is already a Remotion app or the user asks for Remotion.
- Use HyperFrames when the project is already a HyperFrames HTML composition or the user asks for HyperFrames.
- If neither runtime is present and the user did not specify one, ask which runtime they want instead of defaulting to either one.

If the body source is an animated mouthless video, the render must keep that body animation. Do not replace it with a single still frame as an optimization. If direct video rendering is too slow or unstable, extract the video into a frame sequence and render the frame matching the same `trackFrameIndex` used for the mouth canvas.

Also handle narration audio generation when the user provides a VOICEVOX-compatible engine. VOICEVOX and AivisSpeech should be treated as local HTTP TTS engines with the same basic flow: inspect `/speakers`, create an audio query with `/audio_query`, then synthesize WAV with `/synthesis`.

## Workflow

1. Confirm or extract required inputs:
   - Runtime: `remotion` or `hyperframes`, chosen from the existing project unless the user specifies one.
   - MotionPNGTuber asset directory containing `mouth_track.json`, `mouth/*.png`, and a mouthless body video or frame sequence.
   - If no MotionPNGTuber asset directory/model is provided, use the bundled default model at `../../assets/default-pngtuber/nike_loop_fix` relative to this `SKILL.md`. It contains `mouth_track.json`, `mouth/closed.png`, `mouth/half.png`, `mouth/open.png`, and `loop_mouthless_h264.mp4`.
   - When the target Remotion project needs public/static assets, copy the bundled default model into the target project, for example `public/pngtuber/nike_loop_fix`, and reference that copy in the composition.
   - When the target HyperFrames project needs local assets, copy the model into the project root, for example `pngtuber/nike_loop_fix`, and reference that copy from the HTML composition.
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

5. Drive synchronization from the chosen runtime:
   - In Remotion, use composition FPS for timeline position.
   - Convert to track FPS: `trackFrameIndex = Math.floor((loopFrame / compositionFps) * track.fps) % track.frames.length`.
   - Use the same `trackFrameIndex` for the body frame sequence when body frames are extracted from the same source.
   - A body frame sequence extracted from a source video must loop over `mouthTrack.frames.length`, not over the composition duration. This preserves the original MotionPNGTuber motion and keeps mouth tracking aligned.
   - In HyperFrames, use the seeked GSAP timeline time during preview/render and compute `trackFrameIndex = Math.floor((timelineTime % loopDurationSeconds) * track.fps) % track.frames.length`.
   - Do not drive HyperFrames mouth state from `requestAnimationFrame`, `Date.now()`, WebAudio realtime analysis, or a manually played video element during render.

6. Choose mouth state deterministically:
   - Prefer mouth event windows derived offline from generated WAV amplitude, with `start`, `end`, and `state`.
   - Use generated voice cue windows as a fallback, but animate at speech cadence rather than at track FPS.
   - Do not use `mouthTrack.fps` to decide how often the mouth opens; it only maps timeline time to tracked mouth/body coordinates.
   - Avoid WebAudio realtime analysis inside Remotion or HyperFrames renders.
   - Keep `closed`, `half`, `open` fallback order.

7. Validate visually and aurally:
   - Render at least one still where the character is speaking and one where the mouth should be closed.
   - Render or compare at least two stills several frames apart; verify the body changes, not only the mouth.
   - Check that green-screen or solid-color source backgrounds are transparent in the final composition.
   - Confirm generated audio is audible and aligned with subtitles/dialogue.
   - Check the generated mp4 frame, not only the Studio preview.
   - For Remotion, run the project's Remotion render/check commands.
   - For HyperFrames, run `hyperframes lint`, `hyperframes validate`, and a draft `hyperframes render`, then inspect the rendered MP4 or extracted frames.
   - If the mouth is invisible, verify sprite loading, canvas dimensions, alpha, z-index, and whether canvas styling matches the body.

## Reference

For reusable implementation details, read only what is needed:

- Remotion: `references/canvas-overlay-pattern.md`
- HyperFrames: `references/hyperframes-canvas-overlay-pattern.md`
- `references/tts-generation.md`

## Avoid

- Do not make a separate mouth video and paste it onto the face.
- Do not replace MotionPNGTuber behavior with static mouth PNGs positioned by hand.
- Do not replace an animated body source with a single still frame. If optimization is needed, use a synchronized transparent frame sequence.
- Do not leave a green-screen body source as-is in the composition. Key it to alpha or otherwise remove the background before final render.
- Do not describe either Remotion or HyperFrames as preferred by default. Pick the runtime from project context or the user's request.
- Do not use SVG `<image>` clipping as the primary implementation; it can fail in Remotion output depending on asset loading and alpha handling.
- Do not pre-bake `mouth-frames` unless explicitly requested or needed as an optimization after the canvas version is correct.
- Do not infer the TTS speaker/model silently. Use `/speakers` and the user's stated model/style.
