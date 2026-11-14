# 09 - GitHub Actions (CI/CD)

## Qué es
Sistema de automatización integrado en GitHub. Ejecuta tareas cada vez que se sube código.

## Rol en el proyecto
En cada `push` y `pull request` ejecuta el linter (`ruff`) y las pruebas (`pytest`). Más
adelante construye la imagen Docker.

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| Jenkins | Muy usado en empresas grandes | Hay que instalar y mantener un servidor | Demasiado trabajo de mantenimiento |
| GitLab CI | Muy bueno | Nuestro repo estará en GitHub | |
| CircleCI | Fácil | Servicio externo adicional | |
| **GitHub Actions** | Gratis en repos públicos, integrado, muy usado | YAML con detalles | **Elegido** |

## Por qué elegimos GitHub Actions
El portafolio vive en GitHub, y un check verde en el repo es lo primero que ve un reclutador.

## Beneficios que obtendremos
- Calidad automática en cada cambio.
- Un badge de estado para el README.

## Parámetros que más nos interesan

| Parámetro | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| `on` | Cuándo se dispara | `push` y `pull_request` | `schedule`, `workflow_dispatch` (manual) |
| `runs-on` | Sistema del ejecutor | `ubuntu-latest` | `windows-latest`, `macos-latest` (más lentos) |
| `actions/checkout` | Descarga el código | Siempre | - |
| `actions/setup-python` + `python-version` | Instala Python | `"3.11"` | Matriz de versiones (más lento) |
| `cache: pip` | Guarda dependencias | Activado | Sin caché (más lento) |
| `run` | Ejecuta comandos | `ruff check .` y `pytest` | `flake8`, `pylint` |
| `secrets.NAME` | Secretos del repositorio | Para claves en CI | Nunca escribirlos en el YAML |

Importante: la CI no se conecta a Snowflake ni a Claude; las pruebas usan datos pequeños y
simulaciones (mocks), para no gastar créditos.

## Documentación
- General: https://docs.github.com/en/actions
- Sintaxis de workflows: https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions
- Ruff: https://docs.astral.sh/ruff/

## Dónde se usa
Etapa E (se puede añadir desde la Etapa A).
