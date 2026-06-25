# Casos de Prueba - Internships

## Alcance

Estos casos documentan las pruebas de valor del módulo `internships`. El foco está en reglas de aprobación, requisitos institucionales, secuencialidad académica, permisos, trazabilidad, edición administrativa, anulación lógica, contratos de entrada y acceso a información de prácticas.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino un comportamiento de negocio verificable con una o más pruebas automatizadas.

## End-to-End

### CU-E2E-IN-01: Flujo completo de Práctica I aprobada

- **Tipo de prueba:** End-to-end
- **Dominio:** Internships
- **Contexto:** Este es un flujo principal del sistema: estudiante inicia proceso, cumple inducción, crea práctica y administración aprueba.
- **Objetivo:** Validar que autenticación, inducción, creación de práctica, aprobación, historial y requisito académico funcionan juntos.
- **Escenario:** Estudiante inicia sesión, aprueba inducción, crea `Práctica de Estudio I`; un administrador inicia sesión y aprueba la práctica.
- **Variantes cubiertas:**
  - Flujo completo pendiente de automatización.
- **Resultado esperado:** La práctica queda `Aprobada`, existe historial de creación y aprobación, y el requisito académico queda actualizado.
- **Valor de negocio:** Da confianza sobre el flujo base del proceso de prácticas.
- **Pruebas automatizadas:**
  - Pendiente de implementación.

### CU-E2E-IN-02: Flujo completo fuera de periodo regular bloqueado y luego aprobado

- **Tipo de prueba:** End-to-end
- **Dominio:** Internships / Admin
- **Contexto:** La aprobación de una práctica fuera de marzo-junio o agosto-noviembre depende de validación de seguro escolar por solicitud o excepción. El estudiante puede crear la solicitud sin seguro validado, pero no debería obtener aprobación final hasta regularizarla.
- **Objetivo:** Confirmar que la regla de seguro escolar funciona en el flujo completo y no solo como validación aislada.
- **Escenario:** Estudiante crea práctica fuera de periodo regular; administración intenta aprobar y falla por falta de seguro; Dirección valida `insurance_status` de la solicitud; administración aprueba nuevamente.
- **Variantes cubiertas:**
  - Flujo completo pendiente de automatización.
- **Resultado esperado:** El primer intento devuelve `409 Conflict`; luego de la validación por solicitud, el segundo intento deja la práctica `Aprobada`.
- **Valor de negocio:** Protege una regla institucional crítica dentro del flujo real de API.
- **Pruebas automatizadas:**
  - Pendiente de implementación.
