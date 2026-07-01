<h1 align="center"><em>Internships Technical Reference</em></h1>

> [!NOTE]
> Esta documentación técnica describe el comportamiento actual del módulo
> `internships` desde una perspectiva funcional e interna. Su objetivo es
> explicar cómo está implementado y qué debe saber alguien antes de modificarlo.
> El contrato HTTP formal queda en OpenAPI y el detalle interactivo queda en
> Swagger.

## Contenidos
- [Resumen operativo](#resumen-operativo)
- [Ámbito y responsabilidades](#ámbito-y-responsabilidades)
- [Estructura interna](#estructura-interna)
- [Funcionalidades](#funcionalidades)
- [Endpoints disponibles](#endpoints-disponibles)
- [Contratos principales](#contratos-principales)
- [Reglas de negocio](#reglas-de-negocio)
- [Configuración por entorno](#configuración-por-entorno)
- [Consideraciones operativas](#consideraciones-operativas)

## Resumen operativo

El módulo **`internships`** es el núcleo funcional del proceso de prácticas.
Permite registrar solicitudes, consultar prácticas, revisar su avance, ejecutar
acciones administrativas y aplicar reglas institucionales antes de formalizar una
aprobación.

**Permite:**

- Registrar solicitudes de práctica asociadas al estudiante autenticado.
- Listar prácticas del estudiante y prácticas visibles para dashboard.
- Consultar detalle, historial administrativo, seguimiento de ciclo de vida y
  excepciones de una práctica.
- Ejecutar acciones administrativas de aprobación, rechazo, derivación DIRAE y reapertura documental.
- Iniciar revisión automáticamente al abrir el detalle de una solicitud
  pendiente.
- Registrar excepciones administrativas sobre reglas de negocio específicas.
- Corregir campos editables con trazabilidad administrativa.
- Anular lógicamente una práctica con motivo obligatorio.
- Exponer contenido e intentos de inducción obligatoria.
- Administrar versiones de inducción para actores académicos.
- Diagnosticar elegibilidad antes de continuar con el trámite.

## Ámbito y responsabilidades

El módulo **`internships`** concentra el ciclo de vida administrativo de una
solicitud de práctica. También contiene la inducción obligatoria porque su
resultado se usa como prerrequisito del flujo de aprobación.

#### Responsabilidades principales

- Registro de solicitudes de práctica.
- Consulta de prácticas propias y administrativas.
- Normalización de estados para dashboard.
- Gestión de transiciones de estado con historial.
- Construcción del seguimiento agregado de solicitud, ejecución, evaluación y
  cierre.
- Validación de reglas de aprobación.
- Registro y consulta de excepciones administrativas.
- Edición administrativa acotada.
- Anulación lógica de prácticas.
- Evaluación de inducción obligatoria.
- Gestión administrativa de versiones de inducción.
- Sincronización de requisitos académicos al aprobar.
- Emisión de eventos de notificación asociados al flujo.

#### Fuera de alcance

- Autenticación, emisión de tokens y administración de roles.
- Gestión administrativa general de estudiantes.
- Carga, validación y almacenamiento de documentos.
- Envío final de correos o simulación SMTP.
- Modelamiento completo de malla curricular y co-requisitos.

> [!IMPORTANT]
> La creación de una práctica no significa aprobación ni formalización. La
> solicitud queda en `Pendiente` y las reglas críticas se verifican cuando un
> actor administrativo intenta avanzar o aprobar el trámite.

## Estructura interna

| Capa | Archivo | Responsabilidad |
| --- | --- | --- |
| Controller | `app/modules/internships/controllers/internship_controller.py` | Define rutas HTTP, dependencias, roles y permisos de lectura. |
| Controller | `app/modules/internships/controllers/induction_admin_controller.py` | Define CRUD administrativo y publicación de versiones de inducción. |
| Service | `app/modules/internships/services/internship_service.py` | Orquesta casos de uso, reglas de negocio, transiciones y notificaciones. |
| Service | `app/modules/internships/services/induction_admin_service.py` | Orquesta borradores, edición, publicación y eliminación de versiones. |
| Repository | `app/modules/internships/repositories/internship_repository.py` | Encapsula persistencia y consultas de prácticas, estados, inducción y excepciones. |
| Schemas | `app/modules/internships/schemas/internship_schema.py` | Define contratos HTTP de solicitudes, respuestas, dashboard, acciones, excepciones e inducción. |
| Schemas | `app/modules/internships/schemas/induction_admin_schema.py` | Define contratos administrativos de versiones, preguntas y publicación. |
| Models | `app/modules/internships/models/internship_model.py` | Define `Internship`, periodos, tipos de práctica y relaciones principales. |
| Models | `app/modules/internships/models/current_state_model.py` | Define estados funcionales de práctica. |
| Models | `app/modules/internships/models/internship_status_history_model.py` | Registra historial de cambios de estado y acciones administrativas. |
| Models | `app/modules/internships/models/internship_exception_model.py` | Registra excepciones administrativas por práctica y regla. |
| Models | `app/modules/internships/models/induction_model.py` | Define contenido versionado, videos, preguntas e intentos de inducción. |
| Models | `app/modules/internships/models/student_internship_requirement_model.py` | Define requisitos académicos e institucionales usados por el flujo. |

El módulo reutiliza autenticación y roles desde `auth`, configuración global desde
`app/core/config.py` y notificaciones mediante `NotificationService`.

## Funcionalidades

#### Registro de solicitud de práctica

1. El estudiante llama a `POST /internships`.
2. El controller exige rol `Estudiante`.
3. El service obtiene el estado `Pendiente` desde `currentstate`.
4. El backend calcula `has_school_insurance` desde `StudentRegistrationRequirement`.
5. Se valida que el estudiante tenga inducción aprobada para poder crear la solicitud.
6. Se valida que no exista otra solicitud bloqueante del mismo `internship_type`.
7. Se persiste la práctica asociada al usuario autenticado con
   `blocks_new_registration=true`, `completion_status=not_started` y
   `final_result=pending`.
8. Se registra una entrada inicial en `InternshipStatusHistory`.
9. Se intenta notificar a `Encargado de practica` y `Director de carrera`.
10. Se retorna `InternshipResponse`.

#### Dashboard de revisión

1. Un actor autorizado llama a `GET /internships` o `GET /internships/stats`.
2. El controller exige `Encargado de practica`, `Director de carrera` o `Secretaria de Carrera`.
3. El service carga prácticas con estudiante y estado actual.
4. Los estados se normalizan a `submitted`, `in_review`, `approved` o `rejected`.
5. Se retorna una lista resumida o conteos agregados.

#### Consulta propia o privilegiada

1. El usuario llama a `GET /internships/me`, `GET /internships/{internship_id}`,
   `GET /internships/{internship_id}/tracking` o
   `GET /internships/{internship_id}/lifecycle-tracking`.
2. El controller valida Bearer token.
3. Para detalle, tracking y ciclo de vida se verifica que el usuario sea
   propietario o tenga rol privilegiado.
4. Se retorna la práctica, su historial cronológico o su avance agregado.

#### Inducción obligatoria

1. Cualquier usuario autenticado llama a `GET /internships/induction`.
2. El service retorna la versión activa y publicada con videos y preguntas.
3. El estudiante llama a `POST /internships/induction/attempts` con sus respuestas.
4. El backend compara respuestas contra la versión activa.
5. Se calcula `score` y `passed` según `min_score`.
6. Se persiste el intento y se retorna el resultado.

#### Administración de inducción

1. Un actor con rol `Encargado de practica` o `Director de carrera` llama a
   `/induction/admin/versions`.
2. Puede listar versiones existentes, crear un borrador, consultar detalle,
   editar, publicar y eliminar versiones.
3. La publicación deja una versión como activa y publicada.
4. El detalle administrativo incluye información suficiente para editar preguntas,
   videos, puntaje mínimo y respuesta correcta.

#### Diagnóstico de elegibilidad

> [!NOTE]
> Esta funcionalidad funciona como una revisión previa del contexto del estudiante.
> Sirve para que el frontend pueda mostrar advertencias y próximos pasos antes de
> que la práctica avance administrativamente. No crea, aprueba ni rechaza una
> práctica. Solo responde si, según los datos actuales, hay requisitos pendientes
> que podrían bloquear una aprobación posterior.

1. El usuario llama a `GET /internships/registration-eligibility`.
2. Opcionalmente envía `internship_period` e `internship_type`.
3. El service evalúa seguro escolar, inducción, práctica previa aprobada,
   excepciones existentes y solicitudes bloqueantes del mismo tipo.
4. Se retorna un diagnóstico con `blocked`, `can_create_request` y `next_step`.

> [!IMPORTANT]
> La elegibilidad guía al frontend y al estudiante, pero el backend mantiene la
> validación autoritativa en `POST /internships`. Si `has_induction=false` o
> `can_create_request=false`, el frontend debe impedir el acceso al formulario o
> el envío de la solicitud, y el backend responderá error si se intenta forzar el
> flujo por API.

#### Aprobación administrativa

1. Un actor autorizado llama a `POST /internships/{internship_id}/approve`.
2. El service valida permiso `approve` según roles internos.
3. Se rechaza si la práctica no existe o está en estado terminal.
4. Se validan reglas de inducción, secuencialidad, tesis y ramo paralelo.
5. Se determina el estado destino según actor y estado actual.
6. Si la transición termina en `Aprobada`, se valida seguro escolar según las
   fechas de la práctica: marzo-junio y agosto-noviembre se auto-validan;
   fuera de esos rangos se exige validación explícita de Dirección o excepción
   `school_insurance`.
7. Se actualiza el estado y se registra historial.
8. Si queda `Aprobada`, se sincroniza `StudentInternshipRequirement`.
9. Se intenta notificar al estudiante.

#### Inicio automático de revisión

1. El frontend de coordinación o dirección puede llamar a
   `POST /internships/{internship_id}/start-review` al abrir el detalle.
2. El controller exige rol `Encargado de practica` o `Director de carrera`.
3. El service valida permiso `approve`.
4. Si la solicitud está `Pendiente`, se registra `Pendiente -> En revisión` con
   historial y metadata `{"action": "start_review"}`.
5. Si la solicitud ya no está `Pendiente` o está anulada, la acción es
   idempotente y retorna la práctica sin modificarla.

#### Seguimiento de ciclo de vida

1. El usuario llama a `GET /internships/{internship_id}/lifecycle-tracking`.
2. El controller valida que sea propietario o rol privilegiado.
3. El service combina historial administrativo, autoevaluación, evaluación del
   supervisor, invitaciones al supervisor y presentaciones finales.
4. Se retorna una lista normalizada de eventos con avance porcentual, paso
   actual y banderas para habilitar acciones.
5. El endpoint no modifica estados; solo calcula la vista de avance.

#### Rechazo y preparación documental para DIRAE

1. Un actor autorizado llama a `POST /internships/{internship_id}/reject`, `POST /internships/{internship_id}/derive` o `POST /internships/{internship_id}/dirae-reopen`.
2. El service valida el permiso de acción.
3. Se exige comentario no vacío.
4. Se rechazan prácticas inexistentes.
5. El rechazo cambia el estado administrativo de la solicitud a `Rechazada` y
   libera `blocks_new_registration`.
6. La preparación documental no aprueba ni rechaza la solicitud; actualiza `dirae_status` para habilitar revisión, rectificación y exportación.
7. La preparación documental para DIRAE exige solicitud en estado `Aprobada` y práctica con
   `completion_status=finalized`.
8. La reapertura DIRAE solo permite pasar a `observed` desde paquetes `ready` o `exported`.
9. Se registra historial interno y se intenta notificar al estudiante. Este
   historial no representa seguimiento del trámite externo en DIRAE.

#### Excepciones administrativas

1. Un actor autorizado llama a `POST /internships/{internship_id}/exceptions`.
2. El service valida permiso `grant_exception`.
3. Se verifica que la regla esté dentro del conjunto exceptuable.
4. Se rechaza si la práctica no existe o está en estado terminal.
5. Si ya existe una excepción para la misma práctica y regla, se retorna la existente.
6. Si no existe, se registra la excepción con actor, motivo y fecha.

#### Edición administrativa y anulación lógica

1. Un actor autorizado llama a `PATCH /internships/{internship_id}/admin` o `POST /internships/{internship_id}/cancel`.
2. El service valida permisos `admin_edit` o `cancel`.
3. Se exige motivo no vacío.
4. Se bloquean prácticas anuladas o en estado terminal.
5. La edición exige al menos un campo editable y valida fechas y monto.
6. La anulación marca `is_cancelled`, `cancelled_at`, `cancelled_by` y `cancellation_reason`.
7. Ambas operaciones registran historial administrativo.

## Endpoints disponibles

**Todos los endpoints requieren autenticación, salvo que el acceso exacto dependa de roles indicados en la tabla.**

| Método | Ruta | Propósito | Acceso |
| --- | --- | --- | --- |
| POST | `/internships` | Crea una solicitud de práctica en estado inicial. | Estudiante |
| GET | `/internships` | Lista prácticas para dashboard de revisión. | Encargado de practica, Director de carrera, Secretaria de Carrera |
| GET | `/internships/stats` | Retorna conteos agregados para dashboard. | Encargado de practica, Director de carrera, Secretaria de Carrera |
| GET | `/internships/me` | Lista prácticas del usuario autenticado. | Bearer token |
| GET | `/internships/induction` | Retorna contenido de inducción activo y publicado. | Bearer token |
| POST | `/internships/induction/attempts` | Registra y evalúa un intento de inducción. | Estudiante |
| GET | `/induction/admin/versions` | Lista versiones de inducción para administración. | Encargado de practica, Director de carrera |
| POST | `/induction/admin/versions` | Crea un borrador de inducción. | Encargado de practica, Director de carrera |
| GET | `/induction/admin/versions/{version_id}` | Obtiene detalle administrativo de una versión. | Encargado de practica, Director de carrera |
| PATCH | `/induction/admin/versions/{version_id}` | Edita una versión de inducción. | Encargado de practica, Director de carrera |
| POST | `/induction/admin/versions/{version_id}/publish` | Publica o activa una versión. | Encargado de practica, Director de carrera |
| DELETE | `/induction/admin/versions/{version_id}` | Elimina una versión de inducción. | Encargado de practica, Director de carrera |
| GET | `/internships/registration-eligibility` | Retorna diagnóstico de requisitos y siguiente paso. | Bearer token |
| GET | `/internships/{internship_id}` | Obtiene detalle de una práctica. | Propietario o rol privilegiado |
| GET | `/internships/{internship_id}/tracking` | Lista historial cronológico de estados. | Propietario o rol privilegiado |
| GET | `/internships/{internship_id}/lifecycle-tracking` | Retorna seguimiento agregado de solicitud, ejecución, evaluaciones, presentación y cierre. | Propietario o rol privilegiado |
| GET | `/internships/{internship_id}/dirae-tracking` | Lista historial interno de preparación/exportación del expediente local. No consulta estado externo en DIRAE. | Propietario o rol privilegiado |
| GET | `/internships/{internship_id}/student-actions` | Indica acciones disponibles para el estudiante. | Estudiante propietario |
| PATCH | `/internships/{internship_id}/student` | Permite corrección acotada por estudiante cuando el estado lo permite. | Estudiante propietario |
| POST | `/internships/{internship_id}/student/cancel` | Anula una solicitud propia con motivo. | Estudiante propietario |
| PATCH | `/internships/{internship_id}/admin` | Corrige campos editables con trazabilidad. | Encargado de practica, Director de carrera |
| POST | `/internships/{internship_id}/cancel` | Anula lógicamente una práctica. | Encargado de practica, Director de carrera |
| POST | `/internships/{internship_id}/start-review` | Marca una solicitud pendiente como `En revisión` al abrir detalle administrativo. | Encargado de practica, Director de carrera |
| POST | `/internships/{internship_id}/approve` | Avanza o aprueba una práctica. | Encargado de practica, Director de carrera |
| POST | `/internships/{internship_id}/reject` | Rechaza una práctica con motivo obligatorio. | Encargado de practica, Director de carrera |
| POST | `/internships/{internship_id}/derive` | Inicia preparación/revisión local del expediente para exportación, sin cambiar `currentstate`. | Secretaria de Carrera |
| POST | `/internships/{internship_id}/dirae-reopen` | Reabre el expediente local para rectificación documental previa a una nueva exportación. | Secretaria de Carrera |
| POST | `/internships/{internship_id}/exceptions` | Registra excepción administrativa. | Encargado de practica, Director de carrera |
| GET | `/internships/{internship_id}/exceptions` | Lista excepciones de una práctica. | Propietario o rol privilegiado |

> [!WARNING]
> Además de las dependencias HTTP, el service vuelve a validar la acción concreta
> con `ROLE_PERMISSIONS`. Por eso los permisos efectivos deben revisarse en el
> service antes de cambiar roles o endpoints.

## Contratos principales

<details>
<summary><strong>InternshipCreateRequest</strong></summary>

Payload usado por estudiantes para registrar una solicitud. El backend no acepta
`has_school_insurance` en este contrato.

```json
{
  "org_name": "Empresa X",
  "sector": "Tecnología",
  "address": "Av. Principal 123",
  "city": "Temuco",
  "supervisor_name": "Ana Pérez",
  "supervisor_profession": "Ingeniera Civil Informática",
  "supervisor_position": "Jefa de Proyectos",
  "supervisor_department": "Tecnología",
  "supervisor_email": "ana.perez@example.com",
  "supervisor_phone": "+56912345678",
  "start_date": "2026-06-01",
  "end_date": "2026-08-31",
  "schedule": "09:00-18:00",
  "days": "Lunes a viernes",
  "modality": "Presencial",
  "internship_address": "Av. Práctica 456",
  "act_description": "Desarrollo de funcionalidades backend.",
  "ben_description": "Apoyo al equipo de plataforma.",
  "amount": 120000,
  "internship_period": "Semestre",
  "internship_type": "Práctica de Estudio I"
}
```

</details>

<details>
<summary><strong>InternshipActionRequest</strong></summary>

Payload usado por aprobación, rechazo, derivación y reapertura DIRAE. El comentario es obligatorio para rechazo, derivación y reapertura DIRAE.

```json
{
  "comment": "Documentación revisada por coordinación."
}
```

</details>

<details>
<summary><strong>InternshipExceptionRequest</strong></summary>

Payload usado para registrar una excepción administrativa sobre una regla
exceptuable.

```json
{
  "rule": "school_insurance",
  "reason": "Regularización autorizada por la unidad académica."
}
```

</details>

<details>
<summary><strong>RegistrationEligibilityResponse</strong></summary>

Diagnóstico de requisitos para orientar el siguiente paso antes del avance
administrativo.

```json
{
  "has_school_insurance": false,
  "insurance_status": "requires_exception",
  "has_induction": true,
  "has_school_insurance_exception": false,
  "has_approved_practice_1": true,
  "sequentiality_blocked": false,
  "has_sequentiality_exception": false,
  "has_blocking_internship": false,
  "blocking_internship_id": null,
  "blocking_internship_status": null,
  "can_create_request": true,
  "blocked": true,
  "next_step": "Dirección de carrera debe validar el seguro escolar de la solicitud o registrar una excepción antes de aprobar."
}
```

</details>

<details>
<summary><strong>InductionResponse</strong></summary>

`GET /internships/induction` retorna la versión publicada activa. Las opciones
de cada pregunta se exponen como objeto estable `{clave: texto}` y el frontend
debe enviar la clave seleccionada, no el texto visible.

```json
{
  "id": 1,
  "title": "Inducción obligatoria",
  "min_score": 1,
  "videos": [],
  "questions": [
    {
      "id": 1,
      "question_text": "Confirma que revisaste la inducción obligatoria antes de tramitar tu práctica.",
      "options": {
        "accept": "Entiendo y acepto",
        "reject": "No acepto"
      },
      "order": 1
    }
  ]
}
```

`POST /internships/induction/attempts`:

```json
{
  "answers": {
    "1": "accept"
  }
}
```

</details>

<details>
<summary><strong>InternshipTrackingResponse</strong></summary>

Entrada cronológica del historial de estados o acciones administrativas.

```json
{
  "id": 20,
  "internship_id": 15,
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
    "id": 5,
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

Seguimiento agregado del ciclo completo. A diferencia de
`InternshipTrackingResponse`, no es auditoría de transiciones: es una vista
funcional para mostrar avance y habilitar acciones.

```json
{
  "internship_id": 15,
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
      "id": "request_approved",
      "type": "request_approved",
      "title": "Solicitud de práctica aprobada",
      "description": "La solicitud administrativa fue aprobada.",
      "status": "completed",
      "occurred_at": "2026-06-16T12:30:00Z",
      "metadata": {}
    },
    {
      "id": "supervisor_evaluation_submitted",
      "type": "supervisor_evaluation_submitted",
      "title": "Evaluación del supervisor completada",
      "description": "El supervisor completó la evaluación del estudiante.",
      "status": "current",
      "occurred_at": null,
      "metadata": {}
    }
  ]
}
```

Estados posibles por evento: `completed`, `current`, `pending`, `blocked`.

</details>

## Reglas de negocio

#### Estados funcionales

| Estado | Uso |
| --- | --- |
| `Pendiente` | Estado inicial de una solicitud registrada. |
| `En revisión` | Revisión administrativa previa a aprobación final. |
| `En revisión DIRAE` | Estado histórico legado tratado como revisión administrativa para compatibilidad. El flujo actual usa `dirae_status`. |
| `Aprobada` | Estado terminal exitoso. |
| `Rechazada` | Estado terminal de rechazo. |
| `Reprobada` | Estado histórico tratado como rechazo en dashboard y terminalidad. |

#### Estados de ejecución y cierre

Además del estado administrativo de la solicitud, `Internship` expone campos
para el ciclo posterior de ejecución/cierre:

| Campo | Valores | Uso |
| --- | --- | --- |
| `completion_status` | `not_started`, `in_progress`, `pending_evaluations`, `pending_presentation`, `finalized` | Estado de ejecución/cierre posterior a la aprobación administrativa. |
| `final_result` | `pending`, `passed`, `failed` | Resultado final consolidado de la práctica. |
| `dirae_status` | `not_started`, `in_review`, `observed`, `ready`, `exported` | Marca técnica interna del expediente local para preparación, rectificación y exportación. No representa el estado externo del trámite en DIRAE. |

Estos campos no sustituyen a `currentstate`: `Aprobada` y `Rechazada` siguen
representando el resultado administrativo de la solicitud. El cierre final de
la práctica debe usar `completion_status` y `final_result`.

#### Estados normalizados para dashboard

| Estado normalizado | Estados incluidos |
| --- | --- |
| `submitted` | Sin estado, `Pendiente` o estados no mapeados explícitamente. |
| `in_review` | `En revisión`. |
| `approved` | `Aprobada`. |
| `rejected` | `Rechazada`, `Reprobada`. |

> [!NOTE]
> `En revisión DIRAE` no debe tratarse como estado administrativo nuevo de la
> solicitud ni como seguimiento externo del trámite en DIRAE. En el modelo
> actual esa preparación local se representa con `dirae_status=in_review` y
> conserva `currentstate=Aprobada`.

#### Duplicidad por tipo de práctica

`POST /internships` impide crear otra solicitud vigente para el mismo
`user_id + internship_type` cuando existe una práctica con
`blocks_new_registration=true`.

El backend responde `409 Conflict` con `code=duplicate_internship_type` e
incluye `existing_internship_id`, `internship_type`, `existing_status` y
`message`. La elegibilidad preventiva expone `has_blocking_internship`,
`blocking_internship_id`, `blocking_internship_status` y `can_create_request`.

El bloqueo se libera al rechazar o anular la solicitud. La liberación por
`final_result=failed` pertenece al flujo de cierre final y no debe asumirse como
operativa hasta que ese flujo actualice explícitamente `blocks_new_registration`.

#### Transiciones permitidas

| Estado origen | Estados destino permitidos |
| --- | --- |
| `Pendiente` | `En revisión`, `En revisión DIRAE`, `Aprobada`, `Rechazada`. |
| `En revisión` | `Aprobada`, `Rechazada`, `En revisión DIRAE`. |
| `En revisión DIRAE` | `Aprobada`, `Rechazada`. |
| `Aprobada` | Ninguno. |
| `Rechazada` | Ninguno. |

La preparación local del expediente se expresa por transiciones internas de
`dirae_status`:

| Marca interna origen | Marca interna destino permitido | Condición |
| --- | --- | --- |
| `not_started` | `in_review` | Solicitud `Aprobada` y `completion_status=finalized` (al derivar). |
| `in_review` | `observed`, `ready` | Revisión documental local. Transiciona a `ready` automáticamente cuando todos los documentos requeridos están aprobados (y no quedan pendientes de revisión o con observaciones), y a `observed` si se añade alguna observación. |
| `observed` | `in_review`, `ready` | Rectificación documental. Transiciona a `in_review` si hay archivos nuevos por revisar, y a `ready` automáticamente si todo queda aprobado. |
| `ready` | `exported`, `observed`, `in_review` | Exportación CSV generada correctamente o reapertura para rectificación. Si se vuelve a subir, borrar o modificar el estado de algún documento, retorna a `in_review` u `observed` automáticamente. |
| `exported` | `observed`, `in_review`, `ready` | La exportación marca el paquete como exportado; una reapertura o cambio documental puede devolverlo a revisión local. |

#### Permisos de acción

| Rol | Acciones internas |
| --- | --- |
| `Encargado de practica` | `approve`, `reject`, `grant_exception`, `admin_edit`, `cancel`. |
| `Director de carrera` | `approve`, `reject`, `grant_exception`, `admin_edit`, `cancel`. |
| `Secretaria de Carrera` | `derive`. |

> [!NOTE]
> `grant_exception` aplica a varias reglas. Para `school_insurance`, el service
> exige además rol `Director de carrera`; el encargado puede registrar otras
> excepciones habilitadas, pero no autorizar seguro escolar.

#### Tipos y periodos

| Tipo | Valor |
| --- | --- |
| Práctica de Estudio I | `Práctica de Estudio I`. |
| Práctica de Estudio II | `Práctica de Estudio II`. |
| Práctica Controlada | `Práctica Controlada`. |
| Tesis | `Tesis`. |

| Periodo | Valor |
| --- | --- |
| Semestre | `Semestre`. |
| Verano | `Verano`. |
| Invierno | `Invierno`. |

#### Reglas de aprobación

- `Práctica de Estudio I` requiere inducción aprobada antes de avanzar administrativamente.
- `Práctica de Estudio II` requiere `Práctica de Estudio I` aprobada en `StudentInternshipRequirement` o excepción `sequentiality`.
- `Tesis` requiere `Práctica de Estudio II` aprobada en `StudentInternshipRequirement` o excepción `sequentiality_thesis`.
- `Práctica Controlada` requiere excepción `parallel_course` mientras no exista modelamiento de co-requisitos.
- Las fechas dentro de marzo-junio o agosto-noviembre se auto-validan para
  seguro escolar. Fuera de esos rangos, la solicitud requiere
  `insurance_status=validated` o excepción `school_insurance` antes de llegar a
  `Aprobada`.

#### Reglas exceptuables

| Regla | Cuándo aplica |
| --- | --- |
| `school_insurance` | Permite aprobar una solicitud fuera de periodo regular sin seguro validado, solo con autorización de Dirección de carrera. |
| `sequentiality` | Permite avanzar Práctica de Estudio II sin Práctica de Estudio I aprobada. |
| `sequentiality_thesis` | Permite avanzar Tesis sin Práctica de Estudio II aprobada. |
| `parallel_course` | Permite avanzar Práctica Controlada pese a co-requisitos pendientes o no modelados. |

> [!IMPORTANT]
> Una excepción no modifica el dato base. Por ejemplo, una excepción
> `school_insurance` no cambia `has_school_insurance` a `true`. Solo habilita el
> avance administrativo con trazabilidad.

#### Seguro escolar

`has_school_insurance` se calcula desde `StudentRegistrationRequirement` o desde
la auto-validación por fechas regulares al crear la solicitud, pero la fuente
autoritativa para aprobar es `insurance_status` de la solicitud concreta.

Si la práctica está completamente dentro de marzo-junio o agosto-noviembre, el
backend puede marcar `insurance_status=validated` automáticamente. Si está fuera
de esos rangos, Dirección de carrera debe validar la solicitud mediante
`PATCH /admin/internships/{internship_id}/school-insurance` o registrar una
excepción `school_insurance`.

Aunque `Encargado de practica` y `Director de carrera` pueden registrar
excepciones generales, la excepción `school_insurance` queda restringida a
`Director de carrera`.

#### Inducción

La inducción se considera cumplida si existe un prerrequisito institucional
`induction` completado o un intento aprobado en `InductionAttempt`. El contenido
visible para estudiantes debe estar publicado y activo.

Si la versión activa tiene `requires_retake=true`, una inducción histórica deja
de ser suficiente y debe existir un intento aprobado para esa versión activa.
La creación de solicitudes por estudiante requiere inducción aprobada; si falta,
`POST /internships` responde `409 Conflict` con `code=induction_required`.

#### Sincronización académica

Cuando una práctica llega a `Aprobada`, el service crea o actualiza
`StudentInternshipRequirement` con estado `Aprobada`. Si esta sincronización falla,
la práctica ya aprobada no se revierte automáticamente.

## Configuración por entorno

El módulo **`internships`** no define variables de entorno propias. Sin embargo,
construye `NotificationService` para emitir eventos asociados a creación,
aprobación, rechazo y derivación. También usa `STUDENT_EFFECTIVE_CORRECTION_WINDOW_HOURS` para calcular la ventana de corrección/anulación reciente del estudiante.

| Variable indirecta | Impacto |
| --- | --- |
| `NOTIFICATION_MODE` | Define si las notificaciones se simulan o se despachan con configuración real. |
| `MAIL_*` | Requeridas por el módulo `notifications` cuando `NOTIFICATION_MODE=real`. |
| `STUDENT_EFFECTIVE_CORRECTION_WINDOW_HOURS` | Horas durante las cuales una solicitud pendiente puede ser corregida o anulada por su estudiante antes de intervención administrativa. |

> [!NOTE]
> Las notificaciones son efectos secundarios no bloqueantes. Si falla el envío o
> el servicio no está configurado, el flujo principal de práctica continúa.

## Consideraciones operativas

- Los estados base deben existir en la tabla `currentstate`. Si falta un estado requerido, el service falla al resolverlo.
- El historial `InternshipStatusHistory` es la fuente para auditoría de transiciones, edición administrativa y anulación.
- Las prácticas anuladas conservan su estado actual y se marcan con `is_cancelled`.
- No se permite editar ni anular prácticas anuladas o en estados terminales.
- Los filtros del dashboard usan estados normalizados, no necesariamente los títulos exactos de base de datos.
- Las reglas de aprobación dependen de datos de requisitos académicos e institucionales mantenidos por flujos administrativos.
- El modelo de co-requisitos de `Práctica Controlada` aún no existe, por lo que la excepción `parallel_course` actúa como control operativo.
