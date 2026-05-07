# Storage Solutions (S3, RDS, DynamoDB)

## 1. Explicación técnica
Las arquitecturas maduras asumen un enfoque **Polyglot Persistence**: usar el sistema de almacenamiento adecuado para la carga de trabajo adecuada, en lugar de meter todo en una base de datos relacional masiva.

Opciones core de AWS:
- **Amazon S3 (Object Storage):** Almacenamiento infinito, durabilidad extrema (99.999999999%), inmutable. Ideal para archivos (imágenes, CSV, backups) o Data Lakes. Soporta Versioning y Lifecycle Policies (pasar de "Hot" a Glacier "Cold").
- **Amazon RDS / Aurora (RDBMS):** Para datos relacionales estructurados, transacciones complejas (ACID) e integridad referencial (PostgreSQL / MySQL). Aurora es la versión cloud-native de Amazon, super rápida y con separación de cómputo y almacenamiento (Storage Auto-Scaling).
- **Amazon DynamoDB (NoSQL Key-Value):** Base de datos serverless masivamente escalable. Escala de ceros a petabytes. Ofrece latencias milisegundas garantizadas a cualquier escala, SI diseñas la partición de datos correctamente. Carece de `JOIN`s complejos.

## 2. Uso en producción
Instagram usa S3 para almacenar petabytes de fotos originales. Usa RDS (Postgres/MySQL) para el motor de facturación y perfiles críticos de cuentas (donde se necesitan transacciones ACID seguras). Y utiliza DynamoDB o Cassandra (NoSQL) para los feeds de inicio (Home feed) y los contadores de likes, donde importa más el rendimiento a alta velocidad de lectura que las relaciones complejas.

## 3. Ejemplo práctico (Python)

Uso de SQLAlchemy (ORM) robusto para conectarse a Aurora PostgreSQL con buenas prácticas de manejo de pools de conexiones y timeouts.

```python
import os
import contextlib
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from sqlalchemy.exc import OperationalError

# Buenas Prácticas Senior: No usar solo "postgresql://", usar configuraciones de pool
# - pool_size: Cuántas conexiones mantener permanentemente abiertas
# - max_overflow: Cuántas conexiones extra permitir temporalmente en picos
# - pool_timeout: Tiempo máx de espera para adquirir conexión antes de fallar
# - connect_args (statement_timeout): No esperar eternamente si RDS está colgado

DATABASE_URL = os.environ.get("DATABASE_URL")

engine = create_engine(
    DATABASE_URL,
    pool_size=10,
    max_overflow=20,
    pool_timeout=5,
    pool_recycle=1800, # Evitar que conexiones antiguas (>30 min) se corrompan (MySQL/AWS Network Drops)
    connect_args={
        "options": "-c statement_timeout=5000" # Mata queries de > 5 segundos a nivel BD
    }
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

@contextlib.contextmanager
def get_db_session():
    """Generador de contexto para garantizar la liberación de la conexión."""
    session: Session = SessionLocal()
    try:
        yield session
        # Si no hubo excepciones en el bloque 'with', podemos hacer commit de lógica (opcional)
        # o delegar el commit manual al servicio.
    except OperationalError as e:
        session.rollback()
        # Log del error y relanzar como un error 503 custom
        raise RuntimeError("Database unavailable") from e
    except Exception:
        session.rollback()
        raise
    finally:
        # ¡CRÍTICO! Devuelve la conexión al connection pool. 
        # Si no haces close, agotarás el pool en minutos bajo carga.
        session.close()

# --- Uso en caso de uso de negocio ---
# def process_transfer(user_id, amount):
#     with get_db_session() as db:
#         user = db.query(User).with_for_update().filter_by(id=user_id).first() # Row-level lock
#         user.balance -= amount
#         db.commit()
```

## 4. Diagrama (texto)

Polyglot Persistence Pattern:

                  ┌──► Perfiles y Contratos ────► [ Amazon RDS Postgres ] (ACID, Relaciones)
                  │ 
[ API Backend ] ──┼──► Documentos / Imágenes ───► [ Amazon S3 ] (Blob, Barato)
                  │
                  └──► Eventos IoT / Sesiones ──► [ Amazon DynamoDB ] (Velocidad, Key-Value)

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Qué es un índice compuesto (Composite Index) en Postgres y en qué se diferencia del Partition Key (PK) y Sort Key (SK) en DynamoDB?
- **Pregunta 2 (Arquitectura):** Vas a diseñar una API donde el usuario lee su carrito de la compra 50 veces por sesión, y lo modifica 1 vez. La DB relacional se está saturando. ¿Cómo alivias la carga usando patrones de almacenamiento en AWS?
- **Pregunta 3 (Práctica):** Tienes un job de Python corriendo que hace un `SELECT * FROM users` (con 5 millones de registros) en SQLAlchemy usando `.all()`. El contenedor ECS falla por Out of Memory (OOM Killed). ¿Cómo procesas esos registros de forma segura?

## 6. Respuestas esperadas
- **Respuesta 1:** En Postgres, un índice compuesto involucra múltiples columnas (ej. `user_id, status`) indexadas mediante un B-Tree, permitiendo búsquedas rápidas (ej. `WHERE user_id=1 AND status='active'`). En DynamoDB, el PK es obligatorio y determina físicamente (hash) en qué nodo del clúster se guarda el dato (distribución). El SK permite ordenar los datos dentro de ese mismo nodo y permite búsquedas de rango (`starts_with`, `between`). En DynamoDB NO puedes consultar de forma eficiente si no provees al menos el PK exacto.
- **Respuesta 2:** Usaría el patrón **Cache-Aside**. Colocaría Amazon ElastiCache (Redis/Memcached) frente a RDS. Cuando el cliente hace un `GET /cart`, la API busca primero en Redis (latencia sub-milisegundo). Si está allí (Cache Hit), devuelve la respuesta sin tocar RDS. Si no está (Cache Miss), lee de RDS, guarda en Redis con un TTL (Time To Live), y responde. Las escrituras (`POST /cart`) siempre van a la base de datos principal e invalidan o actualizan inmediatamente la caché (Write-Through/Write-Around).
- **Respuesta 3:** `.all()` carga los 5 millones de objetos ORM en la RAM simultáneamente, agotando la memoria. En SQLAlchemy usaría iteradores de servidor (Server-Side Cursors) usando `.yield_per(1000)`. Esto le dice al driver de Postgres que entregue los resultados en lotes pequeños (chunks), permitiendo que Python procese y descarte los objetos de la RAM progresivamente con un consumo de memoria plano. 

## 7. Errores comunes
- **Guardar BLOBs en Base de Datos:** Guardar archivos PDF o Imágenes grandes en columnas `BYTEA` o `BLOB` de Postgres/MySQL. Esto engorda la base de datos, ralentiza los backups, ensucia los logs y encarece terriblemente el coste (RDS es caro). Los archivos van en S3, la base de datos solo guarda el enlace (URL pre-firmada) hacia ese objeto.
- **Hot Partitions en DynamoDB:** Usar un Partition Key (PK) que reciba demasiado tráfico concentrado, como una fecha truncada (`PK="2023-10-15"`). AWS dirige el tráfico hacia el nodo asociado a esa fecha y lo sobrecarga, mientras los nodos de otras fechas están inactivos. El PK debe tener **Alta Cardinalidad** (ej. un `user_id` único o UUID).
- **N+1 Selects Problem:** En Django ORM o SQLAlchemy, consultar una lista de 100 Posts y luego iterar en un loop por el Autor del post (`post.author.name`). Por defecto, el ORM dispara 1 query para los posts, y luego 100 queries extra para los autores. (Anti-patrón masivo). Solución: usar `joinedload()` o `select_related` para hacer un `SQL JOIN` inicial y resolverlo en 1 sola petición.

## 8. Buenas prácticas
- **Read Replicas (Escalabilidad en BD):** RDS permite crear réplicas de solo lectura. Configura la aplicación en Python para rutear todas las operaciones `GET` (lectura pesada, reportes) hacia la URL del clúster de réplica, reservando el Endpoint Master/Writer exclusivo para `INSERT`, `UPDATE` y `DELETE`.
- **Soft Deletes:** Rara vez eliminar información financiera o importante con un `DELETE FROM`. Usar Soft Delete: añadir un campo `deleted_at=TIMESTAMP` y actualizar ese campo. Permite auditorías, recuperaciones y es más amigable con bloqueos de base de datos.
- **Infrastructure:** S3 Lifecycle Policies. Configura S3 para mover archivos antiguos (> 30 días) a la capa "S3 Infrequent Access" o "S3 Glacier", reduciendo los costes de almacenamiento corporativo en un 70%.

## 9. Trade-offs y decisiones técnicas
**RDBMS (Postgres) vs NoSQL (DynamoDB/MongoDB)**
- *Postgres (RDS):* Excelente flexibilidad para consultar los datos de cualquier forma posible gracias a SQL y JOINS. Fácil de pivotar a nivel de producto. Trade-off: Escalar verticalmente (máquinas más grandes) tiene un límite físico costoso, y escalar horizontalmente (Sharding/Clustering transaccional) es extremadamente complejo. 
- *DynamoDB (NoSQL):* Escalamiento horizontal ilimitado nativo y barato si se optimizan lecturas. Trade-off: Debes saber *exactamente* cómo vas a consultar los datos (Access Patterns) antes de diseñar la tabla (Single-Table Design). Cambiar los patrones de acceso en el futuro a menudo requiere un esfuerzo masivo de re-indexado o re-escritura (ETL).
