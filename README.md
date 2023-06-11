# K86

Sure, I can guide you through the process of deploying a Spring Boot "Hello World" application to Kubernetes using Minikube and Docker. We'll go through the following steps:

1. Create a Spring Boot application
2. Dockerize the application
3. Install Minikube and start a local cluster
4. Write a Kubernetes deployment configuration
5. Deploy the application to Minikube

## Step 1: Create a Spring Boot application

First, we need to create a simple Spring Boot "Hello World" application. If you already have the application, you can skip this step.

Here's how you can create a simple application:

1. Open IntelliJ IDEA and create a new project.

2. Choose `Spring Initializr` from the options provided.

3. Fill in the details. For example:
   - Group: `com.example`
   - Artifact: `helloworld`
   - Name: `helloworld`
   - Package name: `com.example.helloworld`
   - Packaging: `Jar`
   - Java version: `11`

4. In the dependencies section, add `Spring Web`.

5. Click on `Next` and then `Finish`.

After the project is generated, create a new Rest Controller.

```java
package com.example.helloworld;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @GetMapping("/")
    public String helloWorld() {
        return "Hello, world!";
    }
}
```

Now, you can run this application and it will start a local server at `localhost:8080`. When you navigate to this URL in a web browser, you should see the message "Hello, world!".

## Step 2: Dockerize the application

Now we'll Dockerize the Spring Boot application. First, we need to create a Dockerfile in the project root directory. Here's a sample Dockerfile:

```Dockerfile
FROM openjdk:11
EXPOSE 8080
ADD /target/helloworld-0.0.1-SNAPSHOT.jar helloworld.jar
ENTRYPOINT ["java","-jar","/helloworld.jar"]
```

This Dockerfile does the following:

- Starts with a base image containing Java Runtime Environment (JRE 11).
- Exposes port 8080 to map it to the Docker host, so you can connect to it outside the container.
- Adds the jar file (which we will generate in the next step) to the Docker image.
- Defines the command to start the Spring Boot application.

Before we build the Docker image, we need to generate the jar file. You can do this by running the following command in the terminal at the project root directory:

```bash
./mvnw package
```

This will generate a jar file in the `target` directory.

Now you can build the Docker image by running the following command in the terminal:

```bash
docker build -t helloworld .
```

This will create a Docker image named `helloworld`.

## Step 3: Install Minikube and start a local cluster

Before we can deploy our application to Kubernetes, we need to set up a local Kubernetes cluster. We will use Minikube for this. Please ensure you have Minikube and Docker installed on your machine. If not, please download and install them:

- [Install Docker](https://docs.docker.com/get-docker/)
- [Install Minikube](https://minikube.sigs.k8s.io/docs/start/)

After installing Minikube, start it with the following command:

```bash
minikube start --driver=docker
```

This will start a local Kubernetes cluster. You can check the status of the cluster with the following command:

```

```bash
minikube status
```
This command will give you the status of the Minikube's VM, the cluster, and Kubernetes itself【6†source】.

## Step 4: Write a Kubernetes deployment configuration

Now that we have our Docker image and our Minikube cluster running, we can write a Kubernetes deployment configuration. Create a new file named `deployment.yaml` in your project root directory with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-deployment
  labels:
    app: helloworld
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
      - name: helloworld
        image: helloworld
        ports:
        - containerPort: 8080
```

This YAML file defines a Deployment which will ensure that a single instance (replica) of our `helloworld` application is running. It specifies that it should use the `helloworld` Docker image we created earlier. The application is exposed on port 8080.

## Step 5: Deploy the application to Minikube

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

You should see your `helloworld-deployment` listed.

Now we need to expose the deployment as a service so we can access it. Run the following command:

```bash
kubectl expose deployment helloworld-deployment --type=NodePort --port=8080
```

This will create a service that exposes our application to external traffic. By specifying `type=NodePort`, Kubernetes will allocate a port on each node for our service.

You can view the service with the following command:

```bash
kubectl get services
```

You should see your `helloworld-deployment` service listed, along with the port it's been assigned on the node.

Finally, to access the application, you can ask Minikube to give you the URL of the service:

```bash
minikube service helloworld-deployment --url
```

When you navigate to this URL in a web browser, you should see your "Hello, world!" message.

That's it! You've successfully deployed your Spring Boot "Hello World" application to a Kubernetes cluster using Minikube and Docker. Please let me know if you have any questions or run into any issues!
