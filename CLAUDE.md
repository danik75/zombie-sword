# Zombie Sword — Project Guide for Claude

## Live URL
**https://danik75.github.io/zombie-sword/**
Deployed via GitHub Pages (main branch root). Any `git push` to `main` auto-redeploys within ~60s.

## What this is
Browser-based first-person DOOM-style zombie survival game. Single `index.html` (~2420 lines, pure HTML5 Canvas + vanilla JS, no npm deps). Node.js HTTP server (`index.js`) serves it on port 3000.

```
node index.js          # start server
open http://localhost:3000
# Kill port if busy:  lsof -ti:3000 | xargs kill -9
```

Syntax-check JS after every edit:
```
node -e "
const fs=require('fs');
const src=fs.readFileSync('index.html','utf8');
const m=src.match(/<script>([\s\S]*?)<\/script>/g);
let ok=true;
m.forEach((s,i)=>{const js=s.replace(/<\/?script[^>]*>/g,'');try{new Function(js);}catch(e){console.log('Block',i,e.message);ok=false;}});
if(ok)console.log('OK');
"
```

---

## Architecture

Everything lives in `index.html` (~2420 lines). Key sections top-to-bottom:

| Section | What it does |
|---|---|
| `const WORLDS = [...]` | 14 world configs (colors, maps, torch, fog) |
| `solid()` | Wall collision using `MAP[y][x]` |
| `const SWORD_*` | Sword name/color/reach/arc arrays (5 levels) |
| Audio functions | Web Audio API synth — no external files |
| `init(wIdx)` | Reset game state, load selected world |
| `update(dt)` | Game logic tick (movement, zombies, timers) |
| `castRays()` | DDA raycaster — 600 strips, 2px each, FOV 66° |
| `drawMinecraftZombie()` | Blocky pixel-art zombie for world 13 only |
| `drawSprites()` | Billboarded zombie sprites with 3D gradients |
| `drawGrail()` | Billboarded golden grail sprite |
| `drawTorch()` | Radial torch glow + vignette overlay |
| `drawSword()` | First-person sword viewmodel + black sweatshirt arm |
| `bladeEffects()` | Elemental blade FX per sword level |
| `drawCF()` | Ceiling + floor gradients, calls `drawGrass()` |
| `drawGrass(mid)` | Animated swaying grass blades at floor horizon (all worlds) |
| `drawHUD()` | Health bar, kills, level, sword name, mode badge |
| `drawRadar()` | Mini-map radar bottom-right |
| `drawWarn()` | Centered warning text (level-up / speed) |
| `drawWorldSelect()` | 7×2 world card grid |
| `drawTitle()` | Title screen |
| `drawGameOver()` | Game over screen + compact scoreboard |
| `drawPause()` | Pause overlay |
| `drawHelp()` | Full help/controls overlay |
| `drawLeaderboard()` | Full-screen leaderboard overlay (top 10) |
| `draw()` | Master render loop |
| Input handlers | keydown/keyup, mousemove, pointerlockchange, click |
| `saveScore()` / `getScores()` | localStorage top-10 persistence |
| `loop()` | `requestAnimationFrame` driver |

---

## Game flow
`title` → `worldselect` → `playing` → `gameover` → `worldselect`

- Title: click → world select
- World select: click a card → `init(i)` → playing
- Gameover: click → world select
- Pause (P/Esc) → Q to quit to world select, L for leaderboard

---

## Key globals

```js
let MAP, worldCfg, selectedWorld   // active world (index 0–13)
let gs    // { phase, kills, swordLevel, level, shake, shakeAmt, warnText, warnTimer,
          //   flash, goldenFlash, paused, danceTimer }
let pl    // { x, y, angle, hp, maxHp, iTimer, bobPhase, bobY, moving }
let sw    // { on, t, dur:220ms, hits:Set }
let zombies, particles, grail
let gameTime   // ms accumulated while playing (drives blade FX + shine animations)
let locked     // pointer lock active
let morning    // day/night toggle (M key)
let showHelp   // H key overlay
let showLeader // L key leaderboard overlay
```

---

## World system — 14 worlds

`WORLDS[i]` shape:
```js
{
  name, desc, col,          // card UI
  w0:[r,g,b], w1:[r,g,b],  // wall colors side 0 / side 1
  shd,                      // shade falloff divisor (~1.5–5.2)
  wr, wg,                   // torch warmth tint (red, green add)
  ct, cb,                   // ceiling top/bottom color
  ft, fb,                   // floor top/bottom color
  tc, tc2,                  // torch gradient colors (rgba strings)
  tR,                       // torch radius px
  va,                       // vignette alpha (0–1)
  sx, sy,                   // player spawn tile (map[sy][sx-0.5] === 0)
  map: number[17][24]       // 17 rows × 24 cols, 1=wall 0=floor
}
```

| # | Name | Vibe |
|---|---|---|
| 0 | DEAD RUINS | Dark medieval rubble |
| 1 | CATACOMBS | Underground bone halls |
| 2 | HELLFIRE | Red lava glow |
| 3 | FROZEN TEMPLE | Cold blue ice |
| 4 | DESERT TOMB | Sandy orange |
| 5 | CURSED FOREST | Dark green |
| 6 | CRYSTAL CAVES | Purple crystal |
| 7 | DEAD FACTORY | Industrial grey |
| 8 | VOID TEMPLE | Deep black |
| 9 | BLOOD ARENA | Dark red |
| 10 | MAGMA DEPTHS | Volcanic orange walls |
| 11 | PERMAFROST | Bright reflective ice |
| 12 | THE OVERWORLD | Minecraft — bright sky, grass floor, blocky houses |
| 13 | BRAINROOT VAULT | Purple/magenta bio-horror, organic maze |

World-select grid: **7 cols × 2 rows** (`cw=158, ch=210, cols=7, gap=10`).

---

## Raycaster

DDA algorithm, 600 rays. `zBuf[col]` stores perpendicular depth for sprite occlusion.
Camera plane: `plX = -sin(angle)*0.66, plY = cos(angle)*0.66`
Wall shading: `worldCfg.shd / (pd + 0.4)` clamped to 1.

---

## Zombies

- Billboarded sprites projected via camera matrix inverse
- 3D volumetric shading via `createRadialGradient` on head/torso/arms
- 4 skin variants by `z.id % 4`
- Dance mode (`gs.danceTimer > 0`): sway, arms raised, ♪ notes, happy eyes
- Multi-hit health scales with `gs.level`
- `z.wp` drives walk wobble
- **World 13 only**: `drawMinecraftZombie()` — blocky pixel-art, green skin, blue shirt, gray pants, square head, arms outstretched

---

## Player arm (sword viewmodel)

```js
blen = 270 + swordLevel * 55    // blade length px (270–485)
ox = CW * 0.78,  oy = CH * 0.85 // rest position
ang = -PI/4                      // angle at rest
```

Draw order: blade → blade effects → ricasso → quillon → grip → pommel → **black sweatshirt sleeve + bare skin hand**

- Sweatshirt: black fabric with fold lines, ribbed cuff, soft elbow bunching, shoulder
- Hand: bare skin palm, 4 fingers, thumb
- Shine: animated diagonal gloss sweep + static knuckle specular (`gameTime * 0.0008`)

Sword levels: 0 Dagger · 1 Short Sword (sparkles) · 2 Longsword (lightning) · 3 Greatsword (fire) · 4 Claymore (void aura + all)

---

## Grass

`drawGrass(mid)` called from `drawCF()` for every world — 380 animated grass blades (two layers) swaying at the floor horizon using `gameTime`.

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

## Leaderboard

- `showLeader` global — toggled with **L** key
- `drawLeaderboard()` — full-screen overlay, top 10 scores
- Each score: `{ kills, level, sword, world, date }`
- Sorted by kills descending; capped at 10 entries
- **L** — open/close (pauses game if playing)
- **C** (while open) — clear all scores
- Hint shown on pause screen and game-over screen

---

## Progression

| Milestone | Effect |
|---|---|
| Every 10 kills | Sword level up (max 4) + grail spawns |
| Grail pickup | +level, full heal, 5.2s zombie dance, "KEEP DANCING!" warning |
| Every 5 kills | Zombies get faster (speed cap 0.036) |
| gs.level | Zombie HP = max(1, gs.level) |

---

## Controls

- **WASD / Arrows** — move/rotate
- **A/D (mouse locked)** — strafe
- **Click / Space** — swing sword
- **Click canvas** — enable mouse look (pointer lock)
- **P / Esc** — pause
- **Q** (while paused) — return to world select
- **H** — help overlay
- **L** — leaderboard overlay
- **M** — toggle day/night
- **C** (leaderboard open) — clear scores

---

## Potential next steps
- Volume slider / mute toggle in pause menu
- Footstep sounds when moving
- Zombie growl on spawn
- Difficulty select on world screen
- Animated lava floor for MAGMA DEPTHS (world 10)
- Frost crystal overlay for PERMAFROST (world 11)
- Zombie variety (fast/slow/tank types)
- Boss zombie every 25 kills
- More worlds (user asked about "Steal the Brainroot" — unknown game, needs clarification)
- Perspective toggle (user asked, not yet implemented)
