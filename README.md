# Video Transcript Clip Extraction Pipeline

This repository demonstrates, in a Google Colab notebook, the pipeline behind
a **video clipping agent**: start with a source video and a producer-tagged
transcript, resolve transcript segments to media timestamps, render the selected
segments as MP4 clips, and preserve enough state to plan an incremental rerun
when the transcript changes.

```text
FIRST RUN
video.mp4 + transcript.json
        |
        v
ingest -> locate make_clip segments -> transcribe audio -> align known text
        -> verify boundaries -> ffmpeg cut -> manifest.json + state.json

LATER RUN (prototype planner)
edited transcript + prior manifest/state
        |
        v
exact diff -> semantic comparison -> alignment decisions -> clip decisions
        |                                      |
        v                                      v
REUSE / PATCH / FULL_REALIGN / NEW     REUSE_CLIP / RECUT / CUT_NEW
VERIFY_SHIFT / REMOVE                  / RETIRE_CLIP
```

> [!IMPORTANT]
> This is a research notebook, not a packaged or autonomous agent. Stage 1 has
> rendered clips in the saved Colab run. Stage 2 currently demonstrates the
> agent's planning and invalidation logic only: edited and inserted durations
> are mocked, local realignment is not implemented, and the resulting clip
> actions are not executed. Review [Known limitations](#known-limitations)
> before using any output.

## Repository contents

| File | Purpose |
| --- | --- |
| `project_notebook_v1.ipynb` | Colab prototype containing the baseline clip pipeline and incremental-rerun planner. |
| `brewster_kahle_transcript.json` | Original 140-segment text transcript; all `make_clip` values are false. |
| `brewster_kahle_transcript_flagged.json` | Producer-tagged copy with 16 segments selected for clipping and titled. |
| `brewster_kahle_transcript_with_ground_truth.json` | Reference copy with source timestamps for future evaluation; it is not model input in the notebook. |
| `LICENSE` | MIT license. |

The one-hour Brewster Kahle interview video and the generated artifacts are not
stored in Git. The notebook reads and writes those files in Google Drive.

## What the notebook currently demonstrates

### Stage 1 — establish a baseline

The current notebook:

1. mounts Drive and recursively finds the original transcript, its flagged
   copy, and the Brewster Kahle interview MP4;
2. reads `make_clip` and `clip_title` metadata from the flagged transcript;
3. probes the 3,613.3-second source with `ffprobe` and extracts 16 kHz mono WAV
   audio with `ffmpeg`;
4. runs WhisperX's `small` ASR model to obtain an audio-backed word stream;
5. CTC-aligns that stream, uses `SequenceMatcher` to associate the known
   transcript with ASR words, then maps aligned words back to the 140 producer
   segments;
6. creates four `_check_<segment>.wav` snippets for human boundary review;
7. renders each of the 16 selected segments with frame-accurate seeking,
   H.264/AAC encoding, 0.15 seconds of leading padding, and 0.35 seconds of
   trailing padding; and
8. writes a clip `manifest.json` plus a reusable `state.json` containing short
   SHA-256 text fingerprints, timestamps, confidence, clip flags, and run
   history.

The saved run produced `clip_001.mp4` through `clip_016.mp4`. It is useful
evidence that the flow can render media, but not proof that every boundary is
correct. The same output reports 8,221 of 9,035 normalized transcript words
matched in the first global diff, 17 weak segment matches, a later 10-word
count mismatch during positional re-attribution, and 2 unresolved segments.
Those warnings make the listening checks mandatory.

### Stage 2 — plan an incremental rerun

Given the old transcript, Stage 1 state, and a proposed
`brewster_kahle_transcript_flagged_v2.json`, the notebook is intended to:

- verify that the old transcript still matches the hashes in `state.json`;
- classify exact text changes as unchanged, edited, inserted, or deleted with
  `SequenceMatcher`;
- cache `all-MiniLM-L6-v2` sentence embeddings by text hash and use cosine
  similarity to distinguish a minor reword from a substantial rewrite;
- plan transcript actions: `REUSE`, `PATCH`, `FULL_REALIGN`, `NEW`,
  `VERIFY_SHIFT`, and `REMOVE`;
- estimate and propagate a cumulative timing delta for the synthetic revision;
- append the plan and explanations to `decisions.log`; and
- separately plan `REUSE_CLIP`, `RECUT`, `CUT_NEW`, or `RETIRE_CLIP`, because a
  clip with unchanged text can still become stale after an upstream time shift.

The v2 fixture is **not included in this repository**, and the saved Stage 2
run is incomplete. The notebook records both a local-upload/variable-order
`NameError` and a later Drive lookup failure. It also calls
`difflib.SequenceMatcher` without importing the `difflib` module. These issues
must be corrected in the Colab session before the planner can run end to end;
see the exact workaround below.

## Input contract

The transcript is ordered JSON with a stable `video_id` and stable, unique
segment IDs:

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

Requirements for reliable reruns:

- segments must remain in spoken order;
- IDs must be unique and should remain stable across revisions;
- `text` should represent the spoken words as closely as possible;
- `make_clip: true` selects a segment for rendering; and
- selected segments should have a `clip_title`.

The included ground-truth timestamps are evaluation data. Do not feed them
into the alignment path if the goal is an honest test of timestamp recovery.

## Run the pipeline in Google Colab

### 1. Prepare Drive and Colab

Use a GPU runtime if available and create:

```text
MyDrive/Projects/LEC AI Project/
```

Place these inputs beneath that directory:

```text
brewster_kahle_transcript.json
brewster_kahle_transcript_flagged.json
Brewster Kahle - Interview - 26 Feb 2021.mp4
```

They may be inside a `brewster_kahle/` child directory because the ingest code
searches recursively. Avoid duplicate names: the notebook silently takes the
first match. For another project, change `DRIVE_ROOT`, all transcript search
patterns, the video glob, and `VIDEO_DIR` in the notebook.

Open **`project_notebook_v1.ipynb`** in Colab. The notebook installs WhisperX,
PyTorch/TorchAudio/TorchVision, NumPy, and later `sentence-transformers`, and it
downloads model weights. A standard Colab runtime already supplies `ffmpeg` and
`ffprobe`; internet access is needed for packages and models.

> [!CAUTION]
> The dependency cells are exploratory rather than pinned: they reinstall the
> Torch stack and then request `numpy<2.1`, while the saved installation output
> says the installed WhisperX requires NumPy 2.1 or newer. Run installation
> cells first, restart if Colab asks, and verify imports before spending time on
> the hour-long alignment. A reproducible lockfile is not yet provided.

### 2. Run Stage 1 in order

Run cells from `# Stage 1` through `## Phase 5: Exporting the clips` in order.
The operational sequence is:

1. **Ingest:** mount Drive; confirm the notebook reports 140 segments, the
   expected video, and a duration near 60.22 minutes.
2. **Locate:** confirm that the flagged transcript prints the 16 intended clip
   IDs and titles.
3. **Align:** extract audio, run ASR, associate known transcript text to the ASR
   timeline, and perform CTC refinement. Do not confuse the proportional rough
   boundaries with final timestamps.
4. **Validate:** inspect all warnings and listen to `_check_seg_005.wav`,
   `_check_seg_057.wav`, `_check_seg_108.wav`, and `_check_seg_071.wav`. Confirm
   each clip begins and ends on the expected speech, especially late in the
   video.
5. **Cut and export:** only after validation, run the `ffmpeg` cutting cell and
   the manifest/state export cell.

Before accepting the run, verify that selected segments have non-null bounds,
bounds are monotonic and inside the media duration, clip files play correctly,
and manifest times describe the intended source speech. The current cutting
cell does not guard against unresolved selected segments, so this review is a
required safety gate rather than an optional spot check.

### 3. Inspect and preserve the baseline

Stage 1 writes:

```text
MyDrive/Projects/LEC AI Project/brewster_kahle/
├── interview_audio.wav
├── _check_seg_005.wav
├── _check_seg_057.wav
├── _check_seg_071.wav
├── _check_seg_108.wav
├── clips/
│   ├── clip_001.mp4
│   └── ...
│       └── clip_016.mp4
├── manifest.json
└── state.json
```

`manifest.json` connects each rendered file to its segment, title, unpadded
aligned interval, confidence, and text. `state.json` is the incremental run's
baseline. Back up the clips, manifest, and state together before experimenting
with an edited transcript.

### 4. Exercise the Stage 2 planner

Create an edited copy named `brewster_kahle_transcript_flagged_v2.json`. Keep
unchanged IDs stable; use new IDs for genuine insertions. Put this file beneath
`DRIVE_ROOT` so the Stage 2 recursive lookup can find it.

In a session where Stage 1 has already initialized its variables and artifacts:

1. skip the local `files.upload()` cell and the immediately following cell;
2. before the old/new transcript lookup cell, run:

   ```python
   import difflib
   ```

3. run the lookup cell and confirm the state hash sanity check passes;
4. run the exact diff and embedding cells, then inspect every non-unchanged
   classification and semantic score;
5. run the decision, mocked delta propagation, and log cells; and
6. run the clip-reuse decision cell to inspect the proposed media operations.

Do **not** treat `resolved_v2`, `decisions.log`, or the clip decision summary as
production output. Any reason containing `MOCKED` uses an average
seconds-per-word estimate, not audio evidence. The planner does not call
WhisperX on a changed window, render/retire files, or write a validated v2
manifest and state.

## Artifact roles in an eventual clipping agent

| Artifact | Agent use |
| --- | --- |
| `manifest.json` | Inventory of rendered clips and provenance for deciding whether an existing file remains valid. |
| `state.json` | Trusted prior transcript fingerprints and resolved timing used to avoid blind full reruns. |
| `decisions.log` | Auditable explanation of the proposed transcript-level actions and timing shifts. |
| `clips/*.mp4` | Reusable outputs when both source selection and verified bounds remain unchanged. |

This separation is central to the agent design: **planning** decides what work
is necessary, **execution** performs alignment and rendering, and
**verification** decides whether new state may replace the trusted baseline.
Only planning is prototyped for incremental runs today.

## Known limitations

- **Notebook-only:** there is no CLI, package, config schema, automated test
  suite, or non-interactive runner.
- **Not cleanly reproducible:** dependency versions are unpinned and saved
  outputs include installation conflicts, a `NameError`, and a missing-v2-file
  error.
- **Stage 1 attribution warning:** the transcript/WhisperX word counts differ by
  10, so positional mapping can drift after the first tokenization difference;
  17 segments were weak in the preceding global match and 2 ended unresolved.
- **Approximate bootstrapping:** proportional rough timing assumes speaking time
  follows word count and does not model silence, introductions, music, or
  transcript omissions.
- **Human validation only:** the ground-truth fixture is not used to calculate
  median, p95, or worst-case boundary error.
- **Missing Stage 2 fixture:**
  `brewster_kahle_transcript_flagged_v2.json` is not committed.
- **Synthetic incremental timing:** patches, insertions, deletions, and
  downstream shifts are estimated rather than measured from audio.
- **In-memory embedding cache:** embeddings disappear when the runtime ends.
- **Decision/execution gap:** proposed recuts, new clips, retirements, and state
  updates are not applied atomically—or applied at all—in Stage 2.
- **Weak integrity model:** source media and render settings are not
  fingerprinted, and duplicate recursive matches are not rejected.

## Roadmap to a runnable agent

1. Make one clean Colab run reproducible with pinned, compatible dependencies
   and remove stale/error cells.
2. Replace positional word attribution with robust per-segment/context matching
   and explicit confidence thresholds.
3. Evaluate predicted boundaries against the ground-truth fixture and fail runs
   outside defined tolerances.
4. Implement bounded local alignment with left/right anchors, measured timing
   deltas, and full-alignment fallback.
5. Persist alignment and embedding caches using media, model, schema, text, and
   render-setting fingerprints.
6. Implement clip reuse/recut/retirement and atomically commit the new manifest,
   state, clips, and decision history only after verification succeeds.
7. Extract notebook functions into a tested package and add a dry-run planner,
   structured logs, schema validation, and recovery from interrupted writes.

### Intended command-line experience

The notebook's stages map naturally to a future interface such as:

```bash
# Build and verify the trusted baseline.
python -m video_agent init \
  --video input/interview.mp4 \
  --transcript input/brewster_kahle_transcript_flagged.json \
  --workspace runs/brewster

# Explain reuse, alignment, and clip work without changing files.
python -m video_agent plan \
  --transcript input/brewster_kahle_transcript_flagged_v2.json \
  --workspace runs/brewster

# Execute measured alignment/render actions and atomically update state.
python -m video_agent apply \
  --transcript input/brewster_kahle_transcript_flagged_v2.json \
  --workspace runs/brewster

# Validate hashes, boundaries, manifests, and rendered media.
python -m video_agent verify --workspace runs/brewster
```

These commands demonstrate how the completed clipping agent should be run;
they are **not implemented in this repository**.

## License

This project is available under the [MIT License](LICENSE).
