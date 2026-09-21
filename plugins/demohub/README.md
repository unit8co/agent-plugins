# DemoHub plugin

Gives Claude Code and Codex tools to work with unit8 [DemoHub](https://demo.unit8.io)
demos directly from the agent — list, create, verify, build, start, and stop — without
clicking through the portal.

## Setup

Nothing to provision. The first time the agent calls a DemoHub tool, it opens a Google
OAuth login in your browser — sign in with your `@unit8.co` account, same as the portal.
There's no token to mint, copy, or rotate; the MCP server authenticates the agent as you.
Both Claude Code and Codex also ask you to approve/trust the plugin's MCP config the
first time — that's expected.

The demos an agent creates or touches are owned by whichever account signed in, and
owner/admin restrictions (who can edit settings, env vars, or delete a demo) are the
same ones the portal enforces.

## Tools

`list_demos`, `get_demo`, `get_status`, `create_demo`, `update_demo`, `delete_demo`,
`verify_deploy_key`, `get_deploy_key`, `build`, `get_build`, `start`, `stop`, `restart`,
`touch`, `get_logs`, `get_env`, `set_env`, `clear_data`, `get_current_user`.

## Files

- `.claude-plugin/plugin.json` — Claude Code plugin manifest
- `.mcp.json` — MCP server config for Claude Code (auto-loaded from the plugin root)
- `.codex-plugin/plugin.json` — Codex plugin manifest
- `mcp.codex.json` — MCP server config for Codex (referenced from the Codex manifest)
