# Yap video: parameter cookbook

All values were taken from reviewed repos and are starting points. Test on a 5-10 s clip before batch rendering. Always output `-pix_fmt yuv420p` and `setsar=1`.

## Silence / cuts
```
# detect
ffmpeg -i in.mp4 -af silencedetect=noise=-30dB:d=0.5 -f null -
# keep padding: 0.1-0.15 s each side; kept pause 0.3 s (standard), 0.08-0.2 s (tight)
# join frame-accurately (no -c copy):
#   trim/atrim + setpts/asetpts + concat inside -filter_complex_script
-c:v libx264 -preset fast -crf 18 -c:a aac -b:a 192k
# per-boundary audio fades (30 ms):
afade=t=in:st=0:d=0.03,afade=t=out:st={dur-0.03}:d=0.03
```
- Edge pad: 50 ms before first kept word, 80 ms after last; snap edge to quietest 10 ms within +/-0.25 s.
- Safe cut windows: silence >=400 ms clean; 150-400 ms check visually; <150 ms unsafe. Speaker handoffs keep 400-600 ms.
- Crossfade at joins in cleanup profiles: 0.02 s; pad 0.035 s beside kept word.
- Keep a `segment_map` JSON (source time -> output time) so captions/overlays/SFX remap correctly: `out_t = word.start - seg_start + seg_offset`.
- Map full audio track rather than per-section audio to avoid cumulative drift when zoom-cropping video only.

## Zoom
```
# hard-cut punch-in (crop then scale back; even dimensions; clamp inside frame)
crop=CW:CH:CX:CY,scale=W:H:flags=lanczos
# punch via split/overlay for timed sections:
split -> crop -> scale -> overlay=enable='between(t,s,e)'
# eased push-in (test first)
scale=w='iw*(1+0.15*(1-pow(1-min(t/0.45,1),3)))':h=-2:eval=frame,crop=W:H
```
- Ladder: 1.0 / 1.25 / 1.6 (landscape); 9:16 from 16:9: height fractions 1.0 / 0.92 / 0.85. Short-form alternate: 1.0 / 1.15, hook zoomed at origin (x 0.5, y 0.36), emphasis 1.30-1.32.
- Sections 3-6 s (about 20 per minute); under 3 s looks manic. Mix 60/30/10. Max uses: punch-in ~1 per 20 s; punch-and-stay 1-2 per video.
- Slow drift on stills/talking head: 1.00 -> 1.03 per section; Ken Burns on stills 1 -> 1.08 alternate in/out.
- Face detect: `haarcascade_frontalface_default.xml`, `detectMultiScale(gray,1.1,5,minSize=(30,30))` on a 0.25x frame; or MediaPipe `model_selection=1`, confidence 0.5.

## Hit effects (peaks only; one signature effect + one grade per edit)
```
white flash   fade=t=in:st=0:d=0.07:color=white
punch-in      scale=iw*1.15:ih*1.15,crop=iw/1.15:ih/1.15,setsar=1
h-stretch hit scale=iw*1.25:ih,crop=iw/1.25:ih,setsar=1
rgb split     rgbashift=rh=6:bh=-6
shake         crop=iw-20:ih-20:'10+8*sin(t*40)':'10+8*cos(t*37)'
strobe        negate=enable='lt(mod(t,0.25),0.04)'
speed ramp    setpts=0.5*PTS  (fast) / setpts=2*PTS + minterpolate (slow)
xfade flash   xfade=transition=fadewhite:duration=0.5:offset={t}
glow          gblur=sigma=8,blend=all_mode=screen,all_opacity=0.35 (via split)
```
Speed: audio `atempo` chained in factors of 2.0/0.5 for extremes; pitch-preserving, floor 0.85. Speech speed: body 1.2-1.25x, hook up to 1.5x, never above ~1.3x in English long-form (sounds artificial).

## Captions (libass force_style starting points)
```
# bold-overlay (video-use)
FontName=Helvetica,FontSize=18,Bold=1,PrimaryColour=&H00FFFFFF,OutlineColour=&H00000000,BackColour=&H00000000,BorderStyle=1,Outline=2,Shadow=0,Alignment=2,MarginV=35
# Hormozi-lite fallback (small at 1080p; scale for PlayRes)
FontName=Arial Black,FontSize=28,Bold=1,PrimaryColour=&H00FFFFFF,OutlineColour=&H00000000,BackColour=&H80000000,Outline=3,Shadow=2,Alignment=10,MarginV=40
```
- Per-word karaoke: generate `.ass` with `\k`/`\t` tags. Pop: `\fscx108\fscy108\t(0,90,\fscx100\fscy100)`; pop-in 86% -> 95% -> 100% over 67 + 33 ms. Active-word color `#FFD84D` / `#FFD700`, `active_pop` 12% eased back in 110 ms.
- Remotion route: `createTikTokStyleCaptions({captions, combineTokensWithinMilliseconds})`; Bold 800 ms/2-3 words, Bounce 600 ms/1-2 words, Clean 1500 ms/3-5 words. Springs: snappy `{damping:14,stiffness:160,mass:0.6}`, bouncy `{damping:11,stiffness:170,mass:0.7}`, Bold pop `{mass:1,damping:12,stiffness:200}`.
- Font sizes: ~4.6% of canvas height portrait (~88 px at 1920), 6% landscape. Inter ExtraBold 92 px at 1080x1920 is a proven natural-case reel look.
- Safe zones 9:16 (universal x60-940, y285-1480): Reels caption_y ~1360, TikTok ~1400, Shorts ~1440. Hard top margin ~220 px, bottom ~420 px, right ~140 px for UI.
- Text hygiene: rejoin split contractions, typographic apostrophe, name-fix list, no leading punctuation, never `\n` in drawtext.
- Quiet-filler captions: strip um/uh/like from caption text but keep timestamps (audio cut is a separate step).

## Audio
```
# voice chain (pick one; both tested by their authors)
highpass=f=80,lowpass=f=14000,equalizer=f=3000:t=q:w=1.5:g=3,equalizer=f=200:t=q:w=1.0:g=2,acompressor=threshold=-21dB:ratio=3:attack=5:release=50,volume=0.6
highpass=f=85,lowpass=f=14500,equalizer=f=250:t=q:w=1.2:g=-2.5,equalizer=f=3200:t=q:w=1.0:g=2.2,equalizer=f=9000:t=h:w=1:g=1.5,acompressor=threshold=-20dB:ratio=3.2:attack=6:release=120:makeup=4,alimiter=limit=0.9
# loudness (two-pass), then limiter: AAC re-encode overshoots TP by 0.3-0.6 dB
loudnorm=I=-14:TP=-1:LRA=11   (social)   |  I=-16 for YouTube/podcast
alimiter=limit=0.79:attack=1:release=50:level=disabled
-c:a aac -b:a 320k -aac_coder twoloop -ar 48000
# verify
ffmpeg -nostats -i out.mp4 -af ebur128=peak=true -f null -
# duck music under voice
sidechaincompress=threshold=0.02:ratio=8:attack=20:release=350 ; amix normalize=0
```
- Gotcha: loudnorm can silently undershoot (-16.6 observed); add pre-gain + limiter.
- Levels: voice ~-16 LUFS stem; music bed ~-30 LUFS ducked ~10 dB more (or -18 dB gain when ducked); SFX peaks -18 to -10 dBFS, accent impact ~-7 dBFS used 2-3 times; mix priority voice > SFX > music. Check mono compatibility.
- SFX gains by type (dBFS peak): soft transition -14, whoosh -19 (or -4 dB relative gain), pop -16, tick -15, ding -19, impact -7, stamp -5, riser -12, sparkle -12. Talking-head profile: -3 dB offset.
- Riser: trim so it ENDS on the hit (`start = t + dur - len`). Finale = riser -> impact -> sparkle (+0.6 s, -3 dB).

## Grade
```
warm punch      eq=contrast=1.12:brightness=-0.02:saturation=0.88,colorbalance=rs=0.02:bs=-0.03:rm=0.04:bm=-0.02:rh=0.08:gh=0.02:bh=-0.05,curves=master='0/0 0.25/0.22 0.75/0.78 1/1'
indoor talking  colorbalance=rs=0.02:gs=-0.01:bs=-0.02,curves=m='0/0 0.25/0.20 0.75/0.82 1/1',eq=brightness=0.02:contrast=1.05:saturation=1.05
reel look       eq=contrast=1.10:saturation=0.86:gamma=0.96:brightness=-0.015,colorbalance=rs=-0.03:bs=0.05:rh=0.03:bh=-0.03,unsharp=5:5:0.45
auto-grade      signalstats-driven, bounded +/-8%
```
Grain 3-6%, vignette light. Do not apply focus blur to the static speaker.

## Export presets
```
YouTube  -c:v libx264 -preset slow -b:v 12M -maxrate 14M -bufsize 24M -profile:v high -level 4.2
TikTok   -crf 18 -maxrate 10M
Reels    -b:v 4500k -maxrate 5000k
all      -af loudnorm=I=-14:TP=-1:LRA=11 -pix_fmt yuv420p -movflags +faststart -c:a aac -b:a 192k -ar 48000
```
Apple Silicon: `h264_videotoolbox` is faster; NVENC `h264_nvenc -preset p5 -tune hq` on NVIDIA.

## Beat / montage (only for hype, meme, music vibes)
- librosa `beat_track(hop_length=512)`; energy RMS smoothed (gaussian sigma 5); peaks > 0.6, valleys < 0.3.
- Cut every beat by default; `beat_stride N` for breathing room; high-energy clips on peaks, low on valleys. Keep cut period 0.36-0.95 s (double/halve beat unit to fit). Verify cuts land within 3 frames.
- Final hero shot >= 3.4 s. Hard cut only on a downbeat, at most twice per transition sequence.

## Pacing rules of thumb
- Hook 0-3 s; payoff then CTA; one clear action CTA on screen >= 2 s.
- Reels cut every 0.5-2 s in hype mode; movement in the first 15 frames; no still frame > 1.5 s in hype mode, > 8 s in any mode.
- On-screen text <= 6-8 words; reading time 0.3 s/word + 0.5 s (min 1 s).
- Words/min: comfortable narration 140-160 wpm; > 170 reads rushed unless the style wants it.
