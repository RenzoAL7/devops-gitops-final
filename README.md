# DevOps GitOps Final

Proyecto final del curso de DevOps.

Implementación práctica de GitOps usando:

- K3s
- Argo CD
- Kubernetes manifests
- Prometheus
- Grafana

## Objetivo

Demostrar cómo una aplicación puede desplegarse de forma declarativa usando Git como fuente única de verdad y Argo CD como agente de reconciliación continua.

## Flujo GitOps

```txt
GitHub Repo
   ↓
Argo CD detecta cambios
   ↓
Argo CD sincroniza el cluster K3s
   ↓
La aplicación queda desplegada en Kubernetes