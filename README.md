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
    └── all-in-one.yaml
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
<img width="1600" height="142" alt="image" src="https://github.com/user-attachments/assets/7b285f33-0082-438a-a495-921430b72cb5" />
<img width="1597" height="136" alt="image" src="https://github.com/user-attachments/assets/ffe4f16b-71ab-420f-a507-cec6e62ca4d5" />
<img width="1530" height="445" alt="image" src="https://github.com/user-attachments/assets/5fbaf0f7-61d9-4798-a72e-ca5b0a70ad5f" />
<img width="1600" height="463" alt="image" src="https://github.com/user-attachments/assets/5cd7ffa4-a3c2-46a9-ae6d-4dae1abd50c0" />
<img width="1600" height="677" alt="image" src="https://github.com/user-attachments/assets/495bb42c-b039-4511-9d52-6e2ec5c08c53" />
<img width="1600" height="857" alt="image" src="https://github.com/user-attachments/assets/516ff33f-3a98-4f52-994c-f8c5b15043c8" />
<img width="1600" height="230" alt="image" src="https://github.com/user-attachments/assets/3b908aa4-d870-48c0-b687-5106c4fc1aae" />

---
