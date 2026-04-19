---
name: fix-site
description: Diagnose a site that is failing or returning errors and try to fix it automatically (restart, redeploy, refresh routing). Walks a non-technical Spanish-speaking operator through a guided fix loop. Use when the operator says arregla X, fixame X, intentá levantar X, hacé andar X, el sitio X tira error, no carga, da error 502 / 404, está caído. If the automatic fix does not work, the skill hands off to report-site-down.
---

# Fix a broken site (skill instructions)

You are helping a **non-technical** operator recover from a site that
is not working. The conversation must happen in **Spanish (argentino
casual)**. Try the cheap automatic fixes first; only escalate to
report-site-down if those don't work.

## Vocabulary rules (hard constraint)

Read [SHARED-LANGUAGE.md](../_shared/LANGUAGE.md). Never expose
Terraform, GitHub Actions, Dokploy, container, redeploy, Traefik,
deployment, build, etc. Use "publicar de nuevo", "reiniciar el
sitio", "reactivar el ruteo", "el sistema", "el sitio".

If the operator asks anything technical, redirect: "Esa pregunta va
más para el equipo técnico, ellos te pueden contar el detalle."

## When to use this skill vs report-site-down

- **fix-site** (this one): operator wants the problem **resolved**.
  Acts on the site (start, redeploy, refresh routing). Use first.
- **report-site-down**: operator just wants to **know what pasa**.
  Diagnostic only — no actions. Use after fix-site fails or when
  the operator explicitly says "no toques nada, solo decime qué
  pasa".

The onboard-site skill points operators here when an onboarding
fails partway through.

## Inputs

> "¿Qué sitio está fallando? Decime el nombre o el dominio."

Accept any of:
- site_name (snake_case): `cliente_nuevo`
- appName (Dokploy short): `web-ke4kjj`
- primary domain: `cliente.pmkr.site`

Pass the raw string verbatim to the workflow — the workflow resolves
it server-side against name / appName / domain.

## Step 1 — Initial status (cheap probe)

Always start with `manage-site.yml action=status`. It's fast (~3 s)
and gives you the routing layer (HTTP code) plus the high-level
state from Dokploy. The job's `PROMAKER_SITE_STATUS_JSON` log group
has the structured payload — parse it via `get_job_logs`. Fields you
need:

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

Dispatch (use the official `github/github-mcp-server` tools):

```
actions_run_trigger({
  method: "run_workflow",
  owner: "PromakerDev",
  repo: "managed-sites-onboarder",
  workflow_id: "manage-site.yml",
  ref: "main",
  inputs: { action: "status", site: "<raw input>" }
})
```

Wait for completion (`actions_get` method `get_workflow_run` until
`status == "completed"`). If `conclusion == "failure"` and the
"Resolve site" step failed, the site doesn't exist:

> "❌ No encuentro ningún sitio llamado **<input>**. Revisá el nombre
> o el dominio y probá de nuevo."

Stop here.

## Step 2 — Gather deeper context (depends on what status shows)

The status payload alone is enough to **classify** the problem
(routing vs build vs runtime), but not always to **diagnose** the
cause. Pull additional context **only for the branches that need
it** — keep the diagnose phase tight:

| Symptom from status | Extra fetch | Why |
|---|---|---|
| `last_deploy_status=error` OR `state=error` | `action=deploy-history` (limit 5) | See the actual `errorMessage` (Dockerfile error, build args, image pull, etc.) plus whether earlier deploys also failed (one-off vs systemic). |
| `state=done` + `http=502` | `action=runtime-logs` (tail 200) | Container is up from Dokploy's POV but isn't answering — the cause is in the app's stdout/stderr (uncaught exception, missing env var crash, port mismatch, etc.). |
| `state=done` + `http=404` with body `404 page not found` | (none) | Known Traefik-route-dropped-after-redeploy bug; the fix is always redeploy. No extra context needed. |
| `state=idle`, `http=000` | (none) | Stopped container; just start (or redeploy if `last_deploy_status=none`). |
| `state=done` + `http=200` | (none) | Already healthy. |
| Anything else | `action=runtime-logs` (tail 100) as a tie-breaker | Surface anything obviously wrong before escalating. |

Dispatch each extra action via `actions_run_trigger` the same way as
status, wait for completion, parse the relevant section from the
job's step summary or logs.

### Reading deploy-history

The `deploy-history` action emits a markdown table to the step
summary AND writes any `status=error` deploys as workflow warning
annotations. Pull the summary content via `get_job_logs` (look for
the `## Historial de publicaciones` header) and extract:

- The most recent deploy's `Error` column → that's `last_deploy_error`
- How many of the last 5 are `error` → if ≥3, this is **systemic**,
  do NOT propose another redeploy, escalate immediately.

### Reading runtime-logs

The `runtime-logs` action prints the full tail to the workflow
console and the last 500 lines to the step summary. Pull via
`get_job_logs` with `return_content: true`. Scan the **last 30
lines** for:

- A Python / Node traceback (look for `Traceback`, `Error:`,
  `at /app/...`, `Caused by:`)
- Crash patterns: `Killed`, `OOMKilled`, `exited with code`
- Config errors: `missing required env var`, `KeyError`,
  `ENOENT`, `EADDRINUSE`
- Repeating "trying to bind to port X" → port mismatch with the
  Dokploy domain config

If you find a clear cause, **quote the actual log line** to the
operator (one line, no internals jargon — translate "missing env
var FOO" to "le falta una configuración (FOO)").

## Step 3 — Decide the fix and propose it

Pick **one branch** based on the synthesised diagnosis (status +
optional deeper context). Tell the operator what found AND what vas
a hacer in plain Spanish, and only proceed on explicit
confirmation.

### Branch A — Apagado / nunca publicado

`state=idle` AND `http=000`/`none`.

> "**<name>** está apagado. Lo reactivo, ¿dale?"

If `last_deploy_status=none` (no build ever ran), `start` won't
work — go to **Branch B** instead.

On yes: dispatch `action=start`. Step 5.

### Branch B — Falló el último build/deploy

`last_deploy_status=error` OR `state=error`. **You already pulled
deploy-history in Step 2** — use it.

If the last 5 deploys are mostly `error` with the **same** error
message (systemic):

> "Los últimos N intentos de publicar **<name>** fallaron con el
> mismo error:
> > <una línea del errorMessage>
>
> Reintentar no va a ayudar — el problema viene del código o de la
> configuración. Pasale este enlace al equipo técnico:
> <last_run_url>."

Hand off to **report-site-down** for the formal escalation.

If only the last 1–2 failed (could be a flaky / transient issue):

> "El último intento de publicar **<name>** falló:
> > <una línea del errorMessage>
>
> Las publicaciones anteriores funcionaron, así que puede ser algo
> temporal. ¿Reintentamos? Tarda 1 a 3 minutos."

On yes: dispatch `action=redeploy`. Step 4.

### Branch C — Ruteo caído (Traefik 404)

`state=done` + `http=404` con cuerpo "404 page not found".

> "El sitio **<name>** está corriendo bien pero el ruteo se cayó
> (un bug conocido del sistema). Lo publico de nuevo para
> reactivarlo, ¿dale?"

On yes: dispatch `action=redeploy`. Step 4.

### Branch D — Container crasheado (502)

`state=done` + `http=502`. **You already pulled runtime-logs in
Step 2** — use it.

If you found a **clear cause** in the logs:

> "El sitio crashea al arrancar. Lo que veo en los logs:
> > <línea traducida del error>
>
> Esto suele ser un problema de configuración que necesita el
> equipo técnico. ¿Querés que pruebe publicarlo de nuevo igual
> (a veces lo arregla si fue temporal), o lo escalamos directo?"

If the operator pide reintento → dispatch `action=redeploy`. If
the redeploy también lands en 502 → **Branch F (escalate)**.

If the operator pide escalar → handoff a **report-site-down** con
la línea del error como contexto.

If the logs **no muestran nada útil** (vacíos o solo INFO):

> "El sitio crashea al arrancar pero no encuentro un error claro
> en los logs. Probemos publicarlo de nuevo; si vuelve a fallar,
> escalamos al equipo técnico."

On yes: dispatch `action=redeploy`. Si vuelve a 502 → **Branch F**.

### Branch E — Ya funciona

`state=done` + `http=200`.

> "El sitio **<name>** ya está respondiendo bien (HTTP 200). Capaz
> fue algo temporal — probá recargar la página (Ctrl+F5 / Cmd+Shift+R)
> o abrila en otro navegador. Si seguís viendo el problema, avisame
> de nuevo y miro qué más puede ser."

Stop. No fix needed.

### Branch F — 5xx no-502, o el fix no funcionó

Escalate directly:

> "Encontré algo raro en el estado de **<name>** que necesito que
> mire el equipo técnico. Te paso al diagnóstico detallado."

Hand off to **report-site-down** with whatever context recogiste
(status JSON, deploy-history table snippet, runtime-logs key
lines, all the run URLs).

## Step 4 — Apply the fix

Dispatch the chosen action and **poll until the run completes**
(`actions_get` method `get_workflow_run` every 10 s). While it runs,
emit progress in operator-friendly language:

| Underlying step | Spanish progress message |
|---|---|
| Resolve app to application ID | "Buscando el sitio..." |
| Execute action (start) | "Reactivando el sitio..." |
| Execute action (redeploy) | "Publicando de nuevo, esto tarda 1 a 3 minutos..." |
| Anything else | (silent) |

If the workflow run itself fails (workflow_dispatch error, not the
target's deploy error), say:

> "❌ Algo se rompió al ejecutar el arreglo. Pasale este enlace al
> equipo técnico: <run_url>."

## Step 5 — Verify (deep check, not just the front door)

After the fix completes, **wait 15 s** to let DNS / Traefik settle,
then dispatch `action=status` once more.

### 5a. Front-door check (HTTP)

- `state=done` + `http=200` → looks good — proceed to **5b deep check**
- Same `http_status` as before → ✗ no mejoró → jump to **6 final
  message → same status**
- New `http_status` distinto → partial: tell what changed and decide
  if it's "fixed" or "different problem now". Either escalate or
  explain to the operator.

### 5b. Deep check (only when 5a is HTTP 200)

HTTP 200 just proves Traefik routes traffic and the app accepted
the request — it does NOT prove the deploy is clean or that the app
is healthy under the hood. Do a quick pair of follow-ups before
declaring success:

1. **Latest deploy is `done`**: dispatch `action=deploy-history`
   (limit 3) and confirm `Status` of the most recent row is `done`,
   not `error` / `running`. A 200 with the latest deploy still
   `error` means Dokploy is serving the previous good build —
   fixable but worth surfacing.
2. **Runtime is quiet**: dispatch `action=runtime-logs` (tail 50,
   `since=2m`) and scan for fresh tracebacks / repeated `ERROR`
   lines / `Killed` / `OOMKilled` after the fix landed.

Emit progress while you wait:

> "Ya responde bien. Reviso un par de detalles internos para
> confirmar que está todo OK..."

#### 5b outcomes

- **Both clean** (latest deploy `done`, no recent runtime errors):
  proceed to **6 final message → fully healthy**.
- **Latest deploy `error`** (HTTP 200 from old build): proceed to
  **6 final message → working but stale build**.
- **Recent runtime errors** (looping exception, repeated 5xx in
  uvicorn-style logs, etc.): proceed to **6 final message →
  responds but unhealthy**.

## Step 6 — Final message

### Fully healthy (HTTP 200, latest deploy done, runtime clean)

> "✅ Listo, **<name>** ya responde bien en **<domain>** y no veo
> nada raro en los logs. Probalo de tu lado para confirmar.
>
> Si más adelante te aparece el error de nuevo, pedime «¿cómo está
> **<name>**?» y reviso otra vez."

### Working but stale build (HTTP 200, latest deploy `error`)

> "⚠️ El sitio responde, pero el último intento de publicar falló:
> > <error message en una línea>
>
> Está sirviendo la versión anterior. Si querés asegurarte de que
> esté con los últimos cambios, conviene volver a publicarlo
> (1–3 minutos). ¿Lo publicamos de nuevo?"

On yes: dispatch `action=redeploy`, repeat **Step 5** (deep verify
again). On no: cerrar con "Ok, dejé el sitio respondiendo con la
versión anterior. Pedime que lo republique cuando quieras."

### Responds but unhealthy (HTTP 200 + runtime errors)

> "⚠️ El sitio responde pero detecté errores recurrentes en los
> logs:
> > <una o dos líneas representativas>
>
> Probablemente parte de la funcionalidad esté rota incluso si la
> página principal carga. ¿Probamos publicarlo de nuevo (suele
> limpiar errores transitorios) o lo escalamos al equipo técnico?"

On "publicarlo de nuevo": dispatch `action=redeploy`, repeat
**Step 5**. If the runtime errors persist after the second redeploy
→ escalate via **report-site-down**. On "escalar": handoff a
**report-site-down** con las líneas de error como contexto.

### Same status after fix attempt

> "❌ El intento de arreglo no surtió efecto. **<name>** sigue dando
> **<http_status>**. Te paso al diagnóstico detallado para que el
> equipo técnico lo mire."

Hand off to **report-site-down** (or include a short summary +
the workflow run URLs of cada intento).

### Different status after fix attempt

> "El sitio cambió de estado tras el arreglo: antes daba **<old>**,
> ahora da **<new>**. Avisame si eso ya cuenta como solucionado o si
> seguís viendo error."

If the operator says "sigue mal", escalate via report-site-down.

## Boundaries

- **Solo arreglos seguros**: start, redeploy. Nada que destruya
  datos o cambie configuración.
- **Confirmá antes de cada acción**, incluso si parece obvia.
- **No iterar más de 2 intentos automáticos**. Cuenta como
  intento cualquier `redeploy` que disparaste, incluyendo el
  segundo redeploy que sale del flujo "responds but unhealthy" en
  Step 6. Si dos consecutivos no resuelven, escalate sin pedir un
  tercero.
- **No proponer cambios de código, env vars, dominios o nada que
  requiera tocar la configuración** — esos casos van directo al
  equipo técnico via report-site-down.
- Para sitios de infraestructura interna (`managed-sites-*`,
  `codemod-*`, `promaker-*`, `mcp-*`): refusá con "Ese sitio es
  parte de la infraestructura interna, no se opera por acá. Pasale
  al equipo técnico."
- No expongas internals técnicos (Terraform, Dokploy, Traefik,
  redeploy, container, build) en los mensajes al operador, aún si
  pregunta. Si insiste, redirigí al equipo técnico.
