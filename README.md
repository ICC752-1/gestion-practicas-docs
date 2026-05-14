<h1 align="center"><em>Documentación del Sistema de Gestión de Prácticas</em></h1>

> [!NOTE]
> Este repositorio centraliza la documentación técnica del Sistema de Gestión de Prácticas.

El repositorio organiza la documentación del sistema según sus áreas principales: `backend`, `frontend` y `deployment`. Cada directorio agrupa documentos relacionados con una parte específica del proyecto, permitiendo mantener una estructura clara, ordenada y fácil de consultar.

## Backend

La carpeta `backend/` contiene la documentación técnica asociada al backend del sistema. En esta sección se documentan aspectos relacionados con la configuración interna, estándares de desarrollo y componentes transversales. 

* [`logging.md`](backend/logging.md)
  Describe el sistema de logging implementado en el backend. Incluye el flujo de inicialización, el enrutamiento por severidad, los destinos de salida, el formato de consola, el formato JSONL, la configuración por entorno, la rotación de archivos y los eventos registrados actualmente.

* [`development-standards.md`](backend/development-standards.md)
  Define los estándares de desarrollo del backend. Establece criterios, convenciones y prácticas utilizadas para mantener consistencia en la implementación del código.

* [`authentication.md`](backend/authentication.md)
  Describe el módulo de autenticación implementado en el backend. Incluye la estructura interna del módulo, los endpoints disponibles para autenticación y administración de usuarios y roles, los schemas utilizados para validar requests y responses, el flujo principal de inicio de sesión basado en JWT, las reglas actuales de autorización según roles y las consideraciones operativas relacionadas con almacenamiento de tokens, gestión administrativa inicial y futuras funcionalidades previstas.

> [!NOTE]
> La documentación del backend describe el comportamiento real del sistema y las decisiones técnicas aplicadas en la implementación actual.

---

## Frontend

La carpeta `frontend/` agrupa la documentación técnica asociada al frontend del sistema.

---

## Deployment

La carpeta `deployment/` agrupa la documentación técnica asociada al despliegue y ejecución del sistema.
