# Lap Counting & Race Logic

**Primary files:** `src/game/racer.c`, `src/overlays/ovl_i2/race.c`

## Lap Counting (`Racer_UpdateFromControls()`, ~lines 4298–4370)

`racer->lapDistance` tracks the racer's position along the spline from the start line (interpolated segment length).

Each frame, the delta is checked:

```c
sp128 = (previous lapDistance) - (new lapDistance);

if (sp128 > sCourseHalfLength) {
    racer->lap++;                              // crossed finish line forward
    racer->lapsCompletedDistance += courseLength;
}
if (sp128 < sCourseNegativeHalfLength) {
    racer->lap--;                              // crossed finish line backward (penalty)
}
```

A lap "completes" when `racer->lap == racer->lapsCompleted + 1`:
- `lapTimes[]` for the just-finished lap is recorded
- `RACER_STATE_CAN_BOOST` is restored (one new boost token)

## Racer Struct Lap Fields

Defined in `include/unk_structs.h`:

| Field | Type | Meaning |
|---|---|---|
| `lap` | `s16` | Current lap number (0 before crossing start, 1 after first crossing) |
| `lapsCompleted` | `s16` | Number of lap completions recorded |
| `lapTimes[3]` | `s32[3]` | Split times for each completed lap |
| `lapDistance` | `f32` | Distance along the current lap's spline |
| `raceDistance` | `f32` | `lapsCompletedDistance + lapDistance` — monotonically increasing, used for ranking |

## Race Finish (~lines 4341–4365)

```c
if (lap == gTotalLapCount + 1) {
    stateFlags |= RACER_STATE_FINISHED | RACER_STATE_CPU_CONTROLLED;
    energy = maxEnergy;        // energy restored to full on finish
    gRacersFinished++;
    if (isPlayer) gPlayerRacersFinished++;
}
```

After finishing, the racer is switched to CPU control so it keeps driving cleanly around the remaining laps while others finish.

## Race Position Ranking

`Racer_UpdateRacePositions()` (~line 1102): sorts all active racers by `raceDistance` descending, then writes:
- `racer->position` — current race position (1st, 2nd, …)
- `gRacersByPosition[]` — sorted array of racer pointers

## Race Session Lifecycle (`src/overlays/ovl_i2/race.c`)

```c
Race_Init()  // line 63 — initializes course, all racers, camera, effects
Race_Update() // line 94 — called every frame:
    Racer_Update()           // iterates all racers
        Racer_UpdateMovement()   // per racer:
            Cpu_GenerateInputs() // (CPU only) synthesize controller
            Racer_UpdateFromControls() // same physics for CPU and player
    Racer_UpdateRacePositions()
```

## Related Files
- `include/unk_structs.h` — `Racer` struct lap fields
- `include/fzx_racer.h` — `RACER_STATE_FINISHED`, `RACER_STATE_CAN_BOOST`
- `src/overlays/ovl_i3/hud.c` — HUD lap counter display
- [AI.md](AI.md) — how CPU racers use `raceDistance` for rubber-banding
