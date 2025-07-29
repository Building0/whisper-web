# Phase 3 — User Feedback: Loading/Processing Progress & Verbose Logs

## Goal
Make “what’s happening” visible: show model/audio loading stages and live, verbose logs during transcription.

## Success Criteria
- Users see a staged progress UI while audio is decoded, model loads, and chunks process.
- A collapsible “Logs” panel streams status messages (e.g., “Worker #2 processing chunk 7/24”).
- Users can start transcription as soon as first chunk is ready (if feasible), not after everything loads.

## Tasks

### A. Instrumentation Points
- [ ] Audio decode start/end events.
- [ ] Model fetch/download progress (bytes %).
- [ ] Model init (warm-up) event.
- [ ] Chunk queue length, chunk start/end events.
- [ ] Worker status (idle/busy/error).

### B. Progress UI
- [ ] “Loading audio…” stepper/progress bar.
- [ ] “Downloading model X%” bar with speed/time estimate.
- [ ] “Transcribing chunks: 3/24 done” indicator.
- [ ] Allow user to click “Transcribe” immediately, but disable if mandatory steps unfinished.

### C. Verbose Log Panel
- [ ] Collapsible drawer or console-like area.
- [ ] Append log lines with timestamps: `[12:03:15] Worker-1 finished chunk 5`.
- [ ] Clear / copy logs button.

### D. UX Polish
- [ ] Spinner → determinate bars when possible.
- [ ] Hover tooltips: explain what each phase does (“Decoding audio to PCM (16kHz mono)”).
- [ ] “Advanced → Debug mode” toggle to auto-open logs & overlays.

### E. Testing
- [ ] Simulate slow network/model load to verify UI continues updating.
- [ ] Check mobile layout for logs panel.
- [ ] Ensure logs don’t leak PII (audio never leaves browser anyway, but keep text safe).

## Deliverables
- `hooks/useProgress.ts` or similar global state for steps.
- `components/ProgressStepper.tsx`, `LogsPanel.tsx`.
- Instrumented code emitting progress/log events.
