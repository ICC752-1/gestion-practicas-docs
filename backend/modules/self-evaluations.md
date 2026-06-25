<h1 align="center"><em>Self Evaluations</em></h1>

> [!NOTE]
> Esta documentación técnica describe el comportamiento actual del módulo
> `self_evaluations` desde una perspectiva funcional e interna. Su objetivo es
> explicar qué hace el módulo, cómo se conecta con el resto del backend y qué debe
> saber alguien antes de modificarlo. El contrato HTTP formal queda en OpenAPI y
> el detalle interactivo queda en Swagger.

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

El módulo **`self_evaluations`** gestiona la autoevaluación que completa el
estudiante al final de una práctica. Sirve para registrar cómo el estudiante
evalúa su desempeño, aprendizajes y experiencia antes del cierre académico.

**Permite:**

- Consultar si una práctica ya tiene habilitada la autoevaluación.
- Entregar al frontend el formulario, escala y criterios vigentes.
- Guardar respuestas parciales como borrador.
- Enviar la autoevaluación definitiva.
- Listar autoevaluaciones propias del estudiante.
- Reabrir una autoevaluación enviada cuando existe una corrección administrativa.

En términos simples, este módulo recoge la evidencia del estudiante sobre su
propia práctica y controla cuándo puede completarla.

## Ámbito y responsabilidades

El módulo **`self_evaluations`** se enfoca solo en la evaluación hecha por el
estudiante. No evalúa al supervisor ni decide por sí solo el resultado final de la
práctica, pero sí forma parte del proceso de cierre.

#### Responsabilidades principales

- Definir el formulario actual de autoevaluación.
- Validar que la práctica esté en una etapa apta para evaluar.
- Permitir borradores antes del envío definitivo.
- Bloquear edición después del envío, salvo reapertura administrativa.
- Registrar trazabilidad de reaperturas.
- Notificar eventos relevantes del flujo.

#### Fuera de alcance

- Crear o aprobar solicitudes de práctica.
- Registrar la evaluación del supervisor externo.
- Calcular por sí solo el resultado final de la práctica.
- Administrar usuarios, roles o sesiones.
- Hacer configurable el formulario desde base de datos.

> [!IMPORTANT]
> La autoevaluación es una pieza del cierre de práctica. El seguimiento completo
> del ciclo de vida sigue perteneciendo al módulo `internships`.

## Estructura interna

| Capa | Archivo | Responsabilidad |
| --- | --- | --- |
| Controller | `app/modules/self_evaluations/controllers/self_evaluation_controller.py` | Define rutas HTTP para formulario, borrador, envío, listado y reapertura. |
| Service | `app/modules/self_evaluations/services/self_evaluation_service.py` | Orquesta permisos, ventana de habilitación, estados, notificaciones y reapertura. |
| Repository | `app/modules/self_evaluations/repositories/self_evaluation_repository.py` | Encapsula consultas y persistencia de autoevaluaciones y prácticas asociadas. |
| Models | `app/modules/self_evaluations/models/self_evaluation_model.py` | Define la autoevaluación y sus estados funcionales. |
| Schemas | `app/modules/self_evaluations/schemas/self_evaluation_schema.py` | Define formulario, escala, criterios y payloads del módulo. |

El módulo reutiliza autenticación desde `auth`, datos de prácticas desde
`internships`, notificaciones mediante `notifications` y coordina con
`supervisor_evaluations` después del envío.

## Funcionalidades

#### Consulta del formulario

1. El usuario llama a `GET /self-evaluations/internships/{internship_id}/form`.
2. El backend valida que pueda acceder a la práctica.
3. Se responde si el formulario está habilitado o no.
4. Si está disponible, se entregan criterios, escala y evaluación existente.

Esta respuesta ayuda al frontend a mostrar el formulario o explicar por qué aún
no corresponde completarlo.

#### Guardado de borrador

1. El estudiante llama a `PUT /self-evaluations/internships/{internship_id}/draft`.
2. El backend valida que sea propietario de la práctica.
3. Se verifica que la autoevaluación esté habilitada.
4. Se guardan respuestas parciales y observaciones.

El borrador permite avanzar sin completar todos los criterios en una sola sesión.

#### Envío definitivo

1. El estudiante llama a `POST /self-evaluations/internships/{internship_id}/submit`.
2. El backend exige todas las respuestas requeridas.
3. La autoevaluación queda con estado `submitted`.
4. Desde ese momento no puede editarse sin reapertura.
5. Se notifica el envío y se puede iniciar el flujo de evaluación del supervisor.

#### Listado propio

1. El estudiante llama a `GET /self-evaluations/me`.
2. El backend lista las autoevaluaciones asociadas al usuario autenticado.
3. Se retorna metadata, estado, respuestas y fechas relevantes.

#### Reapertura administrativa

1. Un rol administrativo llama a `POST /self-evaluations/{evaluation_id}/reopen`.
2. El backend exige que la autoevaluación exista y esté enviada.
3. Se registra motivo, actor y fecha de reapertura.
4. El estudiante puede volver a editar y reenviar la autoevaluación.

## Endpoints disponibles

**Todos los endpoints requieren autenticación.**

| Método | Ruta | Propósito | Acceso principal |
| --- | --- | --- | --- |
| GET | `/self-evaluations/internships/{internship_id}/form` | Consulta formulario y estado de habilitación. | Estudiante propietario o rol administrativo |
| GET | `/self-evaluations/me` | Lista autoevaluaciones propias. | Estudiante |
| PUT | `/self-evaluations/internships/{internship_id}/draft` | Guarda borrador de autoevaluación. | Estudiante propietario |
| POST | `/self-evaluations/internships/{internship_id}/submit` | Envía la autoevaluación definitiva. | Estudiante propietario |
| POST | `/self-evaluations/{evaluation_id}/reopen` | Reabre una autoevaluación enviada. | Roles administrativos |

## Contratos principales

<details>
<summary><strong>SelfEvaluationFormResponse</strong></summary>

Respuesta usada para saber si el estudiante puede completar el formulario y qué
criterios debe responder.

```json
{
  "form_version": "student-self-evaluation-v1",
  "enabled": true,
  "status": "not_started",
  "reason": null,
  "scale": {
    "min_score": 1,
    "max_score": 5
  },
  "criteria": []
}
```

</details>

<details>
<summary><strong>SelfEvaluationDraftRequest</strong></summary>

Payload usado para guardar respuestas parciales.

```json
{
  "responses": {
    "communication": 4,
    "teamwork": 5
  },
  "observations": "Durante la práctica mejoré mi comunicación con el equipo."
}
```

</details>

<details>
<summary><strong>SelfEvaluationSubmitRequest</strong></summary>

Payload usado para enviar la autoevaluación. A diferencia del borrador, debe
incluir todos los criterios requeridos.

```json
{
  "responses": {
    "communication": 4,
    "teamwork": 5,
    "organization_understanding": 4,
    "process_understanding": 4,
    "risk_prevention": 5,
    "ethics": 5,
    "learning_application": 4
  },
  "observations": "La experiencia fue positiva y aportó a mi formación."
}
```

</details>

<details>
<summary><strong>SelfEvaluationReopenRequest</strong></summary>

Payload usado por administración para permitir una corrección posterior al envío.

```json
{
  "reason": "Se solicita corregir una respuesta enviada por error."
}
```

</details>

## Reglas de negocio

#### Habilitación

**Reglas actuales:**

- La práctica no debe estar anulada.
- La práctica debe estar aprobada administrativamente.
- La autoevaluación se habilita cerca del cierre de la práctica.
- También puede habilitarse si la práctica ya está en una etapa de cierre o
  evaluaciones pendientes.

#### Respuestas

**Reglas actuales:**

- Los puntajes válidos van de `1` a `5`.
- El borrador puede tener respuestas parciales.
- El envío definitivo debe incluir todos los criterios requeridos.
- No se aceptan criterios desconocidos.

#### Estados

**Reglas actuales:**

- Una autoevaluación puede estar en `draft`, `submitted` o `reopened`.
- Una autoevaluación enviada no puede editarse sin reapertura.
- Solo roles administrativos pueden reabrir una autoevaluación enviada.
- La reapertura debe registrar motivo, actor y fecha.

> [!WARNING]
> No se debe usar una reapertura para borrar trazabilidad. La reapertura existe
> para corregir una evaluación enviada manteniendo registro administrativo.

## Consideraciones operativas

- El formulario actual está definido en código con versión
  `student-self-evaluation-v1`.
- La escala funcional va de `1` a `5`.
- Existe una autoevaluación única por estudiante y práctica.
- El envío puede activar efectos relacionados con la evaluación del supervisor.
- Los contratos exactos de campos, validaciones y errores deben consultarse en
  Swagger/OpenAPI.
- Las pruebas unitarias documentadas están en
  `backend/tests/modules/self-evaluations/self-evaluations-unitarias.md`.
