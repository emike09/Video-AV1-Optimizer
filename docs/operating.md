# Running it day to day

[README](../README.md) · [Settings](settings.md) · [Tools](tools.md)

## Setup and launch

Use Windows and PowerShell 7. Keep `Video2AV1Queue.bat` beside `Video2AV1Queue.ps1`. The script searches beside itself and on PATH for FFmpeg/ffprobe; if either is missing it requests a folder, verifies both tools, and adds the selected folder to session and Windows user PATH. The current implementation announces and performs this update automatically after folder selection; it does not offer the older Y/N persistence prompt. Blank input cancels setup.

Drag video files or folders onto the batch file to queue them, or double-click to open the menu. Folder input expands recursively over the [accepted extensions](../SUPPORTED-FORMATS.md). Partial `.encoding.tmp` files are excluded.

```powershell
# Open menus
pwsh -NoProfile -File .\Video2AV1Queue.ps1
pwsh -NoProfile -File .\Video2AV1Queue.ps1 -Mode Tools
pwsh -NoProfile -File .\Video2AV1Queue.ps1 -Mode Candidates

# Queue a file with an explicit encoder preference
pwsh -NoProfile -File .\Video2AV1Queue.ps1 -EncoderPreferenceOverride Intel 'D:\Movies\test-copy.mkv'
```

Top-level switches include `Mode`, `EncoderPreferenceOverride`, `AutoCRFOffsetOverride` and `TargetGiBPerHourOverride`. Tool-specific function parameters such as Doctor's `TestFile` are not top-level CLI switches. Use the appropriate menu to supply them. Do not dot-source the entire application simply to access a helper: its startup code runs.

Interactive offers Auto, Aggressive (+2 CRF), Balanced (0), Quality (-2), and a custom target output rate, followed by encoder selection. A requested target is a heuristic, not a promised final file size. Per-job preferences are recorded with queued work.

## Original files and output names

The shipped AV1 queue has `ReplaceOriginal = $true` and `KeepBackupOriginal = $false`. Successful conversions can therefore replace originals without retaining a backup. Test copies first, or disable replacement and inspect the results. Loudness copy behavior is separate.

Output is Matroska. Recognized codec tokens such as x264, x265, H.264, H.265 and HEVC become AV1 in the basename; other basenames are retained. In-progress output is named `<basename>.encoding.tmp.mkv` beside the source. The intended final name is `<basename>.mkv`. Collision handling and validation still apply; do not assume a renamed file is a backup of the source.

## Queue and concurrency

State is stored under the script's `.queue` directory, including pending/working entries, preflight data, logs and current-job state. A named mutex prevents concurrent queue-manager instances. Moving the application to another directory creates a different local queue location; it is not an automatic queue migration.

Auto routes according to source suitability and capacity. CPU-only content can wait for CPU; GPU absence or busy capacity is handled separately from failed encoding. CPU and suitable GPU workers can overlap, sharing RAM, bandwidth and power. Capability and metadata checks may prevent a requested hardware route.

### Live controls

These are two-key commands where shown, not simultaneous shortcuts:

| Keys | Action |
|---|---|
| `H` | Toggle help. |
| Worker number `1`–`9`, then `P` / `R` / `S` | Pause, resume or stop that worker. |
| `Q`, then `P` | Pause starting new queued jobs. Existing workers are distinct from queue scheduling. |
| `Q`, then `R` | Resume queue scheduling. |
| `Q`, then `C` | Clear pending work through the queue command. |
| `X` | Confirm soft exit: finish active jobs and start no new ones. |
| `Ctrl+C` | Ask to cancel the current queue. |

The dashboard redraws in place and adapts to terminal dimensions. Short windows may hide secondary information. Log files are the durable record; terminal scrollback is not the full job history.

After interruption, restart the launcher and let queue recovery inspect state. Partial output is not complete output, and recovery does not imply a bitstream resumes at its previous frame. Prefer soft exit when possible. Restart after a script update; an existing process retains loaded code.

## Logs and estimates

`.queue/encode_log.csv` records outcomes, source/output sizes and durations, reductions, selected streams, source profile, resolved lane/settings, preflight/quality measurements and notes. Session text logs retain detailed decisions and diagnostics. Fields include `Status`, `ResolvedEncodeLane`, `ResolutionTier`, `CodecClass`, `BPP`, `SourceDurationSec` and `ElapsedSec`, which Candidates uses for learning.

The complete CSV schema is the `$LogColumns` declaration in [the script](../Video2AV1Queue.ps1). New versions can migrate existing headers. Do not treat a quality-fallback result as a passed target merely because its final job status is successful.

Candidates CSVs and `.queue/calibration.json` serve different purposes from the encode log. See [Tools](tools.md) for summary rows, calibration precedence and the difference between encode-hours and queue completion time.
