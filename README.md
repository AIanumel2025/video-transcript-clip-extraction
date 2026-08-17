# Incremental Video Clipping Agent

An agent that takes a video file and its transcript and produces correctly-timed MP4 clips for the segments an editor has flagged. Run it again with an updated transcript — a correction, a reworded line, a deleted segment, a newly inserted one — and it detects exactly what changed, decides whether to patch just the affected region or re-align the whole file, reuses cached embeddings/alignment/clips wherever they're still valid, and only regenerates what actually moved.

Built and validated against real content: a ~60-minute recorded interview and its full transcript, run twice — once as a cold start, once against a genuinely edited version of the transcript.

## What it does

1. **Ingest** a video + a segmented JSON transcript.
2. **Align** each segment to real audio timestamps using [whisperx](https://github.com/m-bain/whisperX) (Whisper transcription + wav2vec2 CTC forced alignment) — not a proportional guess.
3. **Cut** MP4 clips at the resolved boundaries with ffmpeg.
4. **Export** a manifest (clip → source segment → timing → confidence) and a state file (per-segment text hashes, for future runs).
5. On a **second run** with an updated transcript: diff old vs. new at the segment level, embed both sides to tell "reworded" from "actually different," decide REUSE / PATCH / VERIFY_SHIFT / NEW / REMOVE per segment with a reason for each, propagate timing deltas downstream of any edit or insert/delete, and commit — re-cutting only what moved, leaving everything else untouched on disk.

## Why this exists

Video and podcast teams re-edit transcripts constantly — a correction here, a new pull-quote there — and re-cutting every clip from scratch every time doesn't scale. This agent makes "edit the transcript, republish the clips" fast enough to do on every pass instead of once at launch.

## Repository layout

```
├── notebook/
│   └── the working notebook, run end to end, outputs included
│
├── sample_input/
│   ├── brewster_kahle_transcript_flagged.json      run 1 transcript (140 segments, 16 flagged)
│   └── brewster_kahle_transcript_flagged_v2.json   run 2 transcript (1 reworded, 1 deleted, 1 inserted)
│
├── sample_output/
│   ├── manifest_v1.json          run 1 manifest — 16 clips, all freshly cut
│   ├── manifest.json             run 2 manifest — clips tagged REUSED or RECUT, with reasons
│   ├── state.json                per-segment hashes + resolved timestamps + run_history (4 runs)
│   ├── current_transcript.json   snapshot of the transcript state.json currently reflects
│   └── decisions.log             full reasoning trail — every decision, every run, human-readable
│
└── docs/
    └── build-plan.md             design notes: caching strategy, decision-engine rules, open items
```

The source video/audio and the 16 rendered MP4 clips aren't included here — see [Running it yourself](#running-it-yourself). Source video/audio and its output clips will be demonstrated in walkthrough.

## Results

**Run 1 (cold start).** Real interview video, real 140-segment transcript. Word-count cross-check matched exactly; by-ear spot checks across all 16 flagged clips confirmed clean start/end boundaries. Output: 16 MP4 clips, `manifest_v1.json`, `state.json`.

**Run 2 (incremental).** A deliberately edited transcript — `seg_015` reworded, `seg_021` deleted, a new `seg_025a` inserted — run through the full pipeline:

| Stage | Result |
|---|---|
| Diff | `{unchanged: 138, edited: 1, deleted: 1, inserted: 1}` |
| Embedding check | reworded pair scored 0.880 similarity (same idea); an unrelated pair scored 0.158 |
| Decisions | `{REUSE: 14, PATCH: 1, VERIFY_SHIFT: 124, REMOVE: 1, NEW: 1}` |
| Clip reuse | `{REUSE_CLIP: 2, RECUT: 14}` |

The two segments before the edit point (`seg_008`, `seg_010`) reused their existing clips byte-for-byte — no ffmpeg call, no re-render. The 14 flagged clips downstream of the edit all got re-cut with corrected timestamps, each with a logged before/after delta (e.g. `seg_022`: `566.23s–655.81s → 551.26s–640.84s`). Re-running the diff against the committed state reports zero further changes — proof the incremental loop is idempotent, not just a one-off before/after comparison.

## Caching / memory

Three things persist across runs, all backed by files rather than in-memory state, so they survive closing and reopening the notebook entirely:

- **`state.json`** — per-segment text hash, resolved start/end, confidence, and a `run_history` log. This is the source of truth for what changed between runs.
- **`current_transcript.json`** — a snapshot of the transcript `state.json` was built from, so the next run diffs against what was actually committed last time, not against the original transcript forever.
- **Embedding cache** (`cache/embeddings/embeddings.json`) — semantic vectors keyed by text hash, so unchanged segments are never re-embedded.
- **Alignment cache** (`cache/alignments/asr_words_<audio_hash>.json`) — the full whisperx-aligned word list, keyed by a hash of the audio file itself. Since a transcript-only edit doesn't change the audio, this lets run 2 skip the multi-minute ASR + forced-alignment pass entirely; the second load off this cache is near-instant.

## Known simplification

The duration estimate for a reworded or inserted segment (in the "PATCH" / "NEW" decisions above) is currently derived from an average seconds-per-word rate, not a fresh windowed re-alignment call to whisperx. This is intentionally flagged as `MOCKED` in both the decision reasoning and `decisions.log` rather than hidden. A real implementation would re-run whisperx only on the audio window spanning the edit, not the whole file — that's the next thing to build (see `docs/build-plan.md`).

## Running it yourself

The notebook was built and run in Google Colab, with the source video/audio and a `LEC AI Project` Drive folder as the working directory. To reproduce:

1. Open `notebook/` in Colab, mount Google Drive, and point `VIDEO_DIR` at a folder containing your source video.
2. Run Stage 1 top to bottom: extracts audio, runs whisperx (transcribe + align), text-matches your transcript onto the aligned words, cuts clips, writes `manifest.json` + `state.json`.
3. Edit your transcript (or drop in `sample_input/brewster_kahle_transcript_flagged_v2.json` as a worked example) and run Stage 2: diff → embed → decide → commit. Only the segments the decision engine flags as `RECUT` or `NEW` actually get re-rendered.

Requires: `whisperx`, `torch`, `sentence-transformers`, `ffmpeg` on PATH. GPU is optional — whisperx runs on CPU with `compute_type="int8"`, just slower; the ffmpeg cutting step is CPU-bound regardless (`libx264`/`aac` re-encoding isn't GPU-accelerated by whisperx's GPU usage).

## Edge Cases Handled

1. Making sure timestamps came from the real audio, not a guess.
An earlier approach assumed people speak at a constant, even pace and just divided up time accordingly. This doesn't really apply in real speech. Conversational speech often carries pauses, talking over each other and even long pauses for laughs. We fixed this by running real speech-to-text on the actual audio first, so every word got its own real timestamp based on when it was actually spoken.
2. Fixing timestamps that got progressively more wrong the further into the video you went.
The first version of the alignment lined up transcript text to audio one line at a time, moving forward step by step. Whenever one line didn't match perfectly, the next line started searching from the wrong spot. This made small errors stack up on top of each other the further into the interview you got. The fix was to stop doing it line-by-line and instead compare the entire transcript against the entire audio transcript at once, so one bad match couldn't throw off everything after it.

## Next steps

My next priority would be to extract the working pipeline logic out of the notebook into version-controlled `.py` modules with a CLI entry point and tests.