# CI/CD (Continuous Integration & Continuous Deployment)

## 1. Explicación técnica
CI/CD es la columna vertebral de la cultura DevOps moderna. Su propósito es reducir el riesgo de los despliegues de software al automatizar la validación y la entrega.
- **CI (Continuous Integration):** Frecuencia alta de merges a la rama principal (`main`). Cada commit dispara un pipeline automatizado que ejecuta linters, análisis estático (SAST), y tests unitarios/integración. El objetivo es dar retroalimentación inmediata al desarrollador si introdujo un bug.
- **CD (Continuous Delivery):** Empaquetar y dejar listo para desplegar (ej. subir imagen Docker a ECR). Requiere un clic manual (aprobación) para ir a producción.
- **CD (Continuous Deployment):** Despliegue 100% automatizado hacia Producción si todos los tests pasan. Cero intervención humana. 

Estrategias de despliegue:
- **Rolling Update:** Apaga instancias viejas progresivamente y levanta nuevas. Cero downtime.
- **Blue/Green Deployment:** Tienes dos entornos idénticos (Azul=Prod, Verde=Nuevo). Despliegas en el Verde. Cambias el DNS/Load Balancer para que apunte al Verde al instante. Rollback inmediato si algo falla cambiando el DNS al Azul otra vez.
- **Canary Release:** Rutear el 5% del tráfico a los nuevos servidores. Monitorear errores (CloudWatch/Datadog). Si es estable, subir al 100%.

## 2. Uso en producción
En un equipo maduro de backend, nadie corre `pytest` o crea un contenedor Docker en su máquina para subirlo. Cuando abres un Pull Request (PR) en GitHub, GitHub Actions levanta servicios en caché (Postgres, Redis), ejecuta `black`/`flake8` (linters), `mypy` (tipado estricto) y `pytest` con cobertura. Solo si pasa todo, permite el Merge. Tras el merge a `main`, construye la imagen Docker, la pushea a Amazon ECR (Elastic Container Registry) y dispara un update a ECS (Elastic Container Service).

## 3. Ejemplo práctico (YAML)

Un pipeline realista de GitHub Actions para un proyecto backend en Python con AWS. Omitimos las credenciales a favor del moderno OIDC (OpenID Connect).

```yaml
name: CI/CD Backend Pipeline

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

# Permisos para usar OIDC con AWS
permissions:
  id-token: write
  contents: read

jobs:
  test-and-lint:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python 3.11
      uses: actions/setup-python@v4
      with:
        python-version: "3.11"
        cache: 'pip' # Acelera drásticamente las builds
        
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements-dev.txt
        
    - name: Run Linters & Type Checking
      run: |
        flake8 src/
        mypy src/
        
    - name: Run Tests with Pytest
      # Idealmente aquí configurarías servicios de base de datos en background si hay tests de integración
      run: |
        pytest tests/ --cov=src --cov-report=xml
        
  build-and-deploy:
    needs: test-and-lint
    # Solo ejecutar en push a main, no en Pull Requests
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    # Best practice: Asumir rol mediante OIDC, sin hardcodear AWS_ACCESS_KEY
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsDeployRole
        aws-region: us-east-1

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build, tag, and push image to Amazon ECR
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: my-python-api
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

    # Ejemplo desplegando en AWS ECS
    - name: Render Amazon ECS task definition
      id: task-def
      uses: aws-actions/amazon-ecs-render-task-definition@v1
      with:
        task-definition: ecs-task-def.json
        container-name: python-api-container
        image: ${{ steps.login-ecr.outputs.registry }}/my-python-api:${{ github.sha }}

    - name: Deploy to Amazon ECS
      uses: aws-actions/amazon-ecs-deploy-task-definition@v1
      with:
        task-definition: ${{ steps.task-def.outputs.task-definition }}
        service: python-api-service
        cluster: production-cluster
        wait-for-service-stability: true
```

## 5. Preguntas de entrevista
- **Pregunta 1 (Conceptual):** ¿Cuál es la diferencia entre Continuous Delivery y Continuous Deployment? ¿Por qué la mayoría de corporaciones usan la primera?
- **Pregunta 2 (Arquitectura):** Estás migrando tu API de un despliegue monolítico a un modelo de CI/CD masivo. El equipo requiere cambiar la base de datos (renombrar una columna) en este release. ¿Cómo gestionas el despliegue del esquema de DB para no causar *downtime*?
- **Pregunta 3 (Práctica):** En GitHub Actions, estás usando Secrets (`AWS_ACCESS_KEY_ID`) configurados en los settings del repo. ¿Por qué esto ya se considera una mala práctica de seguridad y cuál es la alternativa moderna?

## 6. Respuestas esperadas
- **Respuesta 1:** Continuous Delivery automatiza todo el empaquetado y subida de artefactos a los repositorios (ECR/S3), pero requiere la aprobación manual de un humano (botón "Deploy") para aplicar los cambios a producción. Continuous Deployment automatiza el paso final de despliegue sin intervención. Corporaciones grandes prefieren *Delivery* por razones de cumplimiento (Compliance/Auditoría, ej. SOC2/PCI), ventanas de despliegue planificadas, o para alinear releases con lanzamientos de Marketing.
- **Respuesta 2:** Los cambios destructivos en bases de datos (borrar/renombrar columnas) son los asesinos del Zero Downtime en CI/CD. Nunca se pueden hacer en un solo paso. Usaría el patrón **Expand and Contract**: 
  1. Release 1: Añado la nueva columna. La app escribe en ambas.
  2. Release 2: Migro datos históricos a la nueva columna en background.
  3. Release 3: Cambio el código para leer de la nueva columna.
  4. Release 4: Borro la columna vieja. Cada despliegue mantiene compatibilidad hacia atrás total.
- **Respuesta 3:** Las credenciales a largo plazo almacenadas estáticamente son un vector de ataque enorme (si el repo es comprometido, pueden vaciar tu cuenta de AWS). La alternativa moderna es usar **OIDC (OpenID Connect)**. Se configura una relación de confianza en AWS IAM indicando "Confía en los repositorios de mi org de GitHub". El Pipeline solicita un token temporal (STS) usando firmas criptográficas de GitHub al momento de la ejecución. Es *passwordless*, las credenciales duran 1 hora y se expiran solas.

## 7. Errores comunes
- **Flaky Tests:** Tests de integración que a veces pasan y a veces fallan porque dependen del orden de ejecución o de llamadas de red a terceros sin simular (mockear). Esto causa pérdida total de confianza en el CI y los developers empiezan a usar `git push --no-verify` o ignorar las alertas.
- **Pipelines monolíticos que tardan 40 minutos:** Tener un CI que tarda 40 minutos rompe la productividad (el ciclo de feedback es muy lento). Optimizar usando *Docker Layer Caching*, paralelizando jobs (`matrix`), e instalando herramientas rápidamente en lugar de compilar todo desde cero (evitar `apt-get install` masivos en cada run).

## 8. Buenas prácticas
- **Fail Fast (Falla Rápido):** Pon las validaciones más rápidas al principio del pipeline. (Ej. formato `black`, linters `flake8`, escaneo de vulnerabilidades en dependencias `bandit`/`safety`). No tiene sentido ejecutar 10 minutos de tests si hay un error de sintaxis evitable.
- **GitFlow vs Trunk-based Development:** Para CI/CD verdadero, abandonar GitFlow (ramas dev, ramas release infinitas) a favor de **Trunk-based development** (Ramas de feature cortas, de corta vida, que se integran diariamente a `main`). Promueve la Integración Continua real.

## 9. Trade-offs y decisiones técnicas
**Feature Flags (Toggles) vs Feature Branches largas**
- *Ramas largas (Long-lived branches):* Desarrollar una épica enorme (ej. nuevo sistema de pagos) en una rama durante 3 meses. Trade-off: Al intentar integrar a `main`, enfrentar un "Merge Conflict Hell" masivo que requiere semanas para resolverse y estabilizarse, deteniendo otras entregas.
- *Feature Flags:* El código del nuevo sistema de pagos se sube a `main` diariamente y se despliega a producción oculto tras un condicional (`if feature_flags.is_enabled('new_payments'): ...`). Trade-off: Introduce ligera deuda técnica/complejidad temporal en el código (ifs), requiere un servicio gestor (LaunchDarkly o AppConfig), pero permite CI masivo y pruebas en producción controladas habilitando el flag solo para empleados o usuarios beta.
