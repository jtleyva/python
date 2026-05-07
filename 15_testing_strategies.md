# Testing Strategies in Python (Mocking & Pytest)

## 1. Explicación técnica
A nivel Senior, no escribimos tests solo para alcanzar el "100% de cobertura de código", escribimos tests para diseñar mejores contratos (TDD), prevenir regresiones (regressions), y tener la confianza arquitectónica para refactorizar sin miedo.

La **Pirámide de Pruebas**:
- **Tests Unitarios (Base de la pirámide):** Rápidos, aislados. Prueban la lógica de negocio pura. Si dependes de la red, el disco o la DB, *no es* un test unitario. Mocks obligatorios.
- **Tests de Integración (Centro):** Prueban cómo interactúan tus módulos con el mundo exterior (bases de datos reales, colas o librerías de AWS reales/simuladas).
- **Tests E2E (Pico de la pirámide):** Lentos y frágiles. Levantan toda la aplicación de principio a fin (API -> DB -> Terceros) simulando el cliente final. Mantenerlos al mínimo necesario.

Librerías estándar:
- `pytest`: Motor de testing superior al `unittest` nativo por su legibilidad (usar simple `assert`), Fixtures y parametrización.
- `unittest.mock` (`patch`): Para reemplazar objetos pesados o de red con simulaciones.
- `moto`: Librería específica que actúa como un servidor de AWS falso para probar Lambdas o S3 localmente.

## 2. Uso en producción
Imagina un microservicio que procesa un reembolso en Stripe, actualiza el saldo en la BD y envía un recibo PDF a S3. Un test unitario "mockeará" todas las llamadas HTTP a Stripe y S3, pero no detectará si olvidaste hacer `db.commit()`. Un test de integración usará un contenedor de Postgres real efímero (TestContainers) en el CI, interceptará Stripe usando `responses`, y usará `moto` para validar que el archivo fue enviado al "S3 falso", validando la transacción entera en milisegundos.

## 3. Ejemplo práctico (Python)

Ejemplo avanzado usando Pytest Fixtures, Mocking de librerías externas y `moto` para AWS.

```python
import pytest
import boto3
from unittest.mock import patch, MagicMock
from moto import mock_s3

# --- CÓDIGO A PROBAR (src/invoice_service.py) ---
def generate_and_upload_invoice(user_id: str, amount: float) -> str:
    """Regla de negocio que interactúa con S3 y pasarela de pagos simulada"""
    # 1. Simulación de lógica de pago externa
    import requests
    response = requests.post("https://api.payment.com/charge", json={"id": user_id, "amount": amount})
    if response.status_code != 200:
        raise ValueError("Payment failed")

    # 2. Subida a AWS S3
    s3 = boto3.client("s3", region_name="us-east-1")
    file_key = f"invoices/{user_id}.txt"
    s3.put_object(Bucket="my-billing-bucket", Key=file_key, Body=f"Paid: {amount}")
    return file_key


# --- TESTS (tests/test_invoice.py) ---

# Fixture de AWS S3 usando 'moto'. Intercepta llamadas Boto3 para que sean locales y en memoria.
@pytest.fixture
def mock_aws_s3():
    with mock_s3():
        # Setup: El entorno AWS falso necesita que le creemos el bucket antes de usarlo
        s3 = boto3.client("s3", region_name="us-east-1")
        s3.create_bucket(Bucket="my-billing-bucket")
        yield s3  # Devuelve el control al test
        # Teardown automático (la memoria se borra al acabar el context manager)

# Patching (Mocking) de la librería 'requests'
@patch("src.invoice_service.requests.post")
def test_generate_and_upload_invoice_success(mock_post, mock_aws_s3):
    # Arrange (Preparar)
    # Configuramos el mock de red para simular un 200 OK
    mock_response = MagicMock()
    mock_response.status_code = 200
    mock_post.return_value = mock_response

    # Act (Actuar)
    result_key = generate_and_upload_invoice("user_99", 50.0)

    # Assert (Validar)
    assert result_key == "invoices/user_99.txt"
    
    # Validamos que se llamó a la API correctamente
    mock_post.assert_called_once_with(
        "https://api.payment.com/charge", 
        json={"id": "user_99", "amount": 50.0}
    )

    # Validamos usando el S3 de moto que el archivo realmente existe (Test de integración ligero)
    obj = mock_aws_s3.get_object(Bucket="my-billing-bucket", Key=result_key)
    assert obj["Body"].read().decode("utf-8") == "Paid: 50.0"

@patch("src.invoice_service.requests.post")
def test_payment_failure_raises_error(mock_post, mock_aws_s3):
    # Arrange: Simulamos fallo de red
    mock_post.return_value.status_code = 500

    # Act & Assert: Validamos que la excepción explota y el mensaje es correcto
    with pytest.raises(ValueError, match="Payment failed"):
        generate_and_upload_invoice("user_99", 50.0)
```

## 4. Diagrama (texto)

Estrategia de Test de Integración:

[ Test Runner (Pytest) ] 
          │
          ├── [ 1. Setup ] ──► Levanta Postgres en Docker (Testcontainers) / Moto(S3)
          │
          ├── [ 2. Act   ] ──► Llama a Endpoint FastAPI (TestClient)
          │
          ├── [ 3. Mock  ] ──► Intercepta llamadas HTTP a Twilio/Stripe
          │
          ├── [ 4. Assert] ──► ¿Devolvió JSON 201? ¿Se insertó en Postgres? ¿Existe en Moto S3?
          │
          └── [ 5. Tear  ] ──► Destruye el Docker DB y limpia mocks

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Qué es un Test Double (Mocks, Stubs, Fakes, Spies)? ¿Cuál es la diferencia entre un Mock y un Stub?
- **Pregunta 2 (Práctica):** ¿Por qué la gente usa `@pytest.fixture` en lugar de la herencia tradicional de clases `setUp()` y `tearDown()` de la librería nativa `unittest`?
- **Pregunta 3 (Arquitectura):** Tienes código acoplado fuertemente a `datetime.now()` que expira licencias de usuarios. Los tests fallan según la hora en la que los ejecutas (Flaky tests). ¿Cómo solucionas esto?

## 6. Respuestas esperadas
- **Respuesta 1:** Un *Test Double* es cualquier objeto falso usado en pruebas. Un **Stub** es un objeto tonto que provee respuestas enlatadas pre-programadas ("Si te preguntan X, responde Y"), no le importa si fue llamado o no. Un **Mock** es un objeto inteligente que, además de devolver respuestas simuladas, memoriza las interacciones; permite hacer aserciones de comportamiento ("Verifica que el método `send_email` fue llamado exactamente 2 veces con este payload").
- **Respuesta 2:** Los *Fixtures* de Pytest utilizan Inyección de Dependencias a nivel de parámetros de funciones en lugar del rígido modelo Orientado a Objetos (herencia) de `unittest`. Son modulares, reusables y se pueden definir con diferentes ámbitos (`scope="session"`, `"module"`, `"function"`). Permiten encadenar dependencias complejas (un fixture de `db_session` puede depender internamente de un fixture `db_engine`) sin llenar las clases de boilerplates de estado (`self.db`).
- **Respuesta 3:** Nunca debes depender directamente del reloj del sistema global en lógica de negocio crítica porque arruina la testeabilidad. La solución rápida es usar la librería externa **`freezegun`** en el test (añadiendo el decorador `@freeze_time("2023-01-01")`) que intercepta mágicamente las llamadas a `datetime`. La solución arquitectónica y más pura es inyectar un proveedor de tiempo explícito en la clase/función (Dependency Injection), permitiendo en el test pasar un reloj falso (Fake) controlado.

## 7. Errores comunes
- **Mockear el SUT (System Under Test):** Hacer `patch` a las mismas funciones internas que estás intentando testear. Un test debe observar los "Inputs" y los "Outputs" públicos. Si mockeas las tripas de la función (Whitebox testing rígido), el test perderá su valor porque será frágil: cualquier refactorización interna romperá los tests, aunque el resultado externo no cambie.
- **No limpiar el estado de Base de Datos:** Que un test haga `session.commit()` creando un usuario, y el siguiente test asuma que la base de datos está vacía, causando fallos aleatorios según el orden de ejecución. *Práctica: Usar fixtures de pytest que encapsulan el test entero en una transacción de BD y hacen `session.rollback()` al final de cada test.*

## 8. Buenas prácticas
- **Factory Boy / Faker:** Para tests que requieren datos de base de datos complejos (ej. un "Usuario" con 10 relaciones obligatorias), no crees diccionarios ni objetos masivos a mano en cada test. Usa factorías dinámicas para generar data de prueba sintética y válida instantáneamente.
- **Coverage Driven Mínimo:** 100% de cobertura es un número de vanidad que genera tests de baja calidad que solo sirven para ejecutar las líneas. Apunta a ~80%, enfocando el 100% de cobertura exclusivamente en el *Core Business Domain* (Casos de Uso) y relajando la red (Wrappers).

## 9. Trade-offs y decisiones técnicas
**Moto vs LocalStack vs Real AWS Accounts**
- *Moto (Mock en Memoria Python):* Extremadamente rápido para CI. Excelente para pruebas unitarias. Trade-off: Puede divergir de la realidad de AWS. A veces `moto` devuelve éxitos en llamadas a recursos configurados erróneamente que en AWS real fallarían.
- *LocalStack:* Clona la infraestructura de AWS en Docker. Súper preciso. Trade-off: Consumo inmenso de memoria en la máquina (requiere ~4GB RAM) y alarga sustancialmente el inicio de la suite de integración en el pipeline de CI/CD.
- *Cuentas AWS Efímeras reales:* La mayor fidelidad posible. Creas la infra con Terraform para el test y la borras al acabar. Trade-off: Mucho más caro, lento (desplegar un RDS lleva 10 mins) y propenso a dejar basura si el test de destrucción falla. Reservar solo para los escasos End-to-End tests críticos nocturnos.
