# CLAUDE.md

Notes for working on this repo.

## What this is

Random Wikipedia Figures: one static HTML file that fetches random captioned images from English Wikipedia and lays them out as a viewport-filling flexbox grid of Wikipedia-style thumbnails. See `README.md` for the user-facing description.

## Ground rules

- **One file.** All markup, CSS and JS live in `random-wikipedia-figures.html` (or `index.html` if it has been renamed for GitHub Pages). No build step, no framework, no dependencies, no backend. Keep it that way unless asked.
- **No Jekyll front matter.** The file may be served from a Jekyll site. It must not start with a `---` block, or Jekyll will run it through Liquid.
- **Captioned images only.** Never show an image without a real caption. This is the core requirement.
- **The grid fills the viewport and never scrolls.**
- **The look is Wikipedia's thumbnail vernacular:** Arial, light grey thumb box with a 1px border, blue links. Don't restyle it into something else.

## Structure of the script

One IIFE. In order:

| Part | Role |
| --- | --- |
| `api(params)` | `fetch` wrapper for `https://en.wikipedia.org/w/api.php`. Always sends `origin=*` (required for CORS) and `formatversion=2`. |
| `randomTitles()` | `generator=random`, 50 titles, drops articles under 3,500 bytes. |
| `cleanInto(src, dest)` | Rebuilds a caption from a tag whitelist (`A`, `I`, `B`, `EM`, `STRONG`, `SUP`, `SUB`). Everything else is unwrapped or dropped. |
| `captioned(title)` | `action=parse&prop=text`, parsed with `DOMParser`. Returns every `<figure>` with an `<img>` and a non-empty `<figcaption>`, skipping anything inside `table, .infobox, .navbox, .metadata, .gallery` and images under 120×80. |
| `fill(need, id)` | Fetch loop. Six concurrent workers per batch of titles, one random figure per article, until `pool` holds `need` items or 40 batches have run. |
| `plan()` | Picks the row count and random row weights from the viewport and type size. Returns `{ rows, need }`. |
| `tile(item, delay)` / `render(rows, items)` | Build the DOM. `render` returns leftovers, which go back into `pool`. |
| `build(fresh)` | The one entry point. `fresh = true` fetches a new set (load, Shuffle, hourly). `fresh = false` re-flows what is on screen (resize) and tops up only if needed. |
| Hourly refresh | `scheduleHourly()` / `hourTick()`, plus a `visibilitychange` catch-up. |
| Idle controls | `wake()` hides `#controls` 1.5s after the last pointer or key event. |

State: `pool` (fetched, not shown), `shown` (on screen), `seen` (image URLs already used), `run` (a counter that cancels stale async work; every async step checks `id === run`).

## Things that are easy to break

- **Never inject fetched HTML directly.** Captions come from Wikipedia's parser output. They go through `cleanInto` and nothing else. No `innerHTML` with fetched content.
- **`run` token.** Any new async path that touches `pool`, `grid` or `statusEl` must check `id === run` after each `await`.
- **Sizing is one variable.** `--size` is the caption font size: `0.875rem` by default, `0.8125rem` at `max-width: 600px`. `--gap`, `--pad` and, in JS, row heights all derive from it. `plan()` reads the computed body font size and divides by `BASE` (11.5, the size the 200–250px row-height range was tuned at). Change the type size through `--size`, not by editing those numbers.
- **Fresh versus re-flow animation.** `build` toggles `.still` on `#grid` for re-flows, which disables the tile fade-in. Resizes must not animate.
- **Aspect ratios come from the `width` and `height` attributes** in the parser output, so the layout is planned before any image loads. Tiles get `flex-grow` equal to the clamped aspect ratio (0.55 to 2.4).
- **`object-fit`.** `cover` for photos. `contain` on a white plate for `.diagram` tiles (PNG, SVG, GIF by file extension).
- **Image URLs.** Use the `src` and `srcset` exactly as the parser gives them, with the protocol added. Wikimedia only serves certain thumbnail widths; don't construct other sizes.
- **Theme tokens.** Every colour is a custom property on `:root`, redefined in the `prefers-color-scheme: dark` block. No literal colours in component rules, apart from the white plate behind diagrams, which is deliberate in both themes.
- **localStorage** is wrapped in try/catch (`store.get` / `store.set`, keys prefixed `rc-`). The only key in use is `rc-hourly`; the checkbox is on unless that key is `'0'`.

## Testing

There is no test suite. The Wikipedia API may not be reachable from a sandboxed environment, so the practical check is Playwright with network routes stubbed:

- `generator=random` requests: return `{ query: { pages: [{ title, length }] } }`.
- `action=parse` requests: return `{ parse: { title, text } }` where `text` contains `<figure><a><img src="//upload.wikimedia.org/…" width height></a><figcaption>…</figcaption></figure>`. Include some figures with empty captions to confirm they are filtered out.
- `upload.wikimedia.org` requests: return a small placeholder SVG.

Check at a desktop size and at about 400px wide: no empty captions on screen, the last row's bottom edge sits at the viewport bottom minus padding, and `document.documentElement.scrollWidth` equals the viewport width.

Stubbed tests do not prove the live API behaves as assumed. When the real API is reachable, also load the page for real and confirm figures appear.

## Possible next steps

Nothing here is committed to; these came up while building.

- Bias away from dull results by sampling from a category or seed list instead of pure random.
- Skip non-free images if a grid is going to be reproduced outside the page.
- A language switch (the API host is the only English-specific part).
