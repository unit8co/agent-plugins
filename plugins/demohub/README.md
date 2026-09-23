# DemoHub plugin

Gives Claude Code and Codex tools to work with unit8 [DemoHub](https://demo.unit8.io)
demos directly from the agent — list, create, verify, build, start, and stop — without
clicking through the portal.

## Setup

Nothing to provision. The first time the agent calls a DemoHub tool, it opens an Okta
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

The MCP server itself also carries top-level instructions describing the
create → deploy-key → verify → start sequence, so any MCP client gets that workflow even
without this plugin's skill below.

## `/demo-init` — get a repo ready to deploy

`skills/demo-init/` ships alongside the MCP config and is auto-discovered by both Claude
Code and Codex (Codex plugins don't support bundled slash-commands, only skills — this is
the one artifact that works the same way in both). Run `/demo-init` in a repo you want on
DemoHub, or just ask to "deploy this to DemoHub" / "set up a demo for this repo" — the
skill's description triggers it either way. It:

1. Checks what already exists, both via the MCP (`list_demos`/`get_demo` for this repo)
   and on disk (`Dockerfile`/`demo.yaml`), and resumes from wherever that leaves off.
2. Always targets the single-container shape (`Dockerfile` + optional `demo.yaml`) —
   doesn't ask the user to pick a build shape.
3. Scaffolds whichever files are missing — never overwrites anything you already have.
4. Calls `create_demo` (asking for a name only if one isn't obvious), adds the returned
   deploy key to the GitHub repo (via `gh repo deploy-key add` if `gh` is authenticated,
   otherwise prints it for you to paste in), then `verify_deploy_key`.
5. Builds and starts the demo, polling the build and checking status/logs until it's
   confirmed live at `<slug>.demo.unit8.io` — it doesn't stop at "registered."

Full walkthrough this plugin is the fast path through:
[`docs/CREATING_A_DEMO.md`](https://github.com/unit8co/demohub/blob/main/docs/CREATING_A_DEMO.md)
in `unit8co/demohub`.

## Files

- `.claude-plugin/plugin.json` — Claude Code plugin manifest
- `.mcp.json` — MCP server config for Claude Code (auto-loaded from the plugin root)
- `.codex-plugin/plugin.json` — Codex plugin manifest
- `mcp.codex.json` — MCP server config for Codex (referenced from the Codex manifest)
- `skills/demo-init/SKILL.md` — the `/demo-init` skill, auto-discovered by both tools
