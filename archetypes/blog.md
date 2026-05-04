---
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
date: {{ .Date }}
draft: true
type: "post"
status: "draft"
tags: []
abstract: |
  One- or two-sentence abstract for SEO and listing pages.
medium_url: ""        # populate after cross-posting to Medium
canonical_url: ""     # set ONLY if this article was first published elsewhere
                      # (e.g. Stackademic, Medium) and that venue should remain
                      # the canonical source. Leaving blank means the blog is
                      # canonical, which is the default for new posts.
---

<!--
Authoring notes (delete before publishing):
- This is a blog-style post. For long-form papers use `hugo new papers/<slug>.md`.
- Once published on the blog, paste the canonical URL into Medium's
  "Import a story" feature — Medium will set rel="canonical" automatically.
- After import, paste the resulting Medium URL into the `medium_url` field above.
-->
