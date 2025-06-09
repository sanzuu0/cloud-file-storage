# MVP Definition – Minimal Viable Platform

This document defines the architecture of the MVP version of the file storage platform and serves as the entry point to service-level documentation.

> ℹ️ **Note:**
> Detailed descriptions for each service are being added incrementally in the [`docs/services/`](./services/) directory.
> If a link is inactive, the documentation is still in progress.
> The final version will cover all 16 core MVP components.

---

## Goal

Build a minimal yet secure, observable, and scalable file storage platform with a microservice architecture.
All components must be designed to support future extensions, such as RBAC, ML analytics, consent management, and a full-featured UI.

---

## MVP Structure

The platform is divided into two phases:

* **Phase 1: Core Foundation** — critical infrastructure required for authentication, access control, auditing, and observability.
* **Phase 2: MVP Enhancers** — improvements focused on security, external integration, and developer experience.

---

## Phase 1: Core Foundation

| #  | Component               | Purpose                                  | Why It's Included                           |
| -- | ----------------------- | ---------------------------------------- | ------------------------------------------- |
| 1  | **Auth Service**        | JWT + refresh tokens, session handling   | Secure user authentication and isolation    |
| 2  | **User Service**        | Profile CRUD (name, email)               | Basic user identity, used by IAM & Audit    |
| 3  | **API Gateway**         | Entry point, JWT validation, rate-limit  | Unified access control and routing          |
| 4  | **File Service**        | Upload/download, MinIO integration       | Core business logic: file I/O               |
| 5  | **Metadata Service**    | File metadata (size, type, author, etc.) | Fast lookup and IAM policy enforcement      |
| 6  | **Access Policy**       | IAM-lite: user-scoped file access        | File-level isolation and authorization      |
| 7  | **Kafka (Event Bus)**   | Service-to-service event transmission    | Decoupling, scaling, audit foundation       |
| 8  | **Audit Service**       | ClickHouse logging, event ingestion      | Traceability, compliance, analytics         |
| 9  | **Observability Stack** | Logs, metrics, traces                    | Production readiness and debugging          |
| 10 | **Postmortem Template** | Incident response report format          | Engineering culture and root-cause analysis |

---

## Phase 2: MVP Enhancers

| #  | Component                           | Purpose                                  | Why It's Included                        |
| -- | ----------------------------------- | ---------------------------------------- | ---------------------------------------- |
| 11 | **Session Store** (Redis)           | TTLs, logout, token revocation           | Session management and lifecycle control |
| 12 | **mTLS**                            | Secure gRPC connections                  | Internal network-level security          |
| 13 | **Webhook Adapter**                 | External event delivery (Outbox pattern) | Integrations: CI, ML, external hooks     |
| 14 | **Rate Limiting**                   | Protect API Gateway from overload        | Defense against abuse and DDoS attacks   |
| 15 | **HMAC Verifier**                   | File signature and integrity checks      | Ensures uploaded content is untampered   |
| 16 | **Dev Tooling** (Makefile, Scripts) | CI/CD, migrations, builds                | Streamlined developer experience         |

---

## Platform Architecture Notes

* **Core Foundation** must be fully implemented and tested before MVP release.
* **Enhancers** may be implemented progressively, but their architectural hooks must be present from the start.
* All services must expose:

  * gRPC APIs (internally)
  * REST endpoints via API Gateway (OpenAPI)
  * Prometheus-compatible metrics
  * Structured stdout logging
  * Unit tests and CI pipelines

---

## Service Documentation Index

Refer to [`docs/services/`](./services/) for service-level architecture and implementation details:

| #  | Service               | Documentation                                     |
| -- | --------------------- | ------------------------------------------------- |
| 1  | Auth Service          | [`auth.md`](./services/auth.md)                   |
| 2  | User Service          | [`user.md`](./services/user.md)                   |
| 3  | API Gateway           | [`gateway.md`](./services/gateway.md)             |
| 4  | File Service          | [`file.md`](./services/file.md)                   |
| 5  | Metadata Service      | [`metadata.md`](./services/metadata.md)           |
| 6  | Access Policy Service | [`policy.md`](./services/policy.md)               |
| 7  | Kafka Bus             | [`kafka.md`](./services/kafka.md)                 |
| 8  | Audit Service         | [`audit.md`](./services/audit.md)                 |
| 9  | Observability Stack   | [`observability.md`](./services/observability.md) |
| 10 | Postmortem Template   | [`postmortem.md`](./services/postmortem.md)       |
| 11 | Session Store (Redis) | [`session.md`](./services/session.md)             |
| 12 | mTLS Setup            | [`mtls.md`](./services/mtls.md)                   |
| 13 | Webhook Adapter       | [`webhook.md`](./services/webhook.md)             |
| 14 | Rate Limit (Gateway)  | [`rate-limit.md`](./services/rate-limit.md)       |
| 15 | HMAC Verifier         | [`hmac.md`](./services/hmac.md)                   |
| 16 | Dev Tooling           | [`devtools.md`](./services/devtools.md)           |

---