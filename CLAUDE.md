# Publication Workflow — Todd W. Bucy Research Blog

You are helping author and publish content on a Hugo + Hugo Book static site
deployed to GitHub Pages. Read this file at the start of every session before
making changes.

## What this repo is

- **Site:** the canonical home for Todd W. Bucy's research and practitioner
  writing — long-form papers, framework documents, and shorter blog posts.
- **Stack:** Hugo (extended) + Hugo Book theme (git submodule under
  `themes/book/`) → static HTML → GitHub Pages.
- **Live URL:** https://toddwbucy.github.io/toddwbucy_research_blog/
- **Repo:** `git@github-toddwbucy:toddwbucy/toddwbucy_research_blog.git`
  (uses the `github-toddwbucy` SSH alias defined in `~/.ssh/config`; do not
  use plain `github.com`).

## Content types and where they live

| Type      | Path                       | Layout                  | Archetype              |
|-----------|----------------------------|-------------------------|------------------------|
| Blog post | `content/blog/YYYY/MM/`    | `_default/single.html`  | `archetypes/blog.md`   |
| Paper     | `content/papers/<series>/` | `_default/single.html`  | `archetypes/papers.md` |
| Framework | `content/framework/`       | `_default/single.html`  | `archetypes/framework.md` |
| Project   | `content/projects/`        | `_default/single.html`  | `archetypes/projects.md` |
| Archive   | `content/archive/`         | `_default/single.html`  | (manual)               |

Use `hugo new content <path>/<slug>.md` so the right archetype is applied. The
archetype defines the front-matter schema for each content type — read it
before adding new fields.

## The publication workflow

### Step 1 — Drafting

```bash
# Blog post
hugo new content blog/2026/03/my-new-post.md

# Long-form paper inside a series
hugo new content papers/constraint-organization/paper-3-some-title.md

# Framework document
hugo new content framework/conveyance-hypothesis.md
```

The new file opens with the correct YAML front matter and `draft: true`.

### Step 2 — Local preview

```bash
hugo server --buildDrafts --bind 127.0.0.1 --port 1313
```

Open http://localhost:1313/. Hot reload fires on every save. The dev server
includes drafts; production deploys do not (see Step 4).

### Step 3 — Authoring

- Write Markdown. Use Hugo Goldmark conventions; `unsafe = true` is on so
  inline HTML works.
- For math: `{{< katex >}}` shortcode (theme-provided).
- For "cite this" block on papers/framework: `{{< cite >}}` shortcode at
  the bottom; reads `cite.bibtex` from front matter.
- For framework version banners: `{{< version-banner >}}` at the top;
  reads `version`, `status`, `supersedes`, `superseded_by`.

### Step 4 — Publishing

1. Set `draft: false` (or remove the line) and `status: "published"`.
2. Commit with a clear message.
3. `git push origin main`.
4. The GitHub Actions workflow at `.github/workflows/deploy.yml` builds and
   deploys in ~30 seconds. No manual deploy steps.

### Step 5 — Cross-posting (blog-first articles only)

For articles authored on the blog first:
1. Wait for the GitHub Pages deploy to complete.
2. On Medium/Stackademic: New Story → "..." menu → **Import a story**.
3. Paste the canonical blog URL. The platform sets `rel="canonical"` back
   to the blog automatically.
4. Add a footer linking back to the blog.
5. Paste the resulting Medium/Stackademic URL into the original
   Markdown's `medium_url:` front-matter field, commit, push.

## Document conversion (DOCX/ODT/etc.) — fidelity rule

**When the user hands you a docx/doc/odt to convert into a blog post or
paper, your job is conversion, not editing.** Apply structural transforms
only. Keep prose verbatim — every word, every comma, every semicolon.

### Allowed transforms

- Heading-level demotion (e.g. H1 → H2, since the title moves to front matter)
- Image syntax: `<img src="..." />` → Markdown `![](path)`
- Image-path remapping into `/images/blog/<slug>/`
- Code block conversion (e.g. blockquote-styled code → fenced ` ```python `)
- Strip `<u>` tags from links (Word's hyperlink character-style artifact)
- Strip `\#`, `\-` etc. table-cell escape artifacts from pandoc
- Add front matter

### Forbidden transforms (these are editorial, not structural)

- Substituting em-dashes (`—`) for hyphens, commas, or semicolons
- Substituting words ("PhD candidate" → "PhD student", "MIRAS" → "Miras")
- Restructuring sentences or paragraphs
- "Smoothing" or "polishing" prose
- Adding clarifying phrases that weren't in the source

### Fidelity guardrail (run before committing)

After converting, diff the source's plain text against your output's plain
text. If words appear in your version that weren't in the source, you've
edited rather than converted.

```bash
# 1. Extract source as plain words
pandoc -f docx -t plain --wrap=none <source.docx> 2>/dev/null \
  | tr -s '[:space:]' ' ' | grep -oE '[A-Za-z]+' | sort -u > /tmp/src.txt

# 2. Extract your committed markdown as plain words (strips front matter)
sed '/^---$/,/^---$/d' content/blog/<path>/<slug>.md \
  | pandoc -f gfm -t plain 2>/dev/null \
  | tr -s '[:space:]' ' ' | grep -oE '[A-Za-z]+' | sort -u > /tmp/mine.txt

# 3. Words in mine but not in source = vocabulary drift (must be empty
#    except for title words moved to front matter)
comm -13 /tmp/src.txt /tmp/mine.txt

# 4. Em-dash count comparison — must match exactly
echo "source: $(grep -c '—' source plain text) | mine: $(grep -c '—' your.md)"
```

If the diff shows added vocabulary or em-dash drift, fix it before committing.

## Image processing

Word docs often embed images at unreasonable sizes (multi-MB PNGs).
Compress before committing.

```bash
# 1. Extract images via pandoc to a scratch dir
pandoc -f docx -t gfm --wrap=none --extract-media=./_tmp_extract \
  <source.docx> -o /tmp/converted.md

# 2. Batch-convert to web-friendly JPGs (1200px max width, q=85, strip metadata)
mkdir -p static/images/blog/<slug>
for f in _tmp_extract/media/image*.png; do
  n=$(basename "$f" .png)
  convert "$f" -resize 1200x -quality 85 -strip "static/images/blog/<slug>/${n}.jpg"
done
rm -rf _tmp_extract

# 3. Reference in markdown as:
#    ![](/images/blog/<slug>/imageN.jpg)
#    The render-image hook at layouts/_markup/render-image.html handles
#    the GitHub Pages baseURL prefix automatically.
```

For images that are diagrams/screenshots with text (sharp-edge content),
keep PNG instead of JPG — `convert source.png -resize 1200x out.png`.

## Canonical URL logic

The `canonical_url:` front-matter field tells search engines and humans
where the *original* version of an article lives. Decision tree:

| Scenario                                      | `canonical_url`              |
|-----------------------------------------------|------------------------------|
| Article authored on the blog first (default)  | leave blank                  |
| Article first published on Medium/Stackademic | set to the original venue URL |
| Article first published on a journal/preprint | set to the journal URL       |

When set, two things happen automatically:

1. `<link rel="canonical" href="...">` in `<head>` points to the origin
   (overrides Hugo Book's default self-canonical via the
   `layouts/_partials/docs/html-head.html` override).
2. A visible "Originally published at <host>" callout renders above the
   article abstract (driven from the same field by
   `layouts/_default/single.html`).

This is for already-published-elsewhere articles where retroactively
changing canonical on the original venue would feel like rewriting history.
For blog-first articles, leave the field blank — the blog self-canonicals
and the cross-post platform's canonical setting points back to the blog.

## Versioning a framework document

Framework docs are living references. To replace one with a revised version
while preserving citation stability:

1. Move the current version to `content/archive/<slug>-<YYYY-MM-DD>.md`.
2. In the archived copy, set `status: "superseded"` and
   `superseded_by: "<URL of new version>"`.
3. `hugo new content framework/<slug>.md` to create the new version.
4. In the new file, bump `version: "v2"` and set
   `supersedes: "<URL of archived copy>"`.

The `{{< version-banner >}}` shortcode at the top of each framework/archive
document renders the version status automatically based on these fields.

## Site-level conventions worth remembering

- **Author byline** is centralized in `hugo.toml` under `params.author`.
  Don't hard-code the author name in front matter unless overriding for a
  specific article.
- **Tags** are free-form but should reuse existing tag slugs where applicable
  (check `content/blog/**/*.md` and `content/papers/**/*.md` for existing
  tags before inventing new ones).
- **Image paths in markdown** start with `/` (site-absolute). The render-
  image hook prefixes the baseURL path on subdirectory deploys; do not
  prepend the deploy path manually.
- **No emojis** in committed content unless the user explicitly asks for one.
- **TOML gotcha** in `hugo.toml`: scalar settings before `[table]` blocks.
  Once you open a `[section]` header, every subsequent line is scoped to
  that section until the next header. This bit us once.

## Quick reference

```bash
# Local preview (with drafts)
hugo server --buildDrafts --bind 127.0.0.1 --port 1313

# Production-style local build (catches subdirectory-deploy issues early)
hugo --gc --baseURL "https://toddwbucy.github.io/toddwbucy_research_blog/"

# Deploy to live site
git push origin main    # auto-deploys via GitHub Actions in ~30s

# Watch the deploy
gh run watch --repo toddwbucy/toddwbucy_research_blog --exit-status

# Verify a deploy
curl -sI https://toddwbucy.github.io/toddwbucy_research_blog/ | head -3
```

## Things NOT to do

- Don't push directly to `main` from another GitHub identity. The repo's
  `.git/config` is set up with `user.name = Todd W. Bucy` and the SSH
  alias `github-toddwbucy`. Verify with `git config user.name`.
- Don't introduce stylistic edits to user-authored prose during conversion
  (see fidelity rule above).
- Don't add planning/decision/analysis Markdown files to the repo unless
  the user explicitly requests them. Work from conversation context.
- Don't commit `.docx`, `.doc`, `.odt` source files — they're gitignored
  for a reason. The Markdown version under `content/` is canonical.
- Don't `--minify` during local preview — minification strips HTML
  comments which makes debugging templates harder. Only the deploy
  workflow uses `--minify`.
