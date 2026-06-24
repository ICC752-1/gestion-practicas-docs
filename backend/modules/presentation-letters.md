<h1 align="center"><em>Presentation Letters</em></h1>

> [!NOTE]
> Esta documentación técnica describe el comportamiento actual del módulo
> `presentation_letters` desde una perspectiva funcional e interna. El sistema no
> implementa una bandeja manual de solicitudes con estados `requested`,
> `in_progress`, `issued`, `rejected` o `cancelled`. El contrato HTTP formal queda
> en OpenAPI y el detalle interactivo queda en Swagger.

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

El módulo **`presentation_letters`** permite que un estudiante genere
automáticamente una carta de presentación para acercarse a una organización. La
carta se construye con datos reales del estudiante, el tipo de práctica y una
plantilla institucional editable por roles administrativos.

**Permite:**

- Listar plantillas de cartas disponibles para roles administrativos.
- Consultar la plantilla activa de un tipo de práctica.
- Editar plantillas institucionales.
- Generar una carta de presentación en PDF.
- Guardar metadata de cada carta generada.
- Listar cartas generadas por el estudiante.
- Descargar una carta ya generada mediante un endpoint autenticado.

El comportamiento replica el flujo institucional usado con formularios externos:
el estudiante solicita la carta al sistema y recibe una emisión inmediata si
existe plantilla activa y sus datos mínimos están disponibles.

## Ámbito y responsabilidades

El módulo **`presentation_letters`** concentra la creación y descarga de cartas de
presentación. No decide si una práctica está aprobada ni reemplaza el flujo de
documentos; solo genera un archivo institucional a partir de plantillas y datos
del estudiante.

#### Responsabilidades principales

- Mantener plantillas activas para `Práctica de Estudio I` y
  `Práctica de Estudio II`.
- Generar cartas PDF desde una plantilla DOCX institucional.
- Guardar metadata de cada carta generada.
- Guardar el archivo generado en storage privado.
- Permitir descarga autenticada de cartas.
- Notificar al estudiante cuando la generación se despacha correctamente.

#### Fuera de alcance

- Crear o aprobar solicitudes de práctica.
- Validar documentos cargados por estudiantes.
- Exponer archivos como contenido público.
- Administrar usuarios, roles o sesiones.
- Crear una solicitud manual con cola de revisión administrativa.
- Procesar estados manuales `requested`, `in_progress`, `issued`, `rejected` o
  `cancelled`.
- Enviar cartas a organizaciones externas desde la plataforma.

> [!IMPORTANT]
> Las cartas contienen datos personales del estudiante. Por eso la descarga debe
> pasar por la API y no por una ruta pública del servidor de archivos.

## Estructura interna

| Capa | Archivo | Responsabilidad |
| --- | --- | --- |
| Controller | `app/modules/presentation_letters/controllers/presentation_letter_controller.py` | Define rutas HTTP para plantillas, generación, listado y descarga. |
| Service | `app/modules/presentation_letters/services/presentation_letter_service.py` | Orquesta permisos, generación PDF, storage, notificación y descarga. |
| Repository | `app/modules/presentation_letters/repositories/presentation_letter_repository.py` | Encapsula consultas y persistencia de plantillas y cartas. |
| Models | `app/modules/presentation_letters/models/presentation_letter_model.py` | Define plantillas y cartas generadas. |
| Schemas | `app/modules/presentation_letters/schemas/presentation_letter_schema.py` | Define contratos de entrada y salida del módulo. |

El módulo reutiliza identidad y roles desde `auth`, configuración global desde
`app/core/config.py` y notificaciones mediante `notifications`.

## Funcionalidades

#### Consulta de plantillas

1. Un rol administrativo llama a `GET /presentation-letters/templates`.
2. El backend valida que el usuario tenga permisos de lectura administrativa.
3. Se retornan las plantillas activas disponibles.

También puede consultarse una plantilla específica con
`GET /presentation-letters/templates/{practice_type}`.

#### Edición de plantillas

1. Dirección de carrera llama a `PUT /presentation-letters/templates/{practice_type}`.
2. El backend valida que el tipo de práctica sea soportado.
3. Se actualizan los textos, horas mínimas, aprendizajes esperados y firma.
4. La plantilla queda disponible para futuras cartas generadas.

#### Generación de carta

1. El estudiante llama a `POST /presentation-letters/generate`.
2. El backend valida que exista una plantilla activa para el tipo de práctica.
3. Se construye la carta con datos del estudiante y contenido institucional.
4. Se renderiza una plantilla DOCX y se convierte a PDF.
5. El PDF se guarda en storage privado y se registra la carta en base de datos.
6. Si corresponde, se notifica al estudiante.

#### Listado y descarga

1. El estudiante lista sus cartas con `GET /presentation-letters/me`.
2. Para descargar, llama a `GET /presentation-letters/{letter_id}/download`.
3. El backend verifica que la carta exista y que el usuario pueda acceder.
4. Se retorna el archivo PDF mediante una respuesta autenticada.
5. La descarga queda registrada en la metadata de la carta.

## Endpoints disponibles

**Todos los endpoints requieren autenticación.**

| Método | Ruta | Propósito | Acceso principal |
| --- | --- | --- | --- |
| GET | `/presentation-letters/templates` | Lista plantillas activas. | Roles administrativos |
| GET | `/presentation-letters/templates/{practice_type}` | Consulta una plantilla por tipo de práctica. | Roles administrativos |
| PUT | `/presentation-letters/templates/{practice_type}` | Edita una plantilla institucional. | Director de carrera |
| POST | `/presentation-letters/generate` | Genera una carta PDF para el estudiante. | Estudiante |
| GET | `/presentation-letters/me` | Lista cartas generadas del estudiante. | Estudiante |
| GET | `/presentation-letters/{letter_id}/download` | Descarga una carta generada. | Propietario o rol administrativo |

## Contratos principales

<details>
<summary><strong>PresentationLetterGenerateRequest</strong></summary>

Payload mínimo para generar una carta.

```json
{
  "practice_type": "Práctica de Estudio I"
}
```

Tipos soportados: `Práctica de Estudio I` y `Práctica de Estudio II`.

</details>

<details>
<summary><strong>PresentationLetterResponse</strong></summary>

Metadata pública de una carta generada. El archivo no se entrega en este
contrato; se descarga con el endpoint correspondiente.

```json
{
  "id": 12,
  "student_id": 8,
  "practice_type": "Práctica de Estudio I",
  "template_id": 1,
  "generated_file_name": "carta-presentacion.pdf",
  "recipient_email": "estudiante@ufromail.cl",
  "sent_at": "2026-06-24T10:00:00",
  "downloaded_at": null
}
```

</details>

<details>
<summary><strong>PresentationLetterTemplateUpdateRequest</strong></summary>

Payload usado por Dirección para mantener el contenido institucional de una
plantilla. Incluye textos, horas mínimas, aprendizajes esperados y firma.

```json
{
  "title": "Carta de presentación",
  "subtitle": "Estudiante en Práctica de Estudio I",
  "minimum_hours": 168,
  "learning_outcomes": [
    "Aplicar conocimientos disciplinares en un contexto real."
  ],
  "is_active": true
}
```

El contrato real contiene más campos de texto. El detalle completo debe revisarse
en Swagger/OpenAPI.

</details>

## Reglas de negocio

#### Plantillas

**Reglas actuales:**

- Solo `Director de carrera` puede editar plantillas.
- La lectura de plantillas está disponible para roles administrativos.
- Debe existir una plantilla activa para poder generar una carta.
- Cada tipo de práctica tiene contenido institucional diferenciado.

#### Generación

**Reglas actuales:**

- Solo usuarios con rol `Estudiante` pueden generar cartas propias.
- El tipo de práctica debe ser uno de los valores soportados.
- La carta se construye con datos actuales del estudiante autenticado.
- Si falta la plantilla activa, el backend responde con error explícito
  `presentation_letter_template_not_found`.
- La generación registra `sent_at` cuando la notificación al estudiante queda
  despachada o simulada correctamente.
- La carta de presentación es opcional y no bloquea ningún hito obligatorio del
  proceso de práctica.

#### Descarga

**Reglas actuales:**

- El estudiante solo puede descargar sus propias cartas.
- Roles administrativos autorizados pueden descargar cartas cuando corresponde.
- Si el archivo físico no existe, la descarga responde error.
- La fecha de descarga se actualiza en la metadata de la carta.

> [!WARNING]
> No se debe guardar ni servir una carta generada en una ruta pública. El control
> de acceso pertenece al backend.

## Configuración por entorno

| Variable | Uso |
| --- | --- |
| `PRESENTATION_LETTER_STORAGE_DIR` | Directorio privado donde se guardan las cartas PDF generadas. |
| `PRESENTATION_LETTER_DOCX_TEMPLATE_PATH` | Ruta de la plantilla DOCX institucional usada para renderizar cartas. |
| `LIBREOFFICE_BINARY` | Binario usado para convertir DOCX a PDF. |
| `LIBREOFFICE_TIMEOUT_SECONDS` | Tiempo máximo permitido para la conversión a PDF. |

## Consideraciones operativas

- El módulo necesita una plantilla DOCX institucional disponible en el backend.
- La conversión a PDF depende de LibreOffice o del binario configurado.
- Las cartas generadas deben mantenerse en storage privado.
- Las notificaciones son efectos secundarios y pueden operar en modo simulado o real.
- Los contratos exactos de campos, validaciones y errores deben consultarse en
  Swagger/OpenAPI.
- Las pruebas unitarias documentadas están en
  `backend/tests/modules/presentation-letters-unitarias.md`.
