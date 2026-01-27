# Kubernetes Manifests - Padel App

Esta carpeta contiene los manifests de Kubernetes para desplegar la aplicación Padel usando **Kustomize**, **Traefik** y **ArgoCD**.

## 📁 Estructura

```
manifests/
├── namespaces.yaml             # Namespaces para prod y dev
├── base/                       # Recursos base
│   ├── backend/
│   ├── database/
│   └── frontend/
└── overlays/                   # Entornos específicos
    ├── develop/
    │   ├── kustomization.yaml
    │   └── ingressroute.yaml
    └── production/
        ├── kustomization.yaml
        └── ingressroute.yaml
```

## 🚀 Despliegue con ArgoCD

### Prerequisitos

- **Traefik** como Ingress Controller (NO Nginx)
- **ArgoCD** instalado en el cluster

### Instalar Traefik

```bash
# Usando Helm
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik -n traefik --create-namespace

# O usando kubectl
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v2.10/docs/content/reference/dynamic-configuration/kubernetes-crd-definition-v1.yml
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v2.10/docs/content/reference/dynamic-configuration/kubernetes-crd-rbac.yml
```

### Opción 1: Desplegar Producción (Completo)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: padel-app-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/TU_USUARIO/TU_REPO_MANIFESTS.git
    targetRevision: main
    path: manifests/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: padel-app-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Opción 2: Desplegar Desarrollo (Completo)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: padel-app-develop
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/TU_USUARIO/TU_REPO_MANIFESTS.git
    targetRevision: develop
    path: manifests/overlays/develop
  destination:
    server: https://kubernetes.default.svc
    namespace: padel-app-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
```

### Opción 3: Ambos entornos simultáneamente

Gracias a los namespaces separados, puedes tener ambos entornos corriendo al mismo tiempo:

```bash
# Crear ambas aplicaciones
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: padel-app-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/TU_USUARIO/TU_REPO_MANIFESTS.git
    targetRevision: main
    path: manifests/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: padel-app-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: padel-app-develop
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/TU_USUARIO/TU_REPO_MANIFESTS.git
    targetRevision: develop
    path: manifests/overlays/develop
  destination:
    server: https://kubernetes.default.svc
    namespace: padel-app-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

## 🔧 Configuración por Entorno

### Desarrollo (develop)
- **Namespace**: `padel-app-dev`
- **Replicas**: Backend: 1, Frontend: 1
- **Prefix**: `dev-` para todos los recursos
- **IngressRoute**: `/dev/frontend` y `/dev/backend`

### Producción (production)
- **Namespace**: `padel-app-prod`
- **Replicas**: Backend: 2, Frontend: 2
- **IngressRoute**: `/frontend` y `/backend`

## 🌐 Acceso con IngressRoute (Traefik)

Después del despliegue:

**Producción:**
- Frontend: `http://<TRAEFIK_IP>/frontend`
- Backend API: `http://<TRAEFIK_IP>/backend/health`

**Desarrollo:**
- Frontend: `http://<TRAEFIK_IP>/dev/frontend`
- Backend API: `http://<TRAEFIK_IP>/dev/backend/health`

Para obtener la IP de Traefik:
```bash
kubectl get svc -n traefik traefik
```

## 📝 Personalización

### Cambiar imágenes de contenedor

Edita los archivos `kustomization.yaml` en cada overlay:

```yaml
images:
- name: victor2campos/ci-backend
  newName: TU_REGISTRY/backend
  newTag: v1.0.0
```

### Cambiar StorageClass

Edita `manifests/base/database/pvc.yaml`:

```yaml
spec:
  storageClassName: standard  # Descomenta y ajusta según tu proveedor
```

### Cambiar credenciales de Database

**⚠️ IMPORTANTE**: Para producción, usa Sealed Secrets o External Secrets Operator.

Edita `manifests/base/database/secret.yaml`:

```yaml
stringData:
  MYSQL_ROOT_PASSWORD: "tu_password_seguro"
  MYSQL_PASSWORD: "tu_password_seguro"
```

## 🧪 Validación Local

```bash
# Validar sintaxis de Kustomize - Producción
kustomize build manifests/overlays/production

# Validar sintaxis de Kustomize - Desarrollo
kustomize build manifests/overlays/develop

# Aplicar manualmente (sin ArgoCD)
kubectl apply -k manifests/overlays/production
kubectl apply -k manifests/overlays/develop

# Ver recursos generados en cada namespace
kubectl get all -n padel-app-prod
kubectl get all -n padel-app-dev
```

## 🔍 Verificar Despliegue

```bash
# Ver estado de las apps en ArgoCD
argocd app list

# Ver detalles de una app
argocd app get padel-app-production
argocd app get padel-app-develop

# Sincronizar manualmente
argocd app sync padel-app-production
argocd app sync padel-app-develop

# Ver logs del backend
kubectl logs -n padel-app-prod -l app=backend --tail=50 -f
kubectl logs -n padel-app-dev -l app=dev-backend --tail=50 -f

# Ver IngressRoutes
kubectl get ingressroute -n padel-app-prod
kubectl get ingressroute -n padel-app-dev
```

## 📦 Componentes Desplegados

| Componente | Imagen | Puerto | Servicio | Namespace Prod | Namespace Dev |
|------------|--------|--------|----------|----------------|---------------|
| Frontend | victor2campos/ci-frontend | 80 | ClusterIP | frontend | dev-frontend |
| Backend | victor2campos/ci-backend | 5000 | ClusterIP | backend | dev-backend |
| Database | mysql:8.0 | 3306 | ClusterIP | mysql | dev-mysql |

## 🆘 Troubleshooting

### IngressRoute no funciona

```bash
# Verificar que Traefik está corriendo
kubectl get pods -n traefik

# Ver logs de Traefik
kubectl logs -n traefik -l app.kubernetes.io/name=traefik

# Verificar IngressRoutes
kubectl describe ingressroute -n padel-app-prod
kubectl describe ingressroute -n padel-app-dev

# Verificar CRDs de Traefik
kubectl get crd | grep traefik
```

### Servicio no responde

```bash
# Probar desde dentro del cluster
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- sh
# Dentro del pod:
curl http://frontend.padel-app-prod.svc.cluster.local
curl http://backend.padel-app-prod.svc.cluster.local:5000/health
```

### Conflictos entre entornos

```bash
# Verificar que los namespaces son diferentes
kubectl get namespaces | grep padel

# Verificar recursos en cada namespace
kubectl get all -n padel-app-prod
kubectl get all -n padel-app-dev
```

## 🔒 Seguridad

- Las credenciales están en **Secret** con `stringData` (base64 automático)
- Para producción real, considera usar:
  - **Sealed Secrets**: https://github.com/bitnami-labs/sealed-secrets
  - **External Secrets Operator**: https://external-secrets.io/
  - **Vault**: https://www.vaultproject.io/

## 📚 Referencias

- [Kustomize Documentation](https://kustomize.io/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Traefik IngressRoute](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/)
- [Traefik Middlewares](https://doc.traefik.io/traefik/middlewares/overview/)
