# Backend Innovatech — Despliegue en AWS EKS con ECR y GitHub Actions

Este repositorio contiene los microservicios backend de Innovatech, desarrollados con Spring Boot y desplegados en AWS EKS mediante contenedores Docker, Amazon ECR y GitHub Actions.

El backend está compuesto por los siguientes microservicios:

* `ventas`: microservicio encargado de la gestión de ventas.
* `despachos`: microservicio encargado de la gestión de despachos.

Ambos servicios se ejecutan dentro de un clúster Kubernetes en AWS EKS y se comunican con una base de datos MySQL/MariaDB alojada en una instancia EC2 privada.

---

## 1. Arquitectura general

El backend funciona como la capa de servicios internos de la aplicación.

```text
Frontend Nginx
   ↓
ventas-service / despachos-service
   ↓
Pods backend en EKS
   ↓
MySQL/MariaDB en EC2 privada
```

Los microservicios no se exponen directamente a Internet. Solo son accesibles dentro del clúster mediante services tipo ClusterIP.

```text
ventas-service      → puerto 8086
despachos-service   → puerto 8085
```

---

## 2. Requisitos previos

Antes de desplegar este backend, se asume que ya existen y están correctamente configurados:

* VPC en AWS.
* Subredes públicas y privadas.
* Security Groups.
* Clúster EKS creado.
* Nodos activos en EKS.
* Namespace `innovatech`.
* Base de datos MySQL/MariaDB en EC2 privada.
* Acceso desde EKS hacia la base de datos por el puerto 3306.
* Repositorio ECR para backend.
* GitHub Actions habilitado.
* GitHub Secrets configurados.

También se requiere tener instalado localmente:

```bash
aws --version
kubectl version --client
git --version
```

---

## 3. Crear repositorio en Amazon ECR

Antes de ejecutar el pipeline, se debe crear el repositorio del backend en Amazon ECR.

Nombre del repositorio:

```text
backend-innovatech
```

Crear repositorio:

```bash
aws ecr create-repository \
  --repository-name backend-innovatech \
  --region us-east-1
```

Verificar repositorios:

```bash
aws ecr describe-repositories \
  --region us-east-1
```

Las imágenes del backend se publican en el mismo repositorio usando distintos tags:

```text
backend-innovatech:ventas
backend-innovatech:despachos
```

URLs esperadas:

```text
308769698189.dkr.ecr.us-east-1.amazonaws.com/backend-innovatech:ventas
308769698189.dkr.ecr.us-east-1.amazonaws.com/backend-innovatech:despachos
```

---

## 4. Estructura esperada del repositorio

```text
Back-Innovatech/
│
├── back-Ventas_SpringBoot/
│   ├── Dockerfile
│   └── src/
│
├── back-Despachos_SpringBoot/
│   ├── Dockerfile
│   └── src/
│
├── k8s/
│   ├── namespace.yaml
│   ├── ventas-deployment.yaml
│   ├── despachos-deployment.yaml
│   └── hpa.yaml
│
└── .github/
    └── workflows/
        └── deploy.yml
```

---

## 5. Namespace Kubernetes

Todos los recursos se despliegan en el namespace:

```text
innovatech
```

Archivo:

```text
k8s/namespace.yaml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: innovatech
```

Aplicar manualmente:

```bash
kubectl apply -f k8s/namespace.yaml
```

Verificar:

```bash
kubectl get namespaces
```

---

## 6. Base de datos MySQL/MariaDB en EC2 privada

La base de datos debe estar instalada en una EC2 privada.

Datos utilizados por la aplicación:

```text
DB_NAME=db_innovatech
DB_USERNAME=api_user
DB_PASSWORD=********
DB_ENDPOINT=IP_PRIVADA_DE_LA_EC2_DB
```

Ejemplo de creación de base de datos y usuario:

```sql
CREATE DATABASE db_innovatech;

CREATE USER 'api_user'@'%' IDENTIFIED BY 'api_password_segura';

GRANT ALL PRIVILEGES ON db_innovatech.* TO 'api_user'@'%';

FLUSH PRIVILEGES;
```

Para que MariaDB inicie automáticamente:

```bash
sudo systemctl enable mariadb
sudo systemctl start mariadb
sudo systemctl status mariadb
```

Si se usa MySQL:

```bash
sudo systemctl enable mysqld
sudo systemctl start mysqld
sudo systemctl status mysqld
```

---

## 7. GitHub Secrets requeridos

En el repositorio de GitHub se deben configurar los siguientes secrets:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_SESSION_TOKEN
AWS_ACCOUNT_ID
AWS_REGION
DB_ENDPOINT
DB_NAME
DB_USERNAME
DB_PASSWORD
```

Ejemplo:

```text
AWS_ACCOUNT_ID=308769698189
AWS_REGION=us-east-1
DB_ENDPOINT=10.0.X.X
DB_NAME=db_innovatech
DB_USERNAME=api_user
DB_PASSWORD=********
```

En AWS Academy las credenciales son temporales. Si el pipeline falla por autenticación, se deben actualizar las credenciales y el `AWS_SESSION_TOKEN`.

---

## 8. Kubernetes Secret para la base de datos

El pipeline crea o actualiza automáticamente el secret:

```text
mysql-secret
```

Este secret contiene:

```text
DB_ENDPOINT
SPRING_DATASOURCE_URL
SPRING_DATASOURCE_USERNAME
SPRING_DATASOURCE_PASSWORD
```

Comando usado:

```bash
kubectl create secret generic mysql-secret \
  --from-literal=DB_ENDPOINT="${{ secrets.DB_ENDPOINT }}" \
  --from-literal=SPRING_DATASOURCE_URL="jdbc:mysql://${{ secrets.DB_ENDPOINT }}:3306/${{ secrets.DB_NAME }}?allowPublicKeyRetrieval=true&useSSL=false&serverTimezone=UTC" \
  --from-literal=SPRING_DATASOURCE_USERNAME="${{ secrets.DB_USERNAME }}" \
  --from-literal=SPRING_DATASOURCE_PASSWORD="${{ secrets.DB_PASSWORD }}" \
  -n innovatech \
  --dry-run=client -o yaml | kubectl apply -f -
```

Verificar secret:

```bash
kubectl get secret mysql-secret -n innovatech
```

---

## 9. Microservicio ventas

El microservicio `ventas` se ejecuta en el puerto:

```text
8086
```

Imagen:

```text
308769698189.dkr.ecr.us-east-1.amazonaws.com/backend-innovatech:ventas
```

Service interno:

```text
ventas-service:8086
```

Endpoint principal:

```text
/api/v1/ventas
```

---

## 10. Microservicio despachos

El microservicio `despachos` se ejecuta en el puerto:

```text
8085
```

Imagen:

```text
308769698189.dkr.ecr.us-east-1.amazonaws.com/backend-innovatech:despachos
```

Service interno:

```text
despachos-service:8085
```

Endpoint principal:

```text
/api/v1/despachos
```

---

## 11. Deployment de ventas

Archivo:

```text
k8s/ventas-deployment.yaml
```

Ejemplo:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ventas
  namespace: innovatech
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ventas
  template:
    metadata:
      labels:
        app: ventas
    spec:
      containers:
        - name: ventas
          image: 308769698189.dkr.ecr.us-east-1.amazonaws.com/backend-innovatech:ventas
          ports:
            - containerPort: 8086
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: prod
            - name: SPRING_DATASOURCE_URL
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: SPRING_DATASOURCE_URL
            - name: SPRING_DATASOURCE_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: SPRING_DATASOURCE_USERNAME
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: SPRING_DATASOURCE_PASSWORD
            - name: SPRING_DATASOURCE_HIKARI_MAXIMUM_POOL_SIZE
              value: "3"
            - name: SPRING_DATASOURCE_HIKARI_MINIMUM_IDLE
              value: "1"
            - name: SPRING_DATASOURCE_HIKARI_CONNECTION_TIMEOUT
              value: "30000"
            - name: SPRING_DATASOURCE_HIKARI_IDLE_TIMEOUT
              value: "120000"
            - name: SPRING_DATASOURCE_HIKARI_MAX_LIFETIME
              value: "300000"
            - name: SPRING_DATASOURCE_HIKARI_KEEPALIVE_TIME
              value: "60000"
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
---
apiVersion: v1
kind: Service
metadata:
  name: ventas-service
  namespace: innovatech
spec:
  selector:
    app: ventas
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 8086
      targetPort: 8086
```

---

## 12. Deployment de despachos

Archivo:

```text
k8s/despachos-deployment.yaml
```

Ejemplo:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: despachos
  namespace: innovatech
spec:
  replicas: 2
  selector:
    matchLabels:
      app: despachos
  template:
    metadata:
      labels:
        app: despachos
    spec:
      containers:
        - name: despachos
          image: 308769698189.dkr.ecr.us-east-1.amazonaws.com/backend-innovatech:despachos
          ports:
            - containerPort: 8085
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: prod
            - name: SPRING_DATASOURCE_URL
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: SPRING_DATASOURCE_URL
            - name: SPRING_DATASOURCE_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: SPRING_DATASOURCE_USERNAME
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: SPRING_DATASOURCE_PASSWORD
            - name: SPRING_DATASOURCE_HIKARI_MAXIMUM_POOL_SIZE
              value: "3"
            - name: SPRING_DATASOURCE_HIKARI_MINIMUM_IDLE
              value: "1"
            - name: SPRING_DATASOURCE_HIKARI_CONNECTION_TIMEOUT
              value: "30000"
            - name: SPRING_DATASOURCE_HIKARI_IDLE_TIMEOUT
              value: "120000"
            - name: SPRING_DATASOURCE_HIKARI_MAX_LIFETIME
              value: "300000"
            - name: SPRING_DATASOURCE_HIKARI_KEEPALIVE_TIME
              value: "60000"
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
---
apiVersion: v1
kind: Service
metadata:
  name: despachos-service
  namespace: innovatech
spec:
  selector:
    app: despachos
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 8085
      targetPort: 8085
```

---

## 13. Autoscaling HPA

Archivo:

```text
k8s/hpa.yaml
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ventas-hpa
  namespace: innovatech
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ventas
  minReplicas: 2
  maxReplicas: 3
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: despachos-hpa
  namespace: innovatech
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: despachos
  minReplicas: 2
  maxReplicas: 3
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

Aplicar manualmente:

```bash
kubectl apply -f k8s/hpa.yaml
```

Verificar:

```bash
kubectl get hpa -n innovatech
```

---

## 14. Pipeline CI/CD con GitHub Actions

El pipeline se ejecuta automáticamente al hacer push a la rama:

```text
deploy
```

Flujo del pipeline:

```text
Push a rama deploy
   ↓
GitHub Actions
   ↓
Build imagen ventas
   ↓
Build imagen despachos
   ↓
Push imágenes a Amazon ECR
   ↓
Conexión con Amazon EKS
   ↓
Creación/actualización de mysql-secret
   ↓
Aplicación de manifiestos Kubernetes
   ↓
Actualización de imágenes de deployments
   ↓
Rollout
   ↓
Activación del HPA
```

---

## 15. Workflow recomendado

Archivo:

```text
.github/workflows/deploy.yml
```

```yaml
name: Backend Microservices CI/CD (ECR + EKS)

on:
  push:
    branches:
      - deploy

env:
  AWS_REGION: us-east-1
  EKS_CLUSTER_NAME: innovatech
  K8S_NAMESPACE: innovatech
  AWS_ACCOUNT_ID: "308769698189"
  ECR_REPOSITORY: backend-innovatech

jobs:
  build-and-deploy:
    name: Build, Push & Deploy Backend
    runs-on: ubuntu-latest

    steps:
      - name: Checkout del codigo
        uses: actions/checkout@v4

      - name: Configurar credenciales AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Verificar identidad AWS
        run: aws sts get-caller-identity

      - name: Login Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build y Push imagen Despachos
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./back-Despachos_SpringBoot/Dockerfile
          push: true
          tags: 308769698189.dkr.ecr.us-east-1.amazonaws.com/backend-innovatech:despachos

      - name: Build y Push imagen Ventas
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./back-Ventas_SpringBoot/Dockerfile
          push: true
          tags: 308769698189.dkr.ecr.us-east-1.amazonaws.com/backend-innovatech:ventas

      - name: Instalar kubectl
        uses: azure/setup-kubectl@v4

      - name: Configurar conexion con EKS
        run: |
          aws eks update-kubeconfig --region us-east-1 --name innovatech

      - name: Crear namespace si no existe
        run: |
          kubectl create namespace innovatech --dry-run=client -o yaml | kubectl apply -f -

      - name: Crear o actualizar Secret MySQL en Kubernetes
        run: |
          kubectl create secret generic mysql-secret \
            --from-literal=DB_ENDPOINT="${{ secrets.DB_ENDPOINT }}" \
            --from-literal=SPRING_DATASOURCE_URL="jdbc:mysql://${{ secrets.DB_ENDPOINT }}:3306/${{ secrets.DB_NAME }}?allowPublicKeyRetrieval=true&useSSL=false&serverTimezone=UTC" \
            --from-literal=SPRING_DATASOURCE_USERNAME="${{ secrets.DB_USERNAME }}" \
            --from-literal=SPRING_DATASOURCE_PASSWORD="${{ secrets.DB_PASSWORD }}" \
            -n innovatech \
            --dry-run=client -o yaml | kubectl apply -f -

      - name: Pausar HPA durante deploy
        run: |
          kubectl delete hpa ventas-hpa despachos-hpa -n innovatech --ignore-not-found

      - name: Aplicar manifiestos Kubernetes
        run: |
          kubectl apply -f k8s/ventas-deployment.yaml
          kubectl apply -f k8s/despachos-deployment.yaml

      - name: Actualizar imagen Ventas
        run: |
          kubectl set image deployment/ventas ventas=308769698189.dkr.ecr.us-east-1.amazonaws.com/backend-innovatech:ventas -n innovatech

      - name: Actualizar imagen Despachos
        run: |
          kubectl set image deployment/despachos despachos=308769698189.dkr.ecr.us-east-1.amazonaws.com/backend-innovatech:despachos -n innovatech

      - name: Reiniciar deployments
        run: |
          kubectl rollout restart deployment ventas -n innovatech
          kubectl rollout restart deployment despachos -n innovatech

      - name: Verificar rollout
        run: |
          kubectl rollout status deployment/ventas -n innovatech --timeout=5m
          kubectl rollout status deployment/despachos -n innovatech --timeout=5m
          kubectl get pods -n innovatech

      - name: Activar HPA
        run: |
          kubectl apply -f k8s/hpa.yaml
          kubectl get hpa -n innovatech
```

---

## 16. Despliegue automático

Para desplegar el backend:

```bash
git checkout deploy
git add .
git commit -m "Deploy backend EKS"
git push origin deploy
```

Esto activa GitHub Actions automáticamente.

---

## 17. Verificar estado en Kubernetes

Ver pods:

```bash
kubectl get pods -n innovatech
```

Ver deployments:

```bash
kubectl get deployments -n innovatech
```

Ver services:

```bash
kubectl get svc -n innovatech
```

Ver endpoints:

```bash
kubectl get endpoints -n innovatech
```

Ver HPA:

```bash
kubectl get hpa -n innovatech
```

Ver secret:

```bash
kubectl get secret mysql-secret -n innovatech
```

---

## 18. Probar ventas dentro del clúster

Crear pod temporal:

```bash
kubectl run curl-test-ventas \
  -n innovatech \
  --image=curlimages/curl \
  -it \
  --rm \
  --restart=Never \
  -- sh
```

Consultar ventas:

```bash
curl http://ventas-service:8086/api/v1/ventas
```

Crear venta:

```bash
curl -X POST http://ventas-service:8086/api/v1/ventas \
  -H "Content-Type: application/json" \
  -d '{"direccionCompra":"Av. Siempre Viva 123","valorCompra":25000,"fechaCompra":"2026-06-19","despachoGenerado":false}'
```

Consultar nuevamente:

```bash
curl http://ventas-service:8086/api/v1/ventas
```

Salir:

```bash
exit
```

---

## 19. Probar desde el frontend público

Una vez desplegado el frontend, probar:

```text
http://DNS-DEL-LOADBALANCER/api/v1/ventas
```

Esto valida el flujo:

```text
Internet
   ↓
Frontend LoadBalancer
   ↓
Nginx
   ↓
ventas-service
   ↓
Pod ventas
   ↓
MySQL
```

---

## 20. Ver logs

Logs de ventas:

```bash
kubectl logs deployment/ventas -n innovatech --tail=100
```

Logs de despachos:

```bash
kubectl logs deployment/despachos -n innovatech --tail=100
```

Logs en vivo:

```bash
kubectl logs deployment/ventas -n innovatech -f
```

---

## 21. Probar autorecuperación

Listar pods:

```bash
kubectl get pods -n innovatech
```

Eliminar un pod de ventas:

```bash
kubectl delete pod NOMBRE_DEL_POD -n innovatech
```

Observar recuperación:

```bash
kubectl get pods -n innovatech -w
```

Kubernetes creará automáticamente un nuevo pod para mantener el estado deseado.

---

## 22. Diagnóstico

Describir pod:

```bash
kubectl describe pod NOMBRE_DEL_POD -n innovatech
```

Ver eventos:

```bash
kubectl get events -n innovatech --sort-by=.metadata.creationTimestamp
```

Ver imagen usada por ventas:

```bash
kubectl get deployment ventas -n innovatech -o=jsonpath="{.spec.template.spec.containers[0].image}"
```

Ver imagen usada por despachos:

```bash
kubectl get deployment despachos -n innovatech -o=jsonpath="{.spec.template.spec.containers[0].image}"
```

---

## 23. Problemas comunes

### Error ImagePullBackOff

Causas posibles:

* Imagen mal escrita.
* Repositorio ECR no existe.
* Tag incorrecto.
* Credenciales o permisos insuficientes.

Revisar:

```bash
kubectl describe pod NOMBRE_DEL_POD -n innovatech
```

---

### Error de conexión a MySQL

Causas posibles:

* EC2 de base de datos apagada.
* MariaDB/MySQL detenido.
* Security Group no permite puerto 3306.
* `DB_ENDPOINT` incorrecto.
* Usuario o contraseña incorrectos.
* Secret desactualizado.

Revisar logs:

```bash
kubectl logs deployment/ventas -n innovatech --tail=100
```

Verificar servicio de base de datos en EC2:

```bash
sudo systemctl status mariadb
```

O si se usa MySQL:

```bash
sudo systemctl status mysqld
```

---

### GitHub Actions falla por credenciales

En AWS Academy las credenciales expiran. Actualizar:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_SESSION_TOKEN
```

---

### HPA escala demasiado

En AWS Academy se recomienda mantener límites bajos:

```text
minReplicas: 2
maxReplicas: 3
```

Verificar:

```bash
kubectl get hpa -n innovatech
```

---

## 24. Evidencias recomendadas

Para presentación o video demostrativo:

```bash
kubectl get pods -n innovatech
kubectl get deployments -n innovatech
kubectl get svc -n innovatech
kubectl get endpoints -n innovatech
kubectl get hpa -n innovatech
kubectl logs deployment/ventas -n innovatech --tail=50
```

Además:

* GitHub Actions exitoso.
* Imágenes en Amazon ECR.
* Secret `mysql-secret` creado.
* Prueba con `curl`.
* Autorecuperación eliminando un pod.
* Respuesta del backend desde el frontend público.

---

## 25. Flujo final resumido

```text
git push origin deploy
   ↓
GitHub Actions
   ↓
Docker build ventas/despachos
   ↓
Push imágenes a Amazon ECR
   ↓
Deploy en Amazon EKS
   ↓
Kubernetes ejecuta pods backend
   ↓
Services ClusterIP exponen backend internamente
   ↓
Backend se conecta a MySQL mediante mysql-secret
   ↓
Frontend consume backend mediante DNS interno
```

---

## 26. Estado esperado final

```text
ventas                Running
despachos             Running
ventas-service        ClusterIP 8086
despachos-service     ClusterIP 8085
ventas-hpa            Activo
despachos-hpa         Activo
mysql-secret          Creado
/api/v1/ventas        Respondiendo
```

Con esto, el backend queda desplegado correctamente en AWS EKS, usando imágenes en Amazon ECR, despliegue automático con GitHub Actions, configuración segura mediante Kubernetes Secrets y escalamiento horizontal con HPA.
