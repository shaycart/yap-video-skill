---
name: yap-video
description: Edit raw talking-head / "yap" footage into punchy short-form video (9:16 Reels/TikTok/Shorts, or 16:9). Transcript-driven jump cuts, filler and stutter removal, face-aware punch-in zooms, word-by-word animated captions, sound design, loudness mastering, QA and publish copy. Use when the user gives talking-head footage, a yap, a rant, a podcast clip or a long ramble and wants it edited, cut down, captioned or turned into shorts. Also use for "make this fire", "edit this video", "caption this", "cut the dead air".
---

# Yap Video Editor

Turn raw talking-head footage into a tight, designed short. Everything runs locally with ffmpeg + local Whisper; no API key is required (ElevenLabs Scribe is an optional, better transcriber).

Synthesized from a security-reviewed read of: browser-use/video-use, adukhan98/video-edit, chrislema/videoeditor, zyziyun/video-studio, eyadhussain00-hub/Videoediting (ideas only), AgriciDaniel/claude-shorts, Barty-Bart/motion-graphics, ZiadAbdelkarim/beat-synced-edit, 6missedcalls/video-editing-skill, haidrrrry/claude-remotion-skill, digitalsamba/claude-code-video-toolkit, blixvip/easyedit (ideas only).

Exact numbers live in `references/params.md`. Read it before building filtergraphs.

## Hard rules

1. **Plan first, then wait.** After transcribing, show a plain-English plan (hook, what gets cut, zoom/caption/SFX approach, length, format) and get a yes before rendering. Taste calls (hook line, which sentences go, caption style) are offered as a short menu with a recommendation, never chosen silently.
2. **Never overwrite the source.** Write to `<folder>/edit/`. Keep intermediates until the user approves, then offer to clean up (list files + sizes first; never auto-delete).
3. **Never cut inside a word.** Pad cut edges (50 ms before first kept word, 80 ms after last), snap to the quietest 10 ms within +/-0.25 s, 30 ms audio fade at every join.
4. **Re-encode cuts, never `-c copy`** for anything frame-accurate (copy lands on keyframes and glitches).
5. **Captions go last**, after zooms and overlays, so zooms never crop them.
6. **Prove it.** Render, then inspect: frames at every cut +/-1.5 s, contact sheet, `ebur128` loudness readout. Max 3 fix passes, then report what is left. Report numbers; you cannot hear the audio.
7. **Third-party media:** only user-owned, CC0/CC-BY, or generated. No ripping film/music from YouTube. Never read, send or reuse Claude/Anthropic login tokens or other credentials; never copy other repos' `.claude/` hook configs.
8. **Ask before downloading** models (whisper ggml-medium is ~1.5 GB) or installing packages. State name, source, size.

## Pipeline

### 0. Ingest and brief
- `ffprobe` each clip: resolution, fps, duration, audio. Detect orientation.
- Ask only what matters (use a short menu): target platform/aspect, target length, vibe (see Vibes), caption style, music yes/no, any must-keep lines. If the user says "just do it", default to: 9:16, 30-60 s, "creator" vibe, bold captions, light SFX, no music.

### 1. Transcribe (word-level, verbatim)
- Prefer: mlx-whisper (Apple Silicon) or faster-whisper `medium.en`/`distil-large-v3`, word timestamps on. Use medium, not small: small drops false starts. Optional: ElevenLabs Scribe if the user has a key (never write the key into the project folder).
- Keep fillers in the transcript (verbatim) so they can be cut. Cache the transcript.
- Whisper quirks: it swallows long pauses into one word (a word lasting seconds is a swallowed pause; use an RMS envelope for true speech spans), starts words 0.2-0.4 s late after a pause, and hallucinates on silent tails. Trim at real content end. For caption timing, re-transcribe per final segment.
- Proofread names/terms; apply a fix list (`chat gpt` -> `ChatGPT`).

### 2. Pick the story (for long yaps)
- Score candidate segments 0-100: `hook*0.30 + coherence*0.25 + emotion*0.20 + value*0.15 + payoff*0.10`. Strong hooks: bold/contrarian claim, curiosity gap, specific number, pattern interrupt. Weak: "So today I want to talk about...". Standalone test: no "as I said earlier", no dangling pronouns.
- 15-55 s each (sweet spot 25-40), score >= 60, snap start to a word/sentence start (1.5 s window) and end to a sentence end (3 s window) + 300 ms pad.
- Hook is written/chosen LAST: offer ~5-12 numbered candidate openers with text, type and length. Result-first or bold-claim cold open beats a slow intro. First 1-3 s decide everything.
- For one-take yaps: reorder only if it improves the hook; otherwise keep the speaker's order.

### 3. Cut (tighten)
Tightness profiles (pause squeeze; pauses are shortened, not deleted):
| profile | squeeze pauses over | keep |
|---|---|---|
| gentle | 0.80 s | 0.35-0.70 s |
| standard | 0.45 s | 0.18-0.40 s (0.30 after sentence end) |
| tight (default for yap) | 0.25 s | 0.08-0.20 s |

- Detect: RMS envelope in 10 ms hops, speech = noise floor + 8 dB. Fallback: `silencedetect=noise=-30dB:d=0.5`.
- Remove: ums/uhs, stutters and restarts, repeated words, false starts, retakes (earlier take goes), slate/tail, throat-clearers ("So,", "Basically"). Semantic fillers ("like", "you know") only when the audio isolates them. Keep deliberate repetition for emphasis.
- Low-confidence or ambiguous cuts (retakes, merged fillers, ASR noise) are confirmed with the user, not auto-applied.
- **Verify every join**: re-transcribe a window across each join; it must read as the intended words. Expect 2-3 rounds. After the full cut, diff the content words against the plan; fail if any are lost.
- Visual check: avoid cut edges where the speaker glances down at notes or mid-blink/head-move; end tight (<=30 ms) on head-move edges.
- Pacing: a new visual event (zoom, caption emphasis, graphic, b-roll) at least every 1.5-4 s; no static frame over ~8 s. Optional speed-up 1.1-1.25x on body (never above ~1.3x, it sounds fake; hook up to 1.5x is the ceiling).

### 4. Zoom (face-aware)
- Track the face: OpenCV Haar (`opencv-python-headless<5`) or MediaPipe; sample every 0.5 s per section, take the median, reject detections under 0.6x median width, dead-band 2% of frame width, never pan inside a section. If a section has fewer than 2 hits, borrow a neighbor.
- Zoom ladder per section (3-6 s each): normal 1.0x, emphasis ~1.15-1.25x, critical ~1.3-1.6x (landscape). For 9:16 from 16:9 use locked height fractions 1.0 / 0.92 / 0.85 (tighter looks uncomfortable).
- Mix by duration: ~60% normal / 30% emphasis / 10% critical. First section normal. Never more than 2 consecutive sections at the same level. Emphasis zoom at most 1 in 4 sentences.
- Style choice (ask or infer from vibe):
  - **Hard-cut punch-ins** (default for jump-cut yaps): alternate 1.0x and 1.15x across jump-cut segments, start zoomed on the hook, emphasis sentences at ~1.3x.
  - **Eased push-in** on build-ups: `scale=w='iw*z(t)':h='ih*z(t)':eval=frame` then fixed-size `crop`, cubic ease-out (`1-(1-t)^3`), 0.4-0.5 s in. Test on a 2 s clip first; zoompan is jittery.
  - **Punch-and-stay** (1.16x, 0.5 s) 1-2 times per video on the punchline.
- Vertical framing: face at ~30% from crop top; `cy = min(face_cy - 0.3*ch, head_top - 0.03*ch)`.
- Optional hit effects on peaks only: shake, 2-frame white flash, RGB split, speed ramp (see params).

### 5. Captions (animated, word-level)
- Generate libass `.ass` from word timestamps (static font instances only; variable fonts hang libass). Per-segment re-transcription for accurate timing.
- Defaults: 1-3 words per cue (<=18 chars), break on punctuation and pauses >=0.3 s, min cue 0.35 s (grow to 3 words rather than flash), hold 0.12 s after last word. Font ~4.6% of 1920 canvas height (~88 px), ExtraBold, heavy outline or soft shadow.
- Looks (render 3 on a real frame as a grid and let the user pick):
  - **Bold**: Montserrat 800, UPPERCASE, white, active word yellow `#FFD700`, 3 px black outline, spring pop-in.
  - **Bounce**: display font, 1-2 words, rotating accent colors, overshoot 0.7 -> 1.2 -> 1.0.
  - **Clean**: Inter 700, natural case, 3-5 words, soft fade.
  - **Hormozi-lite** (fallback): white with black outline, centered.
- Emphasis: ~1 keyword in 4 words gets color/scale 112%; numbers/money auto-colored (e.g. `#3CE07A`); one keyword per caption group gets 2-4x size treatment in "designed" vibe. Pop on spoken word: scale 108-112% eased back in ~110 ms.
- Placement: respect platform safe zones (universal 9:16 safe area x60-940, y285-1480). Reels caption y~1360, TikTok ~1400, Shorts ~1440. Never over the mouth/chin. Lines never exceed 4 words, balanced stacking.
- No condensed display faces as the default (reads cramped).

### 6. Sound design
Order of work: music (if any) first, picture next, SFX last.
- **Cues:** whoosh on layout/scene changes and big camera moves (start 2 frames early; start by attack time before the contact frame); pop on graphic/word-pop entry; impact/sub-drop ONCE on the hook or reveal; riser that ENDS on the next hit; tick/flick sparingly on highlights; typing = keys.
- **Restraint (most important):** SFX mark changes, not every word. Nothing on filler. About 8 in 18 s reads designed; about 20 reads generic. Density cap: 2 per 2 s window for talking-head. Same SFX within 1.5 s alternates variant and steps down 1.5 dB per repeat. Intensity ladder: casual yap = sparse; hype reel = sound on most beats; corporate = near none.
- **Sources:** user's library, or generate (ElevenLabs sound-effects/generated pack), or the synthesized kit approach (whoosh/pop/tick/impact/riser built with numpy/ffmpeg; no downloads). Place by transient peak, +/-1 frame.
- **Music:** optional. Bed ducked 10-15 dB under voice (`sidechaincompress=threshold=0.02:ratio=8:attack=20:release=350`), music gain around -18 dB when ducked. For beat-cut vibes: detect beats (librosa), cut every beat / 2 beats / bar, verify cuts land within 3 frames.
- Voice chain, mix, limiter and loudness targets are in `references/params.md`.

### 7. Look
- Light grade only: warm punch preset or auto-grade bounded to +/-8%. Do not blur static talking heads (looks fake). Optional grain 3-6% and subtle vignette for the "designed" vibe.
- **Log footage (DJI Osmo Pocket 3 D-Log M, 10-bit HEVC; camera tags it bt709 even though it is log):** convert first with the DJI D-Log M -> Rec.709 LUT, then grade on top. Work in 16-bit: `format=rgb48le,lut3d=file='<lut>.cube':interp=tetrahedral`. The LUT darkens mids; for low-key indoor shots of darker skin, lift mids rather than crushing. Tested starting point (look: rich blacks, slightly cool room, natural skin): `curves=master='0/0 0.04/0.025 0.25/0.31 0.6/0.70 0.9/0.92 1/0.98',colorbalance=rs=-0.025:bs=0.04:rm=0.03:bm=-0.015,eq=saturation=1.06,unsharp=5:5:0.3`. Always render raw | LUT | grade side by side on 2 frames and look before batch rendering. Grade shots individually when lighting changes; keep the source frame rate (24000/1001).
- Pocket 3 vertical clips are 1728x3072 (or 2160x3840): enough headroom for 1.15-1.3x punch-ins at 1080x1920 output.

### Rendering captions without libass
If `ffmpeg -filters` lacks `subtitles`/`ass` (common Homebrew builds), draw caption frames with Pillow (RGBA) and pipe raw frames to ffmpeg as an alpha overlay: `-f rawvideo -pixel_format rgba -video_size WxH -framerate 24000/1001 -i pipe:0 -c:v qtrle -pix_fmt argb captions.mov`, then `overlay` it last. Cache identical frames. Full control (word pop, active-word color, stroke, shadow) without libass.

### 8. Render and QA
- Master: H.264 High, bt709, `yuv420p`, AAC 48 kHz 192-320k, `+faststart`. Re-render per platform via preset (YouTube 12M, TikTok crf 18 maxrate 10M, Reels 4.5M).
- Self-check loop (max 3 passes): contact-sheet frames at every cut +/-1.5 s plus first/last 2 s and midpoints (faces cut off, bad crops, caption collisions, text over mouth); `verify_cuts` on joins; `ffprobe` duration; `ffmpeg -i out.mp4 -af ebur128=peak=true -f null -` and report LUFS + true peak; no pause >=0.2 s left inside a line; ending must not be the speaker reaching for the phone.
- For anything publishable, brief a separate critic agent to roast the edit: ranked problems, timecodes, evidence, top 5 fixes. Apply, re-check, deliver.

### 9. Deliver
- Output folder `edit/` with: `final_<platform>.mp4`, caption-free master, `.srt`, and (optional) with-music / without-music versions.
- Publish copy (offer, don't impose): platform-specific title (two angles: search/value vs hook/punchline, give 3-4 variants and an honest verdict), body = hook + honest setup, 8-16 tags with the 8 most important first (Shorts 0-2, TikTok/Reels 3-5), chapter timeline in final-video time for longer cuts.
- Cover/thumbnail: bright, full face, big type, same canvas ratio; check platform crop (e.g. 4:3 grid crop).
- Final Cut Pro users: offer an FCPXML of the cut list (cuts, markers, caption titles) so the edit stays editable; say plainly it was not test-imported here.

## Vibes (presets; the user can mix)
- **Creator / fire (default):** tight cuts, alternating 1.0/1.15 hard punch-ins, bold word-pop captions, sub-drop on hook, whoosh on layout change, no music.
- **Podcast clean:** gentle profile, slow drift 1.00 -> 1.03 per section, clean captions, no SFX, light music bed.
- **Hype / meme:** tight profile, speed ramps, shake and flash on punchlines, bounce captions, SFX on most beats, beat-cut montage cutaways.
- **Documentary / premium:** standard profile, restrained zoom (1.0/1.08), clean captions around the subject, pauses kept for breath, designed lower-thirds and chapter cards.

## B-roll and graphics (optional layer)
- Cutaway rules: none in the first second of the hook, none on emotional/opinion lines, at least ~2 s of face between cutaways, 3-10 s each; light 2-4/min, medium 4-7/min, heavy only for explainer vibes.
- Sources: user footage, generated clips/stills (image/video generation tools), or HTML/Remotion motion graphics rendered deterministically. Overlays use `setpts=PTS-STARTPTS+T/TB`; subtitles after overlays.
- Hold graphics 0.3 s/word + 0.5 s reading time; animate in 200-500 ms (ease-out-quint, 2-3 properties), out faster (160 ms); never reveal two new elements at once; always cubic/spring easing, never linear.

## Tools used
ffmpeg/ffprobe (with libass), local Whisper (mlx-whisper / faster-whisper / whisper.cpp), OpenCV or MediaPipe (face), numpy/librosa (envelope, beats). Optional: Remotion or HyperFrames (graphics), ElevenLabs (Scribe, SFX), yt-dlp (only for user-supplied or licensed sources).

## Known gaps (be honest with the user)
- No ear: loudness is measured, timing/feel of comedic pauses is a judgment call; offer a with/without-music A/B and expect one feedback round.
- Smooth-zoom, emphasis-word picking and beat sync are LLM-driven here, not turnkey scripts: test short, then batch.
- Local Whisper drops/misaligns words after long pauses; verify joins.
