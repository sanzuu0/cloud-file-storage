# Roadmap – File Storage Platform

This document outlines the evolution of the platform — from a minimal viable core (MVP) to a fully featured, production-grade system with scalable IAM, ML-based analytics, file collaboration, and reusable open-source components.

### Purpose of this roadmap:
- Define the long-term growth strategy of the platform
- Communicate the architecture and product maturity levels
- Break down features by release phase: MVP, Full, OSS, and Optional
- Serve as a living reference for planning, prioritization, and technical alignment

This roadmap is continuously updated as the system evolves and acts as a shared source of truth for contributors, reviewers, and stakeholders.

---

## 1. Overview

| Version      | Status       | Purpose and Composition                                                                 |
|--------------|--------------|------------------------------------------------------------------------------------------|
| **MVP**      | ✅ Planned    | Establish a secure, observable file platform with authentication, IAM-lite, audit logs, metrics, and CI/CD. |
| **Full**     | 🟡 Later      | Extend the core with full RBAC/ABAC, consent system, WebSocket collaboration, ML anomaly detection, versioning, search, quotas, and billing. |
| **OSS Layer**| 🟢 Parallel   | Extract reusable components into Open Source: Access Policy, Consent, Audit Logger, ML Detector, CLI Tool. |
| **Showcase** | 🔲 Optional   | Demonstrate maturity through UI (Next.js), live dashboards, postmortems, LLM integration, and auto-remediation. |

---

## 2. MVP — Minimal Viable Platform

Build a secure, modular, observable file storage platform based on microservices, event-driven design, and production-grade DevOps practices — with early support for developer-facing tools and demonstration capabilities.

### MVP Focus:
- Security (Authentication, encryption, token management, integrity checks)
- IAM-lite (public/private), public links
- Event-driven audit logging (Kafka + services)
- Metrics and logging (Prometheus, Grafana, Loki)
- Infrastructure-as-Code and CI pipelines (Docker, Terraform)
- Early DX/demo tooling (Swagger, demo.sh, CLI)

---

### Security
- JWT + Refresh tokens
- mTLS between services (gRPC)
- Vault or Secret Manager for keys and tokens
- HMAC file signatures for integrity checking
- Enforce request-level authorization at the gateway layer (JWT claims)

### Users
- Profile CRUD (name, email)
- Binding entities to userID (access control)

### Files
- Upload and download files using File Service and MinIO-compatible API
- Store and query metadata: filename, size, owner, timestamps
- Generate public links with TTL (time-to-live)
- Apply basic access controls (IAM-lite: public/private)


### Auditing and events
- Kafka events (file.uploaded, login, etc.)
- Audit Log (access and action events)
- Webhook architecture (via outbox pattern)

### Observability
- Prometheus (metrics by service)
- Loki + Grafana (centralized logs)
- Health checks (liveness / readiness)
- Postmortem template (docs/postmortem.md)

### API Gateway & Routing
- API Gateway (rate limiting + JWT passthrough)
- AuthZ within gateway by userID/token claims
- gRPC between microservices

### DevOps & Infrastructure
- Docker + docker-compose
- Terraform (resource provisioning: MinIO, Kafka, Vault)
- GitHub Actions (CI: build, lint, test, security)
- demo.sh + Postman collection (ready-made scripts)

### Developer Experience / Showcase
- Swagger UI for API documentation and testing
- Minimalistic CLI (upload, download)
- Simple Mock UI (HTML or React)
- User Activity Log (activity history interface)

---

## 3. Full Version — Feature-Complete Platform

Turn the MVP into a mature, secure, and flexible platform suitable for production use at scale — with advanced IAM, real-time collaboration, machine learning, full audit streaming, product-level features, and reusable open-source components.

---

### Security & Access Management
- Enforce full RBAC/ABAC (roles, groups, attributes)
- Enable scoped sharing (restricted per user/group/policy)
- Implement TTL and revoke access for links and sessions
- Restrict access via IP, geo, and device fingerprinting
- Support trusted devices and device binding

---

### Consent & Privacy
- Record user consent with full audit trail
- Support temporary access with auto-expiration
- Ensure GDPR/CCPA compliance (erasure, transparency)

---

### ML and Behavioral Analysis
- Detect behavioral anomalies using ML
- Learn user activity patterns over time
- Trigger alerts based on risk-based access scoring

---

### Audit and Analytics
- Track a complete audit trail of actions (via ClickHouse)
- Analyze session and geographic access patterns
- Stream audit logs externally via webhook endpoints

---

### Advanced File Features & Structure
- Support file versioning and rollback
- Compare diffs between file versions
- Optimize storage via deduplication and metadata indexing
- Visualize tags, file types, formats, and activity history

---

### Real-Time Collaboration
- Enable live editing via WebSockets
- Allow in-file commenting and threaded discussions
- Apply role-based permissions: view / edit / comment

---

### User & Team Management
- Manage user groups and team structures
- Support invitations with role assignments (admin, editor, guest)
- Tag and segment users with custom fields and metadata

---

### Observability & Incident Response
- Collect advanced metrics per service and endpoint
- Trace logs with request ID and context-awareness
- Trigger alerts via Alertmanager and expose health status
- Generate incident reports with postmortem templates and bots

---

### DevOps, CI/CD, and Reliability
- Run CI pipelines with GitHub Actions (test, lint, security scan)
- Deploy production via Docker and Helm charts
- Follow a test pyramid: unit, integration, and e2e layers
- Apply chaos engineering to validate system resilience

## 4. OSS Components — Reusable Open-Source Services

Expose mature microservices as reusable OSS components to maximize reuse, demonstrate architectural maturity, and increase engineering impact across teams and ecosystems.

- Maximize reusability and external adoption
- Showcase clean boundaries and service-level thinking
- Promote modular platform design over monolithic solutions
- Simplify integration into other systems and products

---

### Access, Identity & Consent

| Component             | What it does                                    | Why it's needed                              |
|-----------------------|--------------------------------------------------|----------------------------------------------|
| `access-policy-service` | Evaluates access policies (RBAC, ABAC, conditions) | Enables centralized IAM for any system       |
| `session-store`       | Manages refresh tokens, TTL, revocation          | Provides scalable and secure authentication  |
| `consent-service`     | Stores user consents with TTL and auto-expiry    | Ensures GDPR/CCPA compliance and fine-grained consent tracking |

---

### Audit, Webhooks & Controls

| Component             | What it does                                    | Why it's needed                              |
|-----------------------|--------------------------------------------------|----------------------------------------------|
| `audit-service`       | Ingests Kafka events and stores them in ClickHouse | Enables full user and system activity tracking |
| `webhook-broker`      | Routes incoming/outgoing webhooks                | Supports CI/CD hooks, alerting, integrations |
| `file-verifier`       | Verifies HMAC signatures of files                | Ensures file integrity and tamper detection  |

---

### ML & Behavior Analysis

| Component             | What it does                                    | Why it's needed                              |
|-----------------------|--------------------------------------------------|----------------------------------------------|
| `ml-detector`         | Detects anomalies and suspicious behaviors       | Powers adaptive security and risk detection  |
| `anomaly-lib`         | Scores activity and learns normal behavior       | Enables real-time and offline threat analysis |

---

### Infrastructure & Engineering Culture

| Component             | What it does                                    | Why it's needed                              |
|-----------------------|--------------------------------------------------|----------------------------------------------|
| `postmortem-generator`| Generates RCA reports from incident templates    | Promotes engineering culture and root cause transparency |
| `metrics-stack`       | Aggregates metrics and logs for observability    | Accelerates integration of full observability pipelines |

---

### Architectural Principles

- **Strict isolation**: each component is stateless and decoupled from business logic
- **Documented APIs**: gRPC or HTTP interfaces with clear contracts
- **Event-ready**: all services integrate cleanly into Kafka/event architectures
- **Portable config**: ENV-based configuration for cloud/on-prem
- **Self-documented**: comes with Swagger, Postman, and deployment examples

