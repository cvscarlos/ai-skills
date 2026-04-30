---
name: npm-security-check
description: >
  Run a Socket.dev supply-chain security check on every npm/Node package
  BEFORE installing it, updating it, or executing it via npx. Triggers
  IMMEDIATELY whenever the agent is about to run `npm install <pkg>`,
  `npm i <pkg>`, `npm add <pkg>`, `npm update <pkg>`, `npm upgrade`,
  `yarn add`, `yarn upgrade`, `pnpm add`, `pnpm install <pkg>`,
  `pnpm update`, `bun add`, `bun install <pkg>`, `npx <pkg>`,
  `pnpm dlx`, `yarn dlx`, or `bunx <pkg>`. Also triggers on any phrase
  that implies adding or updating a Node/JavaScript dependency — "let's
  use <pkg>", "install the X package", "add <pkg> to deps", "bump
  <pkg>", "try this CLI: npx <pkg>", or editing `package.json` to add a
  dependency. Use this skill EVEN IF the user did not explicitly ask
  for a security check — supply-chain attacks, malware, and typosquats
  on npm are common and the cost of one extra check is minutes vs. days
  of cleanup. The skill checks Socket score, malware verdicts, install
  scripts, network/shell access, CVEs, and maintainer trust, then
  decides whether to proceed, warn, or abort.
---

# npm Security Check (Socket.dev)

Before installing, updating, or executing any npm package, run a Socket.dev supply-chain security check on each target package. npm is a frequent target for malware, typosquatting, and account-takeover attacks, and a single bad postinstall script can exfiltrate credentials from the developer's machine. The few seconds spent on a Socket check are cheap compared to incident response.

This skill uses **only what is already installed on the user's machine**. It does not install or configure anything autonomously — global package installs and credential configuration are the user's call, not the agent's.

The skill picks a mode in this order of preference:

1. **Socket CLI** (`socket` on `$PATH` AND it returns valid data) — fastest, deterministic.
2. **Socket.dev MCP** (`mcp__socketdev__depscore` tool exposed to the agent) — works inside any MCP-aware agent.
3. **Bootstrap mode** — neither is available, or the only one available is unauthenticated and the user has not authorized it. Stop and ask the user to install/authenticate one.

## When to Use

Trigger this skill before running any of these (this is not an exhaustive list — the rule is *any operation that pulls Node code from a registry*):

- `npm install <pkg>` / `npm i <pkg>` / `npm add <pkg>`
- `npm update`, `npm update <pkg>`, `npm upgrade`, `npm-check-updates` apply
- `yarn add <pkg>` / `yarn upgrade <pkg>` / `yarn dlx <pkg>`
- `pnpm add <pkg>` / `pnpm install <pkg>` / `pnpm update <pkg>` / `pnpm dlx <pkg>`
- `bun add <pkg>` / `bun install <pkg>` / `bunx <pkg>`
- `npx <pkg>` / `pnpx <pkg>` (executes a remote package — same risk as installing)
- Editing `package.json` `dependencies`, `devDependencies`, `peerDependencies`, or `optionalDependencies` to add or bump a package
- Editing a `Dockerfile`, CI workflow, or shell script to add an npm install step for a new package

**Skip the check** only when:

- Running `npm install` / `pnpm install` / `yarn` / `bun install` with **no package argument** to install an existing, already-vetted lockfile. (The packages in the lockfile were vetted when they were added.) If the user explicitly asks to re-audit, do it.
- The package is one the workspace already depends on at the same version range and you are only running scripts (`npm run build`, `npm test`).
- The user has explicitly said "skip the check" or "I trust this package" *for this specific install*. Honor that, but still surface the package URL so they can review later.

When in doubt, run the check. False positives waste seconds; false negatives waste days.

## Step 1 — Detect Available Tooling

Run read-only detection. Do **not** install anything, do **not** modify the user's Socket config, and do **not** run `socket login` or `socket config set …` — those touch global state and are the user's decision.

```bash
# Is the Socket CLI on PATH?
command -v socket >/dev/null 2>&1 && socket --version
```

Then check your own tool list for `mcp__socketdev__depscore`. If it is exposed, the Socket.dev MCP is connected to the agent.

Cache the detection result for the session. Re-detect only if the user tells you they installed or configured something.

Pick a mode:

| CLI present & working | MCP present | Mode                          |
| --------------------- | ----------- | ----------------------------- |
| yes                   | (any)       | **CLI mode** (Step 2a)        |
| no / unauthenticated  | yes         | **MCP mode** (Step 2b)        |
| no / unauthenticated  | no          | **Bootstrap mode** (Step 2c)  |

"CLI working" means it returns valid score data on the first attempt. If the first call returns `403`, `401`, or "API token may not have the required permissions", treat the CLI as **unauthenticated** for this session and move down the table — do not auto-run `socket login` or write to `~/.socket/`. Tell the user the CLI is installed but unauthenticated and let them decide whether to authenticate it (see Step 2c, Option A.2).

## Step 2a — CLI Mode

For each target package + version, run:

```bash
socket package score npm <name> <version> --json --no-banner --no-spinner
```

When the version is unknown (e.g., user said "install lodash"), resolve `latest` first:

```bash
npm view <name> version
```

For multiple packages, run the calls in parallel where the agent harness supports it.

If a call returns `403` / `401` / "API token may not have the required permissions", **do not** silently configure a token. Drop down to MCP mode (Step 2b) if the MCP is available, otherwise go to Step 2c and ask the user how they want to authenticate.

## Step 2b — MCP Mode

Call the Socket.dev MCP tool for each package. The canonical tool name is `mcp__socketdev__depscore`. The exact parameter shape can vary between MCP server versions, so inspect the tool schema first (in Claude Code, `ToolSearch` with `select:mcp__socketdev__depscore` loads it). At time of writing the MCP accepts a `packages` array with `{ depname, ecosystem, version }` entries.

Example shape:

```
mcp__socketdev__depscore({
  packages: [
    { depname: "<name>", ecosystem: "npm", version: "<version-or-latest>" }
  ]
})
```

Read the returned scores and alerts. If the response shape has changed and you cannot parse it, surface that to the user and offer to fall back to CLI mode (only if the CLI is authenticated and working) or Bootstrap mode.

## Step 2c — Bootstrap Mode

If neither the CLI nor the MCP can score the package, **stop and ask the user to install or authenticate one before continuing the install**. Do not silently skip the check — that defeats the purpose of the skill. Do not install or authenticate anything yourself — both options below modify global state on the user's machine and require explicit consent.

Tell the user something like:

> I'd normally run a Socket.dev supply-chain check on `<pkg>` before installing it, but I don't have a working way to do that on this machine. Would you like to:
>
> 1. Install / authenticate the **Socket CLI** (one-off setup, then it works for every future install), or
> 2. Add the **Socket.dev MCP server** to your agent config (no global install, just a config file edit), or
> 3. Skip the check this once (I'll proceed without it — please confirm)?

Then show whichever option(s) they pick. Do not run any of these commands yourself unless the user explicitly authorizes it in their reply.

### Option A — Socket CLI

#### A.1 Install (only if not already on PATH)

```bash
npm install -g socket@latest
socket --version
```

#### A.2 Authenticate (only if the CLI is installed but returns 401/403)

Two paths — let the user pick:

- **Free account** (recommended): create one at https://socket.dev, then run `socket login` and follow the browser flow. This removes rate limits and enables full features.
- **Public demo token** (no signup, rate-limited, fine for occasional checks): the user runs these two commands themselves to write to their own `~/.socket/config.json`:
  ```bash
  socket config set apiToken sktsec_t_--RAN5U4ivauy4w37-6aoKyYPDt5ZbaT5JBVMqiwKo_api --no-banner --no-spinner
  socket config set defaultOrg SocketDemo --no-banner --no-spinner
  ```

### Option B — Socket.dev MCP server

No global install — just a config file edit. Show the user the snippet for their agent and let them apply it.

**Claude Code / Claude Desktop** — add to `.mcp.json` (project-scoped) or `~/.claude/mcp.json` (global):

```json
{
  "mcpServers": {
    "socketdev": {
      "type": "http",
      "url": "https://mcp.socket.dev/"
    }
  }
}
```

**Codex CLI** — add to `~/.codex/config.toml`:

```toml
[mcp_servers.socketdev]
url = "https://mcp.socket.dev/"
```

**Cursor / Windsurf / other MCP-aware tools:** consult the tool's MCP config docs and add an HTTP MCP server pointing at `https://mcp.socket.dev/`.

After the user applies the change and restarts the agent, the MCP will be available and this skill will pick it up automatically. The agent does not need to do anything else.

### Option C — Skip the check (last resort)

If the user explicitly says "skip the check" or "just install it", proceed with the install but record in the transcript that the Socket check was skipped at the user's request. This is the only condition under which it is acceptable to bypass the gate.

## Step 3 — Decide: Proceed, Warn, or Abort

For each package, classify the result:

### ABORT — do NOT install

Refuse to run the install command and tell the user why. Suggest alternatives or the original safe version if the user is upgrading.

- **Malware** flagged by Socket (any `malware`, `gptMalware`, `gptSecurity` alert with high confidence).
- **Typosquat** alert — package name is close to a popular package and Socket flags it.
- **Critical CVE** with no fix available, or the requested version is in the affected range.
- **Suspicious install scripts** that perform network exfiltration, write to `~/.ssh`, read env vars wholesale, or run obfuscated code.
- Socket overall score below **40** combined with active critical alerts.

### WARN — surface to the user, ask before proceeding

Report the findings and let the user decide. Do not silently install.

- High-severity CVE with a fix available in a newer version (suggest the fix version).
- Single maintainer for a heavily-depended-on package.
- No publishes in 24+ months on a package the user actively wants to add.
- Permissive license issues (GPL/AGPL into a commercial project, SSPL, no license).
- Socket score 40–60 without critical alerts.
- Heavy transitive footprint relative to the package's purpose (e.g., 200+ deps for a one-line utility).

### PROCEED — run the install

- Socket score above ~70 with no critical alerts.
- Known-good packages from trusted maintainers (still report the score briefly so the user has audit trail).
- Already pinned in lockfile at the same version.

## Step 4 — Report Findings, Then Install

Always show the user a short summary before running the install command, even on PROCEED. The point is auditability — the user should see *what was checked* in the transcript, not just trust that the agent did the right thing.

**Summary template:**

```
Socket check for <pkg>@<version>:
  Score: <n>/100  Mode: <cli|mcp>
  Alerts: <count by severity, or "none">
  CVEs: <count by severity, or "none">
  Recommendation: <PROCEED | WARN | ABORT>
  More: https://socket.dev/npm/package/<name>/overview
```

After PROCEED, run the original install command. After WARN, wait for explicit user confirmation. After ABORT, stop and offer alternatives.

## Examples

**Example 1 — clean package, agent proceeds**

> User: "install lodash"
>
> Agent: *checks `socket package score npm lodash latest`* → score 88, no critical alerts, no malware.
>
> Agent: "Socket check for lodash@4.17.21: score 88/100, no critical alerts. Proceeding with `npm install lodash`."

**Example 2 — typosquat / malware, agent aborts**

> User: "install discord-selfbot-v14"
>
> Agent: *checks Socket* → malware alert, install script exfiltrates Discord tokens.
>
> Agent: "**MALWARE DETECTED — not installing.** Socket flags `discord-selfbot-v14` for credential-stealing install scripts (token exfiltration to a remote server). Do NOT install. If you wanted the legitimate library, the package you likely meant is `discord.js`. Want me to check that one instead?"

**Example 3 — known package with a high-sev CVE, agent warns**

> User: "add axios"
>
> Agent: *checks `axios@latest`* → score 78, but a high CVE on the version range the user has pinned.
>
> Agent: "Socket check for axios@1.6.0: score 78/100, but **CVE-2024-XXXX (high)** affects versions <1.7.4. Recommended: install `^1.7.4` instead. Should I proceed with `npm install axios@^1.7.4`?"

**Example 4 — npx is the same risk as install**

> User: "run `npx some-cli-tool` to scaffold the project"
>
> Agent: *checks Socket on `some-cli-tool`* — npx downloads and executes the package, so it gets the same gate.

## Error Handling

- **CLI returns 401/403 / "API token may not have the required permissions"**: the CLI is installed but unauthenticated. Do not run `socket login` or write to `~/.socket/` yourself. Drop down to MCP mode if available; otherwise go to Bootstrap mode and let the user choose how to authenticate.
- **Socket API rate-limited (HTTP 429)**: tell the user the rate limit was hit and suggest they authenticate with `socket login` for a free account if they want higher limits. Don't retry in a tight loop.
- **Package not found in Socket's index**: rare for npm. Surface the limitation to the user, fall back to `npm view <name>` for basic metadata, and recommend manual review.
- **MCP tool returns an unexpected shape**: the Socket MCP is evolving. Surface the parse error to the user and offer to fall back to the CLI (only if it is authenticated and working) or Bootstrap mode.
- **Network unavailable**: tell the user the check could not complete and ask whether to skip (with explicit acknowledgement) or wait. Never silently skip.
- **User says "just install it"**: proceed, but record the skipped check in your reply so it's auditable later.

## Tips

- Cache the detection result (`cli | mcp | bootstrap`) for the session — re-detect only on user request or after the user installs new tooling.
- For a `package.json` bulk add (e.g., user pastes 10 deps), batch the Socket checks in parallel — the CLI and MCP both handle concurrent calls fine.
- For monorepo workspace adds (`pnpm add -w`, `yarn workspace foo add bar`), check the package once — the workspace target doesn't change the supply-chain risk.
- After install, the `/socket-scan` skill (if installed) does a deeper post-install audit. This skill is intentionally narrower — it only gates the install moment.
- For pre-existing dependencies, see `socket-inspect` and `socket-scan` from the official Socket skill set. This skill is the *gate*, not the audit.
- Don't paraphrase Socket's verdict away — if Socket says "malware", say "malware" to the user. Don't soften it to "potentially risky".
- If you are uncertain whether a command counts as "installing", err on the side of running the check. The cost is a few seconds; the cost of skipping is potentially a compromised dev machine.
