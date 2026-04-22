---
name: fix-app
description: Diagnose a Dokploy app that is failing or returning errors and try to fix it automatically (start, redeploy, refresh routing). Engineering-facing — pulls status + deploy-history + runtime-logs as needed, surfaces raw error messages and stack traces, proposes the cheapest fix, applies it, then deep-verifies (latest deploy `done` + runtime quiet) before declaring success. Use when the engineer says "fix X", "X is down", "X is 502", "X 404s", "X won't start", "diagnose X".
---

# Fix a broken app

Engineering-facing diagnose + recovery loop for apps in
`managed-apps-infrastructure`. Counterpart to the sites suite's
`debug-and-fix-site`, but engineer-tone (quote raw error messages,
mention Dokploy / Traefik / containers freely, no Spanish).

The companion skills are `onboard-app`, `move-app`, `start-app`,
`stop-app`, `report-app-down`, `app-onboarding-status`.

## When to use this vs report-app-down

- **fix-app** (this one): you want the problem resolved. Acts on
  the app (start, redeploy, refresh routing).
- **report-app-down**: you only want the diagnosis. No actions.

## Inputs

> "App identifier (snake_case name, Dokploy appName, or primary domain):"

Pass verbatim — `manage-app.yml` resolves it server-side.

## Step 1 — Initial status (cheap probe)

```
actions_run_trigger({
  method: "run_workflow",
  owner: "PromakerDev",
  repo: "managed-apps-workflows",
  workflow_id: "manage-app.yml",
  ref: "main",
  inputs: { action: "status", app: "<input>" }
})
```

Wait for completion. Parse the `APP_STATUS_JSON` log group via
`get_job_logs`:

```json
{
  "name": "...",
  "state": "done | running | idle | error | unknown",
  "last_deploy_status": "done | error | running | none",
  "last_deploy_error": "...",
  "domains": ["..."],
  "http_status": "200 | 404 | 502 | 5xx | 000 | none"
}
```

If `Resolve app to application ID` failed:

```
✗ App not found: <input>
  No match by app_name, Dokploy appName, or domain.
  Either the app doesn't exist or your DOKPLOY_API_KEY can't see
  its project (Dokploy scopes per user). Check Dokploy UI and
  retry with the correct identifier.
```

Stop.

## Step 2 — Gather deeper context (per branch)

Pull the matching action only when needed:

| Symptom | Extra fetch | Why |
|---|---|---|
| `last_deploy_status=error` OR `state=error` | `action=deploy-history` (limit 5) | Real `errorMessage` + systemic-vs-flake check |
| `state=done` + `http=502` | `action=runtime-logs` (tail 200) | Container is up per Dokploy but isn't answering — cause is in stdout/stderr |
| `state=done` + `http=404` body `404 page not found` | none | Known Traefik-router-drop after deploy. Always: redeploy. |
| `state=idle`, `http=000`/`none` | none | Stopped or never built |
| `state=done` + `http=200` | none | Already healthy |
| Anything else | `action=runtime-logs` (tail 100) | Tie-breaker before escalating |

### Reading deploy-history

The action emits a markdown table in the step summary AND warns on
status=error rows. Pull via `get_job_logs`. Extract:

- Most recent `errorMessage`
- Count of `error` rows in the window. **≥3 in a row = systemic** —
  do NOT propose another redeploy; escalate immediately.

### Reading runtime-logs

`get_job_logs` with `return_content: true`. Scan the last 30 lines:

- Tracebacks: `Traceback`, `Error:`, `at /app/...`, `Caused by:`
- Crashes: `Killed`, `OOMKilled`, `exited with code N`
- Config: `KeyError`, `missing required env var`, `ENOENT`,
  `EADDRINUSE`
- Port mismatch: `Address already in use` or `bind: address already
  in use` on a port that doesn't match the Dokploy domain config

Quote the actual offending line(s) raw — engineers can interpret
them.

## Step 3 — Decide and propose

Pick one branch, surface the diagnosis + the proposed fix, get
explicit confirmation (one keystroke is fine — engineers don't need
the full Spanish prose).

### A. Stopped / never built

`state=idle` AND `http ∈ {000, none}`.

```
Diagnose: container is `idle`. last_deploy_status=<status>.
Proposed fix: action=start  (or redeploy if last_deploy_status=none —
  no image to start)
Proceed? [y/N]:
```

Pick `start` if `last_deploy_status` is `done`. Pick `redeploy` if
`none`.

### B. Last deploy errored

`last_deploy_status=error` OR `state=error`. **deploy-history was
already pulled in Step 2.**

If ≥3 of last 5 deploys errored with the same message:

```
Diagnose: SYSTEMIC failure — last <N> deploys errored with:
  > <one-line errorMessage>
Reintentar won't help. Likely a code or config issue. Escalating.

  Last failed deploy run: <html_url from Dokploy UI or scaffold history>
  deploy-history workflow run: <html_url>
```

Hand off to **report-app-down**. Stop.

If 1-2 failed (likely transient):

```
Diagnose: last deploy errored:
  > <one-line errorMessage>
Earlier deploys succeeded — possibly transient. Proposed fix: redeploy.
Proceed? [y/N]:
```

### C. Traefik 404

`state=done` + `http=404` body `404 page not found`.

```
Diagnose: app is running but the Traefik router was dropped after a
deploy (known Dokploy bug; the cron traefik-refresh covers this on
a 10-min cadence). Proposed fix: redeploy (manage-app's redeploy
action also re-fires domain.update post-build).

Proceed? [y/N]:
```

### D. Container crashed (502)

`state=done` + `http=502`. **runtime-logs was already pulled in Step 2.**

If a clear cause was found:

```
Diagnose: container crashes on startup. From runtime-logs:
  > <raw stack trace, 1-3 lines>

Proposed fix: redeploy (sometimes recovers if transient). If it
doesn't, this is a config problem — escalate.
Proceed with redeploy? [y/N — type N to escalate now]:
```

If logs are empty / only INFO:

```
Diagnose: container crashes on startup but logs don't show a clear
cause. Proposed fix: redeploy as a fresh attempt.
Proceed? [y/N]:
```

### E. Already healthy

`state=done` + `http=200`.

```
Diagnose: already healthy. http=200, last deploy=done. No fix needed.

If you saw an error earlier, it might have been transient. Re-run
this skill if you see the issue again.
```

Stop.

### F. 5xx (non-502) or anything else

```
Diagnose: unexpected state. Escalating to deeper diagnostic.
```

Hand off to **report-app-down** with the status JSON, deploy-history
snippet, runtime-logs snippet, run URLs.

## Step 4 — Apply

Dispatch the chosen action via `actions_run_trigger`
(`manage-app.yml`, `action=<chosen>`, `app=<input>`). Poll
`actions_get` (`get_workflow_run`) every 10s. Surface step
transitions briefly:

| Step | Message |
|---|---|
| `Resolve app to application ID` | "Resolving..." |
| `Execute action` (start) | "Starting container..." |
| `Execute action` (redeploy) | "Redeploying (~1-3 min)..." |

If the workflow run itself fails (non-target error):

```
✗ Fix workflow failed: <run html_url>
  Step: <failed_step_name>
  See logs.
```

## Step 5 — Verify (deep, not just HTTP 200)

Wait 15s. Dispatch `action=status` again.

### 5a. Front door

- `state=done` + `http=200` → **5b deep check**
- Same `http_status` as before → no improvement → **6 → same status**
- Different `http_status` → partial → describe the change, decide
  whether it counts as fixed

### 5b. Deep check (only when 5a is HTTP 200)

200 only proves the front door. Confirm:

1. **Latest deploy is `done`**: `action=deploy-history` (limit 3),
   parse the first row's Status column. If it's `error` / `running`,
   Dokploy is serving a previous build — note it.
2. **Runtime is quiet**: `action=runtime-logs` (tail 50,
   `since=2m`), scan for fresh tracebacks / repeated ERROR / OOM /
   `Killed` after the fix landed.

Surface progress while waiting:

```
HTTP 200. Verifying deploy + runtime are clean...
```

### 5b outcomes

- **Both clean** → **6 → fully healthy**
- **Latest deploy errored** (200 from old build) → **6 → working but stale build**
- **Runtime errors** (loop / OOM / crashes) → **6 → responds but unhealthy**

## Step 6 — Final

### Fully healthy

```
✓ Fixed.
  app:           <name>
  http:          200 (<domain>)
  latest deploy: done (<startedAt>, <duration>)
  runtime:       clean over the last 2m

Re-run fix-app if you see the issue again.
```

### Working but stale build

```
⚠ Working but stale build.
  http: 200 (serving previous good build)
  latest deploy: ERROR — <one-line errorMessage>

Recommend: another redeploy to get the fresh code in. Proceed? [y/N]:
```

On yes: dispatch redeploy, repeat Step 5. On no: leave as-is and
note that redeploy is needed.

### Responds but unhealthy

```
⚠ Responds but unhealthy.
  http: 200 (front door OK)
  runtime errors detected (last 2m):
    > <one or two representative log lines>

Probably part of the app surface is broken even though the entry
endpoint replies. Options:
  [r] redeploy (sometimes clears transient errors)
  [e] escalate to report-app-down
Proceed? [r/e]:
```

On `r`: dispatch redeploy, repeat Step 5. If runtime errors persist
after the second redeploy → escalate via **report-app-down**. On
`e`: hand off with the log lines as context.

### Same status after fix

```
✗ No change. http stays at <code>. Escalating to report-app-down.
```

Hand off with full context.

### Different status after fix

```
~ State changed: was http=<old>, now http=<new>.
Tell me if that counts as fixed or describe the new behavior:
```

## Boundaries

- Only safe operations: `start`, `redeploy`. Never `stop` or
  destructive actions from this skill.
- Confirm before each dispatch.
- Hard cap: 2 automatic redeploys per session (the redeploys that
  come out of branches B/C/D + the optional one out of Step 6
  count). After two, escalate without asking for a third.
- Don't propose code/env/domain/config changes — those go to the
  engineer's own workflow, not through this skill.
- Don't try to operate on apps that aren't in
  `managed-apps-infrastructure` (sites are a different skill set).
