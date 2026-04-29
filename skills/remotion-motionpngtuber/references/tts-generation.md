# VOICEVOX / AivisSpeech TTS Generation

Use this reference when the user wants dialogue audio generated as part of a Remotion MotionPNGTuber video.

## Inputs To Require

The user should provide:

- Engine type: `voicevox` or `aivisspeech`.
- Base URL: for example `http://localhost:50021` or `http://localhost:10101`.
- Model/speaker/style selection. This may be a numeric speaker/style ID, a speaker name, or a statement that only one model exists.
- Dialogue lines and any synthesis text overrides.
- Optional parameters such as `speedScale`, `pitchScale`, `intonationScale`, `volumeScale`, and output directory.

If model/style is ambiguous, query `/speakers` and map the user's requested name to a concrete style ID. If there is only one speaker/style, use it and state that inference.

## API Flow

Both engines are treated as VOICEVOX-compatible:

1. `GET {baseUrl}/speakers`
2. `POST {baseUrl}/audio_query?text={encodedText}&speaker={styleId}`
3. Modify the returned JSON for requested parameters, such as `speedScale = 1.15`.
4. `POST {baseUrl}/synthesis?speaker={styleId}` with the audio query JSON body.
5. Save the response as WAV.

Use `speaker` as the query parameter name for compatibility, even when the UI calls it model or style.

## Shell Pattern

```bash
BASE_URL="http://localhost:10101"
TEXT="こんにちは、ニケです。"
SPEAKER_ID="0"
OUT="public/expo-video/audio/voice-001.wav"

curl -s "$BASE_URL/speakers" > /tmp/speakers.json

curl -s -X POST \
  "$BASE_URL/audio_query?text=$(python3 -c 'import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))' "$TEXT")&speaker=$SPEAKER_ID" \
  > /tmp/audio_query.json

python3 - <<'PY'
import json
p = "/tmp/audio_query.json"
q = json.load(open(p))
q["speedScale"] = 1.15
json.dump(q, open(p, "w"), ensure_ascii=False)
PY

curl -s -X POST \
  -H "Content-Type: application/json" \
  --data-binary @/tmp/audio_query.json \
  "$BASE_URL/synthesis?speaker=$SPEAKER_ID" \
  > "$OUT"
```

For many lines, write a small script or loop that:

- Keeps `displayText` and `synthesisText` separate.
- Numbers files consistently: `voice-001.wav`, `voice-002.wav`, ...
- Writes a manifest with text, file path, duration seconds, and duration frames.

## Remotion Integration

Use generated durations to place audio and dialogue:

```tsx
const voiceFiles = ['voice-001.wav', 'voice-002.wav'];
const cueDurationFrames = [52, 74];
const cueStarts = [0, 52 + samePageGapFrames];

{voiceFiles.map((file, index) => (
  <Sequence key={file} from={cueStarts[index]}>
    <Audio src={staticFile(`expo-video/audio/${file}`)} volume={1} />
  </Sequence>
))}
```

Mouth state should use the same cue windows:

```tsx
const getActiveVoiceWindow = (frame: number) => {
  for (const cue of cues) {
    if (frame >= cue.start && frame < cue.start + cue.duration) return cue;
  }
  return null;
};
```

## Reading Overrides

When pronunciation differs from display text, synthesize a separate string:

```ts
{
  displayText: '次の一手を生成します。',
  synthesisText: 'つぎの一手を生成します。'
}
```

Common cases:

- English product names that should be read in Japanese.
- Kanji that the engine misreads.
- Symbols and abbreviations.

Do not change on-screen text solely to fix TTS pronunciation.

## Validation

After synthesis:

- Confirm every WAV exists and is non-empty.
- Use `ffprobe` or an audio-duration API to calculate duration.
- Spot-check at least one generated WAV if the engine/model changed.
- Re-render the mp4 and verify speech timing, subtitle timing, and mouth movement.
