---
name: handover
description: >
  Generates a comprehensive handover document for another AI agent to seamlessly continue work on a task without prior context. Use this when you are asked to "handover", "save state", "switch providers", or prepare a summary for the next AI session.
---

# Agent Handover Protocol

Generates a structured handover file (`tmp/HANDOVER.md`) to capture the current state, goals, and next steps for an incoming AI agent.

## Execution Steps

1. **State Analysis:**
   - Identify the current Git branch using git commands.
   - Summarize uncommitted changes and recent commits related to the task.
   - Identify failing tests, unresolved edge cases, or compiler warnings.
2. **Stack Identification:**
   - Briefly identify primary languages and frameworks (e.g., by checking `package.json`, `requirements.txt`, etc.).
   - Identify the exact terminal commands required to run tests or start the local server.
3. **Document Generation:**
   - Read the template from [assets/handover-template.md](assets/handover-template.md).
   - Create or overwrite `tmp/HANDOVER.md` in the project root by filling in the template with the gathered context. Ensure `tmp/` exists and is in `.gitignore` as per global conventions.
4. **Output:**
   - Silently write the `tmp/HANDOVER.md` file.
   - Reply to the user: "Handover document generated successfully in `tmp/HANDOVER.md`. State saved for the next agent."
