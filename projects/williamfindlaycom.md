---
title: "My Personal Site"
date: "2026-02-15T06:05:00Z"
description: "Source code for williamfindlay.com, a statically-rendered personal website and blog built with Go."
tags: ["go", "docker", "web"]
repo: "https://github.com/willfindlay/williamfindlaycom"
url: "https://williamfindlay.com"
status: "active"
featured: true
---

williamfindlay.com is built from scratch in Go using only the standard library's `net/http` package with Go 1.22+ routing patterns. It renders blog posts and project pages from markdown with YAML frontmatter, using goldmark for parsing and chroma for syntax highlighting.

Content lives in a separate git repository that the server clones on startup and periodically pulls in the background. When new content arrives, the server parses it and swaps in the updated content store atomically, so the site stays up while content refreshes. This separation keeps the application code and the writing workflow independent of each other.

Templates and static assets are embedded into the binary at compile time via `embed.FS`, which means the final Docker image is a single binary on top of distroless. The whole thing runs in a Kubernetes homelab cluster as a lightweight container with no runtime dependencies beyond the content repo URL.
