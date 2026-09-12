# Posner Cueing Task (PsychoPy)

A spatial cueing experiment implemented in PsychoPy Builder. A peripheral or central cue draws attention to a location before a target appears; the task measures how validly-cued (vs. invalidly-cued) targets are responded to faster, separately for **exogenous** (peripheral, involuntary) and **endogenous** (central, voluntary) cues.

## Requirements

- **PsychoPy** 2023.2 or later (uses the Routine Settings dialog; tested on 2026.2.3)
- Keyboard backend set to **PTB** for accurate RT: `PsychoPy → Preferences → Hardware → keyboardBackend = ptb` (requires a restart of PsychoPy to take effect)
- Python packages `numpy`, `os` — both part of PsychoPy's own standard environment, no extra install needed

## File structure

```
posner_task/
├── Posner_Cueing_Task.psyexp     # the Builder experiment
├── conditions.xlsx                # trial list (see below)
├── posner/                        # stimulus images
│   ├── fixation_cross.png
│   ├── box.png                    # placeholder boxes, left/right
│   ├── cue_box.png                # exogenous cue (highlighted box)
│   ├── cue_arrow_left.png         # endogenous cue, pointing left
│   ├── cue_arrow_right.png        # endogenous cue, pointing right
│   └── target_dot.png
└── data/                          # created automatically on first run
    ├── <participant>_posner_task_<date>.csv
    ├── <participant>_posner_task_<date>.log
    ├── <participant>_posner_task_<date>.psydat
    └── <participant>_posner_task_<date>_triggers.csv
```

## Running it

Open `Posner_Cueing_Task.psyexp` in PsychoPy Builder and click **Run** (▶), or run the compiled `.py` script directly (e.g. via `python Posner_Cueing_Task.py` or PsychoPy's Runner). A participant-info dialog appears first; fill in participant/session and start.

## Task structure

One trial (`trial` routine):

| Phase | Duration | Notes |
|---|---|---|
| Fixation | 500 ms | fixation cross + both placeholder boxes on screen |
| Cue | 100 ms | box highlight (exogenous) or arrow (endogenous) |
| SOA (blank) | jittered, uniform 200–600 ms | drawn fresh each trial, logged in `soa` column |
| Target | until response, 1500 ms timeout | dot appears at the cued or uncued location |

Followed by an **ITI** (~0.8–1.2 s jittered blank, not part of the original spec — added so the next trial's fixation doesn't appear on the same frame as the previous response), then loops to the next trial.

After all trials, an **end screen** ("Thanks for taking part! Press any key to exit.") is shown once before the experiment closes — also not in the original spec, added for participant experience.

**Response keys:** the arrow keys `left` / `right`, mapped to "target appeared on the left/right side." `Force end of Routine` is enabled on the keyboard component, so the trial ends immediately on a keypress rather than waiting out the full 1.5 s.

## Cue types and positions

- **Endogenous** cues (`cue_arrow_left.png` / `cue_arrow_right.png`) are shown at a fixed position **above fixation**, `(0, 0.25)` in height units, regardless of which way they point — the arrow's own artwork conveys the cued side, not its screen position.
- **Exogenous** cues (`cue_box.png`) appear directly **at the cued location**, `(-0.3, 0)` for left or `(0.3, 0)` for right — the peripheral flash itself is what captures attention, so it has to sit where the target might appear.
- Target and placeholder-box positions should match the exogenous cue's coordinates (currently `±0.3`) so the spatial validity manipulation lines up visually with what the participant actually sees.

## Conditions file (`conditions.xlsx`)

Columns: `cueType` (`exogenous` / `endogenous`), `validity` (`valid` / `invalid`), `cueSide` (`left` / `right`), `targetSide` (`left` / `right`), `cueFile` (path to the cue image for that row).

PsychoPy's loop has **no built-in row-weighting** — every row in the conditions file is sampled with equal probability. To get an 80% valid / 20% invalid split (balanced across sides, within each `cueType`), **valid combinations are duplicated 4× and invalid combinations appear once** in the spreadsheet itself, rather than relying on any weight/probability field. Check the row counts in the actual file if you need to confirm the exact ratio — they should work out to 4:1 valid:invalid per `cueType`.

The loop's `nReps` (set to 1) setting multiplies the whole conditions file — e.g. a 20-row file with `nReps=5` would give 100 trials.

## Trigger codes

Event markers use small integers, not strings — the format EEG/MEG systems
actually expect on a trigger line, one byte per event. The full codebook is
also saved as a standalone file, `trigger_codes.csv`, so it can be handed to
whoever is running the amplifier/recording software without needing to read
any PsychoPy code.

| Label | Code | Meaning |
|---|---|---|
| `FIX_ONSET` | `0` | fixation onset |
| `CUE_LEFT` | `11` | cue onset, cued side = left |
| `CUE_RIGHT` | `12` | cue onset, cued side = right |
| `TARGET_LEFT` | `21` | target onset, target on left |
| `TARGET_RIGHT` | `22` | target onset, target on right |
| `RESPONSE_INCORRECT` | `30` | response given, wrong key |
| `RESPONSE_CORRECT` | `31` | response given, correct key |
| `RESPONSE_TIMEOUT` | `32` | no response within the timeout window |
| `ITI_ONSET` | `40` | inter-trial interval begins |

Scheme: tens digit = event type, ones digit = detail. `TRIGGER_CODES`
(defined once, in the `setup` routine's code component) is the single
source of truth — `trigger_codes.csv` is a copy for reference, not a
separate definition, so if the dict ever changes, update this file to match.

**How events are marked:** every trigger goes through one function,
`send_trigger(label)`, which looks up the integer code and logs
`timestamp, code, label, trial_n` to the triggers CSV. Swapping the log-file
stub for a real parallel port or LSL stream only means changing the body of
that one function — every call site (`send_trigger('CUE_LEFT')`, etc.) stays
identical.

**Where events are marked, in time:** `FIX_ONSET`, `CUE_LEFT`/`CUE_RIGHT`,
and `TARGET_LEFT`/`TARGET_RIGHT` all fire from an `Each Frame` block, each
guarded by a "not yet sent" flag and checked against that trial's own onset
variables (`0`, `cue_onset`, `target_onset`). This keeps each marker locked
to the exact frame its stimulus first appears on, since the onset times are
trial-varying (they depend on the jittered SOA) and can only be known by
checking elapsed time on every frame until the condition is met. `ITI_ONSET`,
by contrast, fires from `Begin Routine` — it's the only event in the `ITI`
routine and always occurs at that routine's fixed start (frame 0), so there's
no moving target time to poll for; `Begin Routine` and an `Each Frame` check
of `t >= 0` fire at the identical point in PsychoPy's frame loop, so this is
a simplification, not a difference in timing accuracy.

**Where events are marked, physically:** currently a `.csv` log file
sitting next to the participant's data file. With real hardware attached,
"where" becomes a specific parallel-port address or LSL stream instead —
changing only the body of `send_trigger()`, as above.

## Output files

**`<participant>_..._<date>.csv`** — one row per trial, including:
- Condition columns from the conditions file: `cueType`, `validity`, `cueSide`, `targetSide`, `cueFile`
- `soa` — this trial's jittered SOA in seconds
- `correct` — 1 if the keypress matched `targetSide`, 0 if wrong or timed out
- `rt` — reaction time in seconds, measured from target onset to keypress (PTB-backed, screen-flip-synced)
- `key_resp.keys`, `key_resp.rt`, `key_resp.duration` — PsychoPy's own raw keyboard-component columns
- Per-component `.started`/`.stopped` timestamps (`trial.started`, `fixation.started`, `cue.started`, `target.started`, `ITI.started`, etc.) — useful for double-checking timing empirically
- Standard PsychoPy metadata: `participant`, `session`, `date`, `expName`, `psychopyVersion`, `frameRate`, `expStart`

**`<participant>_..._<date>_triggers.csv`** — the event marker log, columns `timestamp, code, label, trial_n`. One row per event:
- `FIX_ONSET`, `CUE_ONSET`, `TARGET_ONSET` — fired from an `Each Frame` check against the trial's own onset-time variables, so each marker lands on the same frame its stimulus actually first appears
- `RESPONSE` or `TIMEOUT` — fired at the end of the trial depending on whether a key was pressed
- `ITI_START` — fired at the start of the inter-trial blank

`timestamp` uses `core.getTime()`, PsychoPy's monotonic clock — the same clock `rt` is measured against, so `RESPONSE` timestamp minus `TARGET_ONSET` timestamp should reproduce `rt` as a cross-check.

**`<participant>_..._<date>.log`** — PsychoPy's own full experiment log (window creation, frame rate measurement, every trigger mirrored via `logging.exp()`, warnings). Useful for debugging timing issues after the fact.


```python
# Parallel port
from psychopy import parallel
port = parallel.ParallelPort(address=0x0378)
# inside send_trigger(): port.setData(code); core.wait(0.001); port.setData(0)

# LSL
from pylsl import StreamOutlet, StreamInfo
info = StreamInfo('PosnerMarkers', 'Markers', 1, 0, 'string', 'posner001')
outlet = StreamOutlet(info)
# inside send_trigger(): outlet.push_sample([label])
```

No other part of the experiment needs to change — every trigger call site just calls `send_trigger(label)`.


