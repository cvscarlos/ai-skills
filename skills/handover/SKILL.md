---
name: handover
description: >
  Generates a comprehensive handover document for another AI agent to seamlessly
  continue work on a task without prior context. Use when asked to "handover",
  "save state", "switch providers", "switch to claude/codex/opencode/cursor", or
  prepare a summary for the next AI session. Also use when moving work between
  directories of the same repository.
---

# Agent Handover Protocol

Generate a structured handover file (`tmp/HANDOVER.md`) that captures enough context for any AI agent — regardless of provider or tool — to continue the current task from scratch.

## Execution Steps

### 1. Commit Uncommitted Work

If there are uncommitted changes, commit them before creating the handover so nothing is lost:

```bash
git status --short
# If changes exist:
git add <relevant-files>
git commit -m "wip: save progress before handover"
```

### 2. Gather State

Run these commands and capture output:

```bash
# Branch and repo
git branch --show-current
git remote get-url origin

# Detect base branch: PR base → main → master
BASE=$(gh pr view --json baseRefName -q .baseRefName 2>/dev/null \
  || (git rev-parse --verify main 2>/dev/null && echo main) \
  || echo master)

# Recent commits on this branch (since diverging from base)
git log --oneline "$BASE"..HEAD

# Uncommitted changes (if any remain)
git status --short
git diff --stat

# PR context (if any)
gh pr view --json number,title,url,state,baseRefName,reviewDecision 2>/dev/null
```

### 3. Analyze Session Context

From the conversation history, identify:

- **Objective**: The feature, fix, or refactor in progress
- **Completion estimate**: How far along is the overall task (e.g., "~70% — auth done, authz remains")
- **Done / In progress / Remaining**: Distinguish finished tasks, partially-done tasks (with current state), and not-yet-started tasks
- **What didn't work**: Failed approaches and why they were abandoned
- **Key decisions**: Design choices and their rationale
- **Gotchas**: Non-obvious issues discovered during the session
- **Blockers**: Failing tests, unresolved errors, open questions

### 4. Generate Document

1. Ensure `tmp/` exists and is in `.gitignore`.
2. Fill in the template below with gathered context. Omit sections with no relevant content.
3. Write to `tmp/HANDOVER.md`.

### 5. Quality Checklist

Before writing the file, verify:

- [ ] All local changes are committed (or explicitly noted as uncommitted)
- [ ] In-progress tasks describe their current state, not just their name
- [ ] Key decisions include rationale, not just the choice
- [ ] Next steps are specific enough to start immediately
- [ ] Key files are listed in priority order
- [ ] No repo-discoverable info included (stack, build commands, setup instructions)

### 6. Output

Reply: "Handover saved to `tmp/HANDOVER.md`. To resume: `cat tmp/HANDOVER.md`"

## Rules

- **Be specific**: Reference file paths, function names, error messages, and commit hashes.
- **Be actionable**: Next steps must be concrete enough to start immediately.
- **Capture the "why"**: Decisions without rationale are useless to the next agent.
- **Include failures**: What didn't work is often more valuable than what did.
- **Provider-agnostic**: No tool-specific instructions in the output. It must work for any AI tool.
- **Skip discoverable info**: Don't include information the next agent can find from the repo itself (stack, frameworks, build/test/run commands, project setup). The agent has access to `AGENTS.md`, `README`, `Makefile`, `package.json`, etc.

## Bad Handover Examples

**Too vague — useless to the next agent:**

```
## Progress
- Did some stuff with the API
- Made progress on auth

## Next Steps
- Finish the feature
```

**Missing context — leaves the next agent guessing:**

```
## Progress
- [x] Implemented the auth module

## Next Steps
- Fix the tests
```

No explanation of WHICH tests are failing, WHY they fail, or what was already tried.

## Template

```markdown
# Handover

> **Date:** YYYY-MM-DD
> **Repository:** <remote-url>
> **Branch:** `<current-branch>`
> **PR:** <PR-url-or-"N/A">
> **Completion:** <estimated % and one-line summary of what's done vs remaining>

## Objective

<1-3 sentences: what feature, fix, or refactor is in progress and why>

## Progress

### Done

- [x] <completed task — include file paths, function names, commit hashes>
- [x] <completed task>

### In Progress

- [ ] <partially-done task — describe current state and what remains>

### Remaining

- [ ] <not-yet-started task>

## What Didn't Work

- <failed approach> → <why it was abandoned>
- <approach that hit a dead end> → <what went wrong>

## Key Decisions

| Decision   | Rationale |
| ---------- | --------- |
| <decision> | <why>     |

## Gotchas

- <non-obvious issue that would waste time if rediscovered>
- <important context the next agent needs>

## Test Status

- **Passing:** <yes/no/partial>
- **Failing:** <specific test failures or "none">

## Blockers

<failing tests, unresolved errors, open questions — or "None">

## Next Steps

1. <specific first action to take>
2. <subsequent action>
3. <subsequent action>

## Key Files to Read First

1. `<path/to/file>` — <why it's important>
2. `<path/to/file>` — <why it's important>
```

---
