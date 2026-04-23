# video-brief

> **Give an LLM a text brief of a video so it can reason about cuts, chapters, or highlights without ever receiving a single frame.**

An Agentino skill that inspects a local video or audio file with `ffmpeg` / `ffprobe` and returns a compact, LLM-friendly summary: duration, resolution, codecs, silence gaps, scene-change timestamps, and a configurable set of thumbnails. No LLM call, no cloud API — everything runs through local subprocesses.

Inspired by [`browser-use/video-use`](https://github.com/browser-use/video-use): an LLM doesn't need to *watch* a video when it can *read* a well-structured representation of it. That representation costs ~12 KB of text plus a handful of PNGs, instead of 30 000 frames × 1 500 tokens.

## Install

Requires [Agentino](https://github.com/dagoSte/agentino) ≥ `1.2.0-rc.1` and a local `ffmpeg` (+ `ffprobe`) install.

```bash
# 1. Install ffmpeg
brew install ffmpeg              # macOS
# sudo apt install ffmpeg        # Debian / Ubuntu

# 2. Install the skill from this repo
agentino marketplace install agentino-os/agentino-skill-video-brief

# 3. Verify
agentino skill show video-brief
```

After install the skill lives under `skills/imported/video-brief.yaml`. `agentino skill update video-brief` picks up future releases from this repo.

## Use

```bash
agentino skill exec video-brief \
  -i path=/path/to/talk.mp4 \
  -i thumbnail_every_s=60
```

Example output (truncated):

```json
{
  "metadata": {
    "duration_s": 1634.21,
    "duration_readable": "27m 14.21s",
    "width": 1920, "height": 1080,
    "video_codec": "h264", "audio_codec": "aac",
    "fps": 29.97, "audio_channels": 2, "audio_sample_rate": 48000,
    "bit_rate": 5213440
  },
  "silence_gaps": [
    { "start_s": 12.3,  "end_s": 13.8,  "duration_s": 1.5 },
    { "start_s": 105.2, "end_s": 106.9, "duration_s": 1.7 }
  ],
  "scene_changes": [5.0, 134.5, 278.1, …],
  "thumbnails": [
    { "time_s": 30.0, "path": "/tmp/video-brief-abc/thumb-000-t000030s.jpg" },
    …
  ],
  "brief_markdown": "# Video brief — talk.mp4\n\n## Metadata\n…"
}
```

`brief_markdown` is pre-formatted and drops straight into a planner prompt:

```bash
BRIEF=$(agentino --json skill exec video-brief -i path=talk.mp4 \
  | jq -r '.data.output.brief_markdown')
agentino run "Given the following video brief, propose 3 YouTube chapter
titles with timestamps:\n\n$BRIEF"
```

## Use with agentino run

```bash
agentino run "give me a brief of /tmp/talk.mp4 with thumbnails every 30 seconds"
```

`agentino run` lets the LLM planner pick this skill automatically from the task description — the file path and optional cadence are parsed straight from the prose, no `-i` flags needed.

## Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `path` | string | **required** | Absolute path to any `ffprobe`-readable file (mp4/mov/mkv/mp3/wav/…). |
| `thumbnail_every_s` | integer | `30` | Sample one thumbnail every N seconds. |
| `max_thumbnails` | integer | `20` | Hard cap on thumbnails (prevents huge outputs on long videos). |
| `silence_threshold_db` | integer | `-30` | Silence detection threshold in dBFS. Use `-40` for home recordings with room tone. |
| `min_silence_s` | number | `0.5` | Only report silences at least this long. |
| `scene_threshold` | number | `0.3` | `ffmpeg` scene-change sensitivity (0 = every frame, 1 = never). |
| `max_scene_changes` | integer | `50` | Hard cap on scene-change entries. |
| `output_dir` | string | *tempdir* | Where to write thumbnails. Defaults to `~/tmp/video-brief-<hash>/`. |

## Outputs

Structured JSON with:

- **`metadata`** — probe results: duration, codecs, resolution, fps, audio channels + sample rate, bitrate, file size.
- **`silence_gaps`** — list of `{start_s, end_s, duration_s}`.
- **`scene_changes`** — list of timestamps where the filter graph flagged a scene cut.
- **`thumbnails`** — list of `{time_s, time_readable, path, size_bytes}`.
- **`brief_markdown`** — a pre-rendered Markdown summary ready to feed an LLM.

## Chain with `video-cut-silences`

This skill is the first stage of the [`video-auto-trim`](https://github.com/agentino-os/agentino-pipeline-video-auto-trim) pipeline: `video-brief` → [`video-cut-silences`](https://github.com/agentino-os/agentino-skill-video-cut-silences) produces a tightened output with crossfades at every cut.

```bash
agentino pipeline run video-auto-trim \
  -i path=talk.mp4 \
  -i output_path=talk.cut.mp4
```

## How it works

1. **Probe metadata** — `ffprobe -show_format -show_streams` → JSON parsed into a flat metadata block.
2. **Silence gaps** — `ffmpeg -af silencedetect` parses `silence_start` / `silence_end` messages from stderr into `{start_s, end_s, duration_s}` records.
3. **Scene changes** — `ffmpeg -vf select='gt(scene,…)',showinfo` extracts `pts_time` values from the filter-graph log.
4. **Thumbnails** — per-window `ffmpeg -ss t -frames:v 1` with `-vf scale='min(640,iw):-2'` to keep files tiny.
5. **Brief composer** — a few lines of Python produce the Markdown summary.

## Safety

- `sandbox_level: none` because `ffmpeg` typically lives outside the minimal Agentino sandbox `PATH` (`/usr/local/bin`, `/opt/homebrew/bin`, distro-specific). The skill resolves both binaries via `shutil.which` up-front and refuses to hard-code paths.
- `network_access: false` — no remote calls.
- `file_access: read-write` — reads the source video, writes thumbnails to a per-video tempdir.
- `agentino skill exec skill-security-check -i path=skill.yaml` → zero findings.

## License

MIT — see [`LICENSE`](./LICENSE).
