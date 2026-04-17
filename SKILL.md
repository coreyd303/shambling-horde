---
name: review-pr
description: Review a pull request in depth using the Review Horde — parallel specialist zombie agents that cover clarity, correctness, TypeScript hygiene, tests, architecture, pattern adherence, and PR alignment — then post a single consolidated GitHub review. Use this skill whenever the user mentions a PR number, asks you to review code or a pull request, says things like "look at PR 1234", "can you review this", "what do you think of this PR", "review 1234", or wants feedback on a pull request. Also use it when the user says things like "re-review PR", "check if the issues on PR are resolved", "follow up on PR", "did they fix the comments on PR", "any updates on PR", or wants to see if a previously reviewed PR is ready to approve. Also use when the user says "mirror" or wants a self-review of their own code before pushing. Even if the user just drops a number and you suspect it's a PR, use this skill.
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
!`echo "$ARGUMENTS" | grep -oE '[0-9]+' | head -1 | xargs -I{} gh pr view {} 2>&1`

PR diff:
!`echo "$ARGUMENTS" | grep -oE '[0-9]+' | head -1 | xargs -I{} gh pr diff {} 2>&1`

## Mode

- Arguments contain **"re-review"** → Re-Review Mode (see engine.md)
- Arguments contain **"mirror"** OR PR author matches `git config user.name` → Mirror Mode (see engine.md)
- Otherwise → Full Review (see engine.md)
