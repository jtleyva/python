# Proyecto Ejemplo End-to-End: "Legacy2Modernizer"

## El Problema de Negocio
Empresas con repositorios masivos en tecnologías antiguas (ej. monolitos en Java 8 o JS sin tipar) carecen de documentación actualizada. Migrar o refactorizar este código es extremadamente riesgoso y costoso porque nadie entiende las dependencias profundas.

## La Solución: Legacy2Modernizer
Una plataforma asíncrona ("pipeline agentic") que lee un repositorio de código, extrae su estructura, analiza su deuda técnica (estado **AS-IS**) y automáticamente genera un documento de diseño técnico (RFC) proponiendo una arquitectura moderna (estado **TO-BE**).

---

## Arquitectura por Fases (Paso a Paso)

### 1. Ingesta y Procesamiento Asíncrono (Backend)
- **Webhook:** El usuario pega un link de GitHub en el frontend.
- **FastAPI:** Recibe el request, valida permisos y devuelve un `task_id` (Código 202 Accepted) para no bloquear al cliente.
- **Celery + Redis:** Encola el trabajo pesado. Un Worker en Python clona el repositorio localmente usando `subprocess`.

### 2. Parseo Semántico y RAG (Preparación de Contexto)
- **Tree-sitter (AST):** Escanea el código y genera un "esqueleto" (solo nombres de clases, firmas de métodos y dependencias importadas). Descarta la lógica interna para ahorrar tokens.
- **Vectorización:** Si el repo es gigantesco, indexamos los esqueletos y los `READMEs` en una BD Vectorial (ej. pgvector o Pinecone) usando embeddings de OpenAI.

### 3. Orquestación Multi-Agente (LangGraph)
Aquí entra la "inteligencia". LangGraph maneja el estado compartido y la comunicación entre dos sub-agentes especialistas:
- **Agente AS-IS (El Arqueólogo):** 
  - *Misión:* Entender qué hace el código actual.
  - *Herramientas (Tools):* Tiene acceso a hacer RAG sobre la BD Vectorial, ejecutar búsquedas `grep` precisas y pedir el cuerpo completo de un método si le genera dudas.
  - *Salida:* Genera un informe detallado de anti-patrones, dependencias críticas y lógica de negocio.
- **Agente TO-BE (El Arquitecto):**
  - *Misión:* Diseñar el futuro.
  - *Contexto:* Recibe el informe del agente AS-IS (no necesita leer todo el código de nuevo).
  - *Salida:* Propone una partición en Microservicios o Arquitectura Hexagonal.

### 4. Generación y Entrega
- El flujo unifica ambos reportes.
- El LLM (ej. Claude 3.5 Sonnet, por su capacidad analítica) formatea la salida final en **Markdown** e inyecta diagramas de arquitectura usando sintaxis **Mermaid**.
- El PDF final se guarda en un **bucket de AWS S3** y se envía una notificación al usuario por email o WebSockets.

---

## Pitch de Entrevista (Formato Problema-Acción-Resultado)

*Si te preguntan: "Háblame de un proyecto complejo que hayas diseñado desde cero."*

> "Uno de los retos más interesantes que diseñé fue un pipeline asíncrono para automatizar la refactorización de código legacy. **El problema** era que los monolitos eran tan grandes que no cabían en la ventana de contexto de ningún LLM.
> 
> **La acción** que tomé fue crear una API con FastAPI que encola el repositorio en Celery. En background, uso *Tree-sitter* para extraer solo el esqueleto del código (clases y firmas) y genero un índice RAG. Luego, orquesto dos agentes usando *LangGraph*: Un **Agente AS-IS** que actúa como explorador usando herramientas de búsqueda para mapear dependencias y anti-patrones, y un **Agente TO-BE** que recibe ese resumen (ahorrando millones de tokens) y redacta un diseño técnico moderno.
> 
> **El resultado** es un reporte completo en Markdown con diagramas de Mermaid que se guarda en S3. Esta arquitectura resuelve el problema del límite de contexto, evita alucinaciones al forzar al LLM a usar *tools* específicas, y convierte semanas de análisis humano en minutos de procesamiento paralelo."

---

## Retos Técnicos para mencionar en la Entrevista (Bonus Points)
- **Token Overflow:** "Al principio le pasábamos todo el repo al LLM y fallaba. Lo solucionamos implementando *AST-based chunking*."
- **LLM Loops:** "A veces el Agente AS-IS se quedaba atrapado leyendo el mismo archivo una y otra vez. Lo solucioné agregando un `recursion_limit` en LangGraph y una penalización en el prompt si el agente repetía una herramienta."