# AI & Automation Enterprise Architecture

## 1. Explicación técnica
Dado que tu entrevistador es un **AI Automation Enterprise Architect**, evaluará cómo conectas el mundo del Backend tradicional (Python/AWS) con los flujos de trabajo de Inteligencia Artificial Generativa a nivel corporativo.

Los temas clave en los que un Backend Senior debe brillar aquí son:
- **Orquestación de LLMs (Large Language Models):** Integración de APIs (OpenAI, Anthropic, AWS Bedrock) usando llamadas a herramientas (Function/Tool Calling).
- **RAG (Retrieval-Augmented Generation) a Escala:** Ingesta asíncrona de documentos, generación de *embeddings*, y almacenamiento en Vector Databases (pgvector en RDS, OpenSearch Serverless).
- **Asincronía Crítica:** Las llamadas a LLMs son inherentemente **muy lentas** (pueden tardar 5-30 segundos). Un backend moderno no puede bloquear un hilo HTTP esperando al LLM. Requiere WebSockets, Server-Sent Events (SSE) o Event-Driven Architecture (SQS/Step Functions).
- **AI Agents vs State Machines:** Diferencia entre dejar que un LLM decida el flujo (Agentic) vs usar máquinas de estado deterministas (AWS Step Functions) donde el LLM solo es un paso del proceso.

## 2. Uso en producción
Klarna automatiza el 70% de su atención al cliente. Un webhook de Zendesk entra por API Gateway a AWS SQS. Un pool de workers en Python (ECS) procesa el mensaje, busca historial del cliente en DynamoDB y documentos de políticas en una Vector DB (OpenSearch). Con este contexto (RAG), llama asíncronamente a AWS Bedrock (Claude 3) pidiendo una respuesta estructurada en JSON. Si la respuesta sugiere un reembolso, el código Python ejecuta la función (Tool Calling) contra la API de pagos, y finalmente envía la respuesta al usuario.

## 3. Ejemplo práctico (Python)

Implementación robusta de **Tool/Function Calling** con control de fallos y *streaming* asíncrono, aislando la lógica de negocio del LLM.

```python
import asyncio
import json
import httpx
from typing import Dict, Any, Callable

# --- 1. Definición de la Herramienta (Backend Real) ---
async def fetch_user_billing(user_id: str) -> str:
    """Simula una consulta lenta a RDS o API de facturación"""
    await asyncio.sleep(1) # Latencia BD
    # Lógica de base de datos aquí...
    return json.dumps({"status": "paid", "amount": 150.0, "currency": "USD"})

# Registro de herramientas disponibles para el LLM
AVAILABLE_TOOLS: Dict[str, Callable] = {
    "get_billing_info": fetch_user_billing
}

# --- 2. Orquestador LLM Asíncrono ---
async def agent_handle_message(user_message: str) -> str:
    """
    Se comunica con un LLM (ej. OpenAI/Bedrock). 
    Si el LLM decide usar una herramienta, el Backend (Python) la ejecuta y le devuelve el resultado.
    """
    # Payload simulado de lo que enviamos a la API del LLM
    system_prompt = "Eres un asistente de soporte. Usa herramientas si necesitas datos reales."
    tools_schema = [{
        "type": "function",
        "function": {
            "name": "get_billing_info",
            "description": "Obtiene el estado de facturación de un usuario.",
            "parameters": {"type": "object", "properties": {"user_id": {"type": "string"}}}
        }
    }]
    
    async with httpx.AsyncClient(timeout=30.0) as client:
        # 1ra Llamada al LLM
        # response = await client.post("https://api.openai.com/v1/chat/completions", json={...})
        
        # --- Simulación de la respuesta del LLM decidiendo usar la herramienta ---
        llm_response = {
            "finish_reason": "tool_calls",
            "message": {
                "tool_calls": [{"id": "call_123", "function": {"name": "get_billing_info", "arguments": '{"user_id": "usr_99"}'}}]
            }
        }
        
        if llm_response["finish_reason"] == "tool_calls":
            for tool_call in llm_response["message"]["tool_calls"]:
                func_name = tool_call["function"]["name"]
                args = json.loads(tool_call["function"]["arguments"])
                
                # Ejecución segura y dinámica de la función Python
                if func_name in AVAILABLE_TOOLS:
                    try:
                        # Ejecuta la lógica del backend
                        tool_result = await AVAILABLE_TOOLS[func_name](**args)
                        
                        # 2da Llamada al LLM: Entregarle el resultado de la base de datos
                        # return await client.post(..., json={"role": "tool", "content": tool_result})
                        return f"El LLM generó la respuesta final basándose en: {tool_result}"
                    except Exception as e:
                        # Fallback robusto si la API interna falla
                        return "Error interno al consultar la facturación."
                        
        return llm_response["message"].get("content", "")

# Ejecución (Normalmente disparada por un endpoint de FastAPI o worker SQS)
# asyncio.run(agent_handle_message("¿Está pagada mi factura? Mi id es usr_99"))
```

## 4. Diagrama (texto)

Arquitectura Enterprise RAG + Tool Calling en AWS:

[ User App ] ──(WebSocket/SSE)──► [ API Gateway ] ──► [ ECS Fargate (Python API) ]
                                                            │
         ┌──────────────────────────────────────────────────┤
         ▼                                                  ▼
[ OpenSearch (Vector DB) ] ◄── (Busca Contexto)      [ AWS Bedrock / OpenAI ]
         ▲                                           (LLM decide llamar API de pagos)
         │                                                  │
[ AWS SQS / Lambda ] (Ingesta asíncrona de PDFs)            ▼
         ▲                                           [ Python ejecuta DB/API call ]
         │                                                  │
[ Amazon S3 ] (Data Lake)                                   ▼
                                                     [ Retorna contexto final al LLM ]

## 5. Preguntas de entrevista
- **Pregunta 1 (Arquitectura):** Cuando dependes de la API de OpenAI o Anthropic, estás sujeto a sus límites de Rate Limiting (Tokens/Minute) y caídas temporales. Como Arquitecto, ¿cómo diseñas el sistema backend para que esto no rompa tu aplicación empresarial?
- **Pregunta 2 (Práctica):** Tienes un pipeline RAG donde los usuarios suben PDFs de 500 páginas. Necesitas generar embeddings y guardarlos en una base de datos vectorial. ¿Cómo lo procesarías en AWS usando Python sin bloquear la interfaz del usuario?
- **Pregunta 3 (Seguridad):** Tu LLM tiene acceso a una función (`Tool Calling`) que ejecuta `DELETE FROM orders WHERE id = X`. ¿Cómo garantizas que el LLM no sea engañado (Prompt Injection) para borrar órdenes de otros usuarios?

## 6. Respuestas esperadas
- **Respuesta 1:** Implementaría múltiples capas de resiliencia: 1) **Exponential Backoff & Jitter** usando librerías como `tenacity` en Python para reintentar errores 429. 2) **Model Fallback**: Si GPT-4o falla o rate-limitea, el código automáticamente hace fallback a un modelo más pequeño o alojado localmente (ej. Claude 3 Haiku en Bedrock o Llama 3). 3) **Semantic Caching**: Usar Redis + Embeddings para responder preguntas idénticas desde la caché sin llamar al LLM (ej. GPTCache), ahorrando coste y latencia.
- **Respuesta 2:** Totalmente asíncrono y Event-Driven. 1) El usuario sube el PDF a **S3** vía Presigned URL. 2) S3 emite un evento a **SQS**. 3) Un worker en Python (AWS Lambda o ECS) lee la cola, extrae el texto (Textract/PyPDF), hace el *Chunking* (dividir en fragmentos de 1000 tokens conservando solapamiento/overlap), llama al modelo de Embeddings, y hace un Bulk Insert o Upsert (Pinecone/pgvector/OpenSearch). 4) Actualiza una tabla en DynamoDB a "READY" y envía una notificación (Websocket/SNS) al usuario.
- **Respuesta 3:** El LLM **nunca** debe tener autorización directa sobre la base de datos ni conocer IDs reales si no es necesario. El Backend en Python es el mediador. 1) **Validación de Identidad (AuthZ):** Si el LLM pide borrar la orden `X`, el backend Python verifica que `user_session.id == order_X.owner_id` antes de ejecutar la query. 2) **Human-in-the-Loop:** Para acciones destructivas dictadas por LLMs, el backend pausa la ejecución y envía una notificación de aprobación (ej. push notification o Slack) al usuario humano real, y solo procede si el humano acepta la acción (State Machine - Callback pattern).

## 7. Errores comunes
- **Depender de LangChain en Producción (Over-coupling):** Usar wrappers genéricos de LangChain (como `SQLDatabaseChain`) para que el LLM construya y ejecute consultas SQL directamente contra producción. Es un riesgo de seguridad masivo, ineficiente, y difícil de debugear. A nivel Enterprise, el LLM solo debe extraer parámetros; el Python Backend ejecuta las queries de forma controlada y segura.
- **Timeout en API Gateway:** Configurar una API REST síncrona que llama a un LLM. AWS API Gateway tiene un límite estricto de timeout de 29 segundos. Si el LLM tarda 35 segundos en generar la respuesta, el cliente recibe un 504 Gateway Timeout aunque la API de Python siga trabajando en background. *Solución: Devolver un HTTP 202 Accepted, un Task ID, y responder por WebSockets o Polling, o usar SSE (Streaming).*
- **Filtración de PII (Data Privacy):** Enviar datos como tarjetas de crédito o emails reales a APIs de LLMs comerciales. *Solución: Usar librerías de Data Masking (ej. Presidio) en Python para anonimizar `[EMAIL_1]` antes de enviar el prompt, y des-anonimizar la respuesta antes de devolverla al usuario.*

## 8. Buenas prácticas
- **Separación de Prompts como Código:** No hardcodear Prompts masivos de 300 líneas dentro de funciones Python. Tratarlos como plantillas o configuración. Guardarlos en AWS Parameter Store, S3 o herramientas como LangSmith para que los ingenieros de Prompt puedan modificarlos sin hacer un re-deploy del código backend.
- **LLM Observability:** Un log tradicional de "Success 200" no sirve. Debes instrumentar y registrar el *Prompt original*, la *Respuesta cruda*, el *Token Count* (para facturación), y la *Latencia*. Usar herramientas como Phoenix, Langfuse o Datadog LLM Observability.

## 9. Trade-offs y decisiones técnicas
**Agentic Workflows vs AWS Step Functions (Deterministas)**
- *AI Agents (ReAct, AutoGPT):* El LLM actúa como el cerebro. Le das un objetivo y él decide en bucle qué herramientas usar ("Piensa -> Actúa -> Observa") hasta terminar. Trade-off: Altamente impredecible, propenso a bucles infinitos carísimos (costos disparados), difícil de probar unitariamente. Fantástico para investigación abierta o análisis de datos complejos.
- *Orquestación Determinista (Step Functions):* AWS define un grafo de estados estricto. Paso 1: Lambda extrae datos. Paso 2: Llama LLM para resumir. Paso 3: Guarda en S3. Trade-off: Menos flexible, el LLM no puede "improvisar" si falla algo. Ventaja Enterprise: 100% predecible, auditable, fácil manejo de errores, reintentos nativos y costos controlados. Es el estándar preferido por Enterprise Architects para automatización segura.
