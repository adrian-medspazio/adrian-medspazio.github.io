---
title: VPC y Networking
---

# VPC y Networking

> **Contexto**: Usamos la VPC existente `agoracare-vpc` (vpc-021432a1f69b421c6) en región `sa-east-1`, CIDR `10.0.0.0/16`, con 4 subnets existentes (2 public, 2 private). Faltan: 2 protected subnets, 2 NAT GWs, RT protegida, actualizar RTs privadas, VPC Endpoints, Flow Logs.

**Servicio usado:** VPC → Your VPCs → `agoracare-vpc` (vpc-021432a1f69b421c6)

---

### 1.1 — Verificar VPC y DNS (ya habilitados ✅)

```
AWS Console → VPC → Your VPCs → agoracare-vpc (vpc-021432a1f69b421c6) → Details
  → Verificar: CIDR `10.0.0.0/16`, State `Available`, Owner `982759206940`
```

```
Actions → Edit DNS resolution → Enable (ya está ✅)
Actions → Edit DNS hostnames → Enable (ya está ✅)
```

---

### 1.2 — Habilitar Auto-assign IPv4 en subnets públicas

```
AWS Console → VPC → Subnets → Filter by VPC: agoracare-vpc
→ Select agoracare-subnet-public1-sa-east-1a (subnet-0f0d14e5e3f020d9c)
  → Actions → Modify auto-assign IP settings → ✅ Enable → Save
→ Select agoracare-subnet-public2-sa-east-1b (subnet-005c08244dd6264b1)
  → Actions → Modify auto-assign IP settings → ✅ Enable → Save
```

**Verificación:**
- Subnets → Filter by VPC `agoracare-vpc` → 2 public subnets con "Auto-assign public IPv4: Yes"

---

### 1.3 — Crear 2 Protected Subnets (nuevas)

```
AWS Console → VPC → Subnets → Create subnet (×2)

  Subnet 1 (AZ sa-east-1a):
    VPC: agoracare-vpc (vpc-021432a1f69b421c6)
    Availability Zone: sa-east-1a (sae1-az1)
    Name tag: agoracare-subnet-protected1-sa-east-1a
    IPv4 CIDR block: 10.0.32.0/20
    → Create subnet

  Subnet 2 (AZ sa-east-1b):
    VPC: agoracare-vpc
    Availability Zone: sa-east-1b (sae1-az2)
    Name tag: agoracare-subnet-protected2-sa-east-1b
    IPv4 CIDR block: 10.0.48.0/20
    → Create subnet
```

> **Auto-assign IPv4:** ❌ No (ya por defecto)

**Verificación:**
- Subnets → Filter by VPC `agoracare-vpc` → 6 subnets total (2 public, 2 private, 2 protected)

---

### 1.4 — Crear Route Table Protegida (nueva)

```
AWS Console → VPC → Route Tables → Create route table
  Name: agoracare-rtb-protected
  VPC: agoracare-vpc (vpc-021432a1f69b421c6)
  → Create route table
```

> **NO agregar ruta `0.0.0.0/0`** — solo queda la ruta `local`.

```
Route Tables → agoracare-rtb-protected → Subnet associations → Edit subnet associations
  ✅ agoracare-subnet-protected1-sa-east-1a
  ✅ agoracare-subnet-protected2-sa-east-1b
  → Save associations
```

---

### 1.5 — Crear 2 NAT Gateways (en subnets públicas)

**Primero: Crear 2 Elastic IPs (EIPs) — obligatorio antes de NAT Gateway**

```
AWS Console → VPC → Elastic IPs → Allocate Elastic IP address (×2)

  EIP 1:
    Name: agoracare-nat-eip-sa-east-1a
    Network border group: sa-east-1
    → Allocate

  EIP 2:
    Name: agoracare-nat-eip-sa-east-1b
    Network border group: sa-east-1
    → Allocate
```

**Ahora: Crear 2 NAT Gateways asignando cada EIP**

```
AWS Console → VPC → NAT Gateways → Create NAT Gateway (×2)

  NAT Gateway 1:
    Name: agoracare-nat-sa-east-1a
    Subnet: agoracare-subnet-public1-sa-east-1a (subnet-0f0d14e5e3f020d9c)
    Connectivity type: Public
    Elastic IP allocation ID: (seleccionar agoracare-nat-eip-sa-east-1a)
    → Create NAT Gateway

  NAT Gateway 2:
    Name: agoracare-nat-sa-east-1b
    Subnet: agoracare-subnet-public2-sa-east-1b (subnet-005c08244dd6264b1)
    Connectivity type: Public
    Elastic IP allocation ID: (seleccionar agoracare-nat-eip-sa-east-1b)
    → Create NAT Gateway
```

> **Esperar estado `Available`** (2-3 min). Anotar IDs:
> - `nat-xxx` (sa-east-1a) con EIP `x.x.x.x`
> - `nat-xxx` (sa-east-1b) con EIP `x.x.x.x`

---

### 1.6 — Actualizar RTs privadas con rutas a NAT

**RT Private 1 (sa-east-1a):**
```
Route Tables → agoracare-rtb-private1-sa-east-1a (rtb-040730bce4d7c4e9a)
→ Routes → Edit routes → Add route
  Destination: 0.0.0.0/0
  Target: NAT Gateway → agoracare-nat-sa-east-1a
  → Save routes
```

**RT Private 2 (sa-east-1b):**
```
Route Tables → agoracare-rtb-private2-sa-east-1b (rtb-07c149f8da4cbc5ab)
→ Routes → Edit routes → Add route
  Destination: 0.0.0.0/0
  Target: NAT Gateway → agoracare-nat-sa-east-1b
  → Save routes
```

**Verificación:**
- Cada RT privada tiene ruta `0.0.0.0/0 → su NAT GW` + ruta `local`

---

### 1.7 — VPC Endpoints (7 endpoints, región **sa-east-1**)

#### Primero: Security Group para Interface Endpoints

```
AWS Console → VPC → Security Groups → Create security group
  Security group name: agoracare-vpc-endpoints-sg
  Description: SG for VPC Interface Endpoints
  VPC: agoracare-vpc (vpc-021432a1f69b421c6)
  → Create security group
```

```
Inbound rules → Edit inbound rules → Add rule
  Type: HTTPS (443)
  Source: Custom → 10.0.0.0/16  (temporal, restringir en security.md)
  → Save rules
```

#### Gateway Endpoints (2) — en TODAS las RTs (5):

```
AWS Console → VPC → Endpoints → Create Endpoint (×2)

  Endpoint 1:
    Name tag: agoracare-s3-endpoint
    Service category: AWS services
    Service: com.amazonaws.sa-east-1.s3 (Type: Gateway)
    VPC: agoracare-vpc
    Route tables: ✅ agoracare-rtb-public (rtb-0aa1fcde940580eae), ✅ agoracare-rtb-private1-sa-east-1a (rtb-040730bce4d7c4e9a), ✅ agoracare-rtb-private2-sa-east-1b (rtb-07c149f8da4cbc5ab), ✅ agoracare-rtb-protected (nueva), ✅ rtb-00692f2b0a9ab1b5a (main)
    Policy: Full Access
    → Create endpoint

  Endpoint 2:
    Name tag: agoracare-dynamodb-endpoint
    Service: com.amazonaws.sa-east-1.dynamodb (Gateway)
    VPC: agoracare-vpc
    Route tables: ✅ TODAS las 5
    Policy: Full Access
    → Create endpoint
```

#### Interface Endpoints (6) — en subnets PRIVADAS (2 AZs):

```
AWS Console → VPC → Endpoints → Create Endpoint (×6)

  Para CADA uno:
    VPC: agoracare-vpc
    Subnets: ✅ agoracare-subnet-private1-sa-east-1a (subnet-009db096ef7cc5c52), ✅ agoracare-subnet-private2-sa-east-1b (subnet-09e78a1b1c4a3e4df)
    Security group: agoracare-vpc-endpoints-sg
    Private DNS name: Enable

  1. Name: agoracare-ecr-api-endpoint
     Service: com.amazonaws.sa-east-1.ecr.api (Interface)

  2. Name: agoracare-ecr-dkr-endpoint
     Service: com.amazonaws.sa-east-1.ecr.dkr (Interface)

  3. Name: agoracare-ecs-endpoint
     Service: com.amazonaws.sa-east-1.ecs (Interface)

  4. Name: agoracare-cloudwatch-endpoint
     Service: com.amazonaws.sa-east-1.logs (Interface)

  5. Name: agoracare-secretsmanager-endpoint
     Service: com.amazonaws.sa-east-1.secretsmanager (Interface)

  6. Name: agoracare-xray-endpoint
     Service: com.amazonaws.sa-east-1.xray (Interface)
```

---

### 1.8 — Flow Logs

```
AWS Console → VPC → Your VPCs → agoracare-vpc → Flow logs → Create flow log
  Filter: All
  Destination: Send to CloudWatch Logs
  Log group: /aws/vpc/agoracare-vpc/flowlogs
  IAM role: Create new role (flowlogs-role) o usar existente
  Retention setting: 30 days
  → Create flow log
```

**Verificación (en 5-10 min):**
- CloudWatch → Log groups → `/aws/vpc/agoracare-vpc/flowlogs` → Debe tener log streams

---

### 1.9 — DNS (verificar ✅ ya habilitado)

```
VPC → agoracare-vpc → Actions
  Edit DNS resolution → Enable (ya está ✅)
  Edit DNS hostnames → Enable (ya está ✅)
```

---

## Resumen — Valores Reales (completar con tus IDs)

### Infraestructura Base (ya existente)

| Recurso | ID Real | Detalle |
|---------|---------|---------|
| **Región** | `sa-east-1` | — |
| **Account ID** | `982759206940` | — |
| **VPC ID** | `vpc-021432a1f69b421c6` | `10.0.0.0/16` |
| **IGW** | `igw-0034bb77004e8f88b` | attached |
| **Main RT** | `rtb-00692f2b0a9ab1b5a` | local only |

### Subnets (6 total tras crear 2 protected)

| Nombre | ID Real | CIDR | AZ | RT Asociada | Auto-assign IPv4 |
|--------|---------|------|-----|-------------|------------------|
| agoracare-subnet-public1-sa-east-1a | `subnet-0f0d14e5e3f020d9c` | `10.0.0.0/20` | sa-east-1a | `agoracare-rtb-public` | ✅ |
| agoracare-subnet-public2-sa-east-1b | `subnet-005c08244dd6264b1` | `10.0.16.0/20` | sa-east-1b | `agoracare-rtb-public` | ✅ |
| agoracare-subnet-private1-sa-east-1a | `subnet-009db096ef7cc5c52` | `10.0.128.0/20` | sa-east-1a | `agoracare-rtb-private1-sa-east-1a` | ❌ |
| agoracare-subnet-private2-sa-east-1b | `subnet-09e78a1b1c4a3e4df` | `10.0.144.0/20` | sa-east-1b | `agoracare-rtb-private2-sa-east-1b` | ❌ |
| **agoracare-subnet-protected1-sa-east-1a** (nueva) | `subnet-xxx` | `10.0.32.0/20` | sa-east-1a | `agoracare-rtb-protected` | ❌ |
| **agoracare-subnet-protected2-sa-east-1b** (nueva) | `subnet-xxx` | `10.0.48.0/20` | sa-east-1b | `agoracare-rtb-protected` | ❌ |

### Route Tables (5 total)

| Nombre | ID Real | Rutas | Subnets asociadas |
|------|---------|-------|-------------------|
| Main | `rtb-00692f2b0a9ab1b5a` | local only | (ninguna) |
| agoracare-rtb-public | `rtb-0aa1fcde940580eae` | `0.0.0.0/0 → igw-0034bb77004e8f88b` | 2 public |
| agoracare-rtb-private1-sa-east-1a | `rtb-040730bce4d7c4e9a` | `0.0.0.0/0 → NAT GW a` + local | private1-a + protected1-a |
| agoracare-rtb-private2-sa-east-1b | `rtb-07c149f8da4cbc5ab` | `0.0.0.0/0 → NAT GW b` + local | private2-b + protected2-b |
| **agoracare-rtb-protected** (nueva) | `rtb-xxx` | solo local | 2 protected |

### NAT Gateways (2 nuevos)

| Nombre | ID Real | Subnet | AZ | EIP |
|------|---------|--------|-----|-----|
| **agoracare-nat-sa-east-1a** (nueva) | `nat-xxx` | subnet-0f0d14e5e3f020d9c | sa-east-1a | `x.x.x.x` |
| **agoracare-nat-sa-east-1b** (nueva) | `nat-xxx` | subnet-005c08244dd6264b1 | sa-east-1b | `x.x.x.x` |

### VPC Endpoints (7 nuevos)

| Nombre | ID Real | Tipo | Subnets / RTs |
|--------|---------|------|---------------|
| agoracare-s3-endpoint | `vpce-xxx` | Gateway | 5 RTs |
| agoracare-dynamodb-endpoint | `vpce-xxx` | Gateway | 5 RTs |
| agoracare-ecr-api-endpoint | `vpce-xxx` | Interface | 2 private subnets |
| agoracare-ecr-dkr-endpoint | `vpce-xxx` | Interface | 2 private subnets |
| agoracare-ecs-endpoint | `vpce-xxx` | Interface | 2 private subnets |
| agoracare-cloudwatch-endpoint | `vpce-xxx` | Interface | 2 private subnets |
| agoracare-secretsmanager-endpoint | `vpce-xxx` | Interface | 2 private subnets |
| agoracare-xray-endpoint | `vpce-xxx` | Interface | 2 private subnets |

### SG VPC Endpoints

| Nombre | ID Real |
|--------|---------|
| agoracare-vpc-endpoints-sg | `sg-xxx` |

### Flow Logs

| Recurso | Valor |
|---------|-------|
| Log Group | `/aws/vpc/agoracare-vpc/flowlogs` |
| Retention | 30 days |

---

## ✅ Checklist

- [ ] Auto-assign IPv4 ON en 2 public subnets
- [ ] 2 Protected subnets creadas (`10.0.32.0/20` sa-east-1a, `10.0.48.0/20` sa-east-1b)
- [ ] RT Protegida `agoracare-rtb-protected` creada + 2 subnets asociadas
- [ ] 2 NAT Gateways creados (`Available`) en public subnets
- [ ] RT Private1-a → ruta `0.0.0.0/0 → NAT GW a`
- [ ] RT Private2-b → ruta `0.0.0.0/0 → NAT GW b`
- [ ] 2 Gateway Endpoints (S3, DynamoDB) en 5 RTs
- [ ] 6 Interface Endpoints (ECR API/DKR, ECS, Logs, SM, X-Ray) en private subnets + SG `agoracare-vpc-endpoints-sg` + Private DNS Enable
- [ ] Flow Logs a `/aws/vpc/agoracare-vpc/flowlogs` (30d)
- [ ] **Tabla "Valores Reales" completada con nuevos IDs**
