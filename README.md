# ChessTracker

ChessTracker is a backend system for storing chess games, analysing positions and building player statistics. The project is split into independent Spring Boot microservices with an API Gateway planned as the public entry point.

## Project status

The system is being developed incrementally. The current codebase already contains dedicated repositories for UserService, GameService, AnalysisService, StatisticsService and shared infrastructure. The API Gateway is part of the target architecture and is responsible for public ingress and JWT validation.

## Repositories

| Component | Repository | Responsibility | Status |
|---|---|---|---|
| UserService | https://github.com/Krupkolllia/chess-tracker-user-service | Registration, login, user profile and account lifecycle | Active |
| GameService | https://github.com/Krupkolllia/chess-tracker-game-service | Chess games, PGN/game lifecycle and game domain operations | Active |
| AnalysisService | https://github.com/Krupkolllia/chess-tracker-analysis-service | Asynchronous chess analysis pipeline | In progress / planned capabilities |
| StatisticsService | https://github.com/Krupkolllia/chess-tracker-statistics-service | Player statistics and derived profile data | In progress / planned capabilities |
| Infrastructure | https://github.com/Krupkolllia/ChessTrackerInfrastructure | PostgreSQL and Kafka local infrastructure | Active |
| API Gateway | https://github.com/Krupkolllia/chess-tracker-api-gateway | Public entry point, JWT validation and request routing | Planned |

## Tech stack

### Application stack

- Java 21
- Spring Boot 4.1.1
- Spring Security 7.1.1
- Spring Security JOSE / Nimbus JOSE + JWT
- Gradle
- Spring Data JPA / Hibernate
- PostgreSQL 17
- Liquibase
- Apache Kafka 4.1.2
- Docker

### Supporting libraries and tooling

- MapStruct 1.6.3
- Lombok
- springdoc-openapi
- Testcontainers
- JUnit Platform
- Checkstyle
- `chesslib` 1.3.7 in GameService

The repository build files currently confirm Java 21 and Spring Boot 4.1.1 across the services. UserService and GameService also include OpenAPI tooling; GameService uses `chesslib`, while the shared infrastructure repository runs PostgreSQL 17 and Apache Kafka 4.1.2.

## Architecture at a glance

```text
                           Internet / Frontend
                                  |
                                  v
                         +-------------------+
                         |    API Gateway    |
                         |-------------------|
                         | Routing           |
                         | JWT validation    |
                         | User identity     |
                         +---------+---------+
                                   |
                    +--------------+--------------+
                    |              |              |
                    v              v              v
             +-------------+ +-------------+ +-------------+
             | UserService | | GameService | |   Analysis  |
             |-------------| |-------------| |   Service   |
             | Auth        | | Games       | | Async jobs  |
             | Users       | | PGN         | | Chess AI   |
             +------+------+ +------+------+ +------+------+
                    |               |                ^
                    |               |                |
                    v               |                |
               +---------+          |         +------+------+
               |Postgres |<---------+-------->|   Kafka     |
               | DB per  |                    | event bus   |
               | service  |                    +------+------+
               +---------+                           |
                                                      v
                                               +-------------+
                                               | Statistics  |
                                               | Service     |
                                               +-------------+
```

The database-per-service boundary is the target design: each service owns its data and exposes it through its API or events rather than sharing JPA entities across services.

## Core design principles

### 1. Gateway-first public API

Clients should communicate with the platform through the API Gateway. Internal services are intended to remain private to the backend network rather than being publicly exposed.

### 2. UserService issues JWTs

UserService performs username/password authentication and signs access tokens with an RSA private key. The JWT contains the authenticated user identity, currently represented by `sub = userId`, plus `iat` and `exp` claims.

### 3. Gateway validates JWTs

The API Gateway is the JWT consumer. It stores the RSA public key and validates the access token before forwarding an authenticated request to an internal service.

### 4. Internal services receive identity, not credentials

After validating the token, the Gateway forwards the trusted user identity to an internal service through an internal header such as `X-User-Id`. The service reconstructs its request-local Spring Security `Authentication` and exposes the user identity through the `SecurityContext`.

### 5. Synchronous vs asynchronous work

User/profile operations and normal game CRUD operations are synchronous HTTP operations. Expensive analysis work is designed as an asynchronous Kafka-driven flow so a game request does not have to wait for engine/AI analysis to finish.

## Main user flows

### Registration

```text
Client
  -> Gateway
  -> POST /auth/register
  -> UserService
  -> validate DTO
  -> create User
  -> PostgreSQL
  -> 201 Created
```

Registration is public from the **user authentication** perspective, but the UserService is still reachable only through the trusted backend path. No user id exists before the account is created.

### Login

```text
Client
  -> Gateway
  -> POST /auth/login
  -> UserService
  -> AuthenticationManager
  -> DaoAuthenticationProvider
  -> UserDetailsService
  -> PostgreSQL
  -> JwtEncoder (RSA private key)
  -> access token
```

### Authenticated request

```text
Client
  -> Authorization: Bearer <JWT>
  -> Gateway
  -> JwtDecoder / public RSA key
  -> JWT sub = userId
  -> X-User-Id + internal gateway trust header
  -> Internal service
  -> GatewayAuthenticationFilter
  -> SecurityContext
  -> Controller / Service
```

### Game analysis

```text
Client
  -> Gateway
  -> GameService
  -> persist game
  -> publish analysis command/event
  -> Kafka
  -> AnalysisService
  -> engine / AI analysis
  -> publish result/event
  -> consumers such as StatisticsService
```

## Security model

The security architecture is documented in detail in [`docs/SECURITY.md`](docs/SECURITY.md).

At a high level:

- Access tokens are JWTs signed with RSA.
- UserService owns the private signing key.
- Gateway owns only the public verification key.
- Internal services do not need the user's password.
- `X-User-Id` is trusted only when the request came through the internal Gateway path.
- The `SecurityContext` is request-scoped; it is not a persistent user store.
- Access tokens are currently short-lived (30 minutes in the current design).
- Refresh-token rotation/revocation and immediate access-token revocation are intentionally deferred until the basic architecture is stable.

## Documentation

- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — system architecture, service boundaries, data ownership and request flow.
- [`docs/USE-CASES.md`](docs/USE-CASES.md) — concrete end-to-end use cases.
- [`docs/SECURITY.md`](docs/SECURITY.md) — authentication, authorization, JWT and internal trust model.

## Local infrastructure

The infrastructure repository currently provides:

- PostgreSQL 17
- Apache Kafka 4.1.2
- Docker volumes for persistent local data
- a shared Docker network named `chesstracker`

Infrastructure repository:

https://github.com/Krupkolllia/ChessTrackerInfrastructure

## Development notes

Each service is an independent Gradle/Spring Boot application. This keeps service boundaries explicit and allows a service to be developed and tested independently.

For local development, the intended split is:

```text
Infrastructure Docker
  - PostgreSQL
  - Kafka

IDE / containers
  - UserService
  - GameService
  - AnalysisService
  - StatisticsService
  - API Gateway (planned)
```

## Current scope vs future scope

### Current / near-term

- User registration and login
- RSA-signed access JWTs
- Request-local authentication propagation
- User profile operations
- Game service with PostgreSQL, Liquibase and Kafka integration
- Shared PostgreSQL/Kafka infrastructure

### Planned

- API Gateway implementation as the single public ingress
- AnalysisService implementation around Stockfish/AI analysis
- StatisticsService event-driven aggregation and player profile generation
- Refresh tokens
- Token revocation strategy if the product requires immediate invalidation
- Additional observability and deployment automation

## Repository map

```text
ChessTracker
|
+-- chess-tracker-user-service
+-- chess-tracker-game-service
+-- chess-tracker-analysis-service
+-- chess-tracker-statistics-service
+-- ChessTrackerInfrastructure
\-- API Gateway (planned)
```
