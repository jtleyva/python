# Infrastructure as Code (IaC)

## 1. Explicación técnica
A nivel Senior/Staff, no se entra a la consola web de AWS (ClickOps) para crear recursos en producción. Hacerlo es lento, no reproducible, propenso al error humano y no auditable. **Infrastructure as Code (IaC)** significa gestionar la infraestructura usando archivos de configuración (código) que pueden ser versionados (Git), revisados (Pull Requests) y desplegados automáticamente (CI/CD).

Herramientas principales:
- **Terraform:** Declarativo, cloud-agnostic, usa HCL (HashiCorp Configuration Language). Mantiene un archivo de `state` (estado) para saber qué recursos ya están creados y calcular el `diff` (diferencia) antes de aplicar cambios.
- **AWS CloudFormation / SAM:** Declarativo, nativo de AWS (JSON/YAML). 
- **AWS CDK (Cloud Development Kit):** Imperativo. Permite escribir infraestructura usando lenguajes reales (Python, TypeScript) que luego sintetiza (compila) en plantillas de CloudFormation.
- **Pulumi:** Similar a CDK, pero usa Terraform como motor backend.

## 2. Uso en producción
Un equipo que usa Terraform guarda su estado (`terraform.tfstate`) en un bucket de S3 centralizado con versionado activado. Para evitar que dos ingenieros hagan un `terraform apply` al mismo tiempo y corrompan la infraestructura, usan una tabla de DynamoDB como mecanismo de "bloqueo" (State Locking). Toda la infraestructura (VPCs, RDS, EKS, Lambdas) se divide en módulos reutilizables. 

## 3. Ejemplo práctico (Terraform)

Definición básica de un módulo de Terraform para provisionar una AWS Lambda en Python con sus roles IAM.

```hcl
# main.tf

# 1. Proveedor
provider "aws" {
  region = "us-east-1"
  default_tags {
    tags = {
      Environment = "production"
      Project     = "payment-api"
    }
  }
}

# 2. IAM Role de Asunción para Lambda
resource "aws_iam_role" "lambda_exec_role" {
  name = "payment_api_lambda_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

# 3. Política (Permitir a Lambda escribir Logs en CloudWatch)
resource "aws_iam_role_policy_attachment" "lambda_basic_execution" {
  role       = aws_iam_role.lambda_exec_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# 4. Empaquetar el código Python localmente
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = "${path.module}/src"
  output_path = "${path.module}/payment_lambda.zip"
}

# 5. Desplegar el recurso Lambda
resource "aws_lambda_function" "payment_processor" {
  function_name    = "payment-processor-lambda"
  filename         = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256
  
  role    = aws_iam_role.lambda_exec_role.arn
  handler = "main.handler"
  runtime = "python3.11"
  timeout = 10
  memory_size = 256

  environment {
    variables = {
      STRIPE_ENV = "production"
    }
  }
}
```

## 4. Diagrama (texto)

Flujo de trabajo de Terraform en CI/CD:

[ Developer ] ── (Git Push) ──► [ GitHub Repository ]
                                        │
[ CI/CD Pipeline ] ◄────────────────────┘
         │
         ├── 1. `terraform fmt` y `tflint` (Chequeo de sintaxis)
         ├── 2. `tfsec` o `checkov` (Escaneo de vulnerabilidades IAM)
         ├── 3. `terraform plan` ──► Muestra en el PR lo que se va a crear/borrar
         │
[ Manager aprueba el PR y se mergea a Main ]
         │
[ CI/CD Production ]
         └── 4. `terraform apply` ──► [ AWS API ] ──► Crea Lambda / RDS / VPC
                                        │
                        (Actualiza state en S3 + DynamoDB lock)

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Qué diferencia hay entre un enfoque *Declarativo* (Terraform) y uno *Imperativo* (Boto3/Scripts de Python) para provisionar infraestructura?
- **Pregunta 2 (Práctica):** En un proyecto de Terraform, alguien borró accidentalmente una instancia EC2 directamente desde la consola de AWS. Alguien más ejecuta `terraform plan` el día siguiente. ¿Qué hará Terraform?
- **Pregunta 3 (Arquitectura):** Tienes un archivo de estado de Terraform (`.tfstate`) en tu laptop. Contiene las contraseñas temporales generadas para la base de datos RDS. ¿Por qué es esto peligroso y cómo lo solucionas a nivel empresarial?

## 6. Respuestas esperadas
- **Respuesta 1:** El enfoque Imperativo (Scripts) describe *cómo* alcanzar el estado final ("Crea un bucket S3, si ya existe, ignóralo..."). Es complejo de mantener y gestionar errores. El enfoque Declarativo (Terraform) describe *cuál* es el estado final deseado ("Quiero un bucket S3"). El motor subyacente se encarga de determinar *cómo* hacerlo, leyendo el estado actual y calculando las deltas necesarias para llegar al estado deseado (creando, modificando o destruyendo recursos automáticamente).
- **Respuesta 2:** Terraform detectará que hay una discrepancia entre su archivo de estado (donde la EC2 existe) y la realidad en AWS (donde no existe, Drift). El `terraform plan` mostrará que Terraform reconstruirá y aprovisionará una nueva instancia EC2 desde cero para restaurar la infraestructura a su estado declarativo deseado.
- **Respuesta 3:** Es crítico porque el archivo `.tfstate` se guarda en texto plano localmente y a menudo se commitea por accidente en Git, exponiendo secretos críticos (contraseñas de BD, tokens de acceso). La solución empresarial es usar **Remote State Storage** (ej. AWS S3) con encriptación (KMS) activada, versionado de S3 y controles estrictos de IAM, además de bloquear subidas de `.tfstate` con `.gitignore`.

## 7. Errores comunes
- **Drift de Configuración (ClickOps vs IaC):** El equipo usa Terraform, pero cuando hay un problema urgente (Incidente), un ingeniero entra a la consola de AWS y cambia manualmente una regla de Security Group para arreglarlo. En el siguiente despliegue, Terraform sobrescribirá ese cambio devolviéndolo al estado del código, volviendo a romper la aplicación. Los cambios SIEMPRE deben fluir a través del código.
- **Mono-repositorio de Infra (Un solo archivo de estado masivo):** Tener toda la empresa (VPC, IAM, RDS, Lambdas, ECS) en un solo archivo `state`. Un `terraform plan` tardará 15 minutos, y el riesgo del "Blast Radius" (radio de impacto) es total: un error en un módulo no relacionado podría intentar destruir la base de datos productiva. *Solución: Dividir en Workspaces/carpetas por Dominio o Capa de Ciclo de Vida (Network/Data/App).*

## 8. Buenas prácticas
- **Separación de Entornos:** Usar Workspaces de Terraform o directorios completamente separados para mantener los estados de `dev`, `staging` y `production` aislados por completo.
- **Data Sources:** En lugar de hardcodear los IDs de las Subredes (`subnet-1234abcd`), usar `data "aws_subnet"` para buscarlas dinámicamente basándose en sus etiquetas (Tags). Esto hace que los módulos sean portátiles entre cuentas de AWS.

## 9. Trade-offs y decisiones técnicas
**Terraform vs AWS CDK (Python)**
- *Terraform (HCL):* Estándar absoluto de la industria. Ecosistema de proveedores (Providers) masivo (AWS, GitHub, Datadog, PagerDuty). Trade-off: HCL no es un lenguaje de programación completo, los bucles (loops) y la lógica condicional compleja son engorrosos. Tienes que aprender un lenguaje nuevo.
- *AWS CDK:* Escribes en Python estándar. Puedes usar bucles `for`, clases y herencia, encapsulando las mejores prácticas (ej. `SecureVpc()` class) de forma nativa. Genera plantillas limpias de CloudFormation. Trade-off: Dependencia extrema de CloudFormation (sus límites y tiempos lentos de despliegue/rollback), curva de aprendizaje para entender "Constructs", y solo funciona para AWS.
