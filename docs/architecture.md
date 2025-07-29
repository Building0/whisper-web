# Project Architecture: whisper-web 5s Chunking + Silero VAD (transformers.js + ONNX)

> **Repo base**: [https://github.com/PierreMesure/whisper-web](https://github.com/PierreMesure/whisper-web) (frontend).
> **Primary goal (P0)**: deterministic 5‑second chunking + merge pipeline.
> **Secondary (P1)**: Silero VAD via onnxruntime-web to gate chunks by speech.

---

## 1. Context & Constraints

* Runs in browser; models hosted from your backend/CDN.
* Whisper inference via **transformers.js** (WebGPU/CPU fallback).
* VAD via **ONNX Runtime Web (WASM/SIMD)**.
* Must support **file upload + microphone streaming**.
* Exports: **.txt**, **.srt**.
* Advanced settings hidden by default.

---

## 2. High-Level Architecture

```
┌────────────┐   decode   ┌──────────────┐   (opt) VAD   ┌─────────┐  enqueue  ┌──────────────┐  merge   ┌──────────┐
│ Audio Srcs │ ─────────▶ │ PCM Buffer   │ ─────────────▶ │ Segments│ ─────────▶ │ Chunk Worker │ ───────▶ │ Merger   │
│ (file/mic) │            │ (16k mono)   │                │ (start,end)       │ Pool (N)      │          │ + Export │
└────────────┘            └──────────────┘                └─────────┘          └──────────────┘          └──────────┘
        ▲                                                                                                         │
        └────────────────────────────── UI (progress, waveform, controls, downloads) ─────────────────────────────┘
```

### Layers

1. **Audio Pipeline**: decode → (VAD) → chunk.
2. **Inference Layer**: Web Workers run transformers.js model.
3. **Merge & Post**: stitch text, timestamps, export.
4. **UI Layer**: React components, waveform visualization, advanced panel.

---

## 3. Module Breakdown

### 3.1 `audio/decoder.ts`

* Web Audio API decode or ffmpeg.wasm fallback.
* Resample to 16 kHz mono, return `Float32Array`.

### 3.2 `audio/vad.ts`

* Loads Silero ONNX model via `onnxruntime-web`.
* Sliding window (e.g., 20 ms hop) → probability vector.
* Hysteresis thresholding → speech segments list.
* Config: `threshold`, `minSpeechMs`, `minSilenceMs`, `padMs`, `maxSpeechSec`.

### 3.3 `audio/chunker.ts`

* Accepts: raw PCM or VAD segments.
* Emits fixed 5 s windows (`chunkSec`) with `overlapSec` (default 0.25 s).
* If VAD provided: split inside each speech segment, enforce max window.

### 3.4 `workers/inference-worker.ts`

* Init transformers.js model once (warm-up).
* Receive chunk buffers → return transcript + token timestamps.
* Optional: support batching later.

### 3.5 `merge/merger.ts`

* Align overlaps: trim duplicate leading/trailing words.
* Compute global timestamps: `chunkStart + localTs`.
* Produces final transcript array: `{start, end, text}`.

### 3.6 `export/*.ts`

* `exportTxt(transcript[])`
* `exportSrt(transcript[], fps=25)` (or real times).

### 3.7 `ui/*`

* React components:

  * **Waveform** (Wavesurfer.js or custom Canvas).
  * **Segments Overlay** (VAD + chunks).
  * **Progress Bars** per chunk + overall.
  * **Advanced Panel** (accordion): VAD + chunk params.
  * **Export Buttons**.

---

## 4. Data Contracts (Typescript Interfaces)

```ts
// Raw audio
interface PcmBuffer { data: Float32Array; sampleRate: number; }

// VAD output
interface SpeechSegment { startSec: number; endSec: number; }

// Chunk descriptor
interface ChunkMeta {
  id: number;
  startSec: number;
  endSec: number;
  overlapLeftSec: number;
  overlapRightSec: number;
}

// Worker request/response
interface InferenceJob { id: number; pcm: Float32Array; startSec: number; }
interface InferenceResult {
  id: number;
  text: string;
  words?: { start: number; end: number; word: string }[]; // optional
}

// Final merged unit (for SRT)
interface TranscriptSegment { start: number; end: number; text: string; }
```

---

## 5. Sequence (File Upload Path)

1. User selects audio → `decoder.ts` returns PCM.
2. (If VAD on) `vad.ts` runs, returns speech segments.
3. `chunker.ts` splits into 5 s windows (respecting segments or raw PCM).
4. Jobs dispatched to worker pool, each returns partial transcript.
5. `merger.ts` stitches, dedups overlaps, assigns global timestamps.
6. UI updates live; user exports TXT/SRT.

## 6. Sequence (Mic Streaming Path)

1. Continuous `MediaStream` frames appended to rolling PCM buffer.
2. Every N seconds, run VAD on new tail or use rolling threshold.
3. Emit chunks when:

   * Buffer ≥ 5 s, or
   * VAD signals end of speech chunk.
4. Dispatch to worker(s) and merge incrementally.
5. Update live transcript region; allow stop/export.

---

## 7. Build & Deployment

* **Frontend**: Vite/React build; code-split workers & models.
* **Models**: Host gguf/onnx binaries on backend or CDN; Service Worker caches.
* **WASM**: onnxruntime-web wasm & simd builds served with correct MIME.
* **CI**: run unit tests (Vitest/Jest), lint; optional puppeteer perf tests.

---

## 8. Configuration & Defaults

```yaml
chunk:
  seconds: 5
  overlap_seconds: 0.25
vad:
  enabled: true
  threshold: 0.5
  min_speech_ms: 200
  min_silence_ms: 100
  pad_ms: 100
  max_speech_sec: 8
workers:
  max_concurrency: auto  # default to navigator.hardwareConcurrency - 1
```

---

## 9. Error Handling Strategy

* Decode errors → show toast, suggest converting to WAV.
* Model load fail → retry with fallback backend.
* Worker crash → respawn, requeue jobs.
* VAD timeout/failure → disable VAD, fall back to fixed chunking.

---

## 10. Testing Strategy

* **Unit**: chunk boundaries, overlap merge, SRT timing.
* **Integration**: end-to-end on sample files (short, long, noisy).
* **Cross-browser**: Chrome/Edge/Firefox/Safari desktop + mobile.
* **Perf Benchmarks**: measure wall-clock vs baseline.

---

## 11. Observability & Debugging

* Debug overlay: show VAD probability curve & chunk lines on waveform.
* Timing logs (decode, VAD, chunking, inference) in dev console.
* Optional analytics (disabled by default) for perf telemetry.

---

## 12. Roadmap Links to Phases

* **P0** (chunking): Sections 3.1, 3.3, 3.4, 3.5, 3.6, 5.
* **P1** (VAD): Sections 3.2, 3.3 (VAD branch), 6.
* **P2** (Polish): Sections 3.7, 10, 11.

---

## 13. Open Questions (resolved in chat)

* Backend model hosting: **Yes**, model loads from backend/CDN.
* transformers.js first: **Yes**; whisper.cpp WASM optional later.
* Overlap: start at 250 ms, tweak via UI.
* Inputs: file + mic.
* Exports: txt, srt.
* Evaluation: eyeballing for now.
* Advanced settings: hidden by default.
* No extra features for now.

---

## 14. Future Hooks (Backlog)

* Speaker diarization (pyannote or jsdsp alternatives).
* Forced alignment (WhisperX) for word timestamps.
* Speculative decoding / hybrid tiny+base models.
* Server-side batch transcription for huge files.

---

## 15. Glossary

* **VAD**: Voice Activity Detection.
* **PCM**: Pulse-code modulated raw audio.
* **ORT-Web**: onnxruntime-web.
* **WER**: Word Error Rate.
* **PWA**: Progressive Web App.
