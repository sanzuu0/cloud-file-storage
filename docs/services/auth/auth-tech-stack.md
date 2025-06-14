## Context

The `auth-service` is a critical component of a cloud file storage platform, performing user authentication, token generation, session management, and security event transmission. It is not only used within a single microservice, but serves as an entry point for all API interactions and protects access to all sensitive operations in the system.

### Where auth-service is used
- API Gateway calls it to authorize incoming REST requests (login, register, refresh).
- gRPC services check token validity during inter-service communication.
- ML-services analyze login events to detect anomalies.
- Audit-service stores login and registration events.
- CLI tools (fsctl) use auth to retrieve tokens.


### What should be supported in the future:

| Direction | Description |
| ----------------- | ------------------------------------------------------------------------------------- |
| Security | Storing secrets in Vault, JWT signature, key rotation, IP-binding, device fingerprints |
| Performance | High load on login bursts (up to 10k RPS), short latency |
| Scalability | Stateless approach, integration with Kafka and event-driven services |
| Observability | Metrics, structured logs, traces, integration with alerting |
| Extension | RBAC/ABAC, Trusted Devices, Sessions Viewer, Consent Management |

---

## Alternatives

Consider the possible alternatives, referring to the requirements: Security, Performance, Maintainability, future support: Vault, Observability, Kafka, RBAC.
The following alternatives were built against this background:

### Programming Language
| Option | Advantages | Disadvantages |
 | ------- | --------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Go | Fast, statically typed, easy to deploy, low latency, great support for gRPC and infrastructure | Less out-of-the-box high-level security abstractions, need to control everything manually |
| Java/Kotlin | Mature ecosystem, many out-of-the-box solutions, support for OIDC and OAuth2 out of the box | Harder to start and deploy, high complexity of customization, more boilerplate code |
| Python | Fast development, many libraries, familiar to many | Inefficient under load, weaker typing, not suitable for low-level security |
| Rust | Maximum security, zero-cost abstractions | Very high complexity, expensive input, too low-level for MVP / Full version |

---

### Session and token storage
| Option | Advantages | Disadvantages |
 | ----- | -------------------------------------------------------------------- | --------------------------------------------------------------- |
| Redis | Fast, TTL, simplicity, great for refresh tokens and logout | No built-in auditing, requires volume control |
| PostgreSQL | ACID, flexibility, built-in auditing| Slower under heavy load, not TTL-based |
| Vault | Centralized secure storage, key support, auditing | More complex integration, requires ACL and TLS configuration |
| Memcached | Very fast, simple | No persistence, TTL scales poorly, not good for security |

---

### Tokens and their management
| Option | Advantages | Disadvantages |
 | ------------------ | ---------------------------------------------------------- | --------------------------------------------------------------- |
| JWT (self-contained) | Stateless, easy to integrate, works with any language | Not revocable without session store, fixed lifetime |
| Session ID (stateful) | Revocable, stored in Redis/Postgres | Stateful → additional load and complexity |
| Vault-signed JWT | Centralized signing, supports key rotation and audit trail | Requires Vault setup, additional latency, more complex configuration |
| OAuth2 Proxy / Identity Server | Supports OAuth2, PKCE, social login | Severely redundant for current purposes, requires separate platform |

---

#### Interaction Protocols
| Option | Advantages | Disadvantages |
 | ---- | -------------------------------------------------------- | ------------------------------------------------------ |
| gRPC | Fast, strict contracts, good for internal calls | Requires client stub generation, not suitable for front |
| REST | Simple for external interfaces (CLI, UI) | No strict typing, slower |
| Kafka | Suitable for login events, ML, audit | Not suitable for request-response patterns; used only for event streaming |


---

### Token signing and secret storage
| Option | Advantages | Disadvantages |
 | --------------------- | ---------------------------------------------------- | ----------------------------------------------------- |
| HMAC (local key) | Simple, fast, easy to implement | Key must be stored, cannot rotate without manual operations |
| RSA/ECDSA (manual) | Sign with private key, validate with public key | Requires key storage, customization |
| Vault signing | Centralized signing, secure key storage, supports audit trail and rotation | Requires Vault integration, additional latency, complex setup |


---

## Comparison

Based on security, scalability, and future-proofing requirements, the following stack has been chosen for the `auth-service`:

| **Category**        | **Options**                                     | **Chosen**                          | **Rationale**                                                                                                                                                                          |
| ------------------- | ----------------------------------------------- | ----------------------------------- |----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Language**        | Go, Kotlin, Rust                                | **Go**                              | Easy to maintain, great integration with gRPC and Kafka, statically typed, low memory usage, fast startup — ideal for infra-level services.                                            |
| **Token Format**    | JWT (HMAC), JWT (RSA), Opaque token             | **JWT (HMAC)**                      | Simple implementation, strong native support in Go, no need for complex key management. RSA is planned for the Full version as a security upgrade.                                           |
| **Session Store**   | Redis, PostgreSQL, in-memory, file-based        | **Redis**                           | Fast, TTL support, horizontally scalable, simple to integrate. Ideal for short-lived refresh tokens and easy revocation.                                                               |
| **Secrets Storage** | Vault, AWS Secrets Manager, environment variables | **Vault**                           | Vault is included in MVP and planned to be extended in the Full version: secure secret management, audit trail, API-first integration, and safe key rotation with centralized control. |
| **Protocols**       | REST only / gRPC only / REST + gRPC| **gRPC + REST (via API Gateway)**   | Internal services use gRPC for performance and strict typing; external consumers (UI/CLI) interact via REST through the Gateway.                                                       |
| **Messaging**       | Kafka, RabbitMQ, NATS, none                     | **Kafka**                           | Kafka is the platform-wide standard: used for sending login events to ML and Audit services, supports retries, durability, and observability.                                          |
| **Token Signing**   | HMAC, RSA (Vault/KMS), ECDSA                    | **HMAC (via Vault)**                | HMAC is simple and efficient for MVP / Full version. When combined with Vault, secret rotation and storage security are elevated. RSA/ES signing is planned for the Full version.       |
| **Auth Flow**       | Stateless JWT, JWT + session store, Opaque token | **JWT + session store**             | Balanced approach: short-lived stateless access tokens and refresh tokens managed in Redis (TTL, revocation). Flexible, scalable, and secure.                                          |
| **Observability**   | stdout, built-in libraries, Prometheus stack    | **Prometheus + Loki (via library)** | Unified platform observability stack: Prometheus for metrics, Loki for logs, both visualized in Grafana for alerting and debugging.                                                    |

---

## Tradeoffs

Potential compromises made at the MVP stage — and why they are acceptable for now:

| Area                                          | Tradeoff                                                                              | Why it's acceptable at this stage                                                                   |
| --------------------------------------------- |---------------------------------------------------------------------------------------| --------------------------------------------------------------------------------------------------- |
| **HMAC instead of RSA**                       | HMAC is simpler but less flexible for key rotation and external validation            | Vault minimizes secret exposure; RSA can be introduced later without architecture changes           |
| **Redis as session store**                    | Redis doesn't persist data to disk, risk of token loss on restart                     | Used only for short-lived refresh tokens. Tokens are revocable; loss isn't critical                 |
| **gRPC**                                      | gRPC is harder to debug and requires client code generation                           | Strong typing, high performance, and excellent inter-service integration outweigh the cost          |
| **Kafka**                                     | Requires setting up and maintaining a Kafka cluster, might be overkill for simple logs | Kafka is a platform standard; used for auth events via Outbox Pattern and async communication       |
| **Vault**                                     | Requires configuration and IAM integration                                            | Brings immediate security gains, centralized secret management, and future-proof infrastructure     |
| **Go over Kotlin**                            | Lacks OOP classes, may be harder to express complex identity flows                    | For infra-level services, Go is ideal: fast startup, strong gRPC support, memory-safe and efficient |
| **Stateless Access Tokens**                   | Cannot revoke access tokens early (unless using a deny-list)                          | Balanced with Redis-based refresh tokens and short-lived access tokens (5–15 min)                   |
| **Token storage in Redis**                    | Redis downtime may block refresh flow                                                 | Platform-level fallbacks and alerts planned; active access tokens remain valid during outages       |
| **Multiple components (Vault, Kafka, Redis)** | Increases initial deployment complexity in MVP                                        | All infrastructure bootstrapped via `docker-compose` + `Terraform`, deployable in one command       |

---

## Consequences & Next Steps

What is planned for the Full version, why it’s needed, and what is already present in the MVP:

| Area | Planned Addition (Full version) | Purpose | MVP State |
| -------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------- | ------------------------------------------- |
| **MFA & Device Binding**         | Email 2FA, OTP, WebAuthn, trusted devices, fingerprinting                                  | Enhances account takeover protection, aligns with enterprise security standards | Not included                                |
| **Scoped Access Tokens**         | JWTs with time limits, IP restrictions, role- and scope-based access                       | More granular access control, prepares for RBAC/ABAC                            | Simple access/refresh JWTs                  |
| **Access Layer Integration**     | External Policy Engine (gRPC/HTTP) for authorization decisions                             | IAM centralization, better separation of concerns                               | Basic IAM-lite embedded in Auth             |
| **Token Blacklist / Revoke**     | Redis store of revoked token IDs (`jti`) with TTL, checked on each request                 | Enables full logout and compromise recovery                                     | Not included                                |
| **ML Behavior Analysis**         | Integration with ML Detector (geo, frequency, device patterns)                             | Adaptive security and automatic threat detection                                | Not included                                |
| **Key Rotation**                 | Signing key rotation via Vault, JWT `kid` support, audit logs                              | Mitigates key leaks, follows best practices                                     | One static key loaded from Vault            |
| **CLI & UI Integration**         | CLI tool (e.g. `auth-cli login`), UI (React / Figma prototype)                             | Improves DevEx, allows for demos and diagnostics                                | `demo.sh` and Postman collection            |
| **Open Source Packaging**        | Auth as reusable OSS component: ENV config, interfaces, Helm chart, REST/gRPC interfaces   | Showcases maturity, builds trust, encourages reuse                              | Tightly coupled in-app implementation       |
| **Swagger + Postman + Examples** | Full OpenAPI spec, Postman collections, integration guides                                 | Developer onboarding and integration made easier                                | Basic Swagger UI, no detailed documentation |
| **Test Pyramid**                 | Coverage: unit (core logic), integration (Kafka/Vault), e2e (login/register), load testing | Increases confidence, validates production-readiness                            | Unit tests in core logic only               |
| **Helm / Docker Image**          | Ready-to-use Docker images and Helm charts                                                 | Facilitates reuse, deployment, production-readiness                             | Dockerfile and docker-compose only          |

---