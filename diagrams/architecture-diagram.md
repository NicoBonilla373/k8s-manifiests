# Diagrama de Arquitectura de Microservicios
```
┌─────────────────────────────────────────────────────────────────────┐
│                           INTERNET                                  │
└────────────┬──────────────────────────────┬─────────────────────────┘
             │                              │
             │ HTTPS (HTTP)                 │ HTTPS
             │                              │
             ▼                              ▼
┌────────────────────────┐      ┌──────────────────────────┐
│   LoadBalancer (NLB)   │      │    API Gateway (REST)    │
│    (Frontend Public)   │      │   (Backend Public API)   │
└────────────┬───────────┘      └────────────┬─────────────┘
             │                               │
             │ HTTP :80                      │ HTTP :80
             │                               │
┌────────────▼───────────────────────────────▼─────────────────────┐
│                     Kubernetes Cluster (EKS)                      │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                      Namespace: users-app                    │ │
│  │                                                              │ │
│  │  ┌──────────────────┐         ┌─────────────────────────┐  │ │
│  │  │  users-frontend  │         │    users-backend        │  │ │
│  │  │   (2 replicas)   │         │     (2 replicas)        │  │ │
│  │  │   React + NGINX  │         │  Django REST API        │  │ │
│  │  │   Port: 80       │◄────────┤   Port: 8000            │  │ │
│  │  └──────────────────┘         └──────────┬──────────────┘  │ │
│  │                                           │                  │ │
│  │                                           │ HTTP :5000       │ │
│  │                                           │                  │ │
│  │                                           ▼                  │ │
│  │                              ┌────────────────────────────┐ │ │
│  │                              │  notification-service      │ │ │
│  │                              │     (2 replicas)           │ │ │
│  │                              │   Flask + SMTP             │ │ │
│  │                              │   Port: 5000               │ │ │
│  │                              └────────────┬───────────────┘ │ │
│  │                                           │                  │ │
│  └───────────────────────────────────────────┼──────────────────┘ │
│                                              │                    │
└──────────────────────────────────────────────┼────────────────────┘
                                               │
                                               │ Email (SMTP)
                                               │
                                               ▼
                                    ┌──────────────────────┐
                                    │   Gmail SMTP Server  │
                                    │   (nicolasbonilla@)  │
                                    └──────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│                    ┌────────────────────────────┐                  │
│                    │      AWS RDS PostgreSQL    │                  │
│                    │    (Private Subnet)        │                  │
│                    │    users-db.rds.aws.com    │                  │
│                    │    Port: 5432              │                  │
│                    └────────────────────────────┘                  │
│                              ▲                                      │
│                              │                                      │
│                              │ PostgreSQL Connection                │
│                              │                                      │
└──────────────────────────────┴──────────────────────────────────────┘

LEYENDA:
─────────  Comunicación HTTP/HTTPS
═════════  Comunicación Interna (ClusterIP)
```

## Servicios de Kubernetes

### Services Externos (LoadBalancer / API Gateway)
- **users-frontend**: LoadBalancer público en puerto 80
- **users-backend**: Expuesto vía API Gateway

### Services Internos (ClusterIP)
- **users-backend**: ClusterIP 10.100.157.237:80 → Pod:8000
- **notification-service**: ClusterIP 10.100.195.189:5000 → Pod:5000

## Flujo de Datos

1. **Crear Usuario**:
```
   Usuario → Frontend (LoadBalancer) → users-frontend pod
   → API Gateway → users-backend pod → RDS PostgreSQL
   → notification-service pod → Gmail SMTP
```

2. **Listar Usuarios**:
```
   Usuario → Frontend (LoadBalancer) → users-frontend pod
   → API Gateway → users-backend pod → RDS PostgreSQL
```

3. **Comunicación Interna**:
```
   users-backend → notification-service (DNS interno: notification-service:5000)
   users-backend → RDS (Endpoint: users-db.cc1gm1bc4kuk.us-east-1.rds.amazonaws.com:5432)
```

## Configuración de Seguridad

- **Frontend**: Público, accesible desde Internet
- **Backend**: Público solo vía API Gateway
- **Notification Service**: Privado, solo accesible internamente
- **RDS**: Privado, solo accesible desde el cluster EKS
- **Secrets**: Credenciales de DB y SMTP almacenadas en Kubernetes Secrets
- **ConfigMaps**: URLs y configuración no sensible

