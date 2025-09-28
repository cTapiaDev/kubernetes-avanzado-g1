# Taller Práctico de Kubernetes Avanzados

Este repositorio contiene los manifiestos y la configuración para un ejercicio práctico de Kubernetes, enfocado en el despliegue seguro de una aplicación multi-componente utilizando herramientas y conceptos avanzados.

## 🎯 Objetivo
El objetivo de este proyecto es demostrar un flujo de trabajo realista para asegurar una aplicación en Kubernetes, cubriendo tres pilares fundamentales:
1.  **Despliegue con Helm:** Uso y personalización de charts de Helm para desplegar aplicaciones.
2.  **Seguridad a Nivel de Pod:** Aplicación de `PodSecurityContext` para evitar la ejecución con privilegios de root.
3.  **Seguridad de Red:** Aislamiento de microservicios con `NetworkPolicy` para implementar un modelo de "confianza cero".
4.  **Control de Acceso (RBAC):** Definición de roles y permisos de mínimo privilegio para usuarios y servicios.

## 🛠️ Tecnologías Utilizadas
* **Kubernetes (Kind)**: Para crear un clúster local.
* **Docker**: Como motor de contenedores para Kind.
* **Helm**: Como gestor de paquetes para Kubernetes.
* **PowerShell / Bash**: Para la ejecución de comandos.

---

## 🚀 Cómo Replicar el Entorno

### 1. Prerrequisitos
Asegúrate de tener instalado en tu sistema:
* [Docker Desktop](https://www.docker.com/products/docker-desktop/)
* [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)
* [Helm](https://helm.sh/docs/intro/install/)

### 2. Archivos del Proyecto
Para realizar este taller, necesitarás los siguientes tres archivos en tu carpeta de trabajo. Puedes crearlos copiando el contenido de los siguientes bloques.

<details>
<summary><b> Ver código de los archivos </b></summary>

#### `redis-values.yaml`
Este archivo personaliza el despliegue de Redis con Helm, aplicando configuraciones de seguridad.
```yaml
auth:
  enabled: true
  password: "MiPasswordSegura123"

master:
  podSecurityContext:
    enabled: true
    fsGroup: 1001
    runAsUser: 1001
  containerSecurityContext:
    enabled: true
    runAsUser: 1001
    runAsNonRoot: true
    allowPrivilegeEscalation: false

replica:
  replicaCount: 0

sentinel:
  enabled: false
```

#### `cliente-y-red.yaml`
Este manifiesto despliega un pod cliente autorizado y la `NetworkPolicy` que aísla la base de datos
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-client
  namespace: mi-app-segura
  labels:
    app: client
spec:
  containers:
  - name: net-utils
    image: busybox
    command: ["sleep", "3600"]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-allow-client
  namespace: mi-app-segura
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: redis
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: client
    ports:
    - protocol: TCP
      port: 6379
```

#### rbac.yaml
Este manifiesto define un `ServiceAccout` con permisos de solo lectura sobre los pods.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: mi-app-segura
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: devops-user
  namespace: mi-app-segura
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: mi-app-segura
subjects:
- kind: ServiceAccount
  name: devops-user
  namespace: mi-app-segura
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
</details>

### 3. Pasos de Instalación
Ejecuta estos comandos en orden desde tu terminal

#### A. Crear el Clúster y el Namespace
```bash
# Crea el clúster local con Kind
kind create cluster

# Crea el namespace para la aplicación
kubectl create namespace mi-app-segura
```

#### B. Desplegar Redis con Helm y Configuración de Seguridad
```bash
# Añade el repositorio de charts de Bitnami
helm repo add bitnami https://charts.bitnami.com/bitnami

# Instala Redis usando el archivo de valores personalizados (redis-values.yaml)
helm install mi-redis bitnami/redis -n mi-app-segura -f redis-values.yaml
```

#### C. Desplegar el Cliente y la Política de Red
```bash
# Aplica el manifiesto que crea el pod cliente y la NetworkPolicy
kubectl apply -f cliente-y-red.yaml
```

#### D. Aplicar las Reglas de RBAC
```bash
# Aplica el manifiesto que crea el ServiceAccount, Role y RoleBinding
kubectl apply -f rbac.yaml
```

---

## ✅ Verificación Técnica
Una vez desplegado todo, puedes realizar las siguientes pruebas para validar que la configuración funciona como se espera.

#### 1. Verificar el `PodSecurityContext`
Comprueba que el pod de Redis no se está ejecutando como `root`.
* PowerShell:
```powershell
$REDIS_POD = (kubectl get pods -n mi-app-segura -l app.kubernetes.io/name=redis -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $REDIS_POD -n mi-app-segura | Select-String "runAsUser"
```

* Bash:
```bash
REDIS_POD=$(kubectl get pods -n mi-app-segura -l app.kubernetes.io/name=redis -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $REDIS_POD -n mi-app-segura | grep "runAsUser"
```
Resultado esperado: `runAsUser: 1001`

#### 2. Verificar la `NetworkPolicy`
* Prueba de Conexión Fallida (desde un "atacante"):
```bash
# Este comando debe fallar por timeout
kubectl run atacante --rm -it --image=busybox -n mi-app-segura -- sh -c "telnet -w 2 mi-redis-master.mi-app-segura.svc.cluster.local 6379"
```
* Prueba de Conexión Exitosa (desde el cliente autorizado):
```bash
# Esta conexión debe tener éxito. Para salir, presiona Ctrl+] y luego escribe 'quit'.
kubectl exec -it app-client -n mi-app-segura -- telnet mi-redis-master.mi-app-segura.svc.cluster.local 6379
```

#### 3. Verificar RBAC
* Prueba de Permiso Denegado (delete):
```bash
kubectl auth can-i delete pod -n mi-app-segura --as=system:serviceaccount:mi-app-segura:devops-user
```
Resultado esperado: `no`
* Prueba de Permiso Permitido (get):
```bash
kubectl auth can-i get pod -n mi-app-segura --as=system:serviceaccount:mi-app-segura:devops-user
```
Resultado esperado: `yes`

---

## 🧹 Limpieza del Entorno
Para detener y eliminar todos los recursos creados, ejecuta el siguiente comando:
```bash
kind delete cluster
```
