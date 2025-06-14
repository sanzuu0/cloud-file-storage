# Auth Service — Flow Overview

This document describes the key interaction scenarios with the Auth Service, including user registration, login, token refresh, token validation, and logout. It is intended for developers, security engineers, and anyone who needs to understand how authentication works across the platform.

### Covered:

* step-by-step breakdown of each request
* involved components and communication flow
* what data is persisted and what events are published
* architecture diagrams (where applicable)
* summary table of all supported flows

Each flow is documented separately for clarity, ease of testing, and maintainability.

---

## Overview

The Auth Service implements essential authentication workflows such as login, token rotation, access validation, and session termination. These flows form the backbone of platform security and are used both by external clients (UI, CLI) and internal microservices via the API Gateway or gRPC.

Below are the main flows handled by the Auth Service:

### User Registration (`/register`)

A user submits an email and password. The Auth Service validates the input, hashes the password, and calls the User Service to create a new user record. Upon success, it issues an access and a refresh token. An event `user.registered` is published to Kafka. Used by UI/CLI via REST.

---

### Login (`/login`)

The user provides an email and password. The Auth Service verifies the credentials, generates an access/refresh token pair, stores the refresh token in Redis with a TTL, and returns the access token to the client. A `user.logged_in` event is published to Kafka. Invoked via REST (API Gateway) by external clients.

---

### Token Refresh (`/refresh`)

When the access token expires, the client submits a refresh token. The Auth Service retrieves it from Redis, validates it, and issues a new access/refresh token pair, updating the session in Redis. This flow supports persistent login without re-authentication and includes protection against token reuse (rotation).

---

### Token Validation (gRPC `ValidateToken`)

Used internally by microservices via gRPC calls. The Auth Service validates the JWT’s signature (HMAC), expiration time (`exp`), and optionally the token ID (`jti`) if a denylist is implemented. Invalid tokens result in access denial.

---

### Logout (`/logout`)

The client requests logout by submitting the refresh token. The Auth Service removes the session from Redis. If denylist is enabled, the access token’s `jti` is added to Redis with its remaining TTL. Used to forcibly terminate a user session.

---

### Token Validation in HTTP Requests (API Gateway Middleware)

All REST requests passing through the API Gateway are automatically validated. The middleware extracts the access token, checks its signature, expiration, and legitimacy. This secures both public and private endpoints.

---

### System Events & Integrations

All key user actions (registration, login, refresh, logout) are published as Kafka events (`user.*`). These events support:

* user behavior analytics (via ML detectors),
* audit trail logging (via Audit Service),
* triggering security alerts (e.g., suspicious activity or anomaly detection).

---

We will document each flow in a consistent, engineering-focused format. Let’s start with the first five, described below in strict FAANG-style format.

---

## Flows

Each flow includes:

* Who initiates the request (UI, CLI, internal service)
* What data is passed in
* Which components are involved (Auth, Redis, Vault, Kafka)
* Step-by-step breakdown of all operations and checks
* What output/result is returned

---

### Flow: User Registration

**Initiator:** User (UI/CLI)
**Type:** REST → Auth Service → gRPC → User Service

**Steps:**

1. Client sends `email` and `password` to the `/register` endpoint.
2. Auth Service validates the email format and password strength.
3. The password is hashed using argon2.
4. Auth Service calls `UserService.CreateUser` via gRPC to create a new user.
5. Upon success, the following tokens are issued:

   * Access Token (JWT, 10-minute expiry)
   * Refresh Token (UUID, 10-day TTL)
6. Refresh Token is saved to Redis:
   `session:{uuid} → userID + metadata`, with TTL.
7. JSON response is returned with both access and refresh tokens.
8. `user.registered` event is published to Kafka (includes `user_id`, `email`, `ip`, `user_agent`).

---

### Flow: User Login

**Initiator:** User (UI/CLI)
**Type:** REST → Auth Service → gRPC → User Service

**Steps:**

1. Client sends `email` and `password` to the `/login` endpoint.
2. Auth Service validates email format.
3. Calls `UserService.GetUserByEmail` via gRPC.
4. Password hash is verified using **argon2**.
5. Upon successful authentication:

   * Access Token (JWT) is generated
   * Refresh Token (UUID) is generated
6. Refresh Token is stored in Redis:
   `session:{uuid} → userID + metadata`, with TTL.
7. JSON response is returned with both tokens.
8. `user.logged_in` event is published to Kafka (includes `user_id`, `ip`, `geo`, `device`).

---

### Flow: Logout

**Initiator:** User (UI/CLI)
**Type:** REST → Auth Service → Redis

**Steps:**

1. Client sends the refresh token to the `/logout` endpoint.
2. Auth Service looks up the session in Redis.
3. Deletes the session: `DEL session:{uuid}`.
4. Adds the access token’s `jti` to the denylist in Redis. TTL = access token’s `exp`.
5. Publishes `user.logged_out` event to Kafka.

---

### Flow: Token Refresh

**Initiator:** User (UI/CLI)
**Type:** REST → Auth Service → Redis

**Steps:**

1. Client sends a refresh token to the `/refresh` endpoint.
2. Auth Service checks Redis for the corresponding session: `session:{uuid}`.
3. If valid:

   * Generates a new pair of access and refresh tokens.
   * Deletes the old session: `DEL session:{old}`.
   * Saves the new session: `session:{new} → userID + metadata`.
4. Returns the new token pair to the client.
5. Publishes `user.token_refreshed` event to Kafka.

---

### Flow: Token Validation (gRPC Interceptor)

**Initiator:** Internal Microservices
**Type:** gRPC → Auth Interceptor

**Steps:**

1. Incoming gRPC request is received by the target service.
2. Auth Interceptor extracts the JWT from metadata (`authorization`).
3. JWT is validated:

   * Signature (HMAC / RSA)
   * Expiration (`exp`)
   * Claims (`jti`, `sub`, `iat`, `exp`)
4. `jti` is checked against the denylist (if enabled).
5. On success — userID is injected into request context.
6. On failure — request is rejected with `Unauthenticated` error.

---

### Flow: Token Rotation

**Initiator:** User (UI/CLI), via `/refresh`
**Type:** REST → Auth Service → Redis → Vault → Kafka

**Steps:**

1. Client sends a `refresh_token` to the `/refresh` endpoint.
2. Auth Service validates the token format (UUID) and checks for its existence in Redis.
3. If the token exists and is not expired, it is considered valid.
4. New tokens are generated:
   * New `access_token` (JWT, 10-minute expiry)
   * New `refresh_token` (UUID, 10-day TTL)
5. The old refresh token is deleted from Redis.
6. The new refresh token is saved to Redis:
   * `key: session:{uuid}`
   * `value: userID, IP, user-agent, issued_at`
   * with TTL.
7. A JSON response with new tokens is returned to the client.
8. `auth.token.rotated` event is published to Kafka (includes `user_id`, `ip`, `device`).

---

### Flow: Session Listing

**Initiator:** User (UI/CLI)
**Type:** REST → Auth Service → Redis

**Steps:**

1. Client calls `/sessions` with a valid `access_token`.
2. Auth Service extracts `user_id` from the JWT payload.
3. Searches Redis for all keys matching `session:{uuid}` associated with that `user_id`.
4. Retrieves session metadata: IP, user-agent, issued time, and TTL.
5. Builds a list of active sessions.
6. Returns a JSON array (e.g., latest 10 sessions/devices).

---

### Flow: Forced Logout

**Initiator:** ML Detector, Admin Panel, or Policy Engine
**Type:** gRPC → Auth Service → Redis → Kafka

**Steps:**

1. Initiator calls `AuthService.ForceLogout(user_id)` via gRPC.
2. Auth Service finds all refresh token sessions in Redis linked to `user_id`.
3. Deletes each session: `DEL session:{uuid}`.
4. Optionally, adds all associated access tokens' `jti` values to the denylist.
5. Publishes `auth.forced_logout` event to Kafka with:
   * `user_id`, reason, initiator (`admin`, `ml`, `system`)
6. Any subsequent refresh will fail (session missing).
7. Access token validation will fail (token blacklisted).

---

### Flow: Token Blacklist Check

**Initiator:** Any gRPC service or API Gateway
**Type:** gRPC → Auth Service → Redis

**Steps:**

1. Service sends `access_token` (JWT) for validation.
2. Auth Service extracts the `jti` from the token payload.
3. Checks Redis for key `blacklist:{jti}`:
   * If present → token is considered revoked or expired.
   * If absent → proceed with standard validations.
4. On denial, returns `TokenRevoked` or `Unauthorized`.

---

### Flow: Token Signing

**Initiator:** Auth Service (internally, during login/refresh/rotation)
**Type:** Internal → Vault

**Steps:**

1. Auth Service prepares JWT payload:
   * `sub`, `email`, `iat`, `exp`, `jti`, `roles`, `ip`, `user_agent`
2. Calls Vault API (e.g., `/v1/transit/sign/auth`) with the payload.
3. Vault signs it using the `auth-signing-key` (HMAC or RSA).
4. Returns the signed JWT to Auth Service.
5. JWT is sent to the client.

---

### Flow: Key Fetching from Vault

**Initiator:** Auth Service (on startup or hot reload)
**Type:** Internal → Vault

**Steps:**

1. On startup, Auth Service queries Vault:
   * e.g., `/v1/transit/keys/auth-signing-key/config`
2. Retrieves public part of the HMAC or RSA key.
3. Caches the key in memory (supports TTL or signal-based refresh).
4. Used for JWT signature verification during validation flow.
5. On key rotation, Vault notifies or Auth periodically fetches the config again.

---

### Flow: Kafka Event Publication

**Initiator:** Auth Service (on login, register, refresh, forced logout)
**Type:** Internal → Kafka (asynchronous, outbox-compatible)

**Steps:**

1. After a successful operation (e.g., user registration), an event is constructed:
   * `event_type`, `user_id`, `email`, `ip`, `user_agent`, `timestamp`
2. The event is serialized in Avro or JSON (depending on the platform).
3. Published to Kafka (`auth.events` or `user.activity` topic).
4. Consumed by downstream services:
   * Audit Service → for compliance logs
   * ML Detector → for behavioral analysis
   * Notification Service → for push/email alerts
5. Future version may use an outbox pattern (PostgreSQL + CDC + Kafka Connect) for reliable delivery.

---