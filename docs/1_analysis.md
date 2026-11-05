# 1. Domain Analysis and Requirements

## 1.1. Project Overview

The project aims to develop a microservice web platform
for algorithmic testing and real-time pair programming.

The platform combines secure execution of user-submitted
code, automated algorithmic assessment, and collaborative
programming sessions.

The main engineering challenges include:

- Secure execution of untrusted source code.
- Isolation of execution environments.
- Resource management and workload distribution.
- Low-latency synchronization of collaborative editing.
- Concurrent modification conflict resolution.
- Authentication and authorization.

## 1.2. As-Is Analysis

### 1.2.1. Existing Solutions

The domain analysis considers the following solutions:

- Codeshare.io — collaborative source-code editing.
- LeetCode — algorithmic programming and automated assessment.
- Google Docs Collaboration Engine — real-time collaborative
  document editing.

The following comparison reflects the preliminary
engineering analysis. Descriptions of internal architectures
and implementation details should be treated as analytical
assumptions unless independently confirmed by public
technical documentation.

| Criterion | Codeshare.io | LeetCode | Google Docs | Proposed Platform |
|---|---|---|---|---|
| Main purpose | Collaborative editing | Algorithmic assessment | Collaborative documents | Assessment and pair programming |
| Architecture | Client-server | Service-oriented | Distributed collaboration | Microservices |
| Communication | WebSocket | REST / asynchronous processing | Real-time synchronization | REST, WebSocket/STOMP, AMQP |
| Code execution | Not a core capability | Isolated execution | Not a core capability | Docker-based sandbox |
| Data storage | Session state and persistence | Persistent data and caching | Distributed document state | PostgreSQL and Redis |
| Synchronization | Shared editing | Not primary | OT/CRDT-like approaches | CRDT/OT |
| Main engineering focus | Collaboration | Code assessment | Document consistency | Secure execution and collaboration |

### 1.2.2. Identified Problems

The preliminary analysis identifies several engineering
problems:

1. Collaborative editors require efficient management
   of long-lived connections and shared session state.

2. Algorithmic assessment platforms must isolate
   user-submitted code from the host environment.

3. Resource-intensive compilation and execution
   can affect the responsiveness of other services.

4. Collaborative editing requires consistency when
   multiple users modify the same document.

5. A centralized authentication mechanism is needed
   to manage access to platform resources.

### 1.2.3. Engineering Constraints

The proposed solution must account for:

- Remote Code Execution (RCE) risks.
- CPU and memory consumption by user programs.
- Maximum execution time.
- Container lifecycle management.
- Network isolation of execution environments.
- Message delivery and asynchronous task processing.
- WebSocket connection scalability.
- Concurrent editing conflicts.
- Authentication and role-based authorization.

## 1.3. To-Be Model

### 1.3.1. System Architecture

The target system follows a cloud-native microservice
architecture.

The API Gateway provides a common entry point for
client requests.

Backend services handle business operations,
authentication integration, session management,
assessment, and execution coordination.

RabbitMQ provides asynchronous communication between
services and execution workers.

The Code Execution Engine provisions temporary Docker
containers to execute submitted programs in isolation.

PostgreSQL stores persistent application data,
while Redis supports caching and rate limiting.

WebSocket/STOMP provides real-time communication
between participants of collaborative sessions.

CRDT or Operational Transformation (OT) is considered
for resolving concurrent editing operations.

### 1.3.2. Core Features

#### CF-01. User Authentication and Authorization

- Centralized authentication through Keycloak.
- OAuth2/OIDC integration.
- Role-Based Access Control (RBAC).
- Protected access to platform resources.

#### CF-02. Algorithmic Assessment

- Access to programming tasks.
- Submission of source code.
- Compilation and execution of submitted solutions.
- Automated verification using test cases.
- Support for hidden test cases.

#### CF-03. Isolated Code Execution

- Dynamic provisioning of execution containers.
- CPU, RAM and execution-time restrictions.
- Network isolation.
- Automatic cleanup of temporary containers.

#### CF-04. Asynchronous Execution Processing

- Submission of execution jobs to RabbitMQ.
- Distribution of jobs between Worker Nodes.
- Asynchronous result processing.
- Separation of resource-intensive operations
  from request-handling services.

#### CF-05. Real-Time Collaborative Editing

- Shared programming sessions.
- WebSocket/STOMP communication.
- Synchronization of editor changes.
- Concurrent editing support.

#### CF-06. Conflict Resolution

- Consideration of CRDT or OT algorithms.
- Consistent document state across participants.
- Handling of concurrent editing operations.

#### CF-07. Execution History and Results

- Storage of execution results.
- Access to previous submissions.
- Recording of execution logs.
- Display of assessment results.

#### CF-08. Infrastructure Protection and Monitoring

- Redis-based Rate Limiting.
- Authentication and authorization checks.
- Execution resource monitoring.
- Container lifecycle management.

## 1.4. Functional Requirements

| ID | Requirement |
|---|---|
| FR-01 | The system shall authenticate users through Keycloak using OAuth2/OIDC. |
| FR-02 | The system shall enforce role-based access control. |
| FR-03 | The system shall provide access to algorithmic programming tasks. |
| FR-04 | The system shall accept source-code submissions. |
| FR-05 | The system shall execute submitted code in isolated Docker containers. |
| FR-06 | The system shall enforce CPU, RAM and execution-time limits. |
| FR-07 | The system shall distribute execution jobs through RabbitMQ. |
| FR-08 | The system shall verify solutions using test cases, including hidden tests. |
| FR-09 | The system shall support real-time collaborative editing through WebSocket/STOMP. |
| FR-10 | The system shall handle concurrent editing operations using CRDT or OT. |
| FR-11 | The system shall store execution results and history. |
| FR-12 | The system shall apply Rate Limiting to protect platform resources. |

## 1.5. Non-Functional Requirements

The following values are project targets and must be
validated during implementation and testing.

| ID | Category | Requirement | Target |
|---|---|---|---|
| NFR-01 | Performance | API response time | p95 <= 300 ms at 50 RPS |
| NFR-02 | Performance | WebSocket latency | p95 <= 300 ms with 50 participants |
| NFR-03 | Scalability | Concurrent active users | 100 |
| NFR-04 | Scalability | Parallel execution workers | 10 |
| NFR-05 | Security | Authentication protocol | OAuth2/OIDC |
| NFR-06 | Security | Transport encryption | TLS 1.2+ |
| NFR-07 | Security | Execution isolation | Ephemeral Docker containers |
| NFR-08 | Security | Execution restrictions | CPU, RAM and time limits |
| NFR-09 | Security | Container network access | Blocked |
| NFR-10 | Reliability | Availability target | 99% |
| NFR-11 | Reliability | Recovery Time Objective | <= 4 hours |
| NFR-12 | Reliability | Recovery Point Objective | <= 24 hours |
| NFR-13 | Execution | Maximum execution time | 10 seconds |
| NFR-14 | Execution | Container cleanup | <= 5 seconds |
| NFR-15 | Quality | Backend line coverage | >= 70% |

## 1.6. Task Statement

The task is to design and develop a microservice web
platform for algorithmic testing and real-time pair
programming.

The platform must provide secure execution of untrusted
user code through isolated Docker containers, asynchronous
job processing through RabbitMQ, and real-time collaborative
editing using WebSocket/STOMP.

The solution must integrate centralized authentication
through Keycloak, persistent data storage through
PostgreSQL, and caching and rate limiting through Redis.

The developed platform must satisfy the specified
functional and non-functional requirements and undergo
testing to validate its security, performance,
reliability and scalability.

## 1.7. Analysis Conclusions

The domain analysis identified the need to combine
automated algorithmic assessment with real-time
collaborative programming.

The proposed architecture separates business operations
from resource-intensive code execution. RabbitMQ
provides asynchronous task distribution, while
isolated Docker containers reduce the risks associated
with executing untrusted source code.

WebSocket/STOMP enables real-time communication,
while CRDT or OT provides candidate approaches for
resolving concurrent editing operations.

The analysis also established functional requirements
and measurable non-functional targets. These requirements
will serve as the basis for detailed architecture design,
implementation and subsequent validation.