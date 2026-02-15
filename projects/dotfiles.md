---
title: "Dotfiles"
date: "2024-01-01T00:00:00Z"
description: "Personal development environment configuration managed with chezmoi."
tags: ["linux", "tooling"]
repo: "https://github.com/willfindlay/dotfiles"
status: "active"
featured: false
---

My personal dotfiles, managed with [chezmoi](https://www.chezmoi.io/). Includes configuration for:

- **Neovim** with LSP, Treesitter, and a custom plugin configuration
- **Zsh** with a minimal prompt and project-specific environment loading
- **Git** with aliases, delta for diffs, and signing configuration
- **Tmux** with a clean status bar and sensible key bindings

## Philosophy

I keep my dotfiles minimal and fast. No plugin managers that download 50 packages on startup. No fancy prompt that shells out to `git status` on every command. Configuration should make tools faster to use, not slower to load.
