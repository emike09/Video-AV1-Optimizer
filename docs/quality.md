# Quality targeting

[README](../README.md) · [Settings](settings.md) · [HDR](hdr.md)

## What is measured

The queue starts with per-file Auto settings, encodes short samples, compares decoded output with the source, and searches within a bounded quality range. A separate preflight estimates output size and can still refuse a conversion that will not save enough space. A passing metric is evidence about sampled frames, not a guarantee of perceptual transparency throughout the film.

CPU uses SVT-AV1 CRF, NVIDIA uses CQ and Intel uses ICQ. Their numbers are not interchangeable. Hardware settings are mapped from the starting software quality heuristic; QSV sample probes vary the actual ICQ. A manually pinned Intel ICQ prevents automatic adjustment of that value.

## Metric and mode

| Configuration | Behavior |
|---|---|
| `QualityMetric = Auto` | VMAF for SDR; XPSNR for HDR. |
| `QualityMode = Auto` | Absolute target for VMAF; anchor-relative target for XPSNR. |
| `Absolute` | Compare against a fixed threshold. Defaults: VMAF 95, XPSNR 42 dB when explicitly used in absolute mode. |
| `Anchor` | Measure a reference encode of the same content, then allow a configured score drop from that reference. Default reference CRF: 22. |
| Metric `Off`, or targeting disabled | No quality-search protection; the remaining Auto/preflight checks are still separate decisions. |

The default permitted anchor drops are 0.5 VMAF points and 0.25 dB XPSNR. XPSNR defaults to weighted aggregation; `Min` and `Luma` are alternatives. Absolute XPSNR scores depend strongly on content, so the default uses an anchor rather than treating one dB threshold as universal.

## Search bounds and cost

Defaults are two 15-second sample positions, up to three additional search passes, a maximum six-CRF step, and a range from eight CRF below the initial Auto CRF to twelve above it. The default lower allowance is eight (`QualityMaxCrfBelowAuto`); the ceiling can be pinned separately. Reporting the second metric adds a measurement at the selected setting.

More samples improve coverage but cost decoding, encoding and metric work. Even GPU encoding can leave substantial CPU work in metric calculation. See the [complete defaults](settings.md) before changing bounds.

## When no setting meets the target

`QualityFloorUnreachableAction = 'AutoCrf'` is the shipped policy: retain the starting Auto CRF and explain that the target was not reached. `Skip` requests refusal instead. An encode that proceeds under this fallback has **not passed the requested quality threshold**.

Film grain can dominate full-reference metrics. Anchor mode compares against a reference of the same content and can make that comparison more useful, but it does not prove grain or detail preservation. Film-grain synthesis is a software-encoder feature, not an equivalent hardware setting. Inspect motion, dark areas, faces and fine textures on representative outputs.

## Practical workflow

1. Use **Tools → Quality** on representative SDR, HDR and grainy sources. This is quality exploration, distinct from speed calibration under Benchmark.
2. Test copies with `ReplaceOriginal = $false`.
3. Compare output visually and inspect quality/preflight fields in the encode log.
4. Adjust the target, anchor drop or fallback policy deliberately; avoid interpreting a smaller file alone as success.

Capability failures and unavailable metrics are reported by the script. Run Doctor to inspect the installed FFmpeg filters and encoder support. Settings and source details are recorded in the log so that a fallback can be distinguished from a measured pass.
