# ccbedair.github.io

Personal portfolio site — Senior Security Engineer.

## Local Development

### Prerequisites
- [Hugo](https://gohugo.io/installation/) (extended edition, v0.147+)

### Run locally
```
git clone https://github.com/CCbedair/ccbedair.github.io.git
cd ccbedair.github.io
hugo server -D
```

Site available at http://localhost:1313. The `-D` flag shows draft content.

### Build for production
```
hugo --minify
```

Output in `./public/`.

## Structure
- `content/` — All pages and articles (markdown)
- `assets/css/` — Styling (edit `variables.css` to change colors/fonts)
- `layouts/` — Hugo templates
- `.github/workflows/` — Automated deployment to GitHub Pages

## Customization
- **Colors:** Edit `assets/css/variables.css`
- **Fonts:** Change `--font-body` and `--font-mono` in `variables.css`
- **Navigation:** Edit `[menu]` section in `hugo.toml`
- **Content:** Add/edit markdown files in `content/`
- **Publish a draft:** Change `draft: true` to `draft: false` in the page's front matter
