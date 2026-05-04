---
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
date: {{ .Date }}
draft: true
type: "paper"
status: "draft"           # draft | published | superseded
version: "v1"
series: ""                # e.g. "Constraint Organization"
paper_number: 0           # ordinal within series; 0 if standalone
tags: []
abstract: |
  Two- to four-sentence abstract. Surfaces in listing pages, in the
  page <meta description>, and at the top of the rendered paper.
supersedes: ""            # URL of previous version, if applicable
superseded_by: ""         # URL of newer version, if applicable
medium_url: ""            # if cross-posted to Medium
cite:
  bibtex: |
    @article{bucy{{ now.Year }}{{ .File.ContentBaseName | replaceRE "[^a-z0-9]" "" }},
      title  = {{ "{" }}{{ replace .File.ContentBaseName "-" " " | title }}{{ "}" }},
      author = {Bucy, Todd W.},
      year   = {{ "{" }}{{ now.Year }}{{ "}" }},
      url    = {{ "{" }}{{ "{{ .Permalink }}" }}{{ "}" }}
    }
---

<!--
Authoring notes (delete before publishing):
- For multi-part papers, place inside content/papers/<series-slug>/ rather than
  at the papers root. The slug becomes part of the canonical URL.
- Bump `version` when revising; for breaking revisions, move the previous file
  to /content/archive/ and set `supersedes` to its URL here, plus
  `superseded_by` on the archived copy.
- The cite.bibtex block is auto-templated; verify and adjust the citekey if
  you want a more memorable identifier (the default is title-derived).
-->

## Abstract

(repeat or expand the abstract here in prose)

## Introduction

(begin paper)
