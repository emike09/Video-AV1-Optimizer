# HDR and Dolby Vision

[README](../README.md) · [Quality targeting](quality.md) · [Settings](settings.md)

## Preservation depends on the complete pipeline

The script inspects source bit depth, transfer function, colour primaries, matrix and HDR side data, builds an HDR plan, and validates output. A 10-bit AV1 stream alone does not prove HDR metadata preservation. CPU, NVIDIA and Intel have different capabilities; FFmpeg options and external tools are checked before choosing a route.

| Source | Intended handling |
|---|---|
| SDR | Preserve SDR signalling; output commonly uses 10-bit AV1. This is not an SDR-to-HDR conversion. |
| HDR10 / PQ | Preserve colour signalling and, when supported, mastering-display metadata and MaxCLL/MaxFALL. |
| HLG | Preserve HLG signalling when `PreserveHLG` is enabled (the default). |
| HDR10+ | Preserve dynamic metadata where extraction, encoding and reinjection capabilities allow. |
| Dolby Vision | Apply the configured DV source plan; the default seeks an HDR10-compatible base layer rather than promising AV1 Dolby Vision output. |

## Dolby Vision policy

`DolbyVisionMode` defaults to `HDR10`; `Skip` and `Passthrough` are configuration alternatives. The default workflow handles compatible base layers in supported profiles, including profiles 7, 8 and 10. Profile 5 is refused because it has no compatible HDR10 base layer for this conversion path. Source compatibility checks still apply: a profile number alone is not proof that every stream is convertible.

`Passthrough` does not produce Dolby Vision AV1 in this implementation: it explicitly falls back to the compatible base-layer conversion and reports that decision, or skips an incompatible source. Consult the reported plan and validation results. The legacy `SkipDolbyVisionSources` setting also exists. Speed calibration rejects Dolby Vision sources because it does not run the complete DV metadata workflow.

## Static metadata and MaxCLL

`PreserveHdrStaticMetadata = $true` requests mastering-display colour volume and content-light metadata preservation. If the required metadata cannot be carried through the selected hardware route, Auto may select CPU; an explicitly requested unsupported route can be skipped with a reason.

The optional MaxCLL sanity clamp is **off** by default. Its defaults require a mastering peak of at least 400 nits and an overshoot factor of at least 1.5 before clamping. This is a corrective policy for questionable metadata, not tone mapping; enable it only when that behavior is intended.

## Optional external tools

- `hdr10plus_tool.exe`: HDR10+ extraction/injection support.
- `dovi_tool.exe`: discovered and its path recorded, but not invoked by the current pipeline. Dolby Vision conversion uses FFmpeg bitstream filters such as `dovi_split` and `dovi_rpu`; installing this binary cannot replace missing FFmpeg capabilities.

`HdrToolsDir` can name their directory. Otherwise discovery includes the script directory and PATH. The project README describes release packaging; check the actual archive you downloaded rather than assuming these tools are installed globally. They are separate projects with their own licenses.

Intel uses the existing HDR plan, including the post-injection route for HDR10+ instead of SVT-specific inline options. Hardware-decoding support does not by itself establish dynamic-metadata support.

## Checking an output

Run **Tools → Doctor** to inspect capabilities and **Tools → Verify** with representative samples to exercise preservation. Confirm colour tags, bit depth, mastering/content-light metadata where present, and any dynamic metadata you require. Also test playback on your actual display/client. Synthetic PQ or HLG signalling tests cannot establish visual fidelity or support for every real HDR title.

Keep originals until the files and playback path have been checked. The queue's default replacement policy is described in [Operating](operating.md).
