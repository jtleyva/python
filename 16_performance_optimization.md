# Performance Optimization

## 1. Explicación técnica
Optimizar el rendimiento en backend implica identificar y eliminar cuellos de botella (bottlenecks). Rara vez el problema es la velocidad del lenguaje Python en sí; casi siempre el cuello de botella está en operaciones de I/O (Input/Output), como consultas a bases de datos ineficientes, llamadas de red lentas o falta de paginación.

Cuando el problema sí es de cómputo (CPU-bound), Python ofrece alternativas:
- **Indexación y Caché:** La forma más rápida de hacer algo es no hacerlo. Usa índices en la BD y cachés en memoria (Redis/Memcached).
- **Asincronía (asyncio):** Permite manejar miles de conexiones I/O concurrentes sin el overhead de crear miles de hilos del sistema operativo.
- **Procesamiento en Background:** Mover tareas lentas (ej. envío de emails, generación de PDFs) fuera del ciclo de petición-respuesta HTTP usando colas de tareas (Celery/RQ).

### BONUS: Bottlenecks comunes en Python
- **N+1 Queries:** Hacer una consulta para obtener 100 registros y luego iterar sobre ellos haciendo 100 consultas más.
- **Deserialización JSON Masiva:** Usar la librería `json` estándar para payloads gigantes. Librerías como `orjson` o `ujson` (escritas en C/Rust) son órdenes de magnitud más rápidas.
- **Falta de Connection Pooling:** Abrir y cerrar una nueva conexión a la BD en cada petición HTTP (costoso en tiempo y recursos).

### BONUS: Concurrency (asyncio, threading, multiprocessing)
- **threading:** Ideal para operaciones bloqueantes de I/O (ej. librerías legacy como `requests` o `boto3` síncrono). El GIL de Python permite cambiar de hilo mientras uno espera respuesta de red.
- **asyncio:** Ideal para I/O no bloqueante masivo (FastAPI, aiohttp, asyncpg). Un solo hilo maneja miles de conexiones usando un Event Loop.
- **multiprocessing:** La única forma de lograr paralelismo real en Python para tareas CPU-bound (ej. procesamiento de imágenes, cifrado pesado). Cada proceso tiene su propio intérprete Python y su propio GIL, pero consumen más memoria y la comunicación entre procesos (IPC) es costosa.

## 2. Uso en producción
Instagram (que usa Django) optimizó su rendimiento implementando un estricto control de queries a la BD. Crearon herramientas internas para hacer fallar los tests si un endpoint HTTP ejecutaba más de un número predeterminado de queries. También hacen un uso intensivo de Memcached para evitar golpear la base de datos de Postgres en cada lectura del feed.

## 3. Ejemplo práctico (Python)

Ejemplo de optimización de una vista de API (FastAPI) usando Caché (Redis) y procesamiento en Background.

```python
from fastapi import FastAPI, BackgroundTasks, Depends
import redis.asyncio as redis
import json
import time

app = FastAPI()

# Cliente de Redis asíncrono
redis_client = redis.from_url("redis://localhost")

async def get_expensive_data_from_db(user_id: int) -> dict:
    """Simula una consulta SQL compleja y lenta (ej. agregaciones)"""
    await asyncio.sleep(2) # I/O bound blocking
    return {"user_id": user_id, "total_spent": 1500.50, "loyalty_points": 300}

def send_welcome_email(user_id: int):
    """Tarea lenta simulada (I/O bound síncrona o CPU-bound)"""
    time.sleep(1)
    print(f"Email sent to {user_id}")

@app.get("/users/{user_id}/stats")
async def get_user_stats(user_id: int, background_tasks: BackgroundTasks):
    """
    Endpoint optimizado:
    1. Revisa caché (Redis) primero.
    2. Si no hay caché, consulta DB y guarda en caché.
    3. Delega tareas lentas (email) a background sin bloquear la respuesta.
    """
    cache_key = f"user_stats:{user_id}"
    
    # 1. Intentar leer de caché (latencia ~1ms)
    cached_data = await redis_client.get(cache_key)
    if cached_data:
        # Programar tarea en background (retorna inmediatamente)
        background_tasks.add_task(send_welcome_email, user_id)
        return json.loads(cached_data)
        
    # 2. Cache Miss: Consultar base de datos (latencia ~2000ms)
    db_data = await get_expensive_data_from_db(user_id)
    
    # 3. Guardar en caché con TTL (Time To Live) de 60 segundos
    await redis_client.setex(cache_key, 60, json.dumps(db_data))
    
    background_tasks.add_task(send_welcome_email, user_id)
    return db_data
```

## 4. Diagrama (texto)

Optimización con Caché y Background Workers:

[ Cliente ] ──► [ API (FastAPI) ] ──(Cache Hit?)──► [ Redis ] (Rápido, devuelve)
                       │                 ▲
                  (Cache Miss)           │
                       │                 │ (Guarda en caché)
                       ▼                 │
                 [ Base de Datos (Lento) ]

                       │ (Delega tarea)
                       ▼
                 [ Celery / BackgroundTask ] ──► [ SMTP Server (Email) ] (Asíncrono, no bloquea cliente)

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Por qué añadir más servidores (escalado horizontal) no siempre mejora el rendimiento de una aplicación y a veces lo empeora?
- **Pregunta 2 (Práctica):** Tienes un script en Python que debe descargar 10,000 imágenes de S3 y redimensionarlas. ¿Cómo arquitectarías la solución para que sea lo más rápida posible?
- **Pregunta 3 (Arquitectura):** El uso de CPU de tu base de datos RDS Postgres está al 99% constante. La aplicación va muy lenta. ¿Cuáles son los pasos para diagnosticar y solucionar esto?

## 6. Respuestas esperadas
- **Respuesta 1:** Porque el cuello de botella puede no estar en la capa de cómputo (servidores web), sino en la base de datos (recurso compartido). Si la base de datos está saturada por bloqueos (locks) o queries pesadas, añadir más servidores web solo enviará más peticiones concurrentes a la BD, saturándola aún más rápido y causando timeouts masivos (thundering herd problem).
- **Respuesta 2:** Es un problema mixto (I/O-bound para la descarga, CPU-bound para el redimensionamiento). Usaría `multiprocessing.Pool` para crear un proceso por cada núcleo de CPU disponible. Dentro de cada proceso, usaría `asyncio` o `ThreadPoolExecutor` para descargar múltiples imágenes concurrentemente (I/O) sin bloquear. Una vez descargada una imagen en memoria, la redimensiono en ese mismo proceso (CPU) y la subo de nuevo.
- **Respuesta 3:** 1) Habilitar Performance Insights / `pg_stat_statements` en RDS para identificar la query exacta que está consumiendo más CPU. 2) Analizar el plan de ejecución de esa query usando `EXPLAIN ANALYZE`. 3) Solución a corto plazo: Añadir el índice faltante (si es un Full Table Scan) o optimizar la query (ej. evitar `%LIKE%` al inicio). 4) Solución a medio plazo: Implementar caché (Redis) para esa query si los datos cambian poco. 5) Solución a largo plazo: Añadir Read Replicas y desviar tráfico de lectura hacia ellas.

## 7. Errores comunes
- **Premature Optimization:** Perder días optimizando un algoritmo en Python para que tarde 1ms en lugar de 5ms, cuando la consulta a la base de datos en ese mismo endpoint tarda 500ms. *Regla: Perfila (profiling) primero, optimiza después.*
- **Caché sin estrategia de invalidación:** Guardar datos en Redis y nunca borrarlos. Los usuarios ven datos desactualizados (Stale data) para siempre. Siempre usar TTLs o invalidar explícitamente la clave en caché cuando el dato subyacente se actualice (Write-through).

## 8. Buenas prácticas
- **Profiling:** Usar herramientas como `cProfile`, `py-spy` o APMs (Datadog/NewRelic) para encontrar métricas reales (tiempos de DB, red, CPU local) antes de adivinar el cuello de botella.
- **Batching:** Si necesitas insertar 1000 registros en la BD, no hagas 1000 peticiones `INSERT`. Usa inserciones masivas (Bulk Inserts / `executemany` en SQLAlchemy) para hacer 1 sola petición de red con los 1000 registros.

## 9. Trade-offs y decisiones técnicas
**FastAPI (Async) vs Django/Flask (Sync)**
- *FastAPI (ASGI):* Rendimiento masivo en peticiones concurrentes I/O-bound (WebSockets, microservicios que llaman a otras APIs). Trade-off: El ecosistema (ORMs asíncronos, librerías) es más inmaduro y fragmentado. Un solo `time.sleep(1)` bloquea todo el servidor.
- *Django/Flask (WSGI):* Sólido como una roca, gigantesco ecosistema de librerías. Trade-off: Menos requests por segundo en hardware equivalente bajo alta concurrencia, requiere servidores WSGI como Gunicorn/uWSGI gestionando múltiples *workers* (procesos/hilos) que consumen más memoria RAM base.
