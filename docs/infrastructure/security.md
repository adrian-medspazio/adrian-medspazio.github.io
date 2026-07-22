---
title: Seguridad (KMS, Secrets Manager, IAM, Security Groups)
---

# Seguridad — KMS, Secrets Manager, IAM, Security Groups

---

## KMS: 3 CMKs (dev, staging, prod)

| Alias | Tipo | Uso | Rotación anual |
|-------|------|-----|----------------|
| `alias/sphinx-dev` | Symmetric | Encrypt and Decrypt | ✅ Enabled |
| `alias/sphinx-staging` | Symmetric | Encrypt and Decrypt | ✅ Enabled |
| `alias/sphinx-prod` | Symmetric | Encrypt and Decrypt | ✅ Enabled |

> Key administrators: tu rol/user. Key users inicial: tu rol/user (luego agregar `ecsTaskExecutionRole-sphinx` y `ec2InstanceProfile-sphinx`).

**Verificación:** 3 keys con alias `alias/sphinx-dev`, `alias/sphinx-staging`, `alias/sphinx-prod`. Rotation: `Enabled` en las 3.

---

## Secrets Manager: Estructura de secretos

| Nombre | KMS Key | Tipo | Contenido inicial |
|--------|---------|------|-------------------|
| `sphinx/dev/placeholder` | `alias/sphinx-dev` | Other type of secret | `{ "dummy": "placeholder" }` |
| `sphinx/staging/placeholder` | `alias/sphinx-staging` | Other type of secret | `{ "dummy": "placeholder" }` |
| `sphinx/prod/placeholder` | `alias/sphinx-prod` | Other type of secret | `{ "dummy": "placeholder" }` |

**Estructura final (para referencia):**
```
sphinx/dev/{service}      → 49 secretos
sphinx/staging/{service}  → 49 secretos
sphinx/prod/{service}     → 49 secretos
Total: 147 secretos
```

Cada secreto contendrá pares como:
```json
{
  "DB_PASSWORD": "...",
  "JWT_SECRET": "...",
  "API_KEY": "...",
  "AWS_SECRET_ACCESS_KEY": "..."
}
```

---

## IAM Role: ECS Task Execution (`ecsTaskExecutionRole-sphinx`)

| Parámetro | Valor |
|-----------|-------|
| Trusted entity | AWS Service → Elastic Container Service Task |
| Role name | `ecsTaskExecutionRole-sphinx` |
| Managed policies | `AmazonECSTaskExecutionRolePolicy` |
| Inline policy | `SphinxSecretsAccess` |

**Inline policy `SphinxSecretsAccess`:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:*:*:secret:sphinx/*"
    },
    {
      "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": [
        "arn:aws:kms:us-east-1:<account>:key/<key-id-dev>",
        "arn:aws:kms:us-east-1:<account>:key/<key-id-staging>",
        "arn:aws:kms:us-east-1:<account>:key/<key-id-prod>"
      ]
    }
  ]
}
```

**Verificación:**
- Role `ecsTaskExecutionRole-sphinx` existe
- Tiene `AmazonECSTaskExecutionRolePolicy` + inline `SphinxSecretsAccess`

---

## IAM Instance Profile: EC2 ASG (`ec2InstanceProfile-sphinx`)

| Parámetro | Valor |
|-----------|-------|
| Trusted entity | AWS Service → EC2 |
| Role name | `ec2InstanceProfile-sphinx` |
| Managed policies | `AmazonEC2ContainerServiceforEC2Role`, `CloudWatchAgentServerPolicy`, `AmazonSSMManagedInstanceCore` |
| Inline policy | `SphinxEC2InstancePermissions` |

**Instance Profile:**
| Parámetro | Valor |
|-----------|-------|
| Name | `ec2InstanceProfile-sphinx` |
| Role adjunto | `ec2InstanceProfile-sphinx` |

**Inline policy `SphinxEC2InstancePermissions`:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer"
      ],
      "Resource": "arn:aws:ecr:us-east-1:<account>:repository/sphinx/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:<account>:secret:sphinx/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt"
      ],
      "Resource": [
        "arn:aws:kms:us-east-1:<account>:key/<key-id-dev>",
        "arn:aws:kms:us-east-1:<account>:key/<key-id-staging>",
        "arn:aws:kms:us-east-1:<account>:key/<key-id-prod>"
      ]
    }
  ]
}
```

**Verificación:**
- Instance Profile `ec2InstanceProfile-sphinx` existe con role adjunto
- Role tiene 3 AWS managed policies + 1 inline `SphinxEC2InstancePermissions`

---

## Security Groups (5 SGs)

### SG: ALB (`sphinx-alb-sg`)

| Dirección | Tipo | Puerto | Origen/Destino | Descripción |
|-----------|------|--------|----------------|-------------|
| Inbound | HTTP | 80 | `0.0.0.0/0` | Redirect a HTTPS |
| Inbound | HTTPS | 443 | `0.0.0.0/0` | HTTPS de usuarios/CloudFront |
| Outbound | All traffic | All | `0.0.0.0/0` | Default |

### SG: ECS (`sphinx-ecs-sg`)

| Dirección | Tipo | Puerto | Origen/Destino | Descripción |
|-----------|------|--------|----------------|-------------|
| Inbound | Custom TCP | 8042 | `sphinx-alb-sg` | ReverseProxy from ALB |
| Inbound | Custom TCP | 8080 | `sphinx-alb-sg` | Services on 8080 from ALB |
| Inbound | Custom TCP | 3000 | `sphinx-alb-sg` | UI on 3000 from ALB |
| Inbound | Custom TCP | 80 | `sphinx-alb-sg` | PEP from ALB |
| Inbound | Custom TCP | 8080 | `sphinx-ecs-sg` (self) | Service-to-service 8080 |
| Inbound | Custom TCP | 8042 | `sphinx-ecs-sg` (self) | Service-to-service 8042 |
| Inbound | Custom TCP | 3000 | `sphinx-ecs-sg` (self) | Service-to-service 3000 |
| Inbound | Custom TCP | 80 | `sphinx-ecs-sg` (self) | Service-to-service 80 |
| Outbound | HTTPS | 443 | `sphinx-vpc-endpoints-sg` | To VPC endpoints (ECR, CW, SM, X-Ray) |

### SG: EC2 (`sphinx-ec2-sg`)

| Dirección | Tipo | Puerto | Origen/Destino | Descripción |
|-----------|------|--------|----------------|-------------|
| Inbound | Custom TCP | 8080 | `sphinx-alb-sg` | Services on 8080 from ALB |
| Inbound | Custom TCP | 80 | `sphinx-alb-sg` | nginx/OpenResty from ALB |
| Inbound | Custom TCP | 8080 | `sphinx-ec2-sg` (self) | EC2-to-EC2 8080 |
| Inbound | Custom TCP | 80 | `sphinx-ec2-sg` (self) | EC2-to-EC2 80 |
| Outbound | HTTPS | 443 | `sphinx-vpc-endpoints-sg` | To VPC endpoints |
| Outbound | All traffic | All | `sphinx-ecs-sg` | To ECS services |

> NO agregar regla SSH (puerto 22). SSM usa outbound a endpoints, no inbound SSH.

### SG: DB (`sphinx-db-sg`) — futuro RDS/ElastiCache

| Dirección | Tipo | Puerto | Origen/Destino | Descripción |
|-----------|------|--------|----------------|-------------|
| Inbound | PostgreSQL | 5432 | `sphinx-ecs-sg` | RDS from ECS tasks |
| Inbound | PostgreSQL | 5432 | `sphinx-ec2-sg` | RDS from EC2 instances |
| Inbound | Custom TCP | 6379 | `sphinx-ecs-sg` | ElastiCache from ECS |
| Inbound | Custom TCP | 6379 | `sphinx-ec2-sg` | ElastiCache from EC2 |

### SG: VPC Endpoints (`sphinx-vpc-endpoints-sg`) — actualizar

| Dirección | Tipo | Puerto | Origen/Destino | Descripción |
|-----------|------|--------|----------------|-------------|
| Inbound | HTTPS | 443 | `sphinx-ecs-sg` | From ECS tasks |
| Inbound | HTTPS | 443 | `sphinx-ec2-sg` | From EC2 instances |

> Eliminar regla temporal `10.0.0.0/16`.

---

## Actualizar KMS Key Policies (agregar roles como users)

| CMK | Key users a agregar |
|-----|---------------------|
| `alias/sphinx-dev` | `ecsTaskExecutionRole-sphinx`, `ec2InstanceProfile-sphinx` |
| `alias/sphinx-staging` | `ecsTaskExecutionRole-sphinx`, `ec2InstanceProfile-sphinx` |
| `alias/sphinx-prod` | `ecsTaskExecutionRole-sphinx`, `ec2InstanceProfile-sphinx` |

---

## Resumen — Valores Reales (completar)

| Recurso | ID Real / ARN |
|---------|---------------|
| KMS CMK Dev | `arn:aws:kms:us-east-1:...:key/...` (alias/sphinx-dev) |
| KMS CMK Staging | `arn:aws:kms:us-east-1:...:key/...` (alias/sphinx-staging) |
| KMS CMK Prod | `arn:aws:kms:us-east-1:...:key/...` (alias/sphinx-prod) |
| Secrets Manager placeholder dev | `arn:aws:secretsmanager:...:secret:sphinx/dev/placeholder-...` |
| Secrets Manager placeholder staging | `arn:aws:secretsmanager:...:secret:sphinx/staging/placeholder-...` |
| Secrets Manager placeholder prod | `arn:aws:secretsmanager:...:secret:sphinx/prod/placeholder-...` |
| IAM Role ECS Exec | `arn:aws:iam::...:role/ecsTaskExecutionRole-sphinx` |
| IAM Role EC2 Instance | `arn:aws:iam::...:role/ec2InstanceProfile-sphinx` |
| Instance Profile | `arn:aws:iam::...:instance-profile/ec2InstanceProfile-sphinx` |
| SG ALB | `sg-xxxxxxxxx` (sphinx-alb-sg) |
| SG ECS | `sg-xxxxxxxxx` (sphinx-ecs-sg) |
| SG EC2 | `sg-xxxxxxxxx` (sphinx-ec2-sg) |
| SG DB | `sg-xxxxxxxxx` (sphinx-db-sg) |
| SG VPC Endpoints | `sg-xxxxxxxxx` (sphinx-vpc-endpoints-sg) |

---

## ✅ Checklist

- [ ] 3 CMKs creadas (`alias/sphinx-dev`, `staging`, `prod`) con rotación anual ON
- [ ] 3 placeholders en Secrets Manager (`sphinx/dev/placeholder`, etc.) con CMK correcta
- [ ] Role `ecsTaskExecutionRole-sphinx` con `AmazonECSTaskExecutionRolePolicy` + inline `SphinxSecretsAccess`
- [ ] Role `ec2InstanceProfile-sphinx` con 3 AWS managed policies + inline `SphinxEC2InstancePermissions`
- [ ] Instance Profile `ec2InstanceProfile-sphinx` creado y role adjunto
- [ ] 5 SGs creados con reglas correctas (ALB, ECS, EC2, DB, VPC Endpoints)
- [ ] SG `sphinx-vpc-endpoints-sg` actualizado (sin `10.0.0.0/16`, solo desde ECS/EC2 SGs)
- [ ] CMKs actualizadas con `ecsTaskExecutionRole-sphinx` y `ec2InstanceProfile-sphinx` como key users
- [ ] **Completar tabla "Valores Reales" con ARNs/IDs de tu cuenta**
