<h1 align="center"><em>Documentación del Sistema de Gestión de Prácticas</em></h1>

> [!NOTE]
> Este repositorio centraliza la documentación técnica del Sistema de Gestión de Prácticas.

El repositorio organiza la documentación del sistema según sus áreas principales: `backend`, `frontend` y `deployment`. Cada directorio agrupa documentos relacionados con una parte específica del proyecto para mantener una estructura clara, ordenada y fácil de consultar.

## Backend

La carpeta `backend/` contiene la documentación técnica asociada al backend del sistema. Incluye referencias por módulo funcional, componentes transversales, convenciones y plantillas.

### Core

- [`backend/core/logging.md`](backend/core/logging.md): configuración real del sistema de logging, handlers, formatos, rotación y eventos registrados.
- [`backend/core/auditoria-funcional.md`](backend/core/auditoria-funcional.md): trazabilidad funcional existente, límites de auditoría transversal y deuda pendiente.

### Módulos

- [`backend/modules/admin.md`](backend/modules/admin.md): operaciones administrativas, consultas, requisitos y reglas del módulo `admin`.
- [`backend/modules/auth.md`](backend/modules/auth.md): autenticación, usuarios, roles, JWT, refresh tokens y OAuth Google.
- [`backend/modules/data-portability.md`](backend/modules/data-portability.md): exportación de datos personales del estudiante y paquetes de portabilidad.
- [`backend/modules/notifications.md`](backend/modules/notifications.md): notificaciones persistentes, eventos, estados, SMTP y modo simulado.
- [`backend/modules/presentation-letters.md`](backend/modules/presentation-letters.md): cartas de presentación automáticas, plantillas, descarga y alcance vigente.
- [`backend/modules/scheduling.md`](backend/modules/scheduling.md): disponibilidad, reservas y gestión de entrevistas o presentaciones.
- [`backend/modules/self-evaluations.md`](backend/modules/self-evaluations.md): autoevaluaciones de estudiantes asociadas al cierre de práctica.
- [`backend/modules/supervisor-evaluations.md`](backend/modules/supervisor-evaluations.md): invitaciones y evaluaciones del supervisor externo.
- [`backend/modules/internships/internships-technical-reference.md`](backend/modules/internships/internships-technical-reference.md): referencia técnica principal del módulo de prácticas.
- [`backend/modules/internships/internships-business-flow.md`](backend/modules/internships/internships-business-flow.md): reglas de negocio y flujo funcional de prácticas.
- [`backend/modules/internships/internships-tracking.md`](backend/modules/internships/internships-tracking.md): trazabilidad de estados y acciones administrativas de prácticas.
- [`backend/modules/documents/documents-technical-reference.md`](backend/modules/documents/documents-technical-reference.md): referencia técnica del módulo documental.
- [`backend/modules/documents/documents-storage-privacy.md`](backend/modules/documents/documents-storage-privacy.md): almacenamiento, privacidad, retención y respaldo de documentos.

### Convenciones

- [`backend/conventions/insomnia.md`](backend/conventions/insomnia.md): convención para colecciones Insomnia de pruebas manuales de API.
- [`backend/conventions/seed-demo.md`](backend/conventions/seed-demo.md): alcance, usuarios y bordes no cubiertos del seed demo QA.
- [`backend/conventions/rbac-qa-matrix.md`](backend/conventions/rbac-qa-matrix.md): matriz de permisos críticos por rol para QA funcional.
- [`backend/conventions/development-standards.md`](backend/conventions/development-standards.md): estándares de ramas, commits, PRs, versionado y codificación backend.
- [`backend/conventions/database-naming.md`](backend/conventions/database-naming.md): convención de nombres BD/ORM para tablas, enums, modelos y llaves foráneas.

### Pruebas

- [`backend/tests/README.md`](backend/tests/README.md): estrategia, glosario, resumen de cobertura e índice de casos de prueba del backend.

### Plantillas

- [`backend/templates/module-template.md`](backend/templates/module-template.md): plantilla para documentación técnica de módulos backend.
- [`backend/templates/core-template.md`](backend/templates/core-template.md): plantilla genérica para documentación transversal de `core`.
- [`backend/templates/convention-template.md`](backend/templates/convention-template.md): plantilla para documentación de convenciones.

> [!NOTE]
> La documentación del backend describe el comportamiento real del sistema y las decisiones técnicas aplicadas en la implementación actual.

## Frontend

La carpeta `frontend/` agrupa la documentación técnica asociada al frontend del sistema.

## Deployment

La carpeta `deployment/` agrupa la documentación técnica asociada al despliegue y ejecución del sistema.

- [`deployment/modelo-despliegue.md`](deployment/modelo-despliegue.md)
  Describe el modelo vigente de despliegue en VPS, incluyendo DNS/TLS,
  contenedores, flujo CI/CD, transferencia de imágenes, seguridad operacional y
  consideraciones de operación.
