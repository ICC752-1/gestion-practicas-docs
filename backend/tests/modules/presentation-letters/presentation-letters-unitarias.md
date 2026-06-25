# Casos de Prueba - Presentation Letters

## Alcance

Estos casos documentan las pruebas unitarias de valor del módulo `presentation_letters`. El foco está en plantillas editables, permisos, generación automática de cartas PDF, contenido institucional, notificación y descarga autenticada.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos verificables que protegen contratos funcionales del módulo.

## Unitarias

### CU-U-PL-01: Dirección gestiona plantillas de cartas

- **Tipo de prueba:** Unitaria
- **Dominio:** Presentation Letters
- **Contexto:** Las cartas se generan desde plantillas institucionales editables.
- **Objetivo:** Validar lectura y actualización de plantillas por roles autorizados.
- **Escenario:** Dirección lista y actualiza plantillas; un estudiante intenta editar.
- **Variantes cubiertas:**
  - Director lista plantillas para Práctica I y II.
  - Director actualiza contenido y queda registrado como editor.
  - Estudiante no puede actualizar plantillas.
- **Resultado esperado:** Solo Dirección puede modificar plantillas, y la lectura conserva tipos disponibles.
- **Valor de negocio:** Protege el contenido institucional usado en cartas oficiales.
- **Pruebas automatizadas:**
  - `tests/modules/presentation_letters/test_presentation_letter_service.py::test_director_can_list_templates`
  - `tests/modules/presentation_letters/test_presentation_letter_service.py::test_director_can_update_template`
  - `tests/modules/presentation_letters/test_presentation_letter_service.py::test_student_cannot_update_template`

### CU-U-PL-02: Estudiante genera carta con datos reales y notificación

- **Tipo de prueba:** Unitaria
- **Dominio:** Presentation Letters
- **Contexto:** La carta de presentación se genera automáticamente para el estudiante autenticado.
- **Objetivo:** Validar que la generación usa datos reales del estudiante, escribe PDF y registra notificación cuando corresponde.
- **Escenario:** Un estudiante genera carta de Práctica I.
- **Variantes cubiertas:**
  - PDF queda escrito en storage.
  - Contenido contiene nombre, identificador, horas, seguro escolar y firma institucional.
  - Se registra `sent_at` cuando la notificación fue despachada.
- **Resultado esperado:** La carta generada contiene información institucional y del estudiante sin caracteres corruptos.
- **Valor de negocio:** Evita cartas oficiales incompletas o con datos incorrectos.
- **Pruebas automatizadas:**
  - `tests/modules/presentation_letters/test_presentation_letter_service.py::test_student_generates_practice_i_letter_with_real_user_data`

### CU-U-PL-03: Carta de Práctica II usa contenido diferenciado

- **Tipo de prueba:** Unitaria
- **Dominio:** Presentation Letters
- **Contexto:** Práctica I y Práctica II tienen resultados de aprendizaje distintos.
- **Objetivo:** Confirmar que la carta de Práctica II no reutiliza contenido propio de Práctica I.
- **Escenario:** Un estudiante genera carta de Práctica II.
- **Variantes cubiertas:**
  - Se incluyen textos y aprendizajes de Práctica II.
  - No aparecen aprendizajes específicos de Práctica I.
- **Resultado esperado:** El contenido generado corresponde al tipo de práctica solicitado.
- **Valor de negocio:** Protege exactitud académica del documento institucional.
- **Pruebas automatizadas:**
  - `tests/modules/presentation_letters/test_presentation_letter_service.py::test_student_generates_practice_ii_letter_with_distinct_content`

### CU-U-PL-04: Contexto DOCX conserva campos textuales de plantilla

- **Tipo de prueba:** Unitaria
- **Dominio:** Presentation Letters
- **Contexto:** La plantilla DOCX consume un contexto estructurado construido por el service.
- **Objetivo:** Validar que el contexto contiene campos textuales esperados para renderizar la carta.
- **Escenario:** Se construye documento y contexto para Práctica I.
- **Variantes cubiertas:**
  - Título, subtítulo, introducción, presentación del estudiante y cláusula de horas.
  - Firma institucional.
- **Resultado esperado:** El contexto contiene los textos institucionales mínimos.
- **Valor de negocio:** Evita romper la plantilla DOCX sin detectar el cambio.
- **Pruebas automatizadas:**
  - `tests/modules/presentation_letters/test_presentation_letter_service.py::test_docx_context_uses_textual_template_fields`

### CU-U-PL-05: Generación falla con error claro si no hay plantilla activa

- **Tipo de prueba:** Unitaria
- **Dominio:** Presentation Letters
- **Contexto:** La generación depende de una plantilla activa por tipo de práctica.
- **Objetivo:** Validar que el error funcional sea explícito cuando falta plantilla.
- **Escenario:** Se elimina la plantilla activa de Práctica I y el estudiante intenta generar carta.
- **Variantes cubiertas:**
  - Respuesta `404` con código `presentation_letter_template_not_found`.
- **Resultado esperado:** El consumidor puede distinguir configuración faltante de otros errores.
- **Valor de negocio:** Facilita diagnóstico operativo y evita fallos silenciosos.
- **Pruebas automatizadas:**
  - `tests/modules/presentation_letters/test_presentation_letter_service.py::test_missing_active_template_returns_clear_error`

### CU-U-PL-06: Descarga autenticada respeta propiedad de la carta

- **Tipo de prueba:** Unitaria
- **Dominio:** Presentation Letters
- **Contexto:** Las cartas generadas contienen datos personales y deben descargarse con autorización.
- **Objetivo:** Validar descarga propia y rechazo de carta ajena.
- **Escenario:** Un estudiante descarga su carta y luego intenta descargar una carta de otro estudiante.
- **Variantes cubiertas:**
  - Propietario descarga archivo y se registra `downloaded_at`.
  - Estudiante no propietario recibe `403`.
- **Resultado esperado:** Solo el propietario o un rol administrativo puede acceder a la carta.
- **Valor de negocio:** Protege documentos personales del estudiante.
- **Pruebas automatizadas:**
  - `tests/modules/presentation_letters/test_presentation_letter_service.py::test_student_can_download_own_letter`
  - `tests/modules/presentation_letters/test_presentation_letter_service.py::test_student_cannot_download_foreign_letter`
