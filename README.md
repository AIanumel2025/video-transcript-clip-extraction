# Incremental Video Transcript Clip Extraction

This repository contains a **Google Colab notebook prototype** for turning a
video and a producer-tagged transcript into timed MP4 clips, then deciding what
can be reused after the transcript changes.

```text
video + tagged transcript
        |
        v
rough timing -> WhisperX alignment -> ffmpeg clips -> manifest + saved state
                                                        |
edited transcript -> diff + semantic comparison --------+
        |
        v
REUSE / PATCH / FULL_REALIGN / NEW / VERIFY_SHIFT / REMOVE
        |
        v
REUSE_CLIP / RECUT / CUT_NEW / RETIRE_CLIP
```

> [!IMPORTANT]
> This is a research notebook, not yet a packaged command-line agent. The
> notebook demonstrates the complete baseline flow and prototypes the
> incremental decision logic, but the second-run timestamps are currently
> mocked rather than produced by a real windowed alignment. Read
> [Current limitations](#current-limitations) before treating its output as
> production-ready.

## Repository contents

| File | Purpose |
| --- | --- |
| `project notebook_v1.ipynb` | Colab prototype containing Stage 1 (alignment and clipping) and Stage 2 (incremental decisions). |
| `brewster_kahle_transcript.json` | Original 140-segment Brewster Kahle transcript; no segments are selected for clipping. |
| `brewster_kahle_transcript_flagged.json` | The same transcript with 16 `make_clip` selections and clip titles. |
| `brewster_kahle_transcript_with_ground_truth.json` | Transcript copy with source timestamps for evaluation. These timestamps are test/reference data, not required input to the alignment flow. |
| `LICENSE` | MIT license. |

The source video is intentionally not stored in this repository. The notebook
expects it, transcripts, models, and generated artifacts to live in Google
Drive.

## What has been built

### Stage 1: baseline pipeline

The notebook now demonstrates:

1. mounting Google Drive and locating a source MP4 and JSON transcript;
2. selecting segments marked with `"make_clip": true`;
3. estimating coarse boundaries from transcript word counts;
4. aligning the known transcript against extracted 16 kHz mono audio with
   WhisperX's wav2vec2/CTC aligner;
5. mapping word-level timing back to the original transcript segments;
6. generating padded H.264/AAC MP4 clips with `ffmpeg`; and
7. writing `manifest.json` and `state.json`, including segment text hashes,
   timing, confidence, provenance, and run history.

The saved notebook output records a successful alignment of 140 Brewster Kahle
segments (9,043 words, none unresolved) and includes audio spot checks across
the hour-long source. Other persisted outputs in later cells come from an
earlier 149-segment AI-ethics experiment; see the limitations below.

### Stage 2: incremental-processing prototype

The second stage demonstrates the intended agent behavior on an old and edited
transcript:

- exact SHA-256 text fingerprints and `difflib.SequenceMatcher` classify
  unchanged, edited, inserted, and deleted segments;
- `all-MiniLM-L6-v2` sentence embeddings distinguish a minor reword from an
  unrelated replacement (the recorded example scores `0.948` versus a `0.85`
  threshold);
- a per-segment decision engine emits `REUSE`, `PATCH`, `FULL_REALIGN`, `NEW`,
  `VERIFY_SHIFT`, or `REMOVE`, with a plain-language reason;
- cumulative downstream timing shifts are propagated across multiple changes;
- decisions are appended to `decisions.log`; and
- clip validity is evaluated separately, because unchanged text may still need
  a recut when an upstream edit moves its timestamp.

This proves the control flow and invalidation rules. It does **not** yet perform
the real local re-alignment or recut work selected by those decisions.

## Transcript format

Input JSON has a top-level `video_id` and ordered `segments`:

```json
{
  "video_id": "brewster_kahle_interview",
  "segments": [
    {
      "id": "seg_008",
      "speaker": "Ian Milligan",
      "text": "So Brewster, I thought we might start...",
      "make_clip": true,
      "clip_title": "The Question That Started It All"
    }
  ]
}
```

Segment IDs should be stable and unique between revisions. Segment order must
match spoken order. Set `make_clip` to `true` only for segments that should
produce clips; `clip_title` is expected for selected segments.

## How to run the current prototype

### Prerequisites

- a Google account with Drive access;
- a Colab runtime (a GPU is recommended, although the notebook selects CPU when
  CUDA is unavailable);
- the source interview MP4;
- `ffmpeg` and `ffprobe` (already available in standard Colab runtimes); and
- internet access for the WhisperX and sentence-transformer packages/models.

### 1. Prepare Google Drive

Create this directory:

```text
MyDrive/Projects/LEC AI Project/
```

Upload the following files somewhere below it:

```text
brewster_kahle_transcript.json
brewster_kahle_transcript_flagged.json
Brewster Kahle - Interview - 26 Feb 2021.mp4
```

The notebook searches recursively, so the files may be inside a
`brewster_kahle/` subdirectory. Avoid duplicate filenames: the current code
uses the first recursive match.

If using another video, update `DRIVE_ROOT`, transcript filename patterns, and
the `*Brewster Kahle*Interview*.mp4` pattern in the ingest cells. Do not use the
included ground-truth timestamps as model input if you want an honest alignment
evaluation.

### 2. Open the notebook in Colab

Upload or open `project notebook_v1.ipynb` in Google Colab. Start with a fresh
runtime, mount Drive when prompted, and run the Stage 1 cells in order.

The dependency cells install WhisperX and PyTorch. Package versions in Colab
change over time; restart the runtime after installation if Colab requests it,
then rerun from the first import cell. The saved notebook contains one recorded
`GenerationMixin` import error from an incompatible package combination, even
though later saved outputs show alignment results. A reproducible dependency
lock is still outstanding.

### 3. Verify alignment before cutting clips

Do not rely only on the `aligned` label. Listen to the generated `_check_*.wav`
snippets and confirm that the expected sentence starts and ends at the spoken
words. Also confirm:

- transcript and aligned word counts match;
- all segments have non-null start/end times;
- start/end values remain within the media duration; and
- the late-video spot checks have not drifted.

Only then run the cutting and export cells.

### 4. Inspect Stage 1 outputs

The notebook writes beneath the configured Drive run directory:

```text
brewster_kahle/
├── interview_audio.wav
├── clips/
│   └── clip_001.mp4
├── manifest.json
└── state.json
```

`manifest.json` describes rendered files and their source segments.
`state.json` is the baseline for the next run and stores hashes and resolved
timings. Keep both alongside the clips and do not overwrite them until the new
run has been validated.

### 5. Exercise Stage 2 carefully

Stage 2 currently references experiment files named
`transcript_ai_ethics_flagged.json` and
`transcript_ai_ethics_flagged_v2.json`; those files are **not in this
repository**. To exercise it, provide a matching old/new pair in Drive or change
the two filename patterns to your own revisions. The old transcript must be the
exact file used to create the existing `state.json`, or its hash check will
warn.

Run Stage 2 only after Stage 1 has created `state.json` and `manifest.json`.
Review the printed diff and semantic scores, then review `decisions.log` and the
clip decision summary. Treat all timestamps labeled `MOCKED` as simulations;
do not publish clips from them.

## Current limitations

- **Notebook-only:** there is no `cli.py`, installable package, configuration
  file, automated pipeline, or test suite yet.
- **Mixed experiment state:** saved cell outputs include both the 140-segment
  Brewster Kahle run and a prior 149-segment AI-ethics run. Variables and Drive
  paths must be normalized before `Run all` is safe.
- **Stale/error outputs:** the notebook preserves an import error and a
  `NameError` in its execution history. A clean, top-to-bottom reproducibility
  run has not been captured.
- **Approximate initial windows:** rough boundaries assume speaking time is
  proportional to word count, which can fail with silence, introductions,
  music, or transcript omissions.
- **Fragile word attribution:** timing is assigned positionally and requires
  the transcript's whitespace word count to exactly match WhisperX output.
- **Synthetic second run:** edited/inserted durations use average seconds per
  word. Real audio for the revision is not locally re-aligned.
- **In-memory embedding cache:** embeddings are not persisted between runtime
  sessions.
- **Decision/execution gap:** Stage 2 selects recuts and retirements but does not
  execute them or write a new manifest/state atomically.
- **No acceptance metrics:** the ground-truth fixture is not yet used to report
  timing error, precision/recall, or clip-boundary quality.

## What to build next

With more development time, complete the agent in this order:

1. **Make the notebook reproducible.** Separate Brewster and AI-ethics
   experiments, remove reliance on stale in-memory variables, pin compatible
   Python/PyTorch/WhisperX versions, and capture one clean top-to-bottom run.
2. **Evaluate alignment.** Compare predicted boundaries with
   `brewster_kahle_transcript_with_ground_truth.json`; report median, p95, and
   worst-case start/end error and fail when tolerances are exceeded.
3. **Implement real patch alignment.** Build a bounded audio window around an
   edited or inserted segment, align its new text, calculate the measured
   duration delta, and verify anchors on both sides before shifting downstream
   timestamps. Fall back to full alignment when confidence is low.
4. **Persist all caches.** Store embeddings by text hash, alignments by media
   fingerprint + segment/context hash, and clips by source fingerprint + time
   range + render settings. Include model and schema versions in every key.
5. **Execute clip decisions.** Reuse valid files, render `CUT_NEW`/`RECUT`, move
   retired clips safely, and write new manifest/state/log files via an atomic
   transaction so a failed run cannot corrupt the last good state.
6. **Extract a real agent and CLI.** Move notebook functions into modules and
   expose commands such as `init`, `plan`, `apply`, and `verify`. Support a dry
   run that prints the plan without changing artifacts.
7. **Add safety and observability.** Validate schemas and stable IDs, detect
   duplicate Drive matches, fingerprint the source media, log model versions
   and confidence, and retain a machine-readable decision history.
8. **Add automated tests.** Cover unchanged transcripts, typo-only edits,
   insertions, deletions, reorderings, changed clip flags, timestamp cascading,
   interrupted writes, and ffmpeg output duration.
9. **Add production export features.** Once correctness is measured, add
   captions, vertical framing, platform presets, and optional human approval
   before publishing.

### Suggested target interface

The future packaged agent could be operated as follows:

```bash
# Establish a trusted baseline.
python -m video_agent init \
  --video input/interview.mp4 \
  --transcript input/transcript.json \
  --workspace runs/brewster

# Explain work without modifying files.
python -m video_agent plan \
  --transcript input/transcript_v2.json \
  --workspace runs/brewster

# Align changed windows, reuse or recut clips, and atomically commit new state.
python -m video_agent apply \
  --transcript input/transcript_v2.json \
  --workspace runs/brewster

# Check manifest, boundaries, hashes, and media outputs.
python -m video_agent verify --workspace runs/brewster
```

These commands describe the desired end state; they are **not implemented** in
the current repository.

## License

This project is available under the [MIT License](LICENSE).
