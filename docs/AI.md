# AI / CPU Behavior

**Primary file:** `src/overlays/ovl_i3/cpu.c`

## Overview

CPU racers do **not** have separate physics — they run through the exact same `Racer_UpdateFromControls()` as the player. The AI just synthesizes a fake `Controller` struct each frame and passes it in. This means any physics quirk the player experiences, CPU racers experience too.

## Initialization (`Cpu_InitRacer(Racer* racer)`, ~line 803)

- Loads a per-course pre-baked path script: `TextureCache_LoadAssetData(D_i3_8013DB80[courseIndex], ...)` → fills `D_i3_8013DBE8[]`, a 512-entry table of `(speed, lateral_offset)` pairs. This is the CPU's "line" around the track.
- Sets per-character `racer->awarenessFlags` from `D_i3_8013E7A8[character].unk_00` — controls aggression/awareness behavior.
- Sets `racer->unk_360` / `racer->unk_364` (side-attack probability thresholds) from tables indexed by `[character_tier * 4 + gDifficulty]`.
- For CPU racers: copies the player's `unk_1A8` performance factor with difficulty-based noise → rubber-band speed scaling. MASTER difficulty picks random high-end values.

## Per-Frame Input (`Cpu_GenerateInputs()`, ~line 1186)

To save CPU time, only **1 in 4 racers** is updated each frame:

```c
if (racer->id % 4 != gGameFrameCount % 4) {
    // hold last frame's inputs, skip computation
    return;
}
```

### Speed targeting

`racer->unk_1EC` — the CPU's internal target speed. Updated based on proximity to nearby racers:
- When trailing: approach speed (aggressive catch-up)
- When leading: gap speed (maintain lead without pulling too far ahead)

### Rubber-banding (VS mode)

Fully adjusts `unk_1EC` and `racer->unk_1E8` (a 0–1 extra-acceleration factor) based on gap to the player's `raceDistance`. The further behind, the more `unk_1E8` boosts effective acceleration beyond what the stat tables normally allow.

### Lateral position

`racer->unk_33C` = lateral offset target, read from the pre-baked script `D_i3_8013DBE8[]` at the CPU's current course position. The CPU steers toward this target each frame.

### Aggression / side attacks

If the CPU is within **184 units** of another racer (by `raceDistance`), a random roll is checked against `racer->unk_360` / `racer->unk_364`:
- Pass → sets `driftingCounter` (small lateral push) or presses `BTN_R_Z_COMBO` / `BTN_Z_R_COMBO` (full side-attack)
- Only active above NORMAL difficulty and when the player is ahead

## Per-Character Special Behaviors (`sCharacterBehaviorFuncs[]`, ~line 1143)

Most characters use `Cpu_BehaviorDoNothing` (~line 1115) — the core targeting handles them uniformly. A few have custom overrides:

| Character | Function | What it does |
|---|---|---|
| Billy | `Cpu_BehaviorBilly` | Copies the first player's `stickX` under one condition — subtle copy-cat steering |
| Mrs. Arrow | `Cpu_BehaviorMrsArrow` | No-op (placeholder) |
| Draq | `Cpu_BehaviorDraq` | No-op |
| John Tanaka | `Cpu_BehaviorJohnTanaka` | No-op |
| Michael Chain | `Cpu_BehaviorMichaelChain` | No-op |
| Mr. Ead | `Cpu_BehaviorMrEad` | No-op |

## Calling Chain

```
Racer_Update()                    // src/game/racer.c
  └─ Racer_UpdateMovement()
       ├─ Cpu_GenerateInputs()    // synthesize dummyController
       └─ Racer_UpdateFromControls(racer, dummyController)
```

## Difficulty Effects

| Setting | Effect |
|---|---|
| NORMAL | No aggression rolls, low rubber-banding noise |
| EXPERT | Moderate aggression, tighter performance factor range |
| MASTER | Random high-end `unk_1A8` values, max aggression thresholds |

## Related Files
- `include/unk_structs.h` — `Racer` struct (`unk_1E8`, `unk_1EC`, `unk_33C`, `awarenessFlags`)
- `src/game/racer.c` — `Racer_UpdateFromControls()` (CPU uses same physics)
- [SPEED.md](SPEED.md) — `unk_1A8` performance factor and how it drives speed stats
- [LAP_RACE.md](LAP_RACE.md) — `raceDistance` used for rubber-band and position ranking
