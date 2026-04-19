---
name: report-app-down
description: Diagnostic-only deep status report for a Dokploy app. Engineering-facing — pulls status + deploy-history + runtime-logs and presents a structured summary with the raw error messages, stack traces and run URLs you'd need to escalate or write a postmortem. Does NOT take any actions. Use when the engineer says "diagnose X", "what's wrong with X", "why is X failing", "give me the full picture on X" — or when fix-app escalates here.
---

# Report an app down

Engineering-facing diagnostic. Mirror of the operator's
`report-site-down` but technical and exhaustive — surfaces every
piece of context a debugging engineer (or an oncall taking the
escalation) needs in one place.

This skill **never takes actions**. For fixes, use **fix-app**.
For start/stop, use the dedicated skills. For onboarding, use
**onboard-app** / **move-app**.

## Inputs

> "App identifier (snake_case name, Dokploy appName, or primary domain):"

Optional context the engineer wants to include in the report:

> "Any extra context (e.g. 'started failing after the AUTHKIT_DOMAIN
> change at 14:30')? [enter to skip]:"

## Pull everything in parallel

Dispatch the three actions concurrently — each is independent.

```
# 1. status
actions_run_trigger({
  method: "run_workflow", owner: "PromakerDev", repo: "managed-apps-workflows",
  workflow_id: "manage-app.yml", ref: "main",
  inputs: { action: "status", app: "<input>" }
})

# 2. deploy-history (limit 10 — wider window than fix-app uses)
actions_run_trigger({
  method: "run_workflow", owner: "PromakerDev", repo: "managed-apps-workflows",
  workflow_id: "manage-app.yml", ref: "main",
  inputs: { action: "deploy-history", app: "<input>", history_limit: "10" }
})

# 3. runtime-logs (tail 300 — bigger window than fix-app uses)
actions_run_trigger({
  method: "run_workflow", owner: "PromakerDev", repo: "managed-apps-workflows",
  workflow_id: "manage-app.yml", ref: "main",
  inputs: { action: "runtime-logs", app: "<input>", tail: "300", since: "all" }
})
```

Wait for all three to complete (`actions_get` per run id). If
`status` fails at the Resolve step, the others will too — short-
circuit and report "App not found".

## Read each report

- `status`: parse `APP_STATUS_JSON` log group via `get_job_logs`.
- `deploy-history`: parse the markdown table via `get_job_logs`
  (look for `## Deploy history`); also note any
  `::warning title=Failed deploy` annotations.
- `runtime-logs`: full content via `get_job_logs`
  (`return_content: true`). Scan the **last 100 lines** for
  tracebacks, OOM, repeated ERRORs, port-bind failures.

## Synthesize the report

Output a single block, technical, no fluff:

```
═══════════════════════════════════════════
 App diagnostic: <name>
═══════════════════════════════════════════

Domain(s):     <domains>
HTTP probe:    <http_status> on https://<primary>
Dokploy state: <state>
Last deploy:   <last_deploy_status>  (<startedAt>)
              <one-line errorMessage if any>

Operator note: "<context provided>"   ← only if non-empty


─── Deploy history (last 10) ────────────────
| Started | Finished | Status | Duration | Error |
| ...     | ...      | ...    | ...      | ...   |

Pattern: <X of last 10 errored | all done | flaking>


─── Runtime tail (last 100 lines, last fresh issues highlighted) ────
<paste the highlighted lines verbatim — engineers read raw>

Detected:
  - <pattern, e.g. "Python KeyError on AUTHKIT_DOMAIN at startup">
  - <pattern, e.g. "uvicorn restart loop, 5 in last minute">


─── Workflow run URLs ────────────────────────
status:         <html_url>
deploy-history: <html_url>
runtime-logs:   <html_url>


─── Suggested next step ────────────────────
<one of:>
  ▸ Try fix-app — symptoms match a known-fixable pattern
    (Traefik 404 / transient build / single 502).
  ▸ Manual intervention required — <reason>. Suggested action:
    <e.g. "fix the AUTHKIT_DOMAIN env var in Dokploy then retry">.
  ▸ Escalate to platform team — <reason>.
═══════════════════════════════════════════
```

## Pattern recognition heuristics

Use these to populate the "Detected" + "Suggested next step"
sections. They're not authoritative — annotate as such if uncertain.

| Symptom | Likely cause | Next step suggestion |
|---|---|---|
| http=502 + runtime traceback on startup | App crashes early, missing env / bad config | Manual: fix env var or config, then `fix-app` |
| http=502 + runtime "OOMKilled" / "Killed" | Memory pressure | Manual: increase Dokploy memory limit |
| http=502 + runtime "EADDRINUSE" or `bind: address already in use` | Port conflict between PORT env and domain port | Manual: align Dokploy domain port with the app's PORT |
| http=404 (Traefik) + state=done | Traefik router dropped after deploy | `fix-app` (redeploy refreshes Traefik) |
| http=200 but deploy-history shows ERROR latest | Stale build serving — Dokploy fell back | `fix-app` or `manage-app redeploy` |
| ≥3 of last 10 deploys errored same message | Systemic — code or config issue | Manual: read errorMessage, fix upstream |
| All deploys done, http=200, runtime quiet | Already healthy | "No-op — site is fine. Re-test." |
| state=idle + last_deploy=none | Never deployed | `onboard-app` follow-up redeploy |

## Boundaries

- **Read-only.** Never dispatch start / stop / redeploy / scaffold /
  move from this skill. If the diagnosis suggests one of those,
  point the engineer at the right skill instead.
- Don't withhold raw logs / tracebacks — engineers read them
  natively, that's the point.
- If `runtime-logs` returns thousands of lines and 300 isn't
  enough, mention it: "tail=300 only; bump in manage-app or
  re-run report-app-down with more if needed".
