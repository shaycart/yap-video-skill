# yap-video — a Claude Code skill for editing talking-head videos

Turns raw talking-head / "yap" footage into a tight short-form edit (Reels, TikTok, Shorts, or 16:9):
transcript-driven jump cuts, filler and stutter removal, face-aware punch-in zooms, word-by-word animated
captions, restrained sound design, loudness mastering, self-QA, and publish copy. Everything runs locally
with ffmpeg + local Whisper; no API key required.

It was synthesized from a read-only, security-reviewed pass over a dozen open-source video-editing agent
skills (credited in `SKILL.md`), keeping the concrete numbers that work: cut thresholds, zoom ladders,
caption timing, SFX placement, and loudness targets. Ideas only were taken from repos with risky code
(credential reuse, telemetry hooks); none of their code is included.

## Install

```bash
npx skills add shaycart/yap-video-skill
```

or copy `skills/yap-video` into `~/.claude/skills/`.

## Use

In Claude Code, point it at a folder of footage: "edit this yap", "cut the dead air and caption it",
"make this fire for Reels". It plans first, waits for your OK, then renders to `<folder>/edit/`.

## Requirements

- ffmpeg / ffprobe (libass optional; the skill has a Pillow fallback for captions)
- A local Whisper: faster-whisper, mlx-whisper (Apple Silicon), or whisper.cpp
- Python 3 with numpy and Pillow; OpenCV or MediaPipe for face tracking (optional)

## What's inside

- `skills/yap-video/SKILL.md`: the pipeline, hard rules, vibes/presets, and known gaps
- `skills/yap-video/references/params.md`: the ffmpeg recipes and numbers (cuts, zoom, captions, audio, grade, export)
