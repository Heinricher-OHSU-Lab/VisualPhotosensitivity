# Visual Photosensitivity Experiment Analyst Guide

Default timing values below come from `VisualPhotosensitivity.startupInfo3` and can be verified there for a given session.

# Files per session

One pair per E-Prime launch, written to the experiment folder:

- `VSExperimentLog-SSSSS-NN-yyyymmdd-hhmmss.csv` (event log, the file you need)
- `VSExperimentLog-SSSSS-NN-yyyymmdd-hhmmss.txt` (metadata)

`SSSSS` is the subject number zero-padded to 5 digits, `NN` the session number to 2 digits, followed by launch date and time. A session can contain several scan trials, so one CSV normally covers all scans for that subject visit.

The standard E-Prime `.edat3` / E-Recovery `.txt` files are also written but contain nothing needed for alignment.

---

# CSV format

Header row: `Event Type, Time, Trial Num, Trial Type, Block, Brightness` (note the spaces after commas in the header only; strip whitespace when parsing). Data rows have no spaces.

| Column | Meaning |
|---|---|
| Event Type | One of the codes below |
| Time | Milliseconds on E-Prime's experiment clock (starts near 0 at launch, monotonic, integer) |
| Trial Num | Running count of scan trials started in this session, 1-based. 0 for goggle tests and session-level events. Aborted trials still consume a number. |
| Trial Type | Menu key the operator pressed: `1` dim continuous, `2` dim alternating, `3` bright alternating, `4` dark continuous, `0` goggle test. `0` on session events. |
| Block | Block number within an alternating trial (1 to 10) on REST*/STIM* rows; 0 elsewhere |
| Brightness | Goggle command value (0 to 255) on TRIALSTART / STIMSTART / STIMSTOP; 0 elsewhere. Default dim = 60, bright = 150, dark = 0. |

All times are on the same clock. The metadata file gives `StartClockTime` (clock ms) and `StartWallTime` (hh:mm:ss) at launch, which lets you convert to wall clock if needed, but alignment to MRI should be done via TRIGRECV, not wall clock.

---
# Event codes

| Event | When | Time column holds |
|---|---|---|
| TRIALSTART | Operator selected a scan trial; goggles set to 0 | Clock.Read at selection |
| TRIGRECV | Fifth scanner trigger received | Offset of the fifth trigger-wait display, i.e. the time of trigger 5 |
| RESTSTART / RESTSTOP | Rest (goggles 0) block in alternating trials | Display onset / offset |
| STIMSTART / STIMSTOP | Stimulus block in alternating trials, or the single stimulus period in continuous trials | Display onset / offset |
| TRIALDONE | End of an alternating trial (logged even after an abort) | Clock.Read |
| TRIALCOMPLETE | Continuous trial ended by operator (`c`) | Clock.Read |
| TRIALTIMEOUT | Continuous trial hit its max duration (900 s) | Clock.Read |
| TRIALABORTED | Operator aborted (`a`) at any phase | Clock.Read |
| GTESTSTART / GTESTDONE / GTESTABORTED | Operator goggle connectivity check, no subject involved. Ignore. | |
| EXPBREAK / EXPEXIT | Session ended | |

---
# Aligning to the scanner

The scanner is already running when the operator starts a trial. E-Prime counts five scanner triggers (one per TR) before doing anything, so that signal has stabilized. Trigger 5 is the only trigger whose time is logged (TRIGRECV); triggers 1 to 4 are not recorded.

**TRIGRECV is the alignment point.** It coincides with the start of the fifth acquired volume of that scan. Everything after it is expressed in the same clock, so:

- stimulus time relative to scan start = (event Time - TRIGRECV Time) + 4 x TR
- or, if the first four volumes are being discarded anyway, stimulus time relative to the retained series = event Time - TRIGRECV Time

The first RESTSTART (alternating) or STIMSTART (continuous) follows TRIGRECV by a few ms; treat it as coincident with volume 5.

Sanity check: TRIGRECV minus TRIALSTART should be between 4 x TR and 5 x TR. A value near 150000 ms means no triggers reached E-Prime (five 30 s timeouts expired) and the trial ran unsynchronized; discard it. The log will show TRIGRECV in that case anyway, so this check is the only way to catch it from the data.

---
# Trial structures

## Alternating (types 2 and 3), 10 blocks

Per block, nominal: 48 s rest (goggles 0) then 12 s stimulus (goggles at Brightness). 600 s total after TRIGRECV.

```
TRIALSTART,t0,n,2,0,60
TRIGRECV,T,n,2,0,0
RESTSTART,T+~5,n,2,1,0
RESTSTOP,T+48005,n,2,1,0
STIMSTART,T+48005,n,2,1,60
STIMSTOP,T+60005,n,2,1,60
RESTSTART,T+60005,n,2,2,0
...
STIMSTOP,T+600005,n,2,10,60
TRIALDONE,...,n,2,0,0
```

RESTSTOP of block k and STIMSTART of block k are the same display boundary and will agree to within a frame. STIMSTOP of block k and RESTSTART of block k+1 likewise. Use the logged onset/offset times rather than assuming exact 48000/12000 ms; actual values can differ by a display refresh or two.

## Continuous (types 1 and 4)

Single stimulus period from trigger 5 until the operator ends it or 900 s elapses. Type 1 goggles at dim level for the whole period; type 4 goggles at 0 (dark control, identical flow).

```
TRIALSTART,t0,n,1,0,60
TRIGRECV,T,n,1,0,0
STIMSTART,T+~5,n,1,0,60
STIMSTOP,T+d,n,1,0,60
TRIALCOMPLETE,...,n,1,0,0        (or TRIALTIMEOUT / TRIALABORTED)
```

STIMSTOP is the time the operator ended the trial; the scan may have continued past it.

# Pairing trials with MRI runs

Pair by order within the session and by Trial Type. Trial Num increments for every scan trial started, including aborted ones. An aborted trial normally means the scan was also stopped; check the scanner log. Rows with Trial Num 0 (goggle tests, session events) do not correspond to any scan.

---
# Notes

- An aborted alternating trial logs TRIALABORTED and then TRIALDONE. Treat TRIALABORTED as terminal; ignore the following TRIALDONE.
- Continuous trials never log TRIALDONE; their terminal event is TRIALCOMPLETE, TRIALTIMEOUT, or TRIALABORTED.
- STIM*/REST* rows for an aborted block may be partial (e.g. RESTSTART/RESTSTOP present, STIMSTART absent, or STIMSTOP early).
- Timing precision: the goggle brightness command is sent a few ms before the logged STIMSTART/RESTSTART (display onset), and the serial write itself is not logged. Use the logged onset times as the stimulus times.
- Brightness values are goggle command levels (0 to 255), not calibrated luminance.
