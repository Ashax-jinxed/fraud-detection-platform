# 08 - Docker y Kubernetes (contenedores)

## Qué es
- **Docker:** empaqueta una aplicación con todo lo que necesita en una imagen que corre igual en cualquier máquina.
- **Kubernetes:** administra muchos contenedores (los reinicia, los escala, les reparte tráfico).

## Rol en el proyecto
- Docker Compose levanta Kafka, Postgres (pgvector), Airflow, Prometheus y Grafana en local.
- Dockerfiles empaquetan la API y el dashboard.
- Kubernetes (en local con `kind`) despliega la API y el dashboard como se haría en producción.

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| Solo Docker Compose | Simple | No demuestra orquestación de contenedores | Lo usamos, pero no solo |
| Podman | Sin daemon | Menos ofertas | |
| AWS ECS | Simple en AWS | Atado a AWS | |
| Nomad | Más simple que K8s | Muy pocas ofertas | |
| **Docker + Kubernetes** | Kubernetes aparece en 32.6% de las ofertas | Curva de aprendizaje | **Elegido** |

Para correr Kubernetes en local: `kind` (Kubernetes dentro de Docker, liviano) frente a `minikube` o `k3s`. Elegimos `kind` porque también se usa para probar en CI.

## Por qué elegimos esta combinación
Docker es indispensable; Kubernetes diferencia un portafolio. Los conceptos básicos
(Deployment, Service, Secret) se preguntan mucho en entrevistas.

## Beneficios que obtendremos
- "Funciona en mi máquina" pasa a ser "funciona en cualquier máquina".
- Experiencia con despliegue, secretos y chequeos de salud.

## Parámetros y elementos que más nos interesan

### Dockerfile

| Instrucción | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| `FROM` | Imagen base | `python:3.11-slim` | `python:3.12`, `-alpine` (da problemas con librerías científicas) |
| `COPY requirements.txt` antes del código | Aprovecha la caché de capas | Siempre primero | Copiar todo junto (reinstala dependencias en cada cambio) |
| `RUN pip install --no-cache-dir` | Instala sin guardar caché | Imagen más liviana | Sin la bandera (imagen más pesada) |
| `CMD` | Comando de arranque | `uvicorn ...` | `ENTRYPOINT` |

### Docker Compose

| Elemento | Qué hace | Valor elegido |
|---|---|---|
| `services` | Contenedores a levantar | Uno por herramienta |
| `env_file` | Variables desde `.env` | Siempre, nunca secretos en el YAML |
| `volumes` | Datos persistentes | Para Postgres y Grafana |
| `depends_on` + `healthcheck` | Orden y chequeo de arranque | Para Airflow y Postgres |
| `ports` | Puertos expuestos | Solo los necesarios |

### Kubernetes

| Objeto | Qué hace | Nota |
|---|---|---|
| `Deployment` | Mantiene N réplicas de un contenedor | `replicas: 2` para mostrar alta disponibilidad |
| `Service` | Da una dirección estable | `ClusterIP` con `port-forward`, o `NodePort` |
| `Secret` | Guarda credenciales | Se crea con `kubectl create secret`, no se versiona |
| `ConfigMap` | Configuración no sensible | Para parámetros |
| `resources.requests/limits` | CPU y memoria | Valores bajos para correr en local |
| `readinessProbe` | Dice cuándo está lista la app | Apunta a `/health` de FastAPI |

Comandos básicos: `kubectl apply -f`, `kubectl get pods`, `kubectl logs`, `kubectl describe`.

## Documentación
- Docker: https://docs.docker.com/
- Docker Compose: https://docs.docker.com/compose/
- Kubernetes: https://kubernetes.io/docs/home/
- kind: https://kind.sigs.k8s.io/

## Dónde se usa
Etapa B (Compose) y Etapa E (Kubernetes).
