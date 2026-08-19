---
title: "adiff: Annotated Git Diffs in the Terminal"
date: "2026-08-19T00:45:00Z"
description: "I built a CLI that renders a branch as a narrated, per-commit walkthrough, with Claude supplying the editorial layer through a protobuf contract it cannot use to hide anything."
tags: ["ai", "tools", "git", "claude-code", "golang"]
---

## The Problem

Most of the code I review these days was written by an LLM. Sometimes it is a branch Claude Code produced for me overnight; sometimes it is a colleague's agent-assisted PR. Either way, the failure mode is the same: a large diff lands, every hunk is plausible, and reading it cold takes real effort because nobody wrote it line by line, so nobody can walk you through it.

What I wanted was a narrated diff. Every commit, every hunk, in order, with a paragraph of intent before each commit and a sentence or two under each hunk explaining what it does and how it connects to the rest of the branch. The kind of walkthrough a considerate author would give you in a review, except the author here is a model that never gets tired of explaining.

So I built [adiff](https://github.com/willfindlay/adiff), a Go CLI that does exactly that. Run it on a branch and it renders the full diff as a per-commit walkthrough, syntax highlighted, with an editorial layer supplied by Claude.

![An annotated walkthrough of four adiff commits, showing a branch overview that references commit SHAs, an intent paragraph, and syntax-highlighted hunks with line-number gutters](/content/blog/adiff-annotated-git-diffs/walkthrough.png "adiff walking through four of its own commits. The overview and intent paragraphs come from Claude; the diff comes from git.")

## It Started as a Skill, and Failed

The first version was not a CLI at all. I tried to build this inside Claude Code as a skill: have the agent read the branch, then print an annotated walkthrough into the conversation.

Two walls ended that approach. Claude Code [strips ANSI escape codes](https://github.com/anthropics/claude-code/issues/16668) from assistant text, so all the color work (backgrounds behind added lines, dim italic annotations, syntax highlighting) was silently flattened to plain text. And my workaround of replaying hunks through the Edit tool hid every diff inside collapsed tool-call blocks, which is exactly where you do not want the content you are asking someone to read.

The lesson generalized: an agent should not be a terminal renderer. A standalone binary owns its output, its pager, and its color handling. The agent's job moved to the one thing it is actually good at, which is the editorial layer, and everything else became ordinary deterministic Go.

## The Trust Property

Here is the design decision the whole tool hangs on. adiff is meant for reviewing LLM-generated code, which means the review tool must not share the failure mode of the thing being reviewed. If the model rendered the diff, a hallucinated or omitted hunk would be invisible precisely when you most need to see it.

So the model never touches diff content. adiff shells out to git, parses the unified diffs into a typed model itself, and renders every hunk of every commit from that model. Claude receives the parsed structure and returns only metadata keyed to it: an overview, intent paragraphs, annotations attached to hunk indexes adiff already discovered. Validation drops anything that references a commit, file, or hunk that does not exist in the request. The worst a bad reply can do is leave a hunk unannotated, and that hunk still renders with a visible marker saying so.

The same principle shows up in smaller places. A reply can mark a file as generated so its noise is skipped, but only with a stated reason, and adiff detects lockfiles and generated-code markers itself without asking. If the annotation call fails entirely, after one retry you get the complete walkthrough anyway, under a warning banner. Never silently degrade; never die with a renderable diff in hand.

## A Protobuf Contract With a Model

The interface between adiff and Claude is a protobuf schema, `adiff.v1`. Both directions are proto-defined: adiff serializes an `AnnotationRequest` with protojson and embeds it in the prompt, and the reply must parse as a `Walkthrough` under strict protojson, unknown fields rejected.

The part I did not expect to enjoy: the `.proto` file is the prompt. Its doc comments are written for the model as much as for Go readers, because the schema text itself is embedded in every request as the API reference.

```proto
message CommitAnnotation {
  // Must exactly match the short_sha of a CommitInput from the request; use
  // "" when annotating the unstaged pseudo-commit. Non-matching entries are
  // dropped.
  string short_sha = 1;

  // Non-empty only when the commit message claims something its diff does not
  // do, or the diff does something the message does not mention. Mismatches
  // are signal for reviewers of LLM-generated work, so state the
  // disagreement plainly.
  string discrepancy = 3;
  ...
}
```

That `discrepancy` field is my favorite part of the contract. When a commit message and its diff disagree, the walkthrough says so in a highlighted line. For agent-written branches, message/diff drift is one of the strongest signals that something went wrong.

Strictness has a cost: models occasionally emit JSON that does not parse. The first version retried by re-sending the entire request with the error appended, which meant one stray bracket doubled the input cost of a run. The fix was obvious in hindsight: the editorial work is already done, only the serialization broke. The retry now sends a repair prompt containing just the broken reply, the parse error, and the schema, never the diff again.

## Making It Worth Running Twice

A tool that spends a model call per invocation needs cost discipline, so annotations are cached. An unchanged diff replays instantly for free. Extending a branch by a commit reuses the cached annotations, pays for the new commit, and regenerates the overview. The per-commit cache keys hash the patch content rather than the SHA, so a rebase that changes nothing costs nothing. A cost line after each run tells you what you spent, summed from the CLI's reported usage.

There is also a rule for agents, since adiff sometimes gets run by Claude itself from inside a session: with the default `--annotate=auto`, annotation only happens when stdout is a TTY. A script or agent consuming the output gets the plain structure without silently spawning a nested model call, and one stderr line says how to force it.

The rendering layer got the leftover attention, and it shows in the details: word-level emphasis inside changed lines the way delta does it, dual line-number gutters, and long hunks split at logical seams the model chooses, with the annotation for each segment placed where it belongs.

![Word-level intra-line highlighting on the commit that introduced the feature, with segment annotations in dim italics beneath the hunks](/content/blog/adiff-annotated-git-diffs/word-level.png "The word-level highlighting commit, annotating itself. Changed tokens get a brighter background; the rest of the line stays dark.")

## Where It Is Now

v0.2.0 is out. Beyond the basics it does pull requests (`adiff pr 1234`, resolved through gh and fetched into temporary refs without touching your worktree), explicit ranges (`adiff v0.1.0..HEAD`), and parallel per-commit annotation for large branches.

![An annotated pull request walkthrough of an upstream chroma PR, with a long hunk split into annotated segments](/content/blog/adiff-annotated-git-diffs/pr-mode.png "PR mode on a real upstream pull request. The hunk is split at seams the model chose, each segment annotated.")

The Claude CLI is the only annotation backend today because it is the one I use, and that is a fixable limitation: the backend sits behind a small interface, and there are [good first issues](https://github.com/willfindlay/adiff/issues) for codex, gemini, and opencode backends if you use one of those. A direct Anthropic API backend with structured outputs is next on my own list, since grammar-constrained decoding would make the malformed-JSON case impossible rather than merely cheap.

Install it with `go install github.com/willfindlay/adiff/cmd/adiff@latest` and run it from any branch. Fittingly, the project was itself built by Claude Code agents working from a spec, which means most of its own history has been reviewed through the tool it produced. The walkthroughs read well. I would say that, but the screenshots are real output, so you can judge for yourself.
