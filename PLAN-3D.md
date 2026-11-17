# Plan: 3D versions of all Game Night games

For: Claude Opus, carrying this out in later sessions.
Status: plan only. Nothing here has been built yet.

## What exists today

| Repo | Game | Size | How it draws now |
|---|---|---|---|
| `libyan11/spade-and-speak` | Spade & Speak (team word game on a round board) | `index.html` ~214 KB, plus `play/` (installable PWA copy with `sw.js`) | SVG board from `boardSVG()` (index.html:632), tokens moved by `updateTokens()` / `animateLocal()` (661, 690), spinner wheel by `paintWheel()` / `driveWheel()` (2011–2012), word cards from `fullCardHTML()` (754) |
| `libyan11/president-and-peasant` | President card game, 2–8 players plus computer players | `index.html` ~92 KB | DOM cards: `renderSeats()`, `renderTable()`, `renderHand()`, `fitHand()` (1296–1443) |
| `libyan11/grand-tour-tycoon` | Property trading board game, 2–8 players | `index.html` ~160 KB | DOM/CSS 40-cell board from `boardHTML()` (1492), `placeTokens()` (1396), `animateDice(final)` (1296), SVG icons for special cells (around line 479) |
| `libyan11/libyan11.github.io` | Game Night hub page that links to the three games | `index.html` ~9 KB | Plain cards |

Shared facts that shape the plan:

- Each game is **one HTML file** on GitHub Pages with no build step. Libraries are vendored in `vendor/` (PeerJS, MQTT).
- The host's browser runs the game. Other players get state over PeerJS/MQTT, and every device draws that state itself.
- Spade & Speak was just tuned to load fast (connection scripts load in the background, fonts don't block). 3D must not undo that.
- Players are mostly on phones.

## Goal

Give every game an optional 3D view of the board or table, built with Three.js, using Gemini to make the textures and art. The 2D view stays, and is used where 3D can't run well.

## Rules for the whole project

1. **3D is only a view.** Don't change game state, rules, or the network messages. A 3D player and a 2D player in the same room must see the same game. The new code reads the same state the 2D `render()` reads.
2. **2D stays as the fallback.** Use 2D when WebGL2 is missing, when `navigator.deviceMemory < 4`, when the WebGL context is lost, or when the player turns 3D off.
3. **Load Three.js only when 3D is on.** It should not be in the first page load. Load it after the page shows, the same way `window.NET` loads the connection scripts.
4. **Text stays in HTML.** Words, names, prices, timers, chat and buttons stay as DOM on top of the canvas, so they stay sharp, translatable and readable by screen readers. The canvas gets `aria-hidden="true"`. The existing SVG `<title>`s and `aria-label`s move to a visually hidden description.
5. **Dice and spinner results come from the game, not from physics.** The host picks the result as now, and the 3D animation ends on that result on every device.
6. **Draw frames only while something moves.** No constant render loop. Stop when the tab is hidden. Cap device pixel ratio at 2, or 1.5 on weak devices. Aim for 60 fps on a mid-range phone.
7. **With reduced motion on** (`prefers-reduced-motion`): no camera sweeps, and pieces jump straight to where they end up.
8. **Each repo stays self-contained.** Copy the shared 3D helper into each repo. Don't load it across repos.

## Tech choices

- **Three.js**, vendored as one pinned file per repo: `vendor/three-<version>.module.min.js`, the same version in every repo. Load it with a dynamic `import()` from a small `<script type="module">` that sets `window.Board3D`. The existing scripts stay as classic scripts.
- **No model files to begin with.** Build boards, tokens, dice, cards, houses and the table from simple geometry in code (rounded boxes, cylinders, extruded shapes). It stays small and needs no loader. Add glTF only if a piece really needs it, and then only with Draco/meshopt compression.
- **Text on 3D surfaces** (board cell names, card ranks): draw it on a canvas and use that as a texture, generated in the browser from the game's own data.
- **Lighting:** one hemisphere light, one directional light, and a baked soft shadow under the board. No live shadow maps on phones.
- **A setting in each game:** "3D board: On / Off", saved in `localStorage`. Every read and write of the setting goes in try/catch. The default is On only when the device passes the checks in rule 2.

## Where Gemini comes in

Gemini makes the **art**. The geometry and the code come from Opus.

How to use it: the `gemini-web-creator` skill, which drives Gemini in the user's signed-in Chrome. That needs the user's computer to be connected to the session. If it isn't, ask the user once to connect it. If Gemini can't be reached, use the procedural canvas textures from the last row of the table below and keep going.

What to ask Gemini for:

| Asset | Used in | Prompt outline |
|---|---|---|
| A style frame per game (one image) | Agreeing the look with the user before building | "Isometric 3D render of a cosy board game night table, [game] board in the middle, soft studio light, chunky toy-like pieces, bright friendly colours, no text" |
| Seamless felt texture | President & Peasant table | "Seamless tileable green card-table felt texture, top-down, even lighting, 1024×1024" |
| Seamless wood texture | Table edges, board rims | "Seamless tileable warm oak wood grain, top-down, even lighting" |
| Board surface paper/print texture | Grand Tour Tycoon and Spade & Speak boards | "Seamless subtle printed cardboard texture, off-white, no pattern, no text" |
| Card backs (one per game) | President & Peasant cards, Spade & Speak word cards, Tycoon chance/chest cards | "Playing card back design, symmetrical, [palette from game CSS], no text, flat, 512×768" |
| Backdrop gradient or soft room blur | Behind the 3D scene | "Soft out-of-focus warm living room background, no objects in focus" |
| Token reference sheets | Guide for modelling pawns and tokens in code | "Turnaround sheet of 4 chunky toy game tokens: car, top hat, dog, ship, plain colours" |
| Fallback if Gemini is unavailable | Everything above | Procedural canvas textures: noise for felt and paper, stripes for wood, and the existing SVG icons drawn onto canvas |

Handling Gemini output:

- **Show each image to the user before it goes into a repo.** The user chooses which ones to keep.
- Never ask for text inside images. Text comes from the game data (rule 4).
- Prepare each image: crop, resize to a power of two (1024 at most, 512 for card backs), save as WebP with a JPEG fallback, and check that tiles repeat without seams.
- Keep the new images under about 1.5 MB per game in total.
- Store them in `assets/3d/` in each repo. Write where each image came from in `assets/3d/SOURCES.md`.

## The shared 3D helper (`board3d.js`, one copy per repo)

A small module, about 300 lines, that every game uses:

- `Board3D.supported()`: checks WebGL2, device memory and the user's setting.
- `Board3D.mount(container, opts)`: creates the renderer, camera and lights, handles resize with `ResizeObserver`, the DPR cap, context loss (switch to 2D) and visibility (pause).
- `requestFrame()`: draws only while something is animating.
- Tween helpers: `tween(obj, to, ms, ease)` and `hop(mesh, path)` for pieces moving cell to cell.
- `pick(x, y)`: raycasting for taps on cells and pieces, with the same handlers as the 2D clicks.
- `canvasTexture(drawFn, w, h)` for text and icons, and `loadTex(url)` for the Gemini images.
- `dispose()`: frees everything when the player leaves the game.

## Per-game work

### 1. Spade & Speak (do this first: it's this repo)

- **Board:** build the ring from the same `spaces` data and angles `boardSVG()` uses (R0/R1, `seg-<cat>` colours from the CSS variables). Each space is an extruded ring segment with a small height. Mark spade and spinner spaces with raised badges. Add a finish space.
- **Tokens:** a team pawn for each team, in its team colour. `updateTokens()` / `animateLocal()` become hops along the segment centres, at the same timing as now, so 2D and 3D players stay in step.
- **Spinner:** a 3D wheel that eases onto the same slice and angle `driveWheel(sp)` uses, over `SPIN_MS`.
- **Word card:** a 3D card flips up from the deck. The words stay as the `fullCardHTML()` DOM, placed over the card, and still follow the "full card / one word" setting.
- **HUD:** timer, chat, voice call and panel stay as they are.
- **Home screen:** the demo board (`boardSVG(buildSpaces(40), false)`) becomes a slowly turning 3D board. Show it only when 3D is on and reduced motion is off.
- **PWA copy:** `play/index.html` gets the same changes. Add the new vendor and asset files to the precache list in `play/sw.js` and bump the cache name.

### 2. President & Peasant

- **Table:** a felt table (Gemini texture) with wood edges. Seats go round the table for 2–8 players, using the same order as `renderSeats()`. Each opponent's card count shows as a small fanned stack of card backs. Their name and title stay as DOM labels.
- **Pile:** the cards `renderTable()` shows are played into the middle with a short arc and a slight random turn. When the table clears (four of a kind, the 3♠, or everyone passing), the pile slides off.
- **Your hand stays 2D** (`renderHand()` / `fitHand()`). Picking cards on a phone must stay easy. Played cards fly from the bottom edge of the screen into the 3D pile.
- **Card faces:** canvas textures made from rank and suit, matching the current card look, plus the jokers. Card back from Gemini.
- **End of game:** a short camera move for the title swap (President ↔ Peasant). Skip it when reduced motion is on.

### 3. Grand Tour Tycoon (the biggest)

- **Board:** a square slab with the 40 cells laid out from `cellRC()` / `MID`. Each property gets a colour band. Cell names and prices are canvas textures. Draw the special-cell SVG icons (rail, bulb, tap, chest, tax, car, cop) onto canvas textures, so they don't have to be redrawn.
- **Tokens:** low-poly car, hat, dog and ship, modelled from the Gemini reference sheet, in player colours (`PCOLORS`). They move cell by cell through `placeTokens()`'s path and take the short way to jail.
- **Dice:** two rounded-box dice that tumble and land on `animateDice(final)`'s result. The rotation that shows each face is precomputed.
- **Houses and hotels:** small instanced meshes on the colour band, updated from state.
- **Chance and chest cards:** a card lifts from its deck and flips. The text stays in the existing modal from `renderModal()`.
- **Tapping cells:** opens the same property info the 2D board opens.
- **Camera:** angled view of the whole board. Optionally, a gentle follow of whoever's turn it is, off when reduced motion is on.

### 4. Game Night hub (last, optional)

- A light hero scene: a spade token, a card and a die floating gently, one per game card. Load it only on capable devices, and only after the page has shown. The links must work fully without it.

## Order of work

1. **Spike (one session):** add `board3d.js` and Three.js to Spade & Speak. Show the ring board with tokens in the lobby view. Measure page load, fps and memory on a phone-sized viewport. Stop and report the numbers before going on.
2. **Gemini art round 1:** style frames for all three games. The user chooses a direction.
3. **Spade & Speak, all of it:** board, tokens, spinner, card, setting, PWA copy, tests.
4. **Gemini art round 2:** felt, wood, card backs and token references.
5. **President & Peasant.**
6. **Grand Tour Tycoon.**
7. **Hub hero (optional).**

Each game is its own branch with its own PR in its own repo. Don't open PRs unless the user asks.

**Repository access:** this session can push only to `spade-and-speak` (branch `claude/jolly-mccarthy-sj4quc`). `president-and-peasant` and `libyan11.github.io` were readable here. Attaching `grand-tour-tycoon` was refused, so the user needs to allow it, or start a session in that repo, before step 6.

## Testing (Playwright with the preinstalled Chromium)

- Screenshots of each game in 3D and 2D at phone (390×844) and desktop sizes, in light and dark themes.
- **Mixed room:** two browser contexts in one room, one with 3D and one with 2D. Play several turns and check that the token positions, the pile and the dice results agree.
- **Fallback:** launch with `--disable-webgl` and check that the 2D board shows and the game plays.
- **Context loss:** call `WEBGL_lose_context.loseContext()` in the middle of a game and check that it switches to 2D without breaking the game.
- **Reduced motion:** emulate it, and check that pieces jump straight to their place.
- **Load time:** compare first paint with 3D on and off. The 3D bundle must not delay first paint.
- **Frames:** on mid-range phone emulation, frame time stays under about 16 ms while animating, and no frames are drawn when nothing is moving.

## Done when

- [ ] All three games have a 3D view that plays a full game, online and on one device, with 2D still working.
- [ ] 3D and 2D players can share a room.
- [ ] The 3D setting saves, the fallbacks work, and reduced motion is respected.
- [ ] First paint is no slower than now. Each game's 3D extras total under about 2 MB.
- [ ] Every Gemini image was approved by the user and is listed in `SOURCES.md`.
- [ ] Each repo's README mentions the 3D board and the setting.

## Decisions the user still needs to make

1. **Default:** should 3D start On on capable devices (the plan assumes yes), or Off until turned on?
2. **Look:** chunky toy-like pieces (the plan assumes this) or realistic. Gemini round 1 settles it.
3. **President & Peasant hand:** keep the hand 2D (recommended for phones) or put it in 3D too?
