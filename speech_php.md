# Plataforma de alta concurrencia y gestión de eventos asíncronos

Empecé trabajando en un proyecto enfocado en la ingestión y procesamiento de grandes volúmenes de datos en tiempo real. Ahí trabajábamos con arquitecturas orientadas a eventos en PHP, utilizando el ecosistema de Laravel. Separamos la captura de datos del procesamiento pesado mediante un sistema de colas robusto (Laravel Horizon y Redis), evitando bloquear la interacción del usuario y asegurando una experiencia fluida.

_Stack: PHP, Laravel, Redis, Laravel Horizon, PostgreSQL para almacenamiento persistente._

> En este proyecto teníamos un flujo donde necesitábamos recibir continuamente eventos de usuarios e integraciones de terceros sin afectar el tiempo de respuesta (latencia).

> Para eso aplicamos un patrón de desacoplamiento fuerte:
> Un endpoint rápido validaba y despachaba un Job a la cola.
> Workers en segundo plano (consumers) se encargaban de procesar la lógica de negocio pesada, transformar datos y guardarlos en PostgreSQL.

> El principal reto fue garantizar la fiabilidad del procesamiento y evitar condiciones de carrera en la base de datos, lo que resolvimos aplicando bloqueos pesimistas (pessimistic locking) en transacciones críticas y garantizando la idempotencia de los Jobs.

# Ecosistema Cloud, AWS y Calidad Estructural

Después pasé a liderar el backend de una plataforma SaaS, donde consolidé mi experiencia en diseño de software y arquitecturas modernas con Laravel. Diseñamos un sistema (con un enfoque de monolito modular que luego evolucionó hacia microservicios donde tenía sentido) desplegado en Amazon Web Services usando contenedores en Amazon ECS.

A nivel de arquitectura, aplicábamos principios SOLID y Clean Code para mantener el dominio aislado de la infraestructura. Por ejemplo, cuando ocurrían acciones del usuario, emitíamos eventos de dominio que eran procesados de forma asíncrona a través de Amazon SQS. Esto nos ayudaba a mejorar la resiliencia y evitar que un fallo en un servicio de terceros afectara el flujo principal.

La observabilidad era clave: integramos Sentry para monitorización proactiva de errores y SonarQube en nuestros pipelines (GitHub Actions / GitLab CI) para asegurar que el código cumpliera con nuestras estrictas métricas de salud y cobertura de tests antes de llegar a producción.

> Elegimos Amazon ECS porque necesitábamos un equilibrio entre control y simplicidad para nuestros contenedores de Laravel/PHP-FPM. Evaluamos arquitecturas serverless puras, pero para nuestro tráfico constante y necesidades de procesamiento en background, ECS nos daba estabilidad sin problemas de "cold starts".

> En cuanto al CI/CD, teniamos un pipeline donde:
> 1. Pasábamos análisis estático (PHPStan, SonarQube).
> 2. Ejecutábamos la suite de tests (PHPUnit/Pest).
> 3. Construíamos la imagen Docker y la subíamos a Amazon ECR.
> 4. Desplegábamos actualizando la task definition en ECS.

# Sistema IA-Driven y Desarrollo Asistido

Más recientemente, he trabajado en un sistema donde la IA generativa (IA-Driven Development) no solo era parte del producto, sino de nuestra cultura de ingeniería. Fui responsable de integrar capacidades de IA en el núcleo de la aplicación Laravel, utilizando modelos fundacionales a través de AWS Bedrock.

Implementamos un sistema de asistencia para usuarios (Onboarding) basado en un enfoque RAG (Retrieval-Augmented Generation). En PHP, orquestamos un flujo donde:
- Procesábamos consultas usando embeddings almacenados en PostgreSQL (pgvector).
- Teníamos una capa de "Agentes" orquestada desde Laravel que determinaba si la intención del usuario requería buscar en la documentación (RAG) o consultar métricas (Text2SQL).
- Para la capa Text2SQL, el backend de Laravel generaba los esquemas de base de datos relevantes y validaba de forma estricta las consultas generadas por el LLM antes de ejecutarlas en réplicas de solo lectura.

Además de producto, la IA forma parte de mi ciclo de vida de desarrollo. La utilizo para asistir en la definición de criterios de aceptación complejos, generar esqueletos de tests de integración para casos límite y apoyar la redacción de documentación técnica. Sin embargo, mi filosofía es que la IA propone, pero el ingeniero aplica el criterio (arquitectura, patrones y seguridad).

---------------------

# ¿Cómo garantizas la calidad y mantenibilidad del código (Clean Code / SOLID)?

En Laravel es fácil caer en el "fat controller" o acoplar la lógica al framework (Eloquent). Para evitarlo, aplico patrones de diseño con pragmatismo. Suelo separar la lógica de negocio en clases de servicio o "Actions", y uso el Service Container de Laravel para inyectar dependencias (Dependency Inversion, la 'D' de SOLID). Además, me apoyo fuertemente en herramientas de análisis estático como PHPStan al máximo nivel y SonarQube en el pipeline para evitar code smells.

# ¿Cómo manejáis la observabilidad de la plataforma?

La observabilidad total es crítica. Utilizamos Sentry no solo para cazar excepciones no controladas, sino para monitorizar el rendimiento (N+1 queries en Eloquent, lentitud en peticiones HTTP externas). Complemento esto enviando métricas y logs estructurados de los Jobs en background para tener trazabilidad end-to-end de cada petición del usuario usando UUIDs de correlación.

# ¿Cómo gestionabais reintentos y fallos en tareas asíncronas (SQS / Laravel Queues)?

En Laravel Queues apoyado en Amazon SQS, gestionamos los reintentos definiendo un `$tries` máximo y un `$backoff` exponencial. Si el Job falla por un error temporal (ej. una API externa caída), se reintenta más tarde. 
Si el fallo es persistente y agota los intentos, el Job pasa a la tabla `failed_jobs` (o una Dead Letter Queue en SQS). Para evitar efectos colaterales en los reintentos, diseño los Jobs para que sean 100% idempotentes usando `DB::transaction()` y bloqueos.

# ¿Qué medidas de seguridad tomasteis con la funcionalidad Text2SQL?

Implementamos un enfoque de "Defense in Depth":
1. **Aislamiento de DB**: Las consultas solo se ejecutaban contra un usuario de PostgreSQL de solo lectura que solo tenía acceso a vistas materializadas, sin acceso a datos sensibles (PII).
2. **Validación a nivel de framework**: En Laravel, antes de ejecutar la string SQL devuelta por el LLM, usábamos un parser para asegurar que solo contenía sentencias `SELECT`. Cualquier intento de inyección DML era descartado.
3. **AWS Bedrock Guardrails**: Aplicamos filtros nativos del modelo para evitar el *prompt injection* o intentos de engañar al sistema para extraer información confidencial.

# ¿Cómo invalidas cache en Redis dentro de Laravel?

Utilizo principalmente las Cache Tags de Laravel. En lugar de borrar claves enteras y reconstruir todo, etiquetamos las entradas en caché. 
Por ejemplo, si guardo el listado de proyectos de un usuario: `Cache::tags(['user:123', 'projects'])->put(...)`.
Si el usuario actualiza un proyecto, simplemente llamamos a `Cache::tags(['projects', 'user:123'])->flush()`. Así logramos una invalidación muy granular, generalmente disparada mediante Observers de Eloquent o Eventos de Dominio.

# AWS ECS vs enfoques Serverless (Laravel Vapor / Lambda)

Serverless (como Laravel Vapor sobre Lambda) es excelente para proyectos donde el tráfico es intermitente o quieres olvidarte completamente de la infraestructura. Sin embargo, para cargas muy altas y sostenidas de procesamiento, o si necesitas extensiones de PHP muy específicas y control sobre el runtime, Amazon ECS (Fargate) nos ofrecía un rendimiento más predecible (sin cold starts) y un coste más controlado a escala.
