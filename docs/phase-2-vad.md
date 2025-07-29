# Phase 2 — Silero VAD (ONNX / onnxruntime-web)

## Goal
Use Silero VAD to skip silence and still cap segments at 10 s windows.

## Success Criteria
- VAD identifies speech regions with tunable thresholds.
- Chunker respects speech boundaries; still splits long speech into ≤10 s blocks.
- User can toggle VAD on/off and tune params in Advanced panel.
- Visual debug overlay (optional) shows detected speech vs silence.

## Tasks

### A. Model & Runtime
- [ ] Host Silero VAD ONNX model (v5.x) on backend/CDN.
- [ ] Load via `onnxruntime-web` (WASM/SIMD build).
- [ ] Lazy-load model when VAD enabled.

### B. Frame-Level Inference
- [ ] Slice PCM into 20–30 ms frames.
- [ ] Batch frames for inference to reduce overhead.
- [ ] Smooth probabilities (moving average / median filter).

### C. Segment Builder
- [ ] Implement hysteresis thresholding:
  - `threshold` (speech prob)
  - `minSpeechMs`, `minSilenceMs`
  - `padMs` (pre/post padding)
  - `maxSpeechSec` (split very long speech)
- [ ] Output `[startSec, endSec]` segments.

### D. Integrate with Chunker
- [ ] For each VAD segment, subdivide into 10 s chunks + overlap.
- [ ] Fallback to fixed 10 s windows if VAD disabled or model fails.

### E. UI / Debug
- [ ] Advanced panel: sliders/inputs for VAD params.
- [ ] Debug overlay: waveform with VAD regions (toggleable).

### F. Testing
- [ ] Manual QA: ensure no speech dropped; adjust defaults.
- [ ] Unit tests: segment builder logic on synthetic signals.
- [ ] Performance check: ensure VAD step doesn’t dominate runtime.

## Deliverables
- `audio/vad.ts` (Silero inference + segment builder).
- Integration with chunker + tests.
- UI elements (controls + optional overlay).

