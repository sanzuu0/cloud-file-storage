This document outlines the most common and critical engineering problems encountered in file storage systems — particularly those designed for internal platforms, DevOps tooling, or B2B integration.

Each problem is backed by real-world scenarios and architectural consequences, with a structured approach:

- **Context** — where the issue appears and why it matters.
- **Problem** — what goes wrong and what breaks.
- **Solution Objective** — the target outcome and design goal.
- **Implementation** — how the issue is addressed in our platform.

We document these problems not only to improve our own architecture, but also to share reusable patterns and inspire robust, secure, and scalable design practices in the ecosystem.


## 1. Plugging IAM and access roles into file storage requires manual configuration, slowing down integration and increasing the risk of misconfiguration.

### **Context:**
Most teams using Amazon S3, MinIO, or similar object storage solutions must manually implement:

- Access control (IAM, roles, policies).
- User authorization (JWT, OAuth2).
- Restriction configuration (IP-binding, TTL).
- Audit processing (who did what, when, what).

### **Problem:**
Connecting storage to B2B or internal systems often requires deep IAM knowledge and manual setup.
Misconfigurations in access control are a common source of data leaks.
Most teams don’t need full customization — they need safe defaults and a fast path to integration.

### **Solution Objective:**
Reduce the input threshold for storage connectivity and minimize IAM risks without sacrificing flexibility.

### **Implementation:**

- Built-in IAM layer: JWT, refresh tokens, session store, and RBAC.
- Default policies provided at the user, file, and link level (public/private).
- Docker-compose setup and CLI tools for rapid prototyping and deployment.
- Extendable with external providers like Keycloak, Redis, or LDAP — without modifying core logic.


## 2. File access mechanisms often allow reuse of links and do not control downloads, leading to data leaks.

### **Context:**
In many file repositories (S3, MinIO, etc.) file access is implemented via public or semi-protected URLs. Such links often:

- Have no time-to-live (TTL) restrictions.
- Are not tied to a specific user or IP.
- Can be reused multiple times and shared with third parties.

This creates vulnerabilities, especially when dealing with private or sensitive content.

### **Problem:**
In the absence of strict validation, links become a leakage channel:
- A user can download the same file multiple times (or unshare it externally).
- The platform does not limit the number of accesses, does not bind requests to session or IP.
- In case of abuse or automated attacks (bot/script) the system does not react.

### **Solution Objective:**
Provide strictly controlled access links that:
- Can be disposable.
- Have TTLs and restrictions.
- Are logged and validated.
- Preclude the possibility of multiple or anonymous use.

### **Implementation:**

- Support generation of signed links (Signed URLs):
    - TTL (Time to Live Link).
    - Single-use mode.
    - IP restriction (optional).
    - Binding to a session or user token.
- Link signing via HMAC (RSA/ECDSA support is possible in the future).
- Storage and validation of active links.


## 3. Lack of logs and download history makes it impossible to audit and investigate leaks.

### **Context:**
Many file stores either lack a full-fledged access history or have it implemented on the client/application side. In doing so:

- No one tracks who downloaded a particular file.
- IP, user-agent and timestamp are not recorded.
- Action history is lost when scaling or failures occur.

This is critical for teams working with private, financial or legally sensitive files.

---

### **Problem:**
Without centralized logging and auditing:

- Can't investigate the leak (who downloaded the file and when).
- Unable to build a report on access to sensitive information.
- Platform is not compliant with security and auditing requirements (GDPR, SOC2, ISO 27001).

---

### **Solution Objective:**
Provide a centralized, scalable and queryable auditing system for all file activity including downloads, uploads, deletions and errors. Enable both technical and legal analysis.

---

### **Implementation:**

- All actions (upload/download/delete) are logged to ClickHouse or similar OLAP DBMS.
- Key parameters are recorded:
    - User ID, file ID, IP, timestamp, user-agent.
    - Request source (CLI, UI, API).
    - Result (success / error).
- Exposes API endpoints for querying audit logs by user, file, or time range.
- Integration with Grafana and alerting for activity monitoring.

---

## 4. Lack of automatic response to anomalous activity makes the platform vulnerable to abuse and attack.

### **Context:**
Many repositories, even with authorization and signed links, lack a mechanism to actively respond to anomalous activity. Typical scenarios:

- A user or attacker downloads hundreds of files in a short period of time
- Automated scripts or stolen tokens are used.
- Behavior deviates from normal access patterns (time of day, quantity, frequency).

Without automated responses, these behaviors can last for a long time, leading to a complete data breach.

---

### **Problem:**
- The system does not detect or classify anomalies in behavior.
- There is no mechanism to isolate a session or revoke a token when suspected.
- The user or attacker can uncontrollably continue access even after the attack has started.
- The data owner or administrator is not notified.

---

### **Solution Objective:**
Implement a system to detect and respond to suspicious activity with minimum latency and maximum accuracy. Make the abuse visible, interrupt the potential leak and notify the parties involved.

---

### **Implementation:**

- ML Anomaly Module monitors downloads/uploads in real-time using activity patterns (speed, frequency, IP, hours).
- When an abnormality is detected:
  - Temporarily revokes access token or freezes session.
  - The user is notified via email with a verification prompt.
  - Upon confirmation, the session is restored.
- All events are logged and included in the audit.
- Support for webhooks and notifications for external systems (e.g. SIEM or Slack).

---

## 5. Lack of observability makes the system opaque and hinders failure, degradation, and performance analysis.

### **Context:**
In many platforms, logging and metrics are implemented primitively or only at the stdout/file level. There is no centralized system for collecting

- Logs by user actions.
- Technical metrics (latency, errors, throughput).
- Traces between services.
- Business metrics (downloads, anomalies, SLO deviations).

Without this data, it is impossible to understand where and why the failure occurred and how it affects users.

---

### **Problem:**
- The team cannot localize the bug without manually analyzing logs.
- No visibility through distributed tracing - especially important in microservice architecture.
- Metrics are not available in real time → degradation is detected late.
- No systemic link between user actions and internal platform events.

---

### **Solution Objective:**
Implement a complete Observability stack (Logs, Metrics, Traces) for technical and product analysis, SLA/SLO monitoring, error localization and rapid incident response.

---

### **Implementation:**

- Action logging via structured logs (JSON), integration with Loki/Elastic.
- Service level metrics (latency, throughput, error rate) via Prometheus.
- Support OpenTelemetry for tracing between microservices.
- Business metrics (downloads, violations, audit flags) are published to Kafka/ClickHouse.
- Integration with Grafana and alerts on key SLOs.

---

## 6. Lack of fault tolerance testing makes the system unpredictable when infrastructure and services fail.

### **Context:**
Most developers only test systems under stable conditions. Component failure behavior (e.g., Kafka, Redis, MinIO outages or network latency) is not tested as part of the regular development cycle. As a result:

- Failures cause cascading errors.
- SLOs are not sustained under load or degradation.
- Incidents cannot be reproduced in a controlled environment.

---

### **Problem:**
- Unable to predict system behavior when dependencies crash.
- No tooling exists to simulate failures, delays, resource overloads, or unstable network conditions.
- Engineers only learn about problems in production.
- SREs and DevOps cannot verify stability hypotheses.

---

### **Solution Objective:**
Add the ability to regularly and purposefully test system stability against failures. Ensure that components handle failure correctly, do not violate liability boundaries or lead to cascading failure.

---

### **Implementation:**

- Separate CLI tool (`chaosctl`) to run chaos tests.
- Simulation support:
  - Service failures (kill container / stop port).
  - Network call delays.
  - Errors (Kafka not available, Redis timeout).
  - System resource overload (CPU spike, disk full).
- Tests can be run manually or as part of the CI pipeline.
- Results are recorded in audit and metrics.

---

## 7. Lack of file versioning leads to data loss, no rollback capability, and weak change traceability.

### **Context:**
In many storage systems, re-uploading a file results in overwriting an existing file without preserving history. This creates risks:

- The user accidentally loses an important version of a file.
- There is no way to restore the previous state.
- No change history or ability to perform audits.
- It is impossible to confirm who made which changes and when.

Such limitations are especially critical for B2B, legal, financial, and engineering documents.

---

### **Problem:**
- The platform does not distinguish between updating and overwriting a file.
- Unable to roll back to previous version after erroneous upload.
- No change history → no transparency and no protection against internal errors or abuse.
- Unable to audit: who changed what and when.

---

### **Solution Objective:**
Implement a versioning system that:

- Automatically creates a new version upon file updates.
- Allows restoration of any previous version.
- Records change history with metadata and user identity.
- Enables full auditing and legal traceability of changes.

---

### **Implementation:**

- Each `upload` is treated as a new file version (`v1`, `v2`, ...).
- Versioning is scoped by a composite key (e.g., user ID + file name).
- Each version includes metadata:
  - Hash, file size, upload date, uploader ID, optional changelog.
- Any previous version can be restored via the API.
- File content is stored in MinIO as separate objects; version metadata is stored in PostgreSQL.
- API endpoints:
  - `GET /files/{id}/history`
  - `GET /files/{id}?version=n`

---

## 8. Lack of consent and temporary access mechanism limits control over sensitive information and violates privacy requirements.

### **Context:**
In most platforms, if a user is granted access to a file - it is retained until manually revoked. This leads to risks and compliance issues in scenarios such as:

- Temporary access (e.g., to a contractor or client).
- Granting rights only after explicit consent.
- Automatically removing access after task completion or expiration.

Also, many countries and business scenarios require legal recording of the user's consent to access or processing.

---

### **Problem:**
- No ability to define expiration or revocation conditions for granted permissions.
- No mechanism to request and record user consent.
- All access is persistent by default, violating the principle of least privilege.
- No visibility into who granted consent, when, and under what conditions.

---

### **Solution Objective:**
Implement a consent and temporary access system that provides:

- Control the expiration dates of permissions,
- Explicit confirmation of access (e.g. via email or UI),
- Logging of the fact of consent and its terms (time, IP, user),
- Compliance with GDPR, CCPA and internal policies.

---

### **Implementation:**

- `Consent Service` as a standalone module with REST/gRPC API.
- Support for temporary access policies (`expires_at`, `max_downloads`, `revoke after inactivity`).
- API for requesting consent: the user receives a link and must explicitly confirm access.
- All consents are logged and stored in a separate table (`consent_log`).
- In case of no confirmation - access is not activated.

---

## 9. Lack of CLI and automated APIs hinders DevOps integration and reduces the platform's suitability for engineering processes.

### **Context:**
In many systems, storage functions are only accessed through a web interface. This leads to problems:

- It is not possible to integrate storage into CI/CD Pipelines;
- DevOps and SREs cannot manage data via scripts;
- mass operations (batch upload/download, revoke) cannot be implemented;
- engineers cannot automate access checking, auditing, exporting.

This is especially critical for platforms that are positioned as internal services or B2B infrastructure.

---

### **Problem:**
- No stable REST/gRPC API with all the capabilities of the platform.
- No CLI tool for command line.
- All actions are tied to UI → impossible to build into developer tools.
- Platform is not suitable for use in automated systems (CI/CD, scripts, GitOps, Terraform).

---

### **Solution Objective:**
Provide a full engineering interface (API + CLI) that enables complete platform control without relying on the UI. Ensure seamless integration with automation tools, developer workflows, and infrastructure pipelines.

---

### **Implementation:**

- REST and gRPC API with all basic functions:
  - File upload/download.
  - Link generation.
  - Auditing, history, metadata.
  - Management of tokens, roles, users.
- CLI tool `fsctl`:
  - Commands: `fsctl upload`, `fsctl share`, `fsctl audit`, `fsctl revoke`.
  - Support for `.env` and token-based auth.
  - Shell/CI/CD integration.
- Documentation (Swagger/OpenAPI) for all APIs.
- Support for scripting and automation via `bash`, `curl`, `make`.

---

