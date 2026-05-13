# Brasil - Procesamiento en tiempo real y concurrencia (2015-2017)

> Empecé mi etapa técnica fuerte en un proyecto en Brasil enfocado en procesamiento de imágenes en tiempo real (eye-tracking).
> Fue un entorno muy exigente donde aprendí a lidiar con concurrencia a bajo nivel. Separamos la captura de datos del análisis usando hilos en Python para no bloquear nunca al usuario.
> Ahí me enfrenté por primera vez a problemas de sincronización y rendimiento. El verdadero reto fue evitar condiciones de carrera, lo cual resolvimos diseñando estructuras thread-safe y controlando estrictamente el acceso a datos compartidos para asegurar que el sistema no se colgara bajo ninguna circunstancia.

_Notas técnicas para la entrevista:_

- _Herramientas: OpenCV, MediaPipe, scikit-learn, scipy, NumPy (optimización y procesamiento de señales)._
- _Manejo del GIL: No era un problema crítico porque muchas operaciones eran I/O-bound o estaban apoyadas en librerías optimizadas en C, lo que nos daba suficiente paralelismo._

# AWS y arquitecturas distribuidas (Plataforma transaccional)

> Más adelante, di el salto a un rol más full-stack en una plataforma transaccional de alta demanda. Aquí el ecosistema era más heterogéneo: el core del backend estaba construido en Node.js, interactuando con interfaces de usuario y paneles de control desarrollados en Angular, mientras que integrábamos microservicios en Python para tareas específicas.
> En este proyecto el reto cambió: el foco principal era la resiliencia y la escalabilidad de todo el sistema. Diseñamos esta arquitectura basándonos en microservicios contenerizados en AWS (ECS).
> Nos enfocamos mucho en el desacoplamiento. Para la comunicación síncrona estricta entre servicios usábamos APIs REST, pero diseñamos la mayor parte del flujo guiado por eventos para evitar que el backend bloqueara el frontend en Angular. Por ejemplo, utilizábamos RabbitMQ para el enrutamiento complejo (Pub/Sub) de estados transaccionales entre el core de Node y los servicios en Python. Por otro lado, delegábamos a AWS SQS las cargas de trabajo asíncronas más pesadas o de procesamiento diferido, como la generación de reportes en background o el envío masivo de notificaciones.
> Al tener un sistema tan distribuido, la observabilidad era crítica. Centralizamos métricas y logs en CloudWatch y configuramos alarmas integradas directamente con nuestros canales de Slack. Esto nos daba alertas tempranas ante cualquier pico de latencia, fallos en las colas o aumento en la tasa de errores, permitiendo al equipo de ingeniería reaccionar de forma proactiva antes de que escalara a un incidente para el usuario final.
> A nivel de infraestructura, gestionábamos el tráfico utilizando Balanceadores de carga. Configuramos el balanceo para distribuir las peticiones hacia los contenedores mediante algoritmos de enrutamiento como Round Robin. Esto nos permitía escalar horizontalmente la capacidad de forma automática según la carga real del sistema. Para los procesos intermitentes, seguíamos delegando en Serverless (Lambda) para optimizar recursos.
> El CI/CD lo teníamos implementado utilizando AWS CDK como Infraestructura como Código (IaC) integrado con pipelines en Jenkins. Esto nos permitía definir la infraestructura mediante código y automatizar todo el ciclo: desde la construcción de las imágenes Docker y subida a ECR, hasta el despliegue automático configurando las Task Definitions y los servicios en ECS. Así asegurábamos que tanto la provisión como los despliegues fueran siempre inmutables, predecibles y versionados.

_Notas técnicas para la entrevista:_

- _Stack: Node.js (Core APIs), Angular (Frontend / Paneles de control), Python/FastAPI (Microservicios)._
- _Comunicación: REST (síncrona), RabbitMQ (eventos core / Pub-Sub), SQS (tareas background pesadas), Redis (caché)._
- _Observabilidad: CloudWatch integrado con webhooks de Slack para alertas tempranas sobre latencias y errores de consumo._
- _CI/CD: Jenkins + AWS CDK (IaC). Pipeline: Build Docker -> Push a ECR -> Generación/Actualización de Task Definitions -> Despliegue en ECS._
- _ECS vs Lambda: Preferimos ECS para tráfico constante y control de recursos; Lambda para cargas asíncronas orientadas a eventos._

# Arquitectura de microservicios y tolerancia a fallos (Sistema de Onboarding)

> En mi etapa más reciente, co-lideré la arquitectura de un sistema de microservicios bastante complejo para una plataforma de onboarding. Teníamos un core en Node.js y un ecosistema de orquestación en Python con FastAPI. Para escalar el desarrollo y asegurar la mantenibilidad, estructuramos los servicios basándonos en Arquitectura Hexagonal. Esto nos permitió aislar la lógica de negocio de la infraestructura y tener dominios muy bien definidos, lo que fue clave para que distintos equipos pudieran trabajar en nuevas features de forma paralela y totalmente desacoplada.
> El enfoque principal no era solo resolver la lógica, sino construir un sistema altamente predecible y tolerante a fallos. Implementamos un enrutamiento dinámico: si un servicio downstream fallaba, teníamos configurados reintentos automáticos, circuit breakers y estrategias de degradación elegante (fallbacks). Además, estandarizamos el manejo de errores transversalmente: ninguna excepción interna o volcado de pila llegaba al usuario final. Todo se interceptaba, se mapeaba a respuestas HTTP consistentes y se inyectaba con contexto enriquecido hacia nuestra plataforma de logs.
> Todo este enrutamiento y manejo de errores se apoyaba en una capa fuerte de monitorización que nos permitía trazar cada petición, auditar en tiempo real cuándo el sistema activaba un fallback y reaccionar ante cualquier problema.
> Por último, la seguridad fue un pilar innegociable. Protegimos la entrada a los microservicios con autenticación estricta (JWT) y rate limiting desde un API Gateway. Y a nivel de datos, aplicamos el principio de menor privilegio: desacoplamos por completo la orquestación de las bases de datos utilizando protocolos estandarizados (como MCP), interactuando únicamente a través de interfaces validadas y de solo lectura para aislar riesgos operativos.

_Notas técnicas para la entrevista:_

- _Arquitectura: Arquitectura Hexagonal (Ports and Adapters) para desacoplamiento de equipos y dominios. Orquestador supervisor (LangGraph) para IA._
- _Resiliencia y Errores: Circuit breakers, retry policies, fallbacks y Global Exception Handlers para estandarizar respuestas de error._
- _Observabilidad: Alertas sobre fallbacks activados, logs estructurados y monitorización activa de latencias inter-servicio._
- _Seguridad: API Gateway (JWT, Rate Limiting) y aislamiento total de la base de datos (PostgreSQL/pgvector) usando MCP con roles Read-Only._
- _Rendimiento: Flujos asíncronos y respuestas en streaming (SSE) a través de FastAPI para optimizar la latencia._

---

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

# Preguntas:

- ¿Cuáles son hoy los principales desafíos técnicos del equipo?
- ¿Cómo manejan confiabilidad y observabilidad en producción?
- ¿Qué nivel de participación tienen los desarrolladores en despliegues y soporte operacional?

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

# STAR

Situación:
Durante un pico inesperado de tráfico en la plataforma transaccional, varios usuarios reportaron lentitud y retrasos en la generación de reportes y notificaciones. El monitoreo en CloudWatch mostró un aumento en la latencia y mensajes acumulados en las colas SQS.

Tarea:
Identificar el cuello de botella y garantizar que los procesos asíncronos (reportes y notificaciones) se procesaran a tiempo, evitando que el sistema se saturara y afectara la experiencia del usuario.

Acción:

Analicé las métricas de CloudWatch y detecté que la cantidad de mensajes pendientes en SQS superaba el umbral habitual.
Ajusté la configuración de auto scaling en ECS para aumentar temporalmente el número de tareas consumidoras de SQS.
Optimizamos el código de los microservicios Python para hacerlos idempotentes y reducir el tiempo de procesamiento por mensaje.
Implementé alertas adicionales en Slack para detectar rápidamente futuros cuellos de botella en las colas.
Documenté el incidente y la solución en el runbook del equipo para futuras referencias.
Resultado:

El backlog de mensajes en SQS se procesó rápidamente y la latencia volvió a niveles normales.
Los usuarios dejaron de reportar retrasos y el sistema mantuvo su resiliencia ante futuros picos de tráfico.
El equipo ganó mayor visibilidad y capacidad de reacción ante incidentes similares, mejorando la operación y la confianza en la plataforma.
