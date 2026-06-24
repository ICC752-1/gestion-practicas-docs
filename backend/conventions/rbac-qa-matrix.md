# Matriz RBAC para QA funcional

Esta matriz resume permisos críticos por rol para validar que los controles no
dependen solo del frontend. Cada restricción debe verificarse contra backend con
respuestas `401`, `403`, `404` o `409` según corresponda.

## Roles principales

| Rol | Alcance esperado |
| --- | --- |
| Estudiante | Gestiona sus solicitudes, documentos propios, seguimiento, autoevaluación, carta y portabilidad. |
| Encargado de practica | Revisa solicitudes, documentos, agenda, reportes administrativos y acciones operativas permitidas. |
| Director de carrera | Puede resolver solicitudes, gestionar seguro escolar por solicitud y consultar reportes de su carrera. |
| Secretaria de Carrera | Gestiona documentación administrativa y exportación local DIRAE; no decide aprobación/rechazo de solicitudes. |
| FICA | Consulta reportes agregados transversales; no accede a documentos ni datos sensibles individuales. |
| Superadmin | Administra usuarios y roles; no debe intervenir decisiones académicas u operativas de prácticas. |
| Supervisor de practica | Consulta asignaciones limitadas y responde evaluación; no ve documentos, autoevaluación ni prácticas ajenas. |

## Matriz crítica

| Capacidad | Estudiante | Encargado | Director | Secretaría | FICA | Superadmin | Supervisor |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Crear solicitud propia | Sí | No | No | No | No | No | No |
| Ver solicitud propia | Sí | Sí | Sí | Sí | No detalle sensible | No operativo | Solo asignadas |
| Aprobar/rechazar solicitud | No | Sí | Sí | No | No | No | No |
| Iniciar revisión administrativa | No | Sí | Sí | No | No | No | No |
| Gestionar seguro escolar por solicitud | No | Solo lectura | Sí | No | No | No | No |
| Cargar documento propio | Sí | No | No | Documentos administrativos | No | No | No |
| Revisar documentos | No | Sí | Sí | Sí, no sensibles | No | No | No |
| Descargar documento sensible | Propio si aplica | Sí | Sí | No | No | No | No |
| Exportar paquete DIRAE local | No | Sí | Sí | Sí | No | No | No |
| Reportes agregados | No | Sí | Sí | No | Sí | No | No |
| Administrar usuarios/roles | No | No | No | No | No | Sí | No |
| Gestionar inducción administrable | No | Sí | Sí | No | No | No operativo | No |
| Autoevaluación | Sí, propia | Consulta según flujo | Consulta según flujo | No | No | No | No |
| Evaluación supervisor | No | Genera/reenvía si corresponde | Genera/reenvía si corresponde | Consulta limitada si aplica | No | No operativo | Sí, por token o asignación |

## Pruebas negativas mínimas

| Caso | Resultado esperado |
| --- | --- |
| FICA intenta descargar documento o paquete con documentos sensibles. | `403`. |
| FICA intenta mutar solicitud, documento, agenda o usuario. | `403`. |
| Secretaría intenta aprobar/rechazar una solicitud. | `403`. |
| Secretaría intenta acceder a documentos sensibles de seguro/salud. | `403` o recurso omitido según endpoint. |
| Superadmin intenta resolver una solicitud de práctica. | `403`. |
| Estudiante intenta ver práctica o documento de otro estudiante. | `403` o `404` si se oculta existencia. |
| Supervisor intenta ver documentos, autoevaluación o práctica ajena. | `403`. |
| Usuario sin token llama endpoint protegido. | `401`. |

## Evidencia esperada

Para cierre de QA, cada rol sensible debe tener al menos:

1. una captura o request exitoso de su flujo permitido;
2. una captura o request fallido de una acción prohibida;
3. payload/respuesta backend conservada con status HTTP;
4. referencia al endpoint y usuario demo usado.

La ausencia de botón en frontend no basta como evidencia de autorización. La
validación debe incluir rechazo backend.

