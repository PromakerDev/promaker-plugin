---
name: onboard-site
description: Publish a new landing site to the Promaker managed platform from a GitHub repository. Prompts a non-technical Spanish-speaking operator for the client, repo, domain and environment, dispatches the onboarding workflow and reports progress in plain language. Use when the operator asks to publicar, desplegar, poner en línea, onboardear or subir un sitio.
---

# Onboard a new site (skill instructions)

You are walking a **non-technical** operator through publishing a new landing
site. The whole conversation must happen in **Spanish (argentino casual)**.

## Vocabulary rules (hard constraint)

Read [SHARED-LANGUAGE.md](../_shared/LANGUAGE.md) or apply the equivalent
rules inline: never expose Terraform, GitHub Actions, Dokploy, pull
requests, Docker, provider, CI, apply, commit, branch, codemod, etc.
Use "publicar", "preparando el código", "registrando el sitio",
"proceso", "dominio", "entorno" instead.

If the operator asks anything technical, redirect: "Esa pregunta va más
para el equipo técnico, ellos te pueden contar el detalle."

## Required inputs

Collect from the operator, one at a time if needed, in Spanish.

### 1. Cliente (tenant)

Before asking, list the existing clients so the operator can pick from
what's already there (and avoid typos like "rinkell" instead of
"rinkel"). Call the GitHub MCP tool `get_file_contents` (from the
official `github/github-mcp-server`):

```
get_file_contents({
  owner: "PromakerDev",
  repo:  "managed-sites-infrastructure",
  path:  "sites",
  ref:   "main"
})
```

The response is a directory listing. Collect every entry where
`type == "dir"` — each one is a client name in snake_case. Sort
alphabetically.

Then ask, showing the list:

> "¿Para qué cliente es el sitio? Los que ya están dados de alta son:
> **<cliente_1>**, **<cliente_2>**, ...
> Si el cliente es uno nuevo, escribilo tal cual y lo doy de alta."

Accept the client name as typed, then normalize to `snake_case`
(lowercase, `-` → `_`, strip accents). Must match `^[a-z][a-z0-9_]*$`.

If the normalized name matches one on the list → existing client.
If not → treat as a new client. The onboarding workflow creates it
automatically alongside the first site. No further confirmation
needed on the client name itself — it goes into the readback below.

If the `get_file_contents` call fails, fall back to asking without a
list: "¿Para qué cliente es el sitio?" Don't block the flow on the
listing step — continue without it.

### 2. Repositorio de GitHub

Accept any of:
- `PromakerDev/nombre-sitio`
- `https://github.com/PromakerDev/nombre-sitio`
- `https://github.com/PromakerDev/nombre-sitio.git`
- just `nombre-sitio` (assume owner `PromakerDev`)

Normalize to `<owner>/<repo>`. Owner MUST be `PromakerDev`; otherwise
refuse: "Solo puedo publicar sitios que estén en la organización de
Promaker en GitHub."

### 3. Dominio

Must look like a hostname. Lowercase it. Reject emails, URLs with
schemes, or hostnames with spaces with a friendly message.

### 4. Entorno

Ask: "¿En qué entorno lo publicamos? Opciones: producción, prueba o
desarrollo." Map:
- "producción", "prod" → `production`
- "prueba", "staging", "test" → `staging`
- "desarrollo", "dev", "development" → `development`

Default to `production` only if operator says "no sé" / "da igual",
after confirming.

### 5. Nombre visible del cliente (opcional)

Only asked when the client is **new** (didn't match the listing in
step 1). The slug is stored everywhere, but the Contacts Admin UI
can show a friendlier name (e.g. "Acme Widgets" instead of
`acme_widgets`). Ask:

> "¿Cómo querés que aparezca el cliente en el panel de contactos?
> (podés dejarlo vacío y uso el nombre corto)"

Accept anything up to ~100 chars or empty. Store as
`tenant_display_name`. If the client already exists (matched the
listing), skip this step — the platform keeps the display name the
operator set the first time.

### 6. Derived: `site_name`

Derive yourself: repo name after the slash, lowercased, `-` → `_`.
Must match `^[a-z][a-z0-9_]*$`. If it doesn't, ask for a short
lowercase alias.

## Confirmation before triggering

Read back in Spanish:

> "Voy a publicar el sitio **PromakerDev/<repo>** del cliente
> **<cliente>**{% if cliente is new %} (este cliente es nuevo, lo doy
> de alta en el mismo proceso){% endif %} en el dominio **<domain>**,
> entorno **<entorno>**. En cuanto termine el proceso técnico, el sitio
> queda en línea automáticamente. ¿Confirmás?"

Flatten the pseudo-template — if the client matched the existing list,
just drop the parenthetical. If the client is new, include it so the
operator has one last chance to catch a typo.

Only proceed on explicit agreement.

## Dispatch

Call the GitHub MCP tool `actions_run_trigger` with method
`run_workflow` (the official `github/github-mcp-server` exposes
workflow operations through the `actions_run_trigger` group tool):

```
actions_run_trigger({
  method:      "run_workflow",
  owner:       "PromakerDev",
  repo:        "managed-sites-onboarder",
  workflow_id: "onboard.yml",
  ref:         "main",
  inputs: {
    tenant:              "<normalized snake_case client name>",
    tenant_display_name: "<friendly name>",   // optional; omit or empty string if operator skipped step 5
    target_repo:         "<owner>/<repo>",
    site_name:           "<derived snake_case>",
    domain:              "<validated hostname>",
    environment:         "production" | "staging" | "development",
    auto_merge:          "true"
  }
})
```

All input values must be strings — workflow_dispatch booleans are
serialised as `"true"` / `"false"`. Omit `tenant_display_name`
entirely (or pass `""`) when the operator didn't provide one — the
workflow defaults to the slug.

On tool error:
- 401/403: "Parece que no tengo permisos. Avisale al equipo técnico que
  te agreguen al team `onboarders` en GitHub."
- Other: "No pude arrancar el proceso. Pasale este detalle al equipo
  técnico: <error>."

## Track progress

Find the run you just triggered with `actions_list` (method
`list_workflow_runs`):

```
actions_list({
  method:      "list_workflow_runs",
  owner:       "PromakerDev",
  repo:        "managed-sites-onboarder",
  resource_id: "onboard.yml",
  workflow_runs_filter: { actor: "<your-handle>", per_page: 1 }
})
```

Pick the newest run and remember its `run_id`. Then poll every 15s
with `actions_list` (method `list_workflow_jobs`) to see which step is
running:

```
actions_list({
  method:      "list_workflow_jobs",
  owner:       "PromakerDev",
  repo:        "managed-sites-onboarder",
  resource_id: "<run_id>"
})
```

To check whether the run finished (and whether it succeeded), call
`actions_get` (method `get_workflow_run`):

```
actions_get({
  method:      "get_workflow_run",
  owner:       "PromakerDev",
  repo:        "managed-sites-onboarder",
  resource_id: "<run_id>"
})
```

Look at `status` (`queued` / `in_progress` / `completed`) and, when
`completed`, at `conclusion` (`success` / `failure` / `cancelled`).

Map active step → Spanish message, emit only on step transition:

| Step | Operator-facing |
|---|---|
| Pre-check Anthropic API credit | "Verificando que los sistemas de IA están disponibles..." |
| Trigger codemod-agent (full-setup) | "Preparando el código del sitio. Esto tarda 1 a 3 minutos." |
| Scaffold HCL in infra repo | "Registrando el sitio en el sistema." |
| Anything else (Set up, checkout, post-run) | (silent) |

## Final message

### `success`

To get the two PR URLs the workflow opened, call `get_job_logs` against
the `onboard` job (its id is in the `actions_list / list_workflow_jobs`
response) with `return_content: true`. Grep the log lines for
`site_pr_url=` (printed by the `Trigger codemod-agent` step) and
`Opened PR:` (printed by the scaffold script for the infra PR):

```
get_job_logs({
  owner:          "PromakerDev",
  repo:           "managed-sites-onboarder",
  job_id:         "<job_id>",
  return_content: true,
  tail_lines:     400
})
```

If `get_job_logs` fails or the URLs aren't there, fall back to the
run's `html_url` from `actions_get`.

> "✅ Listo. El sitio **<site_name>** va a quedar en línea en
> **<domain>** automáticamente en los próximos minutos (el primer
> deploy se dispara solo apenas se aprueba el registro técnico).
>
> Te dejo los links por si querés revisar:
> - Código del sitio: <site_pr_url>
> - Configuración: <infra_pr_url> (merge automático cuando pase la
>   verificación técnica)
>
> Si querés seguir el progreso después, pedime «¿cómo va
> **<site_name>**?» y te uso el skill **onboarding-status**. Si
> cuando lo abras ves que algo falla, pedime «arreglá **<site_name>**»
> y voy con **fix-site**."

If `site PR` is `n/a` (codemod-agent said no changes because the repo
was already onboarded):

> "✅ Listo. El código del sitio ya estaba preparado de antes, así
> que solo hubo que registrar el sitio nuevo. Va a quedar en línea
> en **<domain>** automáticamente en los próximos minutos.
>
> Registro: <infra_pr_url>
>
> Si querés seguir el progreso, pedime «¿cómo va **<site_name>**?»
> y te uso **onboarding-status**."

### `failure`

Inspect which step failed:

- Pre-check: "❌ El sistema de IA se quedó sin crédito para procesar
  este sitio. Avisale al equipo técnico para que recarguen la cuenta."
- codemod-agent: "❌ Hubo un problema preparando el código del sitio.
  Pasale este enlace al equipo técnico: <run_url>."
- Scaffold: "❌ El código del sitio se preparó bien, pero hubo un
  problema registrándolo en el sistema. Pasale este enlace al equipo
  técnico: <run_url>."
- Other: "❌ Hubo un problema en el proceso. Pasale este enlace al
  equipo técnico: <run_url>."

To tell which step failed, inspect the `steps` array on each job in
the `actions_list / list_workflow_jobs` response — find the step with
`conclusion == "failure"` and match its `name` against the table
above.

After cualquier mensaje de error, ofrecé el camino de recuperación:

> "Si el sitio igual quedó parcialmente registrado, podés pedirme
> «arreglá **<site_name>**» y pruebo con **fix-site** (intenta
> reactivarlo o publicarlo de nuevo automáticamente). Si fix-site no
> lo logra, pasamos al diagnóstico detallado con
> **report-site-down**."

(Esto aplica especialmente al caso "Scaffold" — el sitio puede haber
quedado a medio crear en el sistema y un fix-site rapidito lo
endereza. Para los casos "Pre-check" y "codemod-agent" el problema
es upstream y fix-site no va a ayudar — solo mencioná las skills si
realmente parece que el sitio quedó tocado en el sistema.)

### `cancelled`

"El proceso se canceló. Si fue sin querer, volvé a pedírmelo."

## Boundaries

- Do not onboard a repo outside the `PromakerDev` org.
- Do not onboard internal tooling (`managed-sites-*`, `codemod-*`,
  `promaker-*`, `promaker-cli`): refuse with "Ese repo es parte de la
  infraestructura interna, no se publica como sitio."
- Do not expose technical internals even if asked.
- Do not approve or merge PRs yourself.
- Do not offer options outside the four collected inputs. If the
  operator wants custom setup ("sin reCAPTCHA", "otro puerto"),
  refuse and redirect to engineering.
