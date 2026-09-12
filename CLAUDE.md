# minhtt159.github.io (hnimn.art)

Personal blog. Hugo + PaperMod, deployed by GitHub Actions on merge to `main`. Commands are in `README.md`.

## Content

- Frontmatter: `title`, `date` (in the past, or Hugo hides it), `draft: true` until Minh flips it, `description`, `summary`, `tags`. Archetype: `archetypes/posts.md`.
- ASCII punctuation in prose: `-` not dashes, `->` not arrows, `...` not the ellipsis character, straight quotes. Goldmark renders typographic quotes; the source stays greppable.
- Work material is genericised: no customer, product, internal system, registry, or supplier names. Numbers are real and rounded. Sources and the disclosure rule live in `../resume/docs/blog-inspirations.md`.
- Every claim with a number traces to a verified source (`../resume/docs/facts.md` or a commit). Recalled numbers do not go in.
- One name per thing across a post. Timeless phrasing: no "currently", "soon", "not yet".

## Colour scheme

Catppuccin Latte (light) and Mocha (dark), `assets/css/extended/theme.css`. Roles follow the upstream style guide: base page, mantle secondary panes, surface0 surface elements, text body, subtext0 labels, blue links, green/yellow/red for good/caution/bad, tints at 15%. Change a colour there and only there; nothing else in the repo carries a hex value.

## Fonts

No third-party font requests. Body text uses PaperMod's system stack. Code and SVG mono text use `--font-mono` from `theme.css`: JetBrains Mono, self-hosted under `static/fonts/` (regular and bold woff2, OFL licence file next to them, `local()` first so an installed copy is used). Adding a face means the same recipe: woff2 in `static/fonts/`, licence alongside, `@font-face` with `font-display: swap` in `theme.css`, two weights at most. Never load from fonts.googleapis.com.

## Diagrams

Inline SVG in the markdown, one per post unless the post cannot be understood without a second.

- Open with `<svg class="dg" viewBox="..." xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="<id>-t <id>-d">`, then `<title id="<id>-t">` and `<desc id="<id>-d">` that describe the mechanism in one or two sentences. `<id>` is unique per post; marker ids carry the same prefix.
- No `<style>`, no `fill=`, `stroke=`, or hex anywhere in the SVG. Colour comes only from the `svg.dg` classes in `theme.css`:
  - boxes: `box` (neutral), `box-a` blue primary flow, `box-b` green good/counted, `box-c` red bad/excluded, `box-d` yellow caution, `box-e` mauve external
  - lines: `ln`, `ln-a` .. `ln-e`; arrowhead paths inside `<marker>`: `ar`, `ar-a` .. `ar-e`
  - text: `t` heading, `s` subtitle/muted, `m` monospace
  - `zone` dashed outline for a region or a caption strip
  A colour encodes one meaning per diagram; two or three accents at most; neutral `box` for anything structural.
- No blank lines between `<svg` and `</svg>`. Goldmark closes the raw HTML block at the first blank line and renders the rest as text.
- Text inside the SVG is ASCII; `>` is written `&gt;`.
- Done when `hugo` builds, the diagram reads correctly with the theme toggle in both states, and `grep -c 'fill=\|stroke=\|#[0-9a-fA-F]\{3,6\}' <post>` returns 0 inside the SVG.

## Raster images

Only when the pixels are the point (a screenshot of a UI). Redraw charts and topologies as SVG.

- Path `static/img/<post-slug>/<name>.png`, referenced as `/img/<post-slug>/<name>.png`, under 300 KB, alt text that states what the reader should notice.
- Take screenshots in the dark scheme where the tool offers one, and crop to the panel that carries the claim.
- Nothing in a screenshot may show a customer, hostname, account id, or internal URL. Check before committing.

## Before opening a PR

1. `hugo --minify -D` builds with no `ERROR`.
2. `grep -rnP '[^\x00-\x7F]' content` returns nothing.
3. Disclosure grep over `content/` for the names in `blog-inspirations.md` returns nothing.
4. Preview both theme states: home, the post, the diagram.
5. Cold-read: a fresh subagent with only the post text, briefed as a senior SRE, lists open questions, undefined terms, and arithmetic mismatches. Fix, then hand to Minh.
