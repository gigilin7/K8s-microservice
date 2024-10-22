# Goal
+ Set up a local development environment for Docker and Kubernetes
+ Build simple Java microservices with Helidon
+ Use Docker to turn microservices into container images
+ Deploy microservices in a local Kubernetes cluster
+ Scaling up and down microservices in a cluster

# Requirement
+ Docker version: 24.0.6
+ Kubernetes version: 1.28.2
+ JDK version: 13
+ Maven version: 3.9.9

# Start
### Build a microservice
Use Maven and Helidon to create a new project. The following steps will create and start a simple starter project:
```
$ mvn archetype:generate -DinteractiveMode=false \
    -DarchetypeGroupId=io.helidon.archetypes \
    -DarchetypeArtifactId=helidon-quickstart-se \
    -DarchetypeVersion=1.0.1 \
    -DgroupId=io.helidon.examples \
    -DartifactId=helidon-quickstart-se \
    -Dpackage=io.helidon.examples.quickstart.se
```
![build_microservice](https://github.com/user-attachments/assets/7bbd53f8-72d5-48eb-a4e4-9eee5f40f74c)

Move to the `helidon-quickstart-se` directory and build the service:
```
$ cd helidon-quickstart-se
$ mvn package
```
![build_microservice_jar](https://github.com/user-attachments/assets/6b61dd22-3987-447b-813e-6e5e37b1d604)

Now that you have a working microservice example, this project will build an application JAR file for your example microservice, execute it and confirm that everything is working properly:
```
$ java -jar ./target/helidon-quickstart-se.jar
```
![run_jar](https://github.com/user-attachments/assets/867d65f8-e5f0-40f6-8c77-40468e1f2feb)

Use `curl` in another terminal to test the functionality of the service:
```
$ curl -X GET http://localhost:8080/greet
{"message":"Hello World!"}
$ curl -X GET http://localhost:8080/greet/Mary
{"message":"Hello Mary!"}
$ curl -X PUT -H "Content-Type: application/json" -d '{"greeting" : "Hola"}'  \
http://localhost:8080/greet/greeting
$ curl -X GET http://localhost:8080/greet/Maria
{"message":"Hola Maria!"}
```
![test_service](https://github.com/user-attachments/assets/97db361c-1560-44b9-8fa3-61f1b4df6750)

Create a Docker image file for the microservice:
```
docker build -t helidon-quickstart-se target
```
![docker_build](https://github.com/user-attachments/assets/86d27e79-f238-4002-9021-79d9c1721940)

### Practice using Kubernetes
Start by testing a simple example on a local Kubernetes cluster. First, start the local cluster using `minikube start` command. This command provides the running status and will wait for the completion message before continuing.
```
$ minikube start
```
![minikube_start](https://github.com/user-attachments/assets/8e3a12ee-3782-4176-b957-d20150d87447)

Use `kubectl create` to create a deployment with a pre-prepared hello world example. Then, use `kubectl get pods` to display your deployment and its pods. You will see something similar to the following:
```
$ kubectl create deployment hello-node \
  --image=gcr.io/hello-minikube-zero-install/hello
$ kubectl get deployments
$ kubectl get pods
```
![kubectl_create_example](https://github.com/user-attachments/assets/ebb79cd4-ebe0-4dff-9d73-51c02ce36bbb)

⚠ The `hello-node` deployment was created successfully, but the Pod status did not reach `Running`.

Use `kubectl describe` to examine the Pod's detailed status and event logging to understand why the Pod did not reach the `Running` state:
```
kubectl describe pod hello-node-68b4b4f98-fxksc
```
![kubectl_describe_pod](https://github.com/user-attachments/assets/b89da0f2-020d-4cc2-b3e0-28b1c81b7414)

According to the output of `kubectl describe pod`, Pod is in the `ImagePullBackOff` state. The specific reason is that Kubernetes cannot pull the image from `Google Container Registry (gcr.io)` because the image does not exist or you do not have permission to access it. The Error message is `Caller does not have permission or the resource may not exist`.

Try visiting `https://gcr.io/hello-minikube-zero-install/hello` and find that it has indeed been deleted.

![image_has_deleted](https://github.com/user-attachments/assets/3735d657-9123-485c-9f9a-23050acdc677)

Because this example cannot be used, we delete it directly.
```
kubectl delete deployment hello-node
```
![kubectl_delete](https://github.com/user-attachments/assets/1a2d561b-e16c-4df3-b3fd-e09d061d592a)

### It’s time to deploy microservices to Kubernetes
Helidon's starting project, in `target/app.yaml`, contains sample configurations for deployment and service:
```yaml
kind: Service
apiVersion: v1
metadata:
  name: helidon-quickstart-se
  labels:
    app: helidon-quickstart-se
spec:
  type: NodePort
  selector:
    app: helidon-quickstart-se
  ports:
  - port: 8080
    targetPort: 8080
    name: http
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: helidon-quickstart-se
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: helidon-quickstart-se
        version: v1
    spec:
      containers:
      - name: helidon-quickstart-se
        image: helidon-quickstart-se
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
---
```
⚠ In the deployment specification, the image file name uses the local image file name: `helidon-quickstart-se`. When you deploy to Kubernetes, it will first look for this image locally.

To use Docker on Windows to connect to a Minikube virtual machine and build a container image in Minikube's Docker environment, you can configure the environment through the following steps:

Output some environment variable configurations.
```
$ minikube docker-env
```
![docker_env](https://github.com/user-attachments/assets/7c731376-a5ca-48ef-a8bd-759d0e1da771)

Calling this command with `eval` sets the variables to your current terminal's environment variables.
```
$ eval $(minikube -p minikube docker-env)
```
When you execute docker, it will connect to Docker in the Minikube virtual machine. 
⚠ this configuration is only effective in your current terminal and is not a permanent modification.

Next, create a Docker image file for the microservice. It will be built in Minikube's virtual machine, and the image file will also be stored in it:
```
docker build -t helidon-quickstart-se target
```
![docker_build_microservice](https://github.com/user-attachments/assets/c399478b-8fad-4cb3-aba3-75d89ee0ef6f)

```
$ kubectl create -f target/app.yaml
```
⚠ The error message:
```
service/helidon-quickstart-se created
error: resource mapping not found for name: "helidon-quickstart-se" namespace: "" from "target/app.yaml": no matches for kind "Deployment" in version "extensions/v1beta1"
ensure CRDs are installed first
```

✔ Due to Kubernetes version updates, `extensions/v1beta1` has been deprecated, and the `apps/v1` API version should be used when creating a Deployment.

✔ In Deployment, `selector` is required. This field tells Kubernetes how to match the Pod, making sure it matches the labels in `template.metadata.labels`.

Below is the updated `target/app.yaml`:
```yaml
kind: Service
apiVersion: v1
metadata:
  name: helidon-quickstart-se
  labels:
    app: helidon-quickstart-se
spec:
  type: NodePort
  selector:
    app: helidon-quickstart-se
  ports:
  - port: 8080
    targetPort: 8080
    name: http
---
kind: Deployment
apiVersion: apps/v1  # extensions/v1beta1 已被棄用，應使用 apps/v1
metadata:
  name: helidon-quickstart-se
spec:
  replicas: 1
  selector:
    matchLabels:  # 添加 selector 匹配 labels
      app: helidon-quickstart-se
  template:
    metadata:
      labels:
        app: helidon-quickstart-se
        version: v1
    spec:
      containers:
      - name: helidon-quickstart-se
        image: helidon-quickstart-se
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
---
```

Reapply the updated YAML file.
```
$ kubectl apply -f target/app.yaml
```
![kubectl_apply_yaml](https://github.com/user-attachments/assets/21ef82bf-1f53-479d-9cef-0f1d3a1233fe)

Set up deployment and service.
```
$ kubectl get pods
$ kubectl get service
```
![kubectl_get_service](https://github.com/user-attachments/assets/5b75d646-e7b0-4e7d-ae6e-3256dfc23482)

Get the URL that can access the service.
```
minikube service helidon-quickstart-se –url
```
![get_url](https://github.com/user-attachments/assets/eeddbfcd-aa9b-469b-86dc-b70930ad3770)

Test the service.
```
$ curl -X GET http://192.168.58.2:31774/gree
{"message":"Hello World!"}
```

# 🎉 Successfully deployed a microservice to Kubernetes 🎉

# Supplementary information
When you start the service, you will see the output of the web server startup. This output will also be captured by the container and can be viewed through the `kubectl logs` command.
```
$ kubectl logs helidon-quickstart-se-7f4dcbf4c-zkrdr
```
![kubectl_log](https://github.com/user-attachments/assets/007c4c92-c75f-40be-9533-d5b5b8ca161c)

### Expand or reduce the number of deployments
Edit `target/app.yaml` and specify the number of `replicas`.

### Delete the service and deployment
Delete them if you don't need it.
```
$ kubectl delete service helidon-quickstart-se
$ kubectl delete deployment helidon-quickstart-se
```
![kubectl_delete_microservice](https://github.com/user-attachments/assets/be01ed43-1f4e-4ff3-8757-dab05b2e8392)



[Reference for learning](https://medium.com/java-magazine-translation/kubernetes-%E5%85%A5%E9%96%80-68506652b636)
