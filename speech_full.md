## Mi recorrido y experiencia

"Soy un desarrollador Fullstack. A lo largo de mi carrera me he especializado en el ecosistema de TypeScript/JavaScript con **Vue.js, Angular y React**, pero siempre trabajando muy de cerca con el **backend (Python y Node.js)** y la infraestructura en **AWS**. Os cuento mi recorrido por los tres proyectos que mejor definen mi perfil:

Empecé trabajando con Python en un proyecto entre 2015 y 2017 en Brasil, enfocado en eye-tracking para ayudar a personas con movilidad reducida a escribir con la mirada. Ahí trabajábamos con procesamiento en tiempo real y usamos concurrencia con hilos para separar la captura de datos del procesamiento, evitando bloquear la interacción del usuario. Fue mi primer contacto con problemas de sincronización y rendimiento.

Luego de esta experiencia continue como freelance trabajando remoto para una empresa en Estados Unidos.

### 1. Plataforma de Gaming Online (Alta concurrencia y AWS)

Vengo de trabajar en una plataforma de gaming online en tiempo real, un entorno transaccional y de alta concurrencia.
En el **frontend (Vue.js, principalmente Vue 3)**, el principal reto era el manejo de estado dinámico y la alta frecuencia de actualización de la UI, propia de un entorno de gaming en tiempo real. Lideré la implementación de un sistema de caching y optimizaciones de renderizado, donde aproveché al máximo el sistema de reactividad de Vue 3 y la Composition API para desacoplar lógica y mejorar la mantenibilidad. Por ejemplo, desarrollamos un sistema de virtualización de listas para renderizar solo los elementos visibles, lo que redujo el consumo de memoria y mejoró el rendimiento en dispositivos menos potentes. Además, diseñé un sistema de polling optimizado y mecanismos de actualización incremental del estado global, con esto minimizamos los re-renders innecesarios y aseguramos una experiencia mas fluida incluso con miles de usuarios conectados. Trabajé muy de cerca con el equipo de UI/UX para garantizar que las animaciones y transiciones fueran responsivas y consistentes, aplicando lazy loading y code splitting para optimizar el bundle inicial.
En el **backend (Python con FastAPI)**, montamos una arquitectura de microservicios. Usábamos colas en **Amazon SQS** para procesar eventos de forma asíncrona y **Redis** para reducir latencias de lectura.
A nivel de **AWS**, para los despliegues usábamos **Amazon ECS** (contenedores) para los servicios que necesitaban estar siempre levantados, y **AWS Lambda** para tareas puntuales y eventos, optimizando así los costes de infraestructura.

### 2. SaaS LBM (Industria de materiales de construcción y madera en Estados Unidos)

Aquí el reto técnico no era tanto la concurrencia, sino la **escalabilidad del código en el frontend** a medida que el producto y el equipo crecían.
Este proyecto era un monorepo en Nx, donde teniamos varios microfrontend en Vue.js y Angular. Trabajé en la refactorización hacia la _Composition API_ para poder extraer y reutilizar lógica, y migramos el estado global de **Vuex** a **Pinia**, segmentándolo por dominios de negocio.
Creamos un sistema de componentes reutilizables y configuré el CI/CD con Jenkins para asegurar que todo el equipo mantuviera el mismo estándar. Aquí también trabajamos con integraciones de APIs REST y ecosistemas de contenido.

**Vuex**(Obligaba a separar de forma estricta las mutaciones síncronas de las acciones asíncronas mediante flujos basados en commit y dispatch)

### 3. Sistema de Onboarding con IA (Streaming y Desacoplamiento)

En el último año y medio he estado al frente de en un proyecto de Onboarding(5 desarrolladores), revisando PR, mentoreando a desarrolladores Juniors. La plataforma principal estaba en **Node.js**, ademas construimos un microservicio de IA conversacional satélite.
En el **backend**, desarrollé este servicio con **Python (FastAPI) y LangGraph** para crear un sistema multi-agente conectado a **AWS Bedrock**. El reto arquitectónico fue de seguridad: ¿cómo dejamos que la IA consulte datos sin darle acceso directo a la base de datos? Lo resolvimos implementando **MCP (Model Context Protocol)** para que los agentes consultaran una base de datos vectorial de forma segura y aislada.
En el **frontend (Vue.js / TypeScript)**, el reto fue la percepción de velocidad. Desarrollé el consumo de **Server-Sent Events (SSE)** para que la interfaz renderizara las respuestas del LLM en streaming en tiempo real, mejorando radicalmente la UX."
Fui responsable de implementar un micro-framework para integrar Vue 3 en el ecosistema de Moodle que utiliza RequireJs y AMD. Esto mejoró la experiencia de usuario al permitir la integración de componentes de UI modernos sin romper la arquitectura del LMS.

---

## Q&A Rápido (Por si te preguntan detalles sobre la historia)

### ¿Cómo optimizaste el rendimiento en Vue.js en el SaaS?

Evitando estados globales monolíticos con Pinia, aplicando _lazy loading_ en el router para reducir el bundle inicial, y controlando los re-renders mediante el uso correcto de la Composition API.

### ¿Cuándo usabas ECS y cuándo Lambda en el proyecto de Gaming?

**Lambda** para tareas _event-driven_ (ej. procesar algo que llega a SQS) donde no quería mantener infraestructura. **ECS** para las APIs REST (FastAPI) que tenían tráfico constante y donde un _cold start_ de Lambda nos penalizaría la latencia.

### ¿Cómo aseguras la calidad en el código cuando lideras?

Tipado estricto con TypeScript (validando contratos entre front y back), automatizando reglas de estilo en el CI/CD, y haciendo _code reviews_ enfocadas en arquitectura y no solo en sintaxis.
