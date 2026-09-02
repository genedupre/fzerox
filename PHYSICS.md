# Physics

**Primary file:** `src/game/racer.c`

## Gravity

Each frame:

```c
// ~line 3704
racer->acceleration += -1.0 * racer->gravityUp;
```

`gravityUp` points "up" relative to the track surface normal — it follows the track in pipes and cylinders rather than being world-up. Flat sections use world-up; curved/enclosed sections rotate it to match geometry.

## Track Shape Collision Dispatch

The track is divided into segments, each with a `TRACK_SHAPE` type. Each frame, a handler is dispatched via a table:

```c
// src/game/racer.c ~line 507
D_800CF4B8[] = {
    func_8008EC38,  // ROAD            — flat open track
    func_8008EC58,  // WALLED_ROAD     — flat with side barriers
    func_8008EC98,  // PIPE            — enclosed cylindrical tube
    func_8008F550,  // CYLINDER        — half-open cylinder
    func_8008FC80,  // HALF_PIPE       — U-shaped half-pipe
    func_8008EC78,  // TUNNEL          — enclosed tunnel
    func_80090490,  // AIR             — no track beneath
    func_80090568,  // BORDERLESS_ROAD — no side walls
}
```

### Flat track (ROAD / WALLED_ROAD / TUNNEL)

All three call `func_8008E54C(racer, airborneRange)` (~line 2406) with different height thresholds (40 / 145 / 210 units). If the racer drifts past `currentRadiusLeft / currentRadiusRight`:
- Just outside → `Racer_ReceiveDamage(racer, 2.0f)` (edge scrape damage)
- Far outside → `RACER_STATE_AIRBORNE + FLAGS_80000000` (out-of-bounds fall)

### Cylinder track (`func_8008F550`, ~lines 2796–2973)

Full cylinder collision. Computes `heightAboveGround = radius - distanceFromCenter`, pushes racer back to surface, reflects inward velocity if it goes negative (bounce off inside wall).

### Half-pipe (`func_8008FC80`, ~lines 2975–3175)

Same logic as cylinder but for a U-shaped open-top track. Racer can fly off the top edge.

### Pipe / enclosed tube (`func_8008EC98`, ~lines 2580–2794)

`heightAboveGround = currentRadiusLeft - distanceFromSegmentCenter`. Pushes racer inward if it exceeds the pipe radius. The entire inner surface is valid ground — the game can drive upside-down in pipes.

## Side Rail Bumpers

`Racer_HitWall(Racer*, f32 lateralVelocity, f32 wallDirection, f32 edgeDisplacement)` (~line 2325):

1. Position snapped back inside the boundary
2. Lateral velocity component reflected: `racer->velocity -= racer->unk_234 * lateralVelocity * basis.z`
3. Damage applied: `Racer_ReceiveDamage(racer, lateralVelocity * wallDirection * 0.7f)`
4. `RACER_STATE_COLLISION_RECOIL` set

`racer->unk_234 = 1.7f` (wall elasticity coefficient, set in `Racer_InitRacer()`).

## Boost / Dash Pads

Detected via segment `Effect` list (~line 3610):

```c
if ((stateFlags & COURSE_EFFECT_MASK) == COURSE_EFFECT_DASH) {
    RACER_STATE_DASH_PAD_BOOST = true;
    boostTimer = sInitialBoostTimer; // 100 frames
}
```

Uses `racer->unk_1C4` (mid-rank boost multiplier) rather than the stat-based `unk_1C0`. See [BOOST.md](BOOST.md).

## Jump Pads

Box collision against `Jump` structs on the current `CourseSegment` (~lines 3656–3701):
- Sets `RACER_STATE_JUMP_BOOST`
- Assigns `racer->jumpBoost` (upward velocity impulse)
- `jumpBoost *= 0.8f` per frame (exponential decay)

## Landmines

Sphere proximity check, radius = 30 units (~lines 3633–3654):

```c
racer->acceleration += 15.0 * tiltUp;   // upward kick
Racer_ReceiveDamage(racer, 12.5f);
```

## Racer-to-Racer Collision

Handled in `Racer_Update()` (~lines 5099–5200). Pair-wise check for every racer pair using `sRacerPairInfo[]`:

- Distance threshold: **46 units** (tested as squared: 2116)
- `racer->collidingStrength` (set from body stat + weight in `Racer_InitRacer()`) determines who wins the bump — the lighter/weaker racer bounces more

## Related Files
- `include/fzx_course.h` — `TRACK_SHAPE_*` defines
- `include/unk_structs.h` — `Racer` struct, `CourseSegment`, `Jump`, `Landmine`
- `include/fzx_racer.h` — `RACER_STATE_*` flags, `CourseEffects` enum
- [ENERGY.md](ENERGY.md) — how damage from collisions affects the energy bar
- [BOOST.md](BOOST.md) — dash pad boost details
