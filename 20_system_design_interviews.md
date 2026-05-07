# System Design Interviews (Ejercicios Prácticos)

## 1. Explicación técnica
La entrevista de System Design separa a los Junior (que saben codificar funciones) de los Senior/Staff (que saben arquitectar sistemas tolerantes a fallos bajo carga masiva).
Es una conversación abierta. No hay respuesta 100% correcta. El entrevistador evalúa tu capacidad para:
1. Acotar requisitos (Clarificar alcance).
2. Hacer estimaciones rápidas de datos y tráfico.
3. Proponer un diseño de alto nivel.
4. Identificar cuellos de botella y proponer trade-offs.

A continuación, 3 ejercicios clásicos explicados bajo una mentalidad AWS/Python.

---

## BONUS EJERCICIO 1: Diseñar un "URL Shortener" (tipo bit.ly)

### Paso 1: Requisitos
- Funcional: El usuario pega URL larga, devuelve URL corta (7 caracteres). Redirección rápida.
- No-funcional: Alta disponibilidad, Read-Heavy (100 lecturas por 1 escritura). Latencia bajísima.
- Escala: 100 millones de URLs generadas al mes.

### Paso 2: Estimaciones (Back of the Envelope)
- 100M URLs/mes = ~40 escrituras/seg.
- 100 lecturas/escritura = 4,000 lecturas/seg.
- Tamaño de 1 registro: ~500 bytes. 100M * 500B = 50GB al mes. 3TB en 5 años.

### Paso 3: Diseño Core y Algoritmo
- **Generación de ID:** ¿Base 62 (a-z, A-Z, 0-9)? $62^7$ = 3.5 billones de combinaciones (suficiente). 
- Para evitar colisiones en generación distribuida, podemos usar:
  a) Generador offline pre-generando UUIDs.
  b) Un servicio central (Ticket Server) que da rangos numéricos.

### Paso 4: Arquitectura (AWS)
- **Capa Edge:** Route53 -> AWS API Gateway.
- **Compute:** AWS Lambda (Barato, escalable, ideal para bursts rápidos) ejecutando Python/FastAPI.
- **Base de datos:** DynamoDB. Excelente para búsquedas de latencia 1ms (Key-Value) usando la URL corta como Partition Key. Relacional (RDS) no aporta valor porque no hay JOINs complejos.
- **Optimización de Lectura:** Caché agresivo. Poner CloudFront (CDN) delante y/o ElastiCache (Redis) para guardar el mapeo `{url_corta: url_larga}`. 80% de las URLs cortas famosas serán servidas desde memoria sin tocar DynamoDB.
- **Redirect:** HTTP 301 (Permanent, permite cacheo en navegador del cliente) vs HTTP 302 (Temporal, obliga a golpear el backend siempre, mejor para Analytics/Tracking de clics). Elegimos 302 por Analytics.

---

## BONUS EJERCICIO 2: Diseñar un "Netflix Backend" (Streaming de Video)

### Paso 1: Requisitos
- Funcional: Usuarios pueden ver videos y el reproductor guarda dónde se quedaron (Bookmarks). Los creadores suben videos.
- No-funcional: Video streaming debe ser ultra-estable, no toleramos buffer. El servicio de Bookmarks debe ser masivamente paralelo.

### Paso 2: Diseño de Alto Nivel (Separación de Planos)
Este problema se divide en dos dominios aislados:
1. **Control Plane:** Discovery, Búsqueda, Perfiles de usuario, Historial de vistas (Backend típico Python).
2. **Data Plane:** La entrega real de bytes de video (Heavy I/O, Red).

### Paso 3: Arquitectura de Subida (Upload Flow)
- El creador no sube a un servidor EC2 (agotaría el ancho de banda). Usa una **Presigned URL** para subir directamente el `.mp4` crudo a un Bucket S3 (Upload Bucket).
- S3 dispara un evento (EventBridge) a una cola SQS.
- Workers paralelos (EC2 o AWS Elemental MediaConvert) leen la cola y transcodifican el video en 10 resoluciones/formatos distintos simultáneamente (DASH/HLS chunks).
- Los chunks generados se guardan en el S3 "Distribution Bucket".

### Paso 4: Arquitectura de Entrega (Streaming Flow)
- El usuario NO hace streaming desde nuestro Python Backend ni desde S3 directo.
- Se usa una red de **Content Delivery Network (AWS CloudFront)** a nivel mundial (o Netflix Open Connect, ISP local). 
- El frontend pide a nuestro Backend (Python): "Dame los metadatos de la película y el token de acceso". El Backend valida la suscripción en RDS y devuelve URLs temporales firmadas que apuntan al CDN.

### Paso 5: Servicio de Bookmarks (Progreso del usuario)
- Un usuario viendo una serie emite un ping de "progreso (minuto 12:45)" cada 10 segundos. 
- Millones de usuarios = Carga masiva de escrituras masivas.
- Usar Amazon Kinesis / SQS para agrupar (buffer) las actualizaciones y escribirlas asíncronamente en una DB orientada a escrituras rápidas (Cassandra o DynamoDB).

---

## BONUS EJERCICIO 3: Diseñar un sistema de "Job Processing" masivo (Distributed Queue)

### Paso 1: Requisitos
- Un cliente envía 1 millón de imágenes para aplicar filtros (Job pesado).
- Tolerancia a fallos: Si un servidor muere, la imagen no debe perderse.

### Paso 2: Arquitectura Asíncrona Event-Driven
- **Capa API:** Un Load Balancer recibe peticiones y las envía a un cluster EC2/FastAPI.
- **La Cola (Broker):** El API encola metadatos (S3 Path) en Amazon SQS (RabbitMQ).
- **Los Workers:** Grupo de Auto Scaling de EC2 consumiendo SQS. Extraen el archivo, procesan (CPU), guardan el resultado en S3, y borran el mensaje de SQS (ACK).
- **Base de Datos:** RDS Postgres guarda solo metadatos (`job_id, status: pending/done, result_url`).

### Paso 3: Cuellos de botella y Trade-offs
- *Problema:* El cliente se desconectó y necesita saber cuándo terminó su millón de imágenes.
- *Solución:* Patrón **Webhook / Callback**. En lugar de que el cliente pregunte "¿Terminaste?" 100 veces por segundo (Polling ineficiente), el cliente provee una URL (Webhook). Nuestro último paso en el worker es enviar un POST al cliente (`{job_id: 1, status: done}`).
- *Problema:* Un worker procesando una imagen 8K se queda sin RAM y muere.
- *Solución SQS:* Gracias al Visibility Timeout de SQS, como el worker murió sin borrar el mensaje, pasados X minutos, otro worker verá la misma tarea y la reintentará. (Garantía At-Least-Once delivery).
- *Problema de Idempotencia:* ¿Qué pasa si el worker guarda la foto en S3, pero muere antes de hacer ACK a la cola? El mensaje reaparecerá. El nuevo worker sobrescribirá la foto en S3. En S3 no es un problema grave, pero si cobramos 1 dólar por foto (RDS), cobraríamos dos veces.
- *Solución:* Control de concurrencia en BD (Idempotency Key / Redis Locks) asegurando que el estado del job pase a "DONE" en una transacción, para que reintentos posteriores se ignoren.
