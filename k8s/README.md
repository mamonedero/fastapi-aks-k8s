# Despliegue de Servicios FastAPI en AKS

Este directorio contiene los manifiestos de Kubernetes para desplegar dos servicios FastAPI en Azure Kubernetes Service (AKS).

## Arquitectura

- **Servicio Valoraciones**: Expuesto en `https://api.midominio.com/valoraciones`
- **Servicio Consultas**: Expuesto en `https://api.midominio.com/consultas`

## Componentes

### 1. Namespace
- `namespace.yaml`: Crea el namespace `fastapi-services`

### 2. Deployments y Services
- `valoraciones-deployment.yaml`: Deployment y Service para el servicio de valoraciones
- `consultas-deployment.yaml`: Deployment y Service para el servicio de consultas

Cada deployment incluye:
- 2 réplicas para alta disponibilidad
- Health checks (liveness y readiness probes)
- Límites de recursos (CPU y memoria)

### 3. Ingress
- `ingress.yaml`: Configuración del NGINX Ingress Controller con:
  - Enrutamiento basado en path
  - Soporte TLS/SSL con cert-manager
  - Rewrite de URLs

## Prerequisitos

### 1. Cluster AKS
```powershell
# Crear un cluster AKS (si no existe)
az aks create `
  --resource-group miGrupoRecursos `
  --name miClusterAKS `
  --node-count 2 `
  --enable-managed-identity `
  --generate-ssh-keys

# Obtener credenciales
az aks get-credentials --resource-group miGrupoRecursos --name miClusterAKS
```

### 2. Azure Container Registry (ACR)
```powershell
# Crear ACR
az acr create --resource-group miGrupoRecursos --name miRegistro --sku Basic

# Conectar AKS con ACR
az aks update -n miClusterAKS -g miGrupoRecursos --attach-acr miRegistro
```

### 3. Instalar NGINX Ingress Controller
```powershell
# Agregar el repositorio de Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Instalar NGINX Ingress Controller
helm install nginx-ingress ingress-nginx/ingress-nginx `
  --namespace ingress-nginx `
  --create-namespace `
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

### 4. Instalar cert-manager (para SSL/TLS)
```powershell
# Instalar cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Esperar a que cert-manager esté listo
kubectl wait --for=condition=Available --timeout=300s deployment/cert-manager -n cert-manager
```

### 5. Configurar ClusterIssuer para Let's Encrypt
```powershell
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: tu-email@ejemplo.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

## Construcción y Push de Imágenes Docker

### Servicio Valoraciones
```powershell
cd aks_fastapi_02

# Login en ACR
az acr login --name miRegistro

# Construir imagen
docker build -t miRegistro.azurecr.io/valoraciones:latest .

# Subir imagen
docker push miRegistro.azurecr.io/valoraciones:latest
```

### Servicio Consultas
```powershell
cd aks_fastapi_01

# Construir imagen
docker build -t miRegistro.azurecr.io/consultas:latest .

# Subir imagen
docker push miRegistro.azurecr.io/consultas:latest
```

## Despliegue

### 1. Actualizar los manifiestos
Edita los archivos de deployment y reemplaza `<TU_ACR>` con el nombre de tu Azure Container Registry:
- `valoraciones-deployment.yaml`
- `consultas-deployment.yaml`

### 2. Aplicar los manifiestos
```powershell
# Crear namespace
kubectl apply -f namespace.yaml

# Desplegar servicios
kubectl apply -f valoraciones-deployment.yaml -n fastapi-services
kubectl apply -f consultas-deployment.yaml -n fastapi-services

# Verificar deployments
kubectl get deployments -n fastapi-services
kubectl get pods -n fastapi-services

# Desplegar Ingress
kubectl apply -f ingress.yaml -n fastapi-services
```

### 3. Obtener la IP del Ingress
```powershell
kubectl get ingress -n fastapi-services

# O para ver la IP externa del Load Balancer
kubectl get service -n ingress-nginx
```

### 4. Configurar DNS
Una vez que tengas la IP pública del Ingress Controller, configura un registro DNS A en tu proveedor de DNS:
```
api.midominio.com -> <IP_PUBLICA_INGRESS>
```

## Verificación

### Comprobar que los pods están ejecutándose
```powershell
kubectl get pods -n fastapi-services
kubectl logs -f deployment/valoraciones-deployment -n fastapi-services
kubectl logs -f deployment/consultas-deployment -n fastapi-services
```

### Probar los endpoints (una vez configurado el DNS)
```powershell
# Probar valoraciones
curl https://api.midominio.com/valoraciones

# Probar consultas
curl https://api.midominio.com/consultas
```

### Probar localmente (antes de configurar DNS)
```powershell
# Obtener la IP del Ingress
$INGRESS_IP = kubectl get ingress api-ingress -n fastapi-services -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Probar con curl usando el header Host
curl -H "Host: api.midominio.com" http://$INGRESS_IP/valoraciones
curl -H "Host: api.midominio.com" http://$INGRESS_IP/consultas
```

## Troubleshooting

### Ver logs de los pods
```powershell
kubectl logs -l app=valoraciones -n fastapi-services
kubectl logs -l app=consultas -n fastapi-services
```

### Ver eventos
```powershell
kubectl get events -n fastapi-services --sort-by='.lastTimestamp'
```

### Verificar el Ingress
```powershell
kubectl describe ingress api-ingress -n fastapi-services
```

### Ver logs del NGINX Ingress Controller
```powershell
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

## Escalado

### Escalar manualmente
```powershell
kubectl scale deployment valoraciones-deployment --replicas=3 -n fastapi-services
kubectl scale deployment consultas-deployment --replicas=3 -n fastapi-services
```

### Autoescalado (HPA)
```powershell
kubectl autoscale deployment valoraciones-deployment --cpu-percent=50 --min=2 --max=10 -n fastapi-services
kubectl autoscale deployment consultas-deployment --cpu-percent=50 --min=2 --max=10 -n fastapi-services
```

## Actualización de servicios

```powershell
# Construir nueva versión
docker build -t miRegistro.azurecr.io/valoraciones:v2 .
docker push miRegistro.azurecr.io/valoraciones:v2

# Actualizar deployment
kubectl set image deployment/valoraciones-deployment valoraciones=miRegistro.azurecr.io/valoraciones:v2 -n fastapi-services

# Ver el rollout
kubectl rollout status deployment/valoraciones-deployment -n fastapi-services
```

## Limpieza

```powershell
# Eliminar todos los recursos
kubectl delete namespace fastapi-services

# Desinstalar NGINX Ingress Controller
helm uninstall nginx-ingress -n ingress-nginx
kubectl delete namespace ingress-nginx
```
