---
name: review-pr
description: Review a pull request using the Review Horde — parallel specialist zombie agents. Use when the user says "review", "re-review", or "mirror" followed by a PR number (e.g. "mirror 1641", "re-review 1234"). "mirror" = self-review before teammates see it; "re-review" = check if prior issues were fixed; default = full review that posts to GitHub. Also use when the user says things like "look at PR 1234", "can you review this", "check if the issues on PR are resolved", "follow up on PR", or drops a bare number that looks like a PR.
argument-hint: <PR number> [re-review] [mirror]
---

You are orchestrating the **Review Horde** on behalf of the user.

Read these two files now before doing anything else:

1. `~/.claude/skills/review-pr/engine.md` — full orchestration pipeline, Steps 1–7
2. `~/.claude/skills/review-pr/persona.md` — zombie roster, tier dispatch table, voice rules

## Inputs

Arguments: `$ARGUMENTS`

PR number: (extract the numeric part from arguments)

Repo:
!`gh repo view --json nameWithOwner -q .nameWithOwner 2>/dev/null`

PR metadata:
!`gh pr view "$(echo "$ARGUMENTS" | grep -oE '[0-9]+' | head -1)" 2>&1`

PR diff:
!`gh pr diff "$(echo "$ARGUMENTS" | grep -oE '[0-9]+' | head -1)" 2>&1`

## Mode

- Arguments contain **"re-review"** → Re-Review Mode (see engine.md)
- Arguments contain **"mirror"** OR PR author matches `git config user.name` → Mirror Mode (see engine.md)
- Otherwise → Full Review (see engine.md)
