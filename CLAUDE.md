# Zombie Sword — Project Guide for Claude

## Live URL
**https://danik75.github.io/zombie-sword/**
Deployed via GitHub Pages (main branch root). Any `git push` to `main` auto-redeploys within ~60s.

## What this is
Browser-based first-person DOOM-style zombie survival game. Single `index.html` (pure HTML5 Canvas + vanilla JS, no npm deps). Node.js HTTP server (`index.js`) serves it on port 3000.

```
node index.js          # start server
open http://localhost:3000
# Kill port if busy:  lsof -ti:3000 | xargs kill -9
```

Syntax-check JS after every edit:
```
node -e "const fs=require('fs'); new Function(fs.readFileSync('index.html','utf8').match(/<script>([\s\S]*)<\/script>/)[1])"
```

---

## Architecture

Everything lives in `index.html` (~1800 lines). Key sections top-to-bottom:

| Section | What it does |
|---|---|
| `const WORLDS = [...]` | 10 world configs (colors, maps, torch, fog) |
| `solid()` | Wall collision using `MAP[y][x]` |
| `const SWORD_*` | Sword name/color/reach/arc arrays (5 levels) |
| Audio functions | Web Audio API synth — no external files |
| `init(wIdx)` | Reset game state, load selected world |
| `update(dt)` | Game logic tick (movement, zombies, timers) |
| `castRays()` | DDA raycaster — 600 strips, 2px each, FOV 66° |
| `drawSprites()` | Billboarded zombie sprites with 3D gradients |
| `drawGrail()` | Billboarded golden grail sprite |
| `drawTorch()` | Radial torch glow + vignette overlay |
| `drawSword()` | First-person sword viewmodel + gauntlet/arm |
| `bladeEffects()` | Elemental blade FX per sword level |
| `drawWorldSelect()` | 5×2 world card grid |
| `drawTitle()` / `drawGameOver()` / `drawPause()` / `drawHelp()` | Screens |
| `draw()` | Master render loop |
| Input handlers | keydown/keyup, mousemove, pointerlockchange, click |
| `loop()` | `requestAnimationFrame` driver |

---

## Game flow
`title` → `worldselect` → `playing` → `gameover` → `worldselect`

- Title: click → world select
- World select: click a card → `init(i)` → playing
- Gameover: click → world select (not direct retry)

---

## Key globals

```js
let MAP, worldCfg, selectedWorld   // active world
let gs    // { phase, kills, swordLevel, level, shake, warnText, warnTimer,
          //   flash, goldenFlash, paused, danceTimer }
let pl    // { x, y, angle, hp, maxHp, iTimer, bobPhase, bobY, moving }
let sw    // { on, t, dur:220ms, hits:Set }
let zombies, particles, grail
let gameTime  // ms accumulated while playing (drives blade FX animations)
let locked    // pointer lock active
let morning   // day/night toggle (M key)
let showHelp  // H key overlay
```

---

## World system

`WORLDS[i]` shape:
```js
{
  name, desc, col,          // card UI
  w0:[r,g,b], w1:[r,g,b],  // wall colors side 0 / side 1
  shd,                      // shade falloff divisor (~1.5–3.5)
  wr, wg,                   // torch warmth tint (red, green add)
  ct, cb,                   // ceiling top/bottom color
  ft, fb,                   // floor top/bottom color
  tc, tc2,                  // torch gradient colors (rgba strings)
  tR,                       // torch radius px
  va,                       // vignette alpha (0–1)
  sx, sy,                   // player spawn tile (always map[sy][sx-0.5] === 0)
  map: number[17][24]       // 17 rows × 24 cols, 1=wall 0=floor
}
```

All 10 maps guarantee `map[8][3] === 0` (player spawns at sx:3.5, sy:8.5).

---

## Raycaster

DDA algorithm, 600 rays. `zBuf[col]` stores perpendicular depth for sprite occlusion.
Camera plane: `plX = -sin(angle)*0.66, plY = cos(angle)*0.66`
Wall shading uses `worldCfg.shd / (pd + 0.4)` clamped to 1.

---

## Zombies

- Billboarded sprites, projected via camera matrix inverse
- 3D volumetric shading via `createRadialGradient` on head/torso/arms
- 4 skin variants by `z.id % 4`
- Dance mode (`gs.danceTimer > 0`): stop chasing, sway, arms raised, ♪ notes, happy eyes
- Multi-hit health scales with `gs.level`
- `z.wp` drives walk wobble (and fast during dance)

---

## Sword viewmodel

```js
blen = 270 + swordLevel * 55    // blade length px (270–485)
ox = CW * 0.78,  oy = CH * 0.85 // rest position (origin in local coords)
ang = -PI/4                      // angle at rest (points upper-left)
```

Draw order: blade → blade effects → ricasso collar → quillon/crossguard → grip → pommel → gauntlet → vambrace → elbow joint → upper arm → pauldron

Sword levels: 0 Dagger · 1 Short Sword (magic sparkles) · 2 Longsword (lightning) · 3 Greatsword (fire) · 4 Claymore (void aura + all)

---

## Audio (Web Audio API, zero external files)

| Function | Sound |
|---|---|
| `startMusic()` | Ominous ambient drone, starts on world entry |
| `stopMusic()` | Stops ambient drone |
| `startDanceMusic()` | 128 BPM disco beat, plays during zombie dance |
| `stopDanceMusic()` | Stops dance beat |
| `playSwing(lvl)` | Sword whoosh + metallic ring |
| `playHit()` | Thud + crunch on zombie hit |
| `playDie()` | Descending moan + crack on zombie death |
| `playDamage()` | Dissonant impact when player takes damage |
| `playGrail()` | Ascending 4-note chord on grail pickup |
| `playLevelUp()` | Fanfare on sword upgrade |

`getAudio()` lazily creates `AudioContext` and resumes if suspended (autoplay policy).

---

## Progression

| Milestone | Effect |
|---|---|
| Every 10 kills | Sword level up (max 4) + grail spawns |
| Grail pickup | +level, full heal, 5.2s zombie dance |
| Every 5 kills | Zombies get faster (speed cap 0.036) |
| gs.level | Zombie HP = max(1, gs.level) |

---

## Controls
- **WASD / Arrows** — move/rotate
- **A/D (mouse locked)** — strafe
- **Click / Space** — swing sword
- **Click canvas** — enable mouse look (pointer lock)
- **P / Esc** — pause
- **H** — help overlay
- **M** — toggle day/night

---

## Potential next steps (as of last session)
- Add a volume slider / mute toggle in the pause menu
- Add footstep sounds when moving
- Spawn zombies with a growl sound
- Add a proper title screen soundtrack
- Difficulty select on world screen (affects zombie count/speed multiplier)
- Persistent per-world high scores (currently shared)
- Animated wall textures or floor decals
- Zombie variety (fast/slow/tank types)
- Boss zombie every 25 kills
