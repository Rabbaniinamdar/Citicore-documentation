## TOPIC 1 — PROJECT OVERVIEW

## 1. What is CitiCore?

CitiCore is a microservices-based banking platform designed to demonstrate how core banking operations can be implemented using independently deployable Java and Spring Boot services.

The platform separates authentication, user management, account management, transaction processing, notifications, configuration, service discovery, and API gateway responsibilities into dedicated services. The application is containerized with Docker and deployed on AWS ECS/Fargate. Supporting infrastructure includes Amazon RDS for persistent MySQL storage, Kafka for asynchronous event communication, Redis for caching and rate limiting, Spring Cloud Config for centralized configuration, and Eureka/ECS Service Connect for service discovery and internal communication.

The project was developed with a practical production-oriented mindset rather than as a simple CRUD application. Important concerns addressed in the notes include database read/write separation, replication lag, transaction reliability, event-driven communication, security, container deployment, health checks, resilience, and CI/CD.

---

## 2. Problem Statement

### Concept

A banking application cannot treat account operations as ordinary CRUD operations. Operations such as account creation, deposits, withdrawals, balance inquiries, and fund transfers require reliable data handling, controlled access, consistency, and traceability.

As the number of users and transactions increases, a single monolithic application and single database can become difficult to scale and maintain. The platform therefore separates responsibilities into services and introduces infrastructure patterns for handling read traffic, asynchronous processing, caching, database growth, and deployment.

### CitiCore Context

CitiCore addresses these concerns by dividing the banking platform into independently deployable services. The documented architecture contains eight independently deployable services/components at the application level: Config Server, Eureka Server, API Gateway, Auth Service, User Service, Account Service, Transaction Service, and Notification Service.

The platform also separates critical database operations from read-heavy operations. For example, the Account Service uses the primary database for writes and consistency-sensitive reads while non-critical historical reads can be routed to a replica.

---

## 3. Project Goals

The main goals of CitiCore are:

1. **Build a modular banking platform** using Spring Boot microservices.

2. **Separate service responsibilities** so that authentication, users, accounts, transactions, and notifications can evolve independently.

3. **Support reliable banking operations** such as account creation, deposits, withdrawals, balances, statements, and transaction processing.

4. **Use synchronous communication where an immediate response is required**, such as service-to-service queries.

5. **Use asynchronous communication for event-driven processing**, with Kafka handling application events.

6. **Improve database scalability** using MySQL primary/replica architecture and explicit read/write routing.

7. **Handle large transaction-oriented tables** using monthly MySQL partitioning.

8. **Improve performance** through Redis caching and separate HikariCP connection pools for primary and replica databases.

9. **Secure database communication** using TLS and certificate validation.

10. **Containerize and deploy the platform** using Docker, Amazon ECR, ECS/Fargate, ALB, and related AWS infrastructure.

11. **Automate application delivery** through Jenkins CI/CD with Git-based versioning and ECS deployments.

---

## 4. Business Capabilities

The platform covers the following banking capabilities.

### Customer and Identity Management

The Auth Service handles authentication and JWT-based access, while the User Service manages user information and KYC-related functionality.

### Account Management

The Account Service is responsible for creating and managing accounts, maintaining account balances and status, and providing account-related queries. The documented implementation includes account types such as SAVINGS and CURRENT.

### Banking Operations

The platform supports core account operations such as:

* Account creation
* Deposits
* Withdrawals
* Balance inquiries
* Statements
* Transaction history
* Fund-transfer-related processing

Withdrawals are documented with locking considerations, while transaction references are designed to support idempotent processing.

### Event-Driven Processing

Kafka is used to communicate events asynchronously between services. The project also uses the Transactional Outbox approach so that important database changes and their corresponding events can be coordinated reliably.

### Notifications

The Notification Service consumes relevant events and handles notification delivery. The notes mention email notification integration using SendGrid/Spring Mail.

---

## 5. Technology Stack

| Category              | Technology                  | Purpose                                  |
| --------------------- | --------------------------- | ---------------------------------------- |
| Language              | Java 17                     | Primary development language             |
| Application Framework | Spring Boot 3.2.4           | Microservice development                 |
| Spring Cloud          | Spring Cloud 2023.0.3       | Distributed-system capabilities          |
| API Gateway           | Spring Cloud Gateway        | Routing and cross-cutting concerns       |
| Security              | Spring Security + JWT       | Authentication and authorization         |
| Service Discovery     | Eureka                      | Service registry/discovery               |
| Internal Discovery    | AWS ECS Service Connect     | Internal ECS service communication       |
| Configuration         | Spring Cloud Config         | Centralized configuration                |
| Database              | MySQL 8.0                   | Persistent relational storage            |
| ORM                   | Spring Data JPA / Hibernate | Database access                          |
| Messaging             | Apache Kafka / Spring Kafka | Asynchronous event communication         |
| Cache                 | Redis                       | Caching and distributed rate limiting    |
| HTTP Client           | OpenFeign                   | Service-to-service communication         |
| Resilience            | Resilience4j                | Circuit breaker, retry and resilience    |
| Connection Pool       | HikariCP                    | Database connection management           |
| Containerization      | Docker                      | Application packaging                    |
| Image Registry        | Amazon ECR                  | Docker image storage                     |
| Container Platform    | Amazon ECS/Fargate          | Application deployment                   |
| Load Balancing        | Application Load Balancer   | External traffic distribution            |
| Database Hosting      | Amazon RDS                  | Managed MySQL                            |
| Monitoring            | CloudWatch / Micrometer     | Health and operational monitoring        |
| CI/CD                 | Jenkins                     | Automated build and deployment           |
| Source Control        | Git/GitHub                  | Source code and configuration management |

The notes consistently identify Java 17, Spring Boot 3.2.4, and Spring Cloud 2023.0.3 as the platform versions. Version alignment was specifically important because an earlier Eureka project contained incompatible/incorrect Spring versions.

---

## 6. Microservices

CitiCore separates the major business and infrastructure responsibilities into the following services.

### Config Server

Provides centralized configuration to the other services. Configuration is maintained separately from application images, with service-specific configuration loaded by the applications.

### Eureka Server

Acts as the service registry and provides service-discovery capabilities. Services can register themselves and discover other services through the registry.

For the ECS environment, the notes also use ECS Service Connect for internal service communication.

### API Gateway

Acts as the external entry point for backend APIs. It performs routing and centralizes cross-cutting concerns such as rate limiting, circuit breaking, retry handling, and request filtering.

### Auth Service

Responsible for authentication and JWT token generation/validation. It validates user credentials and provides the security identity used by subsequent requests.

### User Service

Responsible for user information and KYC-related functionality.

### Account Service

Responsible for account lifecycle and account-level banking operations, including account creation, deposits, withdrawals, balances, statements, and transaction history. It integrates with RDS, Redis, Kafka, Config Server, Eureka, and the application load-balancing layer.

### Transaction Service

Responsible for transaction processing and fund-transfer-related operations. It can communicate synchronously with the Account Service through OpenFeign and uses asynchronous event communication where appropriate.

### Notification Service

Consumes relevant application events and performs notification-related processing, including email delivery.

---

## 7. High-Level Technology Architecture

At a project level, the platform can be viewed as four major layers.

```text
                    External Clients
                           |
                           v
                    Application Load Balancer
                           |
                           v
                    API Gateway
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
      Auth Service     User Service     Account Service
                                             |
                                             v
                                      Transaction Service
                                             |
                           +----------------+----------------+
                           |                |                |
                           v                v                v
                         MySQL            Redis            Kafka
                           |
                    Primary / Replica
```

Supporting the application layer:

```text
                    CitiCore Platform
                           |
       +-------------------+-------------------+
       |                   |                   |
       v                   v                   v
 Config Server          Eureka          ECS Service Connect
       |                   |                   |
       +-------------------+-------------------+
                           |
                           v
                     AWS Infrastructure
                           |
       +-------------------+-------------------+
       |                   |                   |
       v                   v                   v
     RDS                 ECS/Fargate          ECR
       |                   |
       v                   v
    MySQL DB              Containers
```

This high-level representation intentionally does not explain the internal mechanics of Kafka, Redis, RDS replication, Eureka, ECS networking, or CI/CD. Those concepts will have one authoritative explanation in their respective later topics.

---

## 8. Core Design Principles

### 8.1 Separation of Responsibilities

Each major business capability is isolated in its own service. This allows services to be developed, tested, deployed, and scaled independently.

The Account Service, for example, owns account-related operations rather than allowing other services to directly manipulate its database.

### 8.2 Database Ownership

The architecture follows service-level database separation rather than treating the entire platform as one shared application database.

The notes explicitly identify the need to prevent incorrect cross-service database access. For example, the Transaction Service must connect to its transaction database rather than accidentally using the Auth database.

### 8.3 Synchronous + Asynchronous Communication

CitiCore does not use one communication mechanism for every interaction.

Synchronous communication is used when one service needs an immediate response, such as a Transaction Service query to the Account Service.

Asynchronous communication through Kafka is used for event-driven processing where the producer does not need to wait for every consumer to finish processing.

This combination allows the system to maintain synchronous request/response behavior where necessary while reducing coupling for event-driven workflows.

### 8.4 Strong vs Eventual Consistency

The platform explicitly distinguishes between reads that require current data and reads where a small replication delay is acceptable.

For example:

```text
Consistency-sensitive
        |
        +---- Balance
        +---- Withdrawal validation
        +---- Transfer validation
        |
        v
      PRIMARY


Eventual-consistency acceptable
        |
        +---- Statement history
        +---- Historical transactions
        +---- Reporting
        |
        v
      REPLICA
```

This distinction is important because the database replica uses asynchronous replication. The notes specifically identify replication lag as the reason that not every read can safely be routed to the replica.

### 8.5 Event-Driven Reliability

For important events, CitiCore uses the Transactional Outbox pattern. The business database update and creation of the corresponding outbox event occur within the same database transaction. A background publisher subsequently publishes the event to Kafka.

The detailed mechanics belong to the Transactional Outbox topic and are intentionally not repeated here.

### 8.6 Defense in Depth

Security is implemented at multiple layers rather than relying on a single mechanism.

Examples include:

* JWT-based authentication
* API-level authorization
* Separate database users
* Read-only database credentials
* TLS for database communication
* AWS security groups
* AWS IAM
* Secrets management

The notes also identify areas where the current deployment still needs hardening, such as moving ECS tasks into private subnets and completing migration of secrets into Secrets Manager.

---

## 9. Deployment Model

The documented deployment model uses AWS ECS/Fargate for container execution.

A simplified deployment relationship is:

```text
GitHub
   |
   v
Jenkins
   |
   +---- Build Java application
   |
   +---- Build Docker image
   |
   +---- Push image to ECR
   |
   v
ECS/Fargate
   |
   +---- Config Server
   +---- Eureka
   +---- API Gateway
   +---- Auth Service
   +---- User Service
   +---- Account Service
   +---- Transaction Service
   +---- Notification Service
```

The deployment notes define an infrastructure-first deployment sequence, followed by Config Server, Eureka, application services, and finally the API Gateway. The purpose is to ensure that service dependencies are available before dependent services start.

---

## 10. What Makes This Project Interview-Worthy?

The strongest part of CitiCore is not simply that it contains multiple Spring Boot services. The project demonstrates practical backend engineering decisions around distributed systems.

The key areas worth discussing in an interview are:

* Why microservices were used.
* How service responsibilities were separated.
* Why synchronous and asynchronous communication were both required.
* How database replication was used.
* Why all reads cannot safely go to a replica.
* How read/write routing was implemented.
* Why Redis was introduced.
* How Kafka events are made reliable through the Outbox pattern.
* How large transaction tables are partitioned.
* How database communication is secured with TLS.
* How services are containerized and deployed on ECS/Fargate.
* How ALB and security groups control network traffic.
* How Jenkins automates deployment.
* What real infrastructure and deployment problems were encountered.

These are stronger interview discussion points than simply listing technologies.

---

## 11. Real Project Experience Captured in the Notes

The original documentation contains several actual implementation problems and lessons. These should be preserved, but explained in detail only in the later Troubleshooting topic.

Examples include:

### ALB Security Group Misconfiguration

The ALB initially used the ECS security group instead of the dedicated ALB security group. This caused both external connectivity failures and unhealthy ECS targets even though the application itself was healthy. Changing the ALB to the correct security group resolved the problem.

### Docker/MySQL Configuration

MySQL ignored a configuration file because it was world-writable. Correcting the permissions allowed the MySQL configuration to be loaded. The notes also document confusion between Docker shell commands and commands executed inside the MySQL prompt.

### Spring Version Alignment

An incorrect Spring Boot/Spring Cloud version combination caused problems in the Eureka project. The project was aligned to Java 17, Spring Boot 3.2.4, and Spring Cloud 2023.0.3.

These experiences are valuable because they demonstrate actual debugging and engineering work rather than only theoretical knowledge.

---

## 12. Known Scope of the Current Implementation

The project documentation should distinguish between what is implemented and what is planned.

The current notes identify limitations including:

* ECS tasks currently running in public subnets.
* Single task per service in the documented deployment.
* No complete multi-AZ service redundancy.
* Some secrets still handled through environment variables.
* HTTPS/ACM configuration identified as a future improvement.
* Auto-scaling identified as a future improvement.
* More advanced monitoring with Prometheus/Grafana identified as future work.

Therefore, the project should **not** be described in an interview as having complete production-grade HA, fully private ECS networking, or fully automated disaster recovery unless those features were actually implemented later.

---

## 13. Interview Explanation — 30–60 Seconds

> "CitiCore is a microservices-based banking platform built using Java 17 and Spring Boot. I separated the system into services for authentication, user management, accounts, transactions, notifications, configuration, service discovery, and API gateway responsibilities.
>
> The services communicate both synchronously and asynchronously. OpenFeign is used when an immediate service response is required, while Kafka is used for event-driven communication. For reliable event publishing, I used the Transactional Outbox approach.
>
> On the database side, the Account Service uses MySQL with primary-replica replication and explicit read/write routing. Consistency-sensitive operations such as balance validation use the primary, while suitable historical reads can use the replica. Redis is used for caching, and large transaction-oriented tables use monthly partitioning.
>
> The application is containerized with Docker and deployed on AWS ECS/Fargate, with ECR for images and ALB for external traffic. Jenkins automates the build and deployment process. A major part of the project was also solving real deployment and networking issues, such as an ALB security-group misconfiguration that caused both health-check failures and external connectivity problems."

---

## 14. Likely Interview Follow-Up Questions

### Q1. Why did you choose microservices?

**Answer:**
The platform contains multiple independent business capabilities with different responsibilities and communication patterns. Separating them allows independent development and deployment and prevents the entire banking application from being tightly coupled to one deployable unit.

### Q2. Why use Kafka if you already have REST/Feign?

**Answer:**
Feign is appropriate when the caller needs an immediate response. Kafka is appropriate when processing can be asynchronous and multiple consumers may need to react to the same business event. CitiCore therefore uses both rather than forcing every interaction into one communication model.

### Q3. Why did you introduce a database replica?

**Answer:**
The primary database handles writes and consistency-sensitive operations, while suitable read-heavy operations can be served by the replica. This separates workloads and allows read traffic to scale without putting all of the load on the primary.

### Q4. Why can't every read use the replica?

**Answer:**
Because replication is asynchronous and the replica can temporarily lag behind the primary. A balance or transaction-validation query must see the latest state, so consistency-sensitive reads are routed to the primary. Historical or reporting reads can tolerate eventual consistency.

### Q5. Why use the Transactional Outbox pattern?

**Answer:**
It solves the dual-write problem between the database and Kafka. The business update and outbox event are persisted in the same database transaction. Kafka publishing then happens asynchronously, allowing failed publishing attempts to be retried instead of losing the event.

### Q6. What was one real production-like issue you faced?

**Answer:**
An ALB had the wrong security group. The application was healthy inside the ECS task, but external requests timed out and the target was marked unhealthy. I debugged layer by layer—from the application and container to ECS and ALB—and found that the ALB was using the ECS security group instead of the dedicated ALB security group.

### Q7. How would you describe the project in one sentence?

**Answer:**
"CitiCore is a Spring Boot microservices banking platform using Kafka, Redis, MySQL primary-replica architecture, Docker, and AWS ECS/Fargate, with a focus on reliability, scalability, security, and practical distributed-system patterns."

---

## 15. Topic-Level Summary

CitiCore can be summarized as:

```text
                    C I T I C O R E
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
   Microservices      Event Driven      AWS Cloud
        |                |                |
        v                v                v
 Auth/User/Account     Kafka          ECS/Fargate
 Transaction/Notify     |             RDS/ECR
        |               Outbox             |
        |                                  ALB
        v                                   |
   Spring Boot                         CloudWatch
        |
        +---- MySQL Primary/Replica
        |
        +---- Redis
        |
        +---- Config Server
        |
        +---- Eureka / Service Connect
```

The central engineering idea is to combine **service separation, reliable asynchronous communication, controlled database consistency, scalable data access, caching, containerization, and automated deployment** into one practical backend system.

---

## TOPIC 2 — COMPLETE SYSTEM ARCHITECTURE

## 1. What is the CitiCore System Architecture?

### Concept

System architecture describes how the different components of an application interact with each other. For a distributed banking application, the architecture must define not only which services exist, but also how requests enter the system, how services communicate, where data is stored, how asynchronous events flow, and how failures are handled.

CitiCore follows a microservices architecture in which business capabilities are separated into independently deployable services. The platform uses a combination of synchronous REST communication and asynchronous Kafka-based communication. AWS infrastructure provides container execution, networking, load balancing, persistent storage, and operational monitoring.

### CitiCore Implementation

At a high level, an external request enters through an Application Load Balancer and reaches the Spring Cloud API Gateway. The gateway routes the request to the appropriate backend service.

The main business services are:

* Auth Service
* User Service
* Account Service
* Transaction Service
* Notification Service

Supporting components provide centralized configuration, service discovery, messaging, caching, persistence, networking, and deployment.

The original architecture documentation identifies the main external path as:

```text
External Client
      |
      v
Gateway ALB :8080
      |
      v
API Gateway :8080
      |
      +---- /auth/**        → Auth Service :8081
      |
      +---- /users/**       → User Service :8082
      |
      +---- /accounts/**    → Account Service :8083
      |
      +---- /transaction/** → Transaction Service :8084
```

The gateway applies its configured cross-cutting controls and then forwards the request to the appropriate service.

---

# 2. Complete Component Architecture

The following diagram represents the consolidated CitiCore architecture.

```text
                              INTERNET
                                  |
                                  v
                    +--------------------------+
                    |    Application Load      |
                    |       Balancer           |
                    |        :8080             |
                    +------------+-------------+
                                 |
                                 v
                    +--------------------------+
                    |      API Gateway         |
                    |  Spring Cloud Gateway    |
                    |        :8080             |
                    +------------+-------------+
                                 |
             +-------------------+-------------------+
             |                   |                   |
             v                   v                   v
      +-------------+     +-------------+     +-------------+
      | Auth Service|     | User Service|     |Account      |
      |    :8081    |     |    :8082    |     |Service :8083|
      +-------------+     +-------------+     +------+------+
                                                     |
                                                     v
                                              +-------------+
                                              | Transaction |
                                              | Service     |
                                              |    :8084    |
                                              +------+------+
                                                     |
                         +---------------------------+
                         |
             +-----------+-----------+---------------+
             |                       |               |
             v                       v               v
      +-------------+         +-------------+   +-------------+
      | RDS MySQL   |         |    Redis    |   |    Kafka    |
      |Primary/     |         |   Cache /   |   |   Events    |
      |Replica      |         | Rate Limit  |   +-------------+
      +-------------+         +-------------+
```

The supporting platform components sit around these services:

```text
                         CitiCore Services
                                |
        +-----------------------+-----------------------+
        |                       |                       |
        v                       v                       v
+---------------+        +---------------+       +---------------+
| Config Server |        | Eureka Server |       | Service       |
| Central Config|        | Discovery     |       | Connect       |
+---------------+        +---------------+       +---------------+
                                |
                                v
                       AWS ECS/Fargate
                                |
              +-----------------+-----------------+
              |                 |                 |
              v                 v                 v
           ECR Images          ALB             CloudWatch
```

The project documentation describes Eureka for service discovery while also using ECS Service Connect for internal ECS service communication.

---

# 3. Architecture Layers

The system can be understood more easily by separating it into logical layers.

## 3.1 Client Layer

The client can be a web application, mobile application, Postman, or another API consumer.

The client does not directly communicate with individual backend services.

Instead:

```text
Client
   |
   v
ALB
   |
   v
API Gateway
   |
   v
Backend Service
```

This gives the client a single external entry point.

---

## 3.2 Gateway Layer

The API Gateway acts as the central entry point for the backend APIs.

Its responsibilities include:

* Request routing
* Path rewriting
* Authentication-related filtering
* Rate limiting
* Circuit breaking
* Retry handling
* Request filtering
* Forwarding requests to internal services

The gateway routes paths such as `/auth/**`, `/users/**`, `/accounts/**`, and `/transaction/**` to their respective services.

For example:

```text
Client
 |
 | POST /accounts/create
 |
 v
API Gateway
 |
 | rewrite
 |
 | POST /api/v1/accounts/create
 |
 v
Account Service
```

The internal URL structure is therefore hidden from the external client.

---

# 4. Business Service Layer

The business services contain the actual banking functionality.

```text
                    Backend Services
                           |
       +-------------------+-------------------+
       |                   |                   |
       v                   v                   v
     Auth                User              Account
       |                   |                   |
       |                   |                   v
       |                   |              Transaction
       |                   |                   |
       +-------------------+-------------------+
                           |
                           v
                     Notification
```

Each service has a specific responsibility.

### Auth Service

Handles authentication, credentials, JWT issuance, and email verification.

### User Service

Handles user profiles and KYC-related information.

### Account Service

Handles accounts, balances, deposits, withdrawals, statements, and account-related operations.

### Transaction Service

Handles transaction processing and fund-transfer orchestration.

### Notification Service

Handles event-driven notification processing, including email delivery.

The documented service dependencies show that Transaction Service can call Account Service using OpenFeign, while Account and Transaction services also participate in Kafka-based event communication.

---

# 5. Data Layer

CitiCore uses MySQL as its primary relational data store.

For the Account Service, the documented architecture separates the primary and replica database paths.

```text
                         Account Service
                               |
                 +-------------+-------------+
                 |                           |
                 v                           v
          Primary HikariCP             Replica HikariCP
                 |                           |
                 v                           v
           RDS Primary                RDS Replica
                 |                           |
                 +------ Replication --------+
                         asynchronous
```

The primary is the source of truth for writes and consistency-sensitive operations. The replica is used for suitable read-heavy operations.

The Account Service documentation specifically describes separate HikariCP pools for primary and replica access.

The detailed routing strategy, replication behavior, HikariCP configuration, and replica lag are intentionally covered later in **Database Architecture**.

---

# 6. Synchronous Communication

### Concept

Synchronous communication means the caller waits for the downstream service to return a response.

CitiCore uses synchronous service-to-service communication when an immediate answer is required.

### CitiCore Implementation

The documented example is Transaction Service communicating with Account Service through OpenFeign.

```text
Transaction Service
        |
        | OpenFeign request
        v
Account Service
        |
        | Query / validation
        v
Account Database
        |
        v
Response
        |
        v
Transaction Service
```

For example, during transaction processing, Transaction Service may need account-related information before continuing.

The notes identify OpenFeign as the declarative REST client used for this service-to-service communication.

### Why synchronous communication?

It is useful when:

* The caller requires an immediate response.
* The result determines whether processing can continue.
* The operation is request/response oriented.
* The downstream result is required before returning a response to the client.

### Important Trade-off

The major disadvantage is runtime coupling.

```text
Transaction Service
        |
        v
Account Service DOWN
        |
        v
Transaction request affected
```

Therefore, timeouts, retries, and circuit breakers are important for synchronous calls.

---

# 7. Asynchronous Communication

### Concept

Asynchronous communication allows a service to publish an event without waiting for every consumer to complete processing.

CitiCore uses Kafka for this purpose.

```text
Producer Service
       |
       v
    Kafka
       |
       +--------> Consumer A
       |
       +--------> Consumer B
       |
       +--------> Consumer C
```

The Kafka architecture documented for CitiCore is used for:

* Asynchronous messaging
* Service decoupling
* Event replay
* Saga coordination
* Eventual consistency

The detailed Kafka architecture and individual topics will be explained once in **Topic 6 — Kafka**.

---

# 8. Synchronous + Asynchronous Communication Together

One of the important architecture decisions in CitiCore is that the system does not force every interaction into either REST or Kafka.

Instead, both models are used based on the requirement.

```text
                    Transaction Request
                           |
                           v
                   Transaction Service
                           |
                 +---------+---------+
                 |                   |
          Immediate data        Business event
                 |                   |
                 v                   v
          OpenFeign            Kafka / Outbox
                 |                   |
                 v                   v
          Account Service       Consumers
```

This provides a balance between immediate request processing and loose coupling for event-driven operations.

### Interview Explanation

> "I use synchronous communication when the calling service needs an immediate response, such as account validation. For asynchronous workflows, I publish domain events through Kafka so consumers can process them independently. This reduces unnecessary coupling while keeping request-response operations straightforward."

---

# 9. Complete Request Flow

Consider a client requesting an account operation.

```text
Step 1
Client
  |
  | HTTP request
  v

Step 2
Application Load Balancer
  |
  | forwards traffic
  v

Step 3
API Gateway
  |
  | identifies route
  | applies gateway policies
  v

Step 4
Account Service
  |
  | validates request
  | executes business logic
  v

Step 5
Database / Redis / Kafka
  |
  | operation-specific processing
  v

Step 6
Account Service
  |
  | creates response
  v

Step 7
API Gateway
  |
  v

Step 8
ALB
  |
  v

Client
```

The documented logical request path follows the same overall structure: external client → ALB → API Gateway → internal service → supporting infrastructure → response back through the gateway and ALB.

---

# 10. Service-to-Service Request Flow

Internal service communication is different from external client communication.

For example:

```text
Transaction Service
        |
        | OpenFeign
        v
Service Connect DNS
        |
        v
Account Service
        |
        v
Account Service Task
```

In the ECS environment, Service Connect provides internal DNS-based communication rather than requiring services to hardcode changing ECS task IP addresses.

The documented example resolves the Account Service through a Service Connect DNS name and routes the request internally within the VPC.

### Why this matters

ECS task IP addresses can change when tasks restart or are redeployed.

Therefore:

```text
Hardcoded Task IP
      |
      X
Unstable


Service Connect DNS
      |
      v
Stable Service Identity
```

This makes internal communication more suitable for dynamically managed containers.

---

# 11. Configuration Flow

CitiCore separates application configuration from application code and container images using Spring Cloud Config.

```text
              Git Configuration Repository
                         |
                         v
                  Config Server
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
      Auth             Account        Transaction
      Service           Service          Service
```

The configuration includes items such as:

* Database configuration
* Kafka configuration
* Redis configuration
* Eureka configuration
* Application-specific settings

Secrets are intended to be kept separately rather than committed directly into the configuration repository.

The complete Config Server implementation will be covered under **Topic 3 — Microservices**, specifically its dedicated service section.

---

# 12. Service Discovery and Internal Networking

There are two discovery-related mechanisms documented in the project.

### Eureka

Eureka acts as a service registry.

```text
             Eureka Server
                   |
       +-----------+-----------+
       |           |           |
       v           v           v
     Auth        User       Account
```

Services register themselves and can discover peer services through Eureka.

### ECS Service Connect

For ECS deployment, Service Connect provides internal service DNS and networking.

```text
Transaction Service
        |
        v
Service Connect DNS
        |
        v
Account Service
```

The notes specifically describe Service Connect as an override for ECS service communication so that internal calls do not depend entirely on Eureka.

The complete comparison and behavior of Eureka will be covered later in **Topic 4 — Service Discovery**.

---

# 13. Event Flow

The event-driven architecture can be represented as:

```text
                    Business Operation
                           |
                           v
                     Source Service
                           |
                           v
                  Transactional Outbox
                           |
                           v
                         Kafka
                           |
              +------------+------------+
              |                         |
              v                         v
       Transaction Service       Notification Service
              |
              v
       Further business event
```

For transaction processing, the documented Kafka architecture contains events for debit, credit, success, failure, and reversal operations.

This allows transaction processing to be coordinated through events rather than requiring every participating service to remain synchronously connected for the entire workflow.

The detailed transaction saga and topic-level design are intentionally deferred to the Kafka and Transactional Outbox topics.

---

# 14. Failure Handling Architecture

Failure handling is distributed across multiple layers.

```text
                     Request
                        |
                        v
                 +--------------+
                 | API Gateway  |
                 +--------------+
                   |    |    |
                   |    |    +---- Rate Limit
                   |    +--------- Retry
                   +-------------- Circuit Breaker
                        |
                        v
                  Backend Service
                        |
              +---------+---------+
              |                   |
              v                   v
           Database             Kafka
              |                   |
              v                   v
       DB error handling      Retry / DLT
```

### Gateway-level failures

The gateway can use:

* Rate limiting
* Retry
* Circuit breaker
* Timeout handling

These mechanisms address different failure types rather than performing the same function.

### Database failures

Database access is separated between primary and replica paths. The Account Service documentation also includes replica health monitoring and fallback behavior.

```text
Read Request
     |
     v
Replica Available?
   /       \
 YES        NO
 |           |
 v           v
Replica     Primary
```

### Kafka failures

Kafka consumers can retry failed processing and eventually send messages to a Dead Letter Topic when processing continues to fail. The documented configuration uses multiple attempts with backoff before DLT handling.

### Infrastructure failures

AWS health checks, ALB target health, ECS task health, and CloudWatch monitoring provide additional operational visibility.

---

# 15. Failure Isolation

A major benefit of the architecture is that different failure types can be isolated.

For example:

```text
Scenario: User Service unavailable

Client
  |
  v
API Gateway
  |
  +----> User Service
  |          X
  |
  +----> Account Service
  |
  +----> Transaction Service
```

The gateway's circuit breaker can prevent repeated calls to an unhealthy downstream service rather than allowing failures to continuously propagate.

The project documentation specifically describes circuit breaking as a mechanism to prevent cascading failures.

Similarly, asynchronous Kafka processing means a consumer failure does not necessarily require the producer to remain blocked waiting for that consumer.

---

# 16. Deployment Architecture

The application is packaged into Docker images and deployed as ECS/Fargate services.

```text
                         GitHub
                           |
                           v
                        Jenkins
                           |
                     Build & Test
                           |
                           v
                       Docker
                           |
                           v
                         ECR
                           |
                           v
                    ECS / Fargate
                           |
       +-------------------+-------------------+
       |                   |                   |
       v                   v                   v
   API Gateway        Account Service     Transaction Service
       |                   |                   |
       +-------------------+-------------------+
                           |
                           v
                       AWS RDS
```

The documented deployment flow builds a Java application, creates a Docker image tagged with the Git SHA, pushes the image to ECR, updates the ECS task definition, and performs a rolling deployment.

The complete deployment mechanics are covered later under **AWS Container Deployment** and **Jenkins CI/CD**.

---

# 17. Network Flow

The high-level network flow is:

```text
                         INTERNET
                            |
                            v
                       Gateway ALB
                            |
                            v
                     API Gateway ECS
                            |
                 +----------+----------+
                 |          |          |
                 v          v          v
               Auth       User      Account
                                      |
                                      v
                                 Transaction
                                      |
                 +--------------------+------------------+
                 |                    |                  |
                 v                    v                  v
             RDS MySQL             Redis               Kafka
```

The important distinction is:

### External traffic

```text
Internet
   |
   v
ALB
   |
   v
API Gateway
```

### Internal service traffic

```text
Service
   |
   v
Service Connect
   |
   v
Another Service
```

### Database traffic

```text
Service
   |
   +----> Primary RDS
   |
   +----> Replica RDS
```

### Event traffic

```text
Service
   |
   v
Kafka
   |
   v
Consumers
```

---

# 18. End-to-End Architecture Example — Fund Transfer

A simplified fund-transfer architecture looks like this:

```text
Client
  |
  | POST /transaction/transfer
  v
ALB
  |
  v
API Gateway
  |
  v
Transaction Service
  |
  | Validate account information
  |
  +------ OpenFeign ------> Account Service
  |                              |
  |                              v
  |                           Primary DB
  |
  | Transaction processing
  |
  v
Transaction DB
  |
  v
Transactional Outbox
  |
  v
Kafka
  |
  +------> Account Service
  |
  +------> Notification Service
```

The actual transaction flow is more detailed and involves debit, credit, success/failure, and reversal events. Those mechanics are intentionally not duplicated here and will be explained in the Kafka and End-to-End Business Flows sections.

---

# 19. Architecture Decision Summary

The major architecture decisions can be summarized as follows.

| Decision              | Reason                                                               |
| --------------------- | -------------------------------------------------------------------- |
| Microservices         | Separate business responsibilities and enable independent deployment |
| API Gateway           | Single external entry point and centralized cross-cutting controls   |
| OpenFeign             | Immediate synchronous service-to-service communication               |
| Kafka                 | Asynchronous, decoupled event processing                             |
| Transactional Outbox  | Reliable database-to-event publishing                                |
| MySQL Primary/Replica | Separate write and suitable read workloads                           |
| Redis                 | Caching and gateway rate-limiting support                            |
| Eureka                | Service registration/discovery                                       |
| ECS Service Connect   | Stable internal ECS service communication                            |
| Docker                | Consistent application packaging                                     |
| ECS/Fargate           | Managed container execution                                          |
| ECR                   | Central Docker image registry                                        |
| ALB                   | External traffic distribution and health checking                    |
| Jenkins               | Automated build and deployment                                       |

These decisions should not be interpreted as universally optimal. They were selected according to the requirements and implementation of CitiCore.

---

# 20. Architecture Trade-offs

## Advantages

The architecture provides clear service boundaries, independent deployment, asynchronous event processing, scalable read handling, centralized gateway controls, and container-based deployment.

It also provides multiple layers for handling failures and makes it possible to introduce specialized scaling strategies for different components.

## Disadvantages

The architecture is more complex than a monolith.

There are additional operational concerns involving:

* Network configuration
* Service discovery
* Distributed debugging
* Kafka operations
* Database replication
* Configuration management
* Container orchestration
* Deployment coordination
* Distributed failure handling

For a small application, this level of infrastructure may be unnecessary.

For CitiCore, the architecture is useful because the project is specifically intended to demonstrate distributed backend and system-design concepts.

---

# 21. Real Architecture Issue

### Problem

During deployment, the application itself was healthy, but external requests were timing out and the ALB target was unhealthy.

### Investigation

The issue was isolated layer by layer:

```text
Application
    ↓
localhost:8083       → WORKING

ECS task networking
    ↓
10.0.2.132:8083      → WORKING

ALB
    ↓
Target unhealthy      → PROBLEM

External request
    ↓
Timeout               → SYMPTOM
```

The investigation found that the ALB was using the ECS security group instead of the dedicated ALB security group.

The ECS security group had an inbound rule allowing traffic from the ALB security group, but the ALB itself was not associated with that expected security group.

### Solution

The ALB security group association was corrected.

```text
Before:

Internet
   |
   v
ALB [ECS SG]
   |
   X
ECS expects traffic from [ALB SG]


After:

Internet
   |
   v
ALB [ALB SG]
   |
   v
ECS [ECS SG]
```

After the correction, the target became healthy and external requests succeeded.

### Lesson

A networking problem can appear as an application problem.

The correct debugging approach was therefore:

```text
Application
    ↓
Container
    ↓
ECS
    ↓
Security Groups
    ↓
ALB
    ↓
External Client
```

This layer-by-layer approach is an important practical lesson for distributed systems.

---

# 22. Important Architecture Principles for Interviews

When explaining CitiCore, focus on the reasoning behind the architecture rather than simply listing technologies.

A strong explanation should connect each technology to a problem:

```text
Problem
   |
   v
Architectural Decision
   |
   v
Technology
   |
   v
Benefit
```

Examples:

```text
Multiple business responsibilities
        ↓
Microservices
        ↓
Spring Boot
        ↓
Independent services


Need central external entry point
        ↓
API Gateway
        ↓
Spring Cloud Gateway
        ↓
Central routing + resilience


Need asynchronous processing
        ↓
Event-driven architecture
        ↓
Kafka
        ↓
Loose coupling


Database read pressure
        ↓
Read/write separation
        ↓
Primary + Replica
        ↓
Scale suitable read traffic


Database + Kafka dual-write problem
        ↓
Transactional Outbox
        ↓
DB transaction + Outbox
        ↓
Reliable event publishing
```

---

# 23. How I Would Explain the Architecture in an Interview

> "CitiCore is designed as a Spring Boot microservices-based banking platform. External clients access the system through an Application Load Balancer and a Spring Cloud API Gateway. The gateway handles routing and common concerns such as rate limiting, retries, and circuit breaking.
>
> The main business services are Auth, User, Account, Transaction, and Notification. For synchronous interactions where an immediate response is required, services use REST-based communication through OpenFeign. For asynchronous processing, the platform uses Kafka and the Transactional Outbox pattern.
>
> The Account Service uses MySQL with primary-replica architecture. Writes and consistency-sensitive reads go to the primary, while suitable read-heavy operations can use the replica. Redis provides caching, while ECS Service Connect provides internal service communication in the containerized AWS environment.
>
> The services are packaged using Docker, stored in ECR, and deployed on ECS/Fargate. Jenkins automates the build and deployment process. The architecture therefore combines synchronous APIs, asynchronous events, database scaling, caching, resilience patterns, and AWS container infrastructure."

---

# 24. Interview Follow-Up Questions

### Q1. Why do you have both ALB and API Gateway?

**Answer:**
The ALB operates at the infrastructure/load-balancing layer and distributes traffic to the gateway service. The API Gateway operates at the application layer and handles API routing and gateway-level policies such as rate limiting, retries, and circuit breaking.

### Q2. Why not expose every microservice through the ALB?

**Answer:**
The gateway provides a controlled single entry point. Exposing every service externally increases the public attack surface and makes routing and cross-cutting policies harder to centralize.

### Q3. Why do you use both Eureka and ECS Service Connect?

**Answer:**
Eureka provides application-level service discovery, while ECS Service Connect provides AWS/ECS-native internal service connectivity and DNS. In the ECS environment, Service Connect is used for internal service communication so services do not depend on changing task IP addresses.

### Q4. When would you use Feign instead of Kafka?

**Answer:**
I use Feign when the caller needs an immediate response from another service. I use Kafka when the operation can be asynchronous, when the producer should be decoupled from consumers, or when multiple consumers need to react to an event.

### Q5. What happens if Account Service is unavailable?

**Answer:**
For synchronous requests, timeout, retry, and circuit-breaker mechanisms can prevent the caller from waiting indefinitely or repeatedly hitting the unhealthy service. For asynchronous processing, Kafka allows events to remain available for retry and further processing.

### Q6. What happens if Kafka is temporarily unavailable?

**Answer:**
The Transactional Outbox pattern prevents the business event from being lost because the event is first stored in the database as part of the business transaction. The publisher can retry Kafka publication later.

### Q7. Why not put everything into one service?

**Answer:**
A monolithic application would be simpler operationally, but CitiCore is designed to demonstrate independently deployable business capabilities and distributed-system patterns. Microservices allow individual components to evolve and scale independently, at the cost of additional operational complexity.

### Q8. What was the most important practical lesson from the deployment?

**Answer:**
I learned that distributed-system failures must be debugged layer by layer. In CitiCore, the application was healthy inside the ECS task, but the ALB target was unhealthy because of a security-group configuration error. Testing each layer independently helped isolate the real root cause.

---

# 25. Architecture at a Glance

```text
                         CLIENT
                           |
                           v
                    +-------------+
                    |     ALB     |
                    +------+------+
                           |
                           v
                    +-------------+
                    | API Gateway |
                    +------+------+
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
        AUTH             USER            ACCOUNT
                                             |
                                             v
                                        TRANSACTION
                                             |
                           +-----------------+----------------+
                           |                 |                |
                           v                 v                v
                        PRIMARY           REPLICA           REDIS
                           |
                           v
                          RDS

                            +----------------+
                            |     KAFKA      |
                            +--------+-------+
                                     |
                       +-------------+-------------+
                       |                           |
                       v                           v
                Transaction                  Notification
                   Service                      Service

Supporting:
    Config Server
    Eureka
    ECS Service Connect
    Docker
    ECR
    ECS/Fargate
    CloudWatch
    Jenkins
```

The architecture's core flow is therefore:

**Client → ALB → API Gateway → Microservice → Database / Redis / Kafka → Response or Event Consumers.**

---
## TOPIC 3 - MICROSERVICES
## 3.1 Microservices Overview

## What is it?

A microservice is an independently deployable application component responsible for a specific business capability or infrastructure responsibility.

Instead of putting authentication, users, accounts, transactions, and notifications into one large application, CitiCore separates these responsibilities into individual services.

```text
                         CitiCore
                            |
        +-------------------+-------------------+
        |                   |                   |
        v                   v                   v
 Infrastructure        Business Services    Supporting Services
        |                   |                   |
 Config / Eureka       Auth / User / Account   Kafka / Redis
 Gateway               Transaction / Notify
```

The important point is that **service boundaries are based on responsibility**, not simply on creating many small applications.

---

# 3.2 CitiCore Service Landscape

The documented CitiCore application contains the following eight service-level components:

| Service              | Port | Primary Responsibility                    |
| -------------------- | ---: | ----------------------------------------- |
| Config Server        | 8888 | Centralized configuration                 |
| Eureka Server        | 8761 | Service registry/discovery                |
| API Gateway          | 8080 | External API entry point and routing      |
| Auth Service         | 8081 | Authentication and JWT                    |
| User Service         | 8082 | User profiles and KYC                     |
| Account Service      | 8083 | Accounts, balances and account operations |
| Transaction Service  | 8084 | Fund-transfer orchestration               |
| Notification Service |    — | Event-driven notifications                |

The project notes also document ECS Service Connect for internal ECS communication.

---

# 3.3 Service Dependency View

A simplified dependency relationship is:

```text
                         Config Server
                              |
                              v
                         All Services
                              |
                              v
                         Eureka Server
                              |
          +-------------------+-------------------+
          |                   |                   |
          v                   v                   v
     Auth Service        User Service        Account Service
          |                   |                   |
          |                   |                   v
          |                   |             Transaction Service
          |                   |                   |
          +-------------------+-------------------+
                              |
                              v
                         Kafka Events
                              |
                              v
                    Notification Service
```

A more important business relationship is:

```text
Client
  |
  v
API Gateway
  |
  +----> Auth Service
  |
  +----> User Service
  |
  +----> Account Service
  |
  +----> Transaction Service
                         |
                         | OpenFeign
                         v
                   Account Service
```

The Transaction Service also participates in Kafka-based transaction processing.

---

# 3.4 Config Server

## Responsibility

The Config Server provides centralized configuration for the microservices.

Instead of embedding every service's configuration inside its Docker image, configuration is maintained in a Git-backed repository and fetched by services.

The documented configuration repository contains shared and service-specific files such as:

```text
config-repo/
├── application.yml
├── auth-service.yml
├── user-service.yml
├── account-service.yml
├── transaction-service.yml
├── apigateway-service.yml
└── ...
```

The notes specify configuration such as database URLs, Kafka settings, Redis connection information, Eureka URLs, application settings, and feature flags.

## How it works

```text
GitHub Configuration Repository
              |
              v
       Config Server :8888
              |
      +-------+-------+-------+
      |       |       |       |
      v       v       v       v
     Auth    User   Account  Transaction
```

Services fetch configuration during startup using Spring Cloud Config.

The documented setup also supports refresh through `/actuator/refresh`.

## Security

The configuration repository should contain configuration, not secrets.

```text
Git
 |
 +-- Application configuration
 +-- Service URLs
 +-- Timeout settings
 +-- Feature flags

AWS Secrets Manager
 |
 +-- Database passwords
 +-- JWT secrets
 +-- API keys
 +-- Sensitive credentials
```

The notes explicitly state that passwords, API keys, private keys, OAuth tokens, and AWS credentials should not be committed to Git.

## Important Implementation Detail

A service can import configuration using:

```yaml
spring:
  config:
    import: optional:configserver:http://<config-server>
```

The actual project configuration uses this approach for the API Gateway and other services.

### Interview Explanation

> "I used Spring Cloud Config Server to centralize service configuration. The configuration is Git-backed, while sensitive values are kept separately. This avoids configuration duplication and allows service configuration to be version controlled."

---

# 3.5 Eureka Server

## Responsibility

Eureka acts as the service registry for CitiCore.

Services register their instances with Eureka, and clients can discover service instances dynamically rather than relying on hardcoded IP addresses.

```text
                    Eureka :8761
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
       AUTH           ACCOUNT       TRANSACTION
       :8081           :8083           :8084
```

The documented Eureka configuration runs a single Eureka instance:

```yaml
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false

server:
  port: 8761
```

The notes explicitly identify this as a single Eureka deployment rather than a clustered Eureka setup.

## Why it exists in CitiCore

AWS ECS/Fargate tasks can be replaced and receive different private IP addresses.

Without discovery:

```text
Account Service
     |
     v
10.0.1.75:8083
     |
   Task replaced
     |
     v
10.0.1.100:8083
     |
     X
Old address invalid
```

With service discovery:

```text
Transaction Service
       |
       v
"Where is Account Service?"
       |
       v
Eureka
       |
       v
Current Account Service instance
```

The project notes specifically describe dynamic registration and discovery for changing ECS task addresses.

## Eureka + ECS Service Connect

CitiCore does not rely on Eureka alone for ECS internal communication.

The documented ECS implementation uses Service Connect DNS for important internal paths.

```text
Transaction Service
        |
        v
Service Connect DNS
        |
        v
Account Service
```

For example:

```text
eureka-server-8761-tcp.citicore:8761
```

is used as the Service Connect DNS name for Eureka.

This allows ECS services to communicate through stable service names rather than changing task IPs.

The detailed service-discovery comparison belongs to **Topic 4 — Service Discovery**.

---

# 3.6 API Gateway

## Responsibility

The API Gateway is the single external entry point into the CitiCore backend.

```text
Internet
   |
   v
Gateway ALB :8080
   |
   v
API Gateway :8080
   |
   +----> Auth
   +----> User
   +----> Account
   +----> Transaction
```

Without a gateway, the client would have to know the endpoint of every microservice.

The documented architecture specifically introduced the gateway to provide a single endpoint, centralized policies, reduced external exposure, and easier API evolution.

## Routes

The documented routes are:

```text
/auth/**         → Auth Service :8081
/users/**        → User Service :8082
/accounts/**     → Account Service :8083
/transaction/**  → Transaction Service :8084
```

For example:

```text
Client
  |
  | POST /accounts/create
  v
API Gateway
  |
  | POST /api/v1/accounts/create
  v
Account Service
```

The gateway therefore separates the external API structure from the internal service API structure.

## Gateway Controls

The implementation documents:

### Rate Limiting

```text
2 requests / second / IP
Burst capacity: 4
Storage: Redis
```

### Circuit Breaker

```text
Failure threshold: 50%
Sliding window: 10 requests
Open-state timeout: 10 seconds
Fallback: /fallback/service
```

### Retry

```text
Retries: 1
Statuses: 503, 504, 500
TimeoutException: retry
Backoff: exponential
```

These values are the documented project configuration, not generic recommended values.

The detailed gateway architecture and resilience behavior will be explained once under the dedicated **API Gateway** topic.

---

# 3.7 Auth Service

## Responsibility

The Auth Service is responsible for:

* User authentication
* JWT token issuance
* Email verification
* Security-related operations

The documented service runs on port `8081`.

## Main APIs

```text
POST /api/v1/auth/register
POST /api/v1/auth/login
POST /api/v1/auth/verify-email
POST /api/v1/auth/refresh-token
```

Another documented API set includes:

```text
POST /api/v1/auth/refresh
POST /api/v1/auth/validate
POST /api/v1/auth/logout
```

These variations appear in different sections of the original notes, so they should not be silently merged into one definitive API list without verifying the final source code.

The common responsibility is clear: registration, login, token validation/refresh, logout, and email verification.

## Authentication Flow

```text
User
 |
 | username/password
 v
Auth Service
 |
 | validate credentials
 v
JWT generated
 |
 v
Client
 |
 | Authorization: Bearer <JWT>
 v
API Gateway
 |
 | validate JWT
 v
Backend Service
```

The notes describe JWT information containing user identity and roles and explain propagation of authenticated user information toward backend services.

## Password Security

Passwords are hashed using BCrypt rather than stored as plaintext.

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

This is specifically documented as part of the Auth Service security implementation.

## Database

The Auth Service has its own authentication-related persistence rather than sharing the Account Service database.

The documented configuration uses a dedicated authentication database:

```yaml
spring:
  datasource:
    url: jdbc:mysql://${AUTH_DB_HOST}:3306/citicore_authdb
```

The same configuration references the JWT secret and email configuration through externalized values.

## Important Real Issue

The registration flow exposed an important distributed-system problem.

A registration request could return a `504` even though the user may already have been created and an OTP may already have been generated or published.

```text
Registration Request
       |
       v
Auth Service
       |
       +---- User created
       |
       +---- OTP generated
       |
       +---- Kafka operation
       |
       X
     504
```

The notes explicitly warn that blindly retrying such a request could create duplicate users.

The lesson was to check the actual state before retrying and introduce idempotency where appropriate.

### Interview Explanation

> "The Auth Service handles registration, authentication, JWT generation and email verification. Passwords are BCrypt-hashed, and authenticated requests carry JWT information through the gateway to downstream services. One practical issue I encountered was a registration timeout where the business operation could already have partially completed, so blindly retrying could create duplicate state."

---

# 3.8 User Service

## Responsibility

The User Service manages:

* User profiles
* User details
* KYC information
* KYC-related operations

The documented service runs on port `8082`.

## User Flow

```text
User Registration
       |
       v
Auth Service
       |
       v
User Profile
       |
       v
User Service
       |
       +---- KYC document
       |
       v
S3
       |
       v
KYC status
```

The notes describe users registering through Auth Service, followed by profile storage and KYC processing. KYC documents are stored in S3 and their status is tracked.

## KYC Event Processing

A KYC status change can produce an asynchronous event.

```text
User Service
     |
     | KycEvent
     v
Kafka
     |
     v
Notification Service
     |
     v
Email
```

The documented event topic is:

```text
kyc-topic
```

The producer creates a `KycEvent` containing user ID, email, status, and timestamp and publishes it through `KafkaTemplate`.

## Kafka Producer Configuration

The notes document:

```yaml
spring:
  kafka:
    producer:
      acks: all
      enable.idempotence: true
      retries: 10
      retry.backoff.ms: 1000
```

This configuration is intended to improve producer reliability and reduce duplicate publication behavior.

## Database and Storage

The documented KYC flow separates the storage responsibilities:

```text
KYC Document
     |
     v
S3

User/KYC Metadata
     |
     v
MySQL
```

This keeps object/document storage separate from relational user metadata.

### Interview Explanation

> "The User Service owns user profiles and KYC-related functionality. KYC documents are stored in S3, while user and KYC metadata is maintained by the service. When KYC status changes, the service can publish a Kafka event so the Notification Service can process the notification asynchronously."

---

# 3.9 Account Service

## Responsibility

The Account Service is one of the core business services in CitiCore.

Its responsibilities include:

* Account creation
* Account metadata
* Balance management
* Deposits
* Withdrawals
* Account inquiries
* Statements
* Transaction history

The documented service runs on port `8083`.

## Account Types

The notes identify account types including:

```text
SAVINGS
CURRENT
```

The Account Service maintains the account and its balance.

## Main APIs

```text
POST /api/v1/accounts/create
GET  /api/v1/accounts/my-accounts
GET  /api/v1/accounts/balance/{accountNo}
POST /api/v1/accounts/deposit
POST /api/v1/accounts/withdraw
```

These are documented as account creation, account listing, balance inquiry, deposit, and withdrawal operations.

## Architecture

```text
                 Account Service :8083
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
   RDS Primary       RDS Replica        Redis
   Read/Write         Read Only          Cache
        |
        v
      Kafka
```

The service also integrates with Config Server and Eureka.

## Read and Write Responsibilities

The documented implementation uses:

```text
Read
 |
 +-- Redis first
 |
 +-- suitable reads → Replica
 |
 +-- consistency-sensitive read → Primary


Write
 |
 +-- Primary
 |
 +-- Update database
 |
 +-- Invalidate cache
```

This is particularly important for balances because replica lag can result in stale data. The detailed implementation is reserved for **Topic 9 — Database Architecture** and **Topic 8 — Redis**.

## Banking Operations

### Deposit

```text
Deposit Request
      |
      v
Account Service
      |
      v
Primary Database
      |
      v
Balance Updated
      |
      v
Cache Invalidated
```

### Withdrawal

The notes identify locking considerations for withdrawals:

```text
Withdrawal
    |
    v
Validate balance
    |
    v
Lock / protect account update
    |
    v
Decrease balance
    |
    v
Record transaction
```

### Integration with Transaction Service

Fund transfers are not treated as a simple direct account operation.

```text
Transaction Service
       |
       | Validate
       v
Account Service
       |
       v
Transfer workflow
       |
       +---- Debit
       |
       +---- Credit
```

The transaction workflow uses Kafka events and compensation/reversal behavior.

### Interview Explanation

> "The Account Service owns account lifecycle and balance-related operations. It uses the primary database for writes and consistency-sensitive operations, while suitable reads can use the replica and Redis. Withdrawals require concurrency protection, and fund transfers are coordinated through the Transaction Service rather than being implemented as one simple database operation."

---

# 3.10 Transaction Service

## Responsibility

The Transaction Service is responsible for:

* Transaction orchestration
* Fund-transfer processing
* Transaction status tracking
* Event-driven transaction workflow
* Coordination of debit, credit and reversal processing

The documented service runs on port `8084`.

## Main APIs

```text
POST /api/v1/transactions/transfer
GET  /api/v1/transactions/{txnId}
```

The first initiates a transfer and the second retrieves transaction status.

## Synchronous Dependency

Before initiating the event-driven transfer workflow, Transaction Service communicates with Account Service to validate account information.

```text
Transaction Service
       |
       | OpenFeign
       v
Account Service
       |
       v
Validation Response
```

The notes specifically document OpenFeign for this interaction.

## Transaction Workflow

The documented flow is:

```text
Client
  |
  v
Transaction Service
  |
  +-- Validate JWT
  |
  +-- Validate limits
  |
  +-- Validate account through Account Service
  |
  +-- Create transaction PENDING
  |
  +-- Create outbox event
  |
  v
Kafka
  |
  v
Account Service
  |
  +-- Debit
  |
  +-- Credit
  |
  v
Result Events
  |
  v
Transaction Service
  |
  v
Update transaction status
```

The detailed implementation records states such as:

```text
PENDING
DEBIT_SUCCESS
CREDIT_SUCCESS
COMPLETED
FAILED
REVERSED
```

and uses an outbox table containing events such as `DEBIT`, `CREDIT`, and `REVERSAL`.

## Failure Handling

The documented compensation flow is:

```text
Debit Success
     |
Credit Failed
     |
     v
Transaction = FAILED
     |
     v
REVERSAL event
     |
     v
Account Service
     |
     v
Reverse debit
     |
     v
REVERSAL_SUCCESS
     |
     v
Transaction = REVERSED
```

This is the project’s event-driven compensation workflow. The original notes refer to it as a Saga pattern; the detailed Kafka and Outbox mechanics will be covered later rather than duplicated here.

### Interview Explanation

> "The Transaction Service acts as the orchestrator for fund transfers. It first performs synchronous validation with the Account Service through OpenFeign, then creates a pending transaction and starts the asynchronous transaction workflow through Kafka. The workflow tracks debit and credit results and can issue a reversal when a later step fails."

---

# 3.11 Notification Service

## Responsibility

The Notification Service handles asynchronous notification processing.

The documented responsibilities include:

* Email notifications
* SMS notifications
* Asynchronous delivery
* Notification tracking

The notes describe event consumption followed by template rendering and delivery through SendGrid or internal SMTP.

## Architecture

```text
Other Services
      |
      | Events
      v
    Kafka
      |
      v
Notification Service
      |
      +---- Render template
      |
      +---- Send email/SMS
      |
      v
Track delivery status
```

## Example — KYC Notification

```text
User Service
    |
    | KycEvent
    v
kyc-topic
    |
    v
Notification Service
    |
    v
Email
```

The documented consumer uses:

```yaml
spring:
  kafka:
    consumer:
      group-id: notification-group
      enable.auto.commit: false
      auto.offset.reset: earliest
```

This indicates manual offset management and starting from the earliest available records when no offset exists.

## OTP / Authentication Notifications

The deployment notes also document an event-driven pattern where Auth Service publishes OTP-related events and Notification Service consumes them to send emails.

### Interview Explanation

> "The Notification Service is intentionally asynchronous. Instead of Auth or User Service waiting for email delivery to complete, they publish an event to Kafka. Notification Service consumes the event, prepares the notification and performs the external delivery."

---

# 3.12 Complete Service Interaction

Putting the eight components together:

```text
                         CONFIG SERVER
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
           AUTH             USER            ACCOUNT
             |                |                |
             |                |                |
             |                +---- KYC ------> S3
             |                |
             |                +---- Kafka ----+
             |                              |
             v                              v
        Authentication                Notification
             |
             v
        API Gateway
             |
     +-------+-------+----------------+
     |               |                |
     v               v                v
   User           Account        Transaction
                                     |
                                     | OpenFeign
                                     v
                                  Account
                                     |
                                     v
                                   Kafka
                                     |
                        +------------+------------+
                        |                         |
                        v                         v
                    Account                Notification
```

Eureka and ECS Service Connect support service connectivity, while Kafka handles the asynchronous event paths.

---

# 3.13 Service Ownership Principle

A critical architectural rule in CitiCore is that a service should own its business data.

Conceptually:

```text
Auth Service
    |
    v
Auth Database


User Service
    |
    v
User/KYC Database


Account Service
    |
    v
Account Database


Transaction Service
    |
    v
Transaction Database
```

Another service should not bypass the owning service and directly modify its database.

For example:

```text
Transaction Service
       |
       X
Directly update Account DB


Transaction Service
       |
       v
Account Service
       |
       v
Account DB
```

This keeps business rules inside the service that owns the data.

---

# 3.14 Why Not One Large Service?

A monolithic design could technically implement the same banking functionality.

However:

```text
Monolith

+---------------------------------------+
| Auth                                  |
| User                                  |
| Account                               |
| Transaction                           |
| Notification                          |
| All deployed together                 |
+---------------------------------------+
```

CitiCore instead uses:

```text
+-------+  +-------+  +---------+
| Auth  |  | User  |  | Account |
+-------+  +-------+  +---------+
                           |
                     +-----+------+
                     | Transaction|
                     +------------+

+--------------+
| Notification |
+--------------+
```

The benefit is independent service boundaries and deployment.

The cost is distributed-system complexity:

* Network failures
* Service discovery
* Distributed tracing/debugging
* Event processing
* Configuration management
* Deployment dependencies
* Data consistency challenges

This trade-off is important to explain in interviews.

---

# 3.15 Deployment Dependency Order

The original deployment notes emphasize that infrastructure dependencies must exist before dependent services.

The documented logical sequence is:

```text
1. AWS Infrastructure
       |
       v
2. Config Server
       |
       v
3. Eureka
       |
       v
4. Auth Service
       |
       v
5. User Service
       |
       v
6. Account Service
       |
       v
7. Transaction Service
       |
       v
8. Notification Service
       |
       v
9. API Gateway
```

The exact deployment sequence is documented as intentional because services depend on configuration and discovery being available.

For example:

```text
Auth Service
     |
     | needs Eureka
     v
Eureka not running
     |
     v
Registration/discovery problem
```

The notes specifically identify this as a reason for deploying Config Server and Eureka before application services.

---

# 3.16 Real Microservice-Level Problems

## Problem 1 — Registration Timeout

A registration request could time out even though parts of the business operation had already completed.

**Lesson:** distributed requests are not automatically atomic from the client's perspective.

Before retrying:

```text
Timeout
  |
  v
Check actual state
  |
  +-- User exists?
  +-- OTP exists?
  +-- Event published?
  |
  v
Then decide whether retry is safe
```

The notes specifically warn that blindly retrying could create duplicate users.

---

## Problem 2 — Kafka Dependency

A service can be infrastructure-healthy while a business operation is broken because a Kafka dependency is unavailable or incorrectly configured.

The notes describe a `504` scenario where a missing Kafka topic caused the operation to hang even though the application's health endpoint could still return successfully.

This demonstrates an important distinction:

```text
Infrastructure Health
        ≠
Business Operation Health
```

---

## Problem 3 — Service Connect Configuration

The notes document an `UnknownHostException` caused by Service Connect not being enabled/configured correctly for the Eureka communication path.

The debugging lesson was to distinguish:

```text
External traffic
      |
      v
ALB

Internal traffic
      |
      v
Service Connect
```

ALB is not a replacement for internal service discovery.

---

# 3.17 Service-Level Interview Summary

If asked **"Explain your microservices architecture"**, a concise answer is:

> "I designed CitiCore around separate infrastructure and business responsibilities. Config Server handles centralized configuration and Eureka provides application-level service discovery. API Gateway is the external entry point. Auth handles authentication and JWT, User manages profiles and KYC, Account manages accounts and balances, Transaction orchestrates fund transfers, and Notification handles asynchronous notifications.
>
> For synchronous interactions, such as transaction validation against Account Service, I use OpenFeign. For asynchronous workflows such as KYC, OTP and transaction events, I use Kafka. Account Service also integrates with MySQL primary-replica storage and Redis.
>
> The services are independently containerized and deployed on ECS/Fargate. ECS Service Connect provides internal service connectivity, while the gateway provides the controlled external entry point."

---

# 3.18 Interview Follow-Up Questions

### Q1. How did you decide the service boundaries?

**Answer:**
I separated services around business responsibilities. Authentication, users, accounts, transactions and notifications have different responsibilities and processing requirements, so keeping them independent makes ownership clearer.

### Q2. Which service owns account data?

**Answer:**
Account Service owns account-related data and business rules. Other services interact with it through APIs/events rather than directly modifying its database.

### Q3. Why does Transaction Service call Account Service synchronously?

**Answer:**
Some information, such as account validation, is required immediately before the transfer workflow can continue. Therefore I use OpenFeign for that request/response interaction.

### Q4. Why doesn't Notification Service get called synchronously?

**Answer:**
Notification delivery doesn't normally need to block the main business operation. Publishing an event allows the notification to be processed independently and avoids coupling transaction processing to an external email provider.

### Q5. What happens if Notification Service is down?

**Answer:**
The producer can still publish events to Kafka. Kafka retains events according to its retention configuration, and the Notification Service can consume pending events when it becomes available.

The notes explicitly describe this behavior: Kafka stores events independently of whether the consumer is currently running.

### Q6. Why do you have both Eureka and Service Connect?

**Answer:**
Eureka provides application-level service registration and discovery, while ECS Service Connect provides ECS-native internal DNS-based communication. In the ECS deployment, Service Connect is used for important internal communication paths.

### Q7. What happens if Config Server is unavailable during startup?

**Answer:**
A dependent service may be unable to obtain its required configuration and therefore fail to start correctly. This is one reason the deployment order starts with infrastructure and Config Server before application services.

### Q8. What happens if Account Service is unavailable during a transfer?

**Answer:**
The synchronous validation call can fail or time out, so the transaction should not blindly continue. Gateway/service resilience mechanisms and appropriate transaction-state handling prevent the failure from being treated as a successful transfer.

### Q9. Is the Transaction workflow a distributed transaction?

**Answer:**
It is not one traditional database transaction spanning multiple services. The project uses an event-driven workflow with transaction states and compensation/reversal behavior. The detailed consistency mechanism is covered by the Kafka and Transactional Outbox design.

### Q10. What is the biggest disadvantage of your microservices architecture?

**Answer:**
Operational complexity. Instead of debugging one application and one database, I have to consider network communication, service discovery, Kafka, database consistency, configuration, container health and distributed failures.

---

# 3.19 Final Microservices Architecture

```text
                           CLIENT
                              |
                              v
                           ALB
                              |
                              v
                       API GATEWAY
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
           AUTH             USER             ACCOUNT
             |                |                |
             |                |                +----> MySQL
             |                |                |
             |                +----> S3        +----> Redis
             |                |                |
             |                +----> Kafka <----+
             |                                 |
             |                                 v
             |                           TRANSACTION
             |                                 |
             |                                 |
             |                           OpenFeign
             |                                 |
             +---------------------------------+
                                               |
                                               v
                                             Kafka
                                               |
                                +--------------+--------------+
                                |                             |
                                v                             v
                         ACCOUNT SERVICE                NOTIFICATION
                                                              |
                                                              v
                                                         Email / SMS


Supporting Infrastructure
-------------------------
Config Server
Eureka
ECS Service Connect
Docker
ECS/Fargate
ECR
RDS
Redis
Kafka
```

## Core Principle

The CitiCore microservice design can be remembered as:

```text
ONE RESPONSIBILITY
        +
CLEAR DATA OWNERSHIP
        +
SYNCHRONOUS CALLS WHEN IMMEDIATE RESPONSE IS REQUIRED
        +
ASYNC EVENTS WHEN LOOSE COUPLING IS PREFERRED
        +
CENTRALIZED CONFIGURATION
        +
SERVICE DISCOVERY
        =
CitiCore Microservices Architecture
```

The most important interview point is **not "I used eight microservices."**

It is:

> **"I separated responsibilities based on business ownership and selected synchronous or asynchronous communication according to the consistency and coupling requirements of each workflow."**

This is the reasoning behind the architecture.

---

## TOPIC 4 — SERVICE DISCOVERY

## 4.1 What Is Service Discovery?

In a microservices architecture, services need to communicate with other services. The problem is that service instances may not have permanent IP addresses.

This is especially relevant in CitiCore because the services are deployed as ECS/Fargate tasks. When a task is replaced or restarted, its private IP can change.

Without service discovery:

```text
Transaction Service
        |
        | hardcoded IP
        v
10.0.1.75:8083
        |
     Task dies
        |
        v
10.0.1.120:8083
        |
        X
Old address is invalid
```

Service discovery removes this dependency on fixed IP addresses.

Instead:

```text
Transaction Service
        |
        | "Where is Account Service?"
        v
 Service Discovery
        |
        v
Current Account Service instance
```

The project notes describe Eureka as solving exactly this dynamic-address problem in Fargate.

---

# 4.2 Why CitiCore Needs Service Discovery

CitiCore has multiple independently deployed services:

```text
Auth
User
Account
Transaction
Notification
Gateway
```

These services may scale, restart, or be redeployed independently.

Therefore, hardcoding addresses creates unnecessary coupling.

### Without discovery

```text
Transaction
    |
    +----> 10.0.11.50:8083
    |
    +----> Fixed address
```

If Account Service moves:

```text
Transaction
    |
    X
10.0.11.50:8083

Account is now:
10.0.11.100:8083
```

The caller's configuration must change.

### With discovery

```text
Transaction
    |
    v
"ACCOUNT-SERVICE"
    |
    v
Service Registry
    |
    v
10.0.11.100:8083
```

The caller only needs to know the **logical service name**, not its current IP.

---

# 4.3 Eureka in CitiCore

CitiCore uses **Netflix Eureka** as an application-level service registry.

The documented Eureka Server runs on:

```text
Port: 8761
```

Its responsibility is:

```text
Eureka Server
     |
     +-- Service registration
     +-- Service discovery
     +-- Instance information
     +-- Health/status information
```

The project notes describe services registering themselves with Eureka and other services querying Eureka for peer locations.

---

# 4.4 How Eureka Works

There are three important operations:

1. Service registration
2. Service discovery
3. Instance health/heartbeat

## Step 1 — Service Registration

When Account Service starts:

```text
Account Service
      |
      | REGISTER
      v
Eureka Server
      |
      v
ACCOUNT-SERVICE
10.x.x.x:8083
```

The service tells Eureka:

> "I am ACCOUNT-SERVICE and I am available at this address."

The same pattern applies to Auth, User, Transaction and other services.

The documented configuration enables clients to register themselves with Eureka.

---

# 4.5 Step 2 — Service Discovery

Suppose Transaction Service needs Account Service.

Instead of:

```text
http://10.0.1.75:8083
```

it works with the logical service identity:

```text
ACCOUNT-SERVICE
```

Conceptually:

```text
Transaction Service
        |
        | Find ACCOUNT-SERVICE
        v
      Eureka
        |
        v
10.0.1.75:8083
```

If Account Service is restarted:

```text
Old:
10.0.1.75:8083

       ↓ restart

New:
10.0.1.120:8083
```

The registry can contain the new instance information.

The project notes specifically describe this dynamic update behavior.

---

# 4.6 Step 3 — Heartbeats and Instance Status

Eureka clients periodically communicate their availability to the Eureka Server.

The documented configuration includes:

```yaml
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true

  instance:
    hostname: auth-service
    lease-renewal-interval-in-seconds: 10
```

This means the Auth Service registers itself, fetches the registry and sends its lease renewal at the configured interval.

Conceptually:

```text
Auth Service
     |
     | heartbeat
     v
Eureka
     |
     | status = UP
     v
Registry
```

If an instance becomes unavailable and is eventually removed from the registry, clients should no longer select it.

---

# 4.7 Eureka Server Configuration

The CitiCore Eureka Server is configured as a Eureka server rather than as a Eureka client.

The documented configuration is:

```yaml
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false

server:
  port: 8761
```

The important distinction is:

```text
Eureka Server
    |
    X Does not register itself
    X Does not fetch another registry
    |
    ✓ Maintains service registry
```

The source specifically documents this single Eureka-server setup.

---

# 4.8 Eureka Client Configuration

Application services use Eureka as clients.

Example:

```yaml
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka-server-8761-tcp.citicore:8761/eureka
```

The important settings are:

### `register-with-eureka`

```text
true
```

The service registers itself.

### `fetch-registry`

```text
true
```

The service obtains information about other registered services.

### `defaultZone`

Specifies where the Eureka Server can be reached.

The documented CitiCore configuration uses the ECS Service Connect DNS name rather than a hardcoded Eureka IP.

---

# 4.9 Eureka Dashboard and Verification

The notes document checking registered applications through Eureka.

For example:

```bash
curl http://eureka-server:8761/eureka/apps
```

A registered service can appear conceptually as:

```text
AUTH-SERVICE
    |
    +-- Instance ID: auth-service
    +-- Host: 10.0.11.50
    +-- Port: 8081
    +-- Status: UP
```

A specific service can also be checked:

```bash
curl http://eureka-server:8761/eureka/apps/AUTH-SERVICE
```

These checks were part of the documented verification process.

---

# 4.10 Eureka Architecture in CitiCore

```text
                         Eureka Server
                            :8761
                               |
          +--------------------+--------------------+
          |                    |                    |
          v                    v                    v
    AUTH-SERVICE         USER-SERVICE        ACCOUNT-SERVICE
       :8081                :8082                :8083
          |                    |                    |
          +--------------------+--------------------+
                               |
                               v
                       TRANSACTION-SERVICE
                              :8084
```

The documented registry example shows Auth, User, Account and Transaction instances registered with Eureka.

---

# 4.11 Eureka vs Hardcoded URLs

### Hardcoded approach

```text
Transaction Service
       |
       v
http://10.0.1.75:8083
```

**Problems:**

* IP can change
* Configuration must be updated
* Tight coupling
* Difficult scaling
* More operational work

### Service discovery

```text
Transaction Service
       |
       v
ACCOUNT-SERVICE
       |
       v
Eureka
       |
       v
Current instance
```

**Benefits:**

* Dynamic discovery
* Less dependence on IP addresses
* Supports changing instances
* Better suited to containerized deployment

The source explicitly identifies dynamic discovery as the reason for using Eureka with Fargate.

---

# 4.12 Eureka + ECS Service Connect

This is one of the most important CitiCore-specific points.

The project does **not** simply use Eureka for every networking requirement.

It uses:

```text
Eureka
  +
ECS Service Connect
```

for different purposes.

## Eureka

Primarily provides:

```text
Application-level
service registration
and discovery
```

## Service Connect

Provides:

```text
ECS-level
internal service networking
DNS discovery
and routing
```

The notes explicitly document Service Connect as the internal communication mechanism for ECS services.

---

# 4.13 Why Service Connect Was Important

In ECS, services need a stable way to reach each other.

CitiCore creates a Service Connect namespace:

```text
citicore
```

For example, Eureka has:

```text
Discovery Name:
eureka-server-8761-tcp

DNS:
eureka-server-8761-tcp.citicore:8761
```

Auth Service can therefore reach Eureka using:

```text
http://eureka-server-8761-tcp.citicore:8761/eureka
```

instead of depending on Eureka's changing task IP.

---

# 4.14 Service Connect Architecture

```text
                    ECS Cluster
                        |
                  citicore namespace
                        |
          +-------------+-------------+
          |                           |
          v                           v
    Eureka Service              Auth Service
          |                           |
          |                           |
          +------ DNS discovery ------+
```

For another example:

```text
Transaction Service
        |
        | OpenFeign
        v
citicore-account-service-8083-tcp.citicore:8083
        |
        v
ECS Service Connect
        |
        v
Account Service Task
```

The project notes explicitly document this Transaction → Account communication path.

---

# 4.15 Why Both Eureka and Service Connect?

This is a common interview question.

A good CitiCore-specific answer is:

> "I used Eureka for application-level service registration and discovery, while ECS Service Connect handles internal ECS networking through DNS. Service Connect avoids depending directly on changing ECS task IPs, and Eureka provides the application-level service registry and discovery mechanism."

In other words:

```text
                    CitiCore
                       |
          +------------+------------+
          |                         |
          v                         v
       Eureka                 Service Connect
          |                         |
          v                         v
Application-level          ECS internal networking
discovery                  + DNS
```

The project notes explicitly describe Service Connect as an override for important ECS paths while retaining Eureka for additional discovery.

---

# 4.16 Service Connect Configuration

The documented Eureka ECS service uses:

```json
{
  "enabled": true,
  "namespace": "citicore",
  "services": [
    {
      "portName": "tcp",
      "discoveryName": "eureka-server-8761-tcp",
      "clientAliases": [
        {
          "port": 8761,
          "dnsName": "eureka-server-8761-tcp.citicore"
        }
      ]
    }
  ]
}
```

The ECS task/service uses `awsvpc` networking for Service Connect.

---

# 4.17 Real Problem Encountered — UnknownHostException

One of the important real CitiCore troubleshooting cases was:

```text
UnknownHostException:
eureka-server-8761-tcp.citicore
```

The application could not resolve the Eureka Service Connect DNS name.

The documented root causes included:

```text
Service Connect not enabled
        OR
Wrong discovery name
        OR
Wrong namespace
        OR
Incorrect network mode
        OR
Services not in same namespace
```

The notes specifically identify this as a real deployment issue.

---

# 4.18 Troubleshooting the DNS Problem

The debugging process was:

### Step 1 — Check Service Connect

```bash
aws ecs describe-services \
  --cluster citicore-cluster \
  --services citicore-eureka-server
```

Verify that Service Connect is enabled.

### Step 2 — Verify namespace

Both services must use:

```text
namespace = citicore
```

### Step 3 — Verify discovery name

Auth Service configuration must match Eureka's discovery name:

```text
eureka-server-8761-tcp
```

### Step 4 — Verify network mode

The task definition should use:

```text
awsvpc
```

### Step 5 — Check application logs

The documented example is:

```bash
aws logs tail /ecs/citicore-auth-service --follow
```

Look for DNS-resolution failures.

---

# 4.19 How We Verified Connectivity

The notes document testing connectivity from inside the Auth Service container:

```bash
curl http://eureka-server-8761-tcp.citicore:8761/
```

Successful result:

```text
Eureka Server page
```

Failure:

```text
UnknownHostException
```

This is a useful troubleshooting principle:

```text
Don't assume DNS works.

Test it from the actual application environment.
```

---

# 4.20 Deployment Order and Service Discovery

Service discovery also influenced deployment order.

The documented sequence starts with:

```text
Infrastructure
      ↓
Config Server
      ↓
Eureka
      ↓
Application Services
```

Why?

Because application services need configuration and discovery infrastructure during startup.

If Auth Service starts before Eureka:

```text
Auth Service
     |
     | Register
     v
Eureka
     |
     X
Not available
     |
     v
Registration problem
```

The project notes explicitly identify this dependency and therefore deploy Eureka before application services.

---

# 4.21 Single Eureka Instance — Current Design

The documented CitiCore setup currently uses:

```text
Eureka Server
      |
      v
1 Instance
```

This is important to state honestly in interviews.

Do **not** say:

> "I implemented a highly available three-node Eureka cluster."

The notes instead document a single Eureka deployment and separately describe a three-instance Eureka cluster as a **future improvement**.

---

# 4.22 Future Improvement — Clustered Eureka

The notes propose:

```text
             +-- Eureka 1
             |
Services ----+-- Eureka 2
             |
             +-- Eureka 3
```

Services could then configure multiple Eureka URLs:

```yaml
eureka:
  client:
    service-url:
      defaultZone: >
        http://eureka-1:8761/eureka,
        http://eureka-2:8761/eureka,
        http://eureka-3:8761/eureka
```

The intended benefit is avoiding a single registry instance becoming a single point of failure.

This is a **future improvement**, not the current CitiCore implementation.

---

# 4.23 Important Trade-Offs

## Benefit 1 — Dynamic discovery

Services don't need to maintain changing IP addresses.

## Benefit 2 — Better container compatibility

It fits environments where tasks can be replaced dynamically.

## Benefit 3 — Service abstraction

A caller can depend on a service identity instead of a specific machine.

## Cost 1 — Additional infrastructure

Eureka itself must be deployed and monitored.

## Cost 2 — Startup dependency

Application services depend on the availability/configuration of discovery.

## Cost 3 — Operational complexity

When communication fails, you must investigate:

```text
Application
   ↓
DNS
   ↓
Service Connect
   ↓
Security Group
   ↓
Network
   ↓
Eureka
   ↓
Target Service
```

---

# 4.24 Important Distinction: Discovery vs Load Balancing

Do not treat these as exactly the same thing.

### Service Discovery

Answers:

```text
"Where is Account Service?"
```

### Load Balancing

Answers:

```text
"Which healthy Account Service instance should receive this request?"
```

In CitiCore, ECS Service Connect provides DNS-based discovery and built-in load-balancing behavior for ECS service communication, while Eureka maintains the application-level registry.

---

# 4.25 Service Discovery Failure Flow

A useful interview troubleshooting model is:

```text
Request
   |
   v
Caller Service
   |
   | Can't reach target
   v
Check service name
   |
   v
Check DNS
   |
   v
Check Service Connect
   |
   v
Check namespace
   |
   v
Check network mode
   |
   v
Check security groups
   |
   v
Check target service
   |
   v
Check application logs
```

This reflects the actual troubleshooting approach documented in the CitiCore notes: check each infrastructure/application layer instead of assuming that a running ECS task means the entire communication path is healthy.

---

# 4.26 Interview Answer — "Explain Service Discovery in CitiCore"

> "CitiCore runs multiple microservices on ECS Fargate, where task IP addresses can change whenever containers are replaced. To avoid hardcoding IP addresses, I used Eureka as an application-level service registry. Services register themselves with Eureka and can discover other services dynamically.
>
> For the ECS environment, I also used AWS Service Connect for internal DNS-based service communication. For example, Auth Service reaches Eureka using the Service Connect DNS name instead of a fixed IP.
>
> I actually faced an `UnknownHostException` when the Service Connect DNS name for Eureka couldn't be resolved. I checked the Service Connect configuration, namespace, discovery name, `awsvpc` networking and CloudWatch logs. This helped me understand that service discovery is not only an application configuration problem; the underlying ECS networking must also be correctly configured."

---

# 4.27 Interview Follow-Up Questions

### Q1. Why can't you hardcode service IP addresses?

Because ECS tasks can be replaced and receive different IP addresses. Hardcoding creates tight coupling and causes communication failures after redeployment.

### Q2. What is Eureka?

Eureka is a service registry. Services register their instance information, and clients can discover available service instances through it.

### Q3. What is the difference between Eureka Server and Eureka Client?

**Eureka Server** maintains the registry.

**Eureka Client** registers its own service and can fetch information about other services.

### Q4. What does `register-with-eureka=true` mean?

The service registers its own instance with Eureka.

### Q5. What does `fetch-registry=true` mean?

The service retrieves service registry information so it can discover other services.

### Q6. What happens when an ECS task gets a new IP?

The service's instance information is updated through the service-discovery mechanism rather than requiring callers to manually update a hardcoded IP.

### Q7. Why did you use Service Connect if you already had Eureka?

Service Connect provides ECS-native internal DNS-based networking. Eureka provides application-level service registration/discovery. In CitiCore, Service Connect was used for important ECS internal paths while Eureka remained part of the application discovery architecture.

### Q8. What was your real Eureka/Service Connect issue?

An `UnknownHostException` occurred because the Eureka Service Connect DNS name could not be resolved. The investigation focused on Service Connect enablement, namespace, discovery name and ECS networking configuration.

### Q9. Is your Eureka cluster highly available?

The current documented setup uses a single Eureka instance. A three-instance Eureka deployment is documented as a future improvement.

### Q10. What happens if Eureka goes down?

This depends on the communication path. The project uses Service Connect for important ECS internal paths, reducing dependence on Eureka for those paths. However, the current single Eureka deployment remains an infrastructure dependency for application-level discovery.

### Q11. How did you verify service registration?

Through the Eureka dashboard/API, including:

```bash
curl http://eureka-server:8761/eureka/apps
```

and by checking individual applications.

---

# 4.28 Key Lessons from CitiCore

### Lesson 1 — Don't confuse service discovery with networking

```text
Eureka
= application-level registry

Service Connect
= ECS internal networking
```

### Lesson 2 — DNS must be tested from the actual container

A DNS name working from your local machine does not prove that it works from an ECS task.

### Lesson 3 — Infrastructure readiness matters

```text
Infrastructure
     ↓
Config
     ↓
Discovery
     ↓
Application
```

Deploying a dependent service before its infrastructure dependencies are ready can cause startup failures.

### Lesson 4 — Current implementation vs future architecture

Current:

```text
1 Eureka
+
Service Connect
```

Future:

```text
Multiple Eureka instances
+
Service Connect
```

Keep these separate when explaining the project.

---

# 4.29 Final Mental Model

Remember Service Discovery in CitiCore like this:

```text
                 CITI CORE
                    |
          +---------+---------+
          |                   |
          v                   v
       Eureka            Service Connect
          |                   |
          v                   v
 Application Registry    ECS Internal DNS
          |                   |
          +---------+---------+
                    |
                    v
             Microservices
```

The fundamental problem is:

```text
Dynamic infrastructure
        ↓
Changing IP addresses
        ↓
Hardcoded addresses become unreliable
        ↓
Service Discovery
        ↓
Stable logical service identity
```

**Interview takeaway:**

> **"Service discovery decouples service communication from the physical location of service instances."**

---
## TOPIC 5 — API GATEWAY

## 5.1 What is an API Gateway?

## Concept

An API Gateway is the **single entry point** through which external clients access backend microservices.

Without an API Gateway:

```text
                    Client
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
        Auth        User       Account
        :8081       :8082        :8083
```

The client must know the location and API structure of every service.

With an API Gateway:

```text
                    Client
                      |
                      v
                API Gateway
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
        Auth        User       Account
```

The client knows only one external endpoint.

---

# 5.2 Why CitiCore Uses an API Gateway

CitiCore has multiple backend services:

```text
Auth Service
User Service
Account Service
Transaction Service
Notification Service
```

Exposing every service directly to clients would create unnecessary complexity and increase the external attack surface.

The gateway provides a controlled entry point for:

* Request routing
* Path rewriting
* JWT validation
* Rate limiting
* Circuit breaking
* Retry handling
* Common gateway policies
* Internal service routing

The project documentation explicitly identifies these as gateway responsibilities.

---

# 5.3 CitiCore API Gateway Architecture

The external request path is:

```text
                         INTERNET
                            |
                            v
                  +-------------------+
                  |    Gateway ALB    |
                  |       :8080       |
                  +---------+---------+
                            |
                            v
                  +-------------------+
                  |    API Gateway    |
                  | Spring Cloud      |
                  | Gateway :8080      |
                  +---------+---------+
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
        Auth Service   User Service   Account Service
           :8081          :8082           :8083
                                             |
                                             v
                                      Transaction Service
                                             :8084
```

The documented architecture uses Spring Cloud Gateway on port `8080`, with the ALB providing the external entry point before traffic reaches the gateway.

---

# 5.4 Technology Used

CitiCore uses:

```text
Java 17
     |
Spring Boot 3.2.4
     |
Spring Cloud Gateway
     |
Resilience / Gateway Filters
     |
Redis
     |
AWS ECS/Fargate
```

The source notes identify Spring Cloud Gateway as the component responsible for routing, request filtering, path rewriting, and resilience-related functionality.

Spring Cloud Gateway uses a reactive architecture, which fits the gateway's role of handling a large number of network-bound requests without requiring a dedicated blocking thread for every request.

---

# 5.5 Gateway Responsibilities

The CitiCore Gateway performs several different responsibilities.

```text
                    API Gateway
                         |
       +-----------------+------------------+
       |                 |                  |
       v                 v                  v
    Routing          Security           Resilience
       |                 |                  |
       v                 v                  v
Path rewriting       JWT             Rate limiting
                     validation       Circuit breaker
                                      Retry
```

The main responsibilities implemented/documented are:

1. Request routing
2. Path rewriting
3. JWT validation
4. Rate limiting
5. Circuit breaking
6. Retry handling
7. Centralized external entry point
8. Internal service routing
9. Configuration through Config Server

---

# 5.6 Request Routing

## Concept

Routing determines which backend service should receive a request.

For example:

```text
/auth/**
      ↓
Auth Service

/users/**
      ↓
User Service

/accounts/**
      ↓
Account Service

/transaction/**
      ↓
Transaction Service
```

The documented CitiCore routes are:

| External Path     | Backend Service     | Port |
| ----------------- | ------------------- | ---: |
| `/auth/**`        | Auth Service        | 8081 |
| `/users/**`       | User Service        | 8082 |
| `/accounts/**`    | Account Service     | 8083 |
| `/transaction/**` | Transaction Service | 8084 |

These mappings are explicitly documented in the project configuration.

---

# 5.7 How a Route is Selected

A gateway route normally contains:

```text
Route
 |
 +-- ID
 +-- URI
 +-- Predicates
 +-- Filters
```

For example, the documented Auth route contains:

```yaml
- id: auth-service
  uri: http://auth-service-8081-tcp.citicore:8081
  predicates:
    - Path=/auth/**
    - Method=GET,POST
  filters:
    - RewritePath=/auth/(?<segment>.*), /api/v1/auth/${segment}
    - CircuitBreaker=auth,forward:/fallback/auth
    - Retry=retries=1,statuses=503;504
```

The route is loaded from the centralized gateway configuration.

---

# 5.8 Route Processing Flow

Consider:

```text
POST /auth/login
```

The gateway processes it approximately as:

```text
Client
  |
  | POST /auth/login
  v
API Gateway
  |
  | 1. Match Path=/auth/**
  |
  | 2. Check Method=POST
  |
  | 3. Apply filters
  |
  | 4. Rewrite path
  |
  | 5. Apply resilience policies
  |
  | 6. Forward request
  v
Auth Service
```

The documented route-processing example follows this same sequence.

---

# 5.9 Path Rewriting

## Why Path Rewriting?

The external API structure does not have to be identical to the internal service API structure.

For example:

```text
External API:

POST /accounts/create
```

The gateway rewrites this to:

```text
Internal API:

POST /api/v1/accounts/create
```

So:

```text
Client
 |
 | /accounts/create
 v
Gateway
 |
 | RewritePath
 v
Account Service
 |
 | /api/v1/accounts/create
```

The project documentation specifically uses path rewriting to keep the external API structure separate from internal service endpoints.

---

# 5.10 Why Path Rewriting is Useful

It provides an abstraction layer.

Suppose the internal API changes from:

```text
/api/v1/accounts/create
```

to:

```text
/api/v2/accounts/create
```

The gateway can potentially keep the external endpoint stable:

```text
Client
 |
 | /accounts/create
 v
Gateway
 |
 | internal mapping changes
 v
/api/v2/accounts/create
```

The client does not necessarily need to know about the internal change.

### Main benefit

```text
External API
     |
     v
Stable Contract
     |
     v
Gateway
     |
     v
Internal API
```

---

# 5.11 JWT Validation at the Gateway

CitiCore uses JWT-based authentication.

The basic flow is:

```text
Client
   |
   | username/password
   v
Auth Service
   |
   | Generate JWT
   v
Client
```

For subsequent requests:

```text
Client
   |
   | Authorization: Bearer <JWT>
   v
API Gateway
   |
   | Validate JWT
   |
   +---- Invalid ----> 401
   |
   +---- Valid
          |
          v
     Backend Service
```

The project documentation describes the gateway validating the JWT signature and extracting user information before forwarding the request.

---

# 5.12 Authentication vs Authorization

These two concepts should not be confused.

### Authentication

Answers:

> "Who is this user?"

Example:

```text
JWT
 |
 v
User ID = 123
Email = user@example.com
```

### Authorization

Answers:

> "Is this user allowed to perform this operation?"

Example:

```text
ROLE_USER
     |
     X
Admin-only operation
```

The gateway can perform common authentication-related validation, while backend services still need to enforce business-specific authorization.

This is important because the gateway should not become the only place where business authorization is enforced.

---

# 5.13 Rate Limiting

## Concept

Rate limiting controls how many requests a client can make within a given period.

Without rate limiting:

```text
Client A
   |
   | 1000 requests/sec
   v
Gateway
   |
   v
Backend
   |
   X
Overloaded
```

With rate limiting:

```text
Client A
   |
   | 2 req/sec
   v
Gateway
   |
   v
Backend
```

The documented CitiCore configuration uses:

```text
Rate:
2 requests / second / IP

Burst:
4 requests

Storage:
Redis
```

These are the project's configured values, not universal recommended values.

---

# 5.14 Why Rate Limiting is Needed

Consider one client sending:

```text
1000 requests/sec
```

Without protection, that client could consume gateway and backend resources.

The documented design therefore limits requests per client IP and allows a small burst.

```text
Normal:
2 requests/sec

Temporary burst:
up to 4 requests
```

The source notes describe this as protection against a single client monopolizing gateway resources and causing a denial-of-service-like effect.

---

# 5.15 Why Redis is Used for Rate Limiting

The gateway may run as multiple ECS tasks.

For example:

```text
                    Load Balancer
                         |
              +----------+----------+
              |                     |
              v                     v
        Gateway Task 1         Gateway Task 2
```

If each gateway task maintained its own rate-limit counter:

```text
Task 1:
2 requests allowed

Task 2:
2 requests allowed
```

A client could potentially bypass the intended global limit by reaching different instances.

Redis provides shared rate-limit state:

```text
Gateway 1
    |
    |
Gateway 2 -----> Redis
    |
    |
Gateway 3
```

The source specifically documents Redis as the backing store for rate-limiter state and distributed request tracking.

The detailed Redis architecture is intentionally covered later in **Topic 8 — Redis**.

---

# 5.16 Circuit Breaker

## Concept

A circuit breaker prevents the gateway from continuously calling an unhealthy downstream service.

Without a circuit breaker:

```text
Gateway
   |
   +----> User Service DOWN
   |
   +----> User Service DOWN
   |
   +----> User Service DOWN
   |
   +----> User Service DOWN
   |
   v
Repeated timeouts
```

This can cause resource exhaustion and cascading failures.

With a circuit breaker:

```text
Gateway
   |
   v
Circuit Breaker
   |
   +---- Healthy → Forward
   |
   +---- Unhealthy → Fast fail
```

---

# 5.17 CitiCore Circuit Breaker Configuration

The documented configuration includes:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      auth:
        slidingWindowSize: 10
        failureRateThreshold: 50
        slowCallRateThreshold: 50
        slowCallDurationThreshold: 3s
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 2
```

The documented behavior is:

```text
Sliding window:
10 calls

Failure threshold:
50%

Open-state wait:
10 seconds

Half-open test calls:
2
```

These are the actual project-documented values.

---

# 5.18 Circuit Breaker States

A circuit breaker has three important states.

## CLOSED

Service is considered healthy.

```text
Request
   |
   v
Circuit CLOSED
   |
   v
Backend
```

Requests are forwarded normally while failures are monitored.

---

## OPEN

Failure threshold has been exceeded.

```text
Request
   |
   v
Circuit OPEN
   |
   X
Backend not called
   |
   v
Fast failure / 503
```

The documented implementation waits 10 seconds before moving toward recovery testing.

---

## HALF-OPEN

The circuit allows a small number of test requests.

```text
Circuit HALF_OPEN
       |
       +---- Test Request 1
       |
       +---- Test Request 2
```

If the service recovers:

```text
HALF_OPEN
    |
    v
Success
    |
    v
CLOSED
```

If it is still failing:

```text
HALF_OPEN
    |
    v
Failure
    |
    v
OPEN
```

---

# 5.19 Why Circuit Breaker and Retry are Different

These are often confused in interviews.

### Retry

Says:

> "The failure might be temporary, so try again."

```text
Request
   |
   X
Failure
   |
   v
Retry
   |
   v
Backend
```

### Circuit Breaker

Says:

> "The service is failing repeatedly, so stop sending requests for a while."

```text
Repeated failures
       |
       v
Circuit OPEN
       |
       X
Stop calls temporarily
```

Therefore:

```text
Retry
= Recover from transient failures

Circuit Breaker
= Prevent repeated calls to an unhealthy service
```

---

# 5.20 Retry Logic

The project documents retry behavior for transient failures.

Configured behavior includes:

```text
Maximum attempts:
2

Therefore:
1 original attempt
+
1 retry
```

The documented retry configuration includes:

```yaml
resilience4j:
  retry:
    instances:
      user:
        maxAttempts: 2
        waitDuration: 100ms
        retryExceptions:
          - java.util.concurrent.TimeoutException
          - java.net.SocketTimeoutException
```

The route-level configuration also specifies retry behavior for suitable HTTP failure statuses.

---

# 5.21 Exponential Backoff

Instead of immediately retrying repeatedly:

```text
Request
  |
  X
  |
  +-- immediate retry
  |
  X
```

the gateway waits before retrying.

The documented route configuration uses:

```text
First backoff: 100ms
Maximum backoff: 500ms
Factor: 2
```

Conceptually:

```text
Attempt 1
   |
 Failure
   |
100ms
   |
Attempt 2
```

The purpose is to avoid immediately overwhelming a temporarily overloaded service.

---

# 5.22 Important Retry Safety Consideration

Retrying is **not automatically safe for every API**.

Consider:

```text
POST /accounts/create
```

The first request might actually succeed, but the response could be lost because of a timeout.

If the gateway blindly retries:

```text
Attempt 1
   |
   +---- Account created
   |
   X
 Response timeout

Attempt 2
   |
   +---- Create account again
```

This can create duplicate business operations unless the API is idempotent.

This is especially important for banking systems.

Therefore:

```text
Retry
  +
Idempotency
  =
Safer distributed request processing
```

The CitiCore notes also document a real registration-timeout scenario where blindly retrying could create duplicate user state.

The lesson is broader than the gateway itself:

> **A timeout does not necessarily mean that the business operation did not happen.**

---

# 5.23 Gateway Configuration Through Config Server

The gateway configuration is externalized.

```text
Git Repository
      |
      v
Config Server :8888
      |
      v
API Gateway :8080
```

The documented gateway configuration contains:

```text
application.yml
apigateway-service.yml
```

The gateway's `application.yml` imports configuration from Config Server, while routes are defined in the gateway-specific configuration.

This means configuration such as:

* Routes
* Rate limits
* Circuit breaker settings
* Retry configuration

can be managed centrally.

---

# 5.24 Why Externalize Gateway Configuration?

Imagine the rate limit needs to change:

```text
2 req/sec
      ↓
5 req/sec
```

Without centralized configuration:

```text
Change code
   |
Build
   |
Docker image
   |
Deploy
```

With externalized configuration:

```text
Update configuration
       |
       v
Config Server
       |
       v
Gateway
```

The source documentation identifies versioning, auditing, route updates, and tuning gateway settings through Git-backed configuration as benefits.

---

# 5.25 Internal Service Routing

The gateway does not expose backend service addresses directly to external clients.

The flow is:

```text
Internet
   |
   v
ALB
   |
   v
API Gateway
   |
   v
Internal Service
```

Within ECS, the documented implementation uses Service Connect DNS names for internal communication.

For example:

```text
auth-service-8081-tcp.citicore:8081
```

rather than relying on a changing ECS task IP.

This complements the Eureka-based service-discovery architecture described earlier.

The Service Connect configuration and internal networking are covered in **Topic 4 — Service Discovery** and should not be duplicated here.

---

# 5.26 Complete Gateway Request Flow

A complete request can be visualized as:

```text
                         CLIENT
                           |
                           | HTTP Request
                           v
                    +--------------+
                    |     ALB      |
                    |    :8080     |
                    +------+-------+
                           |
                           v
                    +--------------+
                    | API Gateway  |
                    |    :8080     |
                    +------+-------+
                           |
                    Route Matching
                           |
                           v
                    Path Rewriting
                           |
                           v
                     JWT Validation
                           |
                           v
                     Rate Limiting
                           |
                           v
                    Circuit Breaker
                           |
                           v
                         Retry
                           |
                           v
                    Internal Service
                           |
                           v
                       Response
                           |
                           v
                    API Gateway
                           |
                           v
                          ALB
                           |
                           v
                        Client
```

The exact filter order can vary by route/configuration; the important architectural point is that the gateway applies configured filters before forwarding requests.

---

# 5.27 Gateway and Backend Service Responsibilities

The gateway should not become a giant business-logic service.

### Gateway should handle:

```text
Routing
Authentication-related filtering
Rate limiting
Resilience
Path transformation
Cross-cutting policies
```

### Backend services should handle:

```text
Business rules
Database operations
Banking logic
Transaction state
Account validation
KYC rules
Business authorization
```

For example:

```text
Gateway
   |
   | Is JWT valid?
   v
YES
   |
   v
Account Service
   |
   | Is withdrawal allowed?
   | Is balance sufficient?
   | Is account active?
   v
Business decision
```

This separation keeps the gateway lightweight and focused on cross-cutting concerns.

---

# 5.28 Gateway vs Load Balancer

This is a common interview question.

## ALB

The Application Load Balancer operates primarily at the infrastructure/load-balancing layer.

```text
Internet
   |
   v
ALB
   |
   v
Gateway ECS Task
```

It handles things such as:

* Receiving external traffic
* Target health checks
* Distributing traffic to healthy targets

## API Gateway

The API Gateway operates at the application/API layer.

```text
API Gateway
   |
   +-- Route
   +-- Rewrite
   +-- JWT validation
   +-- Rate limit
   +-- Circuit breaker
   +-- Retry
```

Therefore:

```text
ALB
= Infrastructure traffic distribution

API Gateway
= API-level routing and policies
```

The CitiCore architecture deliberately uses both layers.

---

# 5.29 Why Not Expose Every Microservice Through the ALB?

Suppose we exposed:

```text
ALB
 |
 +-- Auth
 +-- User
 +-- Account
 +-- Transaction
```

The client would need to understand every service endpoint.

Instead:

```text
ALB
 |
 v
Gateway
 |
 +-- Auth
 +-- User
 +-- Account
 +-- Transaction
```

This provides:

* One external endpoint
* Centralized security controls
* Centralized rate limiting
* Centralized resilience policies
* Less direct exposure of internal services
* Easier API evolution

The source explicitly lists the single endpoint, centralized rate limiting, circuit breaking, retry logic, security policies, and API abstraction as gateway benefits.

---

# 5.30 Real CitiCore Gateway / Networking Issue

One of the important practical problems encountered in CitiCore was not a Java coding problem.

The application was healthy inside the ECS task, but external requests were failing.

The investigation followed the request path:

```text
Application
     |
     v
ECS Container
     |
     v
ECS Networking
     |
     v
ALB Target
     |
     v
External Client
```

The problem was related to ALB/security-group configuration.

The ALB was associated with the ECS security group instead of the expected dedicated ALB security group.

Conceptually:

```text
Expected:

Internet
   |
   v
ALB [ALB SG]
   |
   v
ECS [ECS SG]
```

But the configuration effectively had:

```text
Internet
   |
   v
ALB [ECS SG]
   |
   X
Expected ALB → ECS SG relationship not satisfied
```

Correcting the security-group association allowed the target to become healthy and external traffic to succeed.

### Lesson

A gateway timeout does not automatically mean the gateway code is broken.

Always debug layer by layer:

```text
1. Application
2. Container
3. ECS networking
4. Security Groups
5. ALB
6. Gateway
7. Backend service
```

This is one of the most useful real-world lessons from the project.

---

# 5.31 Gateway Health vs Business Health

A service can report:

```text
/actuator/health
     |
     v
200 OK
```

while a real business request still fails.

For example:

```text
Gateway
   |
   v
Transaction Service
   |
   v
Kafka dependency
   |
   X
Missing/unavailable topic
```

The application may still be technically running even though the business workflow cannot complete.

Therefore:

```text
Application Health
       ≠
Business Request Success
```

The project's deployment documentation explicitly checks infrastructure dependencies such as Kafka, database connectivity, ALB health, and actual API behavior separately.

---

# 5.32 Gateway Failure Scenarios

## Scenario 1 — Backend temporarily unavailable

```text
Client
  |
  v
Gateway
  |
  v
Backend
  |
  X
503
```

Gateway may use:

```text
Retry
   |
   v
Circuit Breaker
```

depending on the configured route and failure type.

---

## Scenario 2 — Backend continuously failing

```text
Backend
   |
   X
Repeated failures
   |
   v
Circuit OPEN
   |
   v
Fast 503
```

This prevents continuous downstream calls.

---

## Scenario 3 — Client exceeds rate limit

```text
Client
  |
  | Too many requests
  v
Gateway
  |
  v
Rate Limiter
  |
  X
Request rejected
```

---

## Scenario 4 — Invalid JWT

```text
Client
  |
  | Invalid JWT
  v
Gateway
  |
  X
401 Unauthorized
```

The request does not need to reach the backend service.

---

## Scenario 5 — Network failure

```text
Client
  |
  v
ALB
  |
  X
Gateway unreachable
```

This requires infrastructure-level troubleshooting rather than changing backend business logic.

---

# 5.33 Gateway Observability

Gateway monitoring should focus on both traffic and downstream behavior.

Useful metrics include:

```text
Request count
Latency
p50
p95
p99
4xx responses
5xx responses
Circuit breaker state
Rate-limit rejections
Downstream failures
```

The project's monitoring notes identify ALB request count, latency, 4xx/5xx rates, target health, and gateway circuit-breaker/rate-limiter state as useful operational signals.

A practical monitoring view is:

```text
                   Gateway
                      |
       +--------------+--------------+
       |              |              |
       v              v              v
    Traffic        Errors         Latency
       |              |              |
       v              v              v
 Request/sec       4xx/5xx       p95/p99
                      |
                      v
              Circuit Breaker
```

---

# 5.34 Gateway Security Model

The external security boundary should look approximately like:

```text
Internet
   |
   v
ALB
   |
   v
API Gateway
   |
   | JWT validation
   v
Internal Services
   |
   v
Private infrastructure
```

The goal is to prevent clients from directly depending on internal service addresses.

For production, the project documentation identifies HTTPS/ACM and moving ECS tasks into private subnets as planned improvements.

Therefore, the current implementation should be distinguished from the target production architecture.

---

# 5.35 Current vs Production Gateway Architecture

## Current documented setup

```text
Internet
   |
   v
HTTP ALB :8080
   |
   v
Gateway
   |
   v
ECS Services
```

The project notes also identify current ECS tasks in public subnets as a limitation.

## Production target

```text
Internet
   |
   v
HTTPS :443
   |
   v
ALB
   |
   v
Private ECS
   |
   v
Internal Services
```

The planned production improvements include ACM certificates, HTTP-to-HTTPS redirection, private ECS subnets, and no public IPs on ECS tasks.

---

# 5.36 Gateway Design Trade-offs

## Advantages

### 1. Single external endpoint

```text
Client
  |
  v
One endpoint
```

Simplifies client integration.

### 2. Centralized cross-cutting policies

```text
             Gateway
                |
     +----------+----------+
     |          |          |
    JWT       Rate       Retry
              Limit
```

### 3. Service abstraction

Clients do not need to know internal service addresses.

### 4. Resilience

Circuit breakers and retries can protect downstream services.

### 5. API evolution

Path rewriting can separate external contracts from internal APIs.

---

## Disadvantages

### 1. Additional network hop

```text
Client
  |
  v
Gateway
  |
  v
Service
```

The gateway adds some latency compared with directly calling a service.

### 2. Gateway becomes critical infrastructure

If the gateway is unavailable:

```text
Client
   |
   X
Gateway
   |
   X
All APIs
```

Therefore the gateway itself must be highly available.

### 3. Configuration complexity

Routing, retry, circuit breaker, rate limiting, authentication filters, and networking introduce additional configuration.

### 4. Risk of becoming a bottleneck

Poorly configured gateway resources can affect every API.

### 5. Retry can create duplicate operations

This is especially dangerous for non-idempotent banking operations.

---

# 5.37 Gateway Request Mental Model

For interviews, remember this:

```text
                REQUEST
                   |
                   v
             Route Matching
                   |
                   v
            Authentication
                   |
                   v
             Rate Limiting
                   |
                   v
           Circuit Breaker
                   |
                   v
                Retry
                   |
                   v
            Path Rewrite
                   |
                   v
          Internal Service
                   |
                   v
              RESPONSE
```

The exact filter execution order depends on the route configuration, but conceptually these are the gateway concerns surrounding the backend request.

---

# 5.38 API Gateway Configuration Summary

| Concern              | CitiCore Implementation |
| -------------------- | ----------------------- |
| Framework            | Spring Cloud Gateway    |
| Port                 | 8080                    |
| External entry       | Gateway ALB             |
| `/auth/**`           | Auth Service            |
| `/users/**`          | User Service            |
| `/accounts/**`       | Account Service         |
| `/transaction/**`    | Transaction Service     |
| Path rewriting       | Yes                     |
| JWT validation       | Gateway                 |
| Rate limiting        | Redis-backed, per IP    |
| Rate                 | 2 requests/sec          |
| Burst                | 4                       |
| Circuit breaker      | Resilience4j            |
| Failure threshold    | 50%                     |
| Sliding window       | 10                      |
| Open duration        | 10 sec                  |
| Half-open test calls | 2                       |
| Retry                | 1 retry / 2 attempts    |
| Retry statuses       | Configured 5xx statuses |
| Backoff              | 100ms → 500ms           |
| Configuration        | Spring Cloud Config     |
| Internal networking  | ECS Service Connect     |
| Deployment           | ECS/Fargate             |

The numerical values above reflect the project's documented configuration rather than generic production defaults.

---

# 5.39 How I Would Explain the API Gateway in an Interview

> "In CitiCore, I use Spring Cloud Gateway as the single external entry point for the microservices. External traffic first reaches an ALB and is forwarded to the gateway.
>
> The gateway uses path-based routing to forward requests to Auth, User, Account and Transaction services. I also use path rewriting so that external APIs such as `/accounts/create` can map to the internal `/api/v1/accounts/create` endpoint.
>
> The gateway handles common cross-cutting concerns such as JWT validation, Redis-backed rate limiting, circuit breaking and retry handling. The rate limiter is configured per client IP, while Resilience4j is used for circuit-breaker and retry behavior.
>
> For internal ECS communication, the gateway uses Service Connect DNS rather than relying on changing task IPs. Gateway configuration is externalized through Spring Cloud Config.
>
> One practical issue I encountered was an ALB security-group configuration problem where the application itself was healthy but the ALB target was unhealthy. Debugging the request path layer by layer helped identify the networking issue rather than incorrectly changing the gateway code."

---

# 5.40 Common Interview Questions

## Q1. Why do you need an API Gateway?

**Answer:**

Because multiple microservices should not all be directly exposed to external clients. The gateway provides a single entry point and centralizes routing, authentication-related filtering, rate limiting and resilience policies.

---

## Q2. Why do you have both ALB and API Gateway?

**Answer:**

They operate at different layers.

The ALB handles infrastructure-level traffic distribution and target health, while the API Gateway handles application-level API routing and policies.

```text
ALB
= Load balancing

Gateway
= API management/routing
```

---

## Q3. Why use path rewriting?

**Answer:**

It separates the external API contract from the internal service API structure.

For example:

```text
External:
POST /accounts/create

Internal:
POST /api/v1/accounts/create
```

The client does not need to know the internal path structure.

---

## Q4. Why use Redis for rate limiting?

**Answer:**

Because the gateway can have multiple instances. Redis provides shared rate-limit state so the limit is not maintained independently by each gateway instance.

---

## Q5. What is the difference between retry and circuit breaker?

**Answer:**

Retry handles temporary failures by trying the request again.

Circuit breaker handles repeated failures by temporarily stopping calls to the unhealthy service.

```text
Retry
Temporary failure → Try again

Circuit Breaker
Repeated failure → Stop calling
```

---

## Q6. Why is blindly retrying a POST dangerous?

**Answer:**

Because the first request may have succeeded even though the response was lost. Retrying can execute the same business operation twice.

For banking operations, APIs should use idempotency mechanisms where retries are required.

---

## Q7. What happens when the circuit breaker is OPEN?

**Answer:**

Requests are fast-failed instead of being forwarded to the unhealthy backend. After the configured wait period, the circuit enters HALF_OPEN and allows test requests to determine whether the service has recovered.

---

## Q8. What happens if the gateway itself goes down?

**Answer:**

The gateway is a critical entry point, so its availability is important. The ALB can distribute traffic across multiple gateway tasks when multiple tasks are deployed. In the documented current setup, services generally run with a desired count of one, so high availability is a known limitation and future improvement.

---

## Q9. Should all authorization happen at the gateway?

**Answer:**

No. The gateway can perform common authentication/token validation, but business-specific authorization should remain inside the owning service because the service understands the business rules.

---

## Q10. Why not put business logic in the gateway?

**Answer:**

That would make the gateway tightly coupled to business services and could turn it into a distributed monolith. The gateway should primarily handle cross-cutting concerns, while business logic belongs to the respective microservices.

---

## Q11. How do you troubleshoot a gateway 504?

**Answer:**

I don't immediately assume the gateway code is the problem. I trace the request layer by layer:

```text
Client
  ↓
ALB
  ↓
Gateway
  ↓
Network / Service Connect
  ↓
Backend Service
  ↓
Database / Kafka / Redis
```

Then I check:

```text
ALB target health
Gateway logs
Backend logs
Service Connect DNS
Security Groups
Backend health
Downstream dependencies
Timeout configuration
```

---

## Q12. Can a service be healthy while a gateway request fails?

**Answer:**

Yes. `/actuator/health` can report the application as healthy while a particular business dependency or workflow is failing. Therefore health checks and real API testing should both be performed.

---

# 5.41 Final API Gateway Architecture

```text
                         INTERNET
                            |
                            v
                    +---------------+
                    |  Gateway ALB  |
                    |     :8080     |
                    +-------+-------+
                            |
                            v
                    +---------------+
                    | API GATEWAY   |
                    | Spring Cloud  |
                    | Gateway       |
                    +-------+-------+
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
          /auth/**       /users/**     /accounts/**
             |              |              |
             v              v              v
           AUTH            USER          ACCOUNT
                                            |
                                            v
                                      TRANSACTION
                                            |
             +------------------------------+
             |
             v
        Cross-Cutting Controls
             |
     +-------+-------+-------+-------+
     |       |       |       |       |
     v       v       v       v       v
    JWT    Rate    Circuit   Retry  Rewrite
          Limit   Breaker
             |
             v
           Redis
```

Internal communication:

```text
API Gateway
     |
     v
ECS Service Connect
     |
     +----> Auth
     +----> User
     +----> Account
     +----> Transaction
```

Configuration:

```text
Git Repository
      |
      v
Config Server
      |
      v
API Gateway
```

---

# 5.42 Core Principle

The API Gateway should be remembered as:

```text
                  API GATEWAY
                       |
       +---------------+---------------+
       |               |               |
       v               v               v
    ROUTING         SECURITY       RESILIENCE
       |               |               |
       v               v               v
 Path rewrite        JWT        Rate limiting
 External → Internal            Circuit breaker
                                Retry
```

The most important interview point is **not**:

> "I used Spring Cloud Gateway."

It is:

> **"I used the API Gateway as a controlled external entry point, keeping business services internally focused while centralizing routing, authentication-related filtering, rate limiting and resilience. I also learned that gateway failures must be debugged across the complete network path rather than assuming the gateway application itself is responsible."**

---

# 5.43 Topic Summary

CitiCore's API Gateway provides:

```text
Single external entry point
        +
Path-based routing
        +
Path rewriting
        +
JWT validation
        +
Redis-backed rate limiting
        +
Circuit breaker
        +
Retry with backoff
        +
Internal Service Connect routing
        +
Centralized configuration
        =
Controlled Microservice API Layer
```

The gateway is therefore the **boundary between external API consumers and internal microservices**, while the actual banking business logic remains inside the individual services.

---
# TOPIC 6 — KAFKA

# 6.1 What is Apache Kafka?

## Concept

Apache Kafka is a distributed event-streaming platform used in CitiCore for **asynchronous communication between microservices**.

Instead of one service directly waiting for another service to complete:

```text
Auth Service
     |
     | HTTP request
     v
Notification Service
```

CitiCore can use:

```text
Auth Service
     |
     | Event
     v
   Kafka
     |
     v
Notification Service
```

The producer does not need the consumer to be immediately available.

The project uses Kafka for:

* Asynchronous communication
* Service decoupling
* Transaction workflow coordination
* KYC events
* OTP events
* Account/transaction events
* Failure/result events
* Event retention and replay

The source notes explicitly identify asynchronous messaging, service decoupling, replay capability, Saga coordination, and eventual consistency as Kafka's purposes in CitiCore.

---

# 6.2 Why CitiCore Uses Kafka

Consider OTP delivery.

Without Kafka:

```text
Client
  |
  v
Auth Service
  |
  | HTTP
  v
Notification Service
  |
  v
Email Provider
```

If the notification service or email provider is slow:

```text
Auth Service
     |
     X
Notification slow
     |
     v
Registration request becomes slow
```

With Kafka:

```text
Client
  |
  v
Auth Service
  |
  | Publish OTP Event
  v
Kafka
  |
  v
Response can continue
```

Later:

```text
Kafka
  |
  v
Notification Service
  |
  v
Email Provider
```

The project notes specifically describe Kafka as solving the synchronous dependency between Auth and Notification and allowing events to remain available when consumers are unavailable.

---

# 6.3 Kafka Architecture in CitiCore

The simplified architecture is:

```text
                       Kafka Broker
                       EC2 :9092
                            |
        +-------------------+-------------------+
        |                   |                   |
        v                   v                   v
   otp-topic          kyc-topic           transaction
                                             topics
        ^                   ^                   ^
        |                   |                   |
        |                   |                   |
   Auth Service        User Service      Transaction /
                                         Account Service
        |                   |                   |
        +-------------------+-------------------+
                            |
                            v
                    Notification Service
```

For transaction processing, Kafka sits between Transaction Service and Account Service:

```text
Transaction Service
        |
        | DEBIT/CREDIT
        v
      Kafka
        |
        v
Account Service
        |
        | SUCCESS/FAILED
        v
      Kafka
        |
        v
Transaction Service
```

This is the core event-driven communication model.

---

# 6.4 CitiCore Kafka Deployment

The documented implementation runs Kafka on a dedicated EC2 instance.

```text
EC2
|
+-- Instance: citicore-infra
+-- Type: t3.small
+-- Kafka Port: 9092
+-- Docker Container
+-- Image: apache/kafka:4.0.1
+-- Mode: KRaft
```

The Kafka broker uses:

```text
Bootstrap Server:
10.0.1.87:9092
```

The source explicitly documents Kafka `4.0.1`, KRaft mode, port `9092`, and the EC2 private IP as the bootstrap server.

---

# 6.5 What is KRaft?

Modern Kafka can run without ZooKeeper using **KRaft**.

The CitiCore Kafka deployment uses:

```text
Kafka
  |
  v
KRaft
  |
  X
No ZooKeeper
```

The documented Docker configuration sets the process roles as:

```text
broker
controller
```

and configures the controller quorum accordingly.

For this project, that means Kafka and its controller functionality are managed without deploying a separate ZooKeeper service.

---

# 6.6 Kafka Core Components

The important Kafka concepts for CitiCore are:

```text
Producer
    |
    v
Topic
    |
    v
Partition
    |
    v
Consumer
    |
    v
Consumer Group
    |
    v
Offset
```

Let's understand each one.

---

# 6.7 Producer

A producer publishes events to Kafka.

Examples:

```text
Auth Service
     |
     v
OTP Event
     |
     v
otp-topic
```

and:

```text
User Service
     |
     v
KycEvent
     |
     v
kyc-topic
```

The source documents Auth Service publishing `VerificationOtpEvent` to `otp-topic` and User Service publishing `KycEvent` to `kyc-topic`.

---

# 6.8 Consumer

A consumer reads events from Kafka.

For example:

```text
Kafka
   |
   | otp-topic
   v
Notification Service
   |
   v
Send OTP email
```

Another example:

```text
Kafka
   |
   | debit-topic
   v
Account Service
   |
   v
Debit account
```

Consumers do not need to be running at the exact moment an event is produced. Kafka retains events according to the configured retention policy.

---

# 6.9 Topic

A topic is a logical stream/category of events.

Think of it as:

```text
Kafka
 |
 +-- otp-topic
 |
 +-- kyc-topic
 |
 +-- debit-topic
 |
 +-- credit-topic
 |
 +-- debit-success-topic
 |
 +-- debit-failed-topic
 |
 +-- credit-success-topic
 |
 +-- credit-failed-topic
 |
 +-- reversal-topic
 |
 +-- reversal-success-topic
```

Each topic represents a particular type of event or workflow.

---

# 6.10 Partition

A topic is divided into partitions.

Conceptually:

```text
debit-topic
    |
    +-- Partition 0
    +-- Partition 1
    +-- Partition 2
```

Kafka stores records inside partitions.

The documented Kafka setup currently creates topics with:

```text
Partitions: 1
Replication factor: 1
```

for the initial EC2 setup.

This is appropriate to describe as the **current documented project setup**, not as a production-scale Kafka configuration.

---

# 6.11 Why Partitions Matter

Partitions provide scalability and parallelism.

Suppose:

```text
Topic
 |
 +-- P0
 +-- P1
 +-- P2
```

Multiple consumers can process different partitions in parallel.

```text
Partition 0 → Consumer 1

Partition 1 → Consumer 2

Partition 2 → Consumer 3
```

Therefore:

```text
More partitions
       |
       v
More potential parallelism
```

However, simply creating many partitions does not automatically make an application faster. Consumers, workload, ordering requirements, and broker resources must also be considered.

---

# 6.12 Consumer Group

A consumer group is a logical group of consumers working together.

Example:

```text
notification-group

Kafka
 |
 +-- P0 → Consumer 1
 |
 +-- P1 → Consumer 2
 |
 +-- P2 → Consumer 3
```

Within one consumer group, a partition is processed by only one consumer at a time.

This allows Kafka to distribute work.

---

# 6.13 Consumer Group in CitiCore

The documented Transaction Service consumer uses:

```yaml
spring:
  kafka:
    consumer:
      group-id: transaction-group
      enable.auto.commit: false
      auto.offset.reset: earliest
```

The Notification Service uses:

```text
notification-group
```

The project notes explicitly document manual commits and stable consumer-group IDs.

---

# 6.14 Offset

An offset identifies the position of a record within a partition.

Conceptually:

```text
Partition 0

Offset 0 → Event A
Offset 1 → Event B
Offset 2 → Event C
Offset 3 → Event D
```

The consumer tracks how far it has processed.

```text
Processed:
0
1
2
     |
     v
Next:
3
```

This allows Kafka consumers to resume processing after restart.

---

# 6.15 Manual Offset Commit

CitiCore uses:

```yaml
enable.auto.commit: false
```

This means the application controls when the offset is committed.

Conceptually:

```text
Receive event
     |
     v
Process event
     |
     +---- Failure → don't acknowledge successfully
     |
     v
Commit offset
```

This is useful because the consumer should not move its position before the business operation has successfully completed.

The documented configuration explicitly uses manual commit.

---

# 6.16 `auto.offset.reset: earliest`

CitiCore configures:

```yaml
auto.offset.reset: earliest
```

This means that when a consumer group has no existing committed offset, the consumer starts from the earliest available record.

Conceptually:

```text
Kafka
 |
 +-- Event 1
 +-- Event 2
 +-- Event 3
       ^
       |
Consumer starts here
       |
       v
Process Event 1 → 2 → 3
```

This is useful for the project's event-processing and phased deployment scenarios.

---

# 6.17 Important Kafka Behavior — Consumer Does Not Need to Exist

A common misconception is:

> "If the Notification Service is down, Kafka cannot accept the event."

That is incorrect.

The documented CitiCore architecture demonstrates:

```text
Auth Service
    |
    v
Kafka
    |
    | Event stored
    |
    X
Notification Service currently unavailable
```

The event remains in Kafka according to the retention policy.

Later:

```text
Notification Service starts
       |
       v
Reads pending events
       |
       v
Processes them
```

The source explicitly documents this behavior and uses it as a reason phased deployment is possible.

---

# 6.18 OTP Event Architecture

The Auth Service publishes an OTP event.

Flow:

```text
User
 |
 | Register
 v
Auth Service
 |
 +-- Validate input
 +-- Hash password
 +-- Save user
 +-- Generate OTP
 +-- Create VerificationOtpEvent
 |
 v
Kafka
 |
 v
otp-topic
 |
 v
Notification Service
 |
 v
Email Provider
 |
 v
User receives OTP
```

The documented event contains:

```text
email
otp
timestamp
```

and is published to:

```text
otp-topic
```

---

# 6.19 KYC Event Architecture

The User Service publishes KYC status changes.

```text
User
 |
 | Upload KYC document
 v
User Service
 |
 +-- Save document to S3
 +-- Update MySQL status
 |
 v
Create KycEvent
 |
 v
Kafka
 |
 v
kyc-topic
 |
 v
Notification Service
 |
 v
Email notification
```

The documented `KycEvent` contains:

```text
userId
email
status
timestamp
```

and is serialized to JSON before being published.

---

# 6.20 Transaction Event Architecture

This is the most important Kafka workflow in CitiCore.

The transaction workflow uses multiple topics.

## Command Topics

```text
debit-topic
credit-topic
reversal-topic
```

## Result Topics

```text
debit-success-topic
debit-failed-topic

credit-success-topic
credit-failed-topic

reversal-success-topic
```

The project documentation defines these topic responsibilities explicitly.

---

# 6.21 Debit Flow

```text
Transaction Service
       |
       | DEBIT event
       v
  debit-topic
       |
       v
 Account Service
       |
       | Debit sender account
       |
       +---- Success
       |       |
       |       v
       | debit-success-topic
       |
       +---- Failure
               |
               v
       debit-failed-topic
```

The event contains information such as:

```text
transactionId
amount
```

The Account Service publishes the result.

---

# 6.22 Credit Flow

```text
Transaction Service
       |
       | CREDIT event
       v
  credit-topic
       |
       v
 Account Service
       |
       | Credit receiver account
       |
       +---- Success
       |       |
       |       v
       | credit-success-topic
       |
       +---- Failure
               |
               v
       credit-failed-topic
```

This allows Transaction Service to update the transaction state based on the Account Service result.

---

# 6.23 Reversal Flow

If the debit succeeded but credit failed:

```text
Debit
  |
  v
SUCCESS
  |
  v
Credit
  |
  X
FAILED
  |
  v
Reversal
```

Transaction Service creates a reversal event:

```text
Transaction Service
       |
       v
reversal-topic
       |
       v
Account Service
       |
       | Reverse debit
       v
reversal-success-topic
       |
       v
Transaction Service
       |
       v
REVERSED
```

The project explicitly documents this compensation flow.

The detailed reliability mechanism that ensures these events are not lost is the **Transactional Outbox**, which is the next topic.

---

# 6.24 Complete Transaction Event Flow

```text
                    TRANSACTION SERVICE
                            |
                            |
                      DEBIT EVENT
                            |
                            v
                     debit-topic
                            |
                            v
                     ACCOUNT SERVICE
                            |
                  +---------+---------+
                  |                   |
              SUCCESS               FAILURE
                  |                   |
                  v                   v
        debit-success-topic   debit-failed-topic
                  |                   |
                  v                   v
          Transaction Service   Transaction FAILED
                  |
                  v
              CREDIT EVENT
                  |
                  v
            credit-topic
                  |
                  v
           Account Service
                  |
             +----+----+
             |         |
          SUCCESS    FAILURE
             |         |
             v         v
    credit-success   credit-failed
             |         |
             v         v
       COMPLETED    FAILED
                         |
                         v
                  REVERSAL EVENT
                         |
                         v
                  reversal-topic
                         |
                         v
                  Account Service
                         |
                         v
             reversal-success-topic
                         |
                         v
                     REVERSED
```

This is the core event-driven transaction workflow documented in CitiCore.

---

# 6.25 Kafka Event Model

The project contains a shared:

```text
kafka-events/
```

module.

Its purpose is to provide:

* Shared event definitions
* Type-safe event structures
* Event schema versioning
* Single source of truth

The documented example is:

```java
public class TransactionEvent {

    private String transactionId;
    private String type;
    private BigDecimal amount;
    private String senderAccount;
    private String receiverAccount;
    private LocalDateTime timestamp;
}
```

The source explicitly identifies the shared `kafka-events` module as the location for common event definitions and schema management.

---

# 6.26 Why a Shared Event Module?

Without a shared event definition:

```text
Transaction Service
    |
    | Own event class
    v
Kafka
    |
    v
Account Service
    |
    | Different event class
```

There is a greater risk of incompatible schemas.

With:

```text
             kafka-events
                  |
        +---------+---------+
        |         |         |
        v         v         v
 Transaction   Account   Notification
```

services can use a common event contract.

The project notes also connect changes to this shared module with CI/CD behavior: if `kafka-events` changes, consumers may need to be rebuilt/deployed.

---

# 6.27 Kafka Serialization

Kafka transfers bytes, so Java objects must be serialized.

```text
Java Object
     |
     v
Serializer
     |
     v
Bytes
     |
     v
Kafka
```

The project documents two producer strategies.

### JSON Producer

Used for dead-letter publishing:

```text
Java Object
     |
     v
JsonSerializer
     |
     v
Kafka
```

### String Producer

Used by the Outbox publisher where the event payload is already represented as JSON.

The source explicitly documents the JSON producer for DLT messages and the String producer for outbox events.

---

# 6.28 Producer Reliability Configuration

The documented producer configuration includes:

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}

    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      acks: all
      enable.idempotence: true
      retries: 10
      retry.backoff.ms: 1000
```

Important settings:

```text
acks=all
        ↓
Wait for broker acknowledgement

enable.idempotence=true
        ↓
Reduce duplicate records caused by producer retries

retries=10
        ↓
Retry transient producer failures

retry.backoff.ms=1000
        ↓
Wait before retrying
```

These values are documented in the project configuration.

---

# 6.29 Why `acks=all`?

`acks=all` means the producer waits for the broker's configured acknowledgement requirements rather than treating the send as successful immediately.

Conceptually:

```text
Producer
   |
   | Event
   v
Kafka
   |
   | Acknowledgement
   v
Producer
```

For banking-related events, the project chooses stronger producer acknowledgement behavior.

However, `acks=all` does not magically guarantee end-to-end exactly-once business processing. Producer acknowledgement and business-level processing are different concerns.

That distinction is important in interviews.

---

# 6.30 Idempotent Producer

The project enables:

```text
enable.idempotence=true
```

The purpose is to reduce duplicate records caused by producer retries.

Conceptually:

```text
Producer
   |
   | Event
   v
Kafka
   |
   X
Acknowledgement lost
   |
   v
Producer retries
```

Without idempotence, retrying could result in duplicate records.

With producer idempotence enabled, Kafka can identify retried producer records appropriately.

But again:

```text
Producer idempotence
        ≠
Business operation idempotence
```

For example, creating a bank account twice is a business-level duplication problem that Kafka producer idempotence alone cannot solve.

---

# 6.31 Consumer Reliability

The project uses:

```text
enable.auto.commit=false
```

along with retry and DLT processing.

The conceptual flow is:

```text
Kafka Event
     |
     v
Consumer
     |
     v
Process
     |
 +---+---+
 |       |
Success Failure
 |       |
 v       v
Commit  Retry
         |
         v
      Failure again
         |
         v
        DLT
```

This prevents a permanently failing message from blocking normal processing forever.

---

# 6.32 Kafka Retry Configuration

The documented listener configuration uses:

```yaml
spring:
  kafka:
    listener:
      ack-mode: manual_immediate
      error-handler: defaultErrorHandler

defaultErrorHandler:
  retry:
    enabled: true
    max-attempts: 3
    backoff:
      delay: 2000ms
      multiplier: 2.0
      max-delay: 16000ms
```

The documented retry sequence is:

```text
Attempt 1
    |
    X
    |
Wait 2 sec
    |
Attempt 2
    |
    X
    |
Wait 4 sec
    |
Attempt 3
    |
    X
    |
    v
DLT
```

The source explicitly documents three attempts with exponential backoff and DLT handling.

---

# 6.33 Dead Letter Topic

A Dead Letter Topic, or DLT, stores messages that cannot be successfully processed after the configured retry attempts.

Example:

```text
credit-success-topic
        |
        v
Consumer
        |
        X
Processing failure
        |
        v
Retry 1
        |
        X
Retry 2
        |
        X
Retry 3
        |
        v
credit-success-topic.DLT
```

The documented project uses DLTs such as:

```text
debit-topic.DLT
credit-topic.DLT
reversal-topic.DLT
```

and similar DLT topics for other event streams.

---

# 6.34 Why DLT is Important

Without a DLT:

```text
Bad Message
    |
    v
Consumer
    |
    X
Failure
    |
    v
Retry forever
    |
    X
Consumer blocked
```

With DLT:

```text
Bad Message
    |
    v
Retry
    |
    X
Still failing
    |
    v
DLT
    |
    v
Normal processing continues
```

This separates permanently problematic messages from normal traffic.

---

# 6.35 CitiCore DLT Storage

The project also stores DLT information in a database table:

```sql
CREATE TABLE dead_letter_events (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  topic VARCHAR(100),
  partition INT,
  offset BIGINT,
  payload TEXT,
  error_message TEXT,
  exception_class VARCHAR(255),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

The DLT consumer records:

```text
Topic
Partition
Offset
Payload
Exception
Timestamp
```

The documented DLT consumer and table schema are intended to make failed events available for investigation and recovery.

---

# 6.36 DLT Recovery Process

The documented recovery workflow is:

```text
DLT
 |
 v
Inspect dead_letter_events
 |
 v
Identify root cause
 |
 +---- Network issue
 |
 +---- Business logic issue
 |
 +---- Data problem
 |
 v
Fix problem
 |
 v
Replay event
 |
 v
Original topic
 |
 v
Consumer
```

This is useful operationally because a failed event is not simply discarded.

The source explicitly documents manual inspection, correction, replay, and monitoring after recovery.

---

# 6.37 Kafka Bootstrap Server Configuration

The services use:

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}
```

with:

```text
KAFKA_BOOTSTRAP_SERVERS=10.0.1.87:9092
```

The project notes explicitly warn that this should not be:

```text
localhost
127.0.0.1
```

when the application is running in a container.

Why?

```text
ECS Container
     |
     | localhost
     v
Same container
```

But Kafka is running elsewhere:

```text
ECS Container
     |
     | 10.0.1.87:9092
     v
Kafka EC2
```

The source explicitly documents the private EC2 IP as the required bootstrap address for the deployed containers.

---

# 6.38 Kafka Network Flow in AWS

The architecture is:

```text
ECS Task
   |
   | TCP :9092
   v
EC2 Kafka
   |
   v
Kafka Broker
```

Therefore the AWS network must allow:

```text
ECS Security Group
       |
       | outbound
       v
Kafka EC2
       |
       | inbound :9092
       v
Kafka
```

If the security group or network configuration blocks port `9092`:

```text
Producer
   |
   X
Kafka unreachable
```

This becomes a networking issue rather than a Java/Kafka code issue.

---

# 6.39 Real CitiCore Kafka Issue — Missing Topic

One of the most useful real issues in the project was:

```text
POST /api/v1/auth/register
        |
        v
Auth Service
        |
        v
Generate OTP
        |
        v
Publish otp-topic
        |
        X
Topic does not exist
        |
        v
Producer waits
        |
        v
504 Gateway Timeout
```

The investigation showed:

```text
ECS Task
   → RUNNING

ALB Target
   → HEALTHY

/actuator/health
   → 200

Database
   → User saved

OTP
   → Generated

Kafka
   → otp-topic missing
```

The root cause was the missing Kafka topic.

The source explicitly documents this troubleshooting sequence and identifies the missing `otp-topic` as the root cause of the timeout.

---

# 6.40 Lesson from the Kafka Timeout

This issue demonstrates:

```text
Application Health
       ≠
Business Operation Health
```

The application was technically running.

But:

```text
Registration
    |
    v
Kafka
    |
    X
Topic missing
    |
    v
Request hangs
```

Therefore when troubleshooting Kafka-related API failures, check both:

```text
Infrastructure health
+
Business workflow dependencies
```

The project notes explicitly identify this distinction as a key troubleshooting lesson.

---

# 6.41 Kafka Topic Creation

The documented setup disables automatic topic creation:

```text
KAFKA_AUTO_CREATE_TOPICS_ENABLE=false
```

Therefore topics must be created explicitly.

Example:

```bash
docker exec kafka bash
```

Then:

```bash
bin/kafka-topics.sh \
  --create \
  --topic otp-topic \
  --bootstrap-server localhost:9092 \
  --partitions 1 \
  --replication-factor 1
```

To verify:

```bash
bin/kafka-topics.sh \
  --list \
  --bootstrap-server localhost:9092
```

The project setup explicitly uses manual topic creation.

---

# 6.42 Kafka Retention

The documented Kafka configuration contains:

```text
KAFKA_LOG_RETENTION_HOURS=168
```

which corresponds to the configured retention period in the project setup.

The important concept is:

```text
Producer
   |
   v
Kafka
   |
   | Retains event
   |
   v
Consumer
```

Retention determines how long records remain available, subject to Kafka's storage policies.

Kafka retention should not be confused with consumer offsets.

```text
Retention
= How long Kafka keeps records

Offset
= Where a consumer has progressed
```

---

# 6.43 Kafka vs REST

This is an important design decision in CitiCore.

## REST / OpenFeign

Used when an immediate response is required.

Example:

```text
Transaction Service
       |
       | Validate account
       v
Account Service
       |
       v
Response immediately
```

## Kafka

Used when asynchronous processing is suitable.

Example:

```text
User Service
       |
       | KYC event
       v
Kafka
       |
       v
Notification Service
```

Therefore:

```text
Need immediate response?
        |
        v
REST / OpenFeign

Can process asynchronously?
        |
        v
Kafka
```

This is more important than simply saying "Kafka is faster."

---

# 6.44 Kafka vs Direct Service-to-Service Call

### Direct call

```text
Service A
    |
    | HTTP
    v
Service B
```

Advantages:

* Immediate response
* Simple request/response model
* Easy to understand

Disadvantages:

* Stronger runtime dependency
* Service B must be reachable
* Caller can be affected by B's latency/failure

### Kafka

```text
Service A
    |
    v
 Kafka
    |
    v
Service B
```

Advantages:

* Loose coupling
* Asynchronous processing
* Event retention
* Consumer scaling
* Consumer can recover later

Disadvantages:

* Eventual consistency
* More operational complexity
* Harder debugging
* Duplicate processing must be considered
* Ordering/partition design matters

---

# 6.45 Kafka and Eventual Consistency

Kafka-based workflows often mean the entire business operation does not complete in one synchronous request.

For example:

```text
Transaction = PENDING
        |
        v
DEBIT event
        |
        v
Debit processed
        |
        v
DEBIT_SUCCESS
        |
        v
CREDIT processed
        |
        v
COMPLETED
```

The transaction state changes over time.

Therefore:

```text
Immediate consistency
        X

Eventual consistency
        ✓
```

This is acceptable because the transaction state explicitly represents progress.

---

# 6.46 Kafka Ordering

Kafka guarantees ordering **within a partition**, not globally across every partition.

Conceptually:

```text
Partition 0

Event A
Event B
Event C
```

Consumers read them in partition order.

For transaction workflows, event keys can be used to keep related records associated with the same partition.

The project documentation shows transaction IDs being used in event payloads and keys in producer examples, but it does not provide enough source evidence to claim a finalized partitioning strategy for all transaction topics.

Therefore, do not claim that CitiCore currently guarantees global transaction ordering across all topics.

---

# 6.47 Duplicate Event Processing

Kafka applications should generally be designed with the possibility that an event can be processed more than once.

For example:

```text
Event
  |
  v
Consumer
  |
  | Process
  v
Database updated
  |
  X
Offset commit fails
  |
  v
Kafka redelivers
  |
  v
Same event processed again
```

Therefore consumers should use business-level idempotency where necessary.

For banking operations:

```text
transactionId
       |
       v
Check whether event already processed
       |
   +---+---+
   |       |
  Yes      No
   |       |
 Ignore   Process
```

This is particularly important for debit, credit and reversal events.

---

# 6.48 Kafka Does Not Automatically Give Exactly-Once Business Processing

This is an important interview distinction.

Do not say:

> "Kafka guarantees exactly-once transactions."

Instead:

```text
Kafka producer idempotence
        +
Acknowledgements
        +
Consumer offset control
        +
Application idempotency
        +
Transactional Outbox
        +
Compensation
```

together provide stronger reliability for the application's business workflow.

CitiCore specifically uses a Transactional Outbox to solve the database-to-Kafka dual-write problem. The detailed mechanism is intentionally covered in **Topic 7**.

---

# 6.49 Kafka and Transactional Outbox

The relationship is:

```text
Transaction Service
       |
       | DB Transaction
       v
+-----------------------+
| transactions          |
| outbox_events         |
+-----------------------+
       |
       v
Outbox Publisher
       |
       v
Kafka
       |
       v
Account Service
```

The Outbox pattern ensures that the business transaction and creation of the event record happen together before the event is published to Kafka.

The source documents this as the solution to the dual-write problem.

Detailed Outbox behavior belongs to the next topic.

---

# 6.50 Kafka Monitoring

Useful Kafka-related monitoring includes:

```text
Producer failures
Consumer failures
Consumer lag
Topic availability
Broker health
Partition health
DLT message count
Processing latency
Throughput
```

For CitiCore specifically, a practical operational check is:

```text
Kafka Broker
    |
    +-- Reachable?
    |
    +-- Topics exist?
    |
    +-- Producer connected?
    |
    +-- Consumer connected?
    |
    +-- Messages being consumed?
    |
    +-- DLT receiving failures?
```

Kafka should also be considered when an API request unexpectedly returns a timeout, as demonstrated by the real `otp-topic` incident.

---

# 6.51 Kafka Troubleshooting Checklist

If a Kafka-based operation fails:

### Step 1 — Check Kafka container

```bash
docker ps
```

### Step 2 — Check Kafka logs

```bash
docker logs kafka
```

### Step 3 — Check topics

```bash
docker exec kafka \
  bin/kafka-topics.sh \
  --list \
  --bootstrap-server localhost:9092
```

### Step 4 — Verify bootstrap server

Check:

```text
KAFKA_BOOTSTRAP_SERVERS
```

Ensure the deployed application uses the reachable Kafka address rather than:

```text
localhost
127.0.0.1
```

### Step 5 — Check security groups

Verify:

```text
ECS → EC2 Kafka :9092
```

is permitted.

### Step 6 — Check producer logs

Look for:

```text
Timeout
UnknownTopicOrPartition
Connection refused
Broker unavailable
Serialization error
```

### Step 7 — Check consumer

Verify:

```text
Consumer group
Topic subscription
Offset
Processing errors
```

### Step 8 — Check DLT

If the consumer repeatedly fails:

```text
Original topic
      |
      v
Retries
      |
      v
DLT
```

---

# 6.52 Kafka Deployment Flow

The documented Kafka setup can be summarized as:

```text
1. Create EC2 instance
        |
        v
2. Install Docker
        |
        v
3. Create Kafka docker-compose.yml
        |
        v
4. Configure KRaft
        |
        v
5. Start Kafka
        |
        v
6. Create topics
        |
        v
7. Configure application bootstrap server
        |
        v
8. Test producer
        |
        v
9. Test consumer
        |
        v
10. Test end-to-end event flow
```

---

# 6.53 Current Kafka Architecture vs Production Consideration

## Current documented setup

```text
              EC2
        +-------------+
        | Kafka       |
        | :9092       |
        | KRaft       |
        +-------------+
```

The documented setup uses a single broker/EC2 deployment with replication factor `1`.

That means it should **not** be presented as a highly available production Kafka cluster.

## Production direction

A production architecture would require considerations such as:

```text
Multiple Kafka brokers
        +
Replication
        +
Failure tolerance
        +
Monitoring
        +
Backup/recovery strategy
        +
Security
```

The source does not document a completed multi-broker production Kafka cluster, so this should be treated as a production improvement rather than a current CitiCore implementation.

---

# 6.54 Important Kafka Design Decisions

| Decision                      | CitiCore Approach         |
| ----------------------------- | ------------------------- |
| Communication style           | Event-driven asynchronous |
| Kafka deployment              | EC2 + Docker              |
| Kafka version                 | 4.0.1                     |
| Kafka mode                    | KRaft                     |
| Broker port                   | 9092                      |
| Bootstrap server              | EC2 private IP            |
| Topic creation                | Explicit/manual           |
| Auto topic creation           | Disabled                  |
| Initial partitions            | 1                         |
| Initial replication           | 1                         |
| Producer acknowledgement      | `acks=all`                |
| Producer idempotence          | Enabled                   |
| Producer retries              | 10                        |
| Consumer auto commit          | Disabled                  |
| Consumer offset reset         | `earliest`                |
| Transaction consumer group    | `transaction-group`       |
| Notification consumer group   | `notification-group`      |
| Consumer retry attempts       | 3                         |
| Retry backoff                 | Exponential               |
| DLT                           | Implemented/documented    |
| Shared events                 | `kafka-events` module     |
| Reliable DB→Kafka publication | Transactional Outbox      |

These values reflect the documented project configuration; they should not be treated as universal Kafka defaults.

---

# 6.55 How I Would Explain Kafka in an Interview

> "In CitiCore, I use Kafka for asynchronous communication between microservices. The main reason is to avoid tight runtime coupling. For example, Auth Service publishes an OTP event to Kafka instead of directly waiting for Notification Service to send an email.
>
> Kafka is also a major part of the transaction workflow. Transaction Service publishes debit and credit events, Account Service processes them and publishes success or failure events back, and Transaction Service updates the transaction state. If credit fails after debit succeeds, a reversal event is produced.
>
> For reliability, I use producer acknowledgements with `acks=all`, producer idempotence, retries, manual consumer acknowledgement and retry/DLT handling. The project also uses a shared `kafka-events` module for event definitions.
>
> One real issue I encountered was a registration API returning a 504 even though the ECS task and ALB were healthy. The root cause was that the `otp-topic` did not exist, so the Kafka publish operation was hanging. That taught me to troubleshoot distributed systems layer by layer rather than assuming an API timeout is an API Gateway problem."

---

# 6.56 Common Interview Questions

## Q1. Why Kafka instead of REST?

**Answer:**

REST is useful when the caller needs an immediate response. Kafka is useful when processing can happen asynchronously and I want to reduce runtime coupling between services.

---

## Q2. What happens if the consumer is down?

**Answer:**

Kafka can retain the event according to its retention policy. When the consumer comes back, it can consume the pending records based on its committed offsets.

---

## Q3. What is a consumer group?

**Answer:**

A consumer group is a group of consumers working together to process a topic. Kafka distributes partitions among consumers in the same group.

---

## Q4. What is an offset?

**Answer:**

An offset represents a record's position within a Kafka partition. Consumers track offsets so they know how far they have processed.

---

## Q5. Why disable auto commit?

**Answer:**

I want the application to control when processing is considered successful. The offset should be committed only after the required business processing has completed successfully.

---

## Q6. What is a DLT?

**Answer:**

A Dead Letter Topic stores messages that continue failing after the configured retry attempts. It allows normal processing to continue while failed messages can be investigated and replayed later.

---

## Q7. What is the difference between partition and consumer?

**Answer:**

A partition is a storage/ordering unit inside a Kafka topic. A consumer reads records from partitions. Multiple consumers in the same group can process different partitions in parallel.

---

## Q8. Does Kafka guarantee exactly-once processing?

**Answer:**

Kafka provides mechanisms such as idempotent producers and transactional features, but exactly-once business behavior is not something I would claim automatically. The application still needs appropriate idempotency and transaction design.

---

## Q9. Why use `acks=all`?

**Answer:**

It provides stronger producer acknowledgement semantics by waiting for the configured broker-side acknowledgement requirements before treating the send as successful.

---

## Q10. Why enable producer idempotence?

**Answer:**

It helps prevent duplicate Kafka records caused by producer retries. But it does not replace business-level idempotency.

---

## Q11. How do you handle a poison message?

**Answer:**

I allow a limited number of retries with backoff. If processing still fails, I send the event to a DLT, store relevant failure information, investigate the root cause, fix it, and replay the event.

---

## Q12. How do you maintain ordering?

**Answer:**

Kafka ordering is guaranteed within a partition. Therefore, related events that require ordering need an appropriate partitioning/key strategy. I would not claim global ordering across multiple partitions or topics.

---

## Q13. What happens if Kafka is unavailable when a database transaction succeeds?

**Answer:**

This is exactly the dual-write problem. In CitiCore, the Transactional Outbox solves it by storing the event in the same database transaction as the business change. A separate publisher later sends the event to Kafka.

---

## Q14. What real Kafka problem did you face?

**Answer:**

During registration, the API returned a 504 even though the application and ALB were healthy. After tracing the flow, I found that `otp-topic` didn't exist, so the producer was waiting while trying to publish. Creating the required topic fixed the issue.

---

# 6.57 Kafka Mental Model

For interviews, remember:

```text
              PRODUCER
                  |
                  v
                TOPIC
                  |
             +----+----+
             |         |
             v         v
          PARTITION  PARTITION
             |         |
             +----+----+
                  |
                  v
             CONSUMER
                  |
                  v
          PROCESS EVENT
                  |
             +----+----+
             |         |
          SUCCESS    FAILURE
             |         |
             v         v
          COMMIT     RETRY
                       |
                       v
                      DLT
```

And for CitiCore:

```text
REST / OpenFeign
=
Immediate synchronous interaction

Kafka
=
Asynchronous event interaction

Transactional Outbox
=
Reliable DB → Kafka publication

DLT
=
Failed event recovery
```

---

# 6.58 Final CitiCore Kafka Architecture

```text
                              KAFKA
                       EC2 :9092 / KRaft
                              |
       +----------------------+----------------------+
       |                      |                      |
       v                      v                      v
   otp-topic              kyc-topic             Transaction
                                                   Topics
       ^                      ^                      ^
       |                      |                      |
       |                      |                      |
   Auth Service          User Service       Transaction Service
       |                      |                      |
       |                      |                      |
       v                      v                      v
 Notification           Notification             Account Service
 Service                 Service                     |
                                                     |
                                                     v
                                              Result Events
                                                     |
                                                     v
                                             Transaction Service
```

Transaction topics:

```text
                   TRANSACTION SERVICE
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
        debit-topic   credit-topic  reversal-topic
             |             |             |
             +-------------+-------------+
                           |
                           v
                     ACCOUNT SERVICE
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
        debit-result   credit-result   reversal-result
             |             |             |
             +-------------+-------------+
                           |
                           v
                   TRANSACTION SERVICE
```

Reliability layer:

```text
Producer
   |
   +-- acks=all
   +-- idempotence
   +-- retries
   |
   v
Kafka
   |
   v
Consumer
   |
   +-- manual acknowledgement
   +-- retry
   +-- exponential backoff
   |
   +---- failure ----> DLT
```

---

# 6.59 Core Principle

The most important thing to remember about Kafka in CitiCore is:

```text
Kafka is not being used simply because
"Kafka is popular."

It is being used because:

Service A
   |
   | Event
   v
Kafka
   |
   v
Service B

allows:

Loose Coupling
      +
Asynchronous Processing
      +
Event Retention
      +
Consumer Scaling
      +
Failure Recovery
      +
Event-Driven Workflows
```

The strongest interview explanation is:

> **"I use Kafka where the producer should not have to wait for the consumer. In CitiCore, this is useful for OTP, KYC notifications and the distributed transaction workflow. I combine Kafka with manual consumer acknowledgement, retries, DLT handling and Transactional Outbox so that asynchronous communication remains reliable rather than simply fire-and-forget."**

---

# 6.60 Topic Summary

CitiCore Kafka provides:

```text
Asynchronous Communication
        +
Event-Driven Architecture
        +
Transaction Coordination
        +
OTP Events
        +
KYC Events
        +
Consumer Groups
        +
Manual Offset Management
        +
Retry + Backoff
        +
Dead Letter Topics
        +
Idempotent Producers
        +
Shared Event Contracts
        +
Event Retention
        =
Reliable Event-Driven Communication Layer
```

The key distinction to remember is:

```text
Kafka
→ transports and retains events

Transactional Outbox
→ makes database-to-Kafka publication reliable

Consumer Idempotency
→ protects business operations from duplicate processing

DLT
→ handles messages that repeatedly fail
```

---
## TOPIC 7 — TRANSACTIONAL OUTBOX

## 7.1 What is the Transactional Outbox Pattern?

## Concept

The **Transactional Outbox Pattern** is a reliability pattern used when a service needs to:

1. Update its database
2. Publish an event to Kafka

The problem is that the database and Kafka are two separate systems.

For example:

```text
Transaction Service
       |
       +----> MySQL
       |
       +----> Kafka
```

A normal database transaction cannot automatically make both operations succeed or fail together.

The Transactional Outbox pattern solves this by first storing the event in the **same database transaction** as the business data.

```text
Transaction Service
       |
       v
+---------------------------+
| Database Transaction      |
|                           |
| transactions              |
| outbox_events             |
+---------------------------+
       |
       | COMMIT
       v
Outbox Publisher
       |
       v
Kafka
```

The source documentation describes this as the solution to the database/Kafka dual-write problem.

---

# 7.2 Why CitiCore Needs Transactional Outbox

CitiCore uses Kafka for important banking events such as:

```text
DEBIT
CREDIT
REVERSAL
```

Consider a money-transfer request.

The Transaction Service needs to:

```text
1. Create transaction record
2. Publish DEBIT event
```

A naive implementation might do:

```text
BEGIN DB TRANSACTION
       |
       v
Save transaction
       |
       v
Commit
       |
       v
Publish Kafka event
```

This creates a dangerous failure window.

---

# 7.3 The Dual-Write Problem

Suppose:

```text
Save transaction
      |
      v
Database COMMIT ✓
      |
      v
Publish Kafka
      |
      X
Kafka unavailable
```

Now the database says:

```text
Transaction = PENDING
```

but Kafka never received:

```text
DEBIT event
```

The transaction can remain stuck.

The opposite problem is also possible:

```text
Publish Kafka ✓
      |
      X
Database operation fails
```

Now another service may process an event for a database operation that was not successfully committed.

Therefore:

```text
Database
   +
Kafka
   =
Two independent systems
```

A normal local database transaction cannot atomically commit both.

This is the **dual-write problem**.

---

# 7.4 Naive Implementation

A problematic implementation could look conceptually like:

```java
transactionRepository.save(transaction);

kafkaTemplate.send(
    "debit-topic",
    transactionEvent
);
```

There are two independent operations:

```text
Database Write
      |
      X
Kafka Publish
```

If one succeeds and the other fails, the systems become inconsistent.

---

# 7.5 CitiCore Solution

CitiCore uses:

```text
Business Data
     +
Outbox Event
     |
     v
Same Database Transaction
```

The flow is:

```text
Client
  |
  v
Transaction Service
  |
  v
BEGIN TRANSACTION
  |
  +---- Save transaction
  |
  +---- Save outbox event
  |
  v
COMMIT
  |
  v
Outbox Publisher
  |
  v
Kafka
```

The source explicitly documents:

```text
1. Save business transaction
2. Save event to outbox table
3. Both in same DB transaction
4. Separate process polls outbox
5. Publish confirmed events to Kafka
6. Mark event SENT
```

---

# 7.6 CitiCore Transactional Outbox Architecture

```text
                    TRANSACTION SERVICE
                            |
                            v
                    BEGIN DB TRANSACTION
                            |
                 +----------+----------+
                 |                     |
                 v                     v
          transactions          outbox_events
                 |                     |
                 |                  PENDING
                 |                     |
                 +----------+----------+
                            |
                            v
                         COMMIT
                            |
                            v
                  OutboxPublisher
                            |
                    Poll every 5 sec
                            |
                            v
                     PENDING events
                            |
                            v
                         Kafka
                            |
                            v
                    Account Service
```

This allows the business transaction and the event record to be committed atomically.

---

# 7.7 Transaction Service Example

In CitiCore, when a transfer is initiated:

```text
POST /api/v1/transactions/transfer
```

Transaction Service performs immediate validation.

The documented flow includes:

```text
1. Validate JWT
2. Check daily limits
3. Call Account Service
4. Validate transfer/balance
5. Save transaction as PENDING
6. Write DEBIT event to outbox
7. Commit database transaction
8. Return transactionId
```

The important part is:

```text
transactions
      +
outbox_events
      |
      v
Same DB transaction
```

---

# 7.8 Database Transaction

Conceptually:

```text
BEGIN TRANSACTION

INSERT INTO transactions
(
    id,
    sender_account_id,
    receiver_account_id,
    amount,
    status
)

INSERT INTO outbox_events
(
    transaction_id,
    event_type,
    payload,
    status
)

COMMIT
```

The source documents the initial outbox status as:

```text
PENDING
```

---

# 7.9 Why Save Both in One Transaction?

Suppose the database transaction succeeds:

```text
transactions       ✓
outbox_events      ✓
```

Now the event definitely exists in the database.

Even if Kafka is temporarily unavailable:

```text
Database
   |
   +-- transaction = PENDING
   |
   +-- outbox event = PENDING
   |
   v
Kafka unavailable
```

The event is not simply lost.

The publisher can attempt to send it later.

---

# 7.10 What Happens if the Database Transaction Fails?

Suppose:

```text
INSERT transaction
      ✓

INSERT outbox event
      X

ROLLBACK
```

Because both operations are inside the same database transaction:

```text
Transaction record
      |
      v
ROLLED BACK

Outbox event
      |
      v
ROLLED BACK
```

There is no committed transaction without its corresponding outbox event.

This is the key reliability property of the pattern.

---

# 7.11 Outbox Table

The documented CitiCore schema is:

```sql
CREATE TABLE outbox_events (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  transaction_id VARCHAR(36),
  event_type VARCHAR(50),
  payload JSON,
  status VARCHAR(20),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  published_at TIMESTAMP,
  FOREIGN KEY (transaction_id) REFERENCES transactions(id)
);
```

---

# 7.12 Outbox Table Fields

| Column           | Purpose                                         |
| ---------------- | ----------------------------------------------- |
| `id`             | Unique outbox event ID                          |
| `transaction_id` | Links event to business transaction             |
| `event_type`     | Event type such as DEBIT, CREDIT or REVERSAL    |
| `payload`        | Event data stored as JSON                       |
| `status`         | Current publication state                       |
| `created_at`     | Event creation timestamp                        |
| `published_at`   | Timestamp when event was successfully published |

The source documents the supported statuses as:

```text
PENDING
SENT
FAILED
```

---

# 7.13 Outbox Event State Machine

The lifecycle can be represented as:

```text
             CREATE
                |
                v
             PENDING
                |
                |
        Publisher attempts
                |
          +-----+-----+
          |           |
       Success      Failure
          |           |
          v           v
         SENT       FAILED
```

### PENDING

The event is stored in the database but has not yet been successfully confirmed by Kafka.

### SENT

Kafka acknowledged the event.

### FAILED

The publisher encountered an exception while attempting publication.

The current implementation documents failed events as candidates for later retry or manual intervention.

---

# 7.14 Outbox Publisher

The second part of the pattern is the **OutboxPublisher**.

Its responsibility is:

```text
Read PENDING events
       |
       v
Publish to Kafka
       |
       v
Wait for acknowledgement
       |
       v
Mark SENT
```

The CitiCore implementation runs the publisher every:

```text
5 seconds
```

The documented implementation uses:

```java
@Scheduled(fixedRate = 5000)
```

---

# 7.15 Outbox Publisher Flow

```text
OutboxPublisher
      |
      v
SELECT PENDING events
      |
      v
For each event
      |
      v
Publish to Kafka
      |
      v
Wait for acknowledgement
      |
      +------ Failure ------> FAILED
      |
      v
Kafka ACK
      |
      v
status = SENT
      |
      v
Save database record
```

The source documents this exact processing sequence.

---

# 7.16 Why Poll Every 5 Seconds?

CitiCore currently uses a scheduled polling approach:

```text
Every 5 seconds
       |
       v
Find PENDING events
       |
       v
Publish them
```

The 5-second interval is the project's configured implementation.

It should **not** be presented as a universal recommendation.

A shorter interval could reduce event-publication delay but increase database polling overhead.

A longer interval could reduce database activity but increase event latency.

Therefore:

```text
Polling interval
=
Latency vs database load trade-off
```

---

# 7.17 Selecting PENDING Events

The implementation uses the repository to find:

```text
status = PENDING
```

Conceptually:

```sql
SELECT *
FROM outbox_events
WHERE status = 'PENDING';
```

Then:

```text
For each event
      |
      v
Publish
```

The documented Java implementation uses:

```java
findByStatus(OutboxStatus.PENDING)
```

---

# 7.18 Critical Point — Wait for Kafka Acknowledgement

This is one of the most important implementation details in CitiCore.

The publisher must **not** do:

```java
kafkaTemplate.send(topic, payload);

event.setStatus(SENT);
```

Why?

Because:

```text
send()
  |
  v
Asynchronous operation
```

The call returning does not by itself prove that Kafka successfully accepted the event.

The source explicitly identifies this as a critical issue.

---

# 7.19 The Incorrect Implementation

```text
Kafka send()
     |
     | returns immediately
     v
Mark SENT
     |
     v
Kafka publish fails
```

Now the database says:

```text
SENT
```

but Kafka never actually confirmed the event.

That can result in an event that cannot be recovered through the normal PENDING workflow.

The project notes explicitly identify this as a real production-style data-loss problem.

---

# 7.20 CitiCore's Correct Implementation

The publisher waits for the Kafka result:

```java
SendResult<String, String> result =
    future.get(5, TimeUnit.SECONDS);
```

Only after successful acknowledgement:

```java
event.setStatus(OutboxStatus.SENT);
event.setPublishedAt(LocalDateTime.now());
outboxRepository.save(event);
```

The sequence is therefore:

```text
Send
 |
 v
Wait for Kafka ACK
 |
 +---- Failure → FAILED
 |
 v
Success
 |
 v
Mark SENT
```

This distinction is extremely important in interviews.

---

# 7.21 Real CitiCore Problem — Premature SENT Status

This was one of the important reliability issues documented in the project.

### Original behavior

```text
Outbox Event
     |
     v
Kafka send()
     |
     v
Immediately mark SENT
```

The problem was that Kafka's send operation was asynchronous.

Therefore:

```text
send()
   |
   +---- Kafka still processing
   |
   v
SENT
```

If the process crashed before Kafka confirmed the message:

```text
Database:
SENT ✓

Kafka:
Event may not exist ✗
```

The event could effectively be lost.

The source explicitly documents this as a real issue and states that it was fixed by waiting on `future.get()` before marking the event SENT.

---

# 7.22 Why This Fix Matters

The corrected flow is:

```text
Outbox
  |
  | PENDING
  v
Kafka send
  |
  v
Wait for ACK
  |
  +---- Failure
  |       |
  |       v
  |     FAILED
  |
  +---- Success
          |
          v
        SENT
```

The database status now represents a confirmed Kafka publication rather than merely an attempted publication.

---

# 7.23 Failure During Kafka Publication

Suppose:

```text
Outbox Event = PENDING
      |
      v
Publisher
      |
      v
Kafka
      |
      X
Connection failure
```

The implementation catches the exception:

```java
catch (Exception e) {
    logger.error(...);

    event.setStatus(OutboxStatus.FAILED);
    outboxRepository.save(event);
}
```

The documented design describes the FAILED event as available for later retry or manual intervention.

---

# 7.24 PENDING vs FAILED

These statuses should be understood carefully.

### PENDING

```text
Event has not yet been successfully published.
```

### SENT

```text
Kafka acknowledgement received.
```

### FAILED

```text
Publication attempt encountered an exception.
```

The current implementation explicitly uses all three states.

```text
PENDING
   |
   +---- success → SENT
   |
   +---- exception → FAILED
```

The source does not document a separate automated FAILED-to-PENDING retry mechanism, so it should not be claimed as an implemented automatic retry workflow.

---

# 7.25 Complete Outbox Flow

```text
                    CLIENT
                       |
                       v
              Transaction Service
                       |
                       v
              BEGIN DB TRANSACTION
                       |
             +---------+---------+
             |                   |
             v                   v
       transactions        outbox_events
          PENDING              PENDING
             |                   |
             +---------+---------+
                       |
                       v
                    COMMIT
                       |
                       v
              Return transactionId
                       |
                       |
                Every 5 seconds
                       |
                       v
                OutboxPublisher
                       |
                       v
               Find PENDING events
                       |
                       v
                 Publish to Kafka
                       |
                       v
                Wait for ACK
                  /         \
                 /           \
             Failure        Success
                |              |
                v              v
             FAILED           SENT
                               |
                               v
                             Kafka
                               |
                               v
                        Account Service
```

---

# 7.26 Transaction Workflow Using Outbox

For a transfer:

```text
Client
  |
  v
Transaction Service
  |
  +-- Validate JWT
  |
  +-- Check daily limit
  |
  +-- Validate transfer with Account Service
  |
  v
BEGIN TRANSACTION
  |
  +-- transactions = PENDING
  |
  +-- outbox_events = DEBIT/PENDING
  |
  v
COMMIT
  |
  v
Return transactionId
```

Then asynchronously:

```text
OutboxPublisher
      |
      v
DEBIT event
      |
      v
Kafka
      |
      v
Account Service
```

This exact transaction flow is documented in the source.

---

# 7.27 Why the Client Gets an Immediate Transaction ID

The initial API request does not need to wait for the entire Saga.

The flow is:

```text
Client
  |
  v
Transaction Service
  |
  v
Validate
  |
  v
Save PENDING transaction
  |
  v
Save outbox event
  |
  v
COMMIT
  |
  v
Return transactionId
```

The rest happens asynchronously.

Therefore the client can later check:

```text
GET /api/v1/transactions/{txnId}
```

and observe:

```text
PENDING
DEBIT_SUCCESS
CREDIT_SUCCESS
COMPLETED
FAILED
REVERSED
```

The source documents these transaction states and endpoints.

---

# 7.28 Outbox and Saga Pattern

The two patterns solve different problems.

## Transactional Outbox

Solves:

```text
How do I reliably publish
a database-backed event?
```

## Saga

Solves:

```text
How do I manage a business transaction
across multiple services?
```

CitiCore uses both:

```text
                 Transfer Saga
                      |
       +--------------+--------------+
       |              |              |
      Debit          Credit       Reversal
       |              |              |
       +--------------+--------------+
                      |
                      v
               Outbox Pattern
                      |
                      v
                    Kafka
```

The source explicitly identifies both Transactional Outbox and Saga as key architectural patterns.

---

# 7.29 Outbox During Successful Transfer

```text
Transaction Service
       |
       v
Save transaction = PENDING
       |
       v
Save DEBIT outbox event = PENDING
       |
       v
OutboxPublisher
       |
       v
Kafka
       |
       v
Account Service
       |
       v
Debit sender
       |
       v
DEBIT_SUCCESS
       |
       v
CREDIT
       |
       v
Credit receiver
       |
       v
CREDIT_SUCCESS
       |
       v
Transaction Service
       |
       v
COMPLETED
```

The source documents the successful flow from transaction creation through debit, credit and completion.

---

# 7.30 Outbox During Failed Transfer

Suppose:

```text
Debit = SUCCESS
Credit = FAILED
```

The Transaction Service does:

```text
Transaction
    |
    v
FAILED
    |
    v
Create REVERSAL outbox event
    |
    v
REVERSAL = PENDING
```

Then:

```text
OutboxPublisher
      |
      v
Kafka
      |
      v
Account Service
      |
      v
Reverse debit
      |
      v
REVERSAL_SUCCESS
      |
      v
Transaction Service
      |
      v
REVERSED
```

The source explicitly documents this failure and compensation flow.

---

# 7.31 Why Reversal Also Uses Outbox

Consider:

```text
Credit failed
      |
      v
Need to publish REVERSAL
```

If Transaction Service simply updates its database and directly calls Kafka:

```text
Update transaction
      ✓
      |
      v
Kafka publish
      X
```

the reversal event could be lost.

Instead:

```text
Transaction status = FAILED
       +
REVERSAL event
       |
       v
Same DB transaction
       |
       v
Outbox
       |
       v
Kafka
```

The source specifically identifies reversals as using the Outbox pattern to avoid the dual-write problem.

---

# 7.32 Outbox vs Direct Kafka Publishing

### Direct publishing

```text
Business DB
    |
    v
Commit
    |
    v
Kafka
```

Problem:

```text
DB ✓
Kafka ✗
```

### Transactional Outbox

```text
Business DB
    |
    +-- Business record
    |
    +-- Outbox event
    |
    v
Same transaction
    |
    v
Commit
    |
    v
Publisher
    |
    v
Kafka
```

The event first becomes durable in the database.

---

# 7.33 What the Outbox Pattern Guarantees

The important guarantee is:

```text
If the business transaction commits,
the corresponding outbox event is also committed.
```

Therefore the event is available for a later publication attempt.

This is much stronger than:

```text
DB commit
+
best-effort Kafka send
```

However, it is important not to overstate the guarantee.

The source notes describe the pattern as providing "100% reliability" and "no dual-write problems."

For interview accuracy, a better explanation is:

> The Outbox pattern removes the database-to-Kafka dual-write inconsistency by making the business change and event record atomic within the database. It does not by itself guarantee exactly-once end-to-end business processing.

---

# 7.34 Important Limitation — Duplicate Publication

The Outbox pattern can still result in duplicate Kafka publication.

Consider:

```text
Outbox Event = PENDING
       |
       v
Publish to Kafka
       |
       v
Kafka accepts event ✓
       |
       X
Publisher crashes before marking SENT
```

After restart:

```text
Outbox Event
     |
     v
Still PENDING
     |
     v
Publish again
```

Potentially:

```text
Kafka:
Event A
Event A
```

Therefore:

```text
Transactional Outbox
        ≠
Exactly-once business processing
```

Consumers should use appropriate idempotency for operations where duplicates are dangerous.

This is particularly important for:

```text
Debit
Credit
Reversal
```

---

# 7.35 Idempotency with Transaction Events

A useful business-level approach is to associate processing with:

```text
transactionId
```

Conceptually:

```text
Receive event
      |
      v
Has this event/business operation
already been processed?
      |
   +--+--+
   |     |
  Yes    No
   |     |
 Ignore Process
         |
         v
       Commit
```

The project notes emphasize transaction IDs in event payloads and the need to consider duplicate processing.

The important interview statement is:

> Producer idempotence and Outbox reliability reduce infrastructure-level failure problems, but consumers still need business-level idempotency where duplicate processing could cause incorrect financial results.

---

# 7.36 Outbox vs Kafka Retry

These solve different problems.

### Outbox

Protects:

```text
Database
    ↓
Kafka publication
```

### Kafka consumer retry

Protects:

```text
Kafka
    ↓
Consumer processing
```

Therefore:

```text
DB → Kafka
    |
    v
Transactional Outbox

Kafka → Consumer
    |
    v
Retry + DLT
```

CitiCore uses both mechanisms.

---

# 7.37 Outbox vs Dead Letter Topic

Another important distinction:

### Outbox

Handles an event **before successful Kafka publication**.

```text
Database
   |
   v
Outbox
   |
   v
Kafka
```

### DLT

Handles an event **after it has reached Kafka but repeatedly fails consumer processing**.

```text
Kafka
   |
   v
Consumer
   |
   X
Retry
   |
   X
DLT
```

Therefore:

```text
Outbox
= Reliable publication

DLT
= Failed consumption/recovery
```

---

# 7.38 Outbox and Eventual Consistency

The initial transaction is committed before the event is processed by another service.

Therefore:

```text
Transaction DB
     |
     v
PENDING
     |
     v
Outbox
     |
     v
Kafka
     |
     v
Account Service
```

There is a time gap between:

```text
Transaction created
```

and:

```text
Account operation completed
```

Therefore the system uses **eventual consistency** for the distributed transaction workflow.

The transaction status represents the current stage:

```text
PENDING
   ↓
DEBIT_SUCCESS
   ↓
CREDIT_SUCCESS
   ↓
COMPLETED
```

or:

```text
PENDING
   ↓
DEBIT_SUCCESS
   ↓
CREDIT_FAILED
   ↓
REVERSED
```

---

# 7.39 Outbox and Database Consistency

The most important local transaction is:

```text
+-----------------------------------+
| Database Transaction              |
|                                   |
| transactions                     |
|                                   |
| outbox_events                    |
|                                   |
+-----------------------------------+
                |
                v
             COMMIT
```

This provides atomicity **inside the database**.

Kafka is then updated asynchronously:

```text
Database Commit
      |
      v
Outbox Publisher
      |
      v
Kafka
```

This separation is the core idea of the pattern.

---

# 7.40 Real Failure Scenarios

## Scenario 1 — Kafka is down

```text
Transaction DB
      |
      v
Commit ✓
      |
      v
Outbox = PENDING
      |
      v
Kafka unavailable
      |
      v
Publication fails
```

The business transaction and event record remain in the database.

---

## Scenario 2 — Database transaction fails

```text
Save transaction
      ✓

Save outbox
      X

ROLLBACK
```

Both changes are rolled back.

---

## Scenario 3 — Kafka ACK not received

```text
Publish
  |
  v
Kafka
  |
  X
No acknowledgement
  |
  v
Timeout / exception
  |
  v
FAILED
```

The current implementation records the failure.

---

## Scenario 4 — Publisher crashes after Kafka ACK

```text
Kafka accepts event ✓
       |
       X
Publisher crashes
       |
       X
SENT update not completed
```

The event may be published again later.

This is why consumer-side idempotency remains important.

---

# 7.41 Current CitiCore Implementation

The documented implementation is:

```text
Business transaction
       +
Outbox event
       |
       v
Same MySQL transaction
       |
       v
COMMIT
       |
       v
Scheduled publisher
       |
       | Every 5 seconds
       v
Find PENDING
       |
       v
Kafka send
       |
       v
Wait up to 5 seconds for acknowledgement
       |
       +---- Failure → FAILED
       |
       v
Success
       |
       v
SENT
```

The publisher code explicitly waits using:

```java
future.get(5, TimeUnit.SECONDS)
```

before updating the event to SENT.

---

# 7.42 Current Implementation vs Production Improvements

## Current documented implementation

```text
Scheduled polling
       |
       v
Every 5 seconds
       |
       v
PENDING events
```

The current source does not document:

* A distributed locking strategy between multiple publishers
* A dedicated outbox worker cluster
* Automated FAILED → PENDING retry scheduling
* CDC-based publication
* Kafka Connect-based outbox publishing

Therefore these should **not** be presented as currently implemented features.

---

# 7.43 Production Considerations

For a larger production deployment, important considerations would include:

### 1. Multiple publisher instances

If multiple Transaction Service instances run:

```text
Publisher 1
Publisher 2
Publisher 3
       |
       v
outbox_events
```

the system needs a strategy to prevent multiple publishers from processing the same event incorrectly.

Possible approaches include:

```text
Row locking
Claim/status mechanism
SKIP LOCKED
Distributed coordination
```

These are production design considerations, not documented current CitiCore implementation details.

---

### 2. Indexing

The publisher frequently searches:

```text
WHERE status = 'PENDING'
```

Therefore an index on the status/created-time access pattern can become important as the table grows.

This should be evaluated based on actual workload.

---

### 3. Outbox Cleanup

SENT events can accumulate.

A production system may need:

```text
Retention policy
       |
       v
Archive / delete old events
```

while preserving whatever audit information the business requires.

---

### 4. Monitoring

Useful metrics include:

```text
Pending outbox count
Failed outbox count
Publication latency
Oldest pending event
Kafka publication failures
Publisher execution time
```

An important operational alert would be:

```text
Oldest PENDING event > threshold
```

because that indicates the event pipeline may be stuck.

---

# 7.44 Outbox Observability

A practical monitoring model is:

```text
                 OUTBOX
                    |
        +-----------+-----------+
        |           |           |
        v           v           v
     PENDING      SENT       FAILED
        |           |           |
        v           v           v
    Backlog      Success     Investigation
```

Important questions:

```text
Are PENDING events increasing?

Are FAILED events increasing?

How old is the oldest PENDING event?

Are Kafka acknowledgements slow?

Is the publisher running every 5 seconds?

Is Kafka reachable?
```

This gives operational visibility into the DB-to-Kafka pipeline.

---

# 7.45 Outbox Troubleshooting Checklist

If an event is not appearing in Kafka:

### Step 1 — Check the business transaction

```text
Was the database transaction committed?
```

---

### Step 2 — Check the outbox table

Look for:

```text
transaction_id
event_type
payload
status
created_at
published_at
```

Expected initial state:

```text
PENDING
```

---

### Step 3 — Check publisher execution

Verify that:

```text
OutboxPublisher
```

is running every 5 seconds.

---

### Step 4 — Check event status

```text
PENDING
```

means it has not been successfully published.

```text
FAILED
```

means publication encountered an exception.

```text
SENT
```

means Kafka acknowledgement was received by the publisher.

---

### Step 5 — Check Kafka

Verify:

```text
Kafka reachable
Topic exists
Producer connection
Broker logs
```

---

### Step 6 — Check publisher logs

Look for:

```text
Timeout
Connection refused
UnknownTopicOrPartition
Serialization error
Broker unavailable
```

---

### Step 7 — Check Kafka consumer

If the event is already in Kafka but business processing has not completed, the issue has moved beyond the Outbox layer.

Then investigate:

```text
Consumer
   |
   v
Retry
   |
   v
DLT
```

---

# 7.46 Outbox and Transaction Status

The Outbox status and business transaction status are different concepts.

### Transaction status

Represents:

```text
Business workflow
```

Example:

```text
PENDING
DEBIT_SUCCESS
CREDIT_SUCCESS
COMPLETED
FAILED
REVERSED
```

### Outbox status

Represents:

```text
Event publication
```

Example:

```text
PENDING
SENT
FAILED
```

Therefore:

```text
Transaction status
        ≠
Outbox status
```

Example:

```text
Transaction:
PENDING

Outbox:
SENT
```

This is completely possible.

It means the event was successfully published but the downstream business operation is still processing.

---

# 7.47 Example State Timeline

A transfer might progress as:

```text
T0
|
| Client submits transfer
v
Transaction = PENDING
Outbox = PENDING

T1
|
| Publisher sends DEBIT
v
Outbox = SENT

T2
|
| Account Service processes DEBIT
v
Transaction = DEBIT_SUCCESS

T3
|
| CREDIT event published
v
Credit processing

T4
|
| Credit succeeds
v
Transaction = COMPLETED
```

This demonstrates why the Outbox status and business status must be modeled independently.

---

# 7.48 Why Not Publish Kafka Event Directly Inside the DB Transaction?

A common question is:

> "Why not start a DB transaction, save the record, call Kafka, and then commit?"

Because Kafka is not part of the database's local transaction.

Example:

```text
BEGIN DB TRANSACTION
      |
      v
Save transaction
      |
      v
Kafka publish
      |
      X
Kafka fails
      |
      v
Rollback DB
```

While this may appear reasonable, it introduces distributed transaction coordination and blocking/failure complexity.

The Outbox approach instead makes the database transaction local and durable:

```text
BEGIN DB TRANSACTION
      |
      +-- Business record
      |
      +-- Outbox record
      |
      v
COMMIT
      |
      v
Asynchronous Kafka publication
```

This is simpler and more resilient.

---

# 7.49 Transactional Outbox vs Distributed Transaction / 2PC

The Outbox pattern avoids trying to create one atomic transaction across:

```text
MySQL
+
Kafka
```

Instead:

```text
MySQL
  |
  | Atomic local transaction
  v
Outbox
  |
  | Async publication
  v
Kafka
```

This introduces eventual consistency but reduces coupling between infrastructure systems.

---

# 7.50 Main Advantages

## 1. Solves the dual-write problem

```text
Business DB
     +
Outbox event
     |
     v
Same transaction
```

---

## 2. Durable event storage

If Kafka is temporarily unavailable:

```text
Event
 |
 v
Database
 |
 v
PENDING
```

The event is not dependent on Kafka being immediately available.

---

## 3. Better failure recovery

The publisher can identify:

```text
PENDING
FAILED
```

events.

---

## 4. Works well with Kafka

The pattern naturally fits:

```text
Database
   |
   v
Outbox
   |
   v
Kafka
```

---

## 5. Supports eventual consistency

Services can process events asynchronously while the transaction state tracks progress.

---

# 7.51 Disadvantages

## 1. Additional database writes

Every event creates another database record.

```text
Business write
+
Outbox write
```

---

## 2. Additional table management

The outbox table grows over time and needs operational management.

---

## 3. Publication latency

CitiCore currently polls every:

```text
5 seconds
```

Therefore an event may wait before publication.

---

## 4. More application complexity

The system now requires:

```text
Outbox entity
Repository
Publisher
Status management
Failure handling
Monitoring
```

---

## 5. Duplicate publication remains possible

If the publisher crashes after Kafka accepts an event but before the database is updated:

```text
Kafka = published
Outbox = still PENDING
```

The event may be sent again.

Therefore consumer idempotency remains important.

---

# 7.52 Key Design Trade-off

The central trade-off is:

```text
Simple direct publishing
        |
        v
Less code
but
Higher dual-write risk
```

versus:

```text
Transactional Outbox
        |
        v
More infrastructure/application complexity
but
Much stronger DB → Kafka reliability
```

For a banking workflow, CitiCore chooses the second approach.

---

# 7.53 End-to-End CitiCore Outbox Architecture

```text
                         CLIENT
                            |
                            v
                  TRANSACTION SERVICE
                            |
                            v
                    BEGIN TRANSACTION
                            |
               +------------+------------+
               |                         |
               v                         v
        Transactions Table        Outbox Events Table
               |                         |
            PENDING                   PENDING
               |                         |
               +------------+------------+
                            |
                            v
                         COMMIT
                            |
                            v
                     Transaction ID
                       returned
                            |
                            |
                    Every 5 seconds
                            |
                            v
                    OUTBOX PUBLISHER
                            |
                            v
                   Find PENDING Events
                            |
                            v
                      Publish to Kafka
                            |
                     Wait for ACK
                       /          \
                      /            \
                 Failure          Success
                    |                 |
                    v                 v
                 FAILED             SENT
                                      |
                                      v
                                    KAFKA
                                      |
                                      v
                              ACCOUNT SERVICE
                                      |
                              +-------+-------+
                              |               |
                              v               v
                           DEBIT           CREDIT
                              |               |
                              +-------+-------+
                                      |
                                      v
                              RESULT EVENTS
                                      |
                                      v
                              TRANSACTION SERVICE
                                      |
                         +------------+------------+
                         |                         |
                      Success                   Failure
                         |                         |
                         v                         v
                    COMPLETED                  REVERSAL
                                                   |
                                                   v
                                             Outbox again
```

---

# 7.54 Most Important Real Issue

If asked:

> "What real problem did you solve using the Outbox pattern?"

A strong answer is:

> "Initially, the OutboxPublisher marked events as SENT immediately after calling Kafka's asynchronous send method. The problem was that send() returning did not mean Kafka had actually acknowledged the event. If the process crashed before Kafka confirmed the message, the database could say SENT even though the event was not safely published. I fixed this by waiting for the Kafka acknowledgement using future.get() with a timeout, and only then marking the outbox event as SENT."

The source explicitly documents this issue and its fix.

---

# 7.55 How I Would Explain Transactional Outbox in an Interview

> "In CitiCore, we use the Transactional Outbox pattern to solve the dual-write problem between MySQL and Kafka.
>
> For example, when a transaction is initiated, Transaction Service saves the transaction record and the corresponding DEBIT event in the outbox table within the same database transaction. Both are committed atomically.
>
> A scheduled OutboxPublisher polls for PENDING events every five seconds and publishes them to Kafka. An important implementation detail is that we wait for Kafka acknowledgement before changing the event status from PENDING to SENT.
>
> We also use the outbox for compensation events such as REVERSAL, so the reversal event is not lost if Kafka is temporarily unavailable.
>
> One real issue I encountered was marking an event SENT immediately after Kafka's asynchronous send call. I fixed it by waiting for the Kafka future acknowledgement before updating the database.
>
> The Outbox pattern solves the DB-to-Kafka dual-write problem, but it does not automatically provide exactly-once business processing. A publisher can still crash after Kafka accepts an event but before marking it SENT, so consumers should be designed with appropriate idempotency."

---

# 7.56 Common Interview Questions

## Q1. What problem does Transactional Outbox solve?

**Answer:**

It solves the dual-write problem between a database and an external message broker such as Kafka.

Instead of directly writing to the database and Kafka independently, the business record and event are stored atomically in the database. A separate publisher later sends the event to Kafka.

---

## Q2. Why do you store the Kafka event in the database?

**Answer:**

So that the event becomes durable as part of the same database transaction as the business operation. If Kafka is unavailable, the event remains in the outbox and can be published later.

---

## Q3. Why not directly call Kafka after saving the transaction?

**Answer:**

Because the database write and Kafka publication are independent operations. One can succeed while the other fails, creating inconsistent state.

---

## Q4. What is stored in the outbox table?

**Answer:**

CitiCore stores:

```text
id
transaction_id
event_type
payload
status
created_at
published_at
```

The event status can be:

```text
PENDING
SENT
FAILED
```

---

## Q5. Why mark SENT only after Kafka acknowledgement?

**Answer:**

Because Kafka's send operation is asynchronous. Calling `send()` does not by itself confirm successful publication. CitiCore waits for the future result and marks the event SENT only after acknowledgement.

---

## Q6. What happens if Kafka is down?

**Answer:**

The business transaction and outbox event can still be committed to the database. The event remains available in the outbox. The publisher can attempt publication later.

---

## Q7. What happens if the database transaction fails?

**Answer:**

Both the business record and outbox record are rolled back because they are part of the same database transaction.

---

## Q8. Does Outbox guarantee exactly-once processing?

**Answer:**

No. It solves the database-to-message-broker dual-write problem, but duplicate publication can still happen if the publisher crashes after Kafka accepts the message but before the outbox record is marked SENT. Consumer idempotency is still important.

---

## Q9. What is the difference between Outbox and DLT?

**Answer:**

Outbox handles reliable publication from the database to Kafka.

DLT handles messages that reached Kafka but repeatedly failed during consumer processing.

```text
DB → Kafka
     Outbox

Kafka → Consumer
          Retry → DLT
```

---

## Q10. Why does CitiCore poll the outbox?

**Answer:**

The current implementation uses a scheduled publisher that checks for PENDING events every five seconds and publishes them to Kafka.

---

## Q11. Why use Outbox for reversal?

**Answer:**

A reversal is also a critical business event. If credit fails after debit succeeds, the reversal must reliably reach Account Service. Storing the reversal event in the outbox provides the same DB-to-Kafka reliability mechanism.

---

## Q12. What happens if the publisher crashes after Kafka receives the event?

**Answer:**

The event may remain PENDING because the publisher did not update the database. When it is processed again, Kafka may receive a duplicate. Therefore the consumer should handle duplicate business events safely.

---

## Q13. What is the biggest disadvantage of the Outbox pattern?

**Answer:**

It adds complexity and database overhead. Every event creates an additional database record, and the outbox table requires monitoring, cleanup and publisher management.

---

## Q14. Is the Outbox publisher synchronous?

**Answer:**

The business request itself does not synchronously publish to Kafka. The application stores the event in the database and the scheduled OutboxPublisher publishes it asynchronously. However, during publication, the publisher waits synchronously for Kafka acknowledgement before marking the event SENT.

---

# 7.57 Interview Mental Model

Remember the pattern as:

```text
       BUSINESS OPERATION
              |
              v
       +--------------+
       |   DATABASE   |
       |              |
       | Business Row |
       |      +       |
       | Outbox Row   |
       +--------------+
              |
           COMMIT
              |
              v
       OUTBOX PUBLISHER
              |
              v
            KAFKA
              |
              v
          CONSUMER
```

The key sentence:

> **"First make the business change and event durable together; then publish the event asynchronously."**

---

# 7.58 The Three Reliability Layers in CitiCore

CitiCore's event-driven transaction architecture uses multiple reliability mechanisms.

```text
                  TRANSACTION
                       |
                       v
              Database Transaction
                       |
                       v
              Transactional Outbox
                       |
                       v
                     Kafka
                       |
                       v
              Consumer Retry
                       |
                       v
                     DLT
                       |
                       v
              Manual Recovery
```

Each layer solves a different problem:

| Layer                | Purpose                                           |
| -------------------- | ------------------------------------------------- |
| Database Transaction | Atomic business data changes                      |
| Transactional Outbox | Reliable DB → Kafka publication                   |
| Kafka                | Durable asynchronous event transport              |
| Consumer Retry       | Handle transient processing failures              |
| DLT                  | Isolate repeatedly failing messages               |
| Idempotency          | Protect against duplicate business processing     |
| Saga/Reversal        | Compensate failed distributed business operations |

---

# 7.59 What Transactional Outbox Does NOT Solve

This is important for interviews.

The Outbox pattern does **not** automatically solve:

```text
Consumer duplicate processing
```

It does **not** automatically solve:

```text
Business authorization
```

It does **not** automatically solve:

```text
Saga compensation
```

It does **not** automatically solve:

```text
Kafka consumer failures
```

It does **not** automatically provide:

```text
Exactly-once business semantics
```

Instead:

```text
Outbox
→ DB-to-Kafka reliability

Retry/DLT
→ Consumer failure handling

Idempotency
→ Duplicate business-event protection

Saga
→ Distributed business compensation
```

This separation is one of the most important concepts to remember.

---

# 7.60 Final Design Summary

CitiCore's Transactional Outbox implementation can be summarized as:

```text
Client
  |
  v
Transaction Service
  |
  v
BEGIN DB TRANSACTION
  |
  +---- Business Transaction
  |
  +---- Outbox Event
  |
  v
COMMIT
  |
  v
Outbox Event = PENDING
  |
  v
Publisher every 5 seconds
  |
  v
Kafka Send
  |
  v
Wait for ACK
  |
  +----------+
  |          |
Failure    Success
  |          |
  v          v
FAILED      SENT
             |
             v
            Kafka
             |
             v
        Account Service
             |
             v
        Result Event
             |
             v
       Transaction Service
             |
             v
       Update Transaction
```

---

# 7.61 Core Principle

The most important thing to remember is:

```text
WITHOUT OUTBOX

Database ✓
Kafka    ✗

        ↓

Inconsistent System
```

With Outbox:

```text
Database
   |
   +-- Business Change
   |
   +-- Event
   |
   v
Atomic Commit
   |
   v
Outbox
   |
   v
Kafka
```

The strongest interview explanation is:

> **"Transactional Outbox makes the business database change and the corresponding event durable in the same local transaction. Kafka publication then happens asynchronously through a publisher. In CitiCore, the publisher waits for Kafka acknowledgement before marking an event SENT. This removes the DB-to-Kafka dual-write problem, while consumer idempotency, retries, DLT and Saga compensation handle the remaining distributed-system failure cases."**

---

# 7.62 Topic Summary

CitiCore uses the Transactional Outbox pattern because critical banking events must not be lost between:

```text
MySQL
  ↓
Kafka
```

The implementation is:

```text
Business Transaction
       +
Outbox Event
       |
       v
Same DB Transaction
       |
       v
COMMIT
       |
       v
PENDING
       |
       v
OutboxPublisher
       |
       | Every 5 seconds
       v
Kafka
       |
       v
Wait for ACK
       |
       +---- Failure → FAILED
       |
       v
Success
       |
       v
SENT
```

The most important real implementation lesson is:

```text
Kafka send()
      ≠
Kafka acknowledgement
```

Therefore:

```text
send()
   |
   v
wait for ACK
   |
   v
mark SENT
```

And the most important architecture lesson is:

```text
Transactional Outbox
        ≠
Exactly Once

Outbox
→ Reliable DB-to-Kafka publication

Idempotency
→ Safe duplicate processing

Retry + DLT
→ Consumer failure recovery

Saga + Reversal
→ Distributed transaction compensation
```
---
# TOPIC 8 — REDIS

## 8.1 What is Redis?

### Concept

**Redis** is an in-memory data store. Unlike a traditional relational database such as MySQL, Redis primarily stores data in memory, which makes it useful for low-latency operations.

Redis supports different data structures, including:

* Strings
* Hashes
* Lists
* Sets
* Sorted sets

In CitiCore, Redis is used primarily for two purposes:

```text
1. Caching
2. API Gateway rate limiting
```

The caching implementation documented in the CitiCore notes focuses on reducing database reads for frequently requested account balance data. Redis is also referenced by the API Gateway as the backing store for rate-limiting functionality. 

---

## 8.2 Why Do We Need Redis?

### Problem

Without caching, every balance request may reach the database.

```text
Client
   |
   v
Account Service
   |
   v
MySQL / RDS
```

For frequently requested data, this can create unnecessary database load.

For example:

```text
100 requests
     |
     v
100 database reads
```

If the data can safely be cached:

```text
100 requests
     |
     v
Redis
     |
     +---- Most requests served from cache
     |
     +---- Database used on cache miss
```

Redis can therefore help:

* Reduce database read load
* Reduce latency for repeated reads
* Improve application responsiveness
* Support centralized distributed rate limiting

However, caching introduces an important problem:

```text
Cached data
     |
     v
Can become stale
```

Therefore, Redis is not simply added for speed. A caching strategy must define:

* What data should be cached
* Cache key structure
* TTL
* Cache invalidation behavior
* What happens when data changes
* What happens when Redis is unavailable

---

# 8.3 CitiCore Context

## CitiCore Implementation

The CitiCore notes document Redis integration primarily in the **Account Service** for caching account balances.

The documented flow is:

```text
GET Balance
    |
    v
Check Redis
    |
    +---- Cache HIT
    |        |
    |        v
    |   Return cached balance
    |
    +---- Cache MISS
             |
             v
        Query database
             |
             v
        Store in Redis
             |
             v
        Return balance
```

The implementation uses:

```text
Cache key prefix:
balance:

Example:
balance:<accountNumber>

TTL:
30 minutes
```

The notes also document cache invalidation after balance-changing operations such as:

```text
Deposit
Withdrawal
```



---

# 8.4 Redis Architecture in CitiCore

```text
                    Client
                       |
                       v
                 API Gateway
                       |
                       v
                Account Service
                       |
              Check Redis Cache
                       |
             +---------+---------+
             |                   |
          CACHE HIT           CACHE MISS
             |                   |
             v                   v
       Return Balance       Query Database
                                 |
                                 v
                           Account Balance
                                 |
                                 v
                          Store in Redis
                                 |
                                 v
                           Return Balance
```

For balance updates:

```text
Deposit / Withdrawal
        |
        v
Account Service
        |
        v
Update Database
        |
        v
Invalidate Redis Cache
        |
        v
Next Read
        |
        v
Load Fresh Value
```

This is a form of the **cache-aside pattern**.

---

# 8.5 Caching Strategy

## Concept

CitiCore uses a practical cache-aside approach.

The application controls the cache directly.

The flow is:

```text
Application
     |
     v
Check Redis
     |
     +---- HIT
     |       |
     |       v
     |   Return cached data
     |
     +---- MISS
             |
             v
         Query database
             |
             v
         Store result in Redis
             |
             v
         Return result
```

This approach is useful because the database remains the primary source of truth.

Redis acts as a temporary performance layer.

---

# 8.6 Cache Key Design

The documented balance cache uses:

```text
balance:<accountNumber>
```

For example:

```text
balance:ACC123456
```

The implementation defines:

```java
private static final String BALANCE_KEY_PREFIX = "balance:";
```

and constructs the key using the account number.



## Why Use a Prefix?

Prefixes help organize keys.

For example:

```text
balance:12345
user:12345
rate-limit:client-123
```

Without prefixes:

```text
12345
```

the meaning of the key becomes unclear.

A structured key naming convention is especially important when multiple services or features share the same Redis infrastructure.

---

# 8.7 Cache TTL

## CitiCore Implementation

The documented implementation uses a cache TTL of:

```text
30 minutes
```

The code defines:

```java
private static final long CACHE_TTL =
    30 * 60 * 1000;
```

The value is then converted to seconds when creating the Redis expiration duration.



---

## Why Use TTL?

Without expiration:

```text
Redis
   |
   v
Old data may remain indefinitely
```

TTL automatically removes cached data after a configured period.

```text
Cache Entry Created
       |
       v
TTL Starts
       |
       v
30 Minutes
       |
       v
Entry Expires
```

The next request becomes a cache miss and retrieves fresh data from the database.

---

# 8.8 Cache Hit

A cache hit occurs when the requested data already exists in Redis.

```text
Request Balance
      |
      v
Redis
      |
      v
Value Found
      |
      v
CACHE HIT
      |
      v
Return Value
```

The documented implementation checks:

```java
String cachedBalance =
    redis.opsForValue()
         .get(BALANCE_KEY_PREFIX + accountNumber);

if (cachedBalance != null) {
    return new BigDecimal(cachedBalance);
}
```



A cache hit avoids a database query.

---

# 8.9 Cache Miss

A cache miss occurs when Redis does not contain the requested value.

```text
Request Balance
      |
      v
Redis
      |
      v
Value Not Found
      |
      v
CACHE MISS
      |
      v
Query Database
      |
      v
Store Result in Redis
      |
      v
Return Result
```

The CitiCore implementation follows this pattern:

```java
BigDecimal balance = accountRepository
    .findByAccountNumber(accountNumber)
    .map(Account::getBalance)
    .orElse(BigDecimal.ZERO);
```

Then the value is stored in Redis with the configured TTL.



---

# 8.10 CitiCore Balance Caching Flow

```text
Client
   |
   v
GET Balance
   |
   v
Account Service
   |
   v
Redis
   |
   +--------------------------+
   |                          |
   v                          v
HIT                        MISS
   |                          |
   v                          v
Return Cached              MySQL / RDS
Balance                       |
                              v
                        Get Balance
                              |
                              v
                        Store in Redis
                              |
                              v
                        Return Balance
```

This reduces repeated database reads for the same account balance.

---

# 8.11 Important Implementation Detail — Source of Truth

Redis is **not** the source of truth for account balances.

The source of truth remains:

```text
MySQL / RDS
```

The architecture is:

```text
                    Source of Truth
                         |
                         v
                    MySQL / RDS
                         |
                         v
                     Redis Cache
```

Redis contains a temporary cached representation.

This distinction is important in a banking application.

A cache may be cleared or restarted.

The system should still be able to recover data from the database.

---

# 8.12 Cache Invalidation

## Problem

Suppose Redis contains:

```text
balance:ACC123 = 1000
```

Then a deposit occurs:

```text
Database balance = 1500
Redis balance    = 1000
```

Now Redis contains stale data.

Therefore the cache must be handled when balance-changing operations occur.

---

## CitiCore Implementation

The notes document cache invalidation using:

```java
public void invalidateBalance(String accountNumber) {

    redis.delete(
        BALANCE_KEY_PREFIX + accountNumber
    );

    logger.debug(
        "Cache invalidated for account: {}",
        accountNumber
    );
}
```



The documented use case is:

```text
Deposit
Withdrawal
     |
     v
Invalidate balance cache
```

The next read retrieves a fresh value from the database.

---

# 8.13 Why Invalidate Instead of Immediately Updating the Cache?

There are two common approaches.

## Option 1 — Update Cache

```text
Database Update
      |
      v
Update Redis Value
```

## Option 2 — Invalidate Cache

```text
Database Update
      |
      v
Delete Redis Key
```

CitiCore's documented implementation uses invalidation.

The next read then follows:

```text
Cache MISS
    |
    v
Database
    |
    v
Fresh Cache Value
```

This approach avoids maintaining separate balance-update logic for both:

```text
Database
+
Cache
```

However, invalidation can temporarily cause the next request to hit the database.

---

# 8.14 Important Banking Consideration

Account balances are sensitive data.

Caching them requires careful consistency handling.

For example:

```text
User checks balance
       |
       v
Redis returns 1000
```

At the same time:

```text
Transaction Service
       |
       v
Debit 500
       |
       v
Database balance = 500
```

If the cache is not invalidated correctly:

```text
Redis may still return:

1000
```

Therefore, balance-changing operations must ensure that stale cache data does not remain valid longer than intended.

For financial operations, the database remains the authoritative source.

Critical write or validation paths should not blindly depend on potentially stale cached data.

---

# 8.15 Redis Configuration

The CitiCore notes document the following Redis configuration:

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 20
          max-idle: 10
          min-idle: 5
```



---

# 8.16 Important Configuration Explained

## Redis Host

```yaml
host: ${REDIS_HOST:localhost}
```

This allows the Redis host to come from an environment variable.

For local development:

```text
REDIS_HOST
   |
   +---- Not provided
          |
          v
       localhost
```

For deployed environments:

```text
REDIS_HOST
   |
   v
Redis server address
```

This makes the application configuration environment-dependent without hardcoding the production address into the application code.

---

## Redis Port

```yaml
port: ${REDIS_PORT:6379}
```

Redis commonly listens on:

```text
6379
```

The environment variable allows the deployment configuration to override the default if necessary.

---

## Timeout

```yaml
timeout: 2000ms
```

The configured timeout is:

```text
2 seconds
```

A timeout prevents an application thread from waiting indefinitely for Redis.

Conceptually:

```text
Application
     |
     v
Redis Request
     |
     +---- Response → Continue
     |
     +---- No response after timeout
               |
               v
            Failure path
```

Timeout configuration is important because Redis is an external dependency.

---

# 8.17 Lettuce Connection Pool

The documented configuration includes:

```yaml
lettuce:
  pool:
    max-active: 20
    max-idle: 10
    min-idle: 5
```



The values represent the configured connection-pool behavior in the CitiCore notes.

### `max-active`

```text
Maximum active connections:
20
```

### `max-idle`

```text
Maximum idle connections:
10
```

### `min-idle`

```text
Minimum idle connections:
5
```

Connection pooling avoids repeatedly creating connections for every Redis operation.

---

# 8.18 Why Connection Pooling Matters

Without reuse:

```text
Request
   |
   v
Create Connection
   |
   v
Redis
   |
   v
Close Connection
```

Repeated connection creation can add unnecessary overhead.

With pooling:

```text
Request
   |
   v
Connection Pool
   |
   v
Reusable Redis Connection
```

However, connection-pool sizes should be based on actual workload and infrastructure capacity.

The configured values should not be presented as universally optimal production values.

---

# 8.19 Redis Implementation

The documented CitiCore implementation uses:

```java
StringRedisTemplate
```

The relevant service structure is:

```java
@Service
public class BalanceCacheService {

    @Autowired
    private StringRedisTemplate redis;

    @Autowired
    private AccountRepository accountRepository;

    private static final String BALANCE_KEY_PREFIX =
        "balance:";

    private static final long CACHE_TTL =
        30 * 60 * 1000;
}
```



---

# 8.20 Balance Retrieval Implementation

A simplified version of the documented implementation is:

```java
public BigDecimal getBalance(String accountNumber) {

    String key = BALANCE_KEY_PREFIX + accountNumber;

    String cachedBalance =
        redis.opsForValue().get(key);

    if (cachedBalance != null) {

        return new BigDecimal(cachedBalance);
    }

    BigDecimal balance = accountRepository
        .findByAccountNumber(accountNumber)
        .map(Account::getBalance)
        .orElse(BigDecimal.ZERO);

    redis.opsForValue().set(
        key,
        balance.toString(),
        Duration.ofMinutes(30)
    );

    return balance;
}
```

The logic follows:

```text
Redis
  |
  +---- Value exists
  |       |
  |       v
  |   Return value
  |
  +---- Value missing
          |
          v
       Database
          |
          v
       Cache value
          |
          v
       Return value
```

---

# 8.21 Why Store Balance as a String?

The documented implementation uses:

```java
StringRedisTemplate
```

Therefore the balance is stored as:

```java
balance.toString()
```

and retrieved as:

```java
new BigDecimal(cachedBalance)
```

This provides explicit conversion between:

```text
Redis String
      ⇅
Java BigDecimal
```

For financial values, using `BigDecimal` in Java is appropriate for precise decimal arithmetic.

The Redis value is still only a cached representation; the database remains authoritative.

---

# 8.22 Cache Invalidation Flow

For a balance-changing operation:

```text
Deposit / Withdrawal
        |
        v
Business Operation
        |
        v
Update Database
        |
        v
Invalidate:
balance:<accountNumber>
        |
        v
Next Read
        |
        v
Cache MISS
        |
        v
Load Fresh Database Value
```

This is the documented CitiCore caching strategy.

---

# 8.23 Redis in API Gateway

Redis is also referenced in the CitiCore architecture as the backing component for API Gateway rate limiting.

The architecture notes describe:

```text
API Gateway
    |
    +---- Rate limiting via Redis
```



This topic should be understood separately from balance caching.

---

## Why Use Redis for Rate Limiting?

Rate limiting needs shared state.

For example:

```text
Request 1
Request 2
Request 3
```

The system needs to track how many requests a client has made.

In a distributed deployment:

```text
Gateway Instance 1
Gateway Instance 2
Gateway Instance 3
```

a local in-memory counter on one gateway would not automatically be shared with the others.

Redis provides centralized shared state:

```text
Gateway 1
    |
Gateway 2 ----> Redis
    |
Gateway 3
```

This makes Redis useful for distributed rate limiting.

Detailed API Gateway rate-limiting configuration belongs to **Topic 5 — API Gateway** and should not be duplicated here.

---

# 8.24 Redis Failure Handling

Redis is an external dependency.

Therefore CitiCore should distinguish between:

```text
Cache failure
```

and:

```text
Database failure
```

For balance reads:

```text
Client
   |
   v
Account Service
   |
   v
Redis
   |
   X
Unavailable
```

The application should ideally be able to fall back to the database rather than treating a cache failure as equivalent to losing the underlying data.

Conceptually:

```text
Redis
   |
   +---- Available
   |       |
   |       v
   |    Use Cache
   |
   +---- Unavailable
           |
           v
        Database
```

This follows the principle:

> **A cache should improve performance, but the application should not treat cached data as the only source of truth.**

The current notes document Redis configuration and caching behavior but do not clearly document a specific production fallback implementation using circuit breakers or custom Redis exception handling. Therefore such behavior should be treated as a production consideration rather than a confirmed CitiCore implementation.

---

# 8.25 Cache Failure vs Database Failure

These failures have different severity.

## Redis Failure

```text
Redis unavailable
      |
      v
Performance impact
```

The application may still retrieve data from the database.

## Database Failure

```text
Database unavailable
      |
      v
Authoritative data unavailable
```

This is generally more serious.

Therefore:

```text
Redis
=
Performance layer

Database
=
Persistent source of truth
```

---

# 8.26 Real CitiCore Redis Configuration

The source notes document these ECS environment values:

```text
REDIS_HOST=10.0.1.87
REDIS_PORT=6379
```



These values represent the documented project environment at the time of implementation.

They should not be treated as permanent infrastructure values or reused as a general production configuration.

The important architectural lesson is:

```text
Application
   |
   v
Environment Variable
   |
   v
Redis Host / Port
```

rather than:

```text
Application Code
   |
   v
Hardcoded Redis Address
```

---

# 8.27 Redis and ECS

A containerized application should not assume:

```text
localhost
```

always refers to Redis.

Inside a container:

```text
localhost
```

usually refers to the same container.

Therefore deployed applications should use the appropriate network address or service hostname.

Conceptually:

```text
Account Service Container
       |
       v
REDIS_HOST
       |
       v
Redis Instance
```

This principle is especially important when moving from local development to Docker or ECS environments.

---

# 8.28 Cache Consistency Trade-off

Caching introduces a trade-off:

```text
Freshness
    vs
Performance
```

### No Cache

```text
Every request
     |
     v
Database
```

Advantages:

* Always reads current database data
* Simpler consistency model

Disadvantages:

* More database load
* Potentially slower repeated reads

---

### Cache

```text
Request
   |
   v
Redis
```

Advantages:

* Faster repeated reads
* Reduced database load

Disadvantages:

* Stale data is possible
* Cache invalidation is required
* Additional infrastructure dependency

CitiCore uses caching for performance while keeping the database as the authoritative source.

---

# 8.29 Why Caching Account Balances Requires Care

A banking application cannot treat every cached value as suitable for every operation.

Consider:

```text
Redis Balance = 1000
```

A withdrawal request requires a correct balance decision.

If another transaction changed the balance:

```text
Actual Database Balance = 500
```

using stale cached data for a critical debit decision could create correctness problems.

Therefore, conceptually:

```text
Frequently viewed data
        |
        v
Cache can be useful
```

but:

```text
Critical financial write validation
        |
        v
Requires strong consistency
```

The exact datasource and consistency routing used by CitiCore is covered in the **Database Architecture** topic.

---

# 8.30 Redis Cache Lifecycle

The CitiCore balance cache lifecycle is:

```text
                    DATABASE
                        |
                        |
                  Balance Read
                        |
                        v
                     REDIS
                        |
              +---------+---------+
              |                   |
            HIT                 MISS
              |                   |
              v                   v
           Return              Load DB
                                  |
                                  v
                            Store in Redis
                                  |
                                  v
                               Return
```

When the balance changes:

```text
Database Write
      |
      v
Delete Cache Key
      |
      v
Fresh Value Loaded
on next read
```

When TTL expires:

```text
Cache Entry
    |
    v
30 Minutes
    |
    v
Expired
    |
    v
Next Request = MISS
```

---

# 8.31 Cache Stampede Consideration

### Concept

Suppose a popular cache entry expires.

```text
Cache Entry Expires
       |
       v
Many Requests Arrive
       |
       v
All See Cache MISS
       |
       v
All Query Database
```

This can create a sudden database load spike.

This problem is called a **cache stampede** or **cache thundering herd** scenario.

The current CitiCore notes do not document a specific cache-stampede protection mechanism.

Possible production strategies include:

```text
Request coalescing
Distributed locking
TTL jitter
Background refresh
```

These are **future production considerations**, not confirmed current implementation details.

---

# 8.32 Cache Penetration Consideration

Another caching concern is repeated requests for data that does not exist.

Example:

```text
Request account: INVALID123
       |
       v
Cache MISS
       |
       v
Database
       |
       v
Not Found
```

If repeated:

```text
Many requests
       |
       v
Many database queries
```

A production system may consider negative caching in carefully selected situations.

However, the current CitiCore notes do not document negative caching as an implementation.

---

# 8.33 Redis Monitoring

Useful Redis-related metrics in a production environment include:

```text
Cache hit rate
Cache miss rate
Redis latency
Connection count
Memory usage
Evicted keys
Expired keys
Command failures
```

For CitiCore balance caching, useful application-level metrics could include:

```text
Balance cache HIT count
Balance cache MISS count
Database fallback count
Cache invalidation count
Redis errors
```

These help determine whether Redis is actually improving performance.

---

# 8.34 Cache Hit Ratio

A useful metric is:

```text
Cache Hit Ratio

=
Cache Hits
-------------------------
Cache Hits + Cache Misses
```

For example:

```text
Illustrative Example

Hits   = 900
Misses = 100

Hit Ratio = 90%
```

A higher hit ratio may reduce database reads, but it is not automatically better in every case.

For example, a very long TTL might improve the hit ratio while increasing the risk of stale data.

Therefore:

```text
Cache efficiency
      +
Data freshness
```

must be balanced.

---

# 8.35 Redis Security Considerations

Redis should not be treated as a publicly exposed data store.

Production considerations include:

```text
Private network access
Authentication
TLS where required
Security groups / firewall rules
No public exposure
Secret management
Access monitoring
```

For CitiCore, environment-specific infrastructure and security details belong to the AWS Infrastructure and Database Security topics.

The main Redis principle is:

```text
Application
      |
      v
Private Redis Access
```

not:

```text
Internet
      |
      v
Public Redis Instance
```

---

# 8.36 Production Considerations

## 1. Redis Should Not Be the Source of Truth

For banking data:

```text
MySQL / RDS
      |
      v
Source of Truth
```

Redis remains:

```text
Performance Layer
```

---

## 2. Cache Failure Should Be Handled Gracefully

```text
Redis Down
     |
     v
Fallback where appropriate
     |
     v
Database
```

The system should not unnecessarily fail an otherwise valid read simply because the cache is unavailable.

---

## 3. Cache Invalidation Must Follow Data Changes

For cached balance data:

```text
Balance Update
      |
      v
Cache Invalidated
```

Otherwise stale data may remain visible.

---

## 4. Sensitive Data Should Be Evaluated Carefully

Not every piece of banking information should be cached.

Before caching, consider:

```text
Sensitivity
TTL
Consistency requirements
Access control
Memory usage
```

---

## 5. Monitor Memory Usage

Redis stores data in memory.

Therefore:

```text
More keys
    |
    v
More memory
```

A production environment should monitor:

```text
Memory
Evictions
Latency
Connections
```

---

# 8.37 Current CitiCore Implementation vs Future Improvements

## Current CitiCore Implementation

The notes document:

```text
Redis
   |
   +---- Account balance caching
   |
   +---- 30-minute TTL
   |
   +---- Cache-aside read flow
   |
   +---- Cache invalidation for balance changes
   |
   +---- API Gateway rate-limiting support
```

The documented Account Service implementation uses:

```text
StringRedisTemplate
balance:<accountNumber>
30-minute TTL
```



---

## Future Production Improvements

Possible future improvements include:

```text
Redis high availability
Redis Cluster
Centralized monitoring
Cache fallback policies
Cache stampede protection
Better metrics
Key naming standards
Distributed tracing
```

These should be described as improvements rather than existing CitiCore functionality unless implemented and verified later.

---

# 8.38 Trade-offs

## Advantages

### Faster Repeated Reads

```text
Redis
   |
   v
Memory-based access
```

can avoid repeated database queries.

---

### Reduced Database Load

```text
Repeated Reads
      |
      v
Redis
      |
      v
Fewer Database Queries
```

---

### Distributed Rate Limiting Support

Redis can provide shared counters/state across multiple API Gateway instances.

---

### Simple Cache-Aside Pattern

The application controls:

```text
Read
Write
Cache
Invalidation
```

which makes the behavior explicit.

---

## Disadvantages

### Stale Data

Cached data can become outdated.

---

### Cache Invalidation Complexity

Every relevant data update must consider the cache.

---

### Additional Infrastructure

The system now depends on:

```text
Redis
```

as another infrastructure component.

---

### Memory Constraints

Redis data consumes memory.

---

### Distributed Consistency Complexity

The application must handle:

```text
Database state
      +
Cache state
```

correctly.

---

# 8.39 Alternatives

## No Cache

```text
Application
     |
     v
Database
```

Simpler but may create more database load.

---

## Local In-Memory Cache

```text
Service Instance
      |
      v
Local Cache
```

Fast but difficult to share or invalidate across multiple service instances.

---

## Redis

```text
Service 1
Service 2
Service 3
     |
     v
   Redis
```

Useful for distributed systems because the cache can be shared.

---

## Write-Through Cache

```text
Application Write
      |
      +---- Database
      |
      +---- Cache
```

Can reduce stale data but adds write-path complexity.

CitiCore's documented implementation instead uses cache invalidation for balance updates.

---

# 8.40 How I Would Explain Redis in an Interview

> "In CitiCore, I used Redis primarily as a caching layer to reduce repeated database reads for account balances. The Account Service follows a cache-aside pattern. When a balance is requested, it first checks Redis using a key such as `balance:<accountNumber>`. On a cache hit, it returns the value directly. On a miss, it reads the balance from the database, stores it in Redis with a 30-minute TTL, and returns it.
>
> When the balance changes through operations such as deposit or withdrawal, the cache entry is invalidated so that the next read loads a fresh value from the database.
>
> The database remains the source of truth. Redis is used as a performance optimization, so critical consistency-sensitive operations should not rely blindly on stale cached data. Redis is also used in the architecture to support API Gateway rate limiting."

---

# 8.41 30-Second Interview Answer

> "Redis in CitiCore is mainly used as a distributed caching layer and for API Gateway rate limiting. For account balances, we use the cache-aside pattern: Redis is checked first, and on a cache miss the value is loaded from MySQL and cached with a 30-minute TTL. When the balance changes, we invalidate the cache. MySQL remains the source of truth because Redis is only a performance layer."

---

# 8.42 Interview Follow-up Questions

## Q1. Why did you choose Redis?

**Answer:**

Redis is an in-memory data store suitable for low-latency caching and shared distributed state. In CitiCore, it reduces repeated database reads and supports API Gateway rate limiting.

---

## Q2. What caching pattern did you use?

**Answer:**

CitiCore uses the cache-aside pattern for account balance caching. The application checks Redis first. On a miss, it loads the value from the database and stores it in Redis.

---

## Q3. What happens on a cache hit?

**Answer:**

The application returns the value directly from Redis without querying the database.

---

## Q4. What happens on a cache miss?

**Answer:**

The application queries the database, stores the retrieved value in Redis with the configured TTL, and returns the result.

---

## Q5. What cache key did you use?

**Answer:**

The documented implementation uses:

```text
balance:<accountNumber>
```

The `balance:` prefix helps organize Redis keys.

---

## Q6. What TTL did you use?

**Answer:**

The documented CitiCore balance cache uses a TTL of 30 minutes.

---

## Q7. How do you prevent stale data?

**Answer:**

The current implementation invalidates the balance cache when balance-changing operations such as deposit or withdrawal occur. The next read then loads fresh data from the database.

---

## Q8. Is Redis the source of truth?

**Answer:**

No. MySQL/RDS is the persistent source of truth. Redis is used as a temporary performance layer.

---

## Q9. What happens if Redis is unavailable?

**Answer:**

For cacheable reads, the application should ideally fall back to the database where appropriate. Redis failure is primarily a performance problem, while database failure affects access to the authoritative data.

The current CitiCore notes document the cache implementation but do not clearly document a specific custom Redis fallback mechanism.

---

## Q10. Can cached balance data be stale?

**Answer:**

Yes. That is why balance-changing operations must invalidate or update the cache. Critical financial operations should also consider strong consistency requirements.

---

## Q11. Why not use local in-memory caching?

**Answer:**

Local caches are isolated per application instance. In a microservices deployment with multiple instances, cache state becomes harder to share and invalidate consistently. Redis provides centralized shared caching.

---

## Q12. What is cache invalidation?

**Answer:**

It means removing or updating a cached entry when the underlying data changes.

In CitiCore:

```text
Balance Update
      |
      v
Delete balance cache key
```

---

## Q13. What is a cache stampede?

**Answer:**

It occurs when many requests experience a cache miss simultaneously and all query the database, potentially creating a sudden load spike.

The current CitiCore implementation does not document a specific stampede protection mechanism.

---

## Q14. Why is Redis useful for rate limiting?

**Answer:**

Rate limiting requires shared counters or state. Redis allows multiple API Gateway instances to use the same centralized rate-limit data.

---

# 8.43 Key Things to Remember

```text
Redis
  |
  +---- In-memory data store
  |
  +---- Used for caching
  |
  +---- Used for distributed rate limiting
```

CitiCore balance flow:

```text
Request
   |
   v
Redis
   |
   +---- HIT
   |       |
   |       v
   |   Return Balance
   |
   +---- MISS
           |
           v
        Database
           |
           v
        Cache Result
           |
           v
        Return Balance
```

Balance update:

```text
Database Update
      |
      v
Invalidate Cache
      |
      v
Next Read Loads Fresh Data
```

Core principle:

> **Redis improves performance, but MySQL remains the source of truth.**

---

# 8.44 Topic Summary

CitiCore uses Redis as a distributed performance and infrastructure component.

The documented implementation includes:

```text
Account Service
      |
      +---- Balance caching
      |
      +---- Cache-aside pattern
      |
      +---- balance:<accountNumber>
      |
      +---- 30-minute TTL
      |
      +---- Cache invalidation after balance changes
```

The broader CitiCore architecture also uses Redis to support:

```text
API Gateway
      |
      v
Rate Limiting
```

The most important design principle is:

```text
Redis
  |
  v
Cache / Performance Layer

MySQL
  |
  v
Persistent Source of Truth
```
---

# TOPIC 9 — DATABASE ARCHITECTURE

# 9. Database Architecture

## 9.1 Overview

The database architecture of CitiCore was designed around an important banking-system requirement:

> **Not every database operation has the same consistency and performance requirements.**

For example, consider these operations:

* Creating an account
* Depositing money
* Withdrawing money
* Validating a transfer
* Checking an account balance
* Viewing transaction history
* Viewing account statements

Some operations require the **latest and strongly consistent data**.

Others are primarily read-heavy and can tolerate a small replication delay.

To handle these different requirements, the CitiCore Account Service notes describe a database architecture based on:

* MySQL
* Primary/Replica architecture
* Read/write separation
* Application-level datasource routing
* Separate HikariCP connection pools
* Strong reads from the Primary database
* Eventual-consistent reads from the Replica database
* Replica health monitoring
* Fallback to Primary when the Replica is unhealthy

The overall design can be represented as:

```text
                         Account Service
                               |
                               v
                    DataSource Routing Layer
                               |
                  +------------+------------+
                  |                         |
                  v                         v
           PRIMARY DATABASE          REPLICA DATABASE
           Read + Write              Read Operations
                  |                         ^
                  |                         |
                  +------ Replication ------+
```

The primary database remains the authoritative database for writes and consistency-sensitive reads. The replica is used to scale selected read operations. 

---

# 9.2 What Is Database Architecture?

## Concept

Database architecture describes how an application organizes and accesses its data infrastructure.

In a simple application:

```text
Application
     |
     v
Single Database
```

All operations go to the same database.

This is easy to manage, but as traffic grows:

```text
Reads
   +
Writes
   +
Transactions
   +
Reports
   +
History Queries
       |
       v
Single Database
       |
       v
Potential Bottleneck
```

A more scalable architecture separates database responsibilities.

In CitiCore:

```text
                       Application
                            |
                            v
                 Database Routing Layer
                            |
              +-------------+-------------+
              |                           |
              v                           v
         Primary DB                   Replica DB
       Read + Write                  Selected Reads
```

This approach allows the system to decide which database should handle a particular operation.

---

# 9.3 Why Do We Need This Architecture?

A banking system handles different categories of data operations.

For example:

### Critical Operations

```text
Deposit
Withdrawal
Transfer Validation
Balance Validation
```

These operations need the latest data.

### Read-Heavy Operations

```text
Account Listing
Transaction History
Account Statements
Historical Queries
```

Some of these operations can tolerate a small replication delay.

If everything goes to one database:

```text
                Primary Database
                       |
        +--------------+--------------+
        |              |              |
        v              v              v
     Writes        Balance Reads   History Reads
```

then read traffic competes with important write operations.

The CitiCore architecture separates them:

```text
                         Account Service
                                |
                +---------------+---------------+
                |                               |
                v                               v
         Critical Operations              Read-heavy Operations
                |                               |
                v                               v
           Primary DB                      Replica DB
```

The purpose is not simply to make every query faster.

The purpose is to balance:

```text
Consistency
     +
Performance
     +
Scalability
     +
Availability
```

---

# 9.4 CitiCore Database Architecture

## CitiCore Implementation

The CitiCore notes describe the following architecture for the Account Service:

```text
┌──────────────────────────────────────────────┐
│              Account Service                 │
│              Spring Boot                     │
│                                              │
│  Service Layer                               │
│        |                                     │
│        v                                     │
│  Routing Context / AOP                       │
│        |                                     │
│        v                                     │
│  AbstractRoutingDataSource                   │
│        |                                     │
│   +----+--------------------+                │
│   |                         |                │
│   v                         v                │
│ Primary DataSource     Replica DataSource    │
│   HikariCP                 HikariCP          │
└───────┬──────────────────────────┬───────────┘
        |                          |
        v                          v
  MySQL / RDS Primary       MySQL / RDS Replica
        |                          ^
        |                          |
        +------ Replication -------+
```

The implementation includes:

* A Primary datasource
* A Replica datasource
* HikariCP connection pooling for each datasource
* Application-level routing
* A routing context
* Read and Primary-read annotations
* Replica health checks
* Primary fallback when the Replica is unhealthy

The database notes specifically describe the use of custom routing logic with `ThreadLocal` context and `AbstractRoutingDataSource`.

---

# 9.5 Primary Database

## What Is the Primary?

The Primary database is the main database responsible for accepting data changes.

Examples include:

```text
INSERT
UPDATE
DELETE
```

In CitiCore, critical banking operations are routed to the Primary.

Examples:

```text
Create Account
Deposit
Withdrawal
Fund Transfer
Balance Validation
Critical Account Updates
```

Conceptually:

```text
Client
   |
   v
Account Service
   |
   v
PRIMARY DATABASE
   |
   v
Write Operation
```

The Primary is also used for reads where stale data could cause a business or financial correctness issue.

---

# 9.6 Why Writes Must Go to the Primary

Consider a withdrawal.

```text
Customer
   |
   v
Withdraw ₹500
   |
   v
Current Balance?
```

The system must check the latest balance before allowing the withdrawal.

Suppose:

```text
Actual Primary Balance = ₹1,000
Replica Balance        = ₹1,500
```

If the application validates the withdrawal using the stale replica value:

```text
Replica
   |
   v
₹1,500 available
```

the system could make an incorrect decision.

Therefore:

```text
Critical Validation
        |
        v
      PRIMARY
```

The important principle is:

> **A read operation is not automatically safe to send to a Replica. The business consistency requirement determines where the read should go.**

---

# 9.7 Replica Database

## What Is a Replica?

A database replica is a copy of the Primary database.

Changes are replicated from:

```text
PRIMARY
   |
   v
REPLICA
```

The replica can then serve selected read operations.

The CitiCore notes describe MySQL replication as asynchronous, meaning there may be a delay between:

```text
Primary Write
```

and:

```text
Replica Update
```

This delay is called:

```text
Replica Lag
```

---

# 9.8 Why Use a Replica?

A replica helps separate read traffic from write traffic.

Without a replica:

```text
                        Primary
                           |
        +------------------+------------------+
        |                  |                  |
        v                  v                  v
      Writes          Balance Reads      History Reads
```

With a replica:

```text
                        Application
                            |
                +-----------+-----------+
                |                       |
                v                       v
             Primary                 Replica
                |                       |
                v                       v
        Writes + Strong Reads      Read-heavy Queries
```

This allows the Primary to focus more on critical operations.

The replica can handle selected read traffic.

---

# 9.9 MySQL Replication

## Concept

Replication is the process of copying changes from one database server to another.

The general flow is:

```text
Primary Database
       |
       | INSERT / UPDATE / DELETE
       v
Binary Log
       |
       v
Replica Reads Changes
       |
       v
Replica Applies Changes
       |
       v
Replica Database
```

The CitiCore notes describe MySQL replication using:

* Binary logs
* Row-based replication
* GTIDs
* Replica relay logs
* Read-only replica configuration

The detailed MySQL replication implementation will be covered only once in the authoritative **Database Architecture / MySQL Replication explanation**, while the upcoming partitioning topic will focus specifically on partitioning rather than repeating replication details. 

---

# 9.10 Primary and Replica Flow

The basic CitiCore data flow is:

```text
                    WRITE
                      |
                      v
                Account Service
                      |
                      v
                  PRIMARY DB
                      |
                      v
                MySQL Binary Log
                      |
                      v
                 Replication
                      |
                      v
                  REPLICA DB
```

For reads:

```text
                    READ REQUEST
                         |
                         v
                 Account Service
                         |
                         v
                Routing Decision
                         |
              +----------+----------+
              |                     |
              v                     v
         Strong Read          Eventual Read
              |                     |
              v                     v
          PRIMARY DB            REPLICA DB
```

---

# 9.11 Read/Write Separation

## What Is Read/Write Separation?

Read/write separation means that database operations are routed based on their purpose.

For example:

```text
WRITE
  |
  v
PRIMARY
```

and:

```text
NON-CRITICAL READ
       |
       v
REPLICA
```

This is different from simply routing:

```text
All Reads
   |
   v
Replica
```

That approach is dangerous for systems where some reads require strong consistency.

---

# 9.12 CitiCore Read Classification

The CitiCore notes classify database operations into different categories.

## Writes

Writes go to the Primary.

Examples:

```text
Create Account
Update Account
Deposit
Withdrawal
Transfer
Status Change
```

Architecture:

```text
Service Method
      |
      v
Write Operation
      |
      v
PRIMARY
```

---

## Strong Reads

Strong reads also go to the Primary.

Examples documented in the notes include:

```text
Balance Check
Withdrawal Validation
Transfer Validation
Consistency-sensitive operations
```

Architecture:

```text
Service Method
      |
      v
Strong Consistency Required
      |
      v
PRIMARY
```

The reason is replica lag.

---

## Eventual-Consistency Reads

Some read operations can use the Replica.

Examples documented in the notes include:

```text
Account List
Transaction History
Account Statements
Historical Data
```

Architecture:

```text
Service Method
      |
      v
Eventual Consistency Acceptable
      |
      v
REPLICA
```

This is an important design decision.

> **The routing decision depends on the business requirement of the read, not simply on whether the operation is technically a `SELECT`.**

---

# 9.13 Strong Consistency vs Eventual Consistency

## Strong Consistency

Strong consistency means that the application reads the latest committed value.

Example:

```text
Primary Balance = ₹5,000
```

A transaction changes it:

```text
Primary Balance = ₹4,000
```

A subsequent critical read should return:

```text
₹4,000
```

Therefore:

```text
Critical Read
     |
     v
PRIMARY
```

---

## Eventual Consistency

With asynchronous replication:

```text
Primary
   |
   v
Update
   |
   v
Replica
```

the Replica may not immediately contain the latest value.

Example:

```text
Primary = ₹4,000

Replica = ₹5,000
```

for a short period.

For historical or non-critical reads, this may be acceptable.

For financial validation, it is not.

---

# 9.14 Why Not Send All Reads to the Replica?

This is one of the most important interview questions about this architecture.

Suppose a customer transfers money.

```text
Primary Balance
₹5,000
```

Transfer:

```text
Transfer ₹1,000
```

Primary updates:

```text
₹5,000
   |
   v
₹4,000
```

But the Replica has not yet caught up:

```text
Replica
   |
   v
₹5,000
```

If the application immediately reads from the Replica:

```text
GET Balance
   |
   v
Replica
   |
   v
₹5,000
```

the user receives stale information.

For critical business decisions:

```text
Balance Validation
Withdrawal Validation
Transfer Validation
```

this can cause correctness problems.

Therefore CitiCore uses selective routing.

```text
                Read Request
                     |
                     v
          Does it require latest data?
                     |
              +------+------+
              |             |
             YES            NO
              |             |
              v             v
          PRIMARY        REPLICA
```

---

# 9.15 DataSource Routing

## Concept

A Spring Boot application normally has one datasource:

```text
Spring Boot
     |
     v
DataSource
     |
     v
Database
```

CitiCore needs two:

```text
Spring Boot
     |
     +--------+
     |        |
     v        v
Primary    Replica
```

The application therefore needs a mechanism to decide which datasource should be used for the current operation.

This is called:

```text
Datasource Routing
```

---

# 9.16 CitiCore Routing Architecture

The documented CitiCore routing flow is:

```text
HTTP Request
     |
     v
Controller
     |
     v
Service Method
     |
     v
Annotation / AOP
     |
     v
Routing Context
     |
     v
AbstractRoutingDataSource
     |
     +--------------------+
     |                    |
     v                    v
Primary DataSource    Replica DataSource
     |                    |
     v                    v
HikariCP Pool       HikariCP Pool
     |                    |
     v                    v
Primary DB          Replica DB
```

The routing decision is made before the database operation executes.

---

# 9.17 AbstractRoutingDataSource

Spring provides:

```java
AbstractRoutingDataSource
```

This class can route database operations to different datasources dynamically.

Conceptually:

```java
protected Object determineCurrentLookupKey() {
    return DataSourceContextHolder.getDataSourceType();
}
```

The routing datasource asks:

```text
Which database should this request use?
```

The routing context answers:

```text
PRIMARY
```

or:

```text
REPLICA
```

The datasource routing layer then selects the corresponding connection pool.

---

# 9.18 Routing Context

## Concept

The application needs to store the current routing decision.

For example:

```text
Current Request
      |
      v
Use PRIMARY
```

or:

```text
Current Request
      |
      v
Use REPLICA
```

The CitiCore notes describe storing this context using:

```text
ThreadLocal
```

Conceptually:

```text
Request Thread
      |
      v
ThreadLocal Context
      |
      v
PRIMARY / REPLICA
```

---

# 9.19 Why ThreadLocal?

In a typical Spring MVC application, a request is handled by an application thread.

The routing decision belongs to that request.

For example:

```text
Request A
Thread 1
PRIMARY
```

and:

```text
Request B
Thread 2
REPLICA
```

Using `ThreadLocal` allows the routing information to remain associated with the current execution thread.

Conceptually:

```java
private static final ThreadLocal<DataSourceType> CONTEXT =
        new ThreadLocal<>();
```

Then:

```java
CONTEXT.set(DataSourceType.PRIMARY);
```

or:

```java
CONTEXT.set(DataSourceType.REPLICA);
```

---

# 9.20 Important ThreadLocal Problem

`ThreadLocal` must be cleaned up.

This is important because application servers reuse threads.

Suppose:

```text
Request 1
   |
   v
Thread A
   |
   v
REPLICA
```

Request 1 finishes.

Later:

```text
Request 2
   |
   v
Thread A reused
```

If the routing context was not cleared:

```text
Thread A
   |
   v
Still contains REPLICA
```

The new request could accidentally inherit the previous routing decision.

Therefore the routing context should be cleared.

Conceptually:

```java
try {
    // Set routing context
    return proceed();
} finally {
    DataSourceContextHolder.clear();
}
```

The `finally` block is important because cleanup should happen even when an exception occurs.

This is a key implementation detail from the CitiCore notes. 

---

# 9.21 Routing Annotations

The CitiCore notes describe using annotations to classify database operations.

Examples include:

```text
@ReadOnly
```

and:

```text
@PrimaryRead
```

The purpose is to make the consistency requirement explicit.

For example:

```java
@ReadOnly
public List<Account> getAccounts() {
    // Route to replica
}
```

and:

```java
@PrimaryRead
public BigDecimal getBalance(String accountNumber) {
    // Route to primary
}
```

The exact implementation can use AOP to intercept these annotations and set the routing context.

---

# 9.22 Why Annotation-Based Routing?

Without explicit routing, developers may accidentally send critical operations to the wrong database.

Annotation-based routing makes the intention visible.

For example:

```java
@PrimaryRead
public BigDecimal getBalance(...) {
```

immediately communicates:

```text
This read requires the Primary.
```

Similarly:

```java
@ReadOnly
public List<AccountStatement> getStatements(...) {
```

communicates:

```text
This operation can tolerate Replica reads.
```

This improves:

* Readability
* Consistency
* Maintainability
* Code review
* Interview explanation

---

# 9.23 Routing Flow Step by Step

Consider a strong read.

### Step 1 — Request Arrives

```text
Client
   |
   v
Account Service
```

### Step 2 — Service Method Is Invoked

```text
Service Method
```

Example:

```text
getBalance()
```

### Step 3 — Routing Annotation Is Detected

```text
@PrimaryRead
```

### Step 4 — AOP Sets Routing Context

```text
ThreadLocal
   |
   v
PRIMARY
```

### Step 5 — Routing Datasource Checks Context

```text
AbstractRoutingDataSource
   |
   v
PRIMARY
```

### Step 6 — Primary Connection Pool Is Selected

```text
Primary HikariCP
```

### Step 7 — Query Executes

```text
PRIMARY DATABASE
```

### Step 8 — Context Is Cleared

```text
finally
   |
   v
ThreadLocal.remove()
```

---

# 9.24 Replica Read Flow

For an eventual-consistency read:

```text
Client
   |
   v
Account Service
   |
   v
@ReadOnly
   |
   v
AOP
   |
   v
ThreadLocal = REPLICA
   |
   v
AbstractRoutingDataSource
   |
   v
Replica HikariCP
   |
   v
Replica Database
```

After execution:

```text
Routing Context Cleared
```

---

# 9.25 HikariCP

## What Is HikariCP?

HikariCP is a JDBC connection pool.

A database connection is relatively expensive to create.

Without pooling:

```text
Request
   |
   v
Create DB Connection
   |
   v
Execute Query
   |
   v
Close Connection
```

Repeated connection creation can add unnecessary overhead.

With HikariCP:

```text
Request
   |
   v
Connection Pool
   |
   v
Reusable Connection
   |
   v
Database
```

After use:

```text
Connection
   |
   v
Returned to Pool
```

---

# 9.26 CitiCore Connection Pool Architecture

CitiCore uses separate pools conceptually:

```text
                     Routing Layer
                          |
                +---------+---------+
                |                   |
                v                   v
         Primary HikariCP      Replica HikariCP
                |                   |
                v                   v
             Primary              Replica
```

This separation ensures that the application can independently manage connections to each database endpoint.

For example:

```text
Primary Pool
   |
   +---- Critical Writes
   |
   +---- Strong Reads
```

and:

```text
Replica Pool
   |
   +---- Read-heavy Queries
```

The database notes describe this dual-pool architecture as part of the Account Service design.

---

# 9.27 Why Separate Connection Pools?

If both databases shared the same connection configuration conceptually:

```text
Application
   |
   v
Single Pool
   |
   +---- Primary
   |
   +---- Replica
```

routing and monitoring would become more complicated.

Separate pools provide clearer boundaries:

```text
Primary Pool
   |
   v
Primary Database
```

and:

```text
Replica Pool
   |
   v
Replica Database
```

This allows independent monitoring of:

```text
Active Connections
Idle Connections
Connection Failures
Pool Exhaustion
```

for each database.

---

# 9.28 Replica Lag

## What Is Replica Lag?

Replica lag is the delay between a change being committed on the Primary and that same change becoming available on the Replica.

Example:

```text
Time 0

Primary Balance = ₹1,000
Replica Balance = ₹1,000
```

A transaction occurs:

```text
Time 1

Primary Balance = ₹500
Replica Balance = ₹1,000
```

After replication catches up:

```text
Time 2

Primary Balance = ₹500
Replica Balance = ₹500
```

The period between Time 1 and Time 2 is the replication lag window.

---

# 9.29 Why Replica Lag Matters in Banking

Replica lag is especially important in banking operations.

Consider:

```text
Customer Balance = ₹1,000
```

A withdrawal occurs:

```text
Withdraw ₹800
```

Primary:

```text
₹1,000
   |
   v
₹200
```

Replica may temporarily remain:

```text
₹1,000
```

If another operation checks the Replica:

```text
Available Balance?
   |
   v
₹1,000
```

the application could make an incorrect decision.

Therefore:

```text
Financial Validation
      |
      v
PRIMARY
```

---

# 9.30 CitiCore Replica Lag Strategy

The CitiCore notes describe a selective routing approach.

```text
                    READ
                     |
                     v
         Does this require latest data?
                     |
             +-------+-------+
             |               |
            YES              NO
             |               |
             v               v
         PRIMARY          REPLICA
```

Examples of Primary reads:

```text
Balance Check
Withdrawal Validation
Transfer Validation
```

Examples of Replica reads:

```text
Account List
Transaction History
Historical Statements
```

This is one of the key architectural decisions in CitiCore.

---

# 9.31 Replica Health Monitoring

A Replica should not be used blindly.

The application must consider whether the Replica is:

```text
Available
Healthy
Connected
Acceptable for reads
```

The CitiCore notes describe monitoring Replica health and falling back to the Primary when the Replica is unavailable or significantly lagging.

Conceptually:

```text
Read Request
      |
      v
Replica Health Check
      |
      +---- Healthy
      |        |
      |        v
      |    Use Replica
      |
      +---- Unhealthy
               |
               v
          Use Primary
```

This improves availability for read operations.

---

# 9.32 Replica Fallback

Suppose:

```text
Replica
   |
   X
Unavailable
```

Without fallback:

```text
Read Request
   |
   v
ERROR
```

With fallback:

```text
Read Request
   |
   v
Replica Available?
   |
   +---- YES
   |       |
   |       v
   |    Replica
   |
   +---- NO
           |
           v
        Primary
```

This allows the application to continue serving reads.

However, the Primary may receive additional load.

Therefore, fallback improves availability but can increase pressure on the Primary.

---

# 9.33 Replica Lag Threshold

The existing notes mention monitoring replication lag and using a threshold-based fallback strategy.

Conceptually:

```text
Replica Lag
     |
     v
Within Acceptable Threshold?
     |
 +---+---+
 |       |
YES      NO
 |       |
 v       v
Replica  Primary
```

The source notes mention a specific lag threshold as part of an implementation explanation, but that value should be treated as a project configuration decision rather than a universal production standard. 

A production threshold should depend on:

* Business requirements
* Query type
* Data sensitivity
* Expected replication behavior
* System load

---

# 9.34 Default-to-Primary Strategy

The CitiCore notes recommend using Primary as the safe default.

Conceptually:

```text
Routing Context Exists?
      |
   +--+--+
   |     |
  YES    NO
   |     |
   v     v
Use     PRIMARY
Context
```

Why?

Because accidentally reading from the Primary generally gives fresh data, while accidentally reading critical data from a lagging Replica can create correctness issues.

This does not mean all traffic should always use the Primary.

It means that when the routing context is missing or uncertain, a correctness-first default is safer.

---

# 9.35 Database Operation Classification

A useful CitiCore classification is:

| Operation           | Database | Reason                                           |
| ------------------- | -------- | ------------------------------------------------ |
| Create Account      | Primary  | Write operation                                  |
| Deposit             | Primary  | Balance update                                   |
| Withdrawal          | Primary  | Balance update and validation                    |
| Fund Transfer       | Primary  | Critical financial consistency                   |
| Balance Validation  | Primary  | Latest data required                             |
| Account List        | Replica  | Read scaling where lag is acceptable             |
| Statement History   | Replica  | Historical read workload                         |
| Transaction History | Replica  | Read-heavy, depending on consistency requirement |

The important rule is:

> **The business requirement determines the datasource, not only the SQL operation type.**

For example:

```text
SELECT Balance
```

is technically a read.

But:

```text
SELECT Balance before Withdrawal
```

is a consistency-critical operation.

Therefore:

```text
SELECT
≠
Automatically Replica
```

---

# 9.36 Strong Reads and Redis

CitiCore also uses Redis for balance caching.

However, caching and datasource routing solve different problems.

```text
Redis
   |
   v
Reduce repeated reads
```

while:

```text
Primary / Replica Routing
   |
   v
Control consistency and database load
```

For consistency-sensitive operations:

```text
Critical Operation
   |
   v
Do not blindly trust stale cached data
   |
   v
Use authoritative data path
```

This distinction is important in a banking application.

The Redis topic contains the authoritative explanation of cache behavior, TTL, and invalidation, so those concepts are not repeated here.

---

# 9.37 Database Source of Truth

The architecture should be understood as:

```text
                    Source of Truth
                         |
                         v
                     Primary DB
                         |
             +-----------+-----------+
             |                       |
             v                       v
         Replica DB              Redis Cache
```

The roles are different.

### Primary

```text
Authoritative Writes
Strong Reads
```

### Replica

```text
Replicated Read Scaling
Eventual Consistency
```

### Redis

```text
Performance Cache
Temporary Data
```

---

# 9.38 Failure Scenario — Replica Unavailable

Consider:

```text
Client
   |
   v
Account Service
   |
   v
Replica
   |
   X
Unavailable
```

The routing layer should detect the failure.

Then:

```text
Client
   |
   v
Account Service
   |
   v
Replica Unavailable
   |
   v
Fallback
   |
   v
Primary
```

### Trade-off

The benefit:

```text
Read remains available
```

The cost:

```text
Primary receives more load
```

Therefore fallback is useful for availability but should be monitored.

---

# 9.39 Failure Scenario — Replica Lag

Consider:

```text
Primary
   |
   v
New Balance = ₹500
```

Replica:

```text
Old Balance = ₹1,000
```

A critical read should not use the Replica.

The CitiCore strategy is:

```text
Critical Read
      |
      v
PRIMARY
```

For non-critical reads:

```text
Historical Read
      |
      v
REPLICA
```

This selective approach is safer than routing all reads to the Replica.

---

# 9.40 Failure Scenario — Connection Pool Exhaustion

A database connection pool can become exhausted when:

```text
Too many concurrent requests
```

or:

```text
Connections are not returned
```

or:

```text
Connection leaks exist
```

The flow can become:

```text
Request
   |
   v
HikariCP
   |
   v
No Available Connection
   |
   v
Timeout / Error
```

Useful monitoring includes:

```text
Active Connections
Idle Connections
Pending Requests
Connection Timeouts
Connection Errors
```

The source notes include connection-pool exhaustion as a troubleshooting scenario and recommend monitoring HikariCP metrics. 

Detailed troubleshooting will be consolidated later under **Topic 16 — Troubleshooting** to avoid duplicating the same problem explanations.

---

# 9.41 Architecture Decision: Why Not Use Only One Database?

A single database architecture would be:

```text
Application
     |
     v
Single MySQL Database
     |
     +---- Writes
     +---- Strong Reads
     +---- History Queries
     +---- Statement Queries
```

### Advantages

* Simple architecture
* Easier deployment
* No replication lag
* No routing logic

### Disadvantages

* Read traffic competes with writes
* Harder to scale read workloads independently
* Heavy reporting queries can affect critical operations

The Primary/Replica approach adds complexity but provides better separation.

---

# 9.42 Architecture Decision: Why Not Send Every Read to the Replica?

Because:

```text
Replica
   |
   v
Eventually Consistent
```

Some reads require:

```text
Latest Data
```

Therefore:

```text
Consistency-sensitive Reads
           |
           v
        Primary
```

and:

```text
Scalable Read Operations
           |
           v
        Replica
```

This is a business-driven consistency decision.

---

# 9.43 Architecture Decision: Why Application-Level Routing?

There are several ways to manage read/write routing.

One approach is infrastructure-level routing.

Another is application-level routing.

CitiCore uses application-aware routing because the application understands the business meaning of an operation.

For example:

```text
getBalance()
```

could be:

```text
Critical validation
```

or:

```text
Simple display operation
```

The SQL statement alone may not contain enough business context to determine the correct datasource.

Application-level routing allows:

```text
Business Requirement
       |
       v
Routing Decision
```

---

# 9.44 Trade-offs

## Advantages

### Read Scalability

```text
Primary
   +
Replica
```

allows selected read workloads to be separated from critical writes.

---

### Better Primary Protection

Heavy read workloads can be moved to the Replica.

```text
Historical Reads
      |
      v
Replica
```

This can reduce unnecessary pressure on the Primary.

---

### Strong Consistency Where Needed

Critical operations remain on the Primary.

```text
Balance Validation
       |
       v
Primary
```

---

### Improved Availability

Replica fallback strategies can redirect reads to the Primary when necessary.

---

### Clear Business Classification

Annotations and routing logic make consistency decisions explicit.

---

## Disadvantages

### More Infrastructure

The system must manage:

```text
Primary
Replica
Replication
Connection Pools
Monitoring
Routing
```

---

### Replica Lag

The Replica may contain stale data.

---

### Application Complexity

Developers must correctly classify operations.

A wrong routing decision can cause correctness problems.

---

### Operational Monitoring

The system needs monitoring for:

```text
Replica Health
Replication Lag
Connection Pools
Database Availability
```

---

# 9.45 Production Considerations

A production database architecture would require additional considerations.

## Monitoring

Monitor:

```text
Primary CPU
Replica CPU
Connection Count
Query Latency
Replica Lag
Replication Errors
Disk Usage
Database Availability
```

---

## Backup

Replication is not the same as backup.

This is important.

```text
Primary
   |
   v
Replica
```

copies changes.

If incorrect data is written:

```text
Bad Data
   |
   v
Primary
   |
   v
Replica
```

the error can also replicate.

Therefore production systems still require:

```text
Automated Backups
Snapshots
Point-in-Time Recovery
```

---

## Failover

A production system should have a documented process for:

```text
Primary Failure
      |
      v
Promote / Failover
      |
      v
Restore Application Connectivity
```

The exact AWS RDS failover architecture belongs to the AWS Infrastructure topic.

---

## Connection Pool Tuning

HikariCP settings should be based on:

```text
Database Capacity
Application Instances
Concurrent Requests
Query Duration
```

A larger pool is not automatically better.

Too many connections can overload the database.

---

## Query Optimization

Read/Write separation does not replace query optimization.

Poor queries remain poor queries.

The system should still use:

```text
Indexes
Query Analysis
Pagination
Partitioning where appropriate
Efficient SQL
```

---

# 9.46 Important Separation of Concerns

The CitiCore documentation should treat these concepts separately:

### Database Architecture

```text
Primary
Replica
Routing
Consistency
Connection Pools
Health
```

### MySQL Partitioning

```text
Large Tables
Range Partitions
Partition Pruning
Maintenance
```

### Database Security

```text
TLS
Database Users
Read-only Users
Secrets
Truststore
```

### Transactional Outbox

```text
Database Transaction
Event Persistence
Reliable Kafka Publishing
```

### Redis

```text
Caching
TTL
Invalidation
Rate Limiting Support
```

This avoids explaining the same concept multiple times.

---

# 9.47 How I Would Explain the Database Architecture in an Interview

> "In CitiCore, I designed the Account Service database layer using a MySQL Primary and Replica architecture with application-level read/write routing. All writes and consistency-sensitive reads, such as balance validation and withdrawal checks, go to the Primary because the Replica can have asynchronous replication lag.
>
> Read-heavy operations where eventual consistency is acceptable, such as historical statements or account listing, can be routed to the Replica. I used separate HikariCP connection pools and Spring's `AbstractRoutingDataSource` to dynamically select the datasource. The routing decision is stored in a request-scoped `ThreadLocal` context and controlled through annotations and AOP.
>
> We also monitor Replica availability and fall back to the Primary when the Replica cannot safely serve reads. The key design decision was that not every read is safe for the Replica—the business consistency requirement determines where the query should go."

---

# 9.48 30-Second Interview Answer

> "CitiCore uses a MySQL Primary-Replica architecture with read/write separation. Writes and strong consistency reads go to the Primary, while selected read-heavy operations go to the Replica. We implemented application-level routing using Spring's `AbstractRoutingDataSource`, separate HikariCP pools, and a `ThreadLocal` routing context controlled through annotations and AOP. This allows us to scale read traffic while protecting critical banking operations from replica lag."

---

# 9.49 Interview Follow-up Questions

## Q1. Why did you use a Primary and Replica architecture?

**Answer:**

To separate write traffic from selected read traffic. This helps scale read-heavy workloads without sending every request to the Primary database.

---

## Q2. Do all reads go to the Replica?

**Answer:**

No. Reads requiring the latest data, such as balance validation and financial transaction checks, go to the Primary. Only reads that can tolerate eventual consistency are routed to the Replica.

---

## Q3. What is replica lag?

**Answer:**

Replica lag is the delay between a change being committed on the Primary and that change becoming available on the Replica.

---

## Q4. How did you handle replica lag?

**Answer:**

I used selective routing. Consistency-sensitive reads go directly to the Primary, while non-critical read-heavy operations can use the Replica. Replica health can also be monitored and unhealthy replicas can fall back to the Primary.

---

## Q5. Why did you use `ThreadLocal`?

**Answer:**

The datasource routing decision belongs to the current request execution context. `ThreadLocal` allows the routing information to remain associated with the current application thread.

The important point is that it must be cleared after the request or method execution to prevent thread reuse from causing routing errors.

---

## Q6. What happens if you forget to clear the ThreadLocal?

**Answer:**

Application server threads can be reused. A new request could inherit the previous request's datasource routing decision, potentially sending queries to the wrong database.

---

## Q7. What is `AbstractRoutingDataSource`?

**Answer:**

It is a Spring abstraction that allows the application to dynamically choose a datasource based on a lookup key, such as `PRIMARY` or `REPLICA`.

---

## Q8. Why separate HikariCP pools?

**Answer:**

Separate pools provide independent connection management for the Primary and Replica databases and make routing, monitoring, and failure handling clearer.

---

## Q9. What happens if the Replica is unavailable?

**Answer:**

For reads that can be served safely from the Primary, the application can fall back to the Primary. This improves availability but may increase load on the Primary.

---

## Q10. Does replication replace backups?

**Answer:**

No. Replication copies changes from the Primary to the Replica, including potentially incorrect or accidental changes. Backups and point-in-time recovery are still required.

---

## Q11. Why not use one database for everything?

**Answer:**

A single database is simpler, but all reads and writes compete for the same resources. Primary-Replica architecture allows selected read workloads to scale independently.

---

## Q12. How do you decide whether a query should use Primary or Replica?

**Answer:**

Based on the business consistency requirement. If stale data could cause an incorrect business or financial decision, the query uses the Primary. If eventual consistency is acceptable, it can use the Replica.

---

# 9.50 Key Takeaways

```text
CitiCore Database Architecture
          |
          v
MySQL Primary + Replica
          |
    +-----+-----+
    |           |
    v           v
Primary      Replica
    |           |
    v           v
Writes       Selected Reads
Strong Reads Eventual Reads
```

Routing:

```text
Read Request
     |
     v
Latest Data Required?
     |
 +---+---+
 |       |
YES      NO
 |       |
 v       v
Primary Replica
```

Application architecture:

```text
Service
   |
   v
AOP / Annotation
   |
   v
ThreadLocal Context
   |
   v
AbstractRoutingDataSource
   |
   +----------+
   |          |
   v          v
Primary     Replica
HikariCP    HikariCP
```

Core principle:

> **In CitiCore, database routing is based on consistency requirements, not simply whether an operation is a read or write.**

---

# 9.51 Topic Summary

The CitiCore database architecture is built around a **Primary-Replica MySQL design with intelligent application-level routing**.

The key components are:

```text
MySQL
   |
   +---- Primary
   |       |
   |       +---- Writes
   |       +---- Strong Reads
   |
   +---- Replica
           |
           +---- Eventual-Consistency Reads
```

The Account Service determines the appropriate database using:

```text
Annotations
   |
   v
AOP
   |
   v
ThreadLocal Routing Context
   |
   v
AbstractRoutingDataSource
   |
   v
Primary / Replica HikariCP Pool
```

The most important architectural decision is:

> **Not all reads are equal. Banking operations that require the latest state must use the Primary, while selected read-heavy operations can use the Replica for scalability.**

---
# TOPIC 10 - MySQL Partitioning

# 10.1 Overview

Database partitioning was introduced in CitiCore to address a common problem in transaction-heavy systems: some tables continue to grow over time and eventually become difficult to manage efficiently.

A banking platform continuously generates data.

Examples include:

* Account statements
* Transaction history
* Audit records
* Outbox events
* Notification events
* Historical records

Unlike account master data, which usually grows gradually, historical tables can grow very quickly.

For example:

```text
Accounts
    |
    +---- Customer A
    |        |
    |        +---- Transactions
    |        +---- Statements
    |
    +---- Customer B
    |        |
    |        +---- Transactions
    |        +---- Statements
    |
    +---- Customer C
             |
             +---- Transactions
             +---- Statements
```

Over several months or years, tables containing historical records can become very large.

The CitiCore database design therefore uses MySQL partitioning for tables where time-based growth is expected and where queries commonly access data from a specific date range.

The notes identify the following tables as important candidates for partitioning:

* `account_statements`
* `account_outbox`

The primary partitioning approach is:

> **RANGE partitioning based on time.**

Conceptually:

```text
account_statements
        |
        +----------------------+
        |                      |
        v                      v
January Partition         February Partition
        |                      |
        v                      v
March Partition           April Partition
```

The goal is not simply to split a table into smaller pieces.

The main objectives are:

* Improve management of large historical tables
* Support efficient time-based queries
* Allow partition pruning
* Simplify archival and retention operations
* Allow old data to be removed efficiently
* Improve maintenance of time-series data

---

# 10.2 What Is MySQL Partitioning?

## Concept

MySQL partitioning is a database feature that allows a logically single table to be physically divided into multiple partitions.

From the application's perspective:

```text
SELECT * FROM account_statements;
```

still queries one table.

However, internally, the database can organize the data into partitions.

Conceptually:

```text
Logical Table
account_statements
        |
        v
+-----------------------------+
|                             |
|  Partitioned Table          |
|                             |
+-------------+---------------+
              |
    +---------+---------+
    |         |         |
    v         v         v
Partition  Partition  Partition
Jan 2026   Feb 2026   Mar 2026
```

The application does not normally need to know which partition contains a particular row.

It continues working with:

```text
account_statements
```

as one table.

MySQL uses the partitioning definition to determine where data should be stored and which partitions need to be searched.

---

# 10.3 Why Do We Need Partitioning?

Without partitioning, a historical table continues growing as one large structure.

For example:

```text
account_statements

2024 data
2025 data
2026 data
2027 data
2028 data
...
```

Eventually:

```text
One Very Large Table
```

can create operational challenges.

Examples include:

* Large index sizes
* Longer maintenance operations
* More difficult archival
* Expensive deletion of old data
* Slower range-based scans
* Increasing storage requirements

Partitioning organizes data according to a defined rule.

For CitiCore:

```text
Data
 |
 v
Partition Key
 |
 v
Time Range
 |
 +--------------------------+
 |                          |
 v                          v
Older Partition        Newer Partition
```

For example:

```text
account_statements

p202601
    |
    +---- January 2026 statements

p202602
    |
    +---- February 2026 statements

p202603
    |
    +---- March 2026 statements
```

This is particularly useful because banking statement data is naturally time-oriented.

Users commonly ask questions such as:

```text
Show my statements for this month.

Show transactions from January to March.

Show statements for the previous financial year.
```

These queries naturally align with time-based partitioning.

---

# 10.4 CitiCore Context

## CitiCore Implementation

The CitiCore notes identify two important categories of data that are suitable for partitioning.

### Account Statements

```text
account_statements
```

Account statements are historical records and can continuously grow as users perform banking operations.

A simplified example:

```text
Statement ID
Account ID
Transaction ID
Amount
Balance
Statement Date
Created At
```

The important characteristic is:

```text
Historical
+
Time-based
+
Continuously growing
```

Therefore, range partitioning based on a date or timestamp is a suitable approach.

---

### Account Outbox

```text
account_outbox
```

The transactional outbox stores events that are created during database transactions and later published to Kafka.

Conceptually:

```text
Database Transaction
        |
        +---- Business Data
        |
        +---- Outbox Event
                  |
                  v
             Kafka Publisher
                  |
                  v
               Kafka
```

Outbox tables can also grow continuously because every relevant business operation can generate an event.

Examples:

```text
ACCOUNT_CREATED
ACCOUNT_UPDATED
ACCOUNT_STATUS_CHANGED
TRANSACTION_EVENT
```

After events are processed, the system may retain them for auditing or operational purposes.

Over time:

```text
Outbox Events
      |
      v
Large Historical Table
```

Time-based partitioning can make retention and cleanup easier.

---

# 10.5 Why These Tables Are Better Candidates for Partitioning

Not every database table should be partitioned.

This is important.

Partitioning adds complexity.

For example, a relatively small table such as:

```text
users
```

may not benefit from partitioning.

Similarly:

```text
accounts
```

should not automatically be partitioned simply because it exists in a large application.

Partitioning should solve a real problem.

The CitiCore candidate tables share these characteristics:

```text
Large Growth Potential
        +
Historical Data
        +
Time-based Queries
        +
Retention Requirements
```

This makes them better candidates than small or frequently accessed master tables.

---

# 10.6 Why RANGE Partitioning?

## Concept

RANGE partitioning divides rows based on ranges of values.

For a time-based table:

```text
Date
 |
 +-----------------------------+
 |             |               |
 v             v               v
January      February         March
Partition    Partition        Partition
```

For example:

```text
p202601
January 2026

p202602
February 2026

p202603
March 2026
```

The database decides which partition should contain the row based on the partition key.

---

## Example

Suppose a statement has:

```text
statement_date = 2026-02-15
```

The row belongs to:

```text
February 2026 Partition
```

Conceptually:

```text
New Statement
      |
      v
statement_date
      |
      v
2026-02-15
      |
      v
Partition Rule
      |
      v
p202602
```

This happens based on the partition definition.

---

# 10.7 CitiCore Partitioning Architecture

The high-level architecture is:

```text
Account Service
      |
      v
MySQL
      |
      v
account_statements
      |
      +-----------------------------+
      |             |               |
      v             v               v
   p202601       p202602        p202603
   January       February       March
```

For the outbox:

```text
Business Operation
       |
       v
Database Transaction
       |
       +--------------------------+
       |                          |
       v                          v
Business Table              account_outbox
                                  |
                                  v
                         Partitioned by Time
                                  |
                       +----------+----------+
                       |                     |
                       v                     v
                    Old Events            New Events
```

The partitioning strategy is designed around the lifecycle of historical data.

---

# 10.8 Partition Key

A partition key is the column used by MySQL to determine where a row belongs.

For a time-based table, this is usually a date or timestamp column.

Examples:

```text
statement_date
```

or:

```text
created_at
```

For CitiCore, the partition key should represent the time dimension that best matches how the table is queried and maintained.

This is an important design decision.

For example:

```text
account_statements
```

could be partitioned using:

```text
statement_date
```

if statement queries are primarily based on statement dates.

Similarly:

```text
account_outbox
```

could be partitioned using:

```text
created_at
```

if events are managed and retained according to creation time.

The general rule is:

> **Choose a partition key that matches the table's data lifecycle and common query patterns.**

---

# 10.9 Example: account_statements Partitioning

A simplified table might look like:

```sql
CREATE TABLE account_statements (
    id BIGINT NOT NULL,
    account_id BIGINT NOT NULL,
    transaction_id BIGINT,
    amount DECIMAL(19,2),
    balance DECIMAL(19,2),
    statement_date DATE NOT NULL,
    created_at TIMESTAMP NOT NULL
);
```

A conceptual monthly RANGE partitioning strategy could be:

```sql
PARTITION BY RANGE COLUMNS (statement_date) (
    PARTITION p202601 VALUES LESS THAN ('2026-02-01'),
    PARTITION p202602 VALUES LESS THAN ('2026-03-01'),
    PARTITION p202603 VALUES LESS THAN ('2026-04-01')
);
```

This is an illustrative example showing the partitioning concept.

The important point is:

```text
January Data
      |
      v
p202601
```

```text
February Data
      |
      v
p202602
```

```text
March Data
      |
      v
p202603
```

The exact partition boundaries should be managed according to the final database schema and operational requirements.

---

# 10.10 How RANGE Partitioning Works

Suppose MySQL receives:

```text
statement_date = 2026-02-15
```

The database evaluates the partition rules.

Conceptually:

```text
2026-02-15
     |
     v
Is it before 2026-02-01?
     |
     v
No
     |
     v
Is it before 2026-03-01?
     |
     v
Yes
     |
     v
Store in p202602
```

The application only executes:

```text
INSERT statement
```

The database handles partition placement.

This is one reason partitioning is useful:

```text
Application
     |
     v
Logical Table
     |
     v
Database Handles Partition Placement
```

---

# 10.11 Partition Pruning

## What Is Partition Pruning?

Partition pruning means MySQL can avoid scanning partitions that cannot contain the requested data.

Consider this query:

```sql
SELECT *
FROM account_statements
WHERE statement_date >= '2026-02-01'
  AND statement_date < '2026-03-01';
```

Without partitioning:

```text
Query
 |
 v
Large Table
 |
 v
Potentially Scan Large Amount of Data
```

With effective partition pruning:

```text
Query
 |
 v
Identify Date Range
 |
 v
Relevant Partition
 |
 v
p202602
```

The database can focus on the relevant partition instead of considering unrelated partitions.

Conceptually:

```text
Query: February 2026
        |
        v
+-----------------------------+
| Jan | Feb | Mar | Apr | May |
+-----------------------------+
       ^
       |
Only Relevant Partition
```

This can improve query efficiency when the partition key is used appropriately.

---

# 10.12 Important Limitation of Partition Pruning

Partitioning does not automatically make every query faster.

This is a very important interview point.

Consider:

```sql
SELECT *
FROM account_statements
WHERE account_id = 100;
```

If the table is partitioned by:

```text
statement_date
```

but the query does not restrict the date:

```text
WHERE statement_date ...
```

MySQL may need to inspect multiple partitions.

Conceptually:

```text
Query
 |
 v
account_id = 100
 |
 v
Date Unknown
 |
 +-----+-----+-----+
 |     |     |     |
 v     v     v     v
Jan   Feb   Mar   Apr
```

Partition pruning is most effective when queries include conditions that match the partitioning key.

Therefore:

> **Partitioning strategy should be designed together with query patterns.**

---

# 10.13 Query Pattern Example

A good partition-aware query:

```sql
SELECT *
FROM account_statements
WHERE account_id = 100
  AND statement_date >= '2026-02-01'
  AND statement_date < '2026-03-01';
```

The query contains:

```text
account_id
+
statement_date range
```

This allows MySQL to identify the relevant partition and then use indexes inside that partition.

Conceptually:

```text
Request
   |
   v
Account ID = 100
Date = February
   |
   v
p202602
   |
   v
Index Lookup
   |
   v
Result
```

This is generally more aligned with a time-based partitioning strategy than a query that ignores the partition key.

---

# 10.14 Partitioning Does Not Replace Indexing

This is another important concept.

Some developers assume:

```text
Partitioning
=
No Need for Indexes
```

This is incorrect.

Partitioning and indexing solve different problems.

Partitioning helps determine:

```text
Which part of the table should be searched?
```

Indexes help determine:

```text
How should rows inside that partition be located efficiently?
```

Conceptually:

```text
Query
 |
 v
Partition Pruning
 |
 v
Relevant Partition
 |
 v
Index Lookup
 |
 v
Rows
```

A good design can use both.

For example:

```text
Partition by:
statement_date

Index by:
account_id
```

This can support:

```text
Account Statement Query
       |
       +---- Date Range
       |
       +---- Account ID
```

The exact indexes must still be chosen based on actual query patterns.

---

# 10.15 account_outbox Partitioning

The CitiCore project uses the Transactional Outbox pattern for reliable event publishing.

The general architecture is:

```text
Business Operation
       |
       v
Database Transaction
       |
       +---------------------------+
       |                           |
       v                           v
Business Data                 Outbox Event
       |                           |
       +------------+--------------+
                    |
                    v
              Transaction Commit
                    |
                    v
             Outbox Publisher
                    |
                    v
                  Kafka
```

The outbox table can grow continuously because every important business event may create a record.

For example:

```text
ACCOUNT_CREATED
ACCOUNT_UPDATED
ACCOUNT_DELETED
ACCOUNT_TRANSACTION_CREATED
```

Over time:

```text
account_outbox
       |
       v
Millions of Historical Events
```

A time-based partitioning strategy can help organize this data.

Conceptually:

```text
account_outbox
      |
      +-----------------------------+
      |             |               |
      v             v               v
 January         February         March
 Events           Events          Events
```

---

# 10.16 Why Partition the Outbox Table?

The outbox table has a different lifecycle compared to a normal business table.

An event usually follows a lifecycle such as:

```text
Created
   |
   v
Pending
   |
   v
Published
   |
   v
Processed / Retained
   |
   v
Archived or Removed
```

The table can accumulate a large number of processed events.

Time-based partitions make lifecycle management easier.

For example:

```text
Current Month
     |
     v
Actively Used
```

and:

```text
Older Months
     |
     v
Retention / Archive Candidate
```

Instead of deleting old records individually:

```text
DELETE
FROM account_outbox
WHERE created_at < old_date;
```

a partition-based approach can potentially manage old data more efficiently.

---

# 10.17 Dropping Old Partitions

One major operational advantage of partitioning is retention management.

Suppose:

```text
p202501
p202502
p202503
...
p202601
```

and the system no longer needs:

```text
p202501
```

Conceptually:

```text
DROP PARTITION p202501
```

The data associated with that partition can be removed as a partition operation instead of deleting rows individually.

Conceptually:

```text
Without Partitioning

DELETE millions of rows
        |
        v
Large Delete Operation
        |
        v
Locks / Load / Long Execution
```

Compared with:

```text
With Partitioning

Drop Old Partition
        |
        v
Remove Old Data Segment
```

This can simplify retention operations.

However, dropping a partition permanently removes the data.

Therefore production systems must ensure that:

```text
Backup
+
Retention Policy
+
Compliance Requirements
```

are considered before removing historical partitions.

---

# 10.18 Adding New Partitions

A partitioned table requires future capacity planning.

Suppose the current partitions are:

```text
January
February
March
```

Eventually the system reaches:

```text
April
```

A new partition may need to be created.

Conceptually:

```text
Existing

Jan | Feb | Mar

New Data

Apr
```

The partition structure must be extended.

This means partition maintenance is not a one-time operation.

It becomes an operational responsibility.

A production process should ensure future partitions are created before new data reaches an uncovered range.

Conceptually:

```text
Scheduler / DBA Process
        |
        v
Check Future Partition Availability
        |
        v
Create Next Partition
```

This can be automated in production.

---

# 10.19 Partition Maintenance

Partitioning requires maintenance.

A good operational process includes:

```text
Monitor
   |
   v
Check Partition Ranges
   |
   v
Create Future Partitions
   |
   v
Archive Old Data
   |
   v
Remove Expired Partitions
```

For CitiCore, maintenance would be particularly important for:

```text
account_statements
```

and:

```text
account_outbox
```

because both can continuously grow.

A maintenance strategy should answer:

* How far ahead are partitions created?
* How long is data retained?
* When is data archived?
* When can old partitions be removed?
* What backups exist before removal?

---

# 10.20 Partition Naming Strategy

A clear naming strategy improves database maintenance.

For monthly partitions:

```text
pYYYYMM
```

Examples:

```text
p202601
p202602
p202603
```

This is easy to understand.

When an engineer sees:

```text
p202603
```

they immediately know:

```text
March 2026
```

The naming strategy should remain consistent.

Avoid unclear names such as:

```text
partition1
partition2
partition3
```

because operational maintenance becomes more difficult.

---

# 10.21 Partition Boundaries

Partition boundaries must be carefully defined.

For example:

```text
p202601

Values before:
2026-02-01
```

This contains January data.

The next partition:

```text
p202602

Values before:
2026-03-01
```

contains February data.

Conceptually:

```text
January
[2026-01-01, 2026-02-01)

February
[2026-02-01, 2026-03-01)

March
[2026-03-01, 2026-04-01)
```

The boundary design must avoid:

```text
Overlap
```

and:

```text
Missing Date Ranges
```

---

# 10.22 Partitioning Flow in CitiCore

Consider an account statement being generated.

```text
Customer Transaction
        |
        v
Transaction Processing
        |
        v
Account Statement Created
        |
        v
account_statements
        |
        v
Check statement_date
        |
        v
Determine Partition
        |
        v
Store Row
```

Example:

```text
statement_date
=
2026-02-15
```

Flow:

```text
2026-02-15
      |
      v
Partition Rule
      |
      v
February 2026
      |
      v
p202602
      |
      v
Row Stored
```

The application does not manually select the partition.

MySQL manages the placement based on the partition definition.

---

# 10.23 Query Flow With Partition Pruning

Consider a customer requesting statements for February.

```text
Client
   |
   v
GET /statements?from=2026-02-01&to=2026-02-28
   |
   v
Account Service
   |
   v
Repository
   |
   v
MySQL Query
   |
   v
Partition Key Condition
   |
   v
Partition Pruning
   |
   v
Relevant Partition
   |
   v
Result
```

Conceptually:

```text
All Partitions

Jan | Feb | Mar | Apr

Query = February

     |
     v

    Feb
```

This is the main performance benefit expected from partition-aware queries.

---

# 10.24 Important MySQL Partitioning Rules

Partitioning has important technical rules and limitations.

The exact rules depend on the MySQL version and partitioning strategy, but several principles are important.

---

## Rule 1 — Partitioning Must Solve a Real Problem

Do not partition simply because:

```text
Large Application
```

A table should be partitioned when there is a clear need.

Examples:

```text
Very Large Historical Data
Time-based Retention
Time-based Queries
Operational Cleanup Requirements
```

---

## Rule 2 — Queries Should Align With the Partition Key

Partitioning provides the greatest benefit when query patterns include the partition key.

For example:

```sql
WHERE statement_date BETWEEN ...
```

is naturally aligned with time-based partitioning.

A query that does not use the partition key may need to search multiple partitions.

---

## Rule 3 — Partitioning Does Not Replace Indexing

Use appropriate indexes inside partitions.

Conceptually:

```text
Partition
    |
    v
Relevant Data Segment
    |
    v
Indexes
    |
    v
Efficient Row Lookup
```

---

## Rule 4 — Partition Maintenance Is Required

Future partitions must be planned.

Old partitions must be managed according to retention requirements.

Partitioning creates operational responsibilities.

---

## Rule 5 — Primary Key Design Must Be Considered

MySQL partitioning has important restrictions around unique keys and partition columns.

When designing a partitioned table, primary and unique key definitions must be validated against MySQL's partitioning requirements.

This should be considered during schema design rather than discovered later during migration.

---

## Rule 6 — Avoid Excessive Partition Counts

Creating too many partitions is not automatically better.

For example:

```text
1 Partition Per Hour
```

for a long-running system could create unnecessary operational complexity.

The partition size should balance:

```text
Query Pattern
+
Data Growth
+
Maintenance
+
Operational Complexity
```

---

# 10.25 Monthly vs Yearly Partitioning

There is no universal answer for partition size.

Consider monthly partitions:

```text
Jan
Feb
Mar
Apr
```

### Advantages

* Smaller partitions
* More granular retention
* Better for monthly date queries
* Easier month-by-month cleanup

### Disadvantages

* More partitions
* More maintenance

Now consider yearly partitions:

```text
2025
2026
2027
```

### Advantages

* Fewer partitions
* Simpler maintenance

### Disadvantages

* Larger partitions
* Less granular retention
* Large yearly historical datasets

The correct strategy depends on:

```text
Data Volume
+
Growth Rate
+
Query Pattern
+
Retention Policy
```

For CitiCore's historical statement and outbox workloads, time-based RANGE partitioning is the core design concept. The final partition interval should be chosen based on actual data growth.

---

# 10.26 Partitioning and Database Replication

Partitioning and replication solve different problems.

Replication:

```text
Primary
   |
   v
Replica
```

solves:

```text
Read Scalability
+
Availability
```

Partitioning:

```text
Large Table
   |
   v
Partitions
```

solves:

```text
Large Data Management
+
Time-based Query Optimization
+
Retention Operations
```

These concepts can work together.

For example:

```text
Primary Database
      |
      v
Partitioned account_statements
      |
      v
Replicated
      |
      v
Replica Database
      |
      v
Read Queries
```

However:

> **Replication does not replace partitioning, and partitioning does not replace replication.**

They solve different architectural problems.

---

# 10.27 Partitioning and Read/Write Separation

CitiCore also uses Primary/Replica routing.

The relationship is:

```text
Primary / Replica
       |
       v
Which Database?
```

Partitioning answers:

```text
Partitioning
       |
       v
Which Data Segment?
```

A request can therefore involve both decisions.

Example:

```text
Request Statement History
          |
          v
Can use eventual consistency?
          |
          v
Replica
          |
          v
Date Range
          |
          v
Partition Pruning
          |
          v
Relevant Partition
```

This can be represented as:

```text
Client
   |
   v
Account Service
   |
   v
Datasource Routing
   |
   v
Replica
   |
   v
Partition Pruning
   |
   v
Relevant Statement Partition
```

---

# 10.28 Partitioning and Transactional Outbox

The Transactional Outbox pattern and partitioning also solve different problems.

Transactional Outbox:

```text
Reliable Event Creation
+
Database Transaction
+
Kafka Publishing
```

Partitioning:

```text
Manage Large Historical Outbox Data
```

Together:

```text
Business Transaction
       |
       v
Business Data + Outbox Event
       |
       v
Outbox Table
       |
       v
Time-based Partition
       |
       v
Outbox Publisher
       |
       v
Kafka
```

The partitioning strategy does not guarantee event delivery.

That responsibility belongs to the Transactional Outbox mechanism.

Partitioning helps manage the growing outbox table.

---

# 10.29 Real Problem Addressed

## Real Project Context

The partitioning design addresses the expected growth of historical CitiCore tables.

The two major concerns are:

```text
account_statements
```

and:

```text
account_outbox
```

Both tables can grow continuously.

Without a data management strategy:

```text
Historical Records
       |
       v
Table Growth
       |
       v
Large Table
       |
       v
Harder Queries
+
Harder Cleanup
+
Harder Retention Management
```

The CitiCore approach is to use time-based RANGE partitioning where it is appropriate.

The partitioning strategy is particularly useful because both datasets have a natural time dimension.

---

# 10.30 Important Production Considerations

## Partition Monitoring

Monitor:

```text
Partition Count
Partition Size
Disk Usage
Future Partition Availability
Query Performance
```

---

## Future Partition Creation

A production system should avoid reaching a date range without a matching partition.

Conceptually:

```text
Today
   |
   v
Check Future Partitions
   |
   v
Create Next Partition Before Needed
```

This can be automated.

---

## Retention Policy

Before dropping partitions, define:

```text
How long is statement data retained?
How long are outbox events retained?
Are records archived?
Are backups available?
Are there regulatory requirements?
```

For a banking application, retention requirements may be particularly important.

---

## Backup

Before removing historical partitions:

```text
Partition
   |
   v
Backup / Archive Decision
   |
   v
Retention Validation
   |
   v
Drop Partition
```

Never treat partition dropping as a substitute for a proper data retention strategy.

---

## Query Monitoring

Partition pruning should be verified rather than assumed.

Useful tools include query analysis and execution plans.

The goal is to confirm:

```text
Expected Partition
       |
       v
Actually Used
```

---

# 10.31 Trade-offs

## Advantages

### Better Management of Large Historical Tables

```text
Large Table
    |
    v
Smaller Logical Segments
```

---

### Partition Pruning

Queries that include the partition key can search fewer partitions.

---

### Easier Data Retention

Old partitions can be managed as logical data units.

---

### Better Operational Organization

Time-based partitions make historical data easier to understand and maintain.

---

### Efficient Historical Data Management

Partitioning is particularly suitable for:

```text
Statements
Events
Logs
Historical Records
```

---

## Disadvantages

### Increased Schema Complexity

Partitioning introduces additional database design decisions.

---

### Operational Maintenance

Future partitions must be created.

Old partitions must be managed.

---

### Query Dependency

Queries that do not align with the partition key may not benefit significantly.

---

### More Careful Key Design

Primary and unique key constraints must comply with MySQL partitioning requirements.

---

### Not Suitable for Every Table

Using partitioning unnecessarily can make a system harder to maintain.

---

# 10.32 Alternatives to Partitioning

## Archival Tables

Old data can be moved to separate tables.

Example:

```text
account_statements
        |
        v
Current Data
```

and:

```text
account_statements_archive
        |
        v
Old Data
```

### Advantage

Simple conceptual separation.

### Disadvantage

Application queries may need to check multiple tables.

---

## Separate Database

Historical data can be moved to another database.

```text
Operational Database
        |
        +---- Current Data
        |
        +---- Historical Database
```

This may be useful for very large systems but increases architectural complexity.

---

## Sharding

Data can be distributed across multiple database instances.

Example:

```text
Customer Range A
       |
       v
Database 1

Customer Range B
       |
       v
Database 2
```

Sharding solves a different scaling problem and is significantly more complex than table partitioning.

For CitiCore's identified historical tables, time-based RANGE partitioning is the more direct solution.

---

# 10.33 How I Would Explain This in an Interview

> "In CitiCore, I used MySQL RANGE partitioning for tables that are expected to grow continuously and are naturally queried by time. The main candidates were account statements and the transactional outbox table.
>
> The idea was to divide large historical datasets into time-based partitions so that queries using date ranges could benefit from partition pruning. It also simplifies retention management because old data can be managed at the partition level instead of performing large row-by-row deletion operations.
>
> Partitioning does not replace indexes or replication. Replication handles read scaling and availability, while partitioning helps manage large datasets. We therefore use them for different concerns in the overall database architecture."

---

# 10.34 30-Second Interview Answer

> "In CitiCore, I used MySQL RANGE partitioning for large time-based tables such as account statements and the outbox table. The data is organized by date ranges, allowing MySQL to use partition pruning when queries include the partition key. This helps manage large historical datasets and makes retention operations easier. I would still use proper indexes because partitioning and indexing solve different problems."

---

# 10.35 Interview Follow-up Questions

## Q1. What is MySQL partitioning?

**Answer:**

MySQL partitioning divides one logical table into multiple physical partitions based on a partitioning rule. The application still queries the table normally, while MySQL manages where rows are stored.

---

## Q2. Why did you use RANGE partitioning?

**Answer:**

The CitiCore statement and outbox data have a natural time dimension. RANGE partitioning works well for date-based data and supports efficient time-range queries and retention management.

---

## Q3. What is partition pruning?

**Answer:**

Partition pruning is when MySQL identifies which partitions can contain the requested data and avoids scanning unrelated partitions.

---

## Q4. Does partitioning make every query faster?

**Answer:**

No. Queries benefit most when they include conditions related to the partition key. Queries that do not align with the partition key may need to access multiple partitions.

---

## Q5. Does partitioning replace indexing?

**Answer:**

No. Partitioning reduces the amount of table data that may need to be searched, while indexes help efficiently locate rows within the relevant partition.

---

## Q6. Why partition the account statements table?

**Answer:**

Account statements are historical and continuously growing. They are also commonly queried using date ranges, which makes time-based RANGE partitioning a suitable design.

---

## Q7. Why partition the outbox table?

**Answer:**

The outbox can continuously accumulate events. Time-based partitioning helps manage historical event data and supports retention and cleanup operations.

---

## Q8. What happens if you do not create future partitions?

**Answer:**

New rows may fail to be inserted if they do not match an available partition range, depending on the partition definition. Therefore future partition creation should be part of operational maintenance.

---

## Q9. What are the disadvantages of partitioning?

**Answer:**

It increases schema and operational complexity, requires maintenance, and does not improve every query. The partition strategy must match actual data and query patterns.

---

## Q10. How is partitioning different from sharding?

**Answer:**

Partitioning divides data inside a logical table according to a database partitioning strategy. Sharding distributes data across separate database instances. Sharding is generally a more complex distributed architecture.

---

# 10.36 Key Takeaways

```text
Large Historical Data
        |
        v
Partitioning
        |
        v
Time-based Partitions
        |
        +----------------------+
        |                      |
        v                      v
Efficient Queries       Easier Retention
```

CitiCore candidates:

```text
account_statements
        |
        v
Time-based RANGE Partitioning
```

and:

```text
account_outbox
        |
        v
Time-based RANGE Partitioning
```

Query flow:

```text
Query
   |
   v
Partition Key Condition
   |
   v
Partition Pruning
   |
   v
Relevant Partition
   |
   v
Index Lookup
   |
   v
Result
```

Most important principle:

> **Partitioning should be used to solve a real data-growth and data-management problem. It is not a replacement for indexing, replication, or proper database design.**

---

# 10.37 Topic Summary

CitiCore uses MySQL partitioning as part of its strategy for managing large, time-oriented historical data.

The main design focus is:

```text
Historical Data
      +
Continuous Growth
      +
Time-based Queries
      +
Retention Requirements
```

The identified tables include:

```text
account_statements
```

and:

```text
account_outbox
```

The core approach is:

```text
RANGE Partitioning
        |
        v
Time-based Data Segmentation
```

The expected benefits are:

```text
Partition Pruning
+
Historical Data Management
+
Simpler Retention Operations
+
Better Organization of Large Tables
```

The most important technical principle is:

> **Choose the partition key based on actual data lifecycle and query patterns. Partitioning provides the greatest value when application queries align with the partitioning strategy.**

---
# Topic 11 - Database Security

# 11.1 Overview

Database security in CitiCore is not limited to protecting a database username and password.

A banking application stores sensitive information such as:

* User information
* Account details
* Transaction records
* KYC-related data
* Authentication-related data
* Financial history
* Internal system events

Therefore, database security must protect multiple layers:

```text
Application
     |
     v
Database Credentials
     |
     v
Network Access
     |
     v
Encrypted Connection
     |
     v
Database Permissions
     |
     v
Sensitive Data
```

The main database security concerns for CitiCore are:

* Who can connect to the database?
* Which network locations can access it?
* Which database operations can each user perform?
* Are credentials stored securely?
* Is communication between the application and database encrypted?
* How are database credentials rotated?
* How are read-only and write operations separated?
* How does the application verify the database server identity?

The major concepts covered in this topic are:

1. Database users
2. Least privilege
3. Read-only users
4. TLS/SSL
5. AWS RDS CA certificates
6. Java truststores
7. `VERIFY_IDENTITY`
8. AWS Secrets Manager
9. Credential management

---

# 11.2 Security Architecture

A simplified CitiCore production database connection looks like:

```text
Spring Boot Service
        |
        |  Database Credentials
        |  Retrieved Securely
        v
AWS Secrets Manager
        |
        v
Spring Boot Application
        |
        | TLS/SSL Encrypted Connection
        | Database Server Identity Verified
        v
AWS RDS MySQL
        |
        v
Database User Permissions
        |
        +----------------------+
        |                      |
        v                      v
Read Operations         Write Operations
```

Network security is also involved:

```text
Internet
    |
    X
    |
No Direct Database Access
```

Instead:

```text
ECS / Application
       |
       | Allowed Security Group Rule
       v
Private RDS Database
```

The complete security model therefore includes:

```text
Identity
+
Credentials
+
Network Security
+
Encryption
+
Authorization
```

---

# 11.3 Database Users

## 1. What Is It?

A database user is an identity used to authenticate with the database.

For example:

```text
Username: citicore_app
Password: ********
```

The database verifies:

```text
Who is connecting?
```

After authentication, the database checks:

```text
What is this user allowed to do?
```

These are two separate concerns.

```text
Authentication
       |
       v
Who are you?
```

```text
Authorization
       |
       v
What are you allowed to do?
```

---

# 11.4 Why Do We Need Different Database Users?

A common security mistake is to use one highly privileged database user everywhere.

For example:

```text
Application
      |
      v
Database Admin User
      |
      v
SELECT
INSERT
UPDATE
DELETE
CREATE
DROP
ALTER
```

This is dangerous.

If application credentials are compromised, an attacker may receive unnecessary permissions.

For example:

```text
Compromised Application Credential
              |
              v
Highly Privileged Database User
              |
              v
DROP DATABASE
```

The better approach is to provide only the permissions required for a specific responsibility.

This is called:

> **Principle of Least Privilege**

---

# 11.5 Principle of Least Privilege

## Concept

The principle of least privilege means:

> A user, service, or system should receive only the minimum permissions required to perform its job.

For example:

```text
Notification Service
```

does not necessarily need permission to:

```text
DROP TABLE
```

Similarly, a read-only reporting process should not have:

```text
INSERT
UPDATE
DELETE
```

permissions.

Instead:

```text
Read User
    |
    +---- SELECT
```

```text
Write User
    |
    +---- SELECT
    +---- INSERT
    +---- UPDATE
    +---- DELETE
```

Administrative users should be separated from application users.

---

# 11.6 CitiCore Context

## CitiCore Implementation

CitiCore's database architecture includes read/write separation between:

```text
Primary Database
```

and:

```text
Read Replica
```

This architecture naturally supports the idea of separate database access responsibilities.

Conceptually:

```text
Spring Boot Service
       |
       +----------------------+
       |                      |
       v                      v
Write Datasource        Read Datasource
       |                      |
       v                      v
Primary DB             Replica DB
```

A production-ready implementation should also ensure that database credentials follow the same separation of responsibility.

For example:

```text
Application Write Access
        |
        v
Primary Database
```

and:

```text
Application Read Access
        |
        v
Replica Database
```

The important principle is that read operations should not receive unnecessary administrative permissions.

---

# 11.7 Read-Only Database Users

## What Is a Read-Only User?

A read-only database user is allowed to retrieve data but cannot modify it.

Conceptually:

```text
Read-Only User
      |
      +---- SELECT  ✓
      |
      +---- INSERT  ✗
      |
      +---- UPDATE  ✗
      |
      +---- DELETE  ✗
```

Example MySQL permission:

```sql
GRANT SELECT ON citicore_db.* TO 'citicore_readonly'@'%';
```

This is an example of the concept. The exact user and host restrictions should be defined according to the production network architecture.

---

# 11.8 Why Use Read-Only Users?

Read-only credentials provide an additional security layer.

Suppose a reporting service only needs:

```text
Transaction History
Account Statements
Customer Information
```

It should not require permission to modify data.

If that service is compromised:

```text
Compromised Read Credential
         |
         v
Can Read Data
         |
         X
Cannot Modify Data
```

This limits the possible impact.

This approach is especially useful when:

* Applications have different responsibilities
* Reporting workloads exist
* Read replicas are used
* Analytics services access production data
* Internal tools need limited access

---

# 11.9 Application Database User

A normal application database user may require:

```text
SELECT
INSERT
UPDATE
DELETE
```

depending on the application's responsibility.

For example:

```text
Account Service
      |
      v
Accounts Database Operations
```

The permissions should be restricted to:

```text
Required Database
+
Required Tables
+
Required Operations
```

The application user should generally not receive unnecessary administrative permissions such as:

```text
DROP DATABASE
CREATE USER
GRANT
SUPER
```

unless there is a specific operational requirement.

---

# 11.10 Database Security Layers

A secure CitiCore database connection should be viewed as multiple layers.

```text
Layer 1
Identity
```

Who is connecting?

```text
Layer 2
Credentials
```

Is the username and password valid?

```text
Layer 3
Network
```

Is the connection coming from an allowed network location?

```text
Layer 4
TLS
```

Is communication encrypted?

```text
Layer 5
Server Verification
```

Is the application connected to the correct database server?

```text
Layer 6
Authorization
```

What database operations are allowed?

Together:

```text
Application
     |
     v
Authentication
     |
     v
Authorization
     |
     v
Network Control
     |
     v
TLS Encryption
     |
     v
Database
```

No single layer should be considered sufficient by itself.

---

# 11.11 TLS/SSL for Database Connections

## 1. What Is TLS?

TLS, previously commonly referred to as SSL, encrypts communication between two systems.

For CitiCore:

```text
Spring Boot Application
          |
          | Plain Text
          X
          |
          | TLS Encrypted
          v
AWS RDS MySQL
```

Without encryption, database communication may expose sensitive information if the network is compromised.

The encrypted connection protects data in transit.

This can include:

* SQL queries
* Query results
* User data
* Transaction information
* Database credentials during authentication

---

# 11.12 Why Database TLS Is Important

Consider a database request:

```text
Application
      |
      v
SELECT account details
      |
      v
Database
```

Without TLS:

```text
Application
      |
      | Unencrypted Network Traffic
      v
Database
```

With TLS:

```text
Application
      |
      |===========================|
      |   Encrypted TLS Traffic   |
      |===========================|
      v
Database
```

For a banking platform, sensitive information should not depend only on network isolation.

Defense in depth is preferred.

That means:

```text
Private Network
+
Security Groups
+
TLS Encryption
```

instead of relying on only one security mechanism.

---

# 11.13 CitiCore Context: AWS RDS TLS

## CitiCore Implementation

The CitiCore database deployment includes AWS RDS.

AWS provides certificate authority information used for secure database connections.

Conceptually:

```text
Spring Boot Application
        |
        | TLS Connection
        v
AWS RDS Endpoint
        |
        v
RDS Certificate
        |
        v
AWS Certificate Authority
```

The Java application must trust the certificate authority used to validate the RDS database certificate.

This requires:

```text
AWS RDS CA Certificate
        |
        v
Java Truststore
        |
        v
JDBC Connection
```

---

# 11.14 AWS RDS Certificate Authority

## What Is a Certificate Authority?

A Certificate Authority, or CA, is responsible for issuing and signing certificates.

The application needs to determine:

```text
Can I trust this database certificate?
```

The answer is based on trusted certificate authorities.

Conceptually:

```text
Application
     |
     v
Database Certificate
     |
     v
Issued / Signed By
     |
     v
Trusted Certificate Authority?
     |
     +--------+
     |        |
    Yes       No
     |        |
     v        v
Connect     Reject
```

For AWS RDS, AWS provides CA certificate bundles and certificate information that applications can use to establish trust.

---

# 11.15 Java Truststore

## What Is a Truststore?

A Java truststore is a repository containing certificates that Java applications trust.

Conceptually:

```text
Java Application
       |
       v
Truststore
       |
       +--------------------+
       |                    |
       v                    v
Trusted CA 1          Trusted CA 2
```

When connecting to an RDS database over TLS:

```text
Spring Boot
     |
     v
Receive Database Certificate
     |
     v
Check Java Truststore
     |
     v
Trusted CA Available?
     |
     +--------+
     |        |
    Yes       No
     |        |
     v        v
Connect     TLS Failure
```

---

# 11.16 Why CitiCore Needs a Truststore

The application should not blindly accept any database certificate.

Otherwise, a malicious or incorrectly configured server could potentially be trusted.

The truststore allows the application to verify that the certificate presented by the database is issued by a trusted authority.

The flow is:

```text
Spring Boot Application
        |
        v
Connect to RDS
        |
        v
RDS Presents Certificate
        |
        v
Java Checks Truststore
        |
        v
Trusted CA?
        |
        +----------+
        |          |
       Yes         No
        |          |
        v          v
Continue       Reject Connection
```

---

# 11.17 Truststore Creation

A common approach involves importing the appropriate RDS CA certificate into a Java truststore.

Conceptually:

```text
AWS RDS CA Certificate
         |
         v
keytool
         |
         v
Java Truststore
```

A command may look similar to:

```bash
keytool -importcert \
  -alias aws-rds-ca \
  -file rds-ca.pem \
  -keystore truststore.jks
```

This is a representative example of the process.

The exact certificate file, alias, keystore format, and password must match the AWS RDS certificate configuration being used.

The truststore should be handled securely.

For example:

```text
truststore.jks
      |
      +---- Do not expose passwords in source code
      |
      +---- Use secure deployment configuration
      |
      +---- Manage certificate updates
```

---

# 11.18 SSL Modes

Database drivers can support different SSL verification modes.

One important production concept is:

```text
VERIFY_IDENTITY
```

The exact JDBC configuration depends on the MySQL connector and connection configuration.

The key security principle is:

> The application should not only encrypt the connection. It should also verify the identity of the database server.

---

# 11.19 VERIFY_IDENTITY

## What Is VERIFY_IDENTITY?

`VERIFY_IDENTITY` provides stronger certificate validation than simply enabling encryption.

Conceptually:

```text
Application
     |
     v
Database Server
     |
     v
Certificate Presented
     |
     +--------------------------+
     |                          |
     v                          v
Certificate Trusted?      Hostname Matches?
     |                          |
     +------------+-------------+
                  |
                  v
            Secure Connection
```

The application verifies:

1. The certificate chain is trusted.
2. The server identity matches the expected host.

This helps protect against connecting to an unexpected or impersonated server.

---

# 11.20 Why Encryption Alone Is Not Enough

Consider this scenario:

```text
Application
     |
     v
Encrypted Connection
     |
     v
Unknown Server
```

The connection may be encrypted, but encryption alone does not necessarily prove that the application connected to the intended server.

A stronger security model is:

```text
Encrypted Connection
       +
Trusted Certificate
       +
Server Identity Verification
```

This provides better protection.

---

# 11.21 JDBC Connection Security

A conceptual secure connection configuration includes:

```text
JDBC URL
   |
   +---- TLS Enabled
   |
   +---- Certificate Verification
   |
   +---- Server Identity Verification
```

For example, depending on the MySQL driver version and configuration:

```properties
useSSL=true
```

and certificate verification settings may be configured.

However, production configuration should be based on the exact:

* MySQL Connector/J version
* AWS RDS certificate configuration
* Deployment environment
* Java version

The important CitiCore design principle is:

> Database connections should use encryption and certificate validation rather than disabling certificate verification for convenience.

---

# 11.22 Development vs Production TLS

Development environments are different from production environments.

Local development might use:

```text
Docker MySQL
      |
      v
Local Network
```

Production uses:

```text
Spring Boot Service
       |
       | TLS
       v
AWS RDS
```

The security requirements are therefore different.

A local development environment may simplify configuration.

Production should enforce stronger controls:

```text
Private Database
+
Restricted Network Access
+
TLS
+
Certificate Validation
+
Secure Credentials
```

A common mistake is copying relaxed local security settings into production.

CitiCore documentation should clearly distinguish:

```text
Local Development Configuration
```

from:

```text
Production Security Configuration
```

---

# 11.23 AWS Secrets Manager

## 1. What Is It?

AWS Secrets Manager is a service used to securely store and manage secrets.

Examples include:

```text
Database Username
Database Password
API Keys
Tokens
Certificates
```

Instead of storing credentials directly in source code:

```java
String password = "my-database-password";
```

the application retrieves the secret securely.

Conceptually:

```text
Spring Boot Application
        |
        v
AWS Secrets Manager
        |
        v
Database Credentials
        |
        v
AWS RDS
```

---

# 11.24 Why Secrets Manager Is Needed

Hardcoding credentials creates multiple risks.

Example:

```text
Source Code
    |
    +---- Database Password
```

Problems include:

* Credentials can be committed to Git
* Credentials can be exposed in logs
* Password rotation becomes difficult
* Different environments require different credentials
* Developers may accidentally share secrets

A better architecture is:

```text
Application Code
       |
       X
No Hardcoded Secret
```

Instead:

```text
Application
      |
      v
Secure Secret Provider
      |
      v
Credential
```

---

# 11.25 CitiCore Context: Credential Management

## CitiCore Implementation

The CitiCore AWS architecture includes the use of secure AWS infrastructure components for deployment.

For production database access, secrets should not be stored directly inside:

```text
application.yml
```

or:

```text
Dockerfile
```

or:

```text
Jenkinsfile
```

Instead:

```text
AWS Secrets Manager
        |
        v
ECS Task / Application
        |
        v
Database Credentials
```

This provides better separation between:

```text
Application Configuration
```

and:

```text
Sensitive Credentials
```

---

# 11.26 Secrets Flow

The production flow can be represented as:

```text
Developer
    |
    v
Creates Application
    |
    v
No Password in Source Code
    |
    v
Deploy Application
    |
    v
ECS Task Starts
    |
    v
IAM Permission
    |
    v
AWS Secrets Manager
    |
    v
Retrieve Database Secret
    |
    v
Application Connects to RDS
```

The application should receive permission to access only the secrets it needs.

This follows the same least-privilege principle used for database users.

---

# 11.27 IAM and Secrets Access

AWS IAM controls who or what can access a secret.

For CitiCore:

```text
ECS Task
    |
    v
IAM Task Role
    |
    v
Permission Check
    |
    v
AWS Secrets Manager
```

The task role should be limited to:

```text
Required Secret
```

rather than:

```text
All Secrets
```

Conceptually:

```text
CitiCore Account Service
          |
          v
IAM Role
          |
          v
Can Access:
account-service-db-secret

Cannot Access:
other-unrelated-secret
```

This limits the impact of a compromised service.

---

# 11.28 Credential Rotation

Passwords should not be permanent.

A secure production system should support credential rotation.

Conceptually:

```text
Old Credential
      |
      v
Rotate
      |
      v
New Credential
      |
      v
Application Uses Updated Credential
```

AWS Secrets Manager can support secret rotation workflows.

However, rotation must be carefully integrated with:

```text
Application Connection Pool
+
Deployment Configuration
+
Database Users
```

If the database password changes but the application continues using an old password:

```text
Application
      |
      v
Old Password
      |
      v
Authentication Failure
```

Therefore credential rotation must be treated as an operational process rather than simply changing a password.

---

# 11.29 HikariCP and Credential Rotation

CitiCore uses Spring Boot database connectivity, where HikariCP is relevant to connection pooling.

Conceptually:

```text
Application
     |
     v
HikariCP
     |
     +---- Connection 1
     +---- Connection 2
     +---- Connection 3
     |
     v
Database
```

When credentials rotate, existing connections may continue to exist until they are refreshed or recreated.

Production credential rotation therefore needs to consider:

```text
Secret Rotation
       |
       v
Application Configuration Refresh
       |
       v
Connection Pool Refresh
       |
       v
New Database Connections
```

The exact implementation depends on how credentials are injected into the application and how configuration refresh is handled.

---

# 11.30 Do Not Store Secrets in application.yml

A configuration file such as:

```yaml
spring:
  datasource:
    username: admin
    password: my-password
```

may be acceptable only as a local development example when the file is excluded from version control and managed appropriately.

For production, the preferred architecture is:

```text
application.yml
       |
       +---- Database Endpoint
       +---- Non-sensitive Configuration
```

Sensitive values should come from:

```text
AWS Secrets Manager
```

or another approved secure secret management mechanism.

---

# 11.31 Environment Variables and Secrets

Environment variables are commonly used in containerized applications.

Example:

```text
DB_USERNAME
DB_PASSWORD
```

This can be useful, but environment variables also require careful handling.

Risks include:

* Accidental logging
* Exposure through debugging tools
* Exposure through container inspection
* Misconfigured CI/CD pipelines

A more secure container deployment architecture is:

```text
AWS Secrets Manager
        |
        v
ECS Secret Injection
        |
        v
Container Environment
        |
        v
Spring Boot Application
```

The secret value should never be printed in logs.

---

# 11.32 Secrets and Jenkins

Jenkins pipelines may require access to infrastructure credentials.

For example:

```text
Git
AWS
Docker
ECR
```

Database credentials should not be placed directly inside:

```groovy
Jenkinsfile
```

For example, avoid:

```groovy
environment {
    DB_PASSWORD = "secret-password"
}
```

Instead, use secure credential management.

Conceptually:

```text
Jenkins Pipeline
       |
       v
Credential Store / Secure Secret Provider
       |
       v
Temporary Secure Access
```

The same principle applies across:

```text
Source Code
Docker
Jenkins
AWS
Spring Boot
```

---

# 11.33 Network Security

Database credentials alone are not sufficient.

The database should also be protected by network controls.

For CitiCore AWS architecture:

```text
Internet
    |
    X
    |
RDS Not Directly Accessible
```

Instead:

```text
ECS Service
    |
    | Allowed Database Port
    v
Security Group
    |
    v
RDS Security Group
    |
    v
RDS MySQL
```

The database should typically be located in a private subnet.

Conceptually:

```text
VPC
 |
 +------------------------+
 |                        |
 | Public Subnet          |
 |     |                  |
 |    ALB                 |
 |     |                  |
 | Private Subnet         |
 |     |                  |
 |    ECS                 |
 |     |                  |
 | Database Subnet        |
 |     |                  |
 |    RDS                 |
 |                        |
 +------------------------+
```

This reduces unnecessary exposure.

---

# 11.34 Security Groups and Database Access

A security group should allow database traffic only from trusted sources.

Conceptually:

```text
RDS Security Group
       |
       +---- Allow MySQL
       |
       +---- Source:
              ECS Service Security Group
```

Avoid:

```text
MySQL Port
3306
Source
0.0.0.0/0
```

because that would unnecessarily expose the database.

The better model is:

```text
Only Trusted Application
          |
          v
Can Reach Database
```

---

# 11.35 Database Access Flow

A secure CitiCore database connection can be summarized as:

```text
Client Request
      |
      v
API Gateway
      |
      v
Spring Boot Service
      |
      +--------------------------+
      |                          |
      v                          v
IAM Role                  Database Configuration
      |                          |
      v                          v
Secrets Manager          RDS Endpoint
      |                          |
      +------------+-------------+
                   |
                   v
            HikariCP Pool
                   |
                   v
         TLS Encrypted Connection
                   |
                   v
            RDS Security Group
                   |
                   v
              AWS RDS
                   |
                   v
          Database Authorization
```

Each layer protects a different part of the connection.

---

# 11.36 Database Authorization Flow

After the application successfully connects:

```text
Application
     |
     v
Database Username
     |
     v
Authentication
     |
     v
Permission Check
     |
     +------------------------+
     |                        |
     v                        v
Allowed                  Not Allowed
     |                        |
     v                        v
Execute Query             Reject Query
```

For example:

```text
Read-Only User
      |
      v
SELECT
      |
      ✓
```

but:

```text
Read-Only User
      |
      v
DELETE
      |
      ✗
```

This is why database authorization remains important even when application-level security exists.

---

# 11.37 Defense in Depth

CitiCore should not depend on one security control.

Instead:

```text
Layer 1
Private Network
```

```text
Layer 2
Security Groups
```

```text
Layer 3
Database Credentials
```

```text
Layer 4
Least Privilege
```

```text
Layer 5
TLS Encryption
```

```text
Layer 6
Certificate Verification
```

```text
Layer 7
Secrets Management
```

Together:

```text
+-----------------------------+
| Application Security        |
+-----------------------------+
              |
+-----------------------------+
| IAM Permissions             |
+-----------------------------+
              |
+-----------------------------+
| Secrets Manager             |
+-----------------------------+
              |
+-----------------------------+
| TLS / Certificate Validation|
+-----------------------------+
              |
+-----------------------------+
| Security Groups             |
+-----------------------------+
              |
+-----------------------------+
| Database Permissions        |
+-----------------------------+
```

If one control fails, other controls still provide protection.

---

# 11.38 Common Database Security Mistakes

## Mistake 1 — Hardcoding Passwords

```java
String password = "database-password";
```

### Problem

Credentials can be exposed through source control.

### Better Approach

```text
Secrets Manager
```

---

## Mistake 2 — Using Root Credentials

```text
Application
     |
     v
Database Root User
```

### Problem

The application receives excessive permissions.

### Better Approach

```text
Dedicated Application User
+
Least Privilege
```

---

## Mistake 3 — Publicly Accessible Database

```text
Internet
    |
    v
RDS
```

### Problem

The attack surface is unnecessarily large.

### Better Approach

```text
Private RDS
    |
    v
Application Security Group Only
```

---

## Mistake 4 — Disabling Certificate Verification

A configuration that accepts any certificate may appear convenient during development.

### Problem

The application may not properly verify the database server identity.

### Better Approach

```text
TLS
+
Trusted CA
+
Identity Verification
```

---

## Mistake 5 — Logging Secrets

Never log:

```text
Passwords
Tokens
Secret Values
Database Connection Secrets
```

Even debugging logs can become a security problem.

---

# 11.39 Real Project Context

The CitiCore notes include database deployment and security work involving AWS RDS and secure database connectivity.

Important implementation areas include:

* RDS database access
* Primary/Replica database architecture
* Application datasource configuration
* AWS security configuration
* TLS/SSL database connectivity
* AWS RDS CA certificates
* Java truststore configuration
* Database credential management

The authoritative design principle is:

> CitiCore database connectivity should be protected using network isolation, restricted access, encrypted communication, certificate verification, and secure credential management.

This topic should not be confused with the earlier **Database Architecture** topic.

Database Architecture explains:

```text
How data is stored and accessed.
```

Database Security explains:

```text
How database access and communication are protected.
```

---

# 11.40 Production Considerations

A real production banking system would require additional security controls.

---

## Credential Rotation

Database credentials should support periodic rotation.

---

## Audit Logging

Important database activities should be monitored.

Examples:

```text
Failed Login Attempts
Permission Changes
Administrative Operations
Unexpected Connections
```

---

## Encryption at Rest

TLS protects:

```text
Data in Transit
```

Production databases should also consider encryption for:

```text
Data at Rest
```

For AWS RDS, this involves appropriate encryption configuration using AWS-managed or customer-managed keys depending on requirements.

---

## Separate Environment Credentials

Never reuse the same credentials across:

```text
Development
Testing
Staging
Production
```

Each environment should have separate secrets and access policies.

---

## Secret Access Monitoring

Monitor:

```text
Who accessed secrets?
When were secrets accessed?
Which application role accessed them?
```

AWS services such as CloudTrail can support auditing of AWS API activity.

---

## Certificate Rotation

RDS certificate authorities and server certificates can change over time.

Applications must ensure that:

```text
Truststore
```

and:

```text
Certificate Configuration
```

remain compatible with supported AWS RDS certificate rotations.

---

# 11.41 Trade-offs

## Dedicated Database Users

### Advantages

* Reduced security risk
* Better separation of responsibility
* Easier auditing

### Disadvantages

* More users and permissions to manage

---

## Read-Only Users

### Advantages

* Limits accidental modification
* Reduces impact of compromised credentials
* Supports read-specific workloads

### Disadvantages

* Requires separate credential management

---

## TLS

### Advantages

* Encrypts data in transit
* Protects sensitive database communication

### Disadvantages

* Certificate management
* Additional configuration complexity

---

## Certificate Identity Verification

### Advantages

* Stronger server validation
* Protection against incorrect server connections

### Disadvantages

* Requires proper CA and hostname configuration

---

## AWS Secrets Manager

### Advantages

* No hardcoded credentials
* Centralized secret management
* Supports rotation workflows

### Disadvantages

* Additional AWS service dependency
* IAM configuration is required
* Application deployment must handle secret retrieval correctly

---

# 11.42 How I Would Explain This in an Interview

> "For database security in CitiCore, I focused on multiple layers rather than relying only on a username and password. The database is protected through restricted network access, dedicated database users, least-privilege permissions, and TLS encryption.
>
> For AWS RDS connectivity, the Java application uses certificate trust configuration so that the application can validate the database certificate instead of blindly trusting the server. Sensitive credentials should be stored outside the application source code using AWS Secrets Manager and accessed through IAM permissions.
>
> Since CitiCore uses primary-replica architecture, database access responsibilities can also be separated between read and write workloads. The main principle was defense in depth: network security, IAM, secrets management, encryption, certificate validation, and database authorization all work together."

---

# 11.43 30-Second Interview Answer

> "In CitiCore, database security is based on defense in depth. We restrict database access through private networking and security groups, use dedicated database users with least-privilege permissions, and secure RDS connections using TLS and certificate verification. Database credentials should be managed through AWS Secrets Manager instead of being hardcoded in application configuration."

---

# 11.44 Interview Follow-up Questions

## Q1. What is the principle of least privilege?

**Answer:**

It means granting only the minimum permissions required for a user or service to perform its responsibility.

---

## Q2. Why should applications not use database root credentials?

**Answer:**

Root credentials provide excessive permissions. If the application or credentials are compromised, the attacker could perform destructive administrative operations.

---

## Q3. What is a read-only database user?

**Answer:**

A read-only user typically has permission to retrieve data but cannot modify it. This is useful for reporting, analytics, and read-specific workloads.

---

## Q4. Why use TLS between Spring Boot and RDS?

**Answer:**

TLS encrypts database communication and protects sensitive information while it travels between the application and the database.

---

## Q5. What is a Java truststore?

**Answer:**

A Java truststore contains certificates that the Java application trusts. During TLS communication, Java uses it to validate the certificate presented by the remote server.

---

## Q6. What is VERIFY_IDENTITY?

**Answer:**

It represents stronger certificate validation where the application verifies both certificate trust and the server's expected identity or hostname.

---

## Q7. Why use AWS Secrets Manager?

**Answer:**

It allows sensitive credentials such as database usernames and passwords to be stored outside application source code and accessed securely using AWS IAM permissions.

---

## Q8. Can environment variables contain secrets?

**Answer:**

They can, but they must be handled carefully because secrets may be exposed through logs, debugging, container inspection, or CI/CD misconfiguration. Managed secret injection is generally preferred in production environments.

---

## Q9. What is the difference between encryption in transit and encryption at rest?

**Answer:**

Encryption in transit protects data while it moves between systems, such as between Spring Boot and RDS. Encryption at rest protects stored data, such as database storage and backups.

---

## Q10. What happens if database credentials are rotated?

**Answer:**

The application must retrieve the updated credentials and establish new database connections. Connection pools such as HikariCP must also be considered because existing connections may continue using the old credentials until refreshed.

---

# 11.45 Key Takeaways

```text
Database Security
       |
       +-----------------------+
       |                       |
       v                       v
Authentication            Authorization
       |                       |
       v                       v
Who connects?           What can they do?
```

Secure connectivity:

```text
Spring Boot
     |
     | TLS
     v
Certificate Verification
     |
     v
Private Network
     |
     v
Security Group
     |
     v
AWS RDS
```

Credential management:

```text
Application Code
      |
      X
No Hardcoded Secrets
      |
      v
AWS Secrets Manager
      |
      v
IAM Controlled Access
```

Most important principle:

> **Database security should use multiple layers. Credentials alone are not enough. Secure networking, least privilege, TLS, certificate verification, and secret management must work together.**

---

# 11.46 Topic Summary

CitiCore database security focuses on protecting sensitive banking data throughout the entire database access lifecycle.

The architecture includes:

```text
Secure Identity
+
Least Privilege
+
Read-Only Access Where Appropriate
+
Private Networking
+
Security Groups
+
TLS Encryption
+
RDS Certificate Validation
+
Java Truststore
+
Server Identity Verification
+
AWS Secrets Manager
```

The complete security flow can be summarized as:

```text
Spring Boot Service
        |
        v
IAM Permission
        |
        v
AWS Secrets Manager
        |
        v
Database Credentials
        |
        v
HikariCP Connection Pool
        |
        v
TLS Encrypted Connection
        |
        v
Certificate Verification
        |
        v
RDS Security Group
        |
        v
AWS RDS
        |
        v
Database Permission Check
```

The most important lesson is:

> **A production database should never depend on a single security mechanism. Database protection should combine secure credentials, least privilege, network isolation, encryption, certificate verification, and controlled access.**

---
## Topic 12 - AWS Infrastructure

## 12.1 What Is AWS Infrastructure?

AWS infrastructure is the collection of cloud resources used to run, secure, network, deploy, and monitor an application.

For CitiCore, AWS infrastructure provides the environment where containerized microservices and supporting services can run.

At a high level, the infrastructure is responsible for:

* Networking services securely
* Running application containers
* Exposing APIs to clients
* Storing container images
* Hosting databases
* Managing permissions
* Managing secrets
* Collecting logs and metrics
* Supporting service-to-service communication

The simplified CitiCore AWS architecture is:

```text
Internet
    |
    v
Application Load Balancer
    |
    v
ECS / Fargate Services
    |
    +----------------------------+
    |            |               |
    v            v               v
Auth Service  Account Service  Transaction Service
    |            |               |
    +------------+---------------+
                 |
                 v
             AWS RDS
```

Supporting infrastructure includes:

```text
AWS
 |
 +-- VPC
 |
 +-- Subnets
 |
 +-- Route Tables
 |
 +-- Internet Gateway
 |
 +-- NAT Gateway
 |
 +-- Security Groups
 |
 +-- ECS / Fargate
 |
 +-- ECR
 |
 +-- ALB
 |
 +-- RDS
 |
 +-- Service Connect
 |
 +-- CloudWatch
 |
 +-- IAM
 |
 +-- Secrets Manager
```

The main goal is to provide a secure and manageable environment for CitiCore services.

---

# 12.2 Why Do We Need Cloud Infrastructure?

During local development, CitiCore services can run using:

```text
Developer Machine
       |
       +-- Docker
       |
       +-- MySQL
       |
       +-- Kafka
       |
       +-- Redis
       |
       +-- Spring Boot Services
```

However, a production-like environment needs more than simply running containers.

The application needs:

* Controlled network access
* Service availability
* Centralized deployment
* Secure database access
* Load balancing
* Container orchestration
* Logging
* IAM permissions
* Secret management

Therefore, the AWS architecture separates responsibilities.

```text
Application Layer
       |
       v
ECS / Fargate
```

```text
Networking Layer
       |
       v
VPC / Subnets / Route Tables / Security Groups
```

```text
Data Layer
       |
       v
RDS
```

```text
Traffic Layer
       |
       v
Application Load Balancer
```

```text
Container Registry
       |
       v
ECR
```

```text
Observability
       |
       v
CloudWatch
```

```text
Security
       |
       v
IAM / Secrets Manager / Security Groups
```

---

# 12.3 CitiCore AWS Architecture

## CitiCore Context

The CitiCore project uses a microservices architecture. This means multiple independently deployable services must communicate and operate inside the AWS environment.

A simplified architecture is:

```text
                         Internet
                            |
                            v
                  Application Load Balancer
                            |
                            v
                       API Gateway
                            |
        +-------------------+-------------------+
        |                   |                   |
        v                   v                   v
   Auth Service        User Service       Account Service
        |                   |                   |
        +-------------------+-------------------+
                            |
                            v
                     Internal Services
                            |
                            v
                         AWS RDS
```

Container images are managed separately:

```text
Developer / Jenkins
        |
        v
Build Docker Image
        |
        v
Amazon ECR
        |
        v
ECS Task
        |
        v
Run Container
```

Networking controls how these components communicate:

```text
VPC
 |
 +---------------------------------------------+
 |                                             |
 | Public Subnets                              |
 |      |                                      |
 |      +---- Application Load Balancer        |
 |                                             |
 | Private Subnets                             |
 |      |                                      |
 |      +---- ECS / Fargate Services           |
 |                                             |
 | Database Subnets                            |
 |      |                                      |
 |      +---- RDS                              |
 |                                             |
 +---------------------------------------------+
```

This separation is important because not every component should be directly exposed to the internet.

---

# 12.4 AWS Account Structure

## What Is an AWS Account?

An AWS account is a logical boundary for AWS resources, billing, permissions, and security controls.

Resources such as:

* VPC
* ECS
* RDS
* ECR
* IAM roles

are created inside an AWS account.

For a simple project environment:

```text
AWS Account
     |
     +-- CitiCore Infrastructure
```

A more mature production organization may separate environments:

```text
Organization
     |
     +--------------------------+
     |            |             |
     v            v             v
Development    Staging      Production
AWS Account    AWS Account  AWS Account
```

The main advantage is isolation.

For example, development resources should not accidentally affect production resources.

---

## CitiCore Context

For CitiCore, the important concept is understanding that AWS resources are organized under an AWS account and deployed in a selected AWS Region.

Example:

```text
AWS Account
      |
      v
AWS Region
      |
      v
VPC
      |
      v
CitiCore Resources
```

The exact multi-account production structure should not be claimed as implemented unless it exists in the final CitiCore deployment configuration.

Therefore:

### CitiCore Implementation

CitiCore infrastructure was deployed within AWS resources such as ECS, RDS, ECR, VPC, ALB, IAM, and CloudWatch.

### Production Improvement

A larger production environment should separate development, staging, and production environments more strictly, potentially using separate AWS accounts.

---

# 12.5 AWS Region and Availability Zones

## What Is an AWS Region?

An AWS Region is a geographic area containing AWS infrastructure.

For example:

```text
AWS
 |
 +-- Region
       |
       +-- Availability Zone A
       |
       +-- Availability Zone B
       |
       +-- Availability Zone C
```

Each Availability Zone contains separate infrastructure.

This supports better availability.

A highly available architecture may distribute resources across multiple Availability Zones.

Example:

```text
                  VPC
                   |
        +----------+----------+
        |                     |
        v                     v
Availability Zone A    Availability Zone B
        |                     |
        v                     v
   ECS Tasks              ECS Tasks
```

An Application Load Balancer can distribute traffic between healthy targets.

---

## CitiCore Context

For CitiCore, the important production design principle is:

> Application containers and infrastructure should not unnecessarily depend on a single Availability Zone.

The actual deployment should be described according to the final implemented AWS configuration.

Where multi-AZ deployment was not fully implemented, it should be treated as a production improvement rather than claimed as an existing feature.

---

# 12.6 Virtual Private Cloud (VPC)

## 1. What Is It?

A VPC is a logically isolated virtual network inside AWS.

It provides control over:

* IP address ranges
* Subnets
* Routing
* Internet access
* Network boundaries

Think of it as a private network created inside AWS.

```text
AWS Cloud
    |
    v
+-----------------------------------+
|               VPC                 |
|                                   |
|  Your Private Cloud Network       |
|                                   |
+-----------------------------------+
```

Inside the VPC, CitiCore resources communicate using controlled network paths.

---

# 12.7 Why Do We Need a VPC?

Without network isolation, all resources would be difficult to organize and secure.

A VPC allows CitiCore to define:

```text
Which resources are public?
```

```text
Which resources are private?
```

```text
Who can communicate with whom?
```

```text
How does traffic enter the system?
```

```text
How does private infrastructure access external services?
```

The architecture becomes:

```text
Internet
    |
    v
+----------------------------+
|            VPC             |
|                            |
| Public Resources           |
|                            |
| Private Resources          |
|                            |
| Database Resources         |
+----------------------------+
```

---

# 12.8 CitiCore VPC Architecture

A simplified CitiCore network design is:

```text
                         VPC
+--------------------------------------------------+
|                                                  |
|  Public Subnet                                   |
|                                                  |
|      Internet Gateway                            |
|             |                                    |
|             v                                    |
|      Application Load Balancer                   |
|                                                  |
|--------------------------------------------------|
|                                                  |
|  Private Subnet                                  |
|                                                  |
|      ECS / Fargate Services                      |
|                                                  |
|      +-- Auth Service                            |
|      +-- User Service                            |
|      +-- Account Service                         |
|      +-- Transaction Service                     |
|                                                  |
|--------------------------------------------------|
|                                                  |
|  Database Subnet                                 |
|                                                  |
|      AWS RDS                                     |
|                                                  |
+--------------------------------------------------+
```

The key idea is:

```text
Public Traffic
      |
      v
ALB
      |
      v
Private Application Services
      |
      v
Private Database
```

The database should not need direct internet exposure.

---

# 12.9 Public and Private Subnets

## What Is a Public Subnet?

A public subnet is a subnet whose route configuration allows traffic to reach the internet through an Internet Gateway.

Example:

```text
Public Subnet
      |
      v
Internet Gateway
      |
      v
Internet
```

Public-facing components may include:

```text
Application Load Balancer
```

The important point is that a resource being in a public subnet does not automatically mean every security rule is open.

Security Groups still control access.

---

## What Is a Private Subnet?

A private subnet does not directly expose resources to incoming internet traffic.

For CitiCore:

```text
Private Subnet
      |
      +-- ECS Services
      |
      +-- Internal Components
```

The architecture becomes:

```text
Internet
    |
    v
Public ALB
    |
    v
Private ECS Services
```

Clients do not directly connect to every microservice container.

---

# 12.10 Why Separate Public and Private Subnets?

The main reason is reducing exposure.

Consider this unsafe design:

```text
Internet
   |
   +---------------------------+
   |                           |
   v                           v
Auth Service              Account Service
```

Every service would potentially need public access.

A better design is:

```text
Internet
    |
    v
ALB
    |
    +-----------------------+
    |                       |
    v                       v
Private Service        Private Service
```

Benefits include:

* Reduced attack surface
* Centralized public entry point
* Better traffic control
* Easier security management

---

# 12.11 Route Tables

## What Is a Route Table?

A route table determines where network traffic should go.

Conceptually:

```text
Request
   |
   v
Route Table
   |
   +----------------------+
   |                      |
   v                      v
Internet Gateway      NAT Gateway
```

For example, a public subnet may have a route:

```text
Destination: Internet
Target: Internet Gateway
```

A private subnet may route outbound traffic through:

```text
NAT Gateway
```

The exact routing configuration depends on the infrastructure design.

---

# 12.12 CitiCore Routing Concept

The basic network flow is:

```text
Incoming Traffic

Internet
    |
    v
Internet Gateway
    |
    v
Public Route Table
    |
    v
Application Load Balancer
```

Internal traffic then flows to:

```text
ALB
 |
 v
Private ECS Services
```

For outbound access from private services:

```text
Private ECS Service
       |
       v
Private Route Table
       |
       v
NAT Gateway
       |
       v
Internet Gateway
       |
       v
Internet / AWS Public Services
```

Not every service necessarily requires unrestricted outbound internet access.

Production systems should restrict outbound access according to actual requirements.

---

# 12.13 Internet Gateway

## What Is It?

An Internet Gateway connects a VPC to the internet.

Conceptually:

```text
Internet
    |
    v
Internet Gateway
    |
    v
VPC
```

For CitiCore, the Internet Gateway supports public-facing infrastructure such as an Application Load Balancer.

The flow is:

```text
Client
   |
   v
Internet
   |
   v
Internet Gateway
   |
   v
Application Load Balancer
```

The Internet Gateway does not replace security controls.

Traffic still depends on:

* Route tables
* Security Groups
* Resource configuration

---

# 12.14 NAT Gateway

## What Is It?

A NAT Gateway allows resources in a private subnet to initiate outbound connections without allowing unsolicited inbound internet connections.

Example:

```text
Private ECS Service
       |
       v
NAT Gateway
       |
       v
Internet
```

The internet cannot directly initiate a connection to the private service through this path.

This is useful when application containers need outbound access.

For example:

```text
Private ECS Task
      |
      +---- Pull dependencies during some workflows
      |
      +---- Access external APIs
      |
      +---- Access permitted external services
```

The exact requirement depends on the application's architecture.

---

# 12.15 NAT Gateway Trade-off

### Advantages

* Private workloads can access external resources
* Private services do not require public IP addresses
* Better network isolation

### Disadvantages

* Additional infrastructure
* Additional cost
* Routing configuration complexity

For a production architecture, NAT usage should be evaluated based on actual network requirements.

---

# 12.16 Security Groups

## 1. What Is a Security Group?

A Security Group acts as a virtual firewall for AWS resources.

It controls:

```text
Inbound Traffic
```

and:

```text
Outbound Traffic
```

For example:

```text
ALB Security Group
```

might allow:

```text
Internet
    |
    +---- HTTPS
    |
    +---- HTTP
```

An ECS service security group might allow:

```text
ALB
   |
   v
Application Port
```

An RDS security group might allow:

```text
ECS Service
     |
     v
MySQL Port
```

---

# 12.17 CitiCore Security Group Architecture

The simplified CitiCore security model is:

```text
Internet
    |
    v
+-------------------+
|   ALB Security    |
|      Group        |
+-------------------+
    |
    | Allowed Application Traffic
    v
+-------------------+
|   ECS Security    |
|      Group        |
+-------------------+
    |
    | Allowed Database Traffic
    v
+-------------------+
|   RDS Security    |
|      Group        |
+-------------------+
```

This creates controlled communication paths.

The desired principle is:

```text
Internet
    |
    v
ALB
    |
    v
ECS
    |
    v
RDS
```

but not:

```text
Internet
    |
    v
RDS
```

---

# 12.18 Security Group Best Practice

Avoid rules such as:

```text
Port: 3306
Source: 0.0.0.0/0
```

This unnecessarily exposes the database.

Instead:

```text
RDS Security Group
      |
      +-- Allow Port 3306
              |
              v
      Source: ECS Security Group
```

Using a security group as the source is often better than allowing a broad IP range when communication occurs between AWS resources.

The same principle applies to internal service communication.

---

# 12.19 AWS RDS

## What Is RDS?

Amazon RDS is a managed relational database service.

For CitiCore, RDS is used for relational database workloads.

RDS manages many operational responsibilities such as:

* Database infrastructure
* Automated backups depending on configuration
* Monitoring integration
* Database instance management
* Managed database engine support

CitiCore services connect to the database using:

```text
Spring Boot
     |
     v
JDBC / JPA
     |
     v
AWS RDS
```

---

# 12.20 CitiCore RDS Architecture

The CitiCore database architecture includes concepts such as:

```text
Primary Database
```

and:

```text
Read Replica
```

The simplified architecture is:

```text
Account Service
      |
      +-------------------+
      |                   |
      v                   v
Primary DB            Read Replica
   Write                 Read
```

This supports read/write separation.

However, database architecture, replication, replica lag, datasource routing, and HikariCP were explained in detail in:

> **Topic 9 — Database Architecture**

This section focuses only on the AWS infrastructure role of RDS.

RDS is the managed database platform supporting CitiCore's relational data layer.

---

# 12.21 RDS Network Placement

A production database should generally not be directly accessible from the internet.

The architecture should resemble:

```text
Internet
    |
    X
    |
    X
    |
RDS
```

Instead:

```text
ECS Service
     |
     v
RDS Security Group
     |
     v
Private RDS
```

This means database access is restricted to trusted application resources.

---

# 12.22 Amazon ECR

## What Is ECR?

Amazon Elastic Container Registry is a managed container image registry.

CitiCore Docker images can be stored in ECR.

The deployment flow is:

```text
Spring Boot Application
        |
        v
Docker Build
        |
        v
Docker Image
        |
        v
Amazon ECR
        |
        v
ECS Pulls Image
        |
        v
Container Runs
```

ECR separates:

```text
Build Artifact Storage
```

from:

```text
Container Execution
```

---

# 12.23 Why ECR Is Needed

Without a container registry, ECS would not know where to retrieve application images.

The workflow is:

```text
Developer / Jenkins
       |
       v
Build Image
       |
       v
Push Image
       |
       v
Container Registry
       |
       v
Deployment Platform Pulls Image
```

For CitiCore:

```text
Jenkins / Local Build
          |
          v
         ECR
          |
          v
         ECS
```

---

# 12.24 ECS

## What Is Amazon ECS?

Amazon Elastic Container Service is a container orchestration service.

It manages how containers are:

* Defined
* Started
* Stopped
* Replaced
* Scaled
* Connected to infrastructure

For CitiCore:

```text
Docker Image
      |
      v
ECS Task Definition
      |
      v
ECS Task
      |
      v
Running Container
```

An ECS Service manages the desired number of running tasks.

---

# 12.25 CitiCore ECS Architecture

A simplified architecture is:

```text
ECS Cluster
     |
     +----------------------------------+
     |                                  |
     v                                  v
Auth Service                       Account Service
ECS Service                        ECS Service
     |                                  |
     v                                  v
Task Definition                    Task Definition
     |                                  |
     v                                  v
Container                           Container
```

Each microservice can have:

* Its own Docker image
* Its own task definition
* Its own ECS service
* Its own environment configuration
* Its own health checks

This supports independent deployment.

For example:

```text
Deploy Account Service
        |
        v
Account Service Updated
```

without necessarily redeploying:

```text
Auth Service
```

This is one of the advantages of a microservices architecture.

---

# 12.26 ECS Cluster

An ECS cluster is a logical grouping of ECS services and tasks.

Conceptually:

```text
CitiCore ECS Cluster
        |
        +--------------------------+
        |                          |
        v                          v
Account Service                User Service
        |                          |
        v                          v
ECS Task                     ECS Task
```

The cluster provides the environment where ECS schedules workloads.

---

# 12.27 ECS Service

An ECS Service maintains the desired number of running tasks.

For example:

```text
Desired Tasks: 2
```

If one task stops:

```text
Task 1  ✓ Running

Task 2  X Failed
```

ECS Service attempts to restore the desired state:

```text
Task 1  ✓ Running

Task 2  ✓ Replacement Task Started
```

This provides basic orchestration and recovery behavior.

---

# 12.28 ECS Task Definition

A task definition describes how a container should run.

It may include:

* Docker image
* CPU
* Memory
* Port mappings
* Environment variables
* Secrets
* Logging configuration
* IAM roles

Conceptually:

```text
Task Definition
      |
      +-- Docker Image
      |
      +-- CPU / Memory
      |
      +-- Environment
      |
      +-- Secrets
      |
      +-- Ports
      |
      +-- Logging
```

The task definition acts as the deployment configuration for the container.

---

# 12.29 AWS Fargate

## What Is Fargate?

Fargate is a serverless compute engine for containers.

Instead of managing EC2 instances:

```text
You Manage:
EC2 Servers
OS
Capacity
Container Runtime
```

with Fargate:

```text
AWS Manages:
Underlying Infrastructure
```

while the application team focuses on:

```text
Container
Task Definition
Service
Resources
```

For CitiCore:

```text
Docker Container
       |
       v
ECS
       |
       v
Fargate
       |
       v
AWS Runs Container
```

---

# 12.30 Why Fargate for CitiCore?

The main advantages include:

* No EC2 server management
* Simpler container deployment
* Integration with ECS
* Easier operational management

For a project focused on microservices deployment, Fargate allows the architecture to focus more on application containers and less on infrastructure server management.

---

# 12.31 Fargate Trade-offs

### Advantages

* No server administration
* Simplified container deployment
* AWS-managed infrastructure
* Good ECS integration

### Disadvantages

* Less low-level control than managing EC2 directly
* Cost may not be optimal for every workload
* Startup behavior and resource configuration must still be managed

The correct choice depends on workload requirements.

---

# 12.32 Application Load Balancer (ALB)

## What Is an ALB?

An Application Load Balancer receives HTTP/HTTPS traffic and routes requests to application targets.

For CitiCore:

```text
Client
   |
   v
Application Load Balancer
   |
   v
Application Target
```

The ALB acts as a controlled entry point for external traffic.

---

# 12.33 CitiCore ALB Architecture

The basic flow is:

```text
Client
   |
   v
Internet
   |
   v
Application Load Balancer
   |
   v
Target Group
   |
   v
ECS Service
   |
   v
Spring Boot Application
```

The ALB can distribute traffic across healthy targets.

For example:

```text
                    ALB
                     |
          +----------+----------+
          |                     |
          v                     v
       Task 1                Task 2
       Healthy               Healthy
```

This improves availability when multiple tasks are deployed.

---

# 12.34 ALB Health Checks

The ALB needs to know whether an application target is healthy.

A health check flow is:

```text
ALB
 |
 v
GET /health
 |
 +---------------------+
 |                     |
 v                     v
Healthy             Unhealthy
 |                     |
 v                     v
Receive Traffic     Stop Routing Traffic
```

For Spring Boot applications, health endpoints can be exposed using Spring Boot Actuator depending on the implementation.

A typical concept is:

```text
/actuator/health
```

The exact health-check endpoint should match the deployed CitiCore configuration.

---

# 12.35 Why Health Checks Matter

Suppose an ECS container is running but the application inside it has failed.

Example:

```text
Container
    |
    +-- Running
          |
          X
          |
    Application Unhealthy
```

A container being started does not always mean the application is ready.

Health checks allow the load balancer to detect unhealthy application targets.

```text
ALB
 |
 v
Health Check
 |
 v
Application Healthy?
 |
 +-------------+
 |             |
Yes            No
 |             |
 v             v
Route        Remove
Traffic      from Routing
```

---

# 12.36 Service-to-Service Communication

CitiCore contains multiple microservices.

Examples include:

* Auth Service
* User Service
* Account Service
* Transaction Service
* Notification Service

Internal communication should not always depend on public internet endpoints.

Conceptually:

```text
Account Service
       |
       v
Internal Service Network
       |
       v
Transaction Service
```

AWS Service Connect can help with internal service communication.

---

# 12.37 AWS Service Connect

## What Is It?

AWS Service Connect provides service discovery and connectivity capabilities for ECS services.

Conceptually:

```text
Service A
    |
    v
Service Connect
    |
    v
Service B
```

Instead of requiring external public addresses for internal communication:

```text
Service A
     |
     v
Internal Service Name
     |
     v
Service B
```

This supports cleaner internal communication inside the ECS environment.

---

# 12.38 CitiCore Context: Eureka vs Service Connect

Earlier in CitiCore, service discovery concepts were implemented using:

```text
Eureka
```

The AWS deployment environment can also use:

```text
AWS Service Connect
```

These solve related problems but operate differently.

Conceptually:

```text
Application-Level Discovery
          |
          v
        Eureka
```

versus:

```text
Infrastructure-Level Service Connectivity
          |
          v
AWS Service Connect
```

The detailed comparison was covered under:

> **Topic 4 — Service Discovery**

The important AWS infrastructure point is that Service Connect can support internal ECS service communication and reduce dependence on externally exposed service addresses.

---

# 12.39 CloudWatch

## What Is CloudWatch?

Amazon CloudWatch provides monitoring and observability capabilities for AWS resources and applications.

It can help collect:

* Logs
* Metrics
* Alarms

For CitiCore:

```text
ECS Container
      |
      v
Application Logs
      |
      v
CloudWatch Logs
```

Infrastructure metrics can also be monitored.

---

# 12.40 CitiCore Logging Architecture

A simplified logging flow is:

```text
Spring Boot Application
        |
        v
Container stdout / stderr
        |
        v
ECS Logging Configuration
        |
        v
CloudWatch Logs
```

This allows centralized log access instead of requiring direct access to individual containers.

Useful log events include:

* Application startup
* Request failures
* Database errors
* Kafka failures
* Service communication failures

Sensitive values must not be logged.

For example:

```text
Passwords
Tokens
Secrets
```

should not appear in application logs.

---

# 12.41 CloudWatch Metrics

Important infrastructure metrics can include:

```text
ECS
 |
 +-- CPU Utilization
 |
 +-- Memory Utilization
 |
 +-- Running Tasks
```

For the Application Load Balancer:

```text
ALB
 |
 +-- Request Count
 |
 +-- Target Response Errors
 |
 +-- Healthy Targets
```

For RDS:

```text
RDS
 |
 +-- CPU
 |
 +-- Connections
 |
 +-- Storage
 |
 +-- Replica Health
```

The exact monitoring configuration should be documented separately under Production Readiness.

---

# 12.42 CloudWatch Alarms

CloudWatch alarms can notify operators when important conditions occur.

Examples:

```text
High CPU
```

```text
High Memory
```

```text
Application Errors
```

```text
Unhealthy ECS Tasks
```

```text
Database Resource Issues
```

Conceptually:

```text
Metric
   |
   v
Threshold
   |
   v
Exceeded?
   |
   +----------+
   |          |
  No         Yes
   |          |
   v          v
Continue    Alarm
```

Production alert thresholds should be based on actual workload behavior rather than arbitrary values.

---

# 12.43 IAM

## What Is IAM?

AWS Identity and Access Management controls access to AWS resources.

IAM answers:

```text
Who can perform an action?
```

and:

```text
What action are they allowed to perform?
```

For CitiCore, IAM is important for:

* ECS task permissions
* ECR access
* Secrets Manager access
* CloudWatch logging
* CI/CD deployment permissions

---

# 12.44 IAM Roles

Applications should generally use IAM roles instead of storing permanent AWS access keys inside containers.

Conceptually:

```text
ECS Task
    |
    v
IAM Role
    |
    v
AWS Permissions
```

For example:

```text
Account Service
       |
       v
IAM Task Role
       |
       +---- Read Required Secret
       |
       +---- Write Application Logs
```

The service should not receive unnecessary permissions.

This follows:

> **Principle of Least Privilege**

---

# 12.45 ECS Task Role vs Execution Role

A useful ECS distinction is:

## Task Execution Role

Used by ECS infrastructure to perform operations required to start and run the task.

Examples can include:

```text
Pull Image from ECR
```

```text
Send Logs to CloudWatch
```

```text
Retrieve configured secrets during task startup
```

depending on configuration.

---

## Task Role

Used by the application running inside the container to access AWS services.

Example:

```text
Spring Boot Application
       |
       v
Task Role
       |
       v
AWS Service
```

For example:

```text
Application
     |
     v
Secrets Manager
```

The two roles should not be confused.

---

# 12.46 Secrets Manager

Sensitive configuration should not be hardcoded into:

```text
Docker Image
```

```text
application.yml
```

```text
Jenkinsfile
```

Instead:

```text
ECS Task
    |
    v
IAM Permission
    |
    v
AWS Secrets Manager
    |
    v
Database Credentials
```

Secrets Manager was explained in detail in:

> **Topic 11 — Database Security**

This section only establishes its role in the AWS infrastructure architecture.

---

# 12.47 Complete CitiCore AWS Infrastructure Flow

The overall infrastructure can be represented as:

```text
                         Internet
                            |
                            v
                    Internet Gateway
                            |
                            v
                 Application Load Balancer
                            |
                            v
                    Target Group / Routing
                            |
                            v
+----------------------------------------------------+
|                       VPC                          |
|                                                    |
|   Private Application Environment                  |
|                                                    |
|      +-------------------------------+             |
|      |        ECS Cluster            |             |
|      |                               |             |
|      |  +-------------------------+  |             |
|      |  | Auth Service            |  |             |
|      |  +-------------------------+  |             |
|      |                               |             |
|      |  +-------------------------+  |             |
|      |  | Account Service         |  |             |
|      |  +-------------------------+  |             |
|      |                               |             |
|      |  +-------------------------+  |             |
|      |  | Transaction Service     |  |             |
|      |  +-------------------------+  |             |
|      +-------------------------------+             |
|                                                    |
|                    |                               |
|                    v                               |
|                 AWS RDS                            |
|                                                    |
+----------------------------------------------------+
```

Supporting services:

```text
ECR
 |
 v
Container Images
 |
 v
ECS
```

```text
IAM
 |
 v
Controlled AWS Permissions
```

```text
Secrets Manager
 |
 v
Sensitive Configuration
```

```text
CloudWatch
 |
 v
Logs and Monitoring
```

---

# 12.48 End-to-End Deployment Architecture

A CitiCore service deployment follows this general path:

```text
Developer
    |
    v
Source Code
    |
    v
Git Repository
    |
    v
Jenkins Pipeline
    |
    v
Maven Build
    |
    v
Docker Image
    |
    v
Amazon ECR
    |
    v
ECS Task Definition
    |
    v
ECS Service Deployment
    |
    v
Fargate Runs Container
    |
    v
ALB Routes Traffic
```

Runtime dependencies:

```text
ECS Task
    |
    +---- IAM Role
    |
    +---- Secrets
    |
    +---- CloudWatch Logs
    |
    +---- RDS Connection
```

---

# 12.49 Failure Handling at the Infrastructure Layer

Infrastructure failures must be considered separately from application failures.

---

## ECS Task Failure

```text
Container Stops
      |
      v
ECS Detects Desired Count Mismatch
      |
      v
Replacement Task Started
```

The ECS Service attempts to maintain the desired number of tasks.

---

## Unhealthy Application

```text
Application Running
        |
        X
     Unhealthy
        |
        v
ALB Health Check Fails
        |
        v
ALB Stops Routing New Traffic
```

The exact recovery behavior depends on ECS health checks, service configuration, and deployment settings.

---

## Database Failure

```text
Application
     |
     v
Database Connection Failure
```

The application must handle:

* Connection failures
* Timeouts
* Retry policies where appropriate
* Error responses

Infrastructure monitoring should detect database health issues.

---

## Deployment Failure

```text
New ECS Deployment
       |
       v
New Tasks Start
       |
       v
Health Check
       |
       +----------------+
       |                |
    Healthy          Unhealthy
       |                |
       v                v
Continue          Investigate / Rollback
```

Rollback strategy is covered in more detail under AWS Container Deployment and Jenkins CI/CD.

---

# 12.50 Real CitiCore Infrastructure Work

Based on the CitiCore implementation and project work, important AWS activities included:

* Creating AWS infrastructure resources
* Working with ECS services and tasks
* Deploying multiple containerized services
* Working with ECR container images
* Configuring ECS/Fargate deployment
* Working with Application Load Balancer concepts
* Managing VPC networking
* Working with security groups
* Using RDS
* Monitoring AWS resources
* Investigating ECS task states and service scaling
* Cleaning up running services and tasks to reduce unnecessary AWS resource usage

These are important because they represent practical project experience rather than only theoretical AWS knowledge.

---

# 12.51 Real Issue Context: ECS Tasks and Services

## Real Issue

During CitiCore AWS work, multiple services and ECS tasks were running in the cluster.

The task list showed several CitiCore services running as ECS tasks.

The practical question encountered was whether stopping tasks or reducing service desired counts would stop the infrastructure workloads.

---

## Root Cause

In ECS, a distinction exists between:

```text
ECS Task
```

and:

```text
ECS Service
```

A task started manually can simply stop.

However, a task managed by an ECS Service behaves differently.

```text
ECS Service
      |
      v
Desired Count
      |
      v
Maintain Number of Tasks
```

If a service has:

```text
Desired Count = 1
```

and its task is manually stopped:

```text
Task Stopped
      |
      v
ECS Service Detects Missing Task
      |
      v
Replacement Task Started
```

---

## Solution

To intentionally stop service-managed workloads:

```text
ECS Service
      |
      v
Update Desired Count
      |
      v
0
```

Conceptually:

```text
Desired Count: 1
        |
        v
Running Task
```

Change to:

```text
Desired Count: 0
        |
        v
No Running Task
```

This is different from simply stopping individual tasks.

---

## Prevention

Before stopping infrastructure resources, identify:

```text
Is this task managed by an ECS Service?
```

If yes:

```text
Update Service Desired Count
```

rather than relying only on manually stopping tasks.

---

## Interview Explanation

> "While working with ECS, I learned the difference between a task and a service. If a task belongs to an ECS Service, manually stopping the task does not permanently stop the workload because ECS tries to maintain the configured desired count. To intentionally stop the service, the desired count must be reduced, for example to zero."

This is a practical AWS troubleshooting point and demonstrates real understanding of ECS behavior.

---

# 12.52 Architecture Decisions

## Decision 1 — Containerized Microservices

CitiCore services are containerized because each service can be independently packaged and deployed.

```text
Account Service
       |
       v
Docker Image
```

```text
Transaction Service
       |
       v
Docker Image
```

### Advantage

Independent deployment.

### Trade-off

More infrastructure and operational complexity.

---

# 12.53 Decision 2 — ECS/Fargate

CitiCore uses AWS container infrastructure rather than manually managing application processes on individual virtual machines.

### Advantage

```text
Less Server Management
```

### Trade-off

```text
More Dependency on AWS Managed Infrastructure
```

---

# 12.54 Decision 3 — ALB as the Entry Point

Instead of exposing every service directly:

```text
Internet
    |
    +---- Auth Service
    |
    +---- Account Service
    |
    +---- Transaction Service
```

CitiCore infrastructure can use:

```text
Internet
    |
    v
ALB
    |
    v
Application Infrastructure
```

### Advantage

Centralized traffic management.

### Trade-off

Requires additional routing and health-check configuration.

---

# 12.55 Decision 4 — Private Application Services

The application services should not require direct internet exposure.

```text
Internet
    |
    v
ALB
    |
    v
Private ECS Services
```

### Advantage

Reduced attack surface.

### Trade-off

Internal networking configuration becomes more complex.

---

# 12.56 Decision 5 — Managed Database Using RDS

Instead of manually running a database server:

```text
EC2
 |
 v
Manual MySQL Installation
```

CitiCore uses managed database infrastructure:

```text
AWS RDS
```

### Advantages

* Reduced infrastructure management
* AWS database service integration
* Managed operational capabilities

### Trade-offs

* Less low-level infrastructure control
* AWS-specific operational considerations

---

# 12.57 Decision 6 — ECR for Container Images

Using ECR keeps container images close to the AWS deployment environment.

```text
Build
  |
  v
ECR
  |
  v
ECS
```

### Advantages

* AWS integration
* Centralized image registry
* Controlled image storage

---

# 12.58 Production Considerations

The current CitiCore project infrastructure provides practical AWS deployment experience, but a real banking production system would require additional controls.

---

## High Availability

Production services should run across multiple Availability Zones where required.

```text
ALB
 |
 +-----------------------+
 |                       |
 v                       v
AZ-A                    AZ-B
ECS Task                ECS Task
```

---

## Auto Scaling

The number of running tasks may need to increase based on demand.

Conceptually:

```text
Traffic Increases
       |
       v
CPU / Memory / Request Metrics
       |
       v
Scaling Policy
       |
       v
More ECS Tasks
```

Scaling policies should be based on actual workload requirements.

---

## Database High Availability

A production database should consider:

* Multi-AZ configuration
* Backups
* Recovery strategy
* Replica monitoring

The exact design depends on RPO and RTO requirements.

---

## Network Segmentation

Production systems should separate:

```text
Public Layer
```

```text
Application Layer
```

```text
Data Layer
```

with controlled communication between them.

---

## Secrets

Production secrets should:

* Not be hardcoded
* Not be stored in Git
* Be accessed through IAM
* Be rotated according to security requirements

---

## Observability

Production infrastructure should monitor:

```text
Application
+
Container
+
Network
+
Database
+
Load Balancer
```

rather than monitoring only whether the server is running.

---

# 12.59 How I Would Explain the AWS Architecture in an Interview

> "For CitiCore, I used AWS infrastructure to deploy containerized microservices. The services are packaged as Docker images and stored in Amazon ECR. ECS with Fargate runs the containers, while the ECS Service maintains the desired number of running tasks.
>
> The application is deployed inside a VPC with network separation between public-facing and private resources. An Application Load Balancer acts as the external traffic entry point and routes requests to healthy application targets. Security Groups restrict communication between the ALB, ECS services, and RDS database.
>
> RDS provides the managed relational database layer, CloudWatch is used for centralized logging and monitoring, and IAM controls access to AWS resources and secrets. For internal service communication, AWS Service Connect can also be used in the ECS environment."

---

# 12.60 30–60 Second Interview Answer

> "In CitiCore, I deployed microservices on AWS using ECS and Fargate. Each Spring Boot service is containerized with Docker, pushed to Amazon ECR, and deployed through an ECS service. The application runs inside a VPC, where the ALB acts as the public entry point and the application containers remain in private networking layers. Security Groups control traffic between the ALB, ECS services, and RDS. I also worked with CloudWatch for monitoring and IAM roles for secure AWS resource access."

---

# 12.61 Interview Follow-up Questions

## Q1. What is the difference between ECS and Fargate?

**Answer:**

ECS is the container orchestration service. Fargate is the serverless compute option that runs ECS containers without requiring us to manage EC2 instances.

---

## Q2. What is the difference between an ECS Task and an ECS Service?

**Answer:**

A task is an instance of a task definition running one or more containers. An ECS Service manages and maintains the desired number of running tasks.

---

## Q3. What happens if you manually stop a task managed by an ECS Service?

**Answer:**

If the service still has a desired count greater than zero, ECS will attempt to start a replacement task to maintain the desired state.

---

## Q4. Why use ECR?

**Answer:**

ECR stores Docker images that ECS can pull during deployment. It integrates directly with AWS container infrastructure.

---

## Q5. Why use an ALB?

**Answer:**

An ALB provides a centralized HTTP/HTTPS entry point, routes traffic to healthy targets, and supports health checks and load distribution.

---

## Q6. What is a VPC?

**Answer:**

A VPC is a logically isolated network in AWS where we define IP ranges, subnets, routing, and network boundaries for resources.

---

## Q7. Why place ECS services in private subnets?

**Answer:**

Private placement reduces direct internet exposure. External traffic can enter through an ALB while application services remain internal.

---

## Q8. What is a NAT Gateway?

**Answer:**

A NAT Gateway allows resources in private subnets to initiate outbound internet connections without accepting unsolicited inbound internet connections.

---

## Q9. How do Security Groups work?

**Answer:**

Security Groups act as virtual firewalls that control inbound and outbound traffic for AWS resources.

---

## Q10. Why should RDS not be publicly accessible?

**Answer:**

A database should generally be protected from direct internet access. Restricting access to trusted application resources reduces the attack surface.

---

## Q11. What is the difference between an ECS task role and task execution role?

**Answer:**

The execution role is used by ECS infrastructure to perform actions required to start the task, while the task role provides AWS permissions to the application running inside the container.

---

## Q12. How does ECS recover from a failed task?

**Answer:**

An ECS Service compares the number of running tasks with the configured desired count and attempts to start replacement tasks when necessary.

---

## Q13. How would you scale CitiCore on AWS?

**Answer:**

I would scale ECS services horizontally based on workload metrics, distribute tasks across Availability Zones, use ALB load balancing, and monitor database capacity separately.

---

# 12.62 Key Takeaways

```text
CitiCore AWS Infrastructure
```

is built around:

```text
VPC
```

for network isolation.

```text
Public and Private Subnets
```

for network segmentation.

```text
Security Groups
```

for controlled communication.

```text
ECR
```

for container image storage.

```text
ECS / Fargate
```

for running microservices.

```text
ALB
```

for incoming application traffic.

```text
RDS
```

for the relational database layer.

```text
IAM
```

for access control.

```text
Secrets Manager
```

for sensitive configuration.

```text
CloudWatch
```

for logs and monitoring.

---

# 12.63 Final Architecture Summary

```text
                         USERS
                           |
                           v
                        INTERNET
                           |
                           v
                    INTERNET GATEWAY
                           |
                           v
                 APPLICATION LOAD BALANCER
                           |
                           v
+--------------------------------------------------+
|                        VPC                       |
|                                                  |
|       +----------------------------------+       |
|       |        PRIVATE SERVICES          |       |
|       |                                  |       |
|       |   ECS / FARGATE                  |       |
|       |                                  |       |
|       |   +-- Auth Service               |       |
|       |   +-- User Service               |       |
|       |   +-- Account Service            |       |
|       |   +-- Transaction Service        |       |
|       |   +-- Notification Service       |       |
|       +----------------------------------+       |
|                        |                         |
|                        v                         |
|                    AWS RDS                       |
|                                                  |
+--------------------------------------------------+

             SUPPORTING AWS SERVICES

ECR ------------> Container Images
IAM ------------> Access Control
Secrets Manager -> Sensitive Configuration
CloudWatch -----> Logs and Monitoring
```

The most important infrastructure principle is:

> **CitiCore's AWS architecture separates traffic entry, application execution, data storage, networking, security, and observability so that microservices can be deployed and managed in a controlled cloud environment.**

---

## Topic 13 - Docker

## 13.1 What Is Docker?

Docker is a containerization platform that allows an application and its required runtime environment to be packaged into a container.

In CitiCore, Docker is used to package Spring Boot microservices so that they can run consistently across different environments.

Without Docker, an application may depend on the machine where it is running.

For example:

```text
Developer Machine A
    |
    +-- Java Version A
    +-- Maven Version A
    +-- Environment Configuration A
```

Another environment may have:

```text
Developer Machine B
    |
    +-- Different Java Version
    +-- Different Configuration
    +-- Different Dependencies
```

This can lead to the common problem:

> "It works on my machine."

Docker reduces this problem by packaging the application with its required runtime environment.

```text
Spring Boot Application
        |
        v
Docker Image
        |
        v
Docker Container
```

The same image can then be used in different environments.

```text
Developer Machine
        |
        v
Docker Image
        |
        +------------------+
        |                  |
        v                  v
Local Environment       AWS ECS
```

---

# 13.2 Why Do We Need Docker?

CitiCore contains multiple microservices and supporting infrastructure components.

These include services such as:

* Auth Service
* User Service
* Account Service
* Transaction Service
* Notification Service
* Config Server
* Eureka Server
* API Gateway

The project also depends on infrastructure components such as:

* MySQL
* Kafka
* Redis

Running everything manually creates several problems.

For example:

```text
Start Config Server
        |
        v
Start Eureka Server
        |
        v
Start API Gateway
        |
        v
Start Auth Service
        |
        v
Start User Service
        |
        v
Start Account Service
        |
        v
Start Transaction Service
```

In addition, supporting services must also be available.

```text
Start MySQL
Start Kafka
Start Redis
```

Docker helps standardize how these components are packaged and executed.

---

# 13.3 CitiCore Context

## CitiCore Implementation

Docker is used in CitiCore to containerize Spring Boot services.

The general flow is:

```text
Java Source Code
       |
       v
Maven Build
       |
       v
Spring Boot JAR
       |
       v
Docker Build
       |
       v
Docker Image
       |
       v
Docker Container
```

For deployment:

```text
Spring Boot Service
        |
        v
Docker Image
        |
        v
Container Registry
        |
        v
AWS ECR
        |
        v
ECS / Fargate
```

For local development, Docker and Docker Compose help run multiple services and dependencies together.

---

# 13.4 Docker Architecture

The basic Docker architecture contains several important concepts.

```text
Developer
    |
    v
Dockerfile
    |
    v
docker build
    |
    v
Docker Image
    |
    v
docker run
    |
    v
Docker Container
```

These concepts should not be confused.

---

# 13.5 Docker Image

A Docker image is a packaged blueprint used to create containers.

For a CitiCore service, an image may contain:

```text
Spring Boot Application
        |
        +-- Compiled JAR
        |
        +-- Java Runtime
        |
        +-- Application Startup Command
```

Conceptually:

```text
Docker Image
     |
     +-- Base Image
     |
     +-- Java Runtime
     |
     +-- CitiCore Application JAR
     |
     +-- Startup Configuration
```

An image is immutable after it is built.

A new application version normally requires a new image.

For example:

```text
citicore-account-service:1.0
```

After code changes:

```text
citicore-account-service:1.1
```

The image represents the packaged application version.

---

# 13.6 Docker Container

A container is a running instance of a Docker image.

The relationship is:

```text
Docker Image
      |
      v
docker run
      |
      v
Docker Container
```

One image can create multiple containers.

Example:

```text
Account Service Image
        |
        +----------------+
        |                |
        v                v
Container 1          Container 2
```

In CitiCore, a container runs a specific service instance.

Example:

```text
Account Service Image
        |
        v
Account Service Container
        |
        v
Spring Boot Application
```

---

# 13.7 Dockerfile

A Dockerfile contains instructions for building a Docker image.

A simplified Spring Boot Dockerfile structure is:

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY target/*.jar app.jar

ENTRYPOINT ["java", "-jar", "app.jar"]
```

The important steps are:

```text
FROM
 |
 v
Choose Base Image
```

```text
WORKDIR
 |
 v
Define Working Directory
```

```text
COPY
 |
 v
Copy Application Artifact
```

```text
ENTRYPOINT
 |
 v
Start Application
```

---

# 13.8 CitiCore Dockerfile Flow

The general CitiCore Docker build process is:

```text
Spring Boot Project
        |
        v
mvn clean package
        |
        v
target/application.jar
        |
        v
Dockerfile
        |
        v
COPY JAR into Image
        |
        v
Run java -jar
```

Conceptually:

```text
+----------------------------------+
|         Docker Image             |
|                                  |
|  Java Runtime                    |
|                                  |
|  /app                            |
|      |                           |
|      +-- app.jar                 |
|                                  |
|  ENTRYPOINT                      |
|      java -jar app.jar           |
+----------------------------------+
```

When the container starts:

```text
Docker Container
        |
        v
java -jar app.jar
        |
        v
Spring Boot Application
```

---

# 13.9 Important Docker Commands

## Check Docker Version

```bash
docker --version
```

---

## View Running Containers

```bash
docker ps
```

This shows currently running containers.

Example information includes:

```text
Container ID
Image
Command
Status
Ports
Container Name
```

---

## View All Containers

```bash
docker ps -a
```

This includes stopped containers.

---

## View Docker Images

```bash
docker images
```

or:

```bash
docker image ls
```

---

## Build an Image

```bash
docker build -t citicore-account-service .
```

The command means:

```text
docker build
```

Build a Docker image.

```text
-t citicore-account-service
```

Assign a tag/name to the image.

```text
.
```

Use the current directory as the Docker build context.

---

## Run a Container

```bash
docker run -d -p 8080:8080 citicore-account-service
```

Conceptually:

```text
Host Port 8080
      |
      v
Container Port 8080
```

The request flow becomes:

```text
Browser / Client
       |
       v
localhost:8080
       |
       v
Docker Container
       |
       v
Spring Boot Application
```

---

# 13.10 Port Mapping

A container has its own network environment.

Suppose the Spring Boot application listens on:

```text
8080
```

inside the container.

Without port mapping:

```text
Host
  X
  |
Container:8080
```

The application may not be accessible from the host through the expected port.

Port mapping exposes the container port.

```bash
docker run -p 8080:8080 application
```

The flow becomes:

```text
Host:8080
    |
    v
Container:8080
    |
    v
Spring Boot Application
```

---

# 13.11 Detached Mode

Docker containers can run in detached mode.

Example:

```bash
docker run -d application
```

Detached mode means the container runs in the background.

This is useful when multiple CitiCore services need to run simultaneously.

```text
Docker
   |
   +-- Config Server
   |
   +-- Eureka Server
   |
   +-- API Gateway
   |
   +-- Account Service
   |
   +-- Transaction Service
```

---

# 13.12 Container Logs

Logs are essential for debugging.

A common command is:

```bash
docker logs <container-id>
```

To continuously follow logs:

```bash
docker logs -f <container-id>
```

The troubleshooting flow is often:

```text
Container Not Working
        |
        v
Check docker ps
        |
        v
Check Container Logs
        |
        v
Identify Application Error
```

For CitiCore, logs can help identify:

* Spring Boot startup failures
* Database connection errors
* Configuration problems
* Kafka connection issues
* Port conflicts

---

# 13.13 Stop a Container

A running container can be stopped using:

```bash
docker stop <container-id>
```

The container remains available but is no longer running.

To start it again:

```bash
docker start <container-id>
```

To remove it:

```bash
docker rm <container-id>
```

---

# 13.14 Remove a Docker Image

An image can be removed using:

```bash
docker rmi <image-id>
```

However, Docker will not remove an image that is actively used by a container unless the related container is handled first.

This relationship is important:

```text
Docker Image
      |
      v
Docker Container
```

The image is the blueprint, while the container is the running or created instance.

---

# 13.15 Docker Networking

Docker containers need to communicate with each other.

For CitiCore, examples include:

```text
Account Service
       |
       v
MySQL
```

```text
Transaction Service
       |
       v
Kafka
```

```text
Service
       |
       v
Redis
```

Docker networking allows containers to communicate through Docker-managed networks.

Conceptually:

```text
Docker Network
 |
 +----------------------+
 |                      |
 v                      v
Account Service       MySQL
```

---

# 13.16 Why `localhost` Can Be a Problem Inside Docker

One important Docker concept is:

> `localhost` inside a container refers to that container itself.

Consider:

```text
Account Service Container
```

If the application uses:

```text
localhost:3306
```

it expects MySQL to be running inside the same container.

But in a multi-container architecture:

```text
Account Service Container
        |
        X localhost
        |
        v
MySQL Container
```

The service should instead connect through the Docker network and service/container hostname.

Conceptually:

```text
Account Service
       |
       v
mysql
       |
       v
MySQL Container
```

This is one of the most important concepts when running microservices using Docker.

---

# 13.17 Docker Networks in CitiCore

A simplified CitiCore local architecture is:

```text
                Docker Network
+------------------------------------------------+
|                                                |
|  Config Server                                 |
|       |                                        |
|       v                                        |
|  Eureka Server                                 |
|       |                                        |
|       v                                        |
|  API Gateway                                   |
|       |                                        |
|       +------------------------------+         |
|       |              |               |         |
|       v              v               v         |
|  Auth Service   Account Service  Transaction   |
|                                    Service      |
|                                                |
|       +----------+------------+                |
|       |          |            |                |
|       v          v            v                |
|    MySQL       Kafka        Redis              |
|                                                |
+------------------------------------------------+
```

The exact service topology depends on the active CitiCore Docker Compose configuration.

The main principle is:

> Services running inside the same Docker network should communicate using the correct internal hostname rather than assuming `localhost`.

---

# 13.18 Docker Volumes

Containers are designed to be replaceable.

This creates a problem for persistent data.

Suppose MySQL runs inside a container:

```text
MySQL Container
       |
       v
Database Data
```

If the container is removed, the data may also be lost if persistence is not configured properly.

Docker volumes provide persistent storage.

```text
MySQL Container
       |
       v
Docker Volume
       |
       v
Persistent Data
```

---

# 13.19 Why Volumes Matter

For infrastructure services such as databases:

```text
MySQL
```

or:

```text
Kafka
```

persistent data may be important depending on the environment.

Conceptually:

```text
Container Removed
       |
       v
New Container Created
       |
       v
Existing Volume Reused
```

The application can continue using the persistent data.

Volumes should be configured carefully because local development requirements and production persistence requirements are different.

---

# 13.20 Docker Compose

## What Is Docker Compose?

Docker Compose allows multiple containers to be defined and started using a single configuration file.

This is particularly useful for a microservices project like CitiCore.

Instead of manually running:

```text
docker run config-server
docker run eureka-server
docker run api-gateway
docker run auth-service
docker run account-service
docker run transaction-service
docker run mysql
docker run kafka
docker run redis
```

Docker Compose allows these services to be managed together.

Conceptually:

```text
docker-compose.yml
        |
        v
Docker Compose
        |
        +----------------------+
        |                      |
        v                      v
Microservices             Infrastructure
```

---

# 13.21 CitiCore Docker Compose Architecture

The local environment can be represented as:

```text
docker-compose.yml
        |
        +-----------------------------+
        |                             |
        v                             v
Application Services            Infrastructure
        |                             |
        +-- Config Server             +-- MySQL
        +-- Eureka Server             +-- Kafka
        +-- API Gateway               +-- Redis
        +-- Auth Service
        +-- User Service
        +-- Account Service
        +-- Transaction Service
        +-- Notification Service
```

This simplifies local setup.

Instead of manually starting every component, the developer can use one orchestration command.

---

# 13.22 Important Docker Compose Commands

## Start Services

```bash
docker compose up
```

---

## Start Services in Background

```bash
docker compose up -d
```

---

## Check Running Services

```bash
docker compose ps
```

This command is useful because it shows the state of services managed by the Compose configuration.

Example:

```text
NAME
IMAGE
SERVICE
STATUS
PORTS
```

---

## Stop Services

```bash
docker compose down
```

---

## Rebuild Services

After application code or Docker configuration changes:

```bash
docker compose up --build
```

or:

```bash
docker compose build
docker compose up -d
```

---

# 13.23 CitiCore Local Development Flow

The local execution flow is:

```text
Source Code Changes
        |
        v
Maven Build
        |
        v
Docker Build
        |
        v
Docker Compose
        |
        v
Start All Required Containers
```

The resulting environment may look like:

```text
Developer Machine
        |
        v
Docker
        |
        +-------------------------+
        |                         |
        v                         v
Spring Boot Services       Supporting Services
        |                         |
        +-- Auth                  +-- MySQL
        +-- Account               +-- Kafka
        +-- Transaction           +-- Redis
        +-- Gateway
```

---

# 13.24 Service Startup Order

Microservices can have dependencies.

For example:

```text
Config Server
       |
       v
Eureka Server
       |
       v
Application Services
```

A service may fail if a required dependency is unavailable during startup.

For example:

```text
Account Service
       |
       v
Needs Config Server
       |
       X
Config Server Not Available
```

Docker Compose can define service dependencies, but an important practical point is:

> Starting a dependency container does not always guarantee that the application inside that container is fully ready.

For example:

```text
MySQL Container Started
        |
        X
MySQL Still Initializing
```

Therefore, production-ready container orchestration should use appropriate health checks and retry behavior.

---

# 13.25 Docker Health Checks

A container being in the `running` state does not always mean the application is healthy.

Example:

```text
Container
   |
   +-- Running
          |
          X
     Spring Boot Failed
```

Health checks help distinguish:

```text
Container Started
```

from:

```text
Application Ready
```

A health-check flow can be:

```text
Health Check
     |
     v
Application Endpoint
     |
     +----------------+
     |                |
 Healthy          Unhealthy
```

For Spring Boot applications, this may use an Actuator health endpoint when configured.

The exact health-check endpoint should match the active CitiCore implementation.

---

# 13.26 Docker Build Lifecycle

The complete build flow is:

```text
Java Source Code
       |
       v
Maven
       |
       v
JAR File
       |
       v
Docker Build Context
       |
       v
Dockerfile
       |
       v
Docker Image
       |
       v
Docker Registry
       |
       v
Deployment Platform
```

For CitiCore:

```text
Spring Boot
     |
     v
Maven Package
     |
     v
Docker Build
     |
     v
CitiCore Service Image
```

---

# 13.27 Image Tagging

Docker images should be tagged properly.

Example:

```text
citicore-account-service:latest
```

However, relying only on:

```text
latest
```

can create deployment ambiguity.

A better approach is versioned or commit-based tags.

Example:

```text
citicore-account-service:1.0.0
```

or:

```text
citicore-account-service:<git-sha>
```

This is especially useful for CI/CD.

```text
Git Commit
    |
    v
Git SHA
    |
    v
Docker Image Tag
    |
    v
Deployment
```

Benefits include:

* Better traceability
* Easier rollback
* Clearer deployment history

---

# 13.28 Docker in the CitiCore CI/CD Flow

The general pipeline is:

```text
Git Push
   |
   v
Jenkins Pipeline
   |
   v
Maven Build
   |
   v
Run Tests
   |
   v
Docker Build
   |
   v
Docker Image
   |
   v
Push to ECR
   |
   v
Deploy to ECS
```

Docker acts as the bridge between:

```text
Application Development
```

and:

```text
Container Deployment
```

The same application artifact is packaged into a container image and deployed to AWS infrastructure.

---

# 13.29 Multi-Service Deployment

CitiCore is a microservices application.

This means each service can have its own image.

```text
+----------------------------+
| Config Server Image        |
+----------------------------+

+----------------------------+
| Eureka Server Image        |
+----------------------------+

+----------------------------+
| API Gateway Image          |
+----------------------------+

+----------------------------+
| Auth Service Image         |
+----------------------------+

+----------------------------+
| Account Service Image      |
+----------------------------+

+----------------------------+
| Transaction Service Image  |
+----------------------------+
```

This provides deployment independence.

For example:

```text
Account Service Updated
        |
        v
Build New Account Service Image
        |
        v
Deploy Account Service
```

Other services do not necessarily need to be rebuilt.

---

# 13.30 Why Containerization Is Useful for Microservices

Without containers, services may depend on machine-specific environments.

```text
Service A
    |
    v
Machine A
```

```text
Service B
    |
    v
Machine B
```

Configuration differences can increase deployment complexity.

With Docker:

```text
Service
   |
   v
Standard Image
   |
   v
Consistent Runtime
```

Each service has a predictable deployment artifact.

---

# 13.31 Docker vs Virtual Machines

Docker containers and virtual machines solve different problems.

## Virtual Machine

```text
Physical Server
      |
      v
Hypervisor
      |
      +------------------+
      |                  |
      v                  v
Virtual Machine      Virtual Machine
      |                  |
      v                  v
Guest OS            Guest OS
      |                  |
      v                  v
Application         Application
```

## Docker

```text
Host OS
   |
   v
Docker Engine
   |
   +------------------+
   |                  |
   v                  v
Container         Container
   |                  |
   v                  v
Application      Application
```

Containers are generally more lightweight because they do not package a complete guest operating system in the same way as traditional virtual machines.

---

# 13.32 Docker Trade-offs

## Advantages

### Consistency

The same Docker image can run across environments.

```text
Local
  |
  v
Same Image
  |
  v
Cloud
```

---

### Portability

Containers can run on different environments supporting the required container runtime.

---

### Independent Deployment

Each CitiCore microservice can have its own image.

---

### Simplified Local Setup

Docker Compose can start multiple dependencies together.

---

### CI/CD Integration

Docker images can be built and deployed through automated pipelines.

---

## Disadvantages

### Additional Complexity

Developers must understand:

* Images
* Containers
* Networks
* Volumes
* Registries
* Port mappings

---

### Debugging Complexity

Problems can occur at multiple layers:

```text
Application
```

```text
Container
```

```text
Docker Network
```

```text
Host Machine
```

---

### Image Management

Images must be:

* Tagged
* Stored
* Versioned
* Cleaned up

---

# 13.33 Common Docker Problems in Microservices

## Problem 1 — Container Starts but Application Is Not Accessible

Possible causes include:

* Incorrect port mapping
* Application listening on another port
* Container networking issue
* Application startup failure

Investigation:

```bash
docker ps
```

Check port mappings.

Then:

```bash
docker logs <container-id>
```

Check application startup.

---

## Problem 2 — Service Cannot Connect to Another Container

Possible cause:

```text
Using localhost
```

inside the container.

Incorrect concept:

```text
Service Container
    |
    v
localhost:9092
```

Correct approach depends on the Docker network configuration.

Conceptually:

```text
Service Container
    |
    v
kafka:9092
    |
    v
Kafka Container
```

---

## Problem 3 — Old Application Version Is Still Running

Possible causes include:

* Old image
* Existing container
* Application not rebuilt
* Docker cache

Investigation should verify:

```text
Source Code
```

```text
JAR
```

```text
Docker Image
```

```text
Running Container
```

The important lesson is that changing source code does not automatically update a running Docker container.

---

# 13.34 Real CitiCore Docker Experience

During CitiCore development, Docker was used to manage multiple services and support the local microservices environment.

A practical environment involved services running through Docker Compose.

The running service status was checked using commands such as:

```bash
docker compose ps
```

This was useful for identifying:

* Which services were running
* Which services were stopped
* Container status
* Port mappings

The project also involved rebuilding services after code or configuration changes.

A typical workflow was:

```text
Change Code
    |
    v
Rebuild Application
    |
    v
Rebuild Docker Image
    |
    v
Restart Container
    |
    v
Verify Service
```

This is important because simply changing Java source code does not update an already-running container.

---

# 13.35 Real Issue Pattern: Rebuilding All Services

During multi-service development, rebuilding only one service may leave other containers running older versions.

A clean rebuild workflow may involve:

```text
Stop Existing Services
        |
        v
Rebuild Required Images
        |
        v
Start Updated Services
        |
        v
Verify Containers
```

Using Docker Compose:

```bash
docker compose down
```

Then:

```bash
docker compose up --build -d
```

Afterward:

```bash
docker compose ps
```

The exact command sequence should depend on whether persistent infrastructure data needs to be preserved.

---

# 13.36 Important Caution About `docker compose down`

Running:

```bash
docker compose down
```

stops and removes Compose-managed containers and networks.

However, persistent volumes may behave differently depending on the command and configuration.

Before performing cleanup, developers should understand:

```text
Container
```

versus:

```text
Volume
```

The goal is to avoid accidentally deleting important persistent development data.

For databases, data persistence should always be considered before removing resources.

---

# 13.37 Production Considerations

Docker is useful for packaging applications, but production deployment requires more than building an image.

A production container environment should consider:

* Immutable image versions
* Resource limits
* Health checks
* Centralized logging
* Secrets management
* Security scanning
* Image vulnerability management
* Rollback strategy
* Monitoring
* Horizontal scaling

---

# 13.38 Production Docker Image Best Practices

A production image should ideally be:

```text
Small
```

```text
Secure
```

```text
Versioned
```

```text
Reproducible
```

Avoid unnecessary dependencies.

The image should contain what the application requires to run.

---

# 13.39 Multi-Stage Docker Builds

A common production optimization is a multi-stage Docker build.

Conceptually:

```text
Stage 1
    |
    v
Build Application
    |
    v
JAR
    |
    v
Stage 2
    |
    v
Small Runtime Image
    |
    v
Run JAR
```

Example structure:

```dockerfile
FROM maven:3.9-eclipse-temurin-17 AS build

WORKDIR /app

COPY . .

RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre

WORKDIR /app

COPY --from=build /app/target/*.jar app.jar

ENTRYPOINT ["java", "-jar", "app.jar"]
```

This approach separates:

```text
Build Environment
```

from:

```text
Runtime Environment
```

The benefit is a potentially smaller and cleaner runtime image.

If the exact CitiCore Dockerfile used a different approach, this should be treated as a production improvement/example rather than the project's exact implementation.

---

# 13.40 Security Considerations

Containers should not contain:

* Hardcoded database passwords
* AWS credentials
* JWT secrets
* Production API keys

Instead:

```text
Container
    |
    v
Environment / Secret Injection
```

For AWS:

```text
ECS Task
    |
    v
Secrets Manager
```

The container image should remain reusable across environments.

---

# 13.41 How Docker Connects to AWS Deployment

Docker itself packages the application.

AWS infrastructure runs the container.

The deployment chain is:

```text
Spring Boot Application
        |
        v
Dockerfile
        |
        v
Docker Image
        |
        v
Amazon ECR
        |
        v
ECS Task Definition
        |
        v
ECS / Fargate
        |
        v
Running Container
```

Therefore:

> Docker creates the deployment artifact, while ECS/Fargate manages and runs that artifact in AWS.

This distinction is important in interviews.

---

# 13.42 How I Would Explain Docker in an Interview

> "In CitiCore, I used Docker to containerize the Spring Boot microservices and create a consistent runtime environment. Each service was packaged as a Docker image containing the application JAR and Java runtime configuration. For local development, Docker Compose helped manage multiple services and dependencies together. The Docker images were also part of the AWS deployment flow, where images could be stored in ECR and then deployed through ECS and Fargate. One important thing I learned was Docker networking, especially that localhost inside a container refers to the container itself, so services need to use the correct Docker service or container hostname when communicating."

---

# 13.43 30–60 Second Interview Answer

> "I used Docker in CitiCore to containerize our Spring Boot microservices. The application is first built into a JAR using Maven, and the JAR is packaged into a Docker image using a Dockerfile. For local development, Docker Compose helps run multiple services and dependencies together, including services such as MySQL, Kafka, and Redis. For AWS deployment, the Docker image acts as the deployment artifact and can be pushed to ECR and run on ECS Fargate. Docker helped us maintain consistency between local and deployment environments."

---

# 13.44 Interview Follow-up Questions

## Q1. What is the difference between a Docker image and a container?

**Answer:**

A Docker image is the immutable blueprint containing the application and runtime environment. A container is a running instance created from that image.

---

## Q2. What is a Dockerfile?

**Answer:**

A Dockerfile contains instructions used to build a Docker image, such as the base image, application files, working directory, and startup command.

---

## Q3. Why did you use Docker in CitiCore?

**Answer:**

Docker allowed each Spring Boot microservice to be packaged consistently and deployed independently. It also simplified local setup and integration with AWS ECS.

---

## Q4. What is Docker Compose?

**Answer:**

Docker Compose is used to define and manage multiple related containers using a single configuration file.

---

## Q5. What is the problem with localhost inside Docker?

**Answer:**

Inside a container, localhost refers to the same container. Therefore, a service connecting to another container must usually use the correct Docker hostname or service name instead of localhost.

---

## Q6. How do containers communicate?

**Answer:**

Containers can communicate through Docker networks. Services on the same configured network can typically communicate using the appropriate service or container hostname.

---

## Q7. What are Docker volumes?

**Answer:**

Docker volumes provide persistent storage that can survive container replacement or recreation.

---

## Q8. What happens when a Docker container stops?

**Answer:**

The running process inside the container stops. The container can remain available for inspection or restart unless it is removed.

---

## Q9. How do you debug a Docker container?

**Answer:**

I first check the container status using `docker ps`, inspect logs using `docker logs`, verify port mappings, and then investigate application-level errors.

---

## Q10. What is a multi-stage Docker build?

**Answer:**

It separates the build environment from the runtime environment. For example, one stage builds the Spring Boot JAR and another smaller stage runs only the final application artifact.

---

## Q11. How does Docker fit into CI/CD?

**Answer:**

The pipeline builds the application, creates a Docker image, tags it, pushes it to a container registry, and deploys that image to the container platform.

---

## Q12. How would you roll back a Docker-based deployment?

**Answer:**

I would redeploy a previously known stable image version. This is why immutable versioned image tags are important.

---

# 13.45 Key Takeaways

Docker provides the containerization layer for CitiCore.

The main flow is:

```text
Java Code
    |
    v
Maven Build
    |
    v
JAR
    |
    v
Docker Image
    |
    v
Docker Container
```

For multiple services:

```text
Docker Compose
    |
    +-- Microservices
    |
    +-- MySQL
    |
    +-- Kafka
    |
    +-- Redis
```

For AWS deployment:

```text
Docker Image
     |
     v
ECR
     |
     v
ECS / Fargate
     |
     v
Running CitiCore Service
```

The most important practical concepts are:

* Image vs Container
* Dockerfile
* Port Mapping
* Docker Networking
* `localhost` behavior inside containers
* Volumes
* Docker Compose
* Rebuilding images
* Container logs
* Image tagging
* CI/CD integration

---

# 13.46 Final Docker Architecture Summary

```text
                 CITICORE DEVELOPMENT

Spring Boot Services
        |
        v
     Maven Build
        |
        v
   Application JAR
        |
        v
     Docker Build
        |
        v
     Docker Image
        |
        +-----------------------------+
        |                             |
        v                             v
Local Docker Environment         AWS Deployment
        |                             |
        v                             v
Docker Compose                       ECR
        |                             |
        +-- Services                  v
        +-- MySQL                    ECS
        +-- Kafka                     |
        +-- Redis                     v
                                  Fargate
                                      |
                                      v
                               Running Containers
```

> **Docker is the packaging and runtime standardization layer that allows CitiCore microservices to run consistently in local development and cloud deployment environments.**

---

## TOPIC- 14. AWS CONTAINER DEPLOYMENT

## 14.1 What Is AWS Container Deployment?

AWS container deployment is the process of taking a containerized application and running it on AWS infrastructure.

In CitiCore, the deployment flow connects several services:

* Docker
* Amazon ECR
* Amazon ECS
* AWS Fargate
* ECS Task Definitions
* ECS Services
* Application Load Balancer
* Target Groups
* ECS Service Connect
* CloudWatch
* IAM
* Security Groups

The overall deployment path is:

```text
Spring Boot Application
        |
        v
Maven Build
        |
        v
Application JAR
        |
        v
Docker Image
        |
        v
Amazon ECR
        |
        v
ECS Task Definition
        |
        v
ECS Service
        |
        v
AWS Fargate Task
        |
        +----------------------+
        |                      |
        v                      v
      ALB              Service Connect
   External Traffic    Internal Traffic
```

The important distinction is:

> **Docker packages the application. ECR stores the image. ECS manages the deployment. Fargate runs the container.**

---

# 14.2 Why Do We Need AWS Container Deployment?

Running a Docker container locally is useful for development, but a production-style environment needs additional capabilities.

For example:

```text
Developer Machine
        |
        v
docker run
        |
        v
Application Running
```

This approach does not automatically provide:

* High availability
* Container orchestration
* Automatic replacement of failed containers
* Load balancing
* Managed networking
* Centralized logging
* IAM-based permissions
* Cloud deployment
* Deployment management

AWS ECS and Fargate provide a managed way to run containerized applications.

For CitiCore:

```text
Docker Image
      |
      v
AWS Cloud
      |
      v
ECS Service
      |
      v
Fargate Tasks
```

This allows the microservices to run without manually managing individual servers for every container.

---

# 14.3 CitiCore Context

## CitiCore Implementation

CitiCore uses a container-based deployment architecture on AWS.

The implementation follows this general flow:

```text
Local Development
        |
        v
Spring Boot Application
        |
        v
Docker Image
        |
        v
Amazon ECR
        |
        v
ECS Cluster
        |
        v
ECS Service
        |
        v
Fargate Task
```

The deployed environment includes infrastructure such as:

```text
AWS
 |
 +-- VPC
 |
 +-- Subnets
 |
 +-- Security Groups
 |
 +-- ECR
 |
 +-- ECS Cluster
 |      |
 |      +-- ECS Services
 |              |
 |              +-- Fargate Tasks
 |
 +-- ALB
 |
 +-- Service Connect
 |
 +-- CloudWatch
 |
 +-- IAM
```

The architecture separates different responsibilities.

```text
ECR
 |
 | Stores
 v
Container Images


ECS
 |
 | Orchestrates
 v
Containers


Fargate
 |
 | Provides
 v
Container Runtime


ALB
 |
 | Routes
 v
External HTTP Traffic


Service Connect
 |
 | Enables
 v
Internal Service Communication
```

---

# 14.4 High-Level CitiCore Deployment Architecture

A simplified architecture is:

```text
                         Internet
                            |
                            v
                   Application Load Balancer
                            |
                            v
                    ECS Target Group
                            |
                            v
                  +----------------------+
                  |    ECS Fargate       |
                  |                      |
                  |  API / Application   |
                  |       Service        |
                  +----------------------+
                            |
                            |
          +-----------------+------------------+
          |                 |                  |
          v                 v                  v
   Service Connect       Amazon RDS          Kafka
   Internal Services
```

Container images are supplied through ECR.

```text
Developer / Jenkins
        |
        v
Build Docker Image
        |
        v
Amazon ECR
        |
        v
ECS Task Pulls Image
        |
        v
Fargate Runs Container
```

---

# 14.5 Deployment Order

Deployment order is important in a microservices environment because some services depend on infrastructure or other platform components.

The documented CitiCore deployment order follows the dependency chain:

```text
AWS Infrastructure
        |
        v
Config Server
        |
        v
Eureka Server
        |
        v
Application Services
        |
        +-- Auth Service
        +-- User Service
        +-- Account Service
        +-- Transaction Service
        +-- Notification Service
        |
        v
API Gateway
```

Supporting infrastructure such as databases, Kafka, Redis, networking, and security configuration must also be available.

The principle is:

> Deploy dependencies before the services that require them.

For example:

```text
Auth Service
     |
     +--> Config Server
     |
     +--> Eureka
     |
     +--> Database
     |
     +--> Kafka
```

If a dependency is unavailable, startup or runtime communication can fail.

---

# 14.6 Why Deployment Order Matters

Consider deploying a service before its dependencies are ready.

```text
Auth Service Starts
       |
       v
Requests Configuration
       |
       X
Config Server Unavailable
```

Or:

```text
Auth Service Starts
       |
       v
Attempts Eureka Registration
       |
       X
Eureka Unavailable
```

This can cause:

* Startup failures
* Retry loops
* Registration failures
* Service discovery failures
* Runtime errors

A better sequence is:

```text
Infrastructure Ready
        |
        v
Config Server Ready
        |
        v
Eureka Ready
        |
        v
Application Services
```

However, production systems should not rely only on deployment order. Applications should also handle temporary dependency failures using:

* Retries
* Timeouts
* Health checks
* Graceful startup behavior
* Circuit breakers where appropriate

---

# 14.7 Amazon ECR

## What Is Amazon ECR?

Amazon Elastic Container Registry (ECR) is a managed container image registry.

In CitiCore, ECR stores Docker images before ECS deploys them.

The flow is:

```text
Docker Build
     |
     v
Docker Image
     |
     v
Push Image
     |
     v
Amazon ECR
     |
     v
ECS Pulls Image
```

For example:

```text
Account Service
      |
      v
Docker Image
      |
      v
ECR Repository
      |
      v
ECS Task Definition
      |
      v
Fargate Task
```

---

# 14.8 Why ECR Is Needed

Without a container registry:

```text
Developer Machine
      |
      v
Docker Image
```

The AWS runtime would not have a managed source from which to retrieve the image.

ECR provides:

* Image storage
* AWS integration
* IAM-based access
* Image versioning through tags
* Integration with ECS

The important deployment relationship is:

```text
Git Commit
     |
     v
Git SHA
     |
     v
Docker Image Tag
     |
     v
ECR Image
     |
     v
ECS Task Definition
```

This provides deployment traceability.

---

# 14.9 Building a Docker Image

The application must first be packaged.

For a Spring Boot service:

```text
Java Source Code
       |
       v
Maven Build
       |
       v
JAR File
       |
       v
Docker Build
       |
       v
Docker Image
```

Example:

```bash
mvn clean package
```

Then:

```bash
docker build -t citicore-account-service .
```

The exact image name depends on the service and repository structure.

---

# 14.10 ECR Authentication

Before pushing an image to ECR, Docker must authenticate with the registry.

A typical AWS CLI flow is:

```bash
aws ecr get-login-password \
  --region ap-south-1
```

The password is then passed to Docker login.

Conceptually:

```text
AWS CLI
    |
    v
Temporary Authentication Token
    |
    v
Docker Login
    |
    v
Authorized to Push Image
```

This is normally automated in CI/CD rather than manually performed for every deployment.

---

# 14.11 Tagging Images

Docker images should be tagged with meaningful versions.

For example:

```text
account-service:latest
```

Using only `latest` creates ambiguity because it does not clearly identify the deployed application version.

CitiCore's CI/CD notes use Git SHA-based tagging.

Example:

```text
account-service:434a66ea...
```

The exact SHA depends on the Git commit.

The relationship is:

```text
Git Commit
    |
    v
Commit SHA
    |
    v
Docker Image Tag
```

This makes it easier to answer:

> Which source code version is currently deployed?

The deployment can be traced through the SHA.

---

# 14.12 Why Git SHA Image Tagging Is Important

Suppose two application versions exist.

```text
Version A
    |
    v
account-service:25951ba...
```

```text
Version B
    |
    v
account-service:434a66e...
```

If Version B causes a problem, the previous version can be identified clearly.

```text
Current Version
      |
      v
434a66e
      |
      X
Problem
      |
      v
Rollback
      |
      v
25951ba
```

This provides:

* Traceability
* Reproducibility
* Easier rollback
* Clear deployment history

---

# 14.13 Pushing an Image to ECR

The deployment flow is:

```text
Local Docker Image
        |
        v
Tag With ECR Repository URI
        |
        v
Push to ECR
        |
        v
Image Available to ECS
```

Conceptually:

```text
Developer / Jenkins
       |
       v
Docker Image
       |
       v
Amazon ECR
       |
       v
ECS Pulls Image
```

The exact repository URI should be environment-specific and should not be hardcoded into application source code.

---

# 14.14 ECS Task Definition

## What Is an ECS Task Definition?

An ECS Task Definition is a deployment blueprint that describes how a container should run.

It can define:

* Docker image
* CPU
* Memory
* Container port
* Environment variables
* Secrets
* Logging
* IAM roles
* Network configuration

Conceptually:

```text
ECR Image
    |
    v
Task Definition
    |
    +-- Image
    +-- CPU
    +-- Memory
    +-- Port
    +-- Environment
    +-- Secrets
    +-- Logging
    |
    v
Fargate Task
```

A Task Definition does not itself run the application.

It defines **how ECS should run it**.

---

# 14.15 Task Definition in CitiCore

A simplified CitiCore task flow is:

```text
CitiCore Service Image
        |
        v
ECS Task Definition
        |
        +----------------------+
        |                      |
        | Container Image      |
        | CPU                  |
        | Memory               |
        | Port                 |
        | Logs                 |
        | IAM                  |
        +----------------------+
        |
        v
ECS Fargate Task
```

When a new application version is deployed, a new task definition revision may be created.

Example:

```text
Task Definition Revision 23
        |
        v
Old Application Image
```

After deployment:

```text
Task Definition Revision 24
        |
        v
New Application Image
```

The ECS service can then be updated to use the new revision.

---

# 14.16 ECS Task Definition Revision

Task definitions are revisioned.

The relationship is:

```text
Task Definition
      |
      +-- Revision 1
      |
      +-- Revision 2
      |
      +-- Revision 3
      |
      +-- Revision N
```

Each deployment can create a new revision when configuration or image details change.

This provides deployment history.

For example:

```text
Old Image
    |
    v
Task Definition Revision 15
```

```text
New Image
    |
    v
Task Definition Revision 16
```

The ECS service is then updated to use the desired revision.

---

# 14.17 ECS Service

## What Is an ECS Service?

An ECS Service manages the long-running tasks for an application.

For CitiCore:

```text
ECS Cluster
     |
     v
ECS Service
     |
     v
Desired Number of Tasks
```

For example:

```text
Desired Count = 1
```

ECS attempts to maintain:

```text
Running Tasks = 1
```

If a task fails:

```text
Task Stops
    |
    v
ECS Detects Desired Count Not Met
    |
    v
Replacement Task Started
```

This is one of the main benefits of using an ECS Service rather than manually running a single container.

---

# 14.18 ECS Service and Fargate Tasks

The hierarchy is:

```text
ECS Cluster
     |
     v
ECS Service
     |
     v
Fargate Task
     |
     v
Docker Container
     |
     v
Spring Boot Application
```

Each layer has a different responsibility.

### ECS Cluster

Logical grouping for ECS services and workloads.

### ECS Service

Maintains the desired number of running tasks.

### Fargate Task

The running task environment.

### Container

Runs the application process.

---

# 14.19 What Is AWS Fargate?

AWS Fargate is a serverless compute engine for containers.

Instead of manually managing EC2 instances:

```text
Developer
    |
    v
Manage EC2
    |
    +-- OS
    +-- Capacity
    +-- Patching
    +-- Container Runtime
```

Fargate abstracts much of the underlying infrastructure management.

The application flow becomes:

```text
ECS Service
     |
     v
Fargate
     |
     v
Container Runs
```

In CitiCore, Fargate is used to run containerized services without manually managing the container host servers.

---

# 14.20 Why CitiCore Uses ECS with Fargate

The main advantages are:

* No direct server management
* Integration with ECR
* Integration with IAM
* Integration with VPC
* Integration with ALB
* Integration with CloudWatch
* ECS Service management
* Easier container deployment

The deployment model is:

```text
Docker Image
      |
      v
Amazon ECR
      |
      v
ECS
      |
      v
Fargate
      |
      v
Running Container
```

---

# 14.21 Environment Variables

Applications often require environment-specific configuration.

Examples include:

* Active profile
* Database host
* Kafka configuration
* Redis configuration
* Service configuration

A container image should ideally remain reusable across environments.

Instead of:

```text
Development Docker Image
```

and:

```text
Production Docker Image
```

a better principle is:

```text
Same Application Image
        |
        +------------------+
        |                  |
        v                  v
Development Config    Production Config
```

This separates:

```text
Application Artifact
```

from:

```text
Environment Configuration
```

---

# 14.22 Secrets

Sensitive values should not be stored directly inside:

* Source code
* Docker images
* Git repositories

Examples include:

* Database passwords
* JWT secrets
* API keys
* AWS credentials

The architecture should be:

```text
Application Container
        |
        v
AWS Secret / Secure Configuration
```

For CitiCore, credential and database security are covered in the dedicated Database Security section.

The key deployment principle is:

> Secrets should be injected securely into the running environment rather than baked into the Docker image.

---

# 14.23 IAM Roles

ECS deployments require AWS permissions.

Two important concepts are commonly involved:

```text
ECS Task Execution Role
```

and:

```text
Application Task Role
```

Conceptually:

```text
ECS Task
   |
   +------------------------+
   |                        |
   v                        v
Execution Role           Task Role
```

### Execution Role

Used by ECS for operations required to start and manage the task, such as accessing container images and logging integrations.

### Task Role

Represents permissions available to the application running inside the task.

The important security principle is:

> Give each role only the permissions required for its responsibility.

---

# 14.24 Networking During Deployment

A Fargate task runs inside the AWS networking environment.

The relationship is:

```text
VPC
 |
 v
Subnet
 |
 v
ECS Task
 |
 v
Network Interface
 |
 v
Security Group
```

The task's network configuration determines how it communicates with:

* ALB
* RDS
* Kafka
* Redis
* Other ECS services
* Internet-facing dependencies

The detailed VPC architecture is covered in Topic 12.

Here, the deployment perspective is:

```text
ECS Task
   |
   +--> Security Group Rules
   |
   +--> Subnet
   |
   +--> Network Connectivity
```

---

# 14.25 Application Load Balancer

## Purpose

The Application Load Balancer handles external HTTP traffic.

The simplified flow is:

```text
Client
   |
   v
Application Load Balancer
   |
   v
Target Group
   |
   v
ECS Fargate Task
   |
   v
Spring Boot Application
```

The ALB should not be confused with internal service discovery.

External traffic:

```text
Internet
    |
    v
ALB
    |
    v
Application Service
```

Internal service communication can use:

```text
Service Connect
```

and application-level service discovery mechanisms such as:

```text
Eureka
```

These technologies solve different problems.

---

# 14.26 Target Groups

A target group represents the backend targets that receive traffic from the ALB.

For ECS/Fargate:

```text
ALB
 |
 v
Target Group
 |
 +-------------------+
 |                   |
 v                   v
Fargate Task     Fargate Task
```

The target group performs health checks.

Healthy targets can receive traffic.

```text
Health Check
     |
     +------------------+
     |                  |
 Healthy            Unhealthy
     |                  |
     v                  X
Receive Traffic    Removed from Routing
```

---

# 14.27 Health Checks

Health checks are critical during deployment.

A container being:

```text
RUNNING
```

does not automatically mean the application is healthy.

Different layers must be considered.

```text
Layer 1
Container Running
```

```text
Layer 2
Application Started
```

```text
Layer 3
Health Endpoint Responding
```

```text
Layer 4
ALB Target Healthy
```

```text
Layer 5
Business Function Working
```

These are separate states.

The CitiCore notes highlighted this distinction during troubleshooting.

A system may have:

```text
ECS Task = RUNNING
```

and:

```text
ALB Target = HEALTHY
```

while a business operation still has a problem.

Therefore:

> Infrastructure health and business correctness must be verified separately.

---

# 14.28 Spring Boot Health Endpoint

A typical health verification flow is:

```text
ECS Task
    |
    v
Spring Boot Application
    |
    v
/actuator/health
    |
    v
Health Response
```

For CitiCore, health checks were used as part of verifying deployed services.

A typical successful response is:

```text
HTTP 200
```

with the application reporting an available health state.

The exact Actuator configuration should match the active service configuration.

---

# 14.29 ALB Health Check Flow

The deployment health flow is:

```text
ALB
 |
 v
Target Group
 |
 v
Health Check Request
 |
 v
Fargate Task
 |
 v
Spring Boot Health Endpoint
```

If successful:

```text
Target = Healthy
```

Then:

```text
ALB
 |
 v
Routes Client Traffic
 |
 v
Healthy Task
```

If unhealthy:

```text
Target = Unhealthy
```

The target should not be considered a healthy destination for normal traffic.

---

# 14.30 ECS Service Connect

## What Is Service Connect?

ECS Service Connect provides service-to-service connectivity and discovery capabilities for services running in ECS.

The simplified flow is:

```text
Service A
   |
   v
Service Connect Name
   |
   v
Service B
```

This avoids hardcoding temporary task IP addresses.

Without service discovery:

```text
Service A
   |
   v
Hardcoded Task IP
```

This is unreliable because Fargate task IP addresses can change.

With Service Connect:

```text
Service A
   |
   v
Stable Service Name
   |
   v
Service B
```

---

# 14.31 Service Connect in CitiCore

The CitiCore architecture includes ECS Service Connect for internal ECS communication.

The notes describe the architecture as:

```text
Microservice
     |
     v
Service Connect DNS
     |
     v
Another ECS Service
```

This is particularly useful because container tasks are dynamic.

The key principle is:

> Never rely on ephemeral Fargate task IP addresses for service-to-service communication.

---

# 14.32 Eureka vs Service Connect in Deployment

CitiCore contains both:

```text
ECS Service Connect
```

and:

```text
Eureka
```

They operate at different layers.

```text
Infrastructure-Level Service Connectivity
                |
                v
       ECS Service Connect
```

```text
Application-Level Service Registry
                |
                v
              Eureka
```

The deployment architecture can therefore be understood as:

```text
ECS Layer
    |
    +--> Service Connect
    |
    +--> Networking
    |
    +--> Task Management


Application Layer
    |
    +--> Eureka
    |
    +--> Spring Cloud
    |
    +--> Service Discovery
```

This distinction is useful during interviews.

---

# 14.33 ECS Deployment Flow

The general deployment process is:

```text
1. Build Application
        |
        v
2. Build Docker Image
        |
        v
3. Push Image to ECR
        |
        v
4. Update Task Definition
        |
        v
5. Register New Revision
        |
        v
6. Update ECS Service
        |
        v
7. ECS Starts New Task
        |
        v
8. Health Checks
        |
        v
9. New Task Receives Traffic
        |
        v
10. Old Task Stops
```

This is the core deployment lifecycle for CitiCore's container-based services.

---

# 14.34 Rolling Deployment

When a new version is deployed:

```text
Old Version
     |
     v
Running ECS Task
```

A new task definition is registered:

```text
New Version
     |
     v
New ECS Task Definition
```

Then:

```text
ECS Service Update
        |
        v
Start New Task
        |
        v
Health Check
        |
        v
Route Traffic
        |
        v
Drain Old Task
        |
        v
Stop Old Task
```

This is the general rolling deployment concept used by ECS services.

The exact deployment behavior depends on the ECS deployment configuration.

---

# 14.35 CitiCore Deployment Example

A documented CitiCore deployment flow follows this pattern:

```text
Code Change
    |
    v
Git Commit
    |
    v
Jenkins Triggered
    |
    v
Maven Build
    |
    v
Docker Build
    |
    v
Git SHA Image Tag
    |
    v
Push Image to ECR
    |
    v
Register ECS Task Definition Revision
    |
    v
Update ECS Service
    |
    v
ECS Rolling Deployment
```

For example:

```text
Account Service Changed
        |
        v
Build New Account Service Image
        |
        v
Push to ECR
        |
        v
Register New Task Definition Revision
        |
        v
Update Account ECS Service
```

The CI/CD automation details are covered separately in Topic 15.

---

# 14.36 Deployment Verification

A deployment should not be considered successful immediately after an update command is sent.

Multiple layers should be verified.

## ECS Service

```text
Desired Count
```

should match the expected deployment configuration.

```text
Running Count
```

should reach the desired state.

```text
Pending Count
```

should eventually reach zero when the deployment stabilizes.

---

## Task Definition

Verify that the ECS service points to the expected task definition revision.

Conceptually:

```text
Expected Task Definition
          |
          v
ECS Service
          |
          v
Actual Task Definition
```

These should match.

---

## Target Health

Verify the ALB target group.

```text
ALB
 |
 v
Target Group
 |
 v
Target Health
```

The target should become healthy before it is considered ready for traffic.

---

## Application Health

Verify:

```text
/actuator/health
```

The service should respond successfully.

---

## Logs

Check:

```text
CloudWatch Logs
```

Look for:

* Startup failures
* Dependency connection errors
* Configuration errors
* Application exceptions

---

# 14.37 Important Deployment Checklist

## Before Deployment

```text
[ ] Application builds successfully
[ ] Tests pass
[ ] Docker image builds successfully
[ ] Correct image tag created
[ ] ECR authentication works
[ ] Image pushed successfully
```

## ECS Deployment

```text
[ ] Task definition registered
[ ] Correct image referenced
[ ] ECS service updated
[ ] Desired count correct
[ ] New task starts
```

## Networking

```text
[ ] Correct VPC
[ ] Correct subnets
[ ] Security groups configured
[ ] ALB reachable where required
[ ] Target group configured
```

## Health

```text
[ ] ECS task running
[ ] Target healthy
[ ] /actuator/health works
[ ] CloudWatch logs reviewed
```

## Dependencies

```text
[ ] Config Server reachable
[ ] Eureka reachable
[ ] Database connectivity verified
[ ] Kafka connectivity verified
[ ] Redis connectivity verified
```

---

# 14.38 Failure Handling During Deployment

Deployment failures can happen at different stages.

```text
Build Failure
    |
    v
No Image
```

```text
Image Push Failure
    |
    v
No New Image in ECR
```

```text
Task Startup Failure
    |
    v
Application Does Not Run
```

```text
Health Check Failure
    |
    v
Target Remains Unhealthy
```

```text
Runtime Failure
    |
    v
Business Operation Fails
```

Each layer should be investigated separately.

---

# 14.39 Real CitiCore Troubleshooting Lesson

A major lesson documented during CitiCore deployment was:

> An HTTP timeout does not necessarily mean the business operation failed.

The troubleshooting evidence showed a situation where a registration request appeared to time out, but subsequent behavior indicated that the database write had completed.

Conceptually:

```text
Client Request
      |
      v
Business Operation Starts
      |
      v
Database Write Completes
      |
      v
Downstream Operation Delayed
      |
      v
Client Timeout
```

The client may see:

```text
504 Timeout
```

but the business operation may already have partially or fully completed.

The investigation should compare:

```text
Client Response
        +
Application Logs
        +
Database State
        +
Downstream Service State
        +
ALB Status
        +
Network / DNS
```

This is an important distributed-systems lesson.

---

# 14.40 Deployment Layer Mental Model

CitiCore deployment can be understood in layers.

## Layer 1 — Container

```text
Docker Image
      |
      v
Fargate Container
```

## Layer 2 — ECS

```text
Task Definition
      |
      v
ECS Service
      |
      v
Fargate Task
```

## Layer 3 — Network

```text
VPC
 |
 v
Subnet
 |
 v
Network Interface
 |
 v
Security Group
```

## Layer 4 — External Traffic

```text
Internet
 |
 v
ALB
 |
 v
Target Group
 |
 v
Fargate Task
```

## Layer 5 — Internal Traffic

```text
Service
 |
 v
Service Connect
 |
 v
Another Service
```

## Layer 6 — Application Discovery

```text
Microservice
 |
 v
Eureka
 |
 v
Service Registry
```

## Layer 7 — Configuration

```text
Configuration Repository
 |
 v
Config Server
 |
 v
Microservices
```

## Layer 8 — Events

```text
Microservice
 |
 v
Kafka
 |
 v
Consumer Service
```

This layered model is useful when debugging because it prevents treating the entire system as a single component.

---

# 14.41 Rollback

Rollback means returning the service to a previously known stable application version.

CitiCore uses versioned Docker images and deployment history to support rollback.

The basic process is:

```text
New Deployment
      |
      v
Problem Detected
      |
      v
Identify Known-Good Version
      |
      v
Verify Image Exists in ECR
      |
      v
Deploy Previous Version
      |
      v
Verify ECS Service
      |
      v
Verify Application
```

The Git SHA-based tagging strategy is important here.

```text
Known-Good Commit
        |
        v
Known-Good Image Tag
        |
        v
Image in ECR
        |
        v
Redeploy
```

---

# 14.42 CitiCore Rollback Approach

The documented CitiCore approach uses a manual rollback process through Jenkins.

The general flow is:

```text
Deployment Issue
       |
       v
Operator Investigation
       |
       v
Identify Previous Stable Git SHA
       |
       v
Verify ECR Image
       |
       v
Trigger Rollback
       |
       v
Deploy Previous Image
       |
       v
Verify ECS
       |
       v
Test API
```

The documented reasoning for manual rollback is that rollback should be a deliberate operational decision rather than blindly switching versions.

This provides:

* Human verification
* Controlled recovery
* Better incident investigation
* Clear audit trail

The complete Jenkins implementation is covered in Topic 15.

---

# 14.43 Health Verification vs Deployment Trigger

An important distinction in the CitiCore pipeline is:

```text
Deployment Triggered
```

is not the same as:

```text
Deployment Fully Stable
```

The pipeline can verify that:

```text
ECS Service
    |
    v
Points to Expected Task Definition
```

while ECS continues the rolling deployment asynchronously.

The stages are:

```text
Pipeline
   |
   v
Register Task Definition
   |
   v
Update ECS Service
   |
   v
ECS Accepts New Revision
   |
   v
Pipeline Verifies Request
```

Then ECS continues:

```text
Start New Task
   |
   v
Application Startup
   |
   v
Health Checks
   |
   v
Traffic Transition
   |
   v
Old Task Drained
```

This distinction prevents incorrectly assuming that a successful API call to ECS means the entire deployment lifecycle has already completed.

---

# 14.44 Production Considerations

A real production deployment should consider more than simply starting containers.

Important areas include:

## High Availability

Run appropriate service replicas across available infrastructure.

```text
ALB
 |
 +-------------------+
 |                   |
 v                   v
Task 1             Task 2
```

The exact scaling configuration should depend on actual workload requirements.

---

## Health Checks

Use meaningful health checks.

```text
Container Running
```

alone is not enough.

---

## Logging

Centralize logs.

```text
Fargate Task
     |
     v
CloudWatch Logs
```

Logs should support investigation of:

* Startup failures
* Application exceptions
* Dependency failures

---

## Secrets

Sensitive credentials should be securely managed.

```text
Application
    |
    v
Secure Secret Source
```

They should not be baked into images.

---

## Image Versioning

Avoid ambiguous production deployment versions.

Prefer:

```text
service:<immutable-version>
```

over relying only on:

```text
service:latest
```

---

## Rollback

A previously known-good version should be identifiable and deployable.

---

# 14.45 Trade-offs

## Advantages of ECS/Fargate

### Reduced Server Management

The team does not need to manually manage container host servers.

### AWS Integration

It integrates with:

* ECR
* IAM
* VPC
* ALB
* CloudWatch

### Service Management

ECS services maintain desired task counts.

### Container-Based Deployment

Each microservice can be independently packaged and deployed.

---

## Disadvantages

### AWS-Specific Operational Knowledge

Teams need to understand:

* ECS
* IAM
* Networking
* Task definitions
* Security groups
* Load balancers

### Deployment Complexity

A microservices environment contains multiple deployment layers.

```text
Application
   |
Docker
   |
ECR
   |
ECS
   |
Network
   |
ALB
   |
Dependencies
```

### Debugging Across Layers

A problem may exist in:

* Application code
* Container configuration
* ECS configuration
* Security groups
* DNS
* ALB
* Service Connect

This requires a structured troubleshooting approach.

---

# 14.46 How I Would Explain AWS Container Deployment in an Interview

> "For CitiCore, I containerized the Spring Boot microservices using Docker and deployed them on AWS ECS with Fargate. The application is first packaged as a JAR, then built into a Docker image. The image is stored in Amazon ECR. ECS uses a task definition to define how the container should run, including the image, resources, ports, logging, and environment configuration. An ECS service manages the running tasks, while Fargate provides the serverless container runtime. For external traffic, we use an Application Load Balancer and target groups with health checks. For internal ECS communication, the architecture also uses Service Connect. Our deployment flow uses versioned images and ECS task definition revisions, which also supports controlled rollback."

---

# 14.47 30–60 Second Interview Answer

> "In CitiCore, the deployment flow starts with building the Spring Boot service using Maven and packaging it into a Docker image. The image is tagged with a Git SHA for traceability and pushed to Amazon ECR. We then use ECS task definitions to specify the container image and runtime configuration, and ECS services run the containers on Fargate. External traffic goes through an ALB and target group, while health checks verify that the service is ready. The deployment uses new task definition revisions, and previous versioned images can be used for rollback."

---

# 14.48 Interview Follow-up Questions

## Q1. What is the difference between ECR and ECS?

**Answer:**

ECR is a container image registry used to store Docker images. ECS is a container orchestration service used to deploy and manage containers.

```text
ECR = Stores Images

ECS = Runs and Manages Containers
```

---

## Q2. What is Fargate?

**Answer:**

Fargate is a serverless compute option for running containers. It allows us to run ECS tasks without directly managing EC2 instances.

---

## Q3. What is an ECS Task Definition?

**Answer:**

A task definition is a blueprint that describes how a container should run, including the Docker image, CPU, memory, ports, environment variables, logging, and IAM configuration.

---

## Q4. What is an ECS Service?

**Answer:**

An ECS service manages long-running tasks and maintains the desired number of task instances.

---

## Q5. What happens if an ECS task fails?

**Answer:**

If the task is managed by an ECS service, ECS attempts to maintain the configured desired count by starting replacement tasks, subject to the service and deployment configuration.

---

## Q6. How does ECS deploy a new application version?

**Answer:**

A new container image is pushed to ECR, a new task definition revision referencing the image is registered, and the ECS service is updated to use that revision.

---

## Q7. Why did you use Git SHA tags?

**Answer:**

Git SHA tags provide an immutable link between the source code commit and the deployed container image. This improves traceability and rollback.

---

## Q8. What is the role of the ALB?

**Answer:**

The ALB receives external HTTP traffic and routes it through target groups to healthy application targets.

---

## Q9. What is the difference between an ECS task being RUNNING and an application being healthy?

**Answer:**

RUNNING means the ECS task/container is active. Application health requires additional verification, such as a successful health endpoint and healthy load balancer target status.

---

## Q10. What is Service Connect?

**Answer:**

Service Connect helps ECS services communicate internally using service-based connectivity rather than relying on changing task IP addresses.

---

## Q11. How do you troubleshoot a failed deployment?

**Answer:**

I investigate layer by layer:

```text
Image
  |
  v
ECS Task
  |
  v
Application Logs
  |
  v
Health Checks
  |
  v
Networking
  |
  v
Dependencies
```

I check the task status, CloudWatch logs, target health, security groups, DNS/service connectivity, and downstream dependencies.

---

## Q12. How do you roll back an ECS deployment?

**Answer:**

I identify a previously known-good image version, verify it exists in ECR, deploy it through the controlled deployment process, and verify the ECS service and application health.

---

# 14.49 Key Takeaways

The CitiCore AWS deployment architecture follows this core path:

```text
Spring Boot
     |
     v
Maven
     |
     v
JAR
     |
     v
Docker Image
     |
     v
Amazon ECR
     |
     v
ECS Task Definition
     |
     v
ECS Service
     |
     v
AWS Fargate
```

External traffic:

```text
Client
   |
   v
ALB
   |
   v
Target Group
   |
   v
Fargate Task
```

Internal service connectivity:

```text
Service
   |
   v
Service Connect
   |
   v
Service
```

Application-level discovery:

```text
Microservice
   |
   v
Eureka
```

Deployment traceability:

```text
Git Commit
    |
    v
Git SHA
    |
    v
Docker Image
    |
    v
ECR
    |
    v
Task Definition Revision
    |
    v
ECS Service
```

The most important practical lessons are:

* Package services as immutable Docker images.
* Store deployment images in ECR.
* Use ECS task definitions as container runtime blueprints.
* Use ECS services to manage long-running tasks.
* Use Fargate to run containers without managing servers.
* Use ALB and target groups for external traffic.
* Treat task status, target health, and business correctness as separate checks.
* Use Service Connect for internal ECS connectivity.
* Avoid hardcoding Fargate task IP addresses.
* Use immutable image tags for traceability and rollback.
* Verify deployments across infrastructure and application layers.

---

# 14.50 Final Deployment Architecture Summary

```text
                   CITICORE AWS DEPLOYMENT

Developer / Jenkins
        |
        v
   Maven Build
        |
        v
   Spring Boot JAR
        |
        v
   Docker Build
        |
        v
Docker Image (Git SHA)
        |
        v
     Amazon ECR
        |
        v
 ECS Task Definition
        |
        v
    ECS Service
        |
        v
   AWS Fargate
        |
        v
 Spring Boot Container
        |
        +-------------------------+
        |                         |
        v                         v
      ALB                  Service Connect
        |                         |
        v                         v
External Clients          Internal Services
        |
        v
   Target Group
        |
        v
   Health Checks
```

> **In CitiCore, AWS container deployment provides the bridge between locally containerized Spring Boot microservices and a managed cloud runtime using ECR, ECS, Fargate, ALB, Service Connect, IAM, and CloudWatch.**

---

---
## TOPIC 15. Jenkins CI/CD

## 15.1 What Is CI/CD?

## Concept

**CI/CD** stands for:

* **CI — Continuous Integration**
* **CD — Continuous Delivery or Continuous Deployment**

Continuous Integration means automatically building and validating code changes when they are integrated into the project.

Continuous Delivery/Deployment extends this process by automating the delivery of the application to an environment.

For CitiCore, the CI/CD flow connects the complete deployment lifecycle:

```text
Developer
    |
    v
Git Push
    |
    v
Jenkins Pipeline
    |
    +--> Checkout Code
    |
    +--> Build
    |
    +--> Test
    |
    +--> Docker Build
    |
    +--> Push to ECR
    |
    +--> Update ECS
    |
    +--> Verify Deployment
```

The main purpose is to reduce manual deployment work and make deployments more consistent and traceable.

---

# 15.2 Why Do We Need CI/CD?

Without CI/CD, deployment can become a manual process.

For example:

```text
Developer Changes Code
        |
        v
Manual Build
        |
        v
Manual Docker Build
        |
        v
Manual Image Push
        |
        v
Manual ECS Update
```

This introduces risks such as:

* Forgetting a deployment step
* Deploying the wrong image
* Human configuration mistakes
* Inconsistent deployments
* Difficulty tracking versions
* Slow release processes

With CI/CD:

```text
Code Change
    |
    v
Automated Pipeline
    |
    v
Consistent Deployment Steps
```

The pipeline acts as a repeatable deployment process.

---

# 15.3 CitiCore Context

## CitiCore Implementation

For CitiCore, Jenkins is used to automate the build and deployment workflow for containerized Spring Boot microservices.

The overall pipeline is:

```text
Git Repository
      |
      v
Jenkins
      |
      +--> Checkout
      |
      +--> Maven Build
      |
      +--> Tests
      |
      +--> Docker Build
      |
      +--> Tag Image
      |
      +--> Push to Amazon ECR
      |
      +--> Register ECS Task Definition
      |
      +--> Update ECS Service
      |
      +--> Verify Deployment
```

The pipeline uses the Git commit SHA as part of the Docker image versioning strategy.

Conceptually:

```text
Git Commit
    |
    v
Commit SHA
    |
    v
Docker Image Tag
    |
    v
Amazon ECR
    |
    v
ECS Deployment
```

This makes it possible to trace a deployed application image back to a specific source code version.

---

# 15.4 Jenkins Architecture

## Concept

Jenkins uses a controller-agent architecture.

```text
                    Jenkins Controller
                           |
              +------------+------------+
              |                         |
              v                         v
       Jenkins Agent A           Jenkins Agent B
```

The controller manages:

* Jobs
* Pipelines
* Build scheduling
* Configuration
* Pipeline orchestration

Agents perform build work.

For example:

```text
Jenkins Controller
        |
        v
Java / Maven Agent
        |
        +--> Compile
        |
        +--> Test
        |
        +--> Package
```

Another agent may perform container-related work.

```text
Jenkins Controller
        |
        v
Docker Agent
        |
        +--> Build Image
        |
        +--> Push Image
```

---

# 15.5 CitiCore Jenkins Setup

## CitiCore Implementation

During the CitiCore learning and deployment work, Jenkins pipelines were moved toward using Jenkins agents for build execution.

The project setup included:

* Jenkins controller
* Build agents
* Java
* Maven
* Git
* Docker
* AWS CLI
* AWS credentials
* Amazon ECR access
* Amazon ECS access

The general architecture is:

```text
                    Jenkins Controller
                           |
                           v
                    Jenkins Agent
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
         Git             Maven            Docker
                                              |
                                              v
                                           AWS CLI
                                              |
                                  +-----------+-----------+
                                  |                       |
                                  v                       v
                                 ECR                     ECS
```

The important concept is that Jenkins orchestrates the workflow while agents perform the actual pipeline work.

---

# 15.6 Jenkins Controller

The Jenkins controller is responsible for coordinating the CI/CD process.

Its responsibilities include:

* Managing pipeline definitions
* Scheduling builds
* Assigning work to agents
* Maintaining job configuration
* Storing build history
* Displaying pipeline results

Conceptually:

```text
Developer Push
      |
      v
Jenkins Controller
      |
      +--> Select Pipeline
      |
      +--> Allocate Agent
      |
      +--> Monitor Execution
      |
      +--> Store Build Result
```

The controller should not unnecessarily perform all heavy build operations in larger production environments.

Instead:

```text
Controller
    |
    v
Agent
    |
    v
Build Work
```

This improves scalability and separation of responsibilities.

---

# 15.7 Jenkins Agent

A Jenkins agent is a machine or runtime environment that executes pipeline steps.

For CitiCore, an agent requires the tools needed by the pipeline.

For example:

```text
Jenkins Agent
     |
     +--> Java
     |
     +--> Maven
     |
     +--> Git
     |
     +--> Docker
     |
     +--> AWS CLI
```

If a pipeline stage requires Docker but Docker is unavailable on the agent:

```text
Pipeline
   |
   v
Docker Build
   |
   X
Docker Not Available
```

Therefore, Jenkins agent configuration is an important part of CI/CD reliability.

---

# 15.8 Why Use Jenkins Agents?

Agents provide several benefits.

## Workload Separation

```text
Controller
    |
    v
Pipeline Coordination
```

```text
Agent
    |
    v
Build Execution
```

## Tool Isolation

Different agents can contain different tools.

For example:

```text
Java Agent
    |
    +--> Java
    +--> Maven
```

```text
Docker Agent
    |
    +--> Docker
    +--> AWS CLI
```

## Scalability

Multiple builds can potentially run on multiple agents.

```text
                Jenkins Controller
                       |
          +------------+------------+
          |                         |
          v                         v
       Agent 1                  Agent 2
       Build A                  Build B
```

---

# 15.9 Jenkins Pipeline

## Concept

A Jenkins Pipeline defines the stages required to build, test, and deploy an application.

A simplified CitiCore pipeline looks like:

```text
Checkout
    |
    v
Build
    |
    v
Test
    |
    v
Docker Build
    |
    v
ECR Push
    |
    v
ECS Deployment
    |
    v
Verification
```

The pipeline should be defined as code whenever possible.

This is commonly stored in a:

```text
Jenkinsfile
```

The main advantage is that the deployment workflow becomes version-controlled along with the application.

---

# 15.10 Pipeline as Code

Without Pipeline as Code:

```text
Jenkins UI
   |
   v
Manual Pipeline Configuration
```

The configuration may be difficult to track historically.

With a Jenkinsfile:

```text
Git Repository
      |
      +--> Application Code
      |
      +--> Jenkinsfile
```

The pipeline can evolve with the application.

Benefits include:

* Version control
* Code review
* Reproducibility
* Easier migration
* Clear deployment documentation

---

# 15.11 CitiCore Pipeline Stages

The logical CitiCore pipeline consists of the following stages.

```text
1. Checkout
       |
       v
2. Build
       |
       v
3. Test
       |
       v
4. Docker Build
       |
       v
5. Tag Image
       |
       v
6. Push to ECR
       |
       v
7. Register Task Definition
       |
       v
8. Update ECS Service
       |
       v
9. Verify Deployment
```

Each stage has a specific responsibility.

---

# 15.12 Stage 1 — Git Checkout

The pipeline starts by retrieving the source code.

```text
Git Repository
       |
       v
Jenkins Agent
       |
       v
Workspace
```

The Jenkins workspace contains the code used for the current build.

For example:

```text
Jenkins Workspace
      |
      +--> pom.xml
      |
      +--> src/
      |
      +--> Dockerfile
      |
      +--> Jenkinsfile
```

The workspace should not be confused with the final Docker image.

The workspace is the build environment where Jenkins performs the pipeline operations.

---

# 15.13 Jenkins Workspace

## What Is It?

A Jenkins workspace is the directory where Jenkins checks out and builds the project.

Conceptually:

```text
Jenkins Agent
      |
      v
Workspace
      |
      +--> Source Code
      |
      +--> Build Output
      |
      +--> Temporary Files
```

For CitiCore, the Maven build and Docker build are executed using the application source available in the workspace.

---

# 15.14 Stage 2 — Maven Build

The Spring Boot application must be compiled and packaged.

The typical flow is:

```text
Java Source Code
       |
       v
Maven
       |
       +--> Compile
       |
       +--> Test
       |
       +--> Package
       |
       v
Application JAR
```

A typical command is:

```bash
mvn clean package
```

Depending on the pipeline structure, tests may be executed as part of this command or in a separate stage.

The important output is the packaged application artifact.

---

# 15.15 Stage 3 — Testing

Testing should validate the application before deployment.

Conceptually:

```text
Source Code
    |
    v
Build
    |
    v
Tests
    |
    +--> Pass
    |      |
    |      v
    |   Continue
    |
    +--> Fail
           |
           v
      Stop Pipeline
```

The basic principle is:

> Do not deploy an application that has failed the pipeline's required validation stages.

For a production-grade pipeline, testing may include:

* Unit tests
* Integration tests
* Static analysis
* Security scanning

The exact set of tests depends on the project's implementation.

---

# 15.16 CitiCore Testing Context

The documented CitiCore pipeline focused primarily on the build and deployment lifecycle.

The project notes contain Maven build and deployment automation work.

Where additional automated quality gates are introduced, they should be clearly distinguished from the currently implemented pipeline.

For example:

```text
Current Pipeline
    |
    +--> Build
    |
    +--> Test
    |
    +--> Docker
    |
    +--> Deploy
```

A future improvement could include:

```text
Future Pipeline
    |
    +--> SonarQube
    |
    +--> Security Scan
    |
    +--> Integration Test Environment
```

These should not be presented as already implemented unless supported by the CitiCore implementation notes.

---

# 15.17 Stage 4 — Docker Build

After the application is packaged, Jenkins builds a Docker image.

```text
Spring Boot JAR
       |
       v
Dockerfile
       |
       v
Docker Build
       |
       v
Container Image
```

Conceptually:

```bash
docker build -t account-service .
```

In CI/CD, the image name should include version information.

For CitiCore, Git SHA-based tagging is used.

---

# 15.18 Docker Image Tagging

The pipeline retrieves the Git commit identifier.

```text
Git Repository
       |
       v
Current Commit
       |
       v
Commit SHA
```

The SHA is used as an image tag.

Example:

```text
account-service:434a66e
```

This is better for traceability than:

```text
account-service:latest
```

because the image can be connected to a specific code version.

The complete relationship is:

```text
Git Commit
      |
      v
Git SHA
      |
      v
Docker Image Tag
      |
      v
ECR Image
      |
      v
ECS Deployment
```

---

# 15.19 Why Git SHA Is Useful

Suppose a deployment causes a problem.

```text
Image
   |
   v
account-service:434a66e
```

The team can identify:

```text
434a66e
   |
   v
Exact Git Commit
```

This helps answer:

* Which code version was deployed?
* What changed?
* Which version was stable?
* Which image should be rolled back to?

This improves deployment traceability.

---

# 15.20 Stage 5 — Authenticate With Amazon ECR

Before Jenkins can push the Docker image, Docker must authenticate with Amazon ECR.

The logical flow is:

```text
Jenkins
   |
   v
AWS CLI
   |
   v
ECR Authentication Token
   |
   v
Docker Login
   |
   v
Authorized ECR Push
```

The AWS credentials should be provided securely.

They should not be hardcoded in:

```text
Jenkinsfile
```

or:

```text
Application Source Code
```

---

# 15.21 Stage 6 — Push Image to ECR

After building and tagging the image:

```text
Docker Image
     |
     v
Amazon ECR
```

The pipeline flow is:

```text
Docker Build
      |
      v
Tag Image
      |
      v
Docker Push
      |
      v
Amazon ECR Repository
```

Once the image is available in ECR, ECS can retrieve it when starting a task.

---

# 15.22 ECR Deployment Flow

```text
Jenkins
   |
   v
Build Docker Image
   |
   v
Tag With Git SHA
   |
   v
Authenticate to ECR
   |
   v
Push Image
   |
   v
ECR Repository
```

Then:

```text
ECS Task Definition
        |
        v
Pulls Image
        |
        v
Fargate Task
```

---

# 15.23 Stage 7 — ECS Task Definition Update

The ECS task definition must reference the new container image.

Conceptually:

```text
Old Task Definition
       |
       v
Old Image Tag
```

A new deployment requires:

```text
New Task Definition Revision
       |
       v
New Image Tag
```

The pipeline can create a new revision using the updated image information.

```text
New ECR Image
      |
      v
Task Definition Revision
      |
      v
ECS Service Update
```

---

# 15.24 Task Definition Revisioning

ECS task definitions are revisioned.

```text
account-service
       |
       +--> Revision 1
       |
       +--> Revision 2
       |
       +--> Revision 3
       |
       +--> Revision N
```

Each revision can contain updated configuration or a different image.

This supports deployment history and controlled rollback.

---

# 15.25 Stage 8 — Update ECS Service

After registering the new task definition revision, Jenkins updates the ECS service.

```text
New Task Definition
        |
        v
Update ECS Service
        |
        v
ECS Deployment Starts
```

The ECS service then manages the rollout.

Conceptually:

```text
Old Task
    |
    v
New Task Starts
    |
    v
Health Check
    |
    v
New Task Receives Traffic
    |
    v
Old Task Stops
```

The exact rollout behavior depends on ECS deployment configuration.

---

# 15.26 Important Jenkins Deployment Principle

A successful ECS update command does not necessarily mean the application is fully deployed and healthy.

For example:

```text
Jenkins
   |
   v
Update ECS Service
   |
   v
AWS Accepts Request
```

At this point:

```text
Deployment Started
```

But ECS still needs to:

```text
Start Task
    |
    v
Pull Image
    |
    v
Start Application
    |
    v
Pass Health Checks
    |
    v
Become Stable
```

Therefore, deployment verification is important.

---

# 15.27 Stage 9 — Deployment Verification

A successful pipeline should verify multiple states.

```text
ECS Service
     |
     v
Desired Task Definition
```

```text
ECS Tasks
     |
     v
Expected Running State
```

```text
Target Group
     |
     v
Healthy Targets
```

```text
Application
     |
     v
Health Endpoint
```

The complete verification flow is:

```text
Pipeline Deployment
       |
       v
ECS Service Updated
       |
       v
New Task Started
       |
       v
Task Running
       |
       v
Application Started
       |
       v
Health Check
       |
       v
Target Healthy
```

---

# 15.28 Health Verification

For Spring Boot services, a health endpoint can be used.

Conceptually:

```text
Jenkins
   |
   v
Application Health Endpoint
   |
   v
HTTP Response
```

A successful deployment should verify more than:

```text
Task = RUNNING
```

The application should also be:

```text
Application = HEALTHY
```

These are different states.

---

# 15.29 Deployment Verification vs Business Verification

This is an important distinction.

Infrastructure verification:

```text
ECS Task Running
```

Application verification:

```text
Health Endpoint Responding
```

Business verification:

```text
Business API Working Correctly
```

The complete confidence chain is:

```text
Container Running
        |
        v
Application Healthy
        |
        v
Dependencies Available
        |
        v
Business Function Working
```

A deployment can succeed at one level while still failing at another.

---

# 15.30 Rollback

## Concept

Rollback means returning the service to a previously known stable version.

The basic flow is:

```text
New Deployment
      |
      v
Problem Detected
      |
      v
Identify Previous Stable Version
      |
      v
Deploy Previous Image
      |
      v
Verify Service
```

---

# 15.31 CitiCore Rollback Approach

The CitiCore deployment notes describe a controlled rollback approach.

The general process is:

```text
Deployment Problem
       |
       v
Identify Stable Git SHA
       |
       v
Verify Image Exists in ECR
       |
       v
Select Previous Version
       |
       v
Deploy Through Jenkins
       |
       v
Verify ECS Service
       |
       v
Verify Application
```

Rollback should use a known image version rather than guessing which version was previously running.

This is one reason immutable image tags are important.

---

# 15.32 Why Manual Rollback Can Be Useful

CitiCore's documented rollback approach is deliberately controlled.

Automatic rollback may be useful in some environments, but it can also hide the root cause or trigger repeated deployment behavior.

A controlled rollback process allows:

```text
Failure
   |
   v
Investigation
   |
   v
Decision
   |
   v
Rollback
```

This provides:

* Human confirmation
* Better incident analysis
* Controlled recovery
* Clear operational decisions

---

# 15.33 Complete CitiCore CI/CD Flow

The complete pipeline can be visualized as:

```text
Developer
    |
    v
Git Push
    |
    v
Jenkins Trigger
    |
    v
Checkout Source Code
    |
    v
Maven Build
    |
    v
Run Tests
    |
    v
Create JAR
    |
    v
Docker Build
    |
    v
Get Git SHA
    |
    v
Tag Docker Image
    |
    v
Authenticate to ECR
    |
    v
Push Image to ECR
    |
    v
Create Task Definition Revision
    |
    v
Update ECS Service
    |
    v
ECS Rolling Deployment
    |
    v
Health Verification
    |
    v
Deployment Complete
```

---

# 15.34 Example Pipeline Structure

The exact Jenkinsfile should remain aligned with the project's actual implementation.

A simplified structure is:

```groovy
pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                // Retrieve source code
            }
        }

        stage('Build') {
            steps {
                // Run Maven build
            }
        }

        stage('Test') {
            steps {
                // Run application tests
            }
        }

        stage('Docker Build') {
            steps {
                // Build Docker image
            }
        }

        stage('Push to ECR') {
            steps {
                // Authenticate and push image
            }
        }

        stage('Deploy to ECS') {
            steps {
                // Register task definition
                // Update ECS service
            }
        }

        stage('Verify') {
            steps {
                // Verify deployment
            }
        }
    }
}
```

The purpose of this example is to show the pipeline structure rather than reproduce a large Jenkinsfile.

---

# 15.35 Important Jenkins Credentials

The pipeline may require access to:

* Git repository
* AWS
* Amazon ECR
* Amazon ECS

Credentials should be managed securely.

Avoid:

```text
Jenkinsfile
    |
    +--> Hardcoded Password
```

Prefer:

```text
Jenkins Credentials
        |
        v
Pipeline
```

The principle is:

> Sensitive credentials should be stored in a secure credential management mechanism and injected only when required.

---

# 15.36 AWS Permissions for Jenkins

Jenkins needs appropriate AWS permissions to perform deployment actions.

Depending on the pipeline responsibilities, it may require access to:

```text
ECR
 |
 +--> Authenticate
 |
 +--> Push Image
```

and:

```text
ECS
 |
 +--> Register Task Definition
 |
 +--> Update Service
 |
 +--> Describe Deployment
```

The permissions should follow least privilege.

```text
Jenkins IAM Identity
        |
        v
Only Required Permissions
```

Avoid giving unnecessary administrative access.

---

# 15.37 Failure Handling in the Pipeline

A CI/CD pipeline should stop when a critical stage fails.

For example:

```text
Build
 |
 X
Failure
 |
 v
Pipeline Stops
```

The deployment should not continue to:

```text
Docker Build
```

if the application did not build successfully.

Similarly:

```text
Tests
 |
 X
Failure
 |
 v
Stop Pipeline
```

The pipeline stages create a validation sequence.

```text
Build
   |
   v
Test
   |
   v
Package
   |
   v
Deploy
```

Each stage provides confidence before the next stage begins.

---

# 15.38 Failure Scenario: Docker Build Failure

## Problem

The application builds successfully, but the Docker image build fails.

```text
Maven Build
    |
    v
Success
    |
    v
Docker Build
    |
    X
Failure
```

## Investigation

Check:

* Dockerfile
* Build context
* JAR path
* Docker daemon availability
* Agent permissions

## Solution

Ensure:

```text
Jenkins Workspace
        |
        +--> Dockerfile
        |
        +--> Required Application Artifact
```

The Docker build context must contain the required files.

---

# 15.39 Failure Scenario: Docker Unavailable on Jenkins Agent

## Problem

The pipeline reaches:

```text
Docker Build
```

but the agent cannot execute Docker.

## Root Cause

The Jenkins agent does not have:

* Docker installed
* Docker daemon access
* Required permissions

## Solution

The agent must be configured with Docker access.

```text
Jenkins Agent
       |
       v
Docker CLI
       |
       v
Docker Runtime
```

This was one of the important practical areas encountered during Jenkins and Docker setup work.

---

# 15.40 Failure Scenario: ECR Push Failure

## Problem

The Docker image builds successfully but cannot be pushed to ECR.

```text
Docker Image
     |
     v
ECR Push
     |
     X
Access Denied / Authentication Failure
```

## Investigation

Check:

* AWS credentials
* AWS region
* ECR repository
* IAM permissions
* ECR login

## Solution

Ensure:

```text
Jenkins
   |
   v
Valid AWS Credentials
   |
   v
ECR Authentication
   |
   v
Correct Repository
```

---

# 15.41 Failure Scenario: ECS Deployment Starts but Application Fails

## Problem

Jenkins successfully updates ECS.

```text
Jenkins
   |
   v
ECS Update = Success
```

But:

```text
Fargate Task
   |
   X
Application Fails
```

## Investigation

Check:

* ECS stopped task reason
* CloudWatch logs
* Environment variables
* Secrets
* Database connectivity
* Config Server
* Eureka
* Kafka
* Redis

## Important Lesson

A successful deployment API call is not the same as a successful application deployment.

---

# 15.42 Failure Scenario: Task Running but ALB Target Unhealthy

## Problem

```text
ECS Task = RUNNING
```

but:

```text
ALB Target = UNHEALTHY
```

## Possible Causes

* Wrong health check path
* Incorrect port
* Application not fully started
* Security group issue
* Health endpoint unavailable

## Investigation

```text
ALB Target Health
       |
       v
Health Check Configuration
       |
       v
Application Logs
       |
       v
Application Endpoint
```

## Solution

Ensure the health check configuration matches the actual running application.

---

# 15.43 Real Practical Lesson: Verify More Than One Layer

One of the most important lessons from CitiCore deployment work is that distributed applications must be verified across multiple layers.

```text
Jenkins Pipeline
      |
      v
ECS Service
      |
      v
Fargate Task
      |
      v
Application
      |
      v
Load Balancer
      |
      v
Business API
```

A problem at any layer can make the overall system appear unavailable.

Therefore, troubleshooting should proceed layer by layer.

---

# 15.44 Jenkins and Git SHA Traceability

The deployment chain is:

```text
Source Code
     |
     v
Git Commit
     |
     v
Commit SHA
     |
     v
Docker Image
     |
     v
ECR
     |
     v
ECS Task Definition
     |
     v
Running Application
```

This allows the deployed service to be traced back to its source version.

This is particularly valuable when answering:

> "Which version introduced the issue?"

---

# 15.45 Jenkins vs Manual Deployment

## Manual Deployment

```text
Developer
   |
   +--> Build
   |
   +--> Docker
   |
   +--> Login
   |
   +--> Push
   |
   +--> ECS Update
```

Problems include:

* Inconsistent steps
* Human errors
* Poor traceability
* Slow deployments

---

## Jenkins Deployment

```text
Git
 |
 v
Jenkins
 |
 +--> Build
 |
 +--> Test
 |
 +--> Docker
 |
 +--> ECR
 |
 +--> ECS
 |
 +--> Verify
```

Advantages include:

* Repeatability
* Automation
* Consistency
* Traceability
* Faster deployment workflow

---

# 15.46 Trade-offs

## Advantages of Jenkins

### Flexible

Jenkins supports:

* Different agents
* Custom pipelines
* Many integrations

### Pipeline as Code

The deployment process can be version-controlled.

### Extensible

Plugins can integrate with different tools and platforms.

---

## Disadvantages

### Infrastructure Management

Jenkins itself requires maintenance.

### Plugin Dependencies

Plugin versions and compatibility can create operational complexity.

### Agent Configuration

Build agents must be configured correctly.

### Security Management

Credentials and access permissions require careful handling.

---

# 15.47 Alternatives

Possible alternatives include:

* GitHub Actions
* GitLab CI/CD
* AWS CodePipeline
* AWS CodeBuild

The choice depends on:

* Existing infrastructure
* Team expertise
* AWS integration
* Cost
* Operational requirements

For CitiCore, Jenkins was used as the CI/CD automation tool.

---

# 15.48 Production Considerations

A production CI/CD system should consider additional controls.

## Separate Environments

```text
Development
     |
     v
Staging
     |
     v
Production
```

Each environment should have appropriate configuration and deployment controls.

---

## Quality Gates

Possible gates include:

```text
Build
  |
  v
Unit Tests
  |
  v
Integration Tests
  |
  v
Static Analysis
  |
  v
Security Scan
  |
  v
Deployment
```

Not all of these should be claimed as current CitiCore implementation unless they are actually configured.

---

## Deployment Approval

For sensitive production systems:

```text
Pipeline
    |
    v
Approval
    |
    v
Production Deployment
```

This can reduce accidental deployments.

---

## Auditability

A deployment should ideally record:

* Build number
* Git commit
* Image version
* Deployment time
* Target environment

---

# 15.49 How I Would Explain Jenkins CI/CD in an Interview

> "In CitiCore, I used Jenkins to automate the CI/CD workflow for our containerized Spring Boot microservices. The pipeline checks out the source code, builds the application with Maven, runs the configured validation steps, creates a Docker image, and tags the image using the Git commit SHA for traceability. The image is pushed to Amazon ECR, and the pipeline updates the ECS task definition and service to deploy the new version on Fargate. After deployment, I verify the service at multiple levels, including ECS task status, health checks, logs, and application availability. For rollback, we can identify a previously stable Git SHA and redeploy the corresponding container image."

---

# 15.50 30–60 Second Interview Answer

> "I implemented a Jenkins-based CI/CD flow for CitiCore microservices. Jenkins checks out the code, builds the Spring Boot application using Maven, creates a Docker image, and tags it with the Git SHA. The image is pushed to Amazon ECR, and Jenkins updates the ECS task definition and service to deploy the new version on Fargate. We use versioned images for traceability and controlled rollback. I also learned that a successful ECS update command does not mean the deployment is fully healthy, so verification needs to include ECS status, application health, logs, and load balancer target health."

---

# 15.51 Interview Follow-up Questions

## Q1. What is the difference between Continuous Integration and Continuous Deployment?

**Answer:**

Continuous Integration focuses on automatically integrating, building, and validating code changes.

Continuous Deployment automatically deploys validated changes to an environment.

---

## Q2. Why did you use Jenkins?

**Answer:**

Jenkins provides flexible pipeline automation and integrates with Git, Maven, Docker, AWS, and other tools required by the CitiCore deployment workflow.

---

## Q3. What is a Jenkins agent?

**Answer:**

A Jenkins agent executes build and pipeline tasks. It can contain the tools required for specific workloads, such as Java, Maven, Docker, and AWS CLI.

---

## Q4. Why should builds run on agents instead of the controller?

**Answer:**

Agents separate build workloads from the Jenkins controller, improve scalability, and allow different environments for different build requirements.

---

## Q5. Why did you use Git SHA as a Docker image tag?

**Answer:**

The Git SHA provides traceability between the source code commit and the deployed container image and makes rollback easier.

---

## Q6. What happens after Jenkins updates an ECS service?

**Answer:**

ECS begins the deployment process. It starts tasks using the new task definition and image, performs the configured deployment and health processes, and eventually transitions the service to the new version according to its deployment configuration.

---

## Q7. How do you handle a failed deployment?

**Answer:**

I investigate the failed layer using Jenkins logs, ECS task events, CloudWatch logs, health checks, and dependency connectivity. If necessary, I identify a known-good image version and perform a controlled rollback.

---

## Q8. How do you secure AWS credentials in Jenkins?

**Answer:**

Credentials should be stored in Jenkins credential management or another secure mechanism rather than being hardcoded in the Jenkinsfile or source code.

---

## Q9. What is the Jenkins workspace?

**Answer:**

The workspace is the directory on the Jenkins agent where source code is checked out and build operations are performed.

---

## Q10. How do you verify that a deployment was successful?

**Answer:**

I verify multiple layers:

```text
Pipeline Success
      |
      v
ECS Service Updated
      |
      v
Task Running
      |
      v
Application Healthy
      |
      v
Target Healthy
      |
      v
API Working
```

---

# 15.52 Key Takeaways

The CitiCore CI/CD architecture follows this path:

```text
Developer
    |
    v
Git Repository
    |
    v
Jenkins
    |
    v
Checkout
    |
    v
Maven Build
    |
    v
Test
    |
    v
Docker Build
    |
    v
Git SHA Tag
    |
    v
Amazon ECR
    |
    v
ECS Task Definition
    |
    v
ECS Service
    |
    v
AWS Fargate
    |
    v
Deployment Verification
```

The most important lessons are:

* Automate repeatable deployment tasks.
* Keep the pipeline as code where possible.
* Use Jenkins agents for build execution.
* Use Git SHA-based image tags for traceability.
* Push immutable application images to ECR.
* Deploy using ECS task definition revisions.
* Verify deployment at multiple layers.
* Treat task status and application health as different states.
* Investigate failures layer by layer.
* Use controlled rollback with known-good versions.
* Secure Jenkins and AWS credentials.
* Avoid claiming a deployment is successful before the service becomes operationally healthy.

---

# 15.53 Final CI/CD Architecture Summary

```text
                    CITICORE CI/CD PIPELINE

Developer
    |
    v
Git Push
    |
    v
+----------------------+
|      JENKINS         |
|                      |
|  Checkout            |
|      |               |
|      v               |
|  Maven Build         |
|      |               |
|      v               |
|  Tests               |
|      |               |
|      v               |
|  Docker Build        |
|      |               |
|      v               |
|  Git SHA Tag         |
+------+---------------+
       |
       v
+----------------------+
|     AMAZON ECR       |
|                      |
| Container Image      |
+----------+-----------+
           |
           v
+----------------------+
|       AWS ECS        |
|                      |
| Task Definition      |
|       |              |
|       v              |
| ECS Service          |
+----------+-----------+
           |
           v
+----------------------+
|    AWS FARGATE       |
|                      |
| Spring Boot Service  |
+----------+-----------+
           |
           v
+----------------------+
|    VERIFICATION      |
|                      |
| ECS Task             |
| Health Endpoint      |
| Target Health        |
| Application Logs     |
+----------------------+
```

> **CitiCore uses Jenkins to automate the journey from source code to a versioned Docker image and finally to a managed ECS/Fargate deployment, with traceability, verification, and controlled rollback as important parts of the delivery process.**

---
