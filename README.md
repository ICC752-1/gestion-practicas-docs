<h1 align="center"><em>Documentación del Sistema de Gestión de Prácticas</em></h1>

> [!NOTE]
> Este repositorio centraliza la documentación técnica del Sistema de Gestión de Prácticas.

El repositorio organiza la documentación del sistema según sus áreas principales: `backend`, `frontend` y `deployment`. Cada directorio agrupa documentos relacionados con una parte específica del proyecto, permitiendo mantener una estructura clara, ordenada y fácil de consultar.

## Backend

La carpeta `backend/` contiene la documentación técnica asociada al backend del sistema. En esta sección se documentan aspectos relacionados con la configuración interna y componentes transversales. 

* [`logging.md`](backend/logging.md)
  Describe el sistema de logging implementado en el backend. Incluye el flujo de inicialización, el enrutamiento por severidad, los destinos de salida, el formato de consola, el formato JSONL, la configuración por entorno, la rotación de archivos y los eventos registrados actualmente.

* [`authentication.md`](backend/authentication.md)
  Describe el módulo de autenticación implementado en el backend. Incluye la estructura interna del módulo, los endpoints disponibles para autenticación y administración de usuarios y roles, los schemas utilizados para validar requests y responses, el flujo principal de inicio de sesión basado en JWT, las reglas actuales de autorización según roles y las consideraciones operativas relacionadas con almacenamiento de tokens, gestión administrativa inicial y futuras funcionalidades previstas.

* [`api-contracts.md`](backend/api-contracts.md)
  Centraliza los contratos HTTP activos del backend para autenticación, usuarios, roles, prácticas, dashboard coordinador y tracking de estados.

* [`admin.md`](backend/admin.md)
  Describe los endpoints administrativos, contratos principales y reglas de negocio del módulo `admin`.

* [`business_rules.md`](backend/business_rules.md)
  Documenta reglas de negocio vigentes, incluyendo la obligatoriedad del seguro escolar según período de práctica.

> [!NOTE]
> La documentación del backend describe el comportamiento real del sistema y las decisiones técnicas aplicadas en la implementación actual.

---

## Frontend

La carpeta `frontend/` agrupa la documentación técnica asociada al frontend del sistema.

---

## Deployment

La carpeta `deployment/` agrupa la documentación técnica asociada al despliegue y ejecución del sistema.

* [`modelo-despliegue.md`](deployment/modelo-despliegue.md)
  Describe el modelo vigente de despliegue en VPS, incluyendo DNS/TLS,
  contenedores, flujo CI/CD, transferencia de imágenes, seguridad operacional y
  consideraciones de operación.

## Convenciones

- En el archivo [convencion-insomnia.md](convencion-insomnia.md) se encuentra definida la convención de Insomnia para pruebas de API en tiempo real.
- En el archivo [development-standards.md](development-standards.md) se encuentra definida la convención de desarrollo para el proyecto.
