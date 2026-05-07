# Python Advanced & Internals

## Arrays
El modulo `array` provee un almacenamiento más eficiente de datos homogéneos que las listas normales, usando menos memoria y siendo más rápidos para ciertas operaciones.

## Garbage Collector
Es un mecanismo que libera la memoria de los objetos que ya no se utilizan. Python usa un sistema de referencia contadora para rastrear los objetos que se utilizan en el programa. Cuando el contador de referencia de un objeto llega a cero, el objeto se marca para ser eliminado. El garbage collector se encarga de eliminar los objetos marcados para ser eliminados.

```python
import gc

# Crear un objeto
obj = [1, 2, 3]

# Eliminar la referencia
del obj

# Forzar la recolección de basura
gc.collect()
```

## Not returning dicts & Lists

Cuando retornas un diccionario o una lista desde una función, Python crea una referencia al objeto. Si no retornas la referencia, el objeto puede ser eliminado por el garbage collector.

```python
def create_dict():
    return {"key": "value"}

def create_list():
    return [1, 2, 3]

# Si no retornas la referencia, el objeto puede ser eliminado por el garbage collector
```

## Method Resolution Order (MRO)

El **MRO** es el orden en el que Python busca métodos en una jerarquía de clases. Python usa el algoritmo C3 para calcular el MRO, que garantiza que las clases padres se busquen antes que las clases hijas.

```python
class A:
    def method(self):
        print("A")

class B(A):
    def method(self):
        print("B")

class C(A):
    def method(self):
        print("C")

class D(B, C):
    pass

# El MRO de D es D -> B -> C -> A
print(D.__mro__)
```

## Walrus Operator (:=)

El operador walrus (:=) permite asignar y retornar un valor en la misma expresión. Es útil para evitar repetir llamadas a funciones o para simplificar condiciones.

```python
# Sin walrus operator
if len(data) > 10:
    process(data)

# Con walrus operator
if (n := len(data)) > 10:
    process(data)
```

## operator.attrgetter

El módulo `operator` proporciona funciones que devuelven objetos callable que pueden ser usados para acceder a atributos de objetos.

```python
from operator import attrgetter

class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

people = [Person("Alice", 30), Person("Bob", 25)]

# Ordenar por edad
people.sort(key=attrgetter('age'))
```

## GIL (Global Interpreter Lock)

Es un mutex que protege el acceso a objetos Python, impidiendo que múltiples hilos nativos ejecuten bytecodes de Python simultáneamente. Esto significa que los hilos en CPython no proporcionan verdadero paralelismo para tareas CPU-bound, pero sí son útiles para tareas I/O-bound (como llamadas a red o base de datos).

## Concurrency vs Parallelism

- **Concurrency**: múltiples tareas ejecutándose en el mismo hilo (I/O-bound).
- **Parallelism**: múltiples tareas ejecutándose en hilos diferentes (CPU-bound).

## 2. Uso en producción
En sistemas de alto rendimiento (ej. web scrapers distribuidos, APIs de alta carga), elegir el modelo de concurrencia incorrecto puede derribar el sistema. 
- Usamos **asyncio** (ej. FastAPI, Aiohttp) para manejar miles de conexiones concurrentes I/O-bound con bajo overhead de memoria por conexión.
- Usamos **multiprocessing** (ej. Celery workers) para procesamiento intensivo de CPU (como transformaciones de video o machine learning), ya que cada proceso tiene su propio intérprete Python y su propio GIL.

## 3. Ejemplo práctico (Python)

```python
import asyncio
import time
import httpx
from typing import List

# Problema: Descargar datos de 100 APIs diferentes.
# Solución síncrona bloquearía la ejecución. Solución con hilos añade overhead.
# Solución óptima: Asyncio para operaciones I/O-bound.

async def fetch_data(client: httpx.AsyncClient, url: str) -> dict:
    try:
        response = await client.get(url, timeout=5.0)
        response.raise_for_status()
        return response.json()
    except httpx.HTTPError as e:
        print(f"Error fetching {url}: {e}")
        return {}

async def process_batch_urls(urls: List[str]) -> List[dict]:
    # Usamos un solo cliente para aprovechar la reutilización de conexiones (connection pooling)
    async with httpx.AsyncClient(limits=httpx.Limits(max_connections=50)) as client:
        # Creamos las tareas sin ejecutarlas aún
        tasks = [fetch_data(client, url) for url in urls]
        # asyncio.gather ejecuta concurrentemente. 
        # return_exceptions=True evita que un fallo cancele las demás tareas.
        results = await asyncio.gather(*tasks, return_exceptions=True)
        return [r for r in results if not isinstance(r, Exception)]

# Ejecución
# urls = ["https://api.example.com/data/1", ...]
# start = time.perf_counter()
# asyncio.run(process_batch_urls(urls))
# print(f"Time taken: {time.perf_counter() - start:.2f}s")
```

*Decisión clave:* Se usa `httpx` porque `requests` es sincrónico y bloquearía el event loop de asyncio. Se configura `max_connections` para no saturar los file descriptors ni hacer DoS al servicio de destino.

## Multiprocessing vs Multithreading

- **Multiprocessing**: Cada proceso tiene su propio intérprete Python y su propio GIL. Útil para tareas CPU-bound.
- **Multithreading**: Todos los hilos comparten el mismo intérprete Python y el mismo GIL. Útil para tareas I/O-bound.

## Multiprocesing race conditions

- **Race conditions**: Situaciones donde múltiples procesos acceden a los mismos recursos compartidos sin sincronización, causando comportamientos impredecibles.
- **Sincronización**: Mecanismos como locks, semáforos, barriers, etc., para coordinar el acceso a recursos compartidos.

## Shared memory in multiprocessing

- **Shared memory**: Mecanismos para que múltiples procesos compartan datos en memoria.
- **Memory views**: Objetos que permiten acceder a la memoria de otro objeto sin copiar los datos.
- **Memory mapping**: Mecanismos para mapear archivos a memoria y acceder a ellos como si fueran arrays.

## Why use collections

- **Collections**: Módulo que proporciona tipos de datos alternativos a los tipos de datos incorporados de Python.
- **Counter**: Clase que cuenta cuántas veces aparece cada elemento en una secuencia.
- **DefaultDict**: Diccionario que devuelve un valor por defecto si la clave no existe.
- **OrderedDict**: Diccionario que mantiene el orden de inserción de las claves.
- **NamedTuple**: Tupla con nombres de campos accesibles por índice o por nombre.
- **Deque**: Lista de doble extremo que permite inserciones y eliminaciones eficientes en ambos extremos.

## Four Pillars of OOP

- **Encapsulation**: Agrupar datos y métodos en una clase y controlar el acceso a ellos.
- **Inheritance**: Crear nuevas clases basadas en clases existentes.
- **Polymorphism**: Permitir que objetos de diferentes clases respondan al mismo mensaje de manera diferente.
- **Abstraction**: Ocultar la complejidad interna de un objeto y exponer solo lo necesario.

Example:

```python
from abc import ABC, abstractmethod

class Animal(ABC):
    @abstractmethod
    def make_sound(self):
        pass

class Dog(Animal):
    def make_sound(self):
        return "Woof!"

class Cat(Animal):
    def make_sound(self):
        return "Meow!"
```

## Python data model

- **Data model**: Mecanismos para que los objetos personalizados puedan interactuar con el lenguaje Python.
- **Magic methods**: Métodos especiales que permiten que los objetos personalizados puedan interactuar con el lenguaje Python.
- **Dunder methods**: Métodos especiales que permiten que los objetos personalizados puedan interactuar con el lenguaje Python.

Example: 
```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age
    
    def __str__(self):
        return f"{self.name} is {self.age} years old"

```

## Iterators

- **Iterator**: Objeto que puede ser iterado (tiene método `__iter__()` y `__next__()`).
- **Iterable**: Objeto que puede ser iterado (tiene método `__iter__()`).
- **Generator**: Función que devuelve un iterador (tiene palabra clave `yield`).

Example: 
```python
def my_generator():
    yield 1
    yield 2
    yield 3

for value in my_generator():
    print(value)
```

## static methods vs class methods

- **Static method**: Método que no recibe `self` ni `cls` como primer argumento.
- **Class method**: Método que recibe `cls` como primer argumento.

Example: 
```python
class MyClass:
    @staticmethod
    def my_static_method():
        return "This is a static method"
    
    @classmethod
    def my_class_method(cls):
        return "This is a class method"

my_instance = MyClass()
print(my_instance.my_static_method())
print(MyClass.my_class_method())
```
## Serialization & deserialization
Es el proceso de convertir un objeto en una secuencia de bytes para almacenarlo o transmitirlo, y luego reconstruirlo. La deserialización es el proceso opuesto.

## \_\_getstate\_\_ y \_\_setstate\_\_
Métodos especiales que permiten controlar la serialización y deserialización de un objeto.
Example:
```python
import pickle

class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age
    
    def __getstate__(self):
        return {"name": self.name, "age": self.age}
    
    def __setstate__(self, state):
        self.name = state["name"]
        self.age = state["age"]

person = Person("John", 30)
pickled_person = pickle.dumps(person)
unpickled_person = pickle.loads(pickled_person)
print(unpickled_person.name)
print(unpickled_person.age)
```

## heapq
Módulo que implementa una cola de prioridad basada en una cola de montículo.
Example:
```python
import heapq

heap = []
heapq.heappush(heap, 3)
heapq.heappush(heap, 1)
heapq.heappush(heap, 2)
print(heapq.heappop(heap))
print(heapq.heappop(heap))
print(heapq.heappop(heap))
```

## Higher order functions
Funciones que toman otras funciones como argumentos o devuelven funciones como resultado.
Example:
```python
def apply_operation(x, y, operation):
    return operation(x, y)

def add(x, y):
    return x + y

def multiply(x, y):
    return x * y

print(apply_operation(2, 3, add))
print(apply_operation(2, 3, multiply))
```

## filter
Función que filtra elementos de una secuencia basándose en una condición.
Example:
```python
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
even_numbers = list(filter(lambda x: x % 2 == 0, numbers))
print(even_numbers)
```

## Advance list comprehension
Comprensión de listas avanzadas con condiciones y expresiones.
Example:
```python
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
even_numbers = [x for x in numbers if x % 2 == 0]
print(even_numbers)
```

## bytes
Tipo de datos que representa una secuencia de bytes.
Example:
```python
byte_data = b"Hello"
print(byte_data)
```

## Python Bytecode and the dis module
Módulo que permite ver el bytecode de Python.
Example:
```python
import dis

def add(x, y):
    return x + y

dis.dis(add)
```

## Nesting & combining context managers
Example:
```python
with open("file1.txt", "r") as f1, open("file2.txt", "w") as f2:
    f1.read()
    f2.write("Hello")
```

# weakref
Módulo que permite crear referencias débiles a objetos. Una referencia débil no incrementa el contador de referencias del objeto, lo que permite que el objeto sea recolectado por el garbage collector.
Example:
```python
import weakref

class MyClass:
    def __init__(self, name):
        self.name = name

obj = MyClass("John")
weak_ref = weakref.ref(obj)
print(weak_ref().name)
```

## Optimizing memory with \_\_slots\_\_
Módulo que permite optimizar el uso de memoria en objetos. Permite definir explicitamente los atributos de una clase, lo que reduce el uso de memoria.
Example:
```python
class MyClass:
    __slots__ = ['name', 'age']
    def __init__(self, name, age):
        self.name = name
        self.age = age
```

## Advanced decorators
Módulo que permite crear decoradores avanzados.  Permite crear decoradores que pueden recibir argumentos.
Example:
```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("Before function")
        result = func(*args, **kwargs)
        print("After function")
        return result
    return wrapper

@my_decorator
def my_function():
    print("Inside function")
```

## Dataclases
Módulo que permite crear clases de datos con menos código. Permite crear clases inmutables y con métodos automáticos (frozen=True).
Example:
```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Person:
    name: str
    age: int
```

## Metaprogramación
Módulo que permite crear clases y objetos de forma dinámica. Permite crear clases y objetos en tiempo de ejecución.
Example:
```python
class MyClass:
    def __init__(self, name):
        self.name = name

MyClass = type('MyClass', (), {'name': 'John'})
```

## functools
Módulo que permite crear funciones que pueden ser usadas como decoradores.
Example:
```python
from functools import wraps

def my_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print("Before function")
        result = func(*args, **kwargs)
        print("After function")
        return result
    return wrapper

@my_decorator
def my_function():
    print("Inside function")
```


-----------------------

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Qué es el GIL y por qué Python lo usa? ¿Cómo afecta al diseño de una API REST?

- **Pregunta 2 (Práctica):** Tienes un endpoint en FastAPI que necesita procesar una imagen de 50MB (CPU-bound) antes de guardarla. Si lo haces directamente en la función de la vista, ¿qué pasará? ¿Cómo lo solucionarías?
- **Pregunta 3 (Arquitectura):** Tienes problemas de memoria (Memory Leaks) en un worker de Python que lleva semanas ejecutándose. ¿Cómo lo investigarías y qué mecanismos de Python podrían estar fallando?

## 6. Respuestas esperadas
- **Respuesta 1:** 
- **Respuesta 2:** Si ejecuto código CPU-bound directamente en una función `async def` de FastAPI, bloquearé el *event loop*, congelando toda la aplicación (nadie más podrá recibir respuestas). Solución: Delegar esa tarea a un thread pool o process pool (`asyncio.to_thread` en Python 3.9+) o mejor, usar un worker asíncrono en background (Celery/RQ) para no saturar los servidores web.
- **Respuesta 3:** Revisaría el crecimiento de memoria (usando herramientas como `tracemalloc` o `objgraph`). El recolector de basura de Python usa conteo de referencias y un detector de ciclos. Un memory leak en Python suele ocurrir cuando mantenemos referencias globales involuntarias (ej. listas estáticas, cachés sin TTL, o closures que capturan objetos grandes) que impiden que el GC libere la memoria.

## 7. Errores comunes
- **Anti-pattern:** Usar `requests` o funciones que bloquean (como `time.sleep`) dentro de corrutinas (`async def`). Esto bloquea el loop entero.
- **Anti-pattern:** Crear un nuevo hilo para cada petición entrante sin usar un ThreadPool, causando exhaustion de recursos (Context Switching overhead).
- **Mala decisión:** Sobrecargar el recolector de basura manipulando manualmente `gc.collect()` en entornos de producción, causando pausas (STW - Stop The World).

## 8. Buenas prácticas
- **Clean code:** Usar Context Managers (`with`) para recursos garantizando el cierre de conexiones, archivos y locks, incluso ante excepciones.
- **Escalabilidad:** Siempre establecer *timeouts* y usar *Connection Pools* cuando se hacen peticiones HTTP asíncronas o consultas a DB.
- **Mantenibilidad:** Usar Type Hints rigurosamente (`typing` module / `pydantic`). Ayuda a documentar y detectar errores con `mypy` en el CI/CD.

## 9. Trade-offs y decisiones técnicas
**Threads vs Asyncio**
- *Asyncio:* Ideal para alta concurrencia I/O con miles de conexiones (WebSockets, microservicios que llaman a otros microservicios). Trade-off: Require que **todo** tu ecosistema de librerías sea asíncrono (drivers de DB asíncronos, clientes HTTP asíncronos). Código más difícil de debuggear.
- *Threads:* Ideal cuando usas librerías legadas que bloquean. Trade-off: Consumen más memoria por hilo y el sistema operativo debe gestionar el context switching, limitando la escalabilidad a unos pocos miles concurrentes a lo sumo.
