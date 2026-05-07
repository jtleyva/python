# Microservices Architecture

## 1. Explicación técnica
Avanzar de un Monolito a Microservicios no es un problema tecnológico, es un problema organizacional y de dominios de negocio (Conway's Law). Los microservicios son servicios desplegables independientemente, modelados en torno a **Business Domains** (Domain-Driven Design / Bounded Contexts). 

La principal complejidad que introducen es que cambias llamadas a funciones en memoria (rápidas, síncronas, transacciones ACID confiables) por llamadas de red (lentas, pueden fallar, timeouts, consistencia eventual).

Para mitigar esto en sistemas distribuidos, se requieren patrones como:
- **API Gateway / BFF (Backend For Frontend):** Punto único de entrada.
- **Service Discovery / Mesh:** (Consul, Istio) Para que los servicios se encuentren.
- **Circuit Breaker:** Evitar fallos en cascada.
- **Event-Driven / Asíncronia:** Desacoplar servicios mediante colas/tópicos (RabbitMQ/Kafka).

## 2. Uso en producción
Amazon empezó como un monolito gigante. Al migrar a microservicios en los 2000s, Jeff Bezos emitió el famoso mandato de API: "Todos los equipos expondrán sus datos y funcionalidades a través de interfaces de servicio... Cualquier equipo que no haga esto será despedido". Esto permitió escalar masivamente los equipos, ya que el equipo de "Carrito" podía hacer deploy 50 veces al día sin esperar al equipo de "Pagos". El costo: infraestructura masivamente compleja.

## 3. Ejemplo práctico (Python)

Patrón **Circuit Breaker** usando la librería `pybreaker`. Previene que un microservicio espere indefinidamente a otro que está caído.

```python
import pybreaker
import requests
from fastapi import FastAPI, HTTPException

app = FastAPI()

# Configura un Circuit Breaker: si fallan 3 llamadas seguidas, se "abre" el circuito.
# Pasados 60 segundos, intentará (Half-Open) una petición de prueba.
payment_breaker = pybreaker.CircuitBreaker(fail_max=3, reset_timeout=60)

class PaymentServiceDownException(Exception):
    pass

@payment_breaker
def call_payment_microservice(user_id: str, amount: float):
    """
    Simula una llamada HTTP asíncrona síncrona a un microservicio de pagos.
    """
    try:
        response = requests.post(
            "http://payment-service/api/charge", 
            json={"user": user_id, "amount": amount},
            timeout=2.0  # Timeout crítico
        )
        response.raise_for_status()
        return response.json()
    except requests.RequestException as e:
        # Registramos el error. El circuit breaker cuenta este fallo.
        raise PaymentServiceDownException(str(e))

@app.post("/checkout")
def checkout(user_id: str, amount: float):
    try:
        result = call_payment_microservice(user_id, amount)
        return {"status": "success", "transaction": result}
        
    except pybreaker.CircuitBreakerError:
        # El circuito está ABIERTO. Fallamos rápido sin saturar la red ni esperar timeouts.
        # Aquí se aplicaría lógica de Fallback (ej. guardar en cola para proceso en background)
        raise HTTPException(
            status_code=503, 
            detail="Payment service is temporarily unavailable. We saved your order."
        )
    except PaymentServiceDownException:
        raise HTTPException(status_code=502, detail="Error charging payment")
```

## 4. Diagrama (texto)

Circuit Breaker Pattern:

         [ Microservice A (Order) ] ──────► [ Microservice B (Payment) ]
                    │
                    ▼
          State: CLOSED (Normal)
          1. Llama a B (Falla / Timeout)
          2. Llama a B (Falla)
          3. Llama a B (Falla) ─── Threshold alcanzado!
                    │
                    ▼
          State: OPEN (Circuito Abierto)
          4. Llama a B ── X (Bloqueado instantáneamente por Breaker, lanza error)
                    │
              (Wait timeout)
                    │
                    ▼
          State: HALF-OPEN
          5. Intenta 1 llamada a B. 
             Si éxito -> Pasa a CLOSED. 
             Si falla -> Vuelve a OPEN.

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Qué es la "Conway's Law" y cómo se relaciona con la arquitectura de microservicios?
- **Pregunta 2 (Arquitectura):** Tienes un monolito y necesitas extraer el módulo de "Notificaciones" a un microservicio. ¿Qué patrón de migración utilizarías para hacerlo sin downtime?
- **Pregunta 3 (Práctica):** En un sistema de microservicios, el Servicio A llama al B, y B llama al C. El usuario se queja de un error 500 intermitente. ¿Cómo encuentras dónde ocurre el error?

## 6. Respuestas esperadas
- **Respuesta 1:** "Las organizaciones diseñan sistemas que copian sus estructuras de comunicación". Significa que si tienes equipos divididos en Frontend, Backend y DBAs, tu arquitectura tenderá a ser monolítica por capas funcionales. Microservicios exige equipos *Cross-funcionales* (DevOps, QA, Backend, Frontend) enfocados en un dominio (ej. Equipo de Pagos, Equipo de Catálogo).
- **Respuesta 2:** Usaría el patrón **Strangler Fig (Higera Estranguladora)**. Construiría el nuevo microservicio de Notificaciones. Pondría un API Gateway / Proxy inverso delante de ambos. Al principio, el Gateway enruta `/notifications` al monolito. Progresivamente (canary release), empiezo a enrutar peticiones al nuevo microservicio, hasta que el módulo dentro del monolito queda sin uso ("estrangulado") y se puede borrar.
- **Respuesta 3:** Es imposible hacerlo mirando logs aislados. Se requiere **Distributed Tracing** (ej. Jaeger, AWS X-Ray, OpenTelemetry). El API Gateway debe generar un `Trace ID` único y pasarlo a través de cabeceras HTTP (ej. `X-B3-TraceId` o `traceparent`). Todos los servicios (A, B, C) inyectan este ID en sus logs, permitiendo reconstruir la cascada completa y ver latencias exactas y dónde falló la llamada.

## 7. Errores comunes
- **Distributed Monolith:** Separar servicios por función (Servicio de Base de Datos, Servicio de Reglas, Servicio Web) pero que todos tengan que llamarse de forma síncrona para servir una petición. Si uno cae, todo cae. Combina los peores aspectos del monolito y de lo distribuido.
- **Bases de Datos Compartidas:** Múltiples microservicios leyendo y escribiendo directamente en la misma base de datos relacional. Viola el principio de autonomía (un cambio de esquema rompe servicios ajenos). *Regla: Cada microservicio debe ser dueño absoluto de sus datos.*
- **No prepararse para el fracaso (Timeouts):** Usar clientes HTTP (como `requests` en Python) sin un timeout explícito, causando que hilos/procesos se queden colgados esperando para siempre.

## 8. Buenas prácticas
- **API Gateway:** Úsalo para centralizar Auth (Autenticación/Autorización), Rate Limiting, CORS y Enrutamiento, quitando esta responsabilidad a los microservicios.
- **Comunicación Asíncrona:** Preferir eventos (Kafka, SQS/SNS) sobre HTTP (REST/gRPC) para comunicación entre servicios siempre que sea posible para aumentar la resiliencia (Choreography vs Orchestration).
- **Graceful Degradation:** Diseñar la UI/API para que si el microservicio de "Recomendaciones" falla, la página principal siga cargando (quizás mostrando productos por defecto), en lugar de dar un error 500 completo.

## 9. Trade-offs y decisiones técnicas
**Monolito Modular vs Microservicios**
- *Monolito Modular:* Un solo repositorio de código y despliegue, pero estrictamente dividido internamente con dominios desacoplados (Hexagonal). Excelente balance: fácil de desplegar, refactorizaciones seguras (mismo tipado), transacciones atómicas simples. Trade-off: Límite teórico de escalamiento organizativo (problemas con repos de cientos de developers).
- *Microservicios:* Aislamiento tecnológico perfecto (un equipo usa Python/FastAPI, otro Go, otro Node). Despliegues independientes. Trade-off: *Data Consistency* (Consistencia Eventual) se vuelve una pesadilla (Sagas pattern, 2PC), latencia de red añadida, y enorme sobrecarga operativa (DevOps). **Consejo Senior: Siempre empieza con un Monolito Modular.**
