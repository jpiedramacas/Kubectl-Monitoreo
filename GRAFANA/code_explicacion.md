# Configuración de Grafana en Kubernetes

Este documento detalla los archivos de configuración utilizados para desplegar Grafana en un clúster de Kubernetes. Cada archivo de configuración se explica en términos de su propósito y contenido, con una descripción detallada de su código.

## Archivos de Configuración

1. **[gfn-configmap.yaml](#gfn-configmapyaml)**
2. **[gfn-deployment.yaml](#gfn-deploymentyaml)**
3. **[gfn-hpa.yaml](#gfn-hpayaml)**
4. **[gfn-pv.yaml](#gfn-pvyaml)**
5. **[gfn-pvc.yaml](#gfn-pvcyaml)**
6. **[gfn-service.yaml](#gfn-serviceyaml)**
7. **[datasources.yaml](#datasourcesyaml)**

## gfn-configmap.yaml

Este archivo define un ConfigMap para Grafana, que incluye configuraciones como las fuentes de datos y el archivo de configuración `grafana.ini`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-provisioning
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        access: proxy
        url: http://prometheus:9090
  grafana.ini: |
    [auth]
    # Cambia las credenciales por las que quieras utilizar
    admin_user = admin
    admin_password = admin
```

#### Explicación:
- **apiVersion** y **kind**: Define el tipo de recurso Kubernetes, en este caso, un ConfigMap.
- **metadata**: Proporciona el nombre del ConfigMap.
- **data**: Contiene dos configuraciones en formato de archivo: `datasources.yaml` y `grafana.ini`.
  - `datasources.yaml`: Configura la fuente de datos de Grafana para conectarse a Prometheus.
  - `grafana.ini`: Contiene configuraciones adicionales, como las credenciales de administrador.

## gfn-deployment.yaml

Este archivo describe el Deployment de Grafana, especificando la imagen del contenedor, el número de réplicas y otras configuraciones del pod.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000
        volumeMounts:
        - name: grafana-data
          mountPath: /var/lib/grafana
        - name: grafana-provisioning
          mountPath: /etc/grafana/provisioning
      volumes:
      - name: grafana-data
        persistentVolumeClaim:
          claimName: grafana-pvc
      - name: grafana-provisioning
        configMap:
          name: grafana-provisioning
```

#### Explicación:
- **apiVersion** y **kind**: Define el tipo de recurso Kubernetes, en este caso, un Deployment.
- **metadata**: Incluye el nombre y las etiquetas del Deployment.
- **spec**: Define las especificaciones del Deployment.
  - **replicas**: Número de réplicas del pod.
  - **selector** y **template**: Detalles sobre los pods que se crearán.
  - **containers**: Define el contenedor Grafana con la imagen `grafana/grafana:latest` y el puerto `3000`.
  - **volumeMounts**: Monta volúmenes en el contenedor.
  - **volumes**: Define los volúmenes que se utilizarán, incluyendo el PersistentVolumeClaim y el ConfigMap.

## gfn-hpa.yaml

Este archivo configura el Horizontal Pod Autoscaler (HPA) para Grafana, permitiendo que Kubernetes ajuste automáticamente el número de réplicas del pod basado en la carga.

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

#### Explicación:
- **apiVersion** y **kind**: Define el tipo de recurso Kubernetes, en este caso, un HorizontalPodAutoscaler.
- **metadata**: Proporciona el nombre del HPA.
- **spec**: Define las especificaciones del HPA.
  - **scaleTargetRef**: Referencia al Deployment de Grafana.
  - **minReplicas** y **maxReplicas**: Define el rango de réplicas permitidas.
  - **targetCPUUtilizationPercentage**: Especifica el umbral de utilización de CPU para escalar.

## gfn-pv.yaml

Este archivo define un PersistentVolume (PV) para Grafana, proporcionando almacenamiento persistente.

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

#### Explicación:
- **apiVersion** y **kind**: Define el tipo de recurso Kubernetes, en este caso, un PersistentVolume.
- **metadata**: Proporciona el nombre del PV.
- **spec**: Define las especificaciones del PV.
  - **capacity**: Capacidad de almacenamiento, en este caso, 1Gi.
  - **accessModes**: Modo de acceso, `ReadWriteOnce`.
  - **hostPath**: Ruta en el host donde se almacenarán los datos.

## gfn-pvc.yaml

Este archivo describe un PersistentVolumeClaim (PVC) para Grafana, que solicita el almacenamiento definido por el PV.

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

#### Explicación:
- **apiVersion** y **kind**: Define el tipo de recurso Kubernetes, en este caso, un PersistentVolumeClaim.
- **metadata**: Proporciona el nombre del PVC.
- **spec**: Define las especificaciones del PVC.
  - **accessModes**: Modo de acceso, `ReadWriteOnce`.
  - **resources**: Solicitud de recursos, en este caso, 1Gi de almacenamiento.

## gfn-service.yaml

Este archivo configura el Service de Grafana, exponiendo el Deployment a través de una dirección IP dentro del clúster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
spec:
  selector:
    app: grafana
  ports:
  - protocol: TCP
    port: 3000
    targetPort: 3000
  type: LoadBalancer
```

#### Explicación:
- **apiVersion** y **kind**: Define el tipo de recurso Kubernetes, en este caso, un Service.
- **metadata**: Proporciona el nombre del Service.
- **spec**: Define las especificaciones del Service.
  - **selector**: Selecciona los pods con la etiqueta `app: grafana`.
  - **ports**: Define el puerto que se expondrá y el puerto objetivo en el contenedor.
  - **type**: Tipo de servicio, `LoadBalancer`.

## datasources.yaml

Este archivo contiene la configuración para la fuente de datos de Grafana, especificando a Prometheus como la fuente de datos.

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
```

#### Explicación:
- **apiVersion**: Versión de la API de Grafana.
- **datasources**: Lista de fuentes de datos.
  - **name**: Nombre de la fuente de datos, `Prometheus`.
  - **type**: Tipo de fuente de datos, `prometheus`.
  - **access**: Modo de acceso, `proxy`.
  - **url**: URL de la fuente de datos, `http://prometheus:9090`.

## Conclusión

Cada archivo de configuración desempeña un papel crucial en el despliegue y configuración de Grafana en un clúster de Kubernetes. Siguiendo las explicaciones y ejemplos proporcionados, puedes configurar y desplegar Grafana de manera eficiente, asegurando un monitoreo efectivo de tus aplicaciones y servicios en Kubernetes.
