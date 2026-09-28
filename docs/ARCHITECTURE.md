# ChessTracker Architecture

## 1. Overview

ChessTracker is designed as a set of independently deployable Spring Boot microservices. The system separates authentication, user management, game management, analysis and statistics so that each domain owns its own persistence and application logic.

The central architectural boundary is an API Gateway. The Gateway is the public ingress point and is responsible for validating access tokens and propagating authenticated user identity to internal services.

## 2. Logical service map

```text
                                     +----------------+
                                     |    Client      |
                                     +-------+--------+
                                             |
                                             v
                                  +----------------------+
                                  |     API Gateway      |
                                  |----------------------|
                                  | Routing              |
                                  | JWT validation       |
                                  | User identity        |
                                  +----+----+----+-------+
                                       |    |    |
                         +-------------+    |    +----------------+
                         |                  |                     |
                         v                  v                     v
                 +---------------+  +---------------+  +-------------------+
                 | UserService   |  | GameService   |  | StatisticsService |
                 |---------------|  |---------------|  |-------------------|
                 | Auth          |  | Games / PGN   |  | Aggregation       |
                 | Users         |  | Game state    |  | Player profile    |
                 +-------+-------+  +-------+-------+  +-------------------+
                         |                  |                     ^
                         v                  |                     |
                  +-------------+           |                     |
                  | User DB     |           |                     |
                  | PostgreSQL  |           |                     |
                  +-------------+           |                     |
                                           v
                                  +-------------------+
                                  |       Kafka        |
                                  |-------------------|
                                  | async commands /  |
                                  | domain events     |
                                  +---------+---------+
                                            |
                                            v
                                  +-------------------+
                                  |  AnalysisService  |
                                  |-------------------|
                                  | Stockfish / AI    |
                                  | analysis pipeline |
                                  +-------------------+
```

The precise event/topic names may evolve with implementation. The important boundary is that Kafka is used where processing should not block the original HTTP request.

## 3. Service responsibilities

### UserService

Owns:

- user registration;
- login and password verification;
- user profile management;
- password changes;
- account deletion/soft deletion;
- issuing access JWTs.

Authentication is implemented with Spring Security's `AuthenticationManager`, `DaoAuthenticationProvider` and a JPA-backed `UserDetailsService`. JWT issuance uses Spring Security JOSE / `NimbusJwtEncoder` with an RSA key pair.

Repository: https://github.com/Krupkolllia/chess-tracker-user-service

### GameService

Owns:

- chess game persistence;
- PGN/game data;
- game lifecycle and game-specific validation;
- publishing events/commands for asynchronous analysis;
- consuming or exposing game-related operations.

The current service uses PostgreSQL, Liquibase, Kafka and `chesslib`.

Repository: https://github.com/Krupkolllia/chess-tracker-game-service

### AnalysisService

Owns the asynchronous analysis workflow. The target design is to consume analysis requests from Kafka, process chess positions/games using an engine such as Stockfish and optionally enrich the analysis with a language/AI model, then publish a result event for downstream consumers.

Repository: https://github.com/Krupkolllia/chess-tracker-analysis-service

### StatisticsService

Owns derived statistics and player-level aggregates. The target architecture is event-driven: game events and analysis results are consumed and transformed into a persistent statistics model, avoiding tight synchronous coupling with GameService.

Repository: https://github.com/Krupkolllia/chess-tracker-statistics-service

### Infrastructure

Provides local shared infrastructure for PostgreSQL and Kafka. The current compose file uses PostgreSQL 17 and Apache Kafka 4.1.2 and puts them on the `chesstracker` Docker network.

Repository: https://github.com/Krupkolllia/ChessTrackerInfrastructure

## 4. Database ownership

Each service should own its database/schema and be the only service allowed to write its domain data directly.

```text
UserService        -> user data
GameService        -> game data
AnalysisService    -> analysis data
StatisticsService  -> derived statistics
```

A shared PostgreSQL server/container does **not** imply shared application ownership. The service boundary is still logical and should be preserved by using separate databases or schemas and service-owned migrations.

This is preferable to cross-service JPA relations such as `@ManyToOne User` from GameService to UserService.

## 5. Synchronous communication

Use synchronous HTTP when the caller needs an immediate answer.

Examples:

- login;
- registration;
- profile retrieval;
- changing a username/display name;
- changing a password;
- deleting an account;
- reading or modifying a game when no long-running processing is required.

## 6. Asynchronous communication

Use Kafka when an operation can complete independently from the original request.

Typical flow:

```text
GameService
    |
    | analysis command/event
    v
  Kafka
    |
    v
AnalysisService
    |
    | analysis result event
    v
  Kafka
    |
    +----------> StatisticsService
```

Advantages:

- HTTP request latency is decoupled from engine/AI processing;
- analysis can be retried independently;
- multiple consumers can react to the same event;
- services remain loosely coupled.

## 7. Request flow through the Gateway

### Public request

```text
Client
  |
  | HTTP request
  v
Gateway
  |
  | route
  v
Internal service
```

### Authenticated request

```text
Client
  |
  | Authorization: Bearer <JWT>
  v
Gateway
  |
  | JwtDecoder + RSA public key
  |
  | validated JWT, sub = userId
  |
  | X-User-Id: <userId>
  | internal gateway trust header
  v
Internal service
  |
  | GatewayAuthenticationFilter
  |
  | PreAuthenticatedAuthenticationToken(userId)
  v
SecurityContext
  |
  v
Controller / Service
```

## 8. Registration flow

Registration is special because no user identity exists yet.

```text
Client
  |
  | POST /auth/register
  v
Gateway
  |
  | trusted internal request
  | (no X-User-Id yet)
  v
UserService
  |
  | permitAll()
  |
  | validate request
  | normalize email
  | encode password
  | persist User
  v
PostgreSQL
```

The user's ID is created by the database during persistence. After registration, login produces the first JWT containing that ID.

## 9. Login flow

```text
Client
  |
  | email + password
  v
Gateway
  |
  v
UserService
  |
  | AuthenticationManager
  v
DaoAuthenticationProvider
  |
  v
JpaUserDetailsService
  |
  v
UserRepository -> PostgreSQL
  |
  v
Authenticated User
  |
  | user.getId()
  v
JwtService
  |
  v
NimbusJwtEncoder
  |
  | RSA private key
  v
JWT { sub, iat, exp }
```

## 10. User request flow after login

```text
Client
  |
  | Bearer access token
  v
Gateway
  |
  | validate signature
  | extract sub
  v
Internal service
  |
  | validate gateway trust
  | read X-User-Id
  v
GatewayAuthenticationFilter
  |
  | Authentication(principal = userId)
  v
SecurityContext
  |
  v
SecurityUtil.getAuthenticatedUserId()
  |
  v
UserRepository.findById(userId)
```

The `SecurityContext` is request-local. It is not a persistent store and does not survive into the next HTTP request.

## 11. Deployment boundaries

The target deployment should expose only the Gateway to the public network.

```text
Public network
      |
      v
+-------------+
| API Gateway |
+------+------+
       |
       | private backend network
       +------------------------------+
       |              |               |
       v              v               v
  UserService    GameService    AnalysisService
       |              |               |
       +--------------+---------------+
                      |
                      v
               PostgreSQL / Kafka
```

Internal service ports should not be published to the internet. Network isolation is the primary boundary; application-level internal trust headers are an additional layer.

## 12. Architectural constraints

- No service should read another service's database directly.
- No service should receive the user's password except the authentication boundary.
- UserService owns JWT signing keys and token issuance.
- Gateway owns JWT verification.
- Internal services should trust propagated user identity only from the protected Gateway path.
- Long-running chess analysis should not block synchronous game CRUD requests.
- Domain models should stay service-local; share contracts/events, not JPA entities.
