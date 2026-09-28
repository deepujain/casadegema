# Casa de Gemas 🏡✨

*A little garden of hidden gems.*

Casa de Gemas is a tiny interactive watercolor world made as a Happy Daughter's Day gift: a whitewashed Spanish house, a bare courtyard, a watering can, and 8 hidden gems waiting to be discovered. Find them all and a message rises from the chimney — with *her* name in it.

**Play it:** https://1xaispark.com/games/casa-de-gemas · mirror on GitHub Pages: https://deepujain.github.io/casadegema/

![Casa de Gemas](preview.png)

It is one self-contained `index.html` — hand-drawn Canvas 2D art, spring physics for every branch and petal, and music generated live with the Web Audio API. No images, no libraries, no build step.

---

## How to play

The garden starts bare. Pick up the watering can and water the tree bed — a bougainvillea grows. Then explore. There are **8 gems**; each one pops up where you found it and flies to the counter in the top-left.

| Gem | How to find it |
| --- | --- |
| 🔊 Music | Tap the little pink speaker on the table. |
| 🌸 Shake | Grab a branch and shake it — every petal falls on its own. |
| 🌱 Pots | Water every pot so all the plants grow. |
| 🦋 Butterfly | Tap the paint palette on the ground and pick a new petal color. Your butterfly cursor changes color too. |
| 🏮 Lantern | Tap a lantern by the door. It goes out and its fireflies drift off to the tree (or the balcony railing). |
| 🦉 Night | Turn both lanterns off: sunset, then night — and an owl flies in. Relight them for early morning; the owl leaves slowly. |
| 🎈 Balloon | Pick up the twig lying on the ground and pop the girl's balloon with it. |
| 🖍️ Name | Pick up the chalk from the table, tap the easel, and write a name. |

Find all 8 and "Happy Daughter's Day, *name*" rises from the chimney with hearts, then gently fades.

**Stuck?** A feather roams the garden. When the little alarm clock at the corner of the house runs down (a random 18–48 seconds), it rings and the feather flies to your next gem. Tap the clock to ring it early.

**Keys:** `Esc` puts down whatever you're holding · `M` toggles music · `R` replants the garden.

**Phones:** touch works — taps snap to the nearest thing you can interact with. Landscape gives the biggest garden.

### Personalize it

Add a name to the link and it's already written on the easel and in the final message:

```
https://1xaispark.com/games/casa-de-gemas?name=Shanaya
```

## Run it locally

It's a single file. Open `index.html` in any modern browser, or serve the folder:

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

To host it, drop `index.html` anywhere that serves static files (GitHub Pages, Netlify, S3, a Next.js `public/` folder, …).

## How it's built

- **Rendering:** Canvas 2D on a fixed 1200×950 logical stage scaled to the window. The house, sky and ground are pre-rendered once into offscreen layers using a watercolor "wash" technique (layered, low-alpha, randomly deformed polygons + paper grain). Small screens crop the empty margins.
- **Tree:** procedurally grown segments with spring physics; grabbing a branch bends its whole parent chain. Petals are attached along segments and detach when shaken hard, then flutter down and settle on the ground.
- **Day cycle:** multiply-tinted overlays that ease between day, dusk, night and dawn based on how many lanterns are off.
- **Music:** fully generative Web Audio — an Andalusian cadence (Am–G–F–E) with pads, plucks, chimes and a convolution reverb, plus petal-flutter, pouring water, pops, owl hoots and lantern chimes. Silent until you tap the speaker.
- **Gems:** a single `SURPRISES` array drives the counter, the fly-to-counter effect and the feather's hints.

## Made with Claude

Casa de Gemas was built entirely through conversation with **Claude Opus 5.5** in Cursor — no code written by hand. It was inspired by a post from [@thebuggeddev](https://x.com/thebuggeddev) showing Claude building "a little artistic world in pure JavaScript where a tree grows over time, you can grab and shake its branches, and the leaves fall naturally."

## License

[MIT](LICENSE) — make one for someone you love.
