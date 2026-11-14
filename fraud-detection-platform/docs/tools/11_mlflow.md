# 11 - MLflow (experimentos y registro de modelos)

## Qué es
Plataforma de código abierto para registrar experimentos de machine learning y administrar
versiones de modelos.

## Rol en el proyecto
Guarda parámetros, métricas y modelos de cada entrenamiento. El modelo ganador se registra y la API lo carga desde el registro.

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| Weights & Biases | Interfaz muy buena | Servicio externo comercial | |
| DVC | Versiona datos y modelos con Git | Es más de versionado que de experimentos | Complemento posible |
| Neptune / Comet | Completos | Servicios comerciales, menos ofertas | |
| **MLflow** | Código abierto, casi 30% de las ofertas de MLOps | Interfaz básica | **Elegido** |

## Por qué elegimos MLflow
Es la herramienta de MLOps más citada en ofertas y funciona con XGBoost y PyTorch.

## Beneficios que obtendremos
- Comparar corridas lado a lado.
- Reproducir un entrenamiento exacto.
- Modelos con versión y alias (`champion`) para producción.

## Funciones y parámetros que más nos interesan

| Función / parámetro | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| `mlflow.set_tracking_uri` | Dónde se guarda todo | `sqlite:///mlflow.db` | Servidor con Postgres (más pasos) |
| Artifact store | Dónde van los archivos de modelos | Carpeta local, luego S3 | - |
| `mlflow.set_experiment` | Agrupa corridas | `fraud-detection` | - |
| `mlflow.start_run(run_name=...)` | Inicia una corrida | Nombre descriptivo | Sin nombre (difícil de leer) |
| `mlflow.log_param` / `log_params` | Registra hiperparámetros | Registro explícito | `autolog()` (automático, pero oculta qué se guarda) |
| `mlflow.log_metric(name, value, step)` | Registra métricas | `pr_auc`, `precision`, `recall` | - |
| `mlflow.xgboost.log_model` / `mlflow.pytorch.log_model` | Guarda el modelo | Uno por tipo | Guardar con `pickle` (sin metadatos) |
| `mlflow.register_model` | Registra una versión | Para el ganador | - |
| Alias | Etiqueta una versión | `champion` | Stages (están en desuso) |

Evitamos `autolog()` al inicio a propósito: registrar de forma explícita enseña qué se guarda y por qué.

## Documentación
- General: https://mlflow.org/docs/latest/
- Tracking: https://mlflow.org/docs/latest/ml/tracking/
- Model Registry: https://mlflow.org/docs/latest/ml/model-registry/

(Las rutas internas de la documentación de MLflow cambian entre versiones; si un enlace falla, entra por la página general.)

## Dónde se usa
Etapa A (registro de experimentos) y Etapa C (la API carga el modelo).
