<h1 align="center"><em>Supervisor Evaluations</em></h1>

> [!NOTE]
> Esta documentación técnica describe el comportamiento actual del módulo
> `supervisor_evaluations` desde una perspectiva funcional e interna. Su objetivo
> es explicar qué hace el módulo, cómo se conecta con el resto del backend y qué
> debe saber alguien antes de modificarlo. El contrato HTTP formal queda en
> OpenAPI y el detalle interactivo queda en Swagger.

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

El módulo **`supervisor_evaluations`** gestiona la evaluación que realiza el
supervisor externo de una práctica. La evaluación se completa desde un enlace con
token, por lo que el supervisor no necesita iniciar sesión en la plataforma para
responder el formulario público.

**Permite:**

- Generar o reenviar una invitación de evaluación para una práctica.
- Entregar un formulario público mínimo mediante token.
- Recibir una evaluación externa una sola vez.
- Consultar una evaluación ya enviada con permisos internos.
- Listar prácticas asociadas al correo de un usuario con rol supervisor.

En términos simples, este módulo permite recoger la mirada de la organización o
empresa donde el estudiante realizó su práctica.

## Ámbito y responsabilidades

El módulo **`supervisor_evaluations`** se enfoca en la evaluación enviada por el
supervisor externo. No reemplaza la autoevaluación del estudiante ni decide por sí
solo el cierre de la práctica, pero entrega una evidencia importante para ese
proceso.

#### Responsabilidades principales

- Crear invitaciones seguras con token de un solo uso.
- Evitar invitaciones prematuras si la práctica aún no cumple prerequisitos.
- Exponer un formulario público con información mínima.
- Validar respuestas y puntajes del supervisor.
- Guardar la evaluación enviada y consumir el token.
- Permitir lectura protegida de evaluaciones y asignaciones.

#### Fuera de alcance

- Crear o aprobar solicitudes de práctica.
- Completar la autoevaluación del estudiante.
- Administrar usuarios, roles o sesiones.
- Convertir automáticamente una recomendación en resultado final.
- Mantener un portal externo completo para supervisores.

> [!IMPORTANT]
> El acceso público existe solo para responder la evaluación mediante token. Las
> consultas internas de evaluaciones siguen protegidas por autenticación y roles.

## Estructura interna

| Capa | Archivo | Responsabilidad |
| --- | --- | --- |
| Controller | `app/modules/supervisor_evaluations/controllers/supervisor_evaluation_controller.py` | Define rutas HTTP para invitaciones, formulario público, envío, consulta y asignaciones. |
| Service | `app/modules/supervisor_evaluations/services/supervisor_evaluation_service.py` | Orquesta prerequisitos, tokens, permisos, envío y notificaciones. |
| Repository | `app/modules/supervisor_evaluations/repositories/supervisor_evaluation_repository.py` | Encapsula consultas y persistencia de invitaciones, evaluaciones y prácticas asociadas. |
| Models | `app/modules/supervisor_evaluations/models/supervisor_evaluation_model.py` | Define invitaciones y evaluaciones del supervisor. |
| Schemas | `app/modules/supervisor_evaluations/schemas/supervisor_evaluation_schema.py` | Define contratos de formulario, envío, respuestas y asignaciones. |

El módulo reutiliza identidad desde `auth`, datos de prácticas desde
`internships`, autoevaluaciones desde `self_evaluations` y notificaciones mediante
`notifications`.

## Funcionalidades

#### Generación de invitación

1. Un rol administrativo llama a
   `POST /supervisor/evaluations/internships/{internship_id}/invitations`.
2. El backend valida que la práctica exista, esté aprobada y no esté anulada.
3. Se exige que la autoevaluación del estudiante ya haya sido enviada.
4. Si había invitaciones activas anteriores, se revocan.
5. Se crea una nueva invitación con token y expiración.
6. Se envía una notificación al correo del supervisor.

#### Formulario público por token

1. El supervisor abre el enlace recibido por correo.
2. El frontend llama a `GET /supervisor/evaluations/invitations/{token}`.
3. El backend valida que el token exista, no esté vencido, usado ni revocado.
4. Se retorna información mínima de la práctica, estudiante, supervisor y criterios.

Este endpoint no requiere sesión porque el token funciona como autorización
temporal y acotada.

#### Envío de evaluación

1. El supervisor envía el formulario con
   `POST /supervisor/evaluations/invitations/{token}/submit`.
2. El backend valida todos los criterios y puntajes.
3. Se crea la evaluación y se marca la invitación como usada.
4. El token no puede reutilizarse para enviar otra evaluación.

#### Consulta interna

1. Un usuario autorizado llama a
   `GET /supervisor/evaluations/internships/{internship_id}`.
2. El backend valida que sea el estudiante propietario o un rol administrativo
   autorizado.
3. Se retorna la evaluación enviada por el supervisor.

#### Asignaciones del supervisor autenticado

1. Un usuario con rol `Supervisor de practica` llama a
   `GET /supervisor/evaluations/me`.
2. El backend busca prácticas cuyo correo de supervisor coincida con el correo
   autenticado.
3. Se retorna un resumen de prácticas y si la evaluación ya fue enviada.

## Endpoints disponibles

| Método | Ruta | Propósito | Acceso principal |
| --- | --- | --- | --- |
| POST | `/supervisor/evaluations/internships/{internship_id}/invitations` | Genera o reenvía invitación al supervisor. | Roles administrativos |
| GET | `/supervisor/evaluations/invitations/{token}` | Consulta formulario público por token. | Público con token válido |
| POST | `/supervisor/evaluations/invitations/{token}/submit` | Envía evaluación pública del supervisor. | Público con token válido |
| GET | `/supervisor/evaluations/internships/{internship_id}` | Consulta evaluación enviada. | Estudiante propietario o roles administrativos |
| GET | `/supervisor/evaluations/me` | Lista asignaciones del supervisor autenticado. | Supervisor de practica |

## Contratos principales

<details>
<summary><strong>SupervisorEvaluationInvitationResponse</strong></summary>

Respuesta al generar una invitación. En modo simulado puede incluir token y URL
de demo para facilitar pruebas locales.

```json
{
  "invitation_id": 15,
  "internship_id": 30,
  "supervisor_email": "supervisor@empresa.cl",
  "expires_at": "2026-07-08T12:00:00",
  "revoked_previous_count": 1,
  "demo_token": null,
  "demo_url": null
}
```

</details>

<details>
<summary><strong>SupervisorEvaluationPublicResponse</strong></summary>

Información mínima que el supervisor externo necesita para responder el
formulario.

```json
{
  "internship_id": 30,
  "org_name": "Empresa Demo",
  "student_name": "Estudiante Demo",
  "internship_type": "Práctica de Estudio I",
  "supervisor_name": "Supervisor Empresa",
  "criteria": []
}
```

</details>

<details>
<summary><strong>SupervisorEvaluationSubmitRequest</strong></summary>

Payload público de envío. Debe incluir todos los criterios esperados con puntajes
entre `1` y `5`.

```json
{
  "criteria_scores": {
    "technical_performance": 5,
    "responsibility": 5,
    "communication": 4,
    "teamwork": 4,
    "autonomy": 5
  },
  "observations": "Buen desempeño general durante la práctica.",
  "recommendation": "recommended"
}
```

Recomendaciones soportadas: `recommended`, `recommended_with_observations` y
`not_recommended`.

</details>

## Reglas de negocio

#### Invitaciones

**Reglas actuales:**

- La práctica debe existir y no estar anulada.
- La práctica debe estar aprobada administrativamente.
- La autoevaluación del estudiante debe estar enviada.
- No se genera una nueva invitación si ya existe evaluación enviada.
- Al generar una nueva invitación se revocan invitaciones activas anteriores.
- El token se guarda hasheado y no como texto plano.

#### Formulario público

**Reglas actuales:**

- El token debe existir, estar vigente y no haber sido usado.
- Una invitación revocada no puede usarse.
- Una invitación expirada no puede usarse.
- El formulario público expone solo información mínima para responder.

#### Envío de evaluación

**Reglas actuales:**

- La evaluación se puede enviar una sola vez por práctica.
- El envío consume la invitación.
- Todos los criterios deben estar presentes.
- Los puntajes válidos van de `1` a `5`.
- La recomendación debe usar uno de los valores permitidos.

#### Lectura interna

**Reglas actuales:**

- El estudiante propietario puede consultar la evaluación de su práctica.
- Roles administrativos autorizados pueden consultar evaluaciones.
- Un supervisor autenticado solo ve asignaciones asociadas a su propio correo.

> [!WARNING]
> El token público debe tratarse como credencial temporal. No debe registrarse en
> logs, exponerse innecesariamente ni guardarse en texto plano.

## Consideraciones operativas

- Las invitaciones expiran después de un periodo acotado.
- En modo `simulated`, la respuesta puede incluir datos de demo para pruebas
  locales.
- El enlace público se construye usando el origen frontend configurado.
- Las notificaciones pueden operar en modo simulado o real según configuración
  global.
- Los contratos exactos de campos, validaciones y errores deben consultarse en
  Swagger/OpenAPI.
- Las pruebas unitarias documentadas están en
  `backend/tests/modules/supervisor-evaluations-unitarias.md`.
