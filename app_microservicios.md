# Práctica Intermedia — Kubernetes con minikube
**Sistemas Distribuidos | Ciencias de la Computación**

> **Nivel:** Intermedio — requiere haber completado la Práctica Básica  
> **Duración estimada:** 90-120 minutos  
> **Prerrequisitos:** Práctica básica completada, Docker, kubectl, minikube  

---

## Objetivos

Al finalizar esta práctica será capaz de:

- Desplegar una arquitectura de microservicios (frontend + backend + base de datos)
- Usar Secrets para gestionar credenciales de forma segura
- Realizar rolling updates sin tiempo de inactividad
- Ejecutar un rollback ante una actualización fallida
- Usar `kubectl port-forward` para depurar servicios internos
- Escalar automáticamente con el Horizontal Pod Autoscaler (HPA)
- Aplicar buenas prácticas: namespaces, labels y resource limits

---

## Arquitectura de la práctica

Desplegarás una **To-Do App** compuesta por tres microservicios que se comunican entre sí:

```
                                  ┌─────────────────────────┐
                                  │      Namespace: todo-app │
                                  │                          │
Usuario ──► NodePort 30090 ──►  Frontend (nginx:1.25)        │
                                  │       │                  │
                                  │       ▼ ClusterIP :8080  │
                                  │  Backend API (httpd:2.4) │
                                  │       │                  │
                                  │       ▼ ClusterIP :5432  │
                                  │  PostgreSQL (postgres:15)│
                                  └─────────────────────────┘
```

| Servicio | Imagen | Exposición | Rol |
|----------|--------|------------|-----|
| Frontend | `nginx:1.25` | NodePort 30090 | Interfaz de usuario |
| Backend API | `httpd:2.4` | ClusterIP :8080 | API interna |
| Base de datos | `postgres:15` | ClusterIP :5432 | Persistencia |

**Punto clave de diseño:** el backend y la base de datos usan `ClusterIP` — solo son accesibles desde dentro del clúster. Solo el frontend tiene exposición externa. Esto es una práctica de seguridad estándar en microservicios.

---

## Parte 1 — Namespace dedicado

### 1.1 ¿Por qué usar un Namespace?

Un **Namespace** es una partición lógica del clúster. Permite que múltiples proyectos o equipos compartan el mismo clúster sin interferir entre sí. Puede aplicar cuotas de recursos, control de acceso y políticas de red por namespace.

Para esta práctica, todo irá en el namespace `todo-app` en lugar del namespace `default`.

Cree `namespace.yaml`:

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: todo-app
  labels:
    proyecto: practica-intermedia
```

```bash
kubectl apply -f namespace.yaml

# Verificar
kubectl get namespaces
```

Ahora configura kubectl para que use `todo-app` como namespace por defecto en esta sesión:

```bash
kubectl config set-context --current --namespace=todo-app
```

A partir de aquí, todos sus comandos `kubectl` operan sobre `todo-app` sin necesidad de agregar `-n todo-app` cada vez.

---

## Parte 2 — Base de datos con Secret

### 2.1 ¿Qué es un Secret?

Un **Secret** es como un ConfigMap pero para datos sensibles: contraseñas, tokens, certificados. Los valores se almacenan en base64 dentro de etcd y K8s los inyecta en los Pods como variables de entorno o archivos montados.

> **Importante:** base64 es codificación, no cifrado. Cualquier persona con acceso a `kubectl` puede decodificarlo. En producción se usan herramientas como Sealed Secrets o HashiCorp Vault.

### 2.2 Crear el Secret de PostgreSQL

Cree el Secret con las credenciales de la base de datos:

```bash
kubectl create secret generic postgres-secret \
  --from-literal=POSTGRES_USER=dbadmin \
  --from-literal=POSTGRES_PASSWORD=MiPassword2024! \
  --from-literal=POSTGRES_DB=todoapp
```

Inspecciona lo que se creó:

```bash
kubectl get secrets

kubectl describe secret postgres-secret
```

Observa que `describe` muestra los nombres de las claves pero **no los valores** — solo dice cuántos bytes tiene cada uno.

Para ver el valor real (decodificando base64):

```bash
kubectl get secret postgres-secret -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d
```

---

### 2.3 Desplegar PostgreSQL

Cree `postgres.yaml` con el Deployment y su Service:

```yaml
# postgres.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: todo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        tier: database
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        envFrom:
        - secretRef:
            name: postgres-secret   # Inyecta TODAS las claves del Secret como env vars
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "300m"
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: todo-app
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

**Qué hace `envFrom` + `secretRef`:** toma cada clave del Secret (`POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`) y la convierte en una variable de entorno dentro del contenedor. Es equivalente a:

```bash
docker run -e POSTGRES_USER=dbadmin -e POSTGRES_PASSWORD=... postgres:15
```

```bash
kubectl apply -f postgres.yaml

# Esperar a que PostgreSQL quede Running (puede tardar 30-60 segundos)
kubectl get pods -w
# Ctrl+C cuando veas Running
```

El Service de tipo `ClusterIP` le da a PostgreSQL un nombre DNS interno: `postgres-service`. Los demás Pods del namespace pueden conectarse a `postgres-service:5432`.

---

## Parte 3 — Backend API

### 3.1 ConfigMap para el backend

Los datos de configuración no sensibles (URL de la base de datos, puertos, nivel de log) van en un ConfigMap:

```yaml
# backend-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: todo-app
data:
  DB_HOST: "postgres-service"   # Nombre DNS del Service de PostgreSQL
  DB_PORT: "5432"
  APP_ENV: "production"
  APP_PORT: "8080"
  LOG_LEVEL: "info"
```

```bash
kubectl apply -f backend-config.yaml
```

---

### 3.2 Desplegar el backend

Cree `backend.yaml`. Observe cómo combina ConfigMap y Secret en el mismo Pod:

```yaml
# backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: todo-app
  labels:
    version: "1.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Pods extra que puede crear durante el update
      maxUnavailable: 0  # Pods que pueden estar caídos durante el update
  template:
    metadata:
      labels:
        app: backend
        tier: api
        version: "1.0"
    spec:
      containers:
      - name: backend
        image: httpd:2.4
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: backend-config      # Variables del ConfigMap
        env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_USER      # Solo esta clave del Secret
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_PASSWORD  # Solo esta clave del Secret
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: todo-app
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 80
```

**Tres conceptos nuevos en este YAML:**

**`strategy.rollingUpdate`** — controla cómo se actualizan los Pods:
- `maxSurge: 1` → puede crear 1 Pod extra durante el update (temporalmente habrá 3 Pods)
- `maxUnavailable: 0` → ningún Pod puede estar caído durante el update

Con esta configuración, el servicio siempre tiene al menos 2 Pods disponibles durante una actualización.

**`readinessProbe`** — antes de enviarle tráfico a un Pod nuevo, K8s verifique que esté listo haciendo un GET a `/`. Si el probe falla, el Pod no recibe tráfico aunque esté "corriendo". Esto evita que los usuarios vean errores durante el inicio del contenedor.

**`env` + `secretKeyRef`** — en vez de inyectar todo el Secret, inyecta solo las claves que necesitas con los nombres de variable que quieres.

```bash
kubectl apply -f backend.yaml
kubectl get pods
```

---

## Parte 4 — Frontend

### 4.1 ConfigMap con la página web

```yaml
# frontend-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-html
  namespace: todo-app
data:
  index.html: |
    <!DOCTYPE html>
    <html lang="es">
    <head>
      <meta charset="UTF-8">
      <title>To-Do App - Kubernetes</title>
      <style>
        body { font-family: Arial; max-width: 800px; margin: 50px auto; padding: 20px; }
        h1 { color: #065A82; }
        .info { background: #E8F5E9; padding: 15px; border-radius: 5px; margin: 15px 0; }
        .badge { background: #1C7293; color: white; padding: 3px 8px; border-radius: 3px; }
      </style>
    </head>
    <body>
      <h1>To-Do App en Kubernetes</h1>
      <div class="info">
        <p>Sistemas Distribuidos — Práctica Intermedia</p>
        <p>Ambiente: <span class="badge">production</span></p>
        <p>Versión Frontend: <span class="badge">1.0</span></p>
        <p>Backend API: <code>backend-service:8080</code></p>
      </div>
    </body>
    </html>
```

```bash
kubectl apply -f frontend-config.yaml
```

---

### 4.2 Desplegar el frontend

```yaml
# frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: todo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
      - name: frontend
        image: nginx:1.25
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: html
        configMap:
          name: frontend-html
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: todo-app
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30090
```

```bash
kubectl apply -f frontend.yaml

# Ver el estado de todos los Pods
kubectl get pods

# Acceder al frontend
minikube service frontend-service -n todo-app
```

Debería ver la página HTML que definió en el ConfigMap.

---

## Parte 5 — Observabilidad y depuración

### 5.1 Vista general del namespace

```bash
# Todos los recursos de todo-app de una sola vez
kubectl get all -n todo-app

# Pods con más detalle (IPs, nodo)
kubectl get pods -o wide

# Eventos recientes ordenados por tiempo (muy útil para depurar)
kubectl get events --sort-by=.lastTimestamp
```

---

### 5.2 Logs por servicio

```bash
# Logs de todos los Pods del frontend a la vez (con -l usas el selector por label)
kubectl logs -l app=frontend

# Logs del backend en tiempo real
kubectl logs -f -l app=backend

# Logs de PostgreSQL
kubectl logs -l app=postgres
```

Usar `-l app=<nombre>` en vez del nombre de un Pod específico es útil cuando hay múltiples réplicas — te muestra los logs de todas a la vez.

---

### 5.3 `kubectl port-forward` — acceso directo para depuración

`port-forward` crea un túnel temporal entre su máquina y un Pod o Service dentro del clúster, sin necesidad de un NodePort externo:

```bash
# Redirigir el puerto 8080 de su máquina al puerto 80 del Deployment backend
kubectl port-forward deployment/backend 8080:80
```

Mientras el comando está corriendo (en otra terminal o browser):

```bash
curl localhost:8080
```

Debería ver la respuesta del servidor Apache. Presiona `Ctrl+C` para cerrar el túnel.

**¿Cuándo es útil?** Cuando un servicio usa `ClusterIP` (sin acceso externo) y necesitas inspeccionarlo directamente: probar una API, conectarte a una base de datos, revisar métricas. No es para tráfico de producción.

```bash
# También puedes hacer port-forward a un Service
kubectl port-forward service/postgres-service 5432:5432
# Con esto podrías conectar un cliente local como DBeaver a localhost:5432
# Ctrl+C para cerrar
```

---

### 5.4 Explorar las variables de entorno dentro de un Pod

```bash
# Entrar al Pod del backend
kubectl exec -it deployment/backend -- sh

# Dentro del Pod, ver las variables de entorno inyectadas
env | grep DB
env | grep APP

# Salir
exit
```

Verá que `DB_HOST`, `DB_PORT`, `APP_ENV`, `DB_USER` y `DB_PASSWORD` están disponibles como variables de entorno normales del sistema operativo — aunque vengan de fuentes distintas (ConfigMap y Secret).

---

## Parte 6 — Rolling Update y Rollback

Esta sección demuestra una de las características más importantes de K8s en producción: **actualizar aplicaciones sin interrumpir el servicio**.

### 6.1 Rolling Update exitoso

Actualiza la imagen del frontend a una versión más ligera:

```bash
kubectl set image deployment/frontend frontend=nginx:1.25-alpine
```

Observa el proceso en tiempo real:

```bash
kubectl rollout status deployment/frontend
```

Mientras el rollout ocurre, verifique que el servicio sigue respondiendo:

```bash
# En otra terminal, obtenga la URL
minikube service frontend-service -n todo-app --url

# Haga varias peticiones seguidas mientras el rollout ocurre
curl <URL-del-servicio>
```

El servicio responde sin interrupciones durante toda la actualización.

**¿Por qué no hay downtime?** Con `maxUnavailable: 0`, K8s crea el Pod nuevo y espera a que su `readinessProbe` pase antes de eliminar el viejo. En ningún momento el número de Pods activos cae por debajo de 2.

Ver el historial de versiones del Deployment:

```bash
kubectl rollout history deployment/frontend
```

---

### 6.2 Simular una actualización fallida

Actualiza a una imagen que no existe:

```bash
kubectl set image deployment/frontend frontend=nginx:esta-version-no-existe
```

Observa lo que pasa:

```bash
# El rollout se queda colgado esperando
kubectl rollout status deployment/frontend

# Ver los Pods — los nuevos quedarán en error
kubectl get pods
```

Los Pods nuevos quedan en `ErrImagePull` o `ImagePullBackOff` porque la imagen no existe. Sin embargo, los Pods viejos (con la versión funcional) **siguen corriendo**. El servicio sigue disponible para los usuarios.

Inspecciona el error de uno de los Pods fallidos:

```bash
kubectl describe pod <nombre-del-pod-con-error>
```

Busca la sección **Events** al final. Verás algo como:

```
Failed to pull image "nginx:esta-version-no-existe": not found
```

---

### 6.3 Rollback

Ante una actualización fallida, revierten en segundos:

```bash
kubectl rollout undo deployment/frontend
```

```bash
# Ver que el rollback está en progreso
kubectl rollout status deployment/frontend

# Todos los Pods deben volver a Running
kubectl get pods

# Confirmar qué imagen está usando ahora
kubectl describe deployment frontend | grep Image
```

Puede volver a una revisión específica del historial:

```bash
# Ver el historial con detalle de imágenes
kubectl rollout history deployment/frontend --revision=1

# Rollback a la revisión 1 específicamente
kubectl rollout undo deployment/frontend --to-revision=1
```

---

## Parte 7 — Horizontal Pod Autoscaler (HPA)

### 7.1 ¿Qué es el HPA?

El **Horizontal Pod Autoscaler** ajusta automáticamente el número de réplicas de un Deployment en función del uso de CPU (u otras métricas). Cuando hay mucha carga, escala hacia arriba. Cuando la carga baja, reduce las réplicas hasta el mínimo configurado.

### 7.2 Habilitar el metrics-server

El HPA necesita datos de uso de CPU, que los provee el `metrics-server`:

```bash
minikube addons enable metrics-server

# Esperar ~30 segundos y verificar
kubectl top nodes
kubectl top pods
```

Si `kubectl top` devuelve datos numéricos, el metrics-server está listo.

---

### 7.3 Crear el HPA para el frontend

```yaml
# frontend-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
  namespace: todo-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50   # Escala cuando el uso promedio supere el 50%
```

```bash
kubectl apply -f frontend-hpa.yaml

# Ver el estado del HPA
kubectl get hpa

kubectl describe hpa frontend-hpa
```

En la columna `TARGETS` verá el uso actual de CPU vs el objetivo (50%). En la columna `REPLICAS` verá cuántas réplicas están activas.

**Regla de funcionamiento:**
- Si uso CPU > 50% → agrega réplicas (hasta 6)
- Si uso CPU < 50% → elimina réplicas (hasta 2)
- La reducción de réplicas tiene un período de enfriamiento de ~5 minutos para evitar oscilaciones

---

### 7.4 (Opcional) Generar carga para ver el HPA en acción

```bash
# Crear un Pod temporal que genere peticiones continuas al frontend
kubectl run load-generator \
  --image=busybox \
  --rm -it \
  --restart=Never \
  -- sh -c 'while true; do wget -q -O- http://frontend-service; done'
```

En otra terminal, observa cómo el HPA responde:

```bash
# Ver el escalado en tiempo real (actualiza cada 30 segundos aprox.)
kubectl get hpa -w
```

Presiona `Ctrl+C` en el load-generator para detener la carga. El HPA reducirá las réplicas gradualmente.

---

## Parte 8 — Limpieza

### 8.1 Eliminar el namespace completo

La forma más rápida de limpiar todo es eliminar el namespace — K8s elimina automáticamente todos los recursos que contenía:

```bash
kubectl delete namespace todo-app

# Verificar
kubectl get namespaces
```

Vuelve al namespace `default`:

```bash
kubectl config set-context --current --namespace=default
```

Para detener minikube:

```bash
minikube stop
```

---

## Preguntas de reflexión

**1. Diseño de servicios**  
El Service de PostgreSQL y el del backend son `ClusterIP` mientras que el frontend es `NodePort`. ¿Por qué se toma esa decisión de diseño? ¿Qué implicaciones de seguridad tiene exponer la base de datos directamente al exterior?

**2. ConfigMap vs Secret**  
En el backend, `DB_HOST` viene de un ConfigMap pero `DB_PASSWORD` viene de un Secret. ¿Por qué se separan en dos objetos distintos? ¿Podrías poner todo en el Secret? ¿Sería una buena práctica?

**3. Rolling Update**  
Explique paso a paso qué sucedió cuando actualizó a la imagen inexistente. ¿Por qué los usuarios no sufrieron interrupciones? ¿Qué parámetros del Deployment lo garantizaron?

**4. readinessProbe**  
¿Cuál es la diferencia entre `readinessProbe` y `livenessProbe`? (Investiga `livenessProbe` en la documentación oficial de K8s en [kubernetes.io](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)). ¿Qué pasa si el `readinessProbe` falla pero el contenedor sigue corriendo?

**5. HPA y límites**  
El HPA tiene `minReplicas=2` y `maxReplicas=6`. Si llegas a un pico de tráfico que requiere 10 réplicas para responder bien, ¿qué sucede? ¿Cómo resolverías esto? ¿Qué es el **Cluster Autoscaler** y en qué se diferencia del HPA?

**6. Namespaces y red**  
¿Puede un Pod del namespace `todo-app` comunicarse con un Pod del namespace `default`? Si es posible, ¿cómo? ¿Qué mecanismo de K8s podrías usar para bloquearlo?

---

## Resumen de nuevos conceptos

| Concepto | Descripción |
|----------|-------------|
| **Namespace** | Partición lógica del clúster para aislar recursos por proyecto o equipo |
| **Secret** | Almacena datos sensibles en base64 con control de acceso separado |
| **`envFrom` + `secretRef`** | Inyecta todas las claves de un Secret como variables de entorno |
| **`env` + `secretKeyRef`** | Inyecta claves individuales de un Secret con nombre personalizado |
| **`rollingUpdate`** | Estrategia de update: `maxSurge` y `maxUnavailable` controlan el ritmo |
| **`readinessProbe`** | K8s no envía tráfico al Pod hasta que el probe responda exitosamente |
| **`kubectl rollout undo`** | Revertir un Deployment a la versión anterior en segundos |
| **HPA** | Escala réplicas automáticamente según uso de CPU u otras métricas |
| **`kubectl port-forward`** | Túnel temporal para acceder a Pods/Services sin exposición externa |
| **DNS interno K8s** | `nombre-service.namespace.svc.cluster.local` — resolución automática entre Pods |

---

## Resumen de comandos nuevos

| Comando | Descripción |
|---------|-------------|
| `kubectl config set-context --current --namespace=<ns>` | Cambiar namespace activo |
| `kubectl create secret generic <n> --from-literal=k=v` | Crear Secret desde CLI |
| `kubectl logs -l app=<label>` | Logs de todos los Pods con ese label |
| `kubectl port-forward deployment/<n> 8080:80` | Túnel a un Deployment |
| `kubectl port-forward service/<n> 5432:5432` | Túnel a un Service |
| `kubectl set image deployment/<n> <c>=<imagen>` | Actualizar imagen de un Deployment |
| `kubectl rollout status deployment/<n>` | Ver progreso de un rollout |
| `kubectl rollout history deployment/<n>` | Ver historial de versiones |
| `kubectl rollout undo deployment/<n>` | Rollback a la versión anterior |
| `kubectl rollout undo deployment/<n> --to-revision=N` | Rollback a revisión específica |
| `kubectl get hpa` | Ver el estado del HPA |
| `kubectl top pods` | Ver uso de CPU y memoria de los Pods |
| `kubectl delete namespace <n>` | Eliminar namespace y todos sus recursos |
