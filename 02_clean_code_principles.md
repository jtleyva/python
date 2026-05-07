# Clean Code Principles

## 1. Explicación técnica
Clean Code a nivel Senior no es solo "nombres de variables bonitos", es diseñar sistemas que toleren el cambio. Los principios **SOLID** son la base:
- **S**ingle Responsibility (SRP): Una clase/módulo debe tener una única razón para cambiar.
- **O**pen/Closed (OCP): Abierto a extensión (añadir nuevas features), cerrado a modificación (no tocar el código core existente).
- **L**iskov Substitution (LSP): Las subclases deben comportarse como sus clases base sin romper la lógica.
- **I**nterface Segregation (ISP): Interfaces pequeñas y específicas mejor que una gigantesca. En Python (duck typing), esto aplica usando Abstract Base Classes (ABCs) o protocolos (Protocols).
- **D**ependency Inversion (DIP): Depender de abstracciones (interfaces/tipos), no de implementaciones concretas.

## 2. Uso en producción
Imagina el sistema de pagos de Uber. Tienen múltiples proveedores: Stripe, PayPal, Braintree. Si el código viola OCP y DIP, cada vez que añaden un proveedor nuevo, tendrían que modificar cientos de condicionales `if provider == 'stripe': ...`. Usando Clean Code, inyectan el proveedor concreto que cumple con una interfaz `PaymentGateway`, permitiendo intercambiar proveedores en runtime sin tocar el core de negocio.

## 3. Ejemplo práctico (Python)

Aplicando Inversión de Dependencias (DIP) y OCP usando `Protocol` de Python (Tipado estructural).

```python
from typing import Protocol
import logging

logger = logging.getLogger(__name__)

# 1. Definimos el contrato (Abstracción)
class NotificationSender(Protocol):
    def send(self, user_id: str, message: str) -> bool:
        ...

# 2. Implementaciones Concretas
class EmailService:
    def __init__(self, smtp_client):
        self.smtp = smtp_client

    def send(self, user_id: str, message: str) -> bool:
        logger.info(f"Sending Email to {user_id}: {message}")
        # Lógica de red...
        return True

class SMSService:
    def __init__(self, twilio_client):
        self.twilio = twilio_client

    def send(self, user_id: str, message: str) -> bool:
        logger.info(f"Sending SMS to {user_id}: {message}")
        # Lógica de red...
        return True

# 3. Core Business Logic que depende de la abstracción, NO de la implementación
class UserOnboardingWorkflow:
    # Inyección de dependencias a través del constructor
    def __init__(self, notifier: NotificationSender):
        self.notifier = notifier

    def execute(self, user_id: str) -> None:
        # Lógica de onboarding...
        success = self.notifier.send(user_id, "Welcome to the platform!")
        if not success:
            logger.error("Failed to notify user during onboarding")

# Uso
# email_notifier = EmailService(smtp_client=...)
# workflow = UserOnboardingWorkflow(notifier=email_notifier)
# workflow.execute("user_123")
```

## 4. Diagrama (texto)

DIP (Dependency Inversion Principle) Visualizado:

[UserOnboardingWorkflow]  ─── depende de ───> <<NotificationSender>> (Protocol)
                                                        ▲
                                                        │ cumple con
                          ┌─────────────────────────────┴─────────────────┐
                          │                                               │
                   [EmailService]                                   [SMSService]

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Qué es la Inyección de Dependencias y por qué no simplemente instanciar las dependencias dentro de la clase?
- **Pregunta 2 (Práctica):** En una Code Review, ves una función de 300 líneas que lee de S3, parsea un CSV, lo valida y guarda en base de datos. ¿Qué principios viola y cómo la refactorizarías?
- **Pregunta 3 (Arquitectura):** En Python no hay interfaces estrictas como en Java. ¿Cómo garantizas por diseño que diferentes servicios implementen el mismo contrato?

## 6. Respuestas esperadas
- **Respuesta 1:** Si instanciamos dependencias dentro de la clase (ej. `self.db = MySQLClient()`), acoplamos la clase a esa implementación específica. Esto hace que el código sea casi imposible de testear de forma aislada (no podemos hacer mock/stub fácilmente) y viola el OCP si queremos cambiar a PostgreSQL en el futuro. Inyectar dependencias permite pasar "dobles de prueba" durante el testing e intercambiar implementaciones dinámicamente.
- **Respuesta 2:** Viola el Principio de Responsabilidad Única (SRP). Lo refactorizaría separándolo en clases/funciones con responsabilidades claras: un `StorageReader` (infraestructura), un `CSVParser` (transformación), un `DataValidator` (reglas de negocio), y un `DatabaseRepository` (persistencia). Un orquestador principal coordinaría estas piezas.
- **Respuesta 3:** Usando el módulo `abc` (Abstract Base Classes) que forzará un error en tiempo de instanciación si faltan métodos, o en Python moderno, usando `typing.Protocol` (Duck Typing estructural) junto con un linter estático como `mypy` para que el CI falle si una clase no cumple el contrato esperado sin necesidad de herencia explícita.

## 7. Errores comunes
- **Anti-pattern:** "God Classes" o módulos "Utils/Helpers". Clases o archivos que agrupan funciones aleatorias sin cohesión. (Ej. `utils.py` que manda emails y formatea fechas).
- **Anti-pattern:** Abuso de herencia. Preferir *Composición sobre Herencia*. Jerarquías de herencia profundas (>3 niveles) hacen el código frágil y difícil de rastrear (LSP violation risk).

## 8. Buenas prácticas
- **Clean code:** Nombres reveladores de intención. No comentes lo que el código *hace* (eso lo dice la sintaxis), comenta *por qué* lo hace (contexto de negocio o decisiones inusuales).
- **Escalabilidad (del equipo):** Aplicar SOLID permite que varios desarrolladores trabajen en distintas implementaciones (ej. distintos proveedores de pago) en paralelo sin generar conflictos de merge masivos en archivos core.
- **Mantenibilidad:** Return early. Evita la indentación profunda (Arrow Anti-Pattern) fallando rápido y devolviendo errores al principio de la función (Guard Clauses).

## 9. Trade-offs y decisiones técnicas
**Abstracción vs Premature Optimization / Over-engineering**
- *Trade-off:* Aplicar SOLID a rajatabla en un script pequeño o prueba de concepto (PoC) es *over-engineering*. Crea demasiadas capas de indirección que dificultan la lectura ("Spaghetti code de interfaces").
- *Decisión:* La abstracción debe ser introducida cuando se prevé un cambio real (Regla de tres: no lo abstraigas hasta que tengas al menos 3 casos de uso) o cuando facilita la testeabilidad de lógicas de negocio críticas.
