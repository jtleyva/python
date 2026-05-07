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

Más recientemente, he trabajé en un sistema de onboarding basado en microservicios. Aunque la plataforma principal estaba en Node.js, fui responsable junto con un equipo de diseñar y desarrollar un microservicio especializado en IA utilizando **FastAPI y LangGraph**. 

El núcleo de este servicio era un sistema de asistencia inteligente. Para orquestar la lógica compleja, implementamos una arquitectura multi-agente utilizando **LangGraph** siguiendo un patrón de supervisión. En lugar de tener un solo flujo lineal de RAG, diseñé un grafo de ejecución gobernado por un **Agente Supervisor**. Este supervisor evaluaba la intención del usuario, delegaba las tareas y monitoreaba continuamente el trabajo de dos agentes especializados:
- Un **Agente RAG** para búsqueda semántica sobre la documentación interna de onboarding (usando embeddings y bases de datos vectoriales con pgvector).
- Un **Agente Text2SQL** capaz de traducir lenguaje natural a consultas SQL para responder preguntas sobre el progreso del usuario o datos específicos del dominio.

Para que estos agentes pudieran interactuar con nuestros sistemas, desarrollamos un conjunto de **Tools (herramientas personalizadas)** mediante *Function Calling*. Estas tools eran funciones en Python, fuertemente tipadas con Pydantic, que encapsulaban la lógica de conexión a la base de datos o al motor de búsqueda vectorial. 

Además, para mantener los microservicios verdaderamente desacoplados, implementamos estas integraciones bajo el estándar **MCP (Model Context Protocol)**. Esto nos permitió separar la lógica de orquestación de la IA (en LangGraph) de las fuentes de datos; los agentes simplemente se conectaban a un servidor MCP que exponía los recursos y las herramientas de forma estandarizada y segura.

El Agente Supervisor no solo enrutaba, sino que revisaba las respuestas de estos sub-agentes. Si la información era incompleta o había un error de ejecución en alguna de las tools, el supervisor era capaz de ordenarles iterar y corregir el fallo antes de devolver el resultado final al usuario.

Integré los LLMs a través de **AWS Bedrock**, lo que nos permitía usar modelos fundacionales (como la familia Claude de Anthropic) de forma segura y sin que los datos salieran de nuestro entorno de AWS. 

Uno de los retos técnicos más interesantes fue la **gestión de la memoria y el estado**. Gracias a LangGraph, implementamos un estado conversacional persistente (state graphs). Esto permitía que los agentes mantuvieran el contexto de interacciones pasadas, recordando qué pasos del onboarding ya había completado el usuario. Expusimos todo este flujo complejo a través de endpoints en FastAPI, manejando respuestas en streaming (Server-Sent Events) para reducir la latencia percibida por el usuario.

En general, me siento cómodo trabajando en backend con Python sobre AWS, diseñando sistemas desacoplados, optimizando rendimiento y tomando decisiones de arquitectura en función de escalabilidad, latencia y coste.

---------------------

# ¿Por qué LangGraph y no LangChain tradicional o un pipeline secuencial?

LangChain estándar es excelente para cadenas lineales (DAGs simples), pero nuestro flujo de onboarding requería ciclos y toma de decisiones dinámica. LangGraph nos permitió definir la lógica como una máquina de estados finita. Al tener un Agente Supervisor, si el Agente Text2SQL generaba una consulta errónea, el supervisor lo detectaba y forzaba un bucle interno para que el sub-agente corrigiera el SQL leyendo el esquema de la base de datos antes de devolver la respuesta al usuario, algo muy difícil de hacer de forma limpia con cadenas tradicionales.

# ¿Cómo gestionaban la memoria en esta arquitectura multi-agente?

Utilizamos el sistema de *checkpointers* de LangGraph. El estado del grafo (que contenía el historial de mensajes y variables de contexto del usuario) se persistía en una base de datos al final de cada nodo. Cuando un usuario enviaba un nuevo mensaje, cargábamos el estado usando su `thread_id`, lo que le daba a todos los agentes acceso inmediato al contexto histórico.

# ¿Cómo optimizaron la latencia de Bedrock en el microservicio FastAPI?

Al tener múltiples agentes interactuando antes de dar una respuesta final, la latencia (Time to First Token) podía subir. Lo mitigamos de dos formas: 
1. Los agentes de enrutamiento y validación interna usaban modelos más rápidos y ligeros, reservando los modelos más potentes (y lentos) de Bedrock solo para tareas complejas de razonamiento. 
2. Implementamos endpoints asíncronos en FastAPI devolviendo respuestas en *streaming* mediante Server-Sent Events (SSE), de modo que el frontend podía empezar a renderizar la respuesta del LLM mientras este seguía generando.

# ¿Qué medidas de seguridad tomaron con Text2SQL?

Gracias a la arquitectura con MCP y AWS, pudimos implementar un enfoque de seguridad en profundidad (Defense in Depth), delegando responsabilidades fuera del microservicio de IA.

Implementamos tres capas de seguridad principales:
1. **Aislamiento a nivel de base de datos (vía MCP)**: El servidor MCP se conectaba a PostgreSQL (pgvector) utilizando un rol de solo lectura (Read-Only) que únicamente tenía acceso a vistas materializadas específicas para analítica y onboarding, ocultando por completo las tablas core.
2. **Validación estricta de la query (vía MCP)**: Antes de que el servidor MCP enviara la consulta a la base de datos, el código del servidor validaba la petición para asegurar que solo fueran consultas `SELECT` y bloqueaba cualquier intento de usar sentencias DML (INSERT, DROP, UPDATE, etc.). Si detectaba una consulta inválida, el MCP devolvía un error explícito al Agente Supervisor en LangGraph para iterar.
3. **AWS Bedrock Guardrails**: Como capa de protección a nivel del LLM, configuramos Guardrails nativos en Bedrock. Esto nos permitía bloquear intentos de *prompt injection*, filtrar lenguaje inapropiado y evitar que el modelo respondiera a temas fuera de dominio (off-topic) antes de que la consulta procesara la intención o llegara al agente de Text2SQL. También ayudaba a enmascarar o bloquear PII (Información de Identificación Personal) si el usuario la incluía por error.

# ¿Cómo definían y gestionaban las Tools de los agentes?

Usábamos *Function Calling* nativo de los modelos (a través de Bedrock). En Python, definíamos las tools usando funciones regulares tipadas con Pydantic. Esto es clave porque Pydantic genera automáticamente el JSON Schema que el LLM necesita para saber qué parámetros enviar. Si el modelo alucinaba un parámetro o enviaba un tipo incorrecto, la validación de Pydantic fallaba y LangGraph capturaba esa excepción, pidiéndole al modelo que corrigiera su llamada.

# ¿Por qué usar MCP (Model Context Protocol) en lugar de integrarlo directamente?

En una arquitectura de microservicios, no queríamos que el microservicio de IA tuviera las credenciales y la lógica directa de acceso a todas las bases de datos o APIs de la empresa. Usando MCP, creamos servidores MCP ligeros cerca de las fuentes de datos. El microservicio de LangGraph actuaba como un cliente MCP. Esto nos dio un estándar unificado: si mañana añadíamos una nueva herramienta (por ejemplo, consultar Jira), solo desplegábamos un servidor MCP para Jira, sin tener que reescribir la lógica del agente supervisor.

---------------------

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
