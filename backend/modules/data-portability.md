<h1 align="center"><em>Data Portability</em></h1>

> [!NOTE]
> Esta documentación técnica describe el comportamiento actual del módulo
> `data_portability` desde una perspectiva funcional e interna. Su objetivo es
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
- [Configuración por entorno](#configuración-por-entorno)
- [Consideraciones operativas](#consideraciones-operativas)

## Resumen operativo

El módulo **`data_portability`** permite que un estudiante exporte sus propios
datos personales registrados en el sistema. La exportación puede entregarse como
archivo JSON o como ZIP con `data.json` y documentos asociados.

**Permite:**

- Exportar perfil personal del estudiante.
- Exportar prácticas, historial, excepciones y documentos asociados.
- Exportar autoevaluaciones y evaluaciones de supervisor vinculadas a sus
  prácticas.
- Descargar solo metadata en JSON o incluir archivos documentales en un ZIP.
- Registrar auditoría funcional de cada solicitud de exportación.

En términos simples, este módulo entrega al estudiante una copia portable de su
información sin exponer secretos internos del sistema.

## Ámbito y responsabilidades

El módulo **`data_portability`** centraliza la descarga personal de datos del
estudiante. No es una herramienta administrativa para extraer datos de terceros;
es un flujo del titular de los datos.

#### Responsabilidades principales

- Validar que el actor tenga rol `Estudiante`.
- Construir una exportación minimizada de datos personales.
- Omitir secretos técnicos como hashes de contraseña, tokens o rutas internas.
- Incluir metadata documental y, opcionalmente, archivos físicos.
- Registrar estado, formato, fecha y resumen de la exportación.

#### Fuera de alcance

- Exportar datos de otros estudiantes.
- Reemplazar reportes administrativos o analíticos.
- Modificar datos durante la exportación.
- Exponer rutas internas del storage documental.
- Mantener una bandeja histórica de descargas desde la API pública.

> [!IMPORTANT]
> Este endpoint debe entenderse como portabilidad personal del estudiante. Si un
> rol administrativo necesita reportes, debe usar módulos administrativos y no
> este flujo.

## Estructura interna

| Capa | Archivo | Responsabilidad |
| --- | --- | --- |
| Controller | `app/modules/data_portability/controllers/data_portability_controller.py` | Define la ruta HTTP de exportación personal. |
| Service | `app/modules/data_portability/services/data_portability_service.py` | Orquesta autorización, construcción del payload, ZIP, JSON y auditoría. |
| Repository | `app/modules/data_portability/repositories/data_portability_repository.py` | Encapsula consultas de usuario, prácticas, documentos, evaluaciones y auditoría. |
| Models | `app/modules/data_portability/models/data_portability_model.py` | Define la auditoría de solicitudes de portabilidad. |
| Schemas | `app/modules/data_portability/schemas/data_portability_schema.py` | Define tipos y respuestas auxiliares del módulo. |

El módulo reutiliza identidad desde `auth`, prácticas desde `internships`,
documentos desde `documents`, autoevaluaciones desde `self_evaluations` y
evaluaciones externas desde `supervisor_evaluations`.

## Funcionalidades

#### Exportación JSON

1. El estudiante llama a `GET /data-portability/me/export?format=json`.
2. El backend valida que el usuario tenga rol `Estudiante`.
3. Se recolectan datos personales y datos asociados a sus prácticas.
4. Se genera un JSON minimizado.
5. Se registra auditoría de la solicitud.
6. El backend responde un archivo `.json` descargable.

Este modo es útil cuando solo se necesita una copia estructurada de la información.

#### Exportación ZIP

1. El estudiante llama a `GET /data-portability/me/export?format=zip`.
2. El backend construye el mismo `data.json` usado en la exportación JSON.
3. Si `include_documents=true`, intenta incluir los archivos documentales
   asociados.
4. Se empaqueta todo en un archivo `.zip`.
5. El backend responde el ZIP como descarga.

Este modo es útil cuando el estudiante necesita datos y archivos documentales en
una misma descarga.

#### Auditoría de solicitud

1. Al iniciar la exportación se crea un registro en `data_portability_requests`.
2. Mientras se construye la exportación, el estado queda como `processing`.
3. Si termina correctamente, queda como `completed` y guarda metadata resumida.
4. Si ocurre un error, queda como `failed` y registra el mensaje de error.

La auditoría permite saber que una exportación fue solicitada y con qué
resultado, sin guardar necesariamente el archivo generado.

## Endpoints disponibles

**Todos los endpoints requieren autenticación.**

| Método | Ruta | Propósito | Acceso principal |
| --- | --- | --- | --- |
| GET | `/data-portability/me/export` | Exporta datos personales del estudiante. | Estudiante |

Parámetros principales:

| Parámetro | Valores | Descripción |
| --- | --- | --- |
| `format` | `json`, `zip` | Define el tipo de archivo descargado. Por defecto usa `zip`. |
| `include_documents` | `true`, `false` | Define si el ZIP incluye archivos documentales. Por defecto usa `true`. |

## Contratos principales

<details>
<summary><strong>Exportación JSON</strong></summary>

Estructura general del archivo `json` o del `data.json` incluido en el ZIP.

```json
{
  "generated_at": "2026-06-24T12:00:00",
  "profile": {
    "id": 8,
    "email": "estudiante@ufromail.cl",
    "first_name": "Estudiante",
    "last_name": "Demo"
  },
  "internships": [],
  "documents": [],
  "self_evaluations": [],
  "supervisor_evaluations": [],
  "export_audit": {
    "request_id": 20,
    "format": "json",
    "include_documents": false
  }
}
```

</details>

<details>
<summary><strong>DataPortabilityAuditResponse</strong></summary>

Representa la auditoría funcional de una solicitud de exportación.

```json
{
  "request_id": 20,
  "status": "completed",
  "export_format": "zip",
  "include_documents": true,
  "result_metadata": {
    "profile": 1,
    "internships": 2,
    "documents": 4,
    "included_files": 4
  }
}
```

</details>

## Reglas de negocio

#### Autorización

**Reglas actuales:**

- Solo usuarios con rol `Estudiante` pueden exportar sus datos por este módulo.
- El estudiante exporta únicamente información asociada a su propio usuario.
- Roles administrativos no deben usar este endpoint para obtener datos de
  terceros.

#### Formatos

**Reglas actuales:**

- Los formatos soportados son `json` y `zip`.
- El JSON contiene datos estructurados y metadata documental.
- El ZIP contiene `data.json` y puede incluir archivos documentales.
- Si `include_documents=false`, no se agregan archivos físicos a la descarga.

#### Minimización

**Reglas actuales:**

- No se exporta `password_hash`.
- No se exportan tokens ni secretos internos.
- No se exponen rutas internas del servidor.
- Los documentos se referencian con rutas de exportación internas al ZIP, no con
  rutas reales del filesystem.

> [!WARNING]
> Antes de agregar nuevos campos a la exportación, se debe revisar si son datos
> personales portables o si podrían exponer secretos, datos de terceros o detalles
> internos de infraestructura.

## Configuración por entorno

| Variable | Uso |
| --- | --- |
| `DOCUMENT_STORAGE_DIR` | Directorio privado desde donde se resuelven archivos documentales al generar un ZIP. |

## Consideraciones operativas

- La exportación se genera bajo demanda y se entrega como descarga directa.
- El backend registra auditoría, pero no almacena necesariamente el archivo
  exportado.
- Si un archivo documental ya no existe físicamente, no se agrega al ZIP.
- Las rutas documentales se validan para evitar escapar del directorio de storage.
- Los contratos exactos de campos, validaciones y errores deben consultarse en
  Swagger/OpenAPI.
- Las pruebas unitarias documentadas están en
  `backend/tests/modules/data-portability-unitarias.md`.
