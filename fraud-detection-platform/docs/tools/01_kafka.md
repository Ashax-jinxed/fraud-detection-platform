# 01 - Apache Kafka (ingesta y streaming)

## Qué es
Plataforma de mensajería distribuida. Los **producers** escriben eventos en un **topic**;
los **consumers** los leen cuando quieren, y Kafka los guarda por un tiempo configurable.

## Rol en el proyecto
Recibe cada transacción simulada y la deja disponible para Spark. Separa quien envía los datos
de quien los procesa.

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| RabbitMQ | Simple, bueno para colas de tareas | No está pensado para reprocesar historial ni alto volumen de eventos | Es otro tipo de herramienta |
| AWS Kinesis | Administrado por AWS | Atado a AWS, menos portable, pago por uso | Menos solicitado que Kafka |
| Redpanda | Compatible con Kafka, más liviano | Menos pedido en ofertas | Lo que aprendamos en Kafka sirve para Redpanda, no al revés |
| Apache Pulsar | Potente | Comunidad y ofertas mucho menores | |
| **Apache Kafka** | Estándar de la industria | Más pesado de operar | **Elegido** |

## Por qué elegimos Kafka
- Aparece en 12.8% de las ofertas de Data Engineering analizadas (31% en otro análisis).
- Conceptos que se preguntan en entrevistas: topics, particiones, offsets, grupos de consumidores.
- Se integra de forma nativa con Spark.

## Beneficios que obtendremos
- Desacoplamiento: si Spark se cae, los mensajes siguen en Kafka.
- Reprocesamiento: podemos volver a leer desde el inicio.
- Orden por clave dentro de cada partición.

## Modo de ejecución
Kafka en **modo KRaft** dentro de Docker. KRaft reemplaza a Zookeeper, así que hay un
contenedor menos que mantener.

## Parámetros y conceptos que más nos interesan

| Parámetro | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| `bootstrap.servers` | Dirección del broker | `localhost:9092` | Lista de varios brokers en producción |
| `partitions` (del topic) | Divide el topic para paralelismo | `3` | `1` (sin paralelismo), muchas (más complejo) |
| `replication.factor` | Copias de cada partición | `1` (un solo broker local) | `3` en producción real |
| `acks` (producer) | Cuántas confirmaciones espera el producer | `all` | `0` (puede perder datos), `1` (solo el líder) |
| `key` del mensaje | Decide la partición | `nameOrig` (cuenta de origen) | Sin clave (reparto al azar) |
| `linger.ms` | Espera para agrupar mensajes | `10` | `0` (envía ya, menos eficiente) |
| `group.id` (consumer) | Grupo que comparte la lectura | `fraud-consumer-group` | Otro nombre = lectura independiente |
| `auto.offset.reset` | Dónde empieza un consumer nuevo | `earliest` | `latest` (solo mensajes nuevos) |
| `retention.ms` | Cuánto guarda Kafka los mensajes | 7 días (default) | Más tiempo = más disco |

Por qué usar `nameOrig` como clave: así todas las transacciones de una misma cuenta caen en la
misma partición y mantienen su orden.

## Librería Python
`confluent-kafka`. Funciones básicas: `Producer(config)`, `producer.produce(topic, key, value, callback)`,
`producer.flush()`, `Consumer(config)`, `consumer.subscribe([topic])`, `consumer.poll(timeout)`.

## Documentación
- Kafka: https://kafka.apache.org/documentation/
- Cliente Python: https://docs.confluent.io/kafka-clients/python/current/overview.html
- Configuración del producer: https://kafka.apache.org/documentation/#producerconfigs

## Dónde se usa
Etapa B (producer y lectura desde Spark).
