---
name: report-site-down
description: Diagnostic-only fallback for a site that "is not working" — use when `debug-and-fix-site` has already exhausted its try-harder loop or when the operator explicitly says "no toques nada, solo decime qué pasa". Resolves the site via the Promaker GitHub MCP, pulls deployments, build logs and runtime logs directly from Dokploy, probes the public URL, and explains what it found in plain Spanish with a concrete next step. Takes no actions. Use when the operator says "el sitio X no funciona", "está caído", "no anda", "no carga", "me aparece error" AND they want just the diagnosis (otherwise start with `debug-and-fix-site`).
---

# Report site down (skill instructions)

Apply shared language rules from [`../_shared/LANGUAGE.md`]. The
operator is worried about a site — stay calm and concrete.

## Inputs

> "¿Qué sitio está fallando? Decime el nombre o el dominio."

Accept site_name, appName, or primary domain.

Optional: if the operator gave extra context ("hace 5 min", "desde
ayer", "solo en Chrome", "mis clientes me avisaron"), note it but do
not act on it yet — first get the actual status.

## How to pull diagnostic data

This skill **uses the Promaker GitHub MCP (`promaker_*` tools)** for
all the deep diagnostic data. Those tools read Dokploy + Terraform
state directly — faster and richer than dispatching a
`manage-site.yml` workflow.

Only fall back to `actions_*` (official `github/github-mcp-server`)
for the **onboarding-status lookup** (step 1b below).

### Step 1a — Resolve the site

```
promaker_list_resources({
  kind:   "site",
  search: "<raw operator input>"
})
```

Pick the single matching entry. If multiple entries match, ask the
operator which one. Remember:

- `application_id` — feed into every other `promaker_*` call
- `name`, `owner` (tenant), `environment`, `primary_url` — for the
  final message

If no entry matches:

> "No encontré ningún sitio llamado **<input>**. Revisá el nombre o
> el dominio y probá de nuevo."

Stop here.

### Step 1b — Onboarding check (only if the site has no deployments)

First pull deployments (step 2a). If the list is empty, the site
has never deployed — check whether the onboarding pipeline is still
running / failed before anything else:

```
actions_list({
  method:      "list_workflow_runs",
  owner:       "PromakerDev",
  repo:        "managed-sites-onboarder",
  resource_id: "onboard.yml",
  workflow_runs_filter: { per_page: 20, event: "workflow_dispatch" }
})
```

Match runs whose `display_title` or inputs reference the site name /
primary domain. If a run is still `in_progress`, tell the operator
(Case 8 below). If a run is `completed` with `conclusion=failure`,
hand off (Case 9).

Skip this step entirely when there ARE deployments — onboarding is
by definition done.

### Step 2 — Pull status in parallel

Dispatch all three in parallel (independent calls):

```
# 2a. deployments (newest-first, includes status + errorMessage)
promaker_list_deployments({
  application_id: "<id>",
  limit:          5
})

# 2b. build logs for the most recent deployment
#     (deployment_id omitted → latest by createdAt)
promaker_get_build_logs({
  application_id: "<id>",
  lines:          300,
  strip_progress: true
})

# 2c. runtime container logs
promaker_get_container_logs({
  application_id: "<id>",
  lines:          200
})
```

If 2c returns empty or errors, note it — could mean the container
never started (Case 4) or crashed immediately (Case 6).

### Step 3 — Probe the public URL

Use `WebFetch` against the resolved `primary_url` (e.g.
`https://cliente.pmkr.site`) with a short prompt like "Return the
HTTP status code and the first 200 chars of the body". Classify:

- HTTP 200 → `http=200`
- HTTP 404 with a Traefik-ish body ("404 page not found") → `http=404`
- HTTP 5xx → `http=5xx`
- Timeout / connection refused / DNS failure → `http=000`

If `WebFetch` itself fails with a network-level error, treat it as
`http=000`.

## Derive state

Build a small summary from what you gathered:

```
state               = running   if the most recent deployment is
                      status="running"
                    | done      if status="done"
                    | error     if status="error"
                    | idle      if deployments list is empty
                    | never     if also no onboarding ever ran
last_deploy_status  = <status field of most recent entry>
last_deploy_error   = <errorMessage field, trimmed to one sentence>
domains             = [primary_url, ...any aliases]
http_status         = <from step 3>
```

## Diagnose & respond

Apply this decision tree in Spanish (argentino casual):

### 1. state = `done` / `running`, http = `200`

The site is actually working from the server's side. The problem is
probably on the operator's end.

> "El sitio **<name>** está en línea y responde bien en este momento.
> Si lo estás viendo caído, probá:
> 1. Recargar la página (Ctrl+F5 en Windows, Cmd+Shift+R en Mac).
> 2. Abrirlo en otro navegador o en modo incógnito.
> 3. Probar desde otra red (por ejemplo, el 4G del celular).
>
> Si sigue sin andar, avisame de nuevo y escalamos al equipo técnico."

### 2. state = `idle` (no recent deploy), http = `000` / `none`

The site was stopped or has never been deployed.

> "El sitio **<name>** está detenido — por eso no responde. ¿Querés
> que lo inicie?"

If they say yes, hand off to the `start-site` skill.

### 3. state = `running`, http = `000` / timeout

Probably mid-deploy.

> "El sitio **<name>** está en medio de un cambio. Esperá 30 segundos
> y probá de nuevo. Si sigue sin responder, avisame."

### 4. state = `error` OR last deploy `error` (build failed)

A deploy broke. The `build logs` from step 2b should show **why**.
Scan the tail for the first obvious error line and translate it into
one plain-Spanish sentence (don't quote raw logs to the operator).

> "El último intento de publicar **<name>** falló. Lo que vi fue:
>
> > <una línea traducida — p. ej. «le falta un archivo en el código»,
> > «no pudo conectarse al registro», «la versión del paquete chocó»>
>
> Pasale este detalle al equipo técnico para que lo revisen."

If ≥3 of the last 5 deploys failed with the same error, add:

> "(Venía fallando parejo los últimos intentos — no tiene sentido
> reintentar, el equipo técnico tiene que mirar el código o la
> configuración.)"

### 5. http = `404` (not 000) with state `done`

The container is up but Traefik is not routing — known bug.

> "El sitio **<name>** está corriendo pero hay un problema con el
> ruteo (quién te lleva al sitio cuando escribís la dirección).
> Podemos probar publicarlo de nuevo, eso suele destrabarlo. ¿Lo
> intento?"

If they accept, hand off to `debug-and-fix-site`. If they prefer to escalate,
tell them a técnico puede mirarlo.

### 6. http = `5xx` / container crash

Container is responding but broken. The `runtime logs` from step 2c
usually contain the traceback.

Scan the last 30 lines of runtime logs for:
- `Traceback`, `Error:`, `Caused by:`
- `Killed`, `OOMKilled`, `exited with code`
- `missing required env var`, `KeyError`, `ENOENT`, `EADDRINUSE`

Translate the first clear one into one plain sentence.

> "El servidor de **<name>** está respondiendo con error. Lo que vi
> en los registros internos:
>
> > <una línea traducida>
>
> Probablemente falla la aplicación. El equipo técnico lo va a tener
> que mirar."

If the runtime logs are empty:

> "El servidor de **<name>** está respondiendo con error y no
> encontré detalles útiles en los registros. El equipo técnico lo va
> a tener que mirar de cerca."

### 7. Any other combination

> "Encontré algo raro en el estado de **<name>**. Pasale al equipo
> técnico para que miren el diagnóstico detallado."

### 8. No deployments + onboarding still running

```
> "El sitio **<name>** todavía se está publicando por primera vez —
> el proceso sigue corriendo. Esperá unos minutos más y probá de
> nuevo. Si querés seguir el progreso, pedime «¿cómo va
> **<name>**?»."
```

### 9. No deployments + onboarding failed

> "El sitio **<name>** nunca llegó a publicarse — el proceso de
> onboarding falló. Hay que empezarlo de nuevo o revisar qué pasó.
> Pasale al equipo técnico."

## Follow-up

After giving the diagnosis, if the operator asks "¿qué puedo hacer?",
offer concrete actions depending on the case:

- Case 2: offer to start it (`start-site`).
- Case 3: remind them to wait 30s.
- Case 4 (transient, not systemic), 5: offer a reintento via
  `debug-and-fix-site`.
- Case 4 (systemic), 6, 7, 9: nothing they can do; confirm
  que pasás el diagnóstico al equipo técnico.
- Case 8: offer `onboarding-status` to track progress.

## Boundaries

- Never dump raw JSON, build logs, or runtime logs to the operator.
  Translate into one plain-Spanish sentence.
- Never suggest they SSH, modify DNS, clear cache beyond
  browser-level, or contact the hosting provider directly.
- If the operator insists on "reiniciá el servidor" or similar,
  translate: "Ok, voy a reiniciar el sitio" → hand off to
  `debug-and-fix-site`. But only after confirmation.
- Use the Promaker GitHub MCP (`promaker_list_resources`,
  `promaker_list_deployments`, `promaker_get_build_logs`,
  `promaker_get_container_logs`) as the primary source. Only use
  `actions_*` for the onboarding-run lookup in step 1b.
- For internal infrastructure sites (`managed-sites-*`, `codemod-*`,
  `promaker-*`, `mcp-*`): refusá con "Ese sitio es parte de la
  infraestructura interna, no se opera por acá. Pasale al equipo
  técnico."
