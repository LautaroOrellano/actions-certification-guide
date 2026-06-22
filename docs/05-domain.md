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

