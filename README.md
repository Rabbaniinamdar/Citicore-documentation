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

## TOPIC 3 — MICROSERVICES

**Current Topic:** 3 — Microservices
**Previous Topics Completed:** 1 — Project Overview, 2 — Complete System Architecture
**Remaining Topics:** 4 — Service Discovery onward

---

# 3.1 Microservices Overview

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

