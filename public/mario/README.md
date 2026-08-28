# mario

A single self-contained 2D platformer level, built with `<canvas>` and
vanilla JavaScript — no build step, no dependencies, no assets.

## Contents

- **`index.html`** — the entire game: markup, styles, and script inline in
  one file. Simple colored-shape graphics (no sprite images).

## Running

Open `index.html` directly in a browser (double-click, or drag into a
browser window). No server needed — the page makes no external requests.

## Controls

- **Arrow keys** or **WASD** — move left/right
- **Space**, **Up**, or **W** — jump
- **R** — restart

## Goal

Reach the flagpole at the far right of the level. Avoid falling into pits
and colliding with enemies from the side (jumping on top of them is safe
and defeats them). Collect coins along the way. Three lives; lose them all
and it's game over — press R to try again.
