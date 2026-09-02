# Boost & Boost Gauge

**Primary file:** `src/game/racer.c`

## Triggering a Boost (~lines 3598–3604)

```c
if ((buttonPressed & BTN_B)
    && (boostTimer == 0)
    && (energy != 0.0)
    && (stateFlags & RACER_STATE_CAN_BOOST)) {
    boostTimer = sInitialBoostTimer;  // 100 frames
    // set RACER_SE_FLAGS_BOOST (engine sound)
}
```

`RACER_STATE_CAN_BOOST` is granted at race start and restored after each lap completion (~line 3314). One boost per lap maximum.

## Countdown & Energy Drain (~lines 4035–4072)

Each frame while `boostTimer > 0` (and it's a player boost, not a dash pad):

```c
energy -= boostEnergyUsage;   // = maxEnergy * 0.0015f per frame
boostTimer--;

if (energy < 0) {
    boostTimer = 1;  // force end next frame
    energy = 0;
}
// when boostTimer reaches 0: RACER_STATE_DASH_PAD_BOOST and sound flags cleared
```

A full 100-frame boost drains `100 × maxEnergy × 0.0015 = 15%` of max energy.

## Speed Effect

While `boostTimer > 0`, the speed multiplier `var_ft4` is forced to at least:

| Source | Multiplier field | How it's set |
|---|---|---|
| B-button boost | `racer->unk_1C0` | `(D_800CF174[BOOST_STAT] + tune) / unk_1B8` |
| Dash pad | `racer->unk_1C4` | Always mid-rank boost (ignores machine Boost stat) |

```c
// Boost stat acceleration bonus table:
D_800CF174[] = { 0.210, 0.207, 0.204, 0.201, 0.198 };
//               A      B      C      D      E
// Boost-A gives the highest multiplier → fastest boost.
```

## Dash Pads vs. B-Button

| Property | B-Button boost | Dash pad |
|---|---|---|
| Duration | 100 frames | 100 frames |
| Energy cost | Yes (`boostEnergyUsage` per frame) | No |
| Speed multiplier field | `unk_1C0` (stat-based) | `unk_1C4` (always mid-rank) |
| Grants per lap | 1 | Unlimited |
| Requires `CAN_BOOST` flag | Yes | No |

## Visual Feedback

`src/overlays/ovl_i3/hud.c` ~line 6044:

```c
temp_fs0 = (f32) racer->boostTimer / sInitialBoostTimer;
// used to animate exhaust flame intensity / scale
```

## CPU Racers & Boost

CPU racers use `racer->unk_1E8` (a 0–1 rubber-band factor) to simulate extra acceleration without ever pressing B or consuming energy. They only legitimately trigger a B-boost if the AI input generator explicitly presses `BTN_B`, which it rarely does.

## Related Files
- `include/fzx_racer.h` — `RACER_STATE_CAN_BOOST`, `RACER_STATE_DASH_PAD_BOOST`
- `include/unk_structs.h` — `boostTimer`, `boostEnergyUsage` fields
- [SPEED.md](SPEED.md) — how `unk_1C0`/`unk_1C4` fit into the acceleration model
- [ENERGY.md](ENERGY.md) — energy drain and depletion consequences
- [PHYSICS.md](PHYSICS.md) — dash pad detection on course segments
