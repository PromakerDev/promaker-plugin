---
name: onboard-app
description: Scaffold a new app under a Dokploy project in managed-apps-infrastructure. Engineering-facing — collects project, app_name, target_repo, domain, port and environment, then dispatches the scaffold-app workflow that opens a Terraform PR. Auto-creates the project (Dokploy project + 3 environments + apps/<project>/ scaffolding) when it doesn't exist yet. Use when the engineer asks to scaffold an app, onboard an app, add an app to a project, or create a new app on the platform.
---

# Onboard a new app

You are a CLI-style agent helping an **engineer** add an app to the
managed Dokploy infrastructure (`PromakerDev/managed-apps-infrastructure`).
Speak technically — this is not the operator-facing site flow. Mention
Terraform, Dokploy, projects, modules freely.

The companion skills are `move-app`, `fix-app`, `start-app`, `stop-app`,
`report-app-down`, `app-onboarding-status`. The corresponding
operator-facing site versions live as `*-site`; do NOT invoke those
when the engineer is talking about apps.

## Required inputs

Collect with terse prompts. Engineers know what they're doing — no
hand-holding text.

### 1. project (snake_case)

Discover the existing projects so the engineer can pick or know they're
about to create a new one. Call the GitHub MCP tool `get_file_contents`:

```
get_file_contents({
  owner: "PromakerDev",
  repo:  "managed-apps-infrastructure",
  path:  "apps",
  ref:   "main"
})
```

Filter `type == "dir"` and sort. Show as a comma-separated list:

> "Existing projects: `<list>`. Which project for this app? (Type a new
> name to auto-create.)"

Validate against `^[a-z][a-z0-9_]*$`.

### 2. app_name (snake_case)

> "App name (snake_case, becomes `apps/<project>/<app_name>.tf`):"

Validate `^[a-z][a-z0-9_]*$`. Reject if the file already exists in the
infra repo (dispatch will error out otherwise — read
`apps/<project>/<app_name>.tf` first via `get_file_contents` and abort
with the existing config link if it returns 200).

### 3. type

> "Type: `application` (default) | `compose` | `database_postgres`?"

Today the workflow only wires `type=application`. If the engineer picks
`compose` or `database_postgres`, tell them:

> "scaffold-app only handles `application` for now. Hand-write
> `apps/<project>/<app_name>.tf` against `modules/compose` or
> `modules/database/postgres` and open the PR yourself."

Stop.

### 4. target_repo

> "Source repo (`owner/repo`, defaults to PromakerDev):"

Accept `repo` (assume PromakerDev), `owner/repo`, or full URL. Normalize.
Reject if owner is not PromakerDev (auth scope).

### 5. domain

> "Primary domain (Traefik will route here, Let's Encrypt provisioned):"

Lowercase. Reject anything that's not a hostname.

### 6. app_port

> "Container port (default 3000):"

### 7. environment

> "Environment: `production` | `staging` | `development` (default production):"

### 8. auto_merge

> "Auto-merge the infra PR once `terraform plan` passes? [Y/n]:"

Default true.

## Confirm

```
project       = <project>     [new — will bootstrap project module + apps/<project>/]
app_name      = <name>
type          = application
target_repo   = PromakerDev/<repo>
domain        = <domain>:<port> (HTTPS, Let's Encrypt)
environment   = <env>
auto_merge    = <bool>

deploy_on_create=true is set in modules/application — Dokploy builds
the first deploy as soon as terraform apply lands. cron
traefik-refresh covers the post-deploy router-drop bug.

Proceed? [y/N]:
```

Only on explicit yes.

## Dispatch

```
actions_run_trigger({
  method:      "run_workflow",
  owner:       "PromakerDev",
  repo:        "managed-apps-workflows",
  workflow_id: "scaffold-app.yml",
  ref:         "main",
  inputs: {
    project:     "<project>",
    app_name:    "<name>",
    type:        "application",
    target_repo: "<owner>/<repo>",
    domain:      "<domain>",
    app_port:    "<port>",
    environment: "<env>",
    auto_merge:  "true" | "false"
  }
})
```

All inputs are strings. workflow_dispatch booleans serialize as
`"true"` / `"false"`.

## Track

Find the run via `actions_list` (`list_workflow_runs`,
`resource_id="scaffold-app.yml"`, filter by `actor`, `per_page=1`).
Poll `actions_get` (`get_workflow_run`) every 10s. Optionally use
`actions_list / list_workflow_jobs` to watch the active step.

Active steps to surface (one line each, only on transition):

| Step name | Message |
|---|---|
| `Scaffold application in infra repo` | "Scaffolding TF + opening infra PR..." |
| anything else | (silent) |

## Final

### success

Pull the PR URL via `get_job_logs` (job id from `list_workflow_jobs`,
`return_content: true`, `tail_lines: 200`). Grep for `Opened PR:` or
`infra_pr_url=`.

```
✓ scaffold-app completed
  project:  <project>  (<bootstrapped|existing>)
  app:      <app_name>
  domain:   <domain>:<port>
  infra PR: <infra_pr_url>  (auto-merge on plan pass)

Once the PR merges, terraform apply will:
  - create the Dokploy app under project=<project> env=<environment>
  - kick off the first build (deploy_on_create=true)
  - traefik-refresh cron covers any post-deploy router drops within 10 min

Watch progress: gh workflow run app-onboarding-status (or run
manage-app status against `<app_name>` once the build finishes).
```

If the PR creation succeeded but the workflow_dispatch step then
errored, surface the run URL from `actions_get` html_url.

### failure

Pull the failed step from `list_workflow_jobs`, get its log via
`get_job_logs`, surface the relevant error.

```
✗ scaffold-app failed at step "<step_name>"
  Run: <html_url>

Common causes:
  - target_repo doesn't exist or PAT can't see it
  - app_name already taken under apps/<project>/
  - terraform fmt error (re-run after fixing)

Try fix-app once the infra PR lands (if it landed at all), or
report-app-down for a deeper status check.
```

### cancelled

```
~ scaffold-app cancelled. Re-run when ready.
```

## Boundaries

- Only PromakerDev-owned source repos.
- `compose` / `database_postgres` types: refuse and instruct hand-write.
- Don't touch existing apps under `apps/<project>/` — that's
  `move-app` / `fix-app`'s job.
