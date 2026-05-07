# Scalability Patterns

## 1. Explicación técnica
Escalar un sistema no significa solo hacer que maneje más peticiones, sino hacer que crezca de forma predecible en coste e infraestructura. Hay dos enfoques principales:
- **Escalado Vertical (Scale Up):** Añadir CPU/RAM al mismo servidor. Límite físico duro, implica tiempo de inactividad (Downtime).
- **Escalado Horizontal (Scale Out):** Añadir más servidores de menor capacidad paralelos. Límite virtualmente infinito. Requiere arquitecturas sin estado (*Stateless*).

Patrones de diseño orientados a escalabilidad:
- **Caching (Read-Heavy workloads):** Absorber golpes de lectura. (CDN para assets estáticos, Redis para consultas BD).
- **Asynchrony (Write-Heavy workloads):** Usar Message Queues (SQS/Kafka) para nivelar picos de escritura (Load Leveling/Rate Limiting pasivo).
- **Database Sharding:** Repartir filas de una tabla gigante de base de datos a través de múltiples servidores separados basándose en una clave (Shard Key). (Ej. Usuarios de A-M en Server 1, N-Z en Server 2).

## 2. Uso en producción
Para el Black Friday, un eCommerce no escala verticalmente su Base de Datos Relacional, ya que es costoso y peligroso. Lo que hace es: 
1. Poner un CDN (CloudFront) frente al tráfico para cachear el HTML/Imágenes del catálogo (quita el 80% de la carga a los servidores web). 
2. Colocar Redis/Memcached frente a la base de datos (Cache-Aside).
3. Todas las creaciones de pedidos (Writes) no se insertan directo en Postgres; se encolan en SQS o RabbitMQ, permitiendo a los workers insertar en Postgres a un ritmo seguro sin sobrecargar bloqueos de tabla.

## 3. Ejemplo práctico (Python)

Patrón de **Desacoplamiento / Load Leveling** usando AWS SQS y un script de productor en Python.

```python
import boto3
import json

# En una arquitectura escalable, el API (Productor) responde increíblemente rápido
# porque NO hace trabajo pesado, solo encola la tarea.
sqs = boto3.client('sqs', region_name='us-east-1')
QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/12345/video-processing-queue"

def handle_upload_api_request(user_id: str, video_path: str):
    """
    Endpoint ficticio: POST /videos
    Recibe el archivo, lo sube rápido a S3, y encola la orden de procesamiento.
    Latencia: ~50ms.
    """
    # 1. Subir a S3 (omitido por brevedad)
    
    # 2. Enviar a la cola para procesamiento en background (Escalamiento Asíncrono)
    message_body = {
        "user_id": user_id,
        "s3_path": video_path,
        "action": "transcode_to_mp4"
    }
    
    sqs.send_message(
        QueueUrl=QUEUE_URL,
        MessageBody=json.dumps(message_body)
    )
    
    # 3. Respondemos al cliente inmediatamente (Accepted)
    # El cliente hará Polling o usará WebSockets luego para ver el estado.
    return {"status": "processing_started", "video_id": "temp-123"}
```

## 4. Diagrama (texto)

Escalamiento Horizontal y Asincronía (Event-Driven Load Leveling):

                        (Spike: 10,000 req/sec Black Friday)
[ Clients ] ──────────────► [ API Load Balancer ]
                                     │
         ┌───────────────────────────┼───────────────────────────┐
         ▼                           ▼                           ▼
  [ API Server 1 ]            [ API Server 2 ]            [ API Server N ] (Auto-Scaled)
         │                           │                           │
         └───────────────────────────┼───────────────────────────┘
                                     ▼
                      [ AWS SQS (Message Queue) ] (Absorbe el pico, Buffer)
                                     │
         ┌───────────────────────────┼───────────────────────────┐ (Consumo a ritmo constante)
         ▼                           ▼                           ▼
  [ Worker EC2 ]              [ Worker EC2 ]              [ Worker EC2 ]
         │                           │                           │
         └───────────────────────────┼───────────────────────────┘
                                     ▼
                     [ Amazon RDS / Base de Datos ] (No se cae, protegida por los workers)

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Qué es la latencia y el Throughput (rendimiento), y cómo afecta el añadir un caché a ambas métricas?
- **Pregunta 2 (Práctica):** Tienes un sistema de votación online en tiempo real (ej. Eurovisión). Usas Python y Redis. ¿Cómo manejas un pico de 100,000 votos por segundo sin saturar el sistema?
- **Pregunta 3 (Arquitectura):** Vas a escalar horizontalmente tu aplicación web en Python detrás de un Load Balancer. ¿Qué requisitos o reglas de diseño debe cumplir el código de la aplicación para que funcione?

## 6. Respuestas esperadas
- **Respuesta 1:** *Latencia* es el tiempo que tarda una sola petición de principio a fin (ms). *Throughput* es la cantidad de operaciones completadas por unidad de tiempo (requests/sec). Un caché de Redis reduce drásticamente la Latencia (de 500ms en SQL a 2ms en RAM), lo que indirectamente permite que el mismo servidor (con los mismos recursos de hilos/conexiones) pueda manejar más peticiones simultáneamente, aumentando enormemente el Throughput del sistema.
- **Respuesta 2:** La base de datos SQL no soportará 100K inserciones por segundo fácilmente. En lugar de insertar cada voto directo en DB, usaría **Redis In-Memory Counters** (`INCR` atómico) o un sistema de colas (Kafka/SQS) para absorber los votos. Un Worker secundario (o un Cron Job) leería el total agregado en Redis cada X segundos (ej. cada 5 seg) y haría un solo `UPDATE` (Bulk update) en la base de datos SQL (Write-Behind / Write-Back caching pattern).
- **Respuesta 3:** El código DEBE ser totalmente **Stateless (Sin Estado)**. Si guardas variables en memoria del módulo (ej. `global_counter += 1`) o escribes la sesión del usuario o imágenes subidas en el disco local (`/tmp/uploads`), el escalamiento horizontal fallará porque la próxima petición de ese usuario será ruteada (Round-Robin) a un Servidor 2 diferente que no tiene esa memoria ni archivo. Todo el estado (sesiones, archivos, cachés) debe vivir en servicios centralizados fuera de los servidores web (Redis, S3, RDS).

## 7. Errores comunes
- **Auto-Scaling basado en la métrica equivocada:** Escalar instancias basado en uso de CPU cuando el código es I/O-bound (esperando a BD). Las instancias escalarán infinitamente pero seguirán atascadas esperando, multiplicando el coste sin resolver el problema. Escalar basado en *Concurrent Requests* o *Queue Length* suele ser mejor.
- **Microservicios "Síncronos":** Separar una app grande en 10 microservicios HTTP que se llaman unos a otros secuencialmente en cadena en la misma request. La latencia se suma (A+B+C+D) y la probabilidad de fallo se multiplica. Esto degrada brutalmente la escalabilidad (Distributed Monolith).

## 8. Buenas prácticas
- **Backpressure (Contrapresión):** Cuando un sistema no da abasto, debe tener mecanismos para negarse a aceptar más trabajo de forma temprana y barata (Rate Limiting devolviendo HTTP 429 Too Many Requests), protegiendo sus recursos críticos. "Es mejor atender a 1,000 usuarios perfectamente y rechazar 200, que intentar atender 1,200 y que la base de datos muera tirando el servicio al 100%".
- **Pagination Everywhere:** Nunca devolver `SELECT * FROM tabla`. Siempre poner un límite duro (`limit=100`) en la capa de API para evitar ataques de Denial of Service (DoS) exhaustivos de memoria.

## 9. Trade-offs y decisiones técnicas
**Read Replicas vs Database Sharding**
- *Read Replicas (Master-Slave):* Una base de datos recibe todas las escrituras y replica datos a N bases de datos de solo lectura. Fácil de configurar (AWS RDS lo hace en 1 clic). Trade-off: Existe "Replication Lag" (inconsistencia eventual de milisegundos). No soluciona el problema si tu cuello de botella son inserciones/escrituras masivas, ya que todas las escrituras siguen yendo a un único servidor Master.
- *Sharding:* Dividir físicamente los datos (ej. Usuarios europeos en DB 1, americanos en DB 2). Resuelve cuellos de botella de lectura Y escritura. Trade-off: Extremadamente complejo. Si necesitas hacer un `JOIN` entre usuarios americanos y europeos, es imposible a nivel SQL. Si el negocio pide búsquedas globales complejas, requerirás sistemas de Data Warehouse externos e indexado paralelo.
