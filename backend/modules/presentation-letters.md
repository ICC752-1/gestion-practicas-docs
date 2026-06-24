<h1 align="center"><em>Presentation Letters</em></h1>

> [!NOTE]
> Esta documentación describe el flujo vigente de cartas de presentación. El
> sistema no implementa una bandeja manual de solicitudes con estados
> `requested`, `in_progress`, `issued`, `rejected` o `cancelled`.

## Resumen operativo

El módulo **`presentation_letters`** permite que un estudiante genere
automáticamente una carta de presentación para acercarse a una organización. La
carta se construye con datos reales del estudiante, el tipo de práctica y una
plantilla institucional editable por roles administrativos.

El comportamiento replica el flujo institucional usado con formularios externos:
el estudiante solicita la carta al sistema y recibe una emisión inmediata si
existe plantilla activa y sus datos mínimos están disponibles.

## Alcance

**Permite:**

- Mantener plantillas institucionales por tipo de práctica.
- Generar una carta PDF automáticamente para el estudiante autenticado.
- Guardar metadata de la carta generada.
- Notificar al estudiante cuando la carta queda generada.
- Listar cartas propias.
- Descargar cartas propias o cartas por rol administrativo autorizado.

**No permite:**

- Crear una solicitud manual con cola de revisión administrativa.
- Procesar estados manuales `requested`, `in_progress`, `issued`, `rejected` o
  `cancelled`.
- Bloquear la inducción, la solicitud de práctica o la aprobación de una
  solicitud por no tener carta.
- Enviar cartas a organizaciones externas desde la plataforma.

## Reglas de negocio

1. La carta de presentación es opcional y no bloquea ningún hito obligatorio del
   proceso de práctica.
2. La carta se genera automáticamente desde una plantilla activa para el tipo de
   práctica solicitado.
3. Los tipos soportados actualmente son `Práctica de Estudio I` y
   `Práctica de Estudio II`.
4. Si no existe plantilla activa para el tipo solicitado, el backend responde con
   error explícito `presentation_letter_template_not_found`.
5. La descarga respeta propiedad: el estudiante solo descarga cartas propias; un
   rol administrativo autorizado puede descargar cartas para gestión interna.
6. La generación registra `sent_at` cuando la notificación al estudiante queda
   despachada o simulada correctamente.
7. La carta contiene datos personales y académicos del estudiante; no debe
   exponerse en listados públicos ni por enlaces anónimos.

## Endpoints

| Método | Ruta | Propósito | Acceso |
| --- | --- | --- | --- |
| GET | `/presentation-letters/templates` | Lista plantillas de cartas. | Rol administrativo autorizado |
| GET | `/presentation-letters/templates/{practice_type}` | Obtiene plantilla por tipo de práctica. | Rol administrativo autorizado |
| PUT | `/presentation-letters/templates/{practice_type}` | Actualiza plantilla institucional. | Rol administrativo autorizado |
| POST | `/presentation-letters/generate` | Genera carta automática para el estudiante autenticado. | Estudiante |
| GET | `/presentation-letters/me` | Lista cartas generadas por el estudiante autenticado. | Estudiante |
| GET | `/presentation-letters/{letter_id}/download` | Descarga una carta autorizada. | Propietario o rol administrativo autorizado |

## Contratos principales

### Generación

```json
{
  "practice_type": "Práctica de Estudio I"
}
```

### Respuesta

```json
{
  "id": 12,
  "student_id": 5,
  "practice_type": "Práctica de Estudio I",
  "template_id": 2,
  "generated_file_name": "carta_presentacion_5_20260623.pdf",
  "recipient_email": "estudiante.demo@ufromail.cl",
  "sent_at": "2026-06-23T12:00:00",
  "downloaded_at": null,
  "created_at": "2026-06-23T12:00:00",
  "updated_at": "2026-06-23T12:00:00"
}
```

## Diferencia con flujo manual

La tarea de planificación puede mencionar solicitud, emisión, rechazo o
cancelación como si existiera una tramitación manual. Ese alcance queda
reemplazado por la regla vigente:

- El estudiante genera la carta directamente desde la plataforma.
- La administración solo mantiene plantillas institucionales.
- No existe decisión administrativa sobre cada carta individual.
- No se modelan estados de cola porque no reflejan el proceso real acordado.

Si en el futuro se quisiera volver a un flujo manual, debe tratarse como una
funcionalidad nueva y no como corrección del módulo actual.

## Pruebas documentadas

La cobertura vigente está registrada en
`backend/tests/modules/presentation-letters-unitarias.md`:

- Gestión de plantillas.
- Generación automática con datos reales.
- Contenido diferenciado por tipo de práctica.
- Error sin plantilla activa.
- Descarga autorizada y rechazo de carta ajena.

