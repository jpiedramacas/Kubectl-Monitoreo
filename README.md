# Proyecto de Monitoreo con Grafana y Prometheus en Kubernetes

Este proyecto implementa un sistema de monitoreo utilizando Grafana y Prometheus desplegados en un clúster de Kubernetes. A continuación, se detallan los archivos de configuración utilizados y los pasos necesarios para desplegar los servicios utilizando Minikube.

## Contenido del Proyecto

### Grafana
- **gfn-configmap.yaml**: Define un ConfigMap para Grafana, que incluye configuraciones como dashboards y otras configuraciones de la aplicación.
- **gfn-deployment.yaml**: Describe el Deployment de Grafana, especificando la imagen del contenedor, el número de réplicas y otras configuraciones del pod.
- **gfn-hpa.yaml**: Configura el Horizontal Pod Autoscaler (HPA) para Grafana, permitiendo que Kubernetes ajuste automáticamente el número de réplicas del pod basado en la carga.
- **gfn-pv.yaml**: Define un PersistentVolume (PV) para Grafana, proporcionando almacenamiento persistente.
- **gfn-pvc.yaml**: Describe un PersistentVolumeClaim (PVC) para Grafana, que solicita el almacenamiento definido por el PV.
- **gfn-service.yaml**: Configura el Service de Grafana, exponiendo el Deployment a través de una dirección IP dentro del clúster.
- **provisioning/datasources.yaml**: Contiene la configuración para la fuente de datos de Grafana, especificando a Prometheus como la fuente de datos.

### Prometheus
- **prometheus.yaml**: Configuración principal de Prometheus, especificando las reglas de scrapeo y otras configuraciones.
- **ptheus-configmap.yaml**: Define un ConfigMap para Prometheus, que incluye configuraciones adicionales de la aplicación.
- **ptheus-deployment.yaml**: Describe el Deployment de Prometheus, especificando la imagen del contenedor, el número de réplicas y otras configuraciones del pod.
- **ptheus-service.yaml**: Configura el Service de Prometheus, exponiendo el Deployment a través de una dirección IP dentro del clúster.

## Despliegue en Kubernetes

Para desplegar Grafana y Prometheus en tu clúster de Kubernetes, sigue estos pasos:

### Desplegar Grafana

1. **Crear ConfigMap de Grafana**:
    ```bash
    kubectl apply -f GRAFANA/gfn-configmap.yaml
    ```

2. **Crear PersistentVolume y PersistentVolumeClaim para Grafana**:
    ```bash
    kubectl apply -f GRAFANA/gfn-pv.yaml
    kubectl apply -f GRAFANA/gfn-pvc.yaml
    ```

3. **Desplegar Grafana**:
    ```bash
    kubectl apply -f GRAFANA/gfn-deployment.yaml
    kubectl apply -f GRAFANA/gfn-service.yaml
    ```

4. **Configurar Horizontal Pod Autoscaler para Grafana**:
    ```bash
    kubectl apply -f GRAFANA/gfn-hpa.yaml
    ```

5. **Agregar la fuente de datos en Grafana**:
    ```bash
    kubectl apply -f GRAFANA/provisioning/datasources.yaml
    ```

### Desplegar Prometheus

1. **Crear ConfigMap de Prometheus**:
    ```bash
    kubectl apply -f PROMETHEUS/ptheus-configmap.yaml
    ```

2. **Desplegar Prometheus**:
    ```bash
    kubectl apply -f PROMETHEUS/ptheus-deployment.yaml
    kubectl apply -f PROMETHEUS/ptheus-service.yaml
    ```

3. **Agregar configuración principal de Prometheus**:
    ```bash
    kubectl apply -f PROMETHEUS/prometheus.yaml
    ```

### Acceder a los Servicios

Una vez que los servicios estén desplegados, puedes acceder a ellos usando Minikube.

#### Acceder a Grafana
Para acceder al servicio de Grafana, utiliza el siguiente comando:
```bash
minikube service grafana-service
```

#### Acceder a Prometheus
Para acceder al servicio de Prometheus, utiliza el siguiente comando:
```bash
minikube service prometheus-service
```

Esto abrirá el dashboard de Grafana y la interfaz de Prometheus en tu navegador web, permitiéndote monitorizar y visualizar los datos recolectados.

## Conclusión

Siguiendo estos pasos, habrás desplegado y configurado exitosamente un sistema de monitoreo usando Grafana y Prometheus en un clúster de Kubernetes. Ahora puedes comenzar a monitorear tus aplicaciones y servicios, obteniendo valiosa información sobre su rendimiento y estado.
