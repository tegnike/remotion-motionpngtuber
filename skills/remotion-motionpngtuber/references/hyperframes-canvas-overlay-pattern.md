# HyperFrames Canvas Overlay Pattern

Use this pattern when adding MotionPNGTuber mouth rendering to a HyperFrames
HTML composition.

## Component Shape

HyperFrames renders by seeking a paused GSAP timeline. Keep MotionPNGTuber
synchronization tied to the timeline's seeked time, not to realtime browser
callbacks.

```html
<div
  id="root"
  data-composition-id="motionpngtuber-demo"
  data-start="0"
  data-duration="8"
  data-width="1920"
  data-height="1080"
>
  <canvas id="body-canvas" width="1920" height="1080"></canvas>
  <canvas id="mouth-canvas" width="1920" height="1080"></canvas>
  <audio
    id="voice-001"
    data-start="0.4"
    data-duration="3.2"
    data-track-index="1"
    src="voice-001.wav"
    data-volume="1"
  ></audio>
</div>
```

```css
#body-canvas,
#mouth-canvas {
  position: absolute;
  left: 0;
  top: 0;
  width: 1920px;
  height: 1080px;
}

#body-canvas {
  z-index: 2;
}

#mouth-canvas {
  z-index: 3;
  pointer-events: none;
}
```

The two canvases must share the same source coordinate system as
`mouth_track.json`. If the body was chroma-keyed from a green-screen video,
export a transparent frame sequence at the original dimensions and draw the
body frame using the same `trackFrameIndex` as the mouth.

## Timeline-Driven Drawing

```html
<script src="https://cdn.jsdelivr.net/npm/gsap@3.14.2/dist/gsap.min.js"></script>
<script>
  window.__timelines = window.__timelines || {};

  const mouthTrack = window.MOTIONPNGTUBER_MOUTH_TRACK;
  const compositionDuration = 8;
  const modelPath = "pngtuber/nike_loop_fix";
  const bodyFrames = Array.from({ length: mouthTrack.frames.length }, (_, index) => {
    return `${modelPath}/body-transparent/frame-${String(index + 1).padStart(3, "0")}.png`;
  });
  const mouthSpriteSources = {
    closed: `${modelPath}/mouth/closed.png`,
    half: `${modelPath}/mouth/half.png`,
    open: `${modelPath}/mouth/open.png`,
  };
  const bodyCanvas = document.querySelector("#body-canvas");
  const mouthCanvas = document.querySelector("#mouth-canvas");
  const voiceCues = [
    { start: 0.4, duration: 3.2 },
  ];
  // Prefer generating these offline from WAV RMS peaks. They override the
  // syllable-rate fallback below and keep mouth movement tied to audio.
  const mouthEvents = [
    { start: 0.42, end: 0.56, state: "open" },
    { start: 0.56, end: 0.68, state: "half" },
    { start: 0.74, end: 0.88, state: "open" },
  ];

  const loadImage = (src) =>
    new Promise((resolve, reject) => {
      const image = new Image();
      image.onload = () => resolve(image);
      image.onerror = () => reject(new Error(`Failed to load image: ${src}`));
      image.src = src;
    });

  const assetsReady = Promise.all([
    Promise.all(bodyFrames.map(loadImage)),
    Promise.all(Object.entries(mouthSpriteSources).map(([state, src]) =>
      loadImage(src).then((image) => [state, image]),
    )),
  ]).then(([bodyImages, mouthEntries]) => {
    window.motionPngTuberAssets = {
      bodyImages,
      mouthSprites: Object.fromEntries(mouthEntries),
    };
    drawMotionPngTuberFrame(0);
  });

  const loopDurationSeconds = mouthTrack.frames.length / mouthTrack.fps;
  const getTrackFrameIndex = (timelineTime) =>
    Math.floor((timelineTime % loopDurationSeconds) * mouthTrack.fps) %
    mouthTrack.frames.length;

  // `mouthTrack.fps` is for tracked quad/body-frame sync only. Do not use it as
  // the mouth open/close cadence, or the mouth will flicker at video-frame rate.
  const getMouthState = (timelineTime) => {
    const explicitEvent = mouthEvents.find((event) => {
      return timelineTime >= event.start && timelineTime < event.end;
    });
    if (explicitEvent) return explicitEvent.state;

    const cue = voiceCues.find((candidate) => {
      return timelineTime >= candidate.start && timelineTime < candidate.start + candidate.duration;
    });
    if (!cue) return "closed";

    const localTime = timelineTime - cue.start;
    const mouthStepSeconds = 0.14; // Human syllable-like cadence, not track FPS.
    const fallbackPattern = ["half", "open", "half", "closed"];
    const fallbackIndex = Math.floor(localTime / mouthStepSeconds) % fallbackPattern.length;
    return fallbackPattern[fallbackIndex];
  };

  const drawMotionPngTuberFrame = (timelineTime) => {
    const assets = window.motionPngTuberAssets;
    if (!assets) return;

    const trackFrameIndex = getTrackFrameIndex(timelineTime);
    drawBodyFrame(
      bodyCanvas,
      assets.bodyImages[trackFrameIndex],
    );
    drawMouthFrame(
      mouthCanvas,
      assets.mouthSprites,
      getMouthState(timelineTime),
      trackFrameIndex,
    );
  };

  const tl = gsap.timeline({
    paused: true,
    onUpdate: () => drawMotionPngTuberFrame(tl.time()),
  });
  tl.to({}, { duration: compositionDuration, onUpdate: () => drawMotionPngTuberFrame(tl.time()) }, 0);

  assetsReady.catch((error) => {
    console.error(error);
  });

  window.__timelines["motionpngtuber-demo"] = tl;
</script>
```

Use the same `drawWarpedSprite`, two-triangle affine transform, and calibration
helpers from `canvas-overlay-pattern.md`. They are runtime-neutral because they
operate on `CanvasRenderingContext2D`, loaded `HTMLImageElement`s, and the
`mouth_track.json` quad data.

## Drawing Helpers

```js
const drawBodyFrame = (canvas, image) => {
  const context = canvas.getContext("2d");
  if (!context || !image) return;

  context.setTransform(1, 0, 0, 1, 0, 0);
  context.clearRect(0, 0, canvas.width, canvas.height);
  context.drawImage(image, 0, 0, canvas.width, canvas.height);
};

const drawMouthFrame = (canvas, sprites, mouthState, trackFrameIndex) => {
  const context = canvas.getContext("2d");
  if (!context) return;

  context.setTransform(1, 0, 0, 1, 0, 0);
  context.clearRect(0, 0, canvas.width, canvas.height);
  context.imageSmoothingEnabled = true;

  const frame = mouthTrack.frames[trackFrameIndex];
  if (!frame?.valid) return;

  const sprite = sprites[mouthState] ?? sprites.open ?? sprites.closed;
  if (!sprite) return;

  drawWarpedSprite(context, sprite, applyMouthCalibration(frame.quad));
};
```

## Asset Preparation

If the mouthless body video has a green or solid-color background, make a
transparent frame sequence before wiring the composition:

```bash
mkdir -p pngtuber/nike_loop_fix/body-transparent
ffmpeg -i pngtuber/nike_loop_fix/loop_mouthless_h264.mp4 \
  -vf "fps=30,colorkey=0x00aa00:0.24:0.04,format=rgba" \
  pngtuber/nike_loop_fix/body-transparent/frame-%03d.png
```

Keep the exported frame dimensions unchanged. The body canvas and mouth canvas
must both be `mouth_track.json` width and height, even if the visible character
is later scaled by a shared wrapper.

## Validation

Run these checks before considering the HyperFrames path complete:

```bash
hyperframes lint
hyperframes validate
hyperframes render --quality draft --fps 30 --output renders/motionpngtuber.mp4
ffmpeg -y -ss 1.0 -i renders/motionpngtuber.mp4 -frames:v 1 renders/speaking.png
ffmpeg -y -ss 4.5 -i renders/motionpngtuber.mp4 -frames:v 1 renders/closed.png
```

Inspect the extracted frames and confirm:

- The body changes across time.
- The mouth is visible and uses the tracked quad.
- Speaking mouth states change at speech cadence or from offline amplitude
  windows, not every render frame.
- The background is transparent or composited cleanly over the intended scene.
- Audio, subtitles, and mouth windows share the same cue timing.
