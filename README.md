# Todd W. Bucy — Research Blog

Source for the personal publication blog at <https://toddwbucy.github.io/toddwbucy_research_blog/> (Phase 1) — eventually to migrate to a custom domain on Cloudflare Pages.

Stack: [Hugo](https://gohugo.io) + [Hugo Book theme](https://github.com/alex-shpak/hugo-book) (vendored as a git submodule under `themes/book/`) → static HTML → GitHub Pages.

## One-time setup on a new machine

```bash
# 1. Clone with the theme submodule
git clone --recurse-submodules git@github-toddwbucy:toddwbucy/toddwbucy_research_blog.git
cd toddwbucy_research_blog

# 2. Install Hugo extended ≥ 0.158
#    (Mint/Ubuntu's apt version is too old for Hugo Book's modern layouts.
#    Grab the latest extended .deb directly:)
LATEST=$(curl -s https://api.github.com/repos/gohugoio/hugo/releases/latest \
         | grep -oP '"tag_name":\s*"\K[^"]+')
VER=${LATEST#v}
curl -fsSL -o /tmp/hugo.deb \
  "https://github.com/gohugoio/hugo/releases/download/${LATEST}/hugo_extended_${VER}_linux-amd64.deb"
sudo dpkg -i /tmp/hugo.deb

# 3. Verify
hugo version
# Expect: hugo vX.Y.Z+extended ...
```

If the submodule wasn't pulled (you forgot `--recurse-submodules`), run:

```bash
git submodule update --init --recursive
```

## Authoring workflow

### Drafting a new piece

```bash
# Blog post (practitioner article, often cross-posted to Medium)
hugo new content blog/2026/05/my-new-post.md

# Long-form paper (lives at /papers/<slug>/, BibTeX exported)
hugo new content papers/constraint-organization/paper-3-some-title.md

# Framework document (versioned living reference)
hugo new content framework/conveyance-hypothesis.md

# Project page
hugo new content projects/weavertools.md
```

`hugo new` reads the matching archetype from `archetypes/` and produces a Markdown file with the correct front matter for the content type.

### Previewing locally

```bash
hugo server --buildDrafts
```

Open <http://localhost:1313/>. The server hot-reloads as you save Markdown files. `--buildDrafts` includes pieces with `draft: true` in the rendered site (otherwise they're hidden).

### Publishing

1. Set `draft: false` (or remove the line) and verify `status: "published"` in front matter.
2. Commit and push:
   ```bash
   git add content/papers/<slug>.md
   git commit -m "Publish: <title>"
   git push
   ```
3. The GitHub Actions workflow at `.github/workflows/deploy.yml` builds and deploys the site to GitHub Pages, typically within 30–60 seconds.

### Cross-posting to Medium

1. After the canonical version is live on the blog, open [Medium](https://medium.com).
2. New Story → "..." menu → **Import a story**.
3. Paste the canonical blog URL. Medium imports the content and sets `rel="canonical"` pointing back to the blog automatically.
4. Verify the canonical link is in place by viewing source on the imported Medium post.
5. Add a short footer on Medium: "Originally published at \<blog URL\>. Related work and the full research thread live on the blog."
6. Paste the resulting Medium URL into the original Markdown's `medium_url` front-matter field, commit, push.

### Versioning a framework document

Framework documents are living references. To replace one with a revised version while preserving citation stability:

```bash
# 1. Move the current version into /content/archive/ with the date appended:
mv content/framework/conveyance-hypothesis.md \
   content/archive/conveyance-hypothesis-2026-05-04.md

# 2. In the archived copy, set:
#      status: "superseded"
#      superseded_by: "/framework/conveyance-hypothesis/"

# 3. Create the new version:
hugo new content framework/conveyance-hypothesis.md

# 4. In the new file, set:
#      version: "v2"
#      supersedes: "/archive/conveyance-hypothesis-2026-05-04/"
```

Both pages render an automatic version banner via the `{{ < version-banner > }}` shortcode (place it as the first line under the title in each framework/archive document).

### Citation block

In any paper or framework document, add this as the last line to render a "Cite this" block with the BibTeX from front matter:

```markdown
{{ < cite > }}
```

(Strip the spaces inside the braces — they're added here so this README itself doesn't trigger Hugo template rendering.)

The shortcode reads `cite.bibtex` from front matter; if absent it generates a minimal `@article{...}` entry from title, author, year, and permalink.

## Repository structure

```
.
├── archetypes/             Templates used by `hugo new`
├── content/                The actual content
│   ├── blog/               Practitioner articles
│   ├── papers/             Long-form papers (organized by series)
│   ├── framework/          Living framework documents
│   ├── projects/           Project-specific writing
│   ├── archive/            Superseded versions, citation-stable
│   └── about/              About page
├── layouts/                Site-specific layout overrides
│   └── _shortcodes/        Custom shortcodes (cite, version-banner)
├── static/                 Static assets (images, files served as-is)
├── themes/book/            Hugo Book theme (git submodule)
├── hugo.toml               Site configuration
├── .github/workflows/      GitHub Actions — builds and deploys to Pages
└── public/                 Build output (gitignored)
```

## Migration to Cloudflare Pages

When a custom domain is purchased and ready:

1. Sign up for Cloudflare, create a Pages project, point it at this repo.
2. Configure build command: `hugo --gc --minify --baseURL "$CF_PAGES_URL"`
3. Output directory: `public`
4. Environment variable: `HUGO_VERSION = 0.161.1` (or whatever current is)
5. Add the custom domain in the Pages project settings; configure DNS at the registrar to point to Cloudflare.
6. Once Cloudflare is serving correctly, delete `.github/workflows/deploy.yml` and disable GitHub Pages in the repo settings.

Everything else (content, theme submodule, archetypes, layouts, hugo.toml) is host-agnostic.

## License

Unless otherwise noted in a specific document's front matter, content is licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The site infrastructure (templates, shortcodes, config) is in the public domain — do whatever you want with it.
