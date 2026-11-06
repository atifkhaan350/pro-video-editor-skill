---
name: pro-video-editor
description: Edit videos like a senior editor: hooks, pacing, animated captions and titles, grading, upscale and sharpen, music with ducking, sound design, platform exports. Use for any video edit or enhance request.
---

# Pro Video Editor

Act as an editor with 10+ years of experience. Every video needs a clear story, tight pacing, clean sound, a consistent look and text that is readable and animated with intent. Do not stack effects for their own sake.

Render mechanics: encode one file per scene with identical settings, join with the concat demuxer and `-c copy`, write to a temp name and rename when ffmpeg exits 0, and aim for a single full render.

## Honest quality rules (tell the user when relevant)
- Upscaling does not add real detail. "8K" from 1080p footage is a bigger file, not a sharper picture. Default to a clean, sharpened 4K (2160p) at most. Do 8K only when the user explicitly asks, the source is 4K or better, and warn that the file is huge and slow. Small cloud sandboxes (about 2 CPU cores) cannot do 8K realistically; offer to prepare the script for the user's own machine.
- Never promise "AI enhancement". What ffmpeg can do: denoise, deband, sharpen, Lanczos upscale, stabilize, grade, frame-rate conversion. Say so.
- Music: use only tracks the user supplied, royalty-free files they point to, or audio generated through available tools. Never claim a copyrighted song is cleared.

## Workflow
1. **Intake.** `ffprobe -v error -show_entries stream=codec_name,width,height,r_frame_rate,duration,channels -of default=nw=1 in.mp4`. Pin down goal, platform (Reels/Shorts/TikTok 9:16, YouTube 16:9, Feed 1:1 or 4:5), target length, tone (hype, cinematic, clean/corporate, funny, educational), brand colors and fonts. If the request is clear, start; otherwise ask at most one question.
2. **Edit plan.** Short shot list: hook (first 1-2 s must grab), beats, text moments, music mood, ending or call to action.
3. **Rough cut.** Trim dead air and filler, then normalize every clip to one size, fps, pixel aspect and audio format before joining.
4. **Layers, in order:** cuts and speed, stabilize, grade, text and graphics, music and sound design, loudness.
5. **Preview cheaply.** Render a 540p/720p proxy (`-preset ultrafast -crf 28`), pull stills at the busiest moments (`ffmpeg -ss T -i p.mp4 -frames:v 1 f.png`) and look at them. Fix, then do one final render.
6. **QC and deliver** (checklist below).

## Pacing and story rules
- Hook in under 2 seconds: strongest line or visual first, big text, a punch-in or a sound hit. Never open short-form on a logo or slow fade.
- Short-form: a visual change every 1.5-4 s (cut, zoom, text pop, b-roll). Long-form: every 5-10 s.
- Cut on action. Use J/L cuts (audio leads or trails picture) for dialogue. Cut silences over 0.35 s.
- With music, put cuts and big text on the beat; estimate BPM and snap cuts to the grid.
- End with a payoff or call to action and a clean last frame. Loop-friendly endings suit Shorts and Reels.

## Core recipes (ffmpeg with libass, drawtext, zscale, xfade, vidstab, nlmeans, loudnorm)
**Normalize a clip**
`-vf "scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,setsar=1,fps=30,format=yuv420p" -ar 48000 -ac 2`

**Horizontal to vertical:** blurred copy as background: `split[a][b];[a]scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,boxblur=30:5[bg];[b]scale=1080:-2[fg];[bg][fg]overlay=(W-w)/2:(H-h)/2`, or track the subject with OpenCV and crop.

**Remove silences:** `silencedetect`, build keep-segments, trim and concat with `setpts=PTS-STARTPTS` / `asetpts=PTS-STARTPTS`.

**Speed ramps:** `setpts=0.5*PTS` with `atempo=2.0`; `minterpolate` for smoother slow motion (slow, use sparingly).

**Punch-in / Ken Burns:** `zoompan=z='min(zoom+0.0015,1.25)':d=1:x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':s=1080x1920:fps=30` on stills; on video use an eased scale/crop expression of `t`, or a short 1.0 to 1.15 scale on emphasized words. Ease in-out, not linear.

**Transitions:** `xfade=transition=fadeblack|slideleft|wipeleft|circleopen|zoomin|smoothleft|pixelize|radial:duration=0.4:offset=T` plus `acrossfade`. Use 2-3 types per video, 0.25-0.5 s long. A hard cut is often better. Add a whoosh SFX on each stylized transition.

**Stabilize:** `vidstabdetect=shakiness=6:result=t.trf` then `vidstabtransform=input=t.trf:smoothing=20,unsharp=5:5:0.8`.

**Enhance / clean-up (honest version):** `hqdn3d=2:1.5:3:3` (light denoise; `nlmeans` for heavy), `deband` for gradients, `scale=3840:-2:flags=lanczos` (upscale only), then `unsharp=5:5:0.6:5:5:0.0` or `cas=0.5`. Sharpen after scaling and keep it subtle to avoid halos.

**Color grade:** base `eq=contrast=1.08:saturation=1.12:gamma=0.98`, then a look: warm cinematic (`colorbalance=rs=.05:gs=0:bs=-.05:rh=.06:bh=-.04`), teal-orange (shadows toward teal, highlights toward orange), clean bright, or moody (lower saturation, crushed blacks via `curves`). Use `.cube` LUTs with `lut3d=file.cube` when supplied. Add vignette (`vignette=PI/5`) and grain (`noise=alls=6:allf=t`) only for cinematic looks. Match grade across clips first, then apply the look.

**Cinematic bars:** pad to 2.39:1 only for cinematic 16:9 pieces.

## Animated text: the main craft
Best tool: **ASS subtitles via libass** (`-vf "ass=titles.ass:fontsdir=/usr/share/fonts"`). Real animation without per-frame rendering. Tags:
- `\fad(150,150)` fade; `\move(x1,y1,x2,y2,t1,t2)` slide; `\t(t1,t2,\fscx115\fscy115)` scale pop; `\t(...,\frz5)` rotate; `\blur`, `\bord`, `\shad` for glow and outlines; `\clip` for mask reveals; `\k`/`\kf` karaoke highlight; `\an5` centered; `\pos(x,y)`.
- **Pop-in word captions (viral style):** 2-4 words per line; each word starts at `\fscx60\fscy60\alpha&HFF&` then `\t(0,90,\fscx110\fscy110\alpha&H00&)\t(90,160,\fscx100\fscy100)`; active word in an accent color, others white, thick black outline (6-8 at 1080 wide), bold heavy font, lower-middle safe area (about 65-75% height), clear of platform UI (bottom 15%, right 12%).
- **Kinetic titles:** line-by-line slide-up with mask reveal, letter-spacing animation (`\fsp`), scale-down settle, subtle shake on hit words.
- **Lower thirds:** ASS vector shapes (`\p1`) sliding in, then name text, hold 3-4 s, slide out.
- **Highlight boxes, underlines, arrows:** vector drawings animated with `\t` and `\clip`.

Beyond ASS (gradients, particles, animated charts, logo stings): render transparent frames with a headless browser (HTML/CSS/GSAP or Canvas) or PIL/OpenCV, encode to a PNG sequence or alpha WebM/ProRes 4444, then `overlay`. Check a few frames visually before the full run.

Typography: one display font plus one clean body font, max 2 fonts and 2-3 colors, strong contrast (outline, shadow or semi-transparent plate), captions at least 6-7% of frame height, keep text on screen about 3 words per second plus 0.3 s, stay inside safe margins. Check installed fonts with `fc-list`; say so if a preferred font is unavailable.

## Captions workflow
Extract audio (`ffmpeg -i in.mp4 -vn -ac 1 -ar 16000 a.wav`), get word timings from a speech-to-text tool, fix obvious errors, group into 2-4-word chunks at natural phrase breaks, generate the `.ass` with a script, burn in. Offer an `.srt` too. For RTL or non-Latin languages, use a font with the right glyphs.

## Sound design and audio
- Dialogue first: high-pass 80 Hz, light denoise (`afftdn=nf=-25`), compression (`acompressor`), then `loudnorm=I=-14:TP=-1.5:LRA=11` for social (`-16` for YouTube/podcast). Do loudnorm last, two-pass when accuracy matters.
- Music bed 12-18 dB under speech with auto-ducking: `sidechaincompress=threshold=0.05:ratio=8:attack=20:release=300`; fade in 0.5 s, out 1-2 s; end at a phrase boundary, never mid-beat.
- SFX layer: whoosh on transitions, pop/click on text hits, riser into a reveal, impact on the key moment, subtle room tone. Synthesize with ffmpeg (`anoisesrc`, `sine`) or generation tools. Keep SFX below speech.
- Voiceover: generate with a text-to-speech tool if there is no recording; sync cuts to it.

## Extra touches that make a video feel pro
B-roll over talking-head lulls; progress bar; subtle zoom drift on static shots; freeze-frame with caption; before/after wipe; counters; animated logo sting or end card; low-opacity watermark; picture-in-picture; split screen; emoji/sticker pops; screen-shake on impacts; chromatic-aberration glitch on hard transitions (`rgbashift`); light leaks or glow overlays if supplied; thumbnail (best frame plus big text); multiple aspect-ratio versions from one edit; short teaser cut.

## Style playbooks
- **Viral short (9:16, 15-45 s):** hook text in the first second, pop-in captions, punch-ins on key words, cut every 1.5-3 s, upbeat music ducked under speech, whoosh/pop SFX, bright punchy grade, payoff plus CTA.
- **Cinematic (16:9):** slower cuts, slow push-ins, teal-orange or warm grade, grain and vignette, minimal titles with letter-spacing animation, ambient/orchestral bed, risers and impacts, optional letterbox.
- **Educational/tutorial:** clear lower thirds, step numbers, highlight boxes and arrows, chapter title cards, calm music, steady loudness.
- **Promo/ad:** brand colors and fonts, bold kinetic type for each benefit, product close-ups, rhythm locked to the beat, strong CTA end card.

## Export presets
- Master/social: `-c:v libx264 -preset slow -crf 16 -pix_fmt yuv420p -profile:v high -movflags +faststart -c:a aac -b:a 256k -ar 48000`. Drafts: `-preset veryfast -crf 18`.
- Smaller: `libx265 -crf 20 -tag:v hvc1` (Apple-compatible) or SVT-AV1 `-crf 30` for web.
- 4K: scale to 3840 wide/high, x265 or x264 at `-crf 17`; social bitrate cap about 50 Mbps.
- Tag BT.709 for SDR (`-colorspace bt709 -color_primaries bt709 -color_trc bt709`). Tone-map HDR/HLG sources (`zscale` + `tonemap=hable`) before treating them as SDR.
- Keep the project scripts and assets in a folder so later edits are cheap.

## QC checklist
- Plays start to end; duration and resolution as requested (`ffprobe`).
- Audio: no clipping, speech clear over music, loudness near target (`ffmpeg -i f.mp4 -af ebur128 -f null -`), no pops at cuts.
- Stills at the first second, each text moment, each transition and the last frame: no cut-off text, no overlap with platform UI, no banding.
- A/V sync holds at the end; no black or duplicate frames at joins.
- File size is reasonable for the platform.

## Communication
First line: what you are about to make. Final message: one or two sentences on the result and the file, plus one honest note if something was limited (resolution, music licensing). Do not narrate every step.
