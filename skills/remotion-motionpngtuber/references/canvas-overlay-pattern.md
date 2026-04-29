# Canvas Overlay Pattern

Use this pattern when adding MotionPNGTuber mouth rendering to a Remotion composition.

## Component Shape

```tsx
import React from 'react';
import {
  cancelRender,
  continueRender,
  delayRender,
  staticFile,
  useCurrentFrame,
} from 'remotion';
import mouthTrackData from '../path/to/mouth_track.json';

type MouthState = 'closed' | 'half' | 'open';
type MouthPoint = [number, number];

type MouthTrackFrame = {
  quad: MouthPoint[];
  valid: boolean;
};

type MouthTrack = {
  fps: number;
  width: number;
  height: number;
  calibrationApplied?: boolean;
  calibration?: {
    offset?: MouthPoint;
    scale?: number;
    rotation?: number;
  };
  frames: MouthTrackFrame[];
};

const mouthTrack = mouthTrackData as unknown as MouthTrack;
const mouthStates: MouthState[] = ['closed', 'half', 'open'];

export const MotionPngTuberCharacter = ({
  compositionFps,
  renderBody,
  getMouthState,
  bodyStyle,
  canvasStyle,
}: {
  compositionFps: number;
  renderBody: (trackFrameIndex: number) => React.ReactNode;
  getMouthState: (frame: number) => MouthState;
  bodyStyle?: React.CSSProperties;
  canvasStyle: React.CSSProperties;
}) => {
  const frame = useCurrentFrame();
  const loopFrames = Math.round((mouthTrack.frames.length / mouthTrack.fps) * compositionFps);
  const loopFrame = ((frame % loopFrames) + loopFrames) % loopFrames;
  const trackFrameIndex =
    Math.floor((loopFrame / compositionFps) * mouthTrack.fps) % mouthTrack.frames.length;

  return (
    <div style={{position: 'absolute', inset: 0}}>
      <div style={bodyStyle}>{renderBody(trackFrameIndex)}</div>
      <MotionPngTuberMouthCanvas
        mouthState={getMouthState(frame)}
        trackFrameIndex={trackFrameIndex}
        style={canvasStyle}
      />
    </div>
  );
};
```

If the body is an extracted frame sequence from the same source video, derive the filename from the same `trackFrameIndex`.

Do not use a single still frame for an animated body. A still frame can make the mouth move while the character body is frozen, which is not a correct MotionPNGTuber port.

For an extracted transparent frame sequence, pass the computed `trackFrameIndex` to the body renderer:

```tsx
const TransparentBodyFrame = ({trackFrameIndex}: {trackFrameIndex: number}) => {
  const fileName = `character/body-transparent/frame-${String(trackFrameIndex).padStart(3, '0')}.png`;

  return (
    <Img
      src={staticFile(fileName)}
      style={{
        width: '100%',
        height: '100%',
        objectFit: 'contain',
        display: 'block',
      }}
    />
  );
};
```

## Canvas Mouth Component

```tsx
const MotionPngTuberMouthCanvas = ({
  mouthState,
  trackFrameIndex,
  style,
}: {
  mouthState: MouthState;
  trackFrameIndex: number;
  style: React.CSSProperties;
}) => {
  const canvasRef = React.useRef<HTMLCanvasElement>(null);
  const spritesRef = React.useRef<Partial<Record<MouthState, HTMLImageElement>>>({});
  const [ready, setReady] = React.useState(false);
  const [loadHandle] = React.useState(() => delayRender('Load MotionPNGTuber mouth sprites'));

  React.useEffect(() => {
    let canceled = false;

    Promise.all(
      mouthStates.map(
        (state) =>
          new Promise<[MouthState, HTMLImageElement]>((resolve, reject) => {
            const image = new Image();
            image.onload = () => resolve([state, image]);
            image.onerror = () => reject(new Error(`Failed to load mouth sprite: ${state}`));
            image.src = staticFile(`path/to/mouth/${state}.png`);
          }),
      ),
    )
      .then((sprites) => {
        if (canceled) return;
        spritesRef.current = Object.fromEntries(sprites) as Partial<
          Record<MouthState, HTMLImageElement>
        >;
        setReady(true);
        continueRender(loadHandle);
      })
      .catch((error) => {
        if (!canceled) cancelRender(error);
      });

    return () => {
      canceled = true;
    };
  }, [loadHandle]);

  React.useLayoutEffect(() => {
    const canvas = canvasRef.current;
    if (!ready || !canvas) return;
    drawMotionPngTuberMouth(canvas, spritesRef.current, mouthState, trackFrameIndex);
  }, [mouthState, ready, trackFrameIndex]);

  return (
    <canvas
      ref={canvasRef}
      width={mouthTrack.width}
      height={mouthTrack.height}
      style={style}
    />
  );
};
```

## Drawing Helpers

```tsx
const drawMotionPngTuberMouth = (
  canvas: HTMLCanvasElement,
  sprites: Partial<Record<MouthState, HTMLImageElement>>,
  mouthState: MouthState,
  trackFrameIndex: number,
) => {
  const context = canvas.getContext('2d');
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

const applyMouthCalibration = (quad: MouthPoint[]) => {
  const calibration = mouthTrack.calibration ?? {offset: [0, 0], scale: 1, rotation: 0};
  if (!mouthTrack.calibrationApplied) {
    return quad.map(([x, y]) => [x, y] as MouthPoint);
  }

  const offset = calibration.offset ?? [0, 0];
  const scale = calibration.scale ?? 1;
  const rotation = ((calibration.rotation ?? 0) * Math.PI) / 180;
  const centerX = quad.reduce((sum, [x]) => sum + x, 0) / quad.length;
  const centerY = quad.reduce((sum, [, y]) => sum + y, 0) / quad.length;
  const cos = Math.cos(rotation);
  const sin = Math.sin(rotation);

  return quad.map(([x, y]) => {
    const dx = (x - centerX) * scale;
    const dy = (y - centerY) * scale;
    return [
      dx * cos - dy * sin + centerX + offset[0],
      dx * sin + dy * cos + centerY + offset[1],
    ] as MouthPoint;
  });
};

const drawWarpedSprite = (
  context: CanvasRenderingContext2D,
  sprite: HTMLImageElement,
  quad: MouthPoint[],
) => {
  const spriteWidth = sprite.naturalWidth || sprite.width;
  const spriteHeight = sprite.naturalHeight || sprite.height;
  if (!spriteWidth || !spriteHeight) return;

  const s0: MouthPoint = [0, 0];
  const s1: MouthPoint = [spriteWidth, 0];
  const s2: MouthPoint = [spriteWidth, spriteHeight];
  const s3: MouthPoint = [0, spriteHeight];
  const [q0, q1, q2, q3] = quad;

  drawTriangle(context, sprite, s0, s1, s2, q0, q1, q2);
  drawTriangle(context, sprite, s0, s2, s3, q0, q2, q3);
};

const drawTriangle = (
  context: CanvasRenderingContext2D,
  image: HTMLImageElement,
  sourceA: MouthPoint,
  sourceB: MouthPoint,
  sourceC: MouthPoint,
  targetA: MouthPoint,
  targetB: MouthPoint,
  targetC: MouthPoint,
) => {
  const matrix = computeAffine(sourceA, sourceB, sourceC, targetA, targetB, targetC);
  if (!matrix) return;

  context.save();
  context.setTransform(1, 0, 0, 1, 0, 0);
  context.beginPath();
  context.moveTo(targetA[0], targetA[1]);
  context.lineTo(targetB[0], targetB[1]);
  context.lineTo(targetC[0], targetC[1]);
  context.closePath();
  context.clip();
  context.setTransform(matrix.a, matrix.b, matrix.c, matrix.d, matrix.e, matrix.f);
  context.drawImage(image, 0, 0);
  context.restore();
};

const computeAffine = (
  sourceA: MouthPoint,
  sourceB: MouthPoint,
  sourceC: MouthPoint,
  targetA: MouthPoint,
  targetB: MouthPoint,
  targetC: MouthPoint,
) => {
  const [sx0, sy0] = sourceA;
  const [sx1, sy1] = sourceB;
  const [sx2, sy2] = sourceC;
  const [dx0, dy0] = targetA;
  const [dx1, dy1] = targetB;
  const [dx2, dy2] = targetC;
  const denominator = sx0 * (sy1 - sy2) + sx1 * (sy2 - sy0) + sx2 * (sy0 - sy1);

  if (denominator === 0) return null;

  return {
    a: (dx0 * (sy1 - sy2) + dx1 * (sy2 - sy0) + dx2 * (sy0 - sy1)) / denominator,
    b: (dy0 * (sy1 - sy2) + dy1 * (sy2 - sy0) + dy2 * (sy0 - sy1)) / denominator,
    c: (dx0 * (sx2 - sx1) + dx1 * (sx0 - sx2) + dx2 * (sx1 - sx0)) / denominator,
    d: (dy0 * (sx2 - sx1) + dy1 * (sx0 - sx2) + dy2 * (sx1 - sx0)) / denominator,
    e:
      (dx0 * (sx1 * sy2 - sx2 * sy1) +
        dx1 * (sx2 * sy0 - sx0 * sy2) +
        dx2 * (sx0 * sy1 - sx1 * sy0)) /
      denominator,
    f:
      (dy0 * (sx1 * sy2 - sx2 * sy1) +
        dy1 * (sx2 * sy0 - sx0 * sy2) +
        dy2 * (sx0 * sy1 - sx1 * sy0)) /
      denominator,
  };
};
```

## Styling Rule

The most common alignment bug is styling the body and canvas differently. Keep these identical:

```tsx
const characterSourceStyle: React.CSSProperties = {
  position: 'absolute',
  left: characterLeft,
  top: characterTop,
  width: trackWidth * characterScale,
  height: trackHeight * characterScale,
  clipPath: 'inset(2px 0 0 0)',
};
```

Use `characterSourceStyle` for both the body image/video and the canvas unless the body is wrapped in a clipping container that already applies the same transform.

If chroma-keying leaves a thin source-border line, crop it in the shared style rather than only on the body:

```tsx
const characterSourceStyle: React.CSSProperties = {
  position: 'absolute',
  left: characterLeft,
  top: characterTop,
  width: trackWidth * characterScale,
  height: trackHeight * characterScale,
  clipPath: 'inset(8px 0 8px 0)',
};
```

## Green-Screen Body Sources

Some MotionPNGTuber mouthless body videos are H.264 videos with a green or solid-color background instead of alpha. In that case:

1. Extract every body frame at the source or track FPS.
2. Chroma-key the background to alpha.
3. Preserve the original frame dimensions.
4. Render the transparent frame sequence using the same `trackFrameIndex` as the mouth canvas.
5. Validate against a non-green page background so missed keying is obvious.

Do not leave the source green-screen video directly in the composition. Also do not work around render cost by replacing it with one PNG still.
