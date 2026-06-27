# Domain 3: Author and Maintain Actions (15–20%)
 
> Guía completa para la certificación GitHub Actions GH-200
 
## Tabla de Contenidos
 
- [3.1 Create and troubleshoot custom actions](#31-create-and-troubleshoot-custom-actions)
- [3.2 Define action structure and metadata](#32-define-action-structure-and-metadata)
- [3.3 Distribute and maintain actions](#33-distribute-and-maintain-actions)

---

## 3.1 Create and troubleshoot custom actions
 
### Tipos de Actions
 
GitHub Actions soporta tres tipos de actions personalizadas:
 
**1. JavaScript Actions**
- Ejecutan código JavaScript/TypeScript
- Rápidas, se ejecutan directamente en el runner
- Multiplataforma (Linux, Windows, macOS)
- Acceso a GitHub Actions toolkit
**2. Docker Actions**
- Ejecutan en un container Docker
- Solo Linux runners
- Ambiente consistente y aislado
- Soportan cualquier lenguaje
**3. Composite Actions**
- Combinan múltiples steps
- Reutilizan workflow steps
- No requieren código personalizado

### JavaScript Action
 
**Estructura:**
 
```
my-js-action/
├── action.yml
├── index.js
├── package.json
└── node_modules/
```
 
**action.yml:**
 
```yaml
name: 'My JavaScript Action'
description: 'A JavaScript action example'
author: 'Your Name'
 
inputs:
  message:
    description: 'Message to print'
    required: true
    default: 'Hello World'
 
  log-level:
    description: 'Log level'
    required: false
    default: 'info'
 
outputs:
  result:
    description: 'Result of the action'
 
runs:
  using: 'node20'
  main: 'index.js'
```
 
**package.json:**
 
```json
{
  "name": "my-js-action",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "@actions/core": "^1.10.1",
    "@actions/github": "^6.0.0"
  }
}
```
 
**index.js:**
 
```javascript
const core = require('@actions/core');
const github = require('@actions/github');
 
async function run() {
  try {
    // Obtener inputs
    const message = core.getInput('message', { required: true });
    const logLevel = core.getInput('log-level');
 
    // Log message
    core.info(`Message: ${message}`);
    core.info(`Log level: ${logLevel}`);
 
    // Obtener contexto de GitHub
    const context = github.context;
    core.info(`Repository: ${context.repo.owner}/${context.repo.repo}`);
    core.info(`Event: ${context.eventName}`);
 
    // Set output
    core.setOutput('result', `Processed: ${message}`);
 
    // Create summary
    await core.summary
      .addHeading('Action Results')
      .addRaw(`Message: ${message}`)
      .write();
 
  } catch (error) {
    core.setFailed(error.message);
  }
}
 
run();
```
 
**Uso:**
 
```yaml
steps:
  - uses: your-org/my-js-action@v1
    with:
      message: 'Hello from workflow'
      log-level: 'debug'
```

### Docker Action
 
**Estructura:**
 
```
my-docker-action/
├── action.yml
├── Dockerfile
└── entrypoint.sh
```
 
**action.yml:**
 
```yaml
name: 'My Docker Action'
description: 'A Docker action example'
author: 'Your Name'
 
inputs:
  name:
    description: 'Name to greet'
    required: true
 
outputs:
  greeting:
    description: 'The greeting message'
 
runs:
  using: 'docker'
  image: 'Dockerfile'
  args:
    - ${{ inputs.name }}
```

**Dockerfile:**
 
```dockerfile
FROM python:3.12-slim
 
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
 
ENTRYPOINT ["/entrypoint.sh"]
```

**entrypoint.sh:**
 
```bash
#!/bin/bash
set -e
 
NAME=$1
GREETING="Hello, $NAME!"
 
echo "$GREETING"
 
# Set output
echo "greeting=$GREETING" >> $GITHUB_OUTPUT
```

### Composite Action
 
**action.yml:**
 
```yaml
name: 'Setup Node.js Project'
description: 'Setup Node.js with caching'
author: 'Your Name'
 
inputs:
  node-version:
    description: 'Node.js version'
    required: true
    default: '20'
 
  install-deps:
    description: 'Install dependencies'
    required: false
    default: 'true'
 
outputs:
  cache-hit:
    description: 'Whether cache was hit'
    value: ${{ steps.cache.outputs.cache-hit }}
 
runs:
  using: 'composite'
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
 
    - name: Cache dependencies
      id: cache
      uses: actions/cache@v4
      with:
        path: ~/.npm
        key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
 
    - name: Install dependencies
      if: inputs.install-deps == 'true'
      run: npm ci
      shell: bash
 
    - name: Show info
      run: |
        echo "Node version: $(node --version)"
        echo "npm version: $(npm --version)"
      shell: bash
```
 
**Uso:**
 
```yaml
steps:
  - uses: actions/checkout@v4
  - uses: your-org/setup-node-project@v1
    with:
      node-version: '20'
      install-deps: 'true'
```
 
**Recursos:**
- [Creating actions](https://docs.github.com/en/actions/creating-actions)
- [Creating a JavaScript action](https://docs.github.com/en/actions/creating-actions/creating-a-javascript-action)
- [Creating a Docker container action](https://docs.github.com/en/actions/creating-actions/creating-a-docker-container-action)
- [Creating a composite action](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action)
- [Actions Toolkit](https://github.com/actions/toolkit)

### Immutable actions en hosted runners
 
A partir de 2024, GitHub está implementando **immutable actions** en runners hosted.
 
**Implicaciones:**
 
1. **Registry sources**: Actions se cargan desde registries inmutables
2. **Version pinning**: Necesitas pinnear a commits SHA específicos
3. **No branch references**: No puedes usar `@main` o `@develop`
```yaml
steps:
  # ❌ NO permitido - referencia a rama
  - uses: actions/checkout@main
 
  # ⚠️ Permitido pero no recomendado - tag puede cambiar
  - uses: actions/checkout@v4
 
  # ✅ RECOMENDADO - SHA completo (immutable)
  - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1
```
 
 **Recursos:**
- [Security hardening - Using third-party actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#using-third-party-actions)


### Troubleshooting actions
 
#### Habilitar debug logging
 
En el repositorio, crear secrets:
- `ACTIONS_STEP_DEBUG = true`
- `ACTIONS_RUNNER_DEBUG = true`
```yaml
steps:
  - name: Debug action
    uses: your-org/my-action@v1
    with:
      debug: true
    env:
      ACTIONS_STEP_DEBUG: true
```
 
#### Testear action localmente con act
 
```bash
# Instalar act (https://github.com/nektos/act)
brew install act  # macOS
# o
curl https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash
 
# Ejecutar workflow localmente
act push
 
# Ejecutar job específico
act -j test
 
# Con secrets
act --secret-file .secrets
```
 
#### Debugging en JavaScript action
 
```javascript
const core = require('@actions/core');
 
async function run() {
  try {
    // Debug logging
    core.debug('Starting action execution');
    core.debug(`Input value: ${core.getInput('input-name')}`);
 
    // Warning
    core.warning('This is a warning message');
 
    // Error but continue
    core.error('This is an error but workflow continues');
 
    // Info
    core.info('Regular info message');
 
    // Group logs
    core.startGroup('Processing data');
    // ... processing steps
    core.endGroup();
 
  } catch (error) {
    core.debug(`Error details: ${JSON.stringify(error)}`);
    core.setFailed(`Action failed: ${error.message}`);
  }
}
```
 
**Recursos:**
- [Enabling debug logging](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/enabling-debug-logging)
- [nektos/act - Run workflows locally](https://github.com/nektos/act)
---
 
## 3.2 Define action structure and metadata

### Action metadata (action.yml)
 
El archivo `action.yml` o `action.yaml` define los metadatos de tu action.
 
**Estructura completa:**
 
```yaml
name: 'Complete Action Example'
description: 'A comprehensive action example'
author: 'Your Name <email@example.com>'
 
# Branding (aparece en Marketplace)
branding:
  icon: 'package'
  color: 'blue'
 
# Inputs
inputs:
  message:
    description: 'Message to process'
    required: true
    default: 'Hello World'
 
  log-level:
    description: 'Logging level'
    required: false
    default: 'info'
 
  api-token:
    description: 'API authentication token'
    required: true
    deprecationMessage: 'Use api-key instead'
 
# Outputs
outputs:
  result:
    description: 'Processing result'
 
  status-code:
    description: 'HTTP status code'
 
  timestamp:
    description: 'Execution timestamp'
 
# Runs configuration
runs:
  using: 'node20'  # o 'node16', 'docker', 'composite'
  main: 'dist/index.js'
  pre: 'dist/setup.js'       # Opcional: ejecuta antes de main
  pre-if: 'runner.os == "Linux"'  # Condicional para pre
  post: 'dist/cleanup.js'    # Opcional: ejecuta después de main
  post-if: 'always()'        # Condicional para post
```
 
**Para Composite Actions:**
 
```yaml
runs:
  using: 'composite'
  steps:
    - name: Setup environment
      run: echo "Setting up..."
      shell: bash
 
    - name: Main task
      run: |
        echo "Input: ${{ inputs.message }}"
        echo "result=Success" >> $GITHUB_OUTPUT
      shell: bash
      env:
        MESSAGE: ${{ inputs.message }}
```
 
**Para Docker Actions:**
 
```yaml
runs:
  using: 'docker'
  image: 'Dockerfile'  # o 'docker://image:tag'
  pre-entrypoint: 'setup.sh'
  entrypoint: 'main.sh'
  post-entrypoint: 'cleanup.sh'
  args:
    - ${{ inputs.message }}
    - '--log-level'
    - ${{ inputs.log-level }}
  env:
    API_TOKEN: ${{ inputs.api-token }}
```
 
**Recursos:**
- [Metadata syntax for GitHub Actions](https://docs.github.com/en/actions/creating-actions/metadata-syntax-for-github-actions)

### Workflow commands dentro de actions
 
#### JavaScript Action con workflow commands
 
```javascript
const core = require('@actions/core');
 
async function run() {
  // Setting outputs
  core.setOutput('result', 'success');
  core.setOutput('timestamp', new Date().toISOString());
 
  // Setting environment variables
  core.exportVariable('MY_VAR', 'value');
 
  // Adding to PATH
  core.addPath('/custom/path');
 
  // Saving state (para post action)
  core.saveState('processedFiles', JSON.stringify(['file1', 'file2']));
 
  // Getting state (en post action)
  const files = JSON.parse(core.getState('processedFiles'));
 
  // Logging
  core.debug('Debug message');
  core.info('Info message');
  core.notice('Notice message');
  core.warning('Warning message');
  core.error('Error message');
 
  // Group logs
  core.startGroup('Installation');
  core.info('Installing dependencies...');
  core.endGroup();
 
  // Annotations
  core.notice('Deployment successful', {
    title: 'Deployment',
    file: 'deploy.sh',
    startLine: 10,
    endLine: 15
  });
 
  core.warning('High memory usage detected', {
    title: 'Performance',
    file: 'server.js',
    startLine: 42
  });
 
  // Job summaries
  await core.summary
    .addHeading('Test Results')
    .addTable([
      [{data: 'Test', header: true}, {data: 'Result', header: true}],
      ['Unit tests', '✅ Passed'],
      ['Integration tests', '✅ Passed']
    ])
    .addLink('View full report', 'https://example.com/report')
    .write();
 
  // Mask sensitive data
  core.setSecret('my-secret-value');
 
  // Fail the action
  if (error) {
    core.setFailed(`Action failed: ${error.message}`);
  }
}
```
 
#### Bash/Shell con workflow commands
 
```bash
#!/bin/bash
set -e
 
# Outputs
echo "result=success" >> $GITHUB_OUTPUT
echo "timestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> $GITHUB_OUTPUT
 
# Environment variables
echo "MY_VAR=value" >> $GITHUB_ENV
 
# PATH
echo "/custom/path" >> $GITHUB_PATH
 
# Job summary
cat << EOF >> $GITHUB_STEP_SUMMARY
## Deployment Complete 🚀
 
- Environment: Production
- Version: 1.2.3
- Time: $(date)
EOF
 
# Logging
echo "::debug::Debug message"
echo "::notice::Notice message"
echo "::warning::Warning message"
echo "::error::Error message"
 
# Grouping
echo "::group::Building application"
npm run build
echo "::endgroup::"
 
# Masking
SECRET="my-secret"
echo "::add-mask::$SECRET"
 
# Stopping commands (para output que contiene ::)
token=$(uuidgen)
echo "::stop-commands::$token"
echo "This may contain :: but won't be processed"
echo "::$token::"
```
 
**Recursos:**
- [Workflow commands for GitHub Actions](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions)
- [Actions Toolkit - Core](https://github.com/actions/toolkit/tree/main/packages/core)
---
 
## 3.3 Distribute and maintain actions
 
### Distribution models

**1. Public Action (GitHub Marketplace)**
- Visible para todos
- Aparece en GitHub Marketplace
- Puede ser usado por cualquier repositorio público/privado
**2. Private Action (mismo repositorio)**
- Solo disponible en el mismo repositorio
- No requiere publicación
**3. Internal Action (organización)**
- Disponible para todos los repos de la organización
- No visible públicamente
**4. Private Action (repositorio privado)**
- Requiere PAT con acceso al repo
- No en marketplace
```yaml
steps:
  # Public action
  - uses: actions/checkout@v4
 
  # Action en el mismo repositorio
  - uses: ./.github/actions/my-action
 
  # Action en otro repositorio (público)
  - uses: other-user/action-repo@v1
 
  # Action en repo privado (requiere PAT)
  - uses: my-org/private-action@v1
    with:
      token: ${{ secrets.REPO_ACCESS_TOKEN }}
```
 
**Recursos:**
- [About custom actions](https://docs.github.com/en/actions/creating-actions/about-custom-actions)
- [Sharing actions and workflows from your private repository](https://docs.github.com/en/actions/creating-actions/sharing-actions-and-workflows-from-your-private-repository)
### Publicar en GitHub Marketplace
 
**Requisitos:**
 
1. Repositorio público
2. Archivo `action.yml` en la raíz
3. README.md
4. LICENSE
5. Releases con tags semánticos
**Pasos:**
 
1. **Preparar el repositorio:**
```
my-action/
├── action.yml
├── README.md
├── LICENSE
├── index.js  # o Dockerfile, etc.
└── package.json
```
 
2. **action.yml con branding:**
```yaml
name: 'My Awesome Action'
description: 'Does something awesome'
author: 'Your Name'
 
branding:
  icon: 'star'  # Feather icon name
  color: 'blue' # blue, green, yellow, orange, red, purple, gray-dark
 
inputs:
  # ...
 
runs:
  using: 'node20'
  main: 'index.js'
```
 
3. **Crear release:**
```bash
# Tag y push
git tag -a v1.0.0 -m "First release"
git push origin v1.0.0
 
# Major version tag (para usuarios que quieran latest v1)
git tag -fa v1 -m "Update v1 tag"
git push origin v1 --force
```
 
4. **Publicar en Marketplace:**
- Ve a tu repositorio en GitHub
- Click en "Releases" → "Draft a new release"
- Selecciona el tag `v1.0.0`
- Marca "Publish this action to the GitHub Marketplace"
- Selecciona categoría principal y secundaria
- Click "Publish release"
**README.md ejemplo:**
 
```markdown
# My Awesome Action
 
Brief description of what your action does.
 
## Usage
 
\`\`\`yaml
name: Example Workflow
on: [push]
 
jobs:
  example:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: your-username/my-awesome-action@v1
        with:
          input-name: 'value'
\`\`\`
 
## Inputs
 
| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `input-name` | Description | Yes | `default-value` |
 
## Outputs
 
| Output | Description |
|--------|-------------|
| `output-name` | Description |
 
## License
 
MIT
```
 
**Recursos:**
- [Publishing actions in GitHub Marketplace](https://docs.github.com/en/actions/creating-actions/publishing-actions-in-github-marketplace)
- [Marketplace badge verification](https://docs.github.com/en/actions/creating-actions/publishing-actions-in-github-marketplace#about-marketplace-badges)

### Versioning y release strategies
 
#### Estrategias de versionado
 
**1. Semantic Versioning (Recomendado)**
 
```bash
# Major version: cambios breaking
v1.0.0 → v2.0.0
 
# Minor version: nuevas features compatibles
v1.0.0 → v1.1.0
 
# Patch version: bug fixes
v1.0.0 → v1.0.1
```
 
**2. Major version tags flotantes**
 
```bash
# Crear release v1.2.3
git tag v1.2.3
git push origin v1.2.3
 
# Mover tag v1 al commit más reciente
git tag -fa v1 -m "Update v1 to v1.2.3"
git push origin v1 --force
 
# Usuarios pueden usar:
# - uses: user/action@v1      # Latest v1.x.x
# - uses: user/action@v1.2.3  # Specific version
# - uses: user/action@SHA     # Pinned to commit
```
 
**3. Pre-release versions**
 
```bash
# Alpha
git tag v2.0.0-alpha.1
 
# Beta
git tag v2.0.0-beta.1
 
# Release candidate
git tag v2.0.0-rc.1
 
# Final
git tag v2.0.0
```
 
#### Workflow para releases automáticos
 
```yaml
name: Release
on:
  push:
    tags:
      - 'v*'
 
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
 
      - name: Build
        run: |
          npm install
          npm run build
          npm run package
 
      - name: Create Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
          draft: false
          prerelease: false
 
      - name: Update major version tag
        run: |
          # Extract major version (v1.2.3 -> v1)
          TAG=${GITHUB_REF#refs/tags/}
          MAJOR=$(echo $TAG | cut -d. -f1)
 
          # Update major tag
          git config user.name github-actions
          git config user.email github-actions@github.com
          git tag -fa $MAJOR -m "Update $MAJOR to $TAG"
          git push origin $MAJOR --force
```
 
**CHANGELOG.md:**
 
```markdown
# Changelog
 
## [2.0.0] - 2024-03-15
 
### Breaking Changes
- Renamed `old-input` to `new-input`
- Removed deprecated `legacy-mode`
 
### Added
- New `advanced-mode` input
- Support for Windows runners
 
### Fixed
- Memory leak in long-running processes
 
## [1.2.0] - 2024-02-01
 
### Added
- Caching support
- Progress indicators
 
### Fixed
- Timeout issues
```
 
**Recursos:**
- [Releasing and maintaining actions](https://docs.github.com/en/actions/creating-actions/releasing-and-maintaining-actions)
- [Semantic Versioning](https://semver.org/)
- [Keep a Changelog](https://keepachangelog.com/)
---
 
**⬅️ Anterior:** [Domain 2: Consume and Troubleshoot Workflows](./02-dominio.md)
**➡️ Siguiente:** [Domain 4: Manage GitHub Actions for the Enterprise](./04-dominio.md)
