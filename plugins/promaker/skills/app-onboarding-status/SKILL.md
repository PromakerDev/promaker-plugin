---
name: app-onboarding-status
description: Report the state of recent or in-progress app onboardings in managed-apps-workflows. Engineering-facing — shows raw run conclusions, step-by-step state, errorMessage from failed deploys. Use when the engineer asks for status of a scaffold-app run, "what's the state of the X onboarding", or when polling progress after a recent dispatch.
---

# App onboarding status

Engineering-facing companion to `onboard-app` and `move-app`. Pulls
recent workflow runs from `managed-apps-workflows` and surfaces what
they're doing without the engineer having to hit `gh run list` in a
loop.

## Inputs

Optional: an app_name or project filter. If omitted, report ALL runs
of `scaffold-app.yml` and `move-app.yml` from the last 24h.

## Query

Use the official `github/github-mcp-server` tools:

```
actions_list({
  method:      "list_workflow_runs",
  owner:       "PromakerDev",
  repo:        "managed-apps-workflows",
  resource_id: "scaffold-app.yml",
  workflow_runs_filter: { per_page: 10, event: "workflow_dispatch" }
})

actions_list({
  method:      "list_workflow_runs",
  owner:       "PromakerDev",
  repo:        "managed-apps-workflows",
  resource_id: "move-app.yml",
  workflow_runs_filter: { per_page: 10, event: "workflow_dispatch" }
})
```

Merge the two lists, sort by `created_at` desc.

For each run that's still `in_progress`, pull the active step:

```
actions_list({
  method:      "list_workflow_jobs",
  owner:       "PromakerDev",
  repo:        "managed-apps-workflows",
  resource_id: "<run_id>"
})
```

Pick the step whose `status == "in_progress"`.

If a filter (app_name / project) was provided, match against the run's
`display_title` or — for finer matching — `get_job_logs` to read the
input echo. Loose match is fine; engineers will recognize their run.

## Output

One row per run. Plain table or compact JSON, whichever fits the
context. Include:

- workflow (`scaffold-app` / `move-app`)
- run id + html_url
- inputs that matter (project, app_name, from/to_project)
- conclusion (`success` / `failure` / `cancelled` / `null`)
- if in_progress: active step name
- if failure: name of failed step + first line of error from
  `get_job_logs` (look for `::error::` or the last non-trace line)
- duration

Example output:

```
Recent app workflow runs:

scaffold-app  24619427318  status=success     promaker/mcp_proxy_github  35s   <url>
scaffold-app  24619789011  status=in_progress  acme/api  step="Scaffold application in infra repo"  <url>
move-app      24618556483  status=failure      api: promakr → promaker  step="Move app in infra repo" — "apps/promakr/ does not exist"  <url>
```

## When the filter matches a single in-progress run

Detail mode — give the engineer everything they'd need without
opening the GitHub UI:

```
scaffold-app  24619789011
  inputs:    project=acme app_name=api type=application
             target_repo=PromakerDev/api domain=api.acme.example
             environment=production auto_merge=true
  status:    in_progress
  active step: Scaffold application in infra repo  (since 1m ago)
  triggered: 2m ago by guillermo
  url:       <html_url>

Next: cron app-onboarding-status will pick up the conclusion when
you re-run, or just `gh run watch <run_id>` directly.
```

## Boundaries

- Read-only. Never trigger workflows from this skill — point to
  `onboard-app` / `move-app` / `fix-app` for actions.
- Don't pull workflow logs unless explicitly needed (a single
  failed-step summary line is enough; the full log link is in
  `html_url`).
