# Ejercicio 3 — Aplicación Distribuida con Ingress (Kubernetes)

## Descripción

Arquitectura de dos servicios independientes orquestados con Kubernetes:

- **todo-front**: SPA React servida por Nginx (puerto 80)
- **todo-api**: API REST Express/Node.js (puerto 3000)

El acceso exterior se gestiona mediante un **Ingress NGINX** que enruta el tráfico según la ruta de la URL.


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
<img width="1441" height="317" alt="image" src="https://github.com/user-attachments/assets/4f8266c7-c351-4484-bdf8-5eb5b659c820" />


En **Windows** el fichero está en `C:\Windows\System32\drivers\etc\hosts`.

## 🚀 Despliegue

### Paso 1 — todo-front

```bash
kubectl apply -f k8s/todo-front-deployment.yaml
kubectl apply -f k8s/todo-front-service.yaml

# Verificar
kubectl get pods -l component=frontend
```
<img width="1174" height="155" alt="image" src="https://github.com/user-attachments/assets/bcb70e94-0aa5-467f-bd3c-05f39bfe876a" />


### Paso 2 — todo-api

```bash
kubectl apply -f k8s/todo-api-configmap.yaml
kubectl apply -f k8s/todo-api-deployment.yaml
kubectl apply -f k8s/todo-api-service.yaml

# Verificar
kubectl get pods -l component=api
```
<img width="1264" height="269" alt="image" src="https://github.com/user-attachments/assets/01024801-9ce2-49f0-bfd0-b221a5aaf222" />

### Paso 3 — ingress

```bash
kubectl apply -f k8s/ingress.yaml

# Verificar que el Ingress tiene ADDRESS asignada
kubectl get ingress todo-ingress
```
<img width="1082" height="115" alt="image" src="https://github.com/user-attachments/assets/62763b14-fad1-428c-9cb7-99d50065fe15" />


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
<img width="1911" height="95" alt="image" src="https://github.com/user-attachments/assets/7f60f48f-fe48-4873-96da-51476e44e162" />

O abrir en el navegador → **http://todo-app.local**

<img width="1249" height="390" alt="image" src="https://github.com/user-attachments/assets/8949fda9-be46-47e5-917f-893aedf3c1b5" />


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
<img width="1035" height="382" alt="image" src="https://github.com/user-attachments/assets/65efb46e-3b30-418e-91e0-6316b8e27ae5" />

### 2. Verificar el Ingress

```bash
kubectl get ingress todo-ingress
```

Salida esperada:
<img width="1054" height="73" alt="image" src="https://github.com/user-attachments/assets/8bf538f4-e242-4141-9aa0-8627a9a38dc8" />


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
<img width="1433" height="87" alt="image" src="https://github.com/user-attachments/assets/018341ee-3038-4c42-9853-8385a3f8986c" />

### 4. Verificar la UI en el navegador

Abrir → **http://todo-app.local**

Deberías ver la interfaz TODO con la lista vacía. Al crear un TODO, la UI llama a `/api/` que el Ingress redirige internamente al servicio `todo-api-service`.
Creo un elemento llamado Kubernetes
<img width="1383" height="609" alt="image" src="https://github.com/user-attachments/assets/3bdaaa4e-50df-4194-9054-91a3706e8b2f" />

### 5. Verificar el rewrite de rutas del Ingress

```bash
# Esta petición llega al Ingress como /api/
# pero la API la recibe como /todos (el rewrite elimina /api)
curl -sv http://todo-app.local/api/ 2>&1 | grep "< HTTP"
# Salida esperada: < HTTP/1.1 200 OK
```
<img width="1369" height="97" alt="image" src="https://github.com/user-attachments/assets/6f603a0e-9ce2-4438-83f2-3f5534ae4b06" />

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
