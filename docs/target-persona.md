# Target Personas

This document describes the key categories of users of our file storage platform.  
The goal is to identify our key users, understand their pain points and goals, and align product and architectural decisions accordingly.

### Each persona contains:

- The role and context of use of the platform
- Key pains and constraints
- The goals and challenges they want to solve
- How our product helps them achieve those goals

### This allows us to:

- Justify product and technical decisions
- Shape MVP chips for real-world scenarios
- Speak the language of users in documentation, CLI, GitHub READMEs
- Build understandable demo scenarios
- Use the project as an argument in open-source

---

## Overview of all personas

| Persona | Role | Core Pain | Goal | Relevant Features |
|--------|------|------------------------------|-----------------------------|----------------------------|
| Small SaaS CTO | Product and tech lead | Difficult and risky IAM integration | Deploy secure file storage in 15 minutes without complexity | Built-in RBAC, JWT, audit logs, CLI |
| DevOps Engineer | Automation / CI/CD | No CLI or API for automation | Integrate file storage into pipelines and scripts | fsctl, REST/gRPC, audit API |
| Security Engineer | Access control and threat monitoring | No revoke, no TTL, no alerts | Monitor usage and respond to threats in real time | TTL links, revoke, ML-based anomaly detection |
| Privacy Officer (GDPR) | Legal / compliance | No consent tracking or access logs | Ensure traceable, compliant access history | Consent API, full audit logs, GDPR compliance |
| SRE / Observability Engineer | Monitoring / SLAs | No metrics, tracing, or visibility | Detect issues and understand impact quickly | Prometheus, OpenTelemetry, Grafana |
| Enterprise Integrator | B2B / corporate IT | Needs LDAP, flexible access policies | Plug file storage into enterprise auth systems | Keycloak, LDAP, policy engine |
| Internal Platform Developer | Backend / platform engineering | Needs fast secure integration for internal tools | Quickly embed file storage with built-in policies | API, CLI, preconfigured IAM and defaults |

---


