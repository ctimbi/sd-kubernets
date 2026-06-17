# Práctica Básica — Kubernetes con minikube
**Sistemas Distribuidos | Ciencias de la Computación**

> **Nivel:** Básico — primer contacto con Kubernetes  
> **Duración estimada:** 60-90 minutos  
> **Prerrequisitos:** Docker instalado y conocido, minikube instalado  

---

## Objetivos

Al finalizar esta práctica serás capaz de:

- Iniciar y gestionar un clúster Kubernetes local con minikube
- Crear y gestionar Pods usando `kubectl`
- Desplegar una aplicación web con un Deployment
- Exponer servicios con un Service de tipo NodePort
- Escalar un Deployment horizontalmente
- Ver logs e inspeccionar el estado de los recursos

---

## Contexto

Ya sabes crear contenedores Docker. Ahora imagina que tienes 50 contenedores corriendo: algunos caen, hay que distribuir el tráfico entre ellos, actualizarlos sin interrumpir el servicio. Hacer todo eso manualmente sería imposible.

**Kubernetes (K8s) automatiza todo esto.** En esta práctica desplegarás una aplicación web simple con Nginx usando los tres objetos más fundamentales de Kubernetes: `Pod`, `Deployment` y `Service`.

---

## Parte 1 — Preparar el entorno

### 1.1 Verificar prerrequisitos

Antes de comenzar, confirma que todas las herramientas están instaladas:

```bash
docker --version
minikube version
kubectl version --client
```

Deberías ver versiones de los tres. Si alguno falla, instálalo antes de continuar.

---

### 1.2 Iniciar el clúster

Arranca un clúster Kubernetes local. Usamos Docker como driver porque ya lo tienes instalado:

```bash
minikube start --driver=docker --cpus=2 --memory=4096
```

La primera vez descarga la imagen base de Kubernetes — puede tomar **2-5 minutos**. Las siguientes veces es mucho más rápido.

Cuando termine, verifica que el clúster está corriendo:

```bash
minikube status
```

Deberías ver algo como:

```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

Ahora explora el clúster con kubectl:

```bash
# Ver los nodos del clúster
kubectl get nodes

# Ver más información del nodo
kubectl get nodes -o wide
```

**¿Qué estás viendo?** En minikube hay un único nodo que actúa simultáneamente como Control Plane (el "cerebro") y Worker Node (donde corren las apps). En un clúster real de producción estos roles están en máquinas separadas.

```bash
# Ver todos los namespaces del sistema
kubectl get namespaces
```

Los namespaces `kube-system` y `kube-public` son del sistema de Kubernetes. El namespace `default` es donde crearás tus recursos.

---

## Parte 2 — Tu primer Pod

### 2.1 ¿Qué es un Pod?

Un **Pod** es la unidad mínima desplegable en Kubernetes. Puede contener uno o más contenedores que comparten la misma red (misma IP) y almacenamiento. En la práctica, la mayoría de los Pods tienen un solo contenedor principal.

La diferencia con Docker: en Docker lanzas contenedores directamente. En Kubernetes lanzas Pods que contienen contenedores.

### 2.2 Crear un Pod manualmente

Crea un archivo llamado `mi-primer-pod.yaml` con este contenido:

```yaml
# mi-primer-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mi-primer-pod
  labels:
    app: nginx
    practica: basica
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
```

**Entiende el YAML antes de ejecutarlo:**

| Campo | Qué significa |
|-------|---------------|
| `apiVersion: v1` | Versión de la API de K8s para este recurso |
| `kind: Pod` | El tipo de objeto que estás creando |
| `metadata.name` | El nombre con el que identificarás este Pod |
| `metadata.labels` | Etiquetas clave-valor para organizar y seleccionar recursos |
| `spec.containers` | La lista de contenedores que vivirán en este Pod |
| `image: nginx:1.21` | La imagen Docker a usar (igual que en `docker run`) |
| `containerPort: 80` | Puerto que expone el contenedor (informativo) |

Aplica el manifiesto para crear el Pod:

```bash
kubectl apply -f mi-primer-pod.yaml
```

Verás: `pod/mi-primer-pod created`

---

### 2.3 Inspeccionar el Pod

```bash
# Ver el Pod y su estado
kubectl get pods
```

El Pod puede tardar ~30 segundos en pasar a `Running`. Los estados posibles son:

- `Pending` → K8s lo recibió y está preparando el nodo
- `ContainerCreating` → descargando la imagen y creando el contenedor
- `Running` → el contenedor está activo
- `Completed` / `Error` / `CrashLoopBackOff` → terminó o falló

```bash
# Ver más detalles (en qué nodo está, su IP interna)
kubectl get pods -o wide

# Descripción completa del Pod
kubectl describe pod mi-primer-pod
```

En la salida de `describe` presta atención a:

- **IP:** La dirección IP interna asignada al Pod dentro del clúster
- **Node:** En qué nodo está corriendo
- **Events:** Historial de lo que le pasó (muy útil para depurar errores)
- **Containers:** Estado del contenedor, imagen usada, puertos

---

### 2.4 Interactuar con el Pod

Puedes abrir una terminal dentro del Pod, igual que con `docker exec`:

```bash
kubectl exec -it mi-primer-pod -- bash
```

Dentro del Pod ejecuta:

```bash
# Hacer una petición al servidor nginx
curl localhost

# Ver el hostname (es el nombre del Pod)
hostname

# Salir
exit
```

**Observa:** el `hostname` dentro del Pod es el mismo nombre del Pod (`mi-primer-pod`). Cada Pod tiene su propio hostname.

Ver los logs del contenedor:

```bash
kubectl logs mi-primer-pod

# Seguir los logs en tiempo real (Ctrl+C para salir)
kubectl logs -f mi-primer-pod
```

---

### 2.5 Eliminar el Pod

```bash
kubectl delete pod mi-primer-pod

# Verificar que fue eliminado
kubectl get pods
```

**Punto clave:** el Pod desapareció y no volverá. Los Pods creados manualmente son efímeros — si los eliminas o el nodo falla, no se recrean solos. Para eso existe el Deployment.

---

## Parte 3 — Deployment

### 3.1 ¿Qué es un Deployment?

Un **Deployment** le dice a Kubernetes: *"quiero que SIEMPRE haya N copias de este Pod corriendo"*. Si un Pod cae, el Deployment crea uno nuevo automáticamente. Si quieres actualizar la imagen, el Deployment lo hace sin interrumpir el servicio.

El Deployment es el objeto que usarás en el 90% de los casos reales.

### 3.2 Crear el Deployment

Crea `nginx-deployment.yaml`:

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "32Mi"
            cpu: "100m"
          limits:
            memory: "64Mi"
            cpu: "200m"
```

**Campos nuevos importantes:**

| Campo | Qué hace |
|-------|----------|
| `replicas: 3` | Mantener siempre 3 Pods de este tipo activos |
| `selector.matchLabels` | Cómo el Deployment identifica "sus" Pods (por label) |
| `template` | La plantilla con la que crea cada Pod |
| `resources.requests` | Mínimo de CPU/memoria que necesita el Pod |
| `resources.limits` | Máximo que puede consumir (evita que acapare recursos) |

> `100m` de CPU = 100 millicores = 0.1 CPU. `32Mi` = 32 Mebibytes de memoria.

```bash
kubectl apply -f nginx-deployment.yaml
```

---

### 3.3 Verificar el Deployment

```bash
# Ver el Deployment
kubectl get deployments

# Ver los 3 Pods creados automáticamente
kubectl get pods

# Ver también el ReplicaSet (intermediario entre Deployment y Pods)
kubectl get replicasets
```

**Nota los nombres de los Pods:** tienen el formato `nginx-app-[hash-replicaset]-[hash-pod]`. Kubernetes les asigna nombres únicos automáticamente.

---

### 3.4 Probar el auto-healing

Esta es una de las características más poderosas de Kubernetes. Vamos a eliminar uno de los Pods y ver qué pasa:

```bash
# Copia el nombre de uno de los Pods de la lista anterior
kubectl get pods

# Elimina ese Pod (sustituye <nombre-pod> por el nombre real)
kubectl delete pod <nombre-pod>

# Observa en tiempo real cómo K8s reacciona
kubectl get pods -w
```

El flag `-w` (watch) actualiza la lista automáticamente. Verás cómo el Pod eliminado pasa a `Terminating` y al mismo tiempo aparece uno nuevo en `ContainerCreating` y luego `Running`.

Presiona `Ctrl+C` para salir del watch.

**¿Por qué sucede esto?** El Deployment tiene `replicas: 3`. El Controller Manager de K8s detecta constantemente que el número real de Pods no coincide con el número deseado y crea uno nuevo para corregirlo.

---

### 3.5 Escalar el Deployment

```bash
# Escalar a 5 réplicas
kubectl scale deployment nginx-app --replicas=5

kubectl get pods
```

K8s crea 2 Pods nuevos inmediatamente. Ahora escala hacia abajo:

```bash
kubectl scale deployment nginx-app --replicas=2

kubectl get pods
```

K8s elimina 3 Pods. El mismo comando que escala hacia arriba funciona hacia abajo.

---

## Parte 4 — Service: exponer la aplicación

### 4.1 ¿Por qué necesitamos un Service?

Los Pods tienen IPs internas que **cambian cada vez que se recrean**. Si el backend de tu app le preguntara directamente la IP de un Pod frontend, esa IP dejaría de funcionar en cuanto el Pod se reiniciara.

Un **Service** provee una IP virtual estable y un nombre DNS que siempre apunta a los Pods correctos, sin importar cuántas veces se hayan reiniciado.

### 4.2 Crear un Service NodePort

Crea `nginx-service.yaml`:

```yaml
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
```

**Entiende el flujo de puertos:**

```
Tú (browser) → nodePort 30080 → port 80 del Service → targetPort 80 del Pod
```

El campo `selector: app: nginx` conecta el Service con todos los Pods que tengan el label `app: nginx`. Así K8s sabe a qué Pods enviar el tráfico.

```bash
kubectl apply -f nginx-service.yaml
```

---

### 4.3 Acceder a la aplicación

```bash
# minikube gestiona el acceso al NodePort automáticamente
minikube service nginx-service
```

Esto abre el browser con la URL correcta. También puedes obtener solo la URL:

```bash
minikube service nginx-service --url
```

Copia esa URL y prueba con curl:

```bash
curl <URL-obtenida>
```

Deberías ver el HTML de la página de bienvenida de Nginx.

---

### 4.4 Inspeccionar el Service

```bash
kubectl get services

kubectl describe service nginx-service
```

En la descripción, observa:

- **Endpoints:** lista de `IP:puerto` de cada Pod conectado al Service. Si eliminas un Pod y K8s crea uno nuevo, los Endpoints se actualizan automáticamente.
- **NodePort:** el puerto externo (30080)
- **Selector:** `app=nginx` — la regla que conecta el Service con los Pods

---

## Parte 5 — ConfigMap: personalizar Nginx

### 5.1 ¿Qué es un ConfigMap?

Un **ConfigMap** almacena configuración no sensible (texto, parámetros, archivos de configuración) separada del contenedor. Así puedes usar la misma imagen Docker en diferentes entornos con diferente configuración.

### 5.2 Crear el ConfigMap

Crea `nginx-config.yaml`:

```yaml
# nginx-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-html
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Mi App en Kubernetes</title></head>
    <body>
      <h1>Hola desde Kubernetes!</h1>
      <p>Sistemas Distribuidos — Práctica Básica</p>
    </body>
    </html>
```

```bash
kubectl apply -f nginx-config.yaml

# Ver que fue creado
kubectl get configmaps
kubectl describe configmap nginx-html
```

---

### 5.3 Montar el ConfigMap en el Deployment

Actualiza `nginx-deployment.yaml` para usar el ConfigMap como volumen:

```yaml
# nginx-deployment.yaml (versión actualizada)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html   # Ruta dentro del contenedor
      volumes:
      - name: html-volume
        configMap:
          name: nginx-html                   # Nombre del ConfigMap
```

**¿Qué hace esto?** Monta el archivo `index.html` del ConfigMap en la ruta `/usr/share/nginx/html` del contenedor. Nginx sirve lo que haya en esa ruta.

```bash
kubectl apply -f nginx-deployment.yaml

# Ver cómo los Pods se actualizan con rolling update
kubectl rollout status deployment/nginx-app
```

Accede nuevamente al servicio y verás tu página personalizada:

```bash
minikube service nginx-service
```

---

## Parte 6 — Dashboard y limpieza

### 6.1 Explorar el Dashboard de Kubernetes

minikube incluye la interfaz gráfica oficial de Kubernetes:

```bash
minikube dashboard
```

Explora estas secciones:

- **Workloads → Deployments:** ve el `nginx-app` con sus réplicas y el estado del rollout
- **Workloads → Pods:** ve cada Pod individualmente, sus logs y eventos
- **Service and Load Balancing → Services:** ve el `nginx-service` con sus Endpoints
- **Config and Storage → ConfigMaps:** ve el `nginx-html` con el contenido del archivo

El dashboard es una excelente forma de visualizar lo que pasa en el clúster.

---

### 6.2 Ver el resumen completo

```bash
# Ver TODOS los recursos del namespace actual de una sola vez
kubectl get all
```

En la salida identificarás: `pod/`, `service/`, `deployment.apps/` y `replicaset.apps/`. Esta vista te muestra el estado completo del namespace.

---

### 6.3 Limpiar los recursos

```bash
kubectl delete -f nginx-service.yaml
kubectl delete -f nginx-deployment.yaml
kubectl delete -f nginx-config.yaml

# Verificar que quedó limpio
kubectl get all
```

Solo debería quedar el Service `kubernetes` que es del sistema.

Para detener el clúster (preserva el estado para la próxima vez):

```bash
minikube stop
```

---

## Preguntas de reflexión

Responde estas preguntas basándote en lo que observaste:

**1. Pod vs Deployment**  
¿Por qué se recomienda usar un Deployment en lugar de crear Pods manualmente? ¿Qué pasó cuando eliminaste un Pod del Deployment?

**2. Auto-healing**  
Explica con tus propias palabras cómo funciona el auto-healing. ¿Qué componente del Control Plane es responsable de detectar que un Pod cayó y crear uno nuevo?

**3. Service y selección**  
¿Cómo sabe el Service a qué Pods enviarle el tráfico? ¿Qué pasaría si crearas un Pod con el label `app: nginx` en otro Deployment — el Service también le enviaría tráfico?

**4. ConfigMap**  
¿Por qué es mejor usar un ConfigMap para inyectar el `index.html` en vez de construir una imagen Docker personalizada con ese archivo? ¿En qué escenario del mundo real sería especialmente útil?

**5. kubectl vs docker**  
Compara los siguientes pares y explica similitudes y diferencias:

| Docker | Kubernetes |
|--------|-----------|
| `docker run nginx` | `kubectl apply -f pod.yaml` |
| `docker exec -it c bash` | `kubectl exec -it pod -- bash` |
| `docker logs c` | `kubectl logs pod` |
| `docker stop c` | `kubectl delete pod pod-name` |

---

## Resumen de comandos

| Comando | Descripción |
|---------|-------------|
| `kubectl apply -f archivo.yaml` | Crear o actualizar recursos desde YAML |
| `kubectl get pods` | Listar Pods en el namespace actual |
| `kubectl get all` | Listar todos los recursos |
| `kubectl describe pod <nombre>` | Detalle completo de un Pod |
| `kubectl logs <pod>` | Ver logs del Pod |
| `kubectl logs -f <pod>` | Seguir logs en tiempo real |
| `kubectl exec -it <pod> -- bash` | Terminal dentro del Pod |
| `kubectl delete -f archivo.yaml` | Eliminar recursos definidos en YAML |
| `kubectl scale deployment <n> --replicas=N` | Escalar el número de réplicas |
| `kubectl rollout status deployment/<n>` | Ver el estado de un rollout |
| `kubectl get pods -w` | Ver cambios en Pods en tiempo real |
| `minikube start` | Iniciar el clúster local |
| `minikube stop` | Detener el clúster |
| `minikube service <nombre>` | Abrir un Service NodePort en el browser |
| `minikube dashboard` | Abrir la UI web de Kubernetes |
