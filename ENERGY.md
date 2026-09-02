# Health / Energy Bar

**Primary files:** `src/game/racer.c`, `src/overlays/ovl_i3/hud.c`

## Racer Struct Fields

Defined in `include/unk_structs.h`:

| Field | Type | Meaning |
|---|---|---|
| `energy` | `f32` | Current energy |
| `maxEnergy` | `f32` | Maximum energy (set by Body stat) |
| `energyIncrease` | `f32` | Energy added per frame on a pit strip |
| `energyRegain` | `f32` | One-time restore from collision energy transfer |
| `boostEnergyUsage` | `f32` | Energy consumed per frame while B-boosting |

## Initialization (`Racer_InitRacer()`, ~line 1660)

Body stat maps to max energy via:

```c
// src/game/racer.c
gBodyHealthValues[] = { 190.0, 178.0, 166.0, 154.0, 142.0 };
//                       A      B      C      D      E
maxEnergy = gBodyHealthValues[machineStats[BODY_STAT]];
energy    = maxEnergy;

boostEnergyUsage = maxEnergy * 0.0015f;
energyIncrease   = maxEnergy * 0.008f;   // pit refill rate per frame
```

Body-A machines have 190 max energy; Body-E machines have 142.

## Taking Damage (`Racer_ReceiveDamage()`, ~line 2227)

```c
racer->energy -= damageStrength;
// clamp to 0
```

If `energy <= 0`:
- `RACER_STATE_SPINNING_OUT` set
- `spinOutTimer = 1`
- `unk_234 = 2.0f` (elasticity briefly doubled — loose on impact)

If already in crashed state and `damageStrength > 25.0f` (`D_800CE770`):
- Permanent destruction: debris particle effects triggered, racer removed from race

## Pit Strip Refill (~line 4483)

```c
if ((stateFlags & COURSE_EFFECT_MASK) == COURSE_EFFECT_PIT) {
    racer->energy += racer->energyIncrease;  // += maxEnergy * 0.008f per frame
}
```

`pitForceFieldSize` ramps at `+0.1f` per frame for the visual shield effect.

`sEnergyRefillScale = 0.008f` is the constant used to derive `energyIncrease` at init time.

## HUD Rendering (`src/overlays/ovl_i3/hud.c`, `Hud_DrawEnergyBar()`, ~line 591)

```c
barWidth = Math_Round((energy / maxEnergy) * 68.0f * scale);
```

Color and screen position come from:
- `sEnergyBarFillColors[]` — bar color per layout
- `sEnergyBarPositions[]` — position for 1P/2P/3P/4P split-screen

## Low Energy Alerts (~lines 4731–4744)

| Threshold | Behavior |
|---|---|
| < 30% | Low-energy alert color, slow beep |
| < 20% | Faster beep |
| < 10% | Rapid beep |

## Related Files
- `include/unk_structs.h` — `Racer` struct
- `include/fzx_racer.h` — `RACER_STATE_SPINNING_OUT`, `COURSE_EFFECT_PIT`
- [BOOST.md](BOOST.md) — energy drain from B-button boost
- [PHYSICS.md](PHYSICS.md) — what triggers `Racer_ReceiveDamage()` (walls, mines, racer bumps)
