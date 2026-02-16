# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is the **content repository** for [williamfindlay.com](https://williamfindlay.com). It contains only markdown files — no application code, no build system, no tests. The Go web server at [willfindlay/williamfindlaycom](https://github.com/willfindlay/williamfindlaycom) clones this repo at runtime, parses the markdown, and serves it.

Pushing to `main` is all that's needed to publish. The server pulls this repo on a background timer and hot-swaps content.

## Content Types and Frontmatter

All files use YAML frontmatter delimited by `---`.

### Blog Posts (`blog/*.md`)

```yaml
---
title: "Post Title"
date: "2026-02-15T16:00:00Z"    # ISO 8601 with Z suffix
description: "Short summary for previews and metadata."
tags: ["lowercase", "hyphenated-words"]
---
```

### Projects (`projects/*.md`)

```yaml
---
title: "Project Name"
date: "2026-02-15T06:05:00Z"
description: "Brief project description."
tags: ["go", "docker", "web"]
repo: "https://github.com/user/repo"   # optional
url: "https://example.com"             # optional
status: "active"                       # optional
featured: true                         # optional
---
```

Projects typically include `## Features` (bulleted list) and `## Stack` (table) sections.

### Resume (`resume/resume.md`)

Minimal frontmatter — just `title: "Resume"`. The body is structured markdown with experience, education, skills, research, awards, publications, and open-source sections.

## Conventions

- **Filenames**: kebab-case (e.g., `building-this-site.md`)
- **Dates**: ISO 8601 with UTC `Z` suffix: `"YYYY-MM-DDTHH:MM:SSZ"`
- **Tags**: lowercase, single words or hyphenated
- **Markdown flavor**: CommonMark + GFM (tables, strikethrough, autolinks, task lists). The server uses goldmark with Chroma syntax highlighting (dracula theme).
