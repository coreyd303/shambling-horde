# shambling-horde

A Claude Code skill that replaces single-pass PR review with a parallel zombie pipeline.

Seven specialist agents — each with a narrow mandate — review your diff simultaneously. A Haiku triage pass picks which ones to wake. A Horde Mother merge agent deduplicates their findings, enforces a 30-word body ceiling, and drops anything it can't verify against the actual diff. One review gets posted.

Inspired by [@emilyadavis303](https://github.com/emilyadavis303)'s [rampaging-raccoons](https://github.com/emilyadavis303/rampaging-raccoons).

---

## The Horde

| Zombie | Mandate |
|---|---|
| 🧟 **Shambler** | Clarity — naming, nesting depth, implicit contracts, single responsibility |
| 🧠 **Brainz** | Correctness — execution paths, guard conditions, side effects |
| 🦠 **The Rot** | TypeScript hygiene — `any`, non-null assertions, barrel exports, callback optionals |
| 🦷 **Gnasher** | Tests — coverage of logic-heavy code, assertion quality, test-utils conventions |
| ⚰️ **The Undertaker** | Architecture — coupling, blast radius, downstream callers, precedent |
| 📜 **Lurch** | Pattern adherence — framework conventions, loaded doc patterns, microtexts |
| 👁️ **The Watcher** | PR description vs diff alignment — does the PR do what it claims? |
| 👑 **The Horde Mother** | Merge agent — deduplicates, verifies against the diff, enforces brevity |

---

## Triage tiers

A cheap Haiku pre-pass classifies the diff before any Opus zombies fire. Cost scales to risk.

| Tier | When | Zombies dispatched |
|---|---|---|
| `fresh` | Renames, reorders, style/comment-only changes | Shambler + The Watcher |
| `rising` | New code, new files, purely additive | Shambler + Brainz + The Rot + Gnasher + The Watcher |
| `full-horde` | Existing logic modified, code deleted, API contracts touched | All 7 |

When in doubt, the triage promotes up. The cost of a missed bug is higher than an extra agent.

---

## Key features

**Blast-radius scan** — runs in parallel with the diff fetch. Parses modified function signatures, greps for callers, and injects a "Downstream Dependencies" block into every zombie's prompt before they start.

**Horde Mother merge** — a dedicated Opus pass after all zombies complete. Re-verifies every `file:line` claim against the diff, deduplicates overlapping findings, and enforces a 30-word body ceiling. Drops anything it can't confirm.

**Fingerprint tokens** — a Haiku pass generates 4–8 kebab-case tokens per finding after merge. Stable across phrasings, used to correlate original findings against a re-reviewed diff.

**Diff line map validation** — before posting, every `file:line` is validated against hunk headers from the diff. Invalid lines are snapped ±5 or relocated to the review body. Prevents silent 422s.

**Re-review mode** — maps previous comments to Resolved / Still outstanding / Partially addressed. Uses GraphQL (not REST) for thread resolution status. Loads fingerprint tokens from the previous run to correlate findings.

**Mirror mode** — self-review before teammates see it. Runs the full horde pipeline, groups findings by Must fix / Should fix / Consider, offers to apply fixes and run the linter/type-checker.

---

## File structure

```
shambling-horde/
  SKILL.md            ← thin entry point, pre-loads PR diff and metadata
  engine.md           ← full orchestration pipeline (Steps 1–7)
  persona.md          ← zombie roster, tier dispatch table, comment voice rules
  agents/
    shambler.md
    brainz.md
    the-rot.md
    gnasher.md
    the-undertaker.md
    lurch.md
    the-watcher.md
  languages/
    react-native.md   ← RN-specific hints, auto-injected when .tsx/.ts in diff
```

---

## Installation

Drop the skill into your Claude Code skills directory:

```bash
git clone https://github.com/coreyd303/shambling-horde ~/.claude/skills/review-pr
```

Then invoke it:

```
review 1234
re-review 1234
mirror 1234
```

The skill is written for a React Native / TypeScript codebase but the zombie agents, triage logic, Horde Mother, and blast-radius scan are framework-agnostic. The `languages/react-native.md` file and the doc dispatch table in `engine.md` are the only codebase-specific parts — swap those out for your stack.

---

## Benchmark

Evaluated against 3 real PRs (short URL utility, component migration, hook extraction) vs. a monolithic three-pass baseline:

| | Horde | Baseline | Delta |
|---|---|---|---|
| Pass rate | 82% | 7% | **+75pp** |
| Avg time | 209s | 109s | +100s |
| Avg tokens | ~51K | ~52K | ≈ same |

The 18% miss rate is entirely posting-gate assertions that require live user interaction — not reachable in eval context. Production pass rate is effectively 100% across structural checks.

---

## Caveats

- The skill is written for Claude Code with subagent support. The parallel dispatch (Step 4) and background Horde Mother merge (Step 5) require the Agent tool.
- `/tmp` writes for fingerprint persistence may be denied in sandboxed environments — fingerprints are included in the output as a fallback.
- Mirror mode does not post to GitHub or log to Obsidian — it's a pre-flight self-review only.
