# Casos unitarios de auditoría

Estos casos documentan las pruebas unitarias de valor del módulo `audit`. El
foco está en sanitización de datos sensibles y autorización estricta del acceso
consultivo.

## CU-U-AT-01: Listado y detalle de auditoría sanitizan valores sensibles

- **Objetivo:** Validar que el servicio de auditoría oculte campos sensibles y
  exponga solo un resumen seguro de cambios.
- **Valor de negocio:** Protege trazabilidad administrativa sin filtrar hashes,
  rutas privadas ni metadata sensible al visor de auditoría.
- **Cobertura actual:** 3 tests automatizados.
- **Escenarios cubiertos:**
  - El listado resume actor, acción y campos cambiados sin exponer secretos.
  - El detalle enmascara `password_hash` y rutas privadas.
  - La consulta individual retorna `404` cuando el evento no existe.
- **Tests asociados:**
  - `tests/modules/audit/test_audit_service.py::test_audit_service_returns_sanitized_list_summary`
  - `tests/modules/audit/test_audit_service.py::test_audit_detail_masks_sensitive_values`
  - `tests/modules/audit/test_audit_service.py::test_audit_detail_returns_404_when_event_does_not_exist`

## CU-U-AT-02: Política de auditoría permite solo Superadmin

- **Objetivo:** Validar que la política de acceso a auditoría quede restringida
  a `Superadmin`.
- **Valor de negocio:** Evita exposición transversal de eventos auditados a
  roles operativos o académicos sin permiso.
- **Cobertura actual:** 2 tests automatizados.
- **Escenarios cubiertos:**
  - `Superadmin` sí puede pasar la dependencia de roles.
  - Estudiante, encargado, director, secretaría, supervisor y FICA son rechazados.
- **Tests asociados:**
  - `tests/modules/audit/test_audit_service.py::test_audit_policy_allows_superadmin`
  - `tests/modules/audit/test_audit_service.py::test_audit_policy_rejects_non_superadmin_roles`
