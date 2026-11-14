# 16 - FastAPI (servicio de predicción)

## Qué es
Framework web de Python para crear APIs rápidas, con validación automática y documentación
interactiva generada sola.

## Rol en el proyecto
Expone el modelo y el agente:
- `POST /predict`: recibe una transacción y devuelve la probabilidad de fraude.
- `POST /explain`: devuelve la explicación del agente.
- `GET /health`: indica si el servicio está vivo (lo usa Kubernetes).
- `GET /metrics`: métricas para Prometheus.

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| Flask | Muy simple y conocido | Sin validación ni documentación automática | Hay que añadir más piezas |
| Django REST | Muy completo | Pesado para una API pequeña | |
| Litestar | Moderno | Pocas ofertas | |
| **FastAPI** | Validación con Pydantic, documentación en `/docs`, muy pedido para servir modelos | Menos "baterías incluidas" que Django | **Elegido** |

## Por qué elegimos FastAPI
Es el estándar actual para servir modelos de ML, y la validación de datos evita errores silenciosos.

## Beneficios que obtendremos
- Documentación interactiva gratis en `/docs`.
- Errores claros si la entrada es inválida.

## Parámetros y elementos que más nos interesan

| Elemento | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| `FastAPI()` | Crea la aplicación | Con título y versión | - |
| `@app.post("/predict")` | Define un endpoint | Un endpoint por tarea | - |
| `BaseModel` de Pydantic | Valida el cuerpo de la petición | Clase `TransactionRequest` con tipos | Diccionarios sin validar |
| `response_model` | Define la forma de la respuesta | Siempre | Sin definir |
| `status_code` / `HTTPException` | Códigos de error claros | `422` entrada inválida, `503` modelo no cargado | Devolver `200` con texto de error |
| `def` vs `async def` | Tipo de función | `def` normal (más simple; el modelo es de CPU) | `async def` (útil con llamadas de red concurrentes) |
| Carga del modelo | Cuándo se carga | Una vez al arrancar (`lifespan`) | En cada petición (lento) |
| `uvicorn app:app --host 0.0.0.0 --port 8000` | Servidor | `0.0.0.0` para Docker | `127.0.0.1` (no accesible desde fuera del contenedor) |
| `--reload` | Recarga al cambiar el código | Solo en desarrollo | En producción (nunca) |

## Documentación
- General: https://fastapi.tiangolo.com/
- Tutorial: https://fastapi.tiangolo.com/tutorial/
- Pydantic: https://docs.pydantic.dev/
- Uvicorn: https://www.uvicorn.org/

## Dónde se usa
Etapa C.
