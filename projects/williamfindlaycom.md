---
title: "My Personal Site"
date: "2026-02-15T06:05:00Z"
description: "Source code for williamfindlay.com, a Go web server with hot-loaded markdown content."
tags: ["go", "docker", "web"]
repo: "https://github.com/willfindlay/williamfindlaycom"
url: "https://williamfindlay.com"
status: "active"
featured: true
---

The site you're looking at right now. A Go web server that embeds its templates and static assets at compile time but loads blog posts, project pages, and resume content from a separate git repository at runtime. It runs in a Kubernetes homelab cluster as a single static binary on top of distroless.

Content and application code live in separate repositories. The server clones the content repo on startup, then pulls it on a background ticker. When new content arrives, it parses every markdown file, builds a new `ContentStore`, and swaps it in via `atomic.Pointer` so readers never see a partial update.

## Features

- **Content hot-loading**: Push markdown to the content repo, and the site picks it up within minutes with no rebuild or restart
- **Atomic content swaps**: An `atomic.Pointer[ContentStore]` ensures readers always see a consistent snapshot, even mid-refresh
- **Embedded assets**: Templates and static files are baked into the binary via `embed.FS`, keeping the container image small and dependency-free
- **Atom feed**: A templated `/feed.xml` endpoint serves an Atom feed of all blog posts
- **Security headers and caching middleware**: Static assets get immutable cache headers; all responses include `X-Content-Type-Options`, `X-Frame-Options`, and `Referrer-Policy`
- **Self-hosted variable fonts**: Syne, Source Serif 4, and JetBrains Mono served as single woff2 files instead of per-subset splits from Google Fonts

## Stack

| Component | Technology |
|-----------|-----------|
| Server | Go stdlib `net/http` with 1.22+ routing patterns |
| Markdown | `goldmark` + `goldmark-highlighting` (chroma, dracula theme) |
| Frontmatter | `go.abhg.dev/goldmark/frontmatter` |
| Git sync | `go-git/v5` (clone, shallow pull) |
| Templates | `html/template` + `embed.FS` |
| Container | Multi-stage Docker, `golang:1.25-alpine` to `distroless/static` |
| Hosting | Kubernetes homelab cluster |
