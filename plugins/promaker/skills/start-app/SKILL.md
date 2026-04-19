---
name: start-app
description: Start a stopped Dokploy app via manage-app's start action. Engineering-facing — minimal wrapper. Use when the engineer says "start X", "bring X back up", "X was stopped, restart it". Not for first-time deploys (use onboard-app or fix-app for that — application.start can't bootstrap an image-less app).
---

# Start an app

Engineering wrapper around `manage-app.yml action=start`. Minimal —
this skill exists so you can say "start mcp_proxy_github" instead of
typing the full `gh workflow run` command.

## Caveat

`application.start` only restarts an existing container. For an app
whose first build never ran (`deploy_on_create=false` was used, or
the first deploy errored and was never retried), use **fix-app** or
**onboard-app**'s follow-up redeploy instead.

If `manage-app status` shows `last_deploy_status=none`, this skill
will refuse and redirect.

## Inputs

> "App identifier (snake_case name, Dokploy appName, or primary domain):"

## Confirm

```
Start: <input>?  [y/N]
```

## Dispatch

```
actions_run_trigger({
  method: "run_workflow",
  owner: "PromakerDev",
  repo: "managed-apps-workflows",
  workflow_id: "manage-app.yml",
  ref: "main",
  inputs: { action: "start", app: "<input>" }
})
```

Poll `actions_get` (`get_workflow_run`) until completed.

## Final

### success

```
✓ start dispatched against <input>.
Give the container ~10-20s to come up, then `curl <domain>` or use
`fix-app` if it still 502s.
```

### failure (Resolve step)

```
✗ App not found: <input>. Check Dokploy UI or api key scope.
```

### failure (other)

```
✗ start workflow failed: <run html_url>
```

## Boundaries

- Don't use this for first-time deploys. Detect via
  `last_deploy_status=none` in a quick prior `manage-app status`
  call (optional pre-check) and redirect to fix-app.
- Pair with **stop-app** for the inverse.
