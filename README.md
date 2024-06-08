# Implementación de Grafana con Autoescalado y Persistencia en Kubernetes

Este README proporciona instrucciones detalladas para implementar Grafana en un clúster de Kubernetes con persistencia de datos y autoescalado habilitado. Cada paso incluye los comandos y explicaciones necesarias para configurar y verificar que el despliegue funcione correctamente.

## Archivos de Configuración Necesarios

1. `gfn-pv.yaml`
2. `gfn-pvc.yaml`
3. `gfn-hpa.yaml`
4. `load-generator.yaml` (para prueba de autoescalado)

Para gestionar datos persistentes y configurar plantillas de escalado automático (autoscaling) en Kubernetes, necesitas seleccionar los archivos que manejan el almacenamiento persistente y la configuración del autoscaling. Aquí tienes los archivos relevantes:

1. **Datos persistentes:**
   - `gfn-pv.yaml`: PersistentVolume (PV) define un volumen de almacenamiento que puede ser utilizado por los contenedores.
   - `gfn-pvc.yaml`: PersistentVolumeClaim (PVC) es una solicitud de almacenamiento persistente que se vincula a un PV.

2. **Autoscaling templates:**
   - `gfn-hpa.yaml`: HorizontalPodAutoscaler (HPA) define la configuración para el escalado automático de los pods basándose en las métricas especificadas.


Los otros archivos (`gfn-configmap.yaml`, `gfn-deployment.yaml`, `gfn-service.yaml`, `load-generator.yaml`) no son directamente necesarios para el almacenamiento persistente ni para el autoscaling, aunque pueden ser importantes para otras configuraciones y despliegues de tu aplicación en Kubernetes.

**En caso de ya tener todos los archivos de configuracion, estos son los pasos para la implementación:**
[PASOS](#pasos-detallados-para-la-implementación)

## Descripción de Cada Archivo de Configuración

### 1. `gfn-pv.yaml`

**Función:** Define un PersistentVolume (PV) que proporciona almacenamiento persistente para Grafana.

**Contenido:**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: grafana-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/grafana"
```

**Explicación:**
- `apiVersion: v1`: Especifica la versión de la API.
- `kind: PersistentVolume`: Define el tipo de recurso.
- `metadata`: Incluye el nombre del PV.
- `spec`: Define las especificaciones del volumen, como la capacidad (`1Gi`), el modo de acceso (`ReadWriteOnce`), y la ruta del host (`/mnt/data/grafana`).

### 2. `gfn-pvc.yaml`

**Función:** Define un PersistentVolumeClaim (PVC) que solicita almacenamiento del PV definido anteriormente.

**Contenido:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

**Explicación:**
- `apiVersion: v1`: Especifica la versión de la API.
- `kind: PersistentVolumeClaim`: Define el tipo de recurso.
- `metadata`: Incluye el nombre del PVC.
- `spec`: Define las especificaciones del PVC, como el modo de acceso (`ReadWriteOnce`) y la cantidad de almacenamiento solicitada (`1Gi`).


### 3. `gfn-hpa.yaml`

**Función:** Define el autoescalador horizontal que ajusta el número de réplicas de Grafana según la utilización de CPU.

**Contenido:**

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: grafana-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: grafana
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 80
```

**Explicación:**
- `apiVersion: autoscaling/v1`: Especifica la versión de la API.
- `kind: HorizontalPodAutoscaler`: Define el tipo de recurso.
- `metadata`: Incluye el nombre del HPA.
- `spec`: Define las especificaciones del HPA.
  - `scaleTargetRef`: Referencia al Deployment de Grafana.
    - `apiVersion: apps/v1`: Versión de la API del Deployment.
    - `kind: Deployment`: Tipo de recurso.
    - `name: grafana`: Nombre del Deployment.
  - `minReplicas`: Número mínimo de réplicas.
  - `maxReplicas`: Número máximo de réplicas.
  - `targetCPUUtilizationPercentage`: Utilización objetivo de CPU para el autoescalado.

### 4. `load-generator.yaml`

**Función:** Pod utilizado para generar carga de trabajo en el despliegue de Grafana y probar el autoescalado.

**Contenido:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: load-generator
spec:
  containers:
  - name: load-generator
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do wget -q -O- http://grafana:3000; done"]
```

**Explicación:**
- `apiVersion: v1`: Especifica la versión de la API.
- `kind: Pod`: Define el tipo de recurso.
- `metadata`: Incluye el nombre del Pod.
- `spec`: Define las especificaciones del Pod.
  - `containers`: Lista de contenedores en el Pod.
    - `name`: Nombre del contenedor.
    - `image`: Imagen del contenedor (`busybox`).
    - `command`: Comando a ejecutar.
    - `args`: Argumentos del comando, en este caso un bucle infinito que realiza solicitudes HTTP a Grafana.

## Pasos Detallados para la Implementación

### Paso 1: Crear PV y PVC

1. Crear el PersistentVolume:
    ```sh
    kubectl apply -f gfn-pv.yaml
    ```

2. Crear el PersistentVolumeClaim:


    ```sh
    kubectl apply -f gfn-pvc.yaml
    ```

### Paso 2: Crear ConfigMap para Grafana

1. Aplicar el ConfigMap:
    ```sh
    kubectl apply -f gfn-configmap.yaml
    ```

### Paso 3: Desplegar Grafana

1. Crear el despliegue:
    ```sh
    kubectl apply -f gfn-deployment.yaml
    ```

2. Crear el servicio:
    ```sh
    kubectl apply -f gfn-service.yaml
    ```

### Paso 4: Crear Autoescalado

1. Aplicar el HPA:
    ```sh
    kubectl apply -f gfn-hpa.yaml
    ```

### Paso 5: Comprobar que Escala Correctamente

#### 5.1 Generar Carga de Trabajo

1. Crear el pod de carga:
    ```sh
    kubectl apply -f load-generator.yaml
    ```

    **Explicación:**
    - Este pod generará tráfico continuo a Grafana, aumentando la carga de CPU.

#### 5.2 Monitorear el Autoescalado

1. Verificar el estado del HPA:
    ```sh
    kubectl get hpa
    ```

    **Ejemplo de salida:**
    ```
    NAME         REFERENCE        TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
    grafana-hpa  Deployment/grafana  85%/80%   1         3         2          3m
    ```

    **Explicación:**
    - `REFERENCE`: Muestra el recurso objetivo del HPA.
    - `TARGETS`: Muestra la utilización actual de CPU respecto al objetivo.
    - `MINPODS` y `MAXPODS`: Muestran el número mínimo y máximo de réplicas configuradas.
    - `REPLICAS`: Número actual de réplicas.
    - `AGE`: Tiempo desde que se creó el HPA.

2. Monitorear los pods de Grafana:
    ```sh
    kubectl get pods -l app=grafana
    ```

    **Ejemplo de salida:**
    ```
    NAME                        READY   STATUS    RESTARTS   AGE
    grafana-5b5f6b7d7d-8xpx2    1/1     Running   0          2m
    grafana-5b5f6b7d7d-ztxnc    1/1     Running   0          1m
    ```

    **Explicación:**
    - `NAME`: Nombre de los pods.
    - `READY`: Número de contenedores listos en el pod.
    - `STATUS`: Estado del pod (Running, Pending, etc.).
    - `RESTARTS`: Número de reinicios del pod.
    - `AGE`: Tiempo desde que se creó el pod.

3. Revisar los eventos del HPA para ver detalles del autoescalado:
    ```sh
    kubectl describe hpa grafana-hpa
    ```

    **Explicación:**
    - Este comando proporciona una descripción detallada del HPA, incluyendo eventos recientes que pueden indicar por qué se escaló el despliegue.

#### 5.3 Finalizar la Prueba

1. Eliminar el pod de carga:
    ```sh
    kubectl delete pod load-generator
    ```

    **Explicación:**
    - Este paso elimina el pod de carga, permitiendo que el HPA reduzca el número de réplicas de Grafana si la carga disminuye.

## Conclusión

Al seguir estos pasos, habrás desplegado Grafana en un clúster de Kubernetes con almacenamiento persistente y autoescalado configurado. Esto asegura que Grafana puede manejar aumentos en la carga de trabajo escalando horizontalmente mientras mantiene la configuración y los datos persistentes entre reinicios. La comprobación detallada del autoescalado garantiza que el sistema responde correctamente a cambios en la carga de trabajo.
