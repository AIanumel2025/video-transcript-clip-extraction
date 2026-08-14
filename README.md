# Incremental Video Clipping Agent

Turn a source video and its JSON transcript into correctly timed MP4 clips—then
make subsequent transcript edits cheap.

This project is an incremental processing agent for podcast and video teams. It
sits between raw media and the publishing workflow:

```text
video + transcript → clipping agent → YouTube Shorts / TikTok / Instagram Reels / X
```

Rather than rebuilding every clip after each transcript correction, the agent
compares the new transcript with the previous version, identifies the affected
segments, and reuses work that is still valid. The goal is to make “edit the
transcript, republish the clips” fast enough to be part of the normal workflow,
not a one-time launch task.

> [!NOTE]
> This repository is under active development. The baseline video-to-clips
> pipeline is complete; incremental transcript diffing is the next milestone.

## How it works

Each run follows four core steps:

1. **Locate segments** — find requested quotes or topics in the transcript.
2. **Determine timestamps** — align the selected text to precise start and end
   times in the source media.
3. **Cut the video** — use `ffmpeg` to slice the source MP4 at those boundaries.
4. **Generate clips** — export finished MP4 files and record their provenance.

On later runs, the agent first diffs the old and new transcripts at the segment
level. It then chooses either a targeted patch or a full realignment and records
both the decision and its reasoning.

## Incremental processing

### Segment diffing

Segments are fingerprinted before media is processed:

- A **text hash** identifies byte-level changes.
- A **semantic embedding** helps distinguish a correction from an insertion,
  deletion, or larger rewrite.

This lets the pipeline detect which segments changed and whether unchanged
segments can safely retain their prior results.

### Cache layers

The design uses three caches, each keyed to the unit of work it protects:

| Cache | Key | Reused when |
| --- | --- | --- |
| Embedding | Text hash | The segment text is unchanged. |
| Alignment | Segment ID + text hash | The segment text and alignment context remain valid. |
| Clip | Timestamp range + text hash | Both the text and resolved timing remain valid. |

Separating these caches allows the agent to recompute timestamps without
re-embedding text or rerendering an unaffected clip.

### Patch or full realign

The decision engine selects **PATCH** when:

- only a small number of segments changed;
- the edit introduces no net duration shift;
- the surrounding segments are byte-identical; and
- the change appears to be a correction rather than an insertion.

It selects **FULL REALIGN** when:

- a large block was inserted or removed;
- the edit shifts timing for downstream segments;
- diff confidence is low; or
- the cached alignment is not a trustworthy base.

Every run appends a human-readable entry to `output/decisions.log` containing a
`Decision:` and `Reason:` so operators can audit why work was reused or rebuilt.

### Cascading invalidation

An edit can invalidate more than the segment containing it. For example, newly
inserted dialogue moves every later timestamp even if the later text is
unchanged. In that case, downstream clips are invalidated and their timings are
recomputed from the cached alignment plus an offset. The unchanged downstream
audio does not need another speech-to-text pass.

## Inputs and outputs

### Inputs

- `input/interview.mp4` — source video.
- `input/transcript.json` — structured transcript containing the segments to
  align and clip.

### Outputs

- `output/clips/*.mp4` — newly rendered clips; unaffected files remain untouched.
- `output/manifest.json` — timing, source segment IDs, and a `new`, `patched`, or
  `reused` status for every clip.
- `output/decisions.log` — one patch-versus-realign decision and explanation per
  run.
- `state/state.json` — persistent segment fingerprints, embedding references,
  resolved timestamps, clip provenance, and decision history.

## Planned project structure

```text
video-agent/
├── agent/
│   ├── align.py                 # Alignment interface + PocketSphinx default
│   ├── locate.py                # Word timings → segment bounds + confidence
│   ├── cutter.py                # ffmpeg clip cutting
│   ├── manifest.py              # output/manifest.json writer
│   ├── logger.py                # output/decisions.log writer
│   ├── state.py                 # state/state.json persistence
│   ├── pipeline.py              # Pipeline orchestration
│   └── config.py                # Paths and tunable settings
├── scripts/
│   ├── generate_fixtures.py     # Synthetic TTS video/transcript generator
│   └── fixture_ground_truth.json
├── input/
│   ├── interview.mp4
│   └── transcript.json
├── state/
│   └── state.json
├── cache/
│   ├── embeddings/
│   └── alignments/
├── output/
│   ├── clips/
│   │   └── clip_001.mp4
│   ├── manifest.json
│   └── decisions.log
├── cli.py
├── requirements.txt
└── README.md
```

PocketSphinx is the planned working default behind an `AlignmentBackend`
interface, allowing other aligners to be added without changing the rest of the
pipeline.

## Roadmap

| Phase | Scope | Status |
| --- | --- | --- |
| 1. Baseline pipeline | Full-file alignment and naive clipping, end to end with a real alignment tool | **Done — 2026-08-14** |
| 2. Diff engine | Segment-level transcript diffing and hash/embedding fingerprints | **Next** |
| 3. Caching layers | Embedding, alignment, and rendered-clip caches | Planned |
| 4. Decision engine | Patch-versus-realign policy and decision logging | Planned |
| 5. Manifest and observability | Manifest generation, state history, and downstream invalidation | Planned |
| 6. Multi-platform export | Aspect-ratio variants for Shorts, Reels, TikTok, and X | Stretch |

## Design goals

- **Correct timing:** text selections resolve to reliable media boundaries.
- **Minimal recomputation:** unchanged embeddings, alignments, and clips are
  reused whenever their cache keys and context remain valid.
- **Safe invalidation:** upstream timing changes cascade to every dependent
  downstream artifact.
- **Explainable decisions:** each incremental run records what strategy was
  chosen and why.
- **Replaceable components:** alignment and other processing backends remain
  behind narrow interfaces.

## License

This project is available under the [MIT License](LICENSE).
