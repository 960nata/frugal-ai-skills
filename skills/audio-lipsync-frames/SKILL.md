---
name: audio-lipsync-frames
description: Drive a 2D character's mouth from voice-over loudness (not phonemes) to produce cheap, stable lip-sync frames for Remotion or any frame-based video renderer, with smoothing, pause closing, and a silent-preview fallback from word timing. Use when animating talking characters, mascots, VTuber-style sprites, or reels/shorts narrators from TTS audio.
---

# Audio-driven lip-sync frames

Phoneme/viseme recognition is heavy and fragile. For short-form video a sprite with
**three mouth drawings** (closed / small / wide) driven by loudness looks natural and
never glitches.

## Steps

1. **Measure loudness per video frame** (e.g. Remotion `visualizeAudio` /
   `getAudioData`, or precompute RMS per `1/fps` window with ffmpeg). Normalize to 0–1.
2. **Sample at ~10 Hz, not every frame.** Mouths that change 30×/s flicker. Use
   `cadence = round(fps / 10)` and hold each sample.
3. **Smooth** with a 1-2-1 kernel: `(prev + 2·cur + next) / 4`.
4. **Close on real pauses**, even between samples: if max(level) of the frame and its
   neighbours ≤ 0.08 → closed.
5. **Map to mouth shapes**: `< 0.10 → 1 (closed)`, `< 0.55 → 2 (small)`, else `3 (wide)`.
6. Mouth only — do not drive arms/body from loudness; body acting is a separate track.
7. **Silent preview fallback**: without audio, estimate per-word timing inside the
   narration window by weighting word length (+ extra weight for punctuation pauses)
   and alternate shapes inside words.

## Reference

```ts
export interface TalkFrame { mouth: 1 | 2 | 3; level: number }
export const REST: TalkFrame = { mouth: 1, level: 0 }

export function buildTalkTrack(levels: number[], start: number, end: number, fps = 30): TalkFrame[] {
  const cadence = Math.max(1, Math.round(fps / 10))
  const at = (f: number) => (f < start || f >= end ? 0 : Math.max(0, Math.min(1, levels[f] ?? 0)))
  return Array.from({ length: Math.max(0, end - start) }, (_, i) => {
    const frame = start + i
    const s = start + Math.floor(i / cadence) * cadence
    const level = (at(s - 1) + at(s) * 2 + at(s + 1)) / 4
    if (Math.max(at(frame - 1), at(frame), at(frame + 1)) <= 0.08) return REST
    return { mouth: level < 0.1 ? 1 : level < 0.55 ? 2 : 3, level }
  })
}
```

## Asset tips
- Draw mouths as separate transparent PNGs aligned to the same anchor as the face.
- Keep one expression sprite per emotion (smile, shocked, thinking…) and swap mouths on top.
- Two characters (duet): give each its own narration window; the idle one stays `REST`.
- Pre-render a contact sheet of frames to audit for glitch frames before full renders.

## Checklist
- [ ] No mouth flicker (cadence ≥ 3 frames at 30 fps)
- [ ] Mouth closed during pauses and outside narration
- [ ] Silent preview works without audio files
