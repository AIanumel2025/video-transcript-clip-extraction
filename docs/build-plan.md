# Incremental Video Clipping Agent — Build Plan

## Product framing
An agent that ingests a video file + its transcript (JSON) and produces correctly-timed MP4 clips. On re-runs with an updated transcript, it diffs old vs. new transcript, reprocesses only changed segments, and reuses cached embeddings / alignment / clips wherever still valid. It decides between patching a region and fully realigning, logs that decision with reasoning, and handles cascading invalidation when a segment shift moves everything downstream.

Positioning: sits in the middle of a podcast/video team's repurposing pipeline (raw video + transcript → this agent → YouTube Shorts / TikTok / IG Reels / X clips), making "edit transcript, republish clips" fast enough to do every time instead of once at launch.

## Directory structure
```
video-agent/
  agent/
    align.py       # AlignmentBackend interface + PocketSphinxAligner (working default)
    locate.py       # word timings -> per-segment start/end, with confidence
    cutter.py        # ffmpeg clip cutting
    manifest.py      # output/manifest.json writer
    logger.py         # output/decisions.log writer
    state.py           # state/state.json writer
    pipeline.py         # orchestrates the above
    config.py            # paths + tunables
  scripts/
    generate_fixtures.py     # builds the synthetic TTS test video + transcript.json
    fixture_ground_truth.json  # true segment timings, for validating alignment accuracy
  input/
    interview.mp4
    transcript.json
  state/
    state.json
  cache/
    embeddings/
    alignments/
  output/
    clips/
      clip_001.mp4
    manifest.json
    decisions.log
  cli.py
  requirements.txt
  README.md
```

## Core pipeline (per run)
1. Locate segments — find target quotes/topics in the transcript text.
2. Determine timestamps — align located text to exact start/end times.
3. Cut the video — slice source .mp4 at resolved boundaries (ffmpeg).
4. Generate MP4 clips — export finished files.
5. Export — write manifest.json (clip provenance) + state.json (per-segment text hashes, for future incremental re-runs).

## The incremental challenge
- Diff old transcript vs. new transcript at the segment level (hash + semantic embedding) before touching any media.
- Three caches, each keyed to a different unit of work:
  - Embedding cache — semantic vector per segment, keyed by text hash.
  - Alignment cache — resolved start/end timestamps (and, for the ASR-anchor approach, the raw audio-aligned word list), keyed by audio content hash / segment id + text hash.
  - Clip cache — rendered MP4, keyed by timestamp range + text hash. Reused only if both text and timing are still valid.
- Decision engine chooses PATCH (targeted regional realignment) vs FULL REALIGN (end-to-end re-alignment pass), and always logs a `Decision:` / `Reason:` entry to decisions.log.
  - Patch when: only a few segments changed text, no net duration shift, segments before/after are byte-identical, change is a correction not an insertion.
  - Full realign when: a large block was inserted/removed, timing shifts for everything downstream, diff confidence is low, cached alignment can't be trusted as a base.
- Downstream invalidation: a segment shift (e.g. new dialogue inserted) invalidates every clip after it even though their own text didn't change. The agent recomputes only their timing (using cached alignment as a base offset) rather than re-running STT on unchanged audio.

## Outputs
- `output/clips/*.mp4` — new clips rendered, unaffected ones untouched on disk.
- `manifest.json` — every clip tagged new / patched / reused, with timing metadata and source segment ids.
- `decisions.log` — human-readable reasoning trail, one entry per run.
- `state/state.json` — persists across runs: segment fingerprints (text hash + embedding ref), resolved timestamps, clip provenance, decision history.
- `current_transcript.json` — canonical snapshot of the transcript that `state.json` currently reflects, so the next run diffs against what was actually committed rather than always against the original v1 transcript.

## What's built and validated (real content, not synthetic)

**Run 1 (baseline pipeline)** — real ~60-minute Brewster Kahle interview video, real 140-segment transcript. Real forced alignment via whisperx (wav2vec2 CTC): Whisper transcribes the raw audio, whisperx aligns that transcription to get genuine per-word timestamps, then a global `difflib.SequenceMatcher` diff maps the human transcript's words onto those ASR words (replacing an earlier, flawed proportional-interpolation approach). Validated by word-count cross-check and by-ear spot checks across all 16 flagged clips. Produces 16 real MP4 clips, `manifest.json`, and `state.json`.

**Run 2 (incremental)** — a genuinely edited v2 transcript (one segment reworded, one deleted, one inserted) run all the way through: diff engine (identifies the 3 changes against 137 unchanged segments), embedding similarity (distinguishes "reworded, same idea" from "unrelated") via `sentence-transformers/all-MiniLM-L6-v2`, a decision engine that classifies every segment as REUSE / PATCH / VERIFY_SHIFT / NEW / REMOVE with a cumulative timing-delta propagated downstream of the edit point, and a commit phase that actually re-cuts the affected clips, retires deleted ones, and writes an updated `manifest.json` + `state.json`. Verified idempotent: re-diffing the committed state against the same v2 transcript reports zero changes.

**Known simplification, disclosed rather than hidden:** the duration change for an edited/inserted segment is estimated from an average seconds-per-word rate rather than a fresh windowed re-alignment call to whisperx. This is explicitly labeled "MOCKED" in the decision reasoning and in `decisions.log`. A real windowed re-alignment (re-running whisperx only on the audio spanning the edited region, not the whole file) is the natural next step; the current approach is a deliberate, time-boxed placeholder that still produces directionally correct, internally consistent timing shifts.

## Memory / caching additions
- **Persistent embedding cache** (`cache/embeddings/embeddings.json`) — semantic vectors keyed by text hash, written to disk so repeat runs reuse embeddings instead of recomputing them.
- **Persistent alignment cache** (`cache/alignments/asr_words_<audio_hash>.json`) — the full whisperx-aligned word list for a given audio file, keyed by a SHA-256 hash of the audio itself. Since the source audio didn't change between run 1 and run 2, this lets a transcript-only edit skip the several-minutes-long ASR + forced-alignment pass entirely on every subsequent run.
- Both caches are Drive-backed (not local Colab runtime storage), so they survive closing and reopening the notebook, not just repeat runs within one session.

## Open items / next steps
- Replace the mocked duration-change estimate with a real windowed re-alignment call.
- Extract the notebook's working logic into version-controlled `.py` modules per the original directory layout above, with a real CLI entry point and test suite — the notebook format was the right choice for fast iterative debugging, but a production version of this agent belongs in scripts, not cells.
- Add a content-hash check on the source video itself to the clip cache, so a re-exported/re-encoded source video forces a recut even if segment timestamps didn't move.
- Multi-platform export (aspect-ratio variants for Shorts/Reels/TikTok/X) — stretch goal, not started.
