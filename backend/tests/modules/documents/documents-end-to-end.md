# Casos de Prueba - Documents

## Alcance

Estos casos documentan las pruebas end-to-end de mayor valor del módulo `documents`. El foco está en el flujo documental básico de upload, descarga autorizada y aprobación por rol documental.

## End-to-End

### CU-E2E-DO-01: Estudiante sube documento y rol documental lo aprueba

- **Tipo de prueba:** End-to-end
- **Dominio:** Documents
- **Contexto:** La revisión documental básica atraviesa autenticación, práctica registrada, upload, storage privado, descarga y aprobación.
- **Objetivo:** Validar el flujo completo de carga y aprobación documental.
- **Escenario:** Estudiante inicia sesión, sube documento; rol documental lo descarga y aprueba.
- **Resultado esperado:** El documento queda `approved`, con metadata de revisión y descarga autorizada.
- **Valor de negocio:** Da confianza sobre el flujo documental principal sin automatizar variantes de DIRAE más costosas.
- **Pruebas automatizadas:**
  - `tests/e2e/test_documents_e2e.py::test_student_uploads_document_and_document_role_approves_it`
