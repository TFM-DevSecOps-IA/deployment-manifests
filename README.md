# Kubernetes Manifests - Kustomize

Este directorio contiene los manifests de Kubernetes con Kustomize para desplegar la aplicación Padel en un cluster k3s gestionado por ArgoCD.

**⚠️ IMPORTANTE**: Estos manifests están en el repositorio de código fuente. El pipeline CI/CD actualiza automáticamente un **repositorio separado de deployment manifests** (`TFM-DevSecOps-IA/deployment-manifests`) con los tags de imagen correctos.

## Estructura

```
k8s/
├── namespace.yaml                    # Namespace padel-app
├── base/                             # Manifests base
│   ├── database/                     # MySQL Database
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── pvc.yaml                  # PersistentVolumeClaim (Azure Disk)
│   │   ├── configmap.yaml            # Init SQL scripts
│   │   └── kustomization.yaml
│   ├── backend/                      # Flask Backend
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── frontend/                     # Nginx Frontend
│       ├── deployment.yaml
│       ├── service.yaml
│       └── kustomization.yaml
└── overlays/                         # Overlays por entorno
    └── production/                   # Producción
        ├── ingress.yaml              # Ingress con path-based routing
        └── kustomization.yaml
```

## Características

### Database (MySQL)
- **PersistentVolumeClaim**: 10Gi con `storageClassName: azure-disk`
- **ConfigMap**: Scripts de inicialización SQL
- **Healthchecks**: Liveness y readiness probes
- **Recursos**: 256Mi-512Mi RAM, 250m-500m CPU

### Backend (Flask)
- **Replicas**: 2 para alta disponibilidad
- **Imagen**: `victor2campos/ci-backend:latest` (actualizada por CI/CD)
- **Variables de entorno**: Configuración de conexión a DB
- **Healthchecks**: HTTP probes en `/health`
- **Recursos**: 128Mi-256Mi RAM, 100m-500m CPU

### Frontend (Nginx)
- **Replicas**: 2 para alta disponibilidad
- **Imagen**: `victor2campos/ci-frontend:latest` (actualizada por CI/CD)
- **Healthchecks**: HTTP probes en `/`
- **Recursos**: 64Mi-128Mi RAM, 50m-200m CPU

### Ingress (Path-based routing)

**Develop**:
- `http://<IP>/dev/frontend` → Frontend
- `http://<IP>/dev/backend` → Backend API
- Namespace: `padel-app-dev`
- Réplicas reducidas: 1 backend, 1 frontend

**Production**:
- `http://<IP>/frontend` → Frontend
- `http://<IP>/backend` → Backend API
- Namespace: `padel-app`
- Réplicas: 2 backend, 2 frontend

**Annotations**: Rewrite-ta
**Develop**:
```bash
# Preview (dry-run)
kubectl kustomize manifests/overlays/develop

# Aplicar
kubectl apply -k manifests/overlays/develop
```

**Production**:**IngressClass**: nginx (compatible con k3s)

## Despliegue

### Crear namespace
```bash
kubectl apply -f k8s/namespace.yaml
```

### Desplegar con Kustomize

**Develop**:
```bash
kubectl get all -n padel-app-dev
kubectl get pvc -n padel-app-dev
kubectl get ingress -n padel-app-dev
```

**Production**:
```bash
kubectl get all -n padel-app
kubectl get pvc -n padel-app
### Verificar despliegue
```bash
# Ver todos los recursos
kubectl get all -n padel-app

# Ver PVCs
kubectl get pvc -n padel-app

# Ver ingress
kubectl get ingress -n padel-app
```

## ArgoCD

### Crear Application
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: padel-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <your-git-repo>
    targetRevision: main
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: padel-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

## Azure Storage Class

Asegúrate de que tu cluster k3s tenga el Azure Disk Storage Class configurado:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-disk
provisioner: disk.csi.azure.com
parameters:
  skuName: Standard_LRS
  kind: Managed
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
**Develop**:
- Frontend: `http://<IP>/dev/frontend`
- Backend API: `http://<IP>/dev/backend`

**Production**

## Acceso

Una vez desplegado, accede a:
- Frontend: `http://<IP>/frontend`
- Backend API: `http://<IP>/backend`

Donde `<IP>` es la IP externa del servicio de ingress de k3s.

## Notas

- Las imágenes Docker deben estar disponibles en un registry accesible desde el cluster
- Actualiza `image:` en los deployments con la ruta completa al registry
- Para producción, considera usar Secrets para las credenciales de la base de datos
- El Ingress usa nginx annotations para path rewriting
