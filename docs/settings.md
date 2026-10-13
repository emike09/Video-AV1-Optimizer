# Settings reference

[README](../README.md) · [Quality](quality.md) · [HDR](hdr.md) · [Operating](operating.md)

Edit the user-configurable settings near the top of `Video2AV1Queue.ps1`, then restart the application. These are PowerShell assignments, not a JSON configuration file. Unquoted `Auto` invokes the script's small Auto helper; preserve the shipped syntax when editing. Per-job interactive/CLI overrides can take precedence over global settings.

The tables below reproduce every user setting in that block, its shipped value and its inline description. Internal caches/session variables are excluded. Defaults are a snapshot of the repository script, not measurements of your hardware. Range recommendations are guidance; see the script's resolver for exact validation.

## Important interactions

- `ReplaceOriginal` is true and `KeepBackupOriginal` false: successful queue encodes can replace sources without a backup.
- CPU CRF, NVIDIA CQ and Intel ICQ are different scales. Auto selects starting values; quality and size workflows may adjust them within their configured bounds.
- `CpuMaxParallel` and `SoftwarePinCores` interact. Benchmark before increasing CPU concurrency; more workers can compete for the same cores and memory.
- Intel hardware decoding can fall back to CPU under `QsvDecode = 'Auto'`; forcing `qsv` requires a successful pipeline. `QsvAdapter` selects one adapter, not a multi-Intel pool.
- Quality target failure defaults to the starting Auto CRF, not refusal. Set `QualityFloorUnreachableAction` to `Skip` if refusal is required.
- Preflight estimates and quality measurements are separate checks. A quality result does not establish space savings.
- HDR preservation depends on detected capabilities, metadata plans and optional tools; a configuration request cannot create unsupported encoder behavior.
- Calibration is invalidated by changes to its tracked settings, hardware/driver or FFmpeg binary. Rerun after tuning and keep encode history representative of the current machine.

## Defaults

| Setting | Default | Description |
|---|---|---|
| `CRF` | `Auto` | Software quality target. 0-63, recommend 10-28, or Auto. Lower = better quality / larger files. |
| `Preset` | `Auto` | Software speed/compression setting. 0-13, recommend 3-6, or Auto. Lower = slower / smaller files. |
| `AutoCRFOffset` | `Auto` | Auto mode CRF adjustment. Integer or Auto, recommend -2 to +2. Positive = smaller files, negative = higher quality. |
| `FilmGrain` | `Auto` | AV1 film grain synthesis strength. 0-50, recommend 0-16, or Auto. Higher = better grain retention / smaller files on grainy sources. |
| `EncoderPreference` | `'Auto'` | Preferred encoder path. Auto \| CPU \| Nvidia \| Intel. Auto chooses per file. |
| `CpuMaxParallel` | `1` | Simultaneous CPU (SVT-AV1) encodes. 1-4. Read the CPU parallelism note below first. |
| `SoftwarePinCores` | `0` | Cores per CPU encode, via svtav1-params pin. 0 = no pinning. Only meaningful when $CpuMaxParallel > 1. |
| `SoftwareEncodePriority` | `'BelowNormal'` | Priority for CPU software encodes. Idle \| BelowNormal \| Normal \| AboveNormal. |
| `HardwareEncodePriority` | `'Normal'` | Priority for GPU/NVENC encodes. Idle \| BelowNormal \| Normal \| AboveNormal. |
| `ScriptProcessPriority` | `'Normal'` | Priority for this PowerShell controller process. Idle \| BelowNormal \| Normal \| AboveNormal. |
| `ApplyProcessPriority` | `$true` | Apply the priority settings above. $true recommended. |
| `QsvMaxParallel` | `1` | Intel AV1 workers. Start with 1; increase after hardware testing. |
| `QsvICQ` | `Auto` | Intel ICQ 1-51 or Auto; lower = higher quality. Not NVENC CQ or SVT CRF. |
| `QsvDecode` | `'Auto'` | Auto probes each file; cpu disables HW decode; qsv requires HW decode. |
| `QsvPreset` | `'slow'` | veryfast \| faster \| fast \| medium \| slow \| slower \| veryslow |
| `QsvAdapter` | `'Auto'` | Auto selects Intel by vendor on Windows; or a DXGI adapter index. |
| `NvencMaxParallel` | `Auto` | Max simultaneous NVENC workers. 1-8 or Auto. Auto uses GPU model/engine lookup. |
| `NvencCQ` | `Auto` | NVENC quality target. 0-51, recommend 18-28, or Auto. Lower = better quality / larger files. |
| `NvencPreset` | `Auto` | NVENC speed/compression setting. p1-p7 or Auto. Higher preset = slower / better compression. |
| `NvencDecode` | `Auto` | Decode path for NVENC jobs. Auto \| cpu \| cuda. Auto prefers the safest path. |
| `NvencTune` | `'auto'` | NVENC tuning bias. auto \| hq \| ll \| ull \| $null. hq = quality, ll/ull = latency-focused. |
| `NvencAllowSplitFrame` | `$false` | Allow split-frame NVENC if supported. $false recommended; $true is advanced/testing only. |
| `EnablePreflightEstimate` | `$true` | Run short sample encodes before the full job to estimate final size. |
| `PreflightSampleCount` | `4` | Number of preflight samples. 1-12, recommend 3-6. More = slower but more accurate. |
| `PreflightSampleDurationSec` | `20` | Seconds per preflight sample. 5-120, recommend 15-30. Longer = slower but more accurate. |
| `PreflightWarnIfEstimatedPctOfSource` | `90` | Warn if projected output reaches this % of source size. 1-1000, recommend 90-100. |
| `PreflightAbortIfEstimatedPctOfSource` | `95` | Skip encode if projected output reaches this % of source size. 1-1000, recommend 95-110. |
| `EnablePreflightAutoTune` | `$true` | Let preflight adjust Auto quality settings before the full encode starts. |
| `EnableSecondPreflightPass` | `$true` | Run a second preflight after major auto-tuning changes to validate the new settings.  |
| `PreflightAutoTuneQuality` | `'High'` | Auto-tune quality profile. Low \| Medium \| High. High = more quality-preserving targets, larger file. |
| `PreflightTinyOutputPctThreshold` | `35` | Flag outputs smaller than this % of source size as suspicious. 1-100, recommend 25-50. |
| `PreflightTinyOutputAbsoluteGiBThreshold` | `1.0` | Also flag projected outputs below this size in GiB. 0.1-100.0, recommend 0.5-2.0. |
| `EnableLiveSizeEstimate` | `$true` | Show estimated final size and savings while encoding. |
| `LiveEstimateStartPercent` | `3` | Start showing the estimate after this % of progress. 1-100, recommend 3-15. |
| `LiveEstimateSmoothingFactor` | `0.30` | Smooth live estimate fluctuations. 0.01-1.00, recommend 0.20-0.40. Lower = steadier, higher = more responsive. |
| `PreflightMaxFractionOfSourceRate` | `0.65` | Cap the target at this fraction of the SOURCE's own GiB/hr. 0.1-1.0, or $null to disable. See the note below. |
| `PreflightAutoTuneCustomTargetGiBPerHour` | `$null` | Override target output rate in GiB/hr. Decimal or $null. Recommend 1.0-20.0 depending on source. |
| `PreflightAutoTuneCustomUpperGiBPerHour` | `$null` | Override upper tuning threshold in GiB/hr. Decimal or $null. Recommend target + 1 to +4. |
| `PreflightAutoTuneCustomLowerGiBPerHour` | `$null` | Override lower tuning threshold in GiB/hr. Decimal or $null. Recommend target - 1 to -4. |
| `EnableQualityTargeting` | `$true` | Measure sample encodes and search for the highest CRF that still looks the same. $true recommended. |
| `QualityMetric` | `Auto` | Metric to use. Auto \| VMAF \| XPSNR \| Off. Auto = VMAF for SDR, XPSNR for HDR. |
| `QualityMode` | `Auto` | Auto \| Absolute \| Anchor. Auto = Absolute for VMAF, Anchor for XPSNR. |
| `QualitySampleCount` | `2` | Sample positions measured per CRF probe. 1-6, recommend 2-3. More = slower but steadier. |
| `QualitySampleDurationSec` | `15` | Seconds per quality sample. 5-60, recommend 10-20. |
| `QualityMaxSearchPasses` | `3` | CRF probes after the first. 1-6, recommend 2-4. Each pass is one more round of sample encodes. |
| `QualityMaxCrfStep` | `6` | Largest CRF jump the search may take in one pass. 1-16. |
| `QualityMaxCrfAboveAuto` | `12` | Never raise CRF more than this above what Auto picked. 0-30. |
| `QualityMaxCrfBelowAuto` | `8` | Never lower CRF more than this below what Auto picked. 0-30. |
| `QualityCrfCeiling` | `Auto` | Absolute CRF cap, or Auto to use Auto CRF + $QualityMaxCrfAboveAuto. |
| `QualityFloorUnreachableAction` | `'AutoCrf'` | When no CRF in range meets the quality target. AutoCrf \| Skip. See the quality notes. |
| `QualityReportSecondMetric` | `$true` | Also report the other metric at the chosen CRF, for the log. Costs one extra measurement per file. |
| `QualityVmafTarget` | `95.0` | Minimum VMAF for the output. 88-98, recommend 93-96. Higher = better quality / larger files. |
| `QualityVmafAnchorDrop` | `0.5` | VMAF points allowed below the anchor, when $QualityMode = Anchor. 0.2-3.0. |
| `QualityVmafConvergenceBand` | `0.5` | Stop searching once VMAF is within this much above target. 0.1-2.0. |
| `QualityVmafThreads` | `8` | Threads for the VMAF calculation. 1-16. |
| `QualityAnchorCRF` | `22` | REFERENCE quality level, not a ceiling. The search finds the cheapest CRF that looks this good. |
| `QualityXpsnrAnchorDropDb` | `0.25` | dB allowed below the anchor. 0.1-1.0, recommend 0.2-0.4. Larger = smaller files. |
| `QualityXpsnrConvergenceBand` | `0.15` | Stop searching once within this many dB above threshold. 0.05-0.50. |
| `QualityXpsnrAbsoluteTarget` | `42.0` | Only used if $QualityMode = Absolute. See the warning in the quality notes. |
| `QualityXpsnrAggregation` | `'Weighted'` | How to combine the y/u/v scores. Weighted \| Min \| Luma. |
| `SoftwareTune` | `Auto` | SVT-AV1 tune. 0 = VQ, 1 = PSNR, 2 = SSIM, or Auto (2). Library default is 1. |
| `SoftwareKeyintSeconds` | `10` | Keyframe interval in seconds. 0 = leave at the library default (161 frames). 5-15 recommended. |
| `SoftwareVarianceBoost` | `Auto` | Protect flat and dark areas at higher CRF. Auto \| $true \| $false. Auto = on. |
| `SoftwareVarianceBoostStrength` | `2` | Variance boost strength. 1-4, recommend 2. |
| `SoftwareFilmGrainDenoise` | `$false` | Let film-grain synthesis replace real grain instead of adding to it. See the film grain note. |
| `SoftwareSceneChangeDetection` | `$false` | svtav1-params scd=1. Off by default; measure it before enabling. |
| `SoftwareEnableOverlays` | `$false` | svtav1-params enable-overlays=1. Off by default; measure it before enabling. |
| `SoftwareQpScaleCompressStrength` | `$null` | svtav1-params qp-scale-compress-strength, 0-3, or $null to leave at the library default. |
| `PreserveHdrStaticMetadata` | `$true` | Carry mastering-display colour volume + MaxCLL/MaxFALL into the output. $true strongly recommended. |
| `PreserveHDR10Plus` | `'Auto'` | HDR10+ dynamic metadata. Auto \| $true \| $false. Auto preserves it when the required tools are present. |
| `PreserveHLG` | `$true` | Keep HLG sources tagged HLG. $false reverts to the old behaviour of mislabelling them as PQ. |
| `HdrToolsDir` | `$null` | Folder containing hdr10plus_tool / dovi_tool. $null = look next to this script, then on PATH. |
| `ClampMaxCllToMasteringPeak` | `$false` | Fix sources that declare MaxCLL brighter than their own mastering peak. See the MaxCLL note. |
| `ClampMaxCllMinPeakNits` | `400` | Only clamp when the mastering peak is at least this bright. 100-4000. |
| `ClampMaxCllMinOvershoot` | `1.5` | Only clamp when MaxCLL exceeds the peak by at least this factor. 1.1-10.0. |
| `DolbyVisionMode` | `'HDR10'` | Dolby Vision sources. Skip \| HDR10 \| Passthrough. HDR10 converts the base layer; Skip is the old behaviour. |
| `SkipDolbyVisionSources` | `$false` | Legacy switch, kept for compatibility. $true forces $DolbyVisionMode to 'Skip'. |
| `KeepBackupOriginal` | `$false` | Keep a backup copy of the original after a successful encode. |
| `ReplaceOriginal` | `$true` | Replace the source file with the finished AV1 output after success. |
| `KeepEnglishSDH` | `$false` | Keep an English SDH subtitle track in addition to the main subtitle. |
| `SubtitleCompatibilityMode` | `'Convert'` | Subtitles Matroska cannot carry (mov_text from MP4, etc). Convert \| Drop \| Skip. |
| `KeepEnglishFallbackAudio` | `$true` | Keep a secondary lossy English audio track when the main track is lossless. |

## Tool options

Candidates thresholds, loudness targets and benchmark sample lengths are parameters inside their respective tool functions, not extra variables in the global settings block. The menus expose a subset of these options. See [Tools](tools.md); do not pass internal function parameters as top-level script switches.
