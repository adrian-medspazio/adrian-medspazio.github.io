# Deploy `web-public` (Next.js) en AWS — Producción

---

## Parte 1: Infraestructura de Red (Pre-requisito)

### Paso 1: Crear VPC con Subnets Públicas y Privadas

**Objetivo:** Tener una red con subnets públicas (para ALB y NAT Gateway) y privadas (para las tasks ECS).

#### 1.1 Crear VPC

```
AWS Console → VPC → Your VPCs → Create VPC

  Name tag: agoracare-vpc
  IPv4 CIDR block: 10.0.0.0/16
  IPv6 CIDR block: No IPv6 CIDR block
  Tenancy: Default
  
  → Create
```

#### 1.2 Crear Subnets Públicas (2)

```
AWS Console → VPC → Subnets → Create subnet

  VPC: agoracare-vpc
  
  Subnet 1:
    - Name: agoracare-subnet-public1
    - Availability Zone: sa-east-1a
    - CIDR: 10.0.0.0/24
  
  Subnet 2:
    - Name: agoracare-subnet-public2
    - Availability Zone: sa-east-1b
    - CIDR: 10.0.1.0/24
  
  → Create
```

#### 1.3 Crear Subnets Privadas (2)

```
AWS Console → VPC → Subnets → Create subnet

  VPC: agoracare-vpc
  
  Subnet 3:
    - Name: agoracare-subnet-private1
    - Availability Zone: sa-east-1a
    - CIDR: 10.0.10.0/24
  
  Subnet 4:
    - Name: agoracare-subnet-private2
    - Availability Zone: sa-east-1b
    - CIDR: 10.0.11.0/24
  
  → Create
```

#### 1.4 Crear Internet Gateway

```
AWS Console → VPC → Internet gateways → Create internet gateway

  Name tag: agoracare-igw
  
  → Create
  
  → Select: agoracare-igw → Actions → Attach to VPC
  
  → Select VPC: agoracare-vpc
  
  → Attach
```

#### 1.5 Crear NAT Gateways (2, uno por AZ)

```
AWS Console → VPC → NAT gateways → Create NAT gateway

  NAT gateway 1:
    - Name: agoracare-nat-sa-east-1a
    - Subnet: agoracare-subnet-public1
    - Connectivity type: Public
    → Create
  
  NAT gateway 2:
    - Name: agoracare-nat-sa-east-1b
    - Subnet: agoracare-subnet-public2
    - Connectivity type: Public
    → Create
  
  ⏳ Esperar 3-5 minutos hasta que el estado sea "Available"
```

#### 1.6 Crear Route Tables

**Route Table Pública:**

```
AWS Console → VPC → Route tables → Create route table

  Name: agoracare-public-rt
  VPC: agoracare-vpc
  
  → Create
  
  → Routes tab → Edit routes → Add route:
    - Destination: 0.0.0.0/0
    - Target: Internet Gateway → agoracare-igw
  
  → Save changes
  
  → Route table associations → Edit associations:
    - ✅ agoracare-subnet-public1
    - ✅ agoracare-subnet-public2
  
  → Save
```

**Route Table Privada 1 (sa-east-1a):**

```
AWS Console → VPC → Route tables → Create route table

  Name: agoracare-private1-rt
  VPC: agoracare-vpc
  
  → Create
  
  → Routes tab → Edit routes → Add route:
    - Destination: 0.0.0.0/0
    - Target: NAT Gateway → agoracare-nat-sa-east-1a
  
  → Save changes
  
  → Route table associations → Edit associations:
    - ✅ agoracare-subnet-private1
  
  → Save
```

**Route Table Privada 2 (sa-east-1b):**

```
AWS Console → VPC → Route tables → Create route table

  Name: agoracare-private2-rt
  VPC: agoracare-vpc
  
  → Create
  
  → Routes tab → Edit routes → Add route:
    - Destination: 0.0.0.0/0
    - Target: NAT Gateway → agoracare-nat-sa-east-1b
  
  → Save changes
  
  → Route table associations → Edit associations:
    - ✅ agoracare-subnet-private2
  
  → Save
```

#### 1.7 Habilitar DNS en la VPC

```
AWS Console → VPC → Your VPCs → Select: agoracare-vpc

  → Actions → Edit DNS settings
  
  ✅ Enable DNS resolution
  ✅ Enable DNS hostnames
  
  → Save
```

---

### Paso 2: Security Groups

**Objetivo:** Crear los security groups para el ALB y las tasks ECS.

#### 2.1 Security Group del ALB

```
AWS Console → EC2 → Security Groups → Create security group

  Name: agoracare-alb-sg
  Description: Security group for ALB
  VPC: agoracare-vpc
  
  Inbound rules:
    - Type: HTTPS, Port: 443, Source: 0.0.0.0/0
    - Type: HTTP, Port: 80, Source: 0.0.0.0/0
  
  Outbound rules:
    - All traffic, Destination: 0.0.0.0/0
  
  → Create
```

#### 2.2 Security Group de ECS Tasks

```
AWS Console → EC2 → Security Groups → Create security group

  Name: agoracare-ecs-tasks-sg
  Description: Security group for ECS tasks
  VPC: agoracare-vpc
  
  Inbound rules:
    - Type: Custom TCP, Port: 3000, Source: Custom → agoracare-alb-sg
  
  Outbound rules:
    - All traffic, Destination: 0.0.0.0/0
  
  → Create
```

---

## Parte 2: Servicios AWS Base

### Paso 3: Crear Repositorio ECR

**Objetivo:** Tener un repositorio donde subir la imagen Docker de `web-public`.

```
AWS Console → ECR → Repositories → Create repository

  Repository type: Private
  Repository name: agoracare/web-public
  
  Encryption:
    - Encryption type: AWS KMS
    - KMS key: Default encryption key (aws/ecr)
  
  Scan on push: ✅ Enable
  
  Lifecycle policy: ✅ Enable
  
  → Edit lifecycle policy:
    
    Rule 1: Expire untagged images
      - Status: Enabled
      - Image tag status: Untagged
      - Time period: More than 7 days
    
    Rule 2: Keep minimum images
      - Status: Enabled
      - Image tag status: Any
      - Count type: Image count more than
      - Count number: 50
  
  → Create repository
```

---

### Paso 4: Crear IAM Role para ECS Tasks

**Objetivo:** Role que permite a ECS descargar imágenes de ECR, escribir logs en CloudWatch y obtener parámetros de Parameter Store.

> **⚠️ IMPORTANTE:** El nombre del role que creés acá debe coincidir EXACTAMENTE con el que uses en la Task Definition (Paso 12). Si ya tenés un role existente (ej: `ecsTaskExecutionRole-web-public`), usá ese nombre en vez de crear uno nuevo.

#### 4.1 Crear el role

```
AWS Console → IAM → Roles → Create role

  Step 1: Trusted entity
    - AWS service
    - Use case: Elastic Container Service → Elastic Container Service Task
    → Next

  Step 2: Permissions
    - ✅ AmazonECSTaskExecutionRolePolicy (AWS managed)
    → Next

  Step 3: Name and create
    - Role name: ecsTaskExecutionRole-web-public (o el nombre que uses en tu task definition)
    → Create role
```

#### 4.2 Agregar policy para Parameter Store

> **⚠️ CRUCIAL:** Sin esta policy, las tasks fallarán con `AccessDeniedException` al intentar leer parámetros de SSM.

```
AWS Console → IAM → Roles → Select: ecsTaskExecutionRole-web-public

  → Permissions tab → Add permissions → Create inline policy
  
  → JSON editor:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath"
      ],
      "Resource": "arn:aws:ssm:sa-east-1:982759206940:parameter/agoracare/web-public/prod/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:sa-east-1:982759206940:key/*"
    }
  ]
}

  → Review → Name: ParameterStoreAccess → Create policy
```

#### 4.3 Verificación

```
AWS Console → IAM → Roles → Select: ecsTaskExecutionRole-web-public → Permissions tab

  Deberías ver:
    ✅ AmazonECSTaskExecutionRolePolicy (AWS managed)
    ✅ ParameterStoreAccess (inline policy)
```

---

### Paso 5: Crear ECS Cluster

**Objetivo:** Cluster donde correrán las tasks de Fargate.

```
AWS Console → ECS → Clusters → Create cluster

  Cluster name: agoracare-prod
  
  Infrastructure:
    - ✅ Fargate (serverless)
    - ❌ EC2 (no seleccionar)
  
  Monitoring:
    - ❌ Enable Container Insights (opcional, tiene costo adicional)
  
  Tags: (opcional)
    - Environment: prod
  
  → Create
```

---

### Paso 6: Crear CloudWatch Log Group

**Objetivo:** Log group configurado con retención de 30 días.

```
AWS Console → CloudWatch → Log groups → Create log group

  Log group name: /ecs/agoracare-web-public
  
  → Create
  
  → Select log group → Actions → Edit retention → Set to 30 days
```

---

## Parte 3: Variables Sensibles (Parameter Store)

### Paso 7: Crear Parámetros en Parameter Store

**Objetivo:** No tener credentials hardcoded en la task definition.

**Costo:** $0/mes (Parameter Store standard es gratis)

```
AWS Console → Systems Manager → Parameter Store → Create parameter

  Parameter 1:
    - Name: /agoracare/web-public/prod/USER_PROXY_URL
    - Type: SecureString
    - Tier: Standard
    - Value: https://user-proxy.agoracare.ch
    - KMS Key Source: My current account
    - KMS Key ID: alias/aws/ssm (default)
    → Create parameter

  Parameter 2:
    - Name: /agoracare/web-public/prod/USER_PROXY_AUTH_USERNAME
    - Type: SecureString
    - Tier: Standard
    - Value: agoraAdmin
    - KMS Key Source: My current account
    - KMS Key ID: alias/aws/ssm
    → Create parameter

  Parameter 3:
    - Name: /agoracare/web-public/prod/USER_PROXY_AUTH_PASSWORD
    - Type: SecureString
    - Tier: Standard
    - Value: <tu password real>
    - KMS Key Source: My current account
    - KMS Key ID: alias/aws/ssm
    → Create parameter
```

**Verificación:**

```
AWS Console → Systems Manager → Parameter Store

  Filtrar por: /agoracare/web-public/prod/
  
  Deberías ver:
    ✅ /agoracare/web-public/prod/USER_PROXY_URL
    ✅ /agoracare/web-public/prod/USER_PROXY_AUTH_USERNAME
    ✅ /agoracare/web-public/prod/USER_PROXY_AUTH_PASSWORD
```

---

## Parte 4: Build & Push de la Imagen Docker

### Paso 8: Build & Push de la Imagen

**Objetivo:** Subir la imagen de `web-public` a ECR.

#### 8.1 Login en ECR

```bash
aws ecr get-login-password --region sa-east-1 | \
  docker login --username AWS --password-stdin \
  982759206940.dkr.ecr.sa-east-1.amazonaws.com
```

#### 8.2 Build & Push

Desde la raíz del monorepo `sphinx`:

```bash
cd /home/kheops/Documents/workspace/sphinx

# Obtener short SHA del commit actual
SHA=$(git rev-parse --short HEAD)

# Build y push simultáneo
docker buildx build \
  --platform linux/amd64 \
  -f web-public/Dockerfile \
  -t 982759206940.dkr.ecr.sa-east-1.amazonaws.com/agoracare/web-public:latest \
  -t 982759206940.dkr.ecr.sa-east-1.amazonaws.com/agoracare/web-public:$SHA \
  --push \
  .
```

**Verificación:**

```
AWS Console → ECR → Repositories → agoracare/web-public → Images

  Deberías ver las imágenes con tags: latest y <SHA>
```

---

## Parte 5: Load Balancer y HTTPS

### Paso 9: Solicitar Certificado ACM

**Objetivo:** Tener un certificado SSL/TLS para HTTPS.

```
AWS Console → ACM → Request certificate

  Request type: Request a public certificate
  
  Domain names:
    - agoracare.ch (o el dominio que uses para web-public)
    - www.agoracare.ch (si aplica)
  
  Validation method: DNS validation
  
  → Request
  
  → Select certificate (estado: Pending validation)
  
  → Create record in Route 53
  
  ⏳ Wait for validation (puede tardar 5-30 minutos)
  
  Estado final: Issued ✅
```

---

### Paso 10: Crear Target Group

**Objetivo:** Crear el Target Group que recibirá el tráfico del ALB.

```
AWS Console → EC2 → Target Groups → Create target group

  Target type: IP addresses
  
  Target group name: agoracare-web-public-tg
  Protocol: HTTP
  Port: 3000
  VPC: agoracare-vpc
  
  Health checks:
    - Protocol: HTTP
    - Path: /
    - Port: traffic port
    - Healthy threshold: 3
    - Unhealthy threshold: 2
    - Timeout: 5 seconds
    - Interval: 15 seconds
    - Success codes: 200-399
  
  → Create
```

---

### Paso 11: Crear ALB con HTTPS

**Objetivo:** Crear el ALB que distribuirá el tráfico HTTPS.

```
AWS Console → EC2 → Load Balancers → Create load balancer → Application Load Balancer

  Step 1: Basic configuration
    
    Name: agoracare-web-public-alb
    Scheme: Internet-facing
    IP address type: IPv4
  
  Step 2: Network mapping
    
    VPC: agoracare-vpc
    Subnets: 
      ✅ agoracare-subnet-public1
      ✅ agoracare-subnet-public2
  
  Step 3: Security groups
    
    Select: agoracare-alb-sg
  
  Step 4: Listeners and routing
    
    Listener 1:
      - Protocol: HTTPS
      - Port: 443
      - Default action: Forward to agoracare-web-public-tg
      - SSL certificate: Select from ACM (el certificado que creaste)
      - Security policy: ELBSecurityPolicy-TLS-1-2-2017-01
    
    Listener 2 (add listener):
      - Protocol: HTTP
      - Port: 80
      - Default action: Redirect
      - Redirect to: HTTPS, Port 443
      - Status code: HTTP 301
  
  → Create load balancer
```

**Esperar:** 2-5 minutos hasta que el estado sea `active`.

---

## Parte 6: ECS Task Definition y Service

### Paso 12: Crear Task Definition

**Objetivo:** Definir cómo se ejecuta el contenedor de `web-public` con parámetros inyectados.

```
AWS Console → ECS → Task Definitions → Create new Task Definition

  Step 1: Configure task definition
    
    Family: agoracare-web-public
    Launch type: Fargate
    Operating system: Linux
    CPU: 1 vCPU (1024)
    Memory: 2 GB (2048)
    
    Task Execution Role: agoracare-ecs-task-execution-role
    
    → Next

  Step 2: Configure container
    
    Container name: web-public
    Image: 982759206940.dkr.ecr.sa-east-1.amazonaws.com/agoracare/web-public:latest
    Port mappings: 3000 (TCP)
    
    Container health check: ❌ Deshabilitado (el ALB hace el health check)
    
    Log configuration:
      - Log driver: awslogs
      - awslogs-group: /ecs/agoracare-web-public
      - awslogs-region: sa-east-1
      - awslogs-stream-prefix: ecs
      - awslogs-create-group: false
    
    Secrets (⚠️ CRUCIAL):
      → Add secret → Parameter from Systems Manager Parameter Store
      
      - USER_PROXY_URL:
        - Parameter: /agoracare/web-public/prod/USER_PROXY_URL
        - Environment variable: USER_PROXY_URL
      
      - USER_PROXY_AUTH_USERNAME:
        - Parameter: /agoracare/web-public/prod/USER_PROXY_AUTH_USERNAME
        - Environment variable: USER_PROXY_AUTH_USERNAME
      
      - USER_PROXY_AUTH_PASSWORD:
        - Parameter: /agoracare/web-public/prod/USER_PROXY_AUTH_PASSWORD
        - Environment variable: USER_PROXY_AUTH_PASSWORD
    
    → Next

  Step 3: Review and create
    → Create
```

---

### Paso 13: Crear ECS Service

**Objetivo:** Desplegar el servicio que ejecuta las tasks de `web-public`.

```
AWS Console → ECS → Clusters → agoracare-prod → Services → Create

  Step 1: Configure service
    
    Compute: Fargate
    Task Definition: agoracare-web-public (latest revision)
    Service name: agoracare-web-public-service
    Service type: REPLICA
    Desired tasks: 2
    
    → Next

  Step 2: Configure networking
    
    VPC: agoracare-vpc
    Subnets:
      ✅ agoracare-subnet-private1
      ✅ agoracare-subnet-private2
    
    Security groups:
      ✅ agoracare-ecs-tasks-sg
    
    Auto-assign public IP: ❌ DISABLED (⚠️ CRUCIAL)
    
    Load balancing:
      ✅ Use load balancing
      - Load balancer type: Application Load Balancer
      - Container: web-public:3000
      - Load balancer: agoracare-web-public-alb
      - Target group: agoracare-web-public-tg
    
    → Next

  Step 3: Configure Auto Scaling
    
    ✅ Use Service Auto Scaling
    
    Minimum tasks: 2
    Maximum tasks: 4
    Desired tasks: 2
    
    Scaling policy: Target tracking
    
    Policy:
      - Metric: Average CPU utilization
      - Target value: 70%
      - Scale-out cooldown: 60 seconds
      - Scale-in cooldown: 300 seconds
    
    → Next

  Step 4: Review and create
    → Create service
```

**Esperar:** 3-5 minutos a que el service esté `ACTIVE`.

---

## Parte 7: Monitoreo

### Paso 14: Crear Alarma de Errores 5XX

**Objetivo:** Saber si la aplicación está fallando.

```
AWS Console → CloudWatch → Alarms → Create alarm

  Metric → Select metric → Application Load Balancers:
    
    By Target Group → agoracare-web-public-tg → HTTPCode_Target_5XX_Count
  
  Threshold:
    - Threshold type: Static
    - Whenever sum is greater than 10
    - For 2 consecutive periods of 1 minute
  
  Actions:
    - Alarm state trigger → Create new topic
    - Topic name: agoracare-web-public-alarms
    - Email: <tu-email@agoracare.ch>
    → Create topic
  
  Alarm name: agoracare-web-public-5xx-errors
  
  → Create
```

**Confirmar** la suscripción desde el email que recibís.

---
