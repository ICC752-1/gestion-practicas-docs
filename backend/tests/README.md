# Estrategia de Pruebas Backend

## Objetivo

Este directorio documenta los casos de prueba de backend que aportan valor real al sistema. El foco no es aumentar la cantidad de pruebas, sino proteger reglas de negocio, permisos, estados, trazabilidad, integridad documental y flujos críticos del proceso de prácticas.

Una prueba se considera valiosa cuando ayuda a detectar una regresión que afectaría al estudiante, a los roles administrativos, a la consistencia del trámite o a la seguridad funcional del sistema.

## Tipos de prueba

| Tipo | Uso en este backend |
| --- | --- |
| Unitaria | Valida una regla, servicio, schema o decisión de negocio en aislamiento, usando dobles de prueba cuando corresponde. |
| Integración | Valida que varias partes internas colaboren correctamente, por ejemplo endpoint + dependencias + servicio, o servicio + repositorio. |
| End-to-end | Valida un flujo completo de negocio atravesando las capas principales del backend. Deben ser pocas y centradas en flujos críticos. |

## Criterios de valor

Una prueba debe existir si cumple al menos uno de estos criterios:

- Protege una regla de negocio crítica.
- Evita aprobar, rechazar, derivar o exportar datos en un estado incorrecto.
- Verifica permisos relevantes por rol.
- Protege trazabilidad administrativa.
- Evita inconsistencias en requisitos académicos o institucionales.
- Valida un contrato documental o de sesión importante.
- Cubre una regresión probable o costosa de detectar manualmente.

Una prueba debería evitarse, consolidarse o reemplazarse si solo verifica detalles internos sin comportamiento observable, duplica otra prueba equivalente o existe únicamente para aumentar cobertura numérica.

## Glosario

| Término | Significado |
| --- | --- |
| Práctica estival | Práctica realizada en periodo `Verano` o `Invierno`, fuera del periodo académico regular. Para aprobación final requiere seguro escolar o excepción administrativa. |
| Seguro escolar | Requisito institucional que indica que el estudiante tiene cobertura registrada para la práctica. |
| Excepción administrativa | Autorización trazable que permite avanzar una práctica aunque falte una regla exceptuable. No modifica el requisito original. |
| Estado terminal | Estado final donde una práctica no debería modificarse por acciones normales. Actualmente: `Aprobada`, `Rechazada` y `Reprobada`. |
| Paquete documental DIRAE | Resumen de documentos requeridos y aprobados para evaluar si una práctica puede exportarse o tramitarse hacia DIRAE. |
| Exportable | Condición que indica que la práctica está aprobada y tiene todos los documentos requeridos aprobados para ser incluida en la exportación DIRAE. |
| Inducción | Requisito obligatorio para tramitar la aprobación de `Práctica de Estudio I`. |
| Secuencialidad | Regla académica que exige aprobar una práctica previa antes de aprobar una posterior, por ejemplo Práctica I antes de Práctica II. |

## Archivos

| Archivo | Propósito |
| --- | --- |
| `unit-tests.md` | Casos unitarios centrados en reglas de negocio aisladas. |
| `integration-tests.md` | Casos que validan interacción entre endpoints, dependencias, servicios, repositorios o permisos. |
| `end-to-end-tests.md` | Flujos completos críticos que justifican una prueba más costosa. |
| `modules/admin.md` | Casos de prueba detallados del módulo `admin`, agrupados por dashboard, requisitos, seguro escolar y errores HTTP. |
| `modules/auth.md` | Casos de prueba detallados del módulo `auth`, agrupados por sesiones, tokens, OAuth, roles y datos de identidad. |
| `modules/documents.md` | Casos de prueba detallados del módulo `documents`, agrupados por carga, permisos, revisión, paquete DIRAE y exportación. |
| `modules/internships.md` | Casos de prueba detallados del módulo `internships`, agrupados por comportamiento y vinculados a pruebas automatizadas. |
| `modules/notifications.md` | Casos de prueba detallados del módulo `notifications`, agrupados por despacho, consulta, reintentos y eventos productores. |
