# Configuración de Prometheus en Kubernetes

Este documento detalla los archivos de configuración utilizados para desplegar Prometheus en un clúster de Kubernetes. Cada archivo de configuración se explica en términos de su propósito y contenido, con una descripción detallada de su código.

## Archivos de Configuración

1. **[prometheus.yaml](#prometheusyaml)**
2. **[ptheus-configmap.yaml](#ptheus-configmapyaml)**
3. **[ptheus-deployment.yaml](#ptheus-deploymentyaml)**
4. **[ptheus-service.yaml](#ptheus-serviceyaml)**

## prometheus.yaml

Este archivo define un ConfigMap para Prometheus, que incluye la configuración principal del servidor Prometheus.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s

    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['localhost:9090']
      - job_name: 'node-exporter'
        static_configs:
          - targets: ['node-exporter:9100']
```

#### Explicación:
- **apiVersion** y **kind**: Define el tipo de recurso Kubernetes, en este caso, un ConfigMap.
- **metadata**: Proporciona el nombre del ConfigMap.
- **data**: Contiene la configuración de Prometheus en formato `prometheus.yml`.
  - **global**: Configuración global de Prometheus.
    - `scrape_interval`: Intervalo de scrapeo, 15 segundos en este caso.
  - **scrape_configs**: Lista de configuraciones de scrapeo.
    - **job_name**: Nombre del trabajo de scrapeo.
    - **static_configs**: Configuraciones estáticas con los objetivos a monitorear.

## ptheus-configmap.yaml

Este archivo también define un ConfigMap para Prometheus, similar a `prometheus.yaml`, que incluye la configuración principal del servidor Prometheus.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |-
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['localhost:9090']
      - job_name: 'node-exporter'
        static_configs:
          - targets: ['node-exporter:9100']
```

#### Explicación:
- **apiVersion** y **kind**: Define el tipo de recurso Kubernetes, en este caso, un ConfigMap.
- **metadata**: Proporciona el nombre del ConfigMap.
- **data**: Contiene la configuración de Prometheus en formato `prometheus.yml`, idéntica a la de `prometheus.yaml`.

## ptheus-deployment.yaml

Este archivo describe el Deployment de Prometheus, especificando la imagen del contenedor, el número de réplicas y otras configuraciones del pod.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-config
```

#### Explicación:
- **apiVersion** y **kind**: Define el tipo de recurso Kubernetes, en este caso, un Deployment.
- **metadata**: Incluye el nombre del Deployment.
- **spec**: Define las especificaciones del Deployment.
  - **replicas**: Número de réplicas del pod.
  - **selector** y **template**: Detalles sobre los pods que se crearán.
  - **containers**: Define el contenedor Prometheus con la imagen `prom/prometheus:latest` y el puerto `9090`.
  - **volumeMounts**: Monta el volumen de configuración en el contenedor.
  - **volumes**: Define el volumen que se utilizará, en este caso, un ConfigMap.

## ptheus-service.yaml

Este archivo configura el Service de Prometheus, exponiendo el Deployment a través de una dirección IP dentro del clúster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
spec:
  selector:
    app: prometheus
  ports:
  - protocol: TCP
    port: 9090
    targetPort: 9090
  type: LoadBalancer
```

#### Explicación:
- **apiVersion** y **kind**: Define el tipo de recurso Kubernetes, en este caso, un Service.
- **metadata**: Proporciona el nombre del Service.
- **spec**: Define las especificaciones del Service.
  - **selector**: Selecciona los pods con la etiqueta `app: prometheus`.
  - **ports**: Define el puerto que se expondrá y el puerto objetivo en el contenedor.
  - **type**: Tipo de servicio, `LoadBalancer`.

## Conclusión

Cada archivo de configuración desempeña un papel crucial en el despliegue y configuración de Prometheus en un clúster de Kubernetes. Siguiendo las explicaciones y ejemplos proporcionados, puedes configurar y desplegar Prometheus de manera eficiente, asegurando un monitoreo efectivo de tus aplicaciones y servicios en Kubernetes.
