# Casos de Prueba - Documents

## Alcance

Estos casos documentan las pruebas de valor del módulo `documents`. El foco está en carga documental, validación de archivos, storage privado, permisos de acceso, revisión documental, eliminación lógica, paquete DIRAE, exportación CSV y contratos de modelo.

Los casos agrupan variantes automatizadas relacionadas. No representan una prueba por función, sino comportamientos verificables que protegen reglas documentales, privacidad y consistencia operativa.

## End-to-End

### CU-E2E-DO-01: Estudiante carga documento y rol documental lo aprueba

- Tipo de prueba: End-to-end
- Dominio: Documents
- Contexto: La revisión documental básica atraviesa autenticación, práctica, upload, descarga y aprobación.
- Objetivo: Validar el flujo completo de carga y aprobación documental.
- Escenario: Estudiante inicia sesión, sube documento; rol documental lo descarga y aprueba.
- Variantes cubiertas:
  - Flujo completo pendiente de automatización.
- Resultado esperado: El documento queda `approved`, con metadata de revisión y descarga autorizada.
- Valor de negocio: Da confianza sobre el flujo documental principal.
- Pruebas automatizadas:
  - Pendiente de implementación.

### CU-E2E-DO-02: Documento observado se corrige con nueva versión aprobada

- Tipo de prueba: End-to-end
- Dominio: Documents
- Contexto: Un documento observado debe permitir corrección mediante una nueva carga que luego pueda aprobarse.
- Objetivo: Validar ciclo completo de observación, corrección y selección de última versión aprobada.
- Escenario: Rol documental observa un documento; estudiante sube nueva versión; rol documental aprueba; paquete selecciona la última aprobada.
- Variantes cubiertas:
  - Flujo completo pendiente de automatización.
- Resultado esperado: La versión observada no se usa para paquete; la nueva aprobada queda seleccionada.
- Valor de negocio: Protege el flujo real de corrección documental.
- Pruebas automatizadas:
  - Pendiente de implementación.

### CU-E2E-DO-03: Exportación DIRAE de práctica aprobada con documentos completos

- Tipo de prueba: End-to-end
- Dominio: Documents / Internships
- Contexto: La exportación DIRAE depende de estado de práctica y documentos requeridos aprobados.
- Objetivo: Validar el flujo completo desde práctica aprobada y documentos aprobados hasta CSV exportable.
- Escenario: Práctica aprobada tiene documentos requeridos aprobados; rol documental exporta paquetes DIRAE.
- Variantes cubiertas:
  - Flujo completo pendiente de automatización.
- Resultado esperado: El CSV contiene la práctica, IDs de documentos aprobados y datos del estudiante.
- Valor de negocio: Da confianza sobre una salida institucional crítica.
- Pruebas automatizadas:
  - Pendiente de implementación.
