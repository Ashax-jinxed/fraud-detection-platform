# 00 - Dataset: PaySim

## Qué es
Dataset **sintético** de transacciones de dinero móvil, generado a partir de registros reales
de una empresa africana. Está en Kaggle como `ealaxi/paysim1`.

## Rol en el proyecto
Es la fuente de datos. Un producer de Python lee el CSV y lo envía a Kafka fila por fila
para simular transacciones llegando en tiempo real.

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| Kaggle Credit Card Fraud (ULB) | Muy conocido, pequeño (284,807 filas) | Columnas anonimizadas con PCA (V1 a V28) | La IA no podría explicar por qué algo es sospechoso |
| IEEE-CIS Fraud Detection | Datos más reales | Más de 400 columnas, muchos nulos, difícil para empezar | Demasiado ruido para un proyecto junior |
| **PaySim** | Columnas legibles, ~6.3 millones de filas, variable de tiempo | Es sintético | **Elegido** |

## Por qué elegimos PaySim
- Las columnas tienen significado (tipo, monto, saldos), así que el agente de IA puede explicar
  una alerta con datos concretos.
- Tiene suficiente volumen para justificar Kafka y Spark.
- Tiene una columna de tiempo (`step`) que permite simular un flujo y separar entrenamiento y
  prueba por tiempo.

## Columnas que más nos interesan

| Columna | Qué significa | Nota |
|---|---|---|
| `step` | Hora simulada (1 paso = 1 hora, unas 743 horas) | La usaremos para dividir train/test por tiempo |
| `type` | CASH_IN, CASH_OUT, DEBIT, PAYMENT, TRANSFER | El fraude solo ocurre en TRANSFER y CASH_OUT |
| `amount` | Monto de la transacción | |
| `nameOrig` / `nameDest` | Cuenta de origen y destino | |
| `oldbalanceOrg` / `newbalanceOrig` | Saldo del origen antes y después | Base para crear features de saldo |
| `oldbalanceDest` / `newbalanceDest` | Saldo del destino antes y después | |
| `isFraud` | 1 si es fraude (etiqueta real) | Muy desbalanceada: ~0.13% de fraudes |
| `isFlaggedFraud` | Regla simple del dataset original | La ignoramos como feature |

## Decisiones importantes
- **Desbalance de clases:** casi todo es normal, así que no usaremos *accuracy*. Usaremos
  precision, recall y PR-AUC (ver documento 10).
- **División por tiempo, no aleatoria:** entrenamos con las primeras horas y probamos con las
  últimas, como pasaría en la vida real.

## Documentación
- Dataset en Kaggle: https://www.kaggle.com/datasets/ealaxi/paysim1
- Paper original de PaySim (E. Lopez-Rojas et al.), citado en la página del dataset.

## Nombres en español (se aplican en el paso A5 de la Etapa A)

| Original | Español |
|---|---|
| `step` | `hora_simulada` |
| `type` | `tipo` |
| `amount` | `monto` |
| `nameOrig` | `cuenta_origen` |
| `oldbalanceOrg` | `saldo_inicial_origen` |
| `newbalanceOrig` | `saldo_final_origen` |
| `nameDest` | `cuenta_destino` |
| `oldbalanceDest` | `saldo_inicial_destino` |
| `newbalanceDest` | `saldo_final_destino` |
| `isFraud` | `es_fraude` |
| `isFlaggedFraud` | `es_marcado_por_regla` |

Valores de `tipo`: `CASH_IN` pasa a `deposito`, `CASH_OUT` a `retiro`, `DEBIT` a `debito`,
`PAYMENT` a `pago` y `TRANSFER` a `transferencia`.

El dataset no trae un índice útil, así que se crea la columna `id_transaccion` (1, 2, 3, ...).
Sin tildes ni ñ para evitar problemas con Parquet, Snowflake y dbt.

## Dónde se usa
Etapa A (análisis y modelo) y Etapa B (ETL).
