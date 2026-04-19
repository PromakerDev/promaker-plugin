---
name: stop-app
description: Stop a running Dokploy app via manage-app's stop action. Engineering-facing — takes the app off-traffic until restarted. Use when the engineer says "stop X", "take X offline", "shut down X", "pause X". The app stays scaffolded and configured; only the container stops.
---

# Stop an app

Engineering wrapper around `manage-app.yml action=stop`. The
container stops; the Dokploy app, env vars, domains and Terraform
state stay intact — engineers can `start-app` to bring it back, or
`fix-app` if `start` doesn't suffice.

## Inputs

> "App identifier (snake_case name, Dokploy appName, or primary domain):"

## Confirm — emphasize the impact

```
Stop <input>?

The container will stop accepting requests. Anything hitting the
domain will get a connection error or 502 until you start-app again.
Use stop-app for maintenance windows or to free VPS resources.

  [y/N]:
```

Only on explicit yes.

## Dispatch

```
actions_run_trigger({
  method: "run_workflow",
  owner: "PromakerDev",
  repo: "managed-apps-workflows",
  workflow_id: "manage-app.yml",
  ref: "main",
  inputs: { action: "stop", app: "<input>" }
})
```

Poll `actions_get` until completed.

## Final

### success

```
✓ stop dispatched against <input>. Container is offline.
Bring it back: start-app <input>  (or fix-app if start alone won't recover it).
```

### failure (Resolve step)

```
✗ App not found: <input>.
```

### failure (other)

```
✗ stop workflow failed: <run html_url>
```

## Boundaries

- DO NOT stop platform infra apps without explicit context. Apps
  whose name or domain matches `mcp-*`, `managed-*`, `codemod-*`,
  `traefik-*`, `dokploy-*`: refuse with a hint that platform infra
  shouldn't be stopped via this skill — use Dokploy UI directly with
  full intent.
- Don't propose stopping prod apps as a debugging step. Engineers
  should reach for `fix-app` or `report-app-down` first.
