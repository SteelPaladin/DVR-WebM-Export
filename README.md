# WebM Alpha for DaVinci Resolve Studio

Renders **transparent VP9 WebM** straight from the Deliver page. Resolve hands frames to the plugin, which encodes them and writes
the `.webm` itself. Designed to get keyed footage out of Resolve to be used as mesh planes in Blender with out having to use large ProRes files. Transparent WebM files are on average **99.5% smaller** than transparent ProRes alternatives.

It adds two things to Resolve:

- a **format** called **WebM** (`.webm`)
- a **codec** group called **WebM Alpha**, with two variants:
  - **VP9 Alpha (RGBA)**: use this one.
  - **VP9 Alpha (AYUV 4:4:4)**: a backup in case Resolve handles RGBA badly.

Requires **DaVinci Resolve Studio**. The free version doesn't load encoder plugins.

## Install (macOS)

```bash
cd webm-alpha-resolve
./scripts/build_mac.sh
```

The script installs `cmake`, `pkg-config` and `ffmpeg` via Homebrew if they're
missing, builds the plugin, and copies it to
`/Library/Application Support/Blackmagic Design/DaVinci Resolve/IOPlugins/`.
It asks for your password for that copy. Quit Resolve before running it.

Uninstall with `./scripts/build_mac.sh --uninstall`.

## Render a transparent WebM

1. Make sure the timeline has transparency: a Fusion title, a keyed clip, or
   anything with nothing underneath it.
2. **Deliver** → **Custom Export**
3. **Format: WebM** → **Codec: VP9 Alpha (RGBA)**
4. Tick **Export Alpha** and set **Alpha Mode → Straight**.
5. Untick **Export Audio**. It's ignored in this version anyway.
6. Render.

If Export Alpha doesn't stick, older Resolve versions only exported alpha with
**Render: Individual clips** and the clip on **Video Track 1**. Try that.

Encoder settings on the Deliver page:

| Setting        | Default | Notes                                                                 |
|----------------|---------|-----------------------------------------------------------------------|
| Encoder Mode   | Fast    | **Fast**: several times quicker, files roughly 1.5-2x bigger at the same CRF. **Quality (slow)**: smallest files. |
| Quality (CRF)  | 30      | 0-63, lower = better/bigger. 24-32 is sensible for overlays.          |
| Encode Speed   | 4       | Quality mode only. 0 = slowest/best, 5 = fastest.                    |
| Keyframe Every | 2 s     | Shorter = easier scrubbing in web players.                            |

Since v0.4 the plugin encodes on a background thread while Resolve renders the
next frame, uses every CPU core for VP9 (tile columns + row multithreading) and
for the colour conversion. VP9 has no hardware encoder on Macs, so 4K will
still be slower than ProRes or HEVC; the log reports where the time went at
the end of each render ("Time spent: converting ..., encoding ..., Resolve
waiting on the encoder ...").

## Check the result

```bash
./scripts/check_webm.sh ~/Movies/render.webm
```

This prints whether the file has the AlphaMode flag and the range of alpha
values, then saves `render_frame.png` and `render_checker.png` (the middle frame
over a checkerboard) beside the file.

Chrome, Firefox, Edge and OBS (browser/media source) play these files with
transparency. **Safari does not support VP9 alpha.** Apple's transparent web video
route is HEVC with alpha, which needs a separate export.

## Diagnostics

After any test render, run `./scripts/collect_diagnostics.sh`. It writes
`~/Desktop/WebMAlpha-diagnostics.txt` with the Resolve version, plugin install and
load check, the plugin log and relevant lines from Resolve's own logs.

Everything the plugin sees is logged to **`~/Library/Logs/WebMAlpha.log`**:
every property Resolve passes in, whether Export Alpha arrived as on, and a
byte-by-byte analysis of the first frame. That analysis shows where the alpha
channel sits and whether it varies or is a flat 255. The log is reset each time
you reinstall.

| Symptom | What it means / what to try |
|---|---|
| No **WebM** in the Format list | Resolve didn't accept the container registration. Send the log. |
| WebM listed but no **WebM Alpha** codecs under it | The codec→container link format is different from the guess. Send the log. |
| No **Export Alpha** checkbox | Check the log for `Registered codec ... ` lines. Send the log. |
| File plays but is opaque | Look in the log for `hasAlpha=0`, or a `byte[3]` line with min 255 max 255. |
| Colours swapped / alpha wrong with the AYUV codec | Change **AYUV Byte Order** in the codec settings, guided by the byte analysis. |

## Limitations (v0.4)

- Video only: audio is accepted and discarded (WebM needs Opus/Vorbis audio; that can be added).
- 8-bit 4:2:0, which is what browsers play anyway.
- Links against Homebrew's FFmpeg. If `brew upgrade` bumps FFmpeg to a new
  major version, re-run `build_mac.sh`.

## How it works

VP9 has no native alpha. WebM stores transparency as a **second VP9 stream**
(the matte, encoded as greyscale) attached to each colour frame as a Matroska
*BlockAdditional*, with `AlphaMode=1` on the track.

- `src/vp9_alpha_encoder.cpp`: the codec Resolve talks to. It registers with
  `hasAlpha=1`, which is what makes Resolve show **Export Alpha**. It asks Resolve
  for RGBA (or AYUV) frames, converts them to YUVA 4:2:0, and encodes both
  streams with libvpx through FFmpeg.
- `src/webm_container.cpp`: a container plugin for `.webm`. Resolve only passes
  one buffer per packet between codec and container, so the codec packs colour
  and alpha together. The container unpacks them and writes the BlockAdditional.
- `tests/mock_resolve.cpp`: a stand-in for Resolve that loads the plugin through
  the same API and renders a synthetic clip. Used to test everything outside
  Resolve:

  ```bash
  cmake -S . -B build -DWEBM_ALPHA_BUILD_TESTS=ON && cmake --build build
  ./build/mock_resolve build/webm_alpha_plugin.dvcp.bundle/Contents/*/webm_alpha_plugin.dvcp out.webm rgba 48 1920 1080
  ```

`sdk/` holds Blackmagic's IOPlugin SDK headers and wrapper. It's the same code
that ships in Resolve Studio's `Developer/CodecPlugin` folder.
