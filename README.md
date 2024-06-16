# README.md

## Despliegue de Grafana y Prometheus en Kubernetes

Este documento proporciona una guía detallada para desplegar Grafana y Prometheus en un clúster de Kubernetes. A continuación, se describen las configuraciones de cada archivo, la estructura del proyecto y los pasos necesarios para realizar el despliegue.

### Estructura del Proyecto

```bash
.
|-- GRAFANA
|   |-- config-map.yaml
|   |-- datasources-config-map.yaml
|   |-- deployment.yaml
|   `-- service.yaml
|-- PROMETHEUS
|   |-- config-map.yaml
|   |-- deployment.yaml
|   `-- service.yaml
|-- README.md
|-- ingress.yaml
|-- kustomization.yaml
`-- namespace.yaml
```

### Descripción de los Archivos

#### Directorio `GRAFANA`

- **config-map.yaml**: Configuración de Grafana.
- **datasources-config-map.yaml**: Configuración de las fuentes de datos de Grafana.
- **deployment.yaml**: Despliegue de Grafana.
- **service.yaml**: Servicio de Grafana.

#### Directorio `PROMETHEUS`

- **config-map.yaml**: Configuración de Prometheus.
- **deployment.yaml**: Despliegue de Prometheus.
- **service.yaml**: Servicio de Prometheus.

#### Archivos en la raíz

- **README.md**: Documentación del proyecto.
- **ingress.yaml**: Configuración de Ingress para acceder a Grafana y Prometheus.
- **kustomization.yaml**: Configuración de Kustomize para el despliegue.
- **namespace.yaml**: Creación del namespace para los recursos.

### Pasos para el Despliegue

#### 1. Crear el Namespace

El namespace es un espacio de nombres en Kubernetes donde se agruparán todos los recursos relacionados con Grafana y Prometheus.

Archivo: `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

Comando:

```bash
kubectl apply -f namespace.yaml
```

#### 2. Desplegar Prometheus

##### Configuración

Archivo: `PROMETHEUS/config-map.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['localhost:9090']
```

Archivo: `PROMETHEUS/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
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
          image: prom/prometheus
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
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

Archivo: `PROMETHEUS/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  selector:
    app: prometheus
  ports:
    - protocol: TCP
      port: 9090
      targetPort: 9090
```

Comandos:

```bash
kubectl apply -f PROMETHEUS/config-map.yaml
kubectl apply -f PROMETHEUS/deployment.yaml
kubectl apply -f PROMETHEUS/service.yaml
```

#### 3. Desplegar Grafana

##### Configuración

Archivo: `GRAFANA/config-map.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-config
  namespace: monitoring
data:
  grafana.ini: |
    [security]
    admin_user = admin
    admin_password = admin
```

Archivo: `GRAFANA/datasources-config-map.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  datasources.yml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus:9090
      access: proxy
      isDefault: true
```

Archivo: `GRAFANA/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
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
          image: grafana/grafana
          ports:
            - containerPort: 3000
          volumeMounts:
            - name: config-volume
              mountPath: /etc/grafana
            - name: datasources-volume
              mountPath: /etc/grafana/provisioning/datasources
      volumes:
        - name: config-volume
          configMap:
            name: grafana-config
        - name: datasources-volume
          configMap:
            name: grafana-datasources
```

Archivo: `GRAFANA/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  selector:
    app: grafana
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
```

Comandos:

```bash
kubectl apply -f GRAFANA/config-map.yaml
kubectl apply -f GRAFANA/datasources-config-map.yaml
kubectl apply -f GRAFANA/deployment.yaml
kubectl apply -f GRAFANA/service.yaml
```

#### 4. Configurar Ingress

Archivo: `ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: monitoring-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: grafana.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana
                port:
                  number: 3000
    - host: prometheus.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus
                port:
                  number: 9090
```

Comando:

```bash
kubectl apply -f ingress.yaml
```

#### 5. Aplicar la configuración con Kustomize

Archivo: `kustomization.yaml`

```yaml
resources:
  - namespace.yaml
  - PROMETHEUS/config-map.yaml
  - PROMETHEUS/deployment.yaml
  - PROMETHEUS/service.yaml
  - GRAFANA/config-map.yaml
  - GRAFANA/datasources-config-map.yaml
  - GRAFANA/deployment.yaml
  - GRAFANA/service.yaml
  - ingress.yaml
```

Comando:

```bash
kubectl apply -k .
```

### Resumen

1. Crear el namespace.
2. Desplegar Prometheus (config-map, deployment, service).
3. Desplegar Grafana (config-map, datasources-config-map, deployment, service).
4. Configurar Ingress para acceder a Grafana y Prometheus.
5. Aplicar todas las configuraciones con Kustomize.

