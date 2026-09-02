# Speed & Acceleration

**Primary file:** `src/game/racer.c`

## How Speed Works

Speed is computed each frame as the magnitude of the velocity vector:

```c
// src/game/racer.c ~line 4158
racer->speed = sqrtf(SQ(velocity.x) + SQ(velocity.y) + SQ(velocity.z));
```

Internal units × `SPEED_CONVERSION` = km/h on screen.
- NTSC: `SPEED_CONVERSION = 21.6` (defined in `include/macros.h`)
- PAL: `SPEED_CONVERSION = 18.0`

Hard speed cap: **138.9 internal units** (~3000 km/h display) enforced around line 4214–4220.

## Acceleration Model

The racer has two acceleration phases based on whether it's below or above a speed threshold (`racer->unk_1BC`):

- Below threshold → uses `racer->unk_1B4` (stronger push, low-speed ramp-up)
- Above threshold → uses `racer->unk_1B0` (gentler push, high-speed sustain)

The selected value is then scaled by the boost multiplier. The final `accelerationForce` ramps toward the target exponentially using `unk_1D0` and `unk_328` each frame — it never snaps instantly.

Turning and drifting subtract from effective acceleration via a `directionChange` penalty (~line 4123).

## Per-Machine Stat Initialization

`Racer_InitMachineStats(Racer* racer, f32 arg1)` (~line 1419) reads encoded ROM performance values and interpolates them by `racer->unk_1A8` (a 0–1 "performance factor"). This populates:

| Field | Meaning |
|---|---|
| `unk_1B0` | High-speed acceleration multiplier |
| `unk_1B4` | Low-speed acceleration multiplier |
| `unk_1B8` | Drag / deceleration base |
| `unk_1BC` | Speed threshold (where high-speed phase kicks in) |
| `unk_1C0` | B-button boost multiplier (scales with Boost stat) |
| `unk_1C4` | Dash-pad boost multiplier (always mid-rank boost) |
| `unk_1C8/CC/D0` | Additional acceleration curve params |

## Boost Stat Data Table

```c
// src/game/racer.c
D_800CF174[] = { 0.210, 0.207, 0.204, 0.201, 0.198 };
//               A      B      C      D      E
// Boost-A gives the highest acceleration bonus.
```

`unk_1C0 = (D_800CF174[BOOST_STAT] + tune) / unk_1B8`

## Grip Stat (Speed Caps)

```c
D_800CF188[] = { 1.277, 1.237, 1.197, 1.157, 1.117 }; // high-speed cap (unk_1F8)
D_800CF19C[] = { 0.397, 0.367, 0.337, 0.307, 0.277 }; // low-speed cap  (unk_1FC)
//               A      B      C      D      E
```

Grip-A = highest caps (harder to spin out at speed, better cornering ceiling).

## Machine Stat Assignments

`sDefaultMachines[]` (~line 457): all 30 characters + 3 super machines, each with:
```c
{ BODY_STAT, BOOST_STAT, GRIP_STAT }   // 0=A (best) … 4=E (worst)
```

## Related Files
- `include/macros.h` — `SPEED_CONVERSION`
- `include/fzx_racer.h` — stat rank enums
- `include/unk_structs.h` — `Racer` struct field offsets
