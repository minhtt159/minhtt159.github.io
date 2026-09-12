# hnimn.art

Personal blog. Hugo + [PaperMod](https://github.com/adityatelange/hugo-PaperMod), deployed to GitHub Pages by `.github/workflows/hugo.yml` on every merge to `main`.

```bash
git submodule update --init          # once, pulls the theme
hugo server -D                       # preview with drafts at http://localhost:1313
hugo new posts/<slug>.md             # new post from archetypes/posts.md, starts as draft: true
```

Flip `draft: false` to publish. Every post: ASCII punctuation only, one inline SVG max, no customer, internal system or supplier names.
