---
title: "williamfindlay.com"
date: "2026-02-14T00:00:00Z"
description: "This website — a Go server with hot-loaded markdown content and particle animations."
tags: ["go", "web"]
repo: "https://github.com/willfindlay/williamfindlaycom"
url: "https://williamfindlay.com"
status: "active"
featured: true
---

The site you're looking at right now. A Go web server that embeds its templates and static assets, but loads blog posts, project pages, and resume content from a separate git repository at runtime.

## Features

- **Content hot-loading**: Push markdown to the content repo, and the site picks it up within minutes — no rebuild needed
- **Atomic content swaps**: Readers never see partially-loaded content thanks to `atomic.Pointer`
- **Particle animation**: Custom Canvas 2D particle field with mouse-responsive repulsion
- **Self-hosted fonts**: Syne, Source Serif 4, and JetBrains Mono served as variable woff2 files
- **Tiny Docker image**: Under 20MB thanks to distroless base

## Stack

| Component | Technology |
|-----------|-----------|
| Backend | Go stdlib `net/http` |
| Templates | `html/template` + `embed.FS` |
| Markdown | goldmark + frontmatter |
| Git sync | go-git/v5 |
| Container | Multi-stage Docker → distroless |
