# Auditoría funcional y trazabilidad

Este documento separa la trazabilidad funcional actualmente implementada de la
auditoría transversal pendiente. Su objetivo es evitar confundir historial de
negocio, logs técnicos y auditoría consultable por roles.

## Estado actual

La plataforma cuenta con trazabilidad funcional en varios dominios, pero no
posee todavía un módulo transversal completo de auditoría con catálogo único,
API de consulta, filtros por rol y visor administrativo.

## Superficies existentes

| Superficie | Persistencia | Qué registra | Uso |
| --- | --- | --- | --- |
| Historial administrativo de práctica | `internship_status_history` | Cambios de `currentstate`, actor, motivo y metadata. | Seguimiento de aprobación, rechazo, edición y anulación. |
| Historial local DIRAE | `internship_dirae_status_history` | Preparación, rectificación y exportación local del expediente. | Trazabilidad interna del paquete documental; no representa estado externo de DIRAE. |
| Exportación local DIRAE | `LogAction` mediante evento `dirae_export_generated` | Actor, prácticas exportadas, documentos aprobados, archivo y resultado local. | Auditoría de negocio de la exportación CSV. |
| Portabilidad de datos | Solicitudes de portabilidad | Estado de exportación, formato, documentos incluidos y metadata de resultado. | Trazabilidad del derecho de acceso/descarga del estudiante. |
| Refresh tokens | Tabla de refresh tokens | Hash, usuario, expiración y revocación de sesión. | Seguridad de sesión, refresh y logout. |
| Notificaciones | `notification` | Evento, destinatario, estado de envío, payload, `sent_at` y `read_at`. | Trazabilidad comunicacional, no auditoría de decisión. |
| Logs técnicos | Logging de aplicación | Errores, advertencias y eventos operativos. | Diagnóstico técnico; no reemplaza auditoría funcional persistente. |

## Qué no está cubierto todavía

La auditoría transversal pendiente debe resolver:

- Catálogo único de eventos de auditoría.
- API de consulta con filtros por actor, entidad, evento, fecha y resultado.
- Autorización para consultar auditoría según rol.
- Visor administrativo o reporte exportable.
- Cobertura uniforme de usuarios/roles, prácticas, documentos sensibles,
  inducción, agenda, autoevaluación, evaluación del supervisor, carta,
  portabilidad y notificaciones críticas.
- Política de retención y minimización de datos para eventos de auditoría.

## Eventos mínimos recomendados

| Dominio | Eventos mínimos |
| --- | --- |
| Auth/usuarios | creación, activación, cambio de contraseña, desactivación, asignación/remoción de rol, revocación de sesión. |
| Prácticas | creación, edición, anulación, inicio de revisión, aprobación de solicitud, rechazo, cierre final. |
| Seguro escolar | validación, excepción autorizada, rechazo/observación. |
| Documentos | carga, descarga sensible, revisión, observación, aprobación, eliminación lógica. |
| DIRAE local | preparación, rectificación, exportación CSV, intento no exportable. |
| Agenda | creación de disponibilidad, reserva, reprogramación, cancelación, asistencia, resultado. |
| Evaluaciones | envío de autoevaluación, generación/reenvío de invitación supervisor, envío evaluación supervisor, reapertura. |
| Carta de presentación | edición de plantilla, generación, descarga. |
| Inducción | creación de borrador, publicación, descarte, versión con repetición obligatoria. |
| Portabilidad | solicitud, generación correcta, fallo y descarga. |

## Regla PO

Para cierre Sprint 11, las superficies existentes permiten trazabilidad funcional
de los flujos principales, pero **no cierran la tarea de auditoría transversal**.
La deuda debe pasar a Sprint 12 como corrección/fortalecimiento técnico con
criterio de cierre verificable:

1. catálogo de eventos aprobado;
2. persistencia uniforme;
3. API/consulta autorizada;
4. evidencia de eventos críticos generados por acciones reales;
5. documentación de retención y minimización.

## Relación con DIRAE

La exportación DIRAE registra un evento local `dirae_export_generated`. Ese
evento confirma que la plataforma generó el archivo localmente; no confirma
recepción, procesamiento ni aprobación por DIRAE, porque ese proceso ocurre fuera
de la plataforma.

