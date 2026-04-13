# Ejercicio 1 — Monolito en Memoria (Kubernetes)

## Descripción

Aplicación TODO con UI React y API Express/Node.js. La persistencia es **en memoria**: los datos se pierden al reiniciar el pod. No requiere base de datos.

Se despliega en Kubernetes con un `Deployment` expuesto al exterior mediante un `LoadBalancer service`:

```
                   ┌─────────────────────────┐
  Internet         │        Minikube          │
  ─────────►       │                          │
  :<nodeport>      │  ┌──────────────────┐    │
  LoadBalancer ────┼─►│    todo-app      │    │
  Service          │  │  (Deployment)    │    │
                   │  │  memoria RAM     │    │
                   │  └──────────────────┘    │
                   └─────────────────────────┘
```

## Árbol de ficheros

```
00-monolith-in-mem/
├── Dockerfile              ← build multi-stage (Node 22 Alpine)
├── k8s/
│   ├── configmap.yaml      ← variables de entorno
│   ├── todo-app-deployment.yaml
│   └── todo-app-service.yaml
└── README.md
```

## Pre-requisitos

- Docker
- kubectl
- Minikube

```bash
minikube start
```

## 🏗️ Construir la imagen (opcional)

> Puedes omitir este paso usando la imagen ya publicada `lemoncodersbc/lc-todo-monolith:v5-2024`.

El código fuente de la app es de Lemoncode:
```bash
git clone https://github.com/Lemoncode/bootcamp-devops-lemoncode.git
cp -r bootcamp-devops-lemoncode/02-orquestacion/exercises/00-monolith-in-mem/todo-app ./

# Build con el Dockerfile de este directorio
docker build -t lc-todo-monolith:local .
```

Si usas tu propia imagen, actualiza el campo `image:` en `k8s/todo-app-deployment.yaml`.

## 🚀 Despliegue

### Paso 1 — Crear el Deployment

```bash
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/todo-app-deployment.yaml

# Verificar que el pod está Running
kubectl get pods -l app=todo-app-mem -w
```

### Paso 2 — Exponer con LoadBalancer

```bash
kubectl apply -f k8s/todo-app-service.yaml
```

#### Opción A — `minikube service` (recomendado, funciona en todos los entornos)

```bash
minikube service todo-app-service --url
```

Devuelve la URL exacta, por ejemplo `http://192.168.49.2:31234`. Úsala en el navegador o con curl.

> ⚠️ En Linux con Docker driver, `minikube tunnel` **no crea rutas de red reales** hacia localhost.
> Esta es la única opción fiable en Linux.

#### Opción B — `kubectl port-forward` (acceso por localhost:3000)

```bash
kubectl port-forward service/todo-app-service 3000:3000
```

Acceder a → **http://localhost:3000**

Útil para desarrollo. Mantener el proceso abierto mientras se usa la app.

#### Opción C — `minikube tunnel` (macOS/Windows únicamente)

```bash
# En una terminal separada (pedirá contraseña sudo)
minikube tunnel
```

Con el tunnel activo, `EXTERNAL-IP` pasa de `<pending>` a `127.0.0.1` y se puede acceder a **http://localhost:3000**.

```bash
kubectl get service todo-app-service
# NAME               TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
# todo-app-service   LoadBalancer   10.96.45.100   127.0.0.1     3000:31XXX/TCP   1m
```

### Despliegue en un solo comando

```bash
kubectl apply -f k8s/
```

## ℹ️ Variables de entorno (ConfigMap)

| Variable   | Valor        | Descripción                              |
|------------|--------------|------------------------------------------|
| `NODE_ENV` | `production` | Entorno de ejecución (no puede ser `test`) |
| `PORT`     | `3000`       | Puerto de escucha del contenedor         |

## 🔍 Verificación — outputs esperados

### 1. Verificar todos los recursos

```bash
kubectl get all -l app=todo-app-mem
```

Salida esperada:
```
NAME                            READY   STATUS    RESTARTS   AGE
pod/todo-app-5c4b8d7a2-pz6kx    1/1     Running   0          1m

NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/todo-app-service   LoadBalancer   10.96.45.100   127.0.0.1     3000:31234/TCP   1m

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/todo-app   1/1     1            1           1m
```

### 2. Verificar que la API responde

```bash
curl -s http://$(minikube service todo-app-service --url | tail -1)/api/
# Salida esperada: []
```

### 3. Verificar que la UI es accesible

Abrir → **http://$(minikube service todo-app-service --url | tail -1)**

### 4. Confirmar que los datos NO persisten (comportamiento esperado)

```bash
# Crear un TODO
curl -s -X POST http://$(minikube service todo-app-service --url | tail -1)/api/ \
  -H "Content-Type: application/json" \
  -d '{"title": "Test persistencia"}'

# Eliminar el pod (Kubernetes lo recreará)
kubectl delete pod -l app=todo-app-mem

# Esperar a que vuelva Running
kubectl get pods -l app=todo-app-mem -w

# Comprobar que el TODO ha desaparecido (memoria volátil)
curl -s http://$(minikube service todo-app-service --url | tail -1)/api/
# Salida esperada: []
```

### 5. Logs en caso de error

```bash
kubectl logs -l app=todo-app-mem --tail=50
kubectl describe pod -l app=todo-app-mem
```

## 🧹 Limpiar el entorno

```bash
kubectl delete -f k8s/
```

## 📂 Nota sobre el código fuente

Este repositorio contiene únicamente los ficheros de configuración. Las imágenes Docker están publicadas por Lemoncode:

- `lemoncodersbc/lc-todo-monolith:v5-2024`

Código fuente original:
> https://github.com/Lemoncode/bootcamp-devops-lemoncode/tree/master/02-orquestacion/exercises/00-monolith-in-mem
