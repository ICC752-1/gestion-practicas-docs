<h1 align="center"><em>Scheduling</em></h1>

> [!NOTE]
> Esta documentación técnica describe el comportamiento actual del módulo
> `scheduling` desde una perspectiva funcional e interna. Su objetivo es explicar
> qué hace el módulo, cómo se conecta con el resto del backend y qué debe saber
> alguien antes de modificarlo. El contrato HTTP formal queda en OpenAPI y el
> detalle interactivo queda en Swagger.

## Contenidos
- [Resumen operativo](#resumen-operativo)
- [Ámbito y responsabilidades](#ámbito-y-responsabilidades)
- [Estructura interna](#estructura-interna)
- [Funcionalidades](#funcionalidades)
- [Endpoints disponibles](#endpoints-disponibles)
- [Contratos principales](#contratos-principales)
- [Reglas de negocio](#reglas-de-negocio)
- [Consideraciones operativas](#consideraciones-operativas)

## Resumen operativo

El módulo **`scheduling`** gestiona la agenda de entrevistas y presentaciones
asociadas al proceso de práctica. Su función principal es permitir que los roles
administrativos publiquen horarios, que estudiantes reserven o soliciten citas y
que luego se registre el resultado de esas citas.

**Permite:**

- Publicar bloques de disponibilidad para entrevistas o presentaciones.
- Listar horarios disponibles y citas ya agendadas.
- Reservar, cancelar o reprogramar citas.
- Crear solicitudes de agendamiento cuando se requiere coordinación manual.
- Registrar asistencia, resultado y observaciones de una cita.
- Confirmar asistencia del estudiante a una cita agendada.
- Asociar documentos de diapositivas a una presentación.

En términos simples, este módulo ordena el calendario del proceso de práctica y
conecta la agenda con el avance real de la solicitud.

## Ámbito y responsabilidades

El módulo **`scheduling`** centraliza las operaciones de agenda. No crea prácticas
ni define por sí mismo si una práctica está aprobada, pero sí puede actualizar
algunos hitos del proceso cuando se registra el resultado de una entrevista o
presentación.

#### Responsabilidades principales

- Administrar disponibilidad futura publicada por roles autorizados.
- Reservar citas para prácticas válidas.
- Gestionar solicitudes de agendamiento creadas por estudiantes.
- Registrar resultados de entrevistas iniciales y presentaciones finales.
- Mantener configuración simple de disponibilidad para consultas generales.
- Emitir notificaciones cuando ocurre un cambio relevante de agenda.

#### Fuera de alcance

- Crear, aprobar o rechazar solicitudes de práctica.
- Almacenar archivos documentales físicos.
- Gestionar usuarios, roles o sesiones.
- Reemplazar el seguimiento completo de la práctica.

> [!IMPORTANT]
> `scheduling` trabaja sobre prácticas existentes. Si una cita cambia el avance de
> una práctica, esa actualización se realiza respetando las reglas del flujo de
> `internships`.

## Estructura interna

| Capa | Archivo | Responsabilidad |
| --- | --- | --- |
| Controller | `app/modules/scheduling/controllers/scheduling_controller.py` | Define las rutas HTTP de disponibilidad, citas, solicitudes y configuración. |
| Service | `app/modules/scheduling/services/scheduling_service.py` | Orquesta permisos, reglas de agenda, cambios de estado y notificaciones. |
| Repository | `app/modules/scheduling/repositories/scheduling_repository.py` | Encapsula consultas y persistencia de disponibilidad, citas y solicitudes. |
| Models | `app/modules/scheduling/models/presentation_model.py` | Define las citas o bloques de presentación/entrevista. |
| Models | `app/modules/scheduling/models/scheduling_request_model.py` | Define solicitudes de agendamiento manual. |
| Models | `app/modules/scheduling/models/scheduling_config_model.py` | Define configuración de agenda para consultas generales. |
| Schemas | `app/modules/scheduling/schemas/scheduling_schema.py` | Define contratos de entrada y salida del módulo. |

El módulo reutiliza autenticación desde `auth`, datos de prácticas desde
`internships`, documentos desde `documents` cuando se asocian diapositivas y
notificaciones mediante `notifications`.

## Funcionalidades

#### Publicación de disponibilidad

1. Un rol administrativo crea bloques de horario con `POST /scheduling/availability`.
2. El backend valida que el actor pueda publicar disponibilidad.
3. Se revisa que los horarios sean futuros y no generen solapamientos inválidos.
4. Se guardan los bloques disponibles para que puedan ser consultados o reservados.

#### Reserva de cita

1. El estudiante consulta horarios con `GET /scheduling/slots`.
2. Selecciona un bloque y llama a `POST /scheduling/slots/{slot_id}/reserve`.
3. El backend valida que la práctica exista, sea válida para agendar y pertenezca
   al estudiante.
4. El bloque queda reservado y asociado a la práctica.

#### Solicitudes de agendamiento

1. El estudiante crea una solicitud con `POST /scheduling/requests`.
2. Un rol administrativo revisa solicitudes pendientes con `GET /scheduling/requests`.
3. La solicitud puede responderse, rechazarse o cancelarse según el caso.
4. Si se responde favorablemente, se crea o asigna una cita.

#### Cambios de cita

1. Una cita puede cancelarse con
   `POST /scheduling/appointments/{appointment_id}/cancel`.
2. El estudiante puede reprogramar una cita propia hacia otro bloque compatible.
3. La administración puede cerrar o eliminar disponibilidad futura cuando corresponde.
4. Cada cambio mantiene trazabilidad suficiente para entender qué ocurrió.

#### Resultado de entrevista o presentación

1. Un rol autorizado registra el resultado con
   `PATCH /scheduling/appointments/{appointment_id}/outcome`.
2. El backend guarda asistencia, resultado y observaciones.
3. Si la cita corresponde a un hito importante, se sincroniza el avance de la
   práctica.
4. Se emiten notificaciones cuando el cambio debe ser informado.

#### Confirmación y documentos

1. El estudiante confirma asistencia con
   `PATCH /scheduling/appointments/{appointment_id}/confirm`.
2. Si la cita corresponde a una presentación, puede asociarse un documento de
   diapositivas.
3. El documento sigue perteneciendo al módulo `documents`; `scheduling` solo
   guarda la relación con la cita.

## Endpoints disponibles

**Todos los endpoints requieren autenticación.**

| Método | Ruta | Propósito | Acceso principal |
| --- | --- | --- | --- |
| POST | `/scheduling/availability` | Publica bloques de disponibilidad. | Roles administrativos |
| PUT | `/scheduling/availability/{slot_id}` | Edita un bloque futuro. | Dueño administrativo del bloque |
| DELETE | `/scheduling/availability/{slot_id}` | Elimina disponibilidad futura. | Dueño administrativo del bloque |
| POST | `/scheduling/availability/{slot_id}/close` | Cierra un bloque disponible. | Roles administrativos |
| GET | `/scheduling/slots` | Lista bloques disponibles. | Usuario autenticado |
| POST | `/scheduling/slots/{slot_id}/reserve` | Reserva un bloque para una práctica. | Estudiante |
| GET | `/scheduling/appointments` | Lista citas visibles para el usuario. | Usuario autenticado |
| POST | `/scheduling/appointments/{appointment_id}/cancel` | Cancela una cita. | Participante o rol autorizado |
| POST | `/scheduling/appointments/{appointment_id}/reschedule` | Reprograma una cita. | Estudiante propietario |
| PATCH | `/scheduling/appointments/{appointment_id}/outcome` | Registra asistencia y resultado. | Roles administrativos |
| POST | `/scheduling/appointments/direct` | Agenda directamente una presentación final. | Roles administrativos |
| PATCH | `/scheduling/appointments/{appointment_id}/confirm` | Confirma asistencia del estudiante. | Estudiante |
| PATCH | `/scheduling/appointments/{appointment_id}/document` | Asocia documento de diapositivas. | Usuario autorizado |
| POST | `/scheduling/requests` | Crea una solicitud de agendamiento. | Estudiante |
| GET | `/scheduling/requests/me` | Lista solicitudes propias. | Estudiante |
| GET | `/scheduling/requests` | Lista solicitudes pendientes. | Roles administrativos |
| POST | `/scheduling/requests/{id}/respond` | Responde una solicitud con fecha u horario. | Roles administrativos |
| POST | `/scheduling/requests/{id}/reject` | Rechaza una solicitud. | Roles administrativos |
| POST | `/scheduling/requests/{id}/cancel` | Cancela una solicitud propia. | Estudiante |
| GET | `/scheduling/config` | Consulta configuración de agenda. | Usuario autenticado |
| PATCH | `/scheduling/config` | Actualiza configuración de consultas generales. | Roles administrativos |

## Contratos principales

<details>
<summary><strong>PresentationSlotResponse</strong></summary>

Representa un bloque de agenda. Puede estar disponible, reservado, cancelado o
cerrado según el flujo.

```json
{
  "id": 10,
  "purpose": "final_presentation",
  "status": "reserved",
  "start_at": "2026-06-24T14:00:00",
  "end_at": "2026-06-24T14:30:00",
  "internship_id": 25
}
```

</details>

<details>
<summary><strong>SchedulingRequestResponse</strong></summary>

Representa una solicitud creada por un estudiante cuando necesita que la agenda
sea coordinada manualmente.

```json
{
  "id": 4,
  "internship_id": 25,
  "purpose": "general_consultation",
  "status": "pending",
  "requested_by_user_id": 8
}
```

</details>

<details>
<summary><strong>AppointmentOutcomeRequest</strong></summary>

Payload usado para registrar qué ocurrió en una cita.

```json
{
  "attended": true,
  "result": "passed",
  "notes": "Presentación realizada correctamente."
}
```

</details>

## Reglas de negocio

#### Disponibilidad

**Reglas actuales:**

- Solo roles autorizados pueden publicar, editar, cerrar o eliminar
  disponibilidad.
- Los bloques deben representar horarios futuros.
- No se permiten solapamientos inválidos para el mismo responsable.
- Una disponibilidad ya reservada no debe eliminarse como si nunca hubiese
  existido.

#### Reservas y reprogramación

**Reglas actuales:**

- Una cita debe asociarse a una práctica válida cuando corresponde.
- Un estudiante no puede reservar sobre una práctica que no le pertenece.
- La reprogramación debe mantener coherencia con el propósito de la cita.
- Las cancelaciones administrativas requieren motivo cuando la regla del flujo lo
  exige.

#### Resultados y cierre

**Reglas actuales:**

- Registrar un resultado puede afectar el avance de la práctica.
- Una entrevista inicial completada puede marcar la práctica como en ejecución.
- Una presentación final puede participar en el cierre académico de la práctica.
- Si faltan prerrequisitos del cierre, el backend debe rechazar el resultado final.

> [!WARNING]
> No se debe usar `scheduling` para forzar estados de práctica sin pasar por las
> reglas existentes del backend. La agenda coordina hitos; no reemplaza el flujo
> completo de `internships`.

## Consideraciones operativas

- Las fechas y horas se trabajan con zona horaria local del sistema definida para
  el proyecto.
- El módulo depende de datos existentes de usuarios, roles y prácticas.
- Las notificaciones son efectos secundarios: informan cambios relevantes, pero
  no deben bloquear la explicación funcional del flujo.
- Los contratos exactos de campos, validaciones y errores deben consultarse en
  Swagger/OpenAPI.
- Las pruebas unitarias documentadas están en
  `backend/tests/modules/scheduling-unitarias.md`.
