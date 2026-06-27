# Domain 4: Manage GitHub Actions for the Enterprise (20–25%)
 
> Guía completa para la certificación GitHub Actions GH-200
 
## Tabla de Contenidos
 
- [4.1 Distribute and govern actions and workflows](#41-distribute-and-govern-actions-and-workflows)
- [4.2 Manage runners at scale](#42-manage-runners-at-scale)
- [4.3 Manage encrypted secrets and variables](#43-manage-encrypted-secrets-and-variables)

---
 
## 4.1 Distribute and govern actions and workflows

### Reusable components y templates
 
Organizar reusable workflows en el contexto empresarial:
 
```
.github/
├── workflows/
│   ├── reusable-ci.yml
│   ├── reusable-deploy.yml
│   ├── reusable-security-scan.yml
│   └── caller-workflow.yml
└── workflow-templates/
    ├── python-ci.yml
    ├── python-ci.properties.json
    ├── node-ci.yml
    └── node-ci.properties.json
```

**Reusable workflow empresarial:**
 
```yaml
# .github/workflows/reusable-compliance.yml
name: Compliance Check
on:
  workflow_call:
    inputs:
      severity-threshold:
        required: false
        type: string
        default: 'high'
    secrets:
      security-token:
        required: true
    outputs:
      passed:
        value: ${{ jobs.check.outputs.passed }}
 
jobs:
  check:
    runs-on: ubuntu-latest
    outputs:
      passed: ${{ steps.scan.outputs.passed }}
    steps:
      - uses: actions/checkout@v4
 
      - name: Run compliance scan
        id: scan
        env:
          TOKEN: ${{ secrets.security-token }}
        run: |
          # Scan de compliance
          ./compliance-tool scan --severity=${{ inputs.severity-threshold }}
          echo "passed=true" >> $GITHUB_OUTPUT
 
      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: compliance-report
          path: compliance-report.json
```

**Recursos:**
- [Sharing workflows with your organization](https://docs.github.com/en/actions/using-workflows/sharing-workflows-secrets-and-runners-with-your-organization)

### Control de acceso a actions y workflows
 
#### A nivel de organización
 
**Settings → Actions → General**
 
Opciones:
- **Allow all actions and reusable workflows**
- **Disable actions**
- **Allow select actions and reusable workflows**
  - Allow actions created by GitHub
  - Allow Marketplace actions by verified creators
  - Allow specified actions and reusable workflows
**Ejemplo de allow list:**
 
```
actions/checkout@*,
actions/setup-node@*,
docker/build-push-action@*,
my-org/*
```
 
**Políticas por patrón:**
 
```
# Permitir:
actions/*                    # Todas las actions de GitHub
my-org/*                     # Todas las actions de tu org
other-org/specific-action@*  # Action específica de otra org
*/action-name@v1             # Version específica de cualquier org
```
 
**Recursos:**
- [Managing GitHub Actions settings for a repository](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository)
- [Allowing select actions to run](https://docs.github.com/en/organizations/managing-organization-settings/disabling-or-limiting-github-actions-for-your-organization#allowing-select-actions-and-reusable-workflows-to-run)

### Organizational use policies
 
```yaml
# Requerir workflow templates específicos
# Settings → Actions → Workflow permissions
 
# Opciones:
# 1. Read repository contents and packages permissions
#    - GITHUB_TOKEN solo puede leer
#    - Más seguro, requiere permisos explícitos
 
# 2. Read and write permissions
#    - GITHUB_TOKEN puede leer y escribir
#    - Menos seguro, más conveniente
 
# 3. Allow GitHub Actions to create and approve pull requests
#    - Permite que actions creen/aprueben PRs
```

**Enforced workflows (GitHub Enterprise):**
 
```yaml
# .github/workflows/required-security-scan.yml
# Este workflow será requerido en todos los repos
 
name: Required Security Scan
on:
  pull_request:
  push:
    branches: [main]
 
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Security scan
        run: ./security-tool scan
```

**Recursos:**
- [Enforcing policies for GitHub Actions](https://docs.github.com/en/enterprise-cloud@latest/admin/policies/enforcing-policies-for-your-enterprise/enforcing-policies-for-github-actions-in-your-enterprise)
- [Required workflows](https://docs.github.com/en/actions/using-workflows/required-workflows)

---

## 4.2 Manage runners at scale
 
### GitHub-hosted vs Self-hosted runners
 
#### GitHub-hosted runners
 
| Runner | Image | Specs |
|--------|-------|-------|
| `ubuntu-latest` | Ubuntu 22.04 | 2-core, 7 GB RAM, 14 GB SSD |
| `ubuntu-22.04` | Ubuntu 22.04 | 2-core, 7 GB RAM, 14 GB SSD |
| `windows-latest` | Windows Server 2022 | 2-core, 7 GB RAM, 14 GB SSD |
| `windows-2022` | Windows Server 2022 | 2-core, 7 GB RAM, 14 GB SSD |
| `macos-latest` | macOS 14 Sonoma | 3-core, 14 GB RAM, 14 GB SSD |
| `macos-13` | macOS 13 Ventura | 3-core, 14 GB RAM, 14 GB SSD |
| `macos-12` | macOS 12 Monterey | 3-core, 14 GB RAM, 14 GB SSD |
 
**Ventajas GitHub-hosted:**
- Sin mantenimiento
- Auto-actualizados
- Limpios en cada ejecución
- Multiple OS disponibles
**Desventajas:**
- Tiempo límite de ejecución (6 horas)
- No personalizables
- Pueden ser lentos para ciertas tareas
- Networking limitado

#### Self-hosted runners
 
**Ventajas:**
- Control total del ambiente
- Sin límite de tiempo
- Acceso a recursos internos
- Personalizable (hardware, software)
- Potencialmente más rápido

**Desventajas:**
- Requiere mantenimiento
- Responsabilidad de seguridad
- Costos de infraestructura

**Recursos:**
- [About GitHub-hosted runners](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners)
- [About self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners)

### Configurar self-hosted runners
 
#### Instalación
 
```bash
# 1. Descargar el runner
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz
 
# 2. Configurar
./config.sh --url https://github.com/YOUR-ORG --token YOUR-TOKEN
 
# 3. Instalar como servicio
sudo ./svc.sh install
sudo ./svc.sh start
 
# 4. Verificar status
sudo ./svc.sh status
```

#### Configuración avanzada
 
```bash
# Con labels personalizados
./config.sh \
  --url https://github.com/YOUR-ORG \
  --token YOUR-TOKEN \
  --name my-runner \
  --labels self-hosted,linux,x64,gpu,large
 
# Con runner group
./config.sh \
  --url https://github.com/YOUR-ORG \
  --token YOUR-TOKEN \
  --runnergroup production
 
# Ephemeral runner (se elimina después de un job)
./config.sh \
  --url https://github.com/YOUR-ORG \
  --token YOUR-TOKEN \
  --ephemeral
```

#### Docker runner
 
**Dockerfile:**
 
```dockerfile
FROM ubuntu:22.04
 
RUN apt-get update && apt-get install -y \
    curl \
    git \
    jq \
    sudo \
    && rm -rf /var/lib/apt/lists/*
 
RUN useradd -m runner && \
    echo "runner ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
 
USER runner
WORKDIR /home/runner
 
# Instalar runner
RUN curl -o actions-runner-linux-x64.tar.gz -L \
    https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz && \
    tar xzf ./actions-runner-linux-x64.tar.gz && \
    rm actions-runner-linux-x64.tar.gz
 
COPY entrypoint.sh ./
RUN sudo chmod +x ./entrypoint.sh
 
ENTRYPOINT ["./entrypoint.sh"]
```
 
**entrypoint.sh:**
 
```bash
#!/bin/bash
set -e
 
# Configurar runner
./config.sh \
  --url "${GITHUB_URL}" \
  --token "${GITHUB_TOKEN}" \
  --name "${RUNNER_NAME:-docker-runner}" \
  --work _work \
  --labels "${RUNNER_LABELS:-self-hosted,linux,docker}" \
  --unattended \
  --replace
 
# Ejecutar runner
./run.sh
```

**Ejecutar:**
 
```bash
docker run -d \
  -e GITHUB_URL=https://github.com/YOUR-ORG \
  -e GITHUB_TOKEN=YOUR-TOKEN \
  -e RUNNER_NAME=docker-runner-1 \
  -e RUNNER_LABELS=self-hosted,linux,docker,production \
  --name github-runner \
  github-runner:latest
```
 
#### Kubernetes runner

```yaml
# runner-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: github-runner
spec:
  replicas: 3
  selector:
    matchLabels:
      app: github-runner
  template:
    metadata:
      labels:
        app: github-runner
    spec:
      containers:
      - name: runner
        image: github-runner:latest
        env:
        - name: GITHUB_URL
          value: "https://github.com/YOUR-ORG"
        - name: GITHUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: github-runner-secret
              key: token
        - name: RUNNER_LABELS
          value: "self-hosted,kubernetes,production"
```

**Recursos:**
- [Adding self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/adding-self-hosted-runners)
- [Using self-hosted runners in a workflow](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/using-self-hosted-runners-in-a-workflow)

### Runner groups
 
Los **runner groups** permiten organizar y controlar acceso a runners.
 
**Crear runner group:**
 
1. **Organization Settings** → **Actions** → **Runner groups**
2. Click **New runner group**
3. Configurar:
   - Name: `production-runners`
   - Repository access: Selected repositories
   - Workflow access: Selected workflows
**Asignar runner a group:**
 
```bash
# Al configurar el runner
./config.sh \
  --url https://github.com/YOUR-ORG \
  --token YOUR-TOKEN \
  --runnergroup production-runners
```
 
**Usar runner group en workflow:**
 
```yaml
jobs:
  deploy:
    runs-on: [self-hosted, production-runners]
    steps:
      - run: ./deploy.sh
```
 
**Recursos:**
- [Managing access to self-hosted runners using groups](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/managing-access-to-self-hosted-runners-using-groups)

### IP allow lists y networking
 
#### Configurar IP allow list
 
1. **Organization Settings** → **Actions** → **Runners**
2. **IP allow list**
3. Agregar rangos CIDR:
```
203.0.113.0/24
198.51.100.0/24
192.0.2.0/24
```
 
#### Self-hosted runner detrás de proxy
 
```bash
# Configurar proxy
export http_proxy=http://proxy.example.com:8080
export https_proxy=http://proxy.example.com:8080
export no_proxy=localhost,127.0.0.1
 
# Configurar runner
./config.sh --url https://github.com/YOUR-ORG --token YOUR-TOKEN
 
# O en .env
echo "http_proxy=http://proxy.example.com:8080" >> .env
echo "https_proxy=http://proxy.example.com:8080" >> .env
```

#### Networking en GitHub-hosted runners
 
```yaml
jobs:
  access-internal:
    runs-on: ubuntu-latest
    steps:
      # GitHub-hosted runners pueden acceder a IPs públicas
      # Para acceso a recursos internos, considera:
 
      # 1. VPN
      - name: Connect to VPN
        run: |
          sudo apt-get install -y openvpn
          sudo openvpn --config company.ovpn --daemon
 
      # 2. Tailscale
      - name: Connect Tailscale
        uses: tailscale/github-action@v2
        with:
          oauth-client-id: ${{ secrets.TS_OAUTH_CLIENT_ID }}
          oauth-secret: ${{ secrets.TS_OAUTH_SECRET }}
          tags: tag:ci
 
      # 3. WireGuard
      - name: Setup WireGuard
        run: |
          sudo apt-get install -y wireguard
          echo "${{ secrets.WG_CONFIG }}" | sudo tee /etc/wireguard/wg0.conf
          sudo wg-quick up wg0
```
 
**Recursos:**
- [About networking for self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners#communication-between-self-hosted-runners-and-github)

### Identificar software preinstalado
 
#### Ver software preinstalado
 
Los GitHub-hosted runners vienen con muchas herramientas preinstaladas.
 
```yaml
steps:
  - name: Show installed software
    run: |
      echo "=== Node.js versions ==="
      node --version
      npm --version
 
      echo "=== Python versions ==="
      python3 --version
      pip3 --version
 
      echo "=== Java versions ==="
      java -version
 
      echo "=== Docker ==="
      docker --version
      docker-compose --version
 
      echo "=== Tools ==="
      git --version
      gh --version
      kubectl version --client
      helm version
      terraform --version
```

#### Consultar herramientas instaladas
 
- [Ubuntu Readme](https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2204-Readme.md)
- [Windows Readme](https://github.com/actions/runner-images/blob/main/images/windows/Windows2022-Readme.md)
- [macOS Readme](https://github.com/actions/runner-images/blob/main/images/macos/macos-14-Readme.md)

#### Instalar software adicional
 
```yaml
jobs:
  custom-tools:
    runs-on: ubuntu-latest
    steps:
      # Via apt
      - name: Install via apt
        run: |
          sudo apt-get update
          sudo apt-get install -y postgresql-client redis-tools
 
      # Via setup actions (cached)
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
 
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
 
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'
 
      # Via package managers
      - name: Install via npm
        run: npm install -g typescript@latest
 
      - name: Install via pip
        run: pip install awscli --break-system-packages
 
      # Via binary download
      - name: Install custom tool
        run: |
          curl -L -o tool.tar.gz https://example.com/tool.tar.gz
          tar xzf tool.tar.gz
          sudo mv tool /usr/local/bin/
 
      # Via container image
      - name: Run in container
        uses: docker://node:20-alpine
        with:
          args: node --version
```
 
#### Caching de herramientas

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'  # Cachea ~/.npm
 
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'  # Cachea pip cache
 
- uses: actions/cache@v4
  with:
    path: |
      ~/.cargo
      ~/.rustup
    key: ${{ runner.os }}-rust-${{ hashFiles('**/Cargo.lock') }}
```
 
**Recursos:**
- [Software installed on GitHub-hosted runners](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners#supported-software)
- [Runner Images Repository](https://github.com/actions/runner-images)
- [Tool Cache](https://github.com/actions/runner-images/blob/main/docs/tool-cache.md)

### Troubleshoot runner issues
 
#### Problemas comunes
 
**1. Runner offline:**
 
```bash
# Verificar status
sudo ./svc.sh status
 
# Ver logs
sudo journalctl -u actions.runner.* -f
 
# Reiniciar servicio
sudo ./svc.sh stop
sudo ./svc.sh start
```
 
**2. Runner no toma jobs:**
 
```yaml
# Verificar labels
jobs:
  test:
    # Asegurar que los labels coincidan
    runs-on: [self-hosted, linux, x64]
```
 
**3. Espacio en disco:**
 
```bash
# Limpiar workspace
cd /path/to/runner/_work
rm -rf */
 
# Limpiar Docker (si aplica)
docker system prune -af --volumes
 
# Monitorear espacio
df -h
```

**4. Permisos:**
 
```bash
# Verificar permisos del usuario runner
sudo -u runner ls -la /path/to/runner
 
# Arreglar permisos
sudo chown -R runner:runner /path/to/runner
```
 
**5. Red/Conectividad:**
 
```bash
# Verificar conectividad a GitHub
curl -I https://github.com
 
# Verificar proxy (si aplica)
echo $http_proxy
echo $https_proxy
 
# Verificar DNS
nslookup github.com
```
 
**6. Token expirado:**
 
```bash
# Regenerar token y reconfigurar
./config.sh remove --token OLD_TOKEN
./config.sh --url https://github.com/YOUR-ORG --token NEW_TOKEN
```
 
#### Monitoreo de runners
 
```yaml
# .github/workflows/monitor-runners.yml
name: Monitor Runners
on:
  schedule:
    - cron: '*/15 * * * *'  # Cada 15 minutos
 
jobs:
  check-runners:
    runs-on: ubuntu-latest
    steps:
      - name: Check runner status
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          # Listar runners
          gh api /orgs/YOUR-ORG/actions/runners | jq '.runners[] | {name, status, busy}'
 
          # Alertar si hay runners offline
          OFFLINE=$(gh api /orgs/YOUR-ORG/actions/runners | jq '[.runners[] | select(.status=="offline")] | length')
          if [ "$OFFLINE" -gt 0 ]; then
            echo "::warning::$OFFLINE runners are offline"
          fi
```

**Recursos:**
- [Monitoring and troubleshooting self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/monitoring-and-troubleshooting-self-hosted-runners)

---

## 4.3 Manage encrypted secrets and variables
 
### Encrypted secrets
 
Los **secrets** son valores encriptados disponibles en workflows.
 
**Niveles de secrets:**
 
1. **Repository-level secrets**
2. **Environment-level secrets** (sobrescriben repository-level)
3. **Organization-level secrets** (compartidos entre repos)

#### Crear secrets
 
```bash
# Via GitHub CLI
gh secret set MY_SECRET --body "secret-value"
gh secret set MY_SECRET --env production --body "prod-secret"
 
# Via API
curl -X PUT \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/OWNER/REPO/actions/secrets/MY_SECRET \
  -d '{"encrypted_value":"...", "key_id":"..."}'
 
# Via organization
gh secret set MY_ORG_SECRET --org my-org --body "org-secret"
```
 
#### Usar secrets en workflows

```yaml
jobs:
  example:
    runs-on: ubuntu-latest
    environment: production  # Para secrets de environment
    steps:
      # Usar secret como variable de entorno
      - name: Use secret
        env:
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          echo "API Key is configured"
          curl -H "Authorization: Bearer $API_KEY" https://api.example.com
 
      # Usar secret en with:
      - name: Use secret in with
        uses: some/action@v1
        with:
          token: ${{ secrets.TOKEN }}
 
      # Secreto heredado de organización
      - name: Use org secret
        env:
          ORG_SECRET: ${{ secrets.ORG_WIDE_SECRET }}
        run: echo "Org secret configured"
```
 
**Pasar secretos a reusable workflow:**
 
```yaml
jobs:
  call-reusable:
    uses: ./.github/workflows/reusable.yml
    secrets:
      api-key: ${{ secrets.API_KEY }}
      # O pasar todos los secretos automáticamente
      # secrets: inherit
```

#### Buenas prácticas con secrets
 
```yaml
steps:
  # ✅ Correcto - usar como env var
  - name: Good
    env:
      SECRET: ${{ secrets.MY_SECRET }}
    run: |
      curl -H "Authorization: $SECRET" https://api.example.com
 
  # ❌ Incorrecto - secreto en run directamente
  - name: Bad
    run: |
      curl -H "Authorization: ${{ secrets.MY_SECRET }}" https://api.example.com
 
  # ✅ Masking si necesitas referenciar directamente
  - name: Mask secret
    run: |
      echo "::add-mask::${{ secrets.MY_SECRET }}"
      curl -H "Authorization: ${{ secrets.MY_SECRET }}" https://api.example.com
```

### Variables
 
Las **variables** son valores no encriptados para configuración.
 
```yaml
# Variables definidas en Settings → Secrets and variables → Variables
 
jobs:
  example:
    runs-on: ubuntu-latest
    steps:
      - name: Use variables
        run: |
          echo "Environment: ${{ vars.DEPLOY_ENV }}"
          echo "Region: ${{ vars.AWS_REGION }}"
          echo "Port: ${{ vars.API_PORT }}"
 
      # Variables de organización
      - name: Org variables
        run: |
          echo "Org config: ${{ vars.ORG_CONFIG }}"
```

#### Crear variables via API
 
```bash
# Repository variable
curl -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/OWNER/REPO/actions/variables \
  -d '{"name":"MY_VAR","value":"my-value"}'
 
# Organization variable
curl -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/orgs/ORG/actions/variables \
  -d '{"name":"ORG_VAR","value":"org-value","visibility":"all"}'
 
# Environment variable
curl -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repositories/REPO_ID/environments/ENVIRONMENT/variables \
  -d '{"name":"ENV_VAR","value":"env-value"}'
```

#### Gestión programática de secrets y variables
 
```bash
#!/bin/bash
# manage-secrets.sh
 
REPO="owner/repo"
TOKEN="$GITHUB_TOKEN"
 
# Listar secrets (solo nombres, no valores)
list_secrets() {
  curl -s -H "Authorization: token $TOKEN" \
    "https://api.github.com/repos/$REPO/actions/secrets" \
    | jq '.secrets[].name'
}
 
# Crear/actualizar variable
set_variable() {
  local name=$1
  local value=$2
 
  curl -X POST \
    -H "Authorization: token $TOKEN" \
    "https://api.github.com/repos/$REPO/actions/variables" \
    -d "{\"name\":\"$name\",\"value\":\"$value\"}"
}
 
# Listar variables
list_variables() {
  curl -s -H "Authorization: token $TOKEN" \
    "https://api.github.com/repos/$REPO/actions/variables" \
    | jq '.variables[] | {name, value, created_at}'
}
 
# Eliminar variable
delete_variable() {
  local name=$1
  curl -X DELETE \
    -H "Authorization: token $TOKEN" \
    "https://api.github.com/repos/$REPO/actions/variables/$name"
}
```
 
**Recursos:**
- [Encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [Variables](https://docs.github.com/en/actions/learn-github-actions/variables)
- [Secrets REST API](https://docs.github.com/en/rest/actions/secrets)
- [Variables REST API](https://docs.github.com/en/rest/actions/variables)

### Scopes y jerarquía
 
```
Organization secrets/variables
    ↓
Repository secrets/variables
    ↓
Environment secrets/variables  (máxima prioridad)
```
 
**Uso por scope:**
 
```yaml
# Repositorio con environment "production"
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Show precedence
        run: |
          # Si existe API_KEY en production env: usa ese
          # Si existe API_KEY en repo: usa ese
          # Si existe API_KEY en org: usa ese
          echo "API Key: ${{ secrets.API_KEY }}"
```
 
---
 
**⬅️ Anterior:** [Domain 3: Author and Maintain Actions](./docs/03-dominio.md)
**➡️ Siguiente:** [Domain 5: Secure and Optimize Automation](./docs/05-dominio.md)














