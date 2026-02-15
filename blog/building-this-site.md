---
title: "Building This Site with Go and Goldmark"
date: "2026-02-14T12:00:00Z"
description: "How I built williamfindlay.com as a Go server that hot-loads content from a separate git repository."
tags: ["go", "web", "architecture"]
---

I wanted a personal site that felt fast, looked distinctive, and let me update content without rebuilding a container. The result is a Go server that clones a content repo at startup, parses markdown with [goldmark](https://github.com/yuin/goldmark), and atomically swaps in new content on a background timer.

## Why Not a Static Site Generator?

Static site generators are great, but they couple content changes to build pipelines. I wanted to push a markdown file and have it appear on the site within minutes — no CI, no Docker rebuild, no deploy.

The solution: a Go binary that embeds its templates and static assets, but loads content from a git repository at runtime. An `atomic.Pointer` ensures readers never see a partially-built content store.

## The Stack

- **Go stdlib `net/http`** with 1.22+ routing patterns
- **goldmark** for CommonMark-compliant markdown rendering
- **go-git** for pure-Go git operations (no `git` binary needed in the container)
- **Canvas 2D particles** for the background animation

## Content Hot-Loading

The server clones the content repo on startup, then runs a background goroutine that pulls every 5 minutes. When new content arrives:

1. Walk the `blog/`, `projects/`, and `resume/` directories
2. Parse frontmatter and render markdown
3. Build an immutable `ContentStore` with sorted posts and tag indices
4. Atomically swap the pointer — readers see the new content immediately

```go
type AtomicStore struct {
    ptr atomic.Pointer[ContentStore]
}

func (s *AtomicStore) Load() *ContentStore {
    return s.ptr.Load()
}
```

The entire approach means zero downtime content updates.

## Docker

The final image is under 20MB thanks to a multi-stage build: compile in `golang:1.25-alpine`, copy the binary into `distroless/static`. Since `go-git` is pure Go, no system dependencies are needed at runtime.

```dockerfile
FROM golang:1.25-alpine AS build
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /server ./cmd/server

FROM gcr.io/distroless/static:nonroot
COPY --from=build /server /server
ENTRYPOINT ["/server"]
```
