# Domain 1: Author and Manage Workflows (20-25%)

---

## 1.1 Configure workflow triggers and events

### ¿Qué son los triggers?

Los triggers son eventos que inician la ejecución de un workflow. GitHub Actions soporta múltiples tipos de eventos que pueden activar tus automatizaciones.

Tipos de triggers principales:

1. Scheduled Events (Eventos Programados)
Usa la sintaxis cron para ejecutar workflows en horarios específicos.

```yaml
name: Scheduled Workflow
on:
  schedule:
    # Ejecuta todos los días a las 2:30 AM UTC
    - cron: '30 2 * * *'
    # Ejecuta cada lunes a las 9 AM UTC
    - cron: '0 9 * * 1'
```

Sintaxis cron:

- * * * * * = minuto hora día mes día-semana
- Ejemplos:

    - 0 0 * * * - Medianoche todos los días
    - */15 * * * * - Cada 15 minutos
    - 0 9-17 * * 1-5 - Cada hora de 9am a 5pm, lunes a viernes

2. Manual Events (Eventos Manuales)
Permite ejecutar workflows manualmente desde la interfaz de GitHub o mediante la API.

```yaml
name: Manual Deployment
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        default: 'staging'
        type: choice
        options:
          - development
          - staging
          - production
      version:
        description: 'Version to deploy'
        required: true
        type: string
      enable-debug:
        description: 'Enable debug logging'
        required: false
        type: boolean
        default: false
      log-level:
        description: 'Log level'
        required: true
        type: choice
        options:
          - info
          - warning
          - debug
```

Tipos de inputs disponibles:

- string - Texto libre
- choice - Menú desplegable con opciones predefinidas
- boolean - Checkbox verdadero/falso
- environment - Selector de ambiente (con protecciones configuradas)

Acceso a los inputs en el workflow:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to environment
        run: |
          echo "Deploying version ${{ inputs.version }} to ${{ inputs.environment }}"
          if [ "${{ inputs.enable-debug }}" == "true" ]; then
            echo "Debug mode enabled"
          fi
```

3. Webhook Events (Eventos de Webhook)
Los webhooks se activan por eventos del repositorio como push, pull request, issues, etc

```yaml
name: CI Pipeline
on:
  push:
    branches:
      - main
      - 'releases/**'
    paths:
      - 'src/**'
      - 'tests/**'
    paths-ignore:
      - 'docs/**'
      - '**.md'
  pull_request:
    types: [opened, synchronize, reopened]
    branches:
      - main
  pull_request_target:  # Para PRs de forks con permisos
    types: [opened, synchronize]
  issue_comment:
    types: [created, edited]
  issues:
    types: [opened, labeled]
  release:
    types: [published, created]
```

Eventos webhook más importantes:

Evento | Cúando se activa | Uso común |
-------|------------------|-----------|
push | Al hacer push a una rama | CI/CD, tests |
pull_request | Acciones en PRs |Code review, tests |
pull_request_target | PR de fork con permisos |Build de PRs externos |
issues | Operaciones en issues | Automatización de triaje |
release | Publicación de release | Despliegues | 
workflow_run | Otro workflow completa | Workflows encadenados |

4. Repository Events
Eventos relacionados con cambios en el repositorio:

```yaml
name: Repository Events
on:
  create:  # Branch o tag creado
  delete:  # Branch o tag eliminado
  fork:    # Repositorio forkeado
  gollum:  # Wiki actualizada
  page_build:  # GitHub Pages construido
  public:  # Repositorio hecho público
  watch:   # Repositorio marcado con estrella
```

### Scopes, Permissions y Events apropiados

```yaml
name: Secure Workflow
on: [push]

# Permisos a nivel de workflow (todos los jobs)
permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  job1:
    runs-on: ubuntu-latest
    # Permisos específicos del job (sobrescriben workflow)
    permissions:
      contents: write
      packages: write
    steps:
      - uses: actions/checkout@v4
```

Niveles de permisos:

- read - Solo lectura
- write - Lectura y escritura
- none - Sin acceso

Permisos disponibles:

- actions, checks, contents, deployments, id-token, issues, discussions, packages, pages, pull-requests, repository-projects, security-events, statuses

#### Reusable Workflows con workflow_call

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy Workflow
on:
  workflow_call:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        type: string
      version:
        required: false
        type: string
        default: 'latest'
      dry-run:
        required: false
        type: boolean
        default: false
    secrets:
      deploy-token:
        required: true
      api-key:
        required: false
    outputs:
      deployment-url:
        description: "URL of the deployment"
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    steps:
      - name: Deploy
        id: deploy
        run: |
          echo "Deploying to ${{ inputs.environment }}"
          echo "url=https://${{ inputs.environment }}.example.com" >> $GITHUB_OUTPUT
        env:
          DEPLOY_TOKEN: ${{ secrets.deploy-token }}
```

#### Llamada al workflow reutilizable:

```yaml
# .github/workflows/main.yml
name: Main Workflow
on: [push]

jobs:
  call-deploy:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
      version: v1.2.3
      dry-run: false
    secrets:
      deploy-token: ${{ secrets.DEPLOY_TOKEN }}
      api-key: ${{ secrets.API_KEY }}
  
  use-output:
    needs: call-deploy
    runs-on: ubuntu-latest
    steps:
      - name: Use deployment URL
        run: echo "Deployed to ${{ needs.call-deploy.outputs.deployment-url }}"
```

---

## 1.2 Design and implement workflow structure

### Jobs, Steps y Conditional Logic

Estructura básica:

```yaml
name: Complete Workflow Structure
on: [push]

jobs:
  # Job 1: Build
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
  
  # Job 2: Test (conditional)
  test:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - name: Run tests
        run: npm test
```

#### Conditional Logic (Lógica Condicional)

A nivel de job:
```yaml
jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to staging"
  
  deploy-production:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to production"
  
  skip-on-bot:
    if: github.actor != 'dependabot[bot]'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Not a bot"
```

A nivel de step:
```yaml
jobs:
  conditional-steps:
    runs-on: ubuntu-latest
    steps:
      - name: Always runs
        run: echo "This always runs"
      
      - name: Only on main branch
        if: github.ref == 'refs/heads/main'
        run: echo "Main branch"
      
      - name: Only on failure
        if: failure()
        run: echo "Previous step failed"
      
      - name: Only on success
        if: success()
        run: echo "All previous steps succeeded"
      
      - name: Always run cleanup
        if: always()
        run: echo "Cleanup"
      
      - name: Only if cancelled
        if: cancelled()
        run: echo "Workflow was cancelled"
```

Funciones de estado:

- success() - Todos los pasos anteriores exitosos
- failure() - Algún paso anterior falló
- cancelled() - Workflow cancelado
- always() - Siempre ejecuta

### Dependencies between jobs

Dependencias con `needs`:

```yaml
jobs:
  # Job 1: Setup
  setup:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Setting up"
  
  # Job 2: Build (depende de setup)
  build:
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building"
  
  # Job 3: Test (depende de build)
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Testing"
  
  # Job 4: Deploy (depende de build y test)
  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying"
  
  # Job 5: Siempre se ejecuta (incluso si test falla)
  notify:
    needs: [build, test, deploy]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: echo "Sending notification"
```

Diagrama de dependencias:

```bash
setup → build → test → deploy → notify
              ↘      ↗
```

Condicionales con needs:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test
  
  deploy:
    needs: test
    # Solo deploy si test fue exitoso
    if: needs.test.result == 'success'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying"
  
  cleanup:
    needs: test
    # Cleanup incluso si test falló
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: echo "Cleanup"
```

### Workflow Commands y Environment Variables

Environment Variables (Variables de Entorno)

Definición a diferentes niveles:

```yaml
name: Environment Variables
on: [push]

# Variables a nivel de workflow
env:
  WORKFLOW_VAR: 'workflow-level'
  NODE_VERSION: '20'

jobs:
  example:
    runs-on: ubuntu-latest
    # Variables a nivel de job
    env:
      JOB_VAR: 'job-level'
      BUILD_ENV: 'production'
    
    steps:
      - name: Step with env vars
        # Variables a nivel de step
        env:
          STEP_VAR: 'step-level'
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          echo "Workflow var: $WORKFLOW_VAR"
          echo "Job var: $JOB_VAR"
          echo "Step var: $STEP_VAR"
          echo "Node version: $NODE_VERSION"
```

Variables de entorno predefinidas:

```yaml 
steps:
  - name: Show GitHub environment variables
    run: |
      echo "Repository: $GITHUB_REPOSITORY"
      echo "Ref: $GITHUB_REF"
      echo "SHA: $GITHUB_SHA"
      echo "Actor: $GITHUB_ACTOR"
      echo "Workflow: $GITHUB_WORKFLOW"
      echo "Run ID: $GITHUB_RUN_ID"
      echo "Run Number: $GITHUB_RUN_NUMBER"
```

### Workflow Commands
Los workflow commands permiten comunicarte con el runner durante la ejecución.

1. Setting outputs (GITHUB_OUTPUT):
```yaml 
steps:
  - name: Set output
    id: set-output
    run: |
      echo "version=1.2.3" >> $GITHUB_OUTPUT
      echo "release-date=$(date +'%Y-%m-%d')" >> $GITHUB_OUTPUT
  
  - name: Use output
    run: |
      echo "Version: ${{ steps.set-output.outputs.version }}"
      echo "Date: ${{ steps.set-output.outputs.release-date }}"
```

2. Setting environment variables (GITHUB_ENV):
```yaml 
steps:
  - name: Set env var
    run: |
      echo "MY_VAR=hello" >> $GITHUB_ENV
      echo "DEPLOY_URL=https://example.com" >> $GITHUB_ENV
  
  - name: Use env var
    run: |
      echo "Value: $MY_VAR"
      echo "URL: $DEPLOY_URL"
```

3. Setting PATH:
```yaml 
steps:
  - name: Add to PATH
    run: echo "$HOME/.local/bin" >> $GITHUB_PATH
  
  - name: Command now available
    run: my-custom-command
```

4. Job summaries (GITHUB_STEP_SUMMARY):
```yaml 
steps:
  - name: Generate summary
    run: |
      echo "## Test Results 🎯" >> $GITHUB_STEP_SUMMARY
      echo "" >> $GITHUB_STEP_SUMMARY
      echo "- ✅ 45 tests passed" >> $GITHUB_STEP_SUMMARY
      echo "- ❌ 2 tests failed" >> $GITHUB_STEP_SUMMARY
      echo "" >> $GITHUB_STEP_SUMMARY
      echo "### Coverage" >> $GITHUB_STEP_SUMMARY
      echo "**87%** line coverage" >> $GITHUB_STEP_SUMMARY
```

5. Grouping log lines:
```yaml 
steps:
  - name: Group logs
    run: |
      echo "::group::Building application"
      npm run build
      echo "::endgroup::"
      
      echo "::group::Running tests"
      npm test
      echo "::endgroup::"
```

6. Setting debug, notice, warning, error messages:
```yaml 
steps:
  - name: Messages
    run: |
      echo "::debug::This is a debug message"
      echo "::notice::This is a notice"
      echo "::warning::This is a warning"
      echo "::error::This is an error"
```

7. Masking values (ocultar secretos en logs):
```yaml 
steps:
  - name: Mask secret
    run: |
      SECRET_VALUE="my-secret-123"
      echo "::add-mask::$SECRET_VALUE"
      echo "Value is: $SECRET_VALUE"  # Se mostrará como ***
```

8. Stop commands:
```yaml 
steps:
  - name: Stop processing commands
    run: |
      echo "::stop-commands::pause-token"
      echo "::warning::This won't be processed"
      echo "::pause-token::"
      echo "::warning::This will be processed"
```

### Service Containers

Los service containers te permiten ejecutar servicios como bases de datos, caches, o colas de mensajes durante tus workflows.
