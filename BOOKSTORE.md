**1. 部署自己之前的作业到Kubernetes上，需要编写一个deployment的yaml文件**

1. Opening a Java Spring Boot application
2. Containerize the application using Docker
3. Push the Docker image to a registry
4. Install and start Minikube
    - brew install minikube
5. Write a Kubernetes deployment YAML file
    - brew install kubectl
6. Deploy the application on Minikube


**Step 1: Use exisiting bookstore application**

Open working bookstore application in intellij
It is importnat to change spring.datasource.url=jdbc:mysql://10.119.12.43:3306/store to a cloud url that can be accessed from inside the docker container.

<img width="502" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/df2540d1-0a0b-4135-9c62-75e039245201">


**Step 2: Containerize the application using Docker**

1. Make sure Docker is installed on your machine. You can verify it by running `docker version` in your terminal.
2. In your project directory, create a Dockerfile with the following content:

```Dockerfile
FROM openjdk:11
ADD target/ORM_backend-0.0.1-SNAPSHOT.jar ORM_backend.jar
ENTRYPOINT ["java","-jar","/ORM_backend.jar"]
```

3. To create the Docker image, first you need to build your application. Navigate to the root directory of your application in terminal and run the following command: 

```bash
mvn package
```

4. After the build is successful, create the Docker image using the following command:

```bash
docker build -t bookstore .
```

**Step 3: Push the Docker image to a registry**

For Minikube to pull the Docker image, it needs to be hosted on a Docker registry. For this guide, we'll use Docker's public registry, Docker Hub.

1. Login to Docker Hub using the command: `docker login`. Provide your Docker Hub username and password.
2. Tag the image with your Docker Hub username: `docker tag bookstore:latest jamesyen220/bookstore:latest`.
3. Push the image to Docker Hub: `docker push jamesyen220/bookstore:latest`.

**Step 4: Install and start Minikube**

1. Install Minikube based on your operating system. 
2. Start Minikube with the command: `minikube start`.

**Step 5: Write a Kubernetes deployment YAML file**

In your project directory, create a file named `deployment.yaml` with the following content:

The YAML content should look like this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookstore
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bookstore
  template:
    metadata:
      labels:
        app: bookstore
    spec:
      containers:
        - name: bookstore
          image: jamesyen220/bookstore:latest
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: bookstore-service
spec:
  type: LoadBalancer
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: bookstore
```

This YAML file includes a Deployment and a Service. The Deployment specifies that we want one replica of our app running, and it should use the Docker image we pushed to Docker Hub. The Service will expose our application on a LoadBalancer type, which means it will be accessible through a public IP address in a real cloud environment. In Minikube, this will be the IP address of the Minikube virtual machine itself.

**Step 6: Deploy the application on Minikube**

Before we can deploy our application, we need to make sure Minikube can pull the Docker image we created. We can do this by pointing Docker's context to Minikube's Docker daemon with the following command:

```bash
eval $(minikube docker-env)
```

Now we can apply our deployment configuration:

```bash
kubectl apply -f deployment.yaml
```

This will create the deployment in our Kubernetes cluster. You can check the status of the deployment with the following command:

```bash
kubectl get deployments
```
<img width="584" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/b1519288-e938-4d89-a555-3b7b2c674ede">

```bash
kubectl get pods
```
<img width="642" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/3d8ec59a-ab5a-4f74-ac6a-aa9b355d752e">


You should see your `bookstore` listed.


**Deployment or Pods not working**

```bash
kubectl describe deployment bookstore-deployment
```

This will create the deployment in our Kubernetes cluster. You can check the status of the deployment with the following command:

```bash
kubectl get deployments
```
<img width="486" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/e0d0a4e9-1ac3-4084-b375-45c916324d88">


This command will show you the events and configuration related to your `bookstore-deployment`. Look for any warning or error messages in the output.

You can also check the logs of the pods themselves. First, get the pod names:

```bash
kubectl get pods
```
<img width="767" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/ef04c585-4432-4e3c-acf0-4224211fc707">


Then, you can get the logs of a specific pod by using its name:

```bash
kubectl logs <pod-name>
```

Replace `<pod-name>` with the name of one of the pods that's not running properly. This should give you more insight into what might be going wrong.


Now we need to expose the deployment as a service so we can access it. Run the following command:

```bash
kubectl expose deployment bookstore --type=NodePort --port=8080
```

This will create a service that exposes our application to external traffic. By specifying `type=NodePort`, Kubernetes will allocate a port on each node for our service.

You can view the service with the following command:

```bash
kubectl get services
```

You should see your `bookstore` service listed, along with the port it's been assigned on the node.
<img width="882" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/483f8813-ff30-4167-a055-5a82ed298023">

Finally, to access the application, you can ask Minikube to give you the URL of the service:

```bash
minikube service bookstore --url
```

<img width="760" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/0e5a2c13-6e20-4ca0-86a3-aac860d750e6">
<img width="1728" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/19c1f448-8b96-4b85-869d-a97685f5354b">

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


**2. 能够动态扩缩容，在yaml文件里写好相关配置后就能实现**
The requirement you mentioned is to dynamically scale the application, and you want the related configurations to be written in the YAML file.

Kubernetes provides a feature called the Horizontal Pod Autoscaler (HPA) that automatically scales the number of pods in a replication controller, deployment, replica set or stateful set based on observed CPU utilization.

Here's how you can set it up:

1. First, you need to make sure that the metrics server is running in your cluster. The Horizontal Pod Autoscaler uses the metrics server to fetch metrics like CPU utilization. In Minikube, you can enable it with the command `minikube addons enable metrics-server`.

2. Next, you'll need to define a HorizontalPodAutoscaler resource. This can be done in the same YAML file as your deployment, or in a separate file. Here's an example configuration:

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: bookstore-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: bookstore
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

This configuration will create a Horizontal Pod Autoscaler that manages the number of pods in the `bookstore` deployment. The number of pods will be between `minReplicas` and `maxReplicas`, scaling based on CPU utilization. In this case, if the average CPU utilization across all pods exceeds 50%, Kubernetes will start creating new pods. If CPU utilization drops below 50%, it will start removing pods, down to a minimum of 1.

3. You can apply this configuration with `kubectl apply -f hpa.yaml`, if you put it in a separate file named `hpa.yaml`.
<img width="600" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/990d3b10-ced4-4d50-8c3b-3847f5ff892e">


4. To check the status of the HPA, you can use the command `kubectl get hpa`.

Remember that HPA is based on the metrics available in your cluster. The example above uses CPU utilization, but Kubernetes can also scale based on memory usage and custom metrics, provided that the metrics are available in your cluster.

Also note that this is a basic example and actual values for `minReplicas`, `maxReplicas` and `averageUtilization` should be chosen based on the requirements of your specific application and environment.

<img width="771" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/f46d264b-5957-4609-b19d-7c3482f7d50d">  

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


**3. 利用Prometheus监控Kubernete以及其上的应用**
Monitoring your Kubernetes cluster and the applications running on it with Prometheus is a great choice. Prometheus, a CNCF project, is a systems and service monitoring system. It collects metrics from configured targets at given intervals, evaluates rule expressions, displays the results, and can trigger alerts if some condition is observed to be true. 

To use Prometheus for monitoring, you'll need to install Prometheus itself, as well as node_exporter for machine metrics, and kube-state-metrics for cluster metrics. You'll also need to configure your applications to expose Prometheus metrics.

Here are the steps to setup Prometheus for Kubernetes:

**Step 1: Install Prometheus**

To install Prometheus, you can create a new namespace and apply the Prometheus operator manifest:


**Create monitoring namespace**

1. **Install Prometheus Operator**: 
    
    ```bash
    kubectl create namespace monitoring

    # Add the prometheus-community Helm repo
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

    # Update the Helm repo
    helm repo update

    # Install Prometheus
    helm install prometheus prometheus-community/prometheus --namespace monitoring
    ```

2. **Verify the installation**: 

    After a while, verify that the Prometheus Operator components are deployed:

    ```
    kubectl get pods -n monitoring
    ```

You should see several pods starting with `prometheus-operator-...` in the "monitoring" namespace. If this works, then you can proceed with the rest of the Prometheus setup.


**Step 2: Create a Prometheus instance**

You can create a Prometheus instance by applying a YAML file with a `Prometheus` resource. Here's an example:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      team: frontend
  resources:
    requests:
      memory: 400Mi
  enableAdminAPI: false
```

This will create a Prometheus instance that will monitor services labeled with `team: frontend`. You can adjust the `matchLabels` to match your environment.

```shell
kubectl apply -f prometheus-instance.yaml
```

**Step 3: Install node_exporter**

Node_exporter is a Prometheus exporter for machine metrics. You can install it with the following commands:

```bash
brew install helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install [RELEASE_NAME] prometheus-community/prometheus-node-exporter
```

In this command, replace `[RELEASE_NAME]` with the name you want to give to this release of the application. If you're running these commands for the first time, Helm will download the chart from the Prometheus community Helm chart repository, update the chart with the latest information from the repository, and then install the application with the name you've specified.

**Step 4: Install kube-state-metrics**

Kube-state-metrics is a service that listens to the Kubernetes API server and generates metrics about the state of the objects. It's not focused on the health of the individual Kubernetes components, but rather on the health of the various objects inside, such as deployments, nodes and pods.

You can install it with the following command:

```yaml
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  labels:  
    app.kubernetes.io/component: exporter  
    app.kubernetes.io/name: kube-state-metrics  
    app.kubernetes.io/version: 2.9.2  
  name: kube-state-metrics  
  namespace: kube-system  
spec:  
  replicas: 1  
  selector:  
    matchLabels:  
      app.kubernetes.io/name: kube-state-metrics  
  template:  
    metadata:  
      labels:  
        app.kubernetes.io/component: exporter  
        app.kubernetes.io/name: kube-state-metrics  
        app.kubernetes.io/version: 2.9.2  
    spec:  
      automountServiceAccountToken: true  
      containers:  
      - image: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.9.2  
        livenessProbe:  
          httpGet:  
            path: /healthz  
            port: 8080  
          initialDelaySeconds: 5  
          timeoutSeconds: 5  
        name: kube-state-metrics  
        ports:  
        - containerPort: 8080  
          name: http-metrics  
        - containerPort: 8081  
          name: telemetry  
        readinessProbe:  
          httpGet:  
            path: /  
            port: 8081  
          initialDelaySeconds: 5  
          timeoutSeconds: 5  
        securityContext:  
          allowPrivilegeEscalation: false  
          capabilities:  
            drop:  
            - ALL  
          readOnlyRootFilesystem: true  
          runAsNonRoot: true  
          runAsUser: 65534  
          seccompProfile:  
            type: RuntimeDefault  
      nodeSelector:  
        kubernetes.io/os: linux  
      serviceAccountName: kube-state-metrics
```

You can save this content into a file (let's call it `kube-state-metrics.yaml`) and then apply it using the following command:

```shell
kubectl apply -f kube-state-metrics.yaml
```

**Step 5: Expose application metrics**

To monitor your applications with Prometheus, they need to expose a `/metrics` HTTP endpoint that returns current metrics in Prometheus format. Spring Boot applications can use the Actuator's `prometheus` micrometer to expose these metrics. You can add these to your application's properties:

```properties
management.endpoints.web.exposure.include=*
management.endpoint.metrics.enabled=true
management.endpoint.prometheus.enabled=true
management.metrics.export.prometheus.enabled=true
```

You'll also need to add the micrometer dependency to your `pom.xml`:

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
            <version>1.7.3</version>
        </dependency>
```

Once your application is exposing metrics, you can add a `ServiceMonitor` to tell Prometheus to scrape metrics from your application:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: bookstore-monitor
  labels:
    team: frontend
spec:
  selector:
    matchLabels:
      app: bookstore
  endpoints:
    - port: web
      path: /actuator/prometheus
```


**Verifying Your Prometheus Setup**

After setting up Prometheus and configuring your application to expose metrics, you can verify that Prometheus is successfully scraping metrics from your application by accessing the Prometheus dashboard:

1. Run the following command to set up port forwarding:

```bash
kubectl port-forward svc/prometheus-operated -n monitoring 9090
```

To check the service name of your Prometheus instance in Kubernetes, you can use the following command:

```
kubectl get services -n monitoring
```
<img width="770" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/b097935a-1d74-4ca5-9f8d-b4bb7668a1bb">


This command will list all the services in the "monitoring" namespace. Look for the Prometheus service in the output. The service name should be displayed in the "NAME" column.

<img width="900" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/43bb3055-cab1-4b5f-9196-76643b1a9673">

2. Open a web browser and navigate to `http://localhost:9090`.
<img width="1728" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/47d977a2-f028-4485-80a1-49da8ef441f6">

3. Click on the "Status" dropdown menu and select "Targets". You should see your application listed as a target, with the status "UP".
<img width="1728" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/9b5332d3-f5bd-43da-a1d3-ed941ede61f9">
<img width="1726" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/c10355bd-8d3c-437a-9a20-e6591ebf2e1f">
<img width="1726" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/fbd8e4be-2fbb-4deb-a028-ac9e23bf3e21">
<img width="1728" alt="image" src="https://github.com/JamesYen220/K86/assets/100248639/5ac28384-4fa1-427b-bfda-0ffa3159d600">

4. You can also query your application's metrics: type the metric's name into the "Expression" input field and click "Execute".

Remember to replace `prometheus-operator-prometheus` with the service name of your Prometheus instance if you named it differently.
