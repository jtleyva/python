# Docker Basics & Best Practices para Backend

## 1. Explicación técnica
Docker no es una máquina virtual completa; es una plataforma que usa funcionalidades del kernel de Linux (cgroups y namespaces) para aislar procesos. Un contenedor incluye la aplicación y todas sus dependencias (librerías, binarios, runtime de Python) empaquetadas en una imagen estandarizada.

A nivel Senior, no basta con saber hacer `docker run`. Debes dominar:
- **Imágenes Multi-stage:** Reducir drásticamente el tamaño de la imagen final separando el entorno de compilación (build) del entorno de ejecución (runtime).
- **Caché de capas (Layer Caching):** Ordenar las instrucciones en el Dockerfile para que los cambios frecuentes (como el código fuente) no invaliden las capas pesadas y estáticas (como la instalación de dependencias `pip install`).
- **Seguridad (Non-root user):** Evitar correr procesos como `root` dentro del contenedor para prevenir vulnerabilidades de escalada de privilegios en el host.

## 2. Uso en producción
En Spotify, los desarrolladores de Python no se preocupan de si el servidor de producción tiene Python 3.9 o 3.11. Empaquetan su microservicio en un contenedor Docker, que es inmutable. El mismo contenedor que pasa los tests en CI (GitHub Actions) es exactamente byte-a-byte el que se despliega en producción en Kubernetes (GKE) o AWS Fargate. Esto elimina el infame problema de *"En mi máquina sí funciona"*.

## 3. Ejemplo práctico (Python)

Un `Dockerfile` optimizado para producción para una API en FastAPI/FastAPI. Usa Multi-stage build y usuario no root.

```dockerfile
# --- STAGE 1: Builder ---
# Usamos una imagen slim para compilar dependencias pesadas (ej. con C-extensions)
FROM python:3.11-slim as builder

# Variables de entorno para que Python no escriba archivos .pyc y no use buffer en stdout
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Instalamos dependencias del SO necesarias para compilar paquetes de Python
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc build-essential && \
    rm -rf /var/lib/apt/lists/*

# Creamos un entorno virtual (venv) para empaquetar las dependencias limpiamente
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copiamos SOLO el archivo de dependencias primero para aprovechar el Layer Caching
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# --- STAGE 2: Runtime (Producción) ---
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/opt/venv/bin:$PATH"

WORKDIR /app

# Creamos un usuario no-root por seguridad
RUN addgroup --system appgroup && \
    adduser --system --ingroup appgroup appuser

# Copiamos el venv ya compilado desde la fase 'builder' (ahorrando espacio de GCC, etc.)
COPY --from=builder /opt/venv /opt/venv

# Copiamos el código fuente de la aplicación
COPY ./src /app/src

# Cambiamos los permisos de los archivos al nuevo usuario
RUN chown -R appuser:appgroup /app

# Cambiamos al usuario no-root
USER appuser

EXPOSE 8000

# Usamos Uvicorn (o Gunicorn) como punto de entrada
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000", "--proxy-headers"]
```

## 4. Diagrama (texto)

Layer Caching en Dockerfile:

Capa 1: FROM python:3.11-slim        <-- Raramente cambia (Cacheada)
Capa 2: WORKDIR /app                 <-- Raramente cambia (Cacheada)
Capa 3: COPY requirements.txt .      <-- Cambia si añades una librería
Capa 4: RUN pip install ...          <-- SE RECONSTRUYE SOLO SI requirements.txt CAMBIA
Capa 5: COPY ./src /app/src          <-- Cambia en CADA commit de código
Capa 6: CMD ["uvicorn", ...]         <-- Cambia en cada commit (invalidada por la anterior)

*Si pones el `COPY ./src` ANTES del `pip install`, cada cambio en tu código obligará a Docker a descargar e instalar todas las dependencias de nuevo, ralentizando el CI de 1 minuto a 10 minutos.*

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Cuál es la diferencia entre una Imagen y un Contenedor en Docker?
- **Pregunta 2 (Práctica):** Tu imagen de Docker en Python pesa 1.5GB. Cuando la despliegas en AWS ECS, tarda mucho en arrancar (Cold Start alto). ¿Cómo reduces el tamaño de la imagen?
- **Pregunta 3 (Arquitectura):** Tienes una aplicación en Docker que genera muchos archivos temporales (logs, reportes CSV). Tras un mes en producción, el servidor se queda sin espacio en disco. ¿Por qué ocurre esto si los contenedores son "stateless"?

## 6. Respuestas esperadas
- **Respuesta 1:** Una Imagen es una plantilla inmutable de solo lectura que contiene el código, las bibliotecas, las variables de entorno y los archivos de configuración. Un Contenedor es una instancia *en ejecución* de esa imagen. Puedes levantar múltiples contenedores (instancias) a partir de una sola imagen.
- **Respuesta 2:** 1) Usar una imagen base más ligera (ej. `python:3.11-slim` o `alpine` en lugar de la imagen `python:3.11` completa que trae compiladores pesados de C). 2) Usar *Multi-stage builds* para descartar herramientas de build (gcc) en la imagen final. 3) Asegurarme de añadir `--no-cache-dir` al usar `pip install` para que pip no guarde en caché los paquetes `.tar.gz` descargados dentro de la imagen. 4) Crear un `.dockerignore` para no meter carpetas inútiles como `.git` o `venv` local en la imagen.
- **Respuesta 3:** El contenedor tiene una capa de escritura (Writeable Layer) efímera en el sistema de archivos (UnionFS). Los archivos escritos allí consumen espacio en el disco del host (o de la capa de almacenamiento subyacente de Docker). Aunque el contenedor se considere stateless, si nunca se destruye o reinicia, esa capa crecerá infinitamente. Solución: Los logs deben ir a `stdout/stderr` (no a archivos), y los archivos temporales deben guardarse en volumenes (Volumes/Bind mounts) gestionados o rotados automáticamente, o usar tmpfs.

## 7. Errores comunes
- **Correr 2 procesos en un contenedor (Anti-patrón):** Configurar un archivo `supervisord` o un script `start.sh` para levantar Nginx, Gunicorn, y un Celery Worker en el mismo contenedor. *Docker philosophy: Un contenedor = Un proceso principal*. Si el worker de Celery crashea, el contenedor no morirá porque Gunicorn sigue vivo, y el orquestador (ECS/K8s) no sabrá que debe reiniciarlo.
- **Ignorar el `.dockerignore`:** No incluir este archivo causa que el contexto de build sea gigante, enviando cientos de megabytes de la carpeta `.git` o de bases de datos locales `db.sqlite3` al daemon de Docker en cada build, enlenteciendo el proceso y comprometiendo la seguridad (puedes filtrar claves privadas accidentalmente).
- **Usar la etiqueta `:latest` en producción:** Depender de `FROM python:latest` en lugar de `FROM python:3.11.4-slim`. La etiqueta latest puede cambiar de versión mayor silenciosamente de un día para otro, rompiendo tu código de producción sin que hayas tocado nada.

## 8. Buenas prácticas
- **Graceful Shutdown (PID 1):** Cuando AWS detiene un contenedor, Docker envía una señal `SIGTERM` al proceso principal (PID 1). Si usas un script en bash como ENTRYPOINT que no propaga señales, tu app en Python no interceptará el `SIGTERM`, no cerrará las conexiones a DB limpiamente, y Docker la matará forzosamente (SIGKILL) a los 10 segundos, causando corrupción o queries interrumpidas.
- **Variables de entorno (12-Factor App):** Nunca hardcodear credenciales en el Dockerfile. La imagen generada debe ser genérica y las configuraciones específicas del entorno (`DB_HOST`, `API_KEY`) se inyectan en runtime (ej. vía AWS Parameter Store inyectado en la Task Definition de ECS).

## 9. Trade-offs y decisiones técnicas
**Alpine vs Debian Slim para imágenes de Python**
- *Alpine Linux:* Es increíblemente pequeño (~5MB base). Excelente para lenguajes estáticamente compilados (Go, Rust). Trade-off: Usa `musl libc` en lugar de `glibc` (estándar de Linux). Las dependencias de Python que requieren compilación de C (Pandas, Numpy, psycopg2) no encuentran ruedas (wheels) precompiladas compatibles con `musl`. Pip se verá forzado a compilarlas desde código fuente en cada build, lo que es lentísimo, propenso a errores sutiles en C, e irónicamente puede resultar en una imagen más grande.
- *Debian Slim (`python:X.Y-slim`):* Usa `glibc`, lo que significa soporte inmediato para el 99% de las ruedas precompiladas de PyPI (`manylinux`). Los builds son rápidos. Trade-off: Ligeramente más grande que Alpine (~40MB base), pero es el estándar *De Facto* recomendado para Python en la industria hoy en día por su tremenda estabilidad y DX (Developer Experience).
