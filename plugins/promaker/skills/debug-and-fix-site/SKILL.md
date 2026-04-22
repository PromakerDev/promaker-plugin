---
name: debug-and-fix-site
description: Diagnose a Promaker managed site that is failing and try to fix it automatically — all in one flow. Resolves the site, pulls deployments + build logs + runtime logs via the Promaker GitHub MCP, probes the public URL, then walks a decision tree from cheapest fix (start / redeploy) to narrow code fixes (e.g. swap `npm ci` → `npm install` in the Dockerfile when the build log shows lockfile drift). Spanish operator UX by default; mirrors the user's language. Use when the user says "arregla X", "fixame X", "debug X", "diagnose X", "why is X failing", "X no anda", "X está caído", "X tira error", "X da 502 / 404", or any "diagnose AND fix X" phrasing. Only hands off to `report-site-down` once the try-harder loop is exhausted.
---

# Debug and fix a broken site (skill instructions)

This skill diagnoses first, then fixes — in one coherent flow. It
absorbs what used to be two separate skills (`diagnose-site` +
`fix-site`) because separating them forced the model to route at
invocation time and the fix branch frequently won by description
overlap.

**Try harder before escalating.** The skill is expected to walk
through the decision tree below aggressively, attempting every cheap
and medium-cost fix that matches an unambiguous pattern, before
handing off to `report-site-down`. Each attempt must be materially
different from the last (redeploy ≠ code fix + redeploy).

## Audience and language

- **Default**: Spanish operator UX (argentino casual). Translate raw
  logs, never expose internals (Dokploy, Traefik, container,
  Terraform, Docker, workflow, redeploy, etc.). Use "publicar de
  nuevo", "reiniciar el sitio", "reactivar el ruteo", "el sistema".
- **English user**: mirror the language. Present an engineer-friendly
  structured summary (raw log quotes OK, technical vocabulary OK).
- Detect audience from the **incoming phrasing** — if the user wrote
  in Spanish, respond in Spanish; if English, English. On a mix,
  default to Spanish.

When in Spanish mode, follow [`../_shared/LANGUAGE.md`].

## When to use this skill vs report-site-down

- **debug-and-fix-site** (this one): any "something is wrong with X"
  complaint, whether the user phrases it as diagnose, debug, fix,
  arregla, no anda, etc. **Always start here.**
- **report-site-down**: only as a fallback when this skill has run
  its full decision tree and cannot fix (systemic failure, resource
  issue, needs human code change). The handoff is automatic — point
  the operator there with a short summary.

The `onboard-site` skill points users here when an onboarding fails
partway through.

## Inputs

> Spanish: "¿Qué sitio querés que revise? Decime el nombre o el dominio."
> English: "Which site should I debug? Give me the name or domain."

Accept:
- `site_name` (snake_case): `cliente_nuevo`
- `appName` (Dokploy short): `web-ke4kjj`
- primary domain: `cliente.pmkr.site`

## Step 1 — Resolve the site

Use the Promaker GitHub MCP to find the site:

```
promaker_list_resources({
  kind:   "site",
  search: "<raw input>"
})
```

Pick the single matching entry. If multiple match, list them back
and ask which one. Remember:

- `application_id` — feeds every other `promaker_*` call
- `name`, `owner` (tenant), `environment`, `primary_url`
- Any repo reference the result carries (field names vary; inspect
  what's there before falling back to the HCL lookup in step 5)

If nothing matches, tell the user the site wasn't found and stop.

## Step 2 — Pull status in parallel

Dispatch the three `promaker_*` calls + a public URL probe in
parallel (all independent):

```
# A. recent deployments (newest-first, includes status + errorMessage)
promaker_list_deployments({ application_id: "<id>", limit: 10 })

# B. build logs for the most recent deployment
promaker_get_build_logs({
  application_id: "<id>", lines: 400, strip_progress: true
})

# C. runtime container logs
promaker_get_container_logs({ application_id: "<id>", lines: 300 })

# D. public URL probe via WebFetch
WebFetch({ url: "<primary_url>", prompt: "Return HTTP status code
           and first 200 chars of body" })
```

If (D) fails at the network level (DNS / connection refused /
timeout), treat `http = 000`.

## Step 3 — Onboarding check (conditional)

**Only** if (A) returned no deployments — the site may still be
onboarding or the onboarding may have failed. Look up recent
workflow runs:

```
actions_list({
  method:      "list_workflow_runs",
  owner:       "PromakerDev",
  repo:        "managed-sites-onboarder",
  resource_id: "onboard.yml",
  workflow_runs_filter: { per_page: 20, event: "workflow_dispatch" }
})
```

Match runs whose `display_title` / inputs reference the site name or
primary domain. Record current state + conclusion + active step +
`html_url`.

If the site has deployments, skip this step — onboarding is done by
definition.

## Step 4 — Classify

Build an internal summary from what you gathered:

```
state              = running   if the most recent deployment is status="running"
                   | done      if status="done"
                   | error     if status="error"
                   | idle      if deployments list is empty and an onboarding ran successfully (but first deploy never fired)
                   | never     if empty and no onboarding ever completed
last_deploy_status = status of most recent entry
last_deploy_error  = errorMessage of most recent entry (one-line trim)
http_status        = 200 | 404 | 502 | 5xx | 000 (from WebFetch probe)
systemic           = true if ≥3 of last 5 deploys errored with the
                     SAME errorMessage signature
build_pattern      = null
                   | "lockfile_drift_npm"   (see step 5)
                   | "lockfile_drift_pnpm"
                   | "lockfile_drift_yarn"
                   | "docker_copy_missing"
                   | "package_not_found"
                   | ...
runtime_pattern    = null
                   | "oom"
                   | "crash_on_startup"
                   | "port_bind_conflict"
                   | ...
```

### Build log patterns to recognise

Scan the tail (~100 lines) of (B) for:

| Signature | `build_pattern` |
|---|---|
| `npm ci` + any of `Missing:` / `EUSAGE` / `npm ci can only install packages when your package.json and package-lock.json...` / `lock file` out of sync / `found missing dependencies in lockfile` | `lockfile_drift_npm` |
| `pnpm install --frozen-lockfile` + `ERR_PNPM_OUTDATED_LOCKFILE` / `lockfile is not up to date` | `lockfile_drift_pnpm` |
| `yarn install --frozen-lockfile` + `error Your lockfile needs to be updated` | `lockfile_drift_yarn` |
| `failed to solve` + `COPY` + `not found` | `docker_copy_missing` |
| `manifest unknown` / `pull access denied` | `registry_auth` |
| `exit code: 137` / `Killed` during build | `build_oom` |

### Runtime log patterns

Scan the last 30 lines of (C) for:

| Signature | `runtime_pattern` |
|---|---|
| Traceback at startup + same error repeating | `crash_on_startup` |
| `OOMKilled` / `Killed` | `oom` |
| `EADDRINUSE` / `bind: address already in use` | `port_bind_conflict` |
| Uvicorn / Gunicorn `Worker failed to boot` | `crash_on_startup` |

## Step 5 — Decide the fix (decision tree, cheapest first)

Pick **one branch** and explain it to the user before acting — wait
for confirmation unless the fix is a pure no-op read.

Track attempts. Cap at **3 attempts per session**, where each
attempt must be materially different (a redeploy and a code-fix +
redeploy are different; two redeploys are not).

### Branch A — Stopped / idle

`state = idle` AND `http = 000 / none`.

> ES: "**<name>** está apagado. Lo reactivo, ¿dale?"
> EN: "**<name>** is stopped. I'll start it. OK?"

On confirmation, dispatch `manage-site.yml action=start`. Go to
Step 6.

If `last_deploy_status = none` (never deployed), `start` won't
work — skip to Branch F (systemic / onboarding gap).

### Branch B — Traefik 404 (routing dropped)

`state = done` + `http = 404` with body "404 page not found".

> ES: "**<name>** está corriendo pero el ruteo se cayó (bug conocido
> del sistema). Lo publico de nuevo para reactivarlo, ¿dale?"
> EN: "**<name>** is running but Traefik dropped its route. Redeploy
> to refresh it — proceed?"

On confirmation, dispatch `manage-site.yml action=redeploy`.

### Branch C — 502 with no clear cause

`state = done` + `http = 502`, runtime logs empty or only INFO.

> ES: "**<name>** está respondiendo con error pero no veo nada claro
> en los registros. Probemos publicarlo de nuevo (a veces lo
> arregla); si vuelve a fallar, paso a revisar el código."
> EN: "**<name>** returns 502 with no clear signal in runtime logs.
> Trying redeploy first — if it doesn't clear, I'll drill into the
> build."

On confirmation, dispatch `manage-site.yml action=redeploy`.

### Branch D — Transient build failure

`last_deploy_status = error`, `systemic = false`, `build_pattern =
null` or non-code (e.g. image pull flake). One-off or flake.

> ES: "El último intento de publicar **<name>** falló:
> > <una línea del errorMessage>
>
> Los anteriores funcionaron, probablemente fue algo temporal.
> ¿Reintentamos?"
> EN: "Last deploy errored with: `<errorMessage>`. Prior deploys
> healthy — likely transient. Retry?"

On confirmation, dispatch `redeploy`.

### Branch E — Lockfile drift (code fix + redeploy)

`build_pattern` in `{lockfile_drift_npm, lockfile_drift_pnpm,
lockfile_drift_yarn}` AND the Dockerfile in the site repo actually
contains the offending command.

This is the first of the **narrow code fix** branches. The plan:

1. Resolve the site's GitHub repo.
2. Fetch the Dockerfile.
3. Patch the offending line.
4. Push the change directly to the deploy branch (no PR dance — the
   site repo is single-purpose and the operator is waiting).
5. Redeploy.

#### 5.E.1 Resolve the site repo

Look at the `promaker_list_resources` result first — it may include
a `github_repository` / `source_repo` / similar field. If it does,
use it.

Fallback: read the site's HCL config from
`PromakerDev/managed-sites-infrastructure` (common path is
`sites/<owner>/<name>.hcl` — adapt if the listing shows otherwise).
Use `get_file_contents` and parse the HCL for the `github_repo` /
`source` / `repository` attribute. If the path guess misses, list
`sites/<owner>/` and find the file whose name matches the site.

If you can't resolve the repo, explain to the operator and skip to
Branch G (escalate).

#### 5.E.2 Fetch the Dockerfile

```
get_file_contents({
  owner: "PromakerDev",
  repo:  "<site_repo>",
  path:  "Dockerfile",
  ref:   "<default_branch>"   // main unless the HCL says otherwise
})
```

Remember the `sha` of the file — you'll need it to update.

#### 5.E.3 Patch

| Pattern | Replacement |
|---|---|
| `npm ci` (standalone token) | `npm install` |
| `npm ci --<flags>` | `npm install --<flags>` (keep all flags) |
| `pnpm install --frozen-lockfile` | `pnpm install` |
| `pnpm i --frozen-lockfile` | `pnpm install` |
| `yarn install --frozen-lockfile` | `yarn install` |
| `yarn --frozen-lockfile` | `yarn install` |

Preserve indentation and surrounding RUN layer structure. Only
change the line(s) that actually trigger the failure — don't rewrite
the whole Dockerfile.

If the offending pattern isn't actually present in the Dockerfile
(e.g. it's in a referenced sub-image or a script the Dockerfile
invokes), skip this branch and go to Branch G.

#### 5.E.4 Confirm with the operator, then push

Before pushing, tell the user exactly what will change:

> ES: "Encontré que **<name>** falla al instalar los paquetes porque
> el archivo de candados está desfasado. Puedo ajustarlo cambiando
> **<línea original>** por **<línea nueva>** en el código del sitio
> y publicarlo de nuevo. Es un cambio chico, seguro y reversible.
> ¿Lo aplico?"
> EN: "Build fails because the lockfile is out of sync with
> package.json. I can patch `<old line>` → `<new line>` in the
> Dockerfile and redeploy. Safe and reversible. Proceed?"

On explicit confirmation, push the patched file directly to the
deploy branch:

```
create_or_update_file({
  owner:   "PromakerDev",
  repo:    "<site_repo>",
  path:    "Dockerfile",
  branch:  "<default_branch>",
  message: "Dockerfile: <old command> → <new command>\n\nLockfile drift was blocking the build. Applied automatically by debug-and-fix-site. Run: <run_url>",
  content: <base64 of patched file>,
  sha:     <sha from step 5.E.2>
})
```

Then dispatch `manage-site.yml action=redeploy`. Go to Step 6.

#### 5.E.5 What if it fails again

If the redeploy after the patch still errors:
- With the same signature (`lockfile_drift_*` again) → the change
  didn't take effect or hit a deeper path; go to Branch G.
- With a NEW signature → Step 4 will reclassify; re-enter the tree
  with the new classification (but count it against the 3-attempt
  cap).

### Branch F — Systemic failure or onboarding gap

Any of:
- `systemic = true` (≥3 of last 5 deploys same error)
- `state = never` + onboarding failed or never ran
- `runtime_pattern` in `{oom, crash_on_startup, port_bind_conflict}`
  AND you've already redeployed once in this session

These need a human. Don't attempt another fix.

> ES: "**<name>** está fallando de forma persistente:
> > <una línea del errorMessage o log>
>
> Este tipo de problema no se arregla publicando de nuevo — hay que
> que mirar el código o la configuración. Te paso al diagnóstico
> detallado."
> EN: "**<name>** is failing systemically: `<error>`. Not recoverable
> via redeploy — needs human intervention. Handing off to
> `report-site-down` for the formal escalation."

Hand off to `report-site-down` with the classification summary,
build/runtime snippets, and run URLs.

### Branch G — Escalate (exhausted / resolve failed)

You hit the 3-attempt cap, couldn't resolve the site repo, or the
branch logic decided the skill can't help further.

Same handoff as Branch F — bundle the context and point at
`report-site-down`.

### Branch H — Already healthy

`state = done` + `http = 200` + no recent runtime errors.

> ES: "**<name>** está respondiendo bien (HTTP 200) y no veo nada
> raro en los registros. Capaz fue algo temporal — probá recargar
> (Ctrl+F5 / Cmd+Shift+R) o abrir en otro navegador. Si sigue
> mal, avisame de nuevo."
> EN: "**<name>** is healthy (HTTP 200, runtime quiet). Was it
> actually failing? If you still see an error, re-run with more
> context."

No fix needed. Stop.

## Step 6 — Apply the fix and track progress

Dispatch the chosen workflow action and **poll until the run
completes** (`actions_get` method `get_workflow_run` every 10 s).
While it runs, emit progress in the user's language:

| Underlying step | ES | EN |
|---|---|---|
| Resolve app to application ID | "Buscando el sitio..." | "Resolving site..." |
| Execute action (start) | "Reactivando el sitio..." | "Starting..." |
| Execute action (redeploy) | "Publicando de nuevo, esto tarda 1 a 3 minutos..." | "Redeploying, 1–3 minutes..." |
| Anything else | (silent) | (silent) |

If the workflow run fails to start (workflow_dispatch error, not the
site's error):

> ES: "❌ Algo se rompió al aplicar el arreglo. Pasale este enlace al
> equipo técnico: <run_url>."
> EN: "❌ Workflow dispatch failed. Run: `<run_url>`."

## Step 7 — Verify (deep check)

After the fix completes, **wait 15 s** to let DNS / Traefik settle,
then re-run step 2 (the full status pull, in parallel). Classify
the new state.

### 7a. Front-door

- `state = done` + `http = 200` → continue to 7b
- Same `http_status` as before → **no improvement** — go to Step 8
  "same status" and count it against the attempt cap
- Different `http_status` — reclassify; may open a new branch (still
  counts as an attempt)

### 7b. Deep check (only when 7a is HTTP 200)

HTTP 200 only proves Traefik routes and the app accepted one
request. Confirm:

1. **Latest deploy `done`**: check (A) from the re-pull.
2. **Runtime quiet**: scan (C) for fresh tracebacks / repeated
   ERROR / Killed / OOMKilled in the last ~2 min.

Emit progress:

> ES: "Ya responde bien. Reviso un par de detalles internos..."
> EN: "Site responds. Running deep checks..."

Outcomes feed Step 8.

## Step 8 — Final message

### Fully healthy (HTTP 200, latest deploy `done`, runtime clean)

> ES: "✅ Listo, **<name>** ya responde bien en **<domain>** y no veo
> nada raro en los registros. Probalo de tu lado."
> EN: "✅ **<name>** is healthy on **<domain>**. Deploy `done`,
> runtime clean. Confirm client-side."

### Working but stale build (HTTP 200 + latest deploy `error`)

> ES: "⚠️ El sitio responde, pero el último intento de publicar falló:
> > <error>
>
> Está sirviendo la versión anterior. ¿Volvemos a publicar para que
> esté con los últimos cambios?"
> EN: "⚠️ Serving a prior good build — latest deploy errored with
> `<error>`. Redeploy to pick up changes?"

If yes, redeploy (counts as an attempt), then re-verify.

### Responds but unhealthy (HTTP 200 + runtime errors)

> ES: "⚠️ Responde pero veo errores en los registros:
> > <una o dos líneas>
>
> ¿Publicamos de nuevo (suele limpiar transitorios) o escalamos?"
> EN: "⚠️ 200 OK but runtime logs show `<lines>`. Retry redeploy or
> escalate?"

One more attempt allowed; if errors persist, Branch G.

### Same status after fix

> ES: "❌ El arreglo no surtió efecto. **<name>** sigue dando
> **<http_status>**. Te paso al diagnóstico detallado."
> EN: "❌ Fix didn't take. Still `<http_status>`. Handing off to
> `report-site-down`."

Escalate via Branch G.

### Different status after fix

> ES: "El sitio cambió: antes daba **<old>**, ahora da **<new>**.
> Avisame si ya estás bien o si seguís viendo error."
> EN: "Status shifted: `<old>` → `<new>`. Confirm resolution or
> re-run."

## Boundaries

- **Safe and reversible actions only**: start, redeploy, and the
  narrow code fixes listed in Branch E. Nothing that destroys data
  or changes site configuration (DNS, env vars, domain routing).
- **Confirm every action before executing**, including obvious ones,
  and especially before pushing a code change.
- **3-attempt cap per session, each attempt materially different.**
  If the same class of fix (e.g. bare redeploy) wouldn't change the
  outcome, it doesn't count as a distinct attempt — escalate
  instead.
- **Never propose changes beyond the patterns listed in Branch E.**
  If the build log shows a different kind of code bug (logic error,
  missing env var, TS compile error, etc.), surface it and escalate;
  this skill is not a general code editor.
- **Spanish UX hides internals.** Don't name Terraform, Dokploy,
  Traefik, Docker, workflow, Dockerfile, `npm ci`, lockfile, etc.
  in the final operator message — translate:
  - "lockfile drift" → "el archivo de candados está desfasado"
  - "Dockerfile" → "el código del sitio"
  - "redeploy" → "publicar de nuevo"
  - "Traefik route" → "ruteo"
  English mode may use the technical terms freely.
- **Internal infrastructure sites** (`managed-sites-*`, `codemod-*`,
  `promaker-*`, `mcp-*`): refuse.
  > ES: "Ese sitio es parte de la infraestructura interna, no se
  > opera por acá. Pasale al equipo técnico."
  > EN: "That site is platform infrastructure — not operated through
  > these skills."
- **Tool choice:**
  - Use the Promaker GitHub MCP (`promaker_list_resources`,
    `promaker_list_deployments`, `promaker_get_build_logs`,
    `promaker_get_container_logs`) for resolution + status + logs.
  - Use the official `github/github-mcp-server` (`actions_*`) for
    dispatching `manage-site.yml` and reading workflow runs /
    onboarding history.
  - Use `get_file_contents` + `create_or_update_file` for code
    fixes in Branch E.
  - Use `WebFetch` for the public URL probe.
