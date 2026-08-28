# mario Architecture

`index.html` is a single self-contained file (inline `<style>` + `<canvas>` +
one `<script>`) implementing one playable level of a Mario-style 2D
platformer. No build step, no dependencies, no external requests.

## Game loop

```
requestAnimationFrame(frame)
        │
        ▼
  frame(timestamp)
        │
   dt = clamp(timestamp - lastTime, ≤ 1/30s)
        │
        ├── gameState === 'playing'? → update(dt)   (state only, no drawing)
        │
        └── render()                                (drawing only, always runs)
        │
        ▼
requestAnimationFrame(frame)   ← loops
```

`update` and `render` are kept strictly separate so the win/lose overlay
keeps drawing every frame even after `gameState` leaves `'playing'` and
`update` stops running. `dt` is clamped so backgrounding the browser tab
(a huge next-frame delta) can't fling the player through geometry.

## State ownership

All state is a handful of top-level `let`/`const` bindings in the one
`<script>` block — no framework, no modules.

| Variable | Holds | Mutated by |
|---|---|---|
| `player` | position, velocity, facing, lives, invulnerability timer | `updatePlayerHorizontal`, `updatePlayerVertical`, `moveAndCollide`, `damagePlayer`, `respawnPlayer` |
| `level` | static solids, goal rect, level width, and the *initial* coin/enemy templates | only replaced wholesale by `buildLevel()` in `resetGame()` |
| `coins` / `enemies` | per-run mutable copies of `level.coinsInit` / `level.enemiesInit` (add `collected`/`alive` flags) | `updateCoins`, `updateEnemies`; rebuilt by `resetGame()` |
| `camera` | horizontal scroll offset | `updateCamera` |
| `input` | held-state booleans (left/right) + edge-triggered jump flag | `keydown`/`keyup` listeners; `jumpPressed` consumed inside `updatePlayerVertical` |
| `score` | coins collected this run | `updateCoins`; reset by `resetGame()` |
| `gameState` | `'playing' \| 'won' \| 'lost'` | `checkWinLose`, `damagePlayer`, `resetGame()` |

`level.solids`/`goal` never change after `buildLevel()`; `coins`/`enemies`
are separate mutable arrays derived from `level`'s `*Init` templates so a
restart (`R`) can cheaply recreate fresh copies without re-declaring the
whole level layout.

## Level data model

Hand-authored flat arrays, not a tilemap engine — appropriately scoped for
one level:

```
solids:  [{ x, y, w, h }]          // static AABB rects: ground + platforms
coins:   [{ x, y, r, collected }]  // collectibles, circle-distance pickup
enemies: [{ x, y, w, h, vx, speed, patrolMin, patrolMax, alive }]
goal:    { x, y, w, h }            // flagpole rect
```

Pits are **implicit**: any x-range not covered by a ground-height `solids`
rect is a gap. There's no explicit "pit trigger" — falling death is
detected generically as `player.y > canvas.height + margin` in
`checkWinLose()`, which also covers any other way the player ends up below
the level.

## Collision approach

Axis-separated AABB resolution (`moveAndCollide`): move the entity along X,
resolve X-overlaps against every solid; then move along Y, resolve
Y-overlaps. Resolving one axis at a time — rather than both at once — is
what correctly distinguishes "landed on top" (`vy > 0` → snap above, set
`onGround`) from "hit the ceiling" (`vy < 0` → snap below) from "hit a
wall" (`vx` resolution), and avoids catching on the seam between two
adjacent ground rects. Full swept-AABB/raycast collision isn't needed at
this scope — speeds are modest relative to rect sizes.

`player.onGround` is set only when that frame's vertical pass resolves a
downward collision; walking off a ledge naturally flips it `false` the
next frame since nothing collides.

Enemy-player collision reuses the same `aabbOverlap` check but is
disambiguated by direction rather than resolved physically: falling
(`player.vy > 0`) with the player's bottom edge close to the enemy's top
edge counts as a stomp (enemy dies, player bounces); anything else counts
as a side hit (`damagePlayer()`).

## Win / lose flow

```
                    ┌───────────┐
                    │  playing  │◄────────────┐
                    └─────┬─────┘              │
        touch flagpole    │    fall in pit /    │ respawn at start
                           │    hit by enemy      │ (lives remain)
              ┌───────────┤                       │
              ▼           ▼                       │
           ┌─────┐   lives -= 1 ──── lives > 0 ───┘
           │ won │        │
           └──┬──┘        │ lives <= 0
              │            ▼
              │        ┌──────┐
              └───────►│ lost │
        press R         └──┬───┘
        (resetGame)         │
              ◄──────────────┘
                 press R
```

Both `won` and `lost` freeze `update()` (input still tracked but no longer
acted on) and draw a full-canvas overlay; `R` calls `resetGame()`, which
rebuilds `level`/`coins`/`enemies`, recreates `player` via `makePlayer()`,
and resets `score`/`gameState`.

## Non-goals

Deliberately out of scope for a one-level demo, not omissions to revisit:
no menu/title screen, no level select, no save/localStorage persistence,
no sound, no sprite images (flat-color `fillRect`/`arc` shapes only), no
tilemap loader, no touch controls.
