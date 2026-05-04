---
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
date: {{ .Date }}
draft: true
type: "project"
status: "active"          # active | archived | subsumed
tags: []
abstract: |
  One-paragraph description of the project's purpose and current state.
repository: ""            # URL to source repo, if public
related_papers: []        # list of paper URLs that came out of this project
---
