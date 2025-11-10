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

