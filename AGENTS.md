# AGENTS.md — music & YouTube channel videos

Notes for AI agents working with Thor's music (the songs in this repo) and his
YouTube channel (@RealThorwegian).

## The approved channel-video look (Thor, 18 Sep 2026)

ffmpeg `showspectrum`, waterfall style:

- **orientation=horizontal** — the spectrum scrolls UP, not sideways
- **color=cividis** — dark colormap, picked by Thor from sampler grids
- `scale=log`, `mode=combined`, `legend=0`
- **RGB level curve, darkening direction**, applied to the spectrum only:
  `eq=gamma_r=0.4545:gamma_g=0.4545:gamma_b=0.4545` (per-channel gamma 0.4545
  = power 2.2). Thor's spec: "apply exp curve … RGB level curve … the other
  way". The brightening direction was rejected, so was master gamma.
- **Render 135x135** (FFT 256), `fps=60`, lanczos upscale to 1080x1080. The
  small render also makes the vertical scroll ~4x faster than a 270 render.
- 60 fps, h264 ~8 Mbps (h264_videotoolbox on the Mac), AAC 48 kHz,
  `-movflags +faststart`.
- Renders run on the Mac mini (ffmpeg 8.0.1 at /opt/homebrew/bin/ffmpeg),
  reached by SSH: `ssh -p23000 thor@home.thj.no` then
  `ssh thor@192.168.0.100`. Ship render scripts as files, never inline long
  filtergraphs through the double hop.

### "Vertical centre = audio" (Thor's spec)

The vertical centre of the frame carries the audio's energy: a white flash
band at the centre rows, driven by the bass envelope (0-120 Hz block FFT,
max-pooled to 60 fps, fast attack / ~120 ms release, percentile-scaled
15/97). Overlay as RGBA raw frames at the render size (135x135) on the
spectrum strip, before the upscale. The level curve applies to the spectrum
only; the white flash is not affected.

### Reference chain (excerpt example)

```bash
ffmpeg -y -v error -ss 77.5 -t 22.3 -i TRACK.mp3 \
  -f rawvideo -pix_fmt rgba -s 135x135 -r 60 -i flash.raw \
  -filter_complex "[0:a]showspectrum=size=135x135:orientation=horizontal:mode=combined:color=cividis:scale=log:slide=rscroll:legend=0,fps=60,format=rgb24,eq=gamma_r=0.4545:gamma_g=0.4545:gamma_b=0.4545[spec];[1:v]format=rgba[flash];[spec][flash]overlay=format=auto:repeatlast=0,scale=1080:1080:flags=lanczos,setsar=1,format=yuv420p[v]" \
  -map "[v]" -map 0:a \
  -c:v h264_videotoolbox -b:v 8000k -c:a aac -b:a 192k -ar 48000 \
  -movflags +faststart OUT.mp4
```

## Rules (learned the hard way)

- Re-rendering a published track: MATCH its look unless Thor asks for the
  new one. He rejects wrong-look re-renders ("VERY wrong", "Nope", "Try
  again").
- "It was going up before and it isnt now": the waterfall must scroll UP.
- Colours are ffmpeg presets; no colour forensics needed. When in doubt,
  show him a 2x2 sampler grid and let him pick.
- Excerpt windows: pick dense/energetic sections, ~20 s with a lead-in.
- Previews go to Thor in Discord (~15 MB); the 8 Mbps final stays in the
  Mac's ~/Downloads; uploading to YouTube is his step.
