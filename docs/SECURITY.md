# ChessTracker Security Architecture

## 1. Threat model

The system assumes that the public client cannot be trusted with internal service credentials. Public traffic should terminate at the API Gateway. Internal services are protected by network isolation plus an additional service-to-service trust mechanism.

The architecture separates two questions:

1. **Is this request coming from the trusted Gateway?**
2. **Which authenticated user is represented by this request?**

These are related but different concerns.

## 2. Authentication model

### Registration

Registration happens before a user has an identity. The request can therefore be permitted without a user `Authentication`, while still requiring the request to reach UserService through the trusted internal network/Gateway path.

```text
POST /auth/register

X-Gateway-Secret = valid
X-User-Id        = absent
```

The absence of `X-User-Id` is expected here.

### Login

Login uses the traditional username/password authentication path:

```text
email + password
      |
      v
AuthenticationManager
      |
      v
DaoAuthenticationProvider
      |
      v
JpaUserDetailsService
      |
      v
UserRepository
```

After successful authentication, the service obtains the user's database ID and creates an access token.

## 3. JWT design

The current design uses RSA asymmetric signing.

```text
UserService
  private RSA key
       |
       | sign
       v
     JWT
       |
       v
API Gateway
  public RSA key
       |
       | verify
       v
 authenticated request
```

The intended claims are:

```json
{
  "sub": "<userId>",
  "iat": "<issued-at>",
  "exp": "<expiration>"
}
```

The current access-token target lifetime is 30 minutes.

### Why asymmetric signing?

The private key never needs to be shared with the Gateway or business services. UserService can sign tokens, while the Gateway only needs the public key to verify them.

## 4. Token issuance implementation

UserService uses Spring Security JOSE and `NimbusJwtEncoder` rather than JJWT.

Conceptually:

```text
JwtClaimsSet
      |
      v
JwtEncoderParameters
      |
      v
NimbusJwtEncoder
      |
      v
signed JWT
```

The signing key material is represented as an RSA key pair. The private key is treated as a secret and must not be committed to source control.

## 5. Gateway token validation

The API Gateway is the JWT consumer.

```text
Authorization: Bearer <token>
             |
             v
       JwtDecoder
             |
             v
      signature valid?
             |
             v
        read sub
```

The Gateway should reject malformed, expired or incorrectly signed tokens before the request reaches internal services.

## 6. User identity propagation

After JWT validation, the Gateway propagates the authenticated user ID to an internal service:

```http
X-User-Id: 157
```

The client must not be able to choose this identity. The Gateway should overwrite any externally supplied identity header with the value derived from the validated JWT.

The internal service also checks a shared Gateway trust value (for example `X-Gateway-Secret`) so that a forged identity header sent directly to the service does not become trusted on its own.

## 7. GatewayAuthenticationFilter

The internal service uses a request filter with logic equivalent to:

```text
1. Read gateway trust header.
2. Reject if the value is invalid.
3. Read X-User-Id.
4. If X-User-Id is absent, continue without a user Authentication.
5. If present, parse it as Long.
6. Create PreAuthenticatedAuthenticationToken(userId, ...).
7. Store it in the current SecurityContext.
8. Continue the filter chain.
```

The filter intentionally does **not** inspect the URI to decide whether a request is registration or not. The endpoint authorization rules handle that separately.

## 8. SecurityContext

The `SecurityContext` contains the authenticated identity for the **current request**.

It is not the database, not the JWT and not a cross-request session in this stateless architecture.

```text
JWT / X-User-Id
      |
      v
current request
      |
      v
SecurityContext
      |
      v
Controller / Service
```

A subsequent request reconstructs the security state again.

## 9. Authorization

Current UserService authorization rules conceptually look like:

```java
.requestMatchers("/auth/login", "/auth/register").permitAll()
.anyRequest().authenticated()
```

### `permitAll()`

The endpoint does not require an authenticated user identity.

This is appropriate for registration and login.

### `authenticated()`

An authenticated `Authentication` must already exist in the `SecurityContext`.

This is appropriate for profile, password, account and game operations.

`permitAll()` does not mean the internal UserService should be exposed directly to the internet. Network isolation and service trust happen at a separate layer.

## 10. Network security

The preferred deployment boundary is:

```text
Internet
   |
   v
API Gateway
   |
   | private backend network
   v
Internal services
```

UserService and other microservices should not be published directly to the public network in production.

The infrastructure repository currently provides a shared Docker network for local development. Production deployment should preserve the same conceptual boundary even if the network technology changes.

## 11. Password security

Passwords are never stored in plaintext.

The current UserService uses Spring Security's `PasswordEncoder` infrastructure and a delegating encoder factory. Registration encodes the password before persistence. Login delegates password verification to `DaoAuthenticationProvider`.

The password is therefore only required at login time. It is not sent on normal authenticated requests.

## 12. Email normalization

Email is treated as the login identifier. The current design normalizes it consistently before registration lookup and authentication lookup, using trimming and deterministic lower-casing.

This is an application policy, not a universal statement about every email provider. Provider-specific aliases such as Gmail dot/plus behavior are intentionally not collapsed by application logic.

## 13. Soft deletion and stale access tokens

The User entity uses soft-delete semantics so deleted users are excluded from normal application queries.

Because the current access token is stateless, deleting the user does not automatically invalidate an already-issued JWT. The token remains cryptographically valid until its expiration time.

This is an acknowledged limitation of the current stage, not a hidden guarantee.

## 14. Refresh tokens

Refresh tokens are not part of the current minimal authentication implementation.

A future design can use:

```text
Access token   -> short-lived JWT
Refresh token -> longer-lived credential, persisted/revocable
```

This allows the user to keep a session without using a multi-day access token and allows server-side revocation of the long-lived credential without maintaining a blacklist for every access token.

## 15. Logout

With access-token-only authentication, clearing `SecurityContextHolder` only affects the current request. It does not revoke a JWT already stored by the client.

Therefore the current practical logout behavior is:

```text
logout request
     |
     v
client deletes access token
```

A future refresh-token flow can make logout server-aware by revoking the refresh credential.

## 16. Security assumptions to preserve

- Never commit RSA private keys.
- Never trust a client-supplied `X-User-Id` directly.
- Gateway must derive user identity from a validated JWT.
- Internal services must not be publicly reachable in production.
- Passwords must only travel over TLS and only to the authentication boundary.
- Do not treat `SecurityContext` as persistent storage.
- Do not duplicate JWT parsing logic in every business service unless there is a concrete architectural reason.
- Keep authentication and authorization separate: authentication establishes identity, authorization decides whether that identity may access the endpoint.
