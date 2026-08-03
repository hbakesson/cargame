# Hakan's Carpool

An endless highway driving game built with [Three.js](https://threejs.org/). One
self-contained HTML file — no build step, no bundler, no game engine, no physics
library.

Weave through traffic, bank into the corners, and see how far you get before you
clip someone.

---

## Running it

```bash
open index.html          # macOS
xdg-open index.html      # Linux
```

That's it. Three.js is pulled from unpkg through an [import map](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script/type/importmap),
so the page needs an internet connection on first load but **not** a local
server.

If you'd rather serve it (or work offline), any static server works:

```bash
python3 -m http.server 8000    # then visit http://localhost:8000
```

To vendor Three.js for fully offline play, download
`https://unpkg.com/three@0.160.0/build/three.module.js` next to `index.html` and
point the import map at `./three.module.js`.

**Browser support:** anything with WebGL 2 and import maps — Chrome 89+,
Firefox 108+, Safari 16.4+.

---

## Controls

| Key | Action |
| --- | --- |
| `SPACE` | Accelerate |
| `M` (or `↓`) | Brake, then reverse |
| `A` (or `←`) | Steer left |
| `D` (or `→`) | Steer right |
| `N` | Mute / unmute |
| `SPACE` / `R` | Restart after a crash |

Release everything and friction brings you to a stop on its own.

---

## Scoring

- **Distance** is the base: `distance × 0.55`, multiplied by `1 + speedRatio` —
  so covering ground fast is worth up to twice as much as crawling.
- **Near miss**: +75 for squeezing past a traffic car within `3.4` units without
  touching it. Each car can only pay out once.
- **Best score** persists in `localStorage` under `endlessDriveBest`.

Hitting a traffic car ends the run: the world drops into slow motion, the camera
shakes, and the game-over card appears after 1.3 seconds.

---

## How it works

### The car never moves

The player's car sits at the world origin permanently. Instead of driving the
car forward, the game advances a single scalar — `game.distance` — and rebuilds
the world around it every frame.

This buys three things:

1. **Stable shadows.** One fixed directional-light shadow camera covers the play
   area for the entire session; nothing has to follow the player.
2. **No floating-point drift.** After ten minutes at top speed a moving car
   would be ~36,000 units from the origin and visibly juddering. Here every
   coordinate stays within a few hundred units.
3. **Trivial collision math.** Everything lives in flat road-local coordinates:
   lateral `x` and road distance. Collisions are a 2D AABB test, no matrices
   involved.

### The road

`roadSegments` is a pool of 26 chunks, 25 units each. Every frame `updateWorld()`
positions each one relative to `game.distance`; a chunk that falls more than
`behindCamera` units behind wraps to the far end and re-randomizes its scenery.
The pool never shrinks or grows, and recycling allocates nothing — props are
built once and only have their transforms rewritten.

### Curves

The road centerline is a sum of two sine waves of distance:

```js
const CURVE = [
  { amp: 55, freq: 0.0016,  phase: 0   },
  { amp: 90, freq: 0.00047, phase: 1.7 },
];
```

- `curveX(d)` — where the centerline sits
- `curveSlope(d)` — its heading (first derivative)
- `curveBend(d)` — how tight the corner is (second derivative)

Each segment is placed at `curveX(d) - curveX(player)` and yawed by
`-atan(curveSlope(d))`. Then `worldGroup` is counter-rotated by
`+atan(curveSlope(player))`, which cancels the yaw exactly at the car — so the
road underfoot always runs dead straight while the road ahead sweeps away.

Corners aren't just decoration: `curveBend` feeds a centrifugal term that pushes
the car toward the outside of the bend, so you have to hold a line through them.

### Physics

Arcade, not simulation. Everything is delta-time driven, with `dt` clamped to
50 ms so an alt-tab pause can't teleport you.

- **Longitudinal** — throttle adds `acceleration`; coasting subtracts constant
  rolling `friction` plus `airDrag × speed`; braking subtracts `brakeForce` down
  through zero into a slow reverse.
- **Steering** — input builds a `heading` angle rather than moving the car
  directly. Authority is gated by a `smoothstep` on speed, so a stopped car
  cannot turn at all.
- **Lateral inertia** — heading sets a *target* sideways velocity that the car
  eases into at `gripLag`. That's what gives lane changes weight instead of
  making them feel like dragging a slider.
- **Clamping** — `x` is bounded to the lanes plus half a shoulder; hitting the
  limit also bleeds off heading so the car straightens rather than grinding
  along the edge.

### Traffic

Nine cars, each with a road distance, a lane, and a speed between 13 and 34
units/s. When one falls behind it respawns 170–640 units ahead.

`isSpotFree()` enforces the one rule that keeps the game fair: a spawn is
rejected if it would leave **no** open lane across that stretch of road. There
is always a way through.

### Sense of speed

- Roadside trees, rocks and signs streaming past
- FOV widening by 10° toward top speed
- 70 white streaks in a single `LineSegments` draw call, fading in above 50%
  speed
- The chase camera trails your lateral position, lagging harder the faster you
  go, so hard steering swings the view
- A WebAudio engine: sawtooth + sub-octave square through a lowpass, with five
  fake gears so the pitch climbs and drops as you accelerate

Audio can only start from a user gesture, so it initializes on your first
keypress.

---

## Tuning

Every number worth touching is in the `CONFIG` object at the top of the module.

### Feel

| Key | Default | What it does |
| --- | --- | --- |
| `maxSpeed` | `62` | Top speed in units/s (~223 km/h) |
| `acceleration` | `17` | Throttle response |
| `friction` | `8` | Constant coasting deceleration |
| `airDrag` | `0.16` | Extra deceleration proportional to speed |
| `brakeForce` | `34` | Braking strength |
| `maxReverseSpeed` | `9` | Reverse speed cap |
| `steerRate` | `3.9` | How fast heading builds — raise for twitchier turns |
| `steerReturn` | `4.4` | How fast the car self-centers |
| `maxHeading` | `0.62` | Sharpest angle the car can hold, in radians |
| `steerMinSpeed` | `0.03` | Speed fraction below which steering barely bites |
| `steerHighSpeedFalloff` | `0` | Raise toward `0.45` to calm the car at top speed |
| `gripLag` | `11` | Lateral response — lower slides more, higher snaps |
| `curveDrift` | `1.0` | How hard corners throw you outward |

### World and difficulty

| Key | Default | What it does |
| --- | --- | --- |
| `laneCount` / `laneWidth` | `3` / `4` | Road layout; both feed `LANE_X` automatically |
| `segmentLength` / `segmentCount` | `25` / `26` | Chunk size and view distance |
| `trafficCount` | `9` | Cars on the road — the main difficulty dial |
| `trafficMinSpeed` / `trafficMaxSpeed` | `13` / `34` | Slower traffic means bigger closing speeds |
| `spawnMin` / `spawnMax` | `170` / `640` | How far ahead traffic appears |
| `pointsPerUnit` | `0.55` | Base scoring rate |
| `nearMissDist` / `nearMissBonus` | `3.4` / `75` | Near-miss window and payout |

### Camera and look

| Key | Default | What it does |
| --- | --- | --- |
| `camHeight` / `camDistance` | `4.2` / `9.6` | Chase camera offset |
| `camLerp` | `4.5` | Follow smoothing — higher is snappier |
| `camLateralLag` | `2.2` | Lower makes the camera swing more through turns |
| `baseFov` | `62` | Field of view at a standstill |
| `skyColor` | `0x8ec5e8` | Background and fog colour (they're deliberately the same) |
| `fogNear` / `fogFar` | `130` / `430` | Where the recycled road fades out |

---

## Code map

All in `index.html`, in roughly this order:

| Function | Responsibility |
| --- | --- |
| `init()` | Renderer, scene, fog, camera, then everything below |
| `createLights()` | Hemisphere ambient + directional sun with a fixed shadow camera |
| `createRoad()` | Grass plane and the segment pool |
| `createRoadSegment()` | One chunk: asphalt, shoulders, edge lines, dashes, prop pool |
| `randomizeSegment()` | Re-rolls a chunk's scenery — transforms only, no allocation |
| `buildCar()` | Shared car builder; `simple` mode drops glass and rims for traffic |
| `createCar()` / `createTraffic()` | The player's car and the nine AI cars |
| `respawnTraffic()` / `isSpotFree()` | Traffic placement, with the always-open-lane rule |
| `createSpeedLines()` | The streak buffer |
| `updatePhysics()` | Throttle, brake, steering, inertia, scoring |
| `updateWorld()` | Lays the road and traffic out along the curve |
| `updateTraffic()` / `checkCollisions()` | Traffic motion, crashes, near misses |
| `updateCamera()` | Chase camera, lateral lag, shake, FOV stretch |
| `initAudio()` / `updateAudio()` | The WebAudio engine |
| `crash()` / `restart()` / `updateCrashSequence()` | Game state machine |
| `animate()` | The `requestAnimationFrame` loop |

---

## Extending it

A few things the structure is already set up for:

- **More lanes** — change `laneCount`; lane centers, dashes and traffic spawning
  all derive from it.
- **New scenery** — add a builder like `makeRock()`, put instances in the pool
  inside `createRoadSegment()`, and place them in `randomizeSegment()`.
- **Power-ups** — traffic already demonstrates the pattern: an object with a
  `roadDist` and a lane, positioned in `updateWorld()` and tested in
  `checkCollisions()`.
- **Day/night** — animate `scene.background`, `scene.fog.color` and the sun's
  colour against `game.distance`.
- **Tighter or looser corners** — add a third entry to `CURVE`, or scale the
  existing amplitudes.

---

## Performance notes

- The prop pool means zero allocation while driving; segment recycling only
  rewrites transforms.
- Scenery casts shadows only within ~90 units of the car (`updateWorld()`), which
  keeps the shadow pass small.
- Speed lines are one draw call, and the position buffer is skipped entirely
  below 50% speed.
- Traffic cars use `simple` geometry — no glass, no rims, 10-sided tires.
- Pixel ratio is capped at 2, so 3× phone displays don't render 9× the pixels.

If you need more headroom, `segmentCount` is the biggest single lever — it maps
directly to draw calls.

---

## Known caveats

- The curved-road geometry is a first-order approximation: segments are placed
  along the tangent at their center, so extremely tight corners would show
  seams at the joints. The asphalt is drawn 4% overlong to hide this, and the
  shipped curve amplitudes stay well within tolerance.
- Traffic cars drive in a fixed lane and never react to you or to each other —
  they're obstacles, not drivers.
- There's no pause, and no mobile/touch input.

## License

Do whatever you like with it.
