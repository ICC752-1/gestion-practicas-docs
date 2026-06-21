<h1 align="center"><em>Internships Tracking</em></h1>

> [!NOTE]
> Este documento complementa `docs/modules/internships/internships-technical-reference.md`. No describe un módulo
> independiente. Explica la funcionalidad de tracking implementada actualmente
> dentro del módulo `internships`.

## Contenidos
- [Resumen operativo](#resumen-operativo)
- [Estado actual](#estado-actual)
- [Ámbito y responsabilidades](#ámbito-y-responsabilidades)
- [Estructura interna](#estructura-interna)
- [Funcionalidad principal](#funcionalidad-principal)
- [Endpoint disponible](#endpoint-disponible)
- [Contrato principal](#contrato-principal)
- [Almacenamiento y trazabilidad](#almacenamiento-y-trazabilidad)
- [Reglas de negocio](#reglas-de-negocio)
- [Consideraciones operativas](#consideraciones-operativas)
- [Documentación relacionada](#documentación-relacionada)

## Resumen operativo

El tracking disponible en el backend tiene dos vistas complementarias:

- **Historial cronológico administrativo:** muestra cómo ha cambiado el estado
  funcional de la solicitud y qué actor ejecutó o disparó cada cambio.
- **Seguimiento agregado de ciclo de vida:** resume solicitud, ejecución,
  autoevaluación, evaluación del supervisor, presentación final y cierre.

**Permite:**

- Consultar transiciones de estado de una práctica.
- Ver estado anterior y estado nuevo.
- Ver el actor asociado a la transición, cuando existe.
- Ver el motivo funcional registrado.
- Ver metadata auxiliar de la acción administrativa.
- Reconstruir la secuencia funcional de revisión de una práctica.
- Consultar avance funcional agregado para dashboards de estudiante y
  administración.

> [!IMPORTANT]
> Este tracking es trazabilidad funcional del flujo de prácticas. No reemplaza la
> auditoría técnica general del sistema ni registra toda actividad de usuario.

## Estado actual

Existe una carpeta `app/modules/tracking/`, pero actualmente contiene solo archivos
`__init__.py` con docstrings placeholder. No hay controller, service, repository,
schemas, modelos, tests propios ni router registrado para `tracking`.

La implementación real vive en `internships`:

- Endpoint: `GET /internships/{internship_id}/tracking`.
- Endpoint: `GET /internships/{internship_id}/lifecycle-tracking`.
- Modelo: `InternshipStatusHistory`.
- Schemas de respuesta: `InternshipTrackingResponse`,
  `InternshipLifecycleResponse`.
- Persistencia del historial administrativo: tabla
  `internship_status_history`. El seguimiento agregado se calcula a partir de
  varias fuentes del dominio y no persiste una tabla propia.

> [!WARNING]
> No existe endpoint `/tracking`. Cualquier cambio que convierta `tracking` en un
> módulo real debe mover o duplicar explícitamente controller, service, repository,
> schemas, modelos y tests.

## Ámbito y responsabilidades

#### Responsabilidades actuales

- Exponer el historial de estados de una práctica.
- Exponer un seguimiento agregado del ciclo real de práctica.
- Mantener orden cronológico de las transiciones.
- Mostrar información básica del actor de la transición.
- Mostrar motivo funcional y metadata asociada.
- Respetar los mismos permisos de lectura que el detalle de la práctica.

#### Fuera de alcance

- Métricas o analítica avanzada del proceso.
- Seguimiento documental del módulo `documents`.
- Seguimiento de notificaciones.
- Auditoría técnica de requests, sesiones o errores.
- Registro general de actividad de usuario.
- Endpoints independientes bajo `/tracking`.

## Estructura interna

| Capa | Archivo | Responsabilidad |
| --- | --- | --- |
| Placeholder | `app/modules/tracking/.../__init__.py` | Declara estructura futura sin implementación funcional actual. |
| Controller real | `app/modules/internships/controllers/internship_controller.py` | Expone `GET /internships/{internship_id}/tracking` y `GET /internships/{internship_id}/lifecycle-tracking`; valida permisos de lectura. |
| Service real | `app/modules/internships/services/internship_service.py` | Delega historial mediante `list_internship_tracking` y calcula ciclo de vida mediante `get_lifecycle_tracking`. |
| Repository real | `app/modules/internships/repositories/internship_repository.py` | Consulta `InternshipStatusHistory`, autoevaluación, evaluación supervisor, invitaciones y presentaciones. |
| Model real | `app/modules/internships/models/internship_status_history_model.py` | Persiste cada transición funcional de estado. |
| Schema real | `app/modules/internships/schemas/internship_schema.py` | Define `InternshipTrackingResponse`, `InternshipLifecycleResponse` y schemas anidados. |

## Funcionalidad principal

#### Consulta de tracking de una práctica

1. El usuario llama a `GET /internships/{internship_id}/tracking`.
2. El controller valida Bearer token.
3. Se busca la práctica por `internship_id`.
4. Si la práctica no existe, se responde `404`.
5. Se valida que el usuario sea propietario o tenga rol privilegiado de lectura.
6. Si no tiene permisos, se responde `403`.
7. El service solicita el historial al repository.
8. El repository lista transiciones ordenadas por `changed_at` e `id`.
9. Se retorna una lista de `InternshipTrackingResponse`.

#### Consulta de seguimiento de ciclo de vida

1. El usuario llama a `GET /internships/{internship_id}/lifecycle-tracking`.
2. El controller valida Bearer token.
3. Se busca la práctica por `internship_id`.
4. Si la práctica no existe, se responde `404`.
5. Se valida que el usuario sea propietario o tenga rol privilegiado de lectura.
6. Si no tiene permisos, se responde `403`.
7. El service combina historial administrativo, autoevaluación, invitaciones y
   evaluación del supervisor, presentaciones finales y campos de cierre.
8. Se retorna `InternshipLifecycleResponse`.

## Endpoint disponible

| Método | Ruta | Propósito | Acceso |
| --- | --- | --- | --- |
| GET | `/internships/{internship_id}/tracking` | Lista el historial cronológico de estados de una práctica. | Propietario o rol privilegiado |
| GET | `/internships/{internship_id}/lifecycle-tracking` | Resume el avance completo de solicitud, ejecución, evaluaciones, presentación y cierre. | Propietario o rol privilegiado |

Roles privilegiados de lectura:

| Rol |
| --- |
| `Encargado de practica` |
| `Director de carrera` |
| `Secretaria de Carrera` |

## Contrato principal

<details>
<summary><strong>InternshipTrackingResponse</strong></summary>

Representa una entrada del historial funcional de una práctica.

```json
{
  "id": 12,
  "internship_id": 7,
  "previous_status": {
    "id": 1,
    "title": "Pendiente",
    "description": "Solicitud registrada."
  },
  "new_status": {
    "id": 2,
    "title": "En revisión",
    "description": "Solicitud en revisión administrativa."
  },
  "actor": {
    "id": 99,
    "email": "encargado@ufro.cl",
    "first_name": "María",
    "last_name": "Rojas"
  },
  "reason": "Documentación inicial revisada.",
  "changed_at": "2026-06-16T12:30:00Z",
  "metadata": {
    "action": "approve",
    "skip_review": false
  }
}
```

</details>

<details>
<summary><strong>InternshipLifecycleResponse</strong></summary>

Representa el avance funcional agregado. No reemplaza el historial
administrativo porque sus eventos pueden ser calculados desde distintas fuentes
del dominio.

```json
{
  "internship_id": 7,
  "progress_percentage": 70,
  "current_step": "Evaluación del supervisor completada",
  "self_evaluation_submitted": true,
  "supervisor_invitation_sent": true,
  "supervisor_evaluation_submitted": false,
  "final_presentation_scheduled": false,
  "final_presentation_completed": false,
  "can_generate_supervisor_invitation": true,
  "can_close_practice": false,
  "events": [
    {
      "id": "self_evaluation_submitted",
      "type": "self_evaluation_submitted",
      "title": "Autoevaluación enviada",
      "description": "El estudiante completó su autoevaluación.",
      "status": "completed",
      "occurred_at": "2026-06-20T15:00:00Z",
      "metadata": {}
    }
  ]
}
```

</details>

## Almacenamiento y trazabilidad

El tracking se persiste en la tabla `internship_status_history`. Cada fila
representa una transición o evento funcional asociado a una práctica.

| Campo | Uso |
| --- | --- |
| `internship_id` | Práctica asociada al evento. |
| `previous_status_id` | Estado anterior. Puede ser `NULL` en la creación inicial. |
| `new_status_id` | Estado asignado después del evento. |
| `actor_id` | Usuario que ejecutó o disparó la transición. Puede ser `NULL`. |
| `reason` | Motivo funcional, comentario o justificación. |
| `changed_at` | Fecha y hora de registro del evento. |
| `metadata` | Datos auxiliares en JSON para identificar la acción o contexto. |

Relaciones principales:

| Relación | Modelo |
| --- | --- |
| Práctica | `Internship`. |
| Estado anterior | `CurrentState`. |
| Estado nuevo | `CurrentState`. |
| Actor | `User`. |

> [!NOTE]
> En el modelo ORM el campo se llama `metadata_json`, pero en la respuesta HTTP se
> expone como `metadata`.

## Reglas de negocio

#### Creación del historial

- La creación de una práctica registra una entrada inicial.
- La entrada inicial usa `previous_status=null` y `new_status=Pendiente`.
- Las transiciones administrativas registran estado anterior y estado nuevo.
- La edición administrativa registra historial manteniendo el mismo estado.
- La anulación lógica registra historial manteniendo el mismo estado.

#### Lectura del historial

- El endpoint solo consulta historial. No crea ni modifica entradas.
- El usuario debe ser propietario de la práctica o tener rol privilegiado.
- El historial se retorna ordenado por `changed_at` ascendente y luego `id` ascendente.
- Si no hay actor asociado, `actor` puede venir como `null`.
- Si no hay estado anterior, `previous_status` puede venir como `null`.

#### Lectura del ciclo de vida

- El endpoint de ciclo de vida no modifica datos.
- Los eventos usan `completed`, `current`, `pending` o `blocked`.
- La invitación del supervisor solo queda habilitada cuando la solicitud está
  aprobada y la autoevaluación del estudiante fue enviada.
- El cierre final solo debe habilitarse si la solicitud está aprobada, la
  autoevaluación fue enviada, la evaluación del supervisor fue completada y la
  presentación final fue completada.

#### Metadata conocida

| Acción | Metadata esperada |
| --- | --- |
| Creación | `{"event": "internship_created"}`. |
| Inicio de revisión por apertura de detalle | `{"action": "start_review"}`. |
| Aprobación | `{"action": "approve", "skip_review": false}`. |
| Rechazo | `{"action": "reject"}`. |
| Derivación | `{"action": "derive"}`. |
| Edición administrativa | `{"action": "admin_update", "changed_fields": [...]}`. |
| Anulación lógica | `{"action": "cancel"}`. |

> [!IMPORTANT]
> La metadata es auxiliar. La interpretación principal del historial debe basarse
> en estados, actor, motivo y fecha. Nuevas acciones pueden agregar nuevas claves.

## Consideraciones operativas

- No documentar ni consumir `/tracking`, porque no existe en el backend actual.
- No asumir que `app/modules/tracking/` está activo solo porque la carpeta existe.
- La trazabilidad actual está acoplada al ciclo de vida de `internships`.
- Si se implementa `tracking` como módulo real, este documento debe actualizarse o moverse.
- El historial funcional no reemplaza logs ni auditoría técnica de infraestructura.
- Las consultas de tracking dependen de que las transiciones usen métodos con historial.

## Documentación relacionada

- `docs/modules/internships/internships-technical-reference.md`: Describe el flujo completo de prácticas y las acciones que generan historial.
- `app/modules/internships/models/internship_status_history_model.py`: Define el modelo ORM de historial.
- `app/modules/internships/controllers/internship_controller.py`: Expone el endpoint de consulta de tracking.
