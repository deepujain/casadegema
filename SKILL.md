---
name: casa-de-gemas
description: Change, extend, or remix Casa de Gemas — a single-file Canvas 2D + Web Audio watercolor garden game that moves through four seasons of hidden gems, narrated by Oli the owl. Use when adding a gem, a season, a scene object, a held tool, a line for Oli, weather, a sound, a day-cycle effect, touch behavior, or when customizing the message, name, colors or layout of index.html.
---

# Casa de Gemas

Everything lives in one file, `index.html` (~3,500 lines): CSS and a few DOM elements at the top, then one `<script>` with pure JavaScript. No libraries, no images, no build step. Keep it that way — every addition must be drawn in code and fit in the same file.

## Mental model

- **Stage:** a fixed logical stage `W = 1200`, `H = 950`. All game coordinates are logical. Each frame, drawing uses `ctx.setTransform(B, 0, 0, B, OX, OY)`, where `B` is the scale in device pixels and `OX`/`OY` the offsets (`resize()` computes them; small screens crop to a focus rectangle).
  - Logical → CSS pixels: `(OX + B * x) / dpr`
  - CSS pixels → logical: `(css * dpr - OX) / B`
  - `setMouse(e)` already converts pointer events into `mouse.x/y` in logical units.
- **Layers:** static art is painted once into offscreen canvases (`bg` via `drawBg`, `fg` via `drawFg`) inside `resize()`. Anything that animates or changes state is drawn every frame in `render(dt)`. If you change static art, edit `drawBg`/`drawFg`; it's re-rendered on resize.
- **Watercolor helpers:** `wash(g, points, color, layers, alpha, amt)` for soft painted shapes, `stucco`, `bricks`, `wallPot`. Seeded randomness (`R = mulberry32(seed)`) keeps static art identical between reloads.
- **Time:** `time` is global seconds; `T` is time since the last `plant()` (replant). `dt` is capped at 0.033.
- **Useful landmarks (logical coords):** house stucco x 425–905, y 360–872; chimney `CHIMNEY {829, 230}`; main tree `ROOT {990, 884}` in the tree bed `BIG_POT {cx ROOT.x, by 902}` (right of the house, so the facade stays clear); pots in `POTS`; tower wall pots `WALL_POTS`; bench top y 842, x 718–806; speaker `SPEAKER {786, 842}`; lanterns `LAMPS`; owl perch `PERCH {355, 134}`; girl `GIRL {700, 934}`; palette `ART {866, 914}`; twig `TWIG_HOME {600, 938}`; can `CAN_HOME {548, 924}`; easel `EASEL` (left, in front of the tower: board x 394–486, y 808–872) with the chalk `CHALK_HOME` on its ledge; clock `CLOCK {900, 868}`; sunflower seed `SEED {300, 896}`; pumpkin patch `PATCH {262, 936}`; butterfly jar `JAR {740, 834}`; rake `RAKE_HOME {772, 944}`; leaf pile `PILE {686, 938}`; umbrella `UMB {868, 874}`; rain cloud `CLOUD {640, 212}`; snowman `SNOWMAN {1074, 944}` and its three `MOUNDS`; balcony railing x 672–848, y 488–542.
- **Mobile crop:** phones only show roughly x 222–1098, so keep new interactive things inside that band.

## Frame order (inside `render`)

Background layer → carpet of settled petals → **season back** (snow on roofs, sills and courtyard; autumn leaf litter; rain cloud) → lantern glass → tree, petals, pots → foreground layer → girl & balloon → **season front** (sunflower, pumpkin, leaf pile, snowman, umbrella, jar lid, chimney smoke, kite) → falling petals → speaker → clock → easel & chalk → twig → watering can → rake → **season color grade** → **night / dusk / dawn overlays** → window glow & fairy lights → rain / snow → fireflies → owl → Oli's speech bubble → chimney message → season transition (sun, leaf gust or snowfall) → feather → gem effects → butterfly friends → butterfly cursor.

Anything drawn before the overlays gets tinted by the season and the night (objects in the world). Anything after stays bright (lights, weather, UI-like effects). Put new objects in the matching place.

## Seasons

`CHAPTERS` is the list of seasons, in order:

```js
{ name: 'autumn', dot: '#e8672a', intro: '…', outro: '…', gems: [ /* gem entries */ ] }
```

- `season` is the index of the current chapter; `SURPRISES` always points at `CHAPTERS[season].gems`, so everything that reads `SURPRISES` (counter, feather, clock, hints) follows the season automatically.
- `buildCounter()` rebuilds the gem icons, the season dots and the "*season* gems n/5" label. Every season has five gems, and the fifth is always a wall-pot visitor (`nookGem(n)`); keep the counts equal when adding gems.
- `updateSeason(dt)` eases `SLV[i]` (0–1 visibility of each season) and, when every gem is done and none is mid-flight, has Oli say the `outro`, waits ~5 s, then calls `setSeason(season + 1)`.
- `setSeason(n)` swaps the gem list, resets the hint timer, drops any held tool, relights the lanterns, sets up that season's objects, and starts the transition (`spawnSwirl`: sun, leaf gust or snowfall, recorded in `trans`) plus Oli's `intro`.
- Tree sprites come from `seasonSprite(p)` (spring: `tree.pal` blossoms with a third `leafSpr`; summer: all `summerLeaf`; autumn: `SPRITES.autumn`; winter: `null`, meaning bare). In winter `snowAcc` climbs 0→1 over ~45 s; a bare petal slot shows a small `SPRITES.snow` clump once `snowOn(p)` (its random `p.th` is under `snowAcc * SNOW_COVER`), and flatter branch segments get a white line on top. `tree.pal` always keeps the chosen bloom color. `recolor(tree, pal, delay)` flags petals to swap; `arriveDelay(n, x, y)` times each swap to when the season's element reaches that spot.
- Painted plants use `PL` (from `PLANT_COLS[season]`). `repaintPlants(n)` repaints the background and foreground layers; `drawOldLayer` keeps the previous layer visible, clipped to where the new season has not arrived yet (outside a growing circle from `SUN`, right of the gust front, below the snow line). `gust(t)` adds to `wind()` during the autumn transition.
- Draw seasonal things with `globalAlpha = SLV[i]` so they fade in and out with the cross-fade. `drawSeasonTint` applies the per-season color grade from `TINTS`; `DRIFT[season]` sets how often petals fall on their own (many in autumn, none in winter).
- `drawMessage` only runs in the last chapter, so the Daughter's Day ending comes after winter.
- `?season=summer|autumn|winter` calls `jumpToSeason()`: grows every tree, pops the balloon (from autumn on), marks earlier gems done, and leaves persistent objects (sunflower, pumpkin) grown.
- To add a season: add a `CHAPTERS` entry (and a `TINTS`, `DRIFT`, `SEASON_PAL`, `PLANT_COLS` and `SIS_WEAR` slot, a `seasonSprite` case, and an element in `spawnSwirl`/`arriveDelay`/`drawOldLayer`), set up its objects in `setSeason`, draw them in `drawSeasonBack`/`drawSeasonFront`, and add taps to `seasonTap`, `seasonHover` and `seasonAnchors`.

## Oli the owl

- Oli sits on the tower roof (`PERCH`) from the start. `say(text, seconds)` shows a speech bubble beside him (latest line wins; long lines stay up longer). It flips below-right on narrow screens.
- Each gem can have a `say` line (a string or a function, e.g. to include `girlName`) that plays when found, except for the last gem of a season (the outro plays instead).
- Each gem has a `tip` (string or function). Tapping Oli (`owlAt`, `owlHint`) says the next gem's tip and rings the clock so the feather flies there too.
- Night gem (autumn): when both lanterns are off, Oli hoots, flies one loop around the tree (`LOOP`), lands back on the tower and calls `found('night')`. He only hoots at night, and wears a scarf in winter.

## Gems

Each gem entry:

```js
{ id: 'balloon', color: '#6cc0f2', at: () => [balloon.x, balloon.y], tip: 'That twig looks sharp…', say: 'Pop! Oh, my feathers!' }
```

- `id` — call `found(id, [x, y])` once when the player earns it; it is idempotent and ignores ids that aren't in the current season.
- `color` — the gem's color in the counter and fly effect.
- `at()` — where the feather should point when hinting this gem. If the gem needs a tool first, return the tool's position until it's held (the `name` gem points at the chalk first, then the easel; `canHint(x, y)` does the same for the can).
- `tip`, `say` — Oli's hint and reaction (see above).

`found()` also resets the hint timer (`lastFound`, `hintWait`). The counter, pop-and-fly animation, sparkle sound, hint targeting and season change all derive from the current list. To add a gem: add an entry to a chapter's `gems`, call `found()` at the right moment, and update the counts in the README.

Watering spots: `SPOTS` holds extra places the can can water (`SEED`, `PATCH`), each with `open()` (is it waterable now?) and `sprout()`, plus an optional `lo` (how far left the zone reaches; `SEED` reaches over the neighboring pot). `zoneAt` checks them first and `waterPlant` calls `sprout()`; the can won't put itself away while a spot is still open.

## Held tools (can, twig, chalk, rake)

All four follow the same pattern — copy it for a new tool:

1. State object `{ held, x, y, a }` plus a `HOME` position and an `xxxAt(x, y)` hit test.
2. In `drawXxx(dt)`: lerp toward `mouse` when held (fast) or `HOME` when not (slower), then draw.
3. `pointerdown` handler: if held, handle the use / put-back and `return`; if not held and `xxxAt(...)`, pick it up and `return`. Order matters — tools are checked before scene objects.
4. `pointermove` hover: set a `hover` string (`'chalk'`, `'can'`…) so the object glows.
5. `Esc` clears `held`; `drawCursor` treats any held tool as "perched" (slow wings).
6. For touch, add the tool's home (and target) to `touchAnchors()`.

The rake (autumn only) stays inside `FLOOR` while held. `sweep(dx, dy)` sends every leaf in `litter` that the moving tine head touches, in any direction, to `pile.leaves` via `toPile`; it skitters along the ground in low hops to the heap at `PILE` (just in front of the girl's feet). `spawnLitter` seeds a small heap there so the spot is visible from the start, and `drawPile` draws a mound that rises with `pile.lv` as leaves land. The tines also erase the carpet under them, and `Music.rake(speed, pushing)` drives the scrape and crunch sound. At 85% swept the rest follow, `floorClean` wipes the courtyard carpet, the girl hops (`girlHop()`), leaves burst up and `found('leaves')` fires. After that, autumn drift drops to a trickle.

## Scene taps

One-tap seasonal objects go through three functions so desktop and touch stay in sync: `seasonTap(x, y)` (act, return `true`), `seasonHover(x, y)` (return a hover string such as `'jar'`, `'cloud'`, `'mound'`, `'chimney'`, `'rail'`) and `seasonAnchors()` (touch snap points). Each hit test (`jarAt`, `cloudAt`, `umbAt`, `moundAt`, `snowmanAt`, `chimneyAt`, `railAt`) checks its own season and state, so it's inert the rest of the year.

Wall-pot visitors: `NOOKS[season]` (nest, ladybug, mouse, robin) hides in `WALL_POTS[nook.pot]`; `setSeason` calls `hideNook()` to pick a different pot. `nookAt` hit-tests every wall pot. A plain tap only puffs petals (`puffNook`); the visitor needs the season's action, which calls `stirNook(i)` (reveals it on the right pot via `nook.t0`, drawn by `drawNook` in the season back): spring waters the pot (`zoneAt(x, y)` returns a `{ wall: i }` zone for the can; each watered pot gets a lasting bloom in `nook.bloom`), summer waters it (the wall zone is open in spring and summer; water keys are per season) or rests the pointer on it for 1.4 s (`nookTools`; the can may be held), autumn presses and wiggles it (`nook.grab`, `wiggleNook`), winter knocks it with the held twig: any part of the twig (`twigNook` samples along it) rubbing inside the pot for 50 px, or a tap on the pot while holding the twig; every pot the twig brushes sheds a little snow (`puffNook`) so the player sees the twig is working.

## Weather

- Rain: `rain.on` / `rain.lv`. Tapping the cloud starts ~30 s of rain; opening the umbrella shortens it to ~9 s. Rain darkens the color grade, draws angled streaks with ground splashes, and feeds `Music.pour` for a pitter-patter.
- Snow: `flakes` drift with `wind(time)` whenever winter is showing (`SLV[3]`).
- From summer on, the little sister (`drawSister`, standing at `SIS {646, 930}`, scale `.86`) walks from the door showing her face, then turns her back and eases `sister.hold` to 1. `drawGirl` then reaches her left hand out past the dress (`GIRL.left`), and the sister's right hand meets it. Her clothes come from `SIS_WEAR[season]`; she stays for the rest of the year.
- The kite's string runs from `GIRL.hand` (set every frame by `drawGirl`) to the kite's crossbar, both while stuck and in flight. On release `kite.from` is read before the state changes to `'rise'`, so it lifts out of its own branch; `kiteUp()` and `umbUp()` keep her arm raised. `umbUp()` is true only while it rains (`umbrella.open` just records that the umbrella was found). `canopy(x, y, r)` draws the open umbrella as a dome: apex on top, six panels curving down to a scalloped rim centered at `(x, y)`. Held up, it tilts from her hand to midway between `GIRL` and `SIS` and widens (r up to 68) to cover both sisters; `umbrella.shade` records the sheltered span so raindrops splash on the canopy instead of falling on them.

Teddy: `teddy` peeks from one of `TEDDY_SPOTS` (chimney top, roof ridge, tower left edge, right house corner; `dir` up/left/right) every 25–40 s while gems remain, once `mainTree` exists and the season isn't ending. `drawTeddy` (right after the season back) clips him to the far side of that edge so he looks hidden behind the house, with paws over the edge. `teddyAt` hit-tests his head (hover `'teddy'`, touch anchor via `seasonAnchors`). `tapTeddy` starts `teddy.help`, which every 1.3 s calls `teddyShow(s)` for the next remaining gem, using the gem's own action where there is one (nook `stirNook`, kite rise, sunflower/pumpkin sprout, `openJar`, rain + `openUmbrella`, snowman built with nose, `lightHearth`, `stringLights`) and plain `found(id)` otherwise. `setSeason` resets him.

## Hints, feather, clock

- `hintWait` is a random 18–48 s (`nextWait()`), re-rolled on each discovery and after each hint.
- `drawFeather` starts a 10 s hint (`hintUntil`) when `time - lastFound > hintWait` and the player has been idle 5 s. It never hints at page load.
- `drawClock` visualizes `hintWait - (time - lastFound)`; `ringClock()` (tap the clock) starts a hint immediately.

## Day cycle and night creatures

- Lantern `mode` is `'on' | 'off' | 'coming'`. `toggleLamp` sends that lantern's fireflies (`flies[]`) to hang spots from `pickHang()` (tree segments, else the balcony railing) and brings them back on relight.
- `drawNight` eases `nightLv` and `warmLv` from the number of dark lanterns: 1 → dusk, 2 → night (stars, moon, balcony glow). Relighting after dark gives a dawn tint via `warmKind`.
- At night in autumn Oli does his loop (see above). Lanterns work the same in every season; `setSeason` relights them, except winter, which starts with both lanterns off and their fireflies already hanging in the tree (the `?season=winter` jump also sets `nightLv = 1`).

## Sound

`Music` is an IIFE around one `AudioContext`, created lazily on the first speaker tap. Every sound function returns early unless `on` is true, so the game is silent until the player finds the music gem. To add a sound: write a small function inside `Music` that schedules oscillators or noise through `bus` (goes to reverb) or `master`, and add it to the returned object. Keep one-shot sounds short and let their gain envelopes decay — long tails stack up and read as "looping". `Music.unlock()` resumes audio on `touchend` for iOS.

## Touch

`touchMode` is set per pointerdown. `snapTouch()` moves a tap to the nearest entry of `touchAnchors()` within ~30 CSS px (unless a branch is closer), `pick()` widens its radius, and the balloon pops from farther away. `pointerleave` clears `hover` so glows don't stick. A CSS media query shows the "turn your phone sideways" pill in portrait on coarse pointers.

## Customizing

- **Message:** `MSG` for the first line; the name comes from the easel or `?name=` (`girlName`, max 16 chars).
- **Oli's lines:** `intro`/`outro` in `CHAPTERS`, `tip`/`say` on each gem.
- **Petal colors:** `PALETTES` (each tree gets a distinct one; `recolor(tree, pal)` swaps sprites via `seasonSprite`). Seasonal sprite sets (`SPRITES.autumn`, `SPRITES.snow`) are kept out of `PALETTES` so they never show up in the palette picker.
- **Title:** the `#hint` element and `<title>`.
- **Timing:** `MSG_FOR` (message duration), `HINT_FOR`, `nextWait()`, `WATER_NEEDED`.

## Testing

Use Playwright (`npm i playwright@1`) against `file://…/index.html`:

- Listen for `pageerror`; zero errors is the bar.
- Drive state directly through globals, for example `waterPlant(BIG_POT)` to grow the tree, `SURPRISES.forEach(s => found(s.id))` to finish a season and watch the transition, or `popBalloon()`. Open `?season=winter` to test a later season directly.
- Convert logical coordinates to page coordinates with `page.evaluate(([x, y]) => [(OX + B * x) / dpr, (OY + B * y) / dpr], [x, y])` before `page.mouse.click`.
- Check a mobile context too (`viewport 844×390, hasTouch, isMobile`) and use `page.touchscreen.tap`.
- Take screenshots and look at them; most bugs here are visual.

## The full prompt

To recreate Casa de Gemas from scratch in one shot, give a coding agent this prompt. It describes the final design, including lessons learned along the way.

```text
Build a single self-contained index.html (pure JavaScript + Canvas 2D, no libraries, no
images, no build step) called "Casa de Gemas" — a little garden of hidden gems, made as a
Happy Daughter's Day gift.

SCENE
- Soft watercolor illustration on warm cream paper. Fixed 1200×950 logical stage scaled to
  fit the window (letterboxed; on small screens crop the empty margins). Pre-render static
  art into offscreen layers using layered low-alpha deformed polygons plus paper grain.
- A whitewashed Spanish house: a tower with a terracotta roof, blue shutters and wall pots,
  a main house with a tiled roof, a chimney, an arched wooden door with blue-tile surround
  and steps, a balcony with iron railing, a barred window with flowers, exposed brick
  patches, two lanterns beside the door.
- In the courtyard: a round brick tree bed on the right of the house (no pot for the main
  tree; it stands clear of the facade), two small pots, a wooden bench/table with two jars
  and a little pink speaker, a wooden paint palette on the ground, a twig lying on the
  ground, a chalkboard easel on the left in front of the tower with a stick of chalk on its
  ledge, a cute watering can, and a little girl seen from behind holding a heart balloon on a
  string, with a bow on the back of her dress.
- Custom cursor: a butterfly in the current petal color, with sparkles. It perches (slow
  wings) on whatever tool you hold. When the pointer is away it flies to the girl's dress
  and rests there as her bow, full size, still fluttering.

GROWTH
- The garden starts bare. Pick up the watering can (click), click-and-hold to pour — water
  drops, splashes and a pouring sound. Watering the bed grows a bougainvillea tree
  (procedural segments with stable spring physics; grabbing a branch bends its parent
  chain). Watering each pot grows a small plant. Every tree uses a different petal color.
  Put the can back by clicking its spot or pressing Esc; it also returns by itself when
  everything is planted.
- Petals: shaking a branch hard detaches petals that flutter side to side as they fall
  (gravity wins; only a light breeze, stronger in autumn) and settle on the ground near
  their tree, then slowly fade; bare spots regrow.

A YEAR IN FOUR SEASONS (about 10–15 minutes)
- Spring, summer, autumn, winter. A counter at the top-left shows four season dots, the
  season name, "n/5" and one gem icon per gem of the current season (five every season). Each
  discovery pops a gem at that spot which then flies to the counter and lights it up.
- When a season's gems are all found, Oli says goodbye to it and the next season arrives
  on its own element, changing the garden as it passes (every effect stays subtle): summer
  on a soft, faint rising sun with a few gentle motes, whose light
  spreads out from the top-right corner, autumn on a gust of wind that blows leaves across
  from the left (trees sway in it), winter on a snowfall that settles from the top down.
  Trees, bushes, geraniums and window boxes change color where the element has reached;
  the color grade cross-fades, the counter resets and Oli introduces the next season.
- Trees and plants match the season; only spring has flowers on the trees. Spring:
  blossoms among fresh green leaves (about a third leaves), green bushes, red geraniums.
  Summer: warm golden grade, trees are full of deep green leaves and no flowers, brighter
  geraniums. Autumn: amber grade, every tree and bush turns orange, red and gold with
  leaf-shaped sprites, leaves drift down on their own. Winter: cool blue grade; as the
  snowfall passes the last leaves drop and the branches go bare, then snow slowly builds up
  over about 45 seconds as small clumps on the twigs and white lines along the tops of the
  flatter branches (shaking a branch knocks snow off). Plants turn grey-green with white
  dabs; snow covers the roofs, sills, steps and courtyard, and flakes keep falling.
- ?season=summer|autumn|winter starts there with everything before it done.

OLI THE OWL
- Oli perches on the tower roof from the start and speaks in a small speech bubble:
  an intro and a goodbye for each season, a short reaction to most discoveries, and a hint
  for the next gem when you tap him (which also sends the feather there). He only hoots at
  night and wears a red scarf in winter.

WALL-POT VISITORS
- Every season one of the seven blue wall pots on the tower hides a little visitor (a
  different pot each season); finding it is that season's fifth gem. Tapping a pot only
  puffs a few petals — the visitor needs an action. Spring: water the pots with the can,
  a nest of chicks; they burst into chirps when they appear, then peep in small bursts
  every few seconds with their beaks opening in time (`Music.chirp`). The nest stays in
  its pot for the rest of the year (`nook.nest`, drawn by `drawNest`, snow on its rim in
  winter; later visitors never pick that pot) and the chicks chirp whenever the cursor
  butterfly or a butterfly friend comes near. The chicks grow a little each season. In
  winter they wait out the long night; once it is light again (`nightLv < .3`, at least
  7 s into winter, or after 60 s regardless) `nest.fly` is set and they take off 0.8 s
  apart with a chirp each, small yellow birds flapping up and away to the upper left
  (`fledgling`). Oli says they are off somewhere warm; the empty nest keeps its snow. Summer: water the pots or rest the butterfly
  cursor still on one (holding the can is fine), a ladybug crawls onto the rim. Autumn: press and wiggle a pot, a field mouse nibbling an acorn.
  Winter: tap or brush the pots with the held twig (any part of it) to knock the snow off,
  a robin. Every pot the twig touches sheds a little snow.

LANTERNS (all year)
- Tap a lantern — it goes out and its fireflies fly off to hang in the tree canopy (or the
  balcony railing if there's no tree); relighting calls them back. One lantern off = sunset
  tint, both off = night with stars, moon and a warm balcony glow. Relighting one =
  early-morning tint, both = day. No sun, no "good morning" text.

SPRING GEMS (5)
1. Music: tap the speaker on the table. Nothing else starts music.
2. Shake: shake a branch.
3. Pots: grow every plant.
4. Butterfly: tap the palette, pick a new petal color (panel closes itself after picking).
5. Nest: the wall-pot visitor.

SUMMER GEMS (5)
- When summer begins, a smaller little sister (pigtails) walks out of the front door,
  facing us with a smiling face while she walks, then turns to face the house and holds
  the girl's left hand for the rest of the year. She dresses for the season: yellow dress
  in summer, orange dress with tights in autumn, red coat with scarf and knit hat in
  winter.
1. Kite: a colorful diamond kite with a bow tail is tangled in the right side of the main
   tree's canopy (the open courtyard side, `placeKite`); shake the tree
   and it rises into the sky on a string held in the girl's raised hand.
2. Sunflower: water a seed mound next to the little pot by the tower; a sunflower grows tall, with a
   bee circling it. It stays for the rest of the year (drooping in autumn, snow on it in
   winter).
3. Friends: tap the blue jar on the bench; its cork pops and three small butterflies fly
   out and follow the cursor (or flit around the girl). They fly away when winter comes.
4. Balloon: pick up the twig and pop the balloon with its tip (a mouse hover never pops
   it; in spring the balloon just bobs away from the twig). "pop!" + shards; the twig
   floats back home slowly; the girl's arm lowers to rest; the balloon does not return.
   Balloon is sky blue so it never matches the butterfly.
5. Ladybug: the wall-pot visitor.

AUTUMN GEMS (5)
1. Leaves: fallen leaves cover the courtyard. Pick up the rake (a held tool); its tines
   stay on the ground and follow the pointer with a little lag and chatter. A small heap of
   leaves already sits at the girl's feet. Sweep any way you like: every leaf the tines
   touch skitters along the stones onto that heap, which visibly grows into a pile. The tines
   also scratch settled petals off the stones. Raking makes a dry scraping sound with leaf
   crunches (when music is on). When most leaves are in, the rest follow, the whole
   courtyard is swept clean, she hops into the pile and leaves burst up; the trees then
   drop only the odd leaf.
2. Umbrella: tap the grey cloud (it has a sleepy face) to make it rain — streaks,
   splashes, a darker grade, pitter-patter sound; then tap the umbrella leaning by the wall
   and it opens in her raised hand, dome up like a real umbrella, tilted over her little
   sister so it keeps them both dry (rain splashes on the canopy). When the rain stops she lowers it and it
   leans by the wall again (and goes back up if it rains again). Tapping the umbrella
   before the rain makes Oli nudge you toward the cloud.
3. Pumpkin: water a patch of soil in front of the small pot; a vine grows and a pumpkin
   swells.
4. Night: turn both lanterns off; Oli hoots and flies one loop around the tree, then lands
   back on the tower.
5. Mouse: the wall-pot visitor.

WINTER GEMS (5)
- Winter opens on a long night: both lanterns are out and their fireflies rest in the bare
  tree. Relighting them brings the day back.
1. Snowman: tap three snowy mounds; each rolls a growing snowball that stacks at the right
   edge of the courtyard. Then tap the snowman to give him a carrot nose (he has coal eyes, twig arms and
   a blue scarf).
2. Hearth: tap the cold chimney; smoke puffs rise and every window glows warmly.
3. Lights: tap the balcony railing; colorful fairy lights string themselves along it and
   twinkle, brighter at night.
4. Name: pick up the chalk from the easel's ledge, tap the board, type a name (DOM input
   over the board), Enter. The name is written on the board in chalk. ?name= in the URL
   pre-fills it; signing it again still counts.
5. Robin: the wall-pot visitor.

HINTS
- A small feather roams the garden on a gentle path. It never hints right away. After a
  random 18–48 s without a discovery (and a few seconds idle), it flies to the next gem's
  location for 10 s (or to the tool that gem needs first).
- A cute pink alarm clock stands on the ground at the bottom-right of the house: a gold
  wedge and a single hand sweep down to zero, with a small seconds number; it rings and
  wobbles when the hint arrives. Tapping it rings the hint early.
- A little teddy bear peeks from behind the casa every 25–40 s (over the chimney, over
  the roof ridge, around the tower, or around the far corner above the clock), stays about
  5 s, then ducks back. The first time, Oli whispers that someone is peeking. Tapping him
  shows every gem still hidden this season, one every ~1.3 s, doing each gem's own action
  where possible, so the season can finish.

ENDING
- After the last winter gem: "Happy Daughter's Day," and the name on a second line come
  out of the chimney letter by letter. Each letter puffs out with a little smoke, starts
  small and tilted, and floats up and over to its place in the line, in a script font
  with a pink-to-orange gradient. Floating hearts follow, while Oli says "Every season she
  grew a little more, just like our tree." It fades after ~22 s and the snowy garden stays
  calm. The finished message sits just below the title, clear of it.

TITLE
- On load, "Casa de Gemas" in a script font with "a little garden of hidden gems" under
  it fades in at the top center (with padding so the swashes aren't clipped) and stays
  there for the whole game. On short screens (phones in landscape) it is smaller and sits
  in the top-right corner, clear of the house.

SOUND (Web Audio, all generated, silent until the speaker is tapped)
- Gentle Andalusian cadence Am–G–F–E: soft pads, plucked notes, chimes, convolution
  reverb. Petal-flutter notes only while your hand is shaking a branch (short notes that
  fade within ~0.6 s of letting go — never loop), water pour and rain, pop, owl hoots,
  lantern chimes. Unlock audio on touchend for iOS.

TOUCH
- Works with fingers: taps snap to the nearest interactive thing within fingertip reach
  (including every seasonal object), bigger branch/balloon hit areas, no double-tap zoom,
  no text selection, no stuck hover glows; in portrait show "turn your phone sideways for
  a bigger garden". Keep interactive objects inside the phone crop.

KEYS: Esc drops the held tool, M toggles music, R starts the year over from spring.
```

## House rules

- One file, no dependencies, no external images. Google Fonts (Great Vibes, Caveat) are optional; fall back to system script fonts.
- Keep interactions intentional — nothing important should trigger from a mere hover.
- Discovery over instruction: no on-screen instructions beyond the title and Oli's short lines; let Oli, the feather and the clock guide.
- Gentle over flashy: soft easing, warm colors, quiet sounds.
