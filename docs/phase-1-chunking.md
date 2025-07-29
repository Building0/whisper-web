# Phase 1 — 10‑Second Chunking with Sliding Window

## Goal
Replace current 30s/stride logic with **10 s windows + sliding overlap** (default 0.5–1.0 s). Works for file uploads and mic streams.

## Success Criteria
- Audio always split into ~10 s chunks (configurable).
- Overlap added to prevent word cuts; merge logic dedupes correctly.
- TXT/SRT still export correctly with global timestamps.
- No noticeable UI freezes; workers keep UI responsive.

## Tasks

### A. Audio Decode & Prep
- [ ] Decode to 16 kHz mono PCM (`audio/decoder.ts`).
- [ ] (Optional) normalize gain / trim leading silence.

### B. Chunker Utility
- [ ] Implement `chunkAudio(pcm, chunkSec=10, overlapSec=0.5)`.
- [ ] Ensure first/last chunk handle boundaries correctly.
- [ ] Emit `ChunkMeta` with absolute start/end + overlap info.

### C. Worker Integration
- [ ] Update job dispatch to send 10 s PCM slices to workers.
- [ ] Load transformers.js model once per worker (warm-up).

### D. Merge Layer
- [ ] Implement string or token overlap trimming.
- [ ] Shift local timestamps to global using `ChunkMeta.startSec`.
- [ ] Ensure exporter uses merged segments.

### E. UI Adjustments
- [ ] Add Advanced panel sliders: `chunkSec`, `overlapSec`.
- [ ] Show per-chunk progress (X/Y) and global progress.

### F. Testing
- [ ] Unit: chunk slicing (edge cases, last chunk < 10 s).
- [ ] Unit: merge dedupe logic (no duplicates, no gaps).
- [ ] Integration: 30+ min file smoke test (memory stability).

## Deliverables
- `audio/chunker.ts` + tests.
- Updated worker dispatch/merge code.
- Advanced settings UI.
- Demo video/GIF of chunking working.
