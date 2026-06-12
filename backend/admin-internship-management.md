# Gestión Administrativa De Prácticas

> [!NOTE]
> Este documento define el contrato funcional para edición administrativa acotada
> y anulación lógica de prácticas. La implementación backend debe ajustarse a
> estas reglas.

## Alcance

La gestión administrativa de prácticas permite corregir datos operativos de una
práctica o anularla lógicamente sin eliminar el registro de la base de datos.

Esta funcionalidad no debe usarse para modificar prerrequisitos sensibles. La
edición directa de seguro escolar e inducción queda fuera de este contrato.

## Roles Autorizados

Pueden editar datos acotados y anular lógicamente una práctica:

- `Encargado de practica`
- `Director de carrera`

No quedan autorizados por este contrato:

- `Estudiante`
- `Supervisor de practica`
- `Secretaria de Carrera`

## Campos Editables

Mientras la práctica no esté anulada ni en estado terminal, los roles
autorizados pueden corregir únicamente estos campos:

- `org_name`
- `sector`
- `address`
- `city`
- `org_phone`
- `web`
- `supervisor_name`
- `supervisor_profession`
- `supervisor_position`
- `supervisor_department`
- `supervisor_email`
- `supervisor_phone`
- `start_date`
- `end_date`
- `schedule`
- `days`
- `modality`
- `internship_address`
- `act_description`
- `ben_description`
- `amount`

Toda edición administrativa debe incluir un motivo obligatorio no vacío.

## Campos No Editables

Por seguridad y trazabilidad, la edición administrativa no puede modificar:

- `id`
- `user_id`
- `status_id`
- `upload_date`
- `has_school_insurance`
- `internship_type`
- `internship_period`

Tampoco puede marcar como cumplida la inducción obligatoria ni modificar estados
de requisitos del estudiante. Esos flujos pertenecen a contratos separados.

## Reglas De Estado

La edición administrativa y la anulación lógica se permiten solo mientras la
práctica no esté anulada ni en estado terminal.

Estados terminales actuales:

- `Aprobada`
- `Rechazada`
- `Reprobada`

Una práctica anulada lógicamente tampoco debe volver a editarse ni anularse por
segunda vez.

## Anulación Lógica

La anulación lógica debe conservar el registro original de la práctica y dejar
trazabilidad mínima de:

- actor que anula
- motivo obligatorio
- fecha de anulación
- marca de práctica anulada

La anulación no debe borrar documentos, historial, excepciones ni datos base de
la práctica.

## Endpoints

Ambos endpoints requieren token Bearer y rol autorizado.

| Método | Ruta | Roles | Uso |
| --- | --- | --- | --- |
| `PATCH` | `/internships/{internship_id}/admin` | `Encargado de practica`, `Director de carrera` | Corrige campos editables de una práctica. |
| `POST` | `/internships/{internship_id}/cancel` | `Encargado de practica`, `Director de carrera` | Anula lógicamente una práctica. |

### Edición Administrativa

Request mínimo:

```json
{
  "reason": "Corrección de teléfono del supervisor",
  "supervisor_phone": "+56912345678"
}
```

El campo `reason` es obligatorio y no puede contener solo espacios en blanco.
La request debe incluir al menos un campo editable además de `reason`.

Respuesta: `InternshipResponse` con la práctica actualizada, incluyendo los
campos de anulación lógica (`is_cancelled`, `cancelled_at`, `cancelled_by`,
`cancellation_reason`).

### Anulación Lógica

Request:

```json
{
  "reason": "Solicitud duplicada"
}
```

Respuesta: datos mínimos de anulación lógica.

```json
{
  "id": 15,
  "is_cancelled": true,
  "cancelled_at": "2026-06-11T15:30:00",
  "cancelled_by": 2,
  "cancellation_reason": "Solicitud duplicada"
}
```

## Errores Esperados

| Código | Caso |
| --- | --- |
| `400` | Motivo vacío, edición sin campos editables, monto negativo o rango de fechas inválido. |
| `403` | Usuario autenticado sin rol autorizado. |
| `404` | La práctica solicitada no existe. |
| `409` | La práctica está en estado terminal o ya fue anulada. |
| `422` | Payload inválido o campo fuera del contrato, por ejemplo `has_school_insurance`, `status_id` o `user_id`. |

## Trazabilidad

Cada edición administrativa y cada anulación lógica debe registrar qué ocurrió,
quién lo ejecutó y por qué.

La implementación actual reutiliza `internship_status_history` sin cambiar el
estado funcional de la práctica. Para estos eventos, `previous_status_id` y
`new_status_id` quedan con el mismo estado actual, y la diferencia se registra en
`metadata`.

Para edición administrativa:

```json
{
  "action": "admin_update",
  "changed_fields": ["city", "supervisor_phone"]
}
```

Para anulación lógica:

```json
{
  "action": "cancel"
}
```

La fecha y hora del evento se guarda en `internship_status_history.changed_at`.
La anulación además queda reflejada directamente en la práctica mediante
`is_cancelled`, `cancelled_at`, `cancelled_by` y `cancellation_reason`.

## Consideración Operativa

El proyecto no usa migraciones de base de datos. Las columnas de anulación lógica
están definidas en `gestion-practicas-backend/app/core/database/init.sql` para
bases nuevas. Una base local creada antes de este cambio debe recrearse o recibir
un `ALTER TABLE` equivalente antes de probar estos endpoints.

## Riesgos

- Permitir campos fuera de la lista blanca puede saltarse reglas de negocio.
- Editar `has_school_insurance` o inducción desde este flujo puede falsear
  prerrequisitos del estudiante.
- Editar una práctica aprobada o rechazada puede invalidar decisiones ya tomadas.
- Anular físicamente registros rompería trazabilidad administrativa e histórica.
