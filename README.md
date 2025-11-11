# Microservice App - PRFT Devops Training

This is the application you are going to use through the whole traninig. This, hopefully, will teach you the fundamentals you need in a real project. You will find a basic TODO application designed with a [microservice architecture](https://microservices.io). Although is a TODO application, it is interesting because the microservices that compose it are written in different programming language or frameworks (Go, Python, Vue, Java, and NodeJS). With this design you will experiment with multiple build tools and environments. 

## Components
In each folder you can find a more in-depth explanation of each component:

1. [Users API](/users-api) is a Spring Boot application. Provides user profiles. At the moment, does not provide full CRUD, just getting a single user and all users.
2. [Auth API](/auth-api) is a Go application, and provides authorization functionality. Generates [JWT](https://jwt.io/) tokens to be used with other APIs.
3. [TODOs API](/todos-api) is a NodeJS application, provides CRUD functionality over user's TODO records. Also, it logs "create" and "delete" operations to [Redis](https://redis.io/) queue.
4. [Log Message Processor](/log-message-processor) is a queue processor written in Python. Its purpose is to read messages from a Redis queue and print them to standard output.
5. [Frontend](/frontend) Vue application, provides UI.

## Architecture

Take a look at the components diagram that describes them and their interactions.
![microservice-app-example](/arch-img/Microservices.png)

## Workshop development

This project implements a complete microservices architecture in Kubernetes using Minikube as the local environment.
It includes:

* Containerization of each service.
* Deployment of multiple microservices.
* Config maps and secrets.
* Autoscaling with HPA.
* Blue/Green Deployment.
* Observability with Prometheus and Grafana.
* Security through Network Policies with Calico.

The main objective was to build a functional, secure, and scalable environment that reproduces best practices in modern architecture on Kubernetes.

### Image building: Dockerfiles

The original project did not include ready-to-use images, so the first step was to create all the Dockerfiles to ensure environment independence and a reproducible build.

Services included:

| Service	  | Technology |	Base image used	   | Port   |
|----------   |:----------:|:---------------------:| ------:|
| frontend	  | Node	   | node:8.17.0	       | 8080   |
| auth-api	  | Go	       | golang:1.18	       | 8000   |
| users-api	  | Java 	   | maven:3.9.6-eclipse...| 8083   |
| todos-api	  | Node	   | node:8.17.0	       | 8082   |
| log-message |	Python	   | python:3.10-slim      |— (jobs)|
| redis	      | Redis	   | redis:7               | 6379   |

Each image was compiled with:

```bash 
docker build -t <service>:1.0 .
minikube image load <service>:1.0
```
This is to ensure that minikube has local access to the images without having to upload and download them to DockerHub.

### Deployment of microservices: 

Each service was deployed using a Deployment that utilizes:
• resource requests and limits
• labels for pod selection 
• RollingUpdate as an update strategy

The information required to deploy each service is located in the /deployment folder of the project.

Before applying the deployments, the namespaces must be created.
In this case, a general namespace called microservice-app is used, and all application services reside inside it.

Redis must also be deployed before the rest of the microservices, since it communicates directly with the log-message-processor for log publishing and consumption. Without Redis running, the logging pipeline will fail.

**Shared configuration via ConfigMap**

The application relies on a shared ConfigMap that provides all key environment variables required by the services.
This prevents hardcoding values inside the Deployments and ensures consistent configuration throughout the system.

The ConfigMap includes predefined values such as:

```bash
USERS_API_ADDRESS: "http://users-api-service:8083"
AUTH_API_ADDRESS: "http://auth-api-service:8000"
TODOS_API_ADDRESS: "http://todos-api-service:8082"
REDIS_HOST: "redis-service"
REDIS_PORT: "6379"
REDIS_CHANNEL: "log_channel"
TODOS_API_PORT: "8082"
SERVER_PORT: "8083"
AUTH_API_PORT: "8000"
REDIS_ADDR: "redis-service.microservice-app.svc.cluster.local:6379"
```

This configuration is loaded automatically by the deployments of:

- frontend
- auth-api
- users-api
- todos-api
- log-message-processor

With this configuration in place, the microservices can communicate with each other using internal DNS, Redis can be reached by the processor, and the application becomes operational.

In addition, a secret was also configured. For this, there is a sample base template that shows how secrets are applied.

### Services: internal and external communication

Services were configured to expose the microservices:

|Service                | Type        |  Port |
|----------------------:| :----------:|------:|
|auth-api-service       | ClusterIP	  | 8000  |
|users-api-service      | ClusterIP   | 8083  |
|todos-api-service      | ClusterIP   | 8082  |
|redis-service          | ClusterIP   | 6379  |
|redis-exporter-service | ClusterIP   | 9121  |
|frontend-service       | LoadBalancer| 8080  |

This enabled internal communication via DNS:

```bash
<service>.<namespace>.svc.cluster.local
```

And external exposure of the frontend via:

```bash
minikube tunnel
```
Once the namespace, ConfigMap, Redis, and the initial Deployments and Services are applied, the application becomes fully functional.
Below is an example image showing the services running successfully:

<img width="1917" height="633" alt="image" src="https://github.com/user-attachments/assets/5cc720af-9f83-481a-a968-f3072fce42dd" />

### Horizontal Pod Autoscaler (HPA) 

Autoscaling was configured for frontend, users-api, todos-api, and auth-api.
The metrics-server was activated with:

```bash
minikube addons enable metrics-server
```

To do this, the HPA is created in a similar way to the one shown below: 

```bash
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: auth-api-hpa
  namespace: microservice-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: auth-api
  minReplicas: 1
  maxReplicas: 3
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

Here, it is important to note that these resources must be specified in the deployment, and the requests and limits must be indicated. In this case, they were set for CPU and memory. 

Para validar que se encuentran activos se emplea el comando visualizado a continuación: 

<img width="1194" height="152" alt="image" src="https://github.com/user-attachments/assets/67444b16-677c-482c-abb9-b8d921f9aabd" />

**NOTE:** It should be noted that the minimum and maximum number of replicas for each pod was changed until one suitable for each service was found. 

### **Blue/Green Deployment**
To update the frontend without downtime, a Blue/Green scheme was implemented:

- Deployment: frontend-blue
- Deployment: frontend-green

The service selected the active version:

```bash
selector:
  app: frontend
  version: blue
```

In this case, to perform the simulation, the frontend-green deployment was created with the same image that was used for frontend-blue, but with a different tag. As mentioned above, the service currently points to the blue version.

### Prometheus and Grafana

To enable full observability of the microservices and infrastructure, a dedicated namespace named monitoring was created.
Inside this namespace, a complete monitoring stack was deployed manually (without Helm charts), consisting of several exporters, RBAC resources, and custom Prometheus configurations.

**Components installed in the monitoring namespace**

The monitoring stack includes:

**Prometheus**

Main metrics collector and time-series database.

**kube-state-metrics:** Exposes Kubernetes object metrics such as Deployments, Pods, and Nodes.

**node-exporter:** Provides hardware and OS-level metrics from the Minikube node.

**blackbox-exporter:** Used for probing HTTP endpoints of the microservices, verifying availability.

**redis-exporter:** Exposes Redis-specific performance metrics.

**cAdvisor (DaemonSet):** Collects container-level stats (CPU, memory, filesystem). Installed as a DaemonSet to run on the Minikube node.

**Custom Prometheus ConfigMap:**

contains all scrape jobs, including:

- Kubernetes nodes
- cAdvisor
- kube-state-metrics
- blackbox probes
- Redis exporter
- Node exporter
- API server metrics (via RBAC)

**Service Account, Roles, and RoleBindings:** Required to allow Prometheus to access the Kubernetes API server and node metrics securely.

**Grafana: ** Used to visualize metrics using dashboards.

**Why this setup was done without helm:**

Although Helm provides the fastest and easiest deployment path, this project opts for manual configuration to show full understanding of:

- how exporters work individually
- how Prometheus discovers and scrapes targets
- how RBAC influences metric permissions
- how each component interacts inside the cluster

Node Exporter was the only component installed via Helm, since it is officially distributed that way and simplifies its setup.

**Prometheus configuration**

Prometheus uses a custom ConfigMap (prometheus-config) that defines all scrape jobs.
These include cluster-level and microservice-level metrics, as shown in the configuration:

- scrape Redis via redis-exporter
- scrape node resources via node-exporter
- scrape pod and container metrics via cAdvisor
- scrape HTTP endpoints via blackbox-exporter
- scrape Stateful Kubernetes data via kube-state-metrics
- scrape API server metrics through RBAC-enabled access

This configuration ensures complete visibility of the system and supports alerting and dashboards.

<img width="1887" height="1045" alt="image" src="https://github.com/user-attachments/assets/6e1b59ec-4a75-44b3-8c1a-fe214c197918" />


**Grafana dashboards**

Grafana was deployed inside the same namespace, using the Prometheus service as the datasource.
Several dashboards were imported and customized to track:

- Pod resource usage
- Redis performance
- API response times and availability (from blackbox-exporter)
- Cluster node health (node-exporter + cAdvisor)

These dashboards allow real-time monitoring of system performance and help detect issues such as resource saturation, pod restarts, or API downtime.

<img width="1520" height="810" alt="image" src="https://github.com/user-attachments/assets/3c6519fd-b0e2-41b0-b2e5-2685c8f9360f" />

<img width="1577" height="939" alt="image" src="https://github.com/user-attachments/assets/f8b11626-f998-42dc-bd67-3659791a5a60" />

<img width="1499" height="911" alt="image" src="https://github.com/user-attachments/assets/5b3d4fa8-ea85-4c26-8ff8-f59750154fb0" />

<img width="1544" height="1673" alt="image" src="https://github.com/user-attachments/assets/726ea1f8-4292-40ec-8c3c-211addec98a5" />

<img width="1530" height="951" alt="image" src="https://github.com/user-attachments/assets/5ff90738-d5c3-457a-b649-2f460542f3d9" />

**Network policies**

To apply the policies, Calico was enabled:

```bash
minikube start --network-plugin=cni --cni=calico
```

| **Policie**                       | **Purpose**                                              |
|-----------------------------------|-------------------------------------------------------------|
| **allow-dns**                     | Allow DNS traffic from all pods                             |
| **allow-monitoring**              | Allows scraping from Prometheus                             |
| **auth-api-policy**               | Strict control of entry and exit                            |
| **users-api-policy**              | Isolate `users-api` from other services                     |
| **todos-api-policy**              | Isolate `all-api`                                           |
| **redis-policy**                  | Allows access to Redis only from authorized services        |
| **frontend-policy**               | Allows external traffic and communication with internal APIs|
| **log-message-processor-policy**  | Authorize access to Redis only                              |



<img width="1919" height="723" alt="image" src="https://github.com/user-attachments/assets/9de349e8-5507-4bf5-a41d-664d87153760" />

