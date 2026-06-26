# Casos de Prueba - Internships

## Alcance

Estos casos documentan las pruebas end-to-end de mayor valor del módulo `internships` y su interacción con `admin`. El foco está en el flujo principal de creación/aprobación y en la regla institucional de seguro escolar para prácticas fuera de periodo regular.

## End-to-End

### CU-E2E-IN-01: Estudiante crea práctica y administración la aprueba

- **Tipo de prueba:** End-to-end
- **Dominio:** Internships
- **Contexto:** Este es el flujo principal del sistema: estudiante cumple inducción, crea una solicitud y administración la aprueba.
- **Objetivo:** Validar que autenticación, inducción cumplida, creación de práctica, aprobación, historial y requisito académico funcionan juntos.
- **Escenario:** Estudiante inicia sesión, crea `Práctica de Estudio I`; Dirección inicia sesión y aprueba la práctica.
- **Resultado esperado:** La práctica queda `Aprobada`, existe historial de creación y aprobación, y el requisito académico queda actualizado a `Aprobada`.
- **Valor de negocio:** Da confianza sobre el flujo base del proceso de prácticas.
- **Pruebas automatizadas:**
  - `tests/e2e/test_internships_e2e.py::test_student_creates_internship_and_admin_approves`

### CU-E2E-IN-02: Práctica fuera de periodo se bloquea hasta validar seguro

- **Tipo de prueba:** End-to-end
- **Dominio:** Internships / Admin
- **Contexto:** La aprobación de una práctica fuera de marzo-junio o agosto-noviembre depende de validación de seguro escolar por solicitud o excepción.
- **Objetivo:** Confirmar que la regla de seguro escolar funciona en el flujo completo y no solo como validación aislada.
- **Escenario:** Estudiante crea práctica fuera de periodo regular; Dirección intenta aprobar y falla por falta de seguro; Dirección valida `insurance_status` de la solicitud; Dirección aprueba nuevamente.
- **Resultado esperado:** El primer intento devuelve `409 Conflict`; luego de la validación por solicitud, el segundo intento deja la práctica `Aprobada`.
- **Valor de negocio:** Protege una regla institucional crítica dentro del flujo real de API.
- **Pruebas automatizadas:**
  - `tests/e2e/test_internships_e2e.py::test_out_of_period_internship_is_blocked_until_director_validates_insurance`
