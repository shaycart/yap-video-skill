# Reference pipeline (from real edits)

Four small scripts in `<footage>/edit/`, each re-runnable on its own so a note only re-runs what it touches:

| script | in | out |
|---|---|---|
| `transcribe_clips.py` | `*.MP4` | `analysis/<clip>.json` (word timestamps, cached) |
| `plan.py` | transcripts + `RANGES` | `build/edl.json` (pieces), `build/words.json` (words on the output timeline) |
| `render_pieces.py` | EDL | `build/assembled.mov` (graded picture + cut audio, L-cuts) |
| `captions.py` | words + EDL | `build/captions-alpha.mov` (full-frame RGBA type/sticker/prop overlay), `captions.srt`, `captions.json` (cue times for SFX) |
| `master.py` | all of the above | `deliver/` (4K HEVC, 1080 H.264, no-captions, srt) |

## transcribe_clips.py
faster-whisper (`small.en` minimum, `medium.en` better), `word_timestamps=True`, `vad_filter=False`, `condition_on_previous_text=False`, an `initial_prompt` with fillers and brand words so they are transcribed verbatim and spelled right. Decode audio with ffmpeg to 16 kHz mono s16le, one JSON per clip, skip clips already done. Run it in the background; read transcripts as they land.

## plan.py
The story is a list of ranges, in output order, by source clip and word times:

```python
RANGES = [
    ("0067", 1.24, 6.30, dict(group="hook1", zoom=1.15)),
    ("0067", 10.45, 13.149, dict(group="laugh", zoom=1.0, protect=True, raw=True)),  # human moment, exact edges
    ("0085", 5.72, 9.46, dict(group="intro", zoom=1.15, tail=0.15)),
    ("0092", 43.22, 46.26, dict(group="money", squeeze=0.10, keep=0.03, zoom=1.15)),  # speaker looks down: squeeze hard
    ...
]
```
- `group` names drive captions/specials/SFX later. Alternate `zoom` 1.0 / 1.15 per line, 1.2-1.3 for punchlines.
- Different takes can be mixed freely; keep a take map so "that line isn't clean" is a one-line swap.

Per range:
1. RMS envelope per clip (10 ms hops, cached `.npy`); `speech = db > percentile(db,10) + 8`.
2. Edges from sound, not Whisper: walk back from `a` (max 0.25 s) while `speech`, start = onset - 0.03; walk forward from `b` (max 0.20 s) while `speech`, end = offset + 0.03 + tail. `raw=True` uses `a,b` exactly.
3. `start = max(start, last_end[clip])` so audio never replays.
4. Unless `protect`, split at inner silences longer than `squeeze` (default 0.16 s), keeping `keep` (0.05 s) each side.
5. Quantize each piece to whole frames: `e = s + round((e-s)*fps)/fps`. Mark the first piece of each range `range_start=True` (L-cut point).
6. Map words to the output timeline: include words that start inside the range OR were "swallowed" (start before `a`, end after `a+0.03`); dedupe by `(clip, start, word)`; `out_end = to_out(end)`, `out_start = max(to_out(start), out_end - 0.45)` (Whisper stretches word starts back across pauses).

## render_pieces.py
- Video per piece, audio per piece, rendered separately; concat each with the concat demuxer and mux.
- Audio: `-ss start -t dur+0.1`, `aresample=48000,afade in/out 15 ms,apad,atrim=end_sample=n*2002` (2002 samples per frame at 48 kHz / 23.976).
- Video window shifted for L-cuts: `shift[i] = 2 frames if range_start and i>0`; piece i covers output frames `[start_i+shift_i, start_{i+1}+shift_{i+1})` from source `start + shift/fps`. Lip sync holds; at the cut the old shot runs 2 frames under the new line.
- Filter: `crop(zoom around face anchor) -> scale 2160x3840 lanczos -> format=rgb48le,lut3d -> curves/colorbalance/eq -> cas=0.45 -> format=yuv420p10le,noise=c0s=5:c0f=t+u`; encode `libx264 -crf 12 -pix_fmt yuv420p10le -preset veryfast`.
- Check: `ffprobe assembled.mov` duration == sum of planned piece durations (to the ms).

## captions.py (Pillow overlay, works without libass)
- Clean words: join `Y` + `'all`, `wait`+`list` -> `waitlist`, brand fixes, swear masks, then display with punctuation stripped (`re.sub(r"[.,!?;:]", "", w)`).
- Cues: <=3 words, <=16 chars, break on punctuation, gaps >=0.25 s, and before any special group. Words appear as spoken; only the newest word is active.
- Each word is rendered on its own canvas (style base/key/script), baseline-aligned, centred, scaled down if wider than 960 px.
- Specials (own renderer, replace captions during their window, end = min(last word + 0.7, next word - 0.04)): punchline slam, thesis line (small top line + giant slam words), product reveal, title build, moment card (laugh). Slam easing: `1.28 - 0.28*ease_out(min(age*fps/4, 1))`.
- Stickers: cutout resized to 560 px tall, white outline = `alpha.filter(MaxFilter(19))` under the image; pop 0 -> 1.12 -> 1.0 in 0.22 s, wobble `-7*side + 2.5*sin(age*5)` deg, out in last 0.15 s; at x=205 or W-205, y~560.
- Moving prop (car): smoothstep x from -620 to W+60 over 2 s, `y = base - |sin(k*pi*5)|*16`, two ghost copies 45/90 px behind at 32%/18% alpha.
- Whole layer gets one shadow pass: blurred alpha (radius 9) at 55%, offset (3,7).
- Pipe raw RGBA to `ffmpeg -f rawvideo -pixel_format rgba -video_size 1080x1920 -framerate 24000/1001 -i pipe:0 -c:v qtrle -pix_fmt argb`. Keep ONLY the previous frame as cache (identical frames are consecutive); caching every unique full frame costs ~8 MB each and can exhaust RAM.
- Write `captions.json` with every special window, sticker and prop start time, so `master.py` places SFX from data, not guesses.

## master.py
- Synthesize the SFX kit with numpy (no downloads): sub hit (68 -> 38 Hz sine sweep + click), soft hit (-4 dB copy for repeats within 1.5 s), whoosh (noise, low->high blend, sin^2 envelope), service-bell ding (2093/2637/4186/5920 Hz partials, fast attack), engine pass-by (rising saw 70 -> 180 Hz + rumble).
- SFX list from `captions.json`: punchline hit on the slam, whoosh 0.3-0.45 s before a sticker or turn, themed SFX on its sticker, engine on the prop, hit on the thesis word, soft hit on the reveal.
- Voice chain -> `amix normalize=0` with delayed SFX -> two-pass `loudnorm=I=-14:TP=-1.5:LRA=11` (linear) -> `alimiter=limit=0.84` -> 24-bit wav, used by every export.
- Exports: 4K = `[captions]scale=2160:3840:flags=lanczos` overlaid on the 4K picture, `libx265 -preset fast -crf 16 -pix_fmt yuv420p10le -tag:v hvc1`; 1080 = picture scaled down + native overlay, `libx264 -preset slow -crf 16 -profile:v high`; no-captions 1080; srt.
- QA: `ebur128=peak=true` (expect I -14.0, true peak <= -1.4), a 12-frame contact sheet at every special/sticker/prop time, durations equal.

## Timing on a 6-core Intel Mac (84 s cut, 52 pieces)
transcribe ~5 min (small.en int8) · pieces 2 min at 1080 / 14 min at 4K · overlay 2-3 min · master 3-10 min (x265 4K dominates).
