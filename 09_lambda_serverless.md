# Lambda & Serverless (Python)

## 1. Explicación técnica
AWS Lambda es un servicio de cómputo Serverless que ejecuta código en respuesta a eventos y administra automáticamente los recursos subyacentes. Como desarrollador Python, el código que subes ("Función") es ejecutado dentro de contenedores micro-VM (Firecracker) por AWS.

Conceptos críticos a dominar en Lambda:
- **Cold Starts vs Warm Starts:** Cuando una función se invoca por primera vez (o tras escalar), AWS debe aprovisionar un contenedor, cargar el entorno de ejecución Python y ejecutar el código de inicialización global (imports, DB connections). Esto añade latencia significativa (Cold Start). Invocaciones subsecuentes reutilizan el contenedor (Warm Start).
- **Concurrencia:** Lambda escala por unidad de trabajo (hasta miles de instancias simultáneas). Si configuras *Reserved Concurrency*, evitas que una función acapare toda la cuota de tu cuenta.
- **Stateless Nature:** La micro-VM puede ser destruida en cualquier momento. No debes guardar estado en disco (salvo el `/tmp` volátil de max 10GB) ni en memoria entre peticiones no relacionadas.

## 2. Uso en producción
Bumble usa una arquitectura Serverless impulsada por eventos para moderar millones de fotos. Cuando un usuario sube una imagen a S3 (evento S3 PutObject), esto dispara una Lambda. La Lambda, usando el SDK de Python, invoca Amazon Rekognition para detectar contenido inapropiado. Si es válido, actualiza DynamoDB. Todo el proceso dura ~300ms y pagan exactamente por el tiempo ejecutado, sin servidores ociosos durante la noche.

## 3. Ejemplo práctico (Python)

Un handler optimizado de Lambda para minimizar Cold Starts. La inicialización pesada ocurre FUERA de la función `handler`.

```python
import os
import json
import boto3
import sqlalchemy
from sqlalchemy.orm import sessionmaker

# --- INICIALIZACIÓN GLOBAL (Cold Start Phase) ---
# Todo lo declarado aquí persiste a través de ejecuciones en "Warm Starts"

db_host = os.environ.get("DB_HOST")
secret_name = os.environ.get("SECRET_NAME")

# Boto3 clients se cachean (crear una sesión HTTPS es costoso)
secrets_client = boto3.client('secretsmanager')

def get_db_credentials():
    # Lógica para llamar a AWS Secrets Manager...
    return "user", "password"

user, pw = get_db_credentials()
# La conexión a la DB se realiza UNA sola vez por instancia de contenedor.
engine = sqlalchemy.create_engine(f"postgresql://{user}:{pw}@{db_host}/mydb", pool_size=1)
SessionLocal = sessionmaker(bind=engine)

# --- HANDLER FUNCTION (Warm Start Phase - Execution) ---
def lambda_handler(event, context):
    """
    Punto de entrada de AWS Lambda. El 'event' contiene datos del trigger 
    (API Gateway, S3, SQS).
    """
    try:
        # Extraemos payload simulando que viene de API Gateway
        body = json.loads(event.get("body", "{}"))
        user_id = body.get("user_id")
        
        if not user_id:
            return {"statusCode": 400, "body": json.dumps({"error": "Missing user_id"})}
            
        # Reutilizamos la conexión de DB en cada invocación rápida
        with SessionLocal() as db_session:
            # db_session.execute(...) 
            pass

        return {
            "statusCode": 200,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({"status": "success", "user": user_id})
        }
    except Exception as e:
        # En AWS, los prints van a CloudWatch Logs. 
        # (El módulo logging de Python es más robusto en producción).
        print(f"ERROR: {str(e)}") 
        return {"statusCode": 500, "body": "Internal server error"}
```

## 4. Diagrama (texto)

Ciclo de vida de Lambda:

[ Evento / Trigger ] ──► (¿Instancia activa disponible?)
                                │
                                ├── No ──► [ COLD START ]
                                │            1. Descarga el ZIP/Imagen.
                                │            2. Inicia micro-VM.
                                │            3. Ejecuta Imports globales / conexiones de BD (LENTO).
                                │            4. Ejecuta `lambda_handler()`
                                │
                                └── Sí ──► [ WARM START ]
                                             1. Ejecuta directamente `lambda_handler()` (RÁPIDO)

## 5. Preguntas de entrevista
- **Pregunta 1 (Práctica):** Tu Lambda en Python que consulta un RDS (PostgreSQL) agota las conexiones máximas de la base de datos bajo picos de carga. ¿Por qué ocurre y cómo lo solucionas?
- **Pregunta 2 (Conceptual):** ¿Cuál es la diferencia técnica entre procesar un evento de API Gateway (Síncrono) vs un evento S3/SNS (Asíncrono) en Lambda?
- **Pregunta 3 (Arquitectura):** Tienes una API pesada en FastAPI (200MB de dependencias como Pandas y Scikit-learn). ¿Lo portarías a Lambda directamente con un archivo `.zip`? ¿Cuáles son las alternativas?

## 6. Respuestas esperadas
- **Respuesta 1:** Ocurre porque cada ejecución concurrente de Lambda crea un nuevo contenedor, y cada contenedor abre una (o más) conexiones a Postgres. Si tienes 1000 Lambdas concurrentes, tendrás 1000 conexiones y agotarás el límite de RDS. Solución: Usar **RDS Proxy** frente a la base de datos (un pooler gestionado de conexiones HTTP a SQL) para concentrar y limitar las conexiones hacia la DB, independientemente de cuántas Lambdas escalen.
- **Respuesta 2:** Si se invoca desde API Gateway (Síncrona), el cliente espera activamente con la conexión HTTP abierta hasta que la Lambda termina y devuelve una respuesta, y los fallos se devuelven al cliente. Si se invoca desde S3 o SNS (Asíncrona), AWS encola el evento internamente. Lambda responde inmediatamente un "202 Accepted" asumiendo el encargo. Si la Lambda falla, AWS reintenta automáticamente por ti (por defecto, 2 veces) antes de mandarlo a una Dead Letter Queue (DLQ).
- **Respuesta 3:** Un `.zip` de Lambda tiene un límite estricto de 250MB (descomprimido). Además, cargar módulos C-extensions masivos y pre-procesar Dataframes causará Cold Starts severos (varios segundos), arruinando la experiencia de una API REST. Alternativas: 1) Usar **Lambda Container Images** (soporta imágenes Docker de hasta 10GB) habilitando *Provisioned Concurrency* para pre-calentar contenedores y evitar el Cold Start. 2) Mejor aún, mover la carga pesada (como inferencia de Machine Learning) a un contenedor en **ECS/Fargate** o SageMaker detrás de un ALB.

## 7. Errores comunes
- **Hardcodear secretos y configuración:** Almacenar variables del entorno estáticas y contraseñas en el código o variables de entorno de consola de AWS plana (que no están encriptadas adecuadamente o no permiten rotación). Siempre usar *AWS Parameter Store* o *Secrets Manager*.
- **Fat Lambdas (Monolitos en Lambda):** Subir todo un framework Django de 100 endpoints dentro de una sola Lambda con un handler monstruoso. Aunque es posible (ej. Zappa/Mangum), aumenta el tamaño, el tiempo de despliegue, complica permisos (IAM) y empeora cold starts.
- **Bucle infinito (Recursión):** Lambda A guarda en el Bucket X -> Bucket X lanza un evento que dispara la Lambda A. Genera facturas masivas ("Denial of Wallet"). Siempre usar buckets/caminos separados para entrada y salida.

## 8. Buenas prácticas
- **IAM "Least Privilege" por función:** No crear un único rol "MyBackendRole" con acceso a todas las bases de datos para asignarlo a todas las funciones. Cada Lambda individual debe tener su propio Rol IAM exclusivo (ej. solo acceso de lectura a S3, solo escritura a DynamoDB tabla Y).
- **X-Ray / Tracing:** Las arquitecturas serverless son inherentemente complejas de rastrear. Habilitar AWS X-Ray para correlacionar trazas de rendimiento desde API Gateway -> Lambda -> DynamoDB, e instrumentar el código Python con la librería `aws-xray-sdk`.
- **Lambda Layers:** Si múltiples Lambdas comparten el mismo set de librerías comunes o drivers (ej. `psycopg2`, `requests`), usar AWS Lambda Layers para externalizarlas del paquete de la función, acelerando el proceso de CI/CD.

## 9. Trade-offs y decisiones técnicas
**Lambda vs EC2/Contenedores para tareas largas**
- *Lambda:* Límite duro de 15 minutos (900 segundos). Si procesar un video tarda 16 minutos, fallará y AWS matará el proceso sin previo aviso. Imposible el acceso a recursos del host como memoria compartida persistente (ej. Redis local o colas de hilos complejos).
- *Decisión técnica:* Si una tarea promedia los 10 minutos, no debe estar en Lambda, ya que picos de tiempo ocasionales romperán el proceso (P99 tail latency > 15m). Usar **AWS Batch** o **ECS Fargate tasks** lanzadas asíncronamente para trabajos de larga duración (ETLs, Data processing).
