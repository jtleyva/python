# Brasil - Eye tracking y procesamiento en tiempo real

Empecé trabajando con Python en un proyecto entre 2015 y 2017 en Brasil, enfocado en eye-tracking para ayudar a personas con movilidad reducida a escribir con la mirada. Ahí trabajábamos con procesamiento en tiempo real y usamos concurrencia con hilos para separar la captura de datos del procesamiento, evitando bloquear la interacción del usuario. Fue mi primer contacto con problemas de sincronización y rendimiento.

_scikit-learn - Raw eye-tracking data (gaze position, pupil size, timestamps) is captured from devices like Pupil Labs headsets, Tobii, or webcams using tools like OpenCV and MediaPipe._

_scipy - NumPy - para optimización, interpolación, transformadas de Fourier, procesamiento de señales e imágenes_

> En el proyecto de eye-tracking teníamos un flujo en tiempo real donde necesitábamos capturar continuamente datos de la mirada y procesarlos sin bloquear la interacción.

> Para eso usamos concurrencia con hilos en Python, separando principalmente dos responsabilidades:
> un hilo encargado de la captura de datos desde el dispositivo otro hilo para el procesamiento y análisis

> Esto nos permitía que la captura fuera continua, sin verse afectada por el tiempo de procesamiento.

> Aunque Python tiene el GIL, en este caso no era un problema porque muchas operaciones eran I/O-bound o estaban apoyadas en librerías optimizadas, así que el uso de threads nos daba suficiente paralelismo.

> El principal reto fue la sincronización entre hilos y evitar condiciones de carrera, que resolvimos con estructuras thread-safe y controlando bien el acceso a los datos compartidos.

# AWS y microservicios

Después pasé a una plataforma de gaming online en tiempo real, donde estuve enfocado principalmente en backend con Python y FastAPI. Diseñamos una arquitectura basada en microservicios desplegada en Amazon Web Services usando contenedores en Amazon ECS. Ahí trabajábamos con task definitions para definir los servicios, gestionar recursos y escalar según la carga.

A nivel de arquitectura, combinábamos servicios persistentes con funciones en AWS Lambda para tareas asíncronas. Por ejemplo, cuando ocurrían eventos en la plataforma, los enviábamos a colas en SQS para desacoplar el procesamiento, lo que nos ayudaba a mejorar la resiliencia y evitar bloquear las APIs.

También usamos Amazon S3 para almacenamiento y Redis como sistema de cache para reducir latencias en operaciones críticas. Uno de los retos fue optimizar el uso de Lambda, especialmente por los cold starts, que mitigábamos manteniendo funciones ligeras y reutilizando recursos cuando era posible.

> Elegimos Amazon ECS porque necesitábamos un equilibrio entre control y simplicidad. Teníamos varios servicios backend en Python que requerían cierta estabilidad y control de recursos, por lo que un enfoque basado en contenedores encajaba mejor que serverless puro. Evaluamos opciones como Lambda, pero para servicios persistentes con tráfico constante preferimos ECS para evitar problemas como cold starts y limitaciones de ejecución.
> Además, ECS nos permitía definir fácilmente los servicios mediante task definitions, controlar CPU/memoria y escalar horizontalmente según carga.

> En cuanto al CI/CD, teníamos un pipeline donde: construíamos la imagen Docker la subíamos a un registry (ECR) y luego desplegábamos actualizando la task definition del servicio en ECS

> Esto nos permitía tener despliegues reproducibles y versionados.

# Sistema de onboarding basado en microservicios con RAG y AWS Bedrock

Recientemente, trabajé en un sistema de Onboarding basado en microservicios. Aunque la plataforma principal estaba en Node.js, fui responsable junto con un equipo de diseñar y desarrollar un microservicio especializado en IA utilizando **FastAPI y LangGraph**.

El núcleo de este servicio era un sistema de asistencia inteligente. Para orquestar la lógica compleja, implementamos una arquitectura multi-agente utilizando **LangGraph** siguiendo un patrón de supervisión. En lugar de tener un solo flujo lineal de RAG, diseñamos un grafo de ejecución gobernado por un **Agente Supervisor**. Este supervisor evaluaba la intención del usuario, delegaba las tareas y monitoreaba continuamente el trabajo de dos agentes especializados:

- Un **Agente RAG** para búsqueda semántica sobre la documentación interna de onboarding (usando embeddings y pgvector como bases de datos vectorial, exploramos alternativas como chromedb, pero pgvector nos dio mejor rendimiento y control).
- Un **Agente Text2SQL** encargado de traducir lenguaje natural a consultas SQL para responder preguntas sobre el progreso del usuario o datos específicos del dominio(evaluaciones, brechas, etc).

Para que estos agentes pudieran interactuar con nuestros sistemas, desarrollamos un conjunto de **Tools (herramientas personalizadas)** mediante _Function Calling_. Estas tools eran funciones en Python, fuertemente tipadas con Pydantic, que encapsulaban la lógica de conexión a la base de datos o al motor de búsqueda vectorial.

Además, para mantener los microservicios verdaderamente desacoplados, implementamos estas integraciones con **MCP (Model Context Protocol)**. Esto nos permitió separar la lógica de orquestación de la IA (en LangGraph) de las fuentes de datos; los agentes simplemente se conectaban a un servidor MCP que exponía los recursos y las herramientas de forma estandarizada y segura.

El Agente Supervisor no solo enrutaba, sino que revisaba las respuestas de estos sub-agentes. Si la información era incompleta o había un error de ejecución en alguna de las tools, el supervisor era capaz de ordenarles iterar y corregir el fallo antes de devolver el resultado final al usuario.

Integré los LLMs a través de **AWS Bedrock**, lo que nos permitía usar modelos fundacionales (como la familia Claude de Anthropic) de forma segura y sin que los datos salieran de nuestro entorno de AWS.

Uno de los retos técnicos fue la **gestión de la memoria y el estado**. Gracias a LangGraph, implementamos un estado conversacional persistente (state graphs). Esto permitía que los agentes mantuvieran el contexto de interacciones pasadas, recordando qué pasos del onboarding ya había completado el usuario. Expusimos todo este flujo complejo a través de endpoints en FastAPI, manejando respuestas en streaming (Server-Sent Events) para reducir la latencia percibida por el usuario. Además, implementamos **Autenticación (JWT) y Rate Limiting** a nivel de API Gateway y middleware de FastAPI para proteger los endpoints contra abusos y garantizar un uso justo de los recursos del LLM.

**Pregunta:** Tu sistema de agentes está costando mucho dinero al mes. Tu jefe te pide reducir costos sin degradar la calidad. ¿Qué haces?
**Respuesta:** Primero optimizo el sistema, luego el modelo si es necesario.
Investigo los costos de:

- Modelo de inferencia : ¿Puedo cambiar a un modelo más barato (ej. Claude 3.5 Sonnet) sin perder calidad?
- Tokens: ¿Puedo reducir el número de tokens enviados (ej. resumiendo el contexto, usando chunking inteligente)?
- Frecuencia: ¿Puedo reducir la frecuencia de llamadas (ej. cacheando respuestas, usando triggers en lugar de llamadas periódicas)?
- Uso de herramientas: ¿Puedo hacer que el agente use herramientas externas (ej. una base de datos vectorial) para reducir la necesidad de inferencia directa?
- Embedings: ¿Puedo usar un modelo de embedding más barato para búsquedas semánticas, en lugar de enviar todo el contexto al LLM?
- Vector DB: ¿Puedo optimizar las consultas a la base de datos vectorial para reducir tokens y llamadas al LLM?
- Costo computacional : ¿Puedo hacer preprocesamiento o postprocesamiento localmente para reducir la carga en el LLM?

** Pregunta:** ¿Dónde se deberian poner los Guardrails en un sistema de agentes?
**Respuesta:** Los guardrails (restricciones) deben implementarse en múltiples niveles:

- **Prompt Level:** Incluir restricciones claras en el prompt (ej. "No hagas llamadas a APIs externas", "Responde solo con JSON", Inyección de ejemplos negativos, PII - Información Personal Identificable). Así se evita el uso de tokens innecesarios y por consiguiente costos adicionales.
- **Middleware Level:** Implementar validaciones en el código que recibe la respuesta del LLM (ej. verificar que el JSON es válido, que no se exceden ciertos límites).
- **Model Level:** Usar modelos con capacidades de seguridad mejoradas o configuraciones de temperatura bajas para reducir alucinaciones.
- **Tool Level:** Si el agente puede usar herramientas, implementar restricciones en las herramientas mismas (ej. limitar qué endpoints puede llamar, validar inputs/outputs).

**Pregunta:** ¿Qué pasa si alguna tool o agente falla? ¿Cuál sería el mecanismo de fallback?
**Respuesta:** En una arquitectura multi-agente con LangGraph, manejamos los fallos en varios niveles para asegurar la resiliencia y una degradación elegante (graceful degradation):

- **Reintentos y Auto-corrección (Nivel Agente):** Si una tool falla (por ejemplo, el Text2SQL genera una query inválida o falta un parámetro requerido), capturamos la excepción. El Agente Supervisor recibe el mensaje de error y le pide al sub-agente que reflexione y corrija su llamada (limitado a un máximo de N reintentos para evitar bucles infinitos).
- **Fallback Routing (Nivel Grafo):** LangGraph permite definir rutas alternativas (conditional edges) y nodos de fallback. Si un sub-agente o servidor MCP no responde tras los reintentos permitidos, el flujo se redirige automáticamente a una estrategia de contingencia.
- **Degradación Elegante:** Si, por ejemplo, el sistema que responde consultas SQL está caído, el Supervisor puede hacer "fallback" hacia el Agente RAG para intentar responder usando solo conocimiento documentado, o bien devolver un mensaje amigable al usuario ("Actualmente no puedo acceder a tus datos de progreso, pero...") en lugar de mostrar un error 500 y romper la experiencia.
- **Circuit Breaker y Timeouts (Nivel Infraestructura):** En las conexiones de FastAPI hacia AWS Bedrock o servidores MCP, aplicamos timeouts estrictos y un patrón Circuit Breaker. Si un servicio externo falla repetidamente, el circuito se abre para fallar rápido (fail-fast) y no agotar los recursos del servidor (hilos/conexiones).

---

# ¿Por qué LangGraph y no LangChain tradicional o un pipeline secuencial?

LangChain estándar es excelente para cadenas lineales (DAGs simples), pero nuestro flujo de onboarding requería ciclos y toma de decisiones dinámica. LangGraph nos permitió definir la lógica como una máquina de estados finita. Al tener un Agente Supervisor, si el Agente Text2SQL generaba una consulta errónea, el supervisor lo detectaba y forzaba un bucle interno para que el sub-agente corrigiera el SQL leyendo el esquema de la base de datos antes de devolver la respuesta al usuario, algo muy difícil de hacer de forma limpia con cadenas tradicionales.

# ¿Cómo gestionaban la memoria en esta arquitectura multi-agente?

Utilizamos el sistema de _checkpointers_ de LangGraph apoyado en PostgreSQL. El estado del grafo se persistía al final de cada nodo de ejecución. Para garantizar la separación de datos y privacidad, asociábamos la memoria a un identificador único por sesión y usuario. Cuando un usuario enviaba un mensaje, el frontend enviaba un `session_id` vinculado a su token de autenticación. Cargábamos el estado usando este `session_id` como `thread_id` en LangGraph, lo que daba acceso al contexto histórico estrictamente limitado a esa sesión. Además, inyectábamos el `user_id` y su rol (RBAC) en el estado del grafo, de modo que los agentes, especialmente el Text2SQL, solo pudieran acceder a información permitida para su nivel de acceso.

# ¿Cómo optimizaron la latencia de Bedrock en el microservicio FastAPI?

Al tener múltiples agentes interactuando antes de dar una respuesta final, la latencia (Time to First Token) podía subir. Lo mitigamos de dos formas:

1. Los agentes de enrutamiento y validación interna usaban modelos más rápidos y ligeros, reservando los modelos más potentes (y lentos) de Bedrock solo para tareas complejas de razonamiento.
2. Implementamos endpoints asíncronos en FastAPI devolviendo respuestas en _streaming_ mediante Server-Sent Events (SSE), de modo que el frontend podía empezar a renderizar la respuesta del LLM mientras este seguía generando.

# Como actuar ante problemas de latencia?

1. Localizar el cuello de botella: ¿Es la inferencia del modelo, la consulta a la base de datos, o la lógica de orquestación?
2. Que optimizacion corrige el problema: ¿Puedo usar un modelo más rápido, reducir tokens, cachear resultados, o paralelizar tareas, comprimir el contexto?
3. Trade-offs: ¿Qué impacto tiene cada optimización en la calidad de la respuesta, el costo y la experiencia del usuario?
   trade-offs: latency vs accuracy, latency vs cost, latency vs freshness of data

# Cuando escoger un single agent vs multi-agent?

- **Single Agent:** Adecuado para tareas simples o flujos lineales donde no se requiere toma de decisiones dinámica. Es más fácil de implementar y mantener, pero puede volverse difícil de escalar o adaptar a cambios futuros.
- **Multi-Agent:** Ideal para flujos complejos con múltiples pasos, decisiones condicionales o cuando se requiere especialización. Permite una mayor modularidad y flexibilidad, pero introduce complejidad adicional en la orquestación y gestión de estados.

# ¿Qué medidas de seguridad tomaron con Text2SQL?

Gracias a la arquitectura con MCP y AWS, pudimos implementar un enfoque de seguridad en profundidad (Defense in Depth), delegando responsabilidades fuera del microservicio de IA.

Implementamos tres capas de seguridad principales:

1. **Aislamiento a nivel de base de datos (vía MCP)**: El servidor MCP se conectaba a PostgreSQL (pgvector) utilizando un rol de solo lectura (Read-Only) que únicamente tenía acceso a vistas materializadas específicas para analítica y onboarding, ocultando por completo las tablas core.
2. **Validación estricta de la query (vía MCP)**: Antes de que el servidor MCP enviara la consulta a la base de datos, el código del servidor validaba la petición para asegurar que solo fueran consultas `SELECT` y bloqueaba cualquier intento de usar sentencias DML (INSERT, DROP, UPDATE, etc.). Si detectaba una consulta inválida, el MCP devolvía un error explícito al Agente Supervisor en LangGraph para iterar.
3. **AWS Bedrock Guardrails y Detección de PII**: Como capa de protección a nivel del LLM, configuramos Guardrails nativos en Bedrock. Esto nos permitía bloquear intentos de _prompt injection_, filtrar lenguaje inapropiado y evitar que el modelo respondiera a temas fuera de dominio (off-topic). Un punto crítico fue la configuración de políticas de privacidad para la **detección y enmascaramiento automático de PII (Información de Identificación Personal)**. Si el usuario enviaba por error un DNI, correo electrónico o teléfono, el Guardrail lo detectaba mediante expresiones regulares y modelos NLP, ofuscando el dato (ej. `[EMAIL]`) antes de que llegara al LLM o quedara guardado en la memoria de la sesión.

# ¿Cómo definían y gestionaban las Tools de los agentes?

Usábamos _Function Calling_ nativo de los modelos (a través de Bedrock). En Python, definíamos las tools usando funciones regulares tipadas con Pydantic. Esto es clave porque Pydantic genera automáticamente el JSON Schema que el LLM necesita para saber qué parámetros enviar. Si el modelo alucinaba un parámetro o enviaba un tipo incorrecto, la validación de Pydantic fallaba y LangGraph capturaba esa excepción, pidiéndole al modelo que corrigiera su llamada.

# ¿Por qué usar MCP (Model Context Protocol) en lugar de integrarlo directamente?

En una arquitectura de microservicios, no queríamos que el microservicio de IA tuviera las credenciales y la lógica directa de acceso a todas las bases de datos o APIs de la empresa. Usando MCP, creamos servidores MCP ligeros cerca de las fuentes de datos. El microservicio de LangGraph actuaba como un cliente MCP. Esto nos dio un estándar unificado: si mañana añadíamos una nueva herramienta (por ejemplo, consultar Jira), solo desplegábamos un servidor MCP para Jira, sin tener que reescribir la lógica del agente supervisor.

1. START: Análisis de costos en un sistema de agentes con AWS Bedrock
   Modelo LLM: Claude 3 Sonnet (más económico que Opus, pero de calidad empresarial)
   Uso: 100,000 a 500,000 interacciones/mes (típico en onboarding o soporte interno)
   Tokens por interacción: 2,000 tokens promedio (entrada + salida)
   Agentes: Orquestación multi-agente, cada interacción puede implicar 2-3 llamadas al LLM (supervisor, RAG, Text2SQL)
   Otros costos: Almacenamiento de embeddings (pgvector), cómputo (ECS/Lambda), tráfico de red, pero el mayor costo es LLM.
2. Cálculo de costos Bedrock (Claude 3 Sonnet, mayo 2026)
   Precio aproximado: $2.00 USD por 1 millón de tokens (entrada), $6.00 USD por 1 millón de tokens (salida)
   Tokens totales/mes: 100,000 interacciones × 2,000 tokens × 2 agentes = 400 millones tokens/mes
   Distribución: 50% entrada, 50% salida
   Costo tokens entrada:
   200 millones tokens × $2.00 / 1M = $400
   Costo tokens salida:
   200 millones tokens × $6.00 / 1M = $1,200
   Total Bedrock LLM:
   $1,600 USD/mes (solo inferencia LLM)
3. Otros costos relevantes
   Embeddings: Si usas Titan Embeddings, calcula $0.10 por 1,000 vectorizaciones (mucho menor que LLM).
   ECS/Lambda: $100–$500/mes según tráfico y tamaño de los servicios.
   Almacenamiento (pgvector, S3): $10–$100/mes.
   Tráfico de red: Generalmente bajo en AWS interno.
4. Optimización de costos y latencia (para STAR)
   Situación: El gasto mensual superaba los $2,000 USD y la latencia era alta por múltiples agentes y tokens.
   Tarea: Reducir el costo sin sacrificar calidad ni experiencia de usuario.
   Acción:
   Cambié el modelo de Opus a Sonnet tras pruebas de calidad.
   Implementé cache de respuestas y embeddings para evitar llamadas redundantes.
   Reduje el contexto enviado (chunking inteligente, resúmenes).
   Usé modelos ligeros para validaciones y enrutamiento, reservando el LLM potente solo para tareas complejas.
   Ajusté el número de agentes y llamadas por flujo.
   Implementé SSE para respuestas en streaming, mejorando la percepción de latencia.
   Resultado:
   Reduje el costo LLM en un 40% (~$1,000 USD/mes).
   Bajé la latencia percibida de 5s a menos de 2s.
   Mantuvimos la calidad de las respuestas y la satisfacción del usuario.

---

# ¿Cómo gestionabais reintentos en SQS?

En Amazon SQS nos apoyábamos en el mecanismo de visibilidad (visibility timeout) y en los reintentos automáticos.

Cuando una AWS Lambda consumía un mensaje y fallaba, el mensaje volvía a la cola tras el timeout, permitiendo un nuevo intento.

Para evitar reintentos infinitos, configurábamos un maxReceiveCount y, al superarlo, enviábamos el mensaje a una Dead Letter Queue.

Además, diseñábamos los consumidores para ser idempotentes, de forma que si el mismo evento se procesaba más de una vez no generara efectos inconsistentes.

En algunos casos, también ajustábamos el visibility timeout según el tipo de tarea para evitar que procesos largos generaran reintentos prematuros.

# ¿Qué pasa si la Lambda falla?

Depende del tipo de invocación, pero en nuestro caso, al trabajar con eventos desde SQS, los fallos se gestionaban automáticamente mediante reintentos.

Si la ejecución de la AWS Lambda fallaba, el mensaje no se eliminaba de la cola y se reintentaba.

Si tras varios intentos seguía fallando, se movía a una Dead Letter Queue para su análisis posterior.

A nivel de diseño, intentábamos:

- hacer funciones idempotentes
- registrar errores de forma centralizada y evitar fallos silenciosos

En casos más críticos, podríamos añadir alerting o re-procesamiento manual desde la DLQ.

# ¿Cómo invalidas cache en Redis?

Con Redis usábamos principalmente estrategias de cache-aside.

Es decir:

- primero consultamos cache
- si no está, leemos de base de datos
- y luego poblamos la cache

Para la invalidación, combinábamos:

- TTLs para evitar datos obsoletos a largo plazo
- invalidación explícita cuando había cambios críticos

Por ejemplo, si se actualizaba información de usuario, eliminábamos directamente las claves relacionadas para forzar su recarga.

Uno de los retos es evitar inconsistencias, así que hay que balancear entre frescura de datos y rendimiento.

# ¿Cómo escalabais ECS?

En Amazon ECS escalábamos principalmente a nivel de servicio, ajustando el número de tareas en función de la carga.

Usábamos auto scaling basado en métricas como CPU o uso de memoria, lo que nos permitía añadir o quitar instancias automáticamente.

Además, al trabajar con contenedores, cada servicio era independiente, lo que facilitaba escalar solo las partes necesarias del sistema.

En algunos casos también teníamos en cuenta el tipo de carga, ya que no todos los servicios necesitaban escalar igual.

El principal reto era encontrar el equilibrio entre rendimiento y coste, evitando sobreaprovisionamiento.

# ¿Cuándo usarías ECS vs Lambda?

Usaría AWS Lambda para cargas event-driven, intermitentes o tareas desacopladas donde quiero evitar gestionar infraestructura.

Y Amazon ECS para servicios persistentes, con tráfico constante o cuando necesito más control sobre runtime, recursos o dependencias.

En muchos casos, lo ideal es combinarlos según el tipo de workload.

---

# Presentación de CourseRobot Backend

## 1. El "Elevator Pitch" (Introducción, 1-2 minutos)

_Objetivo: Enganchar a los entrevistadores y dar contexto claro de qué construiste._

> "Para este proyecto, lideré el desarrollo de un robot de Cursos, un backend diseñado para la gestión y creación automatizada de cursos en plataformas educativas (LMS como Moodle). El objetivo principal era resolver el problema de la creación manual de estructuras de cursos, que suele ser lenta. Lo construímos utilizando **ASP.NET Core 8**, aplicando principios de **Diseño Orientado al Dominio (DDD)** y, lo más importante, se integró **Inteligencia Artificial generativa** para estructurar dinámicamente el contenido de los cursos basándose en parámetros específicos del usuario."

## 2. Arquitectura y Stack Tecnológico (2-3 minutos)

_Objetivo: Demostrar madurez técnica y conocimiento de patrones de diseño empresariales._

> "A nivel arquitectónico, decidimos separar claramente las responsabilidades usando DDD.
>
> - **Stack Principal:** **.NET 8** por su alto rendimiento y las últimas características de C#.
> - **Base de Datos:** Se implementó **Entity Framework Core** como ORM conectándose a **MariaDB**.
> - **Patrones:** Todo está fuertemente acoplado a **Inyección de Dependencias** nativa de .NET.
>
> Esta estructura permitió que la capa de dominio (donde viven las reglas de negocio de cursos, usuarios, paquetes SCORM, etc.) fuera completamente independiente de la infraestructura o de los controladores de la API"

## 3. El Diferenciador: La Integración con IA (Punto Clave, 3-5 minutos)

_Objetivo: Demostrar tu experiencia directa integrando LLMs en un flujo de backend estructurado._

> "El componente más interesante del proyecto es el `CourseContentAIController`. En lugar de ser un simple CRUD, la aplicación actúa como un motor generativo.
>
> Funciona de la siguiente manera:
>
> 1. El cliente envía un payload detallado (perfil de participantes, tiempo estimado, objetivo del curso, estilo de tono, etc.).
> 2. El backend procesa esta solicitud, ensambla un **Prompt estructurado** utilizando plantillas almacenadas (`PromptTemplates`) y gestiona un historial (`PromptHistory`).
> 3. Nos integramos con la **API de OpenAI** (o el LLM correspondiente) para generar una estructura pedagógica completa.
> 4. Una vez recibida la respuesta de la IA, el backend parsea este output y persiste las entidades relacionadas en base de datos de forma transaccional, devolviendo finalmente la estructura generada y el ID del nuevo curso listo para consumirse."

## 4. Calidad, Testing y Buenas Prácticas (1-2 minutos)

_Objetivo: Mostrar que no solo haces que las cosas funcionen, sino que las haces bien._

> "Para asegurar la mantenibilidad y calidad del código:
>
> - Se documentaron todos los endpoints utilizando **Swagger**, lo que facilita enormemente la integración con el equipo de Frontend.
> - Se implementó una suite de **Pruebas Unitarias** (`CourseRobot.Tests` y `CourseRobot.NewTests`) que aseguran el correcto funcionamiento de los casos de uso principales.
> - La aplicación está lista para ser contenerizada, ya que incluye su propio **Dockerfile** y scripts de entrada, lo que facilita su despliegue en cualquier entorno Cloud."

---

## 💡 Anexo: Posibles Preguntas Técnicas y Cómo Responderlas

Si te hacen estas preguntas, aquí tienes los puntos clave para responder:

**Pregunta 1: ¿Por qué usaste DDD (Domain-Driven Design)?**
_Respuesta:_ "Porque en un sistema educativo las reglas de negocio pueden volverse complejas (ej. validaciones de paquetes SCORM, asignación de roles, gestión de equipos). DDD me permitió encapsular estas reglas en las entidades y aggregates, evitando que los controladores de la API se convirtieran en 'Spaghetti code'."

**Pregunta 2: ¿Cómo manejas las latencias o fallos con la API de IA?**
_Respuesta:_ "Es vital tener mecanismos de resiliencia. En .NET esto se puede manejar con librerías como Polly para implementar reintentos (Retries) o Circuit Breakers si la API de OpenAI no responde, además de tener timeouts configurados y guardar logs de cada intento." _(Nota: Si no usaste Polly, puedes mencionarlo como una mejora futura o explicar cómo manejas las excepciones en tu bloque try-catch)._

**Pregunta 3: ¿Cómo te aseguras de que el output de la IA tenga el formato correcto para tu base de datos?**
_Respuesta:_ "Mediante System Prompts estrictos donde exijo que la respuesta sea un JSON con una estructura específica (JSON mode). Luego, en .NET, utilizo deserialización fuertemente tipada (ej. `System.Text.Json`) y aplico validaciones antes de mapear la respuesta a mis entidades de Dominio de Entity Framework."

**Pregunta 4: ¿Cómo gestionas el ciclo de vida de las dependencias (Dependency Injection)?**
_Respuesta:_ "Utilizo los contenedores nativos de .NET. Registro repositorios y servicios IA como `Scoped` (por petición HTTP) para evitar compartir estado entre peticiones de diferentes usuarios, y registros Singleton para clientes HTTP o configuraciones globales."

**Pregunta 5: ¿Cómo manejas la seguridad y autenticación?**
_Respuesta:_ "Aunque no se muestra en el código, la aplicación está diseñada para integrarse con sistemas de autenticación como JWT o IdentityServer. Esto se puede implementar fácilmente añadiendo middleware de autenticación en el pipeline de ASP.NET Core, asegurando que solo usuarios autorizados puedan acceder a los endpoints de generación de cursos."

2002 ───────────────────────────────────────────────────────────────► 2026

.NET Framework 1.0
│
├── WinForms
├── ASP.NET
└── Solo Windows

2005-2015
│
├── .NET Framework 2.0 → 4.8
├── WPF
├── WCF
├── Entity Framework
└── MVC

2016
│
└── .NET Core 1.0
├── Open Source
├── Cross-platform
└── Mucho más rápido

2017 → 2019
│
├── .NET Core 2
└── .NET Core 3
├── Razor Pages
├── Blazor
└── Mejor rendimiento

2020
│
└── .NET 5
└── Unificación del ecosistema

2021
│
└── .NET 6 (LTS)
├── Minimal APIs
├── Hot Reload
└── Cloud Native

2022
│
└── .NET 7
├── Mejoras de rendimiento
├── Native AOT
└── Containers

2023
│
└── .NET 8 (LTS)
├── IA integrada
├── Blazor Full Stack
├── Aspire
└── Mejor rendimiento general

2024-2025
│
└── .NET 9
├── Mejoras en IA
├── Cloud y microservicios
├── OpenAPI nativo
└── Mejoras de productividad

2025-2026
│
└── .NET 10
├── Más optimización para IA
├── Mejor Native AOT
├── Mejor rendimiento
├── Mejor soporte cloud
├── Más integración con Aspire
└── Desarrollo multiplataforma más maduro
