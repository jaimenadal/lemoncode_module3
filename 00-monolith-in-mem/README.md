# Ejercicio 1 — Monolito en memoria (Kubernetes)

## Descripción

Aplicación TODO con UI React y API Express/Node.js. La persistencia es **en memoria**: los datos se pierden al reiniciar el pod. No requiere base de datos.

Se despliega en Kubernetes con un `Deployment` expuesto al exterior mediante un `LoadBalancer service`.

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
<img width="1716" height="647" alt="Captura de pantalla de 2026-04-07 12-32-35" src="https://github.com/user-attachments/assets/2d9402ce-e8f9-4ad0-aa98-2c219430f65f" />


Si usas tu propia imagen, actualiza el campo `image:` en `k8s/todo-app-deployment.yaml`.

## 🚀 Despliegue

### Paso 1 — crear el deployment

```bash
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/todo-app-deployment.yaml

# Verificar que el pod está Running
kubectl get pods -l app=todo-app-mem -w
```
<img width="1312" height="246" alt="Captura de pantalla de 2026-04-07 12-33-29" src="https://github.com/user-attachments/assets/e0394ce1-88ca-4430-8362-0c1751b0f309" />

### Paso 2 — exponer con LoadBalancer

```bash
kubectl apply -f k8s/todo-app-service.yaml
```
<img width="1346" height="63" alt="Captura de pantalla de 2026-04-07 12-33-51" src="https://github.com/user-attachments/assets/1c1b0d03-e79c-451b-affa-4e6a2b8c4bff" />

#### Obtener URL del servicio todo-app-service `minikube service` 

```bash
minikube service todo-app-service --url
```
<img width="1368" height="54" alt="Captura de pantalla de 2026-04-07 12-34-19" src="https://github.com/user-attachments/assets/46050b30-1169-4ae7-bce6-f055a97a1f87" />

> ⚠️ En Linux con docker driver, `minikube tunnel` **no crea rutas de red reales** hacia localhost.
> Esta es la única opción fiable en Linux.

#### Alternativa al tunnel: `kubectl port-forward` (acceso por localhost:3000)

```bash
kubectl port-forward service/todo-app-service 3000:3000
```
<img width="1437" height="155" alt="Captura de pantalla de 2026-04-07 12-36-53" src="https://github.com/user-attachments/assets/4b475d00-0980-4f29-8cbe-17745fca8143" />

Acceder a → **http://localhost:3000**
Mantener el proceso abierto mientras se usa la app.

### Despliegue en un solo comando

```bash
kubectl apply -f k8s/
```

## ℹ️ Variables de entorno (ConfigMap)

| Variable   | Valor        | Descripción                              |
|------------|--------------|------------------------------------------|
| `NODE_ENV` | `production` | Entorno de ejecución                     |
| `PORT`     | `3000`       | Puerto de escucha del contenedor         |

## 🔍 Verificación — outputs esperados

### 1. Verificar todos los recursos

```bash
kubectl get all -l app=todo-app-mem
```

Salida esperada:
<img width="1124" height="368" alt="Captura de pantalla de 2026-04-07 12-35-55" src="https://github.com/user-attachments/assets/74ba3d09-a850-4267-87b8-8750a82b2c0c" />


### 2. Verificar que la API responde

```bash
curl -s http://192.168.49.2:31432/api/
# Salida esperada: []
```
<img width="1702" height="79" alt="Captura de pantalla de 2026-04-07 12-41-35" src="https://github.com/user-attachments/assets/deb2777b-450b-44b0-95b3-11214cadcb5a" />

### 3. Verificar que la UI es accesible

<img width="1321" height="707" alt="Captura de pantalla de 2026-04-07 12-37-08" src="https://github.com/user-attachments/assets/d87b98ef-a4cd-4d52-ae44-52a53c4ce537" />


### 4. Confirmar que los datos NO persisten (comportamiento esperado)

```bash
# Crear un TODO
<img width="1127" height="493" alt="Captura de pantalla de 2026-04-07 12-38-38" src="https://github.com/user-attachments/assets/16166a52-be7f-46d5-bc83-d183a9fa519e" />

# Eliminar el pod 
kubectl delete pod -l app=todo-app-mem

# Esperar a que vuelva Running
kubectl get pods -l app=todo-app-mem -w
<img width="1222" height="96" alt="image" src="https://github.com/user-attachments/assets/d22f39ea-745b-409d-a21d-ff12f67c9f3c" />


# Comprobar que el TODO ha desaparecido (memoria volátil)

<img width="1321" height="707" alt="Captura de pantalla de 2026-04-07 12-37-08" src="https://github.com/user-attachments/assets/fe33f930-d648-4729-973b-fd949d5152c4" />


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
