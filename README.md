# Scratcho 🏀

**Kenya's home of basketball.**

Scratcho is a basketball platform designed to bring Kenya's basketball ecosystem together. It aims to provide a central place for fans, players, teams, leagues, and organizers to access basketball information, follow competitions, explore player statistics, and stay updated on games.

This repository contains the backend rebuild of Scratcho, implemented in **Go**.

The project serves two purposes:

1. **Product development:** Build a reliable backend for a basketball platform.
2. **Engineering learning:** Explore backend engineering, distributed systems, infrastructure, observability, and AI through a progressively evolving system.

> **Project status:** This repository is being rebuilt. The architecture described here represents the intended direction, not a claim that every feature or technology has already been implemented. Check the roadmap and repository for the current implementation status.

---

## Table of Contents

- [Overview](#overview)
- [Core Features](#core-features)
- [Engineering Goals](#engineering-goals)
- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [How the Components Work Together](#how-the-components-work-together)
- [Domain Model](#domain-model)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Development Workflow](#development-workflow)
- [Testing](#testing)
- [Observability](#observability)
- [Security](#security)
- [Roadmap](#roadmap)
- [Architecture Decision Records](#architecture-decision-records)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

Basketball data is often spread across different platforms, social media accounts, websites, and communication channels. Scratcho aims to make this information easier to access through a unified basketball platform.

The backend will provide APIs and supporting services for managing basketball competitions, publishing updates, delivering notifications, and powering data-driven features.

The system will initially be developed as a **modular Go monolith**. As the project evolves, selected components may be extracted into independent services where there is a clear architectural or educational reason to do so.

The goal is not to introduce distributed systems complexity immediately. It is to understand the fundamentals, identify architectural limitations, and introduce additional technologies deliberately.

### Design philosophy

- Start with a working, understandable system.
- Keep business logic independent of infrastructure wherever practical.
- Prefer PostgreSQL as the primary source of truth.
- Introduce asynchronous processing when it solves a real problem.
- Make data consistency, failure handling, and observability explicit.
- Introduce distributed components incrementally.
- Document important architectural decisions.
- Keep the implementation status of every major component visible.

---

## Core Features

The planned product capabilities include:

### Basketball management

- Leagues and competitions
- Seasons and tournaments
- Teams and team rosters
- Players and player profiles
- Fixtures and game schedules
- Game results
- League standings
- Player and team statistics

### Basketball content

- News and announcements
- Game reports
- Highlights and media metadata
- Content categorization
- Publishing and moderation workflows

### Fan engagement

- User accounts and authentication
- Following teams and players
- Notifications about games and results
- Live game updates
- Personalized basketball content

### Intelligence and discovery

- Search across basketball information
- Contextual basketball question answering
- Retrieval-augmented generation (RAG)
- Data-driven insights derived from available statistics

These capabilities will be implemented incrementally. Features listed here should not be assumed to be available until their corresponding implementation and tests exist.

---

## Engineering Goals

Scratcho is also a practical environment for exploring backend engineering concepts.

The project will cover:

- Go application design and concurrency
- REST API design
- GraphQL
- gRPC
- PostgreSQL schema design, transactions, and indexing
- Redis caching and ephemeral data
- Event-driven architecture with Kafka
- Task queues with RabbitMQ
- Docker-based development environments
- Kubernetes deployment and orchestration
- Infrastructure as code with Terraform
- Cloud deployment on Google Cloud Platform (GCP)
- Metrics, logs, traces, and alerting
- Continuous integration and delivery
- Retrieval-augmented generation
- Automated testing and reliability engineering

These technologies will not all be introduced at the beginning. The project will evolve through stages so that each technology addresses a specific requirement or demonstrates a clearly documented architectural trade-off.

---

## Architecture

### High-level architecture

The target architecture separates client access, application logic, persistent storage, asynchronous processing, infrastructure, and observability.

```mermaid
flowchart TB
    subgraph Clients["Clients"]
        WEB["Web Application"]
        MOBILE["Mobile Application"]
        ADMIN["Admin Dashboard"]
    end

    subgraph Access["API Access Layer"]
        REST["REST API"]
        GRAPHQL["GraphQL API"]
        GRPC["gRPC Interfaces"]
    end

    subgraph Application["Go Application Layer"]
        AUTH["Authentication and Authorization"]
        USERS["User Management"]
        BASKETBALL["Basketball Domain"]
        CONTENT["Content Management"]
        NOTIFICATIONS["Notification Logic"]
        SEARCH["Search and AI Integration"]
    end

    subgraph Data["Data Layer"]
        POSTGRES[("PostgreSQL")]
        REDIS[("Redis")]
        OBJECTS[("Object Storage")]
    end

    subgraph Messaging["Asynchronous Processing"]
        OUTBOX["Transactional Outbox"]
        KAFKA[["Kafka Event Streaming"]]
        RABBITMQ[["RabbitMQ Task Queues"]]
        WORKERS["Background Workers"]
    end

    subgraph Intelligence["AI and Retrieval"]
        INGESTION["Document Ingestion"]
        RETRIEVAL["Retrieval Pipeline"]
        LLM["LLM Integration"]
    end

    subgraph Infrastructure["Infrastructure and Observability"]
        CONTAINERS["Docker"]
        K8S["Kubernetes"]
        CLOUD["GCP"]
        METRICS["Prometheus"]
        DASHBOARDS["Grafana"]
        TELEMETRY["OpenTelemetry"]
    end

    WEB --> REST
    WEB --> GRAPHQL
    MOBILE --> REST
    MOBILE --> GRAPHQL
    ADMIN --> REST

    REST --> AUTH
    GRAPHQL --> AUTH
    GRPC --> AUTH

    AUTH --> USERS
    AUTH --> BASKETBALL
    AUTH --> CONTENT
    AUTH --> NOTIFICATIONS
    AUTH --> SEARCH

    USERS --> POSTGRES
    BASKETBALL --> POSTGRES
    CONTENT --> POSTGRES
    NOTIFICATIONS --> POSTGRES

    BASKETBALL --> REDIS
    CONTENT --> REDIS
    CONTENT --> OBJECTS

    POSTGRES --> OUTBOX
    OUTBOX --> KAFKA
    KAFKA --> WORKERS
    RABBITMQ --> WORKERS

    WORKERS --> POSTGRES
    WORKERS --> REDIS

    CONTENT --> INGESTION
    INGESTION --> RETRIEVAL
    RETRIEVAL --> LLM
    SEARCH --> RETRIEVAL

    CONTAINERS --> K8S
    K8S --> CLOUD

    Application -. telemetry .-> TELEMETRY
    WORKERS -. telemetry .-> TELEMETRY
    TELEMETRY --> METRICS
    METRICS --> DASHBOARDS
```

The diagram represents the **target direction**, not the initial deployment. Some components may be combined in the first implementation.

### Architectural evolution

The system will evolve through three broad stages.

**Stage 1: Modular monolith**

A single Go application contains clearly separated business modules. It communicates with PostgreSQL and exposes the required APIs.

**Stage 2: Asynchronous processing**

Redis caching, a transactional outbox, Kafka events, RabbitMQ task queues, and background workers are introduced as specific use cases require them.

**Stage 3: Distributed deployment**

Selected components may become independent services. The system gains deployment orchestration, centralized observability, infrastructure automation, and cloud deployment.

The goal is to understand the operational and architectural costs of each transition, not simply to maximize the number of technologies used.

---

## Technology Stack

| Technology | Intended responsibility |
|---|---|
| Go | Backend application and workers |
| PostgreSQL | Primary relational database and source of truth |
| REST | Resource-oriented HTTP API |
| GraphQL | Flexible client-facing data queries |
| gRPC | Typed service-to-service communication where useful |
| Redis | Caching, rate limiting, and short-lived data |
| Kafka | Durable event streaming and event-driven integration |
| RabbitMQ | Background task delivery and work queues |
| Docker | Reproducible development and deployment environments |
| Kubernetes | Container orchestration when deployment complexity justifies it |
| Prometheus | Metrics collection |
| Grafana | Metrics dashboards and operational visibility |
| OpenTelemetry | Instrumentation for metrics, traces, and logs |
| Terraform | Infrastructure provisioning |
| Google Cloud Platform | Cloud infrastructure and managed services |
| RAG | Retrieval-grounded answers over approved basketball information |
| CI/CD | Automated checks, builds, and deployment workflows |

### Why both Kafka and RabbitMQ?

They solve related but different messaging problems.

- **Kafka** is intended for event streams, durable event history, and independent consumers that process domain events.
- **RabbitMQ** is intended for task queues, routing work to workers, and managing task delivery.

Using both increases operational complexity. Each must have a documented use case before being added to the running system.

### Why both REST, GraphQL, and gRPC?

They provide different communication interfaces.

- **REST** supports conventional resource-oriented HTTP operations.
- **GraphQL** lets clients request specific combinations of related data.
- **gRPC** supports strongly typed remote procedure calls between components when that communication pattern is beneficial.

They should not all expose duplicate implementations of the same business logic. The underlying domain operations should remain reusable.

---

## How the Components Work Together

Consider a basketball game whose final score has just been recorded.

1. An authorized user submits the result through the API.
2. The application validates the request and checks that the game can be updated.
3. PostgreSQL stores the result and related changes in a database transaction.
4. The same transaction records an event in the transactional outbox.
5. An outbox publisher forwards the event to Kafka.
6. Independent consumers process the event for their respective purposes.
7. A notification worker may use RabbitMQ to schedule delivery to eligible users.
8. Redis entries for affected standings or game details may be invalidated or refreshed.
9. Metrics and traces help developers investigate processing failures or delays.

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant DB as PostgreSQL
    participant Outbox
    participant Kafka
    participant Worker
    participant Queue as RabbitMQ
    participant Notify as Notification Worker

    Client->>API: Submit final score
    API->>DB: Begin transaction
    API->>DB: Save result and outbox event
    API->>DB: Commit transaction
    API-->>Client: Result accepted

    Outbox->>DB: Read unpublished events
    Outbox->>Kafka: Publish GameFinalized event
    Kafka-->>Worker: Deliver event
    Worker->>Queue: Enqueue notification task
    Queue-->>Notify: Deliver task
    Notify->>Notify: Validate recipients and preferences
    Notify->>Notify: Send notification
```

This is a proposed flow. It should only be considered implemented once the corresponding components, retry policies, and tests are in place.

### Important reliability considerations

- The API must not report a successful database update if the transaction fails.
- The outbox prevents a committed database change from being separated from its recorded event by a process crash.
- Event publishers may publish duplicates; consumers must be idempotent.
- Consumers need retry and failure-handling policies.
- Notification delivery should respect user preferences.
- Cache invalidation must not allow stale data to persist indefinitely.
- Logs, metrics, and traces should make failures diagnosable.

---

## Domain Model

The initial relational model will center on basketball entities.

```mermaid
erDiagram
    USER ||--o{ USER_FOLLOW : creates
    TEAM ||--o{ USER_FOLLOW : receives
    PLAYER ||--o{ USER_FOLLOW : receives

    LEAGUE ||--o{ SEASON : contains
    SEASON ||--o{ TEAM_SEASON : registers
    TEAM ||--o{ TEAM_SEASON : participates

    TEAM ||--o{ TEAM_PLAYER : registers
    PLAYER ||--o{ TEAM_PLAYER : joins

    SEASON ||--o{ GAME : schedules
    TEAM ||--o{ GAME : home_team
    TEAM ||--o{ GAME : away_team

    GAME ||--o{ PLAYER_GAME_STAT : records
    PLAYER ||--o{ PLAYER_GAME_STAT : contributes

    USER {
        uuid id PK
        string email
        string display_name
    }

    LEAGUE {
        uuid id PK
        string name
    }

    SEASON {
        uuid id PK
        uuid league_id FK
        string name
        string status
    }

    TEAM {
        uuid id PK
        string name
        string slug
    }

    PLAYER {
        uuid id PK
        string full_name
    }

    GAME {
        uuid id PK
        uuid season_id FK
        uuid home_team_id FK
        uuid away_team_id FK
        datetime scheduled_at
        string status
        int home_score
        int away_score
    }
```

This diagram is conceptual. The actual schema will evolve based on domain rules, query patterns, and integrity requirements.

### Data ownership principles

- PostgreSQL is the authoritative store for core basketball records.
- Redis is a cache, not the primary authority for game results or standings.
- Kafka stores events for asynchronous processing; it does not automatically replace the relational database.
- Object storage holds media files, while PostgreSQL stores associated metadata.
- Derived data should have a documented source and refresh strategy.

---

## Repository Structure

The repository will initially favor a modular Go application rather than premature microservices.

An illustrative layout:

```text
scratcho/
├── cmd/
│   ├── api/
│   │   └── main.go
│   ├── worker/
│   │   └── main.go
│   └── migrate/
│       └── main.go
│
├── internal/
│   ├── auth/
│   │   ├── handler.go
│   │   ├── service.go
│   │   ├── repository.go
│   │   └── model.go
│   ├── users/
│   ├── leagues/
│   ├── seasons/
│   ├── teams/
│   ├── players/
│   ├── games/
│   ├── statistics/
│   ├── content/
│   ├── notifications/
│   ├── events/
│   └── platform/
│       ├── database/
│       ├── cache/
│       ├── messaging/
│       ├── observability/
│       └── config/
│
├── api/
│   ├── openapi/
│   ├── graphql/
│   └── proto/
│
├── migrations/
├── scripts/
├── deployments/
│   ├── docker/
│   ├── kubernetes/
│   └── terraform/
│
├── docs/
│   ├── architecture/
│   ├── adr/
│   └── runbooks/
│
├── tests/
│   ├── integration/
│   └── e2e/
│
├── .github/
│   └── workflows/
│
├── .env.example
├── .gitignore
├── Dockerfile
├── compose.yaml
├── go.mod
├── go.sum
└── README.md
```

This tree is a proposed structure, not a requirement to create every directory immediately. Empty packages and placeholder services should not be added just to match the diagram.

---

## Getting Started

### Prerequisites

The initial development environment requires:

- Go — use the version declared in `go.mod`.
- PostgreSQL — for persistent application data.
- Git.
- Docker and Docker Compose, if using the containerized development environment.

Redis, Kafka, RabbitMQ, Kubernetes, and cloud infrastructure can be introduced in later development stages.

### 1. Clone the repository

```bash
git clone https://github.com/asino-nelson/Scratcho-Monolith.git
cd scratcho
```

Replace `<repository-url>` with the actual repository URL.

### 2. Inspect the Go version

```bash
go version
cat go.mod
```

Use a Go version compatible with the project's `go.mod` file.

### 3. Configure environment variables

Create a local environment file from the example when the repository provides one:

```bash
cp .env.example .env
```

On Windows PowerShell, use:

```powershell
Copy-Item .env.example .env
```

Configure the database connection and other required settings before starting the application.

**Do not commit `.env` or real credentials to version control.**

### 4. Start dependencies

If the project includes a working Compose configuration:

```bash
docker compose up -d
```

Inspect the running containers:

```bash
docker compose ps
```

The exact services available depend on the current Compose configuration.

### 5. Download Go dependencies

```bash
go mod download
```

### 6. Run database migrations

Use the migration command documented by the implemented project.

For example, if a migration tool is configured:

```bash
make migrate-up
```

This command is illustrative. It will only work if the corresponding Makefile target exists.

### 7. Run the application

Use the entry point implemented in the repository. For example:

```bash
go run ./cmd/api
```

This assumes `cmd/api/main.go` has been created and configured.

### 8. Run tests

```bash
go test ./...
```

### Development status

This README documents the intended setup and architecture. Commands that depend on components not yet implemented should be updated when the corresponding tooling is added.

---

## Configuration

Application configuration should be explicit and environment-driven.

Typical configuration values include:

| Variable | Purpose |
|---|---|
| `APP_ENV` | Application environment |
| `HTTP_ADDR` | HTTP server bind address |
| `DATABASE_URL` | PostgreSQL connection string |
| `REDIS_ADDR` | Redis address |
| `KAFKA_BROKERS` | Kafka broker addresses |
| `RABBITMQ_URL` | RabbitMQ connection URL |
| `LOG_LEVEL` | Logging verbosity |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OpenTelemetry collector endpoint |

Only add variables required by implemented components. Optional integrations should not prevent the core application from starting unless they are explicitly configured as mandatory.

A local `.env.example` might look like this:

```dotenv
APP_ENV=development
HTTP_ADDR=:8080
DATABASE_URL=postgres://scratcho:local_password@localhost:5432/scratcho?sslmode=disable
LOG_LEVEL=debug
```

The credentials above are local-development placeholders, not production credentials.

Configuration should be validated at startup. Missing required values should produce actionable errors rather than allowing the application to fail unexpectedly later.

---

## Development Workflow

The project should evolve in small, testable increments.

A typical feature workflow:

1. Define the feature and its business rules.
2. Identify the data model and integrity constraints.
3. Define service or domain interfaces where useful.
4. Implement the business logic.
5. Add the necessary API endpoint.
6. Write unit and integration tests.
7. Add migrations if the schema changes.
8. Update API and architecture documentation.
9. Add metrics and structured logs where appropriate.
10. Review failure behavior before merging.

### Code quality

Use Go's standard formatting and analysis tools:

```bash
gofmt -w .
go vet ./...
go test ./...
```

For a project-wide formatter check in CI:

```bash
test -z "$(gofmt -l .)"
```

Additional static analysis and linting tools may be added as the project matures.

---

## Testing

Testing should cover business correctness as well as infrastructure behavior.

### Unit tests

Test domain logic without requiring a running database or message broker.

Examples:

- Validating game transitions.
- Calculating standings.
- Calculating player statistics.
- Validating permissions.
- Applying score-update rules.

### Integration tests

Test components against real dependencies where appropriate.

Examples:

- PostgreSQL transactions and migrations.
- Repository queries.
- Redis cache behavior.
- Event publishing and consumption.
- RabbitMQ task delivery.

### API tests

Verify:

- Request validation.
- Authentication and authorization.
- Response formats.
- Error handling.
- Pagination and filtering.
- Rate limiting.

### Reliability tests

As asynchronous processing is introduced, test:

- Duplicate event delivery.
- Consumer retries.
- Poison messages.
- Database outages.
- Graceful shutdown.
- Recovery after process restarts.

A passing unit test suite does not prove that a distributed workflow is reliable. Integration and failure-path tests are essential.

---

## Observability

Operational visibility should be designed into the application rather than added only after problems appear.

### Logs

Use structured logs that include useful context such as:

- Request or correlation ID.
- Operation name.
- Relevant entity ID.
- Error classification.
- Processing duration.

Avoid logging passwords, tokens, or unnecessary personal information.

### Metrics

Prometheus metrics may cover:

- Request counts and latency.
- HTTP error rates.
- Database connection-pool utilization.
- Cache hit and miss rates.
- Event publishing and consumption failures.
- Queue depth and processing delay.
- Worker execution duration.

### Tracing

OpenTelemetry can connect a request's execution across application boundaries, including database operations and asynchronous processing where trace context is propagated.

### Dashboards

Grafana dashboards should help developers answer questions such as:

- Is the API healthy?
- Which endpoints are slow?
- Are database connections exhausted?
- Are consumers falling behind?
- Are tasks repeatedly failing?
- Did a recent deployment introduce a regression?

Metrics and dashboards should reflect real instrumentation. They should not be represented as operational capabilities until they have been configured and verified.

---

## Security

Security requirements include:

- Password hashing using an established password-hashing algorithm.
- Authentication and server-side authorization.
- Validation of user-controlled input.
- Least-privilege database and service credentials.
- Secret management outside source control.
- Rate limiting for sensitive endpoints.
- Secure handling of file uploads.
- Appropriate CORS configuration.
- Audit logging for sensitive administrative actions.
- Dependency and container image scanning.
- TLS for production network communication.

Administrative actions involving leagues, games, scores, and user permissions must be protected by explicit authorization rules.

---

## Roadmap

The implementation should progress in deliberate stages.

### Phase 1 — Core backend

- [ ] Establish the Go project structure.
- [ ] Configure application startup and shutdown.
- [ ] Add PostgreSQL connectivity.
- [ ] Introduce database migrations.
- [ ] Implement core domain models.
- [ ] Build initial REST endpoints.
- [ ] Add validation and error handling.
- [ ] Write unit and integration tests.

### Phase 2 — Basketball functionality

- [ ] Leagues and seasons.
- [ ] Teams and players.
- [ ] Games and fixtures.
- [ ] Results and standings.
- [ ] Player and team statistics.
- [ ] News and content management.
- [ ] Authentication and authorization.

### Phase 3 — API design and performance

- [ ] Document the REST API.
- [ ] Add pagination, filtering, and sorting.
- [ ] Profile database queries.
- [ ] Add indexes based on query patterns.
- [ ] Introduce Redis where cacheable workloads justify it.
- [ ] Evaluate GraphQL for client-specific data access.

### Phase 4 — Asynchronous processing

- [ ] Implement a transactional outbox.
- [ ] Publish domain events to Kafka.
- [ ] Build idempotent consumers.
- [ ] Introduce RabbitMQ for appropriate background tasks.
- [ ] Implement retries and failure handling.
- [ ] Add notification workflows.

### Phase 5 — Operations and deployment

- [ ] Build a production-ready Docker image.
- [ ] Add CI checks.
- [ ] Add application metrics.
- [ ] Integrate OpenTelemetry.
- [ ] Create Grafana dashboards.
- [ ] Deploy a baseline environment to GCP.
- [ ] Manage infrastructure with Terraform.

### Phase 6 — Distributed systems

- [ ] Identify components that genuinely benefit from independent deployment.
- [ ] Extract a service only when justified.
- [ ] Introduce gRPC where typed service-to-service communication is useful.
- [ ] Add Kubernetes deployment configuration.
- [ ] Test graceful shutdown and recovery.
- [ ] Document service ownership and communication contracts.

### Phase 7 — Basketball intelligence

- [ ] Design a basketball knowledge ingestion pipeline.
- [ ] Index approved content for retrieval.
- [ ] Implement retrieval-augmented generation.
- [ ] Provide source-grounded answers.
- [ ] Evaluate answer quality and retrieval accuracy.
- [ ] Add appropriate safeguards and monitoring.

The roadmap is a plan, not a commitment to ship every technology. Each phase should produce a working, testable system.

---

## Architecture Decision Records

Significant architectural decisions should be recorded in `docs/adr/`.

An Architecture Decision Record (ADR) should explain:

- The problem being addressed.
- The constraints and alternatives considered.
- The decision.
- The expected benefits.
- The trade-offs and risks.
- The conditions under which the decision should be revisited.

Suggested initial ADRs:

- `0001-modular-monolith.md`
- `0002-postgresql-as-source-of-truth.md`
- `0003-rest-api-design.md`
- `0004-redis-caching-strategy.md`
- `0005-kafka-event-streaming.md`
- `0006-rabbitmq-task-queues.md`
- `0007-transactional-outbox.md`
- `0008-observability-standard.md`

These are suggested documents. Create each ADR when the decision has actually been evaluated, rather than treating its filename as proof that a decision has been made.

---

## Contributing

Contributions should preserve the project's focus on maintainable code, explicit design decisions, and reliable behavior.

Before submitting a change:

1. Explain the problem being solved.
2. Keep the change focused.
3. Add or update tests.
4. Run the relevant test suite.
5. Update documentation when behavior or configuration changes.
6. Explain any new infrastructure dependencies.
7. Document significant architectural decisions.

For changes that introduce a new technology, explain:

- What problem it solves.
- Why existing components are insufficient.
- What operational costs it introduces.
- How it will be tested and monitored.
- How the system behaves when it fails.

---

## License

A license has not yet been specified.




