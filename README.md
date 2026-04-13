# 🐳 Laboratorio de Orquestación — Bootcamp DevOps Lemoncode

Este repositorio contiene los entregables de los ejercicios de orquestación de contenedores del Bootcamp DevOps de Lemoncode.

## 📋 Pre-requisitos

- **Docker** (v24+)
- **Node.js 22.\*** o superior
- **kubectl**
- **Minikube**

## 📁 Estructura del repositorio

```
.
├── 00-monolith-in-mem/       # Ejercicio 1: Monolito en memoria (Docker + Kubernetes)
│   ├── Dockerfile
│   ├── k8s/
│   │   ├── configmap.yaml
│   │   ├── todo-app-deployment.yaml
│   │   └── todo-app-service.yaml
│   └── README.md
├── 01-monolith/              # Ejercicio 2: Monolito con BBDD (Kubernetes)
│   ├── k8s/
│   │   ├── configmap.yaml
│   │   ├── storage-class.yaml
│   │   ├── persistent-volume.yaml
│   │   ├── persistent-volume-claim.yaml
│   │   ├── postgres-statefulset.yaml
│   │   ├── postgres-service.yaml
│   │   ├── todo-app-deployment.yaml
│   │   └── todo-app-service.yaml
│   └── README.md
└── 02-distributed/           # Ejercicio 3: App distribuida con Ingress (Kubernetes)
    ├── k8s/
    │   ├── todo-front-deployment.yaml
    │   ├── todo-front-service.yaml
    │   ├── todo-api-configmap.yaml
    │   ├── todo-api-deployment.yaml
    │   ├── todo-api-service.yaml
    │   └── ingress.yaml
    └── README.md
```

## 🧭 Recursos por ejercicio

Los tres ejercicios despliegan en Kubernetes (Minikube). El Ejercicio 1 incluye además un `Dockerfile` para construir la imagen, aunque puede usarse la imagen ya publicada por Lemoncode.

| Ejercicio | Recursos K8s principales | Acceso externo |
|-----------|--------------------------|----------------|
| 00 — Monolito en memoria | Deployment + ConfigMap | LoadBalancer (`minikube tunnel`) |
| 01 — Monolito con BBDD | Deployment + StatefulSet + PV/PVC + ConfigMap | LoadBalancer (`minikube tunnel`) |
| 02 — App Distribuida | 2× Deployment + ConfigMap | Ingress NGINX |

## 🚀 Inicio rápido

```bash
minikube start

# Ejercicio 1
kubectl apply -f 00-monolith-in-mem/k8s/

# Ejercicio 2
kubectl apply -f 01-monolith/k8s/

# Ejercicio 3 (requiere addon ingress)
minikube addons enable ingress
kubectl apply -f 02-distributed/k8s/
```

Consulta el README de cada ejercicio para instrucciones detalladas:

- [Ejercicio 1 →](./00-monolith-in-mem/README.md)
- [Ejercicio 2 →](./01-monolith/README.md)
- [Ejercicio 3 →](./02-distributed/README.md)
