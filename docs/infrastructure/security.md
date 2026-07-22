---
title: Seguridad (KMS, Secrets Manager, IAM, Security Groups)
---

# Seguridad — KMS, Secrets Manager, IAM, Security Groups

**Servicios usados:** KMS, Secrets Manager, IAM, EC2 → Security Groups

---

## KMS: 3 CMKs (dev, staging, prod)

```
AWS Console → KMS → Customer managed keys → Create key (×3)

  Key 1:
    Key type: Symmetric
    Key usage: Encrypt and decrypt
    → Next
    Alias: alias/sphinx-dev
    Description: CMK for Sphinx dev environment
    → Next
    Key administrators: (tu rol/user)
    → Next
    Key users: (tu rol/user) + luego agregaremos ecsTaskExecutionRole-sphinx
    → Next
    → Finish

  Key 2:
    Alias: alias/sphinx-staging
    Description: CMK for Sphinx staging environment
    (mismos admins/users)

  Key 3:
    Alias: alias/sphinx-prod
    Description: CMK for Sphinx production environment
    (mismos admins/users)
```

**Habilitar rotación automática (anual) en las 3:**
```
KMS → Customer managed keys → Click key ID → Key rotation → Edit → ✅ Automatically rotate this KMS key every year → Save changes
```

**Verificación:**
- 3 keys con alias `alias/sphinx-dev`, `alias/sphinx-staging`, `alias/sphinx-prod`
- Rotation: `Enabled` en las 3

---

## Secrets Manager: Estructura de secretos

**Crear placeholder inicial (se poblará por servicio):**

```
AWS Console → Secrets Manager → Store a new secret (×3)

  Secret 1:
    Secret type: Other type of secret
    Key/value pairs:
      dummy: "placeholder"
    → Next
    Secret name: sphinx/dev/placeholder
    Description: Placeholder for dev secrets structure
    KMS key: alias/sphinx-dev
    → Next → Store

  Secret 2:
    Secret name: sphinx/staging/placeholder
    KMS key: alias/sphinx-staging
    → Store

  Secret 3:
    Secret name: sphinx/prod/placeholder
    KMS key: alias/sphinx-prod
    → Store
```

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

```
AWS Console → IAM → Roles → Create role

  Trusted entity type: AWS service
  Use case: Elastic Container Service → Elastic Container Service Task
  → Next

  Permissions policies (buscar y seleccionar):
    ☑ AmazonECSTaskExecutionRolePolicy
  → Next

  Role name: ecsTaskExecutionRole-sphinx
  Description: Execution role for Sphinx ECS tasks (ECR pull, CW Logs, Secrets Manager)
  → Create role
```

**Agregar permisos para Secrets Manager + KMS (inline policy):**

```
Role ecsTaskExecutionRole-sphinx → Permissions → Add permissions → Create inline policy

  Service: Secrets Manager
  Actions: GetSecretValue
  Resources: All resources (o restringir a arn:aws:secretsmanager:*:*:secret:sphinx/*)
  → Next

  Service: KMS
  Actions: Decrypt
  Resources: 
    - arn:aws:kms:us-east-1:<account>:key/<key-id-dev>
    - arn:aws:kms:us-east-1:<account>:key/<key-id-staging>
    - arn:aws:kms:us-east-1:<account>:key/<key-id-prod>
  → Next

  Policy name: SphinxSecretsAccess
  → Create policy
```

**Verificación:**
- Role `ecsTaskExecutionRole-sphinx` existe
- Tiene `AmazonECSTaskExecutionRolePolicy` + inline `SphinxSecretsAccess`

---

## IAM Instance Profile: EC2 ASG (`ec2InstanceProfile-sphinx`)

```
AWS Console → IAM → Roles → Create role

  Trusted entity type: AWS service
  Use case: EC2
  → Next

  Permissions policies (buscar y seleccionar):
    ☑ AmazonEC2ContainerServiceforEC2Role
    ☑ CloudWatchAgentServerPolicy
    ☑ AmazonSSMManagedInstanceCore
  → Next

  Role name: ec2InstanceProfile-sphinx
  Description: Instance profile for Sphinx EC2 ASG instances (ECR, CW, SSM, ECS agent)
  → Create role
```

**Crear Instance Profile:**

```
IAM → Roles → ec2InstanceProfile-sphinx → Permissions → (ver policies adjuntas)
IAM → Instance profiles → Create instance profile

  Name: ec2InstanceProfile-sphinx
  → Next
  Add role: ec2InstanceProfile-sphinx
  → Create instance profile
```

**Agregar permisos ECR + Secrets Manager + KMS (inline policy en el role):**

```
Role ec2InstanceProfile-sphinx → Permissions → Add permissions → Create inline policy

  JSON:
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

  Policy name: SphinxEC2InstancePermissions
  → Create policy
```

**Verificación:**
- Instance Profile `ec2InstanceProfile-sphinx` existe con role adjunto
- Role tiene 3 AWS managed policies + 1 inline `SphinxEC2InstancePermissions`

---

## Security Groups (5 SGs)

### SG ALB (`sphinx-alb-sg`)

```
AWS Console → EC2 → Security Groups → Create security group

  Security group name: sphinx-alb-sg
  Description: ALB public access (HTTP/HTTPS from internet)
  VPC: sphinx-prod
  → Create security group
```

```
Inbound rules → Edit inbound rules → Add rule (×2)

  Rule 1:
    Type: HTTP (80)
    Source: Anywhere-IPv4 (0.0.0.0/0)
    Description: HTTP redirect to HTTPS

  Rule 2:
    Type: HTTPS (443)
    Source: Anywhere-IPv4 (0.0.0.0/0)
    Description: HTTPS from users / CloudFront

  → Save rules
```

**Outbound rules:** (default: All traffic 0.0.0.0/0 — dejar así)

---

### SG ECS (`sphinx-ecs-sg`)

```
AWS Console → EC2 → Security Groups → Create security group

  Security group name: sphinx-ecs-sg
  Description: ECS Fargate tasks (private subnets)
  VPC: sphinx-prod
  → Create security group
```

```
Inbound rules → Edit inbound rules → Add rule (×4)

  Rule 1:
    Type: Custom TCP
    Port range: 8042
    Source: Custom → sphinx-alb-sg (seleccionar el SG)
    Description: ReverseProxy from ALB

  Rule 2:
    Type: Custom TCP
    Port range: 8080
    Source: Custom → sphinx-alb-sg
    Description: Services on 8080 from ALB

  Rule 3:
    Type: Custom TCP
    Port range: 3000
    Source: Custom → sphinx-alb-sg
    Description: UI on 3000 from ALB

  Rule 4:
    Type: Custom TCP
    Port range: 80
    Source: Custom → sphinx-alb-sg
    Description: PEP from ALB

  → Save rules
```

**Reglas adicionales para comunicación interna (entre servicios):**
```
Inbound rules → Edit inbound rules → Add rule (×4)

  Rule 5:
    Type: Custom TCP
    Port range: 8080
    Source: Custom → sphinx-ecs-sg (self-reference)
    Description: Service-to-service 8080

  Rule 6:
    Type: Custom TCP
    Port range: 8042
    Source: Custom → sphinx-ecs-sg
    Description: Service-to-service 8042

  Rule 7:
    Type: Custom TCP
    Port range: 3000
    Source: Custom → sphinx-ecs-sg
    Description: Service-to-service 3000

  Rule 8:
    Type: Custom TCP
    Port range: 80
    Source: Custom → sphinx-ecs-sg
    Description: Service-to-service 80

  → Save rules
```

**Outbound rules → Edit outbound rules → Add rule:**
```
  Type: HTTPS (443)
  Destination: Custom → sphinx-vpc-endpoints-sg
  Description: To VPC endpoints (ECR, CW, SM, X-Ray)
  → Save rules
```

---

### SG EC2 (`sphinx-ec2-sg`)

```
AWS Console → EC2 → Security Groups → Create security group

  Security group name: sphinx-ec2-sg
  Description: EC2 ASG instances (private subnets)
  VPC: sphinx-prod
  → Create security group
```

```
Inbound rules → Edit inbound rules → Add rule (×2)

  Rule 1:
    Type: Custom TCP
    Port range: 8080
    Source: Custom → sphinx-alb-sg
    Description: Services on 8080 from ALB

  Rule 2:
    Type: Custom TCP
    Port range: 80
    Source: Custom → sphinx-alb-sg
    Description: nginx/OpenResty from ALB

  → Save rules
```

**Comunicación interna (self-reference):**
```
Inbound rules → Edit inbound rules → Add rule (×2)

  Rule 3:
    Type: Custom TCP
    Port range: 8080
    Source: Custom → sphinx-ec2-sg
    Description: EC2-to-EC2 8080

  Rule 4:
    Type: Custom TCP
    Port range: 80
    Source: Custom → sphinx-ec2-sg
    Description: EC2-to-EC2 80

  → Save rules
```

**Acceso SSM (para debugging sin SSH):**
```
Inbound rules → NO agregar regla SSH (puerto 22)
# SSM usa outbound a endpoints, no inbound SSH
```

**Outbound rules → Edit outbound rules → Add rule (×2):**
```
  Rule 1:
    Type: HTTPS (443)
    Destination: Custom → sphinx-vpc-endpoints-sg
    Description: To VPC endpoints

  Rule 2:
    Type: All traffic
    Destination: Custom → sphinx-ecs-sg
    Description: To ECS services
  → Save rules
```

---

### SG DB (`sphinx-db-sg`) — para futuro RDS/ElastiCache

```
AWS Console → EC2 → Security Groups → Create security group

  Security group name: sphinx-db-sg
  Description: Future RDS PostgreSQL / ElastiCache Redis (protected subnets)
  VPC: sphinx-prod
  → Create security group
```

```
Inbound rules → Edit inbound rules → Add rule (×4)

  Rule 1:
    Type: PostgreSQL (5432)
    Source: Custom → sphinx-ecs-sg
    Description: RDS from ECS tasks

  Rule 2:
    Type: PostgreSQL (5432)
    Source: Custom → sphinx-ec2-sg
    Description: RDS from EC2 instances

  Rule 3:
    Type: Custom TCP
    Port range: 6379
    Source: Custom → sphinx-ecs-sg
    Description: ElastiCache from ECS

  Rule 4:
    Type: Custom TCP
    Port range: 6379
    Source: Custom → sphinx-ec2-sg
    Description: ElastiCache from EC2

  → Save rules
```

---

### SG VPC Endpoints (`sphinx-vpc-endpoints-sg`) — actualizar regla temporal

```
AWS Console → EC2 → Security Groups → sphinx-vpc-endpoints-sg → Inbound rules → Edit inbound rules
```

**Eliminar regla temporal `10.0.0.0/16` y agregar:**
```
  Rule 1:
    Type: HTTPS (443)
    Source: Custom → sphinx-ecs-sg
    Description: From ECS tasks

  Rule 2:
    Type: HTTPS (443)
    Source: Custom → sphinx-ec2-sg
    Description: From EC2 instances

  → Save rules
```

---

## Actualizar KMS Key Policies (agregar roles como users)

```
AWS Console → KMS → Customer managed keys → alias/sphinx-dev → Key policy → Edit
```

**En "Key users" agregar:**
- `ecsTaskExecutionRole-sphinx`
- `ec2InstanceProfile-sphinx`

**Repetir para `alias/sphinx-staging` y `alias/sphinx-prod`**

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
