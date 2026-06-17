<h1 align="center"><em>Nombre del módulo</em></h1>

> [!NOTE]
> Esta documentación técnica describe el comportamiento actual del módulo `<module>`
> desde una perspectiva funcional e interna. Su objetivo es explicar cómo está
> implementado y qué debe saber alguien antes de modificarlo. El contrato HTTP
> formal queda en OpenAPI y el detalle interactivo queda en Swagger.

## Contenidos
- [Resumen operativo](#resumen-operativo)
- [Ámbito y responsabilidades](#ámbito-y-responsabilidades)
- [Estructura interna](#estructura-interna)
- [Funcionalidades](#funcionalidades)
- [Endpoints disponibles](#endpoints-disponibles)
- [Contratos principales](#contratos-principales)
- [Reglas de negocio](#reglas-de-negocio)
- [Configuración por entorno](#configuración-por-entorno)
- [Consideraciones operativas](#consideraciones-operativas)

## Resumen operativo

El módulo **`<module>`** se encarga de ...

**Permite:**

- ...
- ...
- ...

## Ámbito y responsabilidades

El módulo **`<module>`** centraliza ...

#### Responsabilidades principales

- ...
- ...
- ...

#### Fuera de alcance

- ...
- ...
- ...

> [!IMPORTANT]
> Indicar aquí límites relevantes con otros módulos, ownership de datos o flujos
> que no deberían implementarse en este módulo.

## Estructura interna

| Capa | Archivo | Responsabilidad |
| --- | --- | --- |
| Controller | `app/modules/<module>/controllers/...` | Define rutas HTTP, dependencias y roles requeridos. |
| Service | `app/modules/<module>/services/...` | Orquesta reglas de negocio y validaciones. |
| Repository | `app/modules/<module>/repositories/...` | Encapsula consultas o persistencia. |
| Models | `app/modules/<module>/models/...` | Define entidades ORM del módulo. |
| Schemas | `app/modules/<module>/schemas/...` | Define contratos de entrada y salida. |

El módulo reutiliza ...

## Funcionalidades

#### Nombre de funcionalidad

1. El cliente o módulo consumidor inicia la operación.
2. El controller, service o componente correspondiente valida la entrada.
3. Se ejecuta la regla principal.
4. Se consulta o persiste información si corresponde.
5. Se retorna el resultado o se ejecuta el efecto secundario.

#### Otra funcionalidad

1. ...
2. ...
3. ...

## Endpoints disponibles

<!-- Eliminar esta sección si el módulo no expone rutas HTTP. -->

**Todos los endpoints requieren autenticación.**

| Método | Ruta | Propósito | Rol |
| --- | --- | --- | --- |
| GET | `/...` | ... | ... |
| POST | `/...` | ... | ... |
| PATCH | `/...` | ... | ... |

## Contratos principales

<!-- Usar esta sección para schemas, payloads, eventos o estructuras importantes. -->

<details>
<summary><strong>SchemaName</strong></summary>

Describe brevemente para qué se usa este contrato.

```json
{
  "id": 1,
  "field": "value"
}
```

</details>

## Reglas de negocio

<!-- Eliminar esta sección si el módulo no tiene reglas relevantes. -->

#### Nombre de regla o grupo de reglas

**Reglas actuales:**

- ...
- ...
- ...

> [!WARNING]
> Indicar aquí restricciones, transiciones inválidas o errores esperados.

## Configuración por entorno

<!-- Eliminar esta sección si el módulo no depende directamente de variables de entorno. -->

| Variable | Uso |
| --- | --- |
| `VARIABLE_NAME` | ... |

## Consideraciones operativas

- Servicios externos o internos requeridos.
- Datos seed necesarios.
- Efectos secundarios importantes.
- Comportamiento especial en desarrollo o producción.
