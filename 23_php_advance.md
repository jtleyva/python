# PHP: Conceptos Básicos, Avanzados e Internals

## Conceptos Básicos y Fundamentos
- **Tipos de Datos**: PHP soporta tipado dinámico y estricto. Escalares: `bool`, `int`, `float`, `string`. Compuestos: `array`, `object`. Especiales: `callable`, `iterable`, `null`.
- **Strict Typing**: Declarando `declare(strict_types=1);` al inicio de un archivo, PHP lanza un `TypeError` si los tipos no coinciden exactamente (por ejemplo, pasar un '1' a un parámetro `int`), evitando la conversión de tipos automática (type juggling).
- **Arrays**: Internamente en PHP los arrays son mapas ordenados (hash tables). Son extremadamente flexibles y se usan como listas indexadas, diccionarios (arrays asociativos), pilas o colas.

## Características Modernas (PHP 8.x)
Suelen preguntar por la evolución del lenguaje para asegurar que tu código está actualizado:
- **Nullsafe Operator (`?->`)**: Evita errores fatales si un objeto es null, encadenando llamadas de forma segura (`$user?->profile?->address`).
- **Match Expression**: Alternativa moderna a `switch`. Es más concisa, retorna un valor y realiza comparaciones estrictas (`===`), sin requerir `break`.
- **Named Arguments**: Permite pasar argumentos a una función por su nombre, ignorando el orden posicional (`enviarEmail(to: 'a@b.com', body: 'Hola')`).
- **Constructor Property Promotion**: Permite definir y asignar propiedades directamente en la firma del constructor, reduciendo drásticamente el código repetitivo (boilerplate).
- **Union e Intersection Types**: Declarar múltiples tipos posibles (`int|float`) o requerir que se cumplan múltiples interfaces simultáneamente (`Countable&Iterable`).
- **Readonly Properties/Classes**: Permite crear objetos inmutables cuyas propiedades no pueden cambiar una vez inicializadas (ideal para DTOs).

## OOP: `self` vs `static` (Late Static Binding)
Una pregunta muy común en entrevistas de OOP:
- `self::` hace referencia a la clase donde el método *fue escrito*.
- `static::` hace referencia a la clase que *fue llamada* en tiempo de ejecución. Esto se conoce como **Late Static Binding** y es crucial cuando hay herencia.

## Closures y Paso por Referencia
- **Closures (Funciones Anónimas)**: A diferencia de JavaScript, para acceder a variables del ámbito exterior dentro de una función anónima en PHP, debes pasarlas explícitamente usando la palabra clave `use ($var)`.
- **Arrow Functions (`fn() =>`)**: Introducidas en PHP 7.4, capturan las variables del entorno exterior automáticamente por valor.
- **Paso por Valor vs Referencia (`&`)**: Por defecto, primitivos y arrays se pasan por valor (copia). Los objetos se pasan por "identificador de referencia". Usar `&$var` en la firma de una función fuerza el paso por referencia de memoria pura (cualquier cambio dentro de la función afecta a la variable externa original).

## Garbage Collector & Memory Management
PHP utiliza un sistema de recolección de basura basado en el **conteo de referencias** (reference counting). Cada variable tiene un contenedor (zval) que lleva la cuenta de cuántas veces está siendo referenciada. Cuando este contador llega a 0, la memoria se libera inmediatamente.
Adicionalmente, PHP cuenta con un **Cyclic Garbage Collector** para detectar y limpiar dependencias circulares (objetos que se referencian entre sí) que causarían fugas de memoria (memory leaks).

```php
// Forzar el garbage collector si se procesan grandes volúmenes de datos en scripts CLI prolongados
gc_collect_cycles();
```

## Generators (`yield`)
Los generadores proporcionan una forma fácil de implementar iteradores simples sin el sobrecoste de memoria o la complejidad de implementar una clase que extienda la interfaz `Iterator`. Permiten procesar listas grandes sin cargar todo el array en la memoria.

```php
function getLargeDataset() {
    $file = fopen('data.csv', 'r');
    while (($line = fgetcsv($file)) !== false) {
        yield $line; // Retorna un elemento y pausa la ejecución
    }
    fclose($file);
}

foreach (getLargeDataset() as $row) {
    // Procesa fila por fila (consumo mínimo de RAM)
}
```

## Traits vs Interfaces vs Abstract Classes
- **Interfaces**: Definen qué métodos debe implementar una clase (contrato). No contienen lógica.
- **Abstract Classes**: Pueden contener lógica compartida y métodos abstractos. Una clase solo puede heredar de una clase (Single Inheritance).
- **Traits**: Permiten la reutilización de código (Composición) en lenguajes de herencia simple. Ayudan a evitar la duplicación sin forzar jerarquías de herencia artificiales.

```php
trait Loggable {
    public function log($msg) { /* ... */ }
}

class User {
    use Loggable;
}
```

## OPcache y JIT (PHP 8)
- **OPcache**: PHP es un lenguaje interpretado. OPcache mejora el rendimiento almacenando el bytecode precompilado de los scripts en la memoria compartida, evitando que PHP tenga que cargar y parsear los scripts en cada petición.
- **JIT (Just In Time)**: Introducido en PHP 8, compila partes del bytecode directamente a código máquina (CPU) en tiempo de ejecución para cargas de trabajo intensivas en CPU (matemáticas, machine learning, etc).

## Magic Methods
Métodos especiales que sobreescriben la acción por defecto de PHP cuando se realizan ciertas acciones sobre un objeto. Comienzan con `__`.
- `__construct()` / `__destruct()`
- `__get()`, `__set()`: Interceptan lectura/escritura de propiedades inaccesibles.
- `__call()`, `__callStatic()`: Interceptan llamadas a métodos inexistentes o inaccesibles.
- `__invoke()`: Permite que un objeto sea llamado como una función.

# Laravel Advanced & Architecture

## Service Container & Dependency Injection
El IoC (Inversion of Control) Container es el corazón de Laravel. Gestiona las dependencias de las clases y realiza la inyección de dependencias automáticamente (autowiring). Permite registrar bindings e interfaces a implementaciones concretas.

```php
// En un ServiceProvider
$this->app->bind(PaymentGatewayInterface::class, StripePaymentGateway::class);
$this->app->singleton(CacheService::class, function ($app) {
    return new CacheService();
});
```

## Eloquent: El problema N+1
Ocurre cuando se hace una consulta inicial para obtener N registros y luego una consulta adicional por cada registro para obtener sus relaciones (N+1 queries totales). Se soluciona usando **Eager Loading** (`with`).

```php
// Problema (N+1):
$users = User::all();
foreach ($users as $user) { echo $user->profile->bio; }

// Solución (2 queries):
$users = User::with('profile')->get();
foreach ($users as $user) { echo $user->profile->bio; }
```

## Chunking & Cursors (Procesamiento masivo en Eloquent)
Para procesar miles de registros sin agotar la memoria:
- `chunk(100, callback)`: Obtiene bloques de registros mediante limit/offset y los carga en memoria como modelos.
- `cursor()`: Usa generadores de PHP (`yield`) subyacentes y consultas no cacheadas para iterar sobre los resultados de la base de datos uno a uno, utilizando drásticamente menos memoria.

```php
foreach (User::where('active', 1)->cursor() as $user) {
    // Procesa un modelo a la vez sin cargar toda la colección en memoria
}
```

## Jobs & Queues (Robustez e Idempotencia)
Para tareas lentas (emails, integraciones). Al usar AWS SQS o Redis (Laravel Horizon), es vital garantizar la **idempotencia**: si un Job falla a la mitad y se reintenta, no debe duplicar acciones (ej. cobrar dos veces).

```php
class ProcessPayment implements ShouldQueue {
    public $tries = 3;
    public $backoff = [10, 30, 60]; // Exponential backoff
    
    public function handle() {
        DB::transaction(function () {
            // Lógica con bloqueos (Pessimistic Locking)
            $order = Order::where('id', $this->orderId)->lockForUpdate()->first();
            // ...
        });
    }
}
```

## Events & Listeners
Permiten implementar una arquitectura orientada a eventos, desacoplando la lógica. Por ejemplo, al registrar un usuario, se emite `UserRegistered`. Los Listeners (`SendWelcomeEmail`, `NotifySlack`) reaccionan asíncronamente si implementan `ShouldQueue`.

## Request Lifecycle (Middleware)
1. `index.php` carga el autoloader y crea la aplicación.
2. La request pasa por el HTTP Kernel (global middlewares).
3. Pasa por el Router (route middlewares, form requests para validación).
4. Llega al Controller/Action.
5. Sube de vuelta por la pila de middlewares como un Response.

## Repository / Service Pattern
Un enfoque común en aplicaciones escalables:
- **Controllers**: Validan el input HTTP (Request) y retornan el output HTTP (Response/JSON). No contienen lógica de negocio compleja.
- **Services (Actions)**: Contienen la lógica de negocio pura.
- **Repositories**: Abstraen la capa de datos (Eloquent) para poder cambiar la fuente sin afectar a los Services.

## Multi-Tenancy (Arquitecturas Multi-inquilino)
Una arquitectura donde una sola instancia de la aplicación sirve a múltiples clientes (tenants), manteniendo sus datos completamente aislados.
**Estrategias principales:**
1. **Single Database (Columna `tenant_id`)**: Todos los clientes comparten las mismas tablas. Se añade una columna `tenant_id` y se filtra cada consulta. Es más fácil de mantener pero tiene mayor riesgo de fuga de datos si se olvida filtrar. En Laravel, esto se soluciona usando **Global Scopes**.
2. **Multi-Database**: Una base de datos independiente por cada tenant. Garantiza aislamiento total y facilita respaldos por cliente, pero hace que las migraciones y el despliegue sean más complejos.

Para la identificación del tenant activo en la request, se suele usar el subdominio (ej. `cliente1.saas.com`), un header HTTP, o el usuario autenticado.

## Global Scopes (Eloquent)
Muy utilizados en arquitecturas Multi-Tenant o Soft Deletes. Permiten añadir restricciones automáticamente a *todas* las consultas de un modelo.

```php
class TenantScope implements Scope {
    public function apply(Builder $builder, Model $model) {
        if (session()->has('tenant_id')) {
            $builder->where('tenant_id', session('tenant_id'));
        }
    }
}

class User extends Model {
    protected static function booted() {
        static::addGlobalScope(new TenantScope);
    }
}
// Ahora User::all() automáticamente añade "WHERE tenant_id = ?"
```

## Testing en Laravel (PHPUnit / Pest)
Laravel proporciona un entorno de testing muy robusto y con excelente Developer Experience (DX).
- **Feature Tests**: Prueban una porción grande del código, como una petición HTTP completa (rutas, middleware, base de datos, JSON de respuesta).
- **Unit Tests**: Prueban métodos o clases aisladas (ej. un algoritmo de cálculo) sin cargar toda la aplicación o la base de datos.
- **RefreshDatabase**: Un trait vital. Envuelve las pruebas en transacciones de base de datos o hace un migrate/rollback automático para que cada test empiece con una base de datos limpia, garantizando la predictibilidad.

### Mocking y Faking
Laravel incluye "Fakes" para no ejecutar servicios reales durante los tests (evitar mandar emails o procesar pagos).

```php
// Faking
Mail::fake();
// ... lógica que envía un email
Mail::assertSent(WelcomeEmail::class);

// Mocking con Mockery (inyección de dependencias)
$mock = Mockery::mock(PaymentGateway::class);
$mock->shouldReceive('charge')->once()->andReturn(true);
$this->app->instance(PaymentGateway::class, $mock);
```

## Seguridad: CSRF, XSS y SQL Injection
- **SQL Injection**: Laravel lo previene usando PDO binding (consultas preparadas) por defecto en Eloquent y el Query Builder.
- **XSS (Cross-Site Scripting)**: El motor de plantillas Blade escapa automáticamente todas las variables usando `{{ $var }}` (equivalente a `htmlspecialchars`).
- **CSRF (Cross-Site Request Forgery)**: Laravel incluye un middleware que verifica automáticamente el token `@csrf` en peticiones POST, PUT, DELETE para asegurar que la petición se generó desde la propia aplicación.
- **Mass Assignment Vulnerability**: Ocurre al pasar datos del usuario directamente a un modelo (`User::create($request->all())`). Se previene definiendo las propiedades seguras en el array `$fillable` (o protegidas en `$guarded`) del modelo Eloquent.
