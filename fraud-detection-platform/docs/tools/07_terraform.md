# 07 - Terraform (infraestructura como código)

## Qué es
Herramienta para describir infraestructura en archivos (`.tf`) y crearla, cambiarla o
destruirla con comandos.

## Rol en el proyecto
Crea los recursos de AWS (bucket de S3, usuario y políticas de IAM, repositorio de ECR) de forma
repetible, sin hacer clics en la consola.

## Opciones evaluadas

| Opción | Ventajas | Desventajas | Por qué no |
|---|---|---|---|
| Consola de AWS (a mano) | Rápido para probar | No se puede repetir ni revisar | No es profesional |
| CloudFormation | Nativo de AWS | Solo AWS, YAML largo | Terraform sirve en cualquier nube |
| AWS CDK | Usa lenguajes de programación | Atado a AWS | |
| Pulumi | Usa Python | Menos ofertas | |
| OpenTofu | Fork abierto de Terraform, misma sintaxis | Menos ofertas | Lo que aprendamos aplica igual |
| **Terraform** | 18.1% de las ofertas, multi-nube | Estado que cuidar | **Elegido** |

## Por qué elegimos Terraform
Es el estándar de IaC en ofertas de empleo y funciona con AWS, y también con Snowflake.

## Beneficios que obtendremos
- Con `terraform apply` creamos todo y con `terraform destroy` lo borramos (evita costos olvidados).
- La infraestructura queda versionada en Git.

## Comandos y bloques que más nos interesan

| Elemento | Qué hace | Valor elegido | Otras opciones |
|---|---|---|---|
| `provider "aws"` | Configura el proveedor | Con `region = var.aws_region` | Región fija en el código |
| `resource` | Define un recurso | `aws_s3_bucket`, `aws_ecr_repository`, `aws_iam_user` | - |
| `variable` | Valores configurables | Región, nombre del proyecto | Valores fijos |
| `output` | Muestra datos al final | Nombre del bucket | - |
| `terraform init` | Descarga proveedores | Primer comando | - |
| `terraform plan` | Muestra qué cambiaría | Siempre antes de aplicar | Aplicar sin revisar (riesgoso) |
| `terraform apply` / `destroy` | Crea / borra | Con confirmación manual | `-auto-approve` (solo en CI) |
| `terraform fmt` / `validate` | Formato y validación | Antes de cada commit | - |
| Estado (*state*) | Registro de lo creado | Local al inicio | Backend remoto en S3 |

El archivo de estado nunca se sube a Git porque puede contener datos sensibles.

## Documentación
- General: https://developer.hashicorp.com/terraform/docs
- Proveedor AWS: https://registry.terraform.io/providers/hashicorp/aws/latest/docs
- Comandos de la CLI: https://developer.hashicorp.com/terraform/cli

## Dónde se usa
Etapa E.
