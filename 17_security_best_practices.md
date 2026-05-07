# Security Best Practices (Python/AWS)

## 1. Explicación técnica
A nivel Senior, la seguridad no se limita a usar HTTPS. Implica **Defensa en Profundidad** (Defense in Depth), asumiendo que el atacante eventualmente vulnerará alguna capa del sistema.

Principios de seguridad clave:
- **OWASP Top 10:** Inyección SQL, Broken Authentication, Exposición de Datos Sensibles, etc.
- **Principio del Mínimo Privilegio (PoLP):** Ningún componente (usuario, contenedor, Lambda) debe tener más permisos de los estrictamente necesarios.
- **Gestión de Secretos:** Nunca hardcodear claves. Usar herramientas de rotación y bóvedas seguras.
- **Validación de Entradas:** Nunca confiar en el input del usuario. Desinfectar (sanitize) y validar siempre.

## 2. Uso en producción
En entornos de alta seguridad (ej. Fintech, salud), las bases de datos de producción están en subredes privadas aisladas sin salida a Internet. El acceso SSH está prohibido o fuertemente restringido (usando AWS Systems Manager Session Manager). Las claves maestras de cifrado (KMS) rotan automáticamente cada año, y los datos sensibles en la BD (ej. PII, SSN) se guardan cifrados a nivel de aplicación (Field-Level Encryption) antes de siquiera llegar al motor de la base de datos.

## 3. Ejemplo práctico (Python)

Manejo seguro de contraseñas (Hashing asíncrono con passlib y Argon2/Bcrypt) y sanitización de inputs usando Pydantic en FastAPI.

```python
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel, EmailStr, Field
from passlib.context import CryptContext

app = FastAPI()

# Configuramos Argon2 (superior a bcrypt en resistencia contra ataques de GPU/ASIC)
# Ojo: el hashing es intensivo en CPU, puede ser un vector para ataques de DoS.
pwd_context = CryptContext(schemes=["argon2"], deprecated="auto")

class UserSignupRequest(BaseModel):
    # Validación estricta en la entrada. Previene inyecciones y payloads maliciosos.
    email: EmailStr
    # Contraseña fuerte requerida
    password: str = Field(..., min_length=12, max_length=128)
    # Evitar XSS o comandos extraños limitando el charset
    username: str = Field(..., pattern=r"^[a-zA-Z0-9_-]{3,20}$")

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

@app.post("/signup", status_code=201)
async def signup(user: UserSignupRequest):
    # Simulamos comprobación en DB
    if user.email == "admin@example.com":
        # Prevención de enumeración de usuarios:
        # A veces es mejor devolver 200 OK y enviar un email genérico ("Si estás registrado...")
        # en lugar de 409 Conflict que revela qué emails existen en la BD.
        raise HTTPException(status_code=409, detail="Email exists")

    # Hash seguro. NUNCA guardar la contraseña en texto plano o usar MD5/SHA1.
    hashed_pw = get_password_hash(user.password)
    
    # Guardar en DB... (Omitido)
    return {"message": "User created successfully", "username": user.username}
```

## 4. Diagrama (texto)

Arquitectura de Seguridad en Profundidad (AWS):

[ Internet (Atacante) ]
         │
[ AWS WAF (Web Application Firewall) ] ── (Bloquea SQLi, XSS, IPs maliciosas)
         │
[ ALB (Terminación TLS 1.3) ] ────────── (Encriptación en Tránsito)
         │
[ VPC Pública (NAT Gateway) ]
         │
[ VPC Privada (EC2 / ECS) ] ──────────── (Security Groups: Solo permite tráfico del ALB)
         │ (Usa IAM Role temporal)       (Valida Input, Hashea Passwords)
         │
[ RDS Postgres (KMS Encrypted) ] ─────── (Encriptación en Reposo)
                                         (Security Groups: Solo permite tráfico de la App)

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Qué es la inyección SQL, por qué sigue ocurriendo y cómo la previenes en Python?
- **Pregunta 2 (Práctica):** Tienes un token JWT que usas para autenticar usuarios. Un empleado es despedido y necesitas invalidar su token inmediatamente. El token expira en 24 horas. ¿Cómo lo haces si JWT es "stateless" (no requiere consultar la DB)?
- **Pregunta 3 (Arquitectura):** Se filtraron en GitHub las credenciales de administrador (Access Key y Secret) de tu cuenta de AWS. ¿Cuáles son los pasos inmediatos a seguir para contener y evaluar el daño?

## 6. Respuestas esperadas
- **Respuesta 1:** La inyección SQL ocurre cuando el input no validado del usuario se concatena directamente en una cadena de consulta SQL (ej. `query = f"SELECT * FROM users WHERE name = '{user_input}'"`), permitiendo ejecutar comandos maliciosos (`' OR 1=1; DROP TABLE users; --`). Se previene utilizando **Prepared Statements** (consultas parametrizadas) proporcionadas por ORMs como SQLAlchemy o drivers como `psycopg2` (ej. `execute("SELECT * FROM users WHERE name = %s", (user_input,))`), donde la DB separa el código (SQL) de los datos de forma segura.
- **Respuesta 2:** Dado que el JWT no se consulta en la DB por cada request (stateless), no puedes revocarlo modificando al usuario. Solución: Implementar una **Blacklist (Token Revocation List)** usando una caché de lectura rápida (ej. Redis). Al revocar, guardo el ID único del token (`jti`) en Redis con un TTL igual al tiempo que le queda de vida al token. En cada request, el API Gateway o middleware verifica rápidamente en Redis si el `jti` está bloqueado antes de aceptar el JWT.
- **Respuesta 3:** 1) Ir a AWS IAM y **Desactivar (Make Inactive)** o Eliminar inmediatamente la Access Key comprometida (Contención). 2) Revisar en AWS CloudTrail los logs de actividad recientes de esa Access Key para ver qué acciones realizó el atacante (ej. crear instancias de minado de criptomonedas, exfiltrar S3, crear backdoors en IAM) (Auditoría). 3) Revocar sesiones activas, rotar contraseñas/secretos a los que ese usuario tuviera acceso y notificar según la política de incidentes (Remediación).

## 7. Errores comunes
- **Guardar secretos en el repo:** Subir archivos `.env` o configuraciones con contraseñas en Git. (Usar GitGuardian o herramientas de escaneo pre-commit para evitarlo).
- **IDOR (Insecure Direct Object Reference):** Un endpoint como `GET /invoices/1234` no verifica si la factura `1234` pertenece al usuario autenticado. Un atacante puede iterar `/invoices/1235`, `/1236`, descargando facturas ajenas. Siempre verificar la propiedad (`WHERE id=1234 AND user_id=current_user.id`).
- **Lanzar excepciones detalladas al cliente:** Devolver trazas (Tracebacks) de Python crudas en un HTTP 500 (`"OperationalError: column 'ssn' does not exist"`). Revelan esquema de DB interno y versiones de software. En producción, devolver mensajes genéricos ("Internal Server Error").

## 8. Buenas prácticas
- **HTTPS Everywhere:** Forzar redirecciones de HTTP a HTTPS a nivel de Load Balancer y configurar HSTS (HTTP Strict Transport Security).
- **Manejo seguro de dependencias:** Correr `pip audit` o herramientas similares en el pipeline CI para detectar librerías de terceros con CVEs (Vulnerabilidades) conocidas.
- **CORS restrictivo:** No usar `*` en cabeceras de CORS en APIs productivas. Especificar los orígenes exactos permitidos (ej. `https://app.midominio.com`).

## 9. Trade-offs y decisiones técnicas
**JWT (Stateless) vs Sesiones en Servidor (Stateful/Redis)**
- *JWT:* Escala perfectamente porque el servidor no necesita guardar nada en memoria; el token en sí mismo certifica la identidad. Trade-off: Son inflexibles (difíciles de invalidar antes de expirar), engordan el tamaño de los headers HTTP (cientos de bytes en cada request), y si la clave secreta del servidor (Secret Key) se filtra, el atacante puede falsificar identidades de cualquier usuario.
- *Sesiones (Server-side):* Guardas una cookie de sesión (Session ID opaco corto) y mantienes el estado del usuario en Redis. Trade-off: Requiere infraestructura adicional (Redis), cada request hace un lookup a la caché. Ventaja masiva: Revocación de sesión instantánea, control absoluto sobre el ciclo de vida de la sesión (logout forzado), mayor seguridad contra robo de token (fácil rotación).
