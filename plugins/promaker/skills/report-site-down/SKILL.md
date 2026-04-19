---
name: report-site-down
description: Diagnose a site that "is not working" for a non-technical operator. Pulls the live status, checks whether the URL responds, and explains what it found in plain Spanish with a concrete next step. Use when the operator says "el sitio X no funciona", "está caído", "no anda", "no carga", "me aparece error".
---

# Report site down (skill instructions)

Apply shared language rules from `../_shared/LANGUAGE.md`. The operator
is worried about a site — stay calm and concrete.

## Inputs

> "¿Qué sitio está fallando? Decime el nombre o el dominio."

Accept site_name, appName, or primary domain.

Optional: if the operator gave extra context ("hace 5 min", "desde
ayer", "solo en Chrome", "mis clientes me avisaron"), note it but do
not act on it yet — first get the actual status.

## Run the diagnostic

Use the official `github/github-mcp-server` tool `actions_run_trigger`
with method `run_workflow`:

```
actions_run_trigger({
  method:      "run_workflow",
  owner:       "PromakerDev",
  repo:        "managed-sites-onboarder",
  workflow_id: "manage-site.yml",
  ref:         "main",
  inputs: {
    action: "status",
    site:   "<raw operator input>"
  }
})
```

Poll until complete with `actions_list` (method `list_workflow_runs`)
to find the run id, then `actions_get` (method `get_workflow_run`) for
status / conclusion.

## Read the structured block

After success, fetch the job logs with `get_job_logs` (the job id is
in `actions_list / list_workflow_jobs`):

```
get_job_logs({
  owner:          "PromakerDev",
  repo:           "managed-sites-onboarder",
  job_id:         "<job_id>",
  return_content: true,
  tail_lines:     400
})
```

Search the returned content for the delimited block:

```
##[group]PROMAKER_SITE_STATUS_JSON
{"name": "...", "state": "...", "last_deploy_status": "...",
 "last_deploy_started_at": "...", "last_deploy_error": "...",
 "domains": [...], "http_status": "..."}
##[endgroup]
```

Parse the JSON.

If you cannot locate the block but the run succeeded, fall back to the
run's `html_url` and tell the operator to check from there.

## Diagnose & respond

Apply this decision tree in Spanish:

### 1. App state = `done` or `running`, HTTP = `200`

The site is actually working from the server's side. The problem is
probably on the operator's end.

> "El sitio **<name>** está en línea y responde bien en este momento.
> Si lo estás viendo caído, probá:
> 1. Recargar la página (Ctrl+F5 en Windows, Cmd+Shift+R en Mac).
> 2. Abrirlo en otro navegador o en modo incógnito.
> 3. Probar desde otra red (por ejemplo, el 4G del celular).
>
> Si sigue sin andar, avisame de nuevo y escalamos al equipo técnico."

### 2. App state = `idle`, HTTP = `000` or `none`

The site was stopped.

> "El sitio **<name>** está detenido — por eso no responde. ¿Querés que
> lo inicie?"

If they say yes, hand off to the `start-site` skill (or dispatch
manage-site.yml with action=start yourself and report).

### 3. App state = `running`, HTTP = `000` / timeout

Probably mid-deploy.

> "El sitio **<name>** está en medio de un cambio. Esperá 30 segundos
> y probá de nuevo. Si sigue sin responder, avisame."

### 4. App state = `error` OR last deploy `error`

A deploy broke.

> "El último intento de publicar **<name>** falló. El error fue:
>
> > <last_deploy_error>
>
> Pasale este enlace al equipo técnico para que lo revisen:
> <run_url>."

### 5. HTTP = `404` (not 000) with state `done`

The container is up but Traefik is not routing — likely the
domain/routing issue documented in the infra repo.

> "El sitio **<name>** está corriendo pero hay un problema con el ruteo
> (quién te lleva al sitio cuando escribís la dirección). No es algo
> que puedas resolver vos. Avisale al equipo técnico y pasales este
> enlace: <run_url>."

### 6. HTTP = `5xx`

Container is responding but broken.

> "El servidor de **<name>** está respondiendo con error. Probablemente
> falla la aplicación. El equipo técnico lo va a tener que mirar —
> pasales: <run_url>."

### 7. Any other combination

> "Encontré algo raro en el estado de **<name>**. Pasale este enlace al
> equipo técnico: <run_url>."

## Follow-up

After giving the diagnosis, if the operator asks "¿qué puedo hacer?",
offer concrete actions depending on the case:

- Case 2: offer to start it.
- Case 3: remind them to wait 30s.
- Case 4, 5, 6, 7: nothing they can do; confirm you escalated.

## Boundaries

- Never dump the raw JSON or logs to the operator.
- Never suggest they SSH, modify DNS, clear cache beyond
  browser-level, or contact the hosting provider directly.
- If the operator insists on "reiniciá el servidor" or similar,
  translate: "Ok, voy a reiniciar el sitio" → use `redeploy` on
  `manage-site.yml`. But only after confirmation.
