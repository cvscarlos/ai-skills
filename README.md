# Handover — Agent Skill

A skill for AI coding agents (Claude Code, Cursor, Windsurf, Copilot, etc.) that generates structured handover documents so another agent can seamlessly continue your work without prior context.

Follows the [Agent Skills](https://skills.sh) format.

## Installation

```bash
npx skills add cvscarlos/ai-skills --skill handover
```

## What it Does

Commits in-progress work, captures decisions, failures, and blockers, then writes a provider-agnostic `HANDOVER.md`.

**Use when:**
- Switching between AI tools (Claude Code, Codex, Cursor, etc.)
- Handing off a task to another session or machine
- Saving state before ending a session
- Moving work between directories of the same repository

## License

MIT
