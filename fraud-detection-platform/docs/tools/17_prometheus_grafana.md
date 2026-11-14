# 17 - Prometheus y Grafana (observabilidad)

## Qué es
- **Prometheus:** base de datos de métricas que consulta (scrape) tus servicios cada cierto tiempo.
- **Grafana:** herramienta para dibujar dashboards con esas métricas.

## Rol en el proyecto
Miden la salud de la API: cuántas predicciones se hacen, cuántas se marcan como fraude y cuánto
tarda cada una.

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| Solo logs | Cero configuración | No se ve la tendencia | Insuficiente |
| ELK (Elasticsearch, Logstash, Kibana) | Muy potente para logs | Pesado de correr | Es más de logs que de métricas |
| Datadog | Muy completo | De pago | |
| AWS CloudWatch | Nativo de AWS | Atado a AWS | |
| **Prometheus + Grafana** | Estándar abierto y gratuito, se usa con Kubernetes | Hay que aprender PromQL | **Elegido** |

## Por qué elegimos esta combinación
Observabilidad y monitoreo son habilidades que suben el valor de un perfil en ofertas de Data
Engineering, y este par es el estándar en Kubernetes.

## Beneficios que obtendremos
- Ver latencia y volumen en un panel.
- Base para alertas futuras.

## Parámetros y elementos que más nos interesan

### En el código (`prometheus-client`)

| Elemento | Qué mide | Uso en el proyecto |
|---|---|---|
| `Counter` | Valor que solo sube | `predictions_total` con etiqueta del resultado |
| `Gauge` | Valor que sube y baja | Número de alertas abiertas |
| `Histogram` | Distribución de valores | `prediction_latency_seconds` |
| `make_asgi_app()` | Expone `/metrics` en FastAPI | Lo montamos en la API |

### En `prometheus.yml`

| Parámetro | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| `scrape_interval` | Cada cuánto lee las métricas | `15s` | `5s` (más carga), `1m` (menos detalle) |
| `job_name` | Nombre del grupo | `fraud-api` | - |
| `static_configs.targets` | Direcciones a leer | `api:8000` | Descubrimiento automático en Kubernetes |

### En Grafana

| Elemento | Qué hace | Valor elegido |
|---|---|---|
| Data source | Conecta con Prometheus | Se define por archivo de provisioning, no a mano |
| Consulta de ejemplo | Predicciones por segundo | `rate(predictions_total[5m])` |
| Dashboard | Panel guardado | Archivo JSON versionado en `monitoring/` |

## Documentación
- Prometheus: https://prometheus.io/docs/
- Cliente de Python: https://prometheus.github.io/client_python/
- Grafana: https://grafana.com/docs/
- Introducción a PromQL: https://prometheus.io/docs/prometheus/latest/querying/basics/

## Dónde se usa
Etapa D.
