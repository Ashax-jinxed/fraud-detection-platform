# 06 - AWS (nube)

## Qué es
Plataforma de servicios en la nube de Amazon.

## Rol en el proyecto
- **S3:** guarda los artefactos de MLflow (modelos), un respaldo de datos crudos y archivos del proyecto.
- **IAM:** usuarios y permisos mínimos.
- **ECR:** guarda las imágenes Docker de la API y el dashboard.
- **EC2 o EKS (se decide en la Etapa E):** donde se despliega el servicio.

La cuenta de Snowflake también se crea sobre AWS para que todo quede en la misma región.

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| Azure | Fuerte en empresas | Menos ofertas que AWS en el análisis | |
| GCP | Buen ecosistema de datos | Menos ofertas | |
| Solo local | Gratis | No demuestra habilidades de nube | |
| **AWS** | 47.8% de las ofertas de Data Engineering mencionan AWS | Fácil gastar de más si no se vigila | **Elegido** |

## Por qué elegimos AWS
Es la nube más nombrada en ofertas, y es la que más se repite entre los tres perfiles.

## Beneficios que obtendremos
- Experiencia con S3, IAM y contenedores en la nube.
- Un entorno cercano a lo que se usa en empresas.

## Parámetros y decisiones que más nos interesan

| Parámetro | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| Región | Dónde viven los recursos | `us-east-1` (más servicios y precios bajos) | `sa-east-1` (São Paulo, más cerca de Bolivia pero más caro) |
| S3 *versioning* | Guarda versiones de cada archivo | Activado en el bucket de modelos | Desactivado (se puede perder un modelo al sobrescribir) |
| S3 *block public access* | Impide acceso público | Activado | Desactivado (riesgo de filtrar datos) |
| IAM | Permisos | Usuario propio con política mínima | Usar el usuario root (nunca) |
| Credenciales | Cómo acceden los scripts | Variables de entorno desde `.env` | Roles de IAM (mejor en producción) |
| AWS Budgets | Alerta de gasto | Presupuesto mensual bajo con aviso por correo | Sin alerta |

## Documentación
- General: https://docs.aws.amazon.com/
- S3: https://docs.aws.amazon.com/s3/
- IAM: https://docs.aws.amazon.com/iam/
- Budgets: https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html
- SDK para Python (boto3): https://boto3.amazonaws.com/v1/documentation/api/latest/index.html

## Dónde se usa
Etapa E (despliegue y, opcionalmente, artefactos de MLflow en S3).
