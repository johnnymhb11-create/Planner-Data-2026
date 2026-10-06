---
name: power-automate-flows
description: Build, edit, debug and troubleshoot Power Automate cloud flows (definition.json, triggers, actions, expressions, connections, failed runs). Use when working with flow JSON (e.g. Flows/*/definition.json), Power Automate errors, SharePoint/Planner/Outlook/Approvals actions, or when the user asks to create, fix, diagnose or manage a flow. Adapted from microsoft/power-platform-skills (MIT).
---

# Power Automate flows

Offline reference bundle. Works without the FlowAgent MCP server: read and edit flow JSON in the repo and guide the user on the portal steps.

## How to use

1. Before writing or editing a flow definition, read `references/definition-reference.md`.
2. For connection or auth problems, read `references/connection-patterns.md`.
3. For any error message or failed run, search `references/error-troubleshooting.md` first.
4. `references/cli-reference.md` lists the FlowAgent CLI commands (only usable if the CLI is installed locally).
5. Task playbooks are in `references/workflows/` — pick by intent:
   - New flow: `build-flow.md` (autonomous), `create-flow.md` (guided)
   - Failed run: `debug-flow.md`, `diagnose-flow.md`
   - Lifecycle / inventory: `manage-flows.md`, `browse-flows.md`, `manage-desktop-flows.md`
   - Environments and setup: `route-environments.md`, `setup.md`

## Caveats

- The workflow files call `mcp__flowagent__*` tools. Without the FlowAgent MCP server (full plugin: `/plugin install power-automate@power-platform-skills` in local Claude Code) skip those calls and apply the same steps by editing the JSON and asking the user to run/inspect in the portal.
- Ignore the `allowed-tools` and `model` front matter in the workflow files; they only apply to the original plugin.
- For Dataverse solution packaging or Model Driven App command-bar buttons, prefer the `power-automate-flow-builder`, `mda-flow-button` and `powerapps` skills if available.
- Never edit `definition.json` without validating that every `runAfter`, `@{...}` reference and `$connections` entry still resolves.
