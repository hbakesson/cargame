# Architecture

How *Hakan's Carpool* is put together, and why it's put together that way.

This document is about structure and reasoning. For controls, tuning knobs and
a function index, see [README.md](README.md).

---

## Constraints that shaped everything

The game is a single `index.html` with no build step, no bundler, and no
dependency beyond Three.js loaded from a CDN import map. That's a deliberate
constraint, and it drives most of what follows:

- **No module system.** One ES module in one `<script>` tag. Sections are
  delimited by comment banners rather than files.
- **No state library.** Two plain objects, `physics` and `game`, hold everything
  mutable. Anything that reads them reads them directly.
- **No physics engine.** ~60 lines of arithmetic in `updatePhysics()`.
- **No asset pipeline.** Every mesh is a Three.js primitive; the only textures
  are drawn into a `<canvas>` at startup; the music is synthesized from
  oscillators. Nothing is fetched but the library itself.

The result is a file you can open from disk and read top to bottom.

---

## The central decision: the car never moves

Everything else follows from this.

The player's car sits at the world origin forever. `car.position.z` is always
`0`; `car.position.x` only moves between lanes. Driving forward advances a
single scalar:

```js
game.distance += forward;
```

The world is then rebuilt around that number every frame. Road segments,
traffic and the finish gantry all store a **road distance** and are positioned
relative to `game.distance` in `updateWorld()`.

### Why

**Stable shadows.** One directional light with a fixed 110×110 shadow camera
centered on the origin covers the play area for the entire session. If the car
moved, the shadow camera would have to follow it, and every shadow would swim
as its frustum slid.

**No floating-point drift.** Ten minutes at top speed is ~36,000 units. Single
precision positions at that magnitude quantize visibly — the car would judder.
Here nothing is ever more than a few hundred units from the origin.

**Collision math stops needing matrices.** Both the player and every traffic car
live in a flat 2D space: lateral `x` and road distance. A collision is:

```js
Math.abs(dx) < CAR_HALF_W * 2 && Math.abs(dz) < CAR_HALF_L * 2
```

No world transforms, no bounding volumes, no traversal. The 3D scene graph is
purely a *rendering* of the 2D game state — it is never read back.

### The cost

Two coordinate systems exist, and you have to keep them straight:

| Space | Used by | Units |
| --- | --- | --- |
| **Road-local** | all game logic | lateral `x`, road distance |
| **World** | rendering only | Three.js scene coordinates |

`updateWorld()` is the single translation layer between them. Nothing else
converts. If you add a new game object, give it a `roadDist` and a lane and
position it there — don't reach for `object.position` anywhere else.

---

## Frame flow

```
animate()
  ├─ dt = min(clock.getDelta(), 0.05) × game.timeScale
  │
  ├─ updateLaps(dt)        elapsed, difficulty, arm/cross the finish line
  ├─ updatePhysics(dt)     throttle, lane easing, scoring, → game.distance
  ├─ updateTraffic(dt)     traffic motion, activation, recycling
  ├─ updateWorld()         road-local → world, for everything
  ├─ checkCollisions()     crashes and near misses
  │
  ├─ updateCamera(rawDt)   chase, lag, shake, FOV — always at real time
  ├─ updateAudio()         engine pitch and volume
  └─ renderer.render()
```

Two details in that ordering matter:

**`updateWorld()` runs after everything that can move something.** It is the
only place scene-graph transforms are written, so it must see the final state
of the frame. Put new game logic *before* it.

**The camera uses `rawDt`, not `dt`.** During a crash `game.timeScale` drops to
0.16, and the world goes into slow motion — but the camera should keep
following and shaking at real time, or the impact feels mushy instead of
violent.

`dt` is clamped to 50 ms so that an alt-tab pause can't teleport the car
through traffic on the first frame back.

---

## The world: recycling pools

Nothing is created or destroyed while driving. Three pools cover the world:

| Pool | Size | Recycles when |
| --- | --- | --- |
| Road segments | 26 × 25 units | more than `behindCamera` behind, or a full road length ahead |
| Traffic | 14 cars | passed, or far enough ahead that the player must have reversed |
| Speed-line streaks | 70 | swept past the camera |

A road segment owns its asphalt, shoulders, edge lines, dashes, and a **fixed
pool of scenery** — 4 trees, 2 rocks, 2 grass patches, 1 sign, built once in
`createRoadSegment()`. Recycling calls `randomizeSegment()`, which only rewrites
transforms and toggles `visible`. At top speed a segment recycles roughly every
0.4 seconds, so allocating there would mean constant garbage.

Traffic recycles in **both** directions because the player can reverse.
Segments do too — hence the two-sided wrap in `updateWorld()`.

---

## Curves without curved geometry

The road bends, but no geometry is ever deformed. The centerline is a sum of
sine waves of distance:

```js
const CURVE = [
  { amp: 55, freq: 0.0016,  phase: 0   },
  { amp: 90, freq: 0.00047, phase: 1.7 },
];
```

Three derived functions do all the work:

| Function | Meaning | Used for |
| --- | --- | --- |
| `curveX(d)` | where the centerline sits | placing segments and objects |
| `curveSlope(d)` | its heading (1st derivative) | yawing them to follow it |
| `curveBend(d)` | how tight the corner is (2nd derivative) | centrifugal drift |

Each segment is placed at `curveX(d) - curveX(player)` and yawed by
`-atan(curveSlope(d))`. Then the parent `worldGroup` is counter-rotated by
`+atan(curveSlope(player))`.

That counter-rotation is the trick. It cancels the yaw **exactly at the car**,
so the road underfoot always runs dead straight down `-Z` while the road ahead
sweeps left and right. The player's lane coordinates stay valid — a lane center
is a lane center regardless of where you are on the curve — which is what keeps
the collision math flat.

Corners aren't only cosmetic: `curveBend` feeds a term that pushes the car
toward the outside of the bend, so you hold a line through them rather than
watching them go by.

**Where this approximates.** Segments are placed along the tangent at their own
center, not along a true arc. The mismatch at a joint scales with
`curvature × length²`, which is why segments are 25 units rather than 40, and
why the asphalt is drawn 4% overlong. Much tighter curves than the shipped
amplitudes would show seams.

---

## Game state

A four-state machine on `game.state`:

```
title ──any key──▶ driving ──hit a car──▶ crashing ──1.3s──▶ over
                      ▲                                        │
                      │                          SPACE (continue)
                      └────────────────────────── R (restart) ──┘
```

- **`title`** — attract mode. The car cruises at `attractSpeed`, traffic is
  hidden, the lap timer and difficulty are frozen at zero, and the chiptune
  plays.
- **`driving`** — the only state where input, scoring, difficulty and collisions
  are live.
- **`crashing`** — input ignored, `timeScale` eases to 0.16, the camera shakes,
  a red vignette fades in.
- **`over`** — the loop still renders but skips all simulation; the leaderboard
  is shown.

Two functions handle re-entry, and the split matters:

- `resumeDriving()` puts the car back on the road and leaves run state alone.
- `resetRun()` clears score, lap and difficulty, then calls `resumeDriving()`.

`continueRun()` uses the first, `restart()` and `startGame()` use the second.
`game.distance` is never reset by either — the road, traffic and gantry are all
positioned relative to it, so zeroing it would strand them mid-flight.

---

## Difficulty as a single scalar

One number, climbing forever, asymptotic to 1:

```js
game.difficulty = 1 - Math.exp(-game.elapsed / CONFIG.difficultyRamp);
```

Every traffic parameter is a named easy/hard pair in `CONFIG`, and
`difficultyMix(easy, hard)` interpolates between them at read time. Nothing
caches a difficulty-derived value, so the curve can be changed in one place and
the whole game follows.

The invariant that survives every difficulty level lives in `isSpotFree()`: a
spawn is rejected if it would leave no open lane across that stretch of road.
Difficulty makes the game harder by shrinking reaction time and increasing
closing speed — never by making it impossible.

---

## Laps: the line is authoritative, not the clock

A lap is nominally 60 seconds, but a pure timer would be at odds with having a
visible finish line — the line's position isn't known until you know where the
car will be.

So the line is placed *predictively*, `finishPreview` seconds out:

```js
finishLine.roadDist = game.distance + Math.max(70, physics.currentSpeed * remaining);
```

and the lap completes on **crossing it**, not on the timer expiring. At a steady
pace you cross at almost exactly 60 seconds. Brake hard and the line waits for
you; the lap simply takes longer. A stopped car never finishes a lap, which is
the correct outcome.

One gantry object serves both roles: armed, it's the finish line; crossed, it
just keeps receding as the next lap's start line.

---

## Rendering budget

Roughly 500 draw calls, kept there deliberately:

- **Traffic uses a `simple` variant** of the shared `buildCar()` — no glass, no
  rims, 10-sided tires instead of 18.
- **Scenery casts shadows only within ~90 units**, toggled per frame in
  `updateWorld()`. The shadow pass sees a fraction of the props.
- **Speed lines are one `LineSegments` draw call**, and the position buffer is
  skipped entirely below 50% speed.
- **Materials and geometry are module-level singletons** (`MATERIALS`,
  `GEOMETRY`), so every tree in the world shares one cone geometry.
- **Pixel ratio is capped at 2**, so 3× displays don't render 9× the pixels.

`segmentCount` is the biggest single lever if you need headroom — it maps
almost linearly to draw calls.

---

## Audio

Two independent WebAudio graphs share one `AudioContext`:

**The engine** — a sawtooth plus a sub-octave square through a resonant lowpass.
Pitch comes from speed folded into five fake gears, so it climbs and drops.
Filter cutoff opens with speed and throttle. Only three long-lived nodes; the
per-frame work is `setTargetAtTime` calls.

**The title tune** — one-shot oscillators per note, scheduled ahead of time.
The arpeggio is the exception: a single continuous oscillator whose *frequency*
is rescheduled at 50 Hz, which is both how the SID actually faked a chord from
one voice and the reason it never clicks between steps.

Timing uses the standard lookahead pattern: a 25 ms `setInterval` decides
*what* to play and hands WebAudio precise timestamps for *when*. The audio clock
is never the frame clock, so the tune doesn't stutter when the renderer does.

**The unlock dance.** Browsers refuse to start audio before a user gesture,
which collides with wanting music on the title screen. The compromise:
`init()` optimistically tries `ctx.resume()`; if that's refused, the first
keypress calls `wakeAudio()`, which starts the tune and *doesn't* start the
game. The next key drives. If WebAudio is unavailable entirely, `wakeAudio()`
returns false and the first key drives immediately — no dead end.

---

## Where to add things

| You want to add | Touch |
| --- | --- |
| Anything on the road (power-up, obstacle, sign) | give it a `roadDist` + lane, position it in `updateWorld()`, test it in `checkCollisions()` |
| A new scenery prop | a `makeX()` builder, an entry in the segment pool, placement in `randomizeSegment()` |
| A new difficulty lever | an easy/hard pair in `CONFIG`, read through `difficultyMix()` |
| A new game state | a branch in `animate()` and in `updatePhysics()`'s input gate |
| New HUD | an element in `ui`, and a change-guarded write (never `innerHTML` per frame) |
| A different tune | `CHORDS`, `BASS`, `LEAD` — plain data |

Two rules worth keeping:

1. **Game logic runs in road-local coordinates.** `updateWorld()` is the only
   place that converts to world space.
2. **The render loop allocates nothing.** Pool it, or hoist it to module scope.
