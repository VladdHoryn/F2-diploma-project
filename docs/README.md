# F2 Diploma Project — Documentation

## Project Overview

**Project:** Development of a Microservice Web Platform
for Algorithmic Testing and Real-Time Pair Programming.

The project focuses on developing a cloud-native
microservice platform for secure algorithmic assessment
and real-time collaborative programming.

The system provides isolated execution of untrusted code,
asynchronous processing of compilation tasks,
collaborative code editing, centralized authentication,
and execution result management.

## Documentation Structure

| Document | Description |
|---|---|
| [1_analysis.md](./1_analysis.md) | Domain analysis, As-Is and To-Be models, Core Features and requirements |
| Chapter 1 | Analytical section of the diploma thesis |

## Chapter 1 — Analytical Section

**Status:** Completed as a draft, pending academic review.

**Working document:** [F2_Diploma_Chapter1_Horyn](GOOGLE_DOCS_SHARE_LINK)

The cloud document is intended for academic review.
Access should be configured as **Can comment** for
the supervisor and teacher.

If the chapter is stored locally, use the following
repository path instead:

`./F2_Diploma_Chapter1_Horyn.docx`

## Technology Stack

| Layer | Technology |
|---|---|
| Backend | Java 17+, Spring Boot |
| Frontend | React, Monaco Editor |
| Database | PostgreSQL |
| Cache / Rate Limiting | Redis |
| Message Broker | RabbitMQ |
| Code Isolation | Docker |
| Authentication | Keycloak, OAuth2/OIDC |
| Real-Time Communication | WebSocket, STOMP |
| Synchronization | CRDT / OT |
| Database Migrations | Flyway |
| Deployment | Docker Compose |

## Non-Functional Requirements Matrix

The following values represent project targets and
acceptance criteria. They are not measured production
or load-testing results.

| ID | Category | Requirement | Target |
|---|---|---|---|
| NFR-01 | Performance | API response time | p95 <= 300 ms at 50 RPS |
| NFR-02 | Performance | WebSocket message latency | p95 <= 300 ms with 50 participants |
| NFR-03 | Scalability | Concurrent active users | 100 users |
| NFR-04 | Scalability | Parallel execution workers | 10 workers |
| NFR-05 | Security | Authentication | OAuth2 / OIDC through Keycloak |
| NFR-06 | Security | Network communication | TLS 1.2 or higher |
| NFR-07 | Security | Code execution isolation | Ephemeral Docker containers |
| NFR-08 | Security | Resource restrictions | CPU, RAM and execution-time limits |
| NFR-09 | Security | External network access | Blocked for execution containers |
| NFR-10 | Reliability | Service availability | 99% target |
| NFR-11 | Reliability | Recovery Time Objective | RTO <= 4 hours |
| NFR-12 | Reliability | Recovery Point Objective | RPO <= 24 hours |
| NFR-13 | Execution | Maximum code execution time | 10 seconds |
| NFR-14 | Execution | Container cleanup time | <= 5 seconds |
| NFR-15 | Quality | Backend line test coverage | >= 70% |

## Architectural Constraints

- The platform follows a microservice architecture.
- Resource-intensive code execution is processed
  asynchronously through RabbitMQ.
- User code is executed in isolated ephemeral Docker
  containers with resource restrictions.
- PostgreSQL is used for persistent relational data.
- Redis is used for caching and rate limiting.
- Keycloak provides centralized authentication
  and role-based access control.
- WebSocket/STOMP provides real-time communication.
- CRDT or OT is considered for concurrent editing
  conflict resolution.

## Development Status

| Stage | Status |
|---|---|
| Domain Analysis (As-Is) | Completed |
| Requirements Engineering (To-Be) | Completed |
| Chapter 1 Documentation | Pending academic review |
| Architecture Design | Planned |
| Implementation | Planned |
| Testing and Validation | Planned |

## AI Disclosure

AI assistance was used for domain analysis,
requirements engineering, technical writing,
and preparation of Chapter 1 conclusions.

Detailed interaction logs are maintained in the
PKM Workspace under the **AI Insights** section.