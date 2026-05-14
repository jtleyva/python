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
