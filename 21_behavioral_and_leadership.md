# Behavioral and Leadership (Senior/Staff Level)

## 1. Explicación técnica
Las entrevistas "Behavioral" (Conductuales) son, irónicamente, las que más candidatos técnicamente brillantes reprueban en niveles Senior. Mientras que un Junior es evaluado por código puro, un Senior es evaluado por **Impacto, Liderazgo, Comunicación de Trade-offs y Resolución de Conflictos**.

La métrica principal de un ingeniero avanzado es cómo eleva el nivel de su equipo (Multiplicador de Fuerza) y cómo alinea la tecnología con el negocio.

## 2. Marco de Referencia: Método STAR
Tus respuestas deben estar siempre estructuradas así (evita divagar):
- **S (Situation):** El contexto (Ej: "La API de pagos colapsaba en Black Friday").
- **T (Task):** Tu rol específico (Ej: "Yo fui designado líder técnico para estabilizarla").
- **A (Action):** Qué hiciste (Ej: "Audité los logs en Datadog, detecté falta de conexión pool, lideré la migración a arquitectura asíncrona"). Enfócate en el "YO", no en "nosotros".
- **R (Result):** El impacto medible (Ej: "Redujimos la latencia un 80% y soportamos 3x más tráfico, lo que salvó $1M en ventas").

## 3. Preguntas de entrevista (Clásicas de Seniority) y Enfoques

### 1. "Cuéntame de un proyecto que fracasó o una mala decisión técnica que tomaste."
*Lo que buscan:* Humildad, madurez para aceptar errores y capacidad de aprendizaje. Los ingenieros que echan la culpa a "gerentes tontos" o "tecnologías malas" son rechazados ("Red flag").
*Respuesta esperada:* Cuenta una vez que caíste en "Over-engineering" (Ej: implementar microservicios y Kafka cuando un monolito Django era suficiente). Habla de cómo los costes de infraestructura se dispararon, cómo el equipo de 3 personas se ralentizó debido a la complejidad de CI/CD, y cómo aprendiste a empezar simple y escalar cuando el negocio lo justifique, no por seguir una moda tecnológica (Hype Driven Development).

### 2. "Tuviste un desacuerdo técnico fuerte con otro ingeniero Senior o tu Manager. ¿Cómo lo resolviste?"
*Lo que buscan:* Capacidad de resolución de conflictos, "Disagree and Commit" (Amazon Leadership Principle), y enfoque en datos, no en egos.
*Respuesta esperada:* Explica un escenario técnico (Ej: Él quería migrar de Postgres a NoSQL, tú preferías optimizar índices de Postgres). Cuenta cómo organizaste una reunión para revisar datos objetivos (latencias, throughput) o propusiste hacer una pequeña Prueba de Concepto (PoC - Benchmark) de 2 días. Si perdiste la discusión, recalca que apoyaste al 100% la decisión del equipo (Disagree and commit) para avanzar, demostrando profesionalismo.

### 3. "Describe cómo mentorizas a desarrolladores más junior en tu equipo."
*Lo que buscan:* Que no seas un lobo solitario ("Ninja Coder") que hace todo solo. Eres un multiplicador de valor.
*Respuesta esperada:* Da ejemplos reales: "En las Pull Requests, no solo rechazo el código o dejo comentarios como 'arregla esto'. Dejo enlaces a documentación, o si es un patrón complejo de Clean Architecture en Python, organizo llamadas de 15 minutos de Pair Programming. También implementé sesiones de 'Brown Bag / Tech Talks' cada dos semanas donde el equipo presenta lecciones aprendidas".

### 4. "Cuéntame de una vez que el negocio presionó por un deadline imposible sacrificando la calidad del código."
*Lo que buscan:* Pragmatismo comercial. Un negocio necesita enviar funcionalidades para sobrevivir. Rechazan a puristas de la tecnología que bloquean a la empresa.
*Respuesta esperada:* "Entiendo que el 'Time to Market' es crítico. Propuse un acuerdo (Trade-off): Entregamos la versión rápida con deuda técnica documentada (tickets en Jira). Usamos un monolito simple para llegar a la fecha límite, pero acordé con Product Management dedicar el 20% del tiempo de los Sprints posteriores a pagar esa deuda (refactorización) antes de escalar, protegiendo tanto el lanzamiento del producto como la estabilidad a largo plazo del sistema".

### 5. "Cuéntame de un incidente en Producción donde fuiste el responsable."
*Lo que buscan:* Gestión de crisis, comunicación clara bajo presión y la cultura "Blameless" (sin culpa post-mortem).
*Respuesta esperada:* Cuenta cómo un cambio en tu ORM tumbó la base de datos de producción. Detalla la mitigación: "Inmediatamente hice rollback al despliegue anterior" (Estabilizar primero, investigar después). "Luego de restablecer el servicio, escribí un documento de análisis 'Post-Mortem' sin culpar a personas, sino analizando fallos en el proceso: ¿Por qué nuestro CI no detectó esto? Como resultado final, implementé un paso adicional de Test de Carga en staging que prevendrá esto para siempre".

## 4. Red Flags (Errores fatales en la entrevista)
- **Mentalidad de Víctima:** Hablar mal de ex-jefes, ex-compañeros o empresas ("Era un desastre"). Habla del "entorno desafiante", no de "compañeros tóxicos".
- **Falta de métricas de negocio:** No saber (o no importarte) cómo tu código generó dinero, ahorró costes o mejoró la retención de usuarios.
- **Actitud del "Arquitecto de Torre de Marfil":** Dictar arquitecturas de sistemas complejos pero negarte a escribir código, debugear incidentes o ensuciarte las manos.
- **Usar "Nosotros" en lugar de "Yo":** Está bien ser equipo, pero en una entrevista evalúan *tu* contribución técnica. Si la arquitectura se salvó, sé claro sobre qué pieza diseñaste tú específicamente.
