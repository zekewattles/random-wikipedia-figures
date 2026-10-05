# Random Wikipedia Figures

A single web page that pulls random captioned images from English Wikipedia and lays them out as a grid of Wikipedia-style thumbnails that exactly fills the browser window.

Everything is in one file, `random-wikipedia-figures.html`. There is no build step, no dependencies and no backend.

## Running it

Open `random-wikipedia-figures.html` in a browser. It needs an internet connection to reach Wikipedia.

To host it on GitHub Pages, rename the file to `index.html` and either:

- put it in a folder of an existing Pages site (for example `random-wiki-figs/index.html`), or
- put it at the root of its own repo and turn on Pages for the `main` branch.

## What it does

- **Captioned images only.** It reads random articles and keeps thumbnails that have a real caption. Infobox images, galleries, images inside tables and anything uncaptioned are skipped. It takes at most one image per article so the mix stays wide.
- **Fills the viewport.** Rows get random heights and a random number of tiles. Each tile's width follows its image's proportions. Nothing scrolls.
- **Shuffle.** The Shuffle button or the space bar fetches a fresh set.
- **Refresh every hour.** On by default. A fresh set loads at the top of each clock hour. If the computer sleeps through an hour mark, the page refreshes when it is next visible. Unticking the box is remembered in that browser.
- **Resizing.** Changing the window size re-flows the same images into a new grid without a fade, and fetches more only if the new grid needs them.
- **Links.** Links inside captions work, and clicking an image opens its article. Both open in a new tab.
- **Controls fade out.** The control bar disappears 1.5 seconds after the pointer stops moving, so the page can sit as a display. Moving the pointer or pressing a key brings it back.
- **Light and dark.** Follows the system colour scheme.

## Sizes

Caption type is 14px (`0.875rem`) on desktop and 13px (`0.8125rem`) on screens 600px wide or narrower. Row heights and spacing scale from that one value, the `--size` custom property at the top of the stylesheet.

## How it works

1. `action=query&generator=random` on the MediaWiki API returns 50 random article titles. Articles under 3,500 bytes are dropped, since stubs rarely have a thumbnail.
2. `action=parse&prop=text` returns each article's rendered HTML. The page parses it with `DOMParser` and collects every `<figure>` that has an image and a non-empty `<figcaption>`.
3. Captions are rebuilt from a small whitelist of tags (links, italics, bold, superscript, subscript), so nothing from the fetched HTML runs in the page.
4. A layout plan picks the number of rows and their relative heights at random, then fills each row with tiles until their combined aspect ratios span the row width. Flexbox does the final fitting.

Both API calls use `origin=*`, which is what lets a static page call Wikipedia directly from the browser.

## Known trade-offs

- **Cropping.** Photos are cropped slightly (`object-fit: cover`) so rows fit edge to edge. PNG, SVG and GIF images, which are usually diagrams, are shown whole on a white backing.
- **Long captions** are cut off at four lines. The full text is in the tooltip.
- **First load takes a few seconds.** Finding enough captioned images means reading many articles; most random articles have none.
- **Random is random.** Expect a lot of footballers, railway stations and moths.
- **Licensing.** Images are loaded straight from Wikimedia and are not checked for licence. Some are non-free (logos, album covers). That matters if a grid is ever reproduced outside the page, for example as a print.

## Credits

Text and images come from [Wikipedia](https://en.wikipedia.org) and [Wikimedia Commons](https://commons.wikimedia.org) and remain under their own licences.
