# Backend Python & AWS Senior Interview Prep

Repositorio avanzado de preparación para entrevistas técnicas de Backend (Python/AWS). Orientado a perfiles Mid/Senior y Staff Engineers. No encontrarás tutoriales de sintaxis aquí, sino decisiones de arquitectura, trade-offs, escalabilidad y problemas del mundo real.

## Estrategia de Estudio

Se recomienda abordar este repositorio en el siguiente orden para construir conocimiento de forma incremental:

1. **Core Language & Code Quality** (`01` al `02`): Dominio profundo de Python (GIL, Asyncio, Memory Management) y principios de diseño que los entrevistadores buscan en las pruebas de código (SOLID aplicado, patrones).
2. **Architecture & API Design** (`03` al `06`): Cómo estructurar proyectos, diseñar contratos robustos (REST/gRPC), y entender las implicaciones de los sistemas distribuidos (CAP theorem, consistencia eventual).
3. **AWS & Infrastructure** (`07` al `10`): Servicios core de AWS y patrones arquitectónicos nativos de la nube.
4. **DevOps, Containers & CI/CD** (`11` al `15`): Automatización, despliegues sin downtime, e infraestructura inmutable.
5. **Production Readiness** (`16` al `19`): Rendimiento, seguridad, observabilidad (logs/métricas/trazas) y patrones avanzados de escalabilidad.
6. **Entrevistas Finales & System Design** (`20` al `21`): Ejercicios prácticos de System Design y preparación para las entrevistas conductuales (Behavioral/Leadership).
7. **AI Automation & Enterprise Integration** (`22`): Integración de LLMs, RAG a escala, automatización orientada a eventos y arquitectura de agentes (Enfoque Enterprise Architect).

## Enfoque: Entrevistador"

Dado que tu entrevistador tiene este perfil, **el enfoque de la entrevista será transversal**. No solo querrán saber si sabes hacer un `for loop` en Python o levantar un RDS, sino cómo conectas sistemas empresariales con inteligencia artificial de forma segura y escalable.

**Puntos críticos a destacar con este perfil:**
- **Asincronía y Desacoplamiento:** Los LLMs son lentos. Demuestra que sabes usar SQS/EventBridge/Kafka para evitar bloquear la API HTTP.
- **Seguridad (Data Privacy):** Menciona cómo evitas enviar PII (Datos Personales) a APIs externas de LLMs mediante anonimización (Masking) previa en el backend Python.
- **Observabilidad en IA:** No basta con monitorear CPU; habla de rastrear (Tracing) latencias de LLMs, consumo de tokens y usar cachés semánticos (Redis/pgvector) para ahorrar costes de llamadas API.
- **Orquestación Determinista vs Agents:** Demuestra madurez prefiriendo arquitecturas controladas (ej. AWS Step Functions + Lambdas) frente a "agentes LLM autónomos impredecibles" para automatizaciones de negocio críticas.

## Cómo prepararse para System Design

- **Clarifica los requisitos**: Nunca asumas nada. Pregunta sobre QPS (Queries Per Second), ratio de lectura/escritura, latencia esperada y retención de datos.
- **Back-of-the-envelope estimations**: Acostúmbrate a calcular necesidades de almacenamiento y ancho de banda mentalmente.
- **Empieza simple (High-Level Design)**: Dibuja un balanceador de carga, servidores web y una base de datos.
- **Escala iterativamente (Deep Dive)**: Identifica el cuello de botella. ¿Es la base de datos? Añade una caché (Redis) o réplicas de lectura. ¿Es el procesamiento? Añade colas (SQS/Kafka).
- **Discute Trade-offs**: Todo tiene un costo. Explica *por qué* elegiste DynamoDB en lugar de PostgreSQL (ej. flexibilidad de esquema y escalabilidad vs. transacciones ACID complejas).

## Checklist de Habilidades Clave

- [ ] Entiendo cómo el GIL de Python afecta la concurrencia vs el paralelismo.
- [ ] Puedo diseñar una API idempotente.
- [ ] Sé cómo evitar "N+1 query problems" en un ORM (SQLAlchemy/Django).
- [ ] Comprendo la diferencia entre Message Queues (SQS/RabbitMQ) y Event Streaming (Kafka/Kinesis).
- [ ] Puedo explicar cómo aislar dependencias usando Clean/Hexagonal Architecture en Python.
- [ ] Conozco estrategias de despliegue (Blue/Green, Canary) en entornos AWS.

## Preguntas Frecuentes en Entrevistas (Sneak Peek)

1. *¿Qué pasa cuando escribes la URL en el navegador hasta que renderiza la página? (Enfoque en DNS, Load Balancers, API Gateway, Backend).*
2. *¿Cómo manejarías picos de tráfico impredecibles en un servicio de procesamiento de imágenes?*
3. *Tu API REST de repente tarda 5 segundos en responder en lugar de 50ms. ¿Cómo haces el troubleshooting en producción?*

## Tips para destacar como Senior

- **Habla de impacto comercial**: "Optimicé esta query, lo que redujo el coste de RDS en un 30%".
- **Acepta cuando no sabes**: Es mejor decir "No tengo experiencia directa con Cassandra, pero sé que usa un modelo de consistencia eventual" que inventar.
- **Mentoring y Cultura**: Los roles Senior requieren multiplicar el valor del equipo. Destaca cómo has ayudado a otros a crecer o mejorado procesos (DX, CI/CD).
