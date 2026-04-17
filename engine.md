# Review Horde — Orchestration Engine

This file defines the full pipeline for all three modes: Full Review, Re-Review, and Mirror.

---

## Full Review Pipeline

### Step 1 — Gather (run in parallel)

Fetch these simultaneously:

**Batch A — PR data (already in context from SKILL.md):**
The PR metadata and diff were pre-loaded by SKILL.md inline commands. No re-fetch needed.

**Batch B — Rules and docs:**
- Always read: `~/.claude/skills/review-pr/RULES.md`
- Scan the diff file paths and read any matching docs from the dispatch table in Step 2

**Batch C — Blast-radius scan:**
Parse the diff for modified function/method signatures: lines starting with `-` that contain `function`, `const`, `=>`, or TypeScript method patterns (e.g., `methodName(`). For each modified identifier:
```bash
grep -r "identifierName" src/ --include="*.ts" --include="*.tsx" -l | head -5
```
Cap at 5 callers per identifier. Collect results as a "Downstream Dependencies" block — this gets injected into every zombie agent's prompt.

---

### Step 2 — Context load

#### Doc dispatch table

Scan the diff file paths. Read exactly the docs that match:

| If the diff touches... | Load... |
|---|---|
| `src/microtexts/`, Contentful keys, `useSectionMicrotexts` | `docs/add-contentful-section.md` |
| deep links, `useDeepLink`, `useListingValidation`, `linking.ts` | `docs/deep-links.md` |
| SERP, impressions, `trackImpressionEvent` | `docs/impressions-tracking.md` |
| responsive layout, `useResponsive` | `docs/responsive-design.md` |
| search, SERP filters, `PropertiesList` | `docs/search.md` |
| Storybook stories, `.stories.tsx` | `docs/storybook-guidelines.md` |
| Amplitude events, `amplitudeStore`, `AmplitudeExperimentFlags` | `add-amplitude-feature-flag` or `add-amplitude-experiment` skill |
| `FFAlert`, confirmation dialogs | `alerts` skill |
| deep link setup, `FFPathConfigMap` | `create-deep-link` skill |
| feature folder structure, migrations | `modular-repository-structure` and `migrate-element-to-modular` skills |
| `.agents/rules/` | apply rules convention checks (see below) |
| new screen files added to a `screens/` folder | verify flat structure per RULES.md; check `screens/index.ts` barrel exists |
| `app.config.ts` with `RUNTIME_VERSION` changed | flag OTA implication: RUNTIME_VERSION bumps block OTA until next store release — if the PR also introduces feature-flagged behavior, flag as should-split |

**Rules convention checks** — when `.agents/rules/` files are added or modified:
- Rule files must use `.md` extension (not `.mdc`) — `.mdc` belongs only in `.cursor/rules/`
- Every rule file needs a matching symlink in `.cursor/rules/<rule-name>.mdc` pointing to `../../.agents/rules/<path>.md`
- Verify: `ls -la .cursor/rules/<rule-name>.mdc` — missing symlink = must-fix (rule invisible to Cursor)

#### React Native Reusables conventions (always apply — no file read needed)

- `src/~` is the team's owned copy of RN Reusables. `@/~/...` imports are intentional — never flag them.
- Icons live in `src/~/lib/icons/`. Flag any icon imported from lucide or another library instead.
- Styling: NativeWind `className` + `cn()` from `@/~/lib/utils`. Flag `StyleSheet.create()` on new components.
- Variants use `cva`. Flag custom union types + conditional styles.
- `React.forwardRef` is deprecated — flag it. Prefer React 19 ref-as-prop.
- `asChild` + `Slot` is the composition pattern for flexible triggers.
- Portal-based overlays (Dialog, AlertDialog, Popover, Tooltip): dismiss + present in rapid sequence can cause visual issues due to async unmount/remount.
- All components: `React.FC<Props>` typing required — not bare arrow functions.
- `createContext` must use `null` initial value: `createContext<Type | null>(null)`. Consuming hook must be named `use<Feature>Context`, check for `null`, and throw if outside provider.

#### React Native language hints

If the diff contains `.tsx` or `.ts` files, also load: `~/.claude/skills/review-pr/languages/react-native.md`

#### Preflight confirmation

Output a single line before dispatching:
> **Horde assembling:** [list what was loaded] — [tier] dispatch — [N] files in the diff

---

### Step 3 — Triage (Haiku)

Run a fast Haiku classification pass on the diff to determine the dispatch tier. Output JSON:

```json
{
  "tier": "fresh | rising | full-horde",
  "rationale": "one sentence",
  "modified_identifiers": ["functionName", "ComponentName"]
}
```

**Tier definitions:**
- `fresh` — renames, reorders, style/comment-only changes. No logic added or removed.
- `rising` — new code, new files, purely additive changes. No existing logic mutated.
- `full-horde` — existing logic modified, code deleted, conditional branches changed, API contracts touched.

When in doubt, promote to the next tier. The cost of under-dispatching is a missed bug.

---

### Step 4 — Dispatch (parallel zombie agents)

See `~/.claude/skills/review-pr/persona.md` for the full zombie roster and tier dispatch table.

Dispatch the appropriate zombies as parallel background agents (`run_in_background: true`). Each agent receives:
- The full diff
- The RULES.md content
- Any loaded doc content relevant to their focus
- The Downstream Dependencies block from Step 1
- The RN Reusables conventions summary

Each zombie is defined in `~/.claude/skills/review-pr/agents/<name>.md`. Pass the agent file as context.

Agent output format (each zombie must return this structure):

```json
{
  "zombie": "Shambler",
  "findings": [
    {
      "file": "src/path/to/file.tsx",
      "line": 42,
      "severity": "must-fix | should-fix | consider",
      "body": "One clear finding under 30 words.",
      "claim_verified": true
    }
  ]
}
```

`claim_verified: true` means the agent confirmed the finding against the actual diff before emitting it. Agents must drop any finding they cannot verify.

---

### Step 5 — Merge (The Horde Mother)

Once all zombie agents complete, run The Horde Mother merge pass. The Horde Mother is an Opus agent with one job: produce a tight, deduplicated, verified finding list.

**Horde Mother instructions:**
1. Read all zombie JSON outputs
2. For each finding, re-verify the `file:line` claim against the diff. Drop any finding where the claim doesn't hold.
3. Deduplicate: if two zombies flagged the same issue at the same location, keep the sharper one.
4. Enforce brevity: each `body` must be under 30 words. If a finding exceeds this, rewrite it. The target is 20 words.
5. Output the merged finding list as structured JSON matching the same schema above, plus a `zombie` field showing which agent originated it.

**Brevity example:**

BAD: `This component does not use the React.FC<Props> typing convention that is documented in the RN Reusables guidelines, which means it won't benefit from the prop type inference...`

GOOD: `Missing React.FC<Props> typing — bare arrow function won't surface prop errors at the callsite.`

---

### Step 6 — Fingerprint (Haiku)

After merging, run a Haiku fingerprinting pass. For each finding, generate 4–8 kebab-case tokens that describe the issue in a way that's stable across phrasings:

```json
{
  "finding_index": 0,
  "tokens": ["missing-react-fc-typing", "bare-arrow-function", "props-inference"]
}
```

These tokens are used in Re-Review Mode to correlate findings from the original review against the updated diff — detecting what was addressed vs. still outstanding.

Save the fingerprinted findings to `/tmp/horde-findings-<repo>-<PR>.json`.

---

### Step 7 — Confirm + Post

**Present findings to the user.** Show the merged finding list grouped by severity. For each finding:

```
[N] path/to/file.tsx — line <line>  [<zombie>]
<body>
```

**Inline comment voice rules** (enforced by The Horde Mother, reinforced here):
- Short and direct. One point per comment.
- Plain language — write for a cross-border team. Short sentences. No jargon unless necessary.
- Do not re-state what the code does. Point at the issue.
- Only post grounded findings — verified against the diff.

Ask: "Which of these would you like to include?"

**Build the diff line map.** Before posting, parse all `@@` hunk headers from the diff to build a valid line map. For each approved finding, verify `file:line` against this map. If a line is invalid, snap to the nearest valid line within ±5. If no valid line exists within ±5, move the finding to the review body.

**Generate a Testing Guide** before asking whether the user has tested.

Fetch branch name:
```bash
gh pr view <PR_NUM> --repo <repo> --json headRefName -q .headRefName
```

Present:

---

**Testing PR #\<PR_NUM\>**

**Branch:** `<branch-name>` — check it out with: `gh pr checkout <PR_NUM>`

**Test as:** [derive from feature area:
- Calendar → Landlord only
- Messages / inbox → Both
- SERP / search / listings → Tenant
- Auth / onboarding → Both
- Profile / settings → Both
- Tenant leads / applications → Both
- Dashboard → Landlord
- If unclear → Both]

**Where to test:** [Device when the diff touches push notifications, camera, biometrics, deep links triggered from external browser, or native modules. Simulator for everything else.]

**Steps:**
- [3–6 specific steps derived from the diff. Name the screen, the button, the expected outcome.]

---

Ask: "Have you tested? Let me know when you're ready."

**Do not post without a yes.**

Ask: "How would you like to submit? My suggested verdict was [verdict].
1: Approve  2: Approve with suggestions  3: Request changes  4: Comment only"

Post using `gh api repos/<repo>/pulls/<PR_NUM>/reviews` with method POST:
- `"body"`: one or two sentence summary — not the full analysis
- `"event"`: `"APPROVE"`, `"REQUEST_CHANGES"`, or `"COMMENT"`
- `"comments"`: array of approved inline comments, each with `"path"`, `"line"`, `"side": "RIGHT"`, `"body"`
- If no inline comments, omit `"comments"` and use `gh pr review <PR_NUM> --repo <repo>` with the appropriate flag and `--body`

---

### Obsidian logging (after any full review post)

Update **both** files using Obsidian MCP tools:

**1. Today's daily note — `## PRs I'm Reviewing` section**

Use `obsidian_get_periodic_note(period: "daily")` to read today's note.

Format:
```
- [#<PR>](<url>) <ticket or —> — Waiting on author
```

For an approval: update status to `Approved ✓`.
For changes requested: update status to `Changes requested`.

Get file path via `obsidian_get_periodic_note(period: "daily", type: "metadata")`, then write back with `obsidian_put_content`.

**2. `scratch/pr-tracking.md`**

For a new review — add to **PRs I'm Reviewing — In Flight**:

| [#<PR>](<url>) | <repo short name> | <ticket or —> | <today YYYY-MM-DD> | Waiting on author | <key concern or —> |

For an approval — move to **PRs I'm Reviewing — Completed**:

| [#<PR>](<url>) | <repo> | <ticket> | <original reviewed date> | <today> | Merged / Approved |

---

## Re-Review Mode

Use when the user says "re-review", "did they fix the comments", "follow up on PR", "any updates on PR", or similar.

**Goal:** close the loop. See what was addressed, catch anything new, decide whether to approve. Do not re-post the full review.

### Steps

**1. Gather** (fetch in parallel):
```bash
gh pr diff <PR> --repo <repo>
gh pr view <PR> --repo <repo>
```

Use **GraphQL** (not REST) to check thread resolution status — REST incorrectly reports resolved threads as unresolved:
```bash
gh api graphql -f query='
{
  repository(owner: "<owner>", name: "<repo>") {
    pullRequest(number: <PR>) {
      reviewThreads(first: 50) {
        nodes {
          isResolved
          comments(first: 1) {
            nodes { body path line }
          }
        }
      }
    }
  }
}'
```

Load previous findings from `/tmp/horde-findings-<repo>-<PR>.json` if it exists — use fingerprint tokens to match against current state.

**2. Map each previous finding to a status:**
- **Resolved** — code changed, or author explained why it's a non-issue, or GraphQL `isResolved: true`
- **Dismissed / won't fix** — author declined with reasoning (treat as resolved)
- **Still outstanding** — no change, no reply, `isResolved: false`
- **Partially addressed** — acknowledged but fix is incomplete

**3. Scan for new issues** — lightweight only. Not a full horde dispatch. Look for:
- New code paths that contradict a previously approved pattern
- Obvious regressions
- Anything the original horde would have flagged on the new diff

**4. Extract learnings:**

Append to `~/.claude/projects/-Users-corey-davis-Workspace-harness--repos-ff-rn-app/memory/pr_review_learnings.md`:
```
## PR <number> — <date>
- <learning>: <why it matters>
```

Only save things that change future behavior. "Author fixed the bug" is not a learning.

**5. Present re-review summary:**

```
## Re-Review: PR <number>

### Resolved ✓
- [issue] — how addressed

### Still Outstanding
- [issue] — what's needed

### New Issues (if any)
- [anything new]

### Verdict
[one sentence]
```

**6. Approval flow** (if recommending Approve):

Generate a brief testing guide (branch name, Landlord/Tenant/Both, Simulator vs Device, 2–3 steps). Ask: "Have you tested? Let me know when ready."

If yes: post `gh api repos/<repo>/pulls/<PR>/reviews` with `"event": "APPROVE"` and body: `"Re-reviewed — all previous comments addressed. Approving."`

Do NOT re-post the original review.

Update Obsidian (same as Full Review logging above).

---

## Mirror Mode

Use when the PR author is the current user, or when "mirror" is in the arguments.

**Goal:** catch issues before teammates see them. No GitHub posting. No testing guide. No Obsidian logging.

Run Steps 1–6 exactly as in Full Review (same gather, same triage, same zombie dispatch, same merge). The only difference is the output and what happens after.

### Output format

Label: **Self-Review: PR #\<N\>**

Present merged findings grouped by severity:

**Must fix** — block the PR
- Compilation errors, broken behavior, missing tests for new logic, phantom exports

**Should fix** — convention violations a reviewer will call out
- RULES.md violations, barrel export gaps, TypeScript hygiene, uncovered branches

**Consider** — architecture and clarity notes
- Coupling concerns, split source-of-truth, implicit contracts needing a comment

Omit any category with no findings.

### After presenting findings

Ask: "Want me to fix any of these before we continue?"

If yes: make the fixes, then run `yarn tsc --noEmit` and `yarn lint:ci`. Report results.

Skip the testing guide, posting prompt, and Obsidian logging entirely.

---

## Explaining code

If the user asks to explain any code: explain clearly without assuming deep React Native or TypeScript knowledge. Use Swift/iOS analogues naturally — `useCallback` ≈ capturing a closure, `useEffect` ≈ `viewDidAppear`/`viewDidDisappear`, Zustand store ≈ a singleton service object.
