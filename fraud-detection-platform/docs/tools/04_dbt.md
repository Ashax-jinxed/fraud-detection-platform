# 04 - dbt Core (transformación y calidad de datos)

## Qué es
Herramienta que transforma datos con archivos SQL versionados en Git, con pruebas
automáticas y documentación generada.

## Rol en el proyecto
Convierte `RAW` en `STAGING` (datos limpios) y `MARTS` (tablas de features y alertas),
y valida la calidad con tests.

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| SQL puro en scripts | Sin herramientas nuevas | Sin orden de dependencias, sin tests, sin documentación | Poco profesional |
| Great Expectations | Muy completo para calidad de datos | Solo valida, no transforma | dbt cubre transformar y validar |
| SQLMesh | Moderno, buenas ideas | Mucho menos solicitado | |
| Dataform | Similar a dbt | Atado a Google | |
| **dbt Core** | Estándar, 61% de ofertas en un análisis de Q1 2026 | Aprender su estructura | **Elegido** |

## Por qué elegimos dbt
- Es de las herramientas más pedidas en Data Engineering y Analytics Engineering.
- Las dependencias entre tablas se resuelven solas con `ref()`.
- Tests y documentación vienen incluidos.

## Beneficios que obtendremos
- Transformaciones versionadas y revisables.
- Tests de calidad en cada ejecución.
- Documentación y diagrama de linaje con `dbt docs generate`.

## Parámetros y funciones que más nos interesan

| Elemento | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| `materialized` | Cómo se guarda un modelo | `view` en staging, `table` en marts | `incremental` (tablas muy grandes), `ephemeral` |
| `unique_key` + `incremental_strategy` | Para modelos incrementales | `merge` si hacemos incremental | `append`, `delete+insert` |
| `{{ ref('model') }}` | Referencia otro modelo | Siempre | Escribir el nombre de la tabla a mano (rompe el orden) |
| `{{ source('raw', 'transactions') }}` | Referencia una tabla de origen | Siempre para RAW | - |
| Tests `unique`, `not_null` | Validan claves y nulos | Sobre el id de transacción | - |
| Test `accepted_values` | Valores permitidos | Sobre `type` | - |
| Test `relationships` | Integridad entre tablas | Entre marts y staging | - |
| `threads` (profiles.yml) | Modelos en paralelo | `4` | `1` (más lento), más de 8 (consume más créditos) |
| `dbt build` | Ejecuta modelos y tests en orden | Comando principal | `dbt run` + `dbt test` por separado |
| `--select` | Ejecuta solo parte del proyecto | Para depurar | Ejecutar todo siempre |

Archivos clave: `dbt_project.yml`, `profiles.yml` (conexión; fuera de Git), `models/*.sql`,
`models/schema.yml` (tests y descripciones).

## Documentación
- General: https://docs.getdbt.com/
- Configuración con Snowflake: https://docs.getdbt.com/docs/core/connect-data-platform/snowflake-setup
- Tests: https://docs.getdbt.com/docs/build/data-tests
- Materializaciones: https://docs.getdbt.com/docs/build/materializations

## Dónde se usa
Etapa B, y como tarea dentro del DAG de la Etapa C.
