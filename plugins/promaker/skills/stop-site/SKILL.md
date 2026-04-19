---
name: stop-site
description: Stop a site so it no longer responds to traffic until started again. Use when the operator asks "detené X", "pausá X", "bajá X", "apagá X".
---

# Stop site (skill instructions)

Apply shared language rules from `../_shared/LANGUAGE.md`.

## Inputs

> "¿Qué sitio querés detener? Puede ser el nombre del sitio o su dominio."

Accept site_name, appName, or primary domain.

## Confirm — with an explicit warning

Stopping a site takes it offline. Make sure the operator understands:

> "Voy a detener **<identifier>**. Mientras esté detenido el sitio no
> va a responder; cualquier visitante va a ver un error. Para
> reactivarlo hay que iniciarlo de vuelta. ¿Confirmás?"

Only proceed on a clear `sí`. If the operator hesitates ("mm", "por
ahora no", "esperá"), offer to cancel:

> "Ok, lo dejamos así. Avisame si después querés detenerlo."

## Dispatch

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
    action: "stop",
    site:   "<the raw identifier>"
  }
})
```

## Track & report

Find the run with `actions_list` (method `list_workflow_runs`), then
`actions_get` (method `get_workflow_run`) for status / conclusion. On
failure, `actions_list` (method `list_workflow_jobs`) tells you which
step failed.

### `success`

> "✅ **<identifier>** detenido. El sitio no va a responder hasta que
> lo inicies de nuevo (podés pedírmelo cuando quieras)."

### `failure`

- "Resolve site" step failed → "No encontré ningún sitio llamado
  **<identifier>**. Revisá el nombre o el dominio."
- Other → "❌ No pude detener el sitio. Pasale este enlace al equipo
  técnico: <run_url>."

## Boundaries

- Do NOT stop internal-infra sites even if the operator asks by
  domain. If the primary domain contains `pmkr.site/internal`,
  `promaker.com.ar` admin, or anything suspicious, refuse: "Ese sitio
  es parte de la infraestructura interna — no se detiene por acá."
- Do not offer to delete the site.
