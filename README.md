# Kubernetes Manifests - Users App

> Manifiestos de Kubernetes para despliegue en AWS EKS

## Descripción

Conjunto completo de manifiestos de Kubernetes para desplegar la aplicación de gestión de usuarios en AWS EKS. Incluye configuraciones para deployments, services, secrets, configmaps y jobs de migración.

**Características principales:**
- Deployments con múltiples réplicas
- Services (LoadBalancer y ClusterIP)
- ConfigMaps para configuración
- Secrets para datos sensibles
- Jobs para migraciones de base de datos
- Resource limits y requests
- Health checks y readiness probes
- Labels y selectors consistentes

## Arquitectura de Despliegue

```
Namespace: users-app
│
├── ConfigMaps
│   └── backend-env-config
│
├── Secrets
│   ├── db-credentials
│   └── smtp-credentials
│
├── Jobs
│   └── django-migrations
│
├── Deployments
│   ├── users-frontend (2 replicas)
│   ├── users-backend (2 replicas)
│   └── notification-service (2 replicas)
│
└── Services
    ├── users-frontend (LoadBalancer)
    ├── users-backend (LoadBalancer)
    └── notification-service (ClusterIP)
```

## Estructura del Proyecto

```
k8s-manifiests/
├── namespace.yaml              # Namespace principal
├── backend-env-config.yaml     # ConfigMap para backend
│
├── backend/
│   ├── migration-job.yaml      # Job de migraciones Django
│   ├── deployment.yaml         # Deployment del backend
│   └── service.yaml            # Service LoadBalancer
│
├── frontend/
│   ├── deployment.yaml         # Deployment del frontend
│   └── service.yaml            # Service LoadBalancer
│
├── notification/
│   ├── deployment.yaml         # Deployment de notificaciones
│   └── service.yaml            # Service ClusterIP
│
├── secrets/
│   ├── db-credentials.yaml     # Template para secret de BD
│   └── smtp-credentials.yaml   # Template para secret de SMTP
│
├── scripts/
│   ├── deploy-all.sh           # Script de despliegue completo
│   ├── create-secrets.sh       # Script para crear secrets
│   └── cleanup.sh              # Script de limpieza
│
└── README.md                   # Este archivo
```

## Inicio Rápido

### Prerrequisitos

```bash
# Verificar conexión al cluster
kubectl cluster-info

# Verificar nodos
kubectl get nodes

# Configurar contexto (si es necesario)
aws eks update-kubeconfig --region us-east-1 --name users-app-cluster
```

### Despliegue Rápido

```bash
# 1. Clonar repositorio
git clone https://github.com/NicoBonilla373/k8s-manifiests.git
cd k8s-manifiests

# 2. Crear namespace
kubectl apply -f namespace.yaml

# 3. Crear secrets (editar primero con tus credenciales)
kubectl create secret generic db-credentials \
  --from-literal=DB_HOST=your-rds-endpoint \
  --from-literal=DB_PORT=5432 \
  --from-literal=DB_NAME=postgres \
  --from-literal=DB_USER=masteruser \
  --from-literal=DB_PASSWORD=your-password \
  -n users-app

kubectl create secret generic smtp-credentials \
  --from-literal=SMTP_HOST=smtp.gmail.com \
  --from-literal=SMTP_PORT=587 \
  --from-literal=SMTP_USER=your-email@gmail.com \
  --from-literal=SMTP_PASSWORD=your-app-password \
  --from-literal=SMTP_FROM=your-email@gmail.com \
  --from-literal=ADMIN_EMAIL=admin@gmail.com \
  -n users-app

# 4. Aplicar ConfigMap
kubectl apply -f backend-env-config.yaml

# 5. Ejecutar migraciones
kubectl apply -f backend/migration-job.yaml
kubectl wait --for=condition=complete --timeout=300s job/django-migrations -n users-app

# 6. Desplegar servicios
kubectl apply -f backend/
kubectl apply -f frontend/
kubectl apply -f notification/

# 7. Verificar despliegue
kubectl get all -n users-app
```

## Componentes Detallados

### 1. Namespace

**Archivo:** `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: users-app
  labels:
    name: users-app
    environment: production
```

**Crear:**
```bash
kubectl apply -f namespace.yaml
```

### 2. ConfigMap - Backend

**Archivo:** `backend-env-config.yaml`

Contiene variables de entorno no sensibles para el backend:
- `DEBUG`
- `SECRET_KEY` (cambiar en producción)
- `ALLOWED_HOSTS`
- `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT`
- `NOTIFICATION_SERVICE_URL`
- `CORS_ALLOW_ALL_ORIGINS`

**Aplicar:**
```bash
kubectl apply -f backend-env-config.yaml
kubectl describe configmap backend-env-config -n users-app
```

### 3. Secrets

#### DB Credentials

```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_HOST=users-db.cc1gm1bc4kuk.us-east-1.rds.amazonaws.com \
  --from-literal=DB_PORT=5432 \
  --from-literal=DB_NAME=postgres \
  --from-literal=DB_USER=masteruser \
  --from-literal=DB_PASSWORD=labpassword123 \
  -n users-app
```

#### SMTP Credentials

```bash
kubectl create secret generic smtp-credentials \
  --from-literal=SMTP_HOST=smtp.gmail.com \
  --from-literal=SMTP_PORT=587 \
  --from-literal=SMTP_USER=nicolasbonillaferreira@gmail.com \
  --from-literal=SMTP_PASSWORD=nkdeofycudfrseyo \
  --from-literal=SMTP_FROM=nicolasbonillaferreira@gmail.com \
  --from-literal=ADMIN_EMAIL=nicolasbonillaferreira@gmail.com \
  -n users-app
```

**Verificar secrets:**
```bash
kubectl get secrets -n users-app
kubectl describe secret db-credentials -n users-app
```

### 4. Backend

#### Migration Job

**Archivo:** `backend/migration-job.yaml`

Job que ejecuta `python manage.py migrate` antes del despliegue.

```bash
# Aplicar job
kubectl apply -f backend/migration-job.yaml

# Esperar a que complete
kubectl wait --for=condition=complete --timeout=300s job/django-migrations -n users-app

# Ver logs
kubectl logs job/django-migrations -n users-app

# Ver estado
kubectl get job django-migrations -n users-app
```

**Características:**
- Se ejecuta una sola vez
- Usa las mismas credenciales de BD que el backend
- `backoffLimit: 3` (reintentos automáticos)
- `restartPolicy: Never`

#### Deployment

**Archivo:** `backend/deployment.yaml`

```bash
kubectl apply -f backend/deployment.yaml
kubectl get deployment users-backend -n users-app
kubectl get pods -l app=users-backend -n users-app
```

**Características:**
- **Réplicas:** 2
- **Imagen:** ECR con tu cuenta
- **Puerto:** 8000
- **Resources:**
  - Requests: 256Mi RAM, 250m CPU
  - Limits: 512Mi RAM, 500m CPU
- **Variables:** Desde ConfigMap y Secrets

**Comandos útiles:**
```bash
# Ver logs
kubectl logs -f deployment/users-backend -n users-app

# Escalar réplicas
kubectl scale deployment users-backend --replicas=3 -n users-app

# Reiniciar deployment
kubectl rollout restart deployment/users-backend -n users-app

# Ver historial
kubectl rollout history deployment/users-backend -n users-app

# Rollback
kubectl rollout undo deployment/users-backend -n users-app
```

#### Service

**Archivo:** `backend/service.yaml`

```bash
kubectl apply -f backend/service.yaml
kubectl get svc users-backend -n users-app
```

**Características:**
- **Tipo:** LoadBalancer (NLB en AWS)
- **Puerto externo:** 80
- **Puerto interno:** 8000
- **Anotaciones AWS:**
  - `service.beta.kubernetes.io/aws-load-balancer-type: "nlb"`
  - `service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"`

**Obtener URL:**
```bash
kubectl get svc users-backend -n users-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### 5. Frontend

#### Deployment

**Archivo:** `frontend/deployment.yaml`

```bash
kubectl apply -f frontend/deployment.yaml
kubectl get deployment users-frontend -n users-app
```

**Características:**
- **Réplicas:** 2
- **Imagen:** ECR con build de producción (NGINX)
- **Puerto:** 80
- **Resources:**
  - Requests: 128Mi RAM, 100m CPU
  - Limits: 256Mi RAM, 200m CPU
- **imagePullPolicy:** Always

#### Service

**Archivo:** `frontend/service.yaml`

```bash
kubectl apply -f frontend/service.yaml
kubectl get svc users-frontend -n users-app
```

**Características:**
- **Tipo:** LoadBalancer (público)
- **Puerto:** 80

**Acceder al frontend:**
```bash
FRONTEND_URL=$(kubectl get svc users-frontend -n users-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Frontend: http://$FRONTEND_URL"
curl -I http://$FRONTEND_URL
```

### 6. Notification Service

#### Deployment

**Archivo:** `notification/deployment.yaml`

```bash
kubectl apply -f notification/deployment.yaml
kubectl get deployment notification-service -n users-app
```

**Características:**
- **Réplicas:** 2
- **Imagen:** ECR
- **Puerto:** 5000
- **Resources:**
  - Requests: 128Mi RAM, 100m CPU
  - Limits: 256Mi RAM, 200m CPU
- **Variables:** Todas desde secret `smtp-credentials`

**Ver logs de notificaciones:**
```bash
kubectl logs -f deployment/notification-service -n users-app
kubectl logs -l app=notification-service -n users-app | grep EMAIL
```

#### Service

**Archivo:** `notification/service.yaml`

```bash
kubectl apply -f notification/service.yaml
kubectl get svc notification-service -n users-app
```

**Características:**
- **Tipo:** ClusterIP (solo interno)
- **Puerto:** 5000
- **DNS interno:** `notification-service.users-app.svc.cluster.local`

**Probar desde dentro del cluster:**
```bash
kubectl run test-notif --rm -it --image=curlimages/curl -n users-app -- \
  curl http://notification-service:5000/health
```

## Configuración Personalizada

### Cambiar número de réplicas

```bash
# Editar deployment
kubectl edit deployment users-backend -n users-app

# O con kubectl scale
kubectl scale deployment users-backend --replicas=5 -n users-app
```

### Actualizar imagen

```bash
# Actualizar a nueva versión
kubectl set image deployment/users-backend \
  users-backend=[ACCOUNT_ID].dkr.ecr.us-east-1.amazonaws.com/users-backend:v2.0 \
  -n users-app

# Verificar rollout
kubectl rollout status deployment/users-backend -n users-app
```

### Agregar variables de entorno

```bash
# Editar ConfigMap
kubectl edit configmap backend-env-config -n users-app

# Reiniciar pods para aplicar cambios
kubectl rollout restart deployment/users-backend -n users-app
```

### Modificar recursos

```yaml
# En deployment.yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

```bash
kubectl apply -f backend/deployment.yaml
```

## Monitoreo y Logs

### Ver estado general

```bash
# Todos los recursos
kubectl get all -n users-app

# Pods con detalles
kubectl get pods -n users-app -o wide

# Services con endpoints
kubectl get svc -n users-app

# Ver eventos
kubectl get events -n users-app --sort-by='.lastTimestamp'
```

### Logs

```bash
# Logs de un deployment
kubectl logs -f deployment/users-backend -n users-app

# Logs de un pod específico
kubectl logs users-backend-[pod-id] -n users-app

# Logs de todos los pods con label
kubectl logs -l app=users-backend -n users-app

# Logs con timestamp
kubectl logs -f deployment/users-backend -n users-app --timestamps

# Últimas 100 líneas
kubectl logs deployment/users-backend -n users-app --tail=100

# Logs de contenedor específico (si hay múltiples)
kubectl logs users-backend-[pod-id] -c users-backend -n users-app
```

### Métricas

```bash
# Uso de recursos de nodos
kubectl top nodes

# Uso de recursos de pods
kubectl top pods -n users-app

# Uso de un pod específico
kubectl top pod users-backend-[pod-id] -n users-app
```

### Describe (información detallada)

```bash
# Deployment
kubectl describe deployment users-backend -n users-app

# Pod
kubectl describe pod users-backend-[pod-id] -n users-app

# Service
kubectl describe service users-backend -n users-app

# ConfigMap
kubectl describe configmap backend-env-config -n users-app
```

## Troubleshooting

### Pods en estado CrashLoopBackOff

```bash
# Ver logs del pod
kubectl logs users-backend-[pod-id] -n users-app

# Ver eventos del pod
kubectl describe pod users-backend-[pod-id] -n users-app | grep -A 10 Events

# Ver logs del contenedor anterior (si crasheó)
kubectl logs users-backend-[pod-id] -n users-app --previous
```

### Pods en estado ImagePullBackOff

```bash
# Verificar que la imagen existe en ECR
aws ecr describe-images --repository-name users-backend

# Verificar permisos del node role
NODE_ROLE=$(aws iam list-roles --query "Roles[?contains(RoleName, 'LabEksNodeRole')].RoleName" --output text)
aws iam list-attached-role-policies --role-name $NODE_ROLE

# Adjuntar política de ECR
aws iam attach-role-policy \
  --role-name $NODE_ROLE \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

### Service no tiene EXTERNAL-IP

```bash
# Verificar que LoadBalancer se está creando
kubectl describe svc users-frontend -n users-app

# Ver eventos
kubectl get events -n users-app | grep users-frontend

# Verificar tags de subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=[VPC_ID]" \
  --query 'Subnets[*].[SubnetId,Tags]'

# Las subnets públicas deben tener:
# kubernetes.io/role/elb = 1
```

### Backend no se conecta a RDS

```bash
# Verificar secret
kubectl get secret db-credentials -n users-app -o yaml

# Verificar conectividad desde pod
kubectl run test-db --rm -it --image=postgres:15 -n users-app -- \
  psql postgresql://masteruser:password@rds-endpoint:5432/postgres -c '\l'

# Verificar security groups
aws ec2 describe-security-groups --group-ids [RDS_SG_ID]
```

### Frontend no se conecta al Backend

```bash
# Verificar que config.js tenga la URL correcta del API Gateway
kubectl exec -it deployment/users-frontend -n users-app -- cat /usr/share/nginx/html/config.js

# Reconstruir imagen con nueva URL
# Luego:
kubectl rollout restart deployment/users-frontend -n users-app
```

## Seguridad

### Mejores prácticas implementadas

- Secrets para datos sensibles
- Resource limits en todos los containers
- ClusterIP para servicios internos
- LoadBalancer solo donde es necesario
- Labels consistentes para selección
- Namespace dedicado
- No exponer RDS públicamente

### Recomendaciones adicionales

```yaml
# Agregar Security Context
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000
  capabilities:
    drop:
      - ALL

# Agregar Network Policies
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-network-policy
spec:
  podSelector:
    matchLabels:
      app: users-backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: users-frontend
```

### Rotar secrets

```bash
# 1. Actualizar secret
kubectl delete secret db-credentials -n users-app
kubectl create secret generic db-credentials \
  --from-literal=DB_PASSWORD=nueva_password \
  # ... otros valores
  -n users-app

# 2. Reiniciar deployments que usan el secret
kubectl rollout restart deployment/users-backend -n users-app
```

## Backup y Restore

### Backup de manifiestos

```bash
# Exportar todos los recursos
kubectl get all,configmap,secret -n users-app -o yaml > backup-users-app.yaml

# Exportar un recurso específico
kubectl get deployment users-backend -n users-app -o yaml > backend-deployment-backup.yaml
```

### Restore

```bash
# Aplicar backup
kubectl apply -f backup-users-app.yaml
```

## Actualización de la Aplicación

### Actualización Rolling (sin downtime)

```bash
# 1. Construir y subir nueva imagen
docker build -t users-backend:v2.0 .
docker tag users-backend:v2.0 [ACCOUNT_ID].dkr.ecr.us-east-1.amazonaws.com/users-backend:v2.0
docker push [ACCOUNT_ID].dkr.ecr.us-east-1.amazonaws.com/users-backend:v2.0

# 2. Actualizar imagen en deployment
kubectl set image deployment/users-backend \
  users-backend=[ACCOUNT_ID].dkr.ecr.us-east-1.amazonaws.com/users-backend:v2.0 \
  -n users-app

# 3. Ver progreso
kubectl rollout status deployment/users-backend -n users-app

# 4. Verificar
kubectl get pods -n users-app -l app=users-backend
```

### Rollback

```bash
# Ver historial
kubectl rollout history deployment/users-backend -n users-app

# Rollback a versión anterior
kubectl rollout undo deployment/users-backend -n users-app

# Rollback a versión específica
kubectl rollout undo deployment/users-backend --to-revision=2 -n users-app
```

## Limpieza

### Eliminar todos los recursos

```bash
# Eliminar namespace (elimina todo dentro)
kubectl delete namespace users-app

# O eliminar recursos individuales
kubectl delete -f backend/
kubectl delete -f frontend/
kubectl delete -f notification/
kubectl delete -f backend-env-config.yaml
kubectl delete secret db-credentials smtp-credentials -n users-app
```

### Script de limpieza

```bash
# Usar script incluido
chmod +x scripts/cleanup.sh
./scripts/cleanup.sh
```


## Autor

**Nicolás Bonilla** - [NicoBonilla373](https://github.com/NicoBonilla373)

## Enlaces

- [Repositorio Principal](https://github.com/NicoBonilla373/infraestructura)
- [Backend](https://github.com/NicoBonilla373/users-backend)
- [Frontend](https://github.com/NicoBonilla373/users-frontend)
- [Notification Service](https://github.com/NicoBonilla373/notification-service)
