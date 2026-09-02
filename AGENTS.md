# F-Zero X Decompilation — Learning Guide

This repo is a ~97% complete decompilation of F-Zero X (N64). The goal here is **not** to ship a port — it's to understand how the game was actually programmed: how physics were tuned, how speed was calculated, how the AI cheats, and so on.

The decompiled C source is a direct translation of the original machine code, so variable names are often placeholder (`unk_1B0`, `sp12C`) but the logic is real and accurate.

## Reference Documents

Each file below maps a specific game mechanic to the exact source files, functions, and variables responsible for it:

| File | Mechanic |
|---|---|
| [SPEED.md](SPEED.md) | Vehicle speed, acceleration, top speed, per-machine stats |
| [TURNING.md](TURNING.md) | Steering, turn rate, weight penalty, drift |
| [PHYSICS.md](PHYSICS.md) | Gravity, track collision shapes, side rails, boost/jump/mine pads, racer-to-racer collision |
| [ENERGY.md](ENERGY.md) | Health / energy bar, damage, spin-out, pit refill |
| [BOOST.md](BOOST.md) | B-button boost, dash pads, boost gauge, energy drain |
| [LAP_RACE.md](LAP_RACE.md) | Lap counting, finish line detection, race position ranking, race lifecycle |
| [AI.md](AI.md) | CPU racer behavior, rubber-banding, aggression, per-character quirks |

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
