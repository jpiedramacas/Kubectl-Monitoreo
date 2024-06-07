## Descripción del Proyecto

Este proyecto tiene como objetivo desplegar una infraestructura de monitoreo utilizando Prometheus y Grafana en un clúster de Kubernetes gestionado por Minikube en una instancia EC2. A continuación se detallan los pasos necesarios para desplegar la solución completa.

## Requisitos

- **Cuenta de AWS**: Necesitas una cuenta de AWS para crear y gestionar recursos en la nube.
- **Clave SSH**: Para acceder a la instancia EC2.
- **Conexión a Internet**: Para descargar dependencias y contenedores Docker.

## Instrucciones

### Paso 1: Crear la Infraestructura

1. **Crear una instancia EC2**

   - **Tipo de instancia**: `t3.small`
   - **AMI**: `Amazon Linux 1`

   Durante la creación de la instancia, asegúrate de:

   - **Crear una clave en la sección “par de claves”** y descargarla para acceder a la instancia vía SSH.
   - **Configurar el grupo de seguridad** añadiendo las siguientes reglas de entrada:
     - Puerto 3000 (TCP) para Grafana.
     - Puerto 9090 (TCP) para Prometheus.

### Paso 2: Instalar Dependencias

Accede a la instancia EC2 vía SSH utilizando la clave descargada:

```sh
ssh -i "your-key.pem" ec2-user@your-ec2-public-dns
```

1. **Actualizar librerías**

   ```sh
   sudo yum update -y
   ```

   Este comando actualiza todas las librerías del sistema a sus últimas versiones.

2. **Instalar Docker**

   ```sh
   sudo yum install -y docker
   sudo service docker start
   sudo usermod -a -G docker $(whoami)
   ```

   - `sudo yum install -y docker`: Instala Docker.
   - `sudo service docker start`: Inicia el servicio Docker.
   - `sudo usermod -a -G docker $(whoami)`: Añade el usuario actual al grupo Docker para ejecutar comandos Docker sin `sudo`.

3. **Instalar Minikube y kubectl**

   ```sh
   curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
   sudo install minikube-linux-amd64 /usr/local/bin/minikube

   curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
   chmod +x ./kubectl
   sudo mv ./kubectl /usr/local/bin/kubectl
   ```

   - `curl -LO`: Descarga los binarios de Minikube y kubectl.
   - `sudo install`: Instala Minikube en `/usr/local/bin`.
   - `chmod +x ./kubectl`: Da permisos de ejecución al binario `kubectl`.
   - `sudo mv ./kubectl /usr/local/bin/kubectl`: Mueve `kubectl` al directorio de binarios del sistema.

4. **Iniciar Minikube**

   ```sh
   minikube start --driver=none
   ```

   Este comando inicia un clúster de Minikube utilizando el driver `none`, que es adecuado para entornos de servidor.

### Paso 3: Añadir Configuración

1. **Crear un archivo `docker-compose.yml`**

   ```yaml
   version: '3'
   services:
     grafana:
       image: grafana/grafana:latest
       ports:
         - "3000:3000"
       volumes:
         - grafana-data:/var/lib/grafana
         - ./provisioning:/etc/grafana/provisioning
       environment:
         - GF_SECURITY_ADMIN_USER=admin
         - GF_SECURITY_ADMIN_PASSWORD=admin
       networks:
         - monitoring

     node-exporter:
       image: prom/node-exporter:latest
       ports:
         - "9100:9100"
       networks:
         - monitoring

     prometheus:
       image: prom/prometheus:latest
       ports:
         - "9090:9090"
       volumes:
         - ./prometheus.yml:/etc/prometheus/prometheus.yml
       networks:
         - monitoring

   networks:
     monitoring:

   volumes:
     grafana-data:
   ```

   Este archivo de Docker Compose define los servicios Grafana, Node Exporter y Prometheus, incluyendo sus configuraciones de red y volúmenes.

2. **Crear un archivo `prometheus.yml`**

   ```yaml
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

   Este archivo configura Prometheus para que extraiga métricas de sí mismo y de Node Exporter.

3. **Crear un directorio `provisioning` y un archivo `datasources.yaml`**

   ```sh
   mkdir provisioning
   ```

   Dentro del directorio `provisioning`, crea el archivo `datasources.yaml`:

   ```yaml
   apiVersion: 1
   datasources:
     - name: Prometheus
       type: prometheus
       access: proxy
       url: http://prometheus:9090
   ```

   Este archivo configura Grafana para utilizar Prometheus como su fuente de datos.

### Paso 4: Desplegar en Kubernetes

1. **Crear un archivo de despliegue `k8s-deployment.yaml`**

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: monitoring-deployment
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: monitoring
     template:
       metadata:
         labels:
           app: monitoring
       spec:
         containers:
         - name: grafana
           image: grafana/grafana:latest
           ports:
           - containerPort: 3000
           env:
           - name: GF_SECURITY_ADMIN_USER
             value: "admin"
           - name: GF_SECURITY_ADMIN_PASSWORD
             value: "admin"
           volumeMounts:
           - name: grafana-data
             mountPath: /var/lib/grafana
           - name: grafana-provisioning
             mountPath: /etc/grafana/provisioning
         - name: node-exporter
           image: prom/node-exporter:latest
           ports:
           - containerPort: 9100
         - name: prometheus
           image: prom/prometheus:latest
           ports:
           - containerPort: 9090
           volumeMounts:
           - name: prometheus-config
             mountPath: /etc/prometheus
         volumes:
         - name: grafana-data
           emptyDir: {}
         - name: grafana-provisioning
           configMap:
             name: grafana-provisioning
         - name: prometheus-config
           configMap:
             name: prometheus-config
   ```

   Este archivo de despliegue configura y despliega los contenedores de Grafana, Node Exporter y Prometheus en el clúster de Kubernetes.

2. **Crear un archivo de ConfigMap `k8s-configmap.yaml`**

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

   ---

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
   ```

   Este archivo crea dos ConfigMaps: uno para la configuración de Prometheus y otro para la configuración de Grafana.

3. **Aplicar las configuraciones en Kubernetes**

   ```sh
   kubectl apply -f k8s-configmap.yaml
   kubectl apply -f k8s-deployment.yaml
   ```

   Estos comandos aplican las configuraciones definidas en los archivos `k8s-configmap.yaml` y `k8s-deployment.yaml` al clúster de Kubernetes.

4. **Crear servicios para exponer las aplicaciones**

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: grafana-service
   spec:
     selector:
       app: monitoring
     ports:
       - protocol: TCP
         port: 3000
         targetPort: 3000
     type: LoadBalancer

   ---

   apiVersion: v1
   kind: Service
   metadata:
     name: prometheus-service
   spec:
     selector:
       app: monitoring
     ports:
       - protocol: TCP
         port: 9090
         targetPort: 9090
     type: LoadBalancer
   ```

   Estos archivos definen los servicios de Kubernetes para Grafana y Prometheus, exponiendo sus puertos correspondientes.

5. **Aplicar los servicios**

   ```sh
   kubectl apply

 -f grafana-service.yaml
   kubectl apply -f prometheus-service.yaml
   ```

   Estos comandos aplican las configuraciones de servicios al clúster de Kubernetes.

### Paso 5: Acceder a las Aplicaciones

1. **Obtener la URL del servicio Grafana**

   ```sh
   minikube service grafana-service --url
   ```

2. **Obtener la URL del servicio Prometheus**

   ```sh
   minikube service prometheus-service --url
   ```

Estos comandos proporcionan las URLs para acceder a Grafana y Prometheus respectivamente.

## Conclusión

Este README proporciona una guía paso a paso para desplegar un stack de monitoreo utilizando Prometheus y Grafana en un clúster de Kubernetes gestionado por Minikube. Siguiendo estas instrucciones, deberías poder configurar el entorno, construir y desplegar la solución completa, mejorando así la observabilidad de tu sistema.
