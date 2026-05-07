# Kubernetes (K8s) Intro for Backend Engineers

## 1. Explicación técnica
Kubernetes (K8s) es un orquestador de contenedores de código abierto. Si Docker define *cómo* empaquetar y ejecutar un proceso aislado, K8s define *dónde*, *cuándo* y *cuántos* de esos procesos deben ejecutarse a través de un clúster de máquinas.

A nivel Senior, debes entender su arquitectura:
- **Control Plane:** El cerebro (API Server, etcd, Scheduler, Controller Manager). Decide qué debe correr y dónde.
- **Worker Nodes:** Las máquinas físicas/virtuales (EC2) que ejecutan la carga (Kubelet, Kube-proxy, Container Runtime).
- **Pod:** La unidad más pequeña desplegable. Agrupa uno o más contenedores que comparten red y almacenamiento.
- **Deployment:** Gestiona la creación de Pods (ReplicaSets) y facilita las actualizaciones sin tiempo de inactividad (Rolling Updates).
- **Service:** Provee una IP estática y balanceo de carga interno a un grupo de Pods dinámicos.
- **Ingress:** Expone rutas HTTP/HTTPS desde el exterior hacia los Services internos.

## 2. Uso en producción
Tinder migró su monolito a microservicios en Kubernetes (EKS en AWS) para soportar la enorme volatilidad de tráfico (ej. picos masivos durante la noche). K8s les permite usar **Horizontal Pod Autoscaler (HPA)** para añadir automáticamente miles de Pods de su API en Python/Node cuando el uso de CPU supera el 70%, y usar **Cluster Autoscaler** para que AWS encienda nuevas instancias EC2 físicas subyacentes cuando no hay espacio para más Pods.

## 3. Ejemplo práctico (YAML)

Un manifiesto declarativo de K8s (`Deployment` y `Service`) para una API en FastAPI. 

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-python-api
  labels:
    app: python-api
spec:
  replicas: 3 # Mantenemos 3 instancias corriendo por HA
  selector:
    matchLabels:
      app: python-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1       # Cuántos pods extra pueden crearse al actualizar
      maxUnavailable: 0 # Nunca bajar de las replicas deseadas durante update
  template:
    metadata:
      labels:
        app: python-api
    spec:
      containers:
      - name: fastapi-app
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-api:v1.2.0
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        # Liveness: ¿El pod está vivo o crasheó internamente? (K8s lo reinicia si falla)
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 15
        # Readiness: ¿El pod está listo para recibir tráfico de usuarios?
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: backend-python-service
spec:
  type: ClusterIP # Solo accesible dentro del cluster (usaremos un Ingress para salir)
  selector:
    app: python-api
  ports:
    - protocol: TCP
      port: 80       # Puerto expuesto por el Service
      targetPort: 8000 # Puerto del contenedor
```

## 4. Diagrama (texto)

Arquitectura de Red K8s:

       Internet
          │
  [ Ingress Controller / ALB ] ── (Reglas de ruteo ej. /api/v1 -> Service A)
          │
  [ Kubernetes Service ] ── (IP Estática Interna, Balanceo de Carga L4)
       /      |      \
  [ Pod 1 ] [ Pod 2 ] [ Pod 3 ] ── (IPs Dinámicas, Efímeros, Auto-escalables)
  (Node 1)  (Node 1)  (Node 2)

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Cuál es la diferencia entre un `Liveness Probe` y un `Readiness Probe` y por qué necesitas ambos en una aplicación Python?
- **Pregunta 2 (Práctica):** Haces deploy de tu app y el Pod se queda en estado `CrashLoopBackOff`. ¿Cómo haces troubleshooting para encontrar el problema?
- **Pregunta 3 (Arquitectura):** Tienes un worker de Celery (Python) que procesa trabajos de una cola SQS. ¿Deberías usar un HPA basado en CPU para escalar este worker en K8s?

## 6. Respuestas esperadas
- **Respuesta 1:** El `Liveness Probe` verifica si el contenedor está bloqueado (ej. un deadlock en el GIL de Python). Si falla, K8s **reinicia** (mata y recrea) el contenedor. El `Readiness Probe` verifica si el contenedor puede atender tráfico de red (ej. está vivo, pero perdió conexión a la base de datos). Si falla, K8s **no lo mata**, pero lo **quita del Service**, dejando de enviarle peticiones de usuarios hasta que se recupere. Necesito ambos para asegurar resiliencia total y cero caídas visibles.
- **Respuesta 2:** Primero revisaría el estado del pod con `kubectl describe pod <nombre>` para ver si hay errores del Scheduler (ej. falta de memoria/CPU en los nodos) o del Pull de imagen (ej. `ImagePullBackOff`). Si el pod arrancó pero murió, miraría los logs del contenedor con `kubectl logs <nombre> --previous` (para ver los logs de la instancia que crasheó, no de la actual intentando arrancar). Podría ser un error de sintaxis de Python, un modulo faltante, o una variable de entorno `DATABASE_URL` ausente.
- **Respuesta 3:** No. Un HPA (Horizontal Pod Autoscaler) basado en CPU es ineficaz para workers de colas asíncronas porque el worker siempre consumirá la CPU disponible mientras haya mensajes, pero no indica el tamaño real del atraso (backlog). La práctica estándar es usar **KEDA** (Kubernetes Event-driven Autoscaling) para escalar basándose en una métrica externa (ej. `ApproximateNumberOfMessagesVisible` de SQS). Si la cola tiene 10,000 mensajes, KEDA puede escalar a 50 Pods; si tiene 0, puede escalar a 0 Pods (ahorrando dinero).

## 7. Errores comunes
- **Omitir Request/Limits de Recursos:** Si despliegas Pods en K8s sin definir `resources.requests` y `resources.limits`, un solo Pod de Python con un memory leak puede consumir toda la RAM del Node EC2, provocando que el kernel de Linux (OOM Killer) mate procesos aleatorios del sistema operativo y derribe todo el nodo.
- **Guardar estado local:** Confiar en que el disco del Pod es persistente. Los Pods son ganando, no mascotas. Si el Pod muere, el disco efímero se borra. El estado (DBs, Cachés, uploads de usuarios) debe vivir en servicios gestionados externos (RDS, S3) o en PersistentVolumes (EBS).

## 8. Buenas prácticas
- **Graceful Shutdown (SIGTERM):** Cuando K8s hace una actualización, envía un `SIGTERM` al Pod y espera un periodo de gracia (por defecto 30s) antes de enviar `SIGKILL`. Tu app Python debe capturar esta señal, dejar de aceptar nuevas requests, terminar de procesar las actuales y cerrar conexiones a BD limpiamente.
- **Secrets Management:** Nunca guardar contraseñas en texto plano en los ConfigMaps o YAMLs de K8s. Integrar herramientas como External Secrets Operator, AWS Secrets Manager o HashiCorp Vault para inyectar variables de entorno de forma segura en runtime.

## 9. Trade-offs y decisiones técnicas
**Kubernetes (EKS) vs ECS Fargate (AWS nativo)**
- *EKS (Kubernetes):* Es el estándar universal (cloud-agnostic). Ecosistema inmenso (Helm, Prometheus operator, Istio). Ideal si tienes un equipo grande de DevOps y requieres control absoluto sobre networking y service mesh. Trade-off: Curva de aprendizaje brutal ("Day 2 Operations" muy complejas), y pagas un "Cluster Control Plane Fee" fijo en AWS (~$70/mes solo por existir), más los EC2 de los nodos.
- *ECS Fargate:* Súper simple, nativo de AWS. No hay "Control Plane" que gestionar, no hay instancias EC2 que parchear. Defines la Task (Contenedor) y AWS lo ejecuta. Trade-off: Vendor lock-in total con AWS. Menos opciones de personalización avanzada de red (DaemonSets, Service Mesh complejos). A nivel Senior, se recomienda ECS a menos que la empresa ya tenga un equipo de Plataforma gestionando Kubernetes para todos.
