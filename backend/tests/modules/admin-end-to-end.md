# Casos de Prueba - Admin

## Alcance

Estos casos documentan las pruebas de valor del módulo `admin`. El foco está en consultas administrativas, filtros del dashboard, detalle de prácticas, gestión de requisitos académicos, registro institucional de seguro escolar, permisos y traducción de errores del controller.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos de negocio verificables con una o más pruebas automatizadas.

## End-to-End

### CU-E2E-AD-01: Coordinador consulta dashboard y detalle de práctica

- Tipo de prueba: End-to-end
- Dominio: Admin
- Contexto: El flujo administrativo principal inicia en el dashboard, continúa con listado filtrado y termina en el detalle de una práctica.
- Objetivo: Validar autenticación, permisos, resumen, filtros y detalle administrativo en conjunto.
- Escenario: Un `Encargado de practica` inicia sesión, consulta `/admin/summary`, lista prácticas filtradas y abre el detalle de una práctica.
- Variantes cubiertas:
  - Flujo completo pendiente de automatización.
- Resultado esperado: El usuario autorizado ve métricas, lista y detalle coherentes.
- Valor de negocio: Da confianza sobre la operación diaria de coordinación.
- Pruebas automatizadas:
  - Pendiente de implementación.

### CU-E2E-AD-02: Coordinador actualiza requisito académico y estudiante recibe notificación

- Tipo de prueba: End-to-end
- Dominio: Admin / Notifications
- Contexto: Cambiar estado de un requisito académico debe impactar el avance del estudiante y generar comunicación.
- Objetivo: Validar el flujo completo de actualización administrativa, persistencia, notificación y consulta posterior.
- Escenario: Un coordinador actualiza un requisito de práctica; el estudiante consulta su notificación o estado actualizado.
- Variantes cubiertas:
  - Flujo completo pendiente de automatización.
- Resultado esperado: El requisito queda actualizado con trazabilidad y existe notificación asociada.
- Valor de negocio: Protege la coordinación entre gestión administrativa y comunicación al estudiante.
- Pruebas automatizadas:
  - Pendiente de implementación.

### CU-E2E-AD-03: Director valida seguro escolar y práctica fuera de periodo regular puede aprobarse

- Tipo de prueba: End-to-end
- Dominio: Admin / Internships
- Contexto: El seguro escolar validado por Dirección para una solicitud concreta condiciona la aprobación final de prácticas fuera del periodo regular.
- Objetivo: Validar que la regularización por solicitud se refleja en el flujo completo de aprobación.
- Escenario: Una práctica fuera de periodo regular inicialmente bloqueada por falta de seguro es validada por Dirección; luego se aprueba.
- Variantes cubiertas:
  - Flujo completo pendiente de automatización.
- Resultado esperado: Antes de la validación, la aprobación falla; después de marcar `insurance_status=validated`, la práctica puede aprobarse si cumple el resto de reglas.
- Valor de negocio: Protege una regla institucional crítica atravesando módulos.
- Pruebas automatizadas:
  - Pendiente de implementación.
