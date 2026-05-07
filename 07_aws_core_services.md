# AWS Core Services

## 1. Explicación técnica
A nivel Senior, no basta con saber qué es un servicio de AWS, sino **cómo encaja en una arquitectura, sus límites (quotas) y su modelo de precios**. 

Servicios Core fundamentales:
- **Compute:** EC2 (Máquinas virtuales raw, máximo control, alta carga operativa), ECS/EKS (Orquestación de contenedores), Lambda (Serverless, pago por ms, ideal para workloads esporádicos o event-driven).
- **Networking:** VPC (Nubes privadas, subredes públicas/privadas, NAT Gateways), Route53 (DNS global), ALB/NLB (Load balancers de capa 7 HTTP y capa 4 TCP).
- **Storage:** S3 (Almacenamiento de objetos inmutable y masivo), EBS (Discos duros atados a instancias EC2), EFS (Sistemas de archivos compartidos).
- **Databases:** RDS (Relacional gestionada: Postgres/MySQL), DynamoDB (NoSQL clave-valor de latencia predecible de un solo dígito en milisegundos).

## 2. Uso en producción
Netflix usa S3 como su Data Lake masivo central, pero utiliza instancias EC2 especializadas con discos efímeros (Instance Store) súper rápidos para transcodificar videos. Usa DynamoDB para guardar el estado de visualización de los usuarios en tiempo real en múltiples regiones de forma activa-activa (Global Tables), asegurando que si ves una película en USA y viajas a Europa, la pausa se mantiene.

## 3. Ejemplo práctico (Python)

Interacción eficiente con AWS S3 usando `boto3` mediante URLs pre-firmadas (Presigned URLs).

```python
import boto3
from botocore.exceptions import ClientError
import logging

# Se recomienda inyectar esto o usar IAM Roles, no hardcodear credenciales.
# AWS inferirá las credenciales del entorno (EC2 Role, ECS Task Role, o ~/.aws/credentials)
s3_client = boto3.client('s3', region_name='us-east-1')
logger = logging.getLogger(__name__)

def generate_presigned_upload_url(bucket_name: str, object_name: str, expiration: int = 3600) -> dict:
    """
    Genera una URL pre-firmada para permitir a un cliente frontend subir un archivo 
    directamente a S3, SIN pasar por nuestro backend, ahorrando ancho de banda 
    y memoria del servidor Python.
    """
    try:
        response = s3_client.generate_presigned_post(
            Bucket=bucket_name,
            Key=object_name,
            Fields={"acl": "private"},
            Conditions=[
                {"acl": "private"},
                ["content-length-range", 10, 10485760]  # Limita el tamaño (10B a 10MB)
            ],
            ExpiresIn=expiration
        )
    except ClientError as e:
        logger.error(f"Error generando URL presigned: {e}")
        return None

    # El frontend hace un POST a esta URL con los campos provistos
    return response

# Uso
# upload_data = generate_presigned_upload_url("my-app-uploads", "users/avatar_123.png")
# return {"upload_url": upload_data['url'], "form_fields": upload_data['fields']}
```

## 4. Diagrama (texto)

Arquitectura Web Típica AWS (3-Tier):

         Internet
            │
      [ Route 53 (DNS) ]
            │
            ▼
     [ ALB (Load Balancer) ] ─── (Zona Pública)
        │            │
        ▼            ▼
   [ EC2/ECS ]  [ EC2/ECS ] ─── (VPC Zona Privada 1) (App Backend en Python/Django)
        │            │
        ▼            ▼
   [ Amazon RDS (Postgres) ] ── (VPC Zona Privada 2, sin acceso a Internet)
        │
   [ Multi-AZ Replica ] 

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Cuál es la diferencia entre un Application Load Balancer (ALB) y un Network Load Balancer (NLB)?
- **Pregunta 2 (Práctica):** Tienes un worker en una instancia EC2 que necesita leer archivos de un bucket S3 privado. ¿Cómo le das permisos de forma segura sin configurar credenciales de largo plazo (Access Keys) en el servidor?
- **Pregunta 3 (Arquitectura):** Tu aplicación en Python expuesta a Internet necesita procesar pagos conectándose a una pasarela externa y guardar los datos en RDS. ¿Cómo configuras las subredes de la VPC para máxima seguridad?

## 6. Respuestas esperadas
- **Respuesta 1:** ALB opera en Capa 7 (Aplicación). Entiende HTTP/HTTPS, puede leer cabeceras, rutas de URL (ej. `/api/` a un backend, `/web/` a otro), y es ideal para APIs REST. NLB opera en Capa 4 (Transporte). Solo entiende IPs y puertos TCP/UDP. Es capaz de manejar millones de peticiones por segundo con latencia ultra baja, y permite tener direcciones IP estáticas elásticas, ideal para gaming o servicios legacy TCP.
- **Respuesta 2:** Le asigno un **IAM Role** (Instance Profile) a la instancia EC2. AWS se encarga de rotar y proveer credenciales temporales automáticamente. Mi código en Python (`boto3`) o la CLI de AWS usarán esas credenciales temporales en background sin necesidad de hardcodear variables de entorno inseguras (`AWS_ACCESS_KEY_ID`).
- **Respuesta 3:** Creo dos subredes: Pública y Privada. Coloco un ALB en la subred Pública, asociado a un Internet Gateway. Coloco las instancias EC2 (mi app Python) en la subred Privada, para que no puedan ser atacadas directamente desde Internet. Coloco RDS en otra subred Privada (o la misma). Para que mis EC2 puedan salir a Internet a cobrar en la pasarela externa, coloco un **NAT Gateway** en la subred Pública y apunto el ruteo de mi subred Privada hacia él.

## 7. Errores comunes
- **Instancias EC2 "Mascotas":** Tratar a los servidores EC2 como servidores tradicionales (instalando dependencias a mano, aplicando parches vía SSH). Si la instancia muere, el sistema se cae. En la nube, EC2 deben ser tratadas como "Ganado" (Cattle, not Pets) a través de Auto Scaling Groups e imágenes (AMIs) preconfiguradas e inmutables (Packer/Docker).
- **Security Groups Permisivos:** Abrir el puerto 22 (SSH) o el 5432 (Postgres) a `0.0.0.0/0` (Todo internet). La base de datos solo debe permitir tráfico originado por el Security Group de la capa de aplicación.

## 8. Buenas prácticas
- **Least Privilege (Mínimo Privilegio):** En las políticas IAM, nunca uses `s3:*` o `ec2:*`. Limita los permisos a las acciones exactas (`s3:PutObject`) y los recursos exactos (`arn:aws:s3:::my-bucket/*`) necesarios.
- **Infrastructure as Code (IaC):** AWS consola web solo sirve para explorar o debugear en dev. Producción debe ser aprovisionada por Terraform, CloudFormation o AWS CDK para garantizar reproducibilidad y auditoría.
- **Cost Allocation Tags:** Etiqueta (Tag) cada recurso de AWS (EC2, S3, RDS) por Proyecto, Entorno (`prod`/`dev`) y Equipo. Esto es fundamental para analizar las facturas al fin de mes en AWS Cost Explorer.

## 9. Trade-offs y decisiones técnicas
**EC2 vs ECS/Fargate (Contenedores)**
- *EC2:* Excelente si tienes licencias por cores (software legacy) o dependencias súper específicas a nivel de kernel/OS (drivers de GPU custom). Trade-off: Operativamente complejo, debes parchear SO, gestionar agentes de seguridad, y el escalado es relativamente lento (tarda minutos en arrancar una AMI pesada).
- *ECS/Fargate:* Excelente para arquitecturas modernas. Fargate gestiona los servidores subyacentes por ti (Serverless containers). Entregas un contenedor Docker y defines la CPU/RAM. Trade-off: Ligeramente más caro por hora de computo puro, y menos control sobre el host subyacente. Recomendado como estándar por defecto actual (o EKS para los adeptos a Kubernetes).
