# The seven tools

[README](../README.md) · [Settings](settings.md) · [Operating](operating.md)

Open the launcher and choose **Tools**. `pwsh -File .\Video2AV1Queue.ps1 -Mode Tools` opens the same menu. Mode-specific parameters shown inside function definitions are not automatically top-level command-line switches; use the menus for tool options.

| Tool | Purpose and effects |
|---|---|
| Doctor | Probe FFmpeg/encoder/filter capabilities and inspect a selected test file. Does not convert your library; functional probes can create temporary test data. |
| Candidates | Read container headers, score eligible files, write CSV and a shortlist. Offers to queue the shortlist afterward. |
| Loudness | Scan sampled/full audio loudness, or explicitly apply normalization to output files. |
| Quality | Explore sample quality metrics and thresholds for selected content. |
| Library Scan | Generate a library census/report, including source format and HDR information. |
| Benchmark | Calibrate CPU/GPU speeds for Candidates, or compare CPU threading configurations. |
| Verify | Copy samples into a sandbox, encode them and check metadata/stream preservation. Requires an idle queue. |

## Candidates

The scan uses metadata rather than full video decoding. It excludes already-AV1 and Auto-skipped efficient sources, files below size/savings thresholds, incompatible DV Profile 5, unusable streams and failed probes. Defaults include a 0.2 GiB minimum source size, 0.25 GiB minimum projected saving, and a displayed/queued shortlist of forty. The CSV includes every qualifying candidate.

Ranking defaults to estimated GiB saved per encode-hour. Internal tool options also support savings, source size and BPP rankings. Audio estimates use stream metadata when available, otherwise a codec/channel model; copied audio limits the space a video conversion can recover. Eligibility and rankings remain predictions: queue preflight and quality checks can reject a listed file.

Choose CPU, Nvidia or Intel for the time model (CPU is the menu default). `EstimateLane` records that choice; it does not force subsequent Auto queue routing.

### Learning and calibration

Speed estimates use the median of at least three successful history entries matching resolution tier and lane. Otherwise, a valid local calibration for that lane is used, followed by built-in estimates if necessary. Savings history is grouped by resolution tier, source codec class and BPP tier, also requiring three usable records. It is not learned from benchmark output size.

`SpeedSource` distinguishes `history`, `calibration`, `calibration-pixel-scaled` and `estimate`. Different dimensions/frame rates use rough pixel-throughput scaling. A single sample cannot account for all content, presets, decode paths or thermal constraints. Existing encode history is separate from the machine-validated calibration profile; copying history from a different machine can distort its time estimates.

### CSV summary

The first data row is `TOTAL (N candidates)`. Source GiB, duration minutes, estimated saved GiB and encode-hours are summed across the full CSV. Overall saving percentage and GiB saved/hour are recomputed from totals, not added or averaged per-file. Non-additive fields and the summary Path stay blank. The separate text shortlist contains file paths only.

Totals are static values, not spreadsheet formulas. Encode-hours describe sequential work for one worker on the chosen lane, **not mixed CPU/GPU queue wall time**.

## Benchmark

**Option 1: Calibration.** Select one representative SDR/HDR10 file. The same 20-second source segment is encoded twice per available lane, sequentially, using current sample-encoder settings. Unsupported/absent GPUs are skipped; failed lanes are not saved. Dolby Vision is rejected for this limited workflow. Keep other encoding workloads idle.

Results go to `.queue/calibration.json`; a replaced profile is retained as `.bak`. Machine/hardware/driver identity, tracked encoder settings and FFmpeg binary changes invalidate it. Calibrate separately on each machine. The measurement includes decode and video encode but excludes quality-search passes, audio copying, full HDR metadata processing and final remux. It is not a cross-encoder quality-equivalence test.

**Option 2: CPU threading.** The original SVT-AV1 benchmark compares threading/pinning and single versus parallel CPU work, using a real or synthetic sample. Its results do not feed Candidates. A synthetic clip is useful for exercising the benchmark but is less representative of a film library.

## Loudness

Start with a scan: the quick option samples audio; the full option measures the whole stream. Auto anchoring uses a centre-channel measurement when available, otherwise programme loudness. A centre channel is a dialogue proxy, not speech separation.

The underlying tool supports HomeTheatre (default), EBU-R128, ATSC-A85, Netflix and Custom targets. Default tolerance is 2 LU; centre boost is off. Linear mode applies static gain while respecting the peak constraint; Compress uses two-pass `loudnorm` and can alter dynamics. Its default LRA target is 11 LU. Compression does not guarantee an exact resulting loudness range.

Applying changes requires full measurement; quick scans cannot be combined with Apply. The default track policy adds a normalized track and makes it default while retaining the original track. Track replacement and source-file overwrite are distinct options, and source overwrite is off by default. Review output and logs before replacing library media. These audio-tool defaults differ from the AV1 queue's `ReplaceOriginal` default.

## Verify and Quality

Verify exercises actual sample output and preservation in its sandbox; it is not a certification of every title or device. Quality helps choose metrics and thresholds, whereas calibration measures speed. See [Quality targeting](quality.md) and [HDR](hdr.md) for what the results establish and what still needs visual/playback inspection.
