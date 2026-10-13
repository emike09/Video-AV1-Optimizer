# 🎬 Video AV1 Optimizer

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](#-license)
[![PowerShell 7+](https://img.shields.io/badge/PowerShell-7%2B-5391FE.svg?logo=powershell&logoColor=white)](https://github.com/PowerShell/PowerShell)
[![FFmpeg 9+](https://img.shields.io/badge/FFmpeg-9.0.1%2B-007808.svg)](https://www.gyan.dev/ffmpeg/builds/)
[![Platform: Windows](https://img.shields.io/badge/Platform-Windows-0078D6.svg)](#requirements)

**Convert common video files to AV1 and make the files smaller without losing anything you can see.**  
[View supported containers and codecs →](SUPPORTED-FORMATS.md)

That is the whole goal, and the thing that makes this different from other tools - Most tools answer *"how big will this be?"* with a bitrate heuristic. This one encodes short samples, **measures** them against the source with VMAF or XPSNR, and searches for the highest CRF that still looks the same — then hands that to the size projection, which keeps its veto.

One PowerShell file and a launcher. Drag files on to Video2AV1Queue.bat; or double-click it for an interactive menu.

---

## What it does

- **AV1 via CPU SVT-AV1, NVIDIA NVENC or Intel Quick Sync**, chosen per file, with concurrent workers
- **Measured quality targeting** — the CRF is searched for and verified, not guessed
- **Honest about the metric's limits** — where an absolute VMAF target cannot discriminate (grainy film transfers), the file is converted at the Auto CRF rather than refused, and the run says so
- **HDR10 static metadata survives** the re-encode — mastering display colour volume, MaxCLL/MaxFALL
- **Dolby Vision converted, not skipped** — Profiles 7, 8 and 10 become correct HDR10; Profile 5 is refused by design
- **HDR10+ preserved** where the toolchain supports it; **HLG stays HLG**
- **Per-file Auto analysis** for `CRF`, `Preset`, `FilmGrain` and `AutoCRFOffset`
- **Persistent queue** with live pause / resume / stop controls and crash recovery
- **Seven built-in tools** — diagnose the toolchain, rank conversion candidates, calibrate thresholds, normalise loudness, census a library, calibrate CPU/GPU speed or benchmark CPU threading, verify output
- **CSV + session logging**, and a summary at the end of every run

---

## Requirements

| | |
|---|---|
| OS | Windows |
| Shell | PowerShell 7+ |
| Encoder | full **FFmpeg 9.0.1+** build (`ffmpeg.exe`, `ffprobe.exe`) with `libvmaf` and the `xpsnr` filter |
| NVENC *(optional)* | NVIDIA GPU with AV1 encode, current driver, `nvidia-smi` available |
| Quick Sync *(optional)* | Intel GPU/iGPU with **AV1 hardware encoding**, current Intel graphics driver, and FFmpeg with `av1_qsv` / oneVPL support |

Most full builds — gyan.dev "full", BtbN "gpl" — have everything needed. FFmpeg 8.x runs with **reduced capability** and says so; 6.x / 7.x and "essentials" builds are not supported. The **Doctor** tool tells you exactly what your build can do.

The release bundles hdr10plus_tool.exe for HDR10+ preservation and dovi_tool.exe. The current script discovers dovi_tool but does not invoke it; Dolby Vision conversion relies on FFmpeg bitstream filters. See [HDR documentation](docs/hdr.md#optional-external-tools).

---

## Install

Download **`Video2AV1Queue.ps1`** and **`Video2AV1Queue.bat`** from the [latest release](../../releases/latest) into one folder. That is the whole install.

```text
Video2AV1Queue.bat      drag files onto this, or double-click it
Video2AV1Queue.ps1      everything: encoder, tools, settings
ffmpeg.exe              (or on PATH)
ffprobe.exe
dovi_tool.exe
hdr10plus_tool.exe
```

| | |
|---|---|
| **Drag files or folders onto the .bat** | They are queued for encoding; missing FFmpeg tools trigger first-run setup. |
| **Double-click the .bat** | A menu opens. |

```text
  [1] Auto           Encode files. Everything decided for you.
  [2] Interactive    Encode files, choosing quality and encoder yourself.
  [3] Tools          Check, scan, measure, calibrate, verify.
```

### FFmpeg discovery and first-run setup

Keep `Video2AV1Queue.bat` and `Video2AV1Queue.ps1` together. No separate Intel launcher is needed.

The script looks for **both `ffmpeg` and `ffprobe`**, first beside the script and then on PATH. If either is missing, it prompts for the FFmpeg installation directory. It does not download or install FFmpeg.

- Enter a folder such as `C:\ffmpeg` or `C:\ffmpeg\bin`, or paste the full path to `ffmpeg.exe`. Quoted paths and environment variables are accepted.
- The selected folder and its `bin` subfolder are checked. Both executables must exist in one of them before setup continues.
- An invalid location prompts again. Blank input cancels without changing PATH.
- A valid folder is added to the current process PATH and, on Windows, your **user PATH**. Existing entries are preserved and duplicate folder entries are avoided. Administrator rights are not needed; system PATH is not changed.
- Already-open terminals may need to be reopened to inherit the persistent change. If saving user PATH fails, the current session can still use the selected tools and a warning explains the failure.

The discovery helper updates only session PATH on non-Windows systems. This does **not** establish Linux/macOS support for the full queue or the Windows `.bat` launcher.

Settings live in the `User-configurable settings` block near the top of the `.ps1`.

---

## ✅ First run — do this before touching your library

`$ReplaceOriginal = $true` by default: **the script replaces your source files in place.** Four steps, in order.

**1. Check the toolchain.**

```powershell
pwsh -File .\Video2AV1Queue.ps1 -Mode Doctor
```

Read-only. It probes the same capabilities the encoder does and prints the answers, then lets you select a test file through the Doctor menu to inspect its planned handling. `TestFile` is an internal tool parameter, not a top-level script switch. Because the script degrades gracefully rather than erroring, this is how you tell "working" apart from "working at reduced capability".

**2. Calibrate the quality thresholds against your own content.**

```powershell
pwsh -File .\Video2AV1Queue.ps1 -Mode Quality
```

Run it on two or three genuinely different files — a clean modern UHD HDR title, something grainy, something animated. **The shipped defaults are starting points, not settled constants**, and [Quality targeting](docs/quality.md) shows why that matters more than it sounds.

**3. Encode a handful of files with `$ReplaceOriginal = $false`.** Check the outputs. Then turn it back on.

**4. Find out what is actually worth converting.**

```powershell
pwsh -File .\Video2AV1Queue.ps1 -Mode Candidates
```

Container headers only — roughly 50 ms per file on local storage, so a few thousand files take minutes rather than hours.

---

## CPU, NVIDIA and Intel lanes

The unified script supports CPU SVT-AV1, NVIDIA AV1 NVENC and Intel AV1 Quick Sync. In Auto mode, source suitability and free lane capacity determine routing: difficult content can stay on CPU while suitable files use a GPU. NVIDIA retains precedence when both GPU vendors are available; Intel can fill a slot when NVIDIA is absent or busy. A busy lane waits without dropping the queued item.

Use **Interactive → Intel** to request Intel for particular jobs, or set `$EncoderPreference = 'Intel'` in the configuration. Keep Auto selected to allow CPU and GPU jobs to run together.

```powershell
$QsvMaxParallel = 1
$QsvICQ = Auto
$QsvDecode = 'Auto'
$QsvPreset = 'slow'
$QsvAdapter = 'Auto'
```

| Intel setting | Meaning |
|---|---|
| `QsvMaxParallel` | Concurrent Intel workers; starts at one. |
| `QsvICQ` | Auto or 1–51; lower means higher quality/larger files. ICQ is not numerically equivalent to SVT CRF or NVIDIA CQ. |
| `QsvDecode` | `Auto` probes hardware decode for each source and falls back to CPU decode with a reason; `cpu` disables hardware decoding; `qsv` requires it. |
| `QsvPreset` | `veryfast`, `faster`, `fast`, `medium`, `slow`, `slower`, or `veryslow`. |
| `QsvAdapter` | `Auto` selects Intel by vendor on Windows; a numeric DXGI adapter index selects a specific adapter. Only one selected Intel adapter is scheduled. |

Intel capability is verified with a real 10-bit AV1 test encode. Merely listing `av1_qsv` in FFmpeg is not sufficient. Hardware decoding uses the selected stream's QSV decoder, hardware conversion to 10-bit surfaces, and QSV AV1 encoding. Unsupported decoding can fall back to CPU without disabling Intel encoding.

Quality-search samples vary the Intel ICQ and final encoding uses the selected setting; a manually pinned ICQ keeps it fixed. Existing audio/subtitle selection, chapters, quality/size checks, validation and replacement policy still apply. HDR support remains dependent on the metadata workflow and available FFmpeg features; Intel does not imply blanket support for every HDR or Dolby Vision source.

On Windows, a successful GPU inventory that shows no Intel or NVIDIA adapter skips that vendor's probe quietly. If inventory is inconclusive, normal capability probing still runs. Errors from a detected GPU that fails its capability test remain visible.

### Live console display

The Encoder Lanes dashboard redraws in place instead of appending repeated box headers and worker panels to scrollback. Rendering fits the available terminal dimensions; shorter windows can omit secondary detail. CPU, NVIDIA and Intel workers share the display. Restart the script after updating to load these changes.

## Candidates reports and totals

**Tools → Candidates** scans container headers, filters eligible conversion candidates and exports the full ranked CSV plus a shortlist of paths. It asks which lane to use for time estimates: **CPU, Nvidia or Intel** (Enter defaults to CPU). This selection models time; it does not force queued files onto that lane or prove that every candidate is suitable for GPU encoding.

The first CSV data row, directly beneath the column headers, is `TOTAL (N candidates)`. It covers **all exported candidates**, not just the displayed shortlist:

- `SourceGiB`, `DurationMin`, `ExpectedSavedGiB` and `EncodeHours` are summed.
- `ExpectedSavingPct` is total estimated savings divided by total source size.
- `PayoffGiBPerHour` is total estimated savings divided by total encode-hours.
- Non-additive fields such as codec, resolution and CRF remain blank. The summary's `Path` is blank; the separate queue shortlist contains only file paths.

These are static CSV values calculated when the report is generated, not spreadsheet formulas that update when you filter or delete rows.

## Calibrating Candidates time estimates

Choose **Tools → Benchmark → 1 — Calibrate CPU + available GPUs**, then select exactly one representative video file. The existing **CPU threading comparison** remains option 2.

Calibration encodes the same 20-second source segment twice on CPU and on each available AV1-capable NVIDIA/Intel encoder. Lanes run sequentially, using the current settings and the existing sample argument builders, including the configured decoding path. Run it with other encodes idle. The source file is not replaced; calibration outputs are temporary.

Use representative SDR or HDR10 material. Dolby Vision calibration is rejected because it requires the full metadata workflow. This is speed calibration, not a quality-equivalence comparison between CRF, CQ and ICQ.

Results are saved to `.queue/calibration.json`; rerunning replaces the profile and saves the previous one as `.bak`. Failed or unavailable lanes are not recorded. Run calibration separately on each machine. Hardware/driver, tracked encoder settings or FFmpeg binary changes invalidate the profile and require recalibration.

Candidates resolves speed in this order:

1. **Learned history:** the median of at least three valid successful encodes matching the resolution tier and lane in `.queue/encode_log.csv`.
2. **Calibration:** the selected lane's measured speed when matching history is insufficient.
3. **Built-in estimate:** when neither usable history nor calibration is available.

The CSV includes `EstimateLane` and labels `SpeedSource` as `history`, `calibration`, `calibration-pixel-scaled` or `estimate`, with a sample count where applicable. Different resolutions/frame rates are extrapolated using pixel throughput and explicitly labelled as scaled estimates. A single calibration file does not measure every codec, content type, grain level or Auto preset.

**What the totals mean:** `EncodeHours` is summed sequential work for one worker on the selected lane. It is not the wall-clock completion time of a mixed Auto CPU/GPU queue. Calibration measures decoding and video encoding but excludes quality-search passes, audio copying, HDR metadata processing and final remuxing. Simultaneous jobs, thermal/power limits and different content/settings can change throughput.

Calibration improves **time estimates only**. Space-savings estimates still use matching successful history or existing heuristics; benchmark output size is not used as a library-wide savings prediction. Keep learning from completed encodes for better estimates.

## Changes — October 12, 2026

- Unified CPU, NVIDIA and Intel support into `Video2AV1Queue.bat` / `.ps1`; retired the separate Intel production filenames.
- Added Intel AV1 encoding, per-source QSV hardware decoding with CPU fallback, quality-search integration, lane display/logging and scheduler fixes.
- Fixed repeated console frames and Encoder Lanes headers accumulating in scrollback.
- Added FFmpeg/ffprobe folder discovery and persistent Windows user PATH setup.
- Added Candidates CSV totals and weighted overall savings/rate calculations.
- Added CPU/GPU calibration, local profile validation, Intel time estimates and history-first calibration fallback.
- Silenced absent-GPU capability errors while retaining diagnostics for actual failures.

CPU/NVIDIA calibration and Candidates CSV integration were tested on the desktop. Intel encoding, hardware decoding and concurrent CPU/Intel work were confirmed by the user on the Core Ultra laptop; the new Intel calibration workflow still needs a hardware run there. This does not establish support for every GPU, driver or HDR format.

---

## The idea in one table

Two measurements that shaped how this works, both taken with identical encoder settings:

| Source | CRF 22 | CRF 30 | What a fixed threshold does |
|---|---|---|---|
| clean 1080p | VMAF 98.4 | — | passes easily |
| heavily grained 1080p | VMAF 75.0 | VMAF 73.8, **6.8× smaller** | refuses it at every CRF |

VMAF moves under two points across sixteen CRF steps on grain, because the score is dominated by noise no bitrate reproduces exactly. A fixed "VMAF ≥ 95" gate therefore rejects every film-grain Blu-ray remux — the files most worth converting.

So the script does not treat "no CRF was transparent" as "this file cannot be converted". It keeps the CRF the Auto ladder chose, says so on one line, and carries on. If your library is mostly grainy film transfers, `$QualityMode = 'Anchor'` compares every CRF against a reference encode of the *same* content instead, which makes the comparison relative to that content; it does not guarantee preservation of grain or detail.

**[→ Full explanation, both metrics, and all the settings](docs/quality.md)**

---

## Documentation

| | |
|---|---|
| **[Supported formats](SUPPORTED-FORMATS.md)** | Accepted containers, explicitly tuned codecs, other FFmpeg-decodable inputs, and fidelity limits |
| **[Quality targeting](docs/quality.md)** | How the CRF search works, VMAF vs XPSNR, why XPSNR is anchored, what to do about grain, all quality settings |
| **[HDR and Dolby Vision](docs/hdr.md)** | Static HDR10 metadata, HDR10+, HLG, the MaxCLL sanity check, DV profile handling |
| **[The seven tools](docs/tools.md)** | What each tool is for, plus deep dives on finding candidates and loudness normalisation |
| **[Settings reference](docs/settings.md)** | Every setting, Auto mode, film grain, SVT-AV1 efficiency, preflight, NVENC |
| **[Running it day to day](docs/operating.md)** | Drag-drop / interactive / CLI, queue behaviour, live controls, output naming, log schema |
| **[Release notes](RELEASE-NOTES.md)** | Changelog |

---

## Building it yourself

This repository ships the standalone `Video2AV1Queue.ps1` and launcher. It does not currently include a `src` tree or `Build-Single.ps1`. Edit the distributed script for changes to this checkout; validate PowerShell syntax and test copies before using a modified encoder on your library.

---

## 🧠 Philosophy

- visual quality over maximum compression
- **measure rather than assume** — capabilities are probed functionally, not read out of help text; quality is measured, not inferred from bitrate
- when a capability is missing, say so and degrade honestly rather than silently producing worse output
- when a *measurement* cannot decide, say that too, and fall back to something defensible — never refuse work on the strength of a number that carries no information
- explainable heuristics over black-box tuning
- safe queue behaviour over risky shortcuts
- refuse to produce a file that claims to be something it is not

---

## 📜 License

MIT. Optional external tools (`hdr10plus_tool`, `dovi_tool`) are separate MIT-licensed projects by [quietvoid](https://github.com/quietvoid) and retain their own licenses; the release packaging described above may include their executables.

## 🙌 Contributing

Pull requests welcome. Ideas: an AMD lane, Linux support, broader quality-threshold calibration data across content types.

## ⭐ Support

If this helped you, give it a star.
