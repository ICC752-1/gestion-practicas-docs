# Seed demo QA

Este documento describe el alcance del script `scripts/seed_demo.py` del
backend. El objetivo del seed no es reemplazar pruebas automatizadas ni poblar
producción; es preparar una base local idempotente para QA funcional por roles.

## Uso

### Ejecución local

```bash
DEMO_SEED_PASSWORD=demo-password uv run python scripts/seed_demo.py
```

### Ejecución en Docker Compose

```bash
docker compose exec -e DEMO_SEED_PASSWORD=demo-password backend uv run python scripts/seed_demo.py
```

### Limpieza de datos demo

```bash
DEMO_SEED_PASSWORD=demo-password uv run python scripts/seed_demo.py --clean
```

## Reglas de seguridad

- `DEMO_SEED_PASSWORD` es obligatorio.
- La contraseña debe tener al menos ocho caracteres.
- El script se bloquea si `ENVIRONMENT`, `APP_ENV`, `ENV` o `MODE` indican
  `prod` o `production`.
- No debe ejecutarse contra una base productiva.
- Los datos son idempotentes: una nueva ejecución reutiliza o actualiza registros
  conocidos en lugar de duplicarlos.

## Usuarios demo

| Usuario | Rol principal | Uso QA |
| --- | --- | --- |
| `estudiante.demo@ufromail.cl` | Estudiante | Flujo estudiante con inducción y práctica exportable. |
| `estudiante.otro@ufromail.cl` | Estudiante | Flujo bloqueado/pendiente para contraste. |
| `estudiante.activo@ufromail.cl` | Estudiante | Cuenta activa adicional para pruebas de listados. |
| `encargado.practicas@ufrontera.cl` | Encargado de practica | Revisión, aprobación/rechazo, documentos, agenda y reportes. |
| `director.carrera@ufrontera.cl` | Director de carrera | Decisión directa, seguro escolar y reportes por carrera. |
| `secretaria.carrera@ufrontera.cl` | Secretaria de Carrera | Gestión documental y exportación local DIRAE. |
| `superadmin@ufrontera.cl` | Superadmin | Administración técnica de usuarios/roles. |
| `fica.reportes@ufrontera.cl` | FICA | Reportes agregados de solo lectura. |
| `supervisor.demo@empresa.cl` | Supervisor de practica | Consulta autenticada limitada del supervisor. |

Todos los usuarios creados por el seed usan la contraseña definida en
`DEMO_SEED_PASSWORD`.

## Datos cubiertos

El seed prepara, como mínimo:

- Roles del sistema requeridos por Sprint 11.
- Usuarios demo por rol.
- Estudiantes con correos institucionales `@ufromail.cl`.
- Requisitos académicos coherentes para los estudiantes demo.
- Inducción publicada con opciones estables:
  - `accept`: `Entiendo y acepto`.
  - `reject`: `No acepto`.
- Solicitud/práctica demo aprobada con documentos requeridos aprobados.
- Solicitud demo pendiente para contraste.
- Excepción administrativa demo.
- Autoevaluación enviada para la práctica exportable.
- Invitación/evaluación de supervisor para la práctica exportable.
- Notificaciones demo.
- Documentos requeridos aprobados para paquete documental.

## Bordes no sembrados automáticamente

El propio script declara escenarios que no prepara completamente. Estos no deben
interpretarse como fallas del seed, sino como datos que QA debe crear
manualmente o que deben agregarse al seed en una iteración posterior:

| Borde | Motivo |
| --- | --- |
| Agenda con entrevista, presentación final y conflicto horario | Requiere combinaciones temporales y validación de solapamientos. |
| Invitaciones de supervisor revocadas y expiradas | El seed deja un camino exitoso; los bordes negativos se prueban mejor con fixtures controladas. |
| Solicitud de carta pendiente y carta emitida | El flujo vigente de carta es automático; una solicitud pendiente manual no aplica al alcance actual. |
| Versión de inducción en borrador | El seed publica una versión activa; borradores se validan desde el panel administrativo. |
| Eventos de auditoría transversales | La auditoría funcional todavía no es un módulo transversal completo. |
| Paquete DIRAE no exportable y expediente observado | Requiere combinación de documentos faltantes/observados y práctica no cerrada. |

## Lectura PO

Para Sprint 11, `seed_demo.py` es suficiente como base de QA funcional inicial,
pero no cierra por sí solo la matriz completa de regresión. Los bordes no
sembrados deben quedar cubiertos por una de estas alternativas:

- fixture manual documentada en el guion de QA;
- caso automatizado específico;
- ampliación futura del seed si se necesita repetir el escenario en varias
  rondas de prueba.

## Relación con Insomnia y QA manual

Las variables de colección deberían apuntar a los usuarios demo anteriores y a
los IDs creados o consultados durante la preparación. Cuando el seed no crea un
escenario borde, la colección debe incluir un paso previo explícito para crear el
estado necesario antes de ejecutar la validación.

