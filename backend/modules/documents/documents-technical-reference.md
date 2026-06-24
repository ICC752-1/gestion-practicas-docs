<h1 align="center"><em>Documents Technical Reference</em></h1>

> [!NOTE]
> Esta documentación técnica describe el comportamiento actual del módulo
> `documents` desde una perspectiva funcional e interna. Su objetivo es explicar
> cómo está implementado y qué debe saber alguien antes de modificarlo. El
> contrato HTTP formal queda en OpenAPI y el detalle interactivo queda en Swagger.

## Contenidos
- [Resumen operativo](#resumen-operativo)
- [Ámbito y responsabilidades](#ámbito-y-responsabilidades)
- [Estructura interna](#estructura-interna)
- [Funcionalidades](#funcionalidades)
- [Endpoints disponibles](#endpoints-disponibles)
- [Contratos principales](#contratos-principales)
- [Almacenamiento y metadatos](#almacenamiento-y-metadatos)
- [Reglas de negocio](#reglas-de-negocio)
- [Configuración por entorno](#configuración-por-entorno)
- [Consideraciones operativas](#consideraciones-operativas)
- [Documentación relacionada](#documentación-relacionada)

## Resumen operativo

El módulo **`documents`** gestiona documentos asociados a prácticas. Permite
cargar archivos, listar metadatos documentales, descargar archivos mediante un
endpoint autenticado, revisar documentos, eliminarlos lógicamente y construir el
paquete documental requerido para exportación DIRAE.

**Permite:**

- Listar tipos documentales activos.
- Cargar documentos asociados a una práctica.
- Listar documentos vigentes de una práctica.
- Descargar archivos con autorización previa.
- Restringir documentos sensibles para roles con acceso parcial.
- Aprobar u observar documentos durante revisión administrativa.
- Eliminar documentos de forma lógica.
- Construir el paquete documental de una práctica.
- Exportar paquetes documentales DIRAE en CSV resumen y CSV detalle.
- Notificar cargas y cambios de estado documental.

> [!IMPORTANT]
> Los archivos físicos no se guardan en la base de datos. La base de datos guarda
> metadatos y una clave interna `file_path` que permite ubicar el archivo dentro
> del storage privado.

## Ámbito y responsabilidades

El módulo **`documents`** concentra la gestión documental del proceso de práctica.
Su responsabilidad principal es controlar el ciclo de vida del documento y su
disponibilidad para revisión o exportación, no validar el contenido institucional
del archivo más allá de tamaño, extensión, estado y tipo documental.

#### Responsabilidades principales

- Gestión de tipos documentales activos.
- Carga de archivos en filesystem privado.
- Persistencia de metadatos documentales.
- Control de acceso a listados y descargas.
- Control de acceso a documentos sensibles.
- Revisión documental mediante estados simples.
- Eliminación lógica de documentos.
- Construcción del paquete documental por práctica.
- Exportación CSV resumen y detalle de paquetes DIRAE.
- Emisión de notificaciones documentales.

#### Fuera de alcance

- Creación, aprobación o rechazo de prácticas.
- Administración de usuarios y roles.
- Publicación de archivos como contenido estático.
- Borrado físico automático de archivos.
- Política institucional final de retención documental.
- Validación semántica del contenido del archivo subido.

> [!IMPORTANT]
> La privacidad documental depende de que las descargas pasen por la API. El
> directorio definido por `DOCUMENT_STORAGE_DIR` no debe exponerse como ruta
> pública del frontend ni de Nginx.

## Estructura interna

| Capa | Archivo | Responsabilidad |
| --- | --- | --- |
| Controller | `app/modules/documents/controllers/document_controller.py` | Define rutas HTTP, dependencias, multipart upload, descarga y roles documentales. |
| Service | `app/modules/documents/services/document_service.py` | Orquesta reglas de carga, permisos, revisión, eliminación, paquetes DIRAE, storage y notificaciones. |
| Repository | `app/modules/documents/repositories/document_repository.py` | Encapsula consultas y persistencia de documentos, tipos documentales, prácticas y usuarios revisores. |
| Schemas | `app/modules/documents/schemas/document_schema.py` | Define contratos HTTP para tipos, documentos, revisión y paquetes documentales. |
| Models | `app/modules/documents/models/document_model.py` | Define `DocumentType`, `Document` y enums documentales. |
| Docs | `docs/modules/documents/documents-storage-privacy.md` | Documenta privacidad, storage en VPS, retención y respaldos. |

El módulo reutiliza prácticas desde `internships`, identidad y roles desde `auth`,
configuración desde `app/core/config.py` y notificaciones mediante
`NotificationService`.

## Funcionalidades

#### Listado de tipos documentales

1. El usuario llama a `GET /documents/types`.
2. El controller exige Bearer token válido.
3. El service solicita tipos documentales activos al repository.
4. Se retornan elementos `DocumentTypeResponse`.

Los tipos documentales seed actuales son:

| Tipo documental | Requerido | Categoría |
| --- | --- | --- |
| `Formulario de inscripción` | No | `Académico`. |
| `Carta de aceptación` | No | `Administrativo`. |
| `Seguro escolar` | No | `Administrativo`, sensible. |
| `Documento complementario` | No | `Administrativo`. |
| `Diapositivas de Presentación` | No | `Académico`. |

#### Carga documental

1. El propietario de la práctica llama a `POST /internships/{internship_id}/documents`.
2. Envía `document_type_id` y `file` mediante multipart form.
3. El service verifica que la práctica exista.
4. Se exige que el usuario autenticado sea propietario de la práctica o `Secretaria de Carrera` en los casos administrativos permitidos.
5. Se bloquea la carga si la práctica está en estado terminal, salvo correcciones permitidas.
6. Se valida que el tipo documental exista y esté activo.
7. Se normaliza el nombre original del archivo.
8. Se validan extensión y tamaño.
9. Se escribe el archivo físico en storage privado.
10. Se persisten metadatos con estado `uploaded`.
11. Se intenta notificar a los roles documentales.
12. Se recalcula el estado DIRAE local de la práctica si corresponde.

En prácticas ya `Aprobada`, el estudiante solo puede subir `Diapositivas de Presentación` o reemplazar un tipo documental que tenga una observación vigente. `Secretaria de Carrera` puede cargar correcciones administrativas no sensibles solo cuando la práctica ya está aprobada.

#### Listado y descarga

1. El usuario llama a `GET /internships/{internship_id}/documents` o `GET /documents/{document_id}/download`.
2. El service valida que la práctica o documento exista.
3. Se permite acceso al propietario de la práctica o a un rol documental.
4. Para `Secretaria de Carrera`, los documentos sensibles se filtran del listado y no se pueden descargar.
5. Los documentos eliminados lógicamente no se listan ni descargan.
6. La descarga resuelve `file_path` contra `DOCUMENT_STORAGE_DIR` y retorna `FileResponse`.

#### Revisión documental

1. Un rol documental llama a `PATCH /documents/{document_id}/status`.
2. El payload indica `status` y, opcionalmente, `comment`.
3. El service acepta solo `observed` o `approved`.
4. Si el estado destino es `observed`, el comentario es obligatorio.
5. Se registran `reviewed_at`, `reviewed_by`, `review_comment` y `update_date`.
6. Se intenta notificar al estudiante.
7. Se recalcula el estado DIRAE local de la práctica si corresponde.

#### Eliminación lógica

1. El usuario llama a `DELETE /documents/{document_id}`.
2. El service verifica que el documento exista y no esté eliminado.
3. Un rol documental puede eliminar documentos en cualquier estado vigente.
4. El propietario puede eliminar documentos propios, salvo documentos ya `approved`.
5. Se marca el documento como `deleted` y se registran `deleted_at`, `deleted_by` y `update_date`.
6. El archivo físico no se borra automáticamente.
7. Se recalcula el estado DIRAE local de la práctica si corresponde.

#### Paquete documental

1. El usuario llama a `GET /internships/{internship_id}/documents/package`.
2. El service valida acceso como propietario o rol documental.
3. Se cargan los tipos documentales requeridos y documentos vigentes de la práctica.
4. Si el actor es `Secretaria de Carrera`, se excluyen documentos y tipos documentales sensibles.
5. Para cada tipo requerido se selecciona el último documento `approved`.
6. Se informan documentos requeridos aprobados, requeridos faltantes y documentos opcionales aprobados.
7. Se calcula `exportable` y sus `reasons`.

> [!NOTE]
> El paquete documental recalcula el estado DIRAE local antes de responder. Ese
> estado indica preparación interna del expediente; no confirma recepción externa
> por DIRAE.

#### Exportación DIRAE

1. Un rol documental llama a `GET /dirae/document-packages/export`.
2. Opcionalmente envía `internship_ids` como query repetible.
3. El service valida permisos documentales.
4. Si se solicitaron IDs concretos, verifica que existan.
5. Construye paquetes documentales y conserva solo los exportables.
6. Si se pidieron IDs concretos y alguno no es exportable, responde `409` con razones.
7. Si no se pidieron IDs, exporta todos los paquetes exportables disponibles.
8. Retorna un CSV resumen con nombre `dirae_lote_<timestamp>_<lote>.csv`.
9. Marca las prácticas exportadas con `dirae_status=exported` y registra historial DIRAE.

El endpoint `GET /dirae/document-packages/export/detail` usa la misma selección de paquetes exportables, pero retorna un CSV de detalle con una fila por documento aprobado incluido en el paquete. El archivo se nombra como `dirae_lote_<timestamp>_<lote>_detalle.csv`.

## Endpoints disponibles

**Todos los endpoints requieren autenticación.**

| Método | Ruta | Propósito | Acceso |
| --- | --- | --- | --- |
| GET | `/documents/types` | Lista tipos documentales activos. | Bearer token |
| POST | `/internships/{internship_id}/documents` | Carga un documento para una práctica. | Propietario o Secretaría en casos permitidos |
| GET | `/internships/{internship_id}/documents` | Lista documentos vigentes de una práctica. | Propietario o rol documental |
| GET | `/internships/{internship_id}/documents/package` | Resume paquete documental y exportabilidad DIRAE. | Propietario o rol documental |
| GET | `/dirae/document-packages/export` | Exporta CSV de paquetes documentales DIRAE. | Rol documental |
| GET | `/dirae/document-packages/export/detail` | Exporta CSV de detalle documental por documento aprobado. | Rol documental |
| GET | `/documents/{document_id}/download` | Descarga un archivo mediante endpoint autenticado. | Propietario o rol documental |
| PATCH | `/documents/{document_id}/status` | Aprueba u observa un documento. | Rol documental |
| DELETE | `/documents/{document_id}` | Elimina lógicamente un documento. | Propietario o rol documental |

Roles documentales:

| Rol | Uso principal |
| --- | --- |
| `Encargado de practica` | Consulta, revisión, eliminación administrativa y exportación DIRAE. |
| `Director de carrera` | Consulta, revisión, eliminación administrativa y exportación DIRAE. |
| `Secretaria de Carrera` | Consulta, revisión, eliminación administrativa y exportación DIRAE. |

## Contratos principales

<details>
<summary><strong>DocumentTypeResponse</strong></summary>

Representa un tipo documental activo disponible para carga.

```json
{
  "id": 1,
  "name": "Formulario de inscripción",
  "description": "Formulario de inscripción de práctica firmado o respaldado.",
  "is_required": true,
  "category": "Académico",
  "is_sensitive": false,
  "is_active": true
}
```

</details>

<details>
<summary><strong>DocumentResponse</strong></summary>

Representa metadatos públicos de un documento. No incluye `file_path` porque esa
clave es interna del storage privado.

```json
{
  "id": 55,
  "file_name": "formulario.pdf",
  "extension": "pdf",
  "status": "uploaded",
  "size_bytes": 204800,
  "upload_date": "2026-06-16T10:00:00Z",
  "update_date": "2026-06-16T10:00:00Z",
  "internship_id": 7,
  "type_id": 1,
  "user_id": 10,
  "reviewed_at": null,
  "reviewed_by": null,
  "review_comment": null,
  "deleted_at": null,
  "deleted_by": null,
  "document_type": {
    "id": 1,
    "name": "Formulario de inscripción",
    "description": "Formulario de inscripción de práctica firmado o respaldado.",
    "is_required": true,
    "category": "Académico",
    "is_sensitive": false,
    "is_active": true
  }
}
```

</details>

<details>
<summary><strong>DocumentStatusUpdateRequest</strong></summary>

Payload usado para aprobar u observar un documento. `comment` es obligatorio
cuando `status=observed`.

```json
{
  "status": "observed",
  "comment": "Falta firma del supervisor."
}
```

</details>

<details>
<summary><strong>DocumentPackageResponse</strong></summary>

Resume el estado documental de una práctica y si su expediente local puede
exportarse para trámite externo en DIRAE.

```json
{
  "internship_id": 7,
  "status": "Aprobada",
  "dirae_status": "ready",
  "exportable": true,
  "reasons": [],
  "student": {
    "id": 10,
    "rut": "12.345.678-9",
    "enrollment": "12345678923",
    "first_name": "Juan",
    "last_name": "Perez",
    "email": "juan.perez@correo.cl",
    "degree": "Ingenieria Civil Informatica",
    "cod_degree": "INF-001"
  },
  "internship": {
    "type": "Práctica de Estudio I",
    "period": "Semestre",
    "organization": "Empresa Demo SpA",
    "city": "Temuco",
    "start_date": "2026-06-01",
    "end_date": "2026-08-31"
  },
  "required_documents": [
    {
      "type_id": 1,
      "type_name": "Formulario de inscripción",
      "status": "approved",
      "document": {
        "id": 55,
        "file_name": "formulario.pdf",
        "extension": "pdf",
        "status": "approved",
        "size_bytes": 204800,
        "upload_date": "2026-06-16T10:00:00Z",
        "update_date": "2026-06-16T12:00:00Z",
        "internship_id": 7,
        "type_id": 1,
        "user_id": 10,
        "reviewed_at": "2026-06-16T12:00:00Z",
        "reviewed_by": 99,
        "review_comment": null,
        "deleted_at": null,
        "deleted_by": null,
        "document_type": null
      }
    }
  ],
  "optional_documents": []
}
```

</details>

## Almacenamiento y metadatos

El módulo separa el contenido físico del archivo y sus metadatos. El archivo se
guarda en filesystem privado y la base de datos conserva la información necesaria
para ubicarlo, auditarlo y controlar su ciclo de vida.

| Elemento | Dónde vive | Uso |
| --- | --- | --- |
| Archivo físico | `DOCUMENT_STORAGE_DIR` | Contenido real del documento. |
| `Document.file_path` | Base de datos | Clave interna para ubicar el archivo dentro del storage. |
| `Document.file_name` | Base de datos | Nombre original normalizado para descarga. |
| `Document.extension` | Base de datos | Extensión validada contra configuración permitida. |
| `Document.status` | Base de datos | Estado documental actual. |
| `Document.size_bytes` | Base de datos | Tamaño validado del archivo. |
| `Document.reviewed_at`, `Document.reviewed_by`, `Document.review_comment` | Base de datos | Trazabilidad de revisión documental. |
| `Document.deleted_at`, `Document.deleted_by` | Base de datos | Trazabilidad de eliminación lógica. |

La clave de storage se genera con el formato:

```text
{internship_id}/{uuid}.{extension}
```

Ejemplo:

```text
7/4f2a9d5c4d0a4a5f9c1f2d3e4b5a6c7d.pdf
```

> [!IMPORTANT]
> `file_path` no es una URL pública ni debe entregarse al frontend. El frontend
> recibe metadatos y descarga archivos mediante `GET /documents/{document_id}/download`.

El service valida que la clave de storage no sea absoluta y no contenga `..` antes
de resolverla contra `DOCUMENT_STORAGE_DIR`.

## Reglas de negocio

#### Estados documentales

| Estado | Uso |
| --- | --- |
| `uploaded` | Documento cargado y pendiente de revisión. |
| `observed` | Documento revisado con observaciones. |
| `approved` | Documento revisado y aceptado para paquete documental. |
| `deleted` | Documento eliminado lógicamente. |

#### Extensiones y categorías

| Tipo | Valores |
| --- | --- |
| Extensiones soportadas por modelo | `pdf`, `docx`, `jpg`, `png`, `zip`. |
| Categorías documentales | `Académico`, `Administrativo`. |

> [!NOTE]
> Las extensiones efectivamente permitidas se leen desde
> `DOCUMENT_ALLOWED_EXTENSIONS`. El enum del modelo define el conjunto soportado
> por la base de datos.

#### Estados de práctica que bloquean carga

| Estado de práctica | Efecto |
| --- | --- |
| `Aprobada` | Bloquea cargas generales, pero permite `Diapositivas de Presentación`, correcciones de tipos observados y correcciones administrativas no sensibles por Secretaría. |
| `Rechazada` | Bloquea nuevas cargas documentales. |
| `Reprobada` | Bloquea nuevas cargas documentales. |

#### Permisos

| Operación | Quién puede ejecutarla |
| --- | --- |
| Subir documento | Propietario de la práctica; `Secretaria de Carrera` solo para correcciones administrativas no sensibles en práctica aprobada. |
| Listar documentos | Propietario o rol documental; `Secretaria de Carrera` no ve documentos sensibles. |
| Descargar documento | Propietario o rol documental; `Secretaria de Carrera` no descarga documentos sensibles. |
| Aprobar u observar | Rol documental. |
| Eliminar documento no aprobado | Propietario o rol documental. |
| Eliminar documento aprobado | Rol documental. |
| Consultar paquete documental | Propietario o rol documental; `Secretaria de Carrera` recibe paquete sin documentos/tipos sensibles. |
| Exportar CSV DIRAE | Rol documental. |

Documentos sensibles:

- Un tipo documental es sensible cuando `DocumentType.is_sensitive=true`.
- El seed actual marca `Seguro escolar` como sensible.
- `Secretaria de Carrera` mantiene acceso documental general, pero el service filtra tipos y documentos sensibles.
- Si el paquete contiene o requiere antecedentes sensibles no visibles para el actor, agrega la razón `sensitive_document_restricted`.

#### Revisión documental

- `observed` requiere comentario no vacío.
- `approved` no requiere comentario.
- La revisión actualiza `reviewed_at`, `reviewed_by`, `review_comment` y `update_date`.
- La revisión puede gatillar una transición automática del estado DIRAE local si cambia la preparación del paquete.

#### Eliminación lógica

- La eliminación marca `status=deleted`.
- La eliminación registra `deleted_at`, `deleted_by` y `update_date`.
- Los documentos eliminados no se listan ni descargan por API.
- No existe borrado físico automático del archivo.
- La eliminación puede gatillar una transición automática del estado DIRAE local si deja incompleto el paquete.

#### Paquete documental DIRAE

Un paquete es exportable cuando cumple estas condiciones:

- La solicitud de práctica está en estado `Aprobada`.
- La práctica está cerrada con `completion_status=finalized`.
- El expediente documental local está preparado para exportación. En la
  implementación actual se representa con `dirae_status=ready` o `dirae_status=exported`,
  sin implicar por sí solo estado externo en DIRAE.
- Todos los tipos documentales requeridos tienen un documento `approved` vigente.
- No hay documentos observados vigentes pendientes de corrección.
- El actor puede ver todos los documentos/tipos necesarios para evaluar el paquete.

Razones de no exportabilidad:

| Razón | Significado |
| --- | --- |
| `internship_not_approved` | La solicitud de práctica no está en estado `Aprobada`. |
| `practice_not_finalized` | La práctica aún no está cerrada/finalizada. |
| `dirae_not_ready` | El expediente local todavía no está listo para exportación. |
| `missing_required_documents` | Falta al menos un documento requerido aprobado. |
| `observed_documents_pending` | Existe al menos un documento observado pendiente de corrección. |
| `sensitive_document_restricted` | El actor no tiene permiso para incluir documentos sensibles en el paquete. |

Para cada tipo documental, el paquete selecciona el último documento aprobado
según `upload_date` y luego `id`. Documentos `uploaded`, `observed`, `deleted` o
con `deleted_at` no se consideran aprobados para el paquete.

#### Transición automática DIRAE local

El módulo recalcula `Internship.dirae_status` cuando se carga, revisa o elimina un documento, cuando se consulta el paquete documental y cuando se prepara una exportación DIRAE. La transición automática solo se aplica si la práctica está `completion_status=finalized` y el estado actual está dentro de `not_started`, `in_review`, `observed`, `ready` o `exported`.

Reglas actuales:

| Condición documental | Estado DIRAE local destino |
| --- | --- |
| Hay documentos observados vigentes | `observed` |
| Faltan documentos requeridos aprobados | `in_review` |
| Hay documentos cargados pendientes de revisión | `in_review` |
| No faltan requeridos, no hay observados y no hay cargados pendientes | `ready` |

Cada cambio automático registra historial DIRAE con la razón `Transición automática por cambio en estado de documentos.`.

#### Exportación CSV DIRAE

- Con `internship_ids`, todos los IDs solicitados deben existir.
- Con `internship_ids`, cualquier práctica no exportable produce `409` con razones.
- Sin `internship_ids`, se exportan solo las prácticas exportables disponibles.
- Sin paquetes exportables, el CSV puede contener solo encabezado.
- El service construye un evento `dirae_export_generated` como dato de auditoría de la exportación.
- Cuando se generan filas exportables, el repository marca esas prácticas como `dirae_status=exported` con razón `dirae_document_package_exported`.
- La exportación resumen incluye una fila por práctica exportada.
- La exportación detalle incluye una fila por documento aprobado incluido en el paquete.
- El contrato actual solo genera descarga CSV. No envía correo con archivo
  adjunto; para soportarlo habría que extender el módulo `notifications` con
  adjuntos, destinatario institucional configurable y auditoría de envío.

## Configuración por entorno

| Variable | Uso |
| --- | --- |
| `DOCUMENT_STORAGE_DIR` | Directorio base donde se guardan archivos físicos. |
| `DOCUMENT_MAX_BYTES` | Tamaño máximo permitido por archivo. |
| `DOCUMENT_ALLOWED_EXTENSIONS` | Lista de extensiones permitidas para carga. |
| `DOCUMENT_RETENTION_DAYS` | Política declarativa de retención. Actualmente no activa borrado automático. |

Variables indirectas por notificaciones:

| Variable | Impacto |
| --- | --- |
| `NOTIFICATION_MODE` | Define si las notificaciones documentales se simulan o se despachan con configuración real. |
| `MAIL_*` | Requeridas por el módulo `notifications` cuando `NOTIFICATION_MODE=real`. |

> [!NOTE]
> Las notificaciones documentales son efectos secundarios no bloqueantes. Si falla
> el envío o el servicio no está configurado, el flujo principal continúa.

## Consideraciones operativas

- La base de datos y el directorio documental deben respaldarse juntos.
- Si existen metadatos pero falta el archivo físico, la descarga responde `404`.
- Si falla la persistencia después de escribir el archivo, el service intenta eliminar el archivo recién escrito.
- `DocumentResponse` no debe exponer `file_path`.
- `DocumentTypeResponse` expone `is_sensitive` para que el frontend pueda tratar tipos reservados de forma explícita.
- `DOCUMENT_STORAGE_DIR` no debe servirse como contenido estático.
- El CSV DIRAE resumen usa los encabezados definidos en `DIRAE_CSV_HEADER`.
- El CSV DIRAE detalle usa los encabezados definidos en `DIRAE_CSV_DETAIL_HEADER`.
- La eliminación lógica mantiene trazabilidad pero no libera espacio en disco.
- La limpieza física futura debe preservar trazabilidad y responder a una regla institucional explícita.

## Documentación relacionada

- `docs/modules/documents/documents-storage-privacy.md`: Detalla estrategia de almacenamiento, privacidad, retención, operación en VPS y verificaciones esperadas.
- `docs/modules/internships/internships-technical-reference.md`: Explica el ciclo de vida de prácticas que condiciona carga documental y exportabilidad DIRAE.
- `docs/modules/auth.md`: Explica autenticación, roles y dependencias usadas por los endpoints documentales.
