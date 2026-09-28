# ChessTracker Use Cases

## 1. Register a new account

**Actor:** unauthenticated client

**Goal:** create a ChessTracker account.

**Flow:**

1. Client sends `POST /auth/register` with email, password, password confirmation and display name.
2. Gateway forwards the request to UserService.
3. UserService validates the DTO.
4. Email is normalized before persistence.
5. UserService checks whether the email is already registered.
6. Password is encoded with the configured `PasswordEncoder`.
7. The user is stored in PostgreSQL.
8. The API returns `201 Created` with the created user representation.

**Security state:** there is no authenticated user yet, so `X-User-Id` is absent.

---

## 2. Log in

**Actor:** registered user

**Goal:** obtain an access token.

**Flow:**

1. Client sends `POST /auth/login` with email and password.
2. UserService normalizes the email.
3. `AuthenticationManager` delegates to `DaoAuthenticationProvider`.
4. `JpaUserDetailsService` loads the user by email.
5. The password is verified.
6. UserService extracts `user.getId()` from the authenticated principal.
7. `JwtService` creates a JWT with the user ID in `sub`.
8. `NimbusJwtEncoder` signs the JWT with the RSA private key.
9. The token is returned to the client.

**Current token model:** access token only, with a target lifetime of 30 minutes.

---

## 3. Read the current profile

**Actor:** authenticated user

**Goal:** retrieve the current user's profile.

**Flow:**

```text
Client
  -> Bearer JWT
  -> Gateway validates JWT
  -> Gateway extracts sub
  -> X-User-Id propagated internally
  -> UserService GatewayAuthenticationFilter
  -> SecurityContext
  -> UserService getMe()
  -> UserRepository.findById()
```

The service does not need the user's password and does not need to decode the original JWT itself.

---

## 4. Change display name

**Actor:** authenticated user

**Goal:** update the profile display name.

**Flow:**

1. Request reaches UserService through the Gateway.
2. The Gateway propagates the authenticated user ID.
3. `SecurityUtil.getAuthenticatedUserId()` returns the request-local identity.
4. UserService loads the user in the active transaction.
5. The display name is changed.
6. Hibernate dirty checking persists the update at transaction commit.

No extra `save()` is required when the loaded entity is managed inside the transaction.

---

## 5. Change password

**Actor:** authenticated user

**Goal:** replace the current password.

**Flow:**

1. Load the authenticated user by user ID.
2. Verify the old password with `PasswordEncoder.matches(...)`.
3. Encode the new password.
4. Update the managed entity inside the current transaction.
5. Commit the transaction.

If the old password is wrong, the operation fails without changing the stored password.

---

## 6. Delete the account

**Actor:** authenticated user

**Goal:** remove the account from normal application visibility.

**Flow:**

1. UserService gets the authenticated user ID from the security context.
2. UserService loads the User entity.
3. A transactional delete operation is executed.
4. The current model uses soft-delete semantics (`is_deleted`) so the deleted user is excluded from normal queries.

**Important consequence:** the current access JWT may remain cryptographically valid until its expiration time. Immediate token revocation is deliberately deferred.

---

## 7. Create / update a chess game

**Actor:** authenticated user

**Goal:** persist chess-game data.

**Flow:**

1. Request reaches GameService through the Gateway.
2. GameService receives the authenticated user ID.
3. GameService validates the game/PGN payload.
4. GameService persists the game in its own database.
5. If analysis is requested, GameService publishes an asynchronous analysis command/event.
6. The HTTP response does not have to wait for the analysis to finish.

---

## 8. Run chess analysis

**Actor:** authenticated user / GameService

**Goal:** obtain engine/AI analysis for a game.

**Flow:**

```text
GameService
    |
    | analysis command
    v
 Kafka
    |
    v
AnalysisService
    |
    +--> Stockfish / engine
    |
    +--> optional AI enrichment
    |
    v
analysis result event
    |
    +---------------------> GameService / StatisticsService
```

The analysis workflow is asynchronous so slow computation does not block the original game request.

---

## 9. Build player statistics

**Actor:** background event processing

**Goal:** maintain derived statistics and a player profile.

**Flow:**

1. GameService emits game-related events.
2. AnalysisService emits analysis-related events when analysis completes.
3. StatisticsService consumes relevant events.
4. StatisticsService updates its own aggregate model.
5. Profile/statistics endpoints read only StatisticsService-owned data.

This keeps statistics computation decoupled from the core game write path.

---

## 10. Logout

**Current scope:** client-side access-token removal.

A JWT access token is self-contained and does not automatically disappear from the client when a server request returns `204`.

Therefore the practical current flow is:

```text
Client
  -> POST /auth/logout (optional endpoint)
  -> remove access token locally
```

A future refresh-token design can add server-side token revocation without introducing a blacklist for every access token.

---

## 11. Invalid / missing identity

For a protected internal endpoint:

```text
X-Gateway-Secret present
X-User-Id missing
        |
        v
No Authentication created
        |
        v
.anyRequest().authenticated()
        |
        v
403 / access denied
```

This is expected. Registration and login are explicitly permitted because no user identity exists yet.

---

## 12. Account deleted while an access token is still valid

This is a known temporary edge case of stateless access tokens:

```text
Account 157 deleted
       |
       +--> old JWT still cryptographically valid until exp
                       |
                       v
              next authenticated request
                       |
                       v
             UserService cannot find 157
```

The application can then treat the authenticated identity as invalid and return an authentication-related failure. Immediate invalidation can be added later with refresh-token revocation, token versioning, introspection or another explicit revocation mechanism.
