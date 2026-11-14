# Etapa B - ETL real (Kafka → Spark → Snowflake → dbt)

**Objetivo:** construir el flujo de datos real: las transacciones viajan por Kafka, Spark las guarda en Snowflake
y dbt las limpia, las traduce al español y calcula las mismas features que usó el modelo en la Etapa A.

**Prerrequisito:** la Etapa A terminó con la decisión **SEGUIR** o **SEGUIR CON CAMBIOS**
(`reports/decision_etapa_A.md`). Si no, no se empieza esta etapa.

**Resultado final de la etapa:**
1. 6,362,620 filas en `FRAUD_DB.RAW.TRANSACTIONS`.
2. Capas `STAGING` y `MARTS` creadas por dbt, con todos los tests en verde.
3. Las features de dbt coinciden con las de pandas (comprobado con un script).
4. Los modelos, reentrenados con datos que vienen de Snowflake, dan métricas comparables a la Etapa A.

**Qué se usa:** Docker Desktop, Kafka, Spark (dentro de Docker), Snowflake, dbt, `python-dotenv`.

---

## Arquitectura de la etapa

```
CSV de PaySim ──► producer (tu PC, Python) ──► Kafka (Docker) ──► Spark job (Docker) ──► Snowflake RAW.TRANSACTIONS
                                                                                              │
                                                                          dbt (tu PC, Python) ▼
                                                                  STAGING.STG_TRANSACTIONS (view)
                                                                  MARTS.TRANSACTION_FEATURES (table)
```

---

## Índice de pasos

| Paso | Qué hacemos |
|---|---|
| B0 | Comprobar requisitos |
| B1 | Variables de entorno y paquetes de Python |
| B2 | Kafka con Docker Compose y creación del topic |
| B3 | Producer de Kafka |
| B4 | Cuenta de Snowflake, estructura y llaves |
| B5 | Probar la conexión Python → Snowflake |
| B6 | Crear la tabla RAW |
| B7 | Job de Spark en Docker (carga completa a RAW) |
| B8 | Proyecto dbt: STAGING, MARTS y tests |
| B9 | Verificar que dbt calcula lo mismo que pandas |
| B10 | Reentrenar con datos de Snowflake y comparar con la Etapa A |
| B11 | Cierre de la etapa |

---

## Reglas que valen para toda la etapa

1. **Secretos fuera de Git:** `.env` y las llaves de Snowflake nunca se suben. Las llaves viven fuera del repositorio.
2. **RAW es sagrada:** guarda los datos tal como llegan (solo se normalizan los nombres de columnas a
   MAYÚSCULAS_CON_GUION_BAJO). Nunca se edita a mano; limpieza y traducción ocurren en `STAGING`.
3. **Todo es repetible:** si algo sale mal, se borra el topic, el checkpoint y la tabla, y se repite el paso.
4. **Costos bajo control:** el warehouse es `XSMALL` y se apaga solo a los 60 segundos. Revisa el consumo de
   créditos en Snowflake al terminar cada sesión.
5. **Nombres de objetos en Snowflake sin comillas:** quedan guardados en mayúsculas y se consultan sin importar
   mayúsculas o minúsculas.
6. **Comandos de Docker Compose desde la raíz del proyecto:** el archivo `docker-compose.yml` vive allí y lee `.env` solo.
7. Mismas reglas de código de la Etapa A (README general, sección 5).

---

## B0 - Comprobar requisitos

**Pasos:**
1. Abre `reports/decision_etapa_A.md` y confirma que la decisión es SEGUIR o SEGUIR CON CAMBIOS.
2. Con el entorno virtual activo: `python -m pytest tests -v` debe pasar completo.
3. Instala Docker Desktop (https://www.docker.com/products/docker-desktop/) y ábrelo hasta que diga que está corriendo.
   En Windows, deja activada la opción de WSL 2 que ofrece el instalador.
4. Comprueba Docker:
   ```
   docker --version
   docker compose version
   docker run --rm hello-world
   ```
5. En Docker Desktop → *Settings → Resources*, asigna **al menos 6 GB de memoria** (Spark usa unos 2 a 3 GB y Kafka 1 GB).
6. Comprueba OpenSSL (para generar las llaves de Snowflake): `openssl version`.
   - macOS y Linux: viene instalado.
   - Windows: viene con **Git Bash** (instalado con Git). Si PowerShell no lo encuentra, abre "Git Bash" y úsalo solo para el paso B4.
7. Libera al menos 10 GB de disco (las imágenes de Docker ocupan unos 3 GB).
8. Comprueba que el puerto 9092 está libre (ningún otro Kafka corriendo).

**Verificación:** Etapa A con decisión favorable, pruebas en verde, `hello-world` imprime "Hello from Docker!", `openssl version` responde, Docker con 6 GB o más.

---

## B1 - Variables de entorno y paquetes de Python

**Objetivo:** dejar el entorno de Python listo para la etapa y crear el archivo `.env`.

**Pasos:**
1. Abre `requirements.txt` y **descomenta el bloque de la Etapa B** (quita el `#` de: `python-dotenv[cli]`, `confluent-kafka`, `snowflake-connector-python[pandas]`, `dbt-core`, `dbt-snowflake`). `pyspark` no se instala aquí: Spark corre dentro de Docker.
2. Instala: `pip install -r requirements.txt`
3. Comprueba:
   ```
   dbt --version
   python -c "import confluent_kafka; print(confluent_kafka.version())"
   python -c "import snowflake.connector; print(snowflake.connector.__version__)"
   ```
4. Actualiza las versiones fijadas: `pip freeze > requirements-lock.txt`
5. Crea tu `.env` a partir de la plantilla (todavía vacío; se llena en B4):
   - macOS/Linux: `cp .env.example .env`
   - Windows PowerShell: `Copy-Item .env.example .env`
6. Crea las carpetas y los archivos `__init__.py` nuevos:
   - macOS/Linux:
     ```
     mkdir -p src/common src/ingestion src/streaming infra/snowflake docker/spark checkpoints dbt_project
     touch src/common/__init__.py src/ingestion/__init__.py src/streaming/__init__.py
     ```
   - Windows PowerShell:
     ```
     New-Item -ItemType Directory -Force -Path src/common, src/ingestion, src/streaming, infra/snowflake, docker/spark, checkpoints, dbt_project
     New-Item -ItemType File -Force src/common/__init__.py, src/ingestion/__init__.py, src/streaming/__init__.py
     ```
7. Comprueba que `.gitignore` incluye: `.env`, `checkpoints/`, `*.p8`, `rsa_key*`, `dbt_project/target/`, `dbt_project/logs/`. (El `.gitignore` entregado ya los trae; si no, agrégalos.)
8. Commit:
   ```
   git add requirements.txt requirements-lock.txt .gitignore .env.example src
   git commit -m "chore: add stage B dependencies and folders"
   ```
   Comprueba con `git status` que `.env` **no** aparece para subir.

**Verificación:** los tres comandos del punto 3 imprimen una versión; `.env` existe y no está en Git.

**Si falla:** conflicto de versiones de `pyarrow` al instalar el conector de Snowflake → deja que pip resuelva (`pip install -r requirements.txt` otra vez) y repite el paso A2 (comprobación de versiones) para confirmar que Etapa A sigue funcionando.

---

## B2 - Kafka con Docker Compose

**Objetivo:** tener un broker de Kafka corriendo en Docker y un topic llamado `transactions`.

**Archivo:** `docker-compose.yml` (en la raíz). En esta etapa solo contiene el servicio `kafka` (y luego `spark-job`).

**Contenido del servicio `kafka`:**
```yaml
services:
  kafka:
    image: apache/kafka:3.9.0
    container_name: fraud-kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: HOST://:9092,INTERNAL://:19092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: HOST://localhost:9092,INTERNAL://kafka:19092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: HOST:PLAINTEXT,INTERNAL:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
    volumes:
      - kafka_data:/var/lib/kafka/data

volumes:
  kafka_data:
```

**Qué significa cada configuración clave:**

| Configuración | Qué hace | Por qué |
|---|---|---|
| `image: apache/kafka:3.9.0` | Imagen oficial de Kafka | Verifica en https://hub.docker.com/r/apache/kafka/tags la versión estable vigente y úsala si es distinta |
| `KAFKA_PROCESS_ROLES: broker,controller` | El mismo nodo hace de broker y de controlador | Modo KRaft: sin Zookeeper, un contenedor menos |
| Listener `HOST` (puerto 9092) | Para programas en tu PC (el producer) | Se anuncia como `localhost:9092` |
| Listener `INTERNAL` (puerto 19092) | Para otros contenedores (Spark) | Se anuncia como `kafka:19092`, el nombre del servicio en la red de Docker |
| Listener `CONTROLLER` (puerto 9093) | Comunicación interna del controlador | No se expone |
| `...REPLICATION_FACTOR: 1` | Una sola copia de los temas internos | Hay un solo broker; con 3 brokers (producción) sería 3 |
| `volumes: kafka_data` | Guarda los datos fuera del contenedor | Los mensajes sobreviven a un reinicio |

**Pasos:**
1. Crea `docker-compose.yml` con el contenido de arriba.
2. Levanta Kafka: `docker compose up -d kafka`
3. Comprueba que corre: `docker compose ps` (el estado debe ser `running`).
4. Espera unos 10 segundos y mira el registro: `docker logs fraud-kafka --tail 20` (no debe haber errores; debe mencionar que el servidor inició).
5. Crea el topic:
   ```
   docker exec fraud-kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic transactions --partitions 3 --replication-factor 1
   ```
6. Descríbelo:
   ```
   docker exec fraud-kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic transactions
   ```

**Parámetros del topic:**

| Parámetro | Valor | Por qué | Otras opciones |
|---|---|---|---|
| `--partitions` | `3` | Permite paralelismo y muestra cómo se reparte por clave | `1` (sin paralelismo), muchas (más complejo de operar) |
| `--replication-factor` | `1` | Un solo broker local | `3` en producción (requiere 3 brokers) |
| Retención | 7 días (valor por defecto) | Suficiente para este proyecto | Más tiempo (más disco) |

**Verificación:** `docker compose ps` muestra `fraud-kafka` en `running`; el `--describe` muestra `PartitionCount: 3` y `ReplicationFactor: 1`.

**Si falla:**
- `port is already allocated` → otro programa usa el 9092; ciérralo o cambia el puerto de la izquierda en `ports`.
- El contenedor se detiene → `docker logs fraud-kafka` y revisa que copiaste todas las variables sin cambiar sangrías del YAML.
- `Connection refused` al crear el topic → espera 10 segundos más (el broker tarda en arrancar) y repite.

**Commit:** `git add docker-compose.yml && git commit -m "feat: add Kafka service with Docker Compose"`

---

## B3 - Producer de Kafka

**Objetivo:** un programa que lee el CSV original y envía cada transacción como un mensaje JSON al topic `transactions`.

**Archivos:** `src/ingestion/ingestion_config.py` y `src/ingestion/producer.py`.

### `ingestion_config.py` (constantes)

| Constante | Valor | Por qué | Otras opciones |
|---|---|---|---|
| `KAFKA_BOOTSTRAP_SERVERS` | `os.getenv("KAFKA_BOOTSTRAP_SERVERS", "localhost:9092")` | Dirección del broker, configurable por `.env` | Fija en el código (menos flexible) |
| `TOPIC_NAME` | `"transactions"` | Nombre del topic | - |
| `CSV_CHUNK_SIZE` | `50000` | El CSV se lee por partes para no cargar 6.3 millones de filas de golpe | Todo junto (usa mucha memoria) |
| `PRODUCER_ACKS` | `"all"` | Espera la confirmación de todas las réplicas: no se pierden mensajes | `"1"` (solo el líder), `"0"` (sin confirmación, rápido pero puede perder datos) |
| `PRODUCER_LINGER_MS` | `10` | Espera 10 ms para agrupar mensajes y enviar menos paquetes | `0` (envía cada mensaje al instante, menos eficiente) |
| `PRODUCER_COMPRESSION` | `"lz4"` | Compresión rápida, reduce el tráfico | `"none"`, `"gzip"` (más compacta y lenta), `"snappy"`, `"zstd"` |
| `PROGRESS_EVERY` | `100000` | Imprime avance cada 100,000 mensajes | - |

### `producer.py`

**Qué envía cada mensaje:** un JSON con **los 11 campos originales** (nombres originales del CSV, sin traducir) más
`transaction_id`: un número que sube desde 1 en el orden del archivo. Ese id es el mismo `id_transaccion` de la Etapa A,
y permite comparar resultados después (paso B9).

**Clave del mensaje:** `nameOrig` (cuenta de origen). Así todas las transacciones de una cuenta caen en la misma
partición y conservan su orden. Alternativa: sin clave (reparto al azar, no conserva orden por cuenta).

**Funciones:**

| Función | Qué hace |
|---|---|
| `build_producer()` | Crea el `Producer` de `confluent-kafka` con `bootstrap.servers`, `acks`, `linger.ms` y `compression.type` |
| `build_message(row, transaction_id)` | Devuelve un diccionario con `transaction_id` y los 11 campos, convirtiendo los tipos de numpy a tipos de Python (`int()`, `float()`, `str()`) |
| `delivery_report(error, message)` | Función de respuesta: cuenta mensajes entregados y fallidos |
| `send_chunk(producer, chunk_df, first_id, ...)` | Recorre el trozo del CSV, serializa con `json.dumps` y llama a `producer.produce(topic, key, value, on_delivery=...)`. Si la cola local se llena (`BufferError`), llama a `producer.poll(1)` y reintenta |
| `wait_if_needed(sent_count, start_time, messages_per_second)` | Si se pidió limitar la velocidad, duerme lo necesario para no pasar la tasa |
| `main()` | Lee los argumentos, recorre el CSV con `pd.read_csv(..., chunksize=CSV_CHUNK_SIZE, dtype=config.RAW_DTYPES)` (los tipos vienen de `src/ml/config.py`), llama a `send_chunk`, hace `producer.flush()` al final e imprime el resumen |

**Argumentos de línea de comandos** (con `argparse`):

| Argumento | Valor por defecto | Para qué |
|---|---|---|
| `--max-messages` | `0` (sin límite) | Enviar solo N mensajes (para pruebas) |
| `--messages-per-second` | `0` (sin límite) | Limitar la velocidad (para la demo en vivo) |

**Pasos:**
1. Crea los dos archivos con lo anterior.
2. Prueba con 1,000 mensajes: `python -m src.ingestion.producer --max-messages 1000`
3. Comprueba cuántos mensajes hay (suma de los offsets finales de las 3 particiones):
   ```
   docker exec fraud-kafka /opt/kafka/bin/kafka-get-offsets.sh --bootstrap-server localhost:9092 --topic transactions
   ```
   Imprime líneas `transactions:0:N`, `transactions:1:N`, `transactions:2:N`; la suma debe ser **1000**.
4. Lee 3 mensajes:
   ```
   docker exec fraud-kafka /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic transactions --from-beginning --max-messages 3
   ```
5. Ejecuta `ruff check src tests`.

**Verificación:** el producer imprime `entregados: 1000, fallidos: 0`; la suma de offsets es 1000; los mensajes se ven como
JSON con `transaction_id` y las 11 columnas originales.

**Importante:** esos 1,000 mensajes de prueba son las primeras 1,000 filas. En el paso B7 **se borra y se recrea el topic**
antes de la carga completa, para no duplicarlas.

**Si falla:** `NoBrokersAvailable` o error de conexión → Kafka no está corriendo (`docker compose ps`); `KeyError` en una columna →
revisa que `RAW_DTYPES` en `config.py` use los nombres originales del CSV.

**Commit:** `git add src && git commit -m "feat: add Kafka producer that streams the PaySim CSV"`

---

## B4 - Cuenta de Snowflake, estructura y llaves

**Objetivo:** crear la cuenta, el warehouse, la base de datos, el rol y un usuario de servicio que se conecta con una llave
(sin contraseña).

**Pasos:**
1. Crea una cuenta de prueba en https://signup.snowflake.com/ . Elige **AWS** como nube y la región **US East (N. Virginia)**
   (`us-east-1`: es la misma que usaremos en AWS). Confirma el correo y define tu usuario administrador.
2. Anota tu **identificador de cuenta** (formato `ORGANIZACION-CUENTA`, con guion). Se encuentra en Snowsight, en el menú de tu
   usuario → detalles de la cuenta (el nombre exacto del menú cambia; busca "account identifier"). Este valor va en `SNOWFLAKE_ACCOUNT`.
3. Crea el par de llaves. Hazlo en una carpeta **fuera del repositorio**, por ejemplo `~/.snowflake` (en Windows,
   `C:/Users/TU_USUARIO/.snowflake`; usa Git Bash para estos comandos):
   ```
   mkdir -p ~/.snowflake
   cd ~/.snowflake
   openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out rsa_key.p8 -nocrypt
   openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub
   ```
   - macOS/Linux: `chmod 600 rsa_key.p8` para que solo tú puedas leerla.
4. Obtén la llave pública en una sola línea y sin cabeceras:
   - macOS/Linux/Git Bash: `grep -v "PUBLIC KEY" rsa_key.pub | tr -d '\n'`
   - Windows PowerShell: `(Get-Content rsa_key.pub | Where-Object { $_ -notmatch "KEY" }) -join ""`
5. En Snowsight, abre un **SQL Worksheet** y ejecuta el script `infra/snowflake/01_setup.sql` (contenido abajo), reemplazando
   `YOUR_ADMIN_USER` por tu usuario administrador y `PASTE_PUBLIC_KEY_HERE` por la línea del punto 4.
6. Llena tu `.env`:
   ```
   SNOWFLAKE_ACCOUNT=ORGANIZACION-CUENTA
   SNOWFLAKE_USER=FRAUD_USER
   SNOWFLAKE_PRIVATE_KEY_PATH=C:/Users/TU_USUARIO/.snowflake/rsa_key.p8
   SNOWFLAKE_ROLE=FRAUD_ROLE
   SNOWFLAKE_WAREHOUSE=FRAUD_WH
   SNOWFLAKE_DATABASE=FRAUD_DB
   ```
   (En macOS/Linux la ruta es del tipo `/Users/tu_usuario/.snowflake/rsa_key.p8`.)

**Script `infra/snowflake/01_setup.sql`:**
```sql
USE ROLE ACCOUNTADMIN;

-- 1. Warehouse (el cómputo)
CREATE WAREHOUSE IF NOT EXISTS FRAUD_WH
  WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE;

-- 2. Tope de gasto
CREATE RESOURCE MONITOR IF NOT EXISTS FRAUD_MONITOR
  WITH CREDIT_QUOTA = 10
  TRIGGERS ON 80 PERCENT DO NOTIFY
           ON 100 PERCENT DO SUSPEND;
ALTER WAREHOUSE FRAUD_WH SET RESOURCE_MONITOR = FRAUD_MONITOR;

-- 3. Base de datos y esquemas
CREATE DATABASE IF NOT EXISTS FRAUD_DB DATA_RETENTION_TIME_IN_DAYS = 1;
CREATE SCHEMA IF NOT EXISTS FRAUD_DB.RAW;
CREATE SCHEMA IF NOT EXISTS FRAUD_DB.STAGING;
CREATE SCHEMA IF NOT EXISTS FRAUD_DB.MARTS;

-- 4. Rol con los permisos mínimos que necesita el proyecto
CREATE ROLE IF NOT EXISTS FRAUD_ROLE;
GRANT USAGE, OPERATE ON WAREHOUSE FRAUD_WH TO ROLE FRAUD_ROLE;
GRANT ALL PRIVILEGES ON DATABASE FRAUD_DB TO ROLE FRAUD_ROLE;
GRANT ALL PRIVILEGES ON ALL SCHEMAS IN DATABASE FRAUD_DB TO ROLE FRAUD_ROLE;
GRANT ALL PRIVILEGES ON FUTURE SCHEMAS IN DATABASE FRAUD_DB TO ROLE FRAUD_ROLE;
GRANT ALL PRIVILEGES ON FUTURE TABLES IN DATABASE FRAUD_DB TO ROLE FRAUD_ROLE;
GRANT ALL PRIVILEGES ON FUTURE VIEWS IN DATABASE FRAUD_DB TO ROLE FRAUD_ROLE;
GRANT ROLE FRAUD_ROLE TO USER YOUR_ADMIN_USER;

-- 5. Usuario de servicio con llave
CREATE USER IF NOT EXISTS FRAUD_USER
  DEFAULT_ROLE = FRAUD_ROLE
  DEFAULT_WAREHOUSE = FRAUD_WH
  DEFAULT_NAMESPACE = FRAUD_DB.RAW
  RSA_PUBLIC_KEY = 'PASTE_PUBLIC_KEY_HERE';
GRANT ROLE FRAUD_ROLE TO USER FRAUD_USER;
```

**Parámetros y decisiones** (detalle en `docs/tools/03_snowflake.md`):

| Parámetro | Valor | Por qué | Otras opciones |
|---|---|---|---|
| `WAREHOUSE_SIZE` | `XSMALL` | Suficiente para 6.3 millones de filas; el más barato | `SMALL` o mayor (gastan más créditos) |
| `AUTO_SUSPEND` | `60` | Se apaga al minuto de inactividad | `300`, o `0` (nunca; evitar) |
| `CREDIT_QUOTA` | `10` | Tope de créditos del monitor; ajústalo a lo que te dé tu prueba | Sin monitor (riesgo de gasto) |
| `DATA_RETENTION_TIME_IN_DAYS` | `1` | Un día de Time Travel; menos almacenamiento | `0` (sin Time Travel), más días |
| Rol propio `FRAUD_ROLE` | Permisos solo sobre `FRAUD_DB` | Mínimo privilegio: si se filtra la llave, el daño es limitado | Usar `ACCOUNTADMIN` (demasiado poder) |
| Autenticación | Par de llaves | Snowflake exige cada vez más MFA para contraseñas; las llaves sirven para programas | Contraseña (puede ser bloqueada o exigir MFA) |
| Llave sin contraseña (`-nocrypt`) | Sí | Más simple para un portafolio | Llave cifrada con frase (más segura; requiere pasar la frase a cada programa) |

Notas: `ACCOUNTADMIN` se usa **solo** en este script. Si tu cuenta soporta usuarios de tipo servicio, puedes añadir `TYPE = SERVICE` al
`CREATE USER` (consulta la documentación vigente de Snowflake).

**Verificación (en el worksheet):**
```sql
SHOW WAREHOUSES LIKE 'FRAUD_WH';
SHOW SCHEMAS IN DATABASE FRAUD_DB;
SHOW GRANTS TO ROLE FRAUD_ROLE;
DESC USER FRAUD_USER;
```
Debe verse el warehouse `XSMALL`; los esquemas `RAW`, `STAGING`, `MARTS` (más `INFORMATION_SCHEMA` y `PUBLIC`); los permisos del rol; y
`RSA_PUBLIC_KEY_FP` con un valor (no vacío) en `DESC USER`.

**Si falla:** el monitor de recursos o la creación del usuario fallan por permisos → confirma que la primera línea es `USE ROLE ACCOUNTADMIN;`.

**Commit:** `git add infra && git commit -m "feat: add Snowflake setup script"` (el script no debe contener tu llave real: antes de hacer commit, vuelve a poner `PASTE_PUBLIC_KEY_HERE` si la reemplazaste en el archivo).

---

## B5 - Probar la conexión Python → Snowflake

**Archivo:** `src/common/snowflake_connection.py`, un módulo compartido que reutilizarán los scripts de verificación, el scoring y el dashboard (Spark y dbt tienen su propia conexión).

**Funciones:**

| Función | Qué hace |
|---|---|
| `get_snowflake_connection()` | Llama a `load_dotenv()`, lee las variables `SNOWFLAKE_*` con `os.getenv`, y devuelve `snowflake.connector.connect(account, user, private_key_file, role, warehouse, database, login_timeout=30)` |
| `main()` | Abre la conexión, ejecuta `SELECT CURRENT_USER(), CURRENT_ROLE(), CURRENT_WAREHOUSE(), CURRENT_DATABASE()` y lo imprime; cierra la conexión |

Si falta alguna variable de entorno, `get_snowflake_connection()` lanza un error que dice cuál falta.

| Parámetro | Valor | Por qué | Otras opciones |
|---|---|---|---|
| `private_key_file` | Ruta del `.p8` | Lo más simple: el conector lee el archivo | Pasar la llave como bytes (`private_key`), útil en servidores sin archivos |
| `login_timeout` | `30` | Falla rápido si hay un problema de red | Valor por defecto (espera más) |

**Ejecución:** `python -m src.common.snowflake_connection`

**Verificación:** imprime `FRAUD_USER`, `FRAUD_ROLE`, `FRAUD_WH`, `FRAUD_DB`.

**Si falla:**
- `JWT token is invalid` → la llave pública del usuario no coincide con tu llave privada, o el identificador de cuenta está mal (debe ser `ORGANIZACION-CUENTA`). Repite B4, pasos 3 a 5.
- `Role 'FRAUD_ROLE' specified in the connect string is not granted` → falta el `GRANT ROLE ... TO USER FRAUD_USER`.
- `FileNotFoundError` → revisa `SNOWFLAKE_PRIVATE_KEY_PATH` en `.env` (barras `/`).

**Commit:** `git add src && git commit -m "feat: add shared Snowflake connection helper"`

---

## B6 - Crear la tabla RAW

**Archivo:** `infra/snowflake/02_raw_tables.sql`

```sql
USE ROLE FRAUD_ROLE;
USE WAREHOUSE FRAUD_WH;
USE SCHEMA FRAUD_DB.RAW;

CREATE TABLE IF NOT EXISTS TRANSACTIONS (
  TRANSACTION_ID    NUMBER(10,0),
  STEP              NUMBER(5,0),
  TYPE              VARCHAR,
  AMOUNT            FLOAT,
  NAME_ORIG         VARCHAR,
  OLDBALANCE_ORG    FLOAT,
  NEWBALANCE_ORIG   FLOAT,
  NAME_DEST         VARCHAR,
  OLDBALANCE_DEST   FLOAT,
  NEWBALANCE_DEST   FLOAT,
  IS_FRAUD          NUMBER(1,0),
  IS_FLAGGED_FRAUD  NUMBER(1,0),
  KAFKA_TIMESTAMP   TIMESTAMP_NTZ
);
```

| Decisión | Valor | Por qué | Alternativa |
|---|---|---|---|
| Dinero en RAW | `FLOAT` | RAW refleja lo que llega (decimales de punto flotante) | `NUMBER(18,2)`: exacto, pero RAW debería no transformar nada |
| Dinero en STAGING | `NUMBER(18,2)` (se convierte en dbt) | El dinero debe ser exacto | Seguir con FLOAT (acumula errores de redondeo) |
| `KAFKA_TIMESTAMP` | Marca de tiempo del mensaje | Permite quedarse con el mensaje más reciente si hay duplicados | Fecha de carga (`CURRENT_TIMESTAMP`) |
| Sin clave primaria forzada | Snowflake no obliga unicidad | Se garantiza en STAGING con un test y deduplicación | - |

Ejecuta el script en un worksheet de Snowsight.

**Verificación:** `DESC TABLE FRAUD_DB.RAW.TRANSACTIONS;` muestra las 13 columnas con esos tipos.

**Commit:** `git add infra && git commit -m "feat: add RAW transactions table"`

---

## B7 - Job de Spark en Docker (carga completa a RAW)

**Objetivo:** Spark lee el topic de Kafka y escribe cada micro-lote en `RAW.TRANSACTIONS`.

**Archivos:** `docker/spark/Dockerfile`, `src/streaming/streaming_config.py`, `src/streaming/stream_to_snowflake.py` y un servicio nuevo en `docker-compose.yml`.

### `docker/spark/Dockerfile`
```dockerfile
FROM python:3.11-slim-bookworm
RUN apt-get update \
    && apt-get install -y --no-install-recommends openjdk-17-jre-headless \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /app
RUN pip install --no-cache-dir pyspark==3.5.3 python-dotenv
COPY src /app/src
CMD ["python", "-m", "src.streaming.stream_to_snowflake", "--mode", "load"]
```

| Elemento | Por qué | Otras opciones |
|---|---|---|
| `python:3.11-slim-bookworm` | Debian 12 fija: tiene `openjdk-17` en sus repositorios | `python:3.11-slim` (puede cambiar de Debian y perder Java 17) |
| `openjdk-17-jre-headless` | Java 17 es compatible con Spark 3.5; sin interfaz gráfica | Java 11 o 21 |
| `pyspark==3.5.3` | Rama 3.5 estable (Scala 2.12) compatible con los conectores | Spark 4.x (los conectores de Kafka/Snowflake tienen otras versiones; no lo usamos) |

### Servicio `spark-job` (se añade a `docker-compose.yml`)
```yaml
  spark-job:
    build:
      context: .
      dockerfile: docker/spark/Dockerfile
    container_name: fraud-spark-job
    depends_on:
      - kafka
    env_file:
      - .env
    environment:
      KAFKA_BOOTSTRAP_SERVERS: kafka:19092
      SNOWFLAKE_PRIVATE_KEY_PATH: /secrets/rsa_key.p8
    volumes:
      - ${SNOWFLAKE_PRIVATE_KEY_PATH}:/secrets/rsa_key.p8:ro
      - ./checkpoints:/app/checkpoints
      - spark_ivy_cache:/root/.ivy2
    profiles:
      - spark
```
Y en la sección `volumes:` del final del archivo, agrega `spark_ivy_cache:`.

| Elemento | Por qué |
|---|---|
| `KAFKA_BOOTSTRAP_SERVERS: kafka:19092` | Dentro de la red de Docker, Spark llega a Kafka por su nombre de servicio y el listener `INTERNAL` |
| `SNOWFLAKE_PRIVATE_KEY_PATH: /secrets/rsa_key.p8` | Dentro del contenedor la llave está en esa ruta (se monta desde tu carpeta `.snowflake`, solo lectura) |
| `./checkpoints:/app/checkpoints` | El progreso de Spark sobrevive si el contenedor se detiene |
| `spark_ivy_cache` | Guarda las librerías Java descargadas (evita bajarlas cada vez) |
| `profiles: [spark]` | No arranca con `docker compose up`; se lanza solo con `docker compose run` |

### `streaming_config.py` (constantes)

| Constante | Valor | Por qué | Otras opciones |
|---|---|---|---|
| `KAFKA_TOPIC` | `"transactions"` | - | - |
| `STARTING_OFFSETS` | `"earliest"` | Lee desde el principio la primera vez | `"latest"` (solo mensajes nuevos) |
| `MAX_OFFSETS_PER_TRIGGER` | `50000` | Máximo de mensajes por lote: controla memoria y tamaño de cada escritura | Sin límite (puede saturar la PC), `10000` (más lotes, más lento) |
| `LIVE_TRIGGER_INTERVAL` | `"10 seconds"` | Cada cuánto se procesa un lote en modo en vivo | `"1 minute"` (menos escrituras), `"1 second"` (muy frecuente para Snowflake) |
| `CHECKPOINT_DIR` | `"checkpoints/transactions"` | Guarda el progreso | Ruta en S3 (producción) |
| `SHUFFLE_PARTITIONS` | `4` | Particiones al reagrupar; el valor por defecto (200) es excesivo en una sola máquina | `200` |
| `DRIVER_MEMORY` | `"2g"` | Memoria del proceso de Spark | `"1g"` (puede quedarse corta), `"4g"` (si Docker tiene RAM de sobra) |
| `SPARK_PACKAGES` | `org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.3` y `net.snowflake:spark-snowflake_2.12:2.16.0-spark_3.5` | Conectores de Kafka y Snowflake (Scala 2.12, Spark 3.5) | Verifica en https://mvnrepository.com/artifact/net.snowflake/spark-snowflake_2.12 la versión vigente con sufijo `-spark_3.5` |
| `SNOWFLAKE_SCHEMA` / `SNOWFLAKE_TABLE` | `"RAW"` / `"TRANSACTIONS"` | Destino | - |
| `MESSAGE_SCHEMA` | `StructType` con los 12 campos del JSON (`transaction_id` entero; `step` entero; `type` texto; los montos y saldos `DoubleType`; `isFraud` e `isFlaggedFraud` enteros; nombres de cuenta texto) | Esquema explícito: los datos entran con tipos definidos | `inferSchema` (más lento y menos seguro) |
| `SNOWFLAKE_COLUMN_NAMES` | Diccionario nombre original → nombre de Snowflake (por ejemplo `nameOrig` → `NAME_ORIG`, `oldbalanceOrg` → `OLDBALANCE_ORG`, `isFraud` → `IS_FRAUD`, `transaction_id` → `TRANSACTION_ID`) | Coincide exactamente con las columnas de la tabla RAW | - |

### `stream_to_snowflake.py`

**Funciones:**

| Función | Qué hace |
|---|---|
| `build_spark_session()` | `SparkSession.builder` con `master("local[*]")`, `spark.jars.packages`, `spark.sql.shuffle.partitions` y `spark.driver.memory` |
| `read_private_key_for_spark(key_path)` | Lee el `.p8`, quita las líneas `-----BEGIN/END PRIVATE KEY-----` y los saltos de línea, y devuelve el texto en una sola línea (formato que pide el conector en `pem_private_key`) |
| `build_snowflake_options()` | Diccionario con `sfURL` (`<cuenta>.snowflakecomputing.com`), `sfUser`, `pem_private_key`, `sfDatabase`, `sfSchema`, `sfWarehouse`, `sfRole` |
| `read_kafka_stream(spark)` | `spark.readStream.format("kafka")` con `kafka.bootstrap.servers`, `subscribe`, `startingOffsets` y `maxOffsetsPerTrigger` |
| `parse_messages(kafka_df)` | `from_json(col("value").cast("string"), MESSAGE_SCHEMA)`; expande los campos; agrega `kafka_timestamp` (columna `timestamp` de Kafka); renombra todo con `SNOWFLAKE_COLUMN_NAMES` |
| `write_batch_to_snowflake(batch_df, batch_id)` | Si el lote está vacío, no hace nada; si no, escribe con `format("net.snowflake.spark.snowflake")`, `options(...)`, `dbtable`, `mode("append")`; imprime cuántas filas escribió |
| `main()` | Lee el argumento `--mode`; arma el stream; `writeStream.foreachBatch(...)`; con `load` usa `trigger(availableNow=True)` (procesa todo lo disponible y termina); con `live` usa `trigger(processingTime=LIVE_TRIGGER_INTERVAL)` (no termina); `checkpointLocation`; `awaitTermination()` |

| Parámetro | Valor | Por qué | Otras opciones |
|---|---|---|---|
| `--mode load` | `availableNow=True` | Carga todo el histórico y se detiene: ideal para la carga inicial | `live` para la demo continua |
| `outputMode` | `append` (el predeterminado con `foreachBatch`) | Solo se agregan filas nuevas | `update`, `complete` (para agregaciones) |
| `foreachBatch` | Escribe cada lote con el conector de Snowflake | Control total sobre la escritura | `writeStream.format(...)` directo (el conector de Snowflake no lo soporta como destino de streaming) |
| `mode("append")` | Agrega a la tabla | No borra lo anterior | `overwrite` (la reemplazaría) |

**Detalle importante: entrega "al menos una vez".** Si un lote se reintenta tras una falla, podrían duplicarse filas en RAW.
Por eso `STAGING` deduplica por `transaction_id` y tiene un test de unicidad.

### Pasos

1. Crea los archivos y el servicio. Verifica la versión de los paquetes en `SPARK_PACKAGES` (ver tabla).
2. Construye la imagen (la primera vez tarda varios minutos): `docker compose build spark-job`
3. **Reinicia el topic** para borrar los 1,000 mensajes de prueba de B3:
   ```
   docker exec fraud-kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic transactions
   docker exec fraud-kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic transactions --partitions 3 --replication-factor 1
   ```
4. **Deja RAW vacía y sin checkpoint** (por si hubo pruebas): en un worksheet con rol `FRAUD_ROLE`:
   `TRUNCATE TABLE FRAUD_DB.RAW.TRANSACTIONS;` y borra la carpeta `checkpoints/transactions` si existe.
5. **Terminal 1: carga completa al topic** (sin límite de velocidad): `python -m src.ingestion.producer`
   (puede tardar varios minutos). Al terminar, la suma de offsets debe ser **6,362,620** (comando de B3, punto 3).
6. **Terminal 2: ejecuta el job:** `docker compose run --rm spark-job`
   - Verás el avance por lotes (unos 128 lotes de hasta 50,000 filas). Puede tardar entre 20 y 60 minutos.
   - La primera vez descarga las librerías Java (verás mensajes de `ivy`).
7. Cuando el contenedor termine solo (modo `load`), verifica en Snowsight con el rol `FRAUD_ROLE`:
   ```sql
   SELECT COUNT(*) AS rows_total,
          COUNT(DISTINCT TRANSACTION_ID) AS rows_unique,
          SUM(IS_FRAUD) AS frauds
   FROM FRAUD_DB.RAW.TRANSACTIONS;
   ```
8. **(Opcional) Demo en vivo para el video:** vacía el topic y la tabla como en los puntos 3 y 4 (y borra el checkpoint), ejecuta
   `python -m src.ingestion.producer --messages-per-second 200` en una terminal y
   `docker compose run --rm spark-job python -m src.streaming.stream_to_snowflake --mode live` en otra. Detén ambos con `Ctrl + C`.
   Después, repite la carga completa si quieres volver a tener los 6.3 millones de filas.

**Verificación:** `rows_total = 6362620`, `rows_unique = 6362620`, `frauds = 8213`. Si `rows_total` es mayor que `rows_unique`,
hubo reintentos y hay duplicados: no pasa nada, STAGING los elimina (pero anótalo).

**Si falla:**
| Síntoma | Causa probable | Solución |
|---|---|---|
| `Failed to find data source: kafka` | No se descargó el paquete de Kafka | Revisa conexión a internet y `SPARK_PACKAGES`; vuelve a ejecutar |
| `Connection refused` a Kafka | El job usa la dirección equivocada | Dentro del contenedor debe ser `kafka:19092` (variable del servicio) |
| Error de autenticación de Snowflake (JWT) | Llave mal leída | Revisa que `read_private_key_for_spark` quite cabeceras y saltos de línea; revisa `SNOWFLAKE_ACCOUNT` |
| `java.lang.OutOfMemoryError` | Poca memoria | Baja `MAX_OFFSETS_PER_TRIGGER` a `20000`, o sube la memoria de Docker Desktop |
| Mucho más lento de lo esperado | Lotes pequeños o red lenta | Es normal que la primera carga tarde; no interrumpas |
| Error de volumen en Windows | Ruta de la llave con `\` | Usa `/` en `SNOWFLAKE_PRIVATE_KEY_PATH` |
| Versión de conector no existe | Coordenadas de Maven desactualizadas | Consulta Maven Central (enlace de la tabla) |

**Commit:** `git add docker docker-compose.yml src && git commit -m "feat: add Spark streaming job that loads Kafka into Snowflake RAW"`

---

## B8 - Proyecto dbt: STAGING, MARTS y tests

**Objetivo:** limpiar y traducir los datos (STAGING) y calcular las 15 features (MARTS), con pruebas automáticas.

**Cómo se ejecuta dbt en este proyecto:** dentro de la carpeta `dbt_project/`, con las variables de `.env` cargadas por `python-dotenv`:
```
cd dbt_project
dotenv -f ../.env run -- dbt <comando>
```
(`dbt` busca `profiles.yml` en la carpeta actual.) A continuación se escribe `dbt <comando>` por brevedad, pero siempre con ese prefijo.

### Archivos y contenido

**`dbt_project/dbt_project.yml`**
```yaml
name: fraud_project
version: "1.0.0"
config-version: 2
profile: fraud_project

model-paths: ["models"]
test-paths: ["tests"]
macro-paths: ["macros"]

models:
  fraud_project:
    staging:
      +schema: STAGING
      +materialized: view
    marts:
      +schema: MARTS
      +materialized: table
```

**`dbt_project/profiles.yml`** (no contiene secretos: todo viene de variables de entorno)
```yaml
fraud_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      private_key_path: "{{ env_var('SNOWFLAKE_PRIVATE_KEY_PATH') }}"
      role: "{{ env_var('SNOWFLAKE_ROLE') }}"
      warehouse: "{{ env_var('SNOWFLAKE_WAREHOUSE') }}"
      database: "{{ env_var('SNOWFLAKE_DATABASE') }}"
      schema: STAGING
      threads: 4
```

**`dbt_project/macros/generate_schema_name.sql`**: hace que `+schema: MARTS` cree el esquema `MARTS` y no `STAGING_MARTS` (por defecto dbt concatena).
```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- if custom_schema_name is none -%}
        {{ target.schema }}
    {%- else -%}
        {{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

**`dbt_project/models/staging/sources.yml`**
```yaml
version: 2

sources:
  - name: raw
    schema: RAW
    tables:
      - name: transactions
```

### Modelo `stg_transactions.sql` (carpeta `models/staging/`)

Toma `source('raw', 'transactions')`, **deduplica** (`QUALIFY ROW_NUMBER() OVER (PARTITION BY transaction_id ORDER BY kafka_timestamp DESC) = 1`),
traduce y convierte tipos:

| Columna de salida | Expresión | Nota |
|---|---|---|
| `id_transaccion` | `transaction_id` | |
| `hora_simulada` | `step` | |
| `tipo` | `CASE type WHEN 'CASH_IN' THEN 'deposito' WHEN 'CASH_OUT' THEN 'retiro' WHEN 'DEBIT' THEN 'debito' WHEN 'PAYMENT' THEN 'pago' WHEN 'TRANSFER' THEN 'transferencia' END` | Misma traducción que `TYPE_VALUE_MAP` |
| `monto` | `amount::NUMBER(18,2)` | Dinero exacto |
| `cuenta_origen` | `name_orig` | |
| `saldo_inicial_origen` | `oldbalance_org::NUMBER(18,2)` | |
| `saldo_final_origen` | `newbalance_orig::NUMBER(18,2)` | |
| `cuenta_destino` | `name_dest` | |
| `saldo_inicial_destino` | `oldbalance_dest::NUMBER(18,2)` | |
| `saldo_final_destino` | `newbalance_dest::NUMBER(18,2)` | |
| `es_fraude` | `is_fraud` | |
| `es_marcado_por_regla` | `is_flagged_fraud` | |

### Modelo `transaction_features.sql` (carpeta `models/marts/`)

Lee `ref('stg_transactions')` y devuelve `id_transaccion`, `hora_simulada`, `es_fraude`, `es_marcado_por_regla` y las 15 features,
**con exactamente las mismas definiciones de A7**:

| Columna | Expresión SQL |
|---|---|
| `monto`, `saldo_inicial_origen`, `saldo_final_origen`, `saldo_inicial_destino`, `saldo_final_destino` | Igual que en staging |
| `hora_del_dia` | `MOD(hora_simulada, 24)` |
| `error_saldo_origen` | `saldo_final_origen + monto - saldo_inicial_origen` |
| `error_saldo_destino` | `saldo_inicial_destino + monto - saldo_final_destino` |
| `vacia_cuenta_origen` | `IFF(saldo_inicial_origen > 0 AND saldo_final_origen = 0, 1, 0)` |
| `destino_sin_saldo_previo` | `IFF(saldo_inicial_destino = 0, 1, 0)` |
| `tipo_deposito` | `IFF(tipo = 'deposito', 1, 0)` |
| `tipo_retiro` | `IFF(tipo = 'retiro', 1, 0)` |
| `tipo_debito` | `IFF(tipo = 'debito', 1, 0)` |
| `tipo_pago` | `IFF(tipo = 'pago', 1, 0)` |
| `tipo_transferencia` | `IFF(tipo = 'transferencia', 1, 0)` |

### Tests

**`models/staging/schema.yml`** (tests genéricos de `stg_transactions`):

| Columna | Tests | Qué comprueba |
|---|---|---|
| `id_transaccion` | `unique`, `not_null` | Sin duplicados ni nulos tras la deduplicación |
| `tipo` | `not_null`, `accepted_values` (`deposito`, `retiro`, `debito`, `pago`, `transferencia`) | Todos los tipos se tradujeron |
| `monto` | `not_null` | |
| `es_fraude` | `accepted_values` (0, 1) | La etiqueta es válida |

**`models/marts/schema.yml`** (tests de `transaction_features`):

| Columna | Tests |
|---|---|
| `id_transaccion` | `unique`, `not_null` |
| Las 15 features | `not_null` |
| `vacia_cuenta_origen`, `destino_sin_saldo_previo`, `tipo_*` | `accepted_values` (0, 1) |

**Tests singulares** (archivos `.sql` en `dbt_project/tests/`; fallan si devuelven alguna fila):
- `assert_one_transaction_type_per_row.sql`: selecciona las filas de `transaction_features` donde la suma de los 5 `tipo_*` es distinta de 1.
- `assert_full_load_row_count.sql` con `{{ config(tags=['full_load']) }}`: devuelve una fila si `COUNT(*)` de `stg_transactions` es distinto de 6,362,620.
  Solo tiene sentido tras la carga completa; más adelante se excluye con `--exclude tag:full_load`.

### Pasos

1. Crea los archivos anteriores dentro de `dbt_project/` (carpetas `models/staging`, `models/marts`, `macros`, `tests`).
2. Entra a la carpeta: `cd dbt_project`
3. Comprueba la conexión: `dotenv -f ../.env run -- dbt debug` → debe terminar con `All checks passed!`.
4. Ejecuta todo (modelos y tests, en orden de dependencias): `dotenv -f ../.env run -- dbt build`
5. (Opcional) Documentación con diagrama de linaje: `dotenv -f ../.env run -- dbt docs generate` y luego `dbt docs serve` (puerto 8080; ciérralo con `Ctrl + C` antes de la Etapa C, donde Airflow usa ese puerto).
6. Comprueba en Snowsight:
   ```sql
   SELECT COUNT(*) FROM FRAUD_DB.STAGING.STG_TRANSACTIONS;
   SELECT COUNT(*) FROM FRAUD_DB.MARTS.TRANSACTION_FEATURES;
   ```

**Parámetros y decisiones:**

| Decisión | Valor | Por qué | Otras opciones |
|---|---|---|---|
| Materialización de staging | `view` | Es una transformación ligera y siempre está actualizada | `table` (más rápido de leer, hay que reconstruirla) |
| Materialización de marts | `table` | Es lo que consumen modelos y dashboard: conviene tenerla ya calculada | `view` (lenta con 6.3 millones de filas), `incremental` (se aplica en la Etapa C) |
| `threads` | `4` | 4 modelos en paralelo; con un warehouse XSMALL es suficiente | `1` (más lento), más de 8 (más consumo) |
| Deduplicar con `QUALIFY` | Sí | Elimina duplicados por reintentos de Spark | Dejar duplicados (rompe el test de unicidad) |
| Perfil sin secretos | `env_var(...)` | Se puede subir a Git | Poner credenciales en `profiles.yml` (nunca) |

**Verificación:** `dbt debug` pasa; `dbt build` termina con todos los modelos y tests en `PASS` (0 `ERROR`, 0 `FAIL`); ambos conteos son
**6,362,620**.

**Si falla:**
- `Env var required but not provided` → olvidaste el prefijo `dotenv -f ../.env run --`.
- Un test `FAIL` en `unique` → revisa que la deduplicación esté en `stg_transactions`.
- `accepted_values` falla en `tipo` → algún valor de `type` en RAW no está en el `CASE`; consulta `SELECT DISTINCT TYPE FROM FRAUD_DB.RAW.TRANSACTIONS`.
- Un test falla porque RAW no está completa → repite el paso B7.

**Commit:** vuelve a la raíz (`cd ..`) y `git add dbt_project && git commit -m "feat: add dbt staging and marts models with tests"`

---

## B9 - Verificar que dbt calcula lo mismo que pandas

**Objetivo:** comprobar con evidencia que las features de Snowflake (dbt) son iguales a las de la Etapa A (pandas). Si no lo son, el modelo
entrenado en la Etapa A no es válido para los datos de producción.

**Archivo:** `src/ml/verify_dbt_features.py`

**Constantes:** `SAMPLE_MODULUS = 50` (se compara una de cada 50 filas, unas 127,000), `MONEY_TOLERANCE = 0.01`, `SUM_RELATIVE_TOLERANCE = 1e-6`.

**Funciones:**

| Función | Qué hace |
|---|---|
| `read_pandas_features()` | Lee `data/processed/transactions_features.parquet` (Etapa A) |
| `read_snowflake_sample(connection)` | Ejecuta `SELECT ... FROM MARTS.TRANSACTION_FEATURES WHERE MOD(ID_TRANSACCION, 50) = 0 ORDER BY ID_TRANSACCION` y lo pasa a pandas con `cursor.fetch_pandas_all()`; pone los nombres de columna en minúsculas |
| `compare_row_counts(...)` | Compara el total de filas de ambas fuentes (en Snowflake con `SELECT COUNT(*)`) |
| `compare_column_sums(...)` | Compara la suma de cada una de las 16 columnas (`es_fraude` y las 15 features) con `np.isclose(rtol=1e-6)`. La suma en Snowflake se calcula con `SELECT SUM(col) ...` |
| `compare_sample_values(...)` | Une ambas muestras por `id_transaccion`; compara columna por columna: las banderas y `hora_del_dia` exactas, el dinero con tolerancia de 0.01 (`np.isclose(atol=0.01)`) |
| `main()` | Ejecuta las tres comparaciones, imprime una tabla (columna, resultado) y termina con código distinto de 0 si algo falla (`sys.exit(1)`) |

**Por qué hay tolerancia:** en pandas el dinero es `float64` (puede dejar residuos como 1e-9); en Snowflake es `NUMBER(18,2)` exacto.

**Ejecución:** `python -m src.ml.verify_dbt_features`

**Verificación:** las filas coinciden (6,362,620), las 16 sumas coinciden, la muestra no tiene diferencias y el script termina con `TODO COINCIDE`.

**Si falla:** el script indica la columna. Causas habituales: una fórmula distinta en dbt (compara con la tabla del paso A7), un `CASE` mal escrito en `tipo`, o RAW incompleta.

**Commit:** `git add src && git commit -m "test: verify dbt features match pandas features"`

---

## B10 - Reentrenar con datos de Snowflake y comparar con la Etapa A

**Objetivo:** demostrar que el flujo completo (Kafka → Spark → Snowflake → dbt) no alteró los datos: si se entrena con lo que sale de
Snowflake, los resultados deben ser prácticamente iguales a los de la Etapa A.

**Nota de honestidad:** aquí la prueba (`test`) se usa por segunda vez, pero solo para comprobar que el flujo es equivalente. No se elige
ni se ajusta ningún modelo con ella.

### Cambio en `src/ml/config.py` (un solo bloque nuevo)

Se añade, **antes** de las rutas derivadas de `PROCESSED_DIR`, `MODELS_DIR` y `REPORTS_DIR`, un interruptor que separa los resultados:

| Elemento | Valor | Para qué |
|---|---|---|
| `DATA_SOURCE` | `os.getenv("DATA_SOURCE", "local")` | `"local"` = datos de la Etapa A; `"snowflake"` = datos exportados desde Snowflake |
| Si es `"snowflake"` | `PROCESSED_DIR = data/processed/snowflake`, `MODELS_DIR = models/snowflake`, `REPORTS_DIR = reports/snowflake` y `MLFLOW_EXPERIMENT_NAME = "fraud_detection_snowflake"` | Los resultados de la Etapa A **no se tocan** |

(`FIGURES_DIR` se deriva de `REPORTS_DIR`.) `create_directories()` ya crea las carpetas nuevas.

### Script `src/ml/export_features_from_snowflake.py`

| Función | Qué hace |
|---|---|
| `build_query()` | `SELECT id_transaccion, hora_simulada, es_fraude, es_marcado_por_regla` y las 15 features `FROM MARTS.TRANSACTION_FEATURES ORDER BY id_transaccion` |
| `export_in_batches(connection, output_path)` | `cursor.fetch_pandas_batches()`; por cada lote: nombres en minúsculas, tipos iguales a los de la Etapa A (banderas `int8`, `hora_del_dia` `int8`, dinero `float64`, `hora_simulada` `int16`, `id_transaccion` `int32`) y escritura incremental con `pyarrow.parquet.ParquetWriter` |
| `main()` | Exige `DATA_SOURCE=snowflake` (si no, se detiene con un mensaje) y guarda en `config.FEATURES_PARQUET_PATH` |

Se escribe por lotes para no cargar toda la tabla en memoria.

### Pasos

1. Haz el cambio de `config.py` y crea el script de exportación.
2. En la **misma terminal** donde ejecutarás todo lo demás, activa el modo Snowflake:
   - macOS/Linux: `export DATA_SOURCE=snowflake`
   - Windows PowerShell: `$env:DATA_SOURCE = "snowflake"`
3. Ejecuta, en este orden (son los mismos scripts de la Etapa A):
   ```
   python -m src.ml.export_features_from_snowflake
   python -m src.ml.split_data
   python -m src.ml.scaling
   python -m src.ml.train_baseline
   python -m src.ml.train_xgboost
   python -m src.ml.train_autoencoder
   python -m src.ml.compare_models
   ```
4. Crea `reports/etapa_B_resultados.md` con una tabla que compare, para cada modelo, `valid_pr_auc` y `test_pr_auc` de la Etapa A (`reports/model_comparison.md`) y de Snowflake (`reports/snowflake/model_comparison.md`), y la diferencia absoluta.
5. Si la comparación es favorable, registra el modelo entrenado con datos de Snowflake como nueva versión del campeón:
   ```
   python -m src.ml.register_champion xgboost
   ```
   (con `DATA_SOURCE=snowflake` aún activo: lee `models/snowflake/model_info.json` y registra una **versión nueva** de `fraud_model` con el alias `champion`.)
   El campeón en producción será así el modelo que viene del flujo completo.
6. Desactiva el modo (`unset DATA_SOURCE` o `Remove-Item Env:DATA_SOURCE`) o cierra la terminal.

**Criterios de equivalencia:**
1. Los cortes de la partición (horas) son idénticos a los de la Etapa A.
2. Para cada uno de los tres modelos: |PR-AUC de prueba (Snowflake) − PR-AUC de prueba (Etapa A)| ≤ 0.01.
3. El campeón elegido por validación es el mismo modelo que en la Etapa A.

**Verificación:** los tres criterios se cumplen y están anotados en `reports/etapa_B_resultados.md`. Si alguno falla, **no sigas**: vuelve a B9, porque
algo cambió entre los datos de la Etapa A y los de Snowflake.

**Commit:** `git add src reports && git commit -m "feat: retrain with Snowflake data and compare with stage A"`

---

## B11 - Cierre de la etapa

**Checklist final:**
- [ ] `RAW.TRANSACTIONS` con 6,362,620 filas.
- [ ] `dbt build` en verde (modelos y tests).
- [ ] `verify_dbt_features` termina con `TODO COINCIDE`.
- [ ] `reports/etapa_B_resultados.md` completo con los 3 criterios cumplidos.
- [ ] `fraud_model` tiene una versión nueva con el alias `champion` (visible en MLflow, pestaña *Models*).
- [ ] `python -m pytest tests -v` y `ruff check src tests` sin errores.
- [ ] `.env`, llaves y `checkpoints/` no están en Git (`git status` limpio).
- [ ] Apagaste lo que no necesitas: `docker compose down` y comprobaste que el warehouse de Snowflake figura suspendido.

**Pasos finales:**
```
git add .
git commit -m "docs: stage B results"
git tag etapa-b-completa
git push
git push --tags
```

**Qué se lleva a la Etapa C:**
1. La tabla `MARTS.TRANSACTION_FEATURES` y el modelo `fraud_model@champion` en MLflow.
2. `src/common/snowflake_connection.py` (conexión compartida).
3. `dbt_project/` (se amplía con la materialización incremental y la tabla de alertas).
4. El servicio `kafka` y la imagen de Spark: no se usan en la Etapa C, pero quedan para la demo.

---

## Problemas frecuentes de la etapa

| Síntoma | Causa probable | Solución |
|---|---|---|
| `docker: command not found` | Docker Desktop no está abierto o instalado | Ábrelo y espera a que diga "running" |
| `Cannot connect to the Docker daemon` | Docker Desktop apagado | Ábrelo |
| `dotenv: command not found` | Falta el extra `cli` | `pip install "python-dotenv[cli]"` |
| `JWT token is invalid` (Python, Spark o dbt) | Llave o identificador de cuenta incorrectos | Repite B4 pasos 2 a 6 |
| `Insufficient privileges` en Snowflake | Rol equivocado | Usa `FRAUD_ROLE` (no `ACCOUNTADMIN`) para todo menos `01_setup.sql` |
| La carga a Snowflake no avanza | Warehouse suspendido o monitor de recursos en 100% | Revisa el estado del warehouse y el consumo de créditos |
| `ruff` marca errores | Código sin formatear | `ruff format src tests` y `ruff check src tests` |

---

## Mapa final de archivos de la Etapa B

```
docker-compose.yml                       # B2 y B7
docker/spark/Dockerfile                  # B7
infra/snowflake/01_setup.sql             # B4
infra/snowflake/02_raw_tables.sql        # B6
src/common/snowflake_connection.py       # B5
src/ingestion/ingestion_config.py        # B3
src/ingestion/producer.py                # B3
src/streaming/streaming_config.py        # B7
src/streaming/stream_to_snowflake.py     # B7
src/ml/verify_dbt_features.py            # B9
src/ml/export_features_from_snowflake.py # B10
src/ml/config.py                         # B10 (bloque DATA_SOURCE)
dbt_project/                             # B8 (proyecto completo)
reports/etapa_B_resultados.md            # B10
```

---

**Siguiente etapa:** [Etapa C - Modelo en producción](etapa_C_modelo_en_produccion.md).
