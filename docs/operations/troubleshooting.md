---
title: Troubleshooting
---

# Troubleshooting

Guía de resolución de problemas organizada por categoría.

---

## VPC / Networking

| Error | Causa | Solución |
|-------|-------|----------|
| `NAT Gateway creation failed` | Subnet sin IGW route o EIP limit | Verificar RT pública tiene `0.0.0.0/0 → IGW`; pedir aumento de EIP limit en Service Quotas |
| `VPC Endpoint creation failed` | SG no permite 443 desde subnets | Abrir SG `agoracare-vpc-endpoints-sg` a `10.0.0.0/16` temporal; restringir después |
| `Flow logs not appearing` | IAM role sin permisos | Verificar `flowlogs-role` tiene `logs:CreateLogGroup`, `logs:PutLogEvents` |
| `Cannot resolve VPC endpoint DNS` | Private DNS disabled | Crear endpoint con "Private DNS name: Enable"; verificar `enableDnsHostnames/Support=true` en VPC |
| `Subnet association failed` | RT ya asociada a subnet | Una subnet solo puede tener una RT; desasociar primero |

---

## Security (KMS / IAM / Secrets Manager)

| Error | Causa | Solución |
|-------|-------|----------|
| `AccessDenied` al crear secreto | KMS key policy no permite al user | Agregar tu user/role en Key users de la CMK |
| `ECS task cannot pull image` | Execution role sin ECR permissions | Verificar `AmazonECSTaskExecutionRolePolicy` adjunto |
| `ECS task cannot read secret` | Falta `secretsmanager:GetSecretValue` o `kms:Decrypt` | Revisar inline policy `SphinxSecretsAccess` en execution role |
| `EC2 instance cannot register with ECS` | Instance profile sin `AmazonEC2ContainerServiceforEC2Role` | Verificar policy adjunta al role |
| `SG rule "invalid source"` | Self-reference SG no existe aún | Crear SG primero, luego editar reglas |
| `Cannot resolve VPC endpoint` | SG endpoint no permite tráfico desde ECS/EC2 SG | Actualizar `sphinx-vpc-endpoints-sg` inbound rules |

---

## ECS / Deploy

| Situación | Causa | Solución |
|-----------|-------|----------|
| Service no alcanza estado `ACTIVE` | Task definition mal configurada o imagen no encontrada | Verificar que la imagen existe en ECR y que el execution role tiene permisos ECR |
| Tareas en estado `PENDING` por más de 5 min | Falta capacidad Fargate en la AZ o VPC endpoints mal configurados | Verificar VPC endpoints (ECR API/DKR, ECS, CloudWatch Logs) |
| Health check del target group falla | Puerto incorrecto o ruta de health check no responde | Verificar que el contenedor escucha en el puerto configurado y la ruta existe |
| `Auto-assign public IP: DISABLED` pero tarea no tiene salida a internet | Falta NAT Gateway o ruta `0.0.0.0/0` en RT privada | Verificar RT privada tiene ruta `0.0.0.0/0 → NAT GW` |
| Error `CannotPullContainerError` | Execution role no tiene `ecr:GetAuthorizationToken` | Verificar `AmazonECSTaskExecutionRolePolicy` adjunto al execution role |

### Runbook Rápido: Deploy de nueva versión

1. Build y push de nueva imagen a ECR
2. ECS → Task Definitions → Create new revision (nueva imagen)
3. ECS → Services → Update → Select new revision
4. Monitorear durante 15 minutos

### Runbook Rápido: Rollback

1. ECS → Services → Update
2. Select previous task definition revision
3. Update
4. Monitorear

---

## Monitoreo / Alarmas

| Situación | Causa | Solución |
|-----------|-------|----------|
| Alarma 5XX no se dispara | Threshold muy alto o métrica no configurada | Verificar que el ALB tiene logging habilitado y la métrica `HTTPCode_Target_5XX_Count` existe |
| No llegan notificaciones de alarma | Suscripción SNS no confirmada | Revisar email y confirmar suscripción al SNS topic |
| Alarma en estado `INSUFFICIENT_DATA` | No hay datos de la métrica en el período | Verificar que el target group está recibiendo tráfico; puede ser normal en horas de baja actividad |

### Crear alarma de errores 5XX

```
AWS Console → CloudWatch → Alarms → Create alarm

  Metric → Select metric → Application Load Balancers:
    By Target Group → <nombre-tg> → HTTPCode_Target_5XX_Count

  Threshold:
    - Threshold type: Static
    - Whenever sum is greater than 10
    - For 2 consecutive periods of 1 minute

  Actions:
    - Alarm state trigger → Create new topic (o usar existente)
    - Topic name: <nombre>-alarms
    - Email: <tu-email>

  Alarm name: <nombre>-5xx-errors

  → Create
```

**Confirmar** la suscripción desde el email recibido.

### Pasos de verificación post-deploy

```
1. Estado del Servicio:
   AWS Console → ECS → Clusters → <cluster> → Services
     - Status: ACTIVE
     - Running: 2
     - Desired: 2
     - Pending: 0

2. Health Check del Target Group:
   AWS Console → EC2 → Target Groups → <tg> → Targets
     - Todos los targets: healthy

3. Acceso a la Aplicación:
   URL: https://<dominio><ruta>
     - ✅ Carga correctamente
     - ✅ Sin errores HTTP

4. Logs:
   AWS Console → ECS → Clusters → <cluster> → Tasks
     → Select task → Logs tab
     - Sin errores de inicio
     - Logs de aplicación visibles
```
