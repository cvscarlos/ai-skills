---
name: documentation
description: Writes human-friendly Markdown documentation for non-developer audiences — Implementation, product, project, CS, support, QA, ops, business analysts. Explains how a feature works, how to set it up or configure it for a client, how it ties to business rules and the product roadmap — for technical readers who don't read code. Code blocks appear only when the reader will copy, paste, send, or recognize them (embed snippets, config values, sample payloads) — not to explain how the system is built. Use when asked to "document this for the implementation team", "write a guide for product / PM / CS", "explain how X works for non-devs", or for setup guides, runbooks, FAQs, hand-off docs aimed at internal non-dev teams. Also use when adding or editing a Markdown file inside the repo's `docs/` folder for a non-developer audience. Do NOT use for API reference, developer READMEs, code-level docs (JSDoc, docstrings), or framework / testing guides — those are different docs.
---

# Documentation

Creates a Markdown file in the current repository. The file explains a system, feature, workflow, or setup procedure to people who **work with the software but don't read code**. The reader knows enough to follow workflows, configuration, and edge cases — and they stop reading when they hit a code block they don't need.

The main idea: **read the code as the source of truth, then write the doc in the language of the reader's job**. Turn technical details into the behavior the business sees. Keep only the code blocks the reader will copy, paste, send, or recognize. Remove everything else.

Prefer short and focused docs. A reader who finishes a short, focused doc gets more value than a reader who skims a long one. Most docs are read quickly, in Notion or a wiki tab, with another tab waiting. The doc that respects that wins. Remove any section, sentence, or adjective that does not add clear value.

## Execution Steps

### 1. Establish Scope

Before reading anything, get clear on these — they shape _your_ writing, but **none of them appear in the doc itself**:

- **Subject** — which feature, workflow, integration, or system area
- **Reader profile** (your private focus, not a label for the doc) — Implementation, PM, CS, support, QA, ops, BAs, or a general non-developer audience. This shapes your word choice and what you explain vs. assume. It does **not** appear as an `Audience:` subtitle, a "Who this is for" section, or any other audience label in the final doc. Published docs end up in Notion, wikis, and search, where anyone in the company can read them; a fixed audience label tells unintended readers "this is not for you" without making the doc better for the intended ones.
- **What the reader needs to do or know after reading** — copy an embed snippet? Configure a setting? Answer a customer ticket? Approve a change? Understand what just shipped? This shapes which sections matter and which code blocks (if any) are worth keeping.

If any of this is unclear, ask one short question. The wrong reader profile in your head produces the wrong doc.

### 2. Scout the Repo for Existing Docs

Before writing, scan the repository for existing documentation. This is how the skill matches each project's style, instead of forcing the same shape on every repo.

Look for:

- **Where docs live** — `docs/`, `documentation/`, `wiki/`, root-level `*.md`, or a sibling docs repo referenced in `README`
- **Folder structure and naming** — flat vs nested, kebab-case vs Title Case, slug conventions
- **Tone and voice** — second person ("you set the…") vs third ("the user sets the…"), formal vs conversational, present vs imperative
- **Code-block density** — how much code do existing docs include? Are they prose-only? Heavy on configuration snippets? Sprinkled with sample payloads? **Match this convention** unless the user asks otherwise.
- **Metadata and structure conventions** — front-matter (`title`, `audience`, `last-reviewed`), heading order, "Related docs" sections, callout styles
- **Diagrams** — does the repo use Mermaid, PlantUML, ASCII, image files, or skip diagrams entirely?

Write the new doc into the same folder, using the same naming and structure. If the repo has no existing docs, use the template at the end of this skill, and decide for yourself how much code to include (see Rules: _Include code only when the reader needs it_).

### 3. Investigate the System

Read enough of the code, config, and related docs to understand the full behavior. You are looking for _what the system does_, not _how it is built_:

- The trigger — what event, action, or input starts the workflow
- The path — what happens, in what order, with what branches
- The external connections — what users, customers, queues, external systems, or UIs see, send, or receive
- The configuration settings — what is tunable, by whom, with what effect
- The business rules — thresholds, eligibility, ownership, SLAs, plan-tier differences
- The failure modes — what can go wrong, what the system does about it, what a human has to do
- The artifacts the reader handles — embed snippets, config values, customer-facing copy, sample payloads

Skip implementation details that do not change behavior (framework choice, internal helper structure, recent refactors).

### 4. Translate Implementation into Reader Language

For every technical concept you found, write down its **business meaning** before it goes in the doc. This step is what makes this skill different from a generic doc writer.

| Code-world term            | Reader-world translation                         |
| -------------------------- | ------------------------------------------------ |
| `InvoiceCreated` event     | "when a new invoice is generated"                |
| `retry_after_seconds=300`  | "the system waits 5 minutes before trying again" |
| `if user.tier == "pro"`    | "for Pro-tier customers only"                    |
| `nullable: true` in schema | "this field is optional"                         |
| `429 from upstream`        | "the partner system is rate-limiting us"         |

If a concept has no business meaning, it probably does not belong in the doc. The exceptions are _things the reader copies, pastes, or sends_ — see Rules.

### 5. Draft in One Shot, Keep the Doc Clean

Write the full doc in one pass, using **only what you can confirm** from the code, sibling docs, and what the user has already told you. The file you save is what the reader will see — it must be ready to read, not a worksheet with author notes.

**Do not put `[TODO]` markers, `TBD`, `[needs confirmation]`, or any other author note in the doc.** These exist for the writer, not the reader. Once the doc is published to Notion or a wiki, they become noise that future readers must ignore. If you cannot fill a section with confidence, you have two options — never a placeholder:

1. **Omit** — leave the section out if the information is not important for the reader's job. A missing "Roles and responsibilities" section is better than a section saying "TBD".
2. **Soften** — write what you do know and skip the rest. "Failed retries appear on the operations dashboard" is ready to publish; "Failed retries appear on the [TODO] dashboard, owned by [TODO]" is not.

Keep a private list of open questions as you write — things you could not confirm: roadmap status, plan-tier rules, named owners, SLAs, screenshots, business reasons. You will share these with the user in Step 7, **as a list in your chat reply, not in the doc**.

### 6. Review and Restructure

Before saving, re-read the draft from top to bottom as a reader would. Look for two common problems:

**Duplicated deep content.** The same workflow, business rule, or configuration explained in full in more than one section. When you find it, pick the main section to own the topic (usually the section the reader reaches first), keep the full explanation there, and replace the other occurrence with a one-line summary and a reference. If two sections say the same thing because they _should_ be one section, merge them — feel free to change the structure; the draft is not final.

**Layered content is not duplication.** A short "Quick start" followed by a detailed "Setup guide" is fine — the quick start is intentionally short (5 numbered steps, no context), and the setup explains each step in detail. The same is true for TL;DR/Overview pairs, Examples that show rules stated earlier, and a Glossary that defines terms used in the body. Ask yourself: does each version add something new (more depth, a different view, a concrete example, or a summary), or does the later one just repeat the earlier one?

When you restructure, move sections instead of copying them. If a paragraph belongs in a different section, cut and paste it — do not leave the original behind.

Then run the checklist:

- [ ] Someone matching the intended reader profile (per Step 1, kept private) could explain the feature to a colleague after reading once
- [ ] The doc does **not** advertise its audience anywhere — no `Audience:` subtitle, no `Who this is for` section, no `For X and Y...` tagline under the title
- [ ] Every acronym, internal term, and product name is defined the first time it appears
- [ ] Code-block density matches the repo's existing docs (or the reader's needs, if no prior docs)
- [ ] Each code block in the doc is something the reader will copy, paste, send, recognize, or edit — not something that explains how the system is built
- [ ] No function names, class names, file paths, or internal service names appear in headings or prose, unless the reader uses them directly
- [ ] Workflows are written as numbered steps in plain language, not as pseudocode
- [ ] Every business rule answers "what" _and_ "why it matters in practice"
- [ ] Each failure mode includes "what the reader should do", not just a description
- [ ] When ownership matters (who configures, who gets paged, who approves), it is either stated clearly from what you know, or the section is omitted — **never** filled with a `[TODO]` marker
- [ ] The doc has zero author notes: no `[TODO]`, no `TBD`, no `<TBC>`, no `?` placeholders, no "TODO: confirm with user" lines. The reader sees a finished doc; open questions live in your chat reply
- [ ] No deep explanation is repeated in two places. Layered content (Quick start → Details, TL;DR → Overview, Examples showing Rules) is fine; explaining the same workflow or rule in full in two sections is not — pick one section to own it and reference it from the other

### 7. Save and Follow Up

1. Ensure the target directory exists.
2. Write the file to the path that matches the repo's docs convention (from Step 2). Default if none: `docs/<slug>.md`.
3. Reply with:
   - The path of the saved file
   - A one-line summary of what the doc covers and who it's for
   - A numbered list of **open questions** you couldn't answer confidently from code or sibling docs (roadmap status, plan tiers, owners, screenshots, business rationale, etc.) — these are the things you omitted or softened in Step 5

Then go through the open questions with the user, one at a time. When they answer, add the new information back into the doc — replacing the softened phrasing with the confirmed details, or adding sections you omitted on purpose. The doc stays clean the whole time; the chat is where open questions live.

## Rules

- **Audience is a private focus, not a doc attribute.** Knowing whether you're writing for Implementation, Product, CS, or support shapes your word choice and assumptions. But the published doc does **not** advertise its audience — no `Audience:` subtitle, no `Who this is for` section, no `For X and Y…` tagline. Published docs travel to Notion, wikis, and search, where anyone in the company can read them; a fixed audience label limits readership for no benefit.
- **Explain the why, not the how.** A reader doesn't need to know that you used a queue; they need to know that retries happen automatically for an hour, and then a human gets paged.
- **Include code only when the reader needs it.** Add a code block when the reader will _copy, paste, send, configure, or recognize_ it — for example, embed snippets for a plugin, configuration values, sample webhook payloads, customer-facing email templates. Leave out code that only explains internal structure (class names, framework patterns, file paths). When the repo has existing docs, match how much code they include; when it doesn't, prefer less code and justify each block you keep.
- **Remove short-lived details, keep stable references.** Branch names, ticket IDs, commit hashes, Sentry links, refactor history — these become outdated quickly and mean nothing to the reader. Remove them. But stable, named things the reader actually uses in their job — file paths the customer pastes, config keys they fill in, field names they see in schemas, named services in the architecture — are fine to include, especially in a separate technical-reference section near the end of the doc.
- **Concrete over abstract.** "The system retries failed deliveries up to 5 times over 24 hours" is better than "the system has retry logic". Numbers, names, and concrete examples are worth keeping.
- **Match the repo's existing style.** If existing docs are there, follow their structure, tone, and style. The skill's template is a backup for repos with no existing docs, not a replacement.
- **Short and focused beats long and complete — every time.** This is the default for the skill. A reader who finishes a focused 150-line doc gets more value than a reader who skims a 400-line one. When in doubt, cut. When unsure whether a section is worth keeping, omit it; you can always add it back when the user asks. Length is not a quality signal; covering what the reader needs is the real signal. The template at the end of this skill is a checklist of sections that _can_ be useful, never a list of what _must_ appear.
- **Match the length, too.** When sibling docs in the same folder exist, follow their length — if they are 100–200 lines, stay in that range; don't write a 400-line doc when the others are 150 lines. Use the sibling length as a ceiling, not a target.
- **The doc is ready to read, never a worksheet.** No `[TODO]`, no `TBD`, no `<TBC>`, no "[needs screenshot]", no "[confirm with user]". Author notes are noise to the reader. They stay in Notion and wikis after the conversation that created them ends, and no one remembers what they meant. If you can't fill something with confidence, omit it or use more general words so the gap is hidden. Keep open questions in your private notes and raise them in your chat reply, not in the doc.
- **Write first, ask second.** Write the whole doc in one pass using what you have, then share the open questions with the user in the chat. Don't block the doc behind a long Q&A before any text is on the page; equally, don't fill the doc with placeholders that show your own open questions.
- **Diagrams and tables when they help.** Lifecycle diagrams (Mermaid), responsibility tables, decision matrices, and concrete examples are useful when they replace prose, not when they only decorate the doc. Use them when they save the reader work.
- **One file, one feature.** Don't combine unrelated topics. If the request covers two systems, write two docs and link them.
- **One topic, one main home.** In a single doc, every deep explanation lives in exactly one section. If you find yourself explaining the retry workflow in full under "How it works" _and_ again in full under "Failure modes", pick the main section to own the topic and replace the duplicate with a one-line summary and a reference. Layered content (a short Quick start that points at a detailed Setup section, a TL;DR that introduces an Overview, Examples that show Rules stated earlier) is the opposite of duplication and is good — each layer adds something the others don't.

## Anti-Patterns

**Code-heavy doc — wrong reason for the code:**

```markdown
## How it works

The `InvoiceEventProcessor.handle()` method is invoked when an `InvoiceCreated`
event arrives on the `billing.events` queue. It calls `EmailService.send()`
with a `template_id` of `5`.
```

The reader cannot use class names. They stop reading at the first identifier in backticks.

**Vague doc — saying nothing concrete:**

```markdown
## How it works

When something happens, the system sends an email to the customer. There is
some retry logic in case it fails.
```

No trigger, no timing, no rules, no failure behavior. The reader cannot answer a single real question after reading this.

**Better shape — prose for the workflow, code only for the artifact:**

```markdown
## How it works

When a new invoice is generated, the system emails the customer their receipt
within ~1 minute. If the customer's email provider rejects the message, we
retry every 15 minutes for up to 4 hours. After that, the message is marked
as failed and shows up on the billing-ops dashboard for a human to follow up.

## Embed the chat widget on the customer's website

Paste this snippet into the customer's site template, just before the
closing `</body>` tag. Replace `YOUR_ACCOUNT_ID` with the value from the
customer's onboarding form:

\`\`\`html

<script
  src="https://cdn.example.com/widget.js"
  data-account="YOUR_ACCOUNT_ID"
  async
></script>

\`\`\`

The widget loads asynchronously and adds ~12 KB to the page.
```

The workflow is prose; the snippet is in code because that is what the Implementation engineer hands to the client.

**Audience-labeled doc — limits the reader before they start:**

```markdown
# Feature X

> **Audience:** Implementation, Support
> **Purpose:** Setup guide

For platform admins and Implementation engineers configuring X — no code reading required.

## Overview

...
```

The audience is a focus aid for _you, the writer_. Once published, this doc ends up in Notion, wikis, and search, where anyone in the company can read it. A fixed audience label tells some readers "this is not for you" without making the doc better for the intended readers. Remove the tagline and the frontmatter; the title and the Overview are enough.

## Template

> Default structure for repos with no existing docs. If the repo already has documentation, match its style instead — folder location, heading style, tone, amount of code, and any metadata. Remove sections from this template that are not worth keeping for the current reader and purpose.

```markdown
# <Feature or system name>

<One- to three-sentence intro: a plain statement of what this doc explains and what the feature does. No audience label, no purpose tag, no frontmatter — just the intro a Notion reader expects.>

## Overview

<Plain-language explanation of what the feature does, when it matters, and how it differs from related features. This is the section most readers actually read; make it count.>

## How it works

<Numbered workflow in plain steps. One step per real action. Include timing, branches, and who/what is involved.>

1. <Trigger — the event or action that starts the flow>
2. <Next step — what happens, where, who sees it>
3. <…>

<Add a Mermaid diagram if the flow has branches or parallel paths.>

## How to set it up / deliver it to a client

<If the reader's job is to configure, enable, or hand something to a client: step-by-step setup with the exact snippets, settings, or templates they need. Keep code blocks scoped to what the reader actually pastes or sends.>

## When it applies (and when it doesn't)

<Eligibility, scope, plan tiers, and explicit out-of-scope cases.>

## Business rules and configuration

<What is configurable, by whom, with what effect. State each rule and what it means in practice.>

| Setting / rule       | What it controls        | Default | Who can change it |
| -------------------- | ----------------------- | ------- | ----------------- |
| <name in plain text> | <plain-language effect> | <value> | <role>            |

## Product context

<How this feature fits the product: roadmap position, related features, known limitations, upcoming changes. Include only what you know confidently from the user or visible repo state; omit the section entirely if you can't fill it without guessing.>

## Roles and responsibilities

<Who configures, who monitors, who is paged, who approves changes. A RACI-style table works well when ownership crosses teams.>

## Examples

<1–3 worked scenarios. Use realistic but generic data. Show the reader what "normal" and "edge" look like.>

## Edge cases and failure modes

<What can go wrong, what the system does automatically, and what a human needs to do. Each entry: trigger → system behavior → human action.>

## Troubleshooting / FAQ

<Questions the audience actually asks. Avoid filler. If you can't think of a real question, don't invent one.>

## Glossary

<Terms specific to this system. Plain-language definitions. Skip if no jargon was used.>

## Related docs

<Links to adjacent docs the reader may need: API reference for developers, customer-facing help articles, runbooks for other teams.>

## Technical reference (optional)

<Stable named things the reader handles in their job: key files and what they do, configuration field paths, named services, payload shapes, schema fields. Use a table when listing more than 3 items. Skip this section if there is nothing of this kind worth surfacing.>

| File / config path / service | Role / what it controls   |
| ---------------------------- | ------------------------- |
| `path/to/file.py`            | <one-line role>           |
| `config.some_setting`        | <what the field controls> |
```
