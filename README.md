# DevOps GitOps Final

Proyecto final del curso de DevOps. Este repositorio contiene la configuración declarativa que Argo CD utiliza para desplegar la aplicación y el stack de monitoreo en un clúster K3s.

El repositorio funciona como la **fuente única de verdad del estado deseado de Kubernetes**. Aquí no se desarrolla la aplicación; aquí se define qué imágenes, servicios, rutas y componentes deben existir en el clúster.

## Objetivo académico

Demostrar un flujo GitOps completo:

```text
Cambio en Git → Argo CD detecta el cambio → Kubernetes se reconcilia → aplicación actualizada
```

La implementación forma parte del proyecto final del curso de DevOps y utiliza una VM de OCI con K3s single-node, Argo CD, GitHub, GHCR, Prometheus y Grafana.

## Relación con el repositorio de aplicación

El repositorio de aplicación es:

```text
https://github.com/RenzoAL7/gitops-sentiment-demo
```

Su función es construir y publicar las imágenes Docker. Este repositorio GitOps recibe el tag de la imagen y define cómo debe desplegarse en K3s.

```text
Repositorio de aplicación → GitHub Actions → GHCR
                                      ↓
                              Repositorio GitOps
                                      ↓
                                  Argo CD
                                      ↓
                                     K3s
```

## Estructura

```text
devops-gitops-final/
├── argocd/
│   ├── application.yaml
│   └── monitoring-application.yaml
├── manifests/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── sentiment-service.yaml
│   ├── sentiment-service-monitor.yaml
│   └── namespace.yaml
└── README.md
```

## Applications de Argo CD

### devops-demo-app

Está definida en argocd/application.yaml y observa:

- Repositorio: este repositorio GitOps.
- Rama: main.
- Carpeta: manifests.
- Namespace destino: devops-course.
- Sincronización automática, prune y self-heal habilitados.

Esta Application despliega la aplicación principal y el sentiment-service.

### monitoring-stack

Está definida en argocd/monitoring-application.yaml y utiliza el chart kube-prometheus-stack desde el repositorio oficial de Prometheus Community.

Incluye:

- Prometheus.
- Grafana.
- kube-state-metrics.
- node-exporter.
- Operator de Prometheus.
- ServiceMonitor para recopilar métricas.

La configuración está adaptada a una VM K3s single-node: una réplica de Prometheus, retención de 3 días, compresión WAL, almacenamiento temporal emptyDir y límites de memoria controlados.

La opción Replace=true se utiliza para manejar CRDs grandes y evitar el problema de la annotation last-applied-configuration durante la sincronización.

## Manifiestos de la aplicación

- deployment.yaml: crea el pod de la aplicación principal y utiliza la imagen publicada en GHCR.
- service.yaml: expone internamente la aplicación en el puerto 80 y dirige el tráfico al contenedor en el puerto 8000.
- ingress.yaml: publica la aplicación mediante la ruta /.
- sentiment-service.yaml: crea el microservicio del modelo en el puerto interno 8001, con probes de salud.
- sentiment-service-monitor.yaml: indica a Prometheus que consulte /metrics cada 30 segundos.
- namespace.yaml: crea el namespace devops-course.

## Flujo de despliegue de una nueva versión

1. Se modifica el código en el repositorio de aplicación.
2. GitHub Actions construye y publica una imagen en GHCR.
3. Se actualiza el tag de imagen en manifests/deployment.yaml o manifests/sentiment-service.yaml.
4. Se hace push del cambio a la rama main de este repositorio.
5. Argo CD detecta la diferencia y muestra la Application como OutOfSync.
6. Argo CD sincroniza el nuevo estado.
7. Kubernetes crea un nuevo ReplicaSet y un nuevo pod.

Para mantener trazabilidad se recomienda utilizar tags inmutables, por ejemplo main-<commit-sha>, en lugar de depender solamente de latest.

Para un cambio únicamente de la aplicación principal:

```bash
git add manifests/deployment.yaml
git commit -m "deploy: actualizar imagen de la aplicación"
git push origin main
```

Se debe evitar git add . cuando existan otros archivos modificados que no formen parte del despliegue.

## Validación en el clúster

```bash
kubectl get applications -n argocd
kubectl get pods -n devops-course
kubectl get svc -n devops-course
kubectl get ingress -n devops-course
kubectl get statefulset -n monitoring
kubectl get pods -n monitoring
kubectl get servicemonitor -n devops-course
```

Para observar la aplicación de forma local se pueden utilizar port-forward:

```bash
kubectl port-forward svc/devops-demo-service 8000:80 -n devops-course
kubectl port-forward svc/monitoring-stack-grafana 3000:80 -n monitoring
kubectl port-forward svc/monitoring-stack-prometheus 9090:9090 -n monitoring
```

## Monitoreo

Prometheus recopila métricas de Kubernetes, del nodo, de los componentes del stack y del sentiment-service. Algunas consultas utilizadas son:

- up: disponibilidad de los targets.
- kube_pod_info: información de los pods.
- node_cpu_seconds_total: actividad de CPU del nodo.
- sentiment_predictions_total: cantidad de predicciones por resultado.
- model_inference_seconds: tiempo de inferencia del modelo.

Grafana utiliza Prometheus como datasource para mostrar dashboards y gráficos del clúster y de la aplicación.

## Infraestructura y limitaciones

- K3s funciona como clúster single-node dentro de una VM de OCI.
- La configuración está dimensionada para un entorno académico y de demostración.
- Prometheus utiliza almacenamiento temporal para evitar problemas de PVC en el nodo único.
- Un Load Balancer de OCI puede utilizarse como punto de acceso externo, pero actualmente se configura fuera de estos manifiestos.
- La solución no representa alta disponibilidad porque depende de una sola VM.

## Contexto del proyecto

Este repositorio demuestra la separación entre código de aplicación y configuración de infraestructura, la trazabilidad de cambios mediante Git, la reconciliación automática de Argo CD y la observabilidad del despliegue con Prometheus y Grafana.
