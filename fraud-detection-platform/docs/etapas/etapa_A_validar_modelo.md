# Etapa A - Validar el modelo (local, sin infraestructura)

**Objetivo:** responder con evidencia una sola pregunta: *¿este modelo detecta fraude lo bastante
bien como para justificar construir todo el ecosistema alrededor?*

**Qué NO se usa en esta etapa:** Kafka, Spark, Snowflake, dbt, Airflow, Docker, AWS ni Claude.
Todo corre en tu computadora con archivos Parquet.

**Resultado final de la etapa:** un archivo `reports/decision_etapa_A.md` con el criterio de éxito
(escrito **antes** de entrenar), los resultados y una decisión: seguir, seguir con cambios, o no seguir.

**Cómo usar esta guía:** hazla en orden. Cada paso tiene: objetivo, pasos exactos, parámetros con su
justificación, una **verificación** (si no coincide, no sigas) y un commit. Los comandos aparecen para
Windows (PowerShell) y para macOS/Linux. Si usas Windows, abre PowerShell; si usas macOS o Linux, la terminal.

---

## Índice de pasos

| Paso | Qué hacemos |
|---|---|
| A0 | Comprobar requisitos |
| A1 | Preparar carpeta, Git y entorno virtual |
| A2 | Instalar dependencias y fijar versiones |
| A3 | Descargar PaySim |
| A4 | Crear `config.py` |
| A5 | Cargar, validar y renombrar a español |
| A6 | Explorar los datos (notebook) |
| A7 | Crear features |
| A8 | Partir los datos por tiempo |
| A9 | Crear funciones de métricas |
| A10 | Fijar el criterio de "modelo útil" (antes de entrenar) |
| A11 | Crear el escalado de datos |
| A12 | Líneas base: regla y regresión logística |
| A13 | Entrenar XGBoost |
| A14 | Entrenar el autoencoder |
| A15 | Evaluación final en prueba y comparación |
| A16 | Decisión y registro del modelo campeón |
| A17 | Cierre de la etapa |

---

## Reglas que valen para toda la etapa

1. **Siempre se ejecuta desde la carpeta raíz del proyecto** (`fraud-detection-platform/`) y con el entorno
   virtual activado. Si el prompt no muestra `(.venv)`, actívalo.
2. **Los scripts se ejecutan como módulos:** `python -m src.ml.nombre_del_script` (sin `.py`).
   Así Python encuentra las importaciones `from src.ml import config`.
3. **El conjunto de prueba (`test`) se usa una sola vez**, en el paso A15. Todo lo que se ajusta
   (parámetros, umbrales, parada temprana) se ajusta con validación (`valid`). Si tocas el modelo después de ver la prueba, el resultado deja de ser honesto.
4. **Semilla fija:** `RANDOM_STATE = 42` en todo lo aleatorio.
5. **Al terminar cada script nuevo:** `ruff check src tests` debe salir sin errores.
6. **Reglas de código del README general:** nombres de variables y funciones en inglés, columnas de datos en español, una acción por línea, cada parámetro elegido con su justificación y alternativas.
7. **Una cosa a la vez:** no pases al paso siguiente si la verificación del actual falla.

---

## A0 - Comprobar requisitos

**Objetivo:** asegurarte de que tu computadora puede hacer la etapa antes de empezar.

**Pasos:**
1. Abre una terminal.
2. Comprueba Python 3.11:
   - Windows: `py -3.11 --version` → debe imprimir `Python 3.11.x`.
   - macOS/Linux: `python3.11 --version` → debe imprimir `Python 3.11.x`.
   - Si no está instalado: descárgalo desde https://www.python.org/downloads/ (Windows/macOS) o instálalo con tu gestor de paquetes (Linux). En Windows, marca "Add python.exe to PATH" al instalar.
3. Comprueba Git: `git --version` → debe imprimir una versión. Si no está: https://git-scm.com/downloads.
4. Memoria RAM: abre el Administrador de tareas (Windows), Monitor de actividad (macOS) o `free -h` (Linux).
   Necesitas **8 GB como mínimo**; con 16 GB irá cómodo. Cierra navegadores pesados antes de cargar los datos.
5. Disco: asegúrate de tener **al menos 5 GB libres**.
6. Cuenta de Kaggle: créala gratis en https://www.kaggle.com.
7. Editor: instala VS Code (https://code.visualstudio.com/) con las extensiones *Python* y *Jupyter*.
8. Cuenta de GitHub: https://github.com (para subir el proyecto).

**Por qué Python 3.11:** funciona con todas las librerías del proyecto, incluidas las de las etapas
siguientes. Alternativas: 3.12 o 3.13, que a veces tardan en tener soporte de todas las librerías.

**Verificación:** tienes Python 3.11, Git, 8 GB de RAM o más, 5 GB de disco libres y cuentas de Kaggle y GitHub.

---

## A1 - Preparar carpeta, Git y entorno virtual

**Objetivo:** tener el proyecto organizado, versionado y aislado del resto de tu computadora.

**Pasos:**
1. Descarga y descomprime `fraud-detection-platform-docs.zip` donde guardes tus proyectos
   (por ejemplo `Documentos/proyectos`). Debe quedar la carpeta `fraud-detection-platform/`
   con `README.md`, `requirements.txt`, `.gitignore`, `.env.example` y `docs/`.
2. Abre la terminal y entra a la carpeta:
   ```
   cd ruta/hacia/fraud-detection-platform
   ```
3. Inicia Git y configura tu identidad (una sola vez por computadora):
   ```
   git init
   git config user.name "Tu Nombre"
   git config user.email "tu_correo@ejemplo.com"
   ```
4. Crea las carpetas.
   - macOS/Linux:
     ```
     mkdir -p data/raw data/interim data/processed models notebooks reports/figures src/ml tests
     ```
   - Windows PowerShell:
     ```
     New-Item -ItemType Directory -Force -Path data/raw, data/interim, data/processed, models, notebooks, reports/figures, src/ml, tests
     ```
5. Crea los archivos vacíos `__init__.py` (hacen que Python trate las carpetas como paquetes):
   - macOS/Linux: `touch src/__init__.py src/ml/__init__.py tests/__init__.py`
   - Windows PowerShell: `New-Item -ItemType File -Force src/__init__.py, src/ml/__init__.py, tests/__init__.py`
6. Crea el entorno virtual:
   - Windows PowerShell: `py -3.11 -m venv .venv`
   - macOS/Linux: `python3.11 -m venv .venv`
7. Actívalo:
   - Windows PowerShell: `.venv\Scripts\Activate.ps1`
     (si dice que la ejecución de scripts está deshabilitada, ejecuta una vez
     `Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned` y vuelve a activar)
   - macOS/Linux: `source .venv/bin/activate`
8. Comprueba que está activo: el prompt debe empezar con `(.venv)`, y `python --version` debe mostrar 3.11.x.
9. Actualiza pip: `python -m pip install --upgrade pip`
10. Crea un repositorio **vacío** en GitHub (botón *New repository*, sin README ni .gitignore). Copia su URL.
11. Primer commit y subida:
    ```
    git add .
    git commit -m "chore: initial project structure and documentation"
    git branch -M main
    git remote add origin URL_DE_TU_REPOSITORIO
    git push -u origin main
    ```

**Por qué entorno virtual:** instala las librerías solo para este proyecto y evita conflictos de versiones.
Alternativas: conda (válido, pero más pesado) o instalar todo globalmente (se rompe con el tiempo).

**Por qué `data/` y `models/` no se suben a Git:** son archivos grandes y regenerables. El `.gitignore` ya los excluye.

**Verificación:** `git status` dice "nothing to commit, working tree clean"; `git log --oneline` muestra
tu commit; el repositorio en GitHub muestra los archivos; el prompt tiene `(.venv)`.

---

## A2 - Instalar dependencias y fijar versiones

**Objetivo:** instalar las librerías de la Etapa A y guardar las versiones exactas.

**Pasos:**
1. Con el entorno activo, instala PyTorch en su versión para CPU **antes** que lo demás
   (la versión por defecto en Linux descarga varios GB de soporte para GPU que no necesitamos):
   ```
   pip install torch --index-url https://download.pytorch.org/whl/cpu
   ```
2. Instala el resto (el archivo tiene activo solo el bloque de la Etapa A):
   ```
   pip install -r requirements.txt
   ```
3. macOS únicamente: si XGBoost da un error mencionando `libomp`, ejecuta `brew install libomp`.
4. Comprueba las versiones importantes:
   ```
   python -c "import pandas, numpy, pyarrow, sklearn, xgboost, torch, mlflow; print('pandas', pandas.__version__); print('xgboost', xgboost.__version__); print('torch', torch.__version__); print('mlflow', mlflow.__version__)"
   ```
5. Guarda las versiones exactas:
   ```
   pip freeze > requirements-lock.txt
   ```
6. Commit:
   ```
   git add requirements-lock.txt
   git commit -m "chore: pin exact dependency versions"
   ```

**Dos detalles que dependen de la versión instalada:**
- XGBoost debe ser **2.0 o superior** (el parámetro `early_stopping_rounds` se pasa al crear el modelo).
  Si es menor: `pip install --upgrade xgboost`.
- MLflow 3 cambió el nombre de un argumento de `log_model` (de `artifact_path` a `name`). Cuando llegue
  el momento, usamos el que indique la documentación de la versión instalada.

**Verificación:** el comando del punto 4 imprime las cuatro versiones sin errores y existe `requirements-lock.txt`.

**Si falla:** `ModuleNotFoundError` → el entorno no está activo; actívalo (A1, punto 7).

---

## A3 - Descargar PaySim

**Objetivo:** tener el CSV original en `data/raw/`.

**Pasos:**
1. Entra a https://www.kaggle.com/datasets/ealaxi/paysim1 con tu sesión de Kaggle.
2. Pulsa **Download** (descarga `archive.zip`).
3. Descomprime el zip. Dentro hay un solo archivo: `PS_20174392719_1491204439457_log.csv`.
4. Muévelo a `data/raw/`. La ruta final debe ser:
   `fraud-detection-platform/data/raw/PS_20174392719_1491204439457_log.csv`
5. **No lo renombres** (conserva el nombre original para trazabilidad) y **no lo subas a Git**.
6. Comprueba la primera línea (nombres de columnas):
   ```
   python -c "print(open('data/raw/PS_20174392719_1491204439457_log.csv').readline())"
   ```

**Por qué descarga manual:** la CLI de Kaggle exige configurar un token y esa configuración cambia con el tiempo.
Alternativa: CLI de Kaggle (https://github.com/Kaggle/kaggle-api).

**Verificación:** el comando imprime exactamente:
`step,type,amount,nameOrig,oldbalanceOrg,newbalanceOrig,nameDest,oldbalanceDest,newbalanceDest,isFraud,isFlaggedFraud`
y el archivo pesa aproximadamente medio gigabyte. `git status` no debe mostrar el CSV (está ignorado).

---

## A4 - Crear `src/ml/config.py`

**Objetivo:** un solo lugar con rutas, nombres de columnas, semilla y listas de features. El resto de
los scripts importa de aquí y no repite textos.

**Archivo:** `src/ml/config.py` (no contiene lógica, solo constantes y una función pequeña).

**Contenido, en este orden:**

| Bloque | Nombre en el código | Valor |
|---|---|---|
| Docstring | - | Explica que es la configuración central de la Etapa A |
| Raíz del proyecto | `PROJECT_ROOT` | `Path(__file__).resolve().parents[2]` |
| Carpetas | `RAW_DIR`, `INTERIM_DIR`, `PROCESSED_DIR`, `MODELS_DIR`, `REPORTS_DIR`, `FIGURES_DIR` | `PROJECT_ROOT / "data" / "raw"`, etc. |
| CSV original | `RAW_CSV_PATH` | `RAW_DIR / "PS_20174392719_1491204439457_log.csv"` |
| Salidas | `RENAMED_PARQUET_PATH`, `FEATURES_PARQUET_PATH`, `TRAIN_PATH`, `VALID_PATH`, `TEST_PATH` | `INTERIM_DIR / "transactions_renamed.parquet"`, `PROCESSED_DIR / "transactions_features.parquet"`, `PROCESSED_DIR / "train.parquet"`, `"valid.parquet"`, `"test.parquet"` |
| Modelos | `SCALER_PATH`, `MODEL_INFO_PATH` | `MODELS_DIR / "scaler.joblib"`, `MODELS_DIR / "model_info.json"` |
| Semilla | `RANDOM_STATE` | `42` |
| Tipos de lectura del CSV | `RAW_DTYPES` | diccionario columna original → tipo (tabla de abajo) |
| Renombrado de columnas | `COLUMN_NAME_MAP` | las 11 equivalencias de la tabla de abajo |
| Renombrado de valores de `tipo` | `TYPE_VALUE_MAP` | `{"CASH_IN": "deposito", "CASH_OUT": "retiro", "DEBIT": "debito", "PAYMENT": "pago", "TRANSFER": "transferencia"}` |
| Nombres clave | `ID_COLUMN`, `TARGET_COLUMN`, `RULE_COLUMN`, `TIME_COLUMN` | `"id_transaccion"`, `"es_fraude"`, `"es_marcado_por_regla"`, `"hora_simulada"` |
| Datos esperados | `EXPECTED_ROW_COUNT`, `EXPECTED_FRAUD_COUNT`, `EXPECTED_MAX_HOUR` | `6362620`, `8213`, `743` |
| Features | `FEATURE_COLUMNS` | lista de 15 nombres (ver A7) |
| Columnas monetarias | `MONEY_COLUMNS` | `["monto", "saldo_inicial_origen", "saldo_final_origen", "saldo_inicial_destino", "saldo_final_destino", "error_saldo_origen", "error_saldo_destino"]` |
| Partición | `TRAIN_SHARE`, `VALID_SHARE`, `MIN_FRAUDS_PER_PARTITION` | `0.70`, `0.15`, `500` |
| Umbral | `MIN_PRECISION` | `0.80` |
| MLflow | `MLFLOW_TRACKING_URI`, `MLFLOW_EXPERIMENT_NAME`, `REGISTERED_MODEL_NAME` | `"sqlite:///mlflow.db"`, `"fraud_detection"`, `"fraud_model"` |
| Función | `create_directories()` | Crea todas las carpetas con `mkdir(parents=True, exist_ok=True)` |

**Tipos de lectura (`RAW_DTYPES`) y renombrado (`COLUMN_NAME_MAP`):**

| Columna original | Nombre en español | Tipo de lectura | Por qué ese tipo |
|---|---|---|---|
| `step` | `hora_simulada` | `int16` | Va de 1 a 743; `int16` llega a 32,767 y ahorra memoria frente a `int64` |
| `type` | `tipo` | `string[pyarrow]` | Texto; luego se traduce con `TYPE_VALUE_MAP` |
| `amount` | `monto` | `float64` | Dinero: `float32` pierde centavos en montos grandes |
| `nameOrig` | `cuenta_origen` | `string[pyarrow]` | Casi un identificador único por fila; `category` no ahorra nada |
| `oldbalanceOrg` | `saldo_inicial_origen` | `float64` | Dinero |
| `newbalanceOrig` | `saldo_final_origen` | `float64` | Dinero |
| `nameDest` | `cuenta_destino` | `string[pyarrow]` | Identificador |
| `oldbalanceDest` | `saldo_inicial_destino` | `float64` | Dinero |
| `newbalanceDest` | `saldo_final_destino` | `float64` | Dinero |
| `isFraud` | `es_fraude` | `int8` | Solo 0 o 1 |
| `isFlaggedFraud` | `es_marcado_por_regla` | `int8` | Solo 0 o 1 |

**Verificación:**
```
python -c "from src.ml import config; print(config.PROJECT_ROOT); print(config.RAW_CSV_PATH.exists())"
```
Debe imprimir la ruta de tu proyecto y `True`. Después: `ruff check src tests` sin errores.

**Commit:** `git add src && git commit -m "feat: add central config for stage A"`

---

## A5 - Cargar, validar y renombrar a español

**Objetivo:** leer el CSV con tipos correctos, comprobar que es el dataset esperado, traducir columnas y valores,
crear `id_transaccion` y guardar en Parquet.

**Archivo:** `src/ml/load_and_rename.py`

**Funciones (cada una hace una sola cosa), en este orden:**

| Función | Qué hace |
|---|---|
| `read_raw_csv(csv_path)` | Lee el CSV con `pd.read_csv(csv_path, dtype=config.RAW_DTYPES)`. Devuelve un DataFrame |
| `validate_raw_dataframe(raw_df)` | Comprueba, una por una, y lanza `ValueError` con mensaje claro si falla: (1) las 11 columnas originales exactas, (2) `len == EXPECTED_ROW_COUNT`, (3) cero valores nulos en todo el DataFrame, (4) `step` entre 1 y `EXPECTED_MAX_HOUR`, (5) `type` contiene exactamente los 5 valores de `TYPE_VALUE_MAP`, (6) `isFraud` solo 0 o 1, (7) cantidad de fraudes igual a `EXPECTED_FRAUD_COUNT`, (8) `step` está ordenado de menor a mayor (`raw_df["step"].is_monotonic_increasing`) |
| `rename_to_spanish(raw_df)` | `raw_df.rename(columns=config.COLUMN_NAME_MAP)` y traduce `tipo` con `.map(config.TYPE_VALUE_MAP)` |
| `check_translation(renamed_df)` | Comprueba que `tipo` no tiene nulos tras el `map` (un nulo significa un valor sin traducir) |
| `add_transaction_id(renamed_df)` | Inserta en la posición 0 la columna `id_transaccion` con valores 1 a N, tipo `int32` |
| `save_parquet(dataframe, output_path)` | `dataframe.to_parquet(output_path, index=False, compression="snappy")` |
| `main()` | Llama a las funciones en orden, imprime un resumen (filas, columnas, fraudes) y la ruta guardada |

**Parámetros y decisiones:**

| Parámetro | Valor | Por qué | Otras opciones |
|---|---|---|---|
| `dtype=RAW_DTYPES` | Tipos explícitos | Menos memoria y sin sorpresas | Que pandas adivine (usa `int64`/`object`: el doble de memoria) |
| `compression` | `"snappy"` | Rápida de leer y escribir | `gzip` (menor tamaño, más lenta), `zstd` (equilibrada) |
| `index=False` | Sin índice en el archivo | El índice de pandas no tiene significado; usamos `id_transaccion` | `True` (agrega una columna fantasma) |
| Validar antes de transformar | Sí | Si el archivo no es el esperado, nada de lo que sigue vale | No validar |
| Formato de salida | Parquet | Conserva tipos y es mucho más rápido que CSV | CSV (pierde tipos) |

**Ejecución:** `python -m src.ml.load_and_rename`

**Verificación:**
1. No hay errores y el resumen muestra **6,362,620 filas, 12 columnas y 8,213 fraudes**.
2. Existe `data/interim/transactions_renamed.parquet`.
3. Comprueba los nombres:
   ```
   python -c "import pandas as pd; df = pd.read_parquet('data/interim/transactions_renamed.parquet'); print(df.columns.tolist()); print(df['tipo'].value_counts())"
   ```
   Columnas esperadas, en orden: `id_transaccion, hora_simulada, tipo, monto, cuenta_origen, saldo_inicial_origen, saldo_final_origen, cuenta_destino, saldo_inicial_destino, saldo_final_destino, es_fraude, es_marcado_por_regla`.
   Conteo esperado de `tipo`: retiro 2,237,500; pago 2,151,495; deposito 1,399,284; transferencia 532,909; debito 41,432.

**Si falla:**
- `MemoryError` → cierra otros programas y reintenta; si persiste, tu RAM es insuficiente para esta etapa.
- `ValueError` de validación → el archivo no es el original; vuelve a descargarlo (A3).

**Commit:** `git add src && git commit -m "feat: load PaySim, validate it and rename columns to Spanish"`

---

## A6 - Explorar los datos (notebook)

**Objetivo:** entender los datos antes de modelar y anotar hallazgos que guíen las features y la partición.

**Archivo:** `notebooks/01_exploration.ipynb`

**Pasos para crearlo:**
1. Con el entorno activo y desde la carpeta raíz: `jupyter lab`. Se abre el navegador.
2. En el panel izquierdo entra a `notebooks/`, pulsa **New → Notebook**, elige el kernel Python 3 y renómbralo `01_exploration.ipynb`.
3. **Primera celda (exacta):** hace que el notebook encuentre `src/`:
   ```python
   import sys
   from pathlib import Path

   sys.path.append(str(Path.cwd().parent))

   import pandas as pd
   import matplotlib.pyplot as plt
   from src.ml import config
   ```
4. **Una pregunta por celda**, cada una con una celda de texto (Markdown) que anota lo que ves.

**Preguntas, en orden, con lo esperado:**

| # | Pregunta | Cómo se responde | Esperado / qué anotar |
|---|---|---|---|
| 1 | ¿Cuántas filas y columnas? | `df.shape` | (6362620, 12) |
| 2 | ¿Hay nulos? | `df.isnull().sum()` | Todo en 0 |
| 3 | ¿Hay filas duplicadas (ignorando el id)? | `df.drop(columns=["id_transaccion"]).duplicated().sum()` | Anota el número |
| 4 | ¿Qué tan desbalanceado está? | `df["es_fraude"].value_counts()` y el porcentaje | 8,213 fraudes ≈ 0.13%; unos 774 normales por cada fraude |
| 5 | ¿Cuántas transacciones hay por `tipo`? | `df["tipo"].value_counts()` | Los conteos de A5 |
| 6 | ¿En qué tipos ocurre el fraude? | `df.groupby("tipo")["es_fraude"].sum()` | Solo `transferencia` (4,097) y `retiro` (4,116); los demás, 0 |
| 7 | ¿Cómo se distribuyen los montos? | `df.groupby("es_fraude")["monto"].describe()` e histograma con eje X en escala logarítmica (`plt.xscale("log")`) | Anota si los fraudes tienden a montos mayores |
| 8 | ¿Cuántos `saldo_inicial_origen` son cero y cuántos `saldo_final_origen` son cero, en fraudes y en no fraudes? | Comparar porcentajes con `(df["saldo_final_origen"] == 0).groupby(df["es_fraude"]).mean()` | Anota los porcentajes: guían la feature `vacia_cuenta_origen` |
| 9 | ¿Cambia la tasa de fraude con el tiempo? | `df.groupby("hora_simulada")["es_fraude"].agg(["size", "sum"])` y un gráfico de línea | Anota si el volumen y los fraudes por hora son estables: guía la partición (A8) |
| 10 | ¿Los saldos cuadran? | Calcula `saldo_final_origen + monto - saldo_inicial_origen` y compara `describe()` entre fraude y no fraude | Anota si los fraudes tienen errores distintos de cero: guía `error_saldo_origen` |
| 11 | ¿Cuántas transacciones marcó la regla del dataset? | `df["es_marcado_por_regla"].sum()` y cuántas de ellas son fraude | Debe ser 16; anota cuántas son fraude |
| 12 | ¿Las cuentas se repiten? | `df["cuenta_origen"].nunique()` y `df["cuenta_destino"].nunique()` | Casi una cuenta por fila: no sirven como feature |
| 13 | ¿Los destinos que empiezan con "M" (comercios) tienen fraude? | `df["cuenta_destino"].str.startswith("M")` agrupado por `es_fraude` | Anota el resultado |

5. **Guarda 3 figuras** con `plt.savefig(config.FIGURES_DIR / "nombre.png", dpi=150, bbox_inches="tight")`:
   `fraud_by_type.png` (pregunta 6), `amount_distribution.png` (7), `fraud_over_time.png` (9).
6. **Última celda de texto: "Conclusiones".** Escribe 5 hallazgos con tus palabras. Deben contestar:
   (a) qué tipos de transacción importan, (b) si los montos distinguen el fraude, (c) si los saldos cuadran en los fraudes,
   (d) si el fraude cambia con el tiempo, (e) por qué no usaremos las cuentas como feature.

**Decisiones que dejan estas conclusiones (las usan los pasos siguientes):**

| Decisión | Valor | Por qué | Alternativa |
|---|---|---|---|
| Features con identificadores de cuenta | No | Casi únicos por fila: el modelo memorizaría en lugar de aprender patrones, y no sirven con cuentas nuevas | Features agregadas por cuenta (historial); es más avanzado y queda para una mejora futura |
| `es_marcado_por_regla` como feature | No | Es el resultado de otra regla; se usa solo como línea base | Usarla (infla el resultado sin aportar) |
| `hora_simulada` como feature | No | Con partición por tiempo, el modelo aprendería "las horas tardías tienen más fraude" y no generalizaría | Usarla (riesgo de fuga temporal) |
| Filtrar solo `transferencia` y `retiro` | No, se mantienen todos los tipos | Es más realista; el modelo debe aprender que los otros tipos no tienen fraude | Filtrar (resultado más fácil pero menos realista) |

**Verificación:** el notebook corre de principio a fin con *Kernel → Restart Kernel and Run All Cells* sin errores,
las 3 figuras están en `reports/figures/` y la celda de conclusiones está escrita.

**Commit:** `git add notebooks reports && git commit -m "docs: add exploratory analysis notebook and figures"`

---

## A7 - Crear features

**Objetivo:** convertir las columnas originales en variables que el modelo pueda usar.

**Archivo:** `src/ml/build_features.py`

**Entrada:** `data/interim/transactions_renamed.parquet`. **Salida:** `data/processed/transactions_features.parquet`.

**Definición exacta de cada feature** (esta definición se replica después en SQL, en dbt, así que debe quedar exacta):

| Feature | Fórmula | Tipo | Qué representa |
|---|---|---|---|
| `monto` | columna original | float64 | Monto de la transacción |
| `saldo_inicial_origen` | columna original | float64 | |
| `saldo_final_origen` | columna original | float64 | |
| `saldo_inicial_destino` | columna original | float64 | |
| `saldo_final_destino` | columna original | float64 | |
| `hora_del_dia` | `hora_simulada % 24` | int8 | Hora del día (0 a 23) |
| `error_saldo_origen` | `saldo_final_origen + monto - saldo_inicial_origen` | float64 | Es 0 si el saldo del origen cuadra con el monto |
| `error_saldo_destino` | `saldo_inicial_destino + monto - saldo_final_destino` | float64 | Es 0 si el saldo del destino cuadra con el monto |
| `vacia_cuenta_origen` | 1 si `saldo_inicial_origen > 0` y `saldo_final_origen == 0`, si no 0 | int8 | La transacción deja la cuenta de origen en cero |
| `destino_sin_saldo_previo` | 1 si `saldo_inicial_destino == 0`, si no 0 | int8 | El destino estaba vacío |
| `tipo_deposito` | 1 si `tipo == "deposito"`, si no 0 | int8 | One-hot de `tipo` |
| `tipo_retiro` | 1 si `tipo == "retiro"` | int8 | |
| `tipo_debito` | 1 si `tipo == "debito"` | int8 | |
| `tipo_pago` | 1 si `tipo == "pago"` | int8 | |
| `tipo_transferencia` | 1 si `tipo == "transferencia"` | int8 | |

Esto da **15 features**. Esa lista, en ese orden, es `FEATURE_COLUMNS` en `config.py`: **complétala ahora** con estos 15 nombres.
Orden: `monto`, `saldo_inicial_origen`, `saldo_final_origen`, `saldo_inicial_destino`, `saldo_final_destino`, `hora_del_dia`,
`error_saldo_origen`, `error_saldo_destino`, `vacia_cuenta_origen`, `destino_sin_saldo_previo`, `tipo_deposito`, `tipo_retiro`,
`tipo_debito`, `tipo_pago`, `tipo_transferencia`.
**Funciones:**

| Función | Qué hace |
|---|---|
| `add_time_features(df)` | Agrega `hora_del_dia` |
| `add_balance_features(df)` | Agrega `error_saldo_origen`, `error_saldo_destino`, `vacia_cuenta_origen`, `destino_sin_saldo_previo` |
| `add_type_features(df)` | Agrega las 5 columnas `tipo_*` comparando `df["tipo"] == "..."` y convirtiendo a `int8` (sin `get_dummies`, para que el conjunto de columnas sea siempre el mismo, también en producción) |
| `check_features(df)` | Comprueba que existen todas las columnas de `FEATURE_COLUMNS`, que no hay nulos ni infinitos, y que cada fila tiene exactamente un `tipo_*` en 1 |
| `main()` | Lee el Parquet, aplica las tres funciones, comprueba, guarda con las columnas `id_transaccion`, `hora_simulada`, `es_fraude`, `es_marcado_por_regla` y las `FEATURE_COLUMNS`, e imprime el resumen |

Las columnas `cuenta_origen`, `cuenta_destino` y `tipo` (texto) **no** se guardan en el archivo de features.

**Prueba automática:** `tests/test_build_features.py`, con un DataFrame de 3 filas escrito a mano:

| Fila | tipo | hora_simulada | monto | saldo_inicial_origen | saldo_final_origen | saldo_inicial_destino | saldo_final_destino |
|---|---|---|---|---|---|---|---|
| 1 | transferencia | 25 | 100 | 100 | 0 | 0 | 0 |
| 2 | pago | 48 | 50 | 200 | 150 | 0 | 0 |
| 3 | deposito | 3 | 30 | 0 | 0 | 500 | 470 |

Resultados esperados que la prueba debe afirmar:

| Fila | hora_del_dia | error_saldo_origen | error_saldo_destino | vacia_cuenta_origen | destino_sin_saldo_previo | columna `tipo_*` en 1 |
|---|---|---|---|---|---|---|
| 1 | 1 | 0 | 100 | 1 | 1 | `tipo_transferencia` |
| 2 | 0 | 0 | 50 | 0 | 1 | `tipo_pago` |
| 3 | 3 | 30 | 60 | 0 | 0 | `tipo_deposito` |

Cálculo de la fila 3: `error_saldo_origen = 0 + 30 - 0 = 30` y `error_saldo_destino = 500 + 30 - 470 = 60`.
Los valores de la fila 3 no son realistas a propósito: solo sirven para comprobar la fórmula.

**Ejecución:**
```
python -m pytest tests/test_build_features.py -v
python -m src.ml.build_features
```

**Verificación:** la prueba pasa; el script imprime 6,362,620 filas y las columnas de features; existe
`data/processed/transactions_features.parquet`; la suma de las columnas `tipo_*` en cada fila es 1.

**Commit:** `git add src tests && git commit -m "feat: build model features with tests"`

---

## A8 - Partir los datos por tiempo

**Objetivo:** separar entrenamiento, validación y prueba **por orden cronológico**, como pasaría en la vida real
(el modelo se entrena con el pasado y se usa en el futuro).

**Archivo:** `src/ml/split_data.py`

**Entrada:** `data/processed/transactions_features.parquet`. **Salidas:** `train.parquet`, `valid.parquet`, `test.parquet` en `data/processed/`.

**Funciones:**

| Función | Qué hace |
|---|---|
| `find_cutoff_hours(features_df)` | Cuenta filas por `hora_simulada` (`groupby(...).size().sort_index()`), calcula la proporción acumulada de filas (`cumsum() / total`), y devuelve (a) la primera hora cuya proporción acumulada es ≥ `TRAIN_SHARE` y (b) la primera hora cuya proporción acumulada es ≥ `TRAIN_SHARE + VALID_SHARE` |
| `split_by_hour(features_df, train_last_hour, valid_last_hour)` | Crea tres DataFrames con máscaras: `hora <= train_last_hour`; `hora > train_last_hour` y `<= valid_last_hour`; `hora > valid_last_hour` |
| `check_split(train_df, valid_df, test_df, total_rows, total_frauds)` | Comprueba: (1) las filas suman el total, (2) los fraudes suman el total, (3) cada partición tiene al menos `MIN_FRAUDS_PER_PARTITION` fraudes, (4) la hora máxima de train es menor que la mínima de valid, y la máxima de valid menor que la mínima de test |
| `save_partitions(...)` | Guarda los tres Parquet |
| `main()` | Ejecuta todo e imprime una tabla: partición, rango de horas, filas, fraudes y tasa de fraude |

**Parámetros y decisiones:**

| Parámetro | Valor | Por qué | Otras opciones |
|---|---|---|---|
| Corte por **proporción de filas**, no por número de hora | 70% / 15% / 15% de las filas | El volumen por hora no es uniforme; cortar por hora daría particiones de tamaños muy distintos | Cortar por hora fija (por ejemplo 520 y 630) |
| Nunca partir una hora entre dos particiones | Sí | Evita que transacciones de la misma hora estén en entrenamiento y prueba | Cortar por número de fila |
| Mínimo de fraudes por partición | 500 | Con muy pocos fraudes las métricas de la partición son inestables | Más (más estable) o menos (más ruido) |
| Partición aleatoria | No | Mezclaría futuro y pasado y daría resultados demasiado optimistas | `train_test_split` (válido si no hubiera tiempo) |

**Ejecución:** `python -m src.ml.split_data`

**Verificación:**
1. Los 4 chequeos de `check_split` pasan.
2. Las filas de las tres particiones suman 6,362,620 y los fraudes suman 8,213.
3. Existen los tres archivos Parquet.
4. Anota en un papel (o en el notebook) el rango de horas y la tasa de fraude de cada partición: si la tasa cambia mucho entre particiones (hallazgo de A6, pregunta 9), lo mencionarás en la decisión final.

**Si falla (menos de 500 fraudes en una partición):** cambia `TRAIN_SHARE` en `config.py` (por ejemplo a `0.65`), vuelve a ejecutar y anota el cambio.

**Commit:** `git add src && git commit -m "feat: split data by time into train, valid and test"`

---

## A9 - Crear métricas y utilidades compartidas

**Objetivo:** tener en un solo lugar cómo se miden y guardan los resultados de todos los modelos, para que la comparación sea justa.

### Archivo 1: `src/ml/metrics.py`

| Función | Qué hace |
|---|---|
| `evaluate_scores(y_true, scores, threshold)` | `predictions = (scores >= threshold)`. Devuelve un diccionario con: `pr_auc` (`average_precision_score(y_true, scores)`), `precision`, `recall`, `f1` (con `zero_division=0`), `true_positives`, `false_positives`, `false_negatives`, `true_negatives`, `threshold` y `alert_rate` (alertas / total) |
| `find_threshold_for_precision(y_true, scores, min_precision)` | Usa `precision_recall_curve`. Entre los umbrales con `precision >= min_precision` elige el que tiene **mayor recall** (el umbral más bajo que cumple). Devuelve `(threshold, reached_target)`. Si ninguno cumple, devuelve el umbral de mejor F1 y `reached_target = False` |

**Por qué estas métricas:** con 0.13% de fraude, *accuracy* engaña (decir "nunca hay fraude" da 99.87%).
`pr_auc` mide qué tan bien ordena el modelo (sin depender de un umbral); precision y recall describen el
comportamiento con un umbral concreto. Alternativa: `roc_auc` (se ve inflado con clases tan desbalanceadas).

**Por qué el mismo método de umbral para todos los modelos:** comparación justa. Alternativas: percentil fijo del error
(solo aplicaría al autoencoder), umbral 0.5 (arbitrario con clases desbalanceadas).

### Archivo 2: `src/ml/model_info.py`

Guarda en `models/model_info.json` los datos que necesitan los pasos siguientes.

| Función | Qué hace |
|---|---|
| `save_model_info(model_name, run_id, model_uri, threshold, valid_pr_auc, model_path)` | Lee el JSON (si existe), agrega o reemplaza la entrada de `model_name` y lo guarda con `json.dump(..., indent=2)` |
| `load_model_info()` | Devuelve el diccionario completo (o uno vacío si no existe el archivo) |

Nombres de modelo permitidos: `"rule"`, `"logistic_regression"`, `"xgboost"`, `"autoencoder"`.

### Pruebas: `tests/test_metrics.py`

| Prueba | Datos | Resultado esperado |
|---|---|---|
| Métricas básicas | `y_true=[0,0,1,1]`, `scores=[0.1,0.4,0.35,0.8]`, `threshold=0.5` | TP=1, FP=0, FN=1, TN=2; precision=1.0; recall=0.5; f1≈0.6667; pr_auc≈0.8333 |
| Umbral alcanzable | `y_true=[0,0,1,1,1]`, `scores=[0.1,0.2,0.6,0.7,0.9]`, `min_precision=1.0` | threshold=0.6; reached_target=True |
| Umbral inalcanzable | `y_true=[0,1,0,1]`, `scores=[0.9,0.8,0.7,0.6]`, `min_precision=1.0` | reached_target=False |

**Ejecución:** `python -m pytest tests -v`

**Verificación:** todas las pruebas de `test_build_features.py` y `test_metrics.py` pasan; `ruff check src tests` sin errores.

**Commit:** `git add src tests && git commit -m "feat: add shared metrics and model info helpers with tests"`

---

## A10 - Fijar el criterio de "modelo útil" (antes de entrenar)

**Objetivo:** escribir, **antes de ver resultados**, qué significa que el modelo sirva. Si el criterio se escribe después,
inconscientemente se ajusta al resultado.

**Archivo:** `reports/decision_etapa_A.md`

**Contenido inicial (copia esto y deja las secciones de resultados vacías):**

```markdown
# Decisión de la Etapa A

Fecha del criterio: (escribe la fecha de hoy)

## 1. Criterio de modelo útil (escrito antes de entrenar)

Modelo candidato = el de mayor PR-AUC en validación entre regresión logística, XGBoost y autoencoder.

- C1: su PR-AUC en prueba es mayor que el de la regla del dataset y que el de la regresión logística.
- C2: con el umbral elegido en validación para precision mínima de 0.80, su recall en prueba es de al menos 0.70.
- C3: la diferencia de PR-AUC entre validación y prueba es de 0.05 o menos (estabilidad en el tiempo).

Por qué esos valores:
- Precision 0.80 significa que 4 de cada 5 alertas son fraude real; menos, y los analistas se saturan de falsas alarmas.
- Recall 0.70 significa que se detecta al menos 7 de cada 10 fraudes.
- Estos números son una propuesta razonable, no una verdad. Se pueden cambiar AHORA, no después de ver resultados.

## 2. Resultados (se completa en A16)

## 3. Decisión (se completa en A16)
```

**Parámetros y decisiones:**

| Decisión | Valor | Por qué | Alternativa |
|---|---|---|---|
| Elegir el modelo por validación, no por prueba | Sí | La prueba solo confirma; elegir con ella la "contamina" | Elegir por prueba (optimista) |
| Precision mínima | 0.80 | Equilibrio entre alertas útiles y trabajo para analistas | Un criterio basado en costos (costo de un fraude no detectado frente al de una revisión); más realista pero requiere datos de costos que no tenemos |

**Verificación y commit (el commit es la prueba de que el criterio es anterior a los resultados):**
```
git add reports
git commit -m "docs: fix success criterion before training"
```
`git log --oneline` muestra este commit antes de cualquier commit de entrenamiento.

---

## A11 - Crear el escalado de datos

**Objetivo:** preparar los datos para la regresión logística y el autoencoder, que necesitan variables en escalas parecidas.
(XGBoost no lo necesita y usa los datos sin escalar.)

**Archivo:** `src/ml/scaling.py`

| Función | Qué hace |
|---|---|
| `signed_log1p(values)` | `np.sign(values) * np.log1p(np.abs(values))`. Comprime montos enormes y mantiene el signo (los errores de saldo pueden ser negativos) |
| `transform_money_columns(df)` | Copia `df[FEATURE_COLUMNS]` y aplica `signed_log1p` a las columnas de `MONEY_COLUMNS` |
| `fit_scaler(train_df)` | Transforma con `transform_money_columns`, ajusta un `StandardScaler` **solo con train** y lo guarda en `SCALER_PATH` con `joblib.dump` |
| `load_scaler()` | Carga el scaler con `joblib.load` |
| `scale_features(df, scaler)` | `scaler.transform(transform_money_columns(df))` y devuelve un arreglo `float32` |
| `main()` | Lee `train.parquet`, ajusta y guarda el scaler, e imprime la media y desviación de las columnas escaladas |

**Parámetros y decisiones:**

| Parámetro | Valor | Por qué | Otras opciones |
|---|---|---|---|
| Transformación logarítmica con signo | En columnas monetarias | Los montos van de centavos a millones; sin esto dominan la escala | Sin transformar (el escalado queda dominado por valores extremos) |
| Escalador | `StandardScaler` | Media 0 y desviación 1, estándar para redes y modelos lineales | `MinMaxScaler` (muy sensible a extremos), `RobustScaler` (válido, usa mediana y cuartiles) |
| Ajustar solo con `train` | Sí | Si se ajusta con todos los datos, la prueba "filtra" información al entrenamiento | Ajustar con todo (fuga de datos) |
| Tipo de salida | `float32` | Mitad de memoria; es lo que PyTorch usa | `float64` (el doble de memoria sin ganancia) |
| `hora_del_dia` sin transformación cíclica | Se escala como número | Simple para esta etapa | Codificar con seno y coseno (mejor para ciclos; mejora posible) |

**Prueba:** `tests/test_scaling.py` comprueba que `signed_log1p([-9, 0, 99])` da aproximadamente `[-2.302585, 0.0, 4.605170]`.

**Ejecución:**
```
python -m pytest tests -v
python -m src.ml.scaling
```

**Verificación:** las pruebas pasan; existe `models/scaler.joblib`; las medias impresas están cerca de 0 y las desviaciones cerca de 1.

**Commit:** `git add src tests && git commit -m "feat: add feature scaling fitted on train only"`

---

## A12 - Líneas base: regla y regresión logística

**Objetivo:** tener contra qué comparar. Si un modelo complejo no supera estas líneas base, no vale la pena.

**Archivo:** `src/ml/train_baseline.py`

**Pasos del script (en este orden):**
1. Configurar MLflow: `mlflow.set_tracking_uri(config.MLFLOW_TRACKING_URI)` y `mlflow.set_experiment(config.MLFLOW_EXPERIMENT_NAME)`.
2. Leer `train.parquet` y `valid.parquet`.
3. **Línea base 1, la regla:**
   - Puntaje = columna `es_marcado_por_regla`; umbral = 1.
   - Evaluar en validación con `evaluate_scores`.
   - Abrir un run de MLflow llamado `baseline_rule`; registrar el parámetro `model_type="rule"` y las métricas con prefijo `valid_` (por ejemplo `valid_pr_auc`).
   - Guardar con `save_model_info("rule", ...)` (`model_uri` y `model_path` pueden ir vacíos).
4. **Línea base 2, regresión logística:**
   - Cargar el scaler (A11); `X_train = scale_features(train, scaler)`; `y_train = train["es_fraude"]`; lo mismo para validación.
   - Crear y entrenar `LogisticRegression` con los parámetros de la tabla.
   - `scores = model.predict_proba(X_valid)[:, 1]`.
   - `find_threshold_for_precision(y_valid, scores, config.MIN_PRECISION)`; evaluar con ese umbral.
   - Abrir un run `baseline_logistic_regression`; registrar parámetros, métricas `valid_*` y `threshold_reached_target` (1 o 0); guardar el modelo con `joblib.dump` en `models/logistic_regression.joblib` y con `mlflow.sklearn.log_model`.
   - `save_model_info("logistic_regression", run_id, model_uri, threshold, valid_pr_auc, model_path)`.
5. Imprimir una tabla resumen de las dos líneas base en validación.

**Parámetros de la regresión logística:**

| Parámetro | Valor | Por qué | Otras opciones |
|---|---|---|---|
| `class_weight` | `"balanced"` | Da más peso a los fraudes, que son rarísimos | `None` (el modelo ignora el fraude) |
| `C` | `1.0` | Regularización estándar | `0.1` (más fuerte), `10` (más débil) |
| `solver` | `"lbfgs"` | Rápido para datos densos con muchas filas | `saga` (muy grande o regularización L1), `liblinear` (datasets pequeños) |
| `max_iter` | `1000` | El valor por defecto (100) suele no converger con millones de filas | Más (si sigue sin converger) |
| `random_state` | `42` | Reproducibilidad | Cualquier entero fijo |

**Ejecución:** `python -m src.ml.train_baseline` (puede tardar varios minutos)

**Verificación:**
1. Se imprime la tabla con las métricas de ambas líneas base.
2. La regla tiene recall muy bajo (solo marca 16 transacciones en todo el dataset).
3. Existen `models/logistic_regression.joblib` y `models/model_info.json` con las entradas `rule` y `logistic_regression`.
4. Abre la interfaz de MLflow: `mlflow ui --backend-store-uri sqlite:///mlflow.db` y entra a http://127.0.0.1:5000. Debe verse el experimento `fraud_detection` con 2 runs.
   (si el puerto está ocupado: agrega `--port 5001`). Para cerrar: `Ctrl + C` en la terminal.

**Commit:** `git add src && git commit -m "feat: train rule and logistic regression baselines"`

---

## A13 - Entrenar XGBoost

**Objetivo:** entrenar el clasificador principal y registrar sus resultados.

**Archivo:** `src/ml/train_xgboost.py`

**Pasos del script:**
1. Configurar MLflow igual que en A12.
2. Leer `train.parquet` y `valid.parquet`. `X = df[config.FEATURE_COLUMNS]` e `y = df["es_fraude"]` (sin escalar).
3. Calcular `negative_count`, `positive_count` y `scale_pos_weight = negative_count / positive_count`, solo con train.
4. Crear `XGBClassifier` con los parámetros de la tabla.
5. Entrenar: `model.fit(X_train, y_train, eval_set=[(X_valid, y_valid)], verbose=20)`.
6. Leer `model.best_iteration` (cuántos árboles se usaron realmente).
7. `scores = model.predict_proba(X_valid)[:, 1]`; calcular umbral con `find_threshold_for_precision`; evaluar con `evaluate_scores`.
8. Importancia de variables: `model.get_booster().get_score(importance_type="gain")` → DataFrame ordenado de mayor a menor → guardar `reports/xgboost_feature_importance.csv` y un gráfico de barras en `reports/figures/xgboost_feature_importance.png`.
9. Guardar el modelo: `model.save_model(config.MODELS_DIR / "xgboost_model.json")`.
10. En un run `xgboost`: registrar parámetros (incluido `scale_pos_weight` y `best_iteration`), métricas `valid_*`, `threshold_reached_target`, el CSV y la figura como artefactos y el modelo con `mlflow.xgboost.log_model`.
11. `save_model_info("xgboost", run_id, model_uri, threshold, valid_pr_auc, model_path)`.
12. Imprimir resumen y las 5 variables más importantes.

**Parámetros de XGBoost:**

| Parámetro | Valor | Por qué | Otras opciones |
|---|---|---|---|
| `n_estimators` | `200` | Máximo de árboles; la parada temprana puede usar menos | `50` (corto), `1000` (riesgo de sobreajuste y lentitud) |
| `max_depth` | `6` | Profundidad estándar | `3` (más simple), `10` (sobreajusta) |
| `learning_rate` | `0.1` | Equilibrio entre velocidad y precisión | `0.01` (más lento y fino), `0.3` |
| `subsample` | `0.8` | Cada árbol ve el 80% de las filas; reduce sobreajuste | `1.0` |
| `colsample_bytree` | `0.8` | Cada árbol ve el 80% de las columnas | `1.0` |
| `scale_pos_weight` | `negativos / positivos` (≈ 774) | Compensa el desbalance | `1` (ignora el desbalance), raíz cuadrada del cociente (más suave) |
| `eval_metric` | `"aucpr"` | Mide el área bajo la curva precision-recall durante el entrenamiento | `"auc"`, `"logloss"` |
| `early_stopping_rounds` | `20` | Detiene si la validación no mejora en 20 rondas | Sin parada (entrena todo) |
| `tree_method` | `"hist"` | Rápido con millones de filas | `"exact"` (muy lento aquí) |
| `n_jobs` | `-1` | Usa todos los núcleos | Un número fijo |
| `random_state` | `42` | Reproducibilidad | Cualquier entero fijo |

**Nota de honestidad:** la validación se usa para la parada temprana y para el umbral, así que sus métricas
son un poco optimistas. Por eso existe la prueba, que sigue sin tocarse.

**Ejecución:** `python -m src.ml.train_xgboost` (puede tardar varios minutos)

**Verificación:**
1. Durante el entrenamiento se imprime el progreso cada 20 rondas.
2. El PR-AUC de validación de XGBoost es mayor que el de la regresión logística (si no lo es, revisa A7 antes de seguir).
3. Existen `models/xgboost_model.json`, `reports/xgboost_feature_importance.csv` y la figura.
4. En MLflow aparece el run `xgboost` con sus parámetros y métricas.

**Commit:** `git add src reports && git commit -m "feat: train XGBoost fraud classifier"`

---

## A14 - Entrenar el autoencoder

**Objetivo:** entrenar un detector de anomalías que aprende cómo se ve una transacción **normal**; lo que no puede
reconstruir bien se considera sospechoso.

**Archivo:** `src/ml/train_autoencoder.py`

Este archivo contiene tres cosas reutilizables que importará el paso A15: la clase del modelo, la función de error de reconstrucción y el `main`.

**Elementos del script:**

| Elemento | Qué hace |
|---|---|
| `class FraudAutoencoder(nn.Module)` | Dos partes. Codificador: `Linear(n, 16)` → `ReLU` → `Linear(16, 8)` → `ReLU`. Decodificador: `Linear(8, 16)` → `ReLU` → `Linear(16, n)`. `n` es el número de features (15). El método `forward` devuelve la reconstrucción |
| `compute_reconstruction_error(model, features_array, batch_size)` | Pone el modelo en `model.eval()`, usa `torch.no_grad()`, procesa por lotes de 10,000 filas y devuelve, por fila, el error cuadrático medio entre la entrada y su reconstrucción |
| `main()` | Orquesta los pasos de abajo |

**Pasos de `main()`:**
1. Fijar semillas: `np.random.seed(42)` y `torch.manual_seed(42)`. Configurar MLflow.
2. Leer `train.parquet` y `valid.parquet`; cargar el scaler.
3. Entrenamiento: quedarse **solo con filas normales** (`es_fraude == 0`) y tomar una muestra de **500,000** (`sample(n=500000, random_state=42)`). Escalar con `scale_features`.
4. Validación de pérdida: tomar 100,000 filas normales de `valid` (misma lógica) y escalarlas.
5. Convertir a tensores `float32` y crear un `DataLoader` con `batch_size=256` y `shuffle=True`.
6. Crear el modelo, `nn.MSELoss()` y `torch.optim.Adam(model.parameters(), lr=0.001)`.
7. Bucle de **20 épocas**. En cada época: `model.train()`; por cada lote: `optimizer.zero_grad()`, calcular la salida, calcular la pérdida, `loss.backward()`, `optimizer.step()`. Al terminar la época: calcular la pérdida promedio de entrenamiento y la de validación normal (con `model.eval()` y `torch.no_grad()`), imprimirlas y registrarlas en MLflow con `step=epoch`.
8. Con el modelo entrenado, calcular el error de reconstrucción de **todas** las filas de `valid` (puntaje de anomalía). Escalarlas primero.
9. `find_threshold_for_precision` y `evaluate_scores`, igual que los demás modelos.
10. Guardar `torch.save(model.state_dict(), config.MODELS_DIR / "autoencoder.pt")`, registrar el run `autoencoder` con parámetros, métricas `valid_*` y el modelo con `mlflow.pytorch.log_model`; llamar a `save_model_info("autoencoder", ...)`.

**Parámetros:**

| Parámetro | Valor | Por qué | Otras opciones |
|---|---|---|---|
| Datos de entrenamiento | Solo filas normales | La red aprende lo "normal"; si viera fraude, también aprendería a reconstruirlo | Todas las filas |
| Tamaño de muestra | 500,000 | Suficiente para aprender lo normal y entrena en minutos | Todo el train (4 millones, mucho más lento) |
| Capas | n → 16 → 8 → 16 → n | Cuello de botella de 8: obliga a comprimir | Más capas o neuronas (más complejo, más riesgo de reconstruir también el fraude) |
| Activación | `ReLU` | Simple y estándar | `Tanh`, `LeakyReLU` |
| Pérdida | `MSELoss` | Error cuadrático medio de reconstrucción | `L1Loss` (menos sensible a extremos) |
| Optimizador | `Adam`, `lr=0.001` | Funciona bien sin ajustes finos | `SGD` (más lento de ajustar) |
| `batch_size` | `256` | Equilibrio entre estabilidad y velocidad | `64` (más lento), `1024` (más memoria) |
| `epochs` | `20` | Suficiente para que la pérdida se estabilice | Más (riesgo de sobreajuste) |
| Dispositivo | CPU | La red es muy pequeña | GPU (innecesaria) |
| Umbral | Mismo método que los demás (precision mínima 0.80 en validación) | Comparación justa | Percentil 99 del error normal (válido, pero no comparable con los demás modelos) |

**Ejecución:** `python -m src.ml.train_autoencoder` (entre 2 y 15 minutos según tu CPU)

**Verificación:**
1. La pérdida de entrenamiento de la época 20 es menor que la de la época 1.
2. La pérdida de validación normal baja y se estabiliza (si sube mientras la de entrenamiento baja, hay sobreajuste: reduce `epochs`).
3. Existen `models/autoencoder.pt` y el run `autoencoder` en MLflow.
4. Se imprime su PR-AUC de validación. Si es bajo frente a XGBoost, **no es un error**: es un hallazgo (los detectores no supervisados suelen rendir menos que los supervisados cuando hay etiquetas).

**Commit:** `git add src && git commit -m "feat: train PyTorch autoencoder for anomaly detection"`

---

## A15 - Evaluación final en prueba y comparación

**Objetivo:** medir los modelos **una sola vez** en el conjunto de prueba y compararlos. **Aquí se usa `test.parquet` por primera vez.**

**Archivo:** `src/ml/compare_models.py`

**Pasos del script:**
1. Leer `test.parquet`, el scaler y `model_info.json`.
2. Calcular el puntaje de cada modelo en prueba:
   - Regla: columna `es_marcado_por_regla`.
   - Regresión logística: `joblib.load` + `scale_features` + `predict_proba[:, 1]`.
   - XGBoost: cargar `models/xgboost_model.json` en un `XGBClassifier` y `predict_proba[:, 1]` con las features sin escalar.
   - Autoencoder: cargar `models/autoencoder.pt` en `FraudAutoencoder` (importado de `train_autoencoder`) y `compute_reconstruction_error`.
3. Evaluar cada uno con `evaluate_scores`, usando **el umbral guardado en `model_info.json`** (no se recalcula con la prueba).
4. Armar una tabla con: modelo, `valid_pr_auc`, `test_pr_auc`, `test_precision`, `test_recall`, `test_f1`, TP, FP, FN, TN y `alert_rate`.
5. Guardar la tabla como `reports/model_comparison.md` (tabla Markdown) y como CSV.
6. Dibujar las curvas precision-recall de los tres modelos con puntaje continuo (con `precision_recall_curve`) y marcar la regla como un punto; guardar `reports/figures/precision_recall_curves.png`.
7. Registrar un run `final_test_evaluation` en MLflow con las métricas de prueba (prefijo con el nombre del modelo, por ejemplo `xgboost_test_pr_auc`) y los dos archivos como artefactos.

**Reglas:** no recalcular umbrales con la prueba; no volver a entrenar nada después de mirar esta tabla sin documentarlo.

**Ejecución:** `python -m src.ml.compare_models`

**Verificación:**
1. La tabla tiene 4 filas (regla, regresión logística, XGBoost, autoencoder).
2. Los `valid_pr_auc` coinciden con los de los runs de MLflow.
3. Existen `reports/model_comparison.md` y la figura de curvas.

**Commit:** `git add src reports && git commit -m "feat: evaluate all models once on the test set"`

---

## A16 - Decisión y registro del modelo campeón

**Objetivo:** aplicar el criterio de A10 a los resultados y decidir.

**Pasos:**
1. **Elegir el candidato por validación:** el modelo (entre regresión logística, XGBoost y autoencoder) con mayor `valid_pr_auc`.
2. Completar la sección 2 de `reports/decision_etapa_A.md` con la tabla de `model_comparison.md` y con la evaluación de cada criterio:

   | Criterio | Valor obtenido | ¿Cumple? |
   |---|---|---|
   | C1: PR-AUC de prueba mayor que la regla y la regresión logística | | |
   | C2: recall en prueba ≥ 0.70 con precision mínima 0.80 en validación | | |
   | C3: \|PR-AUC validación − PR-AUC prueba\| ≤ 0.05 | | |

3. Completar la sección 3 con una de estas decisiones y su justificación en 3 a 5 líneas:
   - **SEGUIR:** cumple C1, C2 y C3 → continuar a la Etapa B.
   - **SEGUIR CON CAMBIOS:** cumple C1 pero falla C2 o C3 → documentar el riesgo y qué se vigilará en producción.
   - **NO SEGUIR:** falla C1 → no se construye el ecosistema todavía.
4. Mencionar en la sección 3 las limitaciones: PaySim es sintético y la tasa de fraude por partición (anotada en A8).
5. **Si la decisión es NO SEGUIR (o hay que mejorar):** volver a A7 (nuevas features) o A13 (parámetros), usando **solo validación** para decidir; documentar cuántas veces se evaluó la prueba.
6. **Si la decisión es SEGUIR (o SEGUIR CON CAMBIOS):** crear `src/ml/register_champion.py`, que:
   - Recibe el nombre del modelo como argumento (`sys.argv[1]`) y falla con un mensaje claro si no existe en `model_info.json`.
   - Lee su `model_uri` de `model_info.json`.
   - Registra con `mlflow.register_model(model_uri, config.REGISTERED_MODEL_NAME)`.
   - Asigna el alias `champion` con `MlflowClient().set_registered_model_alias(config.REGISTERED_MODEL_NAME, "champion", version)`.
   - Imprime el nombre, la versión y el alias.

   Ejemplo: `python -m src.ml.register_champion xgboost`

   (La forma exacta de obtener el `model_uri` y de registrar depende de la versión de MLflow instalada: sigue la documentación de esa versión. Se usa **alias** y no *stages* porque los stages están en desuso.)

**Verificación:** en la interfaz de MLflow (`mlflow ui --backend-store-uri sqlite:///mlflow.db`), pestaña **Models**, aparece `fraud_model` con la versión 1 y el alias `champion`. El archivo `reports/decision_etapa_A.md` tiene las 3 secciones completas.

**Commit:** `git add src reports && git commit -m "docs: record stage A decision and register champion model"`

---

## A17 - Cierre de la etapa

**Checklist final (todo debe estar marcado):**
- [ ] `python -m pytest tests -v` pasa completo.
- [ ] `ruff check src tests` sin errores.
- [ ] `reports/decision_etapa_A.md` completo, y su criterio se commiteó antes del entrenamiento.
- [ ] `reports/model_comparison.md` y las figuras existen.
- [ ] `notebooks/01_exploration.ipynb` corre de principio a fin.
- [ ] `requirements-lock.txt` está actualizado (`pip freeze > requirements-lock.txt` si instalaste algo más).
- [ ] `git status` limpio.

**Pasos:**
1. En el `README.md` general, agrega una sección corta "Resultados de la Etapa A" con la tabla de `model_comparison.md` y la decisión.
2. Commit, etiqueta y subida:
   ```
   git add .
   git commit -m "docs: add stage A results to README"
   git tag etapa-a-completa
   git push
   git push --tags
   ```

**Qué se lleva a la Etapa B (guárdalo, lo vamos a usar):**
1. Las 15 definiciones de features de A7: se replican en SQL en dbt y deben dar los mismos resultados.
2. El diccionario de renombrado (`COLUMN_NAME_MAP` y `TYPE_VALUE_MAP`): se replica en la capa `STAGING`.
3. La regla de partición por tiempo (70/15/15 por proporción de filas).
4. `MIN_PRECISION = 0.80` y la forma de elegir umbral.
5. Las métricas de este paso: al reentrenar con datos de Snowflake deben ser comparables.
6. El experimento `fraud_detection` de MLflow y el modelo `fraud_model`.

---

## Tiempos orientativos

Dependen de tu computadora; sirven para saber si algo está tardando demasiado.

| Paso | Tiempo aproximado |
|---|---|
| A2 (instalar) | 5 a 15 minutos |
| A5 (cargar y guardar) | 1 a 3 minutos |
| A7 (features) | 1 a 2 minutos |
| A12 (regresión logística) | Algunos minutos |
| A13 (XGBoost) | Varios minutos |
| A14 (autoencoder) | 2 a 15 minutos |

---

## Problemas frecuentes

| Síntoma | Causa probable | Solución |
|---|---|---|
| `ModuleNotFoundError: No module named 'src'` | Se ejecutó desde otra carpeta o sin `-m` | Entra a la raíz del proyecto y usa `python -m src.ml.nombre` |
| `ModuleNotFoundError` de una librería | El entorno virtual no está activo | Actívalo (A1, punto 7) |
| PowerShell no deja activar el entorno | Política de ejecución | `Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned` y reintentar |
| `MemoryError` al cargar | Poca RAM libre | Cierra programas; si persiste, no hay RAM suficiente para esta etapa |
| XGBoost menciona `libomp` (macOS) | Falta una librería del sistema | `brew install libomp` |
| `TypeError` por `early_stopping_rounds` | XGBoost anterior a 2.0 | `pip install --upgrade xgboost` |
| `mlflow ui` dice que el puerto está en uso | Otro proceso usa el 5000 | Agrega `--port 5001` y abre esa dirección |
| El notebook no encuentra `src` | Falta la primera celda de A6 | Copia exactamente la celda del punto 3 |
| `FileNotFoundError` de un Parquet | Se saltó un paso anterior | Vuelve al paso que genera ese archivo |
| La validación de A5 falla | Archivo distinto del original | Descarga de nuevo el CSV (A3) |

---

## Mapa final de archivos de la Etapa A

```
src/ml/
├── config.py            # A4
├── load_and_rename.py   # A5
├── build_features.py    # A7
├── split_data.py        # A8
├── metrics.py           # A9
├── model_info.py        # A9
├── scaling.py           # A11
├── train_baseline.py    # A12
├── train_xgboost.py     # A13
├── train_autoencoder.py # A14
├── compare_models.py    # A15
└── register_champion.py # A16
tests/
├── test_build_features.py  # A7
├── test_metrics.py         # A9
└── test_scaling.py         # A11
notebooks/01_exploration.ipynb   # A6
reports/decision_etapa_A.md      # A10 y A16
```
