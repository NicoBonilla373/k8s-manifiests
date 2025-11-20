# Diagrama de Red - VPC y Networking
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AWS Region: us-east-1                          │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                        VPC (Default VPC)                              │ │
│  │                        CIDR: 172.31.0.0/16                            │ │
│  │                                                                       │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │ │
│  │  │              Public Subnets (AZ: us-east-1a, us-east-1b)        │ │ │
│  │  │                                                                  │ │ │
│  │  │  ┌──────────────────────────────────────────────────────────┐  │ │ │
│  │  │  │    Internet Gateway                                      │  │ │ │
│  │  │  └────────────────┬─────────────────────────────────────────┘  │ │ │
│  │  │                   │                                             │ │ │
│  │  │  ┌────────────────▼──────────────────────────────────────────┐ │ │ │
│  │  │  │  Network Load Balancer (NLB) - Frontend                   │ │ │ │
│  │  │  │  k8s-usersapp-usersfro-*.elb.us-east-1.amazonaws.com     │ │ │ │
│  │  │  │  Port: 80 (HTTP)                                          │ │ │ │
│  │  │  │  Security Group: Allow 0.0.0.0/0:80                       │ │ │ │
│  │  │  └───────────────────┬───────────────────────────────────────┘ │ │ │
│  │  │                      │                                          │ │ │
│  │  │  ┌───────────────────▼──────────────────────────────────────┐  │ │ │
│  │  │  │  Network Load Balancer (NLB) - Backend                   │  │ │ │
│  │  │  │  a609f7874991b400f8b65f83e42b0f5d-*.elb.us-east-1...    │  │ │ │
│  │  │  │  Port: 80 (HTTP)                                          │  │ │ │
│  │  │  │  Security Group: Allow 0.0.0.0/0:80                       │  │ │ │
│  │  │  └───────────────────┬───────────────────────────────────────┘  │ │ │
│  │  │                      │                                          │ │ │
│  │  │  ┌───────────────────▼──────────────────────────────────────┐  │ │ │
│  │  │  │           EKS Cluster: users-app-cluster                 │  │ │ │
│  │  │  │           Kubernetes Version: 1.34                       │  │ │ │
│  │  │  │                                                           │  │ │ │
│  │  │  │  ┌───────────────────────────────────────────────────┐  │  │ │ │
│  │  │  │  │  Worker Nodes (EC2 Instances)                     │  │  │ │ │
│  │  │  │  │  - ip-172-31-18-195.ec2.internal (t3.medium)      │  │  │ │ │
│  │  │  │  │  - ip-172-31-34-40.ec2.internal (t3.medium)       │  │  │ │ │
│  │  │  │  │                                                    │  │  │ │ │
│  │  │  │  │  Pods Running:                                     │  │  │ │ │
│  │  │  │  │  - users-frontend (2 replicas)                    │  │  │ │ │
│  │  │  │  │  - users-backend (2 replicas)                     │  │  │ │ │
│  │  │  │  │  - notification-service (2 replicas)              │  │  │ │ │
│  │  │  │  └───────────────────────────────────────────────────┘  │  │ │ │
│  │  │  └──────────────────────────────────────────────────────────┘  │ │ │
│  │  └─────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                       │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │ │
│  │  │          Private Subnets (AZ: us-east-1a, us-east-1b)           │ │ │
│  │  │                                                                  │ │ │
│  │  │  ┌───────────────────────────────────────────────────────────┐ │ │ │
│  │  │  │  RDS PostgreSQL Instance                                  │ │ │ │
│  │  │  │  Identifier: users-db                                     │ │ │ │
│  │  │  │  Instance Class: db.t3.micro                              │ │ │ │
│  │  │  │  Engine: PostgreSQL 15.4                                  │ │ │ │
│  │  │  │  Endpoint: users-db.cc1gm1bc4kuk.us-east-1.rds.aws.com   │ │ │ │
│  │  │  │  Port: 5432                                               │ │ │ │
│  │  │  │  Multi-AZ: No                                             │ │ │ │
│  │  │  │  Storage: 20 GB (Encrypted)                               │ │ │ │
│  │  │  │  Security Group: users-rds-sg                             │ │ │ │
│  │  │  │    - Allow 172.31.0.0/16:5432 (VPC CIDR)                 │ │ │ │
│  │  │  │    - Allow sg-EKS-nodes:5432                              │ │ │ │
│  │  │  └───────────────────────────────────────────────────────────┘ │ │ │
│  │  └─────────────────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                        API Gateway (REST API)                         │ │
│  │  ID: v1yi4o5fmd                                                       │ │
│  │  Type: Regional                                                       │ │
│  │  Stage: prod                                                          │ │
│  │  URL: https://v1yi4o5fmd.execute-api.us-east-1.amazonaws.com/prod   │ │
│  │                                                                       │ │
│  │  Resources:                                                           │ │
│  │    /users (GET, POST) → http://backend-nlb/api/users/                │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    Amazon ECR (Container Registry)                    │ │
│  │  - 211865220371.dkr.ecr.us-east-1.amazonaws.com/users-frontend:prod  │ │
│  │  - 211865220371.dkr.ecr.us-east-1.amazonaws.com/users-backend:latest │ │
│  │  - 211865220371.dkr.ecr.us-east-1.amazonaws.com/notification-service │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘

## Security Groups

### Frontend LoadBalancer SG
- Ingress: 0.0.0.0/0 → Port 80 (HTTP)
- Egress: All traffic

### Backend LoadBalancer SG  
- Ingress: 0.0.0.0/0 → Port 80 (HTTP)
- Egress: All traffic

### EKS Nodes SG
- Ingress: 
  - LoadBalancer SGs → Ports 80, 8000, 3000, 5000
  - Self reference → All ports (node-to-node)
- Egress: All traffic

### RDS SG (users-rds-sg)
- Ingress:
  - 172.31.0.0/16 → Port 5432 (PostgreSQL)
  - EKS Nodes SG → Port 5432
- Egress: All traffic

## Network Flow

1. **User Request → Frontend**:
```
   Internet → Internet Gateway → Frontend NLB (Public) 
   → EKS Node → users-frontend Pod (Port 80)
```

2. **Frontend → Backend API**:
```
   users-frontend Pod → API Gateway (HTTPS)
   → Backend NLB → EKS Node → users-backend Pod (Port 8000)
```

3. **Backend → Database**:
```
   users-backend Pod → RDS Endpoint (Private) 
   → PostgreSQL (Port 5432)
```

4. **Backend → Notification**:
```
   users-backend Pod → notification-service (ClusterIP interno)
   → notification-service Pod (Port 5000) → Gmail SMTP (Internet)
```

## Subnets Configuration

- **Public Subnets**: Contienen LoadBalancers y NAT Gateways
- **Private Subnets**: Contienen RDS y recursos internos
- **EKS Subnets**: Pueden ser públicas o privadas según configuración

