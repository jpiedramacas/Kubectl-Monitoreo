Para desplegar correctamente `node-exporter` para monitorear tus nodos de Kubernetes y configurar un dashboard en Grafana, sigue estos pasos:

### Paso 1: Crear el Namespace `monitoring`

1. **Crear el namespace `monitoring`**:

   ```sh
   kubectl create namespace monitoring
   ```

### Paso 2: Desplegar Node Exporter

2. **Crear el archivo `node-exporter.yaml`** con el siguiente contenido:

   ```yaml
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: node-exporter
     namespace: monitoring
   spec:
     selector:
       matchLabels:
         app: node-exporter
     template:
       metadata:
         labels:
           app: node-exporter
       spec:
         containers:
         - name: node-exporter
           image: quay.io/prometheus/node-exporter:latest
           ports:
           - containerPort: 9100
             hostPort: 9100
           resources:
             limits:
               memory: 200Mi
               cpu: 100m
             requests:
               memory: 100Mi
               cpu: 50m
           volumeMounts:
           - name: proc
             mountPath: /host/proc
             readOnly: true
           - name: sys
             mountPath: /host/sys
             readOnly: true
           - name: rootfs
             mountPath: /rootfs
             readOnly: true
           args:
           - --path.procfs=/host/proc
           - --path.sysfs=/host/sys
           - --web.listen-address=0.0.0.0:9100
           - --collector.filesystem.ignored-mount-points=^/(dev|proc|sys|var/lib/docker/.+)($|/)
         securityContext:
           runAsNonRoot: true
           runAsUser: 65534
         volumes:
         - name: proc
           hostPath:
             path: /proc
         - name: sys
           hostPath:
             path: /sys
         - name: rootfs
           hostPath:
             path: /
   ```

3. **Desplegar Node Exporter**:

   ```sh
   kubectl apply -f node-exporter.yaml
   ```

### Paso 3: Desplegar Grafana y Prometheus

4. **Desplegar los ConfigMaps y recursos de Grafana**:

   Asegúrate de que todos los archivos mencionados (`gfn-configmap.yaml`, `gfn-deployment.yaml`, `gfn-hpa.yaml`, `gfn-pv.yaml`, `gfn-pvc.yaml`, `gfn-service.yaml`) estén en tu directorio actual y correctos.

5. **Aplicar los archivos de configuración**:

   ```sh
   kubectl apply -f gfn-configmap.yaml -n monitoring
   kubectl apply -f gfn-pv.yaml -n monitoring
   kubectl apply -f gfn-pvc.yaml -n monitoring
   kubectl apply -f gfn-deployment.yaml -n monitoring
   kubectl apply -f gfn-service.yaml -n monitoring
   kubectl apply -f gfn-hpa.yaml -n monitoring
   ```

### Paso 4: Configurar Grafana

6. **Acceder a Grafana**:

   Si estás utilizando un servicio de tipo `LoadBalancer`, obtén la IP externa:

   ```sh
   kubectl get svc grafana -n monitoring
   ```

   Si estás utilizando `NodePort`, accede a través de `http://<NodeIP>:<NodePort>`.

7. **Configurar Prometheus como fuente de datos en Grafana**:

   - Abre Grafana en tu navegador.
   - Inicia sesión con las credenciales configuradas en `grafana.ini` (`admin`/`admin` por defecto).
   - Navega a `Configuration` > `Data Sources`.
   - Agrega una nueva fuente de datos y selecciona `Prometheus`.
   - Configura la URL como `http://prometheus:9090` y guarda.

### Paso 5: Importar Dashboards

8. **Importar dashboards predefinidos para Node Exporter en Grafana**:

   - Ve a `+` > `Import`.
   - Usa el ID del dashboard de Node Exporter (por ejemplo, 1860) y sigue las instrucciones para importar el dashboard.

### Archivo de Configuración Completo

Aquí tienes todos los archivos necesarios listos para ser aplicados.

#### node-exporter.yaml

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
      - name: node-exporter
        image: quay.io/prometheus/node-exporter:latest
        ports:
        - containerPort: 9100
          hostPort: 9100
        resources:
          limits:
            memory: 200Mi
            cpu: 100m
          requests:
            memory: 100Mi
            cpu: 50m
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: rootfs
          mountPath: /rootfs
          readOnly: true
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --web.listen-address=0.0.0.0:9100
        - --collector.filesystem.ignored-mount-points=^/(dev|proc|sys|var/lib/docker/.+)($|/)
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: rootfs
        hostPath:
          path: /
```

#### gfn-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-provisioning
  namespace: monitoring
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
    admin_user = admin
    admin_password = admin
```

#### gfn-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
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

#### gfn-hpa.yaml

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: grafana-hpa
  namespace: monitoring
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: grafana
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 80
```

#### gfn-pv.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: grafana-pv
  namespace: monitoring
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/grafana"
```

#### gfn-pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

#### gfn-service.yaml

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
  type: LoadBalancer
```

### Paso 6: Desplegar todos los archivos

```sh
kubectl apply -f node-exporter.yaml
kubectl apply -f gfn-configmap.yaml
kubectl apply -f gfn-pv.yaml
kubectl apply -f gfn-pvc.yaml
kubectl apply -f gfn-deployment.yaml
kubectl apply -f gfn-service.yaml
kubectl apply -f gfn-hpa.yaml
```

Con estos pasos,
