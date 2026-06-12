---
name: pr-review
description: Decide whether a GitHub PR can be approved, then optionally write and submit the approval comment when the user explicitly authorizes it. Use when the user says "review this PR", "review PR #X", "can I approve this PR", "PR review", "give me a PR review", "look at this pull request", "approve it", "approve the PR", "submit an approval review", "LGTM comment", or pastes a GitHub PR URL. Produces a verdict (approve / hold / don't approve) with concrete evidence and a "what would I do differently" perspective. Approval mode is opt-in — the skill never approves or posts on its own, only after the user clearly instructs it to.
---

# PR Review

Use this skill when the user wants you to review a GitHub PR and tell them whether it is approvable. The user is the decision maker. You produce a clear verdict and the evidence behind it; the user reads, decides, and posts any comments themselves.

The default mode is review-only: produce a verdict and the evidence, but do not post comments or submit an approving review on your own. Approval is a separate, opt-in step — covered in the "Writing the Approval Comment" section near the end. Only act on it when the user explicitly tells you to approve.

## What "good review" means here

A review is useful when it changes the user's confidence about merging — either by surfacing something they would have missed, or by confirming the PR is safe with concrete evidence. Anything that doesn't change their decision is noise.

The output is for a human reader who already knows the codebase. Skip generic process language ("I fetched the PR", "I read the diff"), skip rubber-stamp praise, skip checklists pasted verbatim. Lead with the verdict, then justify it.

## Always Refresh from GitHub

Local state lies. The branch may have been force-pushed, comments may have been added, checks may have just failed. Before reviewing, refresh:

1. `git fetch origin` — make sure the local ref reflects the remote.
2. If the PR branch is checked out locally, `git pull --ff-only` (or compare local HEAD vs the PR head SHA from GitHub).
3. Pull the live PR state from GitHub. Prefer the `gh` CLI over the GitHub MCP whenever possible. The MCP returns large JSON blobs that burn context tokens quickly; `gh` lets you specify exact fields and pipe through `jq` or `head` to keep things lean. Reach for the MCP only when you need data `gh` doesn't expose cleanly — threaded review-comment metadata (`isResolved`, `isOutdated`), complex code/issue search, or GraphQL-level fields.

Useful commands:

```bash
# Lean PR overview — name only the fields you need
gh pr view <num> --json title,body,state,mergeable,mergeStateStatus,additions,deletions,changedFiles,commits,reviews,statusCheckRollup,headRefOid

# General timeline comments
gh pr view <num> --comments

# Full diff
gh pr diff <num>
```

For threaded **review** comments where you need resolution status (`isResolved`, `isOutdated`), the `gh` CLI does not expose those fields cleanly — use the GitHub MCP `get_review_comments` tool for those specifically. When using `gh --json`, always specify only the fields you need; for large outputs, pipe through `head` or `jq` to keep context usage in check.

If the user gives you a PR URL but the branch isn't checked out locally, that's fine — review against `gh pr diff` plus targeted `gh api` reads for files. Don't refuse for lack of a local checkout.

## Triage Existing Comments

For every comment thread on the PR, decide one of:

- **Addressed** — locate the change in the current diff and verify it actually resolves the concern. "Marked resolved" on GitHub does not count as evidence. The diff does.
- **Outstanding** — the comment is correct and not addressed. Surface it.
- **Incorrect** — the comment was wrong (misread the code, missed a constraint, etc.). Say so plainly with a one-line reason. Don't pretend a wrong comment needs to be addressed.
- **Stale / non-blocking** — was correct at the time but no longer relevant, or was a nit the author chose not to take.

When listing outstanding comments back to the user, group them by severity and use this table format so the user can navigate each thread by appending the Discussion ID to the PR's base URL:

| Column          | Description                                                                       |
| --------------- | --------------------------------------------------------------------------------- |
| `#`             | Sequential number                                                                 |
| `Discussion ID` | The `rXXXXXXXXXX` identifier from the comment URL (used to build a browser link)  |
| `File`          | Filename (without full path)                                                      |
| `Line`          | Line number                                                                       |
| `Issue`         | Brief description of the issue                                                    |

Example:

```markdown
## 🔴 Blocking — 2 comments

| #   | Discussion ID | File            | Line | Issue               |
| --- | ------------- | --------------- | ---- | ------------------- |
| 1   | `r1234567890` | `foo.js`        | 180  | Missing null check  |
| 2   | `r1234567891` | `bar.js`        | 220  | Incorrect parameter |

**Base URL:** `https://github.com/{owner}/{repo}/pull/{number}#discussion_`
```

The user clicks any thread by appending its Discussion ID to the Base URL.

## Test Discipline

Tests are the most common place where a PR looks fine but isn't. For every new or modified test, ask the four questions below. A passing test suite is not the same as a tested change.

### 1. Would it fail on the default branch?

Check whether `main` or `development` is the default for this repo. If the test passes on both branches, it isn't actually testing the new behavior — it's decoration. Run it against the default branch when feasible, or read the diff of the code-under-test and reason about whether the assertion depends on the change.

### 2. Is the same coverage already elsewhere?

If the same code path is covered at another layer with different inputs, an additional same-shape test adds maintenance cost without catching new bugs. Flag duplicates.

### 3. Is it tautological from over-mocking?

If the mock dictates the answer the assertion checks, the test only proves the mock was wired up. The classic shape: mock a function to return `X`, then assert the caller returns `X`. That proves nothing about the production code path.

### 4. Will the test still be valuable in six months?

A test that makes sense today, with full PR context, may be a maintenance burden once that context is gone. The author knows why every assertion exists right now; the maintainer reading it next year will not. Flag tests that look like they will age badly:

- **Coupled to implementation details** — assertions on internal call counts, private state, exact call ordering, or mock invocation specifics. These break on every refactor without the production behavior changing, and they push future maintainers to "fix" tests by mirroring whatever the code now does, which defeats the point.
- **Coupled to a rare edge case** — an input shape the codebase will not realistically receive. The cost of keeping the test green forever exceeds the bug it would catch.
- **Coupled to a temporary assumption** — a feature flag value, a transitional schema, a deprecated dependency, a migration intermediate state. The test will either silently lock that assumption in, or break loudly when it changes — usually at the worst time.
- **Redundant within the PR itself** — if the PR adds eight tests where two well-chosen ones would prove the change, the extra six are debt. Ask whether the same confidence could be reached with fewer, simpler, or more focused tests.

The lens to apply: imagine the PR's context is gone. A maintainer six months from now reads only the test name and body. Do they understand what behavior is being protected, and is that behavior worth protecting? If not, flag it.

## Review Phases

Use these as internal discipline, not as a checklist to recite back to the user.

1. **Context** — What is the PR trying to do? Read the description, linked issue, and commit messages. If you can't articulate the goal in one sentence, that's already a finding.
2. **High-level fit** — Does the approach match the goal? Is it consistent with how the rest of the codebase solves similar problems? Is there a simpler version?
3. **Line-by-line** — Logic correctness, edge cases, null/undefined paths, security (input validation, injection, secret leakage), performance (N+1 queries, blocking I/O in hot paths), error handling.
4. **Tests** — Apply the test discipline above.
5. **Comments** — Triage every existing thread.
6. **Verdict** — Approve / hold / don't approve, with the why.

## Severity Labels

When you list findings, label each one so the user can decide which deserve a comment. Adopt these consistently — they map cleanly to how reviewers think.

- 🔴 **blocking** — must be addressed before merge
- 🟡 **important** — should be addressed; worth a comment but room for discussion
- 🟢 **nit** — optional preference; mention only if asked
- 💡 **suggestion** — alternative approach the author may prefer
- ❓ **question** — clarification needed before approving

Don't label nits as blocking. Don't bury blockers in a "suggestions" list. The label is the user's signal for what to actually post.

## Output Shape

Return the review in this order. Skip sections that are empty rather than padding them.

### 1. Verdict

One line. One of:

- **Approve** — safe to approve as-is
- **Approve with non-blocking notes** — safe to approve; optional comments listed
- **Hold** — needs clarification or the author's response before deciding
- **Don't approve yet** — has 🔴 blockers; list them

### 2. Why

2–4 bullets of concrete evidence. Specific files, behaviors, code paths, test results, or CI status. No generic praise. If you say "the change is correct", say which change in which file and what makes it correct.

### 3. Outstanding comments (only if any)

Use the comment table format from the "Triage Existing Comments" section above. Group by severity if more than a few.

### 4. Critical issues to surface (only if any)

For each 🔴 blocker, write the exact text the user can paste into GitHub. Match the user's PR comment style: terse, concrete, no filler. The user posts these — you do not.

### 5. What I'd do differently

A short, concrete alternative if there is one worth raising — even on an approve. Skip if the current approach is fine. Examples worth raising: a simpler structure, a different boundary, a name that would age better, a missing test that's cheap to add. Don't list rewrites for the sake of having an opinion.

### 6. Approval (only if the user explicitly authorizes it)

If the verdict is **Approve** or **Approve with non-blocking notes** and the user explicitly tells you to approve, follow the "Writing the Approval Comment" section below. Otherwise stop after the verdict — the user posts any comments themselves.

## Writing the Approval Comment

Only do any of this once the user has explicitly authorized the approval after seeing the verdict. The default mode is still review-only.

### Pre-flight check (private to you)

Before drafting the body, internally confirm:

- The implementation is consistent with the PR goal.
- No 🔴 blockers remain.
- Previous review concerns are addressed, outdated, incorrect, or non-blocking.
- Relevant tests and checks were considered; any failing ones are known and unrelated.

Do not paste this list into the approval body — it's only there to decide whether approval is appropriate.

### Body style

Write for the PR author and human reviewers, not as a report that an AI followed review instructions.

- Lead with the engineering conclusion: what is correct, why the PR is safe to approve, or what caveat remains non-blocking.
- Stay anchored to *this* PR. Every point must make sense against the branch the PR merges into and speak to what the PR set out to do — judge it against its title, description, and target branch, not against changes you imagine it could have made. If a point doesn't connect to the author's stated goal, it doesn't belong in the body.
- Include only evidence that changes reviewer confidence — behavior verified, production/log data, affected flows, important files, test results, CI status, known pre-existing failures.
- At least one sentence must be specific to this PR. Avoid generic approval phrases.
- Keep non-blocking concerns clearly labeled as optional. Don't approve with language that sounds like unresolved required work.
- Compact, concrete language over pleasantries. Avoid "thanks", "nice work", "happy to", and bare "looks good".
- If checks are partially blocked by unrelated existing failures, name the failing check or file directly.

### Report what you verified, not what the author decided

The author spent days in this code; you spent minutes. They chose the boundaries and made the trade-offs deliberately, so a bullet that *endorses their decision* — "catching it here is the right boundary", "the split maps cleanly to the two failure modes", "suppressing under `?draft=true` is intended" — hands them nothing they don't already have. It reads as grading their homework. And "is intended" or "is the right call" is you *inferring* intent you can't see: in review you know what the author wrote, not what they were thinking.

What earns a place in the body is work the author *couldn't* have done for themselves:

- **What you ran, and the result** — "17/17 tests pass", "ran the 3 changed test files locally", "verified the affected page on the integration env".
- **What you cross-checked against a source of truth outside the diff** — "verified against installed `dep@7.26.3`", "the `onError` signature in `lib@5.100.14` takes a 4th arg, so `meta` resolves", "cross-checked against the `handleServerError` helper", "matches upstream PR #1674".
- **A concrete, actionable finding** — a bug, a nit, a follow-up.

The test for every bullet: **could the author have written this without me?** If yes — it's their decision, their rationale, or a restatement of their diff — cut it. If it took going outside the diff to know it (running it, checking a dependency version, reading adjacent code, hitting the live app), keep it.

Before — every bullet validates a decision the author already made (all four are cuttable):

```markdown
Approved.

- Catching the config-load errors in the `config` property is the right boundary — exactly where these escaped before.
- Splitting the two error types maps cleanly to the two real failure modes.
- Suppressing the notification under `?draft=true` is intended: an admin previewing their own draft shouldn't be paged.
- Regression tests exercise each distinct branch.
```

After — each bullet reports something checked, not something endorsed:

```markdown
Approved.

- Ran the 5 new regression tests locally; all pass, each covering a distinct branch (unpublished, invalid payload, lazy-settings failure, draft-skip, registration route).
- Confirmed the escaped errors reached the error tracker unhandled before this — the new catch sits on the only path that hits them (`setup()`, before `dispatch`).
- Cross-checked the draft-skip: `?draft=true` is the sole caller that suppresses, so non-draft failures still notify as before.
```

### What not to put in the body

The PR author reads the approval cold — they don't see the chat history that led to the verdict. A bullet that only makes sense in that hidden context reads as filler, or as preemptive defense of a concern nobody raised.

Pass each bullet through this test:

> If someone reads only the PR (no chat history with you), does this sentence give them new information that justifies the approval?

If no, cut it. Specifically:

- **Resolution status of existing threads.** "Codex P2 is resolved", "the author addressed @X's feedback" — the thread itself shows resolution. Mention prior comments only when their resolution materially changes the approval (a security blocker was resolved, a correctness bug was fixed in a later commit).
- **Private review context.** Concerns *you* considered but that were never raised on the PR. If you investigated whether `<div>` inside `<label>` was an issue and decided it wasn't, don't pre-empt that on the PR — readers will wonder why you brought it up.
- **Re-statements of the diff.** "The change moves X out of Y", "the new file adds Z" — the diff already shows this. The body should describe what was *verified*, not what was *changed*.
- **Endorsements of the author's decisions.** "Catching it here is the right boundary", "the split maps cleanly", "X is intended" — the author made these calls deliberately and knows them better than you; restating their design as correct adds nothing, and "is intended" guesses at intent the diff doesn't show. Report what you *verified*, not what they *decided* — see "Report what you verified, not what the author decided".
- **Pure-process evidence.** "I read the comments", "I checked the tests", "I fetched GitHub", "I reviewed the latest head" — assume the reader trusts the review. Only mention a verification step when it surfaces something the reader can't see (e.g., "verified against prod data", "ran the affected tests locally").
- **AI mechanics.** "I am an AI", "the user asked me to approve", "I followed the prompt". Keep that out of the public comment entirely.

When in doubt: did anyone on the PR raise this, or is it leaking from your private investigation? If the latter, it doesn't belong in the body.

### Body vs. line comments

The approval body justifies the approval. It is **not** a place to enumerate every non-blocking observation.

If a bullet starts with "minor:", "nit:", "as a follow-up:", or "non-blocking:" — move it to a line comment on the relevant file, then approve. The default for non-blocking notes is **line comment**, not approval body. This applies even when the observation is genuinely useful; usefulness is the bar for *posting* it, not for putting it in the approval body.

Exceptions that genuinely belong in the body:

- **Coordination notes** — merge order, deployment dependencies, "merge after #1234".
- **Acknowledged unrelated failures** — naming a pre-existing red check so reviewers know it's known and not caused by this PR.
- **Material caveats** — a known limitation of the approval itself (e.g., "approving the API change; the consumer update is tracked in #1235").

Anything else that's "optional, suggestion, or feedback" belongs on a file line, not in the approval body.

#### Prefix line comments with "AI suggestion:"

When you post a line comment as part of an approval flow, **start the body with `AI suggestion:` on its own line, then a blank line, then the actual comment text**. Don't lead the first sentence with the phrase. This keeps AI-authored suggestions visually distinct from human review feedback so the author can weigh them accordingly.

```markdown
AI suggestion:

This fires for every `option.is_active`, but the target only exists for the currently selected option …
```

The prefix applies to line comments only — the approval body is already attributed to the reviewer in the GitHub UI.

### Format for readability

Reviewers skim. A wall of text hides the conclusion and makes the evidence hard to verify at a glance.

**Default structure, always: a one-line lead, a blank line, then bullets — never a single flowing paragraph.** Even a short approval takes the lead-plus-bullets shape. A prose paragraph that strings five facts together with commas and "and" is the single most common way an otherwise-good approval becomes unreadable — if you catch yourself writing one, that's the signal to split it into bullets. And break to a new line after every sentence-ending `.`, `?`, or `!`, because GitHub collapses consecutive sentences into one wrapped block and the structure you intended disappears.

- **One short lead line, blank line, then bullets.** The lead is the verdict ("Approved.", "Approved with one non-blocking note.").
- **3 bullets max — fewer is better.** Keep only the points that change merge confidence. If a fourth point feels essential, it's usually a line comment, not approval-body material (see "Body vs. line comments"). More than three bullets means the PR needs a real review thread, not a longer approval.
- **One short sentence per bullet, one idea.** A bullet is a headline, not a paragraph. If you need a second sentence, a trailing `— because…` clause, or an "and" stitching two facts together to make the point, it's too deep for the approval body — cut it to its core claim or move the detail to a line comment. The author can open the diff for the rest; the body exists to state the verdict and the one or two things that justify it.
- **Blank lines between paragraphs / between the lead and the bullet list.** GitHub markdown collapses without them.
- **Inline `code` is for one short identifier.** If you'd need three or more inline-code spans in a sentence, switch to bullets (one identifier per bullet) or a fenced block.
- **Fenced code block when showing more than one line of code, or an actual snippet rather than just a name.**
- **Spell out acronyms on first use** within a comment ("JWT (JSON Web Token) refresh", "CSP (Content Security Policy)"). After the first occurrence the bare acronym is fine.
- **Start a new line after every sentence — including inside a bullet.** Break after each sentence-ending `.`, `?`, or `!`. GitHub markdown renders consecutive sentences as one wrapped block — hard to scan. (Periods inside identifiers like `option.is_active`, abbreviations like `e.g.`, or version numbers don't count — only sentence-ending punctuation.)

Anti-pattern (wall of text):

```markdown
Approved. The mechanical swap from `/regex/.test(input)` to `isValidFoo()` is consistent across all interfaces, and the divergent paths are intentional per the PR description: acme keeps its existing auth carve-out, while globex intentionally always validates…
```

Better (lead + bullets):

```markdown
Approved.

- Mechanical swap from `/regex/.test(input)` to `isValidFoo()` is consistent across all interfaces.
- Divergent paths are intentional per the PR description: acme keeps the auth carve-out, globex intentionally always validates.
- Non-blocking JSDoc cleanup can land as a follow-up.
```

Each bullet is a headline — trim the paragraph down to its claim. A reviewer who wants the mechanism opens the diff.

Too deep (each bullet is a multi-sentence paragraph — the most common way an approval gets bloated):

```markdown
Approved.

- The validation swap is the correct fix: `validateInput()` now runs on every interface, so malformed payloads are rejected before they reach the mapper, while the legacy `/regex/` path is fully removed. Confirmed no callers still reference the old `rawInputCheck` helper.
- Legacy records are handled by normalize-on-read rather than a backfill. The check only fires on the old payload shape, and the writer now emits the canonical shape, so new writes can't re-trigger it.
- Non-blocking, follow-up: `normalizeLegacyShape` runs on every read path (4 call sites), and the records carry no version marker to self-heal. Worth normalizing once at load.
```

Better (headlines; the follow-up moves to a line comment):

```markdown
Approved.

- Validation swap is the canonical fix: `validateInput()` rejects malformed payloads before the mapper.
- Legacy records normalize-on-read; the new write shape can't re-trigger the old path.
- Swap is complete — no callers reference the old `rawInputCheck` helper.
```

### Approval body shapes

Short approval with evidence:

```markdown
Approved.

- <specific behavior/design> is correct and fits the existing flow.
- <targeted test/check> passes.
- <known unrelated warning/failure> is pre-existing.
```

Approval with a coordination or material caveat:

```markdown
Approved.

- <specific reason this is safe to approve>.
- <coordination/material caveat: e.g. merge after #1234, related migration in #1235, pre-existing red check>.
```

Approval after deeper investigation:

```markdown
Approved. Verified <risk area> against <source of truth>:

- <evidence 1: behavior, data, or code path>
- <evidence 2: unaffected flow or edge case>
- <evidence 3: checks/tests>
```

Genuine one-liner — only when there's truly one point:

```markdown
LGTM — the new validation is scoped to authenticated routes; the public endpoints are unchanged.
```

Padded with non-blocking nits (anti-pattern — move these to line comments instead):

```markdown
Approved.

- Core change looks correct.
- Targeted test passes.
- Minor: JSDoc could be tidied as a follow-up.
- Nit: variable name on line 42 could be clearer.
- Non-blocking: consider extracting the helper into a util.
```

Too process-heavy (anti-pattern):

```markdown
Approved. I reviewed the latest PR head from GitHub, checked existing review threads, and verified tests.
```

Too generic (anti-pattern):

```markdown
Approved. The changes look consistent with the PR goal, and I do not see any blocking issues. Nice to move forward.
```

### Submitting the approval

If line comments are part of the flow, post them first so the approval review wraps everything together. Then submit:

```bash
gh pr review <num> --approve --body "$(cat <<'EOF'
Approved.

- <bullet 1>
- <bullet 2>
EOF
)"
```

Use `--body-file <path>` instead of inline heredoc when the body lives in a file already. Never submit the approval before the user has clearly authorized it for *this* PR — prior authorization on a different PR does not carry over.

## What Not to Do

- **Don't post comments yourself.** The user posts. You provide the text.
- **Don't approve until explicitly authorized.** If the user asks "can I approve this PR?" or "should this be approved?", answer with the verdict first. Submit an approving GitHub review only when the user clearly instructs you to — see "Writing the Approval Comment" below.
- **Don't trust "resolved" markers.** Verify against the diff.
- **Don't recite phases.** They're discipline, not a script.
- **Don't pad with process narration.** "I fetched the latest head, then read the diff, then checked CI..." — the user assumes you did. Show conclusions, not steps.
- **Don't bike-shed nits.** If the linter or formatter could catch it, it's not your job.
- **Don't approve away unrelated CI failures silently.** If checks are red, name which failures are pre-existing/unrelated and which were introduced by this PR.

## Common Failure Modes

These are the patterns that cause a PR review to mislead:

- **Rubber-stamping** — scrolling the diff without checking whether tests actually exercise the change.
- **Trusting `resolved` threads** — the marker means someone clicked a button, not that the fix is correct.
- **Skipping the default-branch check** — a green test suite on the PR branch doesn't prove the new test would have caught the bug.
- **Treating tautological mocks as coverage** — `mock.return_value = X; assert result == X` is not a test.
- **Stale local state** — reviewing against a SHA that's already been replaced by a force-push.
- **Generic verdicts** — "looks good, approving" with no evidence is indistinguishable from not having reviewed.

## Useful Shapes

A clean approve:

```markdown
**Verdict:** Approve.

**Why:**
- The mapper change in `src/foo/mapper.ts` correctly handles nested input fields and the new test exercises the previously-broken path.
- Existing flow is unchanged; verified by reading `src/foo/handler.ts` against the diff.
- CI green; no unrelated failures.

**What I'd do differently:** Nothing material — the structure fits the existing pattern.
```

Hold for clarification:

```markdown
**Verdict:** Hold.

**Why:**
- The new `acceptedFoo` filter changes the calculation in `barService.ts:209-218`, but it's unclear whether partial-acceptance entries should still be eligible. The PR description doesn't say.

**Critical issues to surface:**

> What's the intended behavior when `acceptedQty < expectedQty`? The current code treats partial as full — was that intentional?
```

Don't approve:

```markdown
**Verdict:** Don't approve yet.

**Why:**
- 🔴 The `lookupFoo` test mocks the SQL client to return the exact shape the assertion checks, so the test passes regardless of the production query (`tests/lookup.test.ts:42`).
- 🔴 Existing comment at `r1234567890` about null-handling at `foo.ts:180` is not addressed in the current diff.

**Critical issues to surface:**

> The new test at `tests/lookup.test.ts:42` is tautological — the mock dictates the assertion. Suggest replacing with an integration test that hits the real query path, or removing the test if coverage already exists at the integration layer.

> The null-handling concern from r1234567890 still applies — `item.barQty` can be `undefined` for child entries, and `foo.ts:180` will throw. The diff doesn't change that path.
```
