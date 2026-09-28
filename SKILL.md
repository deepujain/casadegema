---
name: casa-de-gemas
description: Change, extend, or remix Casa de Gemas — a single-file Canvas 2D + Web Audio watercolor garden game with 8 hidden gems. Use when adding a gem, a scene object, a held tool, a sound, a day-cycle effect, touch behavior, or when customizing the message, name, colors or layout of index.html.
---

# Casa de Gemas

Everything lives in one file, `index.html` (~2,550 lines): CSS and a few DOM elements at the top, then one `<script>` with pure JavaScript. No libraries, no images, no build step. Keep it that way — every addition must be drawn in code and fit in the same file.

## Mental model

- **Stage:** a fixed logical stage `W = 1200`, `H = 950`. All game coordinates are logical. Each frame, drawing uses `ctx.setTransform(B, 0, 0, B, OX, OY)`, where `B` is the scale in device pixels and `OX`/`OY` the offsets (`resize()` computes them; small screens crop to a focus rectangle).
  - Logical → CSS pixels: `(OX + B * x) / dpr`
  - CSS pixels → logical: `(css * dpr - OX) / B`
  - `setMouse(e)` already converts pointer events into `mouse.x/y` in logical units.
- **Layers:** static art is painted once into offscreen canvases (`bg` via `drawBg`, `fg` via `drawFg`) inside `resize()`. Anything that animates or changes state is drawn every frame in `render(dt)`. If you change static art, edit `drawBg`/`drawFg`; it's re-rendered on resize.
- **Watercolor helpers:** `wash(g, points, color, layers, alpha, amt)` for soft painted shapes, `stucco`, `bricks`, `wallPot`. Seeded randomness (`R = mulberry32(seed)`) keeps static art identical between reloads.
- **Time:** `time` is global seconds; `T` is time since the last `plant()` (replant). `dt` is capped at 0.033.
- **Useful landmarks (logical coords):** house stucco x 425–905, y 360–872; chimney `CHIMNEY {829, 230}`; tree bed `BIG_POT {cx 440, by 902}`; pots in `POTS`; bench top y 842, x 718–806; speaker `SPEAKER {786, 842}`; chalk `CHALK_HOME {725, 839}`; lanterns `LAMPS`; owl perch `PERCH {355, 134}`; girl `GIRL {700, 934}`; palette `ART {912, 914}`; twig `TWIG_HOME {600, 938}`; can `CAN_HOME {548, 924}`; easel `EASEL` (board x 990–1082, y 808–872); clock `CLOCK {900, 868}`.

## Frame order (inside `render`)

Background layer → carpet of settled petals → lantern glass → tree, petals, pots → foreground layer → girl & balloon → falling petals → speaker → clock → easel & chalk → twig → watering can → **night / dusk / dawn overlays** → fireflies → owl → chimney message → feather → gem effects → butterfly cursor.

Anything drawn before the overlays gets tinted at night (objects in the world). Anything after stays bright (UI-like effects). Put new objects in the matching place.

## Gems

`SURPRISES` is the single source of truth:

```js
{ id: 'balloon', color: '#6cc0f2', at: () => [balloon.x, balloon.y] }
```

- `id` — call `found(id, [x, y])` once when the player earns it; it is idempotent.
- `color` — the gem's color in the counter and fly effect.
- `at()` — where the feather should point when hinting this gem. If the gem needs a tool first, return the tool's position until it's held (see the `name` gem: chalk first, then the easel).

The counter HTML, `n/8` label, pop-and-fly animation, sparkle sound, hint targeting and the end message all derive from this array. To add a gem: add an entry, call `found()` at the right moment, and update the count mentioned in any copy. To remove one, delete the entry and its `found()` call.

`found()` also resets the hint timer (`lastFound`, `hintWait`). The ending (`drawMessage`) triggers when every gem is done, no gem is mid-flight, and 3 s have passed.

## Held tools (can, twig, chalk)

All three follow the same pattern — copy it for a new tool:

1. State object `{ held, x, y, a }` plus a `HOME` position and an `xxxAt(x, y)` hit test.
2. In `drawXxx(dt)`: lerp toward `mouse` when held (fast) or `HOME` when not (slower), then draw.
3. `pointerdown` handler: if held, handle the use / put-back and `return`; if not held and `xxxAt(...)`, pick it up and `return`. Order matters — tools are checked before scene objects.
4. `pointermove` hover: set a `hover` string (`'chalk'`, `'can'`…) so the object glows.
5. `Esc` clears `held`; `drawCursor` treats any held tool as "perched" (slow wings).
6. For touch, add the tool's home (and target) to `touchAnchors()`.

## Hints, feather, clock

- `hintWait` is a random 18–48 s (`nextWait()`), re-rolled on each discovery and after each hint.
- `drawFeather` starts a 10 s hint (`hintUntil`) when `time - lastFound > hintWait` and the player has been idle 5 s. It never hints at page load.
- `drawClock` visualizes `hintWait - (time - lastFound)`; `ringClock()` (tap the clock) starts a hint immediately.

## Day cycle and night creatures

- Lantern `mode` is `'on' | 'off' | 'coming'`. `toggleLamp` sends that lantern's fireflies (`flies[]`) to hang spots from `pickHang()` (tree segments, else the balcony railing) and brings them back on relight.
- `drawNight` eases `nightLv` and `warmLv` from the number of dark lanterns: 1 → dusk, 2 → night (stars, moon, balcony glow). Relighting after dark gives a dawn tint via `warmKind`.
- The owl state machine arrives when `nightLv > .6`, calls `found('night')` on landing, and leaves slowly only when every lantern is `'on'`.

## Sound

`Music` is an IIFE around one `AudioContext`, created lazily on the first speaker tap. Every sound function returns early unless `on` is true, so the game is silent until the player finds the music gem. To add a sound: write a small function inside `Music` that schedules oscillators or noise through `bus` (goes to reverb) or `master`, and add it to the returned object. Keep one-shot sounds short and let their gain envelopes decay — long tails stack up and read as "looping". `Music.unlock()` resumes audio on `touchend` for iOS.

## Touch

`touchMode` is set per pointerdown. `snapTouch()` moves a tap to the nearest entry of `touchAnchors()` within ~30 CSS px (unless a branch is closer), `pick()` widens its radius, and the balloon pops from farther away. `pointerleave` clears `hover` so glows don't stick. A CSS media query shows the "turn your phone sideways" pill in portrait on coarse pointers.

## Customizing

- **Message:** `MSG` for the first line; the name comes from the easel or `?name=` (`girlName`, max 16 chars).
- **Petal colors:** `PALETTES` (each tree gets a distinct one; `recolor(tree, pal)` swaps sprites).
- **Title:** the `#hint` element and `<title>`.
- **Timing:** `MSG_FOR` (message duration), `HINT_FOR`, `nextWait()`, `WATER_NEEDED`.

## Testing

Use Playwright (`npm i playwright@1`) against `file://…/index.html`:

- Listen for `pageerror`; zero errors is the bar.
- Drive state directly through globals, for example `waterPlant(BIG_POT)` to grow the tree, `SURPRISES.forEach(s => s.done = true); lastFound = -10` to see the ending, or `popBalloon()`.
- Convert logical coordinates to page coordinates with `page.evaluate(([x, y]) => [(OX + B * x) / dpr, (OY + B * y) / dpr], [x, y])` before `page.mouse.click`.
- Check a mobile context too (`viewport 844×390, hasTouch, isMobile`) and use `page.touchscreen.tap`.
- Take screenshots and look at them; most bugs here are visual.

## House rules

- One file, no dependencies, no external images. Google Fonts (Great Vibes, Caveat) are optional; fall back to system script fonts.
- Keep interactions intentional — nothing important should trigger from a mere hover.
- Discovery over instruction: no on-screen instructions beyond the title; let the feather and the clock guide.
- Gentle over flashy: soft easing, warm colors, quiet sounds.
