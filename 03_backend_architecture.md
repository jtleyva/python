# Backend Architecture (Clean & Hexagonal)

## 1. Explicación técnica
A nivel arquitectónico, el mayor reto es evitar el **acoplamiento**. Si tu lógica de negocio (cómo calcular impuestos) está mezclada con tu framework web (Django/FastAPI) o tu ORM (SQLAlchemy), actualizar de versión, cambiar de base de datos, o testear la lógica sin levantar la DB se vuelve un infierno.

Aquí entran la **Clean Architecture** (Robert C. Martin) y la **Hexagonal Architecture / Ports and Adapters** (Alistair Cockburn). Ambas predican lo mismo: 
- El dominio (reglas de negocio) está en el centro, puro, sin dependencias externas.
- Las capas externas (UI, Frameworks, DBs) dependen del dominio, pero el dominio no sabe nada de ellas.
- La comunicación se hace a través de Interfaces (Puertos) y la implementación concreta (Adaptadores).

### BONUS: Hexagonal Architecture vs Clean Architecture
- **Clean Architecture** es más prescriptiva (Entities, Use Cases, Interface Adapters, Frameworks). Organizada en anillos concéntricos.
- **Hexagonal Architecture** es más conceptual (Core, Ports de entrada/salida, Adapters). Es agnóstica a cuántas capas internas tienes, enfocándose en aislar el Core del mundo exterior (APIs, UI, DBs, Message Brokers). En la práctica en Python, solemos mezclarlas: un "Core" con casos de uso, repositorios abstractos, y adaptadores HTTP/DB.

## 2. Uso en producción
Empresas de escala enterprise utilizan estas arquitecturas para asegurar la longevidad del código. Un backend de eCommerce puede empezar con un adaptador de Stripe para pagos. Cuando el negocio exige integrar MercadoPago para LATAM, solo se crea un nuevo adaptador que cumple con el puerto (interfaz) existente, sin alterar ni una línea del motor de checkout. Además, permite correr suites de 10,000 tests unitarios en segundos al "mockear" la base de datos usando adaptadores "in-memory".

## 3. Ejemplo práctico (Python)

Implementación de un Use Case con Arquitectura Hexagonal.

```python
from dataclasses import dataclass
from typing import Protocol
import uuid

# --- DOMAIN LAYER (Entities & Value Objects) ---
# Totalmente agnóstico de frameworks o BD.
@dataclass
class Order:
    id: str
    customer_id: str
    total_amount: float
    status: str = "PENDING"

    def approve(self):
        if self.total_amount <= 0:
            raise ValueError("Amount must be positive")
        self.status = "APPROVED"

# --- PORTS (Interfaces / Abstracciones) ---
class OrderRepository(Protocol):
    def save(self, order: Order) -> None: ...
    def get_by_id(self, order_id: str) -> Order: ...

class PaymentGateway(Protocol):
    def charge(self, customer_id: str, amount: float) -> bool: ...

# --- APPLICATION LAYER (Use Cases) ---
# Orquesta el flujo, usa puertos, no sabe de implementaciones.
class ProcessOrderUseCase:
    def __init__(self, repo: OrderRepository, payment_gateway: PaymentGateway):
        self.repo = repo
        self.payment = payment_gateway

    def execute(self, order_id: str) -> Order:
        order = self.repo.get_by_id(order_id)
        
        success = self.payment.charge(order.customer_id, order.total_amount)
        if not success:
            order.status = "FAILED"
        else:
            order.approve()
            
        self.repo.save(order)
        return order

# --- ADAPTERS (Infraestructura / Implementaciones Concretas) ---
class PostgresOrderRepository:
    def __init__(self, session): # SQLAlchemy session
        self.session = session

    def save(self, order: Order) -> None:
        # Mapea Entidad de Dominio -> Modelo de ORM
        # Lógica de inserción DB...
        print(f"Saving order {order.id} to Postgres")

    def get_by_id(self, order_id: str) -> Order:
        # Query a DB, mapeo inverso...
        return Order(id=order_id, customer_id="c1", total_amount=100.0)

class StripePaymentGateway:
    def charge(self, customer_id: str, amount: float) -> bool:
        # Llamada HTTP a la API de Stripe
        print(f"Charging {amount} via Stripe")
        return True

# --- ENTRYPOINT (FastAPI / CLI / Script) ---
# La capa más externa (Dependency Injection Container / Composers)
# def handle_api_request():
#     repo = PostgresOrderRepository(db_session)
#     payment = StripePaymentGateway()
#     use_case = ProcessOrderUseCase(repo, payment)
#     result = use_case.execute("123")
```

## 4. Diagrama (texto)

```
                            │   (Adapters / Infra)                 
                            │   HTTP Controllers (FastAPI)         
   Incoming Request  ───────┼─────────────────────────┐             
                            │                         ▼             
                            │      [ Use Case ] ◄── (Primary Port / Input)
   (Domain / Core)          │           │                           
        Entities  ──────────┼───────────┤                           
                            │           ▼                           
                            │      [ Repository ] ──► (Secondary Port / Output)
   (Adapters / Infra)       │           │                           
                            │           ▼                           
                            │    [ SQLAlchemy Adapter ] ──► Postgres
                            │    [ Stripe Adapter ]     ──► External API
```

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Cuál es la regla de dependencia principal en Clean Architecture?
- **Pregunta 2 (Práctica):** ¿Dónde colocarías la validación de un formato de email y dónde colocarías la validación de que un usuario no tiene deudas pendientes?
- **Pregunta 3 (Arquitectura):** Si tu aplicación usa modelos de Django/SQLAlchemy (`models.py`) extendiendo la clase `Model` del framework, ¿deberías usar esas clases como tus Entidades de Dominio?

## 6. Respuestas esperadas
- **Respuesta 1:** La Regla de Dependencia establece que las dependencias del código fuente solo pueden apuntar **hacia adentro**. El código en las capas internas (Dominio) no puede saber nada sobre el código en las capas externas (Bases de datos, Frameworks web, UI).
- **Respuesta 2:** El formato del email es una regla intrínseca de los datos (Value Object / Entidad) y se valida en la capa de Dominio o en la entrada de la API (Capa de Aplicación/DTOs con Pydantic). Que el usuario no tenga deudas es una regla de negocio que requiere consultar otros agregados o servicios, por lo que pertenece al *Use Case* (Capa de Aplicación).
- **Respuesta 3:** Idealmente, no. Los modelos de ORM están acoplados fuertemente al framework y al diseño de la base de datos relacional (Active Record pattern). En arquitecturas estrictas, se deben mapear los Modelos del ORM (Infraestructura) a Entidades Puras de Python (Dataclasses) para el Dominio (Repository Pattern). Esto evita que detalles de la DB (como lazy loading) "sangren" a la lógica de negocio.

## 7. Errores comunes
- **Anti-pattern: "Anemic Domain Model".** Entidades que solo son clases con getters/setters sin comportamiento, delegando toda la lógica condicional a servicios externos masivos. Las reglas intrínsecas de una entidad deben vivir dentro de la entidad.
- **Filtración de Framework:** Devolver objetos de respuesta HTTP (`Response`, `JSONResponse`) desde dentro del Caso de Uso. El Caso de uso debe devolver DTOs (Data Transfer Objects) o diccionarios nativos.
- **Mapeo infinito:** En proyectos pequeños/CRUD, aplicar Clean Architecture requiere crear interfaces, DTOs, entidades y modelos ORM. Esto resulta en mucho boilerplate (código repetitivo) inútil.

## 8. Buenas prácticas
- **Separation of Concerns:** Divide la estructura de carpetas claramente (ej. `domain/`, `application/`, `infrastructure/`).
- **Data Transfer Objects (DTOs):** Usa herramientas como `pydantic` en el adaptador primario (FastAPI) para validar datos de entrada antes de que toquen tus Casos de Uso.
- **Testing Pyramid:** Al tener el dominio aislado, puedes tener un 80% de tests unitarios súper rápidos instanciando los Use Cases con *In-memory Repositories* (mocks/fakes) sin levantar contenedores de base de datos.

## 9. Trade-offs y decisiones técnicas
**Clean Architecture vs MVC Framework-Centric (Django/Rails)**
- *MVC / Framework-centric:* Excelente Time-to-market. Perfecto para startups probando Product-Market Fit o aplicaciones fuertemente orientadas a CRUD. Trade-off: El código está acoplado al framework. Si el negocio crece y las reglas se complican, los "Fat Models" se vuelven inmanejables e in-testeables.
- *Clean/Hexagonal:* Excelente para aplicaciones de alta complejidad de negocio (Fintech, Core logic). Permite iterar reglas sin miedo a romper la DB. Trade-off: Curva de aprendizaje alta para el equipo, más verbosidad de código, y ralentiza el desarrollo inicial de features simples.
