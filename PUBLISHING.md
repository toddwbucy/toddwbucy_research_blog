# Publishing to the Research Blog from an External Directory

You are helping the user publish a finished draft to their personal research
blog. The user is currently in some other working directory (most likely
where they drafted the article — a project repo, an `~/olympus/...` folder,
etc.). This file is the runbook for getting that draft into the blog repo
and live on the public site.

The blog repo: `/home/todd/git/toddwbucy-research-blog/`
Live URL: https://toddwbucy.github.io/toddwbucy_research_blog/

You will work primarily *inside* the blog repo, but the source draft lives
elsewhere and stays where it is. Do not copy the source `.docx` (or other
binary source) into the blog repo.

---

## What you need from the user

Before doing anything, confirm these. If the user already provided some,
note them; ask only for what's missing.

1. **Source file path** — absolute path to the draft (e.g.,
   `/home/todd/olympus/NestedLearning/docs/some-paper.docx`).
2. **Content type** — `blog` (practitioner article), `paper` (long-form
   research), `framework` (versioned framework document), or `project`
   (project page). If unclear, ask.
3. **Series** (papers only) — e.g., `constraint-organization`,
   `conveyance-hypothesis`. Determines the URL path
   `/papers/<series>/<slug>/`.
4. **Slug** — URL-safe filename without extension. Default to a slug
   derived from the title (lowercase, dashes, no punctuation), but offer
   it for confirmation.
5. **Publication date** — for blog posts, the date the article should
   show. For backdated migrations from another platform, ask for the
   original publication date. Convert relative dates ("last Thursday")
   to absolute dates.
6. **Canonical URL decision** — see "Canonical URL" section below.
7. **Tags** — propose 4–8 based on content; ask user to confirm or add.
8. **Abstract** — pull from the draft's first paragraph or ask. Keep
   to 2–4 sentences.

Do not invent any of these. Ask if unclear.

---

## Step 1 — Verify the environment

Run these checks first. Fix any that fail before proceeding.

```bash
# Blog repo exists and is on main with clean working tree
git -C /home/todd/git/toddwbucy-research-blog status

# Right git identity (must be `Todd W. Bucy`, not the global r3d91ll user)
git -C /home/todd/git/toddwbucy-research-blog config user.name

# SSH alias resolves and authenticates as the right account
ssh -T -o BatchMode=yes git@github-toddwbucy 2>&1
# Expect: "Hi toddwbucy! You've successfully authenticated, ..."

# Hugo extended is installed (only needed for local preview; not for commit/push)
hugo version
# Expect: hugo vX.Y.Z+extended

# Source file exists
ls -la <source-path>
```

If the blog repo's working tree is dirty (uncommitted changes), STOP and
ask the user how to proceed. Do not stash or discard their work.

---

## Step 2 — Convert the source (FIDELITY RULE)

**Document-conversion fidelity is a hard constraint.** Your job is
conversion, not editing. Apply structural transforms only. Keep prose
verbatim — every word, every comma, every semicolon as the user wrote it.

### Allowed structural transforms

- Heading-level demotion (e.g. H1 → H2, since the title moves to front
  matter; section levels shift accordingly).
- Image syntax: `<img src="..." />` → Markdown `![](path)`.
- Image-path remapping into `/images/blog/<slug>/` (or
  `/images/papers/<slug>/` for papers).
- Code-block conversion (e.g. blockquote-styled Word "code" → fenced
  ` ```python ` block).
- Strip `<u>` tags from links (Word's hyperlink character-style artifact
  surfaced by pandoc).
- Strip `\#`, `\-`, `\*` table-cell escape artifacts from pandoc output.
- Add YAML front matter at the top.

### Forbidden transforms (these are editorial)

- Substituting em-dashes (`—`) for hyphens, commas, or semicolons.
- Substituting words ("PhD candidate" → "PhD student", "MIRAS" → "Miras").
- Restructuring sentences or paragraphs.
- "Smoothing" or "polishing" prose.
- Adding clarifying phrases that weren't in the source.

### Conversion procedure

```bash
# Extract images to a scratch dir; produce GFM markdown
mkdir -p /tmp/blog_convert_<slug>
pandoc -f docx -t gfm --wrap=none \
  --extract-media=/tmp/blog_convert_<slug> \
  <source.docx> \
  -o /tmp/blog_convert_<slug>/raw.md
```

Read `raw.md`. Build the final markdown by applying the allowed
structural transforms only — keep every prose word and every
punctuation choice exactly as in the source.

### Mandatory fidelity guardrail (run before committing)

Compare the source's plain-text vocabulary against your output's. The
diff must be empty except for words that legitimately moved from body
to front matter (typically just the title).

```bash
# 1. Source plain words
pandoc -f docx -t plain --wrap=none <source.docx> 2>/dev/null \
  | tr -s '[:space:]' ' ' | grep -oE '[A-Za-z]+' | sort -u \
  > /tmp/src_words.txt

# 2. Your committed markdown's plain words (strip front matter first)
sed '/^---$/,/^---$/d' \
  /home/todd/git/toddwbucy-research-blog/content/<path>/<slug>.md \
  | pandoc -f gfm -t plain 2>/dev/null \
  | tr -s '[:space:]' ' ' | grep -oE '[A-Za-z]+' | sort -u \
  > /tmp/mine_words.txt

# 3. Words in your output that aren't in source = vocabulary drift
comm -13 /tmp/src_words.txt /tmp/mine_words.txt
# Should print only title-line words (or be empty)

# 4. Em-dash count comparison — must match exactly
echo "source: $(pandoc -f docx -t plain --wrap=none <source.docx> 2>/dev/null | grep -c '—')"
echo "mine:   $(grep -c '—' /home/todd/git/toddwbucy-research-blog/content/<path>/<slug>.md)"
```

If either check fails, fix the markdown and re-run. Do not commit until
both checks pass.

---

## Step 3 — Process images

Word docs commonly embed images at unreasonable sizes (multi-MB PNGs).
Compress before placing in the blog.

```bash
mkdir -p /home/todd/git/toddwbucy-research-blog/static/images/blog/<slug>

# Resize and convert to JPG (use this default for photos/illustrations)
for f in /tmp/blog_convert_<slug>/media/image*.png; do
  n=$(basename "$f" .png)
  convert "$f" -resize 1200x -quality 85 -strip \
    /home/todd/git/toddwbucy-research-blog/static/images/blog/<slug>/${n}.jpg
done

# For images that are diagrams/screenshots with text (sharp-edge content),
# keep as PNG instead:
#   convert <src>.png -resize 1200x out.png
```

Reference these in the converted markdown using site-absolute paths:

```markdown
![](/images/blog/<slug>/image1.jpg)
```

The repo's `layouts/_markup/render-image.html` hook prefixes the
GitHub Pages baseURL automatically; do not prepend the deploy path
manually.

After image conversion, clean up the scratch dir:

```bash
rm -rf /tmp/blog_convert_<slug>
```

---

## Step 4 — Place the file and add front matter

For papers: `/content/papers/<series>/<slug>.md`
For blog posts: `/content/blog/YYYY/MM/<slug>.md`
For framework: `/content/framework/<slug>.md`
For projects: `/content/projects/<slug>.md`

**Use the matching archetype's front-matter schema.** Read
`/home/todd/git/toddwbucy-research-blog/archetypes/<type>.md` first to
see the expected fields and their conventions, then fill in. Do not
invent fields the archetype doesn't include unless there's a real
reason.

For blog posts the canonical archetype fields are: `title`, `date`,
`draft` (set `false` to publish), `type: "post"`, `status: "published"`,
`tags`, `abstract`, `medium_url` (blank initially), `canonical_url`
(see next section).

For papers add: `series`, `paper_number`, `version`, `cite.bibtex`.

---

## Canonical URL — decision tree

The `canonical_url:` front-matter field tells search engines and humans
where the *original* version of the article lives.

| Scenario                                              | `canonical_url`                  |
|-------------------------------------------------------|----------------------------------|
| Article authored on the blog first (the default)      | leave blank                      |
| Article first published on Medium / Stackademic       | set to the original venue URL    |
| Article first published in a journal / preprint       | set to the journal/arXiv URL     |
| Migrating older work from a defunct platform          | leave blank (no live original)   |

When the field is set, the blog automatically:

1. Emits `<link rel="canonical" href="...">` in `<head>` pointing to
   the original venue.
2. Renders a visible "Originally published at <host>" callout above
   the article abstract.

Ask the user explicitly: "Was this published anywhere before? If so,
should that earlier publication remain canonical?"

---

## Step 5 — Local preview (optional but recommended)

If Hugo extended is installed:

```bash
cd /home/todd/git/toddwbucy-research-blog
hugo server --buildDrafts --bind 127.0.0.1 --port 1313
```

Open http://localhost:1313/ and visit the new article's URL. Verify:

- Title, date, byline render correctly.
- Abstract callout displays at the top.
- "Originally published at..." callout appears (only if `canonical_url` set).
- Images load and are not absurdly large.
- Code blocks have syntax highlighting.
- No broken inline links.
- Tags appear in the footer and link to working tag-index pages.

Stop the server (Ctrl+C) before proceeding.

If Hugo isn't installed locally, you can skip preview and rely on the
production build at deploy time. The user can always inspect the
`hugo --gc` output for warnings.

---

## Step 6 — Commit and push

```bash
cd /home/todd/git/toddwbucy-research-blog

# Stage the new content file plus its image directory
git add content/<path>/<slug>.md static/images/blog/<slug>/

# Verify status — both should be staged, nothing else
git status --short

# Commit. Use a clear, conventional message.
git commit -m "$(cat <<'EOF'
Publish: <Article Title>

<one-sentence description of what the article argues>

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"

# Push to main; auto-deploys via GitHub Actions in ~30s
git push origin main
```

If the user has explicitly asked you to commit and push, proceed. If
they haven't, stop after staging and ask.

---

## Step 7 — Verify the deploy

```bash
# Wait briefly for the workflow to queue, then watch
sleep 8
gh run list --repo toddwbucy/toddwbucy_research_blog --limit 1

# Watch the latest run to completion
gh run watch --repo toddwbucy/toddwbucy_research_blog --exit-status

# Verify live URL responds
curl -sI https://toddwbucy.github.io/toddwbucy_research_blog/<path>/<slug>/ \
  | head -3
# Expect: HTTP/2 200
```

Report the live URL to the user.

---

## Step 8 — Cross-post acknowledgment (blog-first only)

If `canonical_url` was *not* set (article is blog-first), remind the user
of the cross-post step:

> Your article is now live at `<blog URL>`. To cross-post to Medium /
> Stackademic, paste the blog URL into their "Import a story" feature.
> The platform will set `rel="canonical"` back to the blog automatically.
> After you have the resulting Medium/Stackademic URL, run me again
> with that URL and I'll add it to the article's `medium_url:` front-
> matter field for record-keeping.

If `canonical_url` *was* set (article was first published elsewhere),
no cross-post step — the original venue stays canonical.

---

## Things to ask the user about (don't assume)

- Publication date if not in the source
- Tags beyond the obvious ones
- Whether to publish immediately or leave `draft: true`
- The byline if they want a per-article override (otherwise the global
  `Todd W. Bucy` from `hugo.toml` applies)
- The canonical URL question (see decision tree above)
- For papers: BibTeX citekey if they want a memorable one (the archetype
  generates a default from title + year)

## Things NOT to do

- Don't alter prose during conversion (see fidelity rule above). The
  `comm -13` diff must be empty before commit.
- Don't commit `.docx`, `.doc`, `.odt` source files. The repo's
  `.gitignore` excludes them, but be deliberate about staging.
- Don't push to `main` from a different git identity. Verify
  `git config user.name` returns `Todd W. Bucy` first.
- Don't use `--no-verify` or other commit/push hook bypasses.
- Don't write planning, decision, or analysis Markdown files into the
  blog repo. Conversation context handles intermediate work.
- Don't add stylistic emojis to committed content.
- Don't `--minify` during local preview (strips HTML comments useful
  for debugging). The deploy workflow does its own minification.

## When in doubt

The blog repo's `CLAUDE.md` has additional conventions and the full
content-type taxonomy. Read it if a question arises that this runbook
doesn't answer. If still unclear, ask the user — better to confirm
than to silently make a guess that drifts from their intent.
