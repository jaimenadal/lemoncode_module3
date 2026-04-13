# Ejercicio 3 — Aplicación Distribuida con Ingress (Kubernetes)

## Descripción

Arquitectura de dos servicios independientes orquestados con Kubernetes:

- **todo-front**: SPA React servida por Nginx (puerto 80)
- **todo-api**: API REST Express/Node.js (puerto 3000)

El acceso exterior se gestiona mediante un **Ingress NGINX** que enruta el tráfico según la ruta de la URL:

```
                        ┌────────────────────────────────────┐
  Internet              │            Minikube                 │
  ─────────────►        │                                     │
  todo-app.local        │  ┌──────────────────────────────┐  │
                        │  │         Ingress NGINX         │  │
                        │  │                               │  │
                        │  │  /api/*  ──►  todo-api:3000   │  │
                        │  │  /*      ──►  todo-front:80   │  │
                        │  └──────────────────────────────┘  │
                        │          │              │           │
                        │          ▼              ▼           │
                        │  ┌─────────────┐ ┌───────────────┐ │
                        │  │  todo-api   │ │  todo-front   │ │
                        │  │ (Deployment)│ │  (Deployment) │ │
                        │  │  ClusterIP  │ │   ClusterIP   │ │
                        │  └─────────────┘ └───────────────┘ │
                        └────────────────────────────────────┘
```

## Pre-requisitos

- Docker
- kubectl
- Minikube

## ⚙️ Preparar Minikube

```bash
# Arrancar Minikube
minikube start

# Habilitar el addon de Ingress NGINX (obligatorio)
minikube addons enable ingress

# Verificar que el controlador está corriendo (puede tardar ~1 min)
kubectl get pods -n ingress-nginx
```

## 🌐 Configurar /etc/hosts

El Ingress usa el host `todo-app.local`. Hay que añadirlo al fichero de hosts local:

```bash
# Obtener la IP de Minikube
minikube ip
# Ejemplo de salida: 192.168.49.2

# Añadir al fichero de hosts (requiere sudo)
echo "$(minikube ip) todo-app.local" | sudo tee -a /etc/hosts
```

En **Windows** el fichero está en `C:\Windows\System32\drivers\etc\hosts`.

## 🚀 Despliegue

### Paso 1 — todo-front

```bash
kubectl apply -f k8s/todo-front-deployment.yaml
kubectl apply -f k8s/todo-front-service.yaml

# Verificar
kubectl get pods -l component=frontend
```

### Paso 2 — todo-api

```bash
kubectl apply -f k8s/todo-api-configmap.yaml
kubectl apply -f k8s/todo-api-deployment.yaml
kubectl apply -f k8s/todo-api-service.yaml

# Verificar
kubectl get pods -l component=api
```

### Paso 3 — Ingress

```bash
kubectl apply -f k8s/ingress.yaml

# Verificar que el Ingress tiene ADDRESS asignada
kubectl get ingress todo-ingress
```

### Despliegue en un solo comando

```bash
kubectl apply -f k8s/
```

## ✅ Verificar el acceso

Una vez todos los pods estén `Running` y el Ingress tenga IP:

```bash
# Frontend
curl http://todo-app.local

# API
curl http://todo-app.local/api/
```

O abrir en el navegador → **http://todo-app.local**

## 🔍 Cómo funciona el enrutado del Ingress

| Ruta de entrada | Destino interno | Reescritura |
|-----------------|-----------------|-------------|
| `todo-app.local/api/` | `todo-api-service:3000` | `/todos` |
| `todo-app.local/` | `todo-front-service:80` | `/` |

La anotación `nginx.ingress.kubernetes.io/rewrite-target: /$2` elimina el prefijo `/api` antes de reenviar la petición al servicio de la API, de modo que la API recibe `/todos` y no `/api/`.

## 📝 Variables de entorno (ConfigMap todo-api)

| Variable | Valor | Descripción |
|----------|-------|-------------|
| `NODE_ENV` | `production` | Entorno de ejecución |
| `PORT` | `3000` | Puerto de escucha de la API |

## 🧹 Limpiar el entorno

```bash
kubectl delete -f k8s/

# Eliminar la entrada de /etc/hosts manualmente si ya no se necesita
```

---

## 📂 Nota sobre el código fuente

Este repositorio contiene únicamente los manifiestos de Kubernetes y la documentación. Las imágenes Docker ya están publicadas en Docker Hub por Lemoncode:

- `lemoncodersbc/lc-todo-front:v5-2024` — Frontend React servido por Nginx
- `lemoncodersbc/lc-todo-api:v5-2024` — API REST Express/Node.js

El código fuente original está en:
> https://github.com/Lemoncode/bootcamp-devops-lemoncode/tree/master/02-orquestacion/exercises/02-distributed

---

## 🔍 Verificación — outputs esperados

### 1. Verificar todos los recursos creados

```bash
kubectl get all
```

Salida esperada:
```
NAME                              READY   STATUS    RESTARTS   AGE
pod/todo-api-7d9f6c8b5-nt3qw      1/1     Running   0          1m
pod/todo-front-5c4b8d7a2-pz6kx    1/1     Running   0          1m

NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/kubernetes           ClusterIP   10.96.0.1       <none>        443/TCP    10m
service/todo-api-service     ClusterIP   10.96.112.45    <none>        3000/TCP   1m
service/todo-front-service   ClusterIP   10.96.88.210    <none>        80/TCP     1m

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/todo-api     1/1     1            1           1m
deployment.apps/todo-front   1/1     1            1           1m
```

### 2. Verificar el Ingress

```bash
kubectl get ingress todo-ingress
```

Salida esperada:
```
NAME           CLASS   HOSTS            ADDRESS        PORTS   AGE
todo-ingress   nginx   todo-app.local   192.168.49.2   80      1m
```

> La columna `ADDRESS` debe tener la IP de Minikube. Si aparece vacía, el addon de Ingress no está habilitado o aún está arrancando.

### 3. Verificar el enrutado del Ingress

```bash
# Frontend (debe devolver HTML)
curl -s -o /dev/null -w "%{http_code}" http://todo-app.local
# Salida esperada: 200

# API (debe devolver JSON)
curl -s http://todo-app.local/api/
# Salida esperada: []
```

### 4. Verificar la UI en el navegador

Abrir → **http://todo-app.local**

Deberías ver la interfaz TODO con la lista vacía. Al crear un TODO, la UI llama a `/api/` que el Ingress redirige internamente al servicio `todo-api-service`.

### 5. Verificar el rewrite de rutas del Ingress

```bash
# Esta petición llega al Ingress como /api/
# pero la API la recibe como /todos (el rewrite elimina /api)
curl -sv http://todo-app.local/api/ 2>&1 | grep "< HTTP"
# Salida esperada: < HTTP/1.1 200 OK
```

### 6. Verificar logs en caso de error

```bash
# Logs del frontend
kubectl logs -l component=frontend --tail=30

# Logs de la API
kubectl logs -l component=api --tail=30

# Logs del controlador Ingress
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=30
```

### 7. Diagnóstico si el Ingress no tiene ADDRESS

```bash
# Comprobar que el addon está habilitado
minikube addons list | grep ingress

# Comprobar el pod del controlador
kubectl get pods -n ingress-nginx

# Si el pod no está Running, habilitarlo de nuevo
minikube addons enable ingress
```
