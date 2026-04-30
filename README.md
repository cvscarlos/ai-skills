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
