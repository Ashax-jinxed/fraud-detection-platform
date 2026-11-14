# 02 - Apache Spark Structured Streaming (procesamiento)

## Qué es
Motor de procesamiento distribuido. *Structured Streaming* trata un flujo de datos como una
tabla que crece continuamente y permite usar la API de DataFrames sobre ella.

## Rol en el proyecto
Lee el topic de Kafka, convierte los mensajes JSON en columnas con tipos correctos y escribe
cada micro-lote en Snowflake (esquema `RAW`).

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| Consumidor en Python | Muy simple | No escala, no enseña procesamiento distribuido | Demasiado básico para mostrar nivel de Data Engineering |
| Apache Flink | Streaming "real" evento por evento | Menos ofertas, más difícil | Spark aparece más en empleos |
| Kafka Streams | Integrado en Kafka | Es Java/Scala | Preferimos Python |
| Spark Streaming (DStreams) | - | API antigua | Está obsoleta |
| **Spark Structured Streaming** | Muy solicitado, API de DataFrames, también sirve para batch | Necesita Java y configuración | **Elegido** |

## Por qué elegimos Spark
- Aparece en 12.4% de las ofertas (43% en otro análisis).
- Lo que aprendemos sirve también para procesamiento batch.
- Tiene conector oficial para Kafka y para Snowflake.

## Beneficios que obtendremos
- Esquema explícito: los datos entran con tipos definidos.
- Tolerancia a fallos con *checkpoints*: si se cae, retoma donde quedó.
- Escritura por lotes a Snowflake con `foreachBatch`.

## Parámetros y funciones que más nos interesan

| Parámetro / función | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| `readStream.format("kafka")` | Lee desde Kafka | - | `rate` (datos de prueba) |
| `kafka.bootstrap.servers` | Dirección de Kafka | `localhost:9092` | - |
| `subscribe` | Topic a leer | `transactions` | `subscribePattern` (varios topics) |
| `startingOffsets` | Desde dónde leer la primera vez | `earliest` | `latest` |
| `maxOffsetsPerTrigger` | Máximo de mensajes por lote | `1000` | Sin límite (puede saturar la PC) |
| `from_json` + `StructType` | Convierte JSON a columnas con tipos | Esquema explícito | `inferSchema` (más lento y menos seguro) |
| `outputMode` | Qué se escribe en cada lote | `append` | `update`, `complete` (para agregaciones) |
| `trigger` | Cada cuánto se procesa un lote | `processingTime="10 seconds"` | `availableNow=True` (procesa todo y termina), `once` |
| `checkpointLocation` | Guarda el progreso | Carpeta local | Ruta en S3 (producción) |
| `foreachBatch` | Ejecuta código propio por lote | Escribe en Snowflake | `writeStream.format(...)` directo |
| `spark.sql.shuffle.partitions` | Particiones al agrupar | `4` | `200` (default, excesivo para una PC) |

## Conexión con Snowflake
Se usa el formato `net.snowflake.spark.snowflake` con opciones `sfURL`, `sfUser`, `sfDatabase`,
`sfSchema`, `sfWarehouse`, `sfRole` y `dbtable`. Requiere los paquetes `spark-snowflake`,
`snowflake-jdbc` y `spark-sql-kafka-0-10`. Las versiones deben coincidir con las de Spark y
Scala; las fijaremos en la Etapa B.

## Documentación
- Guía de Structured Streaming: https://spark.apache.org/docs/latest/
  (sección Structured Streaming Programming Guide)
- Integración con Kafka: https://spark.apache.org/docs/latest/ (Structured Streaming + Kafka)
- Conector Spark-Snowflake: https://docs.snowflake.com/en/user-guide/spark-connector

## Dónde se usa
Etapa B.
