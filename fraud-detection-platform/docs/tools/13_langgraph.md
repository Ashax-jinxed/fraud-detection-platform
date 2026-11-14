# 13 - LangGraph (agente)

## Qué es
Librería del ecosistema LangChain para construir flujos de IA como un **grafo de estados**:
nodos (pasos), aristas (orden) y un estado compartido.

## Rol en el proyecto
Orquesta los pasos del agente que explica una alerta:
1. Leer la transacción y la probabilidad de fraude.
2. Buscar casos o tipologías de fraude parecidas en pgvector.
3. Pedir a Claude una explicación usando solo esos datos.

Es un flujo lineal y corto, a propósito, para mantenerlo legible.

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| Llamar a la API directamente | Lo más simple | No muestra frameworks de agentes | Se pierde un punto que piden las ofertas |
| LangChain clásico (chains) | Muy conocido | Menos control del flujo | LangGraph es su evolución para agentes |
| LlamaIndex | Excelente para RAG | Centrado en recuperación | Alternativa válida |
| CrewAI / AutoGen | Multi-agente | Excesivo para este caso | |
| **LangGraph** | Control explícito del flujo, LangChain aparece en 25% de las ofertas de AI Engineer | API que evoluciona | **Elegido** |

## Por qué elegimos LangGraph
Muestra que sabemos modelar un agente como pasos explícitos y depurables.

## Beneficios que obtendremos
- Flujo visible y fácil de explicar.
- Cada nodo se puede probar por separado.

## Conceptos y parámetros que más nos interesan

| Elemento | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| `StateGraph(State)` | Crea el grafo | Con un `TypedDict` | Clases Pydantic |
| `State` | Datos compartidos entre nodos | `transaction`, `fraud_probability`, `similar_cases`, `explanation` | - |
| `add_node(name, function)` | Agrega un paso | 3 nodos | - |
| `add_edge(a, b)` | Orden fijo | Flujo lineal | `add_conditional_edges` (decisiones; no necesarias aquí) |
| `START` / `END` | Inicio y fin | Siempre | - |
| `compile()` / `invoke(state)` | Prepara y ejecuta | Siempre | `stream` (resultados parciales) |
| Checkpointer | Memoria entre ejecuciones | Ninguno | `MemorySaver` (para chat con memoria) |
| `ChatAnthropic` | Conexión con Claude | `temperature=0`, `max_tokens=300` | Otros valores (ver documento 15) |

## Documentación
- LangGraph: https://langchain-ai.github.io/langgraph/
- Integración con Anthropic: https://python.langchain.com/docs/integrations/chat/anthropic/

## Dónde se usa
Etapa C.
