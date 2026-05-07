# AWS Architecture Patterns

## 1. Explicación técnica
Las arquitecturas maduras en AWS rara vez son síncronas. Depender de llamadas directas entre múltiples componentes es frágil. A nivel Senior, debes dominar patrones arquitectónicos que promuevan la alta disponibilidad, desacoplamiento y resiliencia.

Patrones principales:
- **Event-Driven Architecture (EDA):** Componentes publican y consumen eventos asíncronos en lugar de invocarse directamente.
- **Fan-Out Pattern:** Un único mensaje se duplica y se envía a múltiples suscriptores distintos (ej. SNS a múltiples colas SQS).
- **Dead Letter Queue (DLQ):** Un lugar para almacenar mensajes que fallaron su procesamiento tras múltiples intentos (para inspección y reprocesamiento manual).
- **Strangler Fig Pattern:** Migración gradual de monolitos a microservicios proxying tráfico selectivo.

### BONUS: Event-Driven Architecture (EDA)
La EDA usa *Event Brokers* o *Message Queues* (SQS, SNS, EventBridge, Kinesis). Aporta:
- Resiliencia: Si el servicio B está caído, la cola de SQS almacena los mensajes hasta que B se recupere.
- Escalado dinámico: Si llegan miles de eventos, AWS Auto Scaling añade más workers en función de la longitud de la cola SQS.

### BONUS: Serverless vs Containers
- **Serverless (Lambda/API Gateway/DynamoDB):** Se escala automáticamente de 0 a 10,000 en segundos, no pagas si no hay tráfico (pago por uso real), y la gestión operativa es mínima. Trade-offs: *Cold Starts* (latencia inicial), límites de tiempo de ejecución (15 min máx), y *Vendor Lock-in* fuerte.
- **Containers (ECS/EKS Fargate):** Sin límites de tiempo, mayor control del entorno, permite portabilidad (Docker corre en cualquier nube). Trade-offs: Escala más lento que Lambda, hay un coste base por mantener las tareas corriendo incluso sin tráfico, mayor esfuerzo de configuración.

## 2. Uso en producción
Ticketmaster usa el patrón *Fan-Out* con SNS y SQS. Cuando compras una entrada, el servicio de Checkout publica un mensaje "TicketPurchased" a un tópico SNS. Inmediatamente responden al cliente (latencia baja). En background, SNS duplica el evento a tres colas SQS diferentes consumidas por: 1) El servicio de Email (envía confirmación), 2) El servicio de Inventario (marca la butaca como ocupada), 3) El servicio de Fraud Analytics (evalúa la tarjeta).

## 3. Ejemplo práctico (Python)

Implementación de publicación en SNS y proceso de DLQ (Python SDK boto3).

```python
import boto3
import json
import logging
from typing import Dict, Any

sns_client = boto3.client('sns', region_name='us-east-1')
logger = logging.getLogger(__name__)

TOPIC_ARN = "arn:aws:sns:us-east-1:123456789012:OrderCreatedTopic"

def publish_order_event(order_id: str, customer_email: str) -> bool:
    """
    Publica un evento de dominio a un tópico SNS (Fan-out).
    No sabe (ni le importa) qué microservicios escucharán este evento.
    """
    event_payload = {
        "event_type": "OrderCreated",
        "data": {
            "order_id": order_id,
            "customer_email": customer_email
        }
    }
    try:
        response = sns_client.publish(
            TopicArn=TOPIC_ARN,
            Message=json.dumps(event_payload),
            # Atributos útiles para filtrado (Message Filtering en SNS)
            MessageAttributes={
                'source': {'DataType': 'String', 'StringValue': 'web_checkout'}
            }
        )
        logger.info(f"Event published. MessageId: {response['MessageId']}")
        return True
    except Exception as e:
        logger.error(f"Failed to publish event: {e}")
        return False

# Pseudo-código de un Worker (ej. un script Celery o una Lambda) consumiendo SQS
def process_sqs_message(event_record: Dict[str, Any]):
    """
    Procesa un mensaje SQS. Si levanta una excepción, SQS lo reintentará según
    el Visibility Timeout configurado. Si supera los maxReceiveCount, irá a la DLQ.
    """
    try:
        payload = json.loads(event_record['body'])
        # Regla de negocio frágil
        send_confirmation_email(payload['data']['customer_email'])
    except SMTPError:
        # El correo falló. Levantamos excepción para NO borrar el mensaje de la cola.
        raise RuntimeError("Transient error, retry later")
    except KeyError:
        # Formato inválido. No tiene sentido reintentar. 
        # Idealmente moverlo manualmente a DLQ o dejar que expire los intentos.
        logger.error("Malformed message, skipping")
        # No se levanta excepción, el framework (Lambda o Worker) lo asume exitoso y lo borra.
```

## 4. Diagrama (texto)

Patrón Fan-Out con DLQ:

                      ┌──► [ SQS Queue (Email) ] ──► [ Worker EC2 ] ─(Fallo)─► [ DLQ (Email) ]
                      │                                                        ▲ (Analizar luego)
                      │
[ Checkout API ] ──► [ SNS Topic ] 
                      │
                      │
                      └──► [ SQS Queue (Analytics) ] ──► [ Lambda Function ]

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Qué es un "Visibility Timeout" en SQS y por qué es fundamental para sistemas concurrentes?
- **Pregunta 2 (Arquitectura):** Diseña un sistema de ingesta masiva (millones de requests/minuto) de métricas de dispositivos IoT en AWS. Los datos deben ser procesados, agrupados, y guardados en S3 para análisis.
- **Pregunta 3 (Práctica):** En una arquitectura Event-Driven, ¿cómo garantizas que los eventos de modificación del carrito de un mismo usuario se procesen exactamente en el orden en que ocurrieron?

## 6. Respuestas esperadas
- **Respuesta 1:** Cuando un consumer (worker/Lambda) extrae un mensaje de SQS, el mensaje no se borra, se vuelve "invisible" para otros workers durante el Visibility Timeout. Esto permite procesarlo sin condiciones de carrera. Si el worker tiene éxito, borra el mensaje. Si el worker muere o se cuelga (superando el timeout), el mensaje vuelve a ser visible para que otro worker lo procese. Evita pérdida de datos ante fallos.
- **Respuesta 2:** Usaría **Amazon Kinesis Data Streams** (o MSK/Kafka) en lugar de SQS o REST directo. Kinesis está diseñado para streaming masivo de datos continuos. Colocaría API Gateway (configurado como AWS Service Proxy directo a Kinesis) o AWS IoT Core como punto de entrada. Luego, usaría **Kinesis Data Firehose** para agrupar (batching), comprimir (Parquet/GZIP) y guardar los datos directamente en **S3**, sin escribir una sola línea de código complejo.
- **Respuesta 3:** SQS estándar no garantiza orden estricto (best-effort ordering). Para garantizar orden FIFO (First-In-First-Out), usaría una **SQS FIFO Queue** (o Kinesis/Kafka particionado). En SQS FIFO, agruparía los mensajes usando el `MessageGroupId` seteado al `user_id`. AWS garantiza que todos los eventos con el mismo Group ID se procesarán de uno en uno, en orden estricto.

## 7. Errores comunes
- **Polling ineficiente de SQS:** Usar "Short Polling" haciendo peticiones constantes a SQS cuando la cola está vacía, incurriendo en altos costos. La buena práctica es habilitar "Long Polling" (ej. 20 segundos de `WaitTimeSeconds`), reduciendo drásticamente las llamadas a API y los costos.
- **Crear DLQs sin alarmas:** Mover mensajes a una DLQ y olvidarse de ellos es un anti-patrón de "agujero negro". Toda DLQ debe tener una alarma de CloudWatch (ej. `ApproximateNumberOfMessagesVisible > 0`) que envíe un correo o un ticket de Slack/Jira al equipo de guardia (On-call) para intervención manual o redriving (reenvío a la cola original).

## 8. Buenas prácticas
- **SNS Message Filtering:** En lugar de suscribir un microservicio a todos los eventos y que su código de Python ignore el 90% (pagando computo inútil), usar filtros en SNS para que AWS solo envíe los eventos relevantes al SQS de ese microservicio (ej. solo tipo `OrderRefunded`).
- **Idempotencia de Workers:** Dado que SQS/SNS prometen entrega "At-Least-Once", los workers *deben* ser idempotentes (comprobando en BD si el pago ya existe antes de procesarlo, usando un UUID de mensaje único).

## 9. Trade-offs y decisiones técnicas
**EventBridge vs SNS + SQS**
- *SNS + SQS:* Es la arquitectura clásica, robusta y con throughput virtualmente infinito. Excelente para topologías simples de Fan-Out. Trade-off: Menos capacidades de filtrado y requiere configurar infraestructura por separado.
- *EventBridge:* Es un Enterprise Service Bus real. Excelente para integrar SaaS de terceros (Zendesk, Datadog), permite transformaciones de eventos en vuelo y capacidades de enrutamiento basadas en contenido muy complejas (Rule based). Trade-off: Ligeramente más caro por millón de eventos que SNS y tiene cuotas predeterminadas de throughput (TPS) más bajas que deben solicitarse aumentar.
