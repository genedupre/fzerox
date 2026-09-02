# Turning & Steering

**Primary file:** `src/game/racer.c`, inside `Racer_UpdateFromControls()`

## How Steering Works

Each frame, the analog stick X axis is normalized:

```c
// ~line 3475
sp12C = controller->stickX / 63.0f;  // range: -1.0 to 1.0
```

The turn is applied by bending the orientation matrix (`racer->trueBasis`) directly:

```c
// ~line 3515
racer->trueBasis.x -= var_fa1 * trueBasis.z;
// forward axis is then recomputed by cross product and re-normalized
```

This is equivalent to applying a yaw rotation in local space each tick. The amount rotated (`directionChange`) feeds back to reduce `accelerationForce` — turning costs speed.

## Turn Rate Selection (~lines 3475–3526)

`var_fa1` is selected each frame from one of several values:

| Condition | Turn Rate Used |
|---|---|
| Normal driving | `racer->unk_1E0` (full turn rate) |
| Holding A / boost / high speed | `racer->unk_1E4` (reduced turn rate) |
| Drifting (Z or R held) | Blended value between the two |
| Spin-out state | Overridden to `0.2f` (nearly locked) |

## Turn Rate Initialization

`Racer_InitRacer()` (~line 1638):

```c
unk_1E0 = (((machine->weight - 780.0f) * -0.005f) / 1560.0f) + 0.054f;
unk_1E4 = (((machine->weight - 780.0f) * -0.005f) / 1560.0f) + 0.030f;
```

**Heavier machine = lower turn rate.** Weight range across all machines is roughly 780–2340 kg.

Example weights from `sDefaultMachines[]`:
- Blue Falcon: 1260
- Black Shadow: 2340 (heaviest, worst turning)
- Pink Spider: ~780 (lightest, best turning)

## Drifting / Side Attack

Holding Z or R triggers the drift/strafe state. The lateral push is applied as an impulse to `racer->velocity` along `basis.z` (the right axis). A full side-attack (Z+R simultaneously) adds a large lateral force and sets `RACER_STATE_COLLISION_RECOIL` on the target.

## Speed Penalty from Turning

`directionChange` (line 3523) = magnitude of the yaw rotation applied this frame.

This value is subtracted from `accelerationForce` — sharp turns bleed speed. This is why tight corners require lifting off the accelerator or boosting through them.

## Related Files
- `include/unk_structs.h` — `Racer` struct (`unk_1E0`, `unk_1E4`, `trueBasis`)
- `include/fzx_racer.h` — `RACER_STATE_SPINNING_OUT`, `RACER_STATE_COLLISION_RECOIL`
- [SPEED.md](SPEED.md) — how `accelerationForce` and `directionChange` interact
- [PHYSICS.md](PHYSICS.md) — side rail collisions that also affect direction
