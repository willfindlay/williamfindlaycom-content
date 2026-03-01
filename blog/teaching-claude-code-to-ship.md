---
title: "Teaching Claude Code to Ship: Skills, Sub-Agents, and Git Workflow Automation"
date: "2026-02-28T18:00:00Z"
description: "How I built a system of skills, hooks, and sub-agents that turns Claude Code into a disciplined contributor with its own commit, code review, and pull request workflows."
tags: ["ai", "tools", "git", "automation", "claude-code"]
---

## The Problem with AI-Generated Commits

Claude Code can write code, run tests, and edit files. What it cannot do out of the box is ship that code with the discipline a real project demands. Left to its defaults, it produces commit messages that are vaguely correct, PR descriptions that describe intent rather than what actually changed, and reviews that read like a checklist rather than a colleague's feedback.

The gap is not capability. Claude can analyze diffs, reason about code structure, and write prose. The gap is process. Nobody told it how your team commits, what your PR descriptions should look like, or when to flag a problem versus stay quiet.

So I built a system that does. The result is three skills (commit, code review, and pull request) backed by hooks and sub-agents that enforce conventions, verify factual accuracy, and gate every public-facing action on explicit user approval. This post describes how they work.

## Skills as Structured Workflows

Claude Code has a concept called skills: markdown files with YAML frontmatter that define reusable workflows. When you invoke a skill (via a slash command like `/commit`), Claude loads the file and follows its instructions. Skills can declare which tools they're allowed to use, accept arguments, and call other skills.

A skill is not a prompt template. It is a step-by-step procedure with branching logic, tool calls, and quality gates. The markdown format means you can version-control skills alongside your other config, review changes to them in PRs, and iterate on them the same way you'd iterate on code.

Each of the three workflows lives in its own skill file. They share a common commit policy document that defines conventions (conventional commits, signed commits, atomic granularity, branch naming) and a set of hooks that enforce those conventions at the tool level, intercepting every Bash command before execution.

## The Commit Skill

The commit skill is the simplest of the three and serves as a building block for the others. Its job: stage changes, write a commit message, and commit. Two steps.

What makes it more than a wrapper around `git commit` is the policy layer. The skill references a commit policy document that specifies conventional commit format (`type(scope): description`), requires signed commits (`-s` flag), prohibits AI attribution lines, enforces HEREDOC syntax for messages (to avoid shell escaping issues with special characters), and demands atomic granularity where each commit represents the smallest logical unit of change.

The policy document also specifies that when planning multi-commit work, commits should happen immediately after implementing each unit, not deferred to the end. This avoids the painful hunk-splitting with `git add -p` that results when multiple commits touch the same file and you try to carve them apart retroactively.

A golden rule sits at the top of the policy: never commit without first loading the `/commit` or `/pull-request` skill. This is a prompt-level constraint rather than a hard block, but it's reinforced by a PreToolUse hook (discussed later) that intercepts `git commit` commands and denies the first attempt, requiring Claude to format staged files before retrying.

The commit skill also detects chezmoi contexts automatically. A bundled shell script checks whether the working directory is inside a chezmoi source directory or is chezmoi-managed but not inside a git worktree. When detected, the skill transparently rewrites all git commands to use `chezmoi git` instead, so the same workflow applies whether you're committing to a regular repo or to your dotfiles. A companion hook (chezmoi-git-reminder) enforces this at the tool level, denying bare `git` commands that target chezmoi-managed paths and telling Claude to use the `chezmoi git` subcommand equivalents.

## The Pull Request Skill

The PR skill orchestrates an eight-step workflow that takes uncommitted changes and produces a posted pull request. The steps are sequential and each gates the next:

1. Check for uncommitted changes via `git status`
2. Create a feature branch if not already on one
3. Stage and commit following the commit policy
4. Gather context (diff, commit log, base branch) and draft the PR title and body
5. Launch a verification sub-agent to fact-check the draft
6. Present the draft to the user with verification evidence
7. Check commit messages against the finalized PR description
8. Push and create the PR

Steps 1 through 3 use the same conventions as the commit skill. Step 4 gathers the full diff and commit history against the base branch (detected via `gh repo view --json defaultBranchRef`), then drafts the PR body as plain prose paragraphs. The drafting guidelines prohibit markdown headings, rigid sections like "Summary" or "Test plan", and bullet-driven descriptions. Testing should be described inline as part of the narrative, not separated into its own section. The tone is factual; no editorializing.

### Verification Sub-Agent

Step 5 is where the skill does something unusual. After drafting, it launches a Sonnet sub-agent via the Agent tool with a specific mandate: verify every factual claim in the draft against the actual code. The agent receives the draft text, the full diff, a diff stat, and the commit list.

The agent's instructions tell it to work paragraph by paragraph, identify each factual claim (behavior described, files changed, patterns used, test results mentioned), and verify each one by reading source files, examining the diff, and running read-only git commands. It produces a report with one section per paragraph: the quoted text, a list of supporting evidence with `file:line` references, and flags for anything inaccurate, unsupported, or exaggerated.

The sub-agent has full read access to the repository. This is a deliberate departure from the code review skill (discussed below), which constrains its agents to diff-only analysis. PR descriptions make claims about behavior, architecture, and motivation that can only be verified by reading the actual code, not just the changed lines.

The main agent receives the verification report and fixes any flagged inaccuracies before showing anything to the user.

### User Confirmation

Step 6 presents the draft paragraph by paragraph. After each paragraph, the verification evidence appears in a blockquote with file and line references. If any claims were corrected during verification, the corrections are noted.

The user gets three options via an interactive prompt: create the PR, provide feedback, or cancel. Feedback loops back to re-presentation without re-running the sub-agent. The sub-agent catches mistakes in the AI-generated draft; the user catches everything else. Re-verifying after every feedback round would be slow and provide diminishing returns, since the user is the final authority on their own edits.

### Commit Message Consistency

Step 7 happens after user approval of the PR body but before anything touches the remote. The skill reads every commit message on the branch (`git log --format="%H%n%B"`) and compares each one against the finalized PR description. If any commit body contradicts the PR, describes behavior differently, or references code that no longer exists (common after squashing or rewording during the session), the skill flags the discrepancies and invokes the `/reword-commits` skill to fix them.

The reword-commits skill is its own standalone workflow. It audits each commit's message against its diff, drafts replacements, presents old-vs-new for user approval, then executes an interactive rebase using a custom `GIT_SEQUENCE_EDITOR` script. The script is a small Python program that takes a list of SHAs and rewrites the rebase todo list, changing `pick` to `edit` only for the targeted commits. A backup branch is created before the rebase, and a tree-level diff afterwards confirms the code is unchanged.

This ordering (approve PR body, then check commit consistency, then push) means the push happens last, after all local work is finalized. Commit messages, PR body, and code are all consistent before anything leaves the machine.

## The Code Review Skill

The code review skill is the most complex of the three. It reviews a pull request for code quality, bugs, SOLID violations, anti-patterns, and compliance with project-specific instructions (documented in `CLAUDE.md` files). It produces findings as prose, presents them for approval, and optionally posts a GitHub review with inline comments.

### Multi-Agent Architecture

The review spawns three to four Sonnet sub-agents in parallel, each focused on a different concern:

1. Code quality: SOLID violations, anti-patterns, DRY/YAGNI/KISS
2. Project instruction compliance: rules from the project's instruction files
3. Shallow bug scan: logic errors, null handling, off-by-one, race conditions
4. Test coverage: new logic paths lacking tests

Each agent receives the PR diff, summary, commit list, and relevant project instruction files. A scoping constraint embedded verbatim in each prompt limits them to the diff. They cannot read files, run git blame, or access context beyond what's provided. Every issue must reference a specific diff line and attribute itself to a specific PR commit (by short SHA).

The diff-only constraint is intentional. Code review agents that read the entire codebase tend to produce findings about pre-existing issues unrelated to the PR. Constraining them to the diff keeps findings relevant. It also makes the system predictable: you can reason about what the agents will see.

### Structural Filter

Before confidence scoring, a hard structural filter runs. A shell script receives the agent issues as JSON (each with a `commit` field) and a JSON array of the PR's actual commits. It drops any issue whose commit SHA doesn't prefix-match a real PR commit. This is not prompt-dependent; it's a `jq` pipeline that operates on data. Issues with missing SHAs, empty SHAs, or hallucinated SHAs are silently removed.

The distinction between structural filters (scripts) and semantic filters (sub-agents) runs through the whole design. Structural filters catch problems that are unambiguous and mechanically verifiable. Semantic filters catch problems that require judgment. Mixing the two in a single prompt degrades both.

### Confidence Scoring

Issues surviving the structural filter go through a second wave of Haiku sub-agents, one per issue. Each scorer receives the PR, the issue description, and the project instruction files, then returns a confidence score on a 0-100 scale.

The rubric is explicit and included verbatim in the scorer's prompt: 0 means false positive, 25 means possibly real but unverified, 50 means real but minor, 75 means verified and important, 100 means confirmed and will be hit frequently. Issues below 70 are dropped. Borderline issues (50-69) can be included at discretion, but issues at 70 or above are never excluded.

This two-stage architecture (broad detection followed by narrow validation) is deliberately wasteful with compute. Running a Haiku agent per issue is cheap, and it catches the false positives that broad-sweep agents inevitably produce. The alternative, asking the review agents to self-filter, produces worse results because the same reasoning patterns that generated a false positive also prevent the agent from recognizing it as one.

### Deduplication and Posting

Before posting, the skill fetches existing review comments on the PR via the GitHub REST API. If another reviewer has already flagged the same concern (same file, nearby line, similar topic), the finding gets a +1 reaction on the existing comment instead of a new inline comment. This prevents the AI reviewer from restating points that human reviewers have already made.

The user sees all findings in the terminal first, formatted as prose paragraphs. Each reads like a comment from a human reviewer: direct, specific, with file and line references, followed by a commit attribution. Confidence scores are internal bookkeeping and never appear in output. Only after explicit approval does the skill construct the GitHub review payload and post it.

Line numbers in the posted review are verified against the actual diff output before constructing the JSON payload. If a referenced line can't be confirmed in the diff, the comment moves to the review body instead of risking an inline comment on the wrong line.

An authorship check runs early in the workflow. If the PR author is the sole contributor (matching the current git user), the skill skips the GitHub posting flow entirely and presents findings in the terminal only. There's no value in an AI leaving yourself a review on your own single-author PR.

## Hooks as Guardrails

Skills define workflows. Hooks enforce invariants. The distinction matters because skills can be bypassed (Claude might commit without loading the skill), but hooks fire on every tool call regardless of context.

The hook system operates at two lifecycle points. PreToolUse hooks intercept tool calls before execution and can deny, warn, or inject context. PermissionRequest hooks fire when the built-in permission dialog would appear and can auto-approve or deny.

### The Permission Hook

The central guardrail is a Python script that handles all Bash command permissions across a four-tier model:

Tier 1 runs during PreToolUse and hard-denies catastrophic commands: `rm -rf /`, fork bombs, `curl | sh` pipelines, `dd` to raw devices, `kill -9 1`, and `mkfs`. Detection uses token-based parsing via `shlex`, which avoids false positives on quoted strings. Three regex fallbacks handle patterns that can't be tokenized cleanly (fork bombs, `eval` with `curl`, pipe-to-shell).

Tier 2 runs during PermissionRequest and auto-approves safe read-only commands: `git status`, `git log`, `git diff`, `ls`, `cat`, and similar. This prevents the permission dialog from appearing for commands that can't cause harm.

Tier 3 runs during PreToolUse and presents warnings for dangerous-but-sometimes-needed commands: `sudo`, force push to non-protected branches, `git reset --hard`, `pkill`, `chmod 777`. The warning appears in the permission dialog with context about why the command is flagged.

Tier 4 is implicit: everything not caught by the first three tiers falls through to the normal permission dialog.

Deny rules (also in the PermissionRequest path) block context-specific bad practices: `git add -A` or `git add .` (stage specific files instead), `--no-verify` on commits, force push to main/master, and chezmoi commands without explicit target paths. The hook splits chained commands on `&&`, `||`, `;`, and `|`, then checks each stage independently, so `git status && git add -A` still gets caught.

### Other Hooks

A branch name validator intercepts `git checkout -b`, `git switch -c`, and `git branch` commands, extracting the branch name and checking it against a regex of allowed patterns (`pr/will/*`, `dontmerge/will/*`, `backports/will/*`, `backup/*`, `worktree-*`). Non-conforming names trigger a warning with the convention table, not a hard block.

A formatting gate intercepts `git commit` commands and denies the first attempt, telling Claude to format staged files per the project's formatting standards before retrying. The gate uses a session-scoped temp file to track state: first commit attempt is denied, second attempt passes through. A `SKIP_FORMAT_CHECK=1` prefix bypasses the gate for cases where formatting is unnecessary.

A writing style guard intercepts all Edit, Write, and Bash tool calls and denies any that contain em-dashes or en-dashes. It scans for both the literal Unicode characters and escape sequences like `\u2014` or `\u2013` that would produce them. This enforces a writing convention across all output: prose, comments, commits, and PR descriptions. The hook supports a single-use bypass mechanism for legitimate uses: if a session-scoped marker file exists, the hook consumes it and allows the operation through.

A stop hook fires at session end and prompts for cleanup tasks. If the session is inside a git worktree with an auto-generated branch name (like `worktree-fancy-beaming-comet`), it prompts Claude to rename the branch to a convention-compliant name before ending. It also prompts for memory reflection if the session involved significant tool usage.

## Supporting Skills

Two additional skills handle the parts of git history management that the main three don't cover.

The fixup skill creates fixup commits and autosquashes them into their targets. It verifies the target commit by running `git log --oneline -- <file>` rather than guessing from descriptions, creates the fixup with `--fixup=<sha>`, then runs a non-interactive autosquash rebase using `GIT_SEQUENCE_EDITOR=true`. After squashing, it checks whether the target commit's message is now stale (because the fixup changed the nature of the commit) and suggests `/reword-commits` if so.

The commit surgery skill handles more complex history restructuring: squashing groups of commits, reordering, dropping changes from specific commits, and fixing cross-commit regressions. It uses a set of bundled Python scripts that act as `GIT_SEQUENCE_EDITOR` implementations, each handling a different rebase operation (reorder and squash, custom squash messages, marking commits for editing). The skill operates in phases: analyze, fix code issues that will surface after squashing, rewrite history, fix regressions from reordering, and verify against a backup branch.

## What This Gets You

The system produces commits, reviews, and PRs that are consistent with each other and with the project's documented conventions. PR descriptions are fact-checked against the actual code before anyone sees them. Reviews focus on genuine issues introduced by the PR, not pre-existing code smells. Commit messages match what the code actually does, not what it was originally intended to do.

The cost is complexity. Several skill files, a commit policy document, and about a dozen hook scripts, all of which need maintenance as Claude Code evolves. When a new version changes how tools are invoked or how hooks receive input, some of this breaks. The hooks are all tested (the test suite covers the token parsing, tier classification, and edge cases), but the skills are tested by use, not by automated test harnesses.

The benefit is trust. I review every PR and every commit, but I'm reviewing output that has already been through multiple verification stages. The verification sub-agent catches factual errors in PR descriptions that I might skim past. The confidence scoring filters out false positives that would erode trust in the review skill over time. The commit message consistency check catches the kind of drift that accumulates silently across a multi-commit branch.

The broader pattern applies independent of the specific tools. AI assistants that interact with shared systems (GitHub, CI, production infrastructure) need the same kind of process discipline that human contributors need. The mechanisms differ (hooks instead of code review norms, sub-agents instead of pair programming), but the goal is the same: make sure what gets shipped is accurate, consistent, and reviewed before it becomes someone else's problem.
