# AWS Deploy

Documentación de despliegue de la aplicación AgoraCare en AWS.

## Contenido

### Despliegues

- **Producción**: [Deploy `web-public` (Next.js) en Producción](deploy-web-public-production.md)  
  Guía completa para desplegar en ECS con Fargate, ALB, VPC multi-AZ y configuración production-ready.

- **POC**: [Deploy `web-public` (POC)](deploy-web-public-poc.md)  
  Versión proof-of-concept para pruebas y validación.

- **Producción**: [Deploy `web-register` (Vite + React) en Producción](deploy-web-register-production.md)  
  Guía para desplegar web-register usando la infraestructura existente de web-public (ALB compartido, path `/signup/*`).

### Otros Servicios

- [Servicios en VM y ECS](servicios_VM-ECS.md)
