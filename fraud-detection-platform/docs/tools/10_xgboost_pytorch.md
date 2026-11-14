# 10 - XGBoost y PyTorch (modelos)

## Qué es
- **XGBoost:** librería de árboles de decisión con *gradient boosting*, muy fuerte en datos tabulares.
- **PyTorch:** framework de deep learning. Lo usamos para un *autoencoder*, una red que aprende a
  reconstruir transacciones normales; lo que no puede reconstruir bien es sospechoso.

## Rol en el proyecto
- XGBoost: clasificador supervisado que aprende de los casos de fraude conocidos.
- Autoencoder: detector de anomalías no supervisado que puede encontrar patrones nuevos.
- Regresión logística (scikit-learn): modelo base para comparar y demostrar que XGBoost mejora.

## Opciones evaluadas

### Modelo supervisado

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| Regresión logística | Simple, interpretable | Débil con relaciones complejas | La usamos solo como base |
| Random Forest | Robusto | Más lento y suele rendir menos | |
| LightGBM | Rápido, muy bueno | Menos fácil de explicar en entrevistas que XGBoost | Alternativa válida |
| CatBoost | Maneja categorías | Menos ofertas | |
| **XGBoost** | Muy usado en fraude y competencias | Muchos hiperparámetros | **Elegido** |

### Detección de anomalías

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| Isolation Forest | Una línea de scikit-learn | No muestra deep learning | Buena opción simple |
| One-Class SVM | Teoría clara | Lento con millones de filas | |
| **Autoencoder en PyTorch** | Muestra deep learning | Más código | **Elegido** |

## Por qué elegimos esta combinación
PyTorch aparece en 23% de las ofertas de AI Engineer. XGBoost es el estándar para datos
tabulares. Juntos cubren machine learning clásico y deep learning.

## Beneficios que obtendremos
- Comparación honesta entre tres modelos.
- Dos enfoques: aprender de fraudes conocidos y detectar lo raro.

## Decisiones de evaluación (las más importantes del proyecto)
- **No usamos accuracy:** con ~0.13% de fraude, un modelo que dice "nunca hay fraude" tendría 99.87%.
- **Usamos PR-AUC, precision y recall.** Precision: de las alertas, cuántas eran fraude.
  Recall: de los fraudes reales, cuántos detectamos.
- **División por tiempo** usando `hora_simulada`, no aleatoria, para evitar fuga de información.

## Features básicas (se crean primero en pandas en la Etapa A y se replican en dbt en la Etapa B)
`monto`, los cuatro saldos, `hora_del_dia` (`hora_simulada` módulo 24), `error_saldo_origen`, `error_saldo_destino`,
`vacia_cuenta_origen`, `destino_sin_saldo_previo` y el tipo de transacción en forma one-hot (`tipo_deposito`,
`tipo_retiro`, `tipo_debito`, `tipo_pago`, `tipo_transferencia`). Detalle en `docs/etapa_A_validar_modelo.md`.

## Parámetros de XGBoost

| Parámetro | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| `n_estimators` | Número de árboles | `200` | `50` (rápido, puede quedarse corto), `1000` (riesgo de sobreajuste) |
| `max_depth` | Profundidad de cada árbol | `6` | `3` (más simple), `10` (sobreajusta) |
| `learning_rate` | Peso de cada árbol nuevo | `0.1` | `0.01` (más lento y preciso), `0.3` |
| `scale_pos_weight` | Compensa el desbalance | negativos / positivos | `1` (ignora el desbalance) |
| `eval_metric` | Métrica de evaluación | `aucpr` | `auc`, `logloss` |
| `subsample` / `colsample_bytree` | Fracción de filas / columnas por árbol | `0.8` | `1.0` (sin aleatoriedad) |
| `early_stopping_rounds` | Corta si no mejora | `20` | Sin corte |
| `tree_method` | Algoritmo de construcción | `hist` | `exact` (lento con millones de filas) |
| `random_state` | Semilla | `42` | Cualquier número fijo (reproducibilidad) |

## Parámetros del autoencoder (PyTorch)

| Elemento | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| Arquitectura | Capas `Linear` | entrada → 16 → 8 → 16 → entrada | Más capas (más complejo) |
| Activación | Función no lineal | `ReLU` | `Tanh`, `LeakyReLU` |
| `nn.MSELoss()` | Mide el error de reconstrucción | MSE | `L1Loss` |
| Optimizador | Actualiza pesos | `Adam`, `lr=0.001` | `SGD` (más lento de ajustar) |
| `batch_size` | Muestras por paso | `256` | `64` (más lento), `1024` (más memoria) |
| `epochs` | Pasadas por los datos | `20` | Más (sobreajuste) |
| Datos de entrenamiento | Qué ve la red | **Solo transacciones normales** | Todas (aprendería también el fraude) |
| Umbral de anomalía | Error a partir del cual alertamos | Percentil 99 del error en datos normales | Percentil 95 (más alertas) |

## Documentación
- XGBoost: https://xgboost.readthedocs.io/
- Parámetros de XGBoost: https://xgboost.readthedocs.io/en/stable/parameter.html
- PyTorch: https://pytorch.org/docs/stable/index.html
- Tutoriales de PyTorch: https://pytorch.org/tutorials/
- Métricas de scikit-learn: https://scikit-learn.org/stable/modules/model_evaluation.html

## Dónde se usa
Etapa A (y reentrenamiento en la Etapa B).
