# 05 - Apache Airflow (orquestación)

## Qué es
Plataforma para programar y monitorear flujos de trabajo definidos en Python (DAGs).

## Rol en el proyecto
Ejecuta en orden y de forma programada: `dbt build`, puntuación de transacciones nuevas con el
modelo y generación del reporte de drift.

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| cron | Muy simple | Sin reintentos, sin dependencias, sin interfaz | No es orquestación |
| Prefect | API moderna, fácil | Menos presente en ofertas | |
| Dagster | Enfoque en datos y activos | Menos ofertas | |
| AWS Step Functions | Nativo de AWS | Atado a AWS, define flujos en JSON | |
| **Apache Airflow** | Estándar de la industria, 58% de ofertas (junto con Prefect) y lidera la orquestación en el otro análisis | Pesado de instalar | **Elegido** |

## Por qué elegimos Airflow
- Es la herramienta de orquestación más nombrada en ofertas de empleo.
- Interfaz web con historial, logs y reintentos.

## Beneficios que obtendremos
- Reintentos automáticos si una tarea falla.
- Dependencias claras entre tareas.
- Historial visible de cada ejecución.

## Parámetros y funciones que más nos interesan

| Parámetro | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| `dag_id` | Nombre único del DAG | `fraud_scoring_pipeline` | - |
| `schedule` | Frecuencia | `*/15 * * * *` (cada 15 min) | `@hourly`, `@daily`, `None` (manual) |
| `start_date` | Desde cuándo existe el DAG | Fecha fija pasada | `datetime.now()` (genera comportamiento confuso) |
| `catchup` | Ejecutar fechas pasadas pendientes | `False` | `True` (ejecuta todo el histórico) |
| `max_active_runs` | Ejecuciones simultáneas del DAG | `1` | Más de 1 (riesgo de solapar escrituras) |
| `retries` | Reintentos por tarea | `2` | `0` (sin reintentos) |
| `retry_delay` | Espera entre reintentos | 2 minutos | Otros valores |
| Operador | Cómo se define una tarea | `BashOperator` para dbt, `@task` para Python | `PythonOperator`, operadores específicos |
| `task_a >> task_b` | Define el orden | Siempre | `set_downstream` |
| Executor | Dónde corren las tareas | `LocalExecutor` | `CeleryExecutor`, `KubernetesExecutor` (más complejos) |

Instalación: con el Docker Compose oficial de Airflow, no con `pip install` suelto.

## Documentación
- General: https://airflow.apache.org/docs/
- Conceptos de DAGs: https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html
- Docker Compose oficial: https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html

## Dónde se usa
Etapa C.
