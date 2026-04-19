---
name: onboarding-status
description: Report the status of in-progress or recent site onboardings in plain Spanish for a non-technical operator. Use when the operator asks "¿cómo va el onboarding de X?", "¿qué hay corriendo?", "¿se publicó ya el sitio X?".
---

# Onboarding status (skill instructions)

Apply the shared language rules from [`../_shared/LANGUAGE.md`]. Never
expose Terraform, GitHub Actions, workflows, PRs, etc.

## When to use

The operator wants to know the state of an onboarding they or someone
else triggered. Typical triggers:

- "¿cómo va el onboarding de <sitio>?"
- "¿se publicó ya el sitio <nombre>?"
- "¿hay algo corriendo ahora?"
- "¿qué pasó con el último sitio que onboardeé?"

## Inputs

Optional: a site identifier (repo name, site_name or domain).
If omitted, report ALL runs from the last ~24h.

## Query

Use the official `github/github-mcp-server` MCP tools. Workflow
operations live in the `actions_*` group tools — pass the operation
name in `method`:

```
actions_list({
  method:      "list_workflow_runs",
  owner:       "PromakerDev",
  repo:        "managed-sites-onboarder",
  resource_id: "onboard.yml",
  workflow_runs_filter: { per_page: 10, event: "workflow_dispatch" }
})
```

For each run, get the step-by-step state with:

```
actions_list({
  method:      "list_workflow_jobs",
  owner:       "PromakerDev",
  repo:        "managed-sites-onboarder",
  resource_id: "<run_id>"
})
```

Each returned job has a `steps` array; pick the step whose
`status == "in_progress"` (or the latest `completed` one) to know
what's happening right now.

If the operator named a specific site, filter runs whose
`display_title` or inputs reference that site. Match loosely — their
phrasing may differ from the exact `site_name`.

## Report

For each run you report on, translate internal states:

| Run state | Active step | Say |
|---|---|---|
| `in_progress` | Pre-check Anthropic API credit | "Está arrancando — verificando sistemas" |
| `in_progress` | Trigger codemod-agent | "Preparando el código del sitio" |
| `in_progress` | Scaffold HCL in infra repo | "Registrando el sitio" |
| `completed` + `success` | — | "Publicado ✅" |
| `completed` + `failure` | — | "Hubo un problema ❌" |
| `completed` + `cancelled` | — | "Cancelado" |

Format each run as one line. Include:
- Site name (from inputs or display title)
- Current state
- Elapsed / started-at
- If failed: a one-line human reason (see `onboard-site/SKILL.md` for
  failure mapping) + run URL for engineering

Example output (no runs scoped):

> "Tenés estos onboardings recientes:
>
> - **guille_test** → Publicado ✅ (hace 20 min)
> - **cliente_nuevo** → Preparando el código del sitio (hace 2 min)
> - **test_site** → Hubo un problema ❌ (hace 1 h) → `<run_url>`"

Example output (scoped to `cliente_nuevo`):

> "El onboarding de **cliente_nuevo** está preparando el código del
> sitio. Arrancó hace 2 minutos. Normalmente tarda entre 1 y 3 minutos
> en total; te aviso cuando termine si seguís acá."

If the site doesn't match any recent run:

> "No encontré ningún onboarding reciente de **<nombre>**. ¿Querés que
> lo arranque ahora?"

## Boundaries

- No technical step names.
- No URLs beyond the run URL for failures.
- If there's nothing in flight and the operator seems to expect one,
  gently offer to start one.
