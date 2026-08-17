# Todd W. Bucy — Research Blog

Source for the personal publication blog at <https://toddwbucy.github.io/toddwbucy_research_blog/> (Phase 1) — eventually to migrate to a custom domain on Cloudflare Pages.

Stack: [Hugo](https://gohugo.io) + [Hugo Book theme](https://github.com/alex-shpak/hugo-book) (vendored as a git submodule under `themes/book/`) → static HTML → GitHub Pages.

## One-time setup on a new machine

### 1. SSH identity

The repo is cloned through the `github-toddwbucy` SSH alias rather than plain
`github.com`, so pushes always go out under the right GitHub identity. That
alias lives in `~/.ssh/config`, which a reinstall does **not** carry over —
recreate it before cloning:

```bash
cat >> ~/.ssh/config <<'EOF'

Host github-toddwbucy
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
EOF
chmod 600 ~/.ssh/config

# Verify — should greet you by username
ssh -T git@github-toddwbucy
```

If `~/.ssh/id_ed25519` is also gone, generate a fresh key and register it at
<https://github.com/settings/keys> first.

### 2. Clone with the theme submodule

```bash
git clone --recurse-submodules \
  git@github-toddwbucy:toddwbucy/toddwbucy_research_blog.git
cd toddwbucy_research_blog

# Commit identity — also not inherited from a bare reinstall
git config user.name  "Todd W. Bucy"
git config user.email "todd@bucy-medrano.me"
```

If the submodule wasn't pulled (you forgot `--recurse-submodules`), run:

```bash
git submodule update --init --recursive
```

### 3. Install Hugo (extended)

The theme compiles its own Sass via `css.Sass`, so the **extended** build is
mandatory — the plain build fails at render time. Pin it to the version the
deploy workflow uses (`HUGO_VERSION` in `.github/workflows/deploy.yml`,
currently **0.161.1**) so local previews match CI exactly.

**Portable — any distro, no root** (recommended):

```bash
VER=0.161.1   # keep in sync with .github/workflows/deploy.yml
curl -fsSL "https://github.com/gohugoio/hugo/releases/download/v${VER}/hugo_extended_${VER}_linux-amd64.tar.gz" \
  | tar xz -C /tmp hugo
install -Dm755 /tmp/hugo ~/.local/bin/hugo
```

Ensure `~/.local/bin` is on your `PATH`.

**Arch / CachyOS** — convenient, but the rolling package drifts ahead of the
version CI builds with:

```bash
sudo pacman -S hugo
```

**Debian / Ubuntu / Mint** — the apt package is too old for Hugo Book's
layouts; install the release `.deb` directly:

```bash
VER=0.161.1
curl -fsSL -o /tmp/hugo.deb \
  "https://github.com/gohugoio/hugo/releases/download/v${VER}/hugo_extended_${VER}_linux-amd64.deb"
sudo dpkg -i /tmp/hugo.deb
```

### 4. Verify

```bash
hugo version
# Expect: hugo v0.161.1+extended linux/amd64 ...
#         The "+extended" suffix is what matters.

hugo --gc --minify --baseURL "https://toddwbucy.github.io/toddwbucy_research_blog/"
# Expect: a clean build (~110 pages), no errors.
```

Note that `hugo server` serves at `http://localhost:1313/` — it overrides
`baseURL`, so the `/toddwbucy_research_blog/` subpath does *not* apply
locally. Only production builds carry the path prefix.

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
