
# Deploy `web-public` (Next.js) en AWS — PoC

---

## Prerrequisitos

### 1. VPC Existente

Esta guía asume que ya se cuenta con:

| Recurso | Nombre esperado | Cómo verificar |
|---------|-----------------|----------------|
| VPC | `agoracare-vpc` | AWS Console → VPC → Debe existir |
| Subnets Públicas (2) | `agoracare-subnet-public1`, `agoracare-subnet-public2` | VPC → Subnets → Filtrar por VPC |
| Internet Gateway | Adjunto a la VPC | VPC → Internet Gateway → Debe estar "attached" |
| Route Table Pública | Con ruta `0.0.0.0/0 → IGW` | VPC → Route Tables → Verificar rutas |

### 2. Docker Instalado

```bash
docker --version
docker ps
```

---

## Paso 1: Crear Repositorio ECR

**Objetivo:** Tener un repositorio donde subir la imagen Docker de `web-public`.

```
AWS Console → ECR → Repositories → Create repository

  Visibility: Private
  Repository name: agoracare/web-public
  
  Scan on push: ✅ Enable basic scanning
  
  → Create repository
```

**Verificación:** El repositorio debe aparecer en la lista de ECR con visibilidad privada.

---

## Paso 2: Build & Push de la Imagen Docker

**Objetivo:** Subir la imagen de `web-public` a ECR.

### 2.1 Login en ECR

Desde la terminal, autenticar Docker con ECR:

```bash
aws ecr get-login-password --region sa-east-1 | \
  docker login --username AWS --password-stdin \
  982759206940.dkr.ecr.sa-east-1.amazonaws.com
```

**Resultado esperado:** `Login Succeeded`

---

### 2.2 Build & Push

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

**Tiempo estimado:** 3-5 minutos (depende de la conexión)

---

### 2.3 Verificar imagen subida

En AWS Console → ECR → `agoracare/web-public` → Ver las imágenes con tags `latest` y el SHA del commit.

> **Nota:** El Dockerfile de `web-public` usa `output: 'standalone'` en `next.config.js` para una imagen ligera (~200MB).

---

## Paso 3: Crear IAM Role para ECS Tasks

**Objetivo:** Role que permite a ECS descargar imágenes de ECR y escribir logs en CloudWatch.

```
AWS Console → IAM → Roles → Create role

  Step 1: Choose trusted entity
    - Trusted entity type: AWS service
    - Use case: Elastic Container Service → Elastic Container Service Task
    → Next

  Step 2: Add permissions
    - ✅ AmazonECSTaskExecutionRolePolicy (AWS managed)
    → Next

  Step 3: Name, review, and create
    - Role name: agoracare-ecs-task-execution-role
    → Create role
```

**Verificación:** El role debe aparecer en IAM → Roles con el nombre `agoracare-ecs-task-execution-role`.

---

## Paso 4: Crear ECS Cluster

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

**Verificación:** El cluster debe aparecer en estado `ACTIVE` en la lista de ECS → Clusters.

---

## Paso 5: Crear Task Definition

**Objetivo:** Definir cómo se ejecuta el contenedor de `web-public`.

```
AWS Console → ECS → Task Definitions → Create new Task Definition

  Step 1: Configure task definition
    - Family: agoracare-web-public
    - Launch type: Fargate
    - Operating system: Linux
    - CPU: 0.5 vCPU (512)
    - Memory: 1 GB (1024)
    - Task Role: (none)
    - Task Execution Role: agoracare-ecs-task-execution-role
    
    → Next

  Step 2: Configure container
    - Container name: web-public
    - Image: 982759206940.dkr.ecr.sa-east-1.amazonaws.com/agoracare/web-public:latest
    - Port mappings: 3000 (TCP)
    
    - Log configuration:
      Log driver: awslogs
      awslogs-group: /ecs/agoracare-web-public
      awslogs-region: sa-east-1
      awslogs-create-group: true
      awslogs-stream-prefix: ecs
    
    → Next

  Step 3: Review and create
    → Create
```

> **Importante:** No configurar health check a nivel de contenedor. El health check lo realiza el ALB (ver Paso 7).

**Verificación:** La task definition debe aparecer en ECS → Task Definitions con la familia `agoracare-web-public`.

---

## Paso 6: Crear Security Group para ECS Tasks

**Objetivo:** Security Group que permite tráfico desde el ALB hacia las tasks.

```
AWS Console → EC2 → Security Groups → Create security group

  Basic details:
    - Security group name: agoracare-web-public-sg
    - Description: SG for web-public ECS tasks
    - VPC: agoracare-vpc
  
  Inbound rules: (agregar después de crear el ALB SG)
    - Type: Custom TCP
    - Port: 3000
    - Source: Custom → Buscar el SG del ALB (paso 8)
  
  Outbound rules: (default, permitir todo)
  
  → Create security group
```

**Verificación:** El security group debe aparecer en EC2 → Security Groups asociado a la VPC correcta.

---

## Paso 7: Crear Target Group

**Objetivo:** Crear el Target Group que recibirá el tráfico del ALB y lo distribuirá a las tasks.

> **Importante:** Crear el Target Group **antes** del ALB porque el wizard del ALB necesita seleccionarlo.

```
AWS Console → EC2 → Target Groups → Create target group

  Step 1: Choose target type
    - Target type: IP addresses (importante para Fargate)
  
  Step 2: Configure target group
    - Target group name: agoracare-web-public-tg
    - Protocol: HTTP
    - Port: 3000
    - VPC: agoracare-vpc
    
    - Health checks:
      - Protocol: HTTP
      - Path: /
      - Port: traffic port
      - Healthy threshold: 2
      - Unhealthy threshold: 3
      - Timeout: 10 seconds
      - Interval: 30 seconds
      - Success codes: 200-399 (acepta redirects 307)
    
    → Create target group
```

**Verificación:** El target group debe aparecer en EC2 → Target Groups con tipo `IP` y puerto `3000`.

---

## Paso 8: Crear Application Load Balancer (ALB)

**Objetivo:** Crear el ALB que distribuirá el tráfico de Internet hacia el Target Group.

```
AWS Console → EC2 → Load Balancers → Create load balancer → Application Load Balancer

  Step 1: Configure load balancer
    - Name: agoracare-web-public-alb
    - Scheme: Internet-facing
    - IP address type: IPv4
  
  Step 2: Network mapping
    - VPC: agoracare-vpc
    - Subnets: 
      ✅ agoracare-subnet-public1
      ✅ agoracare-subnet-public2
  
  Step 3: Security groups
    - Create new security group:
      - Name: agoracare-web-public-alb-sg
      - Inbound rules:
        - Type: HTTP, Port: 80, Source: 0.0.0.0/0
        - Type: HTTPS, Port: 443, Source: 0.0.0.0/0
  
  Step 4: Listeners and routing
    - Protocol: HTTP
    - Port: 80
    - Default action: Forward to
    - Target group: agoracare-web-public-tg (creado en Paso 7)
  
  → Create load balancer
```

**Tiempo de espera:** 2-3 minutos hasta que el estado sea `active`.

**Verificación:** El ALB debe aparecer en estado `active` en EC2 → Load Balancers. Copiar el DNS name para pruebas.

---

## Paso 9: Actualizar Security Group de ECS Tasks

**Objetivo:** Permitir tráfico desde el ALB hacia las tasks.

```
AWS Console → EC2 → Security Groups → Buscar: agoracare-web-public-sg

  → Inbound rules → Edit inbound rules → Add rule:
    - Type: Custom TCP
    - Port: 3000
    - Source: Custom → Buscar: agoracare-web-public-alb-sg
  
  → Save rules
```

**Verificación:** El security group debe tener una regla de entrada que permite puerto 3000 desde el security group del ALB.

---

## Paso 10: Crear ECS Service

**Objetivo:** Desplegar el servicio que ejecuta las tasks de `web-public` y las registra en el ALB.

```
AWS Console → ECS → Clusters → agoracare-prod → Services tab → Create

  Step 1: Configure service
    - Compute: Fargate
    - Task Definition: agoracare-web-public (latest revision)
    - Service name: agoracare-web-public-service
    - Service type: REPLICA
    - Desired tasks: 1
    - Deployment type: Rolling update
      - Min healthy %: 50
      - Max healthy %: 200
    
    → Next

  Step 2: Configure networking
    - VPC: agoracare-vpc
    - Subnets: 
      ✅ agoracare-subnet-public1
      ✅ agoracare-subnet-public2
    - Security groups:
      ✅ agoracare-web-public-sg
    - Auto-assign public IP: ENABLED
    
    - Load balancing:
      ✅ Use load balancing
      - Load balancer type: Application Load Balancer
      - Container: web-public:3000
      - Select: Use an existing load balancer
      - Load balancer: agoracare-web-public-alb
      - Target group: agoracare-web-public-tg
    
    → Next

  Step 3: Configure Auto Scaling (opcional para PoC)
    - Skip o configurar después
    
    → Next

  Step 4: Review and create
    → Create service
```

---

## Verificación Final

### 1. Estado del Servicio

En ECS → Clusters → `agoracare-prod` → Services:
- `agoracare-web-public-service` debe mostrar:
  - Status: `ACTIVE`
  - Running: `1`
  - Desired: `1`
  - Pending: `0`

### 2. Estado de la Task

En ECS → Clusters → `agoracare-prod` → Tasks:
- La task debe estar en estado `RUNNING`
- Health status: `UNKNOWN` (normal, ya que no hay container health check)

### 3. Health Check del ALB

En EC2 → Target Groups → `agoracare-web-public-tg` → Targets:
- El target debe mostrar estado `healthy`

### 4. Prueba de Acceso

Abrir en el navegador el DNS del ALB
Debe cargar la aplicación `web-public` correctamente.

---
