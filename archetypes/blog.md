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
---

<!--
Authoring notes (delete before publishing):
- This is a blog-style post. For long-form papers use `hugo new papers/<slug>.md`.
- Once published on the blog, paste the canonical URL into Medium's
  "Import a story" feature — Medium will set rel="canonical" automatically.
- After import, paste the resulting Medium URL into the `medium_url` field above.
-->
