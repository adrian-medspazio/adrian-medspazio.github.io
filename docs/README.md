# Sphinx Deployment Guide

Guía de despliegue de la plataforma **Sphinx** en AWS.

Esta documentación cubre la arquitectura, infraestructura y procedimientos para desplegar y operar los servicios de Sphinx en producción.

## Contenido

| Sección | Descripción |
|---------|-------------|
| **Visión General** | Arquitectura del sistema, inventario de servicios y dependencias |
| **Infraestructura** | VPC, redes, seguridad, IAM, KMS y compute (ECS/EC2) |
| **Despliegues** | Guías paso a paso por servicio (Next.js, React, Java, Node.js) |
| **Operaciones** | Troubleshooting, monitoreo y runbooks |

## Entornos

| Entorno | Región | Cuenta |
|---------|--------|--------|
| Producción | `sa-east-1` | `982759206940` |
| Staging | `sa-east-1` | `982759206940` |
| Dev | `sa-east-1` | `982759206940` |

## Stack Principal

- **Container orchestration**: AWS ECS Fargate + EC2 ASG
- **Load balancing**: ALB + CloudFront
- **Base de datos** (futuro): RDS PostgreSQL + ElastiCache Redis
- **DNS/SSL**: Route53 + ACM
- **Secrets**: AWS Secrets Manager + KMS
- **Monitoring**: CloudWatch Logs + Alarms + X-Ray
