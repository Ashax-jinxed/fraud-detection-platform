# 03 - Snowflake (almacenamiento)

## Qué es
Data warehouse en la nube. Separa el **almacenamiento** del **cómputo** (warehouses), y se
consulta con SQL estándar.

## Rol en el proyecto
Guarda los datos en tres capas: `RAW` (tal como llegan), `STAGING` (limpios) y `MARTS`
(listos para ML, dashboard y alertas).

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| Delta Lake en S3 | Formato lakehouse abierto | Más piezas que armar, menos solicitado como "almacén" | Lo reemplazamos por decisión del proyecto |
| BigQuery | Muy usado, sin servidores | Atado a GCP | Elegimos AWS como nube |
| Redshift | Nativo de AWS | Menos moderno de usar, administración más pesada | Snowflake es más pedido |
| Databricks | Muy potente | Más complejo y costoso | Ya usamos Spark aparte |
| DuckDB + Parquet | Gratis y local | No se parece a un entorno de empresa | Poco peso en un portafolio |
| **Snowflake** | Muy solicitado, SQL estándar, funciona con dbt, Spark y Python | Prueba gratuita limitada, hay que controlar costos | **Elegido** |

## Por qué elegimos Snowflake
- Aparece en 9.9% de las ofertas de Data Engineering (y "Snowflake o BigQuery" en 54% en otro análisis).
- Se integra con todo el stack: Spark, dbt, Airflow, Python y Streamlit.
- Enseña conceptos de empresa: roles, warehouses, control de costos.

## Beneficios que obtendremos
- **Time Travel:** consultar datos como estaban hace horas o días.
- **Zero-copy clone:** copiar una base completa al instante para pruebas.
- **VARIANT:** guardar JSON sin esquema fijo (útil en la capa RAW).
- Cómputo que se apaga solo y no gasta créditos cuando no se usa.

## Parámetros y objetos que más nos interesan

| Parámetro | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| `WAREHOUSE_SIZE` | Potencia del cómputo | `XSMALL` | `SMALL`, `MEDIUM`... (gastan más créditos) |
| `AUTO_SUSPEND` | Segundos de inactividad antes de apagar | `60` | `300` (default habitual), `0` (nunca; evitar) |
| `AUTO_RESUME` | Se enciende solo al recibir consultas | `TRUE` | `FALSE` (hay que encenderlo a mano) |
| `INITIALLY_SUSPENDED` | Se crea apagado | `TRUE` | `FALSE` |
| `DATA_RETENTION_TIME_IN_DAYS` | Días de Time Travel | `1` | `0` (sin Time Travel); hasta 90 en ediciones superiores |
| `RESOURCE MONITOR` + `CREDIT_QUOTA` | Tope de créditos con alerta/suspensión | Cuota baja con suspensión al 100% | Sin monitor (riesgo de gasto) |
| Esquemas | Separar capas | `RAW`, `STAGING`, `MARTS` | Una sola capa (se mezcla todo) |
| Rol | Permisos | `FRAUD_ROLE` propio con mínimos permisos | `ACCOUNTADMIN` (demasiado poder) |
| Autenticación | Cómo conectan los scripts | Par de llaves (key-pair) | Usuario y contraseña |

Nota sobre autenticación: Snowflake exige cada vez más MFA para usuarios con contraseña, por
eso usaremos autenticación por par de llaves en los scripts. Lo verificaremos en la Etapa B
con la documentación vigente.

## Conexión desde Python
`snowflake.connector.connect(account, user, private_key, warehouse, database, schema, role)`,
luego `cursor.execute(sql)` y `cursor.fetchall()`.

## Documentación
- General: https://docs.snowflake.com/
- Conector de Python: https://docs.snowflake.com/en/developer-guide/python-connector/python-connector
- Warehouses: https://docs.snowflake.com/en/user-guide/warehouses
- Resource monitors: https://docs.snowflake.com/en/user-guide/resource-monitors
- Time Travel: https://docs.snowflake.com/en/user-guide/data-time-travel

## Dónde se usa
Etapas B, C y D.
