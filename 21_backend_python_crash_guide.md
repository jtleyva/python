# 🚀 Backend Python Interview Crash Guide (30 min)

## 1. Cómo responder en entrevistas (framework mental)

Para entrevistas de 30 minutos, la claridad y estructura son tu mejor arma. Usa el framework **STAR** (Situación, Tarea, Acción, Resultado) para experiencia, y un enfoque **Top-Down** para diseño.

**Estructura ideal para preguntas abiertas/diseño:**
1. **Aclarar requerimientos:** "¿Cuál es el volumen esperado?", "¿Es más importante la disponibilidad o la consistencia?"
2. **High-level design:** Componentes principales sin entrar en código.
3. **Deep dive:** Detallar la parte donde eres más fuerte (ej. diseño de la API o la base de datos).
4. **Trade-offs:** Explicar qué sacrificas con tu decisión.

**Frases clave para sonar Senior:**
- *"Depende del caso de uso. Si priorizamos X, usaría Y, pero el trade-off aquí es Z."*
- *"Para escalar esta solución, introduciría..."*
- *"El cuello de botella de este enfoque sería..."*
- *"En mi experiencia, la forma más robusta de manejar esto es..."*

---

## 2. System Design express (lo mínimo que debes dominar)

### Componentes Clave
1. **Load Balancer (ALB):** Distribuye tráfico, termina SSL.
2. **App Servers (Auto Scaling):** Instancias stateless corriendo tu API Python.
3. **Base de Datos (RDS/Aurora):** Master-Replica para lectura pesada, sharding/particionamiento si es enorme.
4. **Cache (Redis/Memcached):** Aliviar a la DB de queries frecuentes.
5. **Colas (SQS/RabbitMQ):** Desacoplar tareas asíncronas pesadas.

### Ejemplo: "Design a scalable backend"
**Respuesta modelo paso a paso:**
1. **Ingress:** Route53 → Application Load Balancer.
2. **Compute:** ECS (Fargate) o EKS con contenedores de FastAPI/Flask, escalando horizontalmente según uso de CPU.
3. **Data Layer:** Amazon Aurora PostgreSQL primaria. Réplicas de lectura para las consultas GET de la API.
4. **Cache Layer:** Amazon ElastiCache (Redis) para respuestas cacheadas y rate limiting.
5. **Async Workers:** Amazon SQS para recibir eventos y Celery workers procesando tareas en background para no bloquear la API.

Ejemplo explicado en parrafo:
*"Para diseñar un backend escalable, primero considero el tráfico de entrada a través de un Application Load Balancer que distribuye la carga. Luego, las solicitudes llegan a un grupo de instancias EC2 o contenedores en ECS/Fargate, donde ejecuto mi API con FastAPI o Flask. Para la persistencia de datos, uso Amazon Aurora PostgreSQL, que me permite tener una base de datos relacional con réplicas de lectura para manejar la carga de consultas. Además, implemento un sistema de caché con Amazon ElastiCache (Redis) para reducir la carga en la base de datos y mejorar la latencia de las respuestas. Para tareas asíncronas, como el procesamiento de archivos o el envío de correos, uso Amazon SQS para recibir eventos y workers de Celery que procesan estas tareas en background, evitando que bloqueen la API."*

---

## 3. AWS en entrevistas (lo que sí o sí debes decir)

### Cuándo usar qué:
- **EC2 vs Lambda:** Usa EC2/ECS para cargas constantes, predecibles o procesos largos. Usa Lambda para cargas con picos esporádicos, tareas cortas (cron), o eventos (triggers de S3/DynamoDB).
- **S3 vs RDS:** S3 para blobs, archivos, imágenes, backups. RDS (PostgreSQL/MySQL) para datos estructurados, relacionales y transaccionales (ACID).

### Arquitectura Básica (Serverless/Microservicios)
**API Gateway → Lambda → DynamoDB/RDS**
*Explicación:* "Para un microservicio ligero o un endpoint con tráfico variable, pondría API Gateway para routing y rate limiting, Lambda ejecutando Python para la lógica de negocio por ser serverless y autoescalable, y RDS Proxy conectado a Aurora Serverless para evitar el agotamiento de conexiones a la base de datos."

---

## 4. Backend Python clave

### Concurrencia
- **threading:** Útil para I/O-bound tasks (red, disco). Limitado por el GIL en CPU.
- **multiprocessing:** Útil para CPU-bound tasks (cálculos pesados). Evita el GIL creando procesos nuevos.
- **asyncio (FastAPI/Aiohttp):** Ideal para alto throughput I/O concurrente en un solo hilo. Maneja miles de conexiones con muy bajo overhead.

### Manejo de errores
Captura excepciones específicas en la capa correcta y devuelve respuestas HTTP estándar.
```python
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    try:
        user = db.get_user(user_id)
        if not user:
            raise HTTPException(status_code=404, detail="User not found")
        return user
    except DBConnectionError:
        logger.error("Database connection failed")
        raise HTTPException(status_code=503, detail="Service unavailable")
```

---

## 5. APIs REST (preguntas típicas)

### Buen diseño de endpoints
- Usar sustantivos plurales, no verbos: `GET /users`, `POST /users`, `GET /users/{id}`
- Versionado: `GET /api/v1/users`

### Status Codes clave
- `200 OK`, `201 Created`, `204 No Content`
- `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `429 Too Many Requests`
- `500 Internal Server Error`, `503 Service Unavailable`

### Idempotencia
Es vital. Un método es idempotente si llamarlo N veces tiene el mismo efecto que llamarlo 1 vez.
- **Idempotentes:** GET, PUT, DELETE
- **No idempotente:** POST (crea un recurso nuevo cada vez).
*Tip Senior:* "Para hacer un POST seguro ante reintentos de red, usaría un `Idempotency-Key` en los headers y guardaría el estado en Redis."

---

## 6. Preguntas típicas (TOP 15)

1. ¿Cómo diseñas un sistema escalable?
2. ¿Cómo manejarías un pico súbito de alta carga?
3. ¿Cuándo eliges Lambda vs EC2/ECS?
4. ¿Cómo optimizas el rendimiento de una API lenta?
5. ¿Qué es el GIL en Python y cómo lo esquivas?
6. ¿Diferencias entre asyncio y multithreading?
7. ¿Cómo diseñas una API RESTful correcta?
8. ¿Qué es idempotencia y por qué importa?
9. ¿Cómo manejas las migraciones de base de datos en CI/CD?
10. ¿Estrategias de caché que conoces?
11. ¿Cuándo usarías una base de datos NoSQL vs SQL?
12. ¿Cómo aseguras tus APIs?
13. ¿Qué métricas monitorizas en un backend en producción?
14. Cuéntame de un bug difícil que resolviste en producción.
15. ¿Cómo implementarías Rate Limiting?

---

## 7. Respuestas modelo a las TOP 15 Preguntas (Directas y Claras)

1. **¿Cómo diseñas un sistema escalable?**
   *"Arquitectura stateless. Un balanceador de carga (ALB) distribuye el tráfico a contenedores auto-escalables (ECS/EKS). El estado se guarda fuera: sesiones en Redis, datos en RDS, archivos en S3. Las tareas pesadas se encolan en SQS."*

2. **¿Cómo manejarías un pico súbito de alta carga?**
   *"Separaría lecturas de escrituras con read-replicas de la BD. Añadiría Redis para cachear datos frecuentes. Si es carga de procesamiento, usaría SQS para encolar peticiones y procesarlas de forma asíncrona sin bloquear la API."*

3. **¿Cuándo eliges Lambda vs EC2/ECS?**
   *"Lambda para cargas esporádicas, tareas cortas o basadas en eventos (cero mantenimiento, pago por uso). ECS/EC2 para tráfico constante, predecible o procesos pesados que superan los 15 minutos de límite de Lambda."*

4. **¿Cómo optimizas el rendimiento de una API lenta?**
   *"Identifico el cuello de botella con tracing. Si es la BD: añado índices, evito N+1 queries o uso read-replicas. Si es procesamiento de red/cómputo: implemento caché agresiva o paso las tareas lentas a workers asíncronos."*

5. **¿Qué es el GIL en Python y cómo lo esquivas?**
   *"El Global Interpreter Lock impide que múltiples hilos ejecuten código Python simultáneamente, limitando el uso de múltiples núcleos. Lo esquivo usando `multiprocessing` para cálculos pesados (CPU-bound) o `asyncio` para tareas de red (I/O-bound)."*

6. **¿Diferencias entre asyncio y multithreading?**
   *"`threading` usa hilos del sistema operativo (pesados) y está bloqueado por el GIL. `asyncio` usa un solo hilo y un event loop cooperativo, ideal para manejar miles de conexiones I/O concurrentes (ej. FastAPI) con muy bajo uso de memoria."*

7. **¿Cómo diseñas una API RESTful correcta?**
   *"Uso sustantivos plurales (`/users/123`), implemento versionado (`/v1/`), devuelvo status codes HTTP estándar (200, 201, 400, 404, 500) y uso correctamente los verbos (GET, POST, PUT, DELETE) asegurando que las respuestas sean consistentes en JSON."*

8. **¿Qué es idempotencia y por qué importa?**
   *"Es cuando llamar un endpoint una o múltiples veces produce el mismo resultado en el servidor. Es crucial en sistemas distribuidos para que los clientes puedan reintentar peticiones (por fallos de red) sin crear registros duplicados."*

9. **¿Cómo manejas las migraciones de base de datos en CI/CD?**
   *"Uso herramientas como Alembic (SQLAlchemy). Las migraciones corren automáticamente en el pipeline antes de desplegar la nueva app. Todas las migraciones deben ser retrocompatibles para evitar downtime y no bloquear tablas en producción."*

10. **¿Estrategias de caché que conoces?**
    *"Cache-Aside (la app intenta leer de caché, si no está, lee de la BD y la guarda en caché) y Write-Through (se escribe en caché y BD a la vez). Uso Redis típicamente para tokens, conteos o respuestas enteras de la API."*

11. **¿Cuándo usarías una base de datos NoSQL vs SQL?**
    *"SQL (PostgreSQL) por defecto: relacional, datos estructurados, transacciones ACID. NoSQL (DynamoDB/MongoDB) cuando hay un esquema muy dinámico, jerárquico o requiero escalabilidad horizontal masiva con latencias de lectura ultra bajas."*

12. **¿Cómo aseguras tus APIs?**
    *"Uso HTTPS siempre. Autenticación con JWT o OAuth2. Autorización basada en roles. Rate limiting para evitar abusos. Validación estricta de inputs (ej. Pydantic en FastAPI) y nunca expongo datos sensibles; uso AWS Secrets Manager."*

13. **¿Qué métricas monitorizas en un backend en producción?**
    *"Sigo el patrón RED: Rate (peticiones por segundo), Errors (tasa de 5xx y 4xx) y Duration (latencia p95 y p99). Además, monitorizo saturación de recursos como CPU, RAM y conexiones abiertas a la base de datos."*

14. **Cuéntame de un bug difícil que resolviste en producción.**
    *"[Respuesta comodín]: Hubo una vez un problema de rendimiento brutal por N+1 queries bajo carga. La API tardaba segundos. Lo detecté con Datadog, agrupe las consultas usando `select_related` / `joinedload` en el ORM y añadí caché de Redis. Pasó de 2s a 50ms."*

15. **¿Cómo implementarías Rate Limiting?**
    *"Lo implementaría en el API Gateway o con un middleware en Python apoyado en Redis. Usaría el algoritmo de Token Bucket asociando el límite a la IP del cliente o el API Key, devolviendo un HTTP 429 (Too Many Requests) al excederlo."*
    *"El token bucket es el más justo para APIs porque permite picos de tráfico controlados. Funciona con una tasa constante de tokens que se generan periódicamente y se consumen al hacer peticiones." Otras alternativas son el Fixed Window Counter (más simple pero menos justo) y el Sliding Window Log (más preciso pero más complejo)."*

