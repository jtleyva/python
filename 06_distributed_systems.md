# Distributed Systems

## 1. Explicación técnica
Un sistema distribuido es aquel donde los componentes ubicados en computadoras conectadas en red se comunican y coordinan sus acciones pasando mensajes. El objetivo es que parezca un único sistema coherente para el usuario final.

Conceptos fundamentales:
- **Teorema CAP:** En un sistema de datos distribuido, solo puedes garantizar dos de tres: **C**onsistency (todos los nodos ven los mismos datos al mismo tiempo), **A**vailability (cada petición recibe una respuesta de éxito/fallo) y **P**artition Tolerance (el sistema sigue operando a pesar de que se pierdan mensajes entre nodos). Como en redes reales las particiones (P) son inevitables, debes elegir entre CP (Consistencia fuerte) o AP (Alta disponibilidad).
- **Consistencia Eventual:** (AP) Garantiza que, si no se realizan nuevas actualizaciones a un dato, eventualmente todos los accesos devolverán el último valor actualizado.
- **Leader Election:** Si tienes múltiples instancias de un worker procesando pagos programados, necesitas que solo uno actúe como "Líder" en un momento dado para no cobrar dos veces.

## 2. Uso en producción
Sistemas como DynamoDB de Amazon fueron diseñados sacrificando la consistencia estricta por una disponibilidad extrema (AP en el teorema CAP, aunque configurable). Para el carrito de compras de Amazon, es preferible mostrar un producto que compraste hace 2 segundos como si aún estuviera en el carrito (inconsistencia temporal), a mostrar un error 500 porque el nodo de base de datos primario está caído.

## 3. Ejemplo práctico (Python)

Implementación de un bloqueo distribuido (Distributed Lock) usando Redis, crucial para evitar *race conditions* en entornos distribuidos.

```python
import redis
import time
from contextlib import contextmanager

# Cliente Redis (ej. AWS ElastiCache)
r = redis.Redis(host='localhost', port=6379, db=0)

class LockAcquisitionError(Exception):
    pass

@contextmanager
def redis_distributed_lock(lock_name: str, acquire_timeout: int = 10, lock_timeout: int = 10):
    """
    Bloqueo distribuido usando Redis SETNX (Set if Not eXists).
    Evita que 2 instancias del mismo microservicio ejecuten una tarea a la vez.
    """
    identifier = str(time.time())
    end_time = time.time() + acquire_timeout
    lock = False
    
    try:
        while time.time() < end_time:
            # Intenta adquirir el lock atómicamente
            if r.set(lock_name, identifier, nx=True, ex=lock_timeout):
                lock = True
                yield True
                break
            time.sleep(0.1) # Backoff antes de reintentar
            
        if not lock:
            raise LockAcquisitionError(f"Could not acquire lock: {lock_name}")
            
    finally:
        # Libera el lock solo si fuimos nosotros quienes lo adquirimos
        if lock and r.get(lock_name) and r.get(lock_name).decode('utf-8') == identifier:
            r.delete(lock_name)

# Uso en un worker distribuido (ej. Celery task)
def process_monthly_billing(user_id: str):
    try:
        with redis_distributed_lock(f"billing_lock_{user_id}", acquire_timeout=2):
            print(f"Lock adquirido. Procesando pago para {user_id}")
            # Lógica de cobro (lenta)
            time.sleep(5)
            print("Pago procesado.")
    except LockAcquisitionError:
        print(f"Otra instancia ya está procesando a {user_id}. Ignorando.")
```

## 4. Diagrama (texto)

Teorema CAP (En Base de Datos Distribuidas):

                  Consistencia (C)
                     /      \
               CP   /        \   CA  
             (HBase,       (RDBMS 
             MongoDB)     tradicionales, 
                   /            pero no toleran P)
                  /              \
    Partición (P) ──────────────── Disponibilidad (A)
                        AP
                   (Cassandra, 
                    DynamoDB)

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Qué es la Consistencia Eventual y en qué caso de uso de negocio la aceptarías?
- **Pregunta 2 (Práctica):** Tienes dos microservicios que actualizan el saldo de un usuario al mismo tiempo. ¿Cómo garantizas la consistencia de datos sin un bloqueo estricto en base de datos (Pessimistic Locking)?
- **Pregunta 3 (Arquitectura):** En un sistema distribuido, ¿cómo garantizas que un evento publicado en Kafka/RabbitMQ realmente se procesó si el consumer falla a la mitad?

## 6. Respuestas esperadas
- **Respuesta 1:** Es el modelo donde las réplicas de datos se sincronizan en background. Lo aceptaría para los contadores de "Likes" de YouTube o los Tweets. No importa si durante 5 segundos un usuario ve 1000 likes y otro ve 1005. NUNCA lo aceptaría para el balance de una cuenta bancaria (requiere Consistencia Fuerte/Transacciones ACID).
- **Respuesta 2:** Usando **Optimistic Locking** (Control de concurrencia optimista) a nivel de base de datos. Se añade una columna de `version` o `updated_at` a la tabla `User`. Cuando el servicio A lee el registro, obtiene `version=1`. Al intentar actualizar (`UPDATE users SET balance=balance+10, version=2 WHERE id=1 AND version=1`), si el servicio B ya lo actualizó, la query devolverá 0 filas afectadas (porque la versión ya es 2), y el servicio A sabrá que debe reintentar la operación completa leyendo los nuevos datos.
- **Respuesta 3:** Usando el patrón **Transactional Outbox**. En lugar de publicar en la cola directamente, la aplicación guarda el evento en una tabla de base de datos (`outbox_events`) *en la misma transacción* en la que guarda los datos de negocio. Un proceso separado (ej. Debezium o un cron job) lee la tabla `outbox` y publica los eventos de forma segura en Kafka, marcándolos como procesados. Además, el consumer debe ser idempotente.

## 7. Errores comunes
- **Ignorar latencia de red:** Tratar las llamadas de red (REST/RPC) como si fueran llamadas a funciones locales. No configurar *timeouts* o *circuit breakers*, provocando bloqueos masivos en cadena.
- **Clock Drift:** Asumir que los relojes de todos los servidores del clúster están sincronizados exactamente. Depender del timestamp del servidor (ej. `datetime.now()`) para ordenar eventos distribuidos es un error grave (usar Relojes Lógicos o Vector Clocks).

## 8. Buenas prácticas
- **Idempotencia obligatoria:** En sistemas distribuidos, la red es poco fiable. Un mensaje (como un webhook de pago) puede llegar duplicado (At-least-once delivery). Todos los endpoints y consumers deben poder ejecutarse N veces de forma segura.
- **Health Checks & Observabilidad:** Cada nodo/servicio debe exponer endpoints (ej. `/healthz`) que indiquen no solo si el proceso está vivo, sino si puede acceder a sus dependencias críticas (DB, Cache).

## 9. Trade-offs y decisiones técnicas
**Event-Driven (Pub/Sub) vs RPC/REST (Síncrono)**
- *Event-Driven:* Máximo desacoplamiento. El servicio de Pagos emite "PaymentSucceeded", y los servicios de Facturación y Envíos se enteran. Altamente escalable. Trade-off: Difícil razonar sobre el flujo ("¿Quién envía este correo?"), requiere infraestructura compleja (Kafka), y depurar errores requiere tracing distribuido potente.
- *REST/RPC Síncrono:* Fácil de razonar y documentar (OpenAPI/Swagger). Excelente para lecturas inmediatas (ej. `GET /user/profile`). Trade-off: Acoplamiento temporal (si B está caído, A no puede completar su tarea) y menor tolerancia a picos masivos de tráfico comparado con un sistema basado en colas.
