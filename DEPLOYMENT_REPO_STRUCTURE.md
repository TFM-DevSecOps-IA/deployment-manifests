# Deployment Manifests Repository Structure

Este documento explica cómo debe estar estructurado el **repositorio separado de deployment manifests** (`TFM-DevSecOps-IA/deployment-manifests`) que ArgoCD monitoreará.

## ⚠️ Repositorios Separados

1. **Repositorio de código fuente** (este): Contiene código, Dockerfiles, tests y manifests base
2. **Repositorio de deployment** (`TFM-DevSecOps-IA/deployment-manifests`): Contiene solo manifests con tags específicos por ambiente

## 📁 Estructura del Repo de Deployment

```
TFM-DevSecOps-IA/deployment-manifests/
├── backend-overlays/
│   ├── dev/
│   │   └── kustomization.yml
│   ├── rc/
│   │   └── kustomization.yml
│   └── prod/
│       └── kustomization.yml
└── frontend-overlays/
    ├── dev/
    │   └── kustomization.yml
    ├── rc/
    │   └── kustomization.yml
    └── prod/
        └── kustomization.yml
```

## 📝 Contenido de Kustomization Files

### backend-overlays/dev/kustomization.yml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Referencia a los manifests base en el repo de código
resources:
- https://github.com/TU-ORG/CI/manifests/base/backend?ref=main

namespace: padel-app-dev

images:
- name: victor2campos/ci-backend
  newName: victor2campos/ci-backend
  newTag: v1.0.0-dev  # <-- Este tag se actualiza automáticamente por el pipeline

commonLabels:
  environment: dev
```

### backend-overlays/prod/kustomization.yml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- https://github.com/TU-ORG/CI/manifests/base/backend?ref=main

namespace: padel-app

images:
- name: victor2campos/ci-backend
  newName: victor2campos/ci-backend
  newTag: v1.0.0  # <-- Este tag se actualiza automáticamente por el pipeline

commonLabels:
  environment: production
```

### frontend-overlays/dev/kustomization.yml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- https://github.com/TU-ORG/CI/manifests/base/frontend?ref=main

namespace: padel-app-dev

images:
- name: victor2campos/ci-frontend
  newName: victor2campos/ci-frontend
  newTag: v1.0.0-dev  # <-- Este tag se actualiza automáticamente por el pipeline

commonLabels:
  environment: dev
```

### frontend-overlays/prod/kustomization.yml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- https://github.com/TU-ORG/CI/manifests/base/frontend?ref=main

namespace: padel-app

images:
- name: victor2campos/ci-frontend
  newName: victor2campos/ci-frontend
  newTag: v1.0.0  # <-- Este tag se actualiza automáticamente por el pipeline

commonLabels:
  environment: production
```

## 🔄 Flujo de Actualización Automática

1. **Tag creado**: `git tag v1.2.0-dev && git push origin v1.2.0-dev`
2. **Pipeline ejecuta**: Tests → Security → Build & Push images
3. **Update manifests job**:
   - Detecta que el tag contiene `-dev` → overlay = `overlays/dev`
   - Hace checkout del repo `TFM-DevSecOps-IA/deployment-manifests`
   - Actualiza `backend-overlays/dev/kustomization.yml`:
     ```bash
     sed -i "s|newTag: .*|newTag: v1.2.0-dev|" kustomization.yml
     ```
   - Actualiza `frontend-overlays/dev/kustomization.yml` igualmente
   - Commit y push a `main`
4. **ArgoCD detecta cambio** → Despliega automáticamente con el nuevo tag

## 🎯 Lógica de Overlay Selección

El pipeline determina el overlay basándose en el tag:

```bash
if tag contains "dev" or starts with "develop":
    → overlays/dev
elif tag contains "rc":
    → overlays/rc
else:
    → overlays/prod
```

Ejemplos:
- `v1.0.0-dev` → `backend-overlays/dev/`
- `v1.0.0-rc1` → `backend-overlays/rc/`
- `v1.0.0` → `backend-overlays/prod/`

## 🚀 Setup Inicial del Repo de Deployment

### 1. Crear el repositorio

```bash
# En GitHub, crea el repo: TFM-DevSecOps-IA/deployment-manifests
git clone https://github.com/TFM-DevSecOps-IA/deployment-manifests.git
cd deployment-manifests
```

### 2. Crear estructura

```bash
mkdir -p backend-overlays/{dev,rc,prod}
mkdir -p frontend-overlays/{dev,rc,prod}
```

### 3. Crear kustomization files

Copia los ejemplos anteriores en cada directorio.

### 4. Commit inicial

```bash
git add .
git commit -m "Initial deployment manifests structure"
git push origin main
```

## 📌 ArgoCD Application

Crea una Application en ArgoCD apuntando al repo de deployment:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: padel-backend-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/TFM-DevSecOps-IA/deployment-manifests.git
    targetRevision: main
    path: backend-overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: padel-app-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

Repite para frontend-dev, backend-prod, frontend-prod, etc.

## ✅ Ventajas de Esta Arquitectura

1. **GitOps puro**: Separación clara entre código y deployment config
2. **Rollback fácil**: `git revert` en el repo de manifests
3. **Seguridad**: El repo de código no tiene credenciales de k8s
4. **Audit trail**: Historial claro de deployments
5. **Multi-ambiente**: Overlays independientes por ambiente

## 🔑 Secrets Necesarios

Para que `update-manifests.yml` funcione:

- `DEPLOYMENT_REPO_TOKEN`: PAT con permisos `repo` sobre `TFM-DevSecOps-IA/deployment-manifests`

---

**Última actualización**: 17 enero 2026
