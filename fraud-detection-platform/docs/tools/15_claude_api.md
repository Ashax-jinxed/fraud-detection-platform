# 15 - Claude API (LLM que explica las alertas)

## Qué es
API de Anthropic para usar los modelos Claude desde Python.

## Rol en el proyecto
Recibe los datos de una transacción marcada como sospechosa, la probabilidad del modelo y los
casos parecidos de pgvector, y redacta una explicación corta en lenguaje simple.

**Regla de diseño:** el LLM no decide si hay fraude. Solo explica lo que el modelo ya decidió,
usando únicamente los datos que le entregamos.

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| OpenAI (GPT) | Muy conocido | Otro proveedor de pago | Alternativa válida |
| Google Gemini | Buena relación calidad/precio | Menos usado en el stack del proyecto | |
| Ollama (modelo local) | Gratis y privado | Calidad menor y necesita buena PC | Útil como plan B sin costo |
| Mistral | Opción europea | Menos ofertas | |
| **Claude API** | Explicaciones claras, sigue bien instrucciones, aparece en ofertas de AI | De pago por uso | **Elegido** |

## Por qué elegimos Claude
Sigue bien reglas como "usa solo estos datos" y "no acuses a nadie", algo clave en una explicación de fraude.

## Beneficios que obtendremos
- Explicaciones útiles con pocas líneas de código.
- Control del tono y del límite de longitud con el *system prompt*.

## Parámetros que más nos interesan

| Parámetro | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| `model` | Qué modelo usar | Un modelo de la línea Sonnet (equilibrio calidad/costo) | Haiku (más barato y rápido), Opus (más potente y caro) |
| `max_tokens` | Largo máximo de la respuesta | `300` | `100` (muy corto), `1000` (innecesario) |
| `temperature` | Aleatoriedad | `0` | `0.7` (más variado, menos consistente) |
| `system` | Reglas permanentes | Solo usar los datos dados, tono neutral, no afirmar fraude con certeza | Sin system prompt (respuestas impredecibles) |
| `messages` | Conversación | Un mensaje de usuario con los datos | Historial largo (no necesario) |
| Clave de API | Autenticación | Variable `ANTHROPIC_API_KEY` en `.env` | Escribirla en el código (nunca) |

Los nombres de modelo cambian con el tiempo. Antes de codificar se consulta la lista vigente
en la documentación.

## Costos y límites
El agente llama a Claude solo cuando se le pide una explicación, no por cada transacción.

## Documentación
- General: https://docs.claude.com/
- Referencia de la API de mensajes: https://docs.claude.com/en/api/messages
- Lista de modelos: https://docs.claude.com/en/docs/about-claude/models

## Dónde se usa
Etapa C.
