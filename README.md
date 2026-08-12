# Library Events Producer

A production-style event-driven microservice built with **Java, Spring Boot, Apache Kafka, Docker, and Kubernetes**. The service exposes REST APIs for creating and updating library events and publishes those events reliably to Kafka.

## Overview

The Library Events Producer is a Spring Boot service responsible for accepting library events, such as adding or updating a book, and publishing them to an Apache Kafka topic.

The event ID is used as the Kafka message key so that events belonging to the same library record are routed to the same Kafka partition, helping preserve ordering for that record.

### High-Level Architecture

```text
Client
  |
  | HTTP Request
  v
NGINX Ingress
  |
  v
Kubernetes Service
  |
  v
Spring Boot Producer Pod
  |
  | KafkaTemplate
  v
Kafka Broker Cluster
  |
  v
Kafka Topic
  |
  v
Partition Leader
  |
  v
ISR Replicas
  |
  v
Kafka Consumer
```

## Features

- REST APIs for creating and updating library events
- Kafka-based asynchronous event publishing
- `KafkaTemplate` for publishing events
- Event ID used as the Kafka message key
- Transactional Kafka publishing
- Asynchronous, synchronous, and transactional publishing strategies
- Kafka producer reliability configuration
- Multi-broker Kafka cluster configuration
- Spring Boot environment profiles
- Docker containerization
- Docker Compose for Kafka infrastructure
- Kubernetes deployment using Minikube
- Kubernetes Service and NGINX Ingress
- Spring Boot Actuator health monitoring
- Custom Kafka readiness health check
- Flyway database schema migrations
- Swagger/OpenAPI documentation
- Unit and integration tests

## Kafka Architecture

The Kafka infrastructure is configured as a multi-broker cluster with:

| Configuration | Value |
|---|---:|
| Brokers | 3 |
| Partitions | 3 |
| Replication Factor | 3 |
| `min.insync.replicas` | 2 |

Each Kafka partition has a leader responsible for handling writes while other brokers maintain replicas. If a leader becomes unavailable, Kafka can elect another in-sync replica as the new leader.

### Producer Reliability

The producer is configured with reliability-oriented settings including:

- `acks=all`
- Producer retries
- Idempotence
- `max.in.flight.requests.per.connection`
- Retry backoff
- Delivery timeout
- Request timeout

These settings improve resilience against temporary broker or network failures while helping maintain ordering and prevent duplicate records caused by producer retries.

## Kafka Transactions

The project demonstrates multiple Kafka publishing approaches:

- Asynchronous publishing
- Synchronous/blocking publishing
- Transactional publishing
- Synchronous transactional publishing
- Publishing multiple messages within a single Kafka transaction

The main service supports transactional Kafka publishing, allowing multiple Kafka operations to participate in a single transaction.

Failure and rollback scenarios are also covered through test cases.

## Spring Boot Profiles

Configuration is separated by environment using Spring Boot profiles:

```text
application-dev.yml
application-stage.yml
application-prod.yml
```

The same application artifact can therefore be deployed to different environments with environment-specific Kafka and application configuration.

Important configuration values can be supplied through environment variables, for example:

```text
SPRING_PROFILES_ACTIVE
SPRING_KAFKA_BOOTSTRAP_SERVERS
```

This keeps environment-specific configuration outside the application code.

## Docker

The application is packaged as an executable JAR and containerized using Docker.

The Docker image contains:

- Java runtime
- Application JAR
- Application dependencies

The application runs inside the container with:

```bash
java -jar app.jar
```

Docker Compose is also used to run the Kafka infrastructure with multiple brokers.

The project demonstrates practical Docker concepts including:

- Host ports
- Container ports
- Docker networks
- Container-to-container communication
- Kafka broker addresses
- Application-to-Kafka communication

## Kubernetes Deployment

The Spring Boot producer can be deployed to Kubernetes using **Minikube**.

Kubernetes resources include:

- Deployment
- Service
- Ingress
- ConfigMap

The Deployment manages the application container and defines configuration such as:

- Docker image
- Replica count
- Environment variables
- Container port
- CPU requests/limits
- Memory requests/limits

Example environment configuration:

```yaml
SPRING_PROFILES_ACTIVE=dev
SPRING_KAFKA_BOOTSTRAP_SERVERS=host.docker.internal:29092
```

## Kubernetes Networking

Pods are temporary resources and their IP addresses can change. A Kubernetes Service therefore provides a stable endpoint for the application.

An NGINX Ingress can expose the service through a hostname such as:

```text
library-producer.local
```

Request flow:

```text
Browser
   |
   v
NGINX Ingress
   |
   v
Kubernetes Service
   |
   v
Spring Boot Pod
   |
   v
Kafka
```

This demonstrates the roles of Kubernetes Pods, Services, Ingress, container ports, and externally accessible endpoints.

## Health Monitoring

The application integrates **Spring Boot Actuator** and includes a custom Kafka readiness health indicator.

The readiness check verifies whether the application can communicate with Kafka.

This can be used by Kubernetes to determine whether a Pod is ready to receive application traffic.

## Database Schema Management

**Flyway** is used for version-controlled database schema management.

Database changes are represented as migration scripts, for example:

```text
V1__create_library_events_table.sql
V2__add_book_column.sql
```

Flyway executes migrations in version order and tracks which migrations have already been applied.

## API Documentation

The project includes **Swagger/OpenAPI** documentation for the REST APIs.

The OpenAPI specification is maintained as part of the project and can be used to understand and test the available endpoints.

## Project Structure

```text
library-events-producer-v2/
├── .github/
├── docs/
├── gradle/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/learnkafka/
│   │   │       ├── config/
│   │   │       ├── controller/
│   │   │       ├── domain/
│   │   │       ├── exception/
│   │   │       ├── health/
│   │   │       ├── producer/
│   │   │       └── service/
│   │   └── resources/
│   └── test/
│       └── java/
├── Dockerfile
├── build.gradle
├── compose.yaml
├── gradlew
├── settings.gradle
└── Kubernetes YAML manifests
```

## Technology Stack

| Technology | Purpose |
|---|---|
| Java | Application development |
| Spring Boot | REST API and application framework |
| Spring Kafka | Kafka integration |
| Apache Kafka | Event streaming and messaging |
| Docker | Application containerization |
| Docker Compose | Local infrastructure |
| Kubernetes | Container orchestration |
| Minikube | Local Kubernetes environment |
| NGINX Ingress | External HTTP routing |
| Gradle | Build and dependency management |
| Flyway | Database migrations |
| Spring Boot Actuator | Application health monitoring |
| Swagger/OpenAPI | API documentation |
| Git/GitHub | Version control |

## Event Flow

A typical library event follows this flow:

```text
HTTP Request
     |
     v
Library Events Controller
     |
     v
Library Event Service
     |
     v
KafkaTemplate
     |
     v
Kafka Producer
     |
     v
Kafka Topic
     |
     v
Partition
     |
     +----> Leader
     |
     +----> ISR Replica
     |
     +----> ISR Replica
```

The event ID is used as the Kafka key, which allows events associated with the same library record to be routed consistently to the same partition.

## Reliability and Fault Tolerance

The project focuses on reliable event publishing through:

- Kafka replication
- In-sync replicas
- `acks=all`
- Idempotent producer configuration
- Producer retries
- Retry backoff
- Kafka transactions
- Transaction rollback handling
- Readiness health checks
- Kubernetes-managed application replicas

These mechanisms help the application continue operating reliably in the presence of temporary broker, network, or application failures.

## Learning Outcomes

The project demonstrates practical understanding of:

- Event-driven architecture
- Asynchronous communication
- Apache Kafka partitions and replication
- Kafka leader/follower architecture
- Producer reliability
- Kafka transactions
- Message ordering
- Idempotent message publishing
- Docker containerization
- Docker networking
- Kubernetes deployments
- Kubernetes Services and Ingress
- Environment-specific configuration
- Application health monitoring
- Database schema versioning
- REST API design
- Microservice architecture

## Running Locally

### Build the application

```bash
./gradlew build
```

### Run the application

```bash
./gradlew bootRun
```

### Build the Docker image

```bash
docker build -t library-events-producer .
```

### Start the Kafka infrastructure

```bash
docker compose up -d
```

### Kubernetes

Start Minikube:

```bash
minikube start
```

Apply the required Kubernetes resources:

```bash
kubectl apply -f library-events-producer-configmap.yaml
kubectl apply -f library-events-producer-deployment-v1.yaml
kubectl apply -f library-events-producer-service.yaml
kubectl apply -f library-events-producer-ingress.yaml
```

