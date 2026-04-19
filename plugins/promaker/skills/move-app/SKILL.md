---
name: move-app
description: Re-home a Dokploy app from one project to another in managed-apps-infrastructure. Engineering-facing — collects app_name, from_project, to_project and an optional flag to decommission the source project if it ends up empty, then dispatches the move-app workflow that opens a Terraform PR with `moved {}` blocks for state re-parenting. Use when an engineer asks to move an app, re-home an app, change an app's project, or decommission an empty project.
---

# Move an app between projects

Engineering-facing. Mirror of the operator's `move-site` skill but
direct, technical English, no language filtering. The companion
skills are `onboard-app`, `fix-app`, `start-app`, `stop-app`,
`report-app-down`.

## Required inputs

### 1. app_name (snake_case)

> "App to move:"

Validate `^[a-z][a-z0-9_]*$`.

### 2. from_project

> "Current project:"

Validate `^[a-z][a-z0-9_]*$`.

### 3. to_project

> "Target project (auto-created if missing):"

Validate, reject if equal to `from_project`.

### 4. delete_empty_project

> "Decommission `<from_project>` if it ends up empty after the move?
> [Y/n]:"

Default true.

### 5. auto_merge

> "Auto-merge once `terraform plan` passes? [Y/n]:"

Default true.

## What this actually does

Brief the engineer once before dispatching:

```
The workflow will open a single infra PR that:

  1. mv apps/<from>/<app>.tf  →  apps/<to>/<app>.tf
  2. Bootstrap apps/<to>/ if missing (modules/project + apps_<to>)
  3. Regenerate apps/<from>/outputs.tf and apps/<to>/outputs.tf
  4. Append a `moved {}` block to moved.tf so terraform re-parents
     state for module.apps_<from>.module.<app>  →  module.apps_<to>.module.<app>
     — GitHub env / variables / secrets / reCAPTCHA key survive
     because their resource keys don't depend on environment_id
  5. The dokploy_application + domains DO replace because their
     environment_id is ForceNew (different project = different env)
  6. If delete_empty_project=true and from_project becomes empty,
     remove apps/<from>/, the project_<from> module, the
     apps_<from> module, and the outputs block — terraform apply
     will then destroy the Dokploy project and its 3 environments.

Proceed? [y/N]:
```

Only on yes.

## Dispatch

```
actions_run_trigger({
  method:      "run_workflow",
  owner:       "PromakerDev",
  repo:        "managed-apps-workflows",
  workflow_id: "move-app.yml",
  ref:         "main",
  inputs: {
    app_name:             "<app>",
    from_project:         "<from>",
    to_project:           "<to>",
    delete_empty_project: "true" | "false",
    auto_merge:           "true" | "false"
  }
})
```

## Track

`actions_list` (`list_workflow_runs`, `resource_id="move-app.yml"`)
to find the run, then poll `actions_get` (`get_workflow_run`) every
10s. Pull the PR URL from `get_job_logs` (grep `Opened PR:`).

## Final

### success

```
✓ move-app completed
  app:           <app_name>
  from project:  <from>
  to project:    <to>  (<bootstrapped|existing>)
  cleanup from?  <yes|no>
  infra PR:      <infra_pr_url>  (auto-merge on plan pass)

terraform apply will replace dokploy_application + domains; expect
~1-3 min downtime as the build re-runs in the new project. Use
fix-app afterwards if the new app comes up unhealthy.
```

### failure

```
✗ move-app failed at step "<step_name>"
  Run: <html_url>

Common causes:
  - apps/<from>/<app>.tf doesn't exist (wrong app_name?)
  - apps/<to>/<app>.tf already exists (target taken)
  - PAT can't write to managed-apps-infrastructure

If the script crashed mid-way, the infra repo might have orphan
files. Inspect and clean up manually before retrying.
```

### cancelled

```
~ move-app cancelled.
```

## Boundaries

- Single-app moves only. If the engineer wants to move multiple,
  loop the skill — each move opens its own PR.
- Don't propose moves between sites and apps — different infra
  repos, different SA, different state buckets.
- For the source project: cleanup happens automatically when
  `delete_empty_project=true` AND no other apps live in
  `apps/<from>/`. The skill does not enumerate apps to confirm —
  the workflow does. Trust the workflow.
