---
title: "Building This Site with Go and Goldmark"
date: "2026-02-14T12:00:00Z"
description: "How I built williamfindlay.com as a Go server that hot-loads content from a separate git repository."
tags: ["go", "web", "architecture"]
---

I wanted a personal site that felt fast, looked distinctive, and let me update content without rebuilding a container. The result is a Go server that clones a content repo at startup, parses markdown with [goldmark](https://github.com/yuin/goldmark), and atomically swaps in new content on a background timer.

## Why Not a Static Site Generator?

Static site generators are fine for most people. Hugo, Zola, Astro; pick one, write markdown, deploy. But they couple content changes to build pipelines. Every time you push a new post, something has to rebuild the site and redeploy it: a CI job, a webhook, a CDN invalidation chain.

I wanted something simpler. Push a markdown file and have it appear on the site within minutes. No CI. No Docker rebuild. No deploy step at all.

The tradeoff is that the server needs to do more work at runtime. It clones a git repository on startup, parses every markdown file into HTML, and holds the rendered content in memory. But for a personal site with a handful of posts, this is a rounding error in terms of resource usage.

## Architecture

The site is a single Go binary. Templates and static assets are embedded at compile time using `embed.FS`, so the binary contains everything it needs to serve the site except content. Content lives in a separate repository ([williamfindlaycom-content](https://github.com/willfindlay/williamfindlaycom-content)) that the server clones and syncs at runtime.

The separation is the key design choice. Application code and visual design change on a different cadence than blog posts and project pages. Keeping them in separate repos means I can push a new post without touching the server, and I can redesign the site without worrying about content migration.

There are no external web frameworks. Go 1.22 added method-based routing patterns to `net/http`, so routes look like this:

```go
mux.HandleFunc("GET /{$}", s.deps.Home())
mux.HandleFunc("GET /blog", s.deps.BlogList())
mux.HandleFunc("GET /blog/{slug}", s.deps.BlogPost())
mux.HandleFunc("GET /projects", s.deps.ProjectList())
mux.HandleFunc("GET /projects/{slug}", s.deps.ProjectDetail())
mux.HandleFunc("GET /resume", s.deps.Resume())
mux.HandleFunc("GET /feed.xml", s.deps.Feed())
```

No need for Chi or Gin or Echo. The standard library handles method matching, path parameters, and precedence rules.

## Content Hot-Loading

On startup, the server clones the content repository using [go-git](https://github.com/go-git/go-git), a pure-Go git implementation. No `git` binary required in the container. The clone is shallow (`Depth: 1`) since we only need the latest state of the files, not the full history.

```go
_, err := git.PlainClone(cfg.Dir, false, &git.CloneOptions{
    URL:           cfg.RepoURL,
    ReferenceName: refName(cfg.Branch),
    SingleBranch:  true,
    Depth:         1,
    Auth:          auth(cfg.AuthToken),
})
```

After the initial clone, a background goroutine pulls every five minutes. When new content arrives, the server walks the `blog/`, `projects/`, and `resume/` directories, parses each markdown file, and builds a new `ContentStore`. The store holds sorted post lists and lookup maps indexed by slug and tag. Once the store is fully built, a single atomic pointer swap makes it visible to all incoming requests.

The entire cycle, from git pull to live content, takes milliseconds. Readers never see a partially loaded state.

## Goldmark and Markdown Rendering

[Goldmark](https://github.com/yuin/goldmark) handles markdown parsing and HTML rendering. It supports CommonMark out of the box, and its extension system lets me add exactly the features I need:

```go
var md = goldmark.New(
    goldmark.WithExtensions(
        extension.GFM,
        &frontmatter.Extender{},
        highlighting.NewHighlighting(
            highlighting.WithStyle("dracula"),
        ),
    ),
    goldmark.WithParserOptions(
        parser.WithAutoHeadingID(),
    ),
    goldmark.WithRendererOptions(
        html.WithUnsafe(),
    ),
)
```

GFM gives me tables, strikethrough, autolinks, and task lists. The frontmatter extension parses YAML metadata at the top of each file into Go structs. The highlighting extension uses [Chroma](https://github.com/alecthomas/chroma) to syntax-highlight code blocks at render time, so there is no client-side JavaScript for syntax highlighting. The Dracula theme keeps things readable on dark backgrounds.

Each content type has a struct with YAML tags that map to frontmatter fields:

```go
type BlogPost struct {
    Slug        string
    Title       string        `yaml:"title"`
    Date        time.Time     `yaml:"date"`
    Description string        `yaml:"description"`
    Tags        []string      `yaml:"tags"`
    Content     template.HTML
}
```

Rendering is a two-step process. Goldmark converts the markdown body to HTML while simultaneously extracting frontmatter into a parser context. The frontmatter gets decoded into the struct, and the rendered HTML goes into the `Content` field as `template.HTML`.

## The Atomic Pointer Pattern

This is my favourite part of the design. The `ContentStore` is an immutable snapshot: once built, it never changes. When the background sync produces a new store, we swap it in atomically.

```go
type AtomicStore struct {
    ptr atomic.Pointer[ContentStore]
}

func (s *AtomicStore) Load() *ContentStore {
    return s.ptr.Load()
}

func (s *AtomicStore) Store(cs *ContentStore) {
    s.ptr.Store(cs)
}
```

Every request handler calls `Load()` at the top to grab the current store, then works entirely with that snapshot. If a sync happens mid-request, the handler keeps using the old store; it does not see a partially constructed replacement. New requests after the swap see the new content immediately.

No mutexes. No read-write locks. No contention between readers and the single writer. The `ContentStore` itself contains pre-built indexes for O(1) lookups:

```go
type ContentStore struct {
    Posts       []BlogPost
    PostsBySlug map[string]*BlogPost
    PostsByTag  map[string][]*BlogPost

    Projects       []Project
    ProjectsBySlug map[string]*Project

    Resume *Resume
}
```

A request for `/blog/building-this-site` is a single map lookup. Listing posts by tag is a single map lookup. No filtering, no sorting at request time. All the expensive work happens once during the sync, off the hot path.

## Templates and Rendering

Go's `html/template` package does the rendering. A single base template defines the HTML document structure, navigation, and footer. Page-specific templates inject their content via `{{template "content" .}}`.

The renderer clones the base template for each page at startup, parsing the page-specific template on top:

```go
for _, page := range pages {
    t, err := base.Clone()
    if err != nil {
        return nil, fmt.Errorf("cloning base for %s: %w", page, err)
    }
    t, err = t.ParseFS(fsys, page)
    if err != nil {
        return nil, fmt.Errorf("parsing %s: %w", page, err)
    }
    r.templates[page] = t
}
```

All templates are embedded in the binary via `embed.FS`. There is no template directory to mount, no file I/O at render time, and no risk of template files being missing or out of sync with the code.

A handful of custom template functions handle date formatting, string truncation, and safe HTML injection. Nothing fancy; just enough to keep templates clean.

## The Particle Canvas

The homepage background runs an interactive particle animation on a `<canvas>` element. 120 particles drift across the viewport with random velocities. When two particles are within 140 pixels of each other, a faint line connects them, creating a mesh effect that shifts as particles move.

Mouse interaction adds a repulsion force. Move the cursor near a particle cluster, and the particles push away. The repulsion radius is 180 pixels with a force of 0.015, which gives a gentle scattering effect without making the animation feel chaotic.

Most particles are cyan (`rgb(79, 209, 197)`), but about 15% are purple (`rgb(128, 90, 213)`) for visual contrast. Each particle pulses its radius and opacity along a sine wave, so the mesh feels alive rather than mechanical.

Two things were important to get right. First, the animation respects `prefers-reduced-motion`. If the user's OS is set to reduce motion, the canvas does not render at all. Second, the canvas scales for device pixel ratio (capped at 2x) so particles look sharp on retina displays without burning through GPU cycles on high-DPI monitors.

## Docker

The final image is under 20MB thanks to a multi-stage build. The first stage compiles the Go binary in `golang:1.25-alpine`. The second stage copies only the binary and CA certificates into `gcr.io/distroless/static:nonroot`.

```dockerfile
FROM golang:1.25-alpine AS build
RUN apk add --no-cache ca-certificates
RUN mkdir -p /data/content && chown 65534:65534 /data/content

WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /server ./cmd/server

FROM gcr.io/distroless/static:nonroot
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /server /server
COPY --from=build --chown=nonroot:nonroot /data /data

USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

`CGO_ENABLED=0` produces a fully static binary with no libc dependency. The `-ldflags="-s -w"` flags strip the symbol table and DWARF debug info, shaving a few megabytes off the binary. Distroless has no shell, no package manager, and a minimal attack surface.

The server runs as `nonroot` (UID 65534). The `/data/content` directory is pre-created and owned by this user so go-git can clone into it without permission issues.

CA certificates are copied explicitly because distroless does not include them by default, and go-git needs them to clone over HTTPS.

## Configuration

Everything is configured through environment variables:

| Variable | Default | Purpose |
|----------|---------|---------|
| `PORT` | `8080` | HTTP listen port |
| `SITE_TITLE` | | Site name for templates |
| `SITE_URL` | | Canonical URL for feeds and OG tags |
| `CONTENT_REPO_URL` | (required) | Git URL of the content repo |
| `CONTENT_REPO_BRANCH` | `main` | Branch to track |
| `CONTENT_DIR` | `/data/content` | Local clone path |
| `SYNC_INTERVAL` | `5m` | How often to pull |
| `GIT_AUTH_TOKEN` | | Optional token for private repos |

No config files, no flags, no YAML. Environment variables work well with Docker, Kubernetes, and systemd, and they keep the interface dead simple.

## What I Like About This Approach

The whole site is about 1,500 lines of Go. The dependency list is short: go-git, goldmark, and a couple of goldmark extensions. There is no JavaScript framework, no build toolchain for frontend assets, and no database.

Content updates are genuinely zero-downtime. The atomic pointer swap is invisible to readers. I can push a typo fix to the content repo and see it live in under five minutes without any deployment step.

Embedding templates and static assets makes deployment trivial. The container image is a single binary. If the content repo is unavailable on startup, the server just has no content; it does not crash. When the repo comes back, the next sync cycle picks it up.

The main limitation is that this approach does not scale to thousands of posts. Holding all rendered HTML in memory is fine for a personal site, but a blog with ten thousand articles would need a different strategy (probably a content-addressed cache or a database). For my purposes, that is a problem I am happy not to have.
