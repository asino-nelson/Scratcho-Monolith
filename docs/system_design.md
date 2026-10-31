# Scratcho Backend System Design
INTENTIONALLY OVER-ENGINEERED <br>
The project intentionally evolves towards a distributed architecture.

## 1. Problem Framing

Scratcho is a basketball platform focused on making basketball information and community features accessible in one place, with an initial emphasis on Kenyan basketball and room to expand to regional and global competitions. The near-term product goal is simple:

- Users discover games, leagues, teams, players, standings, results, news, highlights, and basketball statistics.
- Admins manage competitions, schedules, teams, players, game results, articles, and media metadata.
- Game events and fimal scores update deriver statistics. standings, rankings, feeds, and notifications.
- Users can follow teams and players a nd receive relevant updates.
- A future basketball assistant to answer questions grounded in Scratcho data and trusted basketball documents.


## 2. Functional Requirements

### Identity and User Management
- Register, sign in, sign out, and retrieve the current user profile.
- JWT-based access control with role/ permission checks for users, editors, competition admin, and platform admin.
- Users follow and unfollow teams and players and manage notification preferences

### Basketball Domain
- Manage leagues, seasons, competitions, stages, groups, teams, players, coaches, venues, and rosters.
- Create fixtures and schedules, record game status, and publish final scores.
- Record game events such as points, fouls, rebounds, assists, substitutions, and timeouts where the data model supports them.
- Calculate standings and player/ team statistics from game data.
- Browse historical results, head-to-head records, team and player performance.
- Repesent postponed, cancelled, abandoned, live, and completed games explicitly.

### Content and Media
- Publish and manage mews articles, game reports, interviews, highlights, short-form videos, and images.
- Store media files in object storage and keep metadata, ownership, captions, and publication state in  PostgreSQL.
- Support moderation, draft/ published states, tags, and links to relevant tams, players, or games.

### Notifications and Enguagement
- Notify followers when games begin, scores change, games finish, or articles are published.
- Support in-app notifications and leave seams for push, email, and SMS providers.
- Use queues and workers for retryable delivery rather than blocking user-facing requests.

### Admin and Data Quality
- Provide auditable admin workflows for managing teams, players, fixtures, results, and content.
- Validate score transitions and prevent duplicate or conflicting game-event submissions.
- Track who changed important records, when they changed them, and the reason where appropriate.
- Provide import jobs for fixtures, rosters, and historical statistics with validation and error reports.
### Basketball intelligence (RAG)
- Answer questions using Scratcho's structured data and an indexed corpus of trusted basketball documents.
- Explain player/ team comparisons with retrieved statistics and source references.
- Answer rules, history, competition, and player-profile questions from indexed documents.
- Clearly separate database-backed numerical answers from document-based generated answers.
- Refuse to invent statistics when authoritative data is missing; return data freshness and source information.

## 3. Non-Functional Requirements

- API Latency:
  - target p95 under 300 ms for common warm-cache read endpoints in local or regional test envirenments.
- Correctness:
  - Game results, standings updates, roles, and admin changes must preserve invariants and be auditable.
- Concurrency:
  - Prevent duplicate game-event processing and conflicting updates; use database constraints and idempotency.
- Scalability:
  - Stateless API instances; scale read-heavy APIs and event consumers independently in later stages.
- Availability:
  - Graceful degradation if Redis, a consumer, or an external notification provider is unavailable.
- Observability:
  - Metrics, structured logs, traces, correlation IDs, consumer lag, and actionable dashboards.
- Security:
  - Authentication, authorization, input validation, rate limiting, secrets management, and least-privilege infrastructure.
- Data lifecycle:
  - Backups, migrations, retention policies, media lifecycle, and tested restore procedures.
- Explainability:
  - Derived standings and statistics can be traced to source game data and processing versions.


## 4. High-Level Architecture

The system evolves in stages. This initial version is a modular Go monolith backed by PostgreSQL. The target architecture introduces independent services, asynchronous messaging, caching, cloud deployment, and a RAG subsystem.

# Scratcho System Architecture

```text
Clients: Mobile / Web / Admin
             |
        Edge / API Gateway
             |
      REST API + GraphQL API
             |
   +--------------------------------------+
   | Go domain services (target state)    |
   | Identity | Basketball | Content      |
   | Statistics | Standings | Notifications|
   | Search / RAG | Admin / Ingestion     |
   +--------------------------------------+
             | gRPC for selected internal calls
             |
       +-----+--------+
       |              |
  PostgreSQL         Redis
       |
  Transactional outbox
       |
      Kafka
       |
   +-------------------+-------------------+
   |                   |                   |
Statistics consumer  Search indexer     Analytics
   |
  RabbitMQ for task-oriented jobs
   |
Email / Push / Media / Import / Report workers

Docker -> Kubernetes -> GCP
Terraform provisions infrastructure
CI/CD builds, tests, scans, publishes, deploys
Prometheus + Grafana + OpenTelemetry provide observability
```


## 5. API Design

### REST
REST is predictable resource-oriented interface for mutations, integrations, admin operations, and simple reads.<br>

Existing:

- `POST /api/v1/auth/register`
- `POST /api/v1/auth/login`
- `GET /api/v1/users/me`

- `GET /api/v1/teams/{teamID}`
- `GET /api/v1/players/{playerID}`
- `GET /api/v1/players/{playerID}/stats`
- `GET /api/v1/articles`
- `POST /api/v1/admin/imports`


- `GET    /api/v1/leagues`
- `GET    /api/v1/leagues/{leagueID}/standings`
- `GET    /api/v1/games?date=YYYY-MM-DD&status=scheduled`
- `GET    /api/v1/games/{gameID}`
- `POST   /api/v1/games`
- `PATCH  /api/v1/games/{gameID}`
- `POST   /api/v1/games/{gameID}/events`
- `POST   /api/v1/games/{gameID}/finalize`

### GraphQL
GraphQL serves client screens that combine related data and need flexible selection. For example, the home screen can request today's games, standings, recent articles, and top performers in one client query.

~~~ 
query HomeScreen {
  gamesToday {
    id
    status
    homeTeam { id name }
    awayTeam { id name }
    score { home away }
  }
  leagueStandings(leagueId: "league-1") {
    team { id name }
    wins
    losses
    points
  }
  latestArticles(limit: 5) {
    id
    title
    publishedAt
  }
} 
~~~

### gRPC 
gRPC for selected service-to-service calls after service boundaries exist. It provides typed contracts and generated clients, but introduces protobuf versioning and operational complexity.
~~~
service StatisticsService {
  rpc GetPlayerStats(GetPlayerStatsRequest)
      returns (GetPlayerStatsResponse);
}

service BasketballService {
  rpc GetGame(GetGameRequest)
      returns (GetGameResponse);
}
~~~


## 6. PostgreSQL Data Model and Consistency

- users, roles, permissions, refresh_sessions, user_preferences, follows
- leagues, seasons, competitions, stages, groups, venues
- teams, players, coaches, team_rosters, player_season_memberships
- games, game_periods, game_events, game_scores, game_results
- player_game_stats, team_game_stats, standings_snapshots
- articles, media_assets, tags, article_entities
- notifications, notification_deliveries, audit_log
- outbox_events, idempotency_keys, import_jobs, import_errors
- rag_documents, document_chunks, ingestion_jobs (or equivalent metadata tables)
- Use foreign keys, unique constraints, check constraints, and transactions to encode invariants.
- Store authoritative game events and final results separately from derived statistics where useful.
- Define whether statistics are recalculated from source events or incrementally projected; support reconciliation.
- Use migrations, indexes guided by query plans, connection pooling, backups, and restore tests.
- Keep media binaries in object storage rather than storing large files directly in PostgreSQL.
- For RAG, PostgreSQL with pgvector is a reasonable starting point.


## 7. Redis: Cache, Rate Limits, and Short-Lived State
- Cache popular read models: standings, team/player profiles, fixture lists, and home-screen aggregates.
- Use TTLs and explicit invalidation/versioning; document acceptable staleness for each cache.
- Rate-limit login, expensive GraphQL operations, and public APIs with a distributed counter strategy.
- Use short-lived coordination keys only when necessary; do not treat a Redis lock as a substitute for database constraints.
- Experiment with sorted sets for leaderboards, while retaining authoritative statistics in PostgreSQL. 
- Measure cache hit rate, latency, memory use, and database load before/after caching.

Failure behavior: if Redis is unavailable, selected reads should fall back to PostgreSQL within controlled timeouts, with protection against a thundering herd. Avoid making all requests fail simply because a cache is down.


## 8. Kafka: Domain Events and Projections
Kafka becomes the event-streaming backbone for durable domain events and multiple independent consumers.

~~~
game.created
game.started
game.event_recorded
game.finalized
standings.recalculation_requested
article.published
player.updated
team.updated
user.followed_team
~~~

Example: when a game is finalized, the Basketball domain commits the result and an outbox event in the same PostgreSQL transaction. An outbox publisher sends the event to Kafka. Statistics, Standings, Notifications, Search/RAG ingestion, and Analytics can consume it independently.

- Choose partition keys intentionally; game_id preserves per-game event order, while team/player projections may need their own ordering strategy.
- Use consumer groups for horizontal processing and monitor lag.
- Assume at-least-once delivery; make consumers idempotent using event IDs, deduplication records, and transactional updates.
- Define schema compatibility, event versioning, retention, replay, poison-message handling, and consumer recovery.
- Do not claim exactly-once business behavior solely because Kafka offers transactional features; end-to-end effects still need careful design.


## 9. RabbitMQ: Task Queues and Background Work
RabbitMQ is for work distribution where a worker should take and acknowledge a task, with retry and dead-letter behavior.

- Send notification-delivery tasks to email/push workers.
- Queue media metadata extraction, thumbnail generation, report generation, data imports, and scheduled recalculation jobs.
- Use acknowledgements, bounded retries, backoff, dead-letter queues, concurrency limits, and idempotent handlers.
- Keep the distinction clear: Kafka records events that multiple consumers may independently process; RabbitMQ distributes queued tasks to workers.

You do not need to route every Kafka event through RabbitMQ. If an event triggers a task, a dedicated consumer can translate it into a queued job where that provides a real operational benefit.


## 10. Real-Time Updates
WebSockets for live game scores, game-event timelines, and notification updates. HTTP remains appropriate for submitting official game events and for ordinary queries.

~~~ text
Driver-style location tracking is not part of Scratcho.
For Scratcho:
Admin scorer / trusted feed -> HTTP event write -> PostgreSQL
                                         |
                                         +-> event/outbox -> Kafka
                                         |
                                         +-> WebSocket hub -> subscribed clients
~~~

- Authorize subscriptions by game, competition, and user role.
- Use a sequence number or event ID so clients can detect gaps and reconnect safely.
- Treat WebSocket messages as a delivery mechanism, not the durable record.
- After reconnect, clients fetch the authoritative game state and resume updates.
- Scale WebSocket fan-out across replicas using a shared pub/sub or event layer; do not rely on one in-process hub once multiple instances are running.

## 11. Docker, Kubernetes, GCP, and Terraform

### Docker
- Create reproducible multi-stage Go builds and small runtime images.
- Docker Compose for local PostgreSQL, Redis, Kafka, RabbitMQ, Prometheus, Grafana, and service dependencies.
- Run containers as non-root where practical; use health checks, explicit resource settings, and environment-specific configuration.
- Separate application configuration from secrets and never bake credentials into images.

### Kubernetes
- Deploy stateless API services and workers with Deployments, Services, and resource requests/limits.
- Use readiness and liveness probes appropriately; do not make liveness probes depend on every external service.
- Practice rolling deployments, rollbacks, autoscaling, disruption budgets, configuration, secrets, and namespaces.
- Deploy stateful components on Kubernetes only as a deliberate learning exercise; managed data services can be more practical for a cloud environment.
- Test pod termination, graceful shutdown, in-flight requests, worker acknowledgements, and consumer rebalancing.

### GCP
- GKE for Kubernetes.
- Cloud SQL for managed PostgreSQL, if selected.
- Artifact Registry for container images.
- Cloud Storage for media and RAG source documents.
- IAM and service accounts with least privilege.
- Cloud Load Balancing and networking as appropriate.
- Secret Manager for sensitive configuration.
- Budget alerts, quotas, and teardown procedures to prevent surprise costs.

### Terraform
Terraform to define repeatable infrastructure and environments. Keep modules and state handling deliberate, protect state, review plans, and avoid committing secrets.
~~~
/infra/terraform
  /modules
    networking/
    gke/
    database/
    storage/
    iam/
  /environments
    dev/
    staging/
~~~
Start with a small local/dev deployment, then provision a minimal GCP environment. Not every component needs a managed cloud equivalent in the first deployment.


## 12. Observability: Prometheus, Grafana, and Tracing
Prometheus for collecting metrics; Grafana visualizes them. Add structured logs and distributed tracing (for example, OpenTelemetry) so metrics can lead to a concrete request or event path.

- HTTP request count, error rate, latency histograms, and in-flight requests.
- PostgreSQL pool utilization, query duration, timeouts, and transaction failures.
- Redis latency, cache hit/miss rate, memory, and connection errors.
- Kafka consumer lag, processing duration, retries, and dead-letter counts.
- RabbitMQ queue depth, unacknowledged messages, delivery failures, and worker throughput.
- WebSocket active connections and message fan-out latency.
- RAG retrieval latency, retrieval quality evaluations, token/cost metrics, and answer feedback.

Create dashboards and alerts for API health, game-event pipeline health, queue backlog, database saturation, and deployment health. Avoid high-cardinality metric labels such as raw user IDs or game IDs.


## 13. CI/CD and Software Quality
~~~
Pull request
  -> formatting and lint
  -> unit tests
  -> integration tests
  -> migration checks
  -> security/dependency scan
  -> build binary and container
  -> publish versioned image
  -> deploy to dev/staging
  -> smoke tests
  -> controlled production-like rollout
~~~

- Use GitHub Actions or an equivalent pipeline.
- Run Go tests, go vet, linting, race detection where appropriate, and integration tests against real dependencies.
- Test database migrations and backward compatibility before deploying new code.
- Use immutable image tags and retain build metadata.
- Separate build permissions from deployment permissions and use short-lived cloud identity where available.
- Practice rollback, failed deployment recovery, and safe schema evolution.


## 14. RAG: Scratcho Basketball Intelligence

RAG should be a grounded knowledge feature, not an unbounded chatbot. It combines structured Scratcho data with trusted unstructured documents.

~~~ text
Trusted documents + approved Scratcho content
                |
        ingestion / cleaning
                |
        chunking + metadata
                |
           embeddings
                |
       vector index (pgvector initially)
                |
User question -> query classification
                |
       structured query and/or retrieval
                |
       context + source metadata
                |
             LLM
                |
      answer + source references
~~~

- Use SQL for exact current statistics, standings, scores, and structured comparisons whenever possible.
- Use vector retrieval for rules, historical documents, articles, interviews, and explanatory text.
- Add metadata filters for competition, season, team, player, publication date, and source trust.
- Cite retrieved sources and show data timestamps; abstain when evidence is weak.
- Protect against prompt injection in ingested documents and do not let retrieved text override system or authorization rules.
- Evaluate retrieval recall, groundedness, answer correctness, latency, and cost using a fixed question set.
- Keep private/admin-only content out of public retrieval results by enforcing access control before context reaches the model.


## 15. Security, Reliability, and Correctness
- Use role-based authorization for admin operations and enforce authorization at the service/domain layer, not only in the UI.
- Validate game state transitions and use optimistic locking or row-level locking where concurrent edits are possible.
- Use idempotency keys for game-event submissions, imports, notification dispatch, and other retryable mutations.
- Use the transactional outbox to avoid committing a game result without recording the event intended for downstream consumers.
- Design consumers for duplicate delivery and out-of-order events; reconcile derived statistics against source records.
- Use timeouts, bounded retries with jitter, cancellation, graceful shutdown, and controlled fallbacks.
- Maintain audit trails for privileged edits and corrections to published scores.
- Use secret management, dependency scanning, rate limits, secure headers, and least-privilege service identities.
- Back up PostgreSQL and test restoring it; document recovery objectives even for a learning environment.


## 16. Current / Expected Bottlenecks

| Likely bottleneck | What to measure or learn | Possible later response |
|----------|----------|----------|
| Repeated homepage reads  |Query latency, DB CPU, request volume.    | Indexes, query tuning, Redis cache, precomputed read models.     |
| Expensive GraphQL requests   | Resolver count, query complexity, N+1 behavior.   | Data loaders, complexity limits, batching, persisted queries.    |
| Statistics recalculation   | 	Processing time and DB write contention.     | Incremental projections, Kafka consumers, batch jobs.     |
| Duplicate game events    | Duplicate IDs, conflicting scores, retry behavior.    | Idempotency, constraints, event deduplication.    |
| Consumer backlog    | 	Kafka lag and processing throughput.    | Partition/consumer tuning, efficient handlers, replay strategy.     |
| Notification spikes    | Queue depth, provider latency, delivery failure rate.    | RabbitMQ workers, bounded concurrency, retries, DLQs.     |
| Multi-replica WebSockets    | Missed broadcasts and reconnect behavior.     | Shared event distribution and resynchronization API.     |
| Cloud deployment complexity    | Deployment failures, resource pressure, recovery time.     | Kubernetes probes, autoscaling, rollout strategy, Terraform.     |
| RAG answer quality    | Retrieval relevance and unsupported claims.     | Better chunking, metadata filters, hybrid retrieval, evaluation set.     |



## 17.  Implementation Roadmap
| Phase | Build / learn | Exit criteria |
|----------|----------|----------|
| 0. Domain and design   | Requirements, domain model, API contracts, ERD, ADRs, threat model.     | Core entities, invariants, and boundaries are documented.    |
| 1. Go monolith + PostgreSQL   | Auth, leagues, teams, players, games, results, standings, content basics; migrations and tests.     | Deployable service with reliable core flows.    |
| 2. Production-quality Go    | Layering, context, config, logging, validation, transactions, pagination, integration tests.     | Observable, testable service with clear module seams.     |
| 3. REST + GraphQL    | REST resource APIs; GraphQL aggregation, batching, complexity limits.     | Both APIs serve real use cases and share domain rules.     |
| 4. Docker + CI/CD    | Container images, Compose, automated test/build pipeline.     | One-command local setup and repeatable CI.     |
| 5. Redis    | Caching, TTL/invalidation, rate limiting, sorted-set leaderboard experiment.     | Benchmarks show impact and fallback behavior is tested.     |
| 6. Kafka + outbox    | Game lifecycle events, schema evolution, consumer groups, replay, idempotency.     | At least two independent consumers process events safely.     |
| 7. RabbitMQ workers   | Notifications, imports, reports, retries, DLQs.     | Failed jobs recover safely and poison jobs are observable.     |
| 8. Real-time updates   | WebSockets, authorized subscriptions, reconnect/resync, multi-instance fan-out.     | Live game updates remain consistent after reconnect.     |
| 9. gRPC + service extraction    | Extract Statistics or Standings; protobuf contracts and deadlines.     | Independent service can be deployed/tested and failures are handled.     |
| 10. Observability   | Prometheus, Grafana, structured logs, OpenTelemetry traces.     | Dashboards show API, DB, cache, queue, and event pipeline health.     |
| 11. Kubernetes    | Deployments, probes, graceful shutdown, autoscaling, rollbacks.     | Failure and rolling-deployment exercises pass.     |
| 12. Terraform + GCP   | Provision network, compute, registry, database/storage, IAM.     | Environment can be recreated and torn down from code.    |
| 13. RAG    | Ingestion, embeddings, retrieval, grounded answers, evaluation.     | Answers cite sources and abstain when evidence is insufficient.     |
| 14. Load/failure testing    | Load tests, dependency failures, duplicate events, lag, DB pressure, restore drills.     | Documented SLO results, bottlenecks, and mitigations.    |


## 18. Deliberate Failure Tests
- Stop Redis and measure the database fallback and cache-stampede risk.
- Stop a Kafka consumer, create a backlog, restart it, and verify recovery without duplicate statistics.
- Deliver the same game-finalized event more than once and prove standings are not incremented twice.
- Publish events out of order and verify the consumer detects or handles invalid transitions.
- Make a RabbitMQ worker fail repeatedly and confirm retry limits and dead-letter routing.
- Kill an API pod during traffic and verify readiness, graceful shutdown, and retry behavior.
- Apply a migration that requires compatibility across old and new application versions; practice expand-and-contract.
- Load-test a GraphQL query and confirm complexity limits protect the service.
- Restore PostgreSQL from backup into a clean environment and validate key records.
- Ask RAG questions with missing or contradictory evidence and confirm it does not fabricate certainty.


## 19. Architecture Decision Records to Maintain
- Why PostgreSQL is the source of truth and which data is derived.
- Why REST and GraphQL coexist and what each API is responsible for.
- Which internal calls use gRPC and why.
- Which events belong in Kafka and which jobs belong in RabbitMQ.
- Cache consistency, TTLs, invalidation, and fallback behavior.
- Event schema/versioning, partition keys, idempotency, and replay strategy.
- Service extraction criteria and data ownership boundaries.
- Kubernetes versus managed GCP services for each dependency.
- Vector storage and embedding model choices for RAG.
- Security, cost, and operational trade-offs for the learning environment.


## 20. My  Definition of Success
Scratcho succeeds as this learning project when it is not merely a list of technologies, but a system whose choices can be explained and whose failure modes have been tested.
- A random developer can run the monolith and dependencies locally.
- Core basketball workflows are correct and covered by unit/integration tests.
- REST and GraphQL are both used for defensible client needs.
- Redis, Kafka, RabbitMQ, and gRPC have documented roles and working demonstrations.
- Metrics, dashboards, traces, and logs make a failure diagnosable.
- The application can be containerized, deployed, rolled back, and recreated using infrastructure code.
- RAG answers are evaluated and grounded in source material.
- A portfolio write-up documents architecture diagrams, benchmarks, trade-offs, failure exercises, and lessons learned.


