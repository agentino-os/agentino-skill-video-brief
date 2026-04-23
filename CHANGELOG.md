# Changelog

All notable changes to the `video-brief` Agentino skill are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this skill adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] — 2026-04-23

First public release for the Agentino marketplace.

### Added

- `agentino skill exec video-brief` produces a structured brief of any local video / audio file:
  - `metadata` — duration, codecs, resolution, fps, audio channels + sample rate, bitrate, file size.
  - `silence_gaps` — list of `{start_s, end_s, duration_s}` derived from `ffmpeg silencedetect`.
  - `scene_changes` — list of timestamps derived from `ffmpeg -vf select='gt(scene,…)',showinfo`.
  - `thumbnails` — sampled frames, scaled to ≤ 640 px wide, one every N seconds up to `max_thumbnails`.
  - `brief_markdown` — a ready-to-feed-an-LLM Markdown summary.
- Config knobs: `thumbnail_every_s`, `max_thumbnails`, `silence_threshold_db`, `min_silence_s`, `scene_threshold`, `max_scene_changes`, `output_dir`.
- `sandbox_level: none` (required for `ffmpeg` discovery across platforms), `network_access: false`, `file_access: read-write`.

### Tested

- Synthetic 30 s lavfi clip with scripted silence 10–15 s → `silence_gap` detected at `10.01 s → 15.00 s (4.99 s)` in 780 ms end-to-end.
- Zero findings from `agentino skill exec skill-security-check -i path=skill.yaml` (fail-on = high).
