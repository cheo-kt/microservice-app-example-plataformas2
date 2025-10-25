# Microservice App - PRFT Devops Training

This is the application you are going to use through the whole traninig. This, hopefully, will teach you the fundamentals you need in a real project. You will find a basic TODO application designed with a [microservice architecture](https://microservices.io). Although is a TODO application, it is interesting because the microservices that compose it are written in different programming language or frameworks (Go, Python, Vue, Java, and NodeJS). With this design you will experiment with multiple build tools and environments. 

## Components
In each folder you can find a more in-depth explanation of each component:

1. [Users API](/users-api) is a Spring Boot application. Provides user profiles. At the moment, does not provide full CRUD, just getting a single user and all users.
2. [Auth API](/auth-api) is a Go application, and provides authorization functionality. Generates [JWT](https://jwt.io/) tokens to be used with other APIs.
3. [TODOs API](/todos-api) is a NodeJS application, provides CRUD functionality over user's TODO records. Also, it logs "create" and "delete" operations to [Redis](https://redis.io/) queue.
4. [Log Message Processor](/log-message-processor) is a queue processor written in Python. Its purpose is to read messages from a Redis queue and print them to standard output.
5. [Frontend](/frontend) Vue application, provides UI.

## Architecture

Take a look at the components diagram that describes them and their interactions.
![microservice-app-example](/arch-img/Microservices.png)



# **Laboratorio de Despliegue de Aplicación Microservicios en Kubernetes**

**Nombre:** Sergio Fernando Florez Sanabria
**Código:** A00396046
**Curso:** Plataformas 2


---

## **1. Introducción**

El presente laboratorio tiene como objetivo desplegar una aplicación basada en microservicios dentro de un clúster de Kubernetes.
El proyecto se compone de múltiples servicios desarrollados en distintos lenguajes (Go, Node.js, etc.), incluyendo componentes de autenticación, gestión de tareas, usuarios, procesamiento de logs y un frontend.
El despliegue se realizó utilizando **Docker** para la construcción de imágenes y **Kubernetes (kubectl)** para la orquestación de contenedores en el espacio de nombres `microservice-app`.

---

## **2. Estructura del proyecto**

Durante la práctica se trabajó dentro del repositorio:

```
microservice-app-example-plataformas2/
│
├── auth-api/
│   ├── Dockerfile
│   ├── main.go
│   ├── user.go
│   ├── tracing.go
│   ├── README.md
│   ├── Gopkg.toml
│   ├── Gopkg.lock
│
├── users-api/
│   ├── Dockerfile
│   ├── main.go
│   ├── user.go
│
├── todos-api/
│   ├── Dockerfile
│   ├── main.go
│
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   ├── src/
│
├── log-message-processor/
│   ├── Dockerfile
│   ├── main.go
│
└── k8s/
    ├── namespace.yaml
    ├── redis-deployment.yaml
    ├── auth-api-deployment.yaml
    ├── users-api-deployment.yaml
    ├── todos-api-deployment.yaml
    ├── frontend-deployment.yaml
    ├── log-message-processor-deployment.yaml
    ├── users-api-deployment.yaml                ✅ Con resources, rolling update, health checks
    ├── todos-api-deployment.yaml                ✅ Con resources, rolling update, health checks
    ├── frontend-deployment.yaml                 ✅ Con resources, rolling update, health checks
    ├── all-in-one.yaml
    ├── todos-api-hpa.yaml                       (🆕 HPA para todos-api)
    ├── users-api-hpa.yaml                       (🆕 HPA para users-api)
    ├── auth-api-hpa.yaml 
    ├── network-policy-deny-all.yaml          
    ├── network-policy-allow-frontend.yaml    
    └── network-policy-allow-backends.yaml    
```

---

## **3. Pasos realizados**

### **3.1. Creación del Namespace**

Se creó el espacio de nombres donde se desplegaron todos los servicios:

```bash
kubectl apply -f namespace.yaml
```

### **3.2. Construcción de las imágenes Docker**

Se generaron las imágenes para cada servicio desde su respectiva carpeta:

```bash
docker build -t chechoiot/auth-api:latest ./auth-api
docker build -t chechoiot/users-api:latest ./users-api
docker build -t chechoiot/todos-api:latest ./todos-api
docker build -t chechoiot/frontend:latest ./frontend
docker build -t chechoiot/log-message-processor:latest ./log-message-processor
```

Posteriormente, se subieron a Docker Hub:

```bash
docker push chechoiot/auth-api:latest
docker push chechoiot/users-api:latest
docker push chechoiot/todos-api:latest
docker push chechoiot/frontend:latest
docker push chechoiot/log-message-processor:latest
```

### **3.3. Despliegue de servicios en Kubernetes**

Se aplicaron los manifiestos YAML del proyecto:

```bash
kubectl apply -f k8s/ -n microservice-app
```

Esto desplegó los pods correspondientes a cada microservicio.
Se verificó su estado con:

```bash
kubectl get pods -n microservice-app
kubectl get deployments -n microservice-app
```

### **3.4. Exposición del frontend**

El servicio frontend se expuso localmente para probar la interfaz:

```bash
kubectl port-forward svc/frontend-service 8080:80 -n microservice-app
```

Accediendo desde el navegador a:

```
http://localhost:8080
```

---

## **4. Archivos agregados y configuraciones**

### **Archivo `namespace.yaml`**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: microservice-app
```

### **Archivo `all-in-one.yaml`**

Se creó para aplicar todos los despliegues de forma unificada:

```yaml
apiVersion: v1
kind: List
items:
  - apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: redis
      namespace: microservice-app
    spec:
      selector:
        matchLabels:
          app: redis
      replicas: 1
      template:
        metadata:
          labels:
            app: redis
        spec:
          containers:
          - name: redis
            image: redis:6.2
            ports:
            - containerPort: 6379
  - apiVersion: v1
    kind: Service
    metadata:
      name: redis
      namespace: microservice-app
    spec:
      selector:
        app: redis
      ports:
      - port: 6379
```

*(El resto de los despliegues seguían la misma estructura para los servicios `auth-api`, `users-api`, `todos-api`, `frontend` y `log-message-processor`.)*

---

## **5. Dificultades encontradas**

* **Error `ImagePullBackOff`:** Inicialmente los pods no iniciaban correctamente debido a que las imágenes no estaban subidas o referenciadas con el nombre correcto en Docker Hub.
  Se resolvió reconstruyendo y subiendo nuevamente las imágenes.

* **Error `exec ./auth-api: no such file or directory`:**
  Este error surgió debido a que la compilación del binario dentro del contenedor Go no se realizaba en la ruta correcta. Se ajustó el Dockerfile y el contexto de compilación.

* **Dependencias de Node.js:**
  El servicio `frontend` presentaba problemas con dependencias deprecadas (`node-sass`). Se solucionó temporalmente limpiando el caché de npm y reinstalando dependencias (`npm install`), aunque se identificó que el repositorio contenía versiones antiguas.

---

## **6. Resultados**

Luego de los ajustes, los servicios `frontend`, `todos-api` y `redis` lograron desplegarse correctamente y comunicarse dentro del clúster.
Aunque algunos servicios presentaron fallos por versiones o dependencias desactualizadas, la estructura funcional del entorno de microservicios quedó operativa.

---

## **7. Conclusiones**

* El laboratorio permitió comprender el proceso completo de **construcción, publicación y despliegue de microservicios** utilizando Docker y Kubernetes.
* Se evidenció la importancia de mantener versiones actualizadas de dependencias, así como definir correctamente los contextos de compilación dentro de los contenedores.
* A pesar de los errores de compatibilidad presentes en el repositorio original, se logró una configuración estable que permite el funcionamiento de los principales servicios.
* La práctica consolidó conocimientos sobre **automatización, despliegue y orquestación** de servicios distribuidos en entornos modernos.

---
---

## **8. Fotos**
<img width="1600" height="338" alt="image" src="https://github.com/user-attachments/assets/61209396-f593-4c5d-85b6-ea1df8db93f2" />
<img width="1600" height="557" alt="image" src="https://github.com/user-attachments/assets/7bc9c4ab-4b93-4e98-ab7a-7fbd3618f772" />


## Taller 1 Parte 2.



## **1. Introducción**

El presente laboratorio tiene como objetivo desplegar una aplicación basada en microservicios dentro de un clúster de Kubernetes con características avanzadas de observabilidad, escalabilidad automática y seguridad.
El proyecto se compone de múltiples servicios desarrollados en distintos lenguajes (Go, Node.js, Java), incluyendo componentes de autenticación, gestión de tareas, usuarios, procesamiento de logs y un frontend.
El despliegue se realizó utilizando **Docker** para la construcción de imágenes, **Kubernetes (kubectl)** para la orquestación de contenedores, **Prometheus** y **Grafana** para monitoreo, **Helm** para gestión de paquetes, y **HPA** para escalamiento automático en el espacio de nombres `microservice-app`.

---

## **2. Estructura del proyecto**

Durante la práctica se trabajó dentro del repositorio:

```
microservice-app-example-plataformas2/
│
├── auth-api/
│   ├── Dockerfile
│   ├── main.go
│   ├── user.go
│   ├── tracing.go
│   ├── README.md
│   ├── Gopkg.toml
│   └── Gopkg.lock
│
├── users-api/
│   ├── Dockerfile
│   ├── main.go
│   └── user.go
│
├── todos-api/
│   ├── Dockerfile
│   └── main.go
│
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│
├── log-message-processor/
│   ├── Dockerfile
│   └── main.go
│
└── k8s/
    ├── namespace.yaml
    ├── redis-deployment.yaml                    ✅ Con resources
    ├── auth-api-deployment.yaml
    ├── users-api-deployment.yaml                ✅ Con resources, rolling update, health checks
    ├── todos-api-deployment.yaml                ✅ Con resources, rolling update, health checks
    ├── frontend-deployment.yaml                 ✅ Con resources, rolling update, health checks
    ├── log-message-processor-deployment.yaml
    ├── all-in-one.yaml
    ├── todos-api-hpa.yaml                       🆕 HPA para todos-api
    ├── users-api-hpa.yaml                       🆕 HPA para users-api
    ├── auth-api-hpa.yaml                        🆕 HPA para auth-api
    ├── network-policy-deny-all.yaml             🆕 Política de seguridad
    ├── network-policy-allow-frontend.yaml       🆕 Política de seguridad
    ├── network-policy-allow-backends.yaml       🆕 Política de seguridad
    ├── deployment-strategy.md                   🆕 Documentación de estrategia
    └── service-monitor-users.yaml               🆕 ServiceMonitor para Prometheus
```

---

## **3. Pasos realizados**

### **3.1. Instalación de Herramientas**

#### **Instalación de kubectl**
```bash
# Descargar la última versión de kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Darle permisos de ejecución
chmod +x kubectl

# Mover kubectl a un directorio en el PATH
sudo mv kubectl /usr/local/bin/

# Verificar la instalación
kubectl version --client
```

#### **Instalación de Minikube**
```bash
# Descargar el binario de minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# Instalar minikube
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Limpiar el archivo descargado
rm minikube-linux-amd64

# Verificar la instalación
minikube version
```

#### **Instalación de Docker**
```bash
# Actualizar repositorios
sudo apt update

# Instalar Docker
sudo apt install -y docker.io

# Agregar usuario al grupo docker
sudo usermod -aG docker $USER

# Aplicar cambios de grupo
newgrp docker

# Verificar Docker
docker --version
```

#### **Instalación de Helm**
```bash
# Instalar Helm con el script oficial
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verificar la instalación
helm version
```

#### **Instalación de herramientas adicionales**
```bash
# Instalar jq para procesar JSON
sudo apt install -y jq

# Instalar yq para procesar YAML
sudo wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O /usr/bin/yq
sudo chmod +x /usr/bin/yq

# Instalar vim
sudo apt install -y vim

# Verificar instalaciones
jq --version
yq --version
vim --version
```

### **3.2. Inicialización del clúster Minikube**

```bash
# Iniciar minikube (ejecutar como usuario normal, no root)
minikube start

# Verificar el estado
minikube status

# Habilitar metrics-server para HPA
minikube addons enable metrics-server

# Verificar que metrics-server está corriendo
kubectl get deployment metrics-server -n kube-system
```

### **3.3. Creación del Namespace**

Se creó el espacio de nombres donde se desplegaron todos los servicios:

```bash
kubectl apply -f k8s/namespace.yaml
```

### **3.4. Construcción de las imágenes Docker**

Se generaron las imágenes para cada servicio desde su respectiva carpeta:

```bash
docker build -t chechoiot/auth-api:latest ./auth-api
docker build -t chechoiot/users-api:latest ./users-api
docker build -t chechoiot/todos-api:latest ./todos-api
docker build -t chechoiot/frontend:latest ./frontend
docker build -t chechoiot/log-message-processor:latest ./log-message-processor
```

Posteriormente, se subieron a Docker Hub:

```bash
docker push chechoiot/auth-api:latest
docker push chechoiot/users-api:latest
docker push chechoiot/todos-api:latest
docker push chechoiot/frontend:latest
docker push chechoiot/log-message-processor:latest
```

### **3.5. Configuración de Resources en los Deployments**

Se agregaron recursos (CPU y memoria) a cada deployment basándose en el consumo real de los pods:

#### **todos-api-deployment.yaml**
```yaml
resources:
  requests:
    cpu: 50m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

#### **users-api-deployment.yaml**
```yaml
resources:
  requests:
    cpu: 50m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 1Gi
```

#### **frontend-deployment.yaml**
```yaml
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 250m
    memory: 128Mi
```

#### **redis-deployment.yaml**
```yaml
resources:
  requests:
    cpu: 50m
    memory: 32Mi
  limits:
    cpu: 200m
    memory: 128Mi
```

### **3.6. Implementación de Estrategia Rolling Update**

Se configuró la estrategia de despliegue Rolling Update en todos los deployments principales para garantizar actualizaciones sin tiempo de inactividad:

```yaml
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # Permite 1 pod extra durante actualización
      maxUnavailable: 0     # Siempre mantiene al menos 1 pod disponible
```

#### **Health Checks implementados:**

**Readiness Probe:** Verifica que el pod esté listo para recibir tráfico
```yaml
readinessProbe:
  tcpSocket:
    port: 8082
  initialDelaySeconds: 10
  periodSeconds: 5
```

**Liveness Probe:** Verifica que el pod siga funcionando correctamente
```yaml
livenessProbe:
  tcpSocket:
    port: 8082
  initialDelaySeconds: 30
  periodSeconds: 10
```

**Corrección de puertos:**
- `todos-api`: Puerto 8082
- `users-api`: Puerto 8083
- `frontend`: Puerto 80

### **3.7. Despliegue de servicios en Kubernetes**

Se aplicaron los manifiestos YAML del proyecto:

```bash
kubectl apply -f k8s/ -n microservice-app
```

Esto desplegó los pods correspondientes a cada microservicio.
Se verificó su estado con:

```bash
kubectl get pods -n microservice-app
kubectl get deployments -n microservice-app
kubectl get svc -n microservice-app
```

### **3.8. Instalación de Prometheus y Grafana con Helm**

#### **Agregar repositorio de Helm**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

#### **Instalar el stack kube-prometheus**
```bash
# Crear namespace para monitoreo
kubectl create namespace monitoring

# Instalar Prometheus y Grafana
helm install kube-prometheus prometheus-community/kube-prometheus-stack -n monitoring

# Verificar instalación
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

#### **Acceder a Grafana**
```bash
# Obtener credenciales de Grafana
kubectl get secret -n monitoring kube-prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode

# Port-forward para acceder a Grafana
kubectl port-forward -n monitoring svc/kube-prometheus-grafana 3000:80
```

Acceder desde el navegador a: `http://localhost:3000`
- Usuario: `admin`
- Contraseña: (obtenida del comando anterior)

#### **Configuración de Data Source en Grafana**
1. En Grafana → Configuration → Data Sources
2. Prometheus ya está configurado automáticamente como `prometheus-k8s`
3. URL: `http://kube-prometheus-kube-prome-prometheus.monitoring:9090`

#### **Dashboards disponibles en Grafana:**
- Alertmanager / Overview
- CoreDNS
- etcd
- Grafana Overview
- Kubernetes / API server
- Kubernetes / Compute Resources / Multi-Cluster
- Kubernetes / Compute Resources / Cluster
- Kubernetes / Compute Resources / Namespace (Pods)
- Kubernetes / Compute Resources / Namespace (Workloads)
- Kubernetes / Compute Resources / Node (Pods)
- Kubernetes / Compute Resources / Pod
- Kubernetes / Compute Resources / Workload
- Kubernetes / Networking / Cluster
- Kubernetes / Networking / Namespace (Pods)
- Kubernetes / Networking / Namespace (Workload)
- Kubernetes / Networking / Pod
- Kubernetes / Networking / Workload
- Kubernetes / Persistent Volumes

### **3.9. Configuración de Horizontal Pod Autoscaler (HPA)**

Se crearon HPAs para escalar automáticamente los servicios basándose en el uso de CPU:

#### **todos-api-hpa.yaml**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: todos-api-hpa
  namespace: microservice-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: todos-api
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

#### **Aplicar los HPAs**
```bash
kubectl apply -f k8s/todos-api-hpa.yaml
kubectl apply -f k8s/users-api-hpa.yaml
kubectl apply -f k8s/auth-api-hpa.yaml

# Verificar HPAs
kubectl get hpa -n microservice-app

# Ver métricas en tiempo real
kubectl top pods -n microservice-app
```

### **3.10. Implementación de Network Policies (Seguridad)**

Se implementaron políticas de red para controlar el tráfico entre pods siguiendo el principio de mínimo privilegio:

#### **Política Deny All (Denegar todo por defecto)**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: microservice-app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

#### **Políticas de Allow (Permitir tráfico específico)**

**Permitir frontend → backends:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-auth
  namespace: microservice-app
spec:
  podSelector:
    matchLabels:
      app: auth-api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

**Permitir backends → Redis:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backends-to-redis
  namespace: microservice-app
spec:
  podSelector:
    matchLabels:
      app: redis
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: auth-api
        - podSelector:
            matchLabels:
              app: users-api
        - podSelector:
            matchLabels:
              app: todos-api
      ports:
        - protocol: TCP
          port: 6379
```

**Nota:** Las NetworkPolicies requieren un CNI compatible (como Calico). Para habilitarlo en Minikube:
```bash
minikube delete
minikube start --cni=calico --force
```

### **3.11. Exposición del frontend**

El servicio frontend se expuso localmente para probar la interfaz:

```bash
kubectl port-forward svc/frontend-service 8080:80 -n microservice-app
```

Accediendo desde el navegador a:
```
http://localhost:8080
```

### **3.12. Monitoreo y observabilidad**

#### **Verificar métricas de Prometheus**
```bash
# Port-forward a Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-kube-prome-prometheus 9090:9090
```

Acceder a: `http://localhost:9090`

#### **Consultas PromQL útiles:**
```promql
# Ver todos los pods corriendo
kube_pod_status_phase{phase="Running"}

# Uso de CPU por pod
rate(container_cpu_usage_seconds_total[5m])

# Uso de memoria por pod
container_memory_usage_bytes

# Requests HTTP (si la app los exporta)
sum(rate(http_requests_total[5m])) by (service)
```

---

## **4. Comandos útiles de verificación**

```bash
# Ver todos los recursos
kubectl get all -n microservice-app

# Ver estado de pods
kubectl get pods -n microservice-app -w

# Ver logs de un pod
kubectl logs <pod-name> -n microservice-app

# Describir un pod (para debugging)
kubectl describe pod <pod-name> -n microservice-app

# Ver HPAs
kubectl get hpa -n microservice-app

# Ver métricas de recursos
kubectl top nodes
kubectl top pods -n microservice-app

# Ver network policies
kubectl get networkpolicies -n microservice-app

# Ver servicios
kubectl get svc -n microservice-app

# Ver deployments
kubectl get deployments -n microservice-app
```

---

## **5. Estrategia de Despliegue**

### **Rolling Update**

La estrategia implementada es **Rolling Update**, que permite actualizaciones sin tiempo de inactividad:

#### **Características:**
- **maxSurge: 1** - Permite crear 1 pod adicional durante la actualización
- **maxUnavailable: 0** - Garantiza que siempre haya al menos 1 pod disponible
- **replicas: 2** - Alta disponibilidad con múltiples réplicas
- **Health Checks** - readinessProbe y livenessProbe para validación

#### **Proceso de actualización:**
```bash
# Actualizar imagen
kubectl set image deployment/todos-api todos-api=chechoiot/todos-api:v2 -n microservice-app

# Monitorear el rollout
kubectl rollout status deployment/todos-api -n microservice-app

# Ver historial de despliegues
kubectl rollout history deployment/todos-api -n microservice-app

# Rollback si es necesario
kubectl rollout undo deployment/todos-api -n microservice-app
```

#### **Flujo visual:**
```
Estado Inicial:  [Pod v1] [Pod v1]
       ↓
Paso 1:         [Pod v1] [Pod v1] [Pod v2]  ← Se crea pod nuevo
       ↓
Paso 2:         [Pod v1] [Pod v2] [Pod v2]  ← Se elimina pod viejo
       ↓
Estado Final:   [Pod v2] [Pod v2]            ← Actualización completa
```

---

## **6. Dificultades encontradas y soluciones**

### **6.1. Error `ImagePullBackOff`**
**Problema:** Los pods no iniciaban correctamente debido a que las imágenes no estaban subidas o referenciadas con el nombre correcto en Docker Hub.

**Solución:** Se reconstruyeron y subieron nuevamente las imágenes con los nombres correctos:
```bash
docker build -t chechoiot/frontend:latest ./frontend
docker push chechoiot/frontend:latest
```

### **6.2. Error `exec ./auth-api: no such file or directory`**
**Problema:** La compilación del binario dentro del contenedor Go no se realizaba en la ruta correcta.

**Solución:** Se ajustó el Dockerfile y el contexto de compilación.

### **6.3. Pods en estado `Running` pero no `Ready` (0/1)**
**Problema:** Los readinessProbe fallaban porque buscaban el endpoint `/health` en puertos incorrectos.

**Solución:** Se corrigieron los puertos en los deployments:
- todos-api: puerto 8082
- users-api: puerto 8083
- Se cambió de `httpGet` a `tcpSocket` para simplificar las verificaciones

### **6.4. Error de indentación en archivos YAML**
**Problema:** Errores de sintaxis YAML causaban fallos al aplicar manifiestos.

**Solución:** Se corrigió la indentación siguiendo el estándar YAML (2 espacios).

### **6.5. Variables de entorno incompatibles en users-api**
**Problema:** Las variables `JAVA_TOOL_OPTIONS` causaban error: `Unrecognized option: --add-opens`

**Solución:** Se eliminaron las variables de entorno incompatibles del deployment.

### **6.6. Resources en lugar incorrecto**
**Problema:** Los `resources` estaban definidos en el Service en vez del Deployment.

**Solución:** Se movieron los `resources` a la sección correcta dentro del `spec.template.spec.containers`.

### **6.7. Minikube requiere permisos no-root**
**Problema:** Error `DRV_AS_ROOT` al ejecutar minikube como root.

**Solución:** Se ejecutó minikube con usuario normal después de agregarlo al grupo docker:
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### **6.8. Metrics-server para HPA**
**Problema:** Los HPAs mostraban `<unknown>` en targets.

**Solución:** Se habilitó metrics-server en minikube:
```bash
minikube addons enable metrics-server
```

---

## **7. Resultados**

### **Estado final de los pods:**
```bash
$ kubectl get pods -n microservice-app

NAME                                    READY   STATUS    RESTARTS   AGE
frontend-6ccd86fd9f-24f68               1/1     Running   0          15m
frontend-6ccd86fd9f-2d2ck               1/1     Running   0          15m
redis-7f67967495-7p7jr                  1/1     Running   0          45m
todos-api-69f97c6c99-h5z64              1/1     Running   0          10m
todos-api-69f97c6c99-559x2              1/1     Running   0          10m
users-api-69778d598f-rdj4j              1/1     Running   0          10m
users-api-69778d598f-jx2cm              1/1     Running   0          10m
users-api-85d7777fb6-bsxnt              1/1     Running   0          10m
log-message-processor-9577c8f4d-l5zml   1/1     Running   0          45m
```

### **HPAs funcionando:**
```bash
$ kubectl get hpa -n microservice-app

NAME             REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS
todos-api-hpa    Deployment/todos-api   2%/50%    1         5         2
users-api-hpa    Deployment/users-api   1%/50%    1         5         3
auth-api-hpa     Deployment/auth-api    0%/50%    1         5         1
```

### **Servicios funcionando correctamente:**
- ✅ Frontend accesible en `http://localhost:8080`
- ✅ Grafana funcionando con dashboards de Kubernetes
- ✅ Prometheus recolectando métricas
- ✅ Redis operativo
- ✅ Todos-API y Users-API respondiendo correctamente
- ✅ Log-message-processor procesando mensajes
- ✅ HPA escalando automáticamente según carga
- ✅ Rolling Updates funcionando sin downtime

---

## **8. Conclusiones**

* El laboratorio permitió comprender el proceso completo de **construcción, publicación y despliegue de microservicios** utilizando Docker y Kubernetes con características empresariales.

* Se implementaron exitosamente **estrategias avanzadas de observabilidad** con Prometheus y Grafana, permitiendo monitoreo en tiempo real de todos los servicios.

* El **Horizontal Pod Autoscaler (HPA)** demuestra la capacidad de Kubernetes para escalar automáticamente según la demanda, fundamental para aplicaciones en producción.

* La **estrategia Rolling Update** garantiza actualizaciones sin tiempo de inactividad, con health checks que aseguran que solo pods saludables reciben tráfico.

* Las **Network Policies** proporcionan una capa adicional de seguridad siguiendo el principio de mínimo privilegio, aunque requieren un CNI compatible como Calico.

* Se evidenció la importancia de:
  - Definir correctamente `resources` (requests y limits) para optimización de recursos
  - Implementar health checks (readiness y liveness probes) para alta disponibilidad
  - Usar herramientas de gestión como Helm para simplificar despliegues complejos
  - Mantener versiones actualizadas de dependencias y entornos de ejecución

* A pesar de los desafíos encontrados (puertos incorrectos, indentación YAML, compatibilidad de imágenes), se logró una configuración estable y robusta que cumple con estándares de producción.

* La práctica consolidó conocimientos sobre **automatización, escalabilidad, monitoreo, seguridad y orquestación** de servicios distribuidos en entornos modernos Cloud Native.

* El proyecto demuestra que una arquitectura de microservicios bien implementada en Kubernetes puede ser:
  - **Resiliente**: Con réplicas y health checks
  - **Escalable**: Con HPA automático
  - **Observable**: Con Prometheus y Grafana
  - **Segura**: Con Network Policies
  - **Mantenible**: Con estrategias de despliegue controladas

---

## **9. Referencias**

- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [Docker Documentation](https://docs.docker.com/)
- [Minikube Documentation](https://minikube.sigs.k8s.io/docs/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Rolling Update Strategy](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)

---

## **10. Fotos**

<img width="1600" height="572" alt="image" src="https://github.com/user-attachments/assets/d38effde-84fd-4727-b01d-c5c36130f194" />
<img width="1600" height="248" alt="image" src="https://github.com/user-attachments/assets/a896a592-c888-4ff6-8ea0-e2ad735251c6" />
<img width="953" height="1006" alt="image" src="https://github.com/user-attachments/assets/550dd952-79a2-4163-ae00-cde8a152b780" />
<img width="1600" height="848" alt="image" src="https://github.com/user-attachments/assets/487e7220-dc2a-412b-bde0-602f508400e1" />
<img width="1600" height="852" alt="image" src="https://github.com/user-attachments/assets/b5e497e6-2544-4cc1-aec1-ac6bdf81b5f7" />
<img width="1600" height="833" alt="image" src="https://github.com/user-attachments/assets/1b7b07c9-75b3-4412-8057-2b8187d6595a" />
<img width="1600" height="854" alt="image" src="https://github.com/user-attachments/assets/2a958557-b76e-4779-924e-367107448784" />
<img width="1600" height="862" alt="image" src="https://github.com/user-attachments/assets/f43cde96-912e-4f9a-af51-fbe53ff33fca" />
<img width="1600" height="832" alt="image" src="https://github.com/user-attachments/assets/1596faeb-6fa4-44a1-9ed6-d43a639a688a" />
<img width="1600" height="406" alt="image" src="https://github.com/user-attachments/assets/4628bd13-1e32-45b1-b18e-a7e11daf2884" />
<img width="1600" height="817" alt="image" src="https://github.com/user-attachments/assets/45b7dc56-0f44-4e34-8ef6-faf7019733f5" />
<img width="1600" height="836" alt="image" src="https://github.com/user-attachments/assets/16e3abe3-968b-48bc-9616-03cbd321595d" />
<img width="1600" height="298" alt="image" src="https://github.com/user-attachments/assets/9bdf71c1-6668-42b1-86ef-835d0a9dc025" />
<img width="1600" height="272" alt="image" src="https://github.com/user-attachments/assets/4238b2ee-93ad-4b5c-b890-7e970a9d5ae7" />
<img width="1600" height="619" alt="image" src="https://github.com/user-attachments/assets/2c23c9b5-3029-48a4-ae50-94ee55fbe25c" />
<img width="1600" height="626" alt="image" src="https://github.com/user-attachments/assets/c0ad4a22-d8fa-4ec6-8583-c5f19ce5d4c5" />
<img width="1600" height="695" alt="image" src="https://github.com/user-attachments/assets/fef4831f-0d7b-4bcb-86e3-7fb988d2ff48" />
<img width="1600" height="760" alt="image" src="https://github.com/user-attachments/assets/06b27c2e-81ed-4215-ab67-9384504f1ea0" />
<img width="1600" height="659" alt="image" src="https://github.com/user-attachments/assets/e87f838e-6f60-420a-a937-8e9b645d8d5b" />
<img width="1600" height="696" alt="image" src="https://github.com/user-attachments/assets/40dd2d94-7d47-489b-bf11-44685c26b4d5" />
<img width="1600" height="806" alt="image" src="https://github.com/user-attachments/assets/bd14efe9-0632-4e47-8ab1-59856c82ecc8" />
<img width="1600" height="135" alt="image" src="https://github.com/user-attachments/assets/e34558cc-4028-4c9c-b8a1-1f5eb60997dc" />
<img width="1600" height="851" alt="image" src="https://github.com/user-attachments/assets/83edd0f3-c005-4082-95d4-dd4650c3d01b" />
<img width="1600" height="618" alt="image" src="https://github.com/user-attachments/assets/40b9433d-5d33-48e3-bd12-132a5e794446" />
<img width="1600" height="158" alt="image" src="https://github.com/user-attachments/assets/c003a179-7dc7-4860-96df-658503bb7498" />
<img width="1600" height="844" alt="image" src="https://github.com/user-attachments/assets/c31f89e1-7af5-4b69-80bb-5c428e6117af" />
<img width="1599" height="302" alt="image" src="https://github.com/user-attachments/assets/ddf905c1-e910-4589-95fc-9cff80bfffca" />






---
