# Domain 2: Consume and Troubleshoot Workflows (15–20%)
 
> Guía completa para la certificación GitHub Actions GH-200
 
## 📑 Tabla de Contenidos
 
- [2.1 Interpret workflow behavior and results](#21-interpret-workflow-behavior-and-results)
- [2.2 Access workflow artifacts and logs](#22-access-workflow-artifacts-and-logs)
- [2.3 Use and manage workflow templates](#23-use-and-manage-workflow-templates)
---
 
## 2.1 Interpret workflow behavior and results
 
### Identificar triggers y efectos

```yaml
# Ver qué activó el workflow
jobs:
  debug:
    runs-on: ubuntu-latest
    steps:
      - name: Show trigger info
        run: |
          echo "Event: ${{ github.event_name }}"
          echo "Actor: ${{ github.actor }}"
          echo "Ref: ${{ github.ref }}"
          echo "SHA: ${{ github.sha }}"
 
          # Para pull_request
          if [ "${{ github.event_name }}" == "pull_request" ]; then
            echo "PR Number: ${{ github.event.pull_request.number }}"
            echo "PR Title: ${{ github.event.pull_request.title }}"
          fi
 
          # Para push
          if [ "${{ github.event_name }}" == "push" ]; then
            echo "Pusher: ${{ github.event.pusher.name }}"
            echo "Commits: ${{ github.event.commits[0].message }}"
          fi
```

**Recursos:**
- [Events that trigger workflows](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)
- [GitHub context](https://docs.github.com/en/actions/learn-github-actions/contexts#github-context)

### Diagnosticar workflow runs fallidos
 
#### Estrategias de debugging
 
```yaml
name: Debug Workflow
on: [push]
 
jobs:
  debug:
    runs-on: ubuntu-latest
    steps:
      # 1. Enable debug logging
      - name: Enable debug
        run: |
          echo "ACTIONS_STEP_DEBUG=true" >> $GITHUB_ENV
          echo "ACTIONS_RUNNER_DEBUG=true" >> $GITHUB_ENV
 
      # 2. Dump contexts
      - name: Dump GitHub context
        env:
          GITHUB_CONTEXT: ${{ toJSON(github) }}
        run: echo "$GITHUB_CONTEXT"
 
      - name: Dump runner context
        env:
          RUNNER_CONTEXT: ${{ toJSON(runner) }}
        run: echo "$RUNNER_CONTEXT"
 
      # 3. Check environment
      - name: Environment info
        run: |
          echo "OS: $(uname -a)"
          echo "Shell: $SHELL"
          echo "User: $(whoami)"
          echo "PWD: $(pwd)"
          env | sort
 
      # 4. Conditional debugging
      - name: Debug on failure
        if: failure()
        run: |
          echo "::group::Debug Information"
          echo "Job status: ${{ job.status }}"
          echo "Run ID: ${{ github.run_id }}"
          echo "::endgroup::"
```

#### Analizar logs
 
```bash
# Descargar logs via API
curl -L \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/OWNER/REPO/actions/runs/RUN_ID/logs \
  -o logs.zip
 
# Ver logs de un job específico
gh run view RUN_ID --log
 
# Ver logs en tiempo real
gh run watch RUN_ID
```

**Recursos:**
- [Enabling debug logging](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/enabling-debug-logging)
- [Using workflow run logs](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/using-workflow-run-logs)
- [GitHub CLI - run commands](https://cli.github.com/manual/gh_run)

### Interpretar Matrix expansions
 
Cuando usas matrix, GitHub genera un job por cada combinación:
 
```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [18, 20]
    include:
      - os: ubuntu-latest
        node: 22
    exclude:
      - os: windows-latest
        node: 18
 
# Esto genera 4 jobs:
# 1. ubuntu-latest + node 18
# 2. ubuntu-latest + node 20
# 3. windows-latest + node 20  (18 excluido)
# 4. ubuntu-latest + node 22  (incluido)
```
 
#### Correlacionar job names con matrix values
 
```yaml
jobs:
  test:
    name: Test on ${{ matrix.os }} with Node ${{ matrix.node }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: [18, 20]
    runs-on: ${{ matrix.os }}
    steps:
      - run: node --version
```

#### Re-run jobs individuales
 
En la UI de GitHub, puedes:
1. Ver todos los jobs del matrix
2. Seleccionar jobs fallidos específicos
3. Hacer "Re-run failed jobs" o "Re-run job" individual

**Recursos:**
- [Using a matrix for your jobs](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs)
- [Re-running workflows and jobs](https://docs.github.com/en/actions/managing-workflow-runs/re-running-workflows-and-jobs)

---
 
## 2.2 Access workflow artifacts and logs
 
### Localizar workflows, logs y artifacts
 
**En la UI:**
1. Ve a tu repositorio
2. Click en "Actions" tab
3. Selecciona un workflow run
4. Click en un job para ver logs
5. Scroll down para ver artifacts
**Via GitHub CLI:**
 
```bash
# Listar workflow runs
gh run list --workflow=ci.yml
 
# Ver detalles de un run
gh run view 123456789
 
# Ver logs
gh run view 123456789 --log
 
# Descargar artifacts
gh run download 123456789
 
# Listar artifacts
gh api repos/OWNER/REPO/actions/artifacts
```
 
**Via REST API:**
 
```bash
# Listar workflows
curl -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/OWNER/REPO/actions/workflows
 
# Listar runs de un workflow
curl -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/OWNER/REPO/actions/workflows/WORKFLOW_ID/runs
 
# Obtener artifact
curl -L -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/OWNER/REPO/actions/artifacts/ARTIFACT_ID/zip \
  -o artifact.zip
```
 
**Recursos:**
- [Managing workflow runs](https://docs.github.com/en/actions/managing-workflow-runs)
- [Downloading workflow artifacts](https://docs.github.com/en/actions/managing-workflow-runs/downloading-workflow-artifacts)
- [Actions REST API](https://docs.github.com/en/rest/actions)

### Download y manage artifacts
 
```yaml
jobs:
  download-artifacts:
    runs-on: ubuntu-latest
    steps:
      # Descargar artifact específico
      - uses: actions/download-artifact@v4
        with:
          name: my-artifact
          path: ./downloads
 
      # Descargar todos los artifacts
      - uses: actions/download-artifact@v4
        with:
          path: ./all-artifacts
 
      # Procesar artifacts
      - name: Process artifacts
        run: |
          ls -la ./downloads
          unzip ./downloads/data.zip
```

```bash
#!/bin/bash
# manage-artifacts.sh
 
REPO="owner/repo"
TOKEN="$GITHUB_TOKEN"
 
# Función para listar artifacts
list_artifacts() {
  curl -s -H "Authorization: token $TOKEN" \
    "https://api.github.com/repos/$REPO/actions/artifacts" \
    | jq '.artifacts[] | {name, id, size_in_bytes, created_at}'
}
 
# Función para eliminar artifact
delete_artifact() {
  local artifact_id=$1
  curl -X DELETE \
    -H "Authorization: token $TOKEN" \
    "https://api.github.com/repos/$REPO/actions/artifacts/$artifact_id"
}
 
# Función para descargar artifact
download_artifact() {
  local artifact_id=$1
  local output_file=$2
  curl -L -H "Authorization: token $TOKEN" \
    "https://api.github.com/repos/$REPO/actions/artifacts/$artifact_id/zip" \
    -o "$output_file"
}
 
# Uso
list_artifacts
# download_artifact 123456 my-artifact.zip
# delete_artifact 123456
```
 
**Recursos:**
- [Artifacts REST API](https://docs.github.com/en/rest/actions/artifacts)
- [actions/download-artifact](https://github.com/actions/download-artifact)

---
 
## 2.3 Use and manage workflow templates
 
### Organization-level workflow templates
 
Los **starter workflows** son plantillas que aparecen cuando creas un nuevo workflow.
 
**Estructura de organización:**

```
.github/
  workflow-templates/
    ci-nodejs.yml              # Template workflow
    ci-nodejs.properties.json  # Metadata
    ci-python.yml
    ci-python.properties.json
```
 
**Ejemplo de template:**
 
```yaml
# .github/workflow-templates/ci-nodejs.yml
name: Node.js CI
 
on:
  push:
    branches: [ $default-branch ]
  pull_request:
    branches: [ $default-branch ]
 
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]
 
    steps:
      - uses: actions/checkout@v4
      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm run build --if-present
      - run: npm test
```

**Metadata file:**
 
```json
{
  "name": "Node.js CI Starter",
  "description": "CI workflow template for Node.js projects",
  "iconName": "nodejs",
  "categories": ["JavaScript", "CI"],
  "filePatterns": [
    "package.json$"
  ]
}
```

**Recursos:**
- [Creating starter workflows](https://docs.github.com/en/actions/using-workflows/creating-starter-workflows-for-your-organization)
- [Sharing workflows with your organization](https://docs.github.com/en/actions/using-workflows/sharing-workflows-secrets-and-runners-with-your-organization)

**Diferencias clave:**
 
| Aspecto | Starter Workflow | Reusable Workflow | Composite Action |
|---------|------------------|-------------------|-------------------|
| **Ubicación** | `.github/workflow-templates/` | `.github/workflows/` | Cualquier lugar (repo/marketplace) |
| **Uso** | Copiado al crear workflow | Invocado con `uses` en job | Invocado con `uses` en step |
| **Actualización** | Independiente después de creado | Centralizado, se actualiza automáticamente | Centralizado |
| **Scope** | Workflow completo | Job level | Step level |
| **Inputs** | No | Sí | Sí |
| **Secrets** | No | Sí | No (usa secrets del caller) |
| **Runners** | Define su propio runner | Define su propio runner | Usa runner del workflow |

**Starter Workflow:**
```yaml
# Solo un scaffold inicial
# Usuario lo copia y modifica
name: CI
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test
```

**Reusable Workflow:**
```yaml
# .github/workflows/reusable.yml
# Centralizado, versionado, invocable
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
 
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy to ${{ inputs.environment }}"
```
 
```yaml
# Caller
jobs:
  prod:
    uses: ./.github/workflows/reusable.yml
    with:
      environment: production
```

**Composite Action:**
```yaml
# actions/setup-app/action.yml
name: Setup Application
description: Setup app with cache
inputs:
  node-version:
    required: true
 
runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
    - run: npm ci
      shell: bash
```
 
```yaml
# Caller
steps:
  - uses: ./actions/setup-app
    with:
      node-version: '20'
```

**Recursos:**
- [Reusing workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [Creating a composite action](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action)

### Deshabilitar vs Eliminar workflows
 
**Deshabilitar un workflow:**
- El archivo `.yml` permanece en el repositorio
- No se ejecutará automáticamente
- Puede ser re-habilitado en cualquier momento
- Los runs históricos permanecen
```bash
# En la UI: Actions → Select workflow → "..." → Disable workflow
 
# Via API
curl -X PUT \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/OWNER/REPO/actions/workflows/WORKFLOW_ID/disable
```













