---
name: pr-review
description: Decide whether a GitHub PR can be approved, then optionally write and submit the approval comment when the user explicitly authorizes it. Use when the user says "review this PR", "review PR \#X", "can I approve this PR", "PR review", "give me a PR review", "look at this pull request", "approve it", "approve the PR", "submit an approval review", "LGTM comment", or pastes a GitHub PR URL. Produces a verdict (approve / hold / don't approve) with concrete evidence and a "what would I do differently" perspective. Approval mode is opt-in — the skill never approves or posts on its own, only after the user clearly instructs it to.
---

# PR Review

Use this skill when the user wants you to review a GitHub PR and tell them whether it is approvable. The user is the decision maker: you produce a clear verdict and the evidence behind it; they read, decide, and post any comments themselves.

Default mode is review-only — produce a verdict and evidence, but don't post comments or submit an approving review on your own. Approval is a separate, opt-in step (see "Writing the Approval Comment"); only act on it when the user explicitly tells you to approve.

## What "good review" means here

A review is useful when it changes the user's confidence about merging — by surfacing something they'd have missed, or confirming the PR is safe with concrete evidence. Anything that doesn't change their decision is noise. Write for a human who already knows the codebase: skip process language ("I fetched the PR", "I read the diff"), rubber-stamp praise, and pasted checklists. Lead with the verdict, then justify it.

## Always Refresh from GitHub

Local state lies — the branch may have been force-pushed, comments added, checks just failed. Before reviewing:

1. `git fetch origin` so the local ref reflects the remote.
2. If the PR branch is checked out, `git pull --ff-only` (or compare local HEAD vs the PR head SHA).
3. Pull live PR state from GitHub. Prefer the `gh` CLI over the GitHub MCP — the MCP returns large JSON blobs that burn tokens; `gh` lets you name exact `--json` fields and pipe through `jq`/`head`. Reach for the MCP only for what `gh` can't expose cleanly: threaded review-comment metadata (`isResolved`, `isOutdated`), complex code/issue search, or GraphQL-level fields.

```bash
gh pr view <num> --json title,body,state,mergeable,mergeStateStatus,additions,deletions,changedFiles,commits,reviews,statusCheckRollup,headRefOid
gh pr view <num> --comments   # timeline comments
gh pr diff <num>              # full diff
```

If the branch isn't checked out locally, that's fine — review against `gh pr diff` plus targeted `gh api` reads. Don't refuse for lack of a local checkout.

## Triage Existing Comments

For every comment thread, decide one of:

- **Addressed** — locate the change in the current diff and verify it resolves the concern. "Marked resolved" is not evidence; the diff is.
- **Outstanding** — correct and not addressed. Surface it.
- **Incorrect** — the comment was wrong (misread code, missed a constraint). Say so plainly with a one-line reason; don't pretend a wrong comment needs addressing.
- **Stale / non-blocking** — was correct once but no longer relevant, or a nit the author declined.

When listing outstanding comments back to the user, group by severity and use this table so they can open each thread by appending the Discussion ID to the base URL:

| Column          | Description                                                       |
| --------------- | ----------------------------------------------------------------- |
| `#`             | Sequential number                                                 |
| `Discussion ID` | The `rXXXXXXXXXX` identifier from the comment URL                 |
| `File`          | Filename (without full path)                                      |
| `Line`          | Line number                                                       |
| `Issue`         | Brief description                                                 |

```markdown
## P1 (Blocking) — 2 comments

| #   | Discussion ID | File            | Line | Issue               |
| --- | ------------- | --------------- | ---- | ------------------- |
| 1   | `r1234567890` | `foo.js`        | 180  | Missing null check  |
| 2   | `r1234567891` | `bar.js`        | 220  | Incorrect parameter |

**Base URL:** `https://github.com/{owner}/{repo}/pull/{number}#discussion_`
```

### Handing replies back for the user to post

When you have a reply the user should post on a specific thread but can't post it yourself (review-only mode, or a tool/permission limit), hand it back so it's trivial to paste: the **thread URL** on its own line as **plain text** (out of a code block, so it stays clickable), then the **reply in a fenced code block** (wrap only the answer so inline backticks render literally).

https://github.com/owner/repo/pull/123#discussion_r2662020009

```
Fixed at line 180-181: filter now uses `item.count > 0` instead of `item.enabled`
```

## Test Discipline

Tests are the most common place where a PR looks fine but isn't. A passing suite is not the same as a tested change. For every new/modified test, ask:

### 1. Would it fail on the default branch?

Check whether `main` or `development` is the default. If the test passes on both branches, it isn't testing the new behavior — it's decoration. Run it against the default branch when feasible, or reason from the diff about whether the assertion depends on the change.

### 2. Is the same coverage already elsewhere?

If the same path is covered at another layer with different inputs, an extra same-shape test adds maintenance cost without catching new bugs. Flag duplicates.

### 3. Is it tautological from over-mocking?

If the mock dictates the answer the assertion checks, the test only proves the mock was wired up (`mock.return_value = X; assert result == X`). That proves nothing about the production path.

### 4. Will the test still be valuable in six months?

Once the PR context is gone, a maintainer reads only the test name and body. Flag tests that will age badly:

- **Coupled to implementation details** — internal call counts, private state, exact ordering, mock-invocation specifics. They break on every refactor without behavior changing, and push maintainers to "fix" tests by mirroring whatever the code now does.
- **Coupled to a rare edge case** — an input the codebase won't realistically receive; the upkeep exceeds the bug it'd catch.
- **Coupled to a temporary assumption** — a feature-flag value, transitional schema, deprecated dependency, migration intermediate state. It silently locks the assumption in or breaks loudly at the worst time.
- **Redundant within the PR** — eight tests where two well-chosen ones would prove the change; the extra six are debt.

The lens: does the maintainer understand what behavior is protected, and is it worth protecting? If not, flag it.

## Review Phases

Internal discipline, not a checklist to recite back:

1. **Context** — what is the PR trying to do (description, linked issue, commits)? If you can't state the goal in one sentence, that's already a finding.
2. **High-level fit** — does the approach match the goal and the codebase's existing patterns? Is there a simpler version?
3. **Line-by-line** — logic, edge cases, null/undefined paths, security (input validation, injection, secret leakage), performance (N+1 queries, blocking I/O in hot paths), error handling.
4. **Tests** — apply the test discipline above.
5. **Comments** — triage every existing thread.
6. **Verdict** — approve / hold / don't approve, with the why.

## Severity Labels

Label each finding so the user can decide what to post — the same P1/P2/P3 scale as the `AI suggestion (P#):` prefix, plain text, no emojis:

- **P1 (blocking)** — must be addressed before merge
- **P2 (important)** — should be addressed; worth a comment, room for discussion
- **P3 (nit)** — optional preference; mention only if asked
- **suggestion** — an alternative approach the author may prefer (not a severity)
- **question** — clarification needed before approving (not a severity)

Don't label a P3 nit as P1, or bury a P1 in a list of suggestions. The label is the user's signal for what to actually post.

## Output Shape

Return the review in this order; skip empty sections rather than padding them.

1. **Verdict** — one line: **Approve** (safe as-is) · **Approve with non-blocking notes** · **Hold** (needs clarification or the author's response) · **Don't approve yet** (has P1 blockers; list them).
2. **Why** — 2–4 bullets of concrete evidence: specific files, behaviors, code paths, test results, CI status. No generic praise; if you say "the change is correct", say which change in which file and what makes it correct.
3. **Outstanding comments** (only if any) — the comment table above, grouped by severity if more than a few.
4. **Critical issues to surface** (only if any) — for each P1 blocker, the exact text the user can paste into GitHub, in their terse style. The user posts these, not you.
5. **What I'd do differently** — a short, concrete alternative if one's worth raising, even on an approve (a simpler structure, a better boundary, a name that'd age better, a cheap missing test). Skip if the approach is fine; don't invent rewrites.
6. **Approval** (only if the user explicitly authorizes it) — follow "Writing the Approval Comment". Otherwise stop after the verdict.

When a blocker needs more than a sentence (confirming a real issue *and* proposing the fix), keep it scannable: **bold call-to-action labels** (`**The problem:**`, `**Why it happens:**`, `**To solve it:**`) carry the structure; **one code block, for the fix only** (context already in the diff stays as prose with inline `identifiers`); **one line per point**, referencing the diff (`see _close_with_operation`) instead of reproducing it. Routine question/suggestion notes stay 1–2 sentences.

## Writing the Approval Comment

Only once the user has explicitly authorized approval after seeing the verdict. Default mode is still review-only.

### The review and the approval comment are different artifacts

Your review (what you report to the user) and the approval comment (what you post on GitHub) have different audiences and rules — don't build the second by reformatting the first.

- **The review is private and answers the user's request.** If they asked you to run tests, confirm they fail on `main`, reproduce a bug, or check a query is index-backed — do it and tell *them* what you found.
- **The approval comment is public and for the PR author.** It's a fresh artifact, filtered through "What not to put in the body" — not a copy of your review notes.

A user instruction like *"verify the new tests fail on the default branch"* tells you what to **do during the review**, not what to **post**. Having done it — and it passing — is not by itself postable. The user's prompt drives the review; this skill drives the posted comment.

### Pre-flight check (private to you)

Before drafting, internally confirm: the implementation matches the PR goal; no P1 blockers remain; previous concerns are addressed/outdated/incorrect/non-blocking; failing tests or checks are known and unrelated. Don't paste this list into the body.

### Body style

Write for the PR author and human reviewers, not as a report that an AI followed instructions.

- Lead with the engineering conclusion: what's correct, why it's safe to approve, or what non-blocking caveat remains.
- Stay anchored to *this* PR — judge it against its title, description, and target branch, not against changes you imagine it could have made. If a point doesn't connect to the author's stated goal, it doesn't belong.
- Include only evidence that changes reviewer confidence — behavior verified, production/log data, affected flows, key files, test results, CI status, known pre-existing failures.
- At least one sentence must be specific to this PR. Avoid generic approval phrases.
- Keep non-blocking concerns clearly optional; don't approve with language that sounds like unresolved required work.
- Compact, concrete language over pleasantries ("thanks", "nice work", "happy to", bare "looks good").
- If checks are partly blocked by unrelated existing failures, name the failing check or file.

### Report what you verified, not what the author decided

The author spent days here; you spent minutes. A bullet that *endorses their decision* — "catching it here is the right boundary", "the split maps cleanly", "suppressing under `?draft=true` is intended" — hands them nothing they don't already have, and "is intended" guesses at intent you can't see from the diff. What earns a place is work the author *couldn't* have done for themselves:

- **What you ran that CI can't show** — a test run against the *default branch* to confirm it fails there (CI runs it on the PR branch, where passing is guaranteed, so the pass proves nothing), reproducing the bug against the pre-fix commit, or "verified the affected page on the integration env". Never report the PR-branch pass or "tests green" — that's the automated gate. Report the discriminating result: *"the new test fails on `main`, so it genuinely covers the change."*
- **What you cross-checked outside the diff** — "verified against installed `dep@7.26.3`", "the `onError` signature in `lib@5.100.14` takes a 4th arg, so `meta` resolves", "cross-checked against the `handleServerError` helper".
- **A concrete, actionable finding** — a bug, a nit, a follow-up.

The test for every bullet: **could the author have written this without me?** If yes (their decision, their rationale, a restatement of the diff), cut it. If it took going outside the diff to know it (running it, checking a dependency version, reading adjacent code, hitting the live app), keep it.

```markdown
Approved.

- The 5 new regression tests each cover a distinct branch (unpublished, invalid payload, lazy-settings failure, draft-skip, registration route) and fail on `main` — real coverage, not decoration.
- Confirmed the escaped errors reached the error tracker unhandled before this — the new catch sits on the only path that hits them (`setup()`, before `dispatch`).
- Cross-checked the draft-skip: `?draft=true` is the sole caller that suppresses, so non-draft failures still notify as before.
```

### What not to put in the body

The author reads the approval cold — no chat history. Pass each bullet through: *does this give a reader-of-only-the-PR new information that justifies the approval?* If no, cut it. Specifically:

- **Resolution status of existing threads** — "Codex P2 is resolved", "addressed @X's feedback". The thread shows resolution; mention it only when it materially changes the approval (a security blocker or correctness bug fixed later).
- **Private review context** — concerns *you* considered that were never raised on the PR. Readers wonder why you brought it up.
- **Re-statements of the diff** — "the change moves X out of Y". The diff shows this; describe what was *verified*, not what changed.
- **Endorsements of the author's decisions** — "is the right boundary", "maps cleanly", "is intended". They made these calls deliberately; report what you *verified*, not what they *decided*.
- **Pure-process evidence** — "I read the comments", "I fetched GitHub". Only mention a step when it surfaces something the reader can't see (e.g. "verified against prod data").
- **Results from automated gates** — anything a linter, formatter, type-checker, pre-commit hook, or CI workflow runs on every push is a foregone conclusion. "Ran `ruff` — clean", "lint green" tells the reader nothing they didn't assume. Glance at `.github/workflows/`, `.pre-commit-config.yaml`, and linter config to see what's automated, and report only what they *can't* catch — logic, behavior, dead code, cross-file references, runtime correctness.
- **AI mechanics** — "I am an AI", "the user asked me to approve". Keep out entirely.

### Body vs. line comments

The approval body justifies the approval — it's **not** a dump of every non-blocking observation. If a bullet starts with "minor:", "nit:", "as a follow-up:", or "non-blocking:", move it to a line comment, then approve. The default for non-blocking notes is **line comment**, not body — usefulness is the bar for *posting* it, not for putting it in the body.

Exceptions that do belong in the body: **coordination notes** (merge order, "merge after #1234"), **acknowledged unrelated failures** (naming a pre-existing red check so reviewers know it's known), **material caveats** (a known limitation of the approval itself, e.g. "approving the API change; the consumer update is tracked in #1235").

#### Prefix line comments with "AI suggestion (P#):"

Whenever you post a line comment — approval flow or not — **start the body with `AI suggestion (P#):` on its own line, then a blank line, then the comment text** (don't lead the first sentence with the phrase). This keeps AI-authored suggestions visually distinct from human feedback, and the `P#` tells the author how much to care:

- **P1** — critical: a correctness, security, or data bug, or a real blocker. Should be fixed before merge.
- **P2** — important: worth addressing, but there's room for discussion.
- **P3** — nitpick: optional preference or polish; take it or leave it.

Format it **one sentence per line** (break after every sentence-ending `.`, `?`, `!`), same as the approval body.

```markdown
AI suggestion (P2):

This fires for every `option.is_active`, but the target only exists for the currently selected option.
Scope the handler to the active option so the others don't get redundant updates.
```

#### Line comment style: say it in a sentence

**A line comment is one sentence by default — write like a busy human reviewer, not a report.** The owner knows their own PR far better than you, so don't re-explain the mechanism, the background, or why it matters — say *what to change* (and where) and stop. An AI-attributed comment that runs several sentences gets skimmed or ignored; a one-liner gets read and acted on. And if a one-line ask won't move the owner, a paragraph won't either — so the extra text is pure cost, for both of you.

Shape it as **finding → ask**, each as short as it can be:

1. **The finding** — the defect or risk in a clause, evidence folded in (`file:line` or a short parenthetical), not narrated.
2. **The ask** — the specific change, concrete enough to act on.

**Cut the windup:** no "I noticed…", no restating what the code does, no mechanism lecture, no selling the benefit. Lead with the fact, not the investigation ("I verified locally that X" → "X (verified locally)"). The priority rides in the `AI suggestion (P#):` prefix, not the body. Politeness that costs no length is welcome ("Could you…"); soften the phrasing, never the claim.

Go past one or two sentences **only when the content needs it, never to justify** — and even then, keep it minimal:

- **Several items** (files, call sites, cases) → a short lead line, then **one bullet each** (with its `file:line`), not a comma-run in a paragraph.
- **A concrete change** → a ` ```suggestion ` block (one-click apply), not prose describing it.
- Anything else → resist adding structure; a simple point stays a sentence.

Before (bloated — narrated evidence, restated context, extra sentences):

```markdown
AI suggestion (P2):

None of the touched tests actually pin this fix — they all pass against `main`'s version of `return_label_service.py` (I verified locally: body courier == request courier everywhere, and `test_return_label` never asserts `label.courier`, so the acme-post fixture change is inert).
The new fallback test only pins behavior that `main` already had.
Can you add the one discriminating case: body `courier="acme-uk"`, request `courier="acme-dropoff"`, assert `label.courier == "acme-uk"`?
```

After (finding → fix):

```markdown
AI suggestion (P2):

All touched tests pass against `main`'s version of this file, so nothing pins the new precedence.
Could you add the discriminating case: body `courier="acme-uk"`, request `courier="acme-dropoff"`, assert `label.courier == "acme-uk"`?
```

When the finding enumerates several locations, bullets beat a comma-run — bad (four `file:line` refs crammed into a sentence):

```markdown
AI suggestion (P2):

Four files bypass these constants with bare strings — `foo/config.py:173`, `foo/utils.py:73`/`:77`, `foo/views.py:498`/`:506`, `foo/service.py:1071` — so an unexpected value silently reads as the default.
```

Better (lead line, one bullet per file, then the ask):

```markdown
AI suggestion (P2):

Four call sites bypass the constants with bare strings, so an unexpected value silently reads as the default:
- `foo/config.py:173`
- `foo/utils.py:73` / `:77`
- `foo/views.py:498` / `:506`
- `foo/service.py:1071`

Could you move them to a `StrEnum` + `*_CHOICES` list in `foo/choices.py`, mirroring the pattern two fields above?
```

### Format for readability

Reviewers skim. **Default structure, always: a one-line lead, a blank line, then bullets — never a single flowing paragraph.** A prose paragraph stringing five facts together with commas and "and" is the most common way a good approval becomes unreadable.

- **3 bullets max — fewer is better.** Keep only points that change merge confidence; a fourth point is usually a line comment.
- **One short sentence per bullet, one idea.** A bullet is a headline, not a paragraph. If you need a second sentence or an "and" stitching two facts together, it's too deep — cut to the core claim or move the detail to a line comment.
- **Blank lines** between the lead and the bullet list (GitHub markdown collapses without them).
- **Inline `code` is for one short identifier**; three or more spans in a sentence → use bullets (one per bullet) or a fenced block.
- **Fenced code block** when showing more than one line of code.
- **Spell out acronyms on first use** ("JWT (JSON Web Token)", "CSP (Content Security Policy)"); the bare acronym is fine after.
- **Start a new line after every sentence** (break after `.`, `?`, `!`), including inside a bullet. (Periods in identifiers like `option.is_active`, `e.g.`, or version numbers don't count.)

Anti-pattern (wall of text) → fix (lead + headline bullets):

```markdown
Approved. The mechanical swap from `/regex/.test(input)` to `isValidFoo()` is consistent across all interfaces, and the divergent paths are intentional per the PR description: acme keeps its auth carve-out while globex always validates…
```

```markdown
Approved.

- Mechanical swap from `/regex/.test(input)` to `isValidFoo()` is consistent across all interfaces.
- Divergent paths are intentional per the PR description: acme keeps the auth carve-out, globex always validates.
- Non-blocking JSDoc cleanup can land as a follow-up.
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
```

Genuine one-liner — only when there's truly one point:

```markdown
LGTM — the new validation is scoped to authenticated routes; the public endpoints are unchanged.
```

### Submitting the approval

If line comments are part of the flow, add them to a pending review first (see "Posting Comments — Pending Review Only") so the approval submission wraps everything together, then submit the pending review with `pull_request_review_write` method `submit_pending`, event `APPROVE`, and the approval body. Without line comments, `gh` works too:

```bash
gh pr review <num> --approve --body "$(cat <<'EOF'
Approved.

- <bullet 1>
- <bullet 2>
EOF
)"
```

Use `--body-file <path>` when the body already lives in a file. Never submit before the user has clearly authorized it for *this* PR — prior authorization on a different PR doesn't carry over.

## Posting Comments — Pending Review Only

When the user authorizes posting comments (with or without an approval), never post them as immediate one-off comments — no `gh pr comment`, no direct single review-comment API calls. Always batch them into a **pending review**:

1. Create a pending review (GitHub MCP `pull_request_review_write`, method `create`, no `event`).
2. Add each comment with `add_comment_to_pending_review`.
3. Leave it pending for the user to inspect, edit, and submit themselves — submit only if they explicitly say so, with the event they name.

A pending review keeps comments invisible to the author until the user signs off, lets them amend or discard before anything goes public, and lands all feedback as one review instead of a comment stream.

## What Not to Do

- **Don't post comments yourself.** The user posts; you provide the text. When authorized, go through a pending review — never one-off comments.
- **Don't approve until explicitly authorized.** If asked "can I approve this?", answer with the verdict first; submit an approving review only when clearly instructed.
- **Don't trust "resolved" markers.** Verify against the diff.
- **Don't recite phases.** They're discipline, not a script.
- **Don't pad with process narration.** The user assumes you did the steps — show conclusions, not steps.
- **Don't bike-shed nits.** If the linter or formatter could catch it, it's not your job.
- **Don't approve away unrelated CI failures silently.** Name which red checks are pre-existing/unrelated vs introduced by this PR.

## Common Failure Modes

- **Rubber-stamping** — scrolling the diff without checking whether tests exercise the change.
- **Trusting `resolved` threads** — the marker means a button was clicked, not that the fix is correct.
- **Skipping the default-branch check** — a green suite on the PR branch doesn't prove the new test would catch the bug.
- **Tautological mocks as coverage** — `mock.return_value = X; assert result == X` is not a test.
- **Stale local state** — reviewing a SHA already replaced by a force-push.
- **Generic verdicts** — "looks good, approving" with no evidence is indistinguishable from not reviewing.

## Useful Shapes

Clean approve:

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
- P1 — The `lookupFoo` test mocks the SQL client to return the exact shape the assertion checks, so it passes regardless of the production query (`tests/lookup.test.ts:42`).
- P1 — Existing comment at `r1234567890` about null-handling at `foo.ts:180` isn't addressed in the current diff.

**Critical issues to surface:**

> The new test at `tests/lookup.test.ts:42` is tautological — the mock dictates the assertion. Replace it with an integration test that hits the real query path, or remove it if coverage already exists at the integration layer.

> The null-handling concern from r1234567890 still applies — `item.barQty` can be `undefined` for child entries, and `foo.ts:180` will throw. The diff doesn't change that path.
```
