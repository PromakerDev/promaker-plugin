# Promaker Claude Code plugin

Bundles the Promaker managed-platform skills (sites + apps) together with the Promaker GitHub MCP server so a single install wires up the full operator toolkit.

## What's inside

- **14 skills** for operating Promaker sites and apps:
  - Sites: `onboard-site`, `debug-and-fix-site`, `start-site`, `stop-site`, `move-site`, `report-site-down`, `onboarding-status`
  - Apps: `fix-app`, `onboard-app`, `start-app`, `stop-app`, `move-app`, `report-app-down`, `app-onboarding-status`
- **Promaker GitHub MCP** (`https://mcp-github.pmkr.site/mcp`) exposed as `promaker-github` over HTTP.

## Install

Add this repo as a marketplace and install the plugin:

```
/plugin marketplace add <owner>/promaker-plugin
/plugin install promaker
```

Once installed, skills show up as `promaker:debug-and-fix-site`, `promaker:onboard-site`, etc. The MCP server connects automatically at session start.

## Layout

```
.claude-plugin/
  plugin.json         # plugin manifest
  marketplace.json    # marketplace manifest (single-plugin, source = ".")
.mcp.json             # promaker-github HTTP MCP
skills/               # 14 skills, one directory each with SKILL.md (+ _shared/LANGUAGE.md)
```
