# Etapa C - Modelo en producción (Airflow + FastAPI + pgvector + LangGraph + Claude)

**Objetivo:** poner el modelo campeón a trabajar. Un flujo programado puntúa las transacciones nuevas y guarda las alertas;
una API permite consultar el modelo y pedir una explicación en lenguaje simple generada por un agente de IA.

**Prerrequisito:** Etapa B terminada (`git tag etapa-b-completa`): `MARTS.TRANSACTION_FEATURES` con 6,362,620 filas y el modelo
`fraud_model` con el alias `champion` en MLflow.

**Resultado final de la etapa:**
1. Las tablas `MARTS.TRANSACTION_SCORES` y la vista `MARTS.FRAUD_ALERTS` en Snowflake.
2. Un DAG de Airflow (`fraud_scoring_pipeline`) que corre cada 15 minutos: construye features, puntúa lo nuevo y valida las alertas.
3. Una base pgvector con tipologías de fraude y un agente que explica alertas usando Claude.
4. Una API con `/health`, `/predict` y `/explain`, con pruebas automáticas.

**Qué se usa:** Airflow (Docker), dbt, Snowflake, XGBoost, PostgreSQL + pgvector (Docker), sentence-transformers, LangGraph, Claude API, FastAPI.

**Suposición importante:** esta etapa asume que el campeón es **XGBoost** (lo esperable en este tipo de datos). Si en la Etapa A ganó otro
modelo, avísame antes de empezar, porque el código de scoring y de la API cambia.

**Regla de diseño que no se rompe:** el modelo decide si hay fraude; el LLM solo explica lo que el modelo ya decidió.

---

## Arquitectura de la etapa

```
Airflow (cada 15 min)
 ├─ 1. dbt build ──► MARTS.TRANSACTION_FEATURES (incremental: solo filas nuevas)
 ├─ 2. score_new_transactions ──► MARTS.TRANSACTION_SCORES  (XGBoost del modelo campeón)
 └─ 3. dbt test ──► MARTS.FRAUD_ALERTS (vista: scores con alerta + datos de la transacción)

FastAPI ──► /predict  : features + modelo campeón
        └─► /explain  : /predict + agente LangGraph
                         ├─ busca tipologías parecidas en pgvector
                         └─ pide a Claude una explicación corta
```

---

## Índice de pasos

| Paso | Qué hacemos |
|---|---|
| C0 | Comprobar requisitos y preparar el entorno |
| C1 | Exportar el modelo campeón a una carpeta |
| C2 | Tablas de scoring en Snowflake y cambios en dbt |
| C3 | Script de scoring por lotes (y backfill inicial) |
| C4 | Airflow en Docker |
| C5 | DAG `fraud_scoring_pipeline` |
| C6 | Probar fallos y reintentos |
| C7 | PostgreSQL con pgvector en Docker |
| C8 | Base de conocimiento de tipologías y embeddings |
| C9 | Agente con LangGraph y Claude |
| C10 | API: `/health` y `/predict` |
| C11 | API: `/explain` |
| C12 | Pruebas automáticas |
| C13 | Cierre de la etapa |

---

## Reglas que valen para toda la etapa

1. **El LLM no decide.** Solo recibe los datos y la probabilidad del modelo y redacta una explicación.
2. **El scoring es idempotente:** puntuar dos veces lo mismo no duplica filas (solo se puntúan transacciones sin score).
3. **Un solo DAG corre a la vez** (`max_active_runs = 1`) para no escribir en paralelo sobre las mismas tablas.
4. **La API reutiliza las funciones de features de la Etapa A** (`src/ml/build_features.py`) para garantizar que el modelo recibe las mismas features que en el entrenamiento.
5. **Importaciones perezosas del agente:** la API solo importa LangGraph, sentence-transformers y Claude dentro del endpoint `/explain`.
   Así `/predict` y las pruebas no necesitan esas librerías pesadas.
6. **Secretos en `.env`**, nunca en el código ni en Git.
7. **Costos:** el agente llama a Claude solo cuando se pide una explicación; ningún DAG la llama.
8. Mismas reglas de código de la Etapa A (README general, sección 5).

---

## C0 - Comprobar requisitos y preparar el entorno

**Pasos:**
1. Confirma que `git tag` muestra `etapa-b-completa` y que `reports/etapa_B_resultados.md` cumple los 3 criterios.
2. Abre MLflow (`mlflow ui --backend-store-uri sqlite:///mlflow.db`) y confirma en *Models* que `fraud_model` tiene el alias `champion`. Ciérralo con `Ctrl + C`.
3. Docker Desktop abierto y con al menos 8 GB de memoria asignada (Airflow y pgvector se suman a lo anterior).
4. Crea una cuenta en https://console.anthropic.com/ , genera una **API key** y, en la sección de facturación/límites, define un **límite de gasto mensual bajo**.
5. Edita `.env`: `ANTHROPIC_API_KEY=` (tu clave), `ANTHROPIC_MODEL=claude-sonnet-5-5` (verifica el nombre vigente en https://docs.claude.com/en/docs/about-claude/models),
   `POSTGRES_PASSWORD=` (una contraseña cualquiera), y confirma `POSTGRES_USER=fraud_user`, `POSTGRES_DB=fraud_kb`, `POSTGRES_HOST=localhost`, `POSTGRES_PORT=5433`.
6. Descomenta el bloque de la **Etapa C** en `requirements.txt` e instala: `pip install -r requirements.txt` (la primera vez tarda por `sentence-transformers`).
7. `pip freeze > requirements-lock.txt`
8. Crea las carpetas:
   - macOS/Linux: `mkdir -p src/scoring src/agent/knowledge src/api airflow/dags docker/airflow models/champion && touch src/scoring/__init__.py src/agent/__init__.py src/api/__init__.py`
   - Windows PowerShell: `New-Item -ItemType Directory -Force -Path src/scoring, src/agent/knowledge, src/api, airflow/dags, docker/airflow, models/champion` y `New-Item -ItemType File -Force src/scoring/__init__.py, src/agent/__init__.py, src/api/__init__.py`
9. Puertos libres: 8080 (Airflow), 5433 (pgvector), 8000 (API). Si usaste `dbt docs serve`, ciérralo.

**Verificación:** la clave de Anthropic está en `.env`; las librerías se instalaron (`python -c "import fastapi, langgraph, sentence_transformers, anthropic; print('ok')"`).

**Commit:** `git add requirements.txt requirements-lock.txt src && git commit -m "chore: add stage C dependencies and folders"`

---

## C1 - Exportar el modelo campeón

**Objetivo:** dejar el modelo campeón y su umbral en una carpeta simple (`models/champion/`), para que Airflow y la API lo lean sin depender de rutas internas de MLflow.

**Archivo:** `src/scoring/export_champion.py`

**Funciones:**

| Función | Qué hace |
|---|---|
| `find_champion_version()` | Con `MlflowClient`, consulta `fraud_model` con el alias `champion` y devuelve la versión y el `run_id` |
| `find_model_entry(run_id)` | Busca en `models/snowflake/model_info.json` (y si no está, en `models/model_info.json`) la entrada cuyo `run_id` coincide; así se sabe qué archivo y qué umbral corresponden |
| `copy_champion_files(entry, target_dir)` | Copia el archivo del modelo (`xgboost_model.json`) a `models/champion/` |
| `write_champion_info(...)` | Escribe `models/champion/champion_info.json` con: `model_name`, `model_version`, `run_id`, `threshold` y `exported_at` |
| `main()` | Ejecuta lo anterior; si el campeón no es XGBoost, se detiene con un mensaje claro |

**Ejecución:** `python -m src.scoring.export_champion`

**Verificación:** existen `models/champion/xgboost_model.json` y `models/champion/champion_info.json`, y el JSON muestra la versión y el umbral del campeón.

**Commit:** `git add src && git commit -m "feat: export champion model to a plain folder"`

---

## C2 - Tablas de scoring en Snowflake y cambios en dbt

**Objetivo:** crear dónde se guardan los scores y hacer que dbt trabaje de forma incremental.

### Script `infra/snowflake/03_scoring_tables.sql` (ejecútalo en Snowsight con rol `FRAUD_ROLE`)
```sql
USE ROLE FRAUD_ROLE;
USE WAREHOUSE FRAUD_WH;
USE SCHEMA FRAUD_DB.MARTS;

CREATE TABLE IF NOT EXISTS TRANSACTION_SCORES (
  ID_TRANSACCION     NUMBER(10,0),
  FRAUD_PROBABILITY  FLOAT,
  IS_ALERT           NUMBER(1,0),
  THRESHOLD          FLOAT,
  MODEL_VERSION      VARCHAR,
  SCORED_AT          TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
```

### Cambios en dbt (`dbt_project/`)

1. **Nueva fuente** en `models/marts/sources.yml` (o ampliando el existente): fuente `scoring`, esquema `MARTS`, tabla `transaction_scores`.
2. **`transaction_features.sql` pasa a incremental.** Añade al inicio `{{ config(materialized='incremental', unique_key='id_transaccion') }}` y, al final de la consulta:
   ```sql
   {% if is_incremental() %}
   where not exists (
       select 1 from {{ this }} existing
       where existing.id_transaccion = source_rows.id_transaccion
   )
   {% endif %}
   ```
   (Se da el nombre `source_rows` a la selección sobre `ref('stg_transactions')`; la comprobación `not exists` agrega solo ids nuevos.)
3. **Nuevo modelo `fraud_alerts.sql`** (vista en `models/marts/`): une `source('scoring', 'transaction_scores')` filtrado por `is_alert = 1` con `transaction_features` y `stg_transactions`, y expone: `id_transaccion`, `hora_simulada`, `tipo`, `monto`, saldos de origen y destino, `fraud_probability`, `threshold`, `model_version`, `scored_at`, `es_fraude` (la etiqueta real, solo para evaluar).
   Configuración: `{{ config(materialized='view') }}`.
4. **Tests nuevos** en `models/marts/schema.yml` para `fraud_alerts`: `id_transaccion` `unique` y `not_null`; `fraud_probability` `not_null`.

| Decisión | Valor | Por qué | Otras opciones |
|---|---|---|---|
| Incremental con `not exists` por id | Sí | Los mensajes de las 3 particiones de Kafka pueden llegar entrelazados; filtrar por "id mayor que el máximo" perdería filas rezagadas | Filtrar por id máximo (simple, pero puede perder filas), por fecha de carga con solape (más complejo) |
| `unique_key` | `id_transaccion` | `merge` evita duplicados si una fila se reprocesa | Sin clave (podría duplicar) |
| `fraud_alerts` como vista | Sí | Siempre está al día con los scores | Tabla (habría que reconstruirla cada vez) |

**Pasos:**
1. Ejecuta `03_scoring_tables.sql` en Snowsight.
2. Haz los cambios de dbt.
3. En `dbt_project/`: `dotenv -f ../.env run -- dbt build --exclude tag:full_load`
4. Comprueba en Snowsight que la tabla `TRANSACTION_SCORES` existe (vacía) y que la vista `FRAUD_ALERTS` se creó.

**Verificación:** `dbt build` termina sin errores; `SELECT COUNT(*) FROM FRAUD_DB.MARTS.TRANSACTION_FEATURES` sigue en 6,362,620 (el cambio a incremental no pierde filas); `FRAUD_ALERTS` existe y devuelve 0 filas (todavía no hay scores).

**Si falla:** `dbt build` dice que cambió la materialización → ejecuta una vez `dbt build --full-refresh --select transaction_features`.

**Commit:** `git add infra dbt_project && git commit -m "feat: add scoring tables, incremental features and alerts view"`

---

## C3 - Script de scoring por lotes

**Objetivo:** puntuar las transacciones que todavía no tienen score y guardarlas en Snowflake.

**Archivo:** `src/scoring/score_new_transactions.py`

**Constantes:** `DEFAULT_LIMIT = 50000` (máximo de filas por ejecución), `CHAMPION_DIR = models/champion`, `WRITE_TABLE = "TRANSACTION_SCORES"`.

**Funciones:**

| Función | Qué hace |
|---|---|
| `load_champion()` | Carga `xgboost_model.json` en un `XGBClassifier` y lee `champion_info.json` (umbral y versión) |
| `read_unscored_features(connection, limit)` | `SELECT ... FROM MARTS.TRANSACTION_FEATURES f LEFT JOIN MARTS.TRANSACTION_SCORES s ON f.ID_TRANSACCION = s.ID_TRANSACCION WHERE s.ID_TRANSACCION IS NULL ORDER BY f.ID_TRANSACCION LIMIT <limit>`; devuelve un DataFrame con nombres en minúsculas |
| `score_features(model, features_df)` | `model.predict_proba(features_df[config.FEATURE_COLUMNS])[:, 1]` (el orden de columnas es el de `FEATURE_COLUMNS`) |
| `build_scores_table(ids, probabilities, threshold, model_version)` | DataFrame con `ID_TRANSACCION`, `FRAUD_PROBABILITY`, `IS_ALERT` (1 si la probabilidad ≥ umbral), `THRESHOLD`, `MODEL_VERSION` |
| `write_scores(connection, scores_df)` | `write_pandas(connection, scores_df, "TRANSACTION_SCORES", database=..., schema="MARTS")` |
| `main()` | Argumentos `--limit` y `--until-empty`; con `--until-empty` repite lectura–scoring–escritura hasta que no queden filas sin score; imprime filas puntuadas y alertas por lote |

| Parámetro | Valor | Por qué | Otras opciones |
|---|---|---|---|
| Lote máximo | `50000` | Equilibra memoria, tiempo y tamaño de cada escritura | `10000` (más viajes a Snowflake), `500000` (mucha memoria) |
| Detectar "ya puntuado" con `LEFT JOIN ... IS NULL` | Sí | Idempotente: reintentar no duplica | Guardar el último id procesado (falla si hay filas rezagadas) |
| `write_pandas` | Sube un archivo temporal y copia | Mucho más rápido que insertar fila por fila | `INSERT` fila por fila (lentísimo) |
| Umbral de `champion_info.json` | El elegido en la validación | Es el mismo con el que se evaluó el modelo | Recalcular (podría diferir) |

**Pasos:**
1. Crea el script.
2. Prueba con un lote pequeño: `python -m src.scoring.score_new_transactions --limit 1000`
3. Comprueba en Snowsight: `SELECT COUNT(*), SUM(IS_ALERT) FROM FRAUD_DB.MARTS.TRANSACTION_SCORES;` → 1000 filas y un número de alertas.
4. **Backfill inicial** (puntúa las 6.3 millones de filas, una sola vez): `python -m src.scoring.score_new_transactions --until-empty` (tarda varios minutos).
5. Comprueba: `SELECT COUNT(*), COUNT(DISTINCT ID_TRANSACCION) FROM FRAUD_DB.MARTS.TRANSACTION_SCORES;` → ambos **6,362,620**.
6. En la carpeta `dbt_project/`: `dotenv -f ../.env run -- dbt test --select fraud_alerts` debe pasar.

**Verificación:** conteo y conteo distinto iguales a 6,362,620; `SELECT COUNT(*) FROM FRAUD_DB.MARTS.FRAUD_ALERTS;` devuelve el número de alertas.

**Si falla:** `write_pandas` dice que no existe la tabla → ejecuta `03_scoring_tables.sql`; nombres de columna en minúsculas al escribir → deben ser mayúsculas (`ID_TRANSACCION`...).

**Commit:** `git add src && git commit -m "feat: add batch scoring script"`

---

## C4 - Airflow en Docker

**Objetivo:** tener Airflow corriendo en un contenedor, con dbt y el scoring disponibles dentro.

**Archivos:** `docker/airflow/Dockerfile` y un servicio nuevo en `docker-compose.yml`.

### `docker/airflow/Dockerfile`
```dockerfile
FROM apache/airflow:3.0.2

RUN python -m venv /home/airflow/dbt_venv \
    && /home/airflow/dbt_venv/bin/pip install --no-cache-dir dbt-core dbt-snowflake

RUN python -m venv /home/airflow/ml_venv \
    && /home/airflow/ml_venv/bin/pip install --no-cache-dir \
       pandas pyarrow numpy xgboost scikit-learn python-dotenv \
       "snowflake-connector-python[pandas]"
```

| Elemento | Por qué | Otras opciones |
|---|---|---|
| `apache/airflow:3.0.2` | Ejemplo de versión 3.x; **usa la última estable** (tags en https://hub.docker.com/r/apache/airflow/tags) | Airflow 2.x (los imports y parámetros cambian) |
| Dos entornos virtuales aislados (`dbt_venv`, `ml_venv`) | Instalar dbt y xgboost junto a Airflow suele causar conflictos de dependencias | Instalarlo todo en el entorno de Airflow (frágil), `DockerOperator` (más complejo) |
| Sin `torch` | El scoring solo usa XGBoost | - |

### Servicio `airflow` (añadir a `docker-compose.yml`)
```yaml
  airflow:
    build:
      context: .
      dockerfile: docker/airflow/Dockerfile
    container_name: fraud-airflow
    command: standalone
    ports:
      - "8080:8080"
    env_file:
      - .env
    environment:
      AIRFLOW__CORE__LOAD_EXAMPLES: "false"
      SNOWFLAKE_PRIVATE_KEY_PATH: /secrets/rsa_key.p8
      PYTHONPATH: /opt/project
      DBT_TARGET_PATH: /tmp/dbt_target
      DBT_LOG_PATH: /tmp/dbt_logs
    volumes:
      - ./airflow/dags:/opt/airflow/dags
      - ./src:/opt/project/src
      - ./dbt_project:/opt/project/dbt_project
      - ./models:/opt/project/models
      - ${SNOWFLAKE_PRIVATE_KEY_PATH}:/secrets/rsa_key.p8:ro
      - airflow_home:/opt/airflow
```
Y en `volumes:` del final, agrega `airflow_home:`.

| Elemento | Por qué | Otras opciones |
|---|---|---|
| `command: standalone` | Interfaz, planificador y procesador de DAGs en un solo contenedor: liviano | Docker Compose oficial con Postgres y LocalExecutor/Celery (más piezas, para producción) |
| `AIRFLOW__CORE__LOAD_EXAMPLES: "false"` | Oculta los DAGs de ejemplo | `true` |
| `DBT_TARGET_PATH`, `DBT_LOG_PATH` en `/tmp` | dbt no escribe en la carpeta montada (evita problemas de permisos) | Escribir en `dbt_project/` |
| Volúmenes de `src`, `dbt_project` y `models` | El contenedor ve tu código y el modelo campeón sin reconstruir la imagen | Copiarlos dentro de la imagen (hay que reconstruir con cada cambio) |
| `airflow_home` | Guarda la base de datos de Airflow entre reinicios | Sin volumen (se pierde el historial) |

**Pasos:**
1. Crea el Dockerfile y agrega el servicio.
2. Construye: `docker compose build airflow` (varios minutos).
3. Levanta: `docker compose up -d airflow`
4. Obtén la contraseña del administrador (Airflow 3 la genera al arrancar): `docker compose logs airflow | grep -i password` o
   `docker exec fraud-airflow cat /opt/airflow/simple_auth_manager_passwords.json.generated`
5. Abre http://localhost:8080 e inicia sesión (usuario `admin`; la contraseña es la del punto 4).

**Verificación:** la interfaz carga y muestra la lista de DAGs (vacía por ahora, sin ejemplos).

**Si falla:**
- La interfaz no carga → espera un minuto (el primer arranque inicializa la base) y mira `docker compose logs airflow --tail 50`.
- La versión de la imagen no existe → elige una de la lista de tags.
- No encuentras la contraseña → en Airflow 3 con otro gestor de autenticación el mecanismo cambia; sigue la documentación de la versión instalada.

**Commit:** `git add docker docker-compose.yml && git commit -m "feat: add Airflow service with isolated dbt and ML environments"`

---

## C5 - DAG `fraud_scoring_pipeline`

**Archivo:** `airflow/dags/fraud_scoring_pipeline.py`

**Tareas, en orden:**

| Tarea | Tipo | Comando (dentro del contenedor) | Qué hace |
|---|---|---|---|
| `dbt_build` | `BashOperator` | `cd /opt/project/dbt_project && /home/airflow/dbt_venv/bin/dbt build --exclude tag:full_load` | Actualiza staging y features (solo lo nuevo) |
| `score_new_transactions` | `BashOperator` | `cd /opt/project && /home/airflow/ml_venv/bin/python -m src.scoring.score_new_transactions --limit 50000` | Puntúa lo que no tiene score |
| `test_alerts` | `BashOperator` | `cd /opt/project/dbt_project && /home/airflow/dbt_venv/bin/dbt test --select fraud_alerts` | Valida la vista de alertas |

Orden: `dbt_build >> score_new_transactions >> test_alerts`.

**Imports en Airflow 3** (si instalaste Airflow 2, sigue la documentación de tu versión):
`from airflow.sdk import DAG` y `from airflow.providers.standard.operators.bash import BashOperator`.

**Parámetros del DAG:**

| Parámetro | Valor | Por qué | Otras opciones |
|---|---|---|---|
| `dag_id` | `"fraud_scoring_pipeline"` | Nombre único | - |
| `schedule` | `"*/15 * * * *"` | Cada 15 minutos | `"@hourly"`, `"@daily"`, `None` (solo manual) |
| `start_date` | Una fecha fija en el pasado (por ejemplo `datetime(2026, 1, 1)`) | Fecha fija evita comportamientos confusos | `datetime.now()` (cambia en cada lectura) |
| `catchup` | `False` | No ejecuta los intervalos pasados pendientes | `True` (ejecutaría cientos de corridas atrasadas) |
| `max_active_runs` | `1` | Evita que dos corridas escriban a la vez | Más de 1 (riesgo de pisarse) |
| `default_args["retries"]` | `2` | Reintenta ante fallos transitorios (red, warehouse despertando) | `0` (sin reintentos) |
| `default_args["retry_delay"]` | 2 minutos | Da tiempo a que el problema pase | Otros valores |
| `tags` | `["fraud", "scoring"]` | Para filtrar en la interfaz | - |

**Pasos:**
1. Crea el DAG.
2. Espera 1 minuto: aparece `fraud_scoring_pipeline` en http://localhost:8080 (si hay un error de importación, la interfaz lo muestra arriba).
3. Actívalo y lánzalo manualmente (botón *Trigger*).
4. Abre la corrida y revisa el log de cada tarea.
5. Para ver una corrida con filas nuevas: sin nada nuevo en RAW, el scoring dirá `0 filas`, y eso es correcto.
   (Opcional: genera filas nuevas con el producer y el job de Spark en modo `--mode load` y vuelve a lanzar el DAG.)

**Verificación:** las 3 tareas terminan en verde (`success`); el log de `score_new_transactions` imprime cuántas filas se puntuaron; dbt imprime `PASS`.

**Si falla:**
| Síntoma | Causa probable | Solución |
|---|---|---|
| `Broken DAG` / error de importación | Imports de Airflow 2 en una imagen 3 (o al revés) | Usa los imports de tu versión |
| `dbt: command not found` | Ruta incorrecta | Usa la ruta completa `/home/airflow/dbt_venv/bin/dbt` |
| `Env var required but not provided` | `.env` no llegó al contenedor | Revisa `env_file: - .env` y reinicia el servicio |
| `JWT token is invalid` | Llave mal montada | Revisa `SNOWFLAKE_PRIVATE_KEY_PATH` en `.env` |
| `ModuleNotFoundError: src` | Falta `PYTHONPATH` o el `cd` | Revisa `PYTHONPATH: /opt/project` y `cd /opt/project` |

**Commit:** `git add airflow && git commit -m "feat: add scoring pipeline DAG"`

---

## C6 - Probar fallos y reintentos

**Objetivo:** ver con tus propios ojos cómo se comporta Airflow cuando algo falla.

**Pasos:**
1. Edita temporalmente el comando de `score_new_transactions` para que falle: añade `&& exit 1` al final del comando. Guarda.
2. Lanza el DAG manualmente.
3. Observa: la tarea falla, pasa a `up_for_retry`, espera 2 minutos y reintenta (hasta 2 veces); al agotar los reintentos queda en `failed` y `test_alerts` queda en `upstream_failed`.
4. Revierte el cambio, guárdalo y, en la corrida fallida, usa *Clear* sobre la tarea para reintentarla: debe terminar en `success`.

**Verificación:** viste los estados `up_for_retry`, `failed`, `upstream_failed` y finalmente `success`. Anota la secuencia en una línea (te servirá para explicarla en una entrevista).

**Commit:** ninguno (el cambio se revirtió). Comprueba con `git status` que el DAG quedó como estaba.

---

## C7 - PostgreSQL con pgvector en Docker

**Servicio `pgvector` (añadir a `docker-compose.yml`):**
```yaml
  pgvector:
    image: pgvector/pgvector:pg16
    container_name: fraud-pgvector
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "5433:5432"
    volumes:
      - pgvector_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
```
Y en `volumes:` del final, agrega `pgvector_data:`.

| Elemento | Por qué | Otras opciones |
|---|---|---|
| `pgvector/pgvector:pg16` | Imagen oficial con PostgreSQL 16 y la extensión ya instalada | Instalar la extensión a mano en `postgres:16` |
| Puerto `5433:5432` | Evita choques con un PostgreSQL local en el 5432 | `5432:5432` |
| `healthcheck` con `pg_isready` | Docker sabe cuándo la base está lista | Sin chequeo |

**Pasos:**
1. Agrega el servicio y levántalo: `docker compose up -d pgvector`
2. Espera a que el estado sea `healthy` (`docker compose ps`).
3. Activa la extensión:
   ```
   docker exec fraud-pgvector psql -U fraud_user -d fraud_kb -c "CREATE EXTENSION IF NOT EXISTS vector;"
   docker exec fraud-pgvector psql -U fraud_user -d fraud_kb -c "\dx"
   ```
   (si cambiaste `POSTGRES_USER` o `POSTGRES_DB` en `.env`, usa esos valores).

**Verificación:** `\dx` lista la extensión `vector`.

**Commit:** `git add docker-compose.yml && git commit -m "feat: add pgvector service"`

---

## C8 - Base de conocimiento de tipologías y embeddings

**Objetivo:** guardar descripciones de patrones de fraude como vectores, para que el agente recupere las más parecidas a una alerta.

### Archivo `src/agent/knowledge/fraud_typologies.json`

Lista de 8 objetos con `title` y `description`:

| # | `title` | `description` |
|---|---|---|
| 1 | Vaciado de cuenta | La transacción deja la cuenta de origen con saldo cero tras mover todo el dinero. Es típico de una cuenta tomada por un tercero que retira todos los fondos de una vez. |
| 2 | Transferencia seguida de retiro | Una transferencia hacia una cuenta intermedia que luego retira el dinero en efectivo. Sirve para convertir los fondos en efectivo antes de que se detecte el fraude. |
| 3 | Destino sin saldo previo | El dinero llega a una cuenta de destino que no tenía saldo antes. Las cuentas nuevas o vacías se usan para recibir fondos y moverlos rápido. |
| 4 | Saldos que no cuadran | El saldo final de la cuenta no coincide con el saldo inicial más o menos el monto de la operación. Una diferencia indica manipulación o registros inconsistentes. |
| 5 | Monto inusualmente alto | Un monto muy superior al de una operación común del mismo tipo. Los fraudes suelen mover montos grandes para maximizar el beneficio antes de ser bloqueados. |
| 6 | Operación de tipo riesgoso | En el conjunto de datos del proyecto, el fraude solo aparece en transferencias y retiros en efectivo; los pagos a comercios y los depósitos no presentan fraude. |
| 7 | Actividad nocturna | Operaciones de montos altos en la madrugada pueden merecer revisión, aunque la hora por sí sola no confirma un fraude. |
| 8 | Pago normal a comercio | Los pagos a comercios con saldos coherentes y montos moderados suelen ser normales y no requieren alerta. |

### `src/agent/retrieval.py` y `src/agent/load_knowledge_base.py`

**Constantes (en `src/agent/agent_config.py`):**

| Constante | Valor | Por qué | Otras opciones |
|---|---|---|---|
| `EMBEDDING_MODEL_NAME` | `"paraphrase-multilingual-MiniLM-L12-v2"` | Entiende español; vectores de 384 dimensiones | `all-MiniLM-L6-v2` (solo inglés), modelos de pago |
| `EMBEDDING_DIMENSIONS` | `384` | Debe coincidir con el modelo y con la columna `VECTOR(384)` | - |
| `TOP_K` | `3` | Tres casos parecidos dan contexto sin ruido | `1` (poco contexto), `10` (ruido) |
| `DISTANCE_OPERATOR` | `<=>` (distancia coseno) | Adecuada para texto con vectores normalizados | `<->` (euclidiana), `<#>` (producto interno) |

**`load_knowledge_base.py`:**

| Paso | Qué hace |
|---|---|
| 1 | Conecta a PostgreSQL (`psycopg2.connect` con `POSTGRES_*`) y llama `register_vector(connection)` |
| 2 | Crea la tabla `fraud_typologies (id SERIAL PRIMARY KEY, title TEXT, description TEXT, embedding VECTOR(384))` si no existe, y la vacía (`TRUNCATE`) para que el script se pueda repetir |
| 3 | Lee el JSON; arma el texto de cada entrada como `title + ". " + description` |
| 4 | Calcula los embeddings con `SentenceTransformer(...).encode(texts, normalize_embeddings=True)` |
| 5 | Inserta una fila por entrada y confirma con `commit` |
| 6 | Imprime cuántas filas insertó |

**`retrieval.py`:** función `retrieve_typologies(query_text, top_k)` que calcula el embedding de la consulta y ejecuta
`SELECT title, description, embedding <=> %s AS distance FROM fraud_typologies ORDER BY distance LIMIT %s`.
No se crea índice: con 8 filas la búsqueda exacta es instantánea (con miles de filas se usaría HNSW con `m = 16` y `ef_construction = 64`).

**Pasos:**
1. Crea los archivos.
2. Ejecuta: `python -m src.agent.load_knowledge_base` (la primera vez descarga el modelo de embeddings, unos cientos de MB).
3. Comprueba la recuperación:
   ```
   python -c "from src.agent.retrieval import retrieve_typologies; print(retrieve_typologies('la cuenta de origen quedó en cero después de la transferencia', 3))"
   ```

**Verificación:** el script imprime `8` filas insertadas; la búsqueda de ejemplo devuelve primero **"Vaciado de cuenta"**.

**Commit:** `git add src && git commit -m "feat: add fraud typologies knowledge base with pgvector"`

---

## C9 - Agente con LangGraph y Claude

**Archivo:** `src/agent/fraud_agent.py`

**Estado del grafo (`AgentState`, un `TypedDict`):** `transaction` (diccionario con los datos), `fraud_probability` (número), `similar_typologies` (lista), `explanation` (texto).

**Funciones y nodos:**

| Elemento | Qué hace |
|---|---|
| `describe_transaction(transaction)` | Convierte la transacción en una frase en español: tipo, monto, saldos del origen y del destino, hora del día |
| Nodo `retrieve_similar_typologies(state)` | Usa `describe_transaction` como consulta, llama a `retrieve_typologies(query, TOP_K)` y guarda el resultado en el estado |
| Nodo `generate_explanation(state)` | Arma el mensaje con los datos, la probabilidad y las tipologías, llama a Claude y guarda el texto en el estado |
| `build_agent_graph()` | `StateGraph(AgentState)`; `add_node` para los dos nodos; `add_edge(START, "retrieve_similar_typologies")`, `add_edge("retrieve_similar_typologies", "generate_explanation")`, `add_edge("generate_explanation", END)`; `compile()` |
| `explain_transaction(transaction, fraud_probability)` | Construye (o reutiliza) el grafo y lo ejecuta con `invoke`; devuelve la explicación y los títulos de las tipologías |

**Parámetros de Claude (`ChatAnthropic`):**

| Parámetro | Valor | Por qué | Otras opciones |
|---|---|---|---|
| `model` | `os.getenv("ANTHROPIC_MODEL")` | El nombre se configura en `.env` y se cambia sin tocar código | Fijo en el código (hay que editar al cambiar de modelo) |
| `temperature` | `0` | Explicaciones consistentes | `0.7` (más variadas, menos repetibles) |
| `max_tokens` | `300` | Explicaciones cortas y baratas | `100` (se corta), `1000` (innecesario) |
| `timeout` | `30` segundos | La API no se queda colgada esperando | Sin límite |

**Mensaje del sistema (`SYSTEM_PROMPT`):**
```
Eres un asistente que explica alertas de posible fraude a un analista.
Reglas:
1. Usa solamente los datos que se te entregan. No inventes información.
2. Explica en español claro, en un máximo de 4 oraciones.
3. Menciona las señales concretas que ves en los datos (montos, saldos, tipo de operación).
4. No afirmes que hay fraude con certeza: habla de "alerta" y de "señales de riesgo".
5. No acuses a ninguna persona.
6. Si las tipologías recuperadas no encajan con los datos, no las uses.
```
El mensaje del usuario incluye: descripción de la transacción, probabilidad del modelo (con el umbral) y las tipologías recuperadas.

**Pasos:**
1. Crea el agente.
2. Prueba con una transacción de ejemplo (un `if __name__ == "__main__"` con una transferencia de 181 donde el origen pasa de 181 a 0 y el destino estaba en 0, probabilidad 0.97):
   `python -m src.agent.fraud_agent`

**Verificación:** imprime una explicación en español de hasta 4 oraciones que menciona señales del dato (por ejemplo, saldo del origen en cero), usa lenguaje de "alerta/señales de riesgo" y no afirma fraude con certeza.

**Si falla:** `AuthenticationError` → revisa `ANTHROPIC_API_KEY`; `model not found` → revisa `ANTHROPIC_MODEL` con la lista vigente de modelos; el grafo falla en `retrieve` → ¿está `fraud-pgvector` corriendo y cargada la base (C8)?

**Commit:** `git add src && git commit -m "feat: add LangGraph agent that explains alerts with Claude"`

---

## C10 - API: `/health` y `/predict`

**Archivos:** `src/api/schemas.py`, `src/api/model_loader.py`, `src/api/main.py`.

### `schemas.py` (modelos de Pydantic)

| Clase | Campos |
|---|---|
| `TransactionRequest` | `hora_simulada` (entero ≥ 1), `tipo` (uno de `deposito`, `retiro`, `debito`, `pago`, `transferencia`; con `Literal`), `monto` (≥ 0), `saldo_inicial_origen`, `saldo_final_origen`, `saldo_inicial_destino`, `saldo_final_destino` (todos ≥ 0), `id_transaccion` (opcional) |
| `PredictionResponse` | `fraud_probability` (0 a 1), `is_alert` (booleano), `threshold`, `model_version` |
| `ExplanationResponse` | Todo lo de `PredictionResponse` más `explanation` (texto) y `similar_typologies` (lista de títulos) |

### `model_loader.py`
- `load_champion_model()`: lee `models/champion/xgboost_model.json` y `champion_info.json`; devuelve el modelo, el umbral y la versión.

### `main.py`

| Elemento | Qué hace |
|---|---|
| `lifespan` | Al arrancar, carga el modelo una sola vez y lo guarda en `app.state.model`, `app.state.threshold`, `app.state.model_version`; si no encuentra el archivo, deja `app.state.model = None` |
| `build_features_from_request(request)` | DataFrame de una fila con las columnas en español; llama a `add_time_features`, `add_balance_features` y `add_type_features` de `src/ml/build_features.py` y devuelve `[config.FEATURE_COLUMNS]` |
| `GET /health` | Responde `{"status": "ok", "model_version": ...}` con 200 si el modelo está cargado; `503` si no |
| `POST /predict` | Valida con `TransactionRequest`; calcula features; `predict_proba`; devuelve `PredictionResponse` |

| Decisión | Valor | Por qué | Otras opciones |
|---|---|---|---|
| Cargar el modelo al arrancar | Sí | Cargar en cada petición sería lentísimo | Cargar en cada llamada |
| `def` normal (no `async def`) | Sí | El modelo es de CPU; es más simple | `async def` (útil con muchas llamadas de red concurrentes) |
| Respuesta `503` si no hay modelo | Sí | El orquestador (Kubernetes) sabe que no está listo | Responder `200` con un error en el texto |
| Reutilizar `build_features.py` | Sí | Garantiza las mismas features que en el entrenamiento | Reescribir la lógica en la API (riesgo de desalinearse) |
| Entrada con campos crudos, no con features | Sí | El cliente no debe conocer las features internas | Pedir las 15 features (acopla al cliente con el modelo) |

**Pasos:**
1. Crea los tres archivos.
2. Arranca: `uvicorn src.api.main:app --reload --port 8000`
3. Abre http://localhost:8000/docs (documentación interactiva).
4. Prueba `GET /health` con *Try it out*.
5. Prueba `POST /predict` con dos cuerpos de ejemplo:
   - Parecida a un fraude: `{"hora_simulada": 1, "tipo": "transferencia", "monto": 181.0, "saldo_inicial_origen": 181.0, "saldo_final_origen": 0.0, "saldo_inicial_destino": 0.0, "saldo_final_destino": 0.0}`
   - Normal: `{"hora_simulada": 10, "tipo": "pago", "monto": 50.0, "saldo_inicial_origen": 200.0, "saldo_final_origen": 150.0, "saldo_inicial_destino": 0.0, "saldo_final_destino": 0.0}`
6. Prueba un cuerpo inválido (`"tipo": "regalo"`): debe devolver `422`.

**Verificación:** `/health` devuelve `ok`; la primera transacción tiene probabilidad **alta** y la segunda **baja** (comportamiento esperable, no una cifra exacta); el cuerpo inválido da `422`.

**Commit:** `git add src && git commit -m "feat: add FastAPI service with health and predict endpoints"`

---

## C11 - API: `/explain`

**Cambio en `main.py`:** agregar `POST /explain`.

**Flujo:**
1. Valida el cuerpo con `TransactionRequest` (el mismo de `/predict`).
2. Calcula la probabilidad igual que `/predict`.
3. **Importa el agente dentro de la función** (importación perezosa): `from src.agent.fraud_agent import explain_transaction`.
4. Llama a `explain_transaction(transaction, fraud_probability)`.
5. Devuelve `ExplanationResponse`.

**Manejo de errores:**

| Situación | Respuesta |
|---|---|
| Falta `ANTHROPIC_API_KEY` o no hay conexión con pgvector | `503` con un mensaje claro (`"El agente no está disponible"`) |
| Claude tarda más de 30 segundos | `504` con un mensaje claro |
| Error inesperado del agente | `500` con un mensaje genérico (el detalle va al registro, no a la respuesta) |

**Decisión:** `/explain` explica la transacción aunque la probabilidad esté por debajo del umbral (el analista puede querer entender por qué **no** hubo alerta). Alternativa: rechazar con `400` si no hay alerta.

**Pasos:**
1. Haz el cambio.
2. Con el servidor corriendo y pgvector arriba, prueba `/explain` en http://localhost:8000/docs con el cuerpo "parecida a un fraude" de C10.
3. Detén pgvector (`docker compose stop pgvector`) y repite: debe devolver `503` con el mensaje claro. Vuelve a levantarlo.

**Verificación:** `/explain` devuelve probabilidad, explicación en español y la lista de tipologías; con pgvector detenido responde `503` sin romper la API.

**Commit:** `git add src && git commit -m "feat: add explain endpoint backed by the LangGraph agent"`

---

## C12 - Pruebas automáticas

**Archivo:** `tests/test_api.py`

**Cómo se prueba sin cargar el modelo real:** se usa `TestClient(app)` **sin** el contexto `with` (así no se ejecuta `lifespan`) y se asignan a mano
`app.state.model`, `app.state.threshold` y `app.state.model_version`. El modelo falso es una clase pequeña con un método `predict_proba` que devuelve probabilidades fijas.

| Prueba | Qué comprueba |
|---|---|
| `test_health_ok` | `GET /health` devuelve 200 con el modelo falso cargado |
| `test_health_without_model` | Con `app.state.model = None`, devuelve 503 |
| `test_predict_returns_expected_fields` | `POST /predict` con un cuerpo válido devuelve las 4 claves y una probabilidad entre 0 y 1 |
| `test_predict_alert_flag` | Con probabilidad falsa 0.9 y umbral 0.5, `is_alert` es verdadero; con 0.1, falso |
| `test_predict_rejects_invalid_type` | `"tipo": "regalo"` devuelve 422 |
| `test_predict_rejects_negative_amount` | `"monto": -5` devuelve 422 |
| `test_explain_returns_503_when_agent_fails` | Con el agente simulado para fallar (`monkeypatch`), `/explain` devuelve 503 |

**Ejecución:** `python -m pytest tests -v` y `ruff check src tests`.

**Verificación:** todas las pruebas de las Etapas A y C pasan.

**Commit:** `git add tests && git commit -m "test: add API tests with a fake model"`

---

## C13 - Cierre de la etapa

**Checklist final:**
- [ ] `TRANSACTION_SCORES` con 6,362,620 filas y `FRAUD_ALERTS` con las alertas.
- [ ] El DAG corre en verde y viste el ciclo de fallo y reintento.
- [ ] pgvector con 8 tipologías; la búsqueda devuelve "Vaciado de cuenta" para el ejemplo.
- [ ] `/health`, `/predict` y `/explain` responden; `/explain` degrada con elegancia (503) si falta pgvector o Claude.
- [ ] `python -m pytest tests -v` y `ruff check src tests` sin errores.
- [ ] `.env`, llaves y `models/` no están en Git.
- [ ] Apagaste lo que no usas: `docker compose down`.

**Pasos finales:**
```
git add .
git commit -m "docs: stage C results"
git tag etapa-c-completa
git push
git push --tags
```

**Qué se lleva a la Etapa D:**
1. La API con sus endpoints (se le añaden métricas).
2. La tabla `MARTS.FRAUD_ALERTS` y las tablas de scores (las lee el dashboard).
3. El DAG (se le añade un reporte diario de drift).
4. El modelo en `models/champion/` (se le añade una muestra de referencia).

---

## Problemas frecuentes de la etapa

| Síntoma | Causa probable | Solución |
|---|---|---|
| `docker compose up` no levanta Airflow | Falta memoria en Docker Desktop | Sube la memoria a 8 GB o apaga otros servicios (`docker compose stop spark-job kafka`) |
| El DAG no aparece | Error de sintaxis o de imports | Mira el mensaje en la parte superior de la interfaz de Airflow |
| `psycopg2.OperationalError` | pgvector apagado o puerto equivocado | `docker compose ps` y revisa `POSTGRES_PORT=5433` |
| La API arranca pero `/predict` da 503 | Falta `models/champion/` | Repite C1 |
| `/explain` muy lento | Primera carga del modelo de embeddings | Normal la primera vez; las siguientes son rápidas |
| `AuthenticationError` de Anthropic | Clave incorrecta | Revisa `.env` y reinicia la API |
| Cambios en el DAG no se ven | Airflow lee los DAGs cada cierto tiempo | Espera 1 minuto o reinicia el servicio |

---

## Mapa final de archivos de la Etapa C

```
docker-compose.yml                         # servicios airflow y pgvector
docker/airflow/Dockerfile                  # C4
airflow/dags/fraud_scoring_pipeline.py     # C5
infra/snowflake/03_scoring_tables.sql      # C2
dbt_project/models/marts/fraud_alerts.sql  # C2 (más cambios en transaction_features.sql y schema.yml)
src/scoring/export_champion.py             # C1
src/scoring/score_new_transactions.py      # C3
src/agent/agent_config.py                  # C8
src/agent/knowledge/fraud_typologies.json  # C8
src/agent/load_knowledge_base.py           # C8
src/agent/retrieval.py                     # C8
src/agent/fraud_agent.py                   # C9
src/api/schemas.py                         # C10
src/api/model_loader.py                    # C10
src/api/main.py                            # C10 y C11
tests/test_api.py                          # C12
```

---

**Siguiente etapa:** [Etapa D - Monitoreo y dashboard](etapa_D_monitoreo_y_dashboard.md).
