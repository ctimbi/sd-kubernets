# Tarea — Migración a Kubernetes
**Sistemas Distribuidos | Ciencias de la Computación**

| | |
|---|---|
| **Modalidad** | Pareja |
| **Entregables** | Informe técnico + Video demo |

---

## Contexto

Durante las prácticas 1 y 2 se aprendieron a desplegar aplicaciones en Kubernetes usando los objetos fundamentales del ecosistema. Ahora darán un paso hacia la realidad: tomar un sistema ya existente, diseñado para Docker Compose, y llevar esa arquitectura a Kubernetes tomando decisiones técnicas fundamentadas.

El sistema de referencia es **[mas-alla-del-localhost](https://github.com/ctimbi/mas-alla-del-localhost)**: una aplicación de producción simulada que implementa balanceo de carga, caché distribuido, rate limiting y observabilidad completa con Prometheus y Grafana.

```
Cliente
  │
  ▼ :80
┌──────────────────────────────────────────┐
│          Nginx (Load Balancer)           │
└─────────────┬──────────────┬────────────┘
              │              │
         ┌────▼────┐    ┌────▼────┐
         │  api1   │    │  api2   │
         │ FastAPI │    │ FastAPI │
         └────┬────┘    └────┬────┘
              └──────┬───────┘
                     ▼
               ┌───────────┐
               │   Redis   │  ← Caché + Rate limiting
               └───────────┘

Observabilidad:
  api1, api2 ──► Prometheus ──► Grafana
  redis-exporter, nginx-exporter
```

Esta arquitectura funciona perfectamente en un solo host. Su objetivo es demostrar que Kubernetes no es solo "más contenedores", sino una plataforma que cambia fundamentalmente cómo se diseña, opera y escala un sistema distribuido.

---

## Objetivo general

Migrar la arquitectura de **mas-alla-del-localhost** desde Docker Compose a Kubernetes, tomando decisiones técnicas justificadas, aplicando las buenas prácticas vistas en las prácticas de laboratorio, y demostrando el funcionamiento real del sistema con observabilidad activa.

---

## Requerimientos

### 1. Análisis previo

Antes de escribir un solo YAML, el grupo debe realizar un análisis de la arquitectura existente que responda:

- ¿Qué servicios se pueden escalar horizontalmente y cuáles no? ¿Por qué?
- ¿Qué configuración es sensible (credenciales) y qué es configuración operacional? ¿Cómo se gestiona cada una en Kubernetes?
- ¿Qué tipo de Service (ClusterIP, NodePort, LoadBalancer) corresponde a cada componente y por qué?
- ¿Cómo se maneja la imagen de la API (`Dockerfile` propio) en un entorno minikube sin registry externo?
- ¿Qué endpoints del sistema pueden usarse como `readinessProbe` y `livenessProbe`? (Revisar `/health` y `/ready` en el repositorio)

Este análisis debe quedar documentado en el informe técnico y es la base de todas las decisiones de implementación.

---

### 2. Implementación en Kubernetes

La migración debe implementarse en **minikube** e incluir como mínimo:

**Organización**
- Namespace dedicado para el proyecto
- Estructura de archivos YAML coherente y organizada en el repositorio

**Gestión de configuración**
- Secrets para credenciales (al menos: contraseña de Redis, credenciales de Grafana)
- ConfigMaps para archivos de configuración (nginx.conf, prometheus.yml)

**Workloads**
- Deployment para cada uno de los servicios del stack
- Réplicas justificadas según el rol de cada servicio
- `readinessProbe` y `livenessProbe` configurados donde corresponda
- Resource requests y limits definidos para todos los Pods

**Redes**
- Services correctamente tipificados para cada componente
- El sistema debe ser accesible desde el exterior (al menos la API vía Nginx)

**Escalado automático**
- HPA configurado para el Deployment de la API
- El umbral de CPU debe estar justificado

**Actualizaciones**
- Estrategia `rollingUpdate` configurada en los Deployments que lo ameriten
- Demostración de un rolling update y un rollback exitoso

---

### 3. Observabilidad

El grupo debe demostrar que el sistema *realmente funciona* desde dentro de Kubernetes.

**Métricas activas**
- Prometheus scrapeando las métricas de la API, redis-exporter y nginx-exporter desde dentro del clúster
- Al menos un dashboard funcional en Grafana mostrando: tráfico de la API, uso de caché Redis, estado de las instancias

**Evidencia durante la demo**
- Generar tráfico y mostrar las métricas actualizándose en Grafana en tiempo real
- Mostrar el comportamiento del HPA bajo carga (usando el load-generator, jmeter o cualquier herramienta similar)

---

### 4. Mejoras propuestas e implementadas

El grupo debe proponer e implementar **al menos dos mejoras** sobre la arquitectura original que Kubernetes habilita y Docker Compose no puede ofrecer de forma nativa. Ejemplos (no limitativos):

- **PodDisruptionBudget** para garantizar disponibilidad mínima durante mantenimiento
- **ResourceQuota** en el namespace para limitar el consumo total del proyecto
- **NetworkPolicy** para restringir qué Pods pueden comunicarse con Redis
- **Init container** para verificar que Redis esté disponible antes de arrancar la API
- **Alertas en Grafana** (error rate > umbral, latencia elevada, Pod caído) conectadas a la realidad observada
- **Ingress** en lugar de NodePort para gestionar el acceso externo de forma más profesional
- **Liveness probe con reinicio automático** — demostrar que K8s reinicia el Pod cuando el probe falla

Cada mejora debe estar justificada: por qué Docker Compose no la resuelve y qué problema operacional soluciona.

---

## Entregables

### Informe técnico (PDF)

Documento estructurado que incluya:

1. **Análisis de la arquitectura original** — respuestas a las preguntas del punto 1
2. **Decisiones técnicas** — para cada componente: qué tipo de objeto K8s se usó y por qué
3. **Comparativa Docker Compose vs Kubernetes** — qué se ganó, qué complicaciones, qué se aprendió
4. **Mejoras implementadas** — descripción, justificación y evidencia de funcionamiento
5. **Reflexión grupal** — ¿En qué escenario real usarían Kubernetes? ¿Cuándo seguirían con Docker Compose?

### Video demo (máximo 10 minutos)

Grabación de pantalla que demuestre el sistema en ejecución. El video debe mostrar:

1. El namespace con todos los Pods en estado `Running` (`kubectl get all -n <namespace>`)
2. Acceso a la API desde el exterior y verificación del balanceo de carga entre réplicas
3. El dashboard de Grafana con métricas activas (tráfico real, no capturas estáticas)
4. Ejecución de un rolling update — sin interrupciones visibles en el tráfico
5. Ejecución de un rollback
6. El HPA respondiendo a carga generada
7. Al menos una de las mejoras implementadas en funcionamiento


---

## Criterios de evaluación

| Criterio | Puntos |
|----------|--------|
| **Análisis técnico** — calidad del análisis previo y justificación de decisiones | 20 |
| **Implementación base** — namespace, Secrets, ConfigMaps, Deployments, Services, probes, resources | 25 |
| **Rolling Update y Rollback** — configuración correcta y demostración | 10 |
| **HPA** — configuración justificada y evidencia de escalado | 10 |
| **Observabilidad** — Prometheus + Grafana funcionales, métricas reales en el video | 20 |
| **Mejoras** — al menos 2 implementadas, justificadas y evidenciadas | 10 |
| **Claridad del informe y video** — estructura, redacción técnica, evidencias | 5 |
| **Total** | **100** |

---

## Repositorio de referencia

[https://github.com/ctimbi/mas-alla-del-localhost](https://github.com/ctimbi/mas-alla-del-localhost)

Clonen el repositorio, estudien el `docker-compose.yml`, el `Dockerfile`, la estructura de `nginx/`, `prometheus/` y `grafana/`, y el código de la API en `app/`. Todo lo que necesitan para entender el sistema está ahí.

---

## Recomendaciones

- Lean el `README.md` completo del repositorio antes de empezar — explica los conceptos demostrados, el troubleshooting conocido y los próximos pasos sugeridos por el autor.
- Para la imagen de la API (que tiene `Dockerfile` propio), investiguen cómo cargar imágenes locales en minikube sin necesidad de un registry externo: `minikube image load` o el uso del daemon Docker de minikube (`eval $(minikube docker-env)`).
- La observabilidad es la evidencia de que el sistema funciona. Un sistema "desplegado" pero sin métricas activas es un sistema que no puede monitorearse en producción.

---

*Cualquier duda técnica sobre el enunciado puede consultarse durante las horas de clase o a través del canal de whatsapp del curso.*
