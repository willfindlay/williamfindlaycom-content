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

The top-level configuration file sets global rules and composes the rest via `@` imports. It establishes writing conventions (no em-dashes, no AI filler phrases, prose over bullets) and a memory policy that tells Claude where to persist lessons across sessions.

```markdown {collapsed="true" filename="~/.claude/CLAUDE.md"}
# Claude Code Context

@COMMITTING.md
@MEMORIES.md
@PRINCIPLES.md

## Writing Style

These rules apply to all output: prose, comments, commits, PRs, reviews.
- No em-dashes; use commas, periods, or parentheses
- No AI filler: "Furthermore", "Moreover", "Additionally", "It's worth noting", etc.
- No bold-labeled fields (`**Key**:`, `**Note**:`); weave context naturally
- Write in plain, direct language
- Prefer prose paragraphs over bullet lists; bullets are fine for reference lists, steps, and comparisons, but not for explaining ideas
- No salesmanship or hype; state facts and provide justification where needed
- No long runs of short punchy sentences; vary sentence length and rhythm
- Don't overuse formatting (bold, italics, inline code); apply where it aids clarity, not as decoration
- Don't pad text with specific numbers ("107 tests passed", "451 lines across 7 modules") unless the number itself matters

## Memory Policy

When saving insights to memory at session end:

**What's worth saving:** patterns, conventions, debugging insights, architectural decisions, user preferences, solutions to recurring problems, lessons from mistakes.
**Not worth saving:** session-specific context, speculative conclusions, anything already documented.

**Where to save:**
- **Project-specific** insights (only relevant to current project) -> auto memory
- **Globally relevant** insights (applicable across projects) -> ask the user before adding to `~/.claude/MEMORIES.md`
```

A principles document sets baseline expectations for how Claude approaches development.

```markdown {collapsed="true" filename="~/.claude/PRINCIPLES.md"}
# Core Development Principles

1. **Reuse Before Create**: Always search for existing implementations, patterns, utilities, and abstractions in the codebase before creating new ones
2. **No Backward Compatibility**: Never preserve unless explicitly requested
3. **Avoid Bloat**: No feature creep; treat every line of code as having a cost
4. **Ask, Don't Assume**: When in doubt, get clarification
5. **Meaningful Tests Only**: Test thoroughly but avoid trivial or tautological tests
6. **Evidence-Based Development**: All claims verifiable through testing, metrics, and documentation
7. **Context-Aware Generation**: Consider existing patterns, conventions, and architecture
8. **Fail Fast, Fail Explicitly**: Detect and report errors immediately with meaningful context
9. **Systems Thinking**: Consider ripple effects across entire system

## Planning and Design

10. **Clarify Before Committing**: When entering plan mode or designing implementation approaches, ask clarifying questions to align on requirements and constraints before finalizing a plan. Surface ambiguities, trade-offs, and key decisions early -- it's cheaper to course-correct during planning than during implementation.

Follow DRY, KISS, YAGNI, and SOLID principles in all design decisions.
```

Each of the three workflows lives in its own skill file. They share a common commit policy document that defines conventions (conventional commits, signed commits, atomic granularity, branch naming) and a set of hooks that enforce those conventions at the tool level, intercepting every Bash command before execution.

The commit policy defines conventions shared by all three skills.

````markdown {collapsed="true" filename="~/.claude/COMMITTING.md"}
# Git Commit Policy

## Golden Rule

**NEVER commit without first loading the `/commit` or `/pull-request` skill.**

## Commit Convention

- **Format**: `type(scope): description` (conventional commits)
- **Types**: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`, `build`, `ci`
- **Lowercase after colon**: e.g., `feat(auth): add token validation`, NOT `feat(auth): Add token validation`
- **Sign commits**: Always use `git commit -s` to auto-add `Signed-off-by`
- **No manual sign-off**: Never write `Signed-off-by:` in the message body
- **No AI attribution**: Never include `Co-Authored-By: Claude` or similar
- **Format before committing**: Format all modified files per Code Formatting Standards before committing
- **Stage specific files**: Prefer `git add <file>...` over `git add -A` or `git add .`
- **HEREDOC for message**: Always pass the commit message via HEREDOC:
  ```
  git commit -s -m "$(cat <<'EOF'
  type(scope): description
  EOF
  )"
  ```

## Commit Granularity

Separate commits by concern. Each commit should represent the smallest logical unit of change -- one bug fix, one feature addition, one refactoring step. Avoid bundling unrelated changes into a single commit. That said, don't go to extremes: a single-line typo fix in a file you're already modifying doesn't need its own commit.

When planning multi-commit work, organize the plan by commits -- group related changes together and identify the commit boundaries upfront. **Commit each unit (via `/commit`) immediately after implementing it, before starting the next.** Deferring all commits to the end forces painful hunk-splitting with `git add -p` when multiple commits touch the same file. Earlier commits also serve as save points.

## Merge Strategy

**ALWAYS use rebase merge (`--rebase`) when merging pull requests.** Never use squash merge or regular merge commits. This preserves individual atomic commits while maintaining a linear history.

Standard (from main worktree):
```
gh pr merge <number> --rebase --delete-branch
```

From a git worktree (`--delete-branch` fails because main is checked out in the main worktree):
```
gh pr merge <number> --rebase
git fetch origin main && git checkout --detach origin/main
git branch -D <branch-name>
git push origin --delete <branch-name>
```

## Branch Naming

The `branch-name-validator` hook enforces these conventions:

| Type | Pattern | Example |
|------|---------|---------|
| PR branch | `pr/<user>/<description>` | `pr/<user>/add-auth-middleware` |
| CI-only (no merge) | `dontmerge/<user>/<description>` | `dontmerge/<user>/test-flaky-ci` |
| Backport | `backports/<user>/<version-branch>/<description>` | `backports/<user>/v2.1/fix-crash` |
| Backup | `backup/<anything>` | `backup/pr/<user>/add-auth/1` |
| Worktree (auto) | `worktree-*` | `worktree-abc123` (auto-generated) |
````

The hook system is configured in `settings.json`, where each hook maps a lifecycle event and tool matcher to a command.

```json {collapsed="true" filename="~/.claude/settings.json"}
{
  "permissions": {
    "allow": [
      "Read",
      "List",
      "Glob",
      "Grep",
      "Task",
      "WebFetch",
      "WebSearch",
      "mcp__sequential-thinking__*",
      "mcp__context7__*",
      "mcp__playwright__*"
    ],
    "deny": [],
    "defaultMode": "default"
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$HOME/.claude/hooks/permissions.py pre-tool-use",
            "timeout": 10
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$HOME/.claude/hooks/git-guardrails.sh",
            "timeout": 10
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$HOME/.claude/hooks/branch-name-validator.py",
            "timeout": 10
          }
        ]
      },
      {
        "matcher": "Edit|Write|Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$HOME/.claude/hooks/writing-style-guard.py",
            "timeout": 10
          }
        ]
      }
    ],
    "PermissionRequest": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$HOME/.claude/hooks/permissions.py",
            "timeout": 10
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$HOME/.claude/hooks/pre-stop-checks.sh",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

## The Commit Skill

The commit skill is the simplest of the three and serves as a building block for the others. Its full definition is short: check for changes, then stage and commit following the policy.

```markdown {collapsed="true" filename="~/.claude/skills/commit/SKILL.md"}
---
name: commit
description: Commit changes. Use when the user asks to commit, save changes, or stage and commit code modifications.
argument-hint: [optional instructions]
allowed-tools: Bash(~/.claude/skills/commit/scripts/detect-chezmoi.sh)
---

!`~/.claude/skills/commit/scripts/detect-chezmoi.sh`

**If CHEZMOI=yes above**, you are in a chezmoi context. Use `chezmoi git` instead of `git` for ALL commands in this workflow:
- `git status` -> `chezmoi git status`
- `git diff ...` -> `chezmoi git -- diff ...`
- `git add <path>` -> `chezmoi git -- add <path>`
- `git commit ...` -> `chezmoi git -- commit ...`
- `git log ...` -> `chezmoi git -- log ...`

Paths in chezmoi git commands are relative to the chezmoi source directory. Use paths exactly as shown in `chezmoi git status` output.

Perform the following steps in order (create a todo list to follow along):
   1. Determine any uncommitted code changes using `git status`. If there are none, inform the user that there are no changes to commit and exit the workflow. If the user provided a list of files or directories in their instructions, consider ONLY those files or directories.
   2. Stage and commit changes following the conventions in COMMITTING.md. If the commit fails, report the error to the user.
```

What makes it more than a wrapper around `git commit` is the policy layer it inherits from `COMMITTING.md` (shown above). The policy specifies conventional commit format, requires signed commits, prohibits AI attribution lines, enforces HEREDOC syntax for messages, and demands atomic granularity. A golden rule sits at the top: never commit without first loading the `/commit` or `/pull-request` skill. This is a prompt-level constraint reinforced by a PreToolUse hook (discussed later) that intercepts `git commit` commands and denies the first attempt, requiring Claude to format staged files before retrying.

The commit skill also detects chezmoi contexts automatically. A bundled shell script checks whether the working directory is inside a chezmoi source directory or is chezmoi-managed but not inside a git worktree. When detected, the skill transparently rewrites all git commands to use `chezmoi git` instead, so the same workflow applies whether you're committing to a regular repo or to your dotfiles. A companion hook (chezmoi-git-reminder) enforces this at the tool level, denying bare `git` commands that target chezmoi-managed paths and telling Claude to use the `chezmoi git` subcommand equivalents.

The detection script is straightforward.

```bash {collapsed="true" filename="~/.claude/skills/commit/scripts/detect-chezmoi.sh"}
#!/usr/bin/env bash
# Detect if CWD is in a chezmoi context:
#   - CWD is inside the chezmoi source directory, OR
#   - CWD is chezmoi-managed and not inside a git worktree

chezmoi_src="$(chezmoi source-path 2>/dev/null)" || { echo "CHEZMOI=no"; exit 0; }
cwd="$(pwd)"

# CWD is inside chezmoi source directory
if [[ "$cwd" == "$chezmoi_src" || "$cwd" == "$chezmoi_src/"* ]]; then
    echo "CHEZMOI=yes"
    exit 0
fi

# CWD is chezmoi-managed but not inside a git worktree
if ! git rev-parse --is-inside-work-tree &>/dev/null && chezmoi source-path . &>/dev/null; then
    echo "CHEZMOI=yes"
    exit 0
fi

echo "CHEZMOI=no"
```

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

The complete skill definition includes the verification sub-agent's verbatim instructions and the user confirmation flow.

````markdown {collapsed="true" filename="~/.claude/skills/pull-request/SKILL.md"}
---
name: pull-request
description: Commit changes and create a pull request. Use when the user asks to create a PR, open a pull request, or submit changes for review.
argument-hint: [optional instructions]
---

Perform the following steps in order (create a todo list to follow along):

   1. Determine any uncommitted code changes using `git status`. If there are none, inform the user that there are no changes to commit and exit the workflow. If the user provided a list of files or directories in their instructions, consider ONLY those files or directories.

   2. If already on a feature branch, continue. Otherwise, create a branch named `pr/<user>/<description>` from the current branch and switch to it (see COMMITTING.md Branch Naming). If the current branch is a worktree branch (e.g. `worktree-*`), do NOT use it as the PR branch. Create a new `pr/<user>/<description>` branch from it instead, since worktree branch names are auto-generated and not meaningful.

   3. Stage and commit changes following the conventions in COMMITTING.md.

   4. Gather the diff and commit history against the base branch, then draft the PR title and body.

      Gather context:
      - `gh repo view --json defaultBranchRef -q .defaultBranchRef.name` for the base branch name
      - `git diff $(git merge-base <base> HEAD)..HEAD` for the full diff
      - `git log --oneline $(git merge-base <base> HEAD)..HEAD` for commits

      Write the PR body as plain prose paragraphs:
      - No markdown headings (`##`), no rigid sections like "Summary" or "Test plan", no bullet-driven descriptions.
      - Open with a short paragraph explaining what changed and why. If the motivation is non-obvious, include it; if the change is self-evident from the title, keep it brief.
      - Describe what you tested and what you observed inline, as part of the narrative. Don't separate testing into its own section.
      - Factual tone. Don't editorialize ("this greatly improves...") or sell the change.
      - A bullet list is fine when it's genuinely the clearest format (e.g., a PR touching many unrelated files), but default to prose.

      Examples of good PR bodies:

      Small fix:
      ```
      The session-start hook was clearing counter files on every SessionStart
      event, including "connect" events that fire when resuming an existing
      session. This meant counters reset mid-session whenever context
      compressed. Now it only clears on "clear" and "compact" events.

      Ran the hook test suite and confirmed the connect event no longer
      triggers a reset.
      ```

      Larger change:
      ```
      This replaces the BM25-based skill matching with a static catalog
      injected via the system reminder. The BM25 approach created anchoring
      bias where Claude would fixate on suggested skills and ignore unlisted
      ones, and the corpus was too small for IDF to be meaningful.

      The skill catalog is already loaded every session, so discovery is
      really a context problem rather than a retrieval problem. Improving
      skill descriptions with explicit trigger phrases gives better results
      without the matching infrastructure.

      All existing tests pass. Manually verified that skill invocation still
      works by running /commit and /pull-request in a scratch repo.
      ```

   5. Launch a verification sub-agent to check the draft against the actual code.

      Use the Agent tool with `subagent_type: "general-purpose"` and `model: "sonnet"`. Pass the agent the following:
      - The draft PR body (full text)
      - The full diff (`git diff <merge-base>..HEAD`)
      - The diff stat (`git diff --stat <merge-base>..HEAD`)
      - The commit list (`git log --oneline <merge-base>..HEAD`)

      Include these instructions verbatim in the agent prompt:

      > You are verifying a pull request description against the actual code changes. Your job is to confirm that every factual claim in the PR body is accurate and supported by evidence.
      >
      > Go paragraph by paragraph through the PR body. For each paragraph:
      > 1. Identify every factual claim (behavior described, files changed, patterns used, test results mentioned, etc.)
      > 2. Verify each claim by reading the relevant source files, examining the diff, running `git log`, `git show`, or other read-only git commands as needed
      > 3. Produce a section with:
      >    - The paragraph text (quoted)
      >    - A list of claims and their supporting evidence, with `file:line` references where applicable
      >    - A flag for anything that is inaccurate, unsupported by the diff/code, or exaggerated
      >
      > You have full read access to the repository. Use it. Don't just trust the diff, read the actual files to confirm behavior.
      >
      > At the end, provide a summary: how many claims verified, how many flagged, and whether the PR body is accurate overall.

      After receiving the verification report, fix any flagged inaccuracies in the draft. Do NOT re-run the sub-agent after making corrections.

   6. Present the draft to the user for confirmation.

      Show the PR title, then the body paragraph by paragraph. After each paragraph, show the verification justification in a blockquote with `file:line` references. If any claims required correction, note what was changed.

      Use `AskUserQuestion` with the question "Create this pull request?" and options:
      - "Create PR" (description: "Post the PR as shown above")
      - "Let me give feedback first" (description: "I want to suggest changes to the title or body")
      - "Cancel" (description: "Abort without creating a PR")

      If the user gives feedback: apply their changes to the draft, re-present the updated version, and ask again. Do NOT re-run the verification sub-agent for user-requested edits.

   7. Check that commit message bodies are consistent with the finalized PR description. Read each commit's full message (`git log --format="%H%n%B" <merge-base>..HEAD`) and compare against the PR body. If any commit body makes claims that contradict the PR description, describes behavior differently, or is stale relative to what the code actually does, inform the user about the discrepancies and use the `/reword-commits` skill to fix them before continuing.

   8. Push the branch and create the pull request:
      ```
      git push -u origin <branch>
      gh pr create --title "..." --body "$(cat <<'EOF'
      ...
      EOF
      )" --assignee @me
      ```
      Display the PR URL.

## Merging from worktrees

`gh pr merge --rebase --delete-branch` fails inside git worktrees because `--delete-branch` tries to checkout `main` locally, which is already checked out in the main worktree. Instead, merge without `--delete-branch` and clean up manually:

```bash
gh pr merge <number> --rebase
git fetch origin main && git checkout --detach origin/main
git branch -D <branch-name>
git push origin --delete <branch-name>
```
````

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

````markdown {collapsed="true" filename="~/.claude/skills/reword-commits/SKILL.md"}
---
name: reword-commits
description: Reword commit messages to accurately reflect their content. Use when commit messages are stale, inaccurate, or need updating after squashing/rebasing.
argument-hint: [base-ref or commit range, e.g. "main" or "HEAD~4"]
---

Reword commit messages on the current branch so each message accurately describes its diff. Follow these steps:

1. **Determine the range.** If the user provided a base ref or range, use it. Otherwise, detect the default branch (`git symbolic-ref refs/remotes/origin/HEAD`, strip prefix; fall back to `origin/main` or `origin/master`) and find the merge base (`git merge-base <default-branch> HEAD`).

2. **Back up the branch:**
   ```
   git branch backup/<branch-name>
   ```

3. **Audit each commit.** For every commit in the range:
   - Read the full diff (`git diff <commit>^..<commit>`)
   - Read the current commit message (`git log -1 --format='%B' <commit>`)
   - Identify inaccuracies: features described that were later reverted, outdated terminology, removed code still referenced, wrong behavior descriptions
   - Track which commits need rewording and which are already accurate

4. **Draft new messages.** For each commit that needs changes, draft a replacement following the project's commit conventions (see COMMITTING.md). Rules:
   - Title: `type(scope): description` in lowercase after colon, under 72 chars
   - Body: describe what the commit actually does based on its diff, not what it was originally intended to do
   - Do not mention removed/reverted features unless the commit itself removes them
   - Keep the original `Signed-off-by` line (use `git commit --amend -s`)
   - Do not add AI attribution lines

5. **Present changes for approval.** Show the user a table of old vs new messages before proceeding. Wait for confirmation.

6. **Execute the rebase.** Use the bundled `reword-todo.py` script to mark only the commits needing rewording as `edit`:

   ```
   GIT_SEQUENCE_EDITOR="~/.claude/skills/reword-commits/scripts/reword-todo.py <sha1> <sha2> ..." \
   git rebase -i <base>
   ```

   At each stop, amend the message with `git commit --amend -s` and continue with `git rebase --continue`.

7. **Verify integrity.** After the rebase:
   - `git diff backup/<branch-name>` should be empty (tree unchanged)
   - `git log --oneline <base>..HEAD` should show the updated titles

8. **Clean up.** Do NOT force push unless the user explicitly asks. Do NOT delete the backup branch, let the user decide.
````

The `GIT_SEQUENCE_EDITOR` script it uses to drive the rebase:

```python {collapsed="true" filename="~/.claude/skills/reword-commits/scripts/reword-todo.py"}
#!/usr/bin/env python3
"""GIT_SEQUENCE_EDITOR script: mark commits for rewording in a rebase todo.

Usage: reword-todo.py <todo-file> <sha> [<sha>...]

Any pick line whose SHA prefix-matches one of the provided SHAs is changed
from 'pick' to 'edit'. All other lines are left unchanged.
"""
import sys


def main():
    if len(sys.argv) < 3:
        print("usage: reword-todo.py <todo-file> <sha> [<sha>...]", file=sys.stderr)
        sys.exit(1)

    todo_file = sys.argv[1]
    reword_shas = set(sys.argv[2:])

    with open(todo_file) as f:
        lines = f.readlines()

    result = []
    for line in lines:
        if line.startswith("pick "):
            sha = line.split()[1]
            if sha in reword_shas or any(sha.startswith(s) for s in reword_shas):
                result.append(line.replace("pick ", "edit ", 1))
            else:
                result.append(line)
        else:
            result.append(line)

    with open(todo_file, "w") as f:
        f.writelines(result)


if __name__ == "__main__":
    main()
```

This ordering (approve PR body, then check commit consistency, then push) means the push happens last, after all local work is finalized. Commit messages, PR body, and code are all consistent before anything leaves the machine.

## The Code Review Skill

The code review skill is the most complex of the three. It reviews a pull request for code quality, bugs, SOLID violations, anti-patterns, and compliance with project-specific instructions (documented in `CLAUDE.md` files). It produces findings as prose, presents them for approval, and optionally posts a GitHub review with inline comments.

The full skill definition runs through all nine steps.

````markdown {collapsed="true" filename="~/.claude/skills/code-review/SKILL.md"}
---
name: code-review
description: Review a pull request for code quality, bugs, SOLID violations, anti-patterns, and project instruction compliance (CLAUDE.md, CLAUDE.local.md, AGENTS.md). Resolves target as explicit PR number or auto-detect PR for current branch. Presents findings as prose and asks for confirmation before posting a GitHub review. Use when the user asks for a code review, wants feedback, or asks to review a PR.
argument-hint: "[PR number] or omit to auto-detect"
allowed-tools: Bash(~/.claude/skills/code-review/scripts/verify-commits.sh)
---

Review code for code quality, bugs, SOLID violations, anti-patterns, and project instruction compliance (CLAUDE.md, CLAUDE.local.md, AGENTS.md).

## Step 1: Target resolution

Resolve the review target:

1. If `$ARGUMENTS` is a number (eg. `123`): use that as the **PR number**
2. If `$ARGUMENTS` is empty: run `gh pr view --json number -q .number` to find a PR for the current branch
3. If no PR is found (command fails or no open PR): tell the user no PR was found for the current branch and suggest they create one first

## Step 2: Gather context

Run in parallel:
- `gh pr view <N> --json title,baseRefName,headRefName,body,url,number`
- `gh repo view --json owner,name -q '.owner.login + "/" + .name'`
- `gh pr diff <N>`
- Glob for `**/CLAUDE.md`, `**/CLAUDE.local.md`, and `**/AGENTS.md`, filter to root + directories touched by the diff

Derive merge base: `git merge-base origin/<baseRefName> HEAD`

Then gather the PR commit list (this becomes the **allowlist** for commit attribution):
```bash
git log --format='%h %H %s' $merge_base..HEAD
```
Store the short SHA, full SHA, and subject for each commit. Any issue reported in step 4 must trace back to one of these commits.

## Step 3: Authorship check

Determine whether GitHub posting is available:

```bash
merge_base=$(git merge-base origin/<baseRefName> HEAD)
authors=$(git log "$merge_base"..HEAD --format=%ae | sort -u)
current_user=$(git config --get user.email)
```

- Single author matching `current_user` -> **sole author** (no GitHub posting; present findings in terminal only)
- Otherwise -> **not sole author** (GitHub posting available in step 9)

## Step 4: Parallel review (3-4 Sonnet subagents)

If no project instruction files were found in step 2, skip agent #2 (project instruction compliance) and launch only 3 agents.

Launch parallel Sonnet subagents via Task. Each agent receives:
- The **PR diff** as its primary source material
- The **PR summary** (title, body)
- The **PR commit list** from step 2 (short SHA, full SHA, subject for each commit)
- The **relevant project instruction file paths** (CLAUDE.md, CLAUDE.local.md, AGENTS.md; contents included, if any exist)

Each agent's prompt must include this scoping constraint verbatim:

> Your analysis scope is strictly limited to the diff provided. Do NOT use tools to read files, git blame, or any context beyond the diff and the project instruction files listed (CLAUDE.md, CLAUDE.local.md, AGENTS.md). Every issue you report MUST reference a specific line that appears in the diff. Determine which PR commit introduced each issue by matching the file and hunk line range to the commit list provided.

Each agent returns a list of issues with: category, severity, **commit (short SHA)**, file:line(s), description, fix suggestion.

1. **Code quality (SOLID + anti-patterns)**: SRP/OCP/LSP/ISP/DIP violations, tautological tests, DRY/YAGNI/KISS violations, god objects, silent failures, logic duplication in tests. Only violations **introduced by this PR**.
2. **Project instruction compliance**: Read each project instruction file (CLAUDE.md, CLAUDE.local.md, AGENTS.md) and audit changes against them. Cite specific rules violated with file paths.
3. **Shallow bug scan**: Scan diff only for obvious bugs -- logic errors, null handling, off-by-one, race conditions, resource leaks. Focus on large bugs, avoid nitpicks. Ignore likely false positives.
4. **Test coverage czar**: Flag new/modified logic paths lacking test coverage. Ignore trivial changes and test file changes. Only flag where missing tests pose real regression risk.

## Step 5: Commit verification gate

Collect all agent issues into a JSON array (each object must have a `commit` field with a short SHA). Save the PR commit list as a JSON array of `{"short_sha": "...", "full_sha": "..."}` objects. Then run:

```
~/.claude/skills/code-review/scripts/verify-commits.sh /tmp/issues.json /tmp/commits.json
```

The script drops issues with missing or non-matching commit SHAs and outputs the filtered array. This is a hard structural filter, not prompt-dependent.

## Step 6: Confidence scoring (parallel Haiku subagents)

For each issue surviving step 5, launch a parallel Haiku subagent that takes the PR, issue description, and list of project instruction files (from step 2), and returns a confidence score (0-100). For issues flagged due to project instructions, the agent should double check that the relevant file actually calls out that issue specifically. Give the agent this rubric verbatim:

- **0**: Not confident at all. This is a false positive that doesn't stand up to light scrutiny, is a pre-existing issue, or the issue cannot be attributed to a specific commit in the PR's commit range.
- **25**: Somewhat confident. This might be a real issue, but may also be a false positive. The agent wasn't able to verify that it's a real issue. If the issue is stylistic, it is one that was not explicitly called out in the relevant project instruction files.
- **50**: Moderately confident. The agent was able to verify this is a real issue, but it might be a nitpick or not happen very often in practice. Relative to the rest of the PR, it's not very important.
- **75**: Highly confident. The agent double checked the issue, and verified that it is very likely it is a real issue that will be hit in practice. The existing approach in the PR is insufficient. The issue is very important and will directly impact the code's functionality, or it is an issue that is directly mentioned in the relevant project instruction files.
- **100**: Absolutely certain. The agent double checked the issue, and confirmed that it is definitely a real issue, that will happen frequently in practice. The evidence directly confirms this.

## Step 7: Filter

Remove all issues with a confidence score less than 70. After filtering, you may use your own judgement to **include** borderline issues (scored 50-69) that you believe are genuine and worth flagging -- but never use discretion to **exclude** issues that scored 70 or above. When in doubt, err on the side of including. If no issues remain, skip to step 9 with "no issues" output.

## Step 8: Deduplication

Fetch existing review comments on the PR:
```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments
```

For each surviving issue from step 7, check whether an existing comment already covers the same concern. A match requires all of:
- Same file
- Nearby line (within ~5 lines)
- Similar topic (the existing comment addresses the same underlying problem)

Split the surviving issues into two groups:
- **New issues**: no match found among existing comments. These become inline comments in step 9.
- **Already-flagged issues**: matched to an existing reviewer comment. Record the reviewer's login and the numeric comment ID (from the REST API response). These get +1 reactions during posting (step 9c) instead of becoming inline comments.

## Step 9: Output

### 9a. Present findings as prose

Write each issue as a short prose paragraph. No structured template headers. One paragraph per issue covering: what the problem is, where it is (file:line), and what should be done about it. Write like a human reviewer leaving comments, direct and specific, no filler.

Writing constraints:
- Keep each comment to 2-4 sentences
- Confidence scores are internal bookkeeping and should not appear in the output. After each prose paragraph, include a short attribution line: `Commit: \`abc1234\` -- commit subject line`

Lead with a natural summary instead of a rigid template. Examples:
- "Agreeing with David's findings about X. One additional thing caught my eye: ..."
- "A few things stood out while reading through the diff..."
- No issues: "Looks good to me, no concerns."

If the dedup step (step 8) found already-flagged issues, present them in a separate section noting which reviewer raised each one and that +1 reactions will be added when posting.

### 9b. Ask user for feedback

If **sole author** (from step 3): skip this sub-step entirely. Do not offer to post. The terminal output from 9a is the final deliverable.

Otherwise, after presenting all comments, use `AskUserQuestion` to ask: "Want me to post these as a GitHub review?" with options:
- "Post as-is"
- "Let me give feedback first"
- "Don't post"

If "Let me give feedback first": the user provides feedback (drop certain comments, reword, etc.). Apply their changes, present the updated list, and ask again.

If "Post as-is": proceed to 9c.

If "Don't post": done.

### 9c. Post GitHub review

**Line-number verification (mandatory before constructing JSON):**
For each inline comment, verify the line number against the actual diff:
1. Search the `gh pr diff <N>` output for the specific code pattern (function name, variable, expression) the comment refers to
2. Extract the exact line number from the diff -- the right-side line number in the new version of the file
3. If the pattern can't be found in the diff, move the comment into the review body instead

**Auto-react to duplicates:**
Before posting the review, add +1 reactions to existing comments matched in the dedup step (step 8):
```bash
gh api repos/{owner}/{repo}/pulls/comments/{comment_id}/reactions -f content='+1'
```
Use numeric REST API comment IDs (not GraphQL node IDs).

**Construct and post the review:**

Write the full JSON payload to a temp file and post via `gh api`:

```bash
cat > /tmp/review.json <<'REVIEW_EOF'
{
  "event": "COMMENT",
  "body": "<summary body>",
  "comments": [
    {"path": "file.ext", "line": 15, "side": "RIGHT", "body": "..."}
  ]
}
REVIEW_EOF
gh api "repos/{owner}/{repo}/pulls/{number}/reviews" --input /tmp/review.json
```

**Comment fields:**
- `path`: file path relative to repo root
- `line`: the line number in the NEW version of the file (right-side number from the diff hunk header `@@ -old,len +new,len @@`). Must be verified against `gh pr diff` output in the line-number verification step above.
- `start_line`: starting line number for multi-line comments (optional, omit for single-line)
- `side`: always `"RIGHT"` (commenting on the new version)
- `body`: the prose paragraph from 9a only -- omit the `Commit:` attribution line (that's for terminal output only)

**Important constraints:**
- Lines referenced in comments **must be part of the diff hunk** and **must be verified** via the line-number verification step above. If an issue is on a line outside the diff, fall back to mentioning it in the review body instead.
- Do not reference project instruction files (CLAUDE.md, etc.) by name in comment bodies -- describe the underlying principle instead
- Avoid emojis

**Review body:**
Write a natural paragraph. Reference agreements with other reviewers by name and introduce new findings conversationally. Examples:
- "Agreeing with David's findings about X. One additional thing caught my eye: ..."
- "A few things stood out while reading through the diff..."
- No issues: "Looks good to me, no concerns."

**After posting:** Extract the `html_url` from the API response and display it to the user as a clickable link to the review.

## False positive guidance (for steps 4 and 6)

Exclude these categories from findings:

- Pre-existing issues not introduced in this PR
- Something that looks like a bug but is not actually a bug
- Pedantic nitpicks that a senior engineer wouldn't call out
- Issues a linter, typechecker, or compiler would catch (eg. missing imports, type errors, broken tests, formatting). Assume CI runs these separately.
- General code quality issues (eg. lack of test coverage, general security issues, poor documentation), unless explicitly required in project instruction files
- Issues called out in project instruction files but explicitly silenced in the code (eg. lint ignore comments)
- Changes in functionality that are likely intentional or directly related to the broader change
- Real issues on lines the user did not modify in their pull request

## Notes

- Do not check build signal or attempt to build/typecheck. These run separately via CI.
- Use `gh` to interact with GitHub, not web fetch.
- Do not reference project instruction files (CLAUDE.md, etc.) by name in review comments posted to GitHub. Describe the underlying rule or principle instead.
- Each comment should read like it was written by a human reviewer.

Never attempt or offer to commit changes under any circumstances.
````

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

The verification script is a `jq` pipeline.

```bash {collapsed="true" filename="~/.claude/skills/code-review/scripts/verify-commits.sh"}
#!/usr/bin/env bash
# Verify that issues reported by review agents reference valid PR commits.
#
# Usage: verify-commits.sh <issues-json> <commits-json>
#
# Arguments:
#   issues-json   - Path to JSON file with array of issue objects, each having
#                   a "commit" field (short SHA string or empty/null)
#   commits-json  - Path to JSON file with array of valid commit objects, each
#                   having "short_sha" and "full_sha" fields
#
# Output: filtered JSON array of issues whose commit SHAs match a PR commit.
#         Issues without a commit SHA or with non-matching SHAs are dropped.
#         A summary line is printed to stderr.
#
# Requires: jq

set -euo pipefail

if [[ $# -lt 2 ]]; then
    echo "usage: verify-commits.sh <issues-json> <commits-json>" >&2
    exit 1
fi

ISSUES_FILE="$1"
COMMITS_FILE="$2"

if ! command -v jq &>/dev/null; then
    echo "error: jq is required but not installed" >&2
    exit 1
fi

# Extract valid SHAs (both short and full) into a newline-separated list
valid_shas=$(jq -r '.[] | .short_sha, .full_sha' "$COMMITS_FILE")

total=$(jq 'length' "$ISSUES_FILE")
# Filter: keep issues where .commit is non-null, non-empty, and prefix-matches
# at least one valid SHA
jq --arg shas "$valid_shas" '
  ($shas | split("\n") | map(select(. != ""))) as $valid |
  [.[] | select(
    .commit != null and .commit != "" and
    (. as $issue | $valid | any(. as $sha |
      ($sha | startswith($issue.commit)) or ($issue.commit | startswith($sha))
    ))
  )]
' "$ISSUES_FILE" > /tmp/verified-issues.json

kept=$(jq 'length' /tmp/verified-issues.json)
dropped=$((total - kept))

echo "Commit verification: $kept/$total issues passed ($dropped dropped)" >&2
cat /tmp/verified-issues.json
```

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

The real implementation is a ~2700-line Python script that handles token parsing, command splitting, and edge cases across git, gh, kubectl, helm, chezmoi, and dozens of other tools. It will be open sourced at a later date. The skeleton below shows the architectural pattern: data-driven rule tables fed into a four-tier dispatcher.

```python {collapsed="true" filename="~/.claude/hooks/permissions.py"}
#!/usr/bin/env python3
"""permissions.py - Bash command permission hook (PreToolUse + PermissionRequest)

Four-tier permission model:
  Tier 1 - Deny:      Catastrophic commands, hard block (token + regex fallbacks, PreToolUse)
  Tier 2 - Allow:     Safe/read-only/idempotent, auto-approve (token, PermissionRequest)
  Tier 3 - Ask+Warn:  Dangerous but sometimes needed, warning dialog (token, PreToolUse)
  Tier 4 - Ask:       Everything else, normal permission dialog (implicit fall-through)

Invocation:
  permissions.py pre-tool-use    # PreToolUse event: Tier 1 + Tier 3
  permissions.py                 # PermissionRequest event: deny rules + Tier 2 allow
"""

import json
import re
import shlex
import sys

# ---------------------------------------------------------------------------
# Rule definitions (abbreviated; real tables cover dozens of tools)
# ---------------------------------------------------------------------------

GIT_READONLY_SIMPLE = frozenset({
    "blame", "status", "log", "diff", "show", "ls-files",
    "rev-parse", "merge-base", "describe", "shortlog",
})

GIT_READONLY_FLAGGED = {
    "branch": frozenset({"-l", "--list", "-a", "--show-current"}),
    "remote": frozenset({"-v", "get-url"}),
    "config": frozenset({"--get", "--list", "-l"}),
    "tag":    frozenset({"-l", "--list"}),
    "stash":  frozenset({"list"}),
}

SAFE_COMMANDS = frozenset({
    "ls", "cat", "head", "tail", "wc", "file", "stat", "which",
    "echo", "printf", "date", "whoami", "pwd", "env", "uname",
    "jq", "yq", "sed", "awk", "grep", "rg", "find", "sort",
    "uniq", "tr", "cut", "tee", "diff", "sha256sum", "md5sum",
})

CATASTROPHIC_PATTERNS = [
    re.compile(r"rm\s+-[^\s]*r[^\s]*f.*(/|~|\$HOME)", re.IGNORECASE),
    re.compile(r":\(\)\s*\{.*\|.*&\s*\};"),   # fork bomb
    re.compile(r"mkfs\b"),
]

DENY_RULES = [
    ("git add -A or git add .", lambda tokens: _is_git_add_all(tokens)),
    ("--no-verify on commits",  lambda tokens: _has_no_verify(tokens)),
    ("force push to main",     lambda tokens: _is_force_push_main(tokens)),
]

WARN_COMMANDS = {
    "sudo":  "Running with elevated privileges",
    "pkill": "Killing processes by name",
    "chmod": "Changing file permissions",
}


# ---------------------------------------------------------------------------
# Token parsing
# ---------------------------------------------------------------------------

def tokenize(command: str) -> list[str]:
    """Split command into tokens, handling quotes and escapes."""
    try:
        return shlex.split(command, comments=True)
    except ValueError:
        return command.split()


def split_stages(command: str) -> list[str]:
    """Split chained commands on &&, ||, ;, and | into stages."""
    # ... splits command string, returns list of individual commands
    pass


# ---------------------------------------------------------------------------
# Tier implementations
# ---------------------------------------------------------------------------

def tier1_deny(tokens: list[str]) -> str | None:
    """Catastrophic command detection (hard block)."""
    command_str = " ".join(tokens)
    for pattern in CATASTROPHIC_PATTERNS:
        if pattern.search(command_str):
            return f"Blocked catastrophic command: {command_str}"
    # ... additional token-based checks for dd, kill -9 1, curl|sh, etc.
    return None


def tier2_allow(tokens: list[str]) -> bool:
    """Safe read-only command detection (auto-approve)."""
    if not tokens:
        return False
    cmd = tokens[0]
    if cmd in SAFE_COMMANDS:
        return True
    if cmd == "git" and len(tokens) > 1:
        subcmd = tokens[1]
        if subcmd in GIT_READONLY_SIMPLE:
            return True
        if subcmd in GIT_READONLY_FLAGGED:
            return any(t in GIT_READONLY_FLAGGED[subcmd] for t in tokens[2:])
    # ... gh, docker inspect, kubectl get, etc.
    return False


def tier3_warn(tokens: list[str]) -> str | None:
    """Dangerous-but-sometimes-needed command detection (warning)."""
    if not tokens:
        return None
    cmd = tokens[0]
    if cmd in WARN_COMMANDS:
        return WARN_COMMANDS[cmd]
    # ... git reset --hard, force push to non-protected branches, etc.
    return None


def check_deny_rules(tokens: list[str]) -> str | None:
    """Context-specific bad practice detection (PermissionRequest deny)."""
    for description, check in DENY_RULES:
        if check(tokens):
            return description
    return None


# ---------------------------------------------------------------------------
# Dispatcher
# ---------------------------------------------------------------------------

def handle_pre_tool_use(command: str) -> dict | None:
    """PreToolUse path: Tier 1 (deny) + Tier 3 (warn)."""
    for stage in split_stages(command):
        tokens = tokenize(stage)
        # Tier 1: hard deny
        reason = tier1_deny(tokens)
        if reason:
            return {"hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": reason,
            }}
        # Tier 3: warn
        reason = tier3_warn(tokens)
        if reason:
            return {"hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "ask",
                "permissionDecisionReason": reason,
            }}
    return None


def handle_permission_request(command: str) -> dict | None:
    """PermissionRequest path: deny rules + Tier 2 (allow)."""
    for stage in split_stages(command):
        tokens = tokenize(stage)
        # Deny rules: block bad practices
        reason = check_deny_rules(tokens)
        if reason:
            return {"hookSpecificOutput": {
                "hookEventName": "PermissionRequest",
                "permissionDecision": "deny",
                "permissionDecisionReason": reason,
            }}
    # Tier 2: auto-approve if all stages are safe
    stages = split_stages(command)
    if all(tier2_allow(tokenize(s)) for s in stages):
        return {"hookSpecificOutput": {
            "hookEventName": "PermissionRequest",
            "permissionDecision": "allow",
        }}
    return None  # Tier 4: fall through to normal dialog


def main() -> None:
    data = json.load(sys.stdin)
    command = data.get("tool_input", {}).get("command", "")
    if not command:
        return

    is_pre_tool_use = len(sys.argv) > 1 and sys.argv[1] == "pre-tool-use"

    if is_pre_tool_use:
        result = handle_pre_tool_use(command)
    else:
        result = handle_permission_request(command)

    if result:
        json.dump(result, sys.stdout)


if __name__ == "__main__":
    main()
```

### Other Hooks

A branch name validator intercepts `git checkout -b`, `git switch -c`, and `git branch` commands, extracting the branch name and checking it against a regex of allowed patterns (`pr/<user>/*`, `dontmerge/<user>/*`, `backports/<user>/*`, `backup/*`, `worktree-*`). Non-conforming names trigger a warning with the convention table, not a hard block.

```python {collapsed="true" filename="~/.claude/hooks/branch-name-validator.py"}
#!/usr/bin/env python3
"""branch-name-validator.py -- PreToolUse hook (matcher: Bash)

Validates branch names against the project's naming conventions when
Claude creates a new branch. Intercepts:
- git checkout -b <name>
- git switch -c / --create <name>
- git branch <name> (when not a read-only flag like -l, -a, -D, etc.)

Allowed patterns:
- pr/<user>/<description>
- dontmerge/<user>/<description>
- backports/<user>/<version>/<description>
- backup/<anything>
- worktree-*

On mismatch: returns permissionDecision "ask" with guidance.
On match or non-branch command: exits 0 silently.
"""

import json
import re
import shlex
import sys

ALLOWED_PATTERNS = re.compile(
    r"^("
    r"(pr|dontmerge|backports)/<user>/"
    r"|backup/"
    r"|worktree-"
    r")"
)

# git branch flags that do NOT create a branch (read-only or destructive ops
# that aren't "create new branch"). When these appear as the first arg after
# `git branch`, skip validation.
BRANCH_READONLY_FLAGS = frozenset({
    "-l", "--list",
    "-a", "--all",
    "-r", "--remotes",
    "-d", "-D", "--delete",
    "-m", "-M", "--move",
    "-c", "-C", "--copy",
    "-v", "-vv", "--verbose",
    "--merged", "--no-merged",
    "--contains", "--no-contains",
    "--sort", "--format",
    "--show-current",
    "--set-upstream-to", "--unset-upstream",
    "--edit-description",
    "--track", "--no-track",
})


def extract_branch_name(command: str) -> str | None:
    """Extract the new branch name from a branch-creation command.

    Returns the branch name if the command creates a branch, None otherwise.
    """
    try:
        tokens = shlex.split(command, comments=True)
    except ValueError:
        return None

    if not tokens:
        return None

    # Handle chained commands: only check the git portion(s)
    # Split on &&, ||, ; and check each segment
    segments = _split_command_segments(command)
    for segment in segments:
        name = _extract_from_segment(segment)
        if name is not None:
            return name

    return None


def _split_command_segments(command: str) -> list[str]:
    """Split a command string on &&, ||, and ; into segments."""
    segments = []
    current = []
    i = 0
    chars = command
    while i < len(chars):
        if chars[i] == ";" or (chars[i] == "&" and i + 1 < len(chars) and chars[i + 1] == "&"):
            segments.append("".join(current).strip())
            current = []
            i += 2 if chars[i] == "&" else 1
            continue
        if chars[i] == "|" and i + 1 < len(chars) and chars[i + 1] == "|":
            segments.append("".join(current).strip())
            current = []
            i += 2
            continue
        current.append(chars[i])
        i += 1
    remainder = "".join(current).strip()
    if remainder:
        segments.append(remainder)
    return segments


def _extract_from_segment(segment: str) -> str | None:
    """Extract branch name from a single command segment."""
    try:
        tokens = shlex.split(segment.strip(), comments=True)
    except ValueError:
        return None

    if not tokens or tokens[0] != "git":
        return None

    args = tokens[1:]
    if not args:
        return None

    subcmd = args[0]

    # git checkout -b <name>
    if subcmd == "checkout":
        return _extract_checkout_branch(args[1:])

    # git switch -c / --create <name>
    if subcmd == "switch":
        return _extract_switch_branch(args[1:])

    # git branch <name>
    if subcmd == "branch":
        return _extract_branch_create(args[1:])

    return None


def _extract_checkout_branch(args: list[str]) -> str | None:
    """Extract branch name from `git checkout -b <name> [start]`."""
    i = 0
    while i < len(args):
        if args[i] in ("-b", "-B"):
            if i + 1 < len(args):
                return args[i + 1]
            return None
        i += 1
    return None


def _extract_switch_branch(args: list[str]) -> str | None:
    """Extract branch name from `git switch -c <name>` or `git switch --create <name>`."""
    i = 0
    while i < len(args):
        if args[i] in ("-c", "-C", "--create"):
            if i + 1 < len(args):
                return args[i + 1]
            return None
        i += 1
    return None


def _extract_branch_create(args: list[str]) -> str | None:
    """Extract branch name from `git branch <name>` (non-read-only usage)."""
    if not args:
        return None

    # If the first non-flag argument is a read-only flag, skip
    for arg in args:
        if arg in BRANCH_READONLY_FLAGS or arg.startswith("--"):
            return None
        if not arg.startswith("-"):
            # First positional arg is the branch name
            return arg

    return None


def build_ask(branch_name: str) -> dict:
    """Build an 'ask' response for a non-conforming branch name."""
    reason = (
        f'Branch name "{branch_name}" does not match naming conventions.\n'
        "Expected patterns:\n"
        "  pr/<user>/<description>        - PR branches\n"
        "  dontmerge/<user>/<description> - CI-only branches\n"
        "  backports/<user>/<v>/<desc>    - Backport branches\n"
        "  backup/<anything>              - Backup branches\n"
        "  worktree-*                     - Auto-generated worktree branches"
    )
    return {
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "ask",
            "permissionDecisionReason": reason,
        }
    }


def main() -> None:
    try:
        data = json.load(sys.stdin)
    except (json.JSONDecodeError, EOFError):
        return

    command = data.get("tool_input", {}).get("command", "")
    if not command:
        return

    branch_name = extract_branch_name(command)
    if branch_name is None:
        return

    if ALLOWED_PATTERNS.match(branch_name):
        return

    json.dump(build_ask(branch_name), sys.stdout)


if __name__ == "__main__":
    main()
```

A formatting gate intercepts `git commit` commands and denies the first attempt, telling Claude to format staged files per the project's formatting standards before retrying. The gate uses a session-scoped temp file to track state: first commit attempt is denied, second attempt passes through. A `SKIP_FORMAT_CHECK=1` prefix bypasses the gate for cases where formatting is unnecessary.

```bash {collapsed="true" filename="~/.claude/hooks/git-guardrails.sh"}
#!/usr/bin/env bash
# git-guardrails.sh -- PreToolUse hook (matcher: Bash)
# Formatting gate: deny first commit attempt so Claude formats first, allow retry.
# PR reminders: prompt to verify tests and consider code review.

set -euo pipefail

input="$(cat)"
command="$(echo "$input" | jq -r '.tool_input.command // ""')"

# Fast-path: only care about git commit and gh pr create
case "$command" in
  git\ commit*) ;;
  gh\ pr\ create*) ;;
  *) exit 0 ;;
esac

# --- git commit: formatting gate (deny) ---
if [[ "$command" == git\ commit* ]]; then
  # Allow intentional skip (e.g., SKIP_FORMAT_CHECK=1 git commit ...)
  if [[ "$command" == SKIP_FORMAT_CHECK=1\ * ]]; then
    exit 0
  fi

  # Collect staged files, filtering out non-formattable extensions (.md, .txt)
  mapfile -t formattable < <(git diff --cached --name-only | grep -vE '\.(md|txt)$' || true)

  # Skip gate entirely if no formattable files are staged (e.g., docs-only commit)
  if [ ${#formattable[@]} -eq 0 ]; then
    exit 0
  fi

  session_id="$(echo "$input" | jq -r '.session_id // ""')"
  flag="/tmp/.claude-format-gate-${session_id}"

  # Formatting gate -- first attempt: deny (Claude sees reason), second: allow
  if [ -f "$flag" ]; then
    rm -f "$flag"
    exit 0
  fi
  touch "$flag"

  file_list=$(printf '  %s\n' "${formattable[@]}")
  reason="$(printf 'Format these staged files per Code Formatting Standards before committing, then retry:\n%s\nDo NOT format .md or .txt files.\nIf formatting is unnecessary, bypass with: SKIP_FORMAT_CHECK=1 git commit ...' "$file_list")"
  jq -n --arg reason "$reason" \
    '{"hookSpecificOutput": {"hookEventName": "PreToolUse", "permissionDecision": "deny", "permissionDecisionReason": $reason}}'
  exit 0
fi

# --- gh pr create ---
if [[ "$command" == gh\ pr\ create* ]]; then
  jq -n --arg reason "Before creating this PR, ensure that tests pass and consider whether a code review is warranted." \
    '{"decision": "ask", "reason": $reason}'
  exit 0
fi
```

A writing style guard intercepts all Edit, Write, and Bash tool calls and denies any that contain em-dashes or en-dashes. It scans for both the literal Unicode characters and escape sequences like `\u2014` or `\u2013` that would produce them. This enforces a writing convention across all output: prose, comments, commits, and PR descriptions. The hook supports a single-use bypass mechanism for legitimate uses: if a session-scoped marker file exists, the hook consumes it and allows the operation through.

```python {collapsed="true" filename="~/.claude/hooks/writing-style-guard.py"}
#!/usr/bin/env python3
"""writing-style-guard.py -- PreToolUse hook (matcher: Edit|Write|Bash)

Enforces the CLAUDE.md writing style rule: no em-dashes or en-dashes.
Scans text that Claude is about to write and denies if dashes are found.
Also catches unicode escape sequences that would produce dashes (e.g.
\\u2014 in bash commands), since Claude sometimes uses those to sneak
dashes past the literal character check.

Single-use bypass: to allow one operation that legitimately needs dash
characters or escape sequences, create the marker file
/tmp/.claude-style-bypass-{session_id} before the tool call. The hook
consumes (deletes) the marker and allows the operation through. The
bypass is single-use; the next operation is checked normally.
"""

import json
import os
import re
import sys

EM_DASH = "\u2014"
EN_DASH = "\u2013"

BYPASS_DIR = "/tmp"
BYPASS_PREFIX = ".claude-style-bypass-"

# Patterns matching unicode escape sequences for em-dash (U+2014) and
# en-dash (U+2013) in various formats: \u2014, \U2014, \x{2013}, etc.
_ESCAPE_RE = re.compile(
    r"\\u2014|\\u2013|\\U2014|\\U2013|\\x\{201[34]\}", re.IGNORECASE
)

# Files that legitimately contain dash characters and escape sequences
# (this hook and its tests).
_EXEMPT_FILES = (
    "writing-style-guard.py",
    "test_writing_style_guard.py",
)


def extract_text(data: dict) -> str:
    """Extract the text to check from the hook input based on tool type."""
    tool_input = data.get("tool_input", {})
    tool_name = data.get("tool_name", "")

    if tool_name == "Edit":
        return tool_input.get("new_string", "")
    elif tool_name == "Write":
        return tool_input.get("content", "")
    elif tool_name == "Bash":
        return tool_input.get("command", "")
    return ""


def is_exempt(data: dict) -> bool:
    """Check if the operation targets an exempt file."""
    tool_input = data.get("tool_input", {})
    tool_name = data.get("tool_name", "")

    if tool_name in ("Edit", "Write"):
        file_path = tool_input.get("file_path", "")
        return any(file_path.endswith(f) for f in _EXEMPT_FILES)
    elif tool_name == "Bash":
        command = tool_input.get("command", "")
        return any(f in command for f in _EXEMPT_FILES)
    return False


def bypass_path(session_id: str) -> str:
    """Return the path to the session-scoped bypass marker file."""
    return os.path.join(BYPASS_DIR, f"{BYPASS_PREFIX}{session_id}")


def consume_bypass(session_id: str) -> bool:
    """If a bypass marker exists for this session, delete it and return True."""
    if not session_id:
        return False
    path = bypass_path(session_id)
    try:
        os.remove(path)
        return True
    except FileNotFoundError:
        return False


def check_dashes(text: str) -> str | None:
    """Return a reason string if em-dash or en-dash found, else None."""
    has_em = EM_DASH in text
    has_en = EN_DASH in text
    if has_em and has_en:
        return "text contains em-dashes and en-dashes"
    if has_em:
        return "text contains em-dashes"
    if has_en:
        return "text contains en-dashes"
    return None


def check_escapes(text: str) -> str | None:
    """Return a reason string if unicode escape sequences for dashes found."""
    match = _ESCAPE_RE.search(text)
    if match:
        return f"text contains unicode escape sequence for a dash ({match.group()})"
    return None


def main() -> None:
    try:
        data = json.load(sys.stdin)
    except (json.JSONDecodeError, EOFError):
        return

    text = extract_text(data)
    if not text:
        return

    exempt = is_exempt(data)

    # Always check for literal dash characters (never exempt)
    reason = check_dashes(text)

    # Check escape sequences only for non-exempt files
    if reason is None and not exempt:
        reason = check_escapes(text)

    if reason is None:
        return

    # Single-use bypass: if Claude created a marker file, consume it and allow
    session_id = data.get("session_id", "")
    if consume_bypass(session_id):
        return

    bypass_hint = ""
    if session_id:
        bp = bypass_path(session_id)
        bypass_hint = (
            f" To bypass once (for legitimate use of these characters), "
            f"run: touch {bp}"
        )

    json.dump(
        {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": (
                    f"Writing style violation: {reason}. "
                    "Use commas, periods, or parentheses instead."
                    f"{bypass_hint}"
                ),
            }
        },
        sys.stdout,
    )


if __name__ == "__main__":
    main()
```

A stop hook fires at session end and prompts for cleanup tasks. If the session is inside a git worktree with an auto-generated branch name (like `worktree-fancy-beaming-comet`), it prompts Claude to rename the branch to a convention-compliant name before ending. It also prompts for memory reflection if the session involved significant tool usage.

```bash {collapsed="true" filename="~/.claude/hooks/pre-stop-checks.sh"}
#!/usr/bin/env bash
# pre-stop-checks.sh -- Stop hook
# Memory reflection -- if the session had significant tool usage.
# On first stop (stop_hook_active=false): exit 2 with prompt, or exit 0 if none applies.
# On second stop (stop_hook_active=true): exit 0 to allow session to end.

set -euo pipefail

input="$(cat)"
stop_hook_active="$(echo "$input" | jq -r '.stop_hook_active // false')"

if [ "$stop_hook_active" = "true" ]; then
  exit 0
fi

prompts=()

# --- Deinit submodules in linked worktrees (enables single-force cleanup) ---
git_dir="$(git rev-parse --git-dir 2>/dev/null)" || true
common_dir="$(git rev-parse --git-common-dir 2>/dev/null)" || true
if [ -n "$git_dir" ] && [ -n "$common_dir" ]; then
  abs_git_dir="$(cd "$git_dir" && pwd)"
  abs_common_dir="$(cd "$common_dir" && pwd)"
  if [ "$abs_git_dir" != "$abs_common_dir" ]; then
    # In a linked worktree; deinit submodules so auto-cleanup can remove it
    if git submodule status 2>/dev/null | grep -qv '^-'; then
      git submodule deinit --all -f >/dev/null 2>&1 || true
    fi

    # --- Worktree branch rename (auto-generated -> convention) ---
    BRANCH=$(git branch --show-current 2>/dev/null || true)
    WORKTREE_NAME=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)")

    if [ -n "$BRANCH" ] && [[ "$BRANCH" == worktree-* ]]; then
      prompts+=("The current branch '$BRANCH' is an auto-generated worktree name. Decide one of:

1. RENAME: if this work will be used in a pull request, rename the branch: git branch -m pr/<user>/<description>

2. DEFER: if you don't have enough context yet or the work is ephemeral, do nothing. This prompt will appear again after the next user interaction.")
    fi
  fi
fi

# --- Parse transcript (needed by multiple concerns) ---
transcript_path="$(echo "$input" | jq -r '.transcript_path // ""')"

tool_count=0

if [ -n "$transcript_path" ] && [ -f "$transcript_path" ]; then
  # Count total tool_use blocks across all transcript lines
  tool_count=$(jq -c '[.message.content[]? | select(.type=="tool_use")] | length' "$transcript_path" 2>/dev/null | awk '{s+=$1} END {print s+0}')
fi

# --- Memory reflection (only if session had significant tool usage) ---
if [ "$tool_count" -gt 3 ]; then
  prompts+=("Reflect: anything worth saving to memory? Skip if nothing noteworthy.
- **Project auto memory dir**: save insights specific to THIS project (architecture, patterns, debugging tips)
- **~/.claude/MEMORIES.md**: ONLY for cross-project lessons useful in ANY repo (e.g., git workflows, user preferences). Ask user before adding. NEVER put project-specific insights here.")
fi

# --- Combine prompts ---
if [ ${#prompts[@]} -eq 0 ]; then
  exit 0
fi

reason=""
for i in "${!prompts[@]}"; do
  if [ "$i" -gt 0 ]; then
    reason="${reason}
---
"
  fi
  reason="${reason}${prompts[$i]}"
done

jq -n --arg reason "$reason" '{"decision": "block", "reason": $reason}'
exit 2
```

## Supporting Skills

Two additional skills handle the parts of git history management that the main three don't cover.

The fixup skill creates fixup commits and autosquashes them into their targets. It verifies the target commit by running `git log --oneline -- <file>` rather than guessing from descriptions, creates the fixup with `--fixup=<sha>`, then runs a non-interactive autosquash rebase using `GIT_SEQUENCE_EDITOR=true`. After squashing, it checks whether the target commit's message is now stale (because the fixup changed the nature of the commit) and suggests `/reword-commits` if so.

````markdown {collapsed="true" filename="~/.claude/skills/fixup/SKILL.md"}
---
name: fixup
description: "Create fixup commits and autosquash them into their target commits via interactive rebase. Use when the user asks to fixup, squash into, or amend a previous commit (not the most recent one)."
argument-hint: "[optional: target commit or file hints]"
---

Create fixup commits for staged/unstaged changes and autosquash them into the correct target commits.

## Steps

1. **Identify changes**: Run `git status` to see what needs to be fixed up. If there are no changes, inform the user and stop.

2. **Identify target commits**: For each changed file, run `git log --oneline -- <file>` to find the commit that introduced the change. Do NOT guess -- verify the target commit actually touched that file.

3. **Create fixup commits**: For each target commit, stage the relevant files and run:
   ```
   git commit --fixup=<target-sha>
   ```
   If changes map to multiple target commits, create separate fixup commits for each.

4. **Autosquash**: Rebase onto the merge base to squash fixups into their targets:
   ```
   GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash $(git merge-base <base-branch> HEAD)
   ```
   Where `<base-branch>` is detected using:
   - `git symbolic-ref refs/remotes/origin/HEAD` (strip prefix), or
   - `origin/main`, or `origin/master` as fallback

5. **Check for stale messages**: After autosquash, check whether the fixup changed the nature of the target commit enough to make its message inaccurate. For example, if the fixup removes a feature the message claims to add, or changes behavior the message describes. If any messages are now stale, suggest the user run `/reword-commits` as a follow-up.

6. **Verify**: Run `git log --oneline <base>..HEAD` to confirm the fixups were squashed and the history is clean.

## Important

- Always use `--fixup=` (not `--amend`), since amend only works on the most recent commit
- Always use `GIT_SEQUENCE_EDITOR=true` to accept the autosquash todo non-interactively
- Always rebase onto the merge base, not onto the base branch directly -- rebasing onto the branch can fail if it has parallel copies of the same commits
- If the working tree has unrelated changes, stash them first and pop after
- Do NOT force push -- leave that to the user unless they explicitly ask
````

The commit surgery skill handles more complex history restructuring: squashing groups of commits, reordering, dropping changes from specific commits, and fixing cross-commit regressions. It uses a set of bundled Python scripts that act as `GIT_SEQUENCE_EDITOR` implementations, each handling a different rebase operation (reorder and squash, custom squash messages, marking commits for editing). The skill operates in phases: analyze, fix code issues that will surface after squashing, rewrite history, fix regressions from reordering, and verify against a backup branch.

````markdown {collapsed="true" filename="~/.claude/skills/commit-surgery/SKILL.md"}
---
name: commit-surgery
description: Squash, reorder, and clean up commit history on a branch. Handles interleaved temporary/DONTMERGE commits, interface mismatches from split commits, and cross-commit regressions. Use when the user needs to restructure commit history before merge.
argument-hint: "[optional: description of desired structure, e.g. 'squash real commits, keep DONTMERGE on top']"
---

Restructure commit history on the current branch: squash groups of commits, reorder, drop changes from specific commits, and verify the result.

## Runbook

### Phase 1: Analyze

1. **Map the branch.** List all commits with files changed:
   ```
   git log --oneline --stat <base>..HEAD
   ```

2. **Classify commits.** Identify:
   - Real commits (to be squashed or kept)
   - Temporary commits (DONTMERGE, WIP, fixup, etc.)
   - Empty commits (trigger CI, etc.)

3. **Detect cross-commit issues.** Look for:
   - Interface methods added in one commit but test fakes not updated
   - Temporary commits that modify files also touched by real commits (these cause regression when reordered)
   - Commits that introduce changes later reverted by subsequent commits

4. **Back up the branch:**
   ```
   git branch backup/<branch-name>
   ```

### Phase 2: Fix Code Issues

Before rewriting history, fix any code issues that will surface after squashing (e.g., missing interface methods on test fakes). Commit fixes as separate commits; they'll be squashed in Phase 3.

Verify fixes compile:
```
go vet ./path/to/package/...
```

### Phase 3: Rewrite History

Use `GIT_SEQUENCE_EDITOR` with the bundled scripts to automate the interactive rebase todo.

#### Reorder and Squash

Use `reorder-todo.py` to squash real commits and move temporary commits to the end:

```
GIT_SEQUENCE_EDITOR="~/.claude/skills/commit-surgery/scripts/reorder-todo.py" \
git rebase -i <base>
```

To keep specific commits as individual picks (instead of squashing them):

```
GIT_SEQUENCE_EDITOR="~/.claude/skills/commit-surgery/scripts/reorder-todo.py --keep <sha1> --keep <sha2>" \
git rebase -i <base>
```

#### Custom Squash Message

Use `squash-msg.py` to replace the combined squash message:

```
GIT_SEQUENCE_EDITOR="~/.claude/skills/commit-surgery/scripts/reorder-todo.py" \
GIT_EDITOR="~/.claude/skills/commit-surgery/scripts/squash-msg.py '<full commit message>'" \
git rebase -i <base>
```

### Phase 4: Fix Reordered Temporary Commits

When temporary commits are reordered to the top, they may contain changes to files that are now in a different state (because real commits they previously preceded now come before them). This causes regressions.

To remove specific file changes from a commit:

1. Use `edit-commit.py` to mark commits for editing:

   ```
   GIT_SEQUENCE_EDITOR="~/.claude/skills/commit-surgery/scripts/edit-commit.py <sha>" \
   git rebase -i <base>
   ```

2. When the rebase stops at that commit, reset the problematic file to the parent state and amend:
   ```
   git checkout HEAD~1 -- path/to/file.go
   git add path/to/file.go
   git commit --amend -s --no-edit
   git rebase --continue
   ```

### Phase 5: Verify

1. **Check history:**
   ```
   git log --oneline --stat <base>..HEAD
   ```

2. **Diff against backup** for only the files your branch modifies (not upstream drift):
   ```
   git diff backup/<branch-name>..HEAD -- path/to/your/files/
   ```
   The only differences should be intentional fixes from Phase 2.

3. **Compile check:**
   ```
   go vet ./path/to/package/...
   ```

4. **Check for stale messages:** Review whether any surviving commits now have messages that no longer match their diffs (e.g., a commit that originally added feature X, but the surgery moved that code elsewhere). If any messages are stale, suggest the user run `/reword-commits` as a follow-up.

5. **Clean up backup:**
   ```
   git branch -D backup/<branch-name>
   ```

## Common Pitfalls

- **Don't diff entire trees when base commits differ.** If the rebase moved to a newer base, `git diff backup..HEAD` will include all upstream changes. Always scope the diff to your branch's files.

- **Temporary commits that touch real files.** When reordering, any file touched by both temporary and real commits will cause conflicts or regressions. Strip the real-file changes from temporary commits (Phase 4).

- **Interface changes split across commits.** When squashing commits that incrementally build an interface (add method in commit A, add fake in commit B), the squashed result is fine, but test fakes may be missing methods if one of the commits was incomplete. Fix before squashing.

- **Empty commits disappear.** Commits with no diff (e.g., "trigger CI") are silently dropped during squash. This is expected.

- **Force push carefully.** Always use `--force-with-lease` to avoid overwriting others' work.

## Do NOT Force Push

Unless the user explicitly asks, do not force push after surgery. Show the result and let them decide.
````

The bundled `GIT_SEQUENCE_EDITOR` scripts handle the rebase operations.

```python {collapsed="true" filename="~/.claude/skills/commit-surgery/scripts/reorder-todo.py"}
#!/usr/bin/env python3
"""GIT_SEQUENCE_EDITOR script: reorder and squash commits in a rebase todo.

Usage: reorder-todo.py <todo-file> [--keep <sha>...]

Commits matching --keep SHAs stay as individual picks and are moved to the
end of the todo (after squashed real commits). All other real commits are
squashed together (first = pick, rest = squash). Commits containing
DONTMERGE or WIP in the subject are classified as temporary and placed last.

Without --keep, all real commits are squashed and temporary commits follow.
"""
import sys


def main():
    args = sys.argv[1:]
    if not args:
        print("usage: reorder-todo.py <todo-file> [--keep <sha>...]", file=sys.stderr)
        sys.exit(1)

    todo_file = args[0]
    keep_shas = set()
    i = 1
    while i < len(args):
        if args[i] == "--keep" and i + 1 < len(args):
            keep_shas.add(args[i + 1])
            i += 2
        else:
            i += 1

    with open(todo_file) as f:
        lines = f.readlines()

    picks = [l for l in lines if l.startswith(("pick ", "squash ", "fixup "))]
    rest = [l for l in lines if not l.startswith(("pick ", "squash ", "fixup "))]

    temporary = [l for l in picks if "DONTMERGE" in l or "WIP" in l]
    real = [l for l in picks if l not in temporary]

    # Separate kept commits from squashable ones
    kept = []
    squashable = []
    for line in real:
        sha = line.split()[1]
        if any(sha.startswith(k) for k in keep_shas):
            kept.append(line)
        else:
            squashable.append(line)

    # First squashable commit stays as pick, rest become squash
    result = []
    for i, line in enumerate(squashable):
        if i == 0:
            result.append(line)
        else:
            result.append(line.replace("pick ", "squash ", 1))

    # Kept commits stay as picks
    result.extend(kept)

    # Temporary commits go last as picks
    result.extend(temporary)
    result.extend(rest)

    with open(todo_file, "w") as f:
        f.writelines(result)


if __name__ == "__main__":
    main()
```

```python {collapsed="true" filename="~/.claude/skills/commit-surgery/scripts/squash-msg.py"}
#!/usr/bin/env python3
"""GIT_EDITOR script: replace the squash commit message.

Usage: squash-msg.py <msg-file> <new-message>

Writes <new-message> to <msg-file>, replacing whatever git prepared.
The message should include the full commit body (subject, blank line,
description, sign-off).
"""
import sys


def main():
    if len(sys.argv) < 3:
        print("usage: squash-msg.py <msg-file> <new-message>", file=sys.stderr)
        sys.exit(1)

    msg_file = sys.argv[1]
    new_message = sys.argv[2]

    with open(msg_file, "w") as f:
        f.write(new_message)
        if not new_message.endswith("\n"):
            f.write("\n")


if __name__ == "__main__":
    main()
```

```python {collapsed="true" filename="~/.claude/skills/commit-surgery/scripts/edit-commit.py"}
#!/usr/bin/env python3
"""GIT_SEQUENCE_EDITOR script: mark specific commits as 'edit' in a rebase todo.

Usage: edit-commit.py <todo-file> <sha> [<sha>...]

Any pick line whose SHA prefix-matches one of the provided SHAs is changed
from 'pick' to 'edit'. All other lines are left unchanged.
"""
import sys


def main():
    if len(sys.argv) < 3:
        print("usage: edit-commit.py <todo-file> <sha> [<sha>...]", file=sys.stderr)
        sys.exit(1)

    todo_file = sys.argv[1]
    target_shas = sys.argv[2:]

    with open(todo_file) as f:
        lines = f.readlines()

    result = []
    for line in lines:
        if line.startswith("pick "):
            sha = line.split()[1]
            if any(sha.startswith(t) for t in target_shas):
                result.append(line.replace("pick ", "edit ", 1))
            else:
                result.append(line)
        else:
            result.append(line)

    with open(todo_file, "w") as f:
        f.writelines(result)


if __name__ == "__main__":
    main()
```

## What This Gets You

The system produces commits, reviews, and PRs that are consistent with each other and with the project's documented conventions. PR descriptions are fact-checked against the actual code before anyone sees them. Reviews focus on genuine issues introduced by the PR, not pre-existing code smells. Commit messages match what the code actually does, not what it was originally intended to do.

The system also builds institutional knowledge. A global memory file accumulates cross-project lessons (git workflow edge cases, tool-specific gotchas, debugging insights) that persist across sessions.

```markdown {collapsed="true" filename="~/.claude/MEMORIES.md"}
# Global Memories

Cross-project lessons that apply to ANY repository. Do NOT put project-specific insights here -- those go in the project's auto memory directory. When in doubt, ask the user before adding entries.

## Git

### Gitignore Anchoring
When adding directory patterns to `.gitignore`, **always anchor root-level entries with `/`** (e.g., `/debug/` not `debug/`). Without anchoring, patterns match anywhere in the tree and can silently exclude nested content.

### Fixup Targeting
Before creating a fixup commit, use `git log --oneline -- <file>` to confirm which commit actually introduced the change. Don't rely on assumptions from commit descriptions -- a fixup targeting the wrong commit silently becomes a no-op during autosquash.

### Rebase onto Merge Base
When `--autosquash` onto `main` fails because main has parallel copies of the same commits (same content, different SHAs), rebase onto the merge base instead: `git rebase -i --autosquash $(git merge-base main HEAD)`. Use `GIT_SEQUENCE_EDITOR=true` to accept the autosquash todo as-is. After rebase, verify with `git diff <backup-branch>`.

### Local File Protocol in Submodule Tests
Git >= 2.38.1 blocks `file://` transport by default. Tests that use `git submodule add` with local paths need `-c protocol.file.allow=always` on git commands, or `GIT_CONFIG_COUNT`/`GIT_CONFIG_KEY_0`/`GIT_CONFIG_VALUE_0` env vars for subprocess calls.

### Renaming Worktrees with Submodules
`git worktree move` fails with "working trees containing submodules cannot be moved or removed". Workaround: manually `mv` both the worktree directory and `.git/worktrees/<name>/`, then update the `.git` file in the worktree (gitdir pointer) and `.git/worktrees/<new-name>/gitdir` (reverse pointer) to reflect the new paths.

### Cross-Commit Regressions During Autosquash
When fixup commits move code FROM a later commit INTO an earlier one (e.g., moving a function from commit 4 into commit 2's fixup), autosquash causes regressions: commit 4 still has its original diff which re-introduces the old version, overwriting the fixup's changes. Always `git branch backup` before autosquash, then `git diff backup` after to catch regressions. Fix by amending the affected commit to match the backup's content.

## GitHub

### Unresolved PR Comments via GraphQL
The REST API (`/pulls/{n}/comments`) doesn't expose resolution status. Use `gh api graphql` with `reviewThreads.isResolved` to get only unresolved PR review comments.

### gh api JSON Arrays
`gh api -f 'key=[...]'` passes the value as a string, not a JSON array, causing 422 errors. For payloads with arrays, write JSON to a temp file and use `gh api --input /tmp/payload.json` instead.

## Go

### `os.ReadDir` Symlinks
`DirEntry.IsDir()` returns `false` for symlinks -- it reports the link itself, not the target. To include symlinked directories in listings, check `e.Type()&os.ModeSymlink != 0` and follow with `os.Stat()` to resolve the target.

## Chezmoi

### Diff Direction
In `chezmoi diff`: `---` = destination (on disk), `+++` = target state (what source wants). `apply` writes `+` to disk; `re-add` writes `-` to source. "Target state" means desired (from source), not "destination" (actual).

### Workflow Decision Tree
When changing chezmoi-managed files, pick the right operation:
- **Already managed (especially templates)**: edit the **source** directly (`chezmoi source-path <file>`), then `chezmoi apply --force <file>`. Do NOT `chezmoi add` on already-managed files.
- **Brand-new file needing portability**: create on disk, then `chezmoi add <file>`.
- **New file that doesn't need portability** (e.g., local scripts referenced by managed config): just create it at the destination -- no chezmoi management needed.

### Empty Template Output
Chezmoi won't create files when a `.tmpl` renders to zero bytes. Use the `empty_` filename prefix (e.g., `empty_foo.md.tmpl`) to force file creation even with empty content. Common with conditional `@` include files that render to nothing on some machines.

## Claude Code

### Project Settings Location
Project-level `settings.json` and `settings.local.json` go in `<project-root>/.claude/`, NOT in `~/.claude/projects/<encoded-path>/`. The `~/.claude/projects/` directory is for Claude Code's internal data (transcripts, auto-memory) only. Never write settings, CLAUDE.md, or other config files there.

### Subagents in Worktrees
When spawning subagents (via the Task tool) from a git worktree, explicitly instruct them to work in the worktree's directory. Subagents inherit the parent's CWD, but if they run git commands or resolve paths, they may drift to the main repo. Always include the worktree path in the agent prompt unless the agent is being given its own dedicated worktree.

### LSP Plugins Mostly Disabled
Most LSP plugins (gopls, rust-analyzer, clangd, typescript, lua, jdtls) are disabled in settings.json. They launch eagerly for every session regardless of project language, consuming ~10GB total (rust-analyzer alone uses 6.5GB). Pyright is kept enabled (useful). Re-enable others selectively with `/plugins` if needed.

### Missing `@` Includes Cause Errors
`@path` imports in CLAUDE.md error when the file doesn't exist (not silently skipped). No optional import syntax yet (open request: #22799). To conditionally include files across machines, use two-stage indirection: CLAUDE.md imports an always-present intermediate file that is itself a chezmoi template, rendering to `@target.md` or empty depending on machine type.

## Shell / Bash Tool

### Do Not Escape `!` in Shell Commands
Never write `\!` in commands passed to the Bash tool. The `!` character does not need escaping inside single-quoted strings, and escaping it breaks Python operators like `!=` (producing `\!=` which is a syntax error). Write `!=` as-is. If this keeps happening, use the heredoc workaround: write the Python script to a temp file (`cat > /tmp/script.py << 'EOF' ... EOF`) then pipe into it. The heredoc avoids all shell escaping issues.
```

The cost is complexity. Several skill files, a commit policy document, and about a dozen hook scripts, all of which need maintenance as Claude Code evolves. When a new version changes how tools are invoked or how hooks receive input, some of this breaks. The hooks are all tested (the test suite covers the token parsing, tier classification, and edge cases), but the skills are tested by use, not by automated test harnesses.

The benefit is trust. I review every PR and every commit, but I'm reviewing output that has already been through multiple verification stages. The verification sub-agent catches factual errors in PR descriptions that I might skim past. The confidence scoring filters out false positives that would erode trust in the review skill over time. The commit message consistency check catches the kind of drift that accumulates silently across a multi-commit branch.

The broader pattern applies independent of the specific tools. AI assistants that interact with shared systems (GitHub, CI, production infrastructure) need the same kind of process discipline that human contributors need. The mechanisms differ (hooks instead of code review norms, sub-agents instead of pair programming), but the goal is the same: make sure what gets shipped is accurate, consistent, and reviewed before it becomes someone else's problem.
