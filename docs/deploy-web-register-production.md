# Deploy `web-register` (Vite + React) en AWS — Producción

Guía para desplegar `web-register` en AWS ECS Fargate usando la infraestructura existente de `web-public`.

---

## Prerrequisitos

### Infraestructura Existente (ya creada para web-public)

- ✅ VPC `agoracare-vpc` con subnets públicas/privadas
- ✅ NAT Gateways (2)
- ✅ ECS Cluster `agoracare-prod`
- ✅ ALB `agoracare-web-public-alb` con HTTPS:443
- ✅ IAM Role `ecsTaskExecutionRole-web-public`
- ✅ Security Groups (ALB + ECS tasks)

### Certificado ACM

El ALB debe tener un certificado ACM que cubra `idp.agoracare.ch`:

- **Opción A:** Wildcard `*.agoracare.ch` (recomendado)
- **Opción B:** Certificado multi-dominio (`agoracare.ch`, `idp.agoracare.ch`)

Si no lo tenés, ver [Paso 9 de deploy-web-public-production.md](deploy-web-public-production.md#paso-9-solicitar-certificado-acm).

---

## Paso 1: Crear Repositorio ECR

```
AWS Console → ECR → Repositories → Create repository

  Repository type: Private
  Repository name: agoracare/web-register
  
  Encryption: AWS KMS (default)
  Scan on push: ✅ Enable
  Lifecycle policy: ✅ Enable
  
  → Edit lifecycle policy:
    - Rule 1: Expire untagged > 7 days
    - Rule 2: Keep max 50 images
  
  → Create repository
```

---

## Paso 2: Build & Push de la Imagen Docker

Desde la raíz del monorepo:

```bash
cd /home/kheops/Documents/workspace/sphinx

SHA=$(git rev-parse --short HEAD)

# Login en ECR
aws ecr get-login-password --region sa-east-1 | \
  docker login --username AWS --password-stdin \
  982759206940.dkr.ecr.sa-east-1.amazonaws.com

# Build & Push
docker buildx build \
  --platform linux/amd64 \
  -f web-register/Dockerfile \
  -t 982759206940.dkr.ecr.sa-east-1.amazonaws.com/agoracare/web-register:latest \
  -t 982759206940.dkr.ecr.sa-east-1.amazonaws.com/agoracare/web-register:$SHA \
  --push \
  .
```

**Verificación:**
```
AWS Console → ECR → Repositories → agoracare/web-register → Images
```

---

## Paso 3: Crear Task Definition

```
AWS Console → ECS → Task Definitions → Create new Task Definition

  Family: agoracare-web-register
  Launch type: Fargate
  CPU: 0.5 vCPU (512)
  Memory: 1 GB (1024)
  Task Execution Role: ecsTaskExecutionRole-web-public
  
  Container:
    - Name: web-register
    - Image: 982759206940.dkr.ecr.sa-east-1.amazonaws.com/agoracare/web-register:latest
    - Port mappings: 8080 (TCP)
    - Log configuration:
      - Log driver: awslogs
      - awslogs-group: /ecs/agoracare-web-register
      - awslogs-region: sa-east-1
      - awslogs-stream-prefix: ecs
  
  → Create
```

**Nota:** No hay secrets ni variables de runtime.

---

## Paso 4: Crear Target Group

```
AWS Console → EC2 → Target Groups → Create target group

  Target type: IP addresses
  
  Target group name: agoracare-web-register-tg
  Protocol: HTTP
  Port: 8080
  VPC: agoracare-vpc
  
  Health checks:
    - Protocol: HTTP
    - Path: /signup/
    - Port: traffic port
    - Healthy threshold: 2
    - Unhealthy threshold: 3
    - Timeout: 10 seconds
    - Interval: 30 seconds
    - Success codes: 200-399
  
  → Create
```

---

## Paso 5: Configurar Regla de ALB (Path-Based Routing)

```
AWS Console → EC2 → Load Balancers → agoracare-web-public-alb → Listeners

  → Select listener HTTPS:443 → View/edit rules
  
  → Add rule:
    
    IF:
      - Field: Path
      - Operator: Is
      - Value: /signup/*
    
    THEN:
      - Action: Forward to
      - Target: agoracare-web-register-tg
    
    → Add
  
  → Save
```

**Orden de las reglas:** Las reglas específicas (`/signup/*`) deben ir **antes** que la regla por defecto (web-public).

---

## Paso 6: Crear ECS Service

```
AWS Console → ECS → Clusters → agoracare-prod → Services → Create

  Step 1: Configure service
    
    Compute: Fargate
    Task Definition: agoracare-web-register (latest)
    Service name: agoracare-web-register-service
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
    
    Auto-assign public IP: ❌ DISABLED
    
    Load balancing:
      ✅ Use load balancing
      - Load balancer type: Application Load Balancer
      - Container: web-register:8080
      - Load balancer: agoracare-web-public-alb
      - Target group: agoracare-web-register-tg
    
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

---

## Paso 7: Crear Alarma de Errores

Reutilizar la alarma existente de `web-public` o crear una específica:

```
AWS Console → CloudWatch → Alarms → Create alarm

  Metric → Select metric → Application Load Balancers:
    
    By Target Group → agoracare-web-register-tg → HTTPCode_Target_5XX_Count
  
  Threshold:
    - Sum > 10
    - For 2 consecutive periods of 1 minute
  
  Actions:
    - SNS topic: agoracare-web-public-alarms (existing)
  
  Alarm name: agoracare-web-register-5xx-errors
  
  → Create
```

---

## Verificación

### 1. Estado del Servicio

```
AWS Console → ECS → Clusters → agoracare-prod → Services

  agoracare-web-register-service:
    - Status: ACTIVE
    - Running: 2
    - Desired: 2
    - Pending: 0
```

### 2. Health Check del Target Group

```
AWS Console → EC2 → Target Groups → agoracare-web-register-tg → Targets

  Todos los targets: healthy
```

### 3. Acceso a la Aplicación

```
URL: https://idp.agoracare.ch/signup/

Debe:
  ✅ Cargar la SPA de registro
  ✅ Mostrar el formulario de registro
  ✅ Redireccionar correctamente (SPA routing)
```

### 4. Logs

```
AWS Console → ECS → Clusters → agoracare-prod → Tasks

  → Select task → Logs tab
  
  Debería ver:
    - nginx access logs
    - Sin errores de inicio
```

---

## Costos Estimados (Mensual)

| Recurso | Configuración | Costo |
|---------|--------------|-------|
| **Fargate** | 2 tasks (0.5 vCPU, 1GB) | ~$60 |
| **ALB** | Compartido con web-public | $0 |
| **CloudWatch** | Logs + 1 alarma | ~$3 |
| **Total** | | **~$63/mes** |

**Ahorro:** Usar el ALB existente ahorra ~$21/mes vs crear un ALB nuevo.

---

## CI/CD

Los workflows están en `.github/workflows/` en la raíz del monorepo:

- `web-register.yml`: Pipeline principal (tests, build, release)
- `web-register-branchlate.yml`: Sync de traducciones

Las imágenes se publican en ECR bajo `agoracare/web-register:*`.

---

## Runbook Rápido

### Deploy de nueva versión

1. Build y push de nueva imagen a ECR
2. ECS → Task Definitions → Create new revision (nueva imagen)
3. ECS → Services → Update → Select new revision
4. Monitorear durante 15 minutos

### Rollback

1. ECS → Services → Update
2. Select previous task definition revision
3. Update
4. Monitorear

### Cambiar variables de build

1. Editar `.env.production` en `web-register/`
2. Build y push de nueva imagen (las variables se injectan en el build)
3. Nuevo deploy con la nueva revisión

---

## Referencias

- [Documentación oficial de web-register](../../web-register/docs/deployment.md)
- [Deploy web-public production](deploy-web-public-production.md)
- [ALB Path-Based Routing](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-update-rules.html)
