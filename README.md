# Intuit Plugins

Official Intuit marketplace for AI coding agent plugins (Claude Code, Codex, and Cursor). Each plugin lives in its own folder under `plugins/`.

## Plugins

| Plugin | Description |
| --- | --- |
| [intuit-developer-plugin](plugins/intuit-developer-plugin) | Build QuickBooks integrations with Intuit Developer Platform: workspace and app setup, sandbox management, browser OAuth, and verified QuickBooks API calls. |

## Install (Claude Code)

```
/plugin marketplace add intuit/intuit-plugins-official
/plugin install intuit-developer-plugin@intuit-marketplace
```

## Layout

```
.claude-plugin/marketplace.json  # Claude Code marketplace catalog
.agents/plugins/marketplace.json # Codex marketplace catalog
plugins/
  <plugin-name>/
    .claude-plugin/plugin.json   # Claude Code manifest
    .codex-plugin/plugin.json    # Codex manifest
    .mcp.json                    # MCP servers (optional)
    skills/                      # Skills shipped with the plugin
```

## License

[Apache 2.0](LICENSE)
