# F-Zero X Decompilation — Learning Guide

This repo is a ~97% complete decompilation of F-Zero X (N64). The goal here is **not** to ship a port — it's to understand how the game was actually programmed: how physics were tuned, how speed was calculated, how the AI cheats, and so on.

The decompiled C source is a direct translation of the original machine code, so variable names are often placeholder (`unk_1B0`, `sp12C`) but the logic is real and accurate.

## Reference Documents

All mechanic docs live in the [`docs/`](docs/) folder. Each one maps a specific system to exact source files, functions, and variables:

### [docs/SPEED.md](docs/SPEED.md) — Vehicle Speed & Acceleration
How speed is calculated from the velocity vector each frame, the two-phase acceleration model (low-speed vs high-speed), the hard top-speed cap (138.9 internal units ≈ 3000 km/h), and how the Body/Boost/Grip stat tables (`gBodyHealthValues`, `D_800CF174`, `D_800CF188`) feed into per-machine performance. Covers `Racer_InitMachineStats()` and the acceleration ramp inside `Racer_UpdateFromControls()`.

### [docs/TURNING.md](docs/TURNING.md) — Steering & Turn Rate
How analog stick input is normalized and applied as a local-space yaw rotation to the orientation matrix (`trueBasis`) each frame. Covers the turn rate formula (heavier machine = slower turn), the four turn-rate states (normal / boosting / drifting / spinning-out), and how sharp cornering bleeds speed via the `directionChange` penalty. Initialized in `Racer_InitRacer()`.

### [docs/PHYSICS.md](docs/PHYSICS.md) — Physics & Collision
The full physics picture: per-frame gravity applied along the track-relative up vector, the 8 track shape types (ROAD, PIPE, HALF_PIPE, CYLINDER, etc.) and their collision handler dispatch table, side rail bouncing and edge damage (`Racer_HitWall()`), boost/dash pad detection, jump pad impulse, landmine knockback, and pair-wise racer-to-racer collision (`collidingStrength` weight contest).

### [docs/ENERGY.md](docs/ENERGY.md) — Health / Energy Bar
How the energy bar is sized by the Body stat (`gBodyHealthValues[]`), drained by `Racer_ReceiveDamage()`, consumed per frame during boost, refilled on pit strips, and rendered on screen. Includes spin-out and destruction thresholds, low-energy alert tiers (<30%/<20%/<10%), and HUD bar width formula in `hud.c`.

### [docs/BOOST.md](docs/BOOST.md) — Boost & Boost Gauge
The B-button boost system: trigger conditions (`RACER_STATE_CAN_BOOST`, one token per lap), 100-frame countdown, energy drain per frame (`maxEnergy × 0.0015`), and how the boost stat rank (`D_800CF174[]`) determines the speed multiplier. Also covers dash pad boosts (no energy cost, mid-rank speed), the exhaust flame animation, and how CPU racers fake boost via `unk_1E8` without spending energy.

### [docs/LAP_RACE.md](docs/LAP_RACE.md) — Lap Counting & Race Logic
How lap crossings are detected from the sign of the `lapDistance` delta each frame, forward vs reverse crossing handling, lap time recording, `RACER_STATE_CAN_BOOST` restoration on lap complete, race finish logic (CPU take-over, energy refill), and position ranking via `raceDistance` sort in `Racer_UpdateRacePositions()`. Includes the full `Race_Init()` / `Race_Update()` call chain.

### [docs/AI.md](docs/AI.md) — CPU / AI Behavior
How CPU racers synthesize fake controller input each frame and feed it into the exact same physics as the player. Covers the pre-baked per-course path script (512 speed/lateral-offset pairs), target speed selection, rubber-band scaling via `unk_1E8`, side-attack aggression rolls (difficulty-gated), the 1-in-4-frame update budget, and per-character behavior overrides (most are no-ops; Billy copies the player's stick input).

## Key Source Files

| Source File | What's in it |
|---|---|
| `src/game/racer.c` | The main physics engine — speed, steering, gravity, collision, boost, energy, lap logic, position ranking. Almost everything lives here. |
| `src/overlays/ovl_i3/cpu.c` | AI input generation — how CPU racers decide speed targets, lateral position, and when to bump |
| `src/overlays/ovl_i2/race.c` | Race session lifecycle: `Race_Init()`, `Race_Update()` |
| `src/overlays/ovl_i3/hud.c` | HUD rendering: energy bar, speed readout, lap counter |
| `include/unk_structs.h` | The `Racer` struct (all 0x3A8 bytes of it), `CourseSegment`, `Jump`, `Landmine` |
| `include/fzx_racer.h` | `RACER_STATE_*` flags, `CourseEffects` enum, stat rank enums |
| `include/fzx_course.h` | Track shape defines (`ROAD`, `PIPE`, `HALF_PIPE`, etc.) |
| `include/macros.h` | `SPEED_CONVERSION` — 21.6 (NTSC) / 18.0 (PAL) — converts internal units to km/h |

## How to Navigate

- Internal speed units × `SPEED_CONVERSION` = km/h displayed on screen.
- The `Racer` struct is the central data structure. Nearly every mechanic reads from or writes to it.
- CPU racers run through the exact same `Racer_UpdateFromControls()` physics as the player — the AI just synthesizes a fake `Controller` input.
- Machine stats (Body, Boost, Grip) are 0–4 ranks (A=0 = best) indexing into small data tables in `racer.c`.
