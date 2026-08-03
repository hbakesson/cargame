# Hakan's Carpool

An endless highway driving game built with [Three.js](https://threejs.org/). One
self-contained HTML file — no build step, no bundler, no game engine, no physics
library.

Weave through traffic, bank into the corners, and see how many one-minute laps
you can string together before you clip someone. The road gets busier and the
traffic gets slower the longer you survive.

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
| `A` (or `←`) | Move one lane left |
| `D` (or `→`) | Move one lane right |
| `N` | Mute / unmute |
| `SPACE` | After a crash: continue the run |
| `R` | After a crash: start over |

Steering is lane-based: one tap moves you one lane and the car slides across
on its own. Tap twice quickly to cross two lanes. Holding the key does nothing
extra, and a stopped car can't change lanes at all.

Release everything and friction brings you to a stop on its own.

### Starting up

The title screen cruises along on its own while a C64-style tune plays. Because
browsers block audio until you interact with the page, the **first** keypress
wakes the music and the **next** one starts the drive — or click anywhere to
start the tune without leaving the title screen.

---

## Laps and difficulty

A lap lasts **one minute**. Six seconds before the timer runs out, a checkered
start/finish gantry is placed on the road exactly where you'll be when it
expires, so it approaches from the distance rather than materializing on top of
you. Crossing the line — not the clock hitting zero — is what ends the lap, so
slowing down never skips one; it just makes the lap take longer. The same
gantry then recedes as the next lap's start line.

Difficulty climbs from the first second and never stops:

```
difficulty = 1 - e^(-elapsed / 150)
```

Every traffic parameter is a blend between an easy end and a hard end:

| | Start | Late in a run |
| --- | --- | --- |
| Cars on the road | 5 | up to 14 |
| Traffic speed | 18–38 | 10–26 (bigger closing speeds) |
| Spawn distance | 230–680 | 140–430 (less warning) |
| Gap enforced around a spawn | 48 | 26 |

The one rule that never bends: a spawn is rejected if it would leave no open
lane. However bad it gets, there is always a way through.

## Scoring

- **Distance** is the base: `distance × 0.55`, multiplied by `1 + speedRatio`
  and again by `1 + difficulty` — so covering ground fast, deep into a run, is
  worth several times a cautious start.
- **Near miss**: +75 for passing a car one lane over without touching it, shown
  bottom-left. Each car can only pay out once.
- **Lap bonus**: `500 × the lap you just finished`, so laps compound.

## Crashing

Hitting a car drops the world into slow motion, shakes the camera, and after
1.3 seconds brings up the leaderboard: your score and completed laps, the top
eight runs, and two choices.

- **`SPACE` — continue this run.** Score, lap counter and difficulty all carry
  over; only the car is put back in the middle lane. The road ahead is
  re-rolled.
- **`R` — start over.** Fresh score, back to lap 1, difficulty back to zero.

The leaderboard lives in `localStorage` under `endlessDriveScores`. Each run
carries an id, so continuing updates that run's row rather than piling up a new
entry for every crash.

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
- **Steering** — A/D set `targetLane`; the car eases toward that lane center at
  `laneChangeRate`, scaled by a `smoothstep` on speed so a stopped car stays
  put. Yaw, body roll and the front-wheel angle are all derived from the
  *measured* sideways slide, so the car visibly leans into a change without any
  of it being driven by the key state.
- **Clamping** — `x` is bounded to the lanes plus half a shoulder, which only
  ever catches the extra drift a hard corner adds.

### Traffic

A fixed pool of 14 cars, each with a road distance, a lane and a speed. Only
the first `activeTrafficCount()` of them are visible; the rest sit out until
difficulty calls them in, at which point they respawn straight into play. Cars
that fall behind respawn ahead with fresh parameters, all blended along the
difficulty ramp by `difficultyMix()`.

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

### The music

Two tracks, both synthesized from scratch — no samples, no audio files. They
share one engine and are described entirely as data in `TRACKS`.

**Title screen** — the C64 action idiom, built from the tricks Rob Hubbard used
on the SID:

- A square-wave bass hammering octave-jumping eighth notes
- A chord **arpeggio cycled at 50 Hz** — one note per PAL frame, the SID's way
  of faking three simultaneous notes out of a single voice. It runs as one
  continuous oscillator whose frequency is rescheduled, exactly like the
  original, which also means it never clicks between steps.
- A vibrato'd sawtooth lead through a slapback echo, standing in for a fourth
  voice nobody had

Eight bars in A minor, `i – VI – VII – i / iv – VI – VII – V`, at 138 BPM.

**Driving** — the sunny Latin-fusion idiom Sega's OutRun ran on. An original
tune in that style, not a transcription of one of theirs:

- **Sixteen bars** of brisk major-key jazz harmony that never sits still —
  maj9 / m9 / dominant chords walking the circle of fifths, with an eight-bar
  lift into C for the B section before the turnaround home to F
- A **detuned twin-oscillator lead** — two sawtooths nine cents apart through
  the slapback echo, standing in for the FM brass every arcade board had. One
  oscillator alone is a bleep; the pair is a lead.
- A plucky filtered-sawtooth **slap bass**, syncopated off the downbeats and
  built from roots, fifths and octaves so one pattern works under every chord
- An **electric-piano comp** voiced two octaves up, hitting the anticipations
  rather than the beats
- **Latin percussion**: a 16th-note shaker under a 3-2 son clave played on a
  cowbell (two detuned squares in a clangy ratio through a bandpass — the
  drum-machine trick)

136 BPM, a 28-second loop, mixed at 0.15 so it sits under the engine rather
than over it. It fades out as you crash and cross-fades back in when you
continue.

A lookahead scheduler (`setInterval` decides *what*, WebAudio decides *when*)
keeps both in time regardless of frame rate.

To write your own, edit a track in `TRACKS`: `chords` are a root plus semitone
offsets, `bass`/`kick`/`snare`/`hat`/`perc` are indexed by sixteenth-note step,
and `lead` entries are `[step, MIDI note, length in steps]`. A track can be any
number of bars long — `lead` just needs one entry per chord.

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
| `laneChangeRate` | `6.5` | How briskly the car crosses to the next lane |
| `laneChangeMinSpeed` | `0.12` | Speed fraction needed for a full-rate lane change |
| `maxHeading` | `0.5` | Visual yaw cap while crossing, in radians |
| `curveDrift` | `0.35` | How hard corners push you wide within your lane |
| `attractSpeed` | `17` | Cruise speed behind the title screen |

### World and difficulty

| Key | Default | What it does |
| --- | --- | --- |
| `laneCount` / `laneWidth` | `3` / `4` | Road layout; both feed `LANE_X` automatically |
| `segmentLength` / `segmentCount` | `25` / `26` | Chunk size and view distance |
| `trafficStart` / `trafficMax` | `5` / `14` | Cars on the road at each end of the difficulty ramp |
| `trafficSpeedEasy` / `trafficSpeedHard` | `[18,38]` / `[10,26]` | Slower traffic means bigger closing speeds |
| `spawnEasy` / `spawnHard` | `[230,680]` / `[140,430]` | How far ahead traffic appears |
| `spawnGapEasy` / `spawnGapHard` | `48` / `26` | Clear road demanded around a spawn |
| `difficultyRamp` | `150` | Seconds; lower makes the game get hard faster |
| `lapDuration` | `60` | Seconds per lap |
| `finishPreview` | `6` | How early the finish gantry is placed on the road |
| `lapBonus` | `500` | Multiplied by the lap just finished |
| `pointsPerUnit` | `0.55` | Base scoring rate |
| `nearMissDist` / `nearMissBonus` | `4.6` / `75` | Near-miss window and payout |

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

For the design reasoning behind all of this — why the car never moves, how the
curve trick works, where to hook new things in — see
[ARCHITECTURE.md](ARCHITECTURE.md).

All in `index.html`, in roughly this order:

| Function | Responsibility |
| --- | --- |
| `init()` | Renderer, scene, fog, camera, then everything below |
| `createLights()` | Hemisphere ambient + directional sun with a fixed shadow camera |
| `createRoad()` | Grass plane and the segment pool |
| `createRoadSegment()` | One chunk: asphalt, shoulders, edge lines, dashes, prop pool |
| `randomizeSegment()` | Re-rolls a chunk's scenery — transforms only, no allocation |
| `buildCar()` | Shared car builder; `simple` mode drops glass and rims for traffic |
| `createCar()` / `createTraffic()` | The player's car and the 14-car traffic pool |
| `respawnTraffic()` / `isSpotFree()` | Traffic placement, with the always-open-lane rule |
| `difficultyMix()` / `activeTrafficCount()` | Blend every traffic parameter along the difficulty ramp |
| `createFinishLine()` / `updateLaps()` / `completeLap()` | The start/finish gantry and lap timing |
| `recordRun()` / `renderLeaderboard()` | Persisted top-eight runs |
| `createSpeedLines()` | The streak buffer |
| `updatePhysics()` | Throttle, brake, steering, inertia, scoring |
| `updateWorld()` | Lays the road and traffic out along the curve |
| `updateTraffic()` / `checkCollisions()` | Traffic motion, crashes, near misses |
| `updateCamera()` | Chase camera, lateral lag, shake, FOV stretch |
| `initAudio()` / `updateAudio()` | The WebAudio engine |
| `startMusic()` / `playTrack()` / `scheduleMusic()` / `scheduleStep()` | The two chiptunes and their scheduler |
| `wakeAudio()` | Unlocks audio from the first gesture |
| `startGame()` / `resetRun()` / `resumeDriving()` | Starting a run, and putting the car back on the road |
| `crash()` / `continueRun()` / `restart()` / `updateCrashSequence()` | Game state machine |
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
  `checkCollisions()`. The finish gantry is the same pattern with a single
  instance.
- **Day/night** — animate `scene.background`, `scene.fog.color` and the sun's
  colour against `game.distance`.
- **Tighter or looser corners** — add a third entry to `CURVE`, or scale the
  existing amplitudes.
- **A different difficulty curve** — everything funnels through `game.difficulty`
  in `updateLaps()`. Swap the exponential for a per-lap step, or drive it from
  `game.lap` instead of elapsed time, and the whole game follows.
- **Your own tune** — a `TRACKS` entry is plain data. Add a third track and call
  `playTrack('name')` wherever it belongs.

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
- Because the finish line is placed from your speed a few seconds out, a big
  speed change inside that window shifts when you cross it. A lap is "about a
  minute", not a stopwatch.
- The leaderboard is per-browser `localStorage` — nothing is shared between
  machines, and clearing site data wipes it.
- There's no pause, and no mobile/touch input.

## License

Do whatever you like with it.
