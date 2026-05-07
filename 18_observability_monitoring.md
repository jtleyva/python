# Observability & Monitoring

## 1. Explicación técnica
A nivel Senior, no puedes decir "el código funciona en mi máquina". En sistemas distribuidos masivos (microservicios), cuando algo falla, no tienes acceso a la terminal del servidor para usar `print()` o un debugger. Aquí es donde entra la **Observabilidad**, basada en los "Tres Pilares":

1. **Métricas (Metrics):** Datos numéricos agregados en el tiempo. Sirven para detectar anomalías (Dashboards y Alertas). Ej: `% de CPU`, `Latencia P99`, `Request Rate`.
2. **Registros (Logs):** Cadenas de texto emitidas por tu código con contexto rico sobre eventos específicos. Sirven para debugear *por qué* ocurrió algo.
3. **Trazas (Traces):** Rastrea el ciclo de vida de una sola petición HTTP a medida que atraviesa múltiples microservicios, bases de datos y colas. Responde *dónde* está el cuello de botella.

Herramientas comunes: AWS CloudWatch, Datadog, Prometheus/Grafana, ELK Stack (Elasticsearch, Logstash, Kibana), AWS X-Ray, OpenTelemetry.

## 2. Uso en producción
En Netflix, si la latencia del servicio de "Recomendaciones" sube de 50ms a 300ms, una alerta automática (basada en métricas Prometheus) avisa por PagerDuty al ingeniero de guardia. El ingeniero abre su dashboard (Grafana/Datadog) y usa las Trazas (Distributed Tracing) para ver que la API Gateway invoca al servicio de Recomendaciones, el cual a su vez llama a Redis. La traza muestra una enorme barra roja de 280ms en la llamada a Redis. Usando el "Trace ID" generado, el ingeniero filtra los Logs y descubre un error de "Connection Pool Timeout".

## 3. Ejemplo práctico (Python)

Implementación de Structured Logging (Logs en formato JSON) usando la librería estándar de Python + `structlog`. (En la nube, enviar texto plano es inútil, los logs deben ser procesables).

```python
import logging
import structlog
from fastapi import FastAPI, Request
import time
import uuid

# Configuración de Structlog para salida en JSON en Producción
structlog.configure(
    processors=[
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer() # Devuelve JSON estructurado
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
)

logger = structlog.get_logger()
app = FastAPI()

@app.middleware("http")
async def observability_middleware(request: Request, call_next):
    # Generar un Trace ID único (Correlation ID)
    trace_id = request.headers.get("X-Trace-Id", str(uuid.uuid4()))
    
    # Inyectamos el trace_id en el contexto del logger.
    # TODOS los logs en esta petición lo incluirán automáticamente.
    log = logger.bind(trace_id=trace_id, path=request.url.path, method=request.method)
    
    start_time = time.perf_counter()
    try:
        response = await call_next(request)
        process_time = (time.perf_counter() - start_time) * 1000
        
        # Log estructurado final con la latencia
        log.info("request_completed", status=response.status_code, duration_ms=round(process_time, 2))
        
        # Devolver el Trace ID al cliente para reporte de soporte
        response.headers["X-Trace-Id"] = trace_id
        return response
    except Exception as e:
        process_time = (time.perf_counter() - start_time) * 1000
        log.error("request_failed", error=str(e), duration_ms=round(process_time, 2))
        raise

@app.get("/checkout")
async def do_checkout():
    # El logger recordará el trace_id inyectado por el middleware
    logger.info("processing_payment", provider="stripe")
    # Lógica de negocio...
    return {"status": "ok"}

"""
Ejemplo de salida en consola (o AWS CloudWatch):
{"level": "info", "timestamp": "2023-10-15T12:00:01Z", "trace_id": "abc-123", "path": "/checkout", "method": "GET", "event": "processing_payment", "provider": "stripe"}
{"level": "info", "timestamp": "2023-10-15T12:00:02Z", "trace_id": "abc-123", "path": "/checkout", "method": "GET", "event": "request_completed", "status": 200, "duration_ms": 1050.4}
"""
```

## 4. Diagrama (texto)

Arquitectura de Distributed Tracing:

[ Cliente Front ] ──(Genera TraceID: 123)──► [ API Gateway ] 
                                                  │ (Log: Start req 123)
                                                  ▼
                                       [ Microservice A (Python) ]
                                           │   (Log: Valida user 123)
                                           ├──► [ Microservice B (Node) ] (Pasa Header X-Trace-Id: 123)
                                           │       (Log: Cobra tarjeta 123)
                                           │
                                           └──► [ Microservice C (Go) ] (Pasa Header X-Trace-Id: 123)
                                                   (Log: Envía Email 123)

*En el buscador centralizado (ELK / Datadog), buscas `trace_id: 123` y ves TODA la historia en orden cronológico en los 3 servicios.*

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Qué ventajas tiene el "Structured Logging" (Logs en JSON) frente al logging tradicional en texto plano (`INFO: User 5 logged in`) en arquitecturas Cloud?
- **Pregunta 2 (Práctica):** Has configurado una alarma que envía un SMS al ingeniero de guardia cada vez que la CPU del servidor llega al 90%. Tras una semana, el equipo ignora los SMS. ¿Por qué es un mal diseño de alertas y cómo lo mejorarías?
- **Pregunta 3 (Arquitectura):** Quieres saber cuánto tarda en promedio una query a Postgres dentro de tu API en producción, pero no quieres añadir prints de tiempo en cientos de endpoints a mano. ¿Qué patrón o tecnología de observabilidad usas?

## 6. Respuestas esperadas
- **Respuesta 1:** El texto plano requiere escribir Expresiones Regulares (Regex/Grok) complejas para extraer datos como el ID del usuario en el sistema centralizado de logs. Los logs en formato JSON permiten búsquedas nativas, indexación y agregaciones inmediatas. En Datadog/CloudWatch, si uso JSON, puedo buscar `user_id: 5 AND status: error` directamente sin configurar *parsers* de texto.
- **Respuesta 2:** Esto se conoce como **Alert Fatigue** (Fatiga de alertas). Una alerta debe ser accionable (Actionable). CPU alta es un "Síntoma", no un impacto al usuario. El servidor puede estar al 99% procesando jobs en background sin afectar a la API pública. Mejoraría la alerta basándola en **SLOs (Service Level Objectives) / RED Metrics (Rate, Errors, Duration)**. Alertaría si la "Tasa de errores 500 sube del 1%" o si la "Latencia P99 de la API supera los 2 segundos". Esas afectan al cliente y requieren intervención real.
- **Respuesta 3:** Usaría **APM (Application Performance Monitoring) / OpenTelemetry Auto-instrumentation**. Las librerías de APM (como `ddtrace` de Datadog o `aws-xray-sdk`) usan "monkey patching" en el momento de arranque del intérprete Python para envolver silenciosamente librerías estándar (`psycopg2`, `requests`, `sqlalchemy`) sin modificar mi código de negocio. Generan trazas y métricas exactas de latencia de red y base de datos automáticamente.

## 7. Errores comunes
- **Loguear PII (Personally Identifiable Information):** Escribir números de tarjeta de crédito, contraseñas o IDs gubernamentales en los logs. Es una violación de GDPR/PCI y un vector de fuga masivo si un desarrollador obtiene acceso a CloudWatch.
- **Mirar el Promedio (Average):** Usar el promedio de latencia (Average) para evaluar el rendimiento. Si 9 peticiones tardan 1ms y 1 tarda 1000ms, el promedio es 100ms, ocultando que el 10% de los usuarios tiene una experiencia horrible. Siempre usar **Percentiles (P90, P95, P99)**. Si el P99 es de 500ms, significa que el 99% de las requests toman 500ms o menos.

## 8. Buenas prácticas
- **Niveles de Log correctos:** 
  - `ERROR`: Requiere intervención humana (Falla conexión a DB). Dispara alertas.
  - `WARN`: El sistema se degradó pero sigue funcionando (Fallback a Caché usado).
  - `INFO`: Hitos de negocio significativos (Usuario completó compra).
  - `DEBUG`: Datos voluminosos para desarrollo. Desactivado en Producción.
- **Correlation IDs:** Propagar un ID único desde la frontera (API Gateway) hacia la base de datos (incluso inyectado en comentarios SQL `/* trace_id: 123 */`) para mapear cuellos de botella transversales.

## 9. Trade-offs y decisiones técnicas
**CloudWatch (Nativo AWS) vs Datadog (SaaS)**
- *CloudWatch:* Barato si generas poco volumen, integrado por defecto en AWS (sin agentes que instalar), excelente para métricas básicas (CPU/Memoria/SQS). Trade-off: La interfaz de usuario es clunky/áspera, crear dashboards potentes es laborioso y las capacidades de APM/Tracing (X-Ray) están muy por detrás del mercado.
- *Datadog:* El "Estándar de Oro" de la industria. Interfaz espectacular, APM sin esfuerzo, correlación mágica entre Logs y Trazas. Trade-off: **Extremadamente caro**. Su modelo de precios (por host y por ingesta masiva de logs) asusta a los CFOs si no filtras (Sample rate) o descartas logs inútiles en la entrada.
