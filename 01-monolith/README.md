# Ejercicio 2 — Monolito con Base de Datos (Kubernetes)

## Descripción

Misma aplicación TODO del ejercicio anterior, pero ahora la persistencia se delega en **PostgreSQL**. Se despliega como clúster Kubernetes con los siguientes recursos:

```
                   ┌─────────────────────────┐
  Internet         │       Minikube           │
  ─────────►       │                          │
 :3000             │  ┌──────────────────┐    │
  LoadBalancer ────┼─►│   todo-app       │    │
  Service          │  │   (Deployment)   │    │
                   │  └────────┬─────────┘    │
                   │           │ ClusterIP    │
                   │           ▼              │
                   │  ┌──────────────────┐    │
                   │  │   postgres       │    │
                   │  │   (StatefulSet)  │    │
                   │  └────────┬─────────┘    │
                   │           │              │
                   │  ┌────────▼─────────┐    │
                   │  │  PersistentVolume│    │
                   │  └──────────────────┘    │
                   └─────────────────────────┘
```

## Pre-requisitos

- Docker
- kubectl
- Minikube

```bash
minikube start
```

## ⚠️ Importante — despliegue limpio

Si ya has ejecutado este ejercicio anteriormente, el `PersistentVolume` puede contener datos residuales. En ese caso postgres omite la inicialización (`Skipping initialization`) y la base de datos queda vacía, lo que provoca el error `42P01: table does not exist` en la app.

Antes de desplegar, asegúrate de partir de un estado limpio:

```bash
# Eliminar recursos previos si existen
kubectl delete -f 00-monolith-in-mem/k8s/
kubectl delete pv postgres-pv

# Limpiar datos del hostPath en Minikube
minikube ssh "sudo rm -rf /mnt/data/postgres"
```

---

## 🚀 Despliegue paso a paso

### Paso 1 — Capa de persistencia (PostgreSQL)

```bash
# Orden importante: StorageClass → PV → PVC → Service → StatefulSet
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/storage-class.yaml
kubectl apply -f k8s/persistent-volume.yaml
kubectl apply -f k8s/persistent-volume-claim.yaml
kubectl apply -f k8s/postgres-service.yaml
kubectl apply -f k8s/postgres-statefulset.yaml

# Verificar que el pod de postgres está Running
kubectl get pods -l component=database -w
```
<img width="1227" height="355" alt="image" src="https://github.com/user-attachments/assets/1ae552e0-db2a-4436-8367-f6c591eebdab" />

### Paso 1b — Seed de la base de datos (Job)

La imagen `lemoncodersbc/lc-todo-monolith-psql:v5-2024` ejecuta el script de inicialización automáticamente solo cuando el directorio de datos está vacío. En Minikube con Docker driver, el PVC puede contener datos residuales del provisioner que hacen que postgres salte la inicialización (`Skipping initialization`).

Para garantizar que la base de datos se inicializa correctamente en cualquier entorno, se usa un **Job de Kubernetes** que ejecuta el script SQL contra postgres usando un `ConfigMap` como volumen — tal como sugiere el propio enunciado del ejercicio:

```bash
kubectl apply -f k8s/todos-db-configmap.yaml
kubectl apply -f k8s/todos-db-seed-job.yaml

# Seguir el progreso del Job
kubectl get jobs -w

# Verificar que completó correctamente
kubectl logs -l job-name=todos-db-seed
```

Salida esperada del Job:
```
Esperando a que postgres esté listo...
Ejecutando script de seed...
SET
...
ALTER TABLE
Seed completado
```
<img width="1158" height="491" alt="image" src="https://github.com/user-attachments/assets/2e6744c1-d0b4-4e64-b16a-0735194bdbe5" />

Verificar que las tablas se crearon correctamente:

```bash
kubectl exec -it postgres-0 -- psql -U postgres -d todos_db -c "\dt"
```
<img width="1396" height="195" alt="image" src="https://github.com/user-attachments/assets/cbe9df59-e4ab-4aac-8363-3db7052906df" />

### Paso 2 — aplicación todo-app

```bash
kubectl apply -f k8s/todo-app-deployment.yaml
kubectl apply -f k8s/todo-app-service.yaml

# Verificar
kubectl get pods -l component=app -w
```
<img width="1190" height="206" alt="image" src="https://github.com/user-attachments/assets/68b5cb0e-cf7d-46b9-8c96-c582176157e0" />

### Paso 3 — Acceder desde fuera del clúster

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

## 🗂️ Despliegue en un solo comando

```bash
kubectl apply -f k8s/
```

> ⚠️ El Job `todos-db-seed` se ejecuta automáticamente al hacer `kubectl apply -f k8s/` pero necesita que postgres esté Running primero. Si el Job falla, reintenta automáticamente hasta 5 veces (`backoffLimit: 5`).

## 🔄 Verificar estado general

```bash
kubectl get all
kubectl get pv,pvc,sc
```

## 🧹 Limpiar el entorno

```bash
# Eliminar recursos de Kubernetes
kubectl delete -f k8s/

# El PV tiene política Retain — eliminarlo manualmente
kubectl delete pv postgres-pv

# Limpiar los datos del disco en Minikube (necesario para un despliegue limpio futuro)
minikube ssh "sudo rm -rf /mnt/data/postgres"
```

---

## 📂 Nota sobre el código fuente

Este repositorio contiene únicamente los manifiestos de Kubernetes y la documentación. Las imágenes Docker ya están publicadas en Docker Hub por Lemoncode:

- `lemoncodersbc/lc-todo-monolith-psql:v5-2024` — PostgreSQL con script de inicialización incluido
- `lemoncodersbc/lc-todo-monolith-db:v5-2024` — Aplicación todo-app con soporte de BBDD

El código fuente original está en:
> https://github.com/Lemoncode/bootcamp-devops-lemoncode/tree/master/02-orquestacion/exercises/01-monolith

---

## 🔍 Verificación — outputs esperados

### 1. Verificar todos los recursos creados

```bash
kubectl get all
```

Salida esperada:
```
NAME                            READY   STATUS    RESTARTS   AGE
pod/postgres-0                  1/1     Running   0          2m
pod/todo-app-6d8f9b7c4d-xk2pj   1/1     Running   0          1m

NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/kubernetes         ClusterIP      10.96.0.1       <none>        443/TCP          10m
service/postgres-service   ClusterIP      10.96.45.123    <none>        5432/TCP         2m
service/todo-app-service   LoadBalancer   10.96.78.200    127.0.0.1     3000:31234/TCP   1m

NAME                       READY   AGE
statefulset.apps/postgres  1/1     2m

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/todo-app   1/1     1            1           1m
```

### 2. Verificar PersistentVolume y PersistentVolumeClaim

```bash
kubectl get pv,pvc
```

Salida esperada:
```
NAME                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS          AGE
persistentvolume/postgres-pv  1Gi        RWO            Retain           Bound    default/postgres-pvc   todo-storage-class    2m

NAME                                 STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS          AGE
persistentvolumeclaim/postgres-pvc   Bound    postgres-pv   1Gi        RWO            todo-storage-class    2m
```

> El campo `STATUS` debe ser `Bound` en ambos. Si aparece `Pending`, revisar que el StorageClass y el PV están correctamente creados.

### 3. Verificar que la API responde

```bash
curl -s http://$(minikube service todo-app-service --url | tail -1)/api/
```

Salida esperada:
```json
[]
```

### 4. Verificar que la UI es accesible

Abrir → **http://$(minikube service todo-app-service --url | tail -1)**

### 5. Verificar que los datos persisten tras reiniciar el pod

```bash
# Crear un TODO
curl -s -X POST http://$(minikube service todo-app-service --url | tail -1)/api/ \
  -H "Content-Type: application/json" \
  -d '{"title": "Test persistencia con BBDD"}'

# Eliminar el pod de la app (Kubernetes lo recreará automáticamente)
kubectl delete pod -l component=app

# Esperar a que vuelva a estar Running
kubectl get pods -l component=app -w

# Comprobar que el TODO sigue existiendo
curl -s http://$(minikube service todo-app-service --url | tail -1)/api/
# Salida esperada: [{"id":1,"title":"Test persistencia con BBDD","completed":false}]
```

### 6. Verificar logs en caso de error

```bash
# Logs de la aplicación
kubectl logs -l component=app --tail=50

# Logs de postgres
kubectl logs -l component=database --tail=50

# Describir un pod para ver eventos
kubectl describe pod -l component=app
```

## 📝 Variables de entorno (ConfigMap)

| Variable | Valor | Descripción |
|----------|-------|-------------|
| `NODE_ENV` | `production` | Entorno de la app |
| `PORT` | `3000` | Puerto de escucha |
| `DB_HOST` | `postgres-service` | Nombre del service ClusterIP |
| `DB_USER` | `postgres` | Usuario de la BBDD |
| `DB_PASSWORD` | `postgres` | Contraseña de la BBDD |
| `DB_PORT` | `5432` | Puerto de Postgres |
| `DB_NAME` | `todos_db` | Nombre de la base de datos |
| `DB_VERSION` | `10.4` | Versión de Postgres |
