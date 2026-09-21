# DemoHub plugin

Gives Claude Code and Codex tools to work with unit8 [DemoHub](https://demo.shnei.de)
demos directly from the agent — list, create, verify, build, start, and stop — without
clicking through the portal.

## Setup (one-time, per person)

1. In DemoHub, go to **Agent tokens** (https://demo.shnei.de/agent-tokens) and create a
   token. Pick scopes for what you want an agent to do unattended — `demos:read` alone is
   enough to check status; add `demos:create`/`demos:start`/`demos:stop` if you want it
   provisioning things. The token is shown exactly once — copy it immediately.
2. Export it wherever your shell picks up env vars (`~/.zshrc`, direnv, etc.):
   ```
   export DEMOHUB_AGENT_TOKEN="dhat_..."
   ```
3. Open a new shell, then install the plugin (see the [repo README](../../README.md)).
   Both Claude Code and Codex ask you to approve/trust the plugin's MCP config the first
   time — that's expected.

Revoke the token any time from the **Agent tokens** page in DemoHub — that immediately
cuts off every agent using it.

## Tools

`list_demos`, `get_demo`, `get_status`, `create_demo`, `update_demo`, `delete_demo`,
`verify_deploy_key`, `get_deploy_key`, `build`, `get_build`, `start`, `stop`, `restart`,
`touch`, `get_logs`, `get_env`, `set_env`, `clear_data`, `get_current_user` — each
requires the token to carry the matching scope.

## Files

- `.claude-plugin/plugin.json` — Claude Code plugin manifest
- `.mcp.json` — MCP server config for Claude Code (auto-loaded from the plugin root)
- `.codex-plugin/plugin.json` — Codex plugin manifest
- `mcp.codex.json` — MCP server config for Codex (referenced from the Codex manifest)
