# Release notes

[README](README.md) · [Operating](docs/operating.md)

These notes describe the current distributed script. They do not invent version numbers or imply that every repository commit corresponds to a packaged release.

## October 13, 2026 — Documentation

- Added the missing quality, HDR, tools, settings and operating guides linked by the README, plus this changelog.
- Documented the distinction between tool-menu inputs and supported top-level command-line parameters.
- Corrected README references to an absent source/build tree and clarified quality-fallback and Dolby Vision limits.

## October 12, 2026 — Unified encoders and estimation

- Unified CPU SVT-AV1, NVIDIA AV1 NVENC and Intel AV1 Quick Sync in one `Video2AV1Queue.ps1` / `.bat` pair.
- Added Intel hardware capability probing, source-specific QSV decoding with CPU fallback, ICQ quality-search integration, lane display/logging and scheduler handling for busy lanes.
- Corrected repeated console frames and Encoder Lanes headers accumulating in scrollback.
- Added interactive FFmpeg/ffprobe discovery, folder validation, duplicate-aware PATH updates and persistent Windows user PATH setup. Current behavior saves a validated selection automatically; an older optional Y/N implementation is not the current interface.
- Added a first-data-row Candidates CSV summary with summed sizes, durations, savings and encode-hours, plus rates recomputed from totals.
- Added sequential CPU/available-GPU calibration on the same source sample, locally validated saved profiles, Intel time estimates and history-first speed fallback.
- Skipped capability probes quietly when Windows inventory establishes that a GPU vendor is absent, while retaining diagnostics for detected hardware failures.

## Validation and boundaries

CPU/NVIDIA calibration and Candidates report integration were tested on the development desktop. Intel encoding, hardware decoding and concurrent CPU/Intel work were confirmed on a Core Ultra laptop. The later Intel calibration workflow still needs a hardware run there. These checks do not establish universal driver, GPU or HDR compatibility.

Speed calibration does not estimate savings, certify equal encoder quality or model mixed-lane wall-clock completion. Linux/macOS operation of the full application remains unverified. Check [HDR](docs/hdr.md), [Quality](docs/quality.md) and [Tools](docs/tools.md) before interpreting a successful job or benchmark as a broader guarantee.
