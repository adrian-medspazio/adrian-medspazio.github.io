---
title: Arquitectura General
---

# Sphinx en AWS: ECS Fargate + EC2 VMs (Producción)

Arquitectura basada en el patrón de despliegue de Kheops/AgoraCare, adaptada para **49 servicios Sphinx** (37 Fargate + 6 EC2 VMs + 5 UIs Next.js + 1 React + 2 Iframes).

## Arquitectura General

```
Internet
   │
   ▼
CloudFront (5 distribuciones: web/public, web/admin, web/register, web/ui, iframes)
   │  (ACM *.sphinx.example.com, WAF, cache static 1y / HTML no-cache)
   ▼
ALB (Application Load Balancer) ─── HTTPS 443 / HTTP 80 (redirect)
   │
   ▼
┌────────────────────────────────────────────────────────────────────┐
│                    VPC: sphinx-prod-vpc                            │
│                       CIDR: 10.0.0.0/16                            │
│                    (65,536 IPs: 10.0.0.0 – 10.0.255.255)          │
│                                                                    │
│  ┌─ Public Subnet us-east-1a ──┐  ┌─ Public Subnet us-east-1b ──┐  │
│  │  ALB + NAT GW (a)            │  │  NAT GW (b)                 │  │
│  │  CIDR: 10.0.0.0/24           │  │  CIDR: 10.0.1.0/24          │  │
│  └──────────────────────────────┘  └──────────────────────────────┘  │
│                                                                    │
│  ┌─ Public Subnet us-east-1c ──┐                                    │
│  │  NAT GW (c)                  │  CIDR: 10.0.2.0/24                │
│  └──────────────────────────────┘                                    │
│                                                                    │
│  ┌─ Private Subnet us-east-1a ──┐  ┌─ Private Subnet us-east-1b ──┐ │
│  │  ECS Tasks (Fargate)          │  │  ECS Tasks (Fargate)          │ │
│  │  EC2 ASG (General)            │  │  EC2 ASG (General)            │ │
│  │  CIDR: 10.0.10.0/24           │  │  CIDR: 10.0.11.0/24           │ │
│  └──────────────────────────────┘  └──────────────────────────────┘ │
│                                                                    │
│  ┌─ Private Subnet us-east-1c ──┐                                    │
│  │  ECS Tasks (Fargate)          │  CIDR: 10.0.12.0/24               │
│  │  EC2 ASG (General)            │                                   │
│  └──────────────────────────────┘                                    │
│                                                                    │
│  ┌─ Protected Subnet us-east-1a ┐  ┌─ Protected Subnet us-east-1b ┐ │
│  │  RDS PostgreSQL (futuro)      │  │  RDS PostgreSQL (futuro)      │ │
│  │  ElastiCache Redis (futuro)   │  │  ElastiCache Redis (futuro)   │ │
│  │  CIDR: 10.0.20.0/24           │  │  CIDR: 10.0.21.0/24           │ │
│  └──────────────────────────────┘  └──────────────────────────────┘ │
│                                                                    │
│  ┌─ Protected Subnet us-east-1c ┐                                    │
│  │  RDS/ElastiCache (futuro)     │  CIDR: 10.0.22.0/24               │
│  └──────────────────────────────┘                                    │
│                                                                    │
│  Servicios por tipo:                                               │
│  ├── ECS Fargate (37): Java/Quarkus (27), Node.js (8), Next.js (4) │
│  ├── EC2 VMs (6):                                                  │
│  │   ├── GPU (g5.xlarge): image-segmentation (PyTorch)             │
│  │   ├── Native deps (t3.medium): pdf-text-extractor,              │
│  │   │                              dicom-pdf-content-validator,    │
│  │   │                              name-matching                   │
│  │   ├── OpenResty (t3.medium): deferred-call (rate-limit/proxy)   │
│  │   └── Nginx (t3.medium): user-registration-redirect (OAuth)     │
│  ├── ECS Fargate Tasks (1): bucket-migration (one-off)             │
│  └── UIs (5 Next.js + 1 React CRA + 2 Iframes) → CloudFront + S3   │
│                                                                    │
│  Cloud Map: sphinx.local (DNS privado)                             │
│  Secrets Manager: sphinx/{dev,staging,prod}/{service} (147 secretos)            │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

## Resumen de Recursos (se irá completando con valores reales)

| Recurso | Nombre en AWS | Valor Real |
|---------|---------------|------------|
| VPC | `sphinx-prod-vpc` | `vpc-xxxxxxxxx` |
| CIDR VPC | — | `10.0.0.0/16` |
| Public Subnet 1a | `sphinx-prod-public-a` | `subnet-xxxxx` / `10.0.0.0/24` |
| Public Subnet 1b | `sphinx-prod-public-b` | `subnet-xxxxx` / `10.0.1.0/24` |
| Public Subnet 1c | `sphinx-prod-public-c` | `subnet-xxxxx` / `10.0.2.0/24` |
| Private Subnet 1a | `sphinx-prod-private-a` | `subnet-xxxxx` / `10.0.10.0/24` |
| Private Subnet 1b | `sphinx-prod-private-b` | `subnet-xxxxx` / `10.0.11.0/24` |
| Private Subnet 1c | `sphinx-prod-private-c` | `subnet-xxxxx` / `10.0.12.0/24` |
| Protected Subnet 1a | `sphinx-prod-protected-a` | `subnet-xxxxx` / `10.0.20.0/24` |
| Protected Subnet 1b | `sphinx-prod-protected-b` | `subnet-xxxxx` / `10.0.21.0/24` |
| Protected Subnet 1c | `sphinx-prod-protected-c` | `subnet-xxxxx` / `10.0.22.0/24` |
| Internet Gateway | `sphinx-prod-igw` | `igw-xxxxx` |
| NAT Gateway 1a | `sphinx-prod-nat-a` | `nat-xxxxx` (EIP: `x.x.x.x`) |
| NAT Gateway 1b | `sphinx-prod-nat-b` | `nat-xxxxx` (EIP: `x.x.x.x`) |
| NAT Gateway 1c | `sphinx-prod-nat-c` | `nat-xxxxx` (EIP: `x.x.x.x`) |
| Route Table Public | `sphinx-prod-public-rt` | `rtb-xxxxx` |
| Route Table Private 1a | `sphinx-prod-private-rt-a` | `rtb-xxxxx` |
| Route Table Private 1b | `sphinx-prod-private-rt-b` | `rtb-xxxxx` |
| Route Table Private 1c | `sphinx-prod-private-rt-c` | `rtb-xxxxx` |
| Route Table Protected | `sphinx-prod-protected-rt` | `rtb-xxxxx` |
| VPC Endpoint S3 | Gateway | `vpce-xxxxx` |
| VPC Endpoint DynamoDB | Gateway | `vpce-xxxxx` |
| VPC Endpoint ECR API | Interface | `vpce-xxxxx` |
| VPC Endpoint ECR DKR | Interface | `vpce-xxxxx` |
| VPC Endpoint ECS | Interface | `vpce-xxxxx` |
| VPC Endpoint CloudWatch | Interface | `vpce-xxxxx` |
| VPC Endpoint Secrets Manager | Interface | `vpce-xxxxx` |
| VPC Endpoint X-Ray | Interface | `vpce-xxxxx` |
| Flow Logs Group | `/aws/vpc/sphinx-prod/flowlogs` | — |
| KMS CMK Dev | `alias/sphinx-dev` | `arn:aws:kms:...` |
| KMS CMK Staging | `alias/sphinx-staging` | `arn:aws:kms:...` |
| KMS CMK Prod | `alias/sphinx-prod` | `arn:aws:kms:...` |
| ECS Task Execution Role | `ecsTaskExecutionRole-sphinx` | `arn:aws:iam::...:role/...` |
| EC2 Instance Profile | `ec2InstanceProfile-sphinx` | `arn:aws:iam::...:instance-profile/...` |
| SG ALB | `sphinx-alb-sg` | `sg-xxxxx` |
| SG ECS | `sphinx-ecs-sg` | `sg-xxxxx` |
| SG EC2 | `sphinx-ec2-sg` | `sg-xxxxx` |
| SG DB (futuro) | `sphinx-db-sg` | `sg-xxxxx` |
| SG VPC Endpoints | `sphinx-vpc-endpoints-sg` | `sg-xxxxx` |
| ECS Cluster | `sphinx-prod-cluster` | `arn:aws:ecs:...:cluster/sphinx-prod-cluster` |
| ECR Repos | 49 x `sphinx/<service>` | — |
| Cloud Map Namespace | `sphinx.local` | `ns-xxxxx` |
| ALB | `sphinx-prod-alb` | `arn:aws:elasticloadbalancing:...` |
| ALB HTTPS Listener | — | `arn:aws:elasticloadbalancing:...:listener/...` |
| Target Groups | 43 x `sphinx-tg-<service>` | — |
| ACM Cert (wildcard) | `*.sphinx.example.com` | `arn:aws:acm:...:certificate/...` |
| CloudFront Distributions | 5 (web/public, admin, register, ui, iframes) | `EXXXXXXXXXXXX` |
| Route53 Zone | `sphinx.example.com` | `ZXXXXXXXXXXXX` |
| SNS Alarm Topic | `sphinx-alarms-prod` | `arn:aws:sns:...` |

## Servicios Sphinx (Orden de Dependencias para Deploy)

```
NIVEL 0 (sin deps):           user-storage, user-logs, consent-listener, email-center,
                              sms-center, metrics, support-center, preventive-center,
                              physician-proxy-manager, pdf-medical-validator,
                              new-studies-notification, llm-proxy, instance-processing,
                              identity-manager, dicomweb-proxy, dicom-validator,
                              dicom-listening, dicomizer, campaign-center, caregiver-manager,
                              bucket-migration (task), action-token, sharing-token,
                              premium-manager, procuration, transfer-requester,
                              zipper, vector-db-manager, registration-delegate

NIVEL 1 (deps nivel 0):       user-proxy, institution-proxy, healthcare-provider-manager,
                              letter-writer, unilabs-downloader, storage-manager,
                              pdp, main-server, token-validator

NIVEL 2 (Python VMs):         image-segmentation (GPU), name-matching, pdf-text-extractor,
                              dicom-pdf-content-validator

NIVEL 3 (Infra VMs):          deferred-call (OpenResty), user-registration-redirect (nginx)

NIVEL 4 (UIs):                web/public, web/admin, web/register, web/server-ui, web/ui,
                              web/iframe-mobilenumber, web/iframe-password
```

> **Nota**: El orden exacto se ajustará según dependencias reales (DB, Keycloak, etc.). Ver `service-inventory.md` para la clasificación completa VM vs Fargate.
