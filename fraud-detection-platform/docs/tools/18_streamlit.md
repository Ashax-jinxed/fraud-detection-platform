# 18 - Streamlit (dashboard)

## Qué es
Librería de Python para crear aplicaciones web de datos escribiendo solo Python.

## Rol en el proyecto
Muestra a una persona no técnica: las transacciones recientes, las alertas de fraude, el
detalle de cada alerta y la explicación generada por el agente.

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| Power BI / Tableau | Muy pedidos en ofertas de ciencia de datos (28% y 25% en un análisis) | Programas de escritorio, difíciles de publicar y de versionar en Git | No encajan con un repositorio de código |
| Gradio | Muy rápido para demos de modelos | Menos pensado para paneles | |
| Dash (Plotly) | Muy flexible | Más código | |
| Grafana | Ya lo usamos | Es para métricas técnicas | Aquí queremos una vista de negocio |
| **Streamlit** | Solo Python, publicable gratis | Menos control del diseño | **Elegido** |

## Por qué elegimos Streamlit
Es la forma más rápida de mostrar el resultado final en una URL, y todo queda en Python,
que es lo que el repositorio ya usa.

## Beneficios que obtendremos
- Un entregable visual que cualquier reclutador abre en segundos.
- Sin código de frontend.

## Funciones y parámetros que más nos interesan

| Función | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| `st.set_page_config` | Título y diseño de la página | `layout="wide"` | `centered` (más angosto) |
| `st.metric` | Indicador grande | Alertas del día, tasa de fraude | - |
| `st.dataframe` | Tabla interactiva | Para listar alertas | `st.table` (estática) |
| `st.line_chart` / `st.bar_chart` | Gráficos rápidos | Transacciones por hora | Plotly (más control, más código) |
| `st.sidebar` | Filtros laterales | Tipo de transacción, rango de monto | - |
| `st.cache_data(ttl=60)` | Guarda resultados de consultas | `ttl=60` segundos | Sin caché (consulta Snowflake en cada clic y gasta créditos) |
| `st.cache_resource` | Guarda conexiones | Para la conexión a Snowflake | Reconectar en cada ejecución |
| `st.session_state` | Memoria entre interacciones | Para la alerta seleccionada | Variables globales (se pierden al recargar) |
| `st.button` + `st.spinner` | Acción y mensaje de espera | Botón "Explain this alert" | Explicar todas las alertas automáticamente (costaría mucho en la API) |
| `st.secrets` | Secretos de la app | En la nube de Streamlit | `.env` en local |

Despliegue: Streamlit Community Cloud permite publicar la app gratis, pero no puede correr Kafka,
Spark ni Airflow. Para la demo pública se usará una muestra de datos y/o un video del flujo
completo.

## Documentación
- General: https://docs.streamlit.io/
- Caché: https://docs.streamlit.io/develop/concepts/architecture/caching
- Publicación: https://docs.streamlit.io/deploy/streamlit-community-cloud

## Dónde se usa
Etapa D.
