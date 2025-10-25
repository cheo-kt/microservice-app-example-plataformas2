# Estrategia de Despliegue - Microservices App

## 1. Tipo de Estrategia: Rolling Update

### Configuración:
- **maxSurge: 1** - Permite 1 pod adicional durante actualización
- **maxUnavailable: 0** - Garantiza disponibilidad continua
- **replicas: 2** - Alta disponibilidad con réplicas

### Ventajas:
✅ Zero downtime
✅ Rollback automático si falla
✅ Actualización gradual y controlada
✅ Validación con health checks

## 2. Health Checks

### readinessProbe:
- Verifica que el pod esté listo para recibir tráfico
- K8s no envía tráfico hasta que pase este check

### livenessProbe:
- Verifica que el pod siga funcionando
- K8s reinicia el pod si falla

## 3. Proceso de Despliegue

### Paso 1: Actualizar imagen
```bash
# Actualizar imagen en el deployment
kubectl set image deployment/todos-api todos-api=chechoiot/todos-api:v2 -n microservice-app
```

### Paso 2: Monitorear el rollout
```bash
kubectl rollout status deployment/todos-api -n microservice-app
kubectl get pods -n microservice-app -w
```

### Paso 3: Verificar
```bash
kubectl get deployment todos-api -n microservice-app
kubectl describe deployment todos-api -n microservice-app
```

### Paso 4: Rollback si falla
```bash
# Ver historial
kubectl rollout history deployment/todos-api -n microservice-app

# Rollback a versión anterior
kubectl rollout undo deployment/todos-api -n microservice-app

# Rollback a versión específica
kubectl rollout undo deployment/todos-api --to-revision=2 -n microservice-app
```

## 4. Orden de Despliegue

1. **Base de datos/Estado**: Redis (primero)
2. **Backend APIs**: auth-api, users-api, todos-api
3. **Procesadores**: log-message-processor
4. **Frontend**: frontend (último)

## 5. Comandos Útiles
```bash
# Aplicar todos los cambios
kubectl apply -f k8s/

# Ver estado de todos los deployments
kubectl get deployments -n microservice-app

# Ver rollout en tiempo real
kubectl rollout status deployment/todos-api -n microservice-app

# Pausar un rollout
kubectl rollout pause deployment/todos-api -n microservice-app

# Reanudar un rollout
kubectl rollout resume deployment/todos-api -n microservice-app
```

## 6. Blue-Green Alternative (Avanzado)

Si necesitas cambios más drásticos sin riesgo:
```bash
# Crear deployment "green" (nueva versión)
kubectl apply -f todos-api-green-deployment.yaml

# Verificar que funcione
kubectl get pods -l version=green

# Cambiar el Service para apuntar a "green"
kubectl patch service todos-api -p '{"spec":{"selector":{"version":"green"}}}'

# Si falla, volver a "blue"
kubectl patch service todos-api -p '{"spec":{"selector":{"version":"blue"}}}'
```
