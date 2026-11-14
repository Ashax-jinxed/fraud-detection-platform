# 14 - pgvector (base de datos vectorial)

## Qué es
Extensión de PostgreSQL que permite guardar *embeddings* (vectores numéricos que representan el
significado de un texto) y buscar los más parecidos.

## Rol en el proyecto
Guarda una pequeña base de conocimiento de tipologías de fraude (por ejemplo, "vaciado de cuenta:
transferencia seguida de retiro total"). El agente busca las más parecidas a una alerta para
dar contexto a la explicación.

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| Pinecone | Administrado, muy conocido | Servicio externo de pago | Menos control, costo |
| Chroma | Muy fácil | Menos usado en producción | |
| FAISS | Muy rápido | Es una librería, no una base de datos | |
| Weaviate / Qdrant | Potentes y dedicados | Otro servicio más que operar | Alternativas válidas |
| **pgvector** | Usa PostgreSQL y SQL, un servicio menos | Menos escalable que las dedicadas | **Elegido** |

## Por qué elegimos pgvector
Reutiliza SQL y PostgreSQL, que ya conocemos, y demuestra RAG sin sumar una tecnología nueva.
Las ofertas piden bases vectoriales de forma creciente (Pinecone, Weaviate, Qdrant y pgvector aparecen entre las nombradas).

## Beneficios que obtendremos
- Búsqueda semántica con una consulta SQL.
- Se levanta con una imagen Docker oficial.

## Modelo de embeddings
Claude no ofrece embeddings, así que usamos `sentence-transformers` con el modelo multilingüe
`paraphrase-multilingual-MiniLM-L12-v2` (vectores de 384 dimensiones), porque nuestra base de conocimiento está
en español. Alternativas: `all-MiniLM-L6-v2` (solo inglés, más liviano) o modelos de OpenAI o Voyage (de pago).

## Parámetros que más nos interesan

| Elemento | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| `CREATE EXTENSION vector` | Activa pgvector | Una vez por base | - |
| Tipo `VECTOR(384)` | Columna de vectores | 384, igual al modelo | La dimensión debe coincidir siempre |
| Operador `<=>` | Distancia coseno | Para texto | `<->` (euclidiana), `<#>` (producto interno) |
| `ORDER BY embedding <=> query LIMIT k` | Busca los más cercanos | `k = 3` | `k = 1` (poco contexto), `k = 10` (ruido) |
| Índice HNSW | Acelera la búsqueda | `m = 16`, `ef_construction = 64` | `IVFFlat` (más rápido de construir) |
| Sin índice | Búsqueda exacta | Con pocos cientos de filas no hace falta índice | - |
| Imagen Docker | Servidor | `pgvector/pgvector` (PostgreSQL 16) | Instalar la extensión a mano |

## Documentación
- Repositorio y guía: https://github.com/pgvector/pgvector
- Cliente Python: https://github.com/pgvector/pgvector-python
- Sentence Transformers: https://www.sbert.net/

## Dónde se usa
Etapa C.
