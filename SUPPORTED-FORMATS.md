# Supported input formats

Video AV1 Optimizer is **not limited to H.264/H.265**.

The current script accepts a fixed set of video-file extensions, probes each file with `ffprobe`, selects the usable streams, and lets FFmpeg decode the source video. Most source codecs are **not blocked by codec name**: codecs the script does not explicitly classify fall back to its normal "standard" source heuristics.

That makes container support and codec support two separate questions:

1. **Will the script discover the file?** The extension must be in the list below.
2. **Can the installed FFmpeg build decode the video and remux the selected audio/subtitle streams into Matroska?**

## Accepted file extensions

The drag/drop and folder-scanning paths currently accept:

| Extension | Typical container/use |
|---|---|
| `.mkv` | Matroska |
| `.mp4` | MPEG-4 / ISO BMFF |
| `.m4v` | MPEG-4 video |
| `.ts` | MPEG transport stream |
| `.m2ts` | Blu-ray / AVCHD transport stream |
| `.avi` | AVI |
| `.mov` | QuickTime |
| `.wmv` | ASF / Windows Media |
| `.webm` | WebM |
| `.mpg` | MPEG program stream |
| `.mpeg` | MPEG program stream |
| `.vob` | DVD MPEG program stream |

The extension is the first gate. A codec can be perfectly decodable by FFmpeg and still not be picked up if it is stored in an extension that is not on this list.

For example, raw `.h264`, `.hevc`, `.vvc`, and `.apv` elementary streams are **not currently discovered**, nor are containers such as `.mxf`, `.flv`, `.ivf`, or `.mts`.

## Video codec support

### Explicitly classified by the script

These codecs have dedicated source-efficiency classifications that influence Auto CRF/preset decisions:

| Codec | Script class | Status |
|---|---|---|
| H.264 / AVC | Standard | Supported |
| H.265 / HEVC | Modern | Supported |
| VP9 | Modern | Supported |
| AV1 | Modern | Accepted as input, but usually skipped because it is already AV1 |
| MPEG-2 Video | Legacy | Supported |
| VC-1 | Legacy | Supported |
| MPEG-4 Part 2 | Legacy | Supported |
| Microsoft MPEG-4 v3 / MSMPEG4v3 | Legacy | Supported |
| H.263 | Legacy | Supported |
| RealVideo 3 / RV30 | Legacy | Supported |
| RealVideo 4 / RV40 | Legacy | Supported |

### Other FFmpeg-decodable codecs

The encoder does **not** reject an unknown video codec simply because it is absent from the table above. If `ffprobe` can identify it and FFmpeg can decode it, the script falls back to the normal **Standard** codec class.

That means these are technically compatible with the current code path when stored in one of the accepted containers:

| Codec | Current handling | Notes |
|---|---|---|
| VP8 | Standard fallback | Generally a good fit; commonly 8-bit 4:2:0 |
| MPEG-1 Video | Standard fallback | Expected to work in accepted MPEG/AVI-style containers |
| Windows Media Video variants | Standard fallback, except VC-1 which is explicit | Expected to work when FFmpeg can decode the stream |
| VVC / H.266 | Standard fallback | FFmpeg can decode VVC; best fit is 4:2:0 at 10-bit or less |
| ProRes | Standard fallback | Decodable, but see the chroma/alpha warning below |
| FFV1 | Standard fallback | Decodable, but archival/high-fidelity sources need special care |
| APV | Standard fallback | Decodable by current FFmpeg releases when containerized in an accepted format; raw `.apv` is not discovered |
| MJPEG, HuffYUV, DNxHD/DNxHR and many other FFmpeg-decodable codecs | Standard fallback | May work, but are not specifically tuned or claimed as fully validated |

In other words, **H.264 and H.265 are only the most common inputs, not hard requirements.**

## Important fidelity boundary: 4:2:0 output

The current encoder is optimized for playback-library compression, not preservation-master conversion.

The software lane always emits:

```text
yuv420p10le
```

The NVIDIA lane emits:

```text
yuv420p
```

for 8-bit SDR sources, or:

```text
p010le
```

for HDR / 10-bit-or-higher sources.

Because of that, a source can be **decodable** without being a safe "preserve everything" input.

Use extra care with:

- ProRes 422 / 4444 / 4444 XQ
- FFV1 archival masters
- APV professional/mastering sources
- DNxHR 4:2:2 / 4:4:4 profiles
- RGB video
- 12-bit or higher video
- video with an alpha channel

Those sources may be converted to 4:2:0 and/or 10-bit, and alpha is not preserved.

The quality-measurement path also normalizes comparison frames to `yuv420p10le`, so VMAF/XPSNR cannot fully protect source-only 4:2:2, 4:4:4, RGB, >10-bit, or alpha information from being reduced before the comparison.

For ordinary delivery/library media — H.264, HEVC, VP8/VP9, MPEG-family, VC-1, and similar 4:2:0 sources — this limitation is normally exactly what the AV1 output path is designed for.

## Audio and subtitle requirements

A file must contain at least one suitable audio stream. The current stream-selection logic rejects video-only inputs.

The encoder also does not preserve every stream indiscriminately:

- one main audio track is selected
- an optional fallback audio track may be retained
- selected English subtitle/SDH tracks may be retained
- audio and subtitle streams are stream-copied rather than re-encoded
- the output container is always Matroska (`.mkv`)

As a result, a source can fail at the muxing stage if a selected copied stream is not compatible with the Matroska output container.

## HDR / Dolby Vision notes

The normal HDR rules still apply regardless of the source codec:

- HDR10 static metadata is preserved where available
- HLG remains HLG
- Dolby Vision Profiles 7, 8 and compatible Profile 10 sources are converted according to the script's HDR plan
- Dolby Vision Profile 5 is refused by design
- HDR10+ preservation depends on the available FFmpeg/SVT-AV1/external-tool capabilities

## Practical definition of "supported"

For this project, a source is a good supported input when all of the following are true:

1. its file extension is in the accepted list
2. `ffprobe` can read the container and streams
3. FFmpeg can decode the selected video stream
4. the file contains a suitable audio stream
5. the selected copied audio/subtitle streams can be muxed into Matroska
6. reducing the source to AV1 4:2:0 at 8/10-bit does not discard information you need to preserve

That intentionally makes the compatibility claim broader than H.264/H.265 while still keeping it honest.
