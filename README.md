# Práctica S08 | AP6 — Exposición de Servicios y Configuración Segura en Minikube

**Curso:** Orquestación y Automatización de Software  
**Sesión:** S08 | AP6  
**Alumno:** Pierre Conca  
**Entorno:** Minikube (Kubernetes Local) + kubectl

---

## 📋 Descripción del Proyecto

Despliegue de una arquitectura de dos capas (Backend Spring Boot WebFlux + MongoDB) en Kubernetes local (Minikube), implementando:

- ✅ **Desacoplamiento de configuración:** Variables no sensibles en ConfigMaps
- ✅ **Seguridad de credenciales:** Secrets codificados en Base64
- ✅ **Arquitectura interna:** MongoDB con ClusterIP (solo accesible por el backend)
- ✅ **Exposición externa:** Backend con NodePort (accesible desde el navegador)

---

## 🏗️ Arquitectura Desplegada

```
[Exterior - Navegador]
         │
         │  NodePort :30081
         ▼
┌─────────────────────────────────────────────────────┐
│  Namespace: practica-s08                            │
│                                                     │
│  ┌──────────────────────┐    ClusterIP              │
│  │  backend-deployment  │ ──────────────► mongo     │
│  │  (Spring Boot :8081) │               service     │
│  └──────────────────────┘               :27017      │
│         ▲ envFrom                          │        │
│   app-configmap + db-secret        mongo-deployment │
│   (Base64)                         (mongo:6.0)      │
└─────────────────────────────────────────────────────┘
```

### Componentes

| Recurso | Tipo | Propósito |
|---------|------|-----------|
| `app-configmap` | ConfigMap | Variables públicas: DB_HOST, DB_NAME, APP_PORT |
| `db-secret` | Secret (Base64) | Credenciales: DB_USER, DB_PASS, MONGODB_URI |
| `mongo-deployment` | Deployment | MongoDB 6.0 con 1 réplica |
| `mongo-service` | ClusterIP | Servicio interno MongoDB (puerto 27017) |
| `backend-deployment` | Deployment | Spring Boot WebFlux (imagen: `concaflores/springboot-webflux-mongodb:latest`) |
| `backend-service` | NodePort | Exposición externa (puerto 30081) |

---

## 📂 Estructura del Repositorio

```
practica-s08-pierreconca/
├── k8s/
│   ├── 01-app-configmap.yaml   ← Variables no sensibles
│   ├── 02-db-secret.yaml       ← Credenciales en Base64
│   ├── 03-mongo-stack.yaml     ← MongoDB Deployment + ClusterIP
│   └── 04-backend-stack.yaml   ← Backend Deployment + NodePort
├── src/
│   └── .gitkeep
└── README.md                   ← Este archivo
```

---

## 🚀 Despliegue en Minikube

### 1. Iniciar Minikube

```bash
minikube start
```

### 2. Crear el Namespace

```bash
kubectl create namespace practica-s08
```

### 3. Aplicar los Manifiestos en Orden

```bash
kubectl apply -f k8s/01-app-configmap.yaml
kubectl apply -f k8s/02-db-secret.yaml
kubectl apply -f k8s/03-mongo-stack.yaml
kubectl apply -f k8s/04-backend-stack.yaml
```

### 4. Verificar el Despliegue

```bash
# Ver todos los recursos del namespace
kubectl get pods,svc,configmaps,secrets -n practica-s08
```

**Salida esperada:**
```
NAME                                     READY   STATUS    RESTARTS   AGE
pod/backend-deployment-xxx               1/1     Running   0          2m
pod/mongo-deployment-xxx                 1/1     Running   0          2m

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/backend-service   NodePort    10.x.x.x        <none>        8081:30081/TCP   2m
service/mongo-service     ClusterIP   10.x.x.x        <none>        27017/TCP        2m

NAME                         DATA   AGE
configmap/app-configmap      8      2m

NAME                  TYPE     DATA   AGE
secret/db-secret      Opaque   7      2m
```

---

## ✅ Validaciones Requeridas

### 1. Estado del Clúster

```bash
kubectl get pods,svc,configmaps,secrets -n practica-s08
```

Todos los pods deben estar en estado `Running`.

### 2. Inyección de Variables (Verificación de Seguridad)

```bash
kubectl exec -it deploy/backend-deployment -n practica-s08 -- env | grep -E "DB_|MONGO|SERVER_PORT"
```

Debes ver las variables del ConfigMap y del Secret inyectadas:
```
DB_HOST=mongo-service
DB_NAME=db_avsa
APP_PORT=8081
SERVER_PORT=8081
MONGODB_URI=mongodb://mongoadmin:mongosecret123@mongo-service:27017/db_avsa?authSource=admin
DB_USER=mongoadmin
DB_PASS=mongosecret123
```

### 3. Conexión y Exposición Externa

```bash
# Obtener la URL de acceso
minikube service backend-service -n practica-s08 --url
```

Abre en el navegador:

- **Swagger UI:** `http://<minikube-ip>:30081/swagger-ui.html`
- **Health Check:** `http://<minikube-ip>:30081/actuator/health`
- **Endpoints:**
  - `http://<minikube-ip>:30081/v1/api/agrochemicals`
  - `http://<minikube-ip>:30081/api/productos`
  - `http://<minikube-ip>:30081/v1/api/customer`

---

## 🔐 Seguridad — ConfigMaps vs Secrets

### ¿Por qué separar ConfigMaps y Secrets?

| Recurso | Contenido | Justificación |
|---------|-----------|---------------|
| **ConfigMap** | DB_HOST, DB_NAME, APP_PORT | Variables públicas que no comprometen seguridad si se exponen. Facilitan cambios de configuración sin rebuilds. |
| **Secret** | DB_USER, DB_PASS, MONGODB_URI | Credenciales sensibles codificadas en Base64. Aunque Base64 no es cifrado, Kubernetes las trata como datos sensibles con permisos restrictivos de acceso. |

**Principio de menor privilegio:**  
Un desarrollador puede leer ConfigMaps para debugging, pero no Secrets sin permisos explícitos.

**Nota importante:**  
Base64 es *encoding*, no cifrado. En producción real se debe usar **Sealed Secrets**, **Vault** u otro sistema de gestión de secretos.

---

## 📊 Comandos Útiles

```bash
# Ver logs del backend
kubectl logs -l app=avsa-backend -n practica-s08 -f

# Ver logs de MongoDB
kubectl logs -l app=mongo -n practica-s08

# Describir un pod (errores detallados)
kubectl describe pod -l app=avsa-backend -n practica-s08

# Reiniciar el deployment después de cambiar un ConfigMap/Secret
kubectl rollout restart deployment backend-deployment -n practica-s08

# Eliminar todo
kubectl delete namespace practica-s08
```

---

## 🛠️ Tecnologías Utilizadas

- **Kubernetes:** Orquestación de contenedores
- **Minikube:** Clúster Kubernetes local
- **Docker:** Runtime de contenedores
- **Spring Boot 3.4.5:** Framework del backend
- **Spring WebFlux:** Programación reactiva
- **MongoDB 6.0:** Base de datos NoSQL
- **Spring Boot Actuator:** Health checks y métricas

---

## 📦 Imagen Docker

La imagen del backend se encuentra publicada en Docker Hub:

```
concaflores/springboot-webflux-mongodb:latest
```

Generada automáticamente mediante CI/CD con GitHub Actions desde el repositorio principal del proyecto.

---

## 📝 Notas Adicionales

- El backend usa **programación reactiva** (Spring WebFlux) con MongoDB reactivo.
- Los health checks (`/actuator/health/liveness` y `/actuator/health/readiness`) están configurados para que Kubernetes reinicie automáticamente el pod si falla.
- R2DBC (SQL Server) está configurado con un driver PostgreSQL dummy para evitar que Spring Boot crashee al iniciar sin SQL Server disponible en Minikube.

---

## 👨‍💻 Autor

**Pierre Conca**  
Curso: Orquestación y Automatización de Software  
Universidad: Valle Grande  
Fecha: Noviembre 2026

---

## 📄 Licencia

Este proyecto es material académico para la práctica S08 | AP6.
