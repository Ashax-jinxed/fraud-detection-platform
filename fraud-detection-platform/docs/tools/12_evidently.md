# 12 - Evidently (monitoreo de drift)

## Qué es
Librería de código abierto que compara datos de referencia contra datos nuevos y genera reportes
de *drift* y de calidad.

## Rol en el proyecto
Compara los datos con los que se entrenó el modelo contra las transacciones recientes, y avisa si
cambió la distribución (por ejemplo, montos mucho más altos).

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| Test KS con scipy a mano | Control total | Hay que construir reportes | Más código sin valor extra |
| NannyML | Muy bueno estimando rendimiento sin etiquetas | Menos conocido | Alternativa válida |
| Alibi Detect | Detectores avanzados | Más técnico | |
| Deepchecks | Completo | Más pesado | |
| **Evidently** | Reportes visuales listos, gratis | La API cambia entre versiones | **Elegido** |

## Por qué elegimos Evidently
Genera reportes HTML claros que se pueden mostrar en un portafolio, con pocas líneas de código.

## Beneficios que obtendremos
- Detectar cuándo reentrenar.
- Un entregable visual (el reporte) para el repositorio.

## Conceptos y parámetros que más nos interesan

| Elemento | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| Dataset de referencia | Datos de entrenamiento | Muestra del set de entrenamiento | Ventana móvil de datos recientes |
| Dataset actual | Datos nuevos a comparar | Últimas 24 horas simuladas | Otra ventana |
| `DataDriftPreset` | Reporte de drift por columna | Siempre | Métricas individuales |
| `stattest` | Prueba estadística | `psi` o `wasserstein` para numéricas | `ks` (muy sensible con millones de filas) |
| `drift_share` | Fracción de columnas con drift para declarar drift global | `0.5` | `0.3` (más sensible) |
| Salida | Formato del reporte | HTML y JSON | Solo en pantalla |

Aviso: la API de Evidently cambió en versiones recientes. Fijamos la versión en `requirements.txt`
y seguimos la documentación de esa versión exacta.

## Documentación
- General: https://docs.evidentlyai.com/
- Repositorio: https://github.com/evidentlyai/evidently

## Dónde se usa
Etapa D.
