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

#### Ejemplo completo con PostgreSQL
 
```yaml
name: CI with Database
on: [push]
 
jobs:
  test:
    runs-on: ubuntu-latest
 
    # Definir service containers
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
 
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
 
    steps:
      - uses: actions/checkout@v4
 
      - name: Run tests
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
        run: |
          npm install
          npm test
```

#### Service container con MongoDB
 
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mongodb:
        image: mongo:7
        env:
          MONGO_INITDB_ROOT_USERNAME: admin
          MONGO_INITDB_ROOT_PASSWORD: secret
        ports:
          - 27017:27017
        options: >-
          --health-cmd "mongosh --eval 'db.adminCommand({ping: 1})'"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 3
 
    steps:
      - name: Test with MongoDB
        env:
          MONGODB_URI: mongodb://admin:secret@localhost:27017
        run: npm test
```

**Opciones importantes:**
 
| Opción | Descripción |
|--------|-------------|
| `image` | Imagen de Docker a usar |
| `env` | Variables de entorno para el container |
| `ports` | Mapeo de puertos (host:container) |
| `options` | Opciones de Docker adicionales |
| Health checks | Aseguran que el servicio esté listo antes de los tests |

**Recursos:**
- [About service containers](https://docs.github.com/en/actions/using-containerized-services/about-service-containers)
- [Creating PostgreSQL service containers](https://docs.github.com/en/actions/using-containerized-services/creating-postgresql-service-containers)
- [Creating Redis service containers](https://docs.github.com/en/actions/using-containerized-services/creating-redis-service-containers)

### Strategy Matrix
 
La **matriz de estrategia** permite ejecutar un job múltiples veces con diferentes configuraciones.

#### Ejemplo básico
 
```yaml
name: Matrix Strategy
on: [push]
 
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [18, 20, 22]
        # Esto generará 9 jobs (3 OS × 3 versiones)
 
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
 
      - name: Test on ${{ matrix.os }}
        run: npm test
```

#### Include y Exclude
 
```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node-version: [18, 20, 22]
 
    # Agregar combinaciones específicas
    include:
      # Prueba experimental con Node 23 solo en Ubuntu
      - os: ubuntu-latest
        node-version: 23
        experimental: true
 
      # Configuración especial para Windows + Node 20
      - os: windows-latest
        node-version: 20
        npm-version: 9
 
    # Excluir combinaciones no deseadas
    exclude:
      # No probar Node 18 en macOS
      - os: macos-latest
        node-version: 18
```

#### Fail-fast y max-parallel
 
```yaml
strategy:
  # Por defecto, fail-fast es true (si un job falla, cancela los demás)
  fail-fast: false  # Continuar aunque fallen algunos jobs
 
  # Limitar trabajos concurrentes (ahorro de costos)
  max-parallel: 2
 
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node-version: [18, 20, 22]
```

#### Ejemplo completo con múltiples dimensiones
 
```yaml
name: Comprehensive Matrix
on: [push]
 
jobs:
  test:
    runs-on: ${{ matrix.os }}
    continue-on-error: ${{ matrix.experimental || false }}
 
    strategy:
      fail-fast: false
      max-parallel: 4
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.9', '3.10', '3.11', '3.12']
 
        include:
          # Python 3.13 experimental solo en Ubuntu
          - os: ubuntu-latest
            python-version: '3.13'
            experimental: true
 
          # Configuración de arquitectura específica
          - os: ubuntu-latest
            python-version: '3.12'
            architecture: 'arm64'
 
        exclude:
          # Python 3.9 está deprecado en macOS más reciente
          - os: macos-latest
            python-version: '3.9'
 
    steps:
      - uses: actions/checkout@v4
 
      - name: Setup Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          architecture: ${{ matrix.architecture || 'x64' }}
 
      - name: Display matrix values
        run: |
          echo "OS: ${{ matrix.os }}"
          echo "Python: ${{ matrix.python-version }}"
          echo "Experimental: ${{ matrix.experimental }}"
```

#### Consideraciones de runners (Ubuntu 20.04 deprecation, Windows Server 2025)
 
```yaml
jobs:
  test:
    strategy:
      matrix:
        include:
          # Ubuntu: Usa ubuntu-latest (actualmente 22.04)
          # Evita ubuntu-20.04 (deprecado)
          - os: ubuntu-latest
            runner-label: ubuntu-22.04
 
          # Windows: windows-latest migrará a Windows Server 2025
          # Si necesitas Server 2022 específicamente:
          - os: windows-2022
            runner-label: windows-2022
 
          # Para preparar migración a 2025:
          - os: windows-latest
            runner-label: windows-2025
            experimental: true
 
    runs-on: ${{ matrix.os }}
    continue-on-error: ${{ matrix.experimental || false }}
 
    steps:
      - name: Show runner info
        run: |
          echo "Runner: ${{ matrix.runner-label }}"
          echo "OS: ${{ runner.os }}"
```

**Recursos:**
- [Using a matrix for your jobs](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs)
- [Workflow syntax - strategy](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstrategy)
- [Ubuntu 20.04 deprecation notice](https://github.blog/changelog/2024-09-16-notice-of-ubuntu-20-04-deprecation-on-github-actions/)
- [GitHub Changelog](https://github.blog/changelog/)


### YAML Anchors y Aliases
 
Los **anchors** (`&`) y **aliases** (`*`) permiten reutilizar configuraciones dentro de un mismo archivo YAML.

#### Ejemplo básico
 
```yaml
name: YAML Anchors Example
on: [push]
 
# Definir configuración común con anchor
.common-env: &common-env
  NODE_ENV: production
  LOG_LEVEL: info
  TIMEOUT: 3000
 
.common-steps: &common-steps
  - name: Checkout
    uses: actions/checkout@v4
  - name: Setup Node
    uses: actions/setup-node@v4
    with:
      node-version: '20'
 
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      # Usar alias para reutilizar configuración
      <<: *common-env
      BUILD_TYPE: release
 
    steps:
      # Reutilizar steps comunes
      - *common-steps
      - name: Build
        run: npm run build
 
  test:
    runs-on: ubuntu-latest
    env:
      <<: *common-env
      BUILD_TYPE: test
 
    steps:
      - *common-steps
      - name: Test
        run: npm test
```

#### Merge de múltiples anchors
 
```yaml
.base-env: &base-env
  LOG_LEVEL: info
  TIMEOUT: 3000
 
.production-env: &production-env
  NODE_ENV: production
  CACHE_ENABLED: true
 
.staging-env: &staging-env
  NODE_ENV: staging
  DEBUG: true
 
jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    env:
      # Merge de múltiples anchors
      <<: [*base-env, *production-env]
      REGION: us-east-1
    steps:
      - run: echo "Deploy to prod"
 
  deploy-staging:
    runs-on: ubuntu-latest
    env:
      <<: [*base-env, *staging-env]
      REGION: us-west-2
    steps:
      - run: echo "Deploy to staging"
```

#### Ejemplo complejo con steps reutilizables
 
```yaml
name: Complex Anchors
on: [push]
 
# Definir conjuntos de steps reutilizables
.setup-steps: &setup-steps
  - uses: actions/checkout@v4
  - uses: actions/setup-node@v4
    with:
      node-version: '20'
      cache: 'npm'
  - run: npm ci
 
.test-steps: &test-steps
  - run: npm run lint
  - run: npm run type-check
  - run: npm test
 
.deploy-steps: &deploy-steps
  - name: Build
    run: npm run build
  - name: Deploy
    run: npm run deploy
 
jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - *setup-steps
      - *test-steps
 
  build-and-deploy:
    needs: lint-and-test
    runs-on: ubuntu-latest
    steps:
      - *setup-steps
      - *deploy-steps
```

**Recursos:**
- [Advanced YAML syntax (YAML spec)](https://yaml.org/spec/1.2.2/#3222-anchors-and-aliases)
- [GitHub Actions workflow syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)

### Contexts y Predefined Contexts

Los **contexts** son objetos que contienen información sobre el workflow, runner, jobs, y steps.

#### Principales contexts

**1. `github` context:**
 
```yaml
steps:
  - name: GitHub context
    run: |
      echo "Repository: ${{ github.repository }}"
      echo "Repository owner: ${{ github.repository_owner }}"
      echo "Ref: ${{ github.ref }}"
      echo "Ref name: ${{ github.ref_name }}"
      echo "SHA: ${{ github.sha }}"
      echo "Actor: ${{ github.actor }}"
      echo "Event name: ${{ github.event_name }}"
      echo "Workflow: ${{ github.workflow }}"
      echo "Job: ${{ github.job }}"
      echo "Run ID: ${{ github.run_id }}"
      echo "Run number: ${{ github.run_number }}"
      echo "Run attempt: ${{ github.run_attempt }}"
```

**2. `github.event` context:**
 
```yaml
on: [pull_request, issues]
 
jobs:
  handle-event:
    runs-on: ubuntu-latest
    steps:
      - name: PR event data
        if: github.event_name == 'pull_request'
        run: |
          echo "PR number: ${{ github.event.pull_request.number }}"
          echo "PR title: ${{ github.event.pull_request.title }}"
          echo "PR author: ${{ github.event.pull_request.user.login }}"
          echo "Base branch: ${{ github.event.pull_request.base.ref }}"
          echo "Head branch: ${{ github.event.pull_request.head.ref }}"
 
      - name: Issue event data
        if: github.event_name == 'issues'
        run: |
          echo "Issue number: ${{ github.event.issue.number }}"
          echo "Issue title: ${{ github.event.issue.title }}"
          echo "Issue author: ${{ github.event.issue.user.login }}"
```

**3. `runner` context:**
 
```yaml
steps:
  - name: Runner context
    run: |
      echo "OS: ${{ runner.os }}"
      echo "Architecture: ${{ runner.arch }}"
      echo "Name: ${{ runner.name }}"
      echo "Temp directory: ${{ runner.temp }}"
      echo "Tool cache: ${{ runner.tool_cache }}"
```

**4. `env` context:**
 
```yaml
env:
  GLOBAL_VAR: global
 
jobs:
  example:
    runs-on: ubuntu-latest
    env:
      JOB_VAR: job
    steps:
      - name: Use env context
        env:
          STEP_VAR: step
        run: |
          echo "Global: ${{ env.GLOBAL_VAR }}"
          echo "Job: ${{ env.JOB_VAR }}"
          echo "Step: ${{ env.STEP_VAR }}"
```

**5. `vars` y `secrets` contexts:**
 
```yaml
jobs:
  example:
    runs-on: ubuntu-latest
    steps:
      - name: Use variables and secrets
        run: |
          echo "Variable: ${{ vars.MY_VARIABLE }}"
          echo "Secret present: ${{ secrets.API_KEY != '' }}"
        env:
          API_KEY: ${{ secrets.API_KEY }}
```

**6. `inputs` context (workflow_dispatch / workflow_call):**
 
```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        required: true
        type: string
 
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Use inputs
        run: echo "Deploying to ${{ inputs.environment }}"
```

**7. `matrix` context:**
 
```yaml
strategy:
  matrix:
    os: [ubuntu, windows]
    version: [18, 20]
 
steps:
  - name: Matrix values
    run: |
      echo "OS: ${{ matrix.os }}"
      echo "Version: ${{ matrix.version }}"
```

**8. `needs` context:**
 
```yaml
jobs:
  job1:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.get-version.outputs.version }}
    steps:
      - id: get-version
        run: echo "version=1.2.3" >> $GITHUB_OUTPUT
 
  job2:
    needs: job1
    runs-on: ubuntu-latest
    steps:
      - name: Use needs context
        run: |
          echo "Job1 result: ${{ needs.job1.result }}"
          echo "Job1 output: ${{ needs.job1.outputs.version }}"
```

**9. `strategy` context:**
 
```yaml
strategy:
  fail-fast: false
  max-parallel: 3
 
steps:
  - name: Strategy info
    run: |
      echo "Fail-fast: ${{ strategy.fail-fast }}"
      echo "Max-parallel: ${{ strategy.max-parallel }}"
      echo "Job index: ${{ strategy.job-index }}"
      echo "Job total: ${{ strategy.job-total }}"
```

**10. `job` context:**
 
```yaml
jobs:
  example:
    runs-on: ubuntu-latest
    steps:
      - name: Job context
        run: |
          echo "Status: ${{ job.status }}"
          echo "Container: ${{ job.container.id }}"
    services:
      postgres:
        image: postgres:15
    container:
      image: node:20
```   

**11. `steps` context:**
 
```yaml
steps:
  - name: Step 1
    id: step1
    run: echo "output=hello" >> $GITHUB_OUTPUT
 
  - name: Step 2
    id: step2
    run: echo "data=world" >> $GITHUB_OUTPUT
 
  - name: Use steps context
    run: |
      echo "Step1 outcome: ${{ steps.step1.outcome }}"
      echo "Step1 output: ${{ steps.step1.outputs.output }}"
      echo "Step2 output: ${{ steps.step2.outputs.data }}"
```

**Recursos:**
- [Contexts](https://docs.github.com/en/actions/learn-github-actions/contexts)
- [github context](https://docs.github.com/en/actions/learn-github-actions/contexts#github-context)
- [job context](https://docs.github.com/en/actions/learn-github-actions/contexts#job-context)
- [steps context](https://docs.github.com/en/actions/learn-github-actions/contexts#steps-context)
- [runner context](https://docs.github.com/en/actions/learn-github-actions/contexts#runner-context)
- [matrix context](https://docs.github.com/en/actions/learn-github-actions/contexts#matrix-context)
- [needs context](https://docs.github.com/en/actions/learn-github-actions/contexts#needs-context)

### Expressions con ${{ }}
 
Las **expressions** permiten evaluación dinámica de valores durante el workflow.
 
#### Evaluación estática vs runtime
 
```yaml
jobs:
  example:
    runs-on: ubuntu-latest
 
    # Evaluación ESTÁTICA (durante parse del workflow)
    if: ${{ github.ref == 'refs/heads/main' }}
 
    steps:
      # Evaluación RUNTIME (durante ejecución del step)
      - name: Check condition
        if: ${{ steps.previous.outputs.value == 'success' }}
        run: echo "Condition met"
```

#### Operadores disponibles
 
```yaml
steps:
  - name: Logical operators
    if: |
      github.event_name == 'push' &&
      github.ref == 'refs/heads/main' &&
      !contains(github.event.head_commit.message, '[skip ci]')
    run: echo "All conditions met"
 
  - name: Comparison operators
    if: |
      github.run_number > 100 ||
      github.run_attempt >= 3
    run: echo "Number check"
```

**Funciones de string disponibles:**
 
```yaml
steps:
  - name: String functions
    run: |
      echo "${{ contains(github.ref, 'release') }}"
      echo "${{ startsWith(github.ref, 'refs/heads/feature/') }}"
      echo "${{ endsWith(github.ref, '/production') }}"
      echo "${{ format('Version {0}.{1}', 1, 2) }}"
      echo "${{ join(matrix.os, ', ') }}"
```

#### Prevenir secret leakage
 
```yaml
steps:
  # ❌ NUNCA hagas esto - el secreto aparecerá en logs
  - name: Bad practice
    if: ${{ secrets.API_KEY == 'expected-value' }}
    run: echo "Secret check"
 
  # ✅ Correcto - usa env y comparación en bash
  - name: Good practice
    env:
      API_KEY: ${{ secrets.API_KEY }}
    run: |
      if [ "$API_KEY" = "expected-value" ]; then
        echo "Secret is valid"
      fi
 
  # ❌ NUNCA imprimas secretos
  - name: Bad - secret in output
    run: echo "Key is ${{ secrets.API_KEY }}"
 
  # ✅ Correcto - mask el secreto
  - name: Good - masked output
    run: |
      echo "::add-mask::${{ secrets.API_KEY }}"
      echo "Key is configured"
```

#### Funciones útiles
 
```yaml
steps:
  - name: Useful functions
    run: |
      # toJSON / fromJSON
      echo '${{ toJSON(github) }}'
 
      # contains
      echo "${{ contains(github.event.pull_request.labels.*.name, 'bug') }}"
 
      # startsWith / endsWith
      echo "${{ startsWith(github.ref, 'refs/tags/v') }}"
 
      # format
      echo "${{ format('Deploy to {0} on {1}', 'production', '2024-01-01') }}"
 
      # join
      echo "${{ join(matrix.*, '-') }}"
 
      # hashFiles (para cache keys)
      echo "${{ hashFiles('**/package-lock.json') }}"
```

**Recursos:**
- [Expressions](https://docs.github.com/en/actions/learn-github-actions/expressions)
- [Context and expression syntax](https://docs.github.com/en/actions/learn-github-actions/contexts#context-availability)
- [Encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)

### Immutable Actions y Version Pinning

#### ¿Qué son las immutable actions?

GitHub está implementando **immutable actions** en los runners hosted, lo que significa que las actions se ejecutarán desde una ubicación inmutable y verificada.

**Implicaciones:**
1. Las actions deben pinnearse a versiones específicas
2. No se pueden usar referencias a ramas (como `@main`)
3. Se recomienda usar commit SHA completo

```yaml
steps:
  # ❌ NO recomendado - referencia a rama (mutable)
  - uses: actions/checkout@main
 
  # ⚠️ Aceptable - tag semántico (puede cambiar)
  - uses: actions/checkout@v4
 
  # ✅ MEJOR - tag específico
  - uses: actions/checkout@v4.1.1
 
  # ✅ MÁS SEGURO - commit SHA completo (immutable)
  - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1
```

#### Estrategia de version pinning
 
```yaml
name: Secure Actions Usage
on: [push]
 
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # Actions de GitHub - pin a SHA con comentario de versión
      - name: Checkout
        uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1
 
      - name: Setup Node
        uses: actions/setup-node@60edb5dd545a775178f52524783378180af0d1f  # v4.0.2
        with:
          node-version: '20'
 
      # Actions de terceros - SIEMPRE pin a SHA
      - name: Third-party action
        uses: docker/build-push-action@4a13e500e55cf31b7a5d59a38ab2040ab0f42f56  # v5.1.0
        with:
          context: .
```

#### Dependabot para actualizar actions
 
Crea `.github/dependabot.yml`:
 
```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    # Agrupar actualizaciones
    groups:
      github-actions:
        patterns:
          - "*"
    reviewers:
      - "my-team"
    labels:
      - "dependencies"
      - "github-actions"
```
 
 **Recursos:**
- [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [Keeping your actions up to date with Dependabot](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/keeping-your-actions-up-to-date-with-dependabot)
- [GitHub Changelog](https://github.blog/changelog/)

### Editor Tooling
 
#### GitHub Actions VS Code Extension
 
La extensión oficial para VS Code proporciona:
- IntelliSense para sintaxis de workflows
- Validación de YAML en tiempo real
- Autocompletado de contexts y expressions
- Snippets para acciones comunes
  
**Recursos:**
- [GitHub Actions VS Code Extension](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-github-actions)
- [VS Code GitHub Actions documentation](https://code.visualstudio.com/docs/azure/github-actions)

#### YAML Schema
 
GitHub proporciona un schema JSON para validación de workflows:
 
```yaml
# Agregar al inicio de tu workflow
# yaml-language-server: $schema=https://json.schemastore.org/github-workflow.json
 
name: My Workflow
on: [push]
```
 
**En VS Code (`settings.json`):**
 
```json
{
  "yaml.schemas": {
    "https://json.schemastore.org/github-workflow.json": [
      ".github/workflows/*.yml",
      ".github/workflows/*.yaml"
    ]
  }
}
```

**Recursos:**
- [JSON Schema Store - GitHub Workflow](https://json.schemastore.org/github-workflow.json)

---

## 1.3 Manage workflow execution and outputs
 
### Caching y Artifact Management
 
#### Caching con actions/cache

```yaml
name: Caching Example
on: [push]
 
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
 
      # Cache de dependencias de Node.js
      - name: Cache Node modules
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
 
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'  # Cache integrado
 
      - run: npm ci
      - run: npm run build
 
      # Cache de build artifacts
      - name: Cache build output
        uses: actions/cache@v4
        with:
          path: |
            dist
            .next/cache
          key: ${{ runner.os }}-build-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-build-
```

#### Cache para diferentes lenguajes
 
**Python:**
```yaml
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

**Java/Maven:**
```yaml
- uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
    restore-keys: |
      ${{ runner.os }}-maven-
```

**Ruby:**
```yaml
- uses: actions/cache@v4
  with:
    path: vendor/bundle
    key: ${{ runner.os }}-gems-${{ hashFiles('**/Gemfile.lock') }}
    restore-keys: |
      ${{ runner.os }}-gems-
```

**Recursos:**
- [Caching dependencies](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows)
- [actions/cache](https://github.com/actions/cache)


### Artifacts
 
```yaml
name: Artifacts Example
on: [push]
 
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build
 
      # Upload artifacts
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-files
          path: |
            dist/
            build/
          retention-days: 5  # Retención por 5 días
          if-no-files-found: error  # Fallar si no hay archivos
 
      # Upload logs
      - name: Upload logs
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: error-logs
          path: logs/*.log
          retention-days: 14
 
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      # Download artifacts
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-files
          path: ./dist
 
      - name: Deploy
        run: ./deploy.sh
```

### Retention Policies via REST API
 
```bash
# Listar artifacts
curl -L \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/OWNER/REPO/actions/artifacts
 
# Eliminar artifact específico
curl -L \
  -X DELETE \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/OWNER/REPO/actions/artifacts/ARTIFACT_ID
 
# Listar workflow runs
curl -L \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/OWNER/REPO/actions/runs
 
# Eliminar workflow run logs
curl -L \
  -X DELETE \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/OWNER/REPO/actions/runs/RUN_ID/logs
```
 
**Script para limpiar artifacts antiguos:**
 
```bash
#!/bin/bash
# cleanup-artifacts.sh
 
REPO="owner/repo"
DAYS_OLD=30
 
# Obtener artifacts más antiguos que DAYS_OLD días
CUTOFF_DATE=$(date -d "$DAYS_OLD days ago" --iso-8601)
 
curl -L \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/$REPO/actions/artifacts?per_page=100" \
| jq -r ".artifacts[] | select(.created_at < \"$CUTOFF_DATE\") | .id" \
| while read artifact_id; do
  echo "Deleting artifact $artifact_id"
  curl -L \
    -X DELETE \
    -H "Accept: application/vnd.github+json" \
    -H "Authorization: Bearer $GITHUB_TOKEN" \
    "https://api.github.com/repos/$REPO/actions/artifacts/$artifact_id"
done
```

**Recursos:**
- [Storing workflow data as artifacts](https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts)
- [actions/upload-artifact](https://github.com/actions/upload-artifact)
- [actions/download-artifact](https://github.com/actions/download-artifact)
- [Artifacts REST API](https://docs.github.com/en/rest/actions/artifacts)


### Pasar datos entre jobs y steps
 
#### Entre steps (mismo job)
 
**Usando outputs:**
```yaml
jobs:
  example:
    runs-on: ubuntu-latest
    steps:
      - name: Generate version
        id: version
        run: |
          VERSION="1.2.3"
          echo "number=$VERSION" >> $GITHUB_OUTPUT
          echo "date=$(date +'%Y-%m-%d')" >> $GITHUB_OUTPUT
 
      - name: Use version
        run: |
          echo "Version: ${{ steps.version.outputs.number }}"
          echo "Date: ${{ steps.version.outputs.date }}"
```
 
**Usando environment files:**
```yaml
steps:
  - name: Set environment variable
    run: |
      echo "BUILD_ID=$(uuidgen)" >> $GITHUB_ENV
      echo "DEPLOY_PATH=/opt/app" >> $GITHUB_ENV
 
  - name: Use environment variable
    run: |
      echo "Build ID: $BUILD_ID"
      echo "Deploy path: $DEPLOY_PATH"
```
 
#### Entre jobs
 
**Usando job outputs:**
```yaml
jobs:
  job1:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.get-version.outputs.version }}
      artifact-path: ${{ steps.build.outputs.path }}
    steps:
      - id: get-version
        run: echo "version=1.2.3" >> $GITHUB_OUTPUT
 
      - id: build
        run: |
          echo "path=dist/app.tar.gz" >> $GITHUB_OUTPUT
 
  job2:
    needs: job1
    runs-on: ubuntu-latest
    steps:
      - name: Use outputs from job1
        run: |
          echo "Version from job1: ${{ needs.job1.outputs.version }}"
          echo "Artifact path: ${{ needs.job1.outputs.artifact-path }}"
```
 
**Usando artifacts:**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "data" > data.txt
      - uses: actions/upload-artifact@v4
        with:
          name: my-data
          path: data.txt
 
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: my-data
      - run: cat data.txt
```
 
#### Reusable workflow outputs
 
```yaml
# reusable-workflow.yml
on:
  workflow_call:
    outputs:
      build-version:
        description: "Version that was built"
        value: ${{ jobs.build.outputs.version }}
      artifact-name:
        description: "Name of the artifact"
        value: ${{ jobs.build.outputs.artifact }}
 
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.number }}
      artifact: ${{ steps.upload.outputs.artifact-name }}
    steps:
      - id: version
        run: echo "number=1.2.3" >> $GITHUB_OUTPUT
      - id: upload
        run: echo "artifact-name=build-artifact" >> $GITHUB_OUTPUT
```
 
```yaml
# caller-workflow.yml
jobs:
  call-reusable:
    uses: ./.github/workflows/reusable-workflow.yml
 
  use-outputs:
    needs: call-reusable
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Build version: ${{ needs.call-reusable.outputs.build-version }}"
          echo "Artifact: ${{ needs.call-reusable.outputs.artifact-name }}"
```

**Recursos:**
- [Defining outputs for jobs](https://docs.github.com/en/actions/using-jobs/defining-outputs-for-jobs)
- [Passing information between jobs](https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts#passing-data-between-jobs-in-a-workflow)


### Job Summaries con GITHUB_STEP_SUMMARY
 
Los **job summaries** permiten crear reportes ricos en Markdown que se muestran en la UI de GitHub Actions.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
 
      - name: Run tests
        run: |
          # Ejecutar tests y capturar resultados
          npm test > test-results.txt || true
 
      - name: Generate test summary
        if: always()
        run: |
          echo "## 🧪 Test Results" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
 
          # Tabla de resultados
          echo "| Metric | Value |" >> $GITHUB_STEP_SUMMARY
          echo "|--------|-------|" >> $GITHUB_STEP_SUMMARY
          echo "| ✅ Passed | 45 |" >> $GITHUB_STEP_SUMMARY
          echo "| ❌ Failed | 2 |" >> $GITHUB_STEP_SUMMARY
          echo "| ⏭️ Skipped | 3 |" >> $GITHUB_STEP_SUMMARY
          echo "| ⏱️ Duration | 2m 34s |" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
 
          # Coverage
          echo "### 📊 Coverage" >> $GITHUB_STEP_SUMMARY
          echo "- **Lines**: 87.5%" >> $GITHUB_STEP_SUMMARY
          echo "- **Branches**: 72.3%" >> $GITHUB_STEP_SUMMARY
          echo "- **Functions**: 91.2%" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
 
          # Links
          echo "### 🔗 Links" >> $GITHUB_STEP_SUMMARY
          echo "- [Full Report](https://example.com/report)" >> $GITHUB_STEP_SUMMARY
          echo "- [Coverage Details](https://example.com/coverage)" >> $GITHUB_STEP_SUMMARY
```

**Ejemplo avanzado con imágenes y code blocks:**
 
```yaml
- name: Advanced summary
  run: |
    cat << 'EOF' >> $GITHUB_STEP_SUMMARY
    ## 🚀 Deployment Summary
 
    ### Status: ✅ SUCCESS
 
    | Environment | Status | URL |
    |-------------|--------|-----|
    | Production | ✅ Deployed | [Visit](https://prod.example.com) |
    | Staging | ⏭️ Skipped | - |
 
    ### 📦 Deployed Version
```
    Version: 1.2.3
    Commit: abc123
    Branch: main
```
 
    ### ⚠️ Warnings
    > **Note**: This deployment includes breaking changes.
    > Please review the [migration guide](https://docs.example.com/migrate).
 
    ---
    _Deployed by ${{ github.actor }} at $(date)_
    EOF
```

**Recursos:**
- [Adding a job summary](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#adding-a-job-summary)
- [Job summaries examples (GitHub Blog)](https://github.blog/2022-05-09-supercharging-github-actions-with-job-summaries/)


### Workflow Status Badges
 
Los **badges** muestran el estado del workflow en tu README.
 
**Sintaxis:**
```markdown
![Workflow Status](https://github.com/OWNER/REPO/actions/workflows/WORKFLOW_FILE/badge.svg)
 
<!-- Con rama específica -->
![Workflow Status](https://github.com/OWNER/REPO/actions/workflows/WORKFLOW_FILE/badge.svg?branch=main)
 
<!-- Con evento específico -->
![Workflow Status](https://github.com/OWNER/REPO/actions/workflows/WORKFLOW_FILE/badge.svg?event=push)
```
 
**Ejemplos:**
 
```markdown
# My Project
 
![CI](https://github.com/myuser/myrepo/actions/workflows/ci.yml/badge.svg)
![Deploy](https://github.com/myuser/myrepo/actions/workflows/deploy.yml/badge.svg?branch=main)
![Tests](https://github.com/myuser/myrepo/actions/workflows/test.yml/badge.svg?event=pull_request)
 
<!-- Con link -->
[![CI Status](https://github.com/myuser/myrepo/actions/workflows/ci.yml/badge.svg)](https://github.com/myuser/myrepo/actions/workflows/ci.yml)
```

**Recursos:**
- [Adding a workflow status badge](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/adding-a-workflow-status-badge)

### Environment Protections
 
Los **environment protections** agregan gates de aprobación y reglas antes de desplegar.
 
```yaml
name: Deploy with Protection
on: [push]
 
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging  # Sin protección
    steps:
      - run: echo "Deploy to staging"
 
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://prod.example.com
    steps:
      - run: echo "Deploy to production"
```

**Configurar protections en GitHub UI:**
 
1. Ve a `Settings` → `Environments`
2. Crea/edita environment "production"
3. Configura:
   - **Required reviewers**: Quién debe aprobar
   - **Wait timer**: Esperar X minutos antes de desplegar
   - **Deployment branches**: Qué ramas pueden desplegar
   - **Environment secrets**: Secretos específicos del ambiente
**Ejemplo con múltiples ambientes:**
 
```yaml
name: Multi-Environment Deploy
on:
  push:
    branches: [main, develop]
 
jobs:
  deploy:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment:
          - name: development
            branch: develop
            url: https://dev.example.com
          - name: staging
            branch: main
            url: https://staging.example.com
          - name: production
            branch: main
            url: https://prod.example.com
            needs_approval: true
 
    # Filtrar por rama
    if: |
      (matrix.environment.branch == 'develop' && github.ref == 'refs/heads/develop') ||
      (matrix.environment.branch == 'main' && github.ref == 'refs/heads/main')
 
    environment:
      name: ${{ matrix.environment.name }}
      url: ${{ matrix.environment.url }}
 
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to ${{ matrix.environment.name }}
        run: |
          echo "Deploying to ${{ matrix.environment.url }}"
```

**Recursos:**
- [Using environments for deployment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)
- [Environment protection rules](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment#environment-protection-rules)

---
 
**➡️ Siguiente:** [Domain 2: Consume and Troubleshoot Workflows](./02-domain2-consume-troubleshoot-workflows.md)




















