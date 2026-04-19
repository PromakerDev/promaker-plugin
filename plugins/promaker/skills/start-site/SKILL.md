---
name: start-site
description: Start a previously-deployed site so it responds to traffic again. Use when the operator asks "iniciá X", "arrancá X", "poné X en línea otra vez", "reactivá X".
---

# Start site (skill instructions)

Apply shared language rules from `../_shared/LANGUAGE.md`. Never expose
Terraform, GitHub Actions, Dokploy, etc.

## Inputs

Ask for a single identifier of the site in Spanish:

> "¿Qué sitio querés iniciar? Puede ser el nombre del sitio o su dominio."

Accept:
- site_name (snake_case): `cliente_nuevo`
- appName (Dokploy-assigned short): `web-ke4kjj`
- primary domain: `cliente.pmkr.site`

Pass the raw string to the workflow — the server resolves it.

## Confirm

> "Voy a iniciar **<identifier>**. Si estaba caído o pausado, vuelve a
> estar en línea en unos segundos. ¿Confirmás?"

Do not proceed without explicit yes.

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
    action: "start",
    site:   "<whatever the operator said, verbatim>"
  }
})
```

## Track & report

Poll the run until it completes — `actions_list` (method
`list_workflow_runs`) to find the run, then `actions_get` (method
`get_workflow_run`) to read its `status` and `conclusion`. No
structured output needed beyond `conclusion`.

### `success`

> "✅ Listo, **<identifier>** está iniciado. Dale unos 10–20 segundos
> para que termine de levantarse y probalo."

### `failure`

Fetch the jobs with `actions_list` (method `list_workflow_jobs`,
`resource_id: "<run_id>"`) and look at each step's `conclusion`:

- "Resolve site to application ID" failed → "No encontré ningún sitio
  llamado **<identifier>**. Revisá el nombre o el dominio y probá de
  nuevo."
- Other → "❌ No pude iniciar el sitio. Pasale este enlace al equipo
  técnico: <run_url>."

## Boundaries

- Do not offer to stop, redeploy or delete — those son other skills.
- Do not attempt to guess which site they mean if the input is
  ambiguous. Ask again.
