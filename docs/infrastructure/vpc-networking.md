---
title: VPC y Networking
---

# VPC y Networking

> **Contexto**: Usamos la VPC existente `agoracare-vpc` (vpc-021432a1f69b421c6) en región `sa-east-1`, CIDR `10.0.0.0/16`, con 4 subnets existentes (2 public, 2 private). Faltan: 2 protected subnets, 2 NAT GWs, RT protegida, actualizar RTs privadas, VPC Endpoints, Flow Logs.

---

### 1.1 — Verificar VPC y DNS (ya habilitados ✅)

| Parámetro | Valor |
|-----------|-------|
| VPC ID | `agoracare-vpc` (vpc-021432a1f69b421c6) |
| CIDR | `10.0.0.0/16` |
| Región | `sa-east-1` |
| State | `Available` |
| Owner | `982759206940` |
| DNS resolution | ✅ habilitado |
| DNS hostnames | ✅ habilitado |

---

### 1.2 — Habilitar Auto-assign IPv4 en subnets públicas

| Subnet | ID | AZ | Acción |
|--------|----|----|--------|
| `agoracare-subnet-public1-sa-east-1a` | `subnet-0f0d14e5e3f020d9c` | sa-east-1a | ✅ Habilitar |
| `agoracare-subnet-public2-sa-east-1b` | `subnet-005c08244dd6264b1` | sa-east-1b | ✅ Habilitar |

**Verificación:** 2 public subnets con "Auto-assign public IPv4: Yes"

---

### 1.3 — Crear 2 Protected Subnets

| Nombre | AZ | CIDR |
|--------|----|------|
| `agoracare-subnet-protected1-sa-east-1a` | sa-east-1a (sae1-az1) | `10.0.32.0/20` |
| `agoracare-subnet-protected2-sa-east-1b` | sa-east-1b (sae1-az2) | `10.0.48.0/20` |

> Auto-assign IPv4: ❌ No (default)

**Verificación:** 6 subnets total (2 public, 2 private, 2 protected)

---

### 1.4 — Route Table: Protegida

| Parámetro | Valor |
|-----------|-------|
| Nombre | `agoracare-rtb-protected` |
| VPC | `agoracare-vpc` (vpc-021432a1f69b421c6) |
| Rutas | Solo `local` (NO agregar `0.0.0.0/0`) |
| Asociaciones | `agoracare-subnet-protected1-sa-east-1a` ✅, `agoracare-subnet-protected2-sa-east-1b` ✅ |

---

### 1.5 — NAT Gateways (crear en subnets públicas)

**Primero: 2 Elastic IPs**

| Nombre | Network border group |
|--------|---------------------|
| `agoracare-nat-eip-sa-east-1a` | sa-east-1 |
| `agoracare-nat-eip-sa-east-1b` | sa-east-1 |

**Luego: 2 NAT Gateways**

| Nombre | Subnet | AZ | Connectivity | EIP |
|--------|--------|----|--------------|-----|
| `agoracare-nat-sa-east-1a` | `agoracare-subnet-public1-sa-east-1a` (subnet-0f0d14e5e3f020d9c) | sa-east-1a | Public | `agoracare-nat-eip-sa-east-1a` |
| `agoracare-nat-sa-east-1b` | `agoracare-subnet-public2-sa-east-1b` (subnet-005c08244dd6264b1) | sa-east-1b | Public | `agoracare-nat-eip-sa-east-1b` |

> Esperar estado `Available` (2-3 min). Anotar IDs: `nat-xxx` (a) con EIP `x.x.x.x`, `nat-xxx` (b) con EIP `x.x.x.x`

---

### 1.6 — Actualizar RTs privadas con rutas a NAT GW

| Route Table | ID | Ruta a agregar | Target |
|-------------|----|----------------|--------|
| `agoracare-rtb-private1-sa-east-1a` | `rtb-040730bce4d7c4e9a` | `0.0.0.0/0` | NAT Gateway → `agoracare-nat-sa-east-1a` |
| `agoracare-rtb-private2-sa-east-1b` | `rtb-07c149f8da4cbc5ab` | `0.0.0.0/0` | NAT Gateway → `agoracare-nat-sa-east-1b` |

**Verificación:** Cada RT privada tiene ruta `0.0.0.0/0 → su NAT GW` + ruta `local`

---

### 1.7 — VPC Endpoints (7 endpoints, región **sa-east-1**)

#### Security Group para Interface Endpoints

| Parámetro | Valor |
|-----------|-------|
| Nombre | `agoracare-vpc-endpoints-sg` |
| Descripción | SG for VPC Interface Endpoints |
| VPC | `agoracare-vpc` (vpc-021432a1f69b421c6) |
| Inbound | HTTPS (443) → Source: `10.0.0.0/16` (temporal, restringir en security.md) |

#### Gateway Endpoints (2) — en TODAS las 5 RTs

| Nombre | Servicio | Tipo | Route Tables | Policy |
|--------|----------|------|-------------|--------|
| `agoracare-s3-endpoint` | `com.amazonaws.sa-east-1.s3` | Gateway | ✅ 5 RTs (public, private1-a, private2-b, protected, main) | Full Access |
| `agoracare-dynamodb-endpoint` | `com.amazonaws.sa-east-1.dynamodb` | Gateway | ✅ 5 RTs | Full Access |

#### Interface Endpoints (6) — en 2 subnets privadas

| Nombre | Servicio | Subnets | SG | Private DNS |
|--------|----------|---------|----|-------------|
| `agoracare-ecr-api-endpoint` | `com.amazonaws.sa-east-1.ecr.api` | ✅ private1-a, ✅ private2-b | `agoracare-vpc-endpoints-sg` | Enable |
| `agoracare-ecr-dkr-endpoint` | `com.amazonaws.sa-east-1.ecr.dkr` | ✅ private1-a, ✅ private2-b | ↑ | Enable |
| `agoracare-ecs-endpoint` | `com.amazonaws.sa-east-1.ecs` | ✅ private1-a, ✅ private2-b | ↑ | Enable |
| `agoracare-cloudwatch-endpoint` | `com.amazonaws.sa-east-1.logs` | ✅ private1-a, ✅ private2-b | ↑ | Enable |
| `agoracare-secretsmanager-endpoint` | `com.amazonaws.sa-east-1.secretsmanager` | ✅ private1-a, ✅ private2-b | ↑ | Enable |
| `agoracare-xray-endpoint` | `com.amazonaws.sa-east-1.xray` | ✅ private1-a, ✅ private2-b | ↑ | Enable |

---

### 1.8 — Flow Logs

| Parámetro | Valor |
|-----------|-------|
| Filter | All |
| Destination | CloudWatch Logs |
| Log group | `/aws/vpc/agoracare-vpc/flowlogs` |
| IAM role | Crear nuevo (`flowlogs-role`) o usar existente |
| Retention | 30 days |

**Verificación (5-10 min):** CloudWatch → Log groups → `/aws/vpc/agoracare-vpc/flowlogs` debe tener log streams

---

### 1.9 — DNS

✅ Ya verificado en sección 1.1 (DNS resolution + DNS hostnames habilitados)

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

### Subnets (6 total)

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
|--------|---------|-------|-------------------|
| Main | `rtb-00692f2b0a9ab1b5a` | local only | (ninguna) |
| agoracare-rtb-public | `rtb-0aa1fcde940580eae` | `0.0.0.0/0 → igw-0034bb77004e8f88b` | 2 public |
| agoracare-rtb-private1-sa-east-1a | `rtb-040730bce4d7c4e9a` | `0.0.0.0/0 → NAT GW a` + local | private1-a + protected1-a |
| agoracare-rtb-private2-sa-east-1b | `rtb-07c149f8da4cbc5ab` | `0.0.0.0/0 → NAT GW b` + local | private2-b + protected2-b |
| **agoracare-rtb-protected** (nueva) | `rtb-xxx` | solo local | 2 protected |

### NAT Gateways (2 nuevos)

| Nombre | ID Real | Subnet | AZ | EIP |
|--------|---------|--------|-----|-----|
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
