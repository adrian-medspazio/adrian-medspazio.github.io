---
title: Inventario de Servicios
---

# Inventario de Servicios Activos — Clasificación VM (EC2) vs ECS Fargate

## Leyenda

| Columna | Significado |
|---------|-------------|
| **#** | Número de orden |
| **Servicio** | Nombre del directorio/servicio |
| **Tipo** | Stack tecnológico |
| **Imagen** | ✅ = construida (`docker images`), ⏳ = pendiente (Dockerfile existe) |
| **Target** | **VM (EC2)** o **ECS Fargate** |
| **Justificación** | Razón técnica de la clasificación |

---

## 1. JAVA / QUARKUS (28 servicios)

| # | Servicio | Tipo | Imagen | Target | Justificación |
|---|----------|------|--------|--------|---------------|
| 1 | action-token | Java/Quarkus | ✅ | ECS Fargate | Stateless, HTTP, escalado horizontal |
| 2 | bucket-migration | Java/Quarkus | ✅ | ECS Fargate Task (run-once) | Job one-off migración, no service |
| 3 | campaign-center | Java/Quarkus | ✅ | ECS Fargate | Stateless, HTTP/gRPC |
| 4 | caregiver-manager | Java/Quarkus | ✅ | ECS Fargate | Stateless, HTTP |
| 5 | consent-listener | Java/Quarkus | ✅ | ECS Fargate | Event-driven (Kafka/SQS), stateless |
| 6 | dicomizer | Java/Quarkus | ✅ | ECS Fargate | Stateless, procesamiento DICOM |
| 7 | dicom-listening | Java/Quarkus | ✅ | ECS Fargate | Listener DICOM, stateless |
| 8 | dicom-validator | Java/Quarkus | ✅ | ECS Fargate | Validador DICOM, stateless |
| 9 | dicomweb-proxy | Java/Quarkus | ✅ | ECS Fargate | Proxy DICOMweb, stateless |
| 10 | email-center | Java/Quarkus | ✅ | ECS Fargate | Envío email, stateless |
| 11 | identity-manager | Java/Quarkus | ✅ | ECS Fargate | Gestión identidades, stateless |
| 12 | instance-processing | Java/Quarkus | ✅ | ECS Fargate | Procesamiento instancias, stateless |
| 13 | llm-proxy | Java/Quarkus | ✅ | ECS Fargate | Proxy LLM, stateless |
| 14 | metrics | Java/Quarkus | ✅ | ECS Fargate | Métricas, stateless |
| 15 | new-studies-notification | Java/Quarkus | ✅ | ECS Fargate | Notificaciones, event-driven |
| 16 | pdf-medical-validator | Java/Quarkus | ✅ | ECS Fargate | Validador PDF médico, stateless |
| 17 | physician-proxy-manager | Java/Quarkus | ✅ | ECS Fargate | Proxy médicos, stateless |
| 18 | premium-manager | Java/Quarkus | ✅ | ECS Fargate | Gestión premium, stateless |
| 19 | preventive-center | Java/Quarkus | ✅ | ECS Fargate | Centro preventivo, stateless |
| 20 | procuration | Java/Quarkus | ✅ | ECS Fargate | Procuraciones, stateless |
| 21 | sharing-token | Java/Quarkus | ✅ | ECS Fargate | Tokens sharing, stateless |
| 22 | sms-center | Java/Quarkus | ✅ | ECS Fargate | Centro SMS, stateless |
| 23 | support-center | Java/Quarkus | ✅ | ECS Fargate | Soporte, stateless |
| 24 | transfer-requester | Java/Quarkus | ✅ | ECS Fargate | Solicitudes transferencia, stateless |
| 25 | user-logs | Java/Quarkus | ✅ | ECS Fargate | Logs usuario, stateless |
| 26 | user-storage | Java/Quarkus | ✅ | ECS Fargate | Storage usuario, stateless |
| 27 | vector-db-manager | Java/Quarkus | ✅ | ECS Fargate | Vector DB manager, stateless |
| 28 | zipper | Java/Quarkus | ✅ | ECS Fargate | Compresión, stateless |
| 29 | **registration-delegate** | Java/Quarkus | ⏳ | ECS Fargate | Delegate registro, stateless |

---

## 2. NODE.JS / EXPRESS / NEXT.JS (8 servicios)

| # | Servicio | Tipo | Imagen | Target | Justificación |
|---|----------|------|--------|--------|---------------|
| 30 | healthcare-provider-manager | Node/Express | ✅ | ECS Fargate | Stateless API, HTTP |
| 31 | institution-proxy | Node/Express | ✅ | ECS Fargate | Proxy instituciones, stateless |
| 32 | letter-writer | Node/Express | ✅ | ECS Fargate | Generación cartas, stateless |
| 33 | main-server | Node/Express | ✅ | ECS Fargate | **API Gateway principal** |
| 34 | pdp | Node/Express | ✅ | ECS Fargate | Policy Decision Point, stateless |
| 35 | storage-manager | Node/Express | ✅ | ECS Fargate | Gestión storage, stateless |
| 36 | unilabs-downloader | Node/Express | ✅ | ECS Fargate | Downloader Unilabs, stateless |
| 37 | user-proxy | Node/Express | ✅ | ECS Fargate | Proxy usuario, stateless |
| 38 | **token-validator** | Node/Express | ⏳ | ECS Fargate | Validador JWT, stateless |

---

## 3. PYTHON (4 servicios — 0 construidos, 4 pendientes)

| # | Servicio | Tipo | Imagen | Target | Justificación |
|---|----------|------|--------|--------|---------------|
| 39 | **image-segmentation** | Python/PyTorch | ⏳ | **VM (EC2 GPU)** | **Requiere GPU (g5.xlarge+), drivers NVIDIA, modelo pesado** |
| 40 | **name-matching** | Python | ⏳ | **VM (EC2)** | Deps nativas (fuzzy, Levenshtein), apt-get heavy |
| 41 | **pdf-text-extractor** | Python | ⏳ | **VM (EC2)** | **Tesseract, poppler-utils, libgl1 — binarios nativos** |
| 42 | **dicom-pdf-content-validator** | Python | ⏳ | **VM (EC2)** | pydicom, gdcm — librerías nativas DICOM |

---

## 4. WEB UIs — NEXT.JS / REACT (5 servicios construidos)

| # | Servicio | Tipo | Imagen | Target | Justificación |
|---|----------|------|--------|--------|---------------|
| 43 | web/ui | React CRA | ✅ | **ECS Fargate** (ideal: S3+CloudFront) | App principal pacientes/DICOM; build estático → S3+CF ideal, Fargate OK para SSR híbrido |
| 44 | web/public | Next.js | ✅ | **ECS Fargate** | Next.js SSR/ISR + Image Optimization |
| 45 | web/admin | Next.js | ✅ | **ECS Fargate** | Next.js SSR admin panel |
| 46 | web/register | Next.js | ✅ | **ECS Fargate** | Next.js SSR registro |
| 47 | web/server-ui | Node/Express | ✅ | **ECS Fargate** | Express wrapper sirviendo web/ui build |

---

## 5. MICRO-FRONTENDS IFRAME (2 servicios construidos)

| # | Servicio | Tipo | Imagen | Target | Justificación |
|---|----------|------|--------|--------|---------------|
| 48 | web/iframe-mobilenumber | React | ✅ | **ECS Fargate** (ideal: S3+CloudFront) | Micro-frontend estático, iframe |
| 49 | web/iframe-password | React | ✅ | **ECS Fargate** (ideal: S3+CloudFront) | Micro-frontend estático, iframe |

---

## 6. INFRA / EDGE (4 servicios — 2 prod, 2 dev)

| # | Servicio | Tipo | Imagen | Target | Justificación |
|---|----------|------|--------|--------|---------------|
| 50 | **deferred-call** | OpenResty/Lua | ⏳ | **VM (EC2)** | **Layer 4/7 proxy, rate-limit, Lua custom, conexiones persistentes** |
| 51 | **user-registration-redirect** | Nginx | ⏳ | **ECS Fargate** | Redirect OAuth ligero, stateless |
| 52 | keycloak | Keycloak/Java | ✅ | **VM (EC2)** ⚠️ **Dev only** | **Prod: evaluar AWS Managed (Cognito) o Keycloak en RDS+EC2 HA** |
| 53 | maintenance-server | Nginx | ✅ | — **NO PROD** | Página mantenimiento, descartar |

---

## RESUMEN CONTEO

| Target | Servicios | Detalle |
|--------|-----------|---------|
| **ECS Fargate Services** | **37** | 29 Java + 8 Node + 1 Python (name-matching si se containeriza bien) |
| **ECS Fargate Tasks (one-off)** | **1** | bucket-migration |
| **EC2 VMs (GPU/Native/Proxy)** | **6** | image-segmentation (GPU), pdf-text-extractor, dicom-pdf-content-validator, name-matching, deferred-call, keycloak (si self-hosted) |
| **Ideal: S3 + CloudFront (static)** | **5 UIs** | web/ui, web/public, web/admin, web/register, iframes — *mejor que Fargate para coste/performance* |
| **NO PROD / DEPRECADOS** | **3** | maintenance-server, POC/summary, integration-test-pdp |

---

## NOTAS TÉCNICAS

- **Service Discovery**: URLs hardcodeadas en código → migrar a AWS Cloud Map / ALB service discovery
- **Secrets**: Actualmente en READMEs/env files → migrar a **AWS Secrets Manager / SSM Parameter Store**
- **Keycloak**: Decidir **Cognito (managed)** vs **Keycloak self-hosted en EC2+RDS** (coste/operación)
- **image-segmentation**: Requiere **g5.xlarge** (1x A10G GPU) + NVIDIA Container Toolkit + ECR con imagen CUDA
- **Python native deps**: Usar **multi-stage Docker** con base `amazonlinux:2023` + `dnf install` para tesseract/poppler/gdcm
