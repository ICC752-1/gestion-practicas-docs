<h1 align="center"><em>Admin</em></h1>

> [!NOTE]
> Esta documentación técnica describe el comportamiento actual del módulo `admin`
> desde una perspectiva funcional e interna. Su objetivo es explicar cómo está
> implementado y qué debe saber alguien antes de modificarlo. El contrato HTTP
> formal queda en OpenAPI y el detalle interactivo queda en Swagger.

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

El módulo **`admin`** expone operaciones administrativas para consultar el estado
general del sistema, revisar estudiantes, listar prácticas registradas y gestionar
requisitos asociados al proceso de práctica.

**Permite:**

- Ver un resumen global del sistema.
- Listar estudiantes registrados.
- Listar y consultar prácticas desde una vista administrativa.
- Revisar requisitos académicos de práctica por estudiante.
- Actualizar el estado de requisitos de práctica.
- Registrar y consultar validaciones de seguro escolar por solicitud de
  práctica.
- Mantener el requisito institucional histórico de seguro escolar del
  estudiante.

## Ámbito y responsabilidades

El módulo **`admin`** centraliza operaciones de _lectura_ y _gestión administrativa_.
No es el dueño del flujo completo de **creación** ni **aprobación** de prácticas.

#### Responsabilidades principales

- Agregaciones para dashboard administrativo.
- Consulta administrativa de estudiantes.
- Consulta administrativa de prácticas.
- Gestión de requisitos académicos de práctica.
- Gestión de seguro escolar por solicitud concreta.
- Gestión del prerrequisito institucional histórico de seguro escolar.

#### Fuera de alcance

- Creación de prácticas.
- Acciones principales del flujo de práctica como aprobar, rechazar o derivar.
- Autenticación de usuarios.
- Carga y revisión documental.

> [!IMPORTANT]
> Los endpoints `/internships/{id}/approve`, `/internships/{id}/reject` y
> `/internships/{id}/derive` pertenecen al módulo `internships`, aunque sean usados
> por actores administrativos.

## Estructura interna

| Capa | Archivo | Responsabilidad |
| --- | --- | --- |
| Controller | `app/modules/admin/controllers/admin_controller.py` | Define rutas HTTP, dependencias y roles requeridos. |
| Service | `app/modules/admin/services/admin_service.py` | Orquesta reglas administrativas y validaciones. |
| Repository | `app/modules/admin/repositories/admin_repository.py` | Encapsula consultas administrativas a base de datos. |
| Schemas | `app/modules/admin/schemas/admin_schema.py` | Define contratos de entrada y salida del módulo. |

El módulo reutiliza modelos definidos en otros dominios, principalmente
**usuarios**, **roles**, **prácticas** y **requisitos de inscripción**.

## Funcionalidades

#### Consulta del dashboard administrativo

1. El cliente llama a `GET /admin/summary`.
2. El controller exige autenticación y rol `Encargado de practica` o
   `Director de carrera`.
3. El service solicita agregaciones al repository.
4. El repository calcula totales y conteos por estado.
5. Se retorna `AdminSummaryResponse`.

#### Listado administrativo de prácticas

1. El cliente llama a `GET /admin/internships`.
2. Opcionalmente envía el query `status`.
3. El controller valida autenticación y rol.
4. El service normaliza el filtro funcional.
5. El repository lista prácticas junto a estudiante y estado actual.
6. Se retorna una lista de `AdminInternshipListItem`.

#### Actualización de requisito académico

1. El cliente llama a `PATCH /admin/students/{student_id}/internship-requirements/{requirement_id}/status`.
2. El controller valida autenticación y rol `Encargado de practica` o
   `Director de carrera`.
3. El service verifica que el requisito exista y pertenezca al estudiante.
4. Se valida la transición de estado.
5. Se actualizan `status`, `status_updated_at` y `status_updated_by`.
6. Se retorna el requisito actualizado.

#### Registro de seguro escolar

1. El cliente llama a `PATCH /admin/internships/{internship_id}/school-insurance`.
2. El controller permite solo `Director de carrera`.
3. El service verifica que la solicitud exista.
4. Se actualiza `insurance_status`, `insurance_validated_by`,
   `insurance_validated_at`, `insurance_notes` y la compatibilidad
   `has_school_insurance`.
5. Se retorna `AdminInternshipDetailResponse`.

#### Registro institucional histórico de seguro escolar

1. El cliente llama a `PATCH /admin/students/{student_id}/registration-requirements/school-insurance`.
2. El controller permite solo `Director de carrera`.
3. El service verifica que el usuario exista y tenga rol `Estudiante`.
4. Se crea o actualiza el requisito `school_insurance`.
5. Si `is_completed=false`, se limpia `completed_at`.
6. Se retorna `AdminRegistrationRequirementItem`.

> [!NOTE]
> Este requisito institucional histórico sirve como dato diagnóstico y de
> compatibilidad. Para aprobar solicitudes fuera del periodo regular, la fuente
> autoritativa es `insurance_status` de la solicitud concreta.

## Endpoints disponibles

**Todos los endpoints requieren autenticación.**

| Método | Ruta | Propósito | Rol |
| --- | --- | --- | --- |
| GET | `/admin/summary` | Resumen global del sistema. | Encargado de practica, Director de carrera |
| GET | `/admin/students` | Listado administrativo de estudiantes. | Encargado de practica, Director de carrera |
| GET | `/admin/internships` | Listado administrativo de prácticas. | Encargado de practica, Director de carrera |
| GET | `/admin/internships/{internship_id}` | Detalle administrativo de una práctica. | Encargado de practica, Director de carrera |
| PATCH | `/admin/internships/{internship_id}/school-insurance` | Valida o marca el seguro escolar de una solicitud concreta. | Director de carrera |
| GET | `/admin/students/{student_id}/internship-requirements` | Lista requisitos académicos del estudiante. | Encargado de practica, Director de carrera |
| PATCH | `/admin/students/{student_id}/internship-requirements/{requirement_id}/status` | Actualiza estado de requisito académico. | Encargado de practica, Director de carrera |
| GET | `/admin/students/{student_id}/registration-requirements` | Lista prerrequisitos institucionales. | Director de carrera |
| PATCH | `/admin/students/{student_id}/registration-requirements/school-insurance` | Crea o actualiza seguro escolar institucional histórico. | Director de carrera |

## Contratos principales

<details>
<summary><strong>AdminSummaryResponse</strong></summary>

Resume métricas globales para _dashboard administrativo_.

```json
{
  "total_students": 120,
  "total_internships": 45,
  "internships_by_status": [
    {
      "status": "Pendiente",
      "total": 10
    },
    {
      "status": "Aprobada",
      "total": 12
    }
  ]
}
```

</details>

<details>
<summary><strong>AdminInternshipListItem</strong></summary>

Representa una práctica en _listados administrativos_.

```json
{
  "id": 15,
  "org_name": "Empresa X",
  "city": "Valdivia",
  "start_date": "2026-05-01",
  "end_date": "2026-07-30",
  "upload_date": "2026-04-15T10:00:00Z",
  "user_id": 1,
  "student": {
    "id": 1,
    "email": "student@correo.cl",
    "first_name": "Juan",
    "last_name": "Pérez",
    "rut": "12.345.678-9",
    "degree": "Ingeniería Civil Informática",
    "cod_degree": "ICI"
  },
  "status": {
    "id": 2,
    "title": "En revisión",
    "description": "La solicitud de práctica fue registrada y se encuentra en revisión administrativa."
  }
}
```

</details>

<details>
<summary><strong>AdminUpdateInternshipSchoolInsuranceRequest</strong></summary>

Request usado por Dirección de carrera para validar el seguro de una solicitud
concreta.

```json
{
  "status": "validated",
  "notes": "Validado por Dirección de carrera para esta solicitud."
}
```

Estados aceptados: `pending`, `validated`, `requires_exception`,
`not_applicable`.

</details>

<details>
<summary><strong>AdminUpdateStudentInternshipRequirementStatusRequest</strong></summary>

Request usado para cambiar el estado de un **requisito académico**.

```json
{
  "status": "En revisión"
}
```

</details>

<details>
<summary><strong>AdminRegistrationRequirementItem</strong></summary>

Representa un **prerrequisito institucional** del estudiante.

```json
{
  "id": 8,
  "user_id": 1,
  "requirement": "school_insurance",
  "is_completed": true,
  "completed_at": "2026-06-12T18:30:00Z",
  "updated_by": 5
}
```

</details>

## Reglas de negocio

#### Filtros de prácticas

`GET /admin/internships` acepta el query opcional **`status`**.

| Query | Estados incluidos |
| --- | --- |
| `submitted` | Sin estado o `Pendiente`. |
| `in_review` | `En revisión`. Registros legacy con `En revisión DIRAE` pueden mapearse aquí solo por compatibilidad histórica; el flujo actual no debe usarlo como estado funcional. |
| `approved` | `Aprobada`. |
| `rejected` | `Rechazada`, `Reprobada`. |

#### Estados de requisitos académicos

**Estados posibles:**

- `Pendiente`
- `Habilitada`
- `En revisión`
- `Aprobada`
- `Rechazada`

**Transiciones permitidas:**

| Origen | Destino |
| --- | --- |
| `Pendiente` | `Habilitada` |
| `Habilitada` | `En revisión` |
| `En revisión` | `Aprobada` |
| `En revisión` | `Rechazada` |
| `Rechazada` | `Habilitada` |

> [!WARNING]
> Las transiciones inválidas retornan error `400`.

#### Seguro escolar por solicitud

El seguro escolar que condiciona la aprobación se maneja por solicitud concreta
mediante `insurance_status`.

| Valor | Significado |
| --- | --- |
| `pending` | Pendiente de validación. |
| `validated` | Validado para la solicitud concreta. |
| `requires_exception` | Requiere regularización o excepción. |
| `not_applicable` | No aplica para la solicitud. |

**Reglas actuales:**

- Solo `Director de carrera` puede actualizar el seguro por solicitud.
- Las solicitudes dentro de marzo-junio o agosto-noviembre pueden quedar
  validadas automáticamente por backend.
- Las solicitudes fuera de esos rangos requieren validación explícita o
  excepción `school_insurance` antes de llegar a `Aprobada`.
- No bloquea la creación de una solicitud en estado `Pendiente`.

#### Seguro escolar institucional histórico

El requisito institucional de seguro escolar del estudiante se maneja como
booleano de apoyo diagnóstico.

| Valor | Significado |
| --- | --- |
| `is_completed=true` | Existe cobertura registrada. |
| `is_completed=false` | No existe cobertura vigente registrada. |

**Reglas actuales:**

- Se registra mediante operación tipo `upsert`.
- Si no existe, se crea.
- Si existe, se actualiza.
- Al enviar `false`, se limpia `completed_at`.
- No sustituye `insurance_status` de una solicitud concreta fuera del periodo
  regular.

## Consideraciones operativas

- Todos los endpoints exigen autenticación.
- Los endpoints generales del módulo requieren rol `Encargado de practica`.
- Los endpoints de lectura administrativa de prácticas también aceptan
  `Director de carrera`.
- Los endpoints de seguro escolar requieren `Director de carrera`.
- El cambio de estado de requisito registra `status_updated_at` y `status_updated_by`.
- Si el requisito académico no existe para el estudiante indicado, se retorna `404`.
- El `PATCH` de seguro escolar retorna `404` si el usuario no existe o no tiene rol `Estudiante`.
- El dashboard coordinador debería consumir `/admin/summary`, `/admin/internships` y `/admin/internships/{internship_id}` para vistas administrativas.
