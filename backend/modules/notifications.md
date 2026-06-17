<h1 align="center"><em>Notifications</em></h1>

> [!NOTE]
> Esta documentación técnica describe el comportamiento actual del módulo
> `notifications` desde una perspectiva funcional e interna. Su objetivo es
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
- [Eventos y productores](#eventos-y-productores)
- [Almacenamiento y estados](#almacenamiento-y-estados)
- [Reglas de negocio](#reglas-de-negocio)
- [Configuración por entorno](#configuración-por-entorno)
- [Consideraciones operativas](#consideraciones-operativas)
- [Documentación relacionada](#documentación-relacionada)

## Resumen operativo

El módulo **`notifications`** gestiona notificaciones persistentes asociadas a
eventos administrativos del sistema y entrega soporte para envío SMTP real o
simulado según configuración de entorno.

**Permite:**

- Persistir notificaciones por usuario destinatario.
- Listar notificaciones del usuario autenticado.
- Consultar el detalle de una notificación propia.
- Crear y despachar notificaciones en modo `simulated` o `real`.
- Reintentar notificaciones `failed` o `pending`.
- Enviar correos directos mediante un endpoint de compatibilidad.
- Construir contenido HTML transaccional para eventos del sistema.

> [!IMPORTANT]
> El módulo tiene dos caminos distintos. Las notificaciones persistentes por
> eventos usan `create_and_dispatch`. El endpoint `POST /notifications/send-email`
> es un flujo directo de SMTP y no persiste una entidad `Notification`.

## Ámbito y responsabilidades

El módulo **`notifications`** es transversal. No decide cuándo ocurre un evento de
negocio, sino que recibe una notificación construida por otro módulo, la persiste
y la despacha según el modo configurado.

#### Responsabilidades principales

- Persistencia de notificaciones.
- Consulta de notificaciones por destinatario.
- Despacho SMTP en modo real.
- Persistencia simulada en modo desarrollo.
- Reintento manual de notificaciones fallidas o pendientes.
- Construcción de notificaciones HTML desde helpers de eventos.
- Exposición de un endpoint legacy de envío SMTP directo.

#### Fuera de alcance

- Determinar reglas de negocio de prácticas, documentos o requisitos.
- Bloquear flujos principales si falla una notificación emitida como efecto secundario.
- Gestionar colas externas o workers de reintento automático.
- Marcar notificaciones como leídas.
- Gestionar plantillas externas o configurables por base de datos.
- Reemplazar auditoría técnica o logs operativos.

> [!NOTE]
> Los módulos productores deciden qué evento notificar. `notifications` se limita
> a persistir, enviar si corresponde y exponer consultas sobre esas notificaciones.

## Estructura interna

| Capa | Archivo | Responsabilidad |
| --- | --- | --- |
| Controller | `app/modules/notifications/controllers/notification_controller.py` | Define endpoints de consulta, envío directo y reintento. |
| Service | `app/modules/notifications/services/notification_service.py` | Orquesta persistencia, modos de operación, SMTP y reintentos. |
| Repository | `app/modules/notifications/repositories/notification_repository.py` | Encapsula creación, consulta por destinatario, consulta por estado y actualización de estado. |
| Models | `app/modules/notifications/models/notification_model.py` | Define `Notification`, `NotificationEventTypeEnum` y `NotificationStatusEnum`. |
| Schemas | `app/modules/notifications/schemas/notification_schema.py` | Define contratos HTTP de envío, listado, detalle y reintento. |
| Utils | `app/modules/notifications/utils/notification_event_helpers.py` | Construye notificaciones HTML para eventos funcionales del sistema. |

El módulo reutiliza configuración desde `app/core/config.py`, autenticación y roles
desde `auth`, y es consumido por `admin`, `internships` y `documents`.

## Funcionalidades

#### Listado de notificaciones

1. El usuario llama a `GET /notifications`.
2. El controller valida Bearer token.
3. Se usan `limit` y `offset` para paginación.
4. El service consulta notificaciones asociadas a `recipient_user_id`.
5. El repository ordena por `created_at` descendente.
6. Se retorna una lista de `NotificationListItemResponse`.

#### Detalle de notificación

1. El usuario llama a `GET /notifications/{notification_id}`.
2. El controller valida Bearer token.
3. El service obtiene la notificación por ID.
4. Si no existe, responde `404`.
5. Si el usuario autenticado no es el destinatario, responde `403`.
6. Se retorna `NotificationDetailResponse`.

#### Creación y despacho por eventos

1. Un módulo productor construye una entidad `Notification` mediante un helper.
2. El productor llama a `NotificationService.create_and_dispatch`.
3. En modo `simulated`, el service asigna estado `simulated` y persiste sin SMTP.
4. En modo `real`, el service asigna estado `pending` y persiste.
5. Si el envío SMTP funciona, actualiza a `sent` y registra `sent_at`.
6. Si el envío SMTP falla, actualiza a `failed`.

#### Envío SMTP directo

1. Un usuario autorizado llama a `POST /notifications/send-email`.
2. El payload incluye destinatarios, asunto y cuerpo HTML.
3. El controller exige rol documental o administrativo autorizado.
4. El service usa `send_email` y envía vía SMTP.
5. Si no hay mailer configurado, responde `400`.
6. Si ocurre un error SMTP inesperado, responde `500`.

> [!WARNING]
> Este endpoint mantiene compatibilidad con un flujo anterior de envío directo.
> No crea registros en la tabla `notification`.

#### Reintento de notificación

1. Un usuario autorizado llama a `POST /notifications/{notification_id}/retry`.
2. El controller exige rol `Encargado de practica` o `Director de carrera`.
3. El service verifica que exista mailer SMTP configurado.
4. Se busca la notificación por ID.
5. Solo son elegibles estados `failed` y `pending`.
6. Si el reintento funciona, se actualiza a `sent` y se registra `sent_at`.
7. Si falla, se actualiza a `failed`.
8. Si no existe, no es elegible o no hay mailer, el controller responde `404`.

## Endpoints disponibles

| Método | Ruta | Propósito | Acceso |
| --- | --- | --- | --- |
| GET | `/notifications` | Lista notificaciones del usuario autenticado. | Bearer token |
| GET | `/notifications/{notification_id}` | Obtiene detalle de una notificación propia. | Destinatario |
| POST | `/notifications/send-email` | Envía correo directo sin persistir notificación. | Encargado de practica, Director de carrera, Secretaria de Carrera |
| POST | `/notifications/{notification_id}/retry` | Reintenta notificación `failed` o `pending`. | Encargado de practica, Director de carrera |

## Contratos principales

<details>
<summary><strong>NotificationListItemResponse</strong></summary>

Representa una notificación en el listado del usuario autenticado.

```json
{
  "id": 10,
  "event_type": "internship_approved",
  "subject": "Práctica aprobada",
  "status": "simulated",
  "created_at": "2026-06-16T12:00:00Z",
  "sent_at": null
}
```

</details>

<details>
<summary><strong>NotificationDetailResponse</strong></summary>

Representa el detalle de una notificación propia.

```json
{
  "id": 10,
  "recipient_user_id": 5,
  "recipient_email": "student@ufromail.cl",
  "event_type": "internship_rejected",
  "subject": "Práctica rechazada",
  "content": "<html>...</html>",
  "status": "simulated",
  "payload": {
    "internship_id": 7,
    "reason": "No cumple requisitos"
  },
  "created_at": "2026-06-16T12:00:00Z",
  "sent_at": null
}
```

</details>

<details>
<summary><strong>EmailNotificationRequest</strong></summary>

Payload usado por el endpoint de envío SMTP directo.

```json
{
  "to_emails": [
    "destinatario@example.com"
  ],
  "subject": "Aviso administrativo",
  "body": "<p>Contenido del correo.</p>"
}
```

</details>

<details>
<summary><strong>NotificationRetryResponse</strong></summary>

Respuesta tras intentar reenviar una notificación persistente.

```json
{
  "success": true,
  "message": "Notificacion enviada exitosamente",
  "notification_id": 10,
  "status": "sent"
}
```

</details>

## Eventos y productores

| Productor | Evento funcional | `event_type` | Payload principal |
| --- | --- | --- | --- |
| `internships` | Práctica creada. | `custom` | `event`, `internship_id`, `student_user_id`. |
| `internships` | Práctica aprobada. | `internship_approved` | `internship_id`. |
| `internships` | Práctica rechazada. | `internship_rejected` | `internship_id`, `reason`. |
| `internships` | Práctica derivada. | `internship_derived` | `internship_id`, `reason`. |
| `documents` | Documento cargado. | `custom` | `event`, `document_id`, `internship_id`, `document_type`. |
| `documents` | Estado documental cambiado. | `custom` | `event`, `document_id`, `internship_id`, `new_status`, `comment`. |
| `admin` | Estado de requisito cambiado. | `requirement_status_changed` | `requirement_id`, `requirement_type`, `new_status`, `previous_status`. |

> [!IMPORTANT]
> El enum `NotificationEventTypeEnum` no tiene valores específicos para
> `internship_created`, `document_uploaded` ni `document_status_changed`. Esos
> casos usan `event_type=custom` y se distinguen mediante `payload.event`.

## Almacenamiento y estados

Las notificaciones persistentes se guardan en la tabla `notification`.

| Campo | Uso |
| --- | --- |
| `recipient_user_id` | Usuario destinatario interno. Puede ser `NULL`. |
| `recipient_email` | Correo de destino para SMTP. Puede ser `NULL`. |
| `event_type` | Tipo de evento normalizado. |
| `subject` | Asunto visible de la notificación. |
| `content` | Contenido de la notificación, normalmente HTML. |
| `status` | Estado de despacho. |
| `payload` | Metadata JSONB del evento. |
| `created_at` | Fecha de creación. |
| `sent_at` | Fecha de envío real. Es `NULL` para `simulated`, `pending` o `failed`. |

#### Estados de notificación

| Estado | Significado |
| --- | --- |
| `simulated` | Notificación persistida sin envío SMTP real. |
| `pending` | Notificación persistida en modo real antes del intento de envío. |
| `sent` | Notificación enviada correctamente vía SMTP. |
| `failed` | Notificación cuyo envío SMTP falló. |

#### Tipos de evento del modelo

| `event_type` | Uso |
| --- | --- |
| `internship_approved` | Notifica aprobación de práctica. |
| `internship_rejected` | Notifica rechazo de práctica. |
| `internship_derived` | Notifica derivación de práctica a DIRAE. |
| `requirement_status_changed` | Notifica cambio de estado de requisito. |
| `custom` | Agrupa eventos que no tienen valor enum propio. |

## Reglas de negocio

#### Modos de operación

- En modo `simulated`, las notificaciones se persisten con estado `simulated` y no se llama SMTP.
- En modo `real`, las notificaciones se persisten con estado `pending` y luego se intenta SMTP.
- Si SMTP funciona, el estado pasa a `sent` y se registra `sent_at`.
- Si SMTP falla, el estado pasa a `failed`.
- Si `NOTIFICATION_MODE=real` pero las credenciales son las de prueba, el mailer no se configura y las notificaciones se persisten como `simulated`.

#### Consulta

- `GET /notifications` solo retorna notificaciones del usuario autenticado.
- `GET /notifications/{notification_id}` solo permite consultar notificaciones propias.
- No existe endpoint para marcar notificaciones como leídas.

#### Reintento

- Solo se reintentan notificaciones `failed` o `pending`.
- Las notificaciones `sent` o `simulated` no son elegibles para reintento.
- Sin mailer SMTP configurado, el reintento no se ejecuta.
- El controller expone los casos no elegibles como `404`.

#### Productores externos

- Los productores pueden omitir notificaciones si `notification_service` es `None`.
- Los errores de despacho en productores no interrumpen el flujo principal.
- `create_and_dispatch` sí registra `failed` cuando falla SMTP en modo real.

## Configuración por entorno

| Variable | Uso |
| --- | --- |
| `NOTIFICATION_MODE` | Define `simulated` o `real`. |
| `MAIL_USERNAME` | Usuario SMTP. |
| `MAIL_PASSWORD` | Password o app password SMTP. |
| `MAIL_FROM` | Remitente del correo. |
| `MAIL_PORT` | Puerto SMTP. |
| `MAIL_SERVER` | Servidor SMTP. |
| `MAIL_STARTTLS` | Habilita STARTTLS. |
| `MAIL_SSL_TLS` | Habilita SSL/TLS directo. |

> [!NOTE]
> `simulated` es el modo seguro para desarrollo y pruebas. `real` requiere
> credenciales SMTP reales. Los valores por defecto `test@example.com` y
> `password` provocan que el mailer no se configure.

## Consideraciones operativas

- Una notificación `simulated` fue persistida, pero no enviada por correo.
- El endpoint `send-email` directo no deja registro en la tabla `notification`.
- El contenido HTML se construye con helpers internos y escapa valores dinámicos con `html.escape`.
- No guardar credenciales SMTP en documentación, logs ni payloads.
- No existe cola externa, scheduler ni reintento automático.
- Si se necesita entrega garantizada, se requiere un worker o cola fuera de la implementación actual.
- El módulo no maneja preferencias de usuario ni estado leído/no leído.

## Documentación relacionada

- `docs/modules/internships/internships-technical-reference.md`: Describe eventos de práctica que generan notificaciones.
- `docs/modules/documents/documents-technical-reference.md`: Describe eventos documentales que generan notificaciones.
- `docs/modules/admin.md`: Describe cambios de requisitos que pueden generar notificaciones.
- `app/modules/notifications/utils/notification_event_helpers.py`: Define helpers de construcción de contenido y payload.
