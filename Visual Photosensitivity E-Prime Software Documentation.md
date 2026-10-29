# Visual Photosensitivity E-Prime Software Documentation

Documentation derived from the generated E-Basic script (`VisualPhotosensitivityScript.txt`, generated 2026-10-19 from E-Studio 3.0.3.219), the external startup file (`VisualPhotosensitivity.startupInfo3`), and the Experiment Object properties. The `.es3` body is encrypted, so object properties not reflected in the script (list row contents, goggle test display durations) are stated as assumptions and marked as such.

Experiment version stamped by E-Studio: 2.0.0.702 (auto-increment on). Author: Donald Hoffman.

# Purpose

An fMRI block-design paradigm that presents light stimuli through a pair of goggles whose brightness is set over a serial link. The subject performs no task; the program only controls goggle brightness, synchronizes to the scanner, and writes a timing log for later alignment with the imaging data.

# Hardware and devices

| Device | E-Prime object | Settings | Role |
|---|---|---|---|
| Light goggles | `LightController` (SerialDevice) | COM3, 115200 8N1 | Receives brightness commands |
| MRI trigger | `SRBOX` (SRBoxDevice, model 200) | COM1, 19200 | Scanner trigger arrives as button `5` |
| Operator console | `Keyboard` | Presses only | Menu selection, manual trigger, abort |
| Display | `Display` | 1024x768, black, DisplayIndex 1 | Operator status text only; subject does not view it |

Mouse, Sound, and Button devices are declared by E-Studio defaults but unused.

## Goggle serial protocol

Each command is an ASCII integer followed by newline (`\n`), then the output buffer is flushed:

- `GoggleSet(n)` sends `n\n`, where `n` is 0 to 255.
- `GoggleReset` sends `0\n` (goggles off).

When `IsDevMode` is True (startup file), serial writes are skipped and only debug output is produced. All other logic runs unchanged.

# Startup parameters

## Prompted at launch (Experiment Object > Startup Info)

| Name | Prompt | Default | Notes |
|---|---|---|---|
| Subject | Subject Number | 1 | 0 disables E-Prime data logging |
| Session | Session Number | 1 | |
| DimLevel | light level for DIM blocks | 60 | 0 to 255 |
| BrightLevel | level for BRIGHT blocks | 150 | 1 to 255 |

A summary dialog confirms the values before the run. In practice only Subject and Session are changed.

## External file `VisualPhotosensitivity.startupInfo3`

"Load values from external startup info files" is enabled, so this XML file is read at launch and its items become context attributes. The script reads the following directly, so they can be changed without opening E-Studio:

| Item | Value | Effect |
|---|---|---|
| FMRITrigWait | 30000 ms | Max wait per trigger (x5 per trial) |
| StimulusDuration | 12000 ms | Light-on block length (alternating trials) |
| RestDuration | 48000 ms | Light-off block length (alternating trials) |
| ContinuousTrialDuration | 900000 ms | Max length of a continuous trial (15 min) |
| DimLevel / BrightLevel | 60 / 150 | Defaults; prompted values override |
| IsDevMode | False | True suppresses serial output |
| LightController.Open / CommPort | True / 3 | Goggle port |
| SRBox.Open | True | Set False to run without the trigger box |
| Display.DisplayIndex | 1 | |
| ExperimentVersion | 2.1.0 | See Known issues: not the value that gets logged |

# Operator workflow

1. Launch, answer the four prompts, confirm the summary.
2. The console shows the trial menu (`TrialStartPrompt`) with Subject ID and the dim/bright levels. The menu repeats after every trial or test until the session is ended.
3. Keys:
   - `0` goggle test (connectivity check, no subject present)
   - `1` Scan 1: DIM continuous
   - `2` Scan 2: DIM alternating
   - `3` Scan 3: BRIGHT alternating
   - `4` Scan 4: DARK continuous (brightness 0)
   - `a` end session (the menu text says `q`; see Known issues)
   - any other key: menu redisplays
4. During a trial the console shows what phase is running and the abort key. `a` aborts the current trial at any phase and returns to the menu. Trial numbering continues across the session; an aborted trial consumes its number.

# Trial types

Every trial begins with the goggles reset to 0 and a `TRIALSTART` log entry.

## Trigger wait (all scan trials)

`TriggerLoopAlternating` / `TriggerLoopContinuous` run a 5-row list (assumed `triggerNumber` 1 to 5). Each row shows "Waiting for MRI Trigger N" and waits up to `FMRITrigWait` (30 s) for `5` from the SRBox or keyboard. Keyboard `5` is a manual trigger. The scanner is already acquiring when a trial is started; the purpose of the loop is to let five volumes pass for signal stabilization and start the stimulus on the fifth trigger. After the loop, `TRIGRECV` is logged with the offset time of the fifth wait display. The 30 s timeout is a safety net and is not expected to fire in normal operation (one trigger per TR).

## Alternating trials (types 2 and 3)

After the trigger wait, `StimulusBlock` runs a 10-row list (assumed `blockNumber` 1 to 10). Each block:

1. `GoggleReset` (0), then `RestBlockDisplay` for `RestDuration` (48 s). Logs `RESTSTART` / `RESTSTOP`.
2. `GoggleSet(trialBrightness)`, then `StimulusBlockDisplay` for `StimulusDuration` (12 s). Logs `STIMSTART` / `STIMSTOP`.
3. `GoggleReset`.

Total: 10 x 60 s = 600 s of task after the fifth trigger. Trial brightness is `DimLevel` for type 2, `BrightLevel` for type 3. `TRIALDONE` is logged at the end.

Timeline per block (t relative to block start):

```
t=0      goggles 0, rest display on        RESTSTART
t=48 s   rest display off                  RESTSTOP
         goggles = brightness
         stim display on                   STIMSTART
t=60 s   stim display off                  STIMSTOP
         goggles 0
```

## Continuous trials (types 1 and 4)

After the trigger wait, `GoggleSet(trialBrightness)` (DimLevel for type 1, 0 for type 4) and `ContinuousScanDisplay` runs until the operator presses `c` (complete) or `a` (abort), or `ContinuousTrialDuration` (900 s) elapses. Then `GoggleReset`. Logs `STIMSTART` / `STIMSTOP` (display onset/offset, block 0) followed by one of `TRIALCOMPLETE`, `TRIALABORTED`, or `TRIALTIMEOUT`.

Type 4 (dark) is the control: identical flow with brightness 0, so the goggles stay off throughout.

## Goggle test (type 0)

Operator-only check that the serial link works. `GoggleTestBlock` runs 20 cycles of goggles at level 15 (hardcoded `g_testLevel`) for about 5 s, then 0 for about 5 s (durations are object properties, not in the startup file). `a` aborts. Logs `GTESTSTART` / `GTESTDONE` or `GTESTABORTED`. Does not increment the trial number.

# Abort behavior

| Phase | Effect of `a` |
|---|---|
| Trigger wait | Terminates the trigger list, logs `TRIALABORTED`, returns to menu. Goggles already 0. |
| Rest block | Terminates `StimulusBlock`, logs `TRIALABORTED`. Goggles already 0. |
| Stimulus block | Terminates `StimulusBlock`, logs `TRIALABORTED`, then `GoggleReset`. |
| Continuous display | `GoggleReset`, logs `TRIALABORTED`. |

Session end (`a` at the menu) sends `GoggleReset`, logs `EXPEXIT`, and writes end times to the metadata file. If E-Prime's user-break key sequence was used, `EXPBREAK` is logged first.

# Output files

Written to the experiment directory in addition to the standard E-Prime `.edat3` / E-Recovery `.txt`:

`VSExperimentLog-SSSSS-NN-yyyymmdd-hhmmss.csv` (Subject zero-padded to 5, Session to 2, session start date and time).

Header: `Event Type, Time, Trial Num, Trial Type, Block, Brightness`

| Column | Meaning |
|---|---|
| Event Type | See table below |
| Time | `Clock.Read()` milliseconds on E-Prime's experiment clock, or the display's OnsetTime/OffsetTime on the same clock |
| Trial Num | 1-based count of scan trials started this session; 0 for goggle test and session events |
| Trial Type | Menu key (`0` to `4`); `0` on session events |
| Block | Block number for RESTSTART/RESTSTOP/STIMSTART/STIMSTOP in alternating trials; 0 otherwise |
| Brightness | Trial brightness on TRIALSTART/STIMSTART/STIMSTOP; 0 otherwise |

| Event | Logged when | Time source |
|---|---|---|
| GTESTSTART / GTESTDONE / GTESTABORTED | Goggle test | Clock.Read |
| TRIALSTART | Trial selected, goggles reset | Clock.Read |
| TRIGRECV | Trigger loop finished | OffsetTime of last trigger-wait display |
| RESTSTART / RESTSTOP | Each rest block | Display OnsetTime / OffsetTime |
| STIMSTART / STIMSTOP | Each stimulus block, or the continuous display | Display OnsetTime / OffsetTime |
| TRIALDONE | Alternating trial finished (also after abort) | Clock.Read |
| TRIALCOMPLETE / TRIALTIMEOUT / TRIALABORTED | Continuous trial ended; abort in any trial | Clock.Read |
| EXPBREAK / EXPEXIT | Session end | Clock.Read |

`VSExperimentLog-...txt` holds metadata lines: ExperimentVersion, StartWallTime, StartClockTime, SubjectID, SessionID, DimLevel, BrightLevel, EndWallTime, EndClockTime. `StartClockTime` plus `StartWallTime` give the offset to convert log times to wall clock.

Note on timing precision for the analyst: goggle brightness commands are sent immediately before the following display's `.Run`, so STIMSTART/RESTSTART (display onset) trail the serial write by a few milliseconds, and the serial write time itself is not logged. This is expected behavior, not a defect. The hemodynamic and photosensitivity responses being measured have time constants of multiple seconds, and this jitter was reviewed with the PIs and accepted. Treat the logged onset times as the stimulus times.

# Known issues

Ordered by impact.

1. `q` does not quit. `Case "q"` in `SelectTrialType` is empty, so `q` falls out of the Select and into the goggle test with the previous trial's brightness. Only `a` ends the session. Fix: `Case "q", "a"`.
2. ExperimentVersion in the metadata file is the E-Studio stamp (2.0.0.702), not the startup file's 2.1.0. `Main` sets the attribute after the startup file is loaded. Fix: log a different attribute name from the startup file, or drop the startup file item.
3. `TriggerWaitContinuous` keyboard mask has an infinite time limit (-1) while the alternating version and the SRBox mask use `FMRITrigWait`. Likely an editing slip; worth making consistent.
4. Inconsistent end-of-trial events: alternating trials log `TRIALDONE` even after `TRIALABORTED`; continuous trials never log `TRIALDONE`. Analysis code should treat `TRIALABORTED` as terminal and ignore a following `TRIALDONE`.
5. Trigger timeout is not distinguished in the log. If the trigger is not reaching E-Prime (SRBox cable, COM port, box off) while the scanner runs, each wait expires after 30 s, the operator sees the "Waiting for MRI Trigger N" display stall instead of advancing every TR, and after 150 s the stimulus starts unsynced with `TRIGRECV` logged. Low likelihood and visible to an attentive operator, but not detectable from the log afterward. Recommended fix: treat a timed-out wait as an abort. In `CheckForAbortTWA` (and likewise `CheckForAbortTWC` with the Continuous objects):

   ```vb
   If TriggerWaitAlternating.RESP = "a" Then
   	Debug.Print "Abort during trigger wait"
   	TriggerLoopAlternating.Terminate
   	g_isAborted = True
   ElseIf TriggerWaitAlternating.RESP = "" Then
   	Debug.Print "Trigger timeout, aborting trial"
   	LogEvent "TRIGTIMEOUT", Clock.Read(), g_trialNumber, g_trialType, 0, 0
   	TriggerLoopAlternating.Terminate
   	g_isAborted = True
   End If
   ```

   The existing post-loop code in `SessionProc` then logs `TRIALABORTED` and returns to the menu. This covers both an unopened SRBox and a dead trigger cable, and preserves the keyboard `5` manual trigger used for debugging. A device-open check at trial start (`SRBOX.GetState() <> ebStateOpen`) would catch the first case sooner but would block manual triggering; not recommended unless manual triggering is retired.
6. Cosmetic: typos in operator text ("Photosenstivity", "simulus", "Brighness", "Continous"); several copy-pasted debug strings.

# Assumptions

- `TriggerLoopAlternating` / `TriggerLoopContinuous` rows are `triggerNumber` 1 to 5 and `StimulusBlock` rows are `blockNumber` 1 to 10, in sequential order (lists use `SequentialOrder`, `Cycles(1)`, `Samples(5)` / `Samples(10)`).
- `GoggleOnDisplay` / `GoggleOffDisplay` durations are 5000 ms each (object properties, not visible in the script).
- `TrialStartPrompt` duration is infinite (menu waits for a key).
