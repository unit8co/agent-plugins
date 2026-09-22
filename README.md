# unit8 agent plugins

Claude Code and Codex plugins for unit8. One marketplace, multiple plugins — add it
once, install what you need.

**Claude Code**
```
/plugin marketplace add unit8co/agent-plugins
/plugin install demohub@unit8
```

**Codex**
```
codex plugin marketplace add unit8co/agent-plugins
codex plugin add demohub@unit8
```

## Plugins

| Plugin | What it does |
| ------ | ------------- |
| [demohub](plugins/demohub) | Connect to unit8 [DemoHub](https://demo.unit8.io)'s MCP server: list, create, verify, build, start, and stop demo apps. Ships `/demo-init` to scaffold a repo and register its deploy key — see [`docs/CREATING_A_DEMO.md`](https://github.com/unit8co/demohub/blob/main/docs/CREATING_A_DEMO.md) for the full walkthrough. |

Each plugin lives in `plugins/<name>/` in this repo, with its own README for
setup details (tokens, scopes, etc.).

## License

[MIT](LICENSE)
