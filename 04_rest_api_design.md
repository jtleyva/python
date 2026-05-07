# REST API Design & Best Practices

## 1. Explicación técnica
Diseñar una API REST a nivel avanzado trasciende devolver JSON; implica crear contratos fiables, escalables y predecibles. Un buen diseño RESTFul asume que la red falla, los clientes cometen errores, y los datos evolucionan. Se basa en recursos (Resources) y sustantivos (no verbos), usando los métodos HTTP (GET, POST, PUT, PATCH, DELETE) con su semántica correcta (Idempotencia y Safeness).

Aspectos críticos:
- **Idempotencia:** Hacer la misma petición *N* veces tiene el mismo efecto en el servidor que hacerla 1 vez. (PUT, DELETE son idempotentes. POST no lo es).
- **Paginación & Filtrado:** Manejar grandes volúmenes de datos.
- **Rate Limiting:** Proteger la API de abusos.
- **Versionado:** Permitir evolución sin romper clientes antiguos.

## 2. Uso en producción
APIs como las de Stripe, GitHub o Twilio son el estándar de oro. Stripe, por ejemplo, implementa **Idempotency Keys** en las cabeceras HTTP. Cuando hay un problema de red (el cliente envía el pago, pero no recibe respuesta HTTP), el cliente puede reintentar con el mismo Idempotency Key, y Stripe garantiza que no cobrará dos veces.

## 3. Ejemplo práctico (Python)

Implementación con FastAPI demostrando Paginación basada en Cursor (mejor para alto rendimiento/escalabilidad) y manejo correcto de códigos de estado.

```python
from fastapi import FastAPI, HTTPException, Header, Depends
from pydantic import BaseModel
from typing import Optional, List
import uuid

app = FastAPI(title="Senior REST API")

class UserResponse(BaseModel):
    id: str
    username: str
    email: str

class CreateUserRequest(BaseModel):
    username: str
    email: str

# Base de datos simulada
fake_db = {}
idempotency_cache = {}

# Endpoint GET con Cursor Pagination
# Limit-Offset es malo en tablas grandes (O(n)). Cursor es O(1).
@app.get("/v1/users", response_model=List[UserResponse])
async def list_users(limit: int = 10, cursor: Optional[str] = None):
    # Lógica pseudo-código: db.query(User).filter(User.id > cursor).limit(limit)
    # Devolvería los usuarios y el next_cursor en las cabeceras o en el body wrapper.
    pass

# Endpoint POST con Idempotency Key
@app.post("/v1/users", response_model=UserResponse, status_code=201)
async def create_user(
    user_req: CreateUserRequest, 
    x_idempotency_key: str = Header(..., description="Unique key for safe retries")
):
    # Verificamos si la operación ya se procesó
    if x_idempotency_key in idempotency_cache:
        # Devolvemos la respuesta cacheada sin afectar estado
        return idempotency_cache[x_idempotency_key]
        
    # Regla de negocio
    if any(u['email'] == user_req.email for u in fake_db.values()):
        # 409 Conflict es el código correcto para violaciones de unicidad, no 400 o 500
        raise HTTPException(status_code=409, detail="Email already exists")
        
    new_id = str(uuid.uuid4())
    new_user = {"id": new_id, **user_req.dict()}
    fake_db[new_id] = new_user
    
    # Guardamos en caché el resultado (ej. Redis) con TTL
    idempotency_cache[x_idempotency_key] = new_user
    
    return new_user

# Endpoint PATCH (Actualización parcial, diferente de PUT que requiere el objeto completo)
@app.patch("/v1/users/{user_id}", status_code=200)
async def partial_update_user(user_id: str, payload: dict): # simplificado
    if user_id not in fake_db:
        raise HTTPException(status_code=404, detail="User not found")
    # Lógica de update...
    return fake_db[user_id]
```

## 4. Diagrama (texto)

Flujo de Idempotencia en POST:

Client                 Load Balancer / API Gateway                 Backend
  │                               │                                   │
  ├── POST /users (Key: a1b2) ───►│──────────────────────────────────►│ (Key no existe)
  │                               │                                   ├── 1. Procesa
  │                               │                                   ├── 2. Guarda resultado en Redis (Key: a1b2)
  │◄── Network Timeout / Error ───┼◄────────── 201 Created ───────────┤
  │                               │                                   │
  │ (Client Retries)              │                                   │
  ├── POST /users (Key: a1b2) ───►│──────────────────────────────────►│ (Key existe!)
  │                               │                                   ├── 1. Lee Redis
  │◄──────── 201 Created ─────────┼◄────────── 201 Created ───────────┤ (Devuelve directo, no duplica BD)

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Cuál es la diferencia entre PUT y PATCH? ¿Son idempotentes?
- **Pregunta 2 (Práctica):** Tienes un endpoint `GET /transactions` que devuelve 5 millones de registros. Un cliente hace la petición y el servidor colapsa por memoria. ¿Cómo lo rediseñas?
- **Pregunta 3 (Arquitectura):** Estás diseñando la API V2 de tu producto que introduce cambios que rompen retrocompatibilidad. ¿Cómo manejas la transición para clientes que usan V1?

## 6. Respuestas esperadas
- **Respuesta 1:** `PUT` es para reemplazo completo del recurso. Si un objeto tiene 5 campos y envías un PUT con 2, los otros 3 deberían borrarse/anularse. Es idempotente. `PATCH` es para modificaciones parciales; solo actualiza los campos enviados. Matemáticamente no siempre es idempotente (ej. si haces PATCH instruyendo `increment: 1`), pero en la práctica de APIs a menudo se diseña para serlo actualizando valores explícitos.
- **Respuesta 2:** Implementando paginación estricta y obligatoria. Evitaría `OFFSET/LIMIT` para un dataset tan grande porque la base de datos debe escanear todos los registros previos, haciéndose más lento conforme se avanza (O(N)). Usaría **Cursor Pagination** o *Keyset pagination* basado en índices secuenciales (ej. `id > cursor_value`), devolviendo un máximo de ej. 100 registros por llamada junto con un token `next_page`.
- **Respuesta 3:** Usando Versionado en la URL (ej. `/v1/`, `/v2/`) o mediante Cabeceras (ej. `Accept: application/vnd.company.v2+json`). Mantendría ambas versiones vivas. Monitorearía los logs para ver qué clientes siguen usando V1. Notificaría a los clientes (Deprecation Notice), y aplicaría el patrón *Expand and Contract* en la base de datos si el esquema subyacente cambió, permitiendo que ambos endpoints apunten a los mismos datos transformados hasta apagar V1.

## 7. Errores comunes
- **Usar HTTP 200 OK para todo:** Ocultar errores de negocio dentro de cuerpos JSON (`HTTP 200, {"error": "true"}`) arruina el monitoreo (DataDog/Prometheus no detectarán fallos a nivel de red/Load Balancer). Usa los códigos correctos (400, 401, 403, 404, 409, 429).
- **URLs basadas en verbos:** `POST /create_user` o `GET /get_all_users`. (Correcto: `POST /users`, `GET /users`).
- **N+1 problems en serializadores:** Si el endpoint `/users` incluye a los amigos de cada usuario, y el ORM hace una query por cada usuario de la lista al serializar, la API será lentísima. (Usar Eager Loading: `.select_related()` / `.joinedload()`).

## 8. Buenas prácticas
- **HATEOAS (Hypermedia as the Engine of Application State):** Opcional pero avanzado; incluir en el JSON enlaces (`_links`) a las acciones permitidas para ese recurso dependiendo del estado (ej. enlace a `/refund` solo si el estado de pago es pagado).
- **Rate Limiting explícito:** Devolver cabeceras informativas: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` y código 429 cuando se excede.
- **CORS restrictivo:** En producción, no usar `Access-Control-Allow-Origin: *`. Limitar a dominios específicos del Frontend.

## 9. Trade-offs y decisiones técnicas
**REST vs GraphQL vs gRPC**
- *REST:* Mejor para APIs públicas y de consumo genérico. Fácil de cachear en CDNs (GET) y gran ecosistema de herramientas. Trade-off: Over-fetching o Under-fetching de datos.
- *GraphQL:* Perfecto para Frontends complejos que requieren datos jerárquicos exactos en una sola llamada. Trade-off: Muy complejo de asegurar, rate-limitear (las queries pueden ser recursivas), y el caching HTTP estándar no funciona (todo es POST a `/graphql`).
- *gRPC:* Diseñado para comunicación Server-to-Server (microservicios) internos. Usa HTTP/2 y Protobufs binarios. Extremadamente rápido y con tipado estricto. Trade-off: No es human-readable, requiere generar código cliente, y los navegadores web no lo soportan nativamente sin proxies (gRPC-Web).
