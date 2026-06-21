# 📘 Guía Completa GitHub Actions - Certificación GH-200

> Material de estudio exhaustivo, dividido por dominios del examen oficial GitHub Actions (GH-200), con ejemplos de código YAML/JS/Bash, tablas comparativas y links oficiales de documentación.

## Estructura de la guía

| Dominio | Peso en el examen | Archivo |
|---------|-------------------|---------|
| Domain 1: Author and Manage Workflows | 20–25% | [`01-domain1-author-manage-workflows.md`](./01-domain1-author-manage-workflows.md) |
| Domain 2: Consume and Troubleshoot Workflows | 15–20% | [`02-domain2-consume-troubleshoot-workflows.md`](./02-domain2-consume-troubleshoot-workflows.md) |
| Domain 3: Author and Maintain Actions | 15–20% | [`03-domain3-author-maintain-actions.md`](./03-domain3-author-maintain-actions.md) |
| Domain 4: Manage GitHub Actions for the Enterprise | 20–25% | [`04-domain4-enterprise-management.md`](./04-domain4-enterprise-management.md) |
| Domain 5: Secure and Optimize Automation | 10–15% | [`05-domain5-secure-optimize-automation.md`](./05-domain5-secure-optimize-automation.md) |

## 📋 Contenido por dominio

### Domain 1 - Author and Manage Workflows
- Triggers: scheduled, manual (`workflow_dispatch`), webhook, repository events
- Permissions y scopes
- `workflow_call` (reusable workflows) con inputs/secrets
- Jobs, steps, lógica condicional, `needs`
- Workflow commands, environment variables
- Service containers (Postgres, Redis, MongoDB)
- Strategy matrix (include/exclude, fail-fast, max-parallel)
- YAML anchors & aliases
- Contexts (`github`, `runner`, `env`, `vars`, `secrets`, `inputs`, `matrix`, `needs`, `strategy`, `job`, `steps`)
- Expressions `${{ }}`, evaluación estática vs runtime
- Immutable actions y version pinning (SHA)
- Caching, artifacts, retention policies via REST API
- `GITHUB_STEP_SUMMARY`, badges, environment protections

### Domain 2 - Consume and Troubleshoot Workflows
- Interpretar triggers y resultados desde logs
- Debugging de workflows fallidos
- Matrix expansions y re-run de jobs individuales
- Acceso a artifacts y logs (UI, CLI, API)
- Starter workflows vs Reusable workflows vs Composite actions
- Disable vs Delete de workflows

### Domain 3 - Author and Maintain Actions
- Tipos de actions: JavaScript, Docker, Composite
- Estructura de `action.yml` y metadata
- Workflow commands dentro de actions (`@actions/core`)
- Distribución (pública, privada, marketplace)
- Publicación en GitHub Marketplace
- Versionado semántico y release strategies

### Domain 4 - Manage GitHub Actions for the Enterprise
- Gobernanza de actions y workflows a nivel organización
- Políticas de uso organizacional
- GitHub-hosted vs Self-hosted runners
- Runner groups, IP allow lists, networking
- Software preinstalado en runners e instalación adicional
- Troubleshooting de runners
- Secrets y variables (repo, environment, organización) + REST API

### Domain 5 - Secure and Optimize Automation
- Environment protections y approval gates
- Trustworthy actions del Marketplace
- Mitigación de script injection
- `GITHUB_TOKEN` lifecycle, permisos, vs PAT
- OIDC para AWS / Azure / GCP (sin long-lived secrets)
- Pin de actions a SHA + Dependabot
- Políticas de uso de actions (allow/deny lists)
- Artifact attestations / SLSA provenance
- Optimización de costos y performance (cache, matrix, concurrency, sharding)

## Recursos generales

- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Workflow syntax reference](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [GitHub Actions REST API](https://docs.github.com/en/rest/actions)
- [actions/starter-workflows](https://github.com/actions/starter-workflows)
- [actions/toolkit](https://github.com/actions/toolkit)
- [actions/runner-images](https://github.com/actions/runner-images)
- [GitHub CLI](https://cli.github.com/)
- [GitHub Certifications](https://docs.github.com/en/get-started/showcase-your-expertise-with-github-certifications/about-github-certifications)

## Tips de examen

- La certificación pone fuerte énfasis en sintaxis YAML exacta - practicá escribiendo workflows de memoria.
- Entendé bien la diferencia entre **starter workflow**, **reusable workflow** y **composite action** - es un tema recurrente.
- Dominá el ciclo de vida del `GITHUB_TOKEN`, sus permisos por defecto y cuándo usar OIDC en lugar de secrets de cloud.
- Practicá con `matrix` (include/exclude/fail-fast) - suelen pedir interpretar resultados de expansión.
- Repasá los workflow commands (`GITHUB_OUTPUT`, `GITHUB_ENV`, `GITHUB_STEP_SUMMARY`, `::add-mask::`, etc.) porque aparecen en preguntas de troubleshooting.

---

*Última actualización: Junio 2026*
