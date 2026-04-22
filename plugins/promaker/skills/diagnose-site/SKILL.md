---
name: diagnose-site
description: Deep technical diagnostic for a Promaker managed site. Engineer-facing — resolves the site, checks its onboarding, pulls deployments, build logs and runtime logs via the Promaker GitHub MCP and synthesises a structured report. Diagnostic only, never takes actions. Use when the engineer says "diagnose site X", "why is site X failing", "give me the full picture on site X", "check build logs for X", "check runtime logs for X".
---

# Diagnose a site (engineer-facing)

Technical diagnostic for a Promaker managed site. Mirror of
`report-app-down` but for **sites** and built on the Promaker GitHub
MCP (direct Dokploy / Terraform state access) instead of only the
GitHub Actions workflows.

This skill **never takes actions**. For self-healing fixes, use
`fix-site`. For the non-technical operator-facing version, use
`report-site-down`.

## Inputs

> "Site identifier (snake_case name, primary domain, or appName):"

Optional extra context:

> "Any extra context (e.g. 'started failing right after the 14:30
> deploy')? [enter to skip]:"

## Step 1 — Resolve the site

Use the Promaker GitHub MCP tool `promaker_list_resources` (part of
the `promaker-github` MCP exposed by this plugin) to find the
matching site and its `application_id`.

```
promaker_list_resources({
  kind:   "site",
  search: "<raw input>"
})
```

The `search` param does case-insensitive substring matching against
`name`, `owner` and `primary_url`. Pick the entry that matches
exactly or is the single obvious hit. If multiple entries match,
list them back and ask which one:

> "Hay varios sitios que coinciden con **<input>**:
>  - <name_a> (<primary_url>, owner=<owner>, env=<environment>)
>  - <name_b> (...)
> ¿Cuál querés diagnosticar?"

If nothing matches, report "Site not found — no resource in either
managed-sites-infrastructure or managed-apps-infrastructure matches
`<input>`." and stop.

Remember the resolved entry:

- `application_id` — pass to every other `promaker_*` tool
- `name`, `owner` (tenant), `environment`, `primary_url` — for the
  report header

## Step 2 — Pull everything in parallel

Dispatch the three `promaker_*` calls concurrently — they are
independent.

```
# A. recent deployments (status, timestamps, errorMessage per deploy)
promaker_list_deployments({
  application_id: "<id>",
  limit:          10
})

# B. build logs for the most recent deployment
#    (deployment_id omitted → newest by createdAt)
promaker_get_build_logs({
  application_id: "<id>",
  lines:          400,
  strip_progress: true
})

# C. runtime container logs
promaker_get_container_logs({
  application_id: "<id>",
  lines:          300
})
```

Each call is independent — issue all three in parallel in the same
response.

## Step 3 — Check onboarding status (conditional)

Only if the deployment list from step 2 is **empty** (site was never
deployed) or the site looks fresh (primary deploy still running /
missing) — check whether the onboarding pipeline is still in
flight. Dispatch via the official `github/github-mcp-server`:

```
actions_list({
  method:      "list_workflow_runs",
  owner:       "PromakerDev",
  repo:        "managed-sites-onboarder",
  resource_id: "onboard.yml",
  workflow_runs_filter: { per_page: 20, event: "workflow_dispatch" }
})
```

Match runs whose `display_title` or inputs reference the site name
or primary domain. For any match, pull the per-step state via
`actions_list / list_workflow_jobs` and record:

- Run status (`queued` / `in_progress` / `completed`)
- Run conclusion (`success` / `failure` / `cancelled`) when
  completed
- Current or last-failed step name
- `html_url` for the run

Skip this step entirely when the site already has deployments — the
onboarding is by definition done.

## Step 4 — Read each report

### 4a. Deployments list

Expect each entry to carry at least: `deploymentId`, `status`
(`done` / `error` / `running`), `createdAt`, `finishedAt`,
`errorMessage`, `log_path`.

Note:
- Most recent deploy's status + error (if any)
- How many of the last 10 errored → if ≥3 with the same
  `errorMessage`, this is **systemic**, not transient
- Whether a deploy is currently `running` (mid-deploy — surface it
  explicitly so the engineer doesn't act on stale state)

### 4b. Build logs

The MCP already strips progress bars when `strip_progress=true`.
Scan the tail (last ~100 lines) for:

- Docker build failures: `ERROR`, `failed to solve`, `exit code`,
  `returned a non-zero code`
- Package / dependency errors: `npm ERR!`, `pnpm ERR_`,
  `pip install` failures, `go: module ... not found`
- Image pull failures: `failed to pull`, `manifest unknown`,
  `unauthorized`
- Buildx / registry issues: `push access denied`, `registry returned
  error`
- Missing build arg or env: `is not defined`, `required but not
  provided`

Quote the **raw offending lines verbatim**. Engineers read logs
natively — do not paraphrase.

### 4c. Runtime container logs

Scan the last ~100 lines for:

- Python / Node / Go tracebacks
- Crash patterns: `Killed`, `OOMKilled`, `SIGTERM`, `exited with
  code`, segfault
- Config errors: `KeyError`, `missing required env var`, `ENOENT`,
  `EADDRINUSE`
- Framework-specific: uvicorn/gunicorn `Worker failed to boot`,
  Next.js `Error: Cannot find module`, Node `UnhandledPromise-
  Rejection`
- Restart loops — same error message repeating at short intervals

If the container has no runtime logs at all (the MCP returns empty
output) and the deployment status is `done`, that's a signal the
container either crashed immediately or is not routed — flag it.

## Step 5 — Synthesise the report

Emit a single structured block. No fluff, no emoji — engineers read
this as raw context.

```
═══════════════════════════════════════════
 Site diagnostic: <name>
═══════════════════════════════════════════

Owner / tenant: <owner>
Environment:    <environment>
Primary URL:    <primary_url>
application_id: <id>

Operator note: "<context>"   ← only if non-empty


─── Onboarding (only if pulled in step 3) ──
Latest onboarding run: <status> / <conclusion>
  Active / last step:  <step name>
  Run URL:             <html_url>
  Note:                <e.g. "never deployed — onboarding still in
                       Scaffold step">


─── Deployments (last 10) ──────────────────
| # | Started | Finished | Status | Duration | Error (one line) |
| 1 | ...     | ...      | ...    | ...      | ...              |
| ...                                                             |

Pattern: <e.g. "3 of last 5 errored with identical 'npm ERR! ENOENT'
         — systemic" | "all done — healthy" | "latest still running">


─── Build logs (latest deploy, tail highlights) ─────
<paste the raw offending lines verbatim — keep file paths and
line numbers intact>

Detected in build:
  - <pattern, e.g. "Docker COPY failed: file not found
    'public/assets/hero.png'">
  - <pattern, e.g. "pnpm install exited 1 — lockfile out of sync">


─── Runtime logs (tail highlights) ──────────
<paste raw offending lines verbatim>

Detected at runtime:
  - <pattern>
  - <pattern>


─── Suggested next step ────────────────────
<pick ONE>
  ▸ Run `fix-site` — symptoms match a cheap self-healing pattern
    (Traefik 404 / single transient build failure / single 502 with
    no crash loop / stopped container). fix-site will attempt
    start/redeploy automatically.
  ▸ Run `report-site-down` — the cause is severe or needs a human:
    systemic build failures (≥3 of last 10 with same error), crash
    loops with a clear config/code bug, OOM, onboarding failure
    before first deploy, anything where another redeploy won't help.
    report-site-down produces the operator-facing Spanish escalation
    summary.
  ▸ No-op — site is already healthy. Re-test from the client side.
═══════════════════════════════════════════
```

Pick the branch deterministically:

| Situation | Route to |
|---|---|
| stopped / idle container | `fix-site` |
| Traefik 404, state=done | `fix-site` |
| Single transient build/runtime error, prior deploys healthy | `fix-site` |
| ≥3 of last 10 deploys errored with same message | `report-site-down` |
| Runtime crash loop with clear config/code bug | `report-site-down` |
| OOMKilled or resource-exhaustion pattern | `report-site-down` |
| Onboarding still in progress | wait, then re-run `diagnose-site` |
| Onboarding failed before first deploy | `report-site-down` |
| All green, http=200, runtime quiet | no-op |

## Pattern recognition heuristics

Populate the "Detected" and "Suggested next step" sections using
these — annotate as tentative when uncertain.

| Symptom | Likely cause | Next step suggestion |
|---|---|---|
| latest deploy `error` + build log `npm ERR! ENOENT ... package.json` | Wrong workdir / monorepo path | Manual: fix Dokploy `Build Path` or Dockerfile `WORKDIR` |
| build log `failed to solve ... no such file` on `COPY` | Asset committed elsewhere (LFS, `.gitignore`) | Manual: commit the missing file or adjust `.dockerignore` |
| build log `lockfile out of sync` / `pnpm-lock.yaml` mismatch | Lockfile drift | Manual: regen lockfile, then `fix-site` redeploy |
| build log `manifest unknown` / `pull access denied` | Private base image or registry auth | Manual: fix registry creds in Dokploy |
| latest deploy `done` + runtime traceback on startup | App crashes early, missing env / bad config | Manual: fix env var, then `fix-site` |
| latest deploy `done` + runtime `OOMKilled` | Memory pressure | Manual: raise Dokploy memory limit |
| latest deploy `done` + runtime `EADDRINUSE` | PORT env / Dokploy domain port mismatch | Manual: align PORT with domain port |
| latest deploy `done` + runtime quiet + site 404 | Traefik router dropped | `fix-site` (redeploy refreshes Traefik) |
| latest deploy `error` but site responds 200 | Dokploy fell back to prior build | `fix-site` once to retry; surface that the live build is stale |
| ≥3 of last 10 deploys errored with same message | Systemic — code or config | Manual: read errorMessage, fix upstream — do NOT redeploy |
| deployments empty + onboarding run in progress | Still onboarding | Wait — re-run this skill after the run finishes |
| deployments empty + onboarding run `failure` | Onboarding broke before first deploy | Manual: read the onboarding failure, re-dispatch `onboard.yml` |
| all deploys done, runtime quiet, site 200 | Already healthy | No-op — re-test from client side |

## Boundaries

- **Read-only.** Never dispatch start / stop / redeploy / scaffold /
  move from this skill. Point the engineer at the right skill
  (`fix-site`, `start-site`, `stop-site`, `move-site`, `onboard-site`)
  if the diagnosis suggests one.
- Don't withhold raw logs or tracebacks — the whole point is
  surfacing them.
- Use the Promaker GitHub MCP (`promaker_*`) for deployments, build
  logs and runtime logs. Do NOT route those through the
  `manage-site.yml` workflow — it's slower and adds a workflow-run
  indirection when the direct path is available.
- Use the official `github/github-mcp-server` (`actions_*`) only for
  the onboarding-status lookup in step 3.
- If a log window fills up and 300 / 400 lines isn't enough, mention
  it in the report: "tail=300 only; re-run diagnose-site with more
  if needed" — don't silently truncate.
- For internal infrastructure sites (`managed-sites-*`, `codemod-*`,
  `promaker-*`, `mcp-*`): refuse with "That site is platform
  infrastructure — not operated through these skills."
