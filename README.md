# AI Skills

**Production-grade skills for AI coding agents.**

Works with Claude Code, Codex, Cursor, Windsurf, Copilot. Any tool that speaks the [Agent Skills](https://skills.sh) format.

AI agents are powerful. They also drop context. They skip security checks. They install whatever you ask, no questions.

These skills fix that. One failure mode at a time.

---

## Skills

### [handover](./skills/handover/) — never lose context

Switching from Claude Code to Codex? Continuing on a different machine? Ending the day mid-feature?

`handover` commits your work-in-progress and writes a structured `HANDOVER.md`. Objective. Status. What's stuck. What didn't work.

The next agent picks up exactly where you left off. No re-explaining. No lost decisions.

```bash
npx skills add cvscarlos/ai-skills --skill handover
```

**Use when:** switching providers · handoffs between sessions · end of day · moving between machines.

---

### [documentation](./skills/documentation/) — docs for humans, not developers

Ask an AI agent to "write documentation" and you usually get a JSDoc-flavored explainer aimed at the next engineer. Fine for an API reference. Useless for the Implementation engineer setting up a feature for a client, the PM tracking what shipped, or the CS rep answering a ticket.

`documentation` writes Markdown for the people who *use* the software but don't read code. Implementation, Product, CS, support, QA, ops, BAs. It reads the code as the source of truth, then writes in the language of the reader's job — workflows, business rules, configuration, what changes for the customer.

Scouts the repo's existing `docs/` folder to match house style. Code blocks survive only when the reader will copy, paste, send, or recognize them (embed snippets, configuration values, sample payloads). Unknowns get `[TODO]` placeholders, not fabrications.

```bash
npx skills add cvscarlos/ai-skills --skill documentation
```

**Use when:** documenting a feature for non-devs · writing a setup guide a client can follow · explaining how a feature ties to business rules · adding or editing files in the repo's `docs/` folder.

---

### [npm-security-check](./skills/npm-security-check/) — gate every npm install

One typosquat. One malicious postinstall script. That's all it takes to ship your SSH keys to a stranger.

The npm registry sees these every week. Your agent shouldn't be the delivery service.

`npm-security-check` runs a [Socket.dev](https://socket.dev) supply-chain check on every package the agent installs or executes. `npm install`. `yarn add`. `pnpm add`. `bun add`. `npx`. Even silent `package.json` edits.

Three modes, picked automatically:

1. **Socket CLI** — if you have it installed and authenticated.
2. **Socket.dev MCP** — if your agent has the MCP connected.
3. **Stop and ask** — if neither. Never auto-installs. Never auto-configures.

Malware → **ABORT**. Critical CVE → **ABORT**. Weak score → **WARN**. Clean → **PROCEED**, with a one-line audit trail.

```bash
npx skills add cvscarlos/ai-skills --skill npm-security-check
```

**Use when:** every time the agent touches an npm dependency. That's the point.

---

### [pr-review](./skills/pr-review/) — a verdict, not a rubber stamp

Ask an AI to review a PR and you get one of two failures: a wall of generic praise that's indistinguishable from not reviewing, or a checklist recited back at you. Neither tells you the one thing you need: *can I merge this?*

`pr-review` produces a verdict — **Approve**, **Hold**, or **Don't approve yet** — and the concrete evidence behind it. Specific files, code paths, failing checks. It triages every existing comment thread against the actual diff (a "resolved" marker isn't evidence — the diff is). It hunts the places PRs look fine but aren't: tautological mocks, tests that pass on the default branch, coverage that'll rot in six months.

Review-only by default. It never posts or approves on its own — only after you explicitly say so. When you do, it writes an approval comment for a human reader, not a report that an AI followed instructions.

Works against GitHub via the `gh` CLI.

```bash
npx skills add cvscarlos/ai-skills --skill pr-review
```

**Use when:** reviewing a PR · deciding whether to approve · triaging review comments · writing an approval that says something.

---

## Why these skills

Three rules. Every skill in this repo follows them.

- **Trigger when it matters.** Skills fire at the right moment — not when the user remembers to ask.
- **Respect the user's machine.** No global installs. No config writes. No `sudo`. If a skill needs something, it asks.
- **Provider-agnostic.** Same `SKILL.md` works in Claude Code, Codex, Cursor, Windsurf, Copilot.

## Install everything

```bash
npx skills add cvscarlos/ai-skills
```

Or one at a time with `--skill <name>`.

## Contributing

Found a failure mode a skill could fix? Open an issue or PR.

Skills should be small. Sharp. Worth keeping installed forever.

## License

MIT
