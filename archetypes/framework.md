---
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
date: {{ .Date }}
draft: true
type: "framework"
status: "draft"           # draft | published | superseded
version: "v1"
tags: []
abstract: |
  One paragraph describing what this framework document covers.
supersedes: ""            # URL of previous version, if applicable
superseded_by: ""         # URL of newer version, if applicable
cite:
  bibtex: |
    @misc{bucy{{ now.Year }}{{ .File.ContentBaseName | replaceRE "[^a-z0-9]" "" }},
      title  = {{ "{" }}{{ replace .File.ContentBaseName "-" " " | title }}{{ "}" }},
      author = {Bucy, Todd W.},
      year   = {{ "{" }}{{ now.Year }}{{ "}" }},
      note   = {Framework document, version v1},
      url    = {{ "{" }}{{ "{{ .Permalink }}" }}{{ "}" }}
    }
---

<!--
Framework documents are versioned living references. When you make a breaking
revision: copy the existing file to /content/archive/<slug>-<date>.md, set its
`status: superseded` and `superseded_by` to point at this URL, then bump the
`version` field here and update the content. The version banner at the top of
each rendered framework page is driven by these fields automatically.
-->
