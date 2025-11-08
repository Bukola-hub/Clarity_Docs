# Clarity — Detailed System Architecture

## 1. Architecture Overview (Layers & Intent)

**Goal:**  
A scalable, reliable, secure SaaS product-management platform that’s easy to use and integrates with developer tools (Jira/GitHub), with web + mobile clients.

### Layers

- **Client Layer** — Web (SPA) + Mobile (iOS/Android)  
- **Edge & CDN** — TLS termination, WAF, CDN for static assets  
- **API Gateway & Auth** — Unified entry for microservices, rate limiting, routing, JWT/OAuth validation  
- **Application Layer** — Modular services: User, Project, Strategy, Roadmap, Ideas, Backlog, Integrations, Notifications, Analytics  
- **Data Layer** — Relational DB (PostgreSQL), Search (Elasticsearch), Cache (Redis), Object Storage (S3)  
- **Integration & Messaging** — Background jobs, message broker (RabbitMQ / Kafka), Webhooks  
- **Infrastructure & Ops** — CI/CD, monitoring, logging, backup, secrets manager  

### Component Responsibilities (Concise)

#### Web SPA / Mobile App
- React (Next.js) SPA for desktop; PWA or React Native for mobile  
- Handles UI/UX, local caching, optimistic updates, limited offline read  

#### API Gateway
- Authentication (OAuth2 / OpenID Connect for SSO)  
- Rate limiting, request routing, and CORS handling  

#### User & Auth Service
- Signup, login, SSO (SAML/OIDC), RBAC, audit logs, MFA  

#### Project Service
- Projects, teams, memberships, roles, billing/plan enforcement (multi-tenant partitioning)  

#### Strategy Hub
- OKRs, goals, personas; alignment logic and linking to work items  

#### Roadmap Service
- Timeline, Gantt, dependencies, timeline export (PDF/Share link)  

#### Idea Portal
- Public submission form, voting, value-effort scoring, moderation workflows  

#### Backlog & Releases
- Epic → Feature → Task hierarchy, release planning board  

#### Integrations
- Jira / GitHub connectors (OAuth apps, webhooks)  
- Two-way sync logic, conflict resolution  

#### Notifications
- In-app, email, and Slack notifications  
- Scheduled digests  

#### Dashboards / Analytics
- Aggregations, charts, risk alerts, ad-hoc queries (Elasticsearch / OLAP)  

#### Background Worker / Message Broker
- Asynchronous jobs: integration sync, exports, notifications, AI scoring, analytics pipelines  

---

## 2. Data Design & Examples

**Primary Datastore:** PostgreSQL  

### Core Tables (Examples)

| Table | Description |
|-------|--------------|
| `users` | (id, name, email, hashed_password, profile, created_at) |
| `organizations` | (id, name, billing_plan, created_at) |
| `projects` | (id, org_id, name, settings) |
| `okrs` | (id, project_id, title, target_value, current_value, owner_id) |
| `roadmaps` | (id, project_id, name, start_date, end_date) |
| `work_items` | (id, type {epic,feature,task}, title, status, assignee_id, parent_id, estimates) |
| `ideas` | (id, project_id, title, description, votes, value, effort, status) |

### Search / Analytics: Elasticsearch
- Index roadmaps, work_items, and ideas for fast filtering, aggregations, and dashboards  

### Cache
- **Redis:** session store, rate limiting, and short-lived caches for boards and heavy queries  

### Object Storage
- **S3:** attachments, export files (PDF/CSV), images  

---

## 3. Integrations & Sync Strategy (Jira / GitHub)

**Pattern:** Hybrid of webhooks + background reconciliation  

- **Outbound:**  
  When a work item is created or updated in Clarity → enqueue change → Integrations service pushes to Jira/GitHub via REST API  
- **Inbound:**  
  Jira/GitHub webhooks → Integrations service receives event → normalizes data → enqueues reconciliation job → updates Clarity state  
- **Conflict Handling:**  
  Timestamp-based last-writer-wins + status mapping + audit log; present merge conflicts in UI  
- **Initial Import:**  
  Bulk import job with progress tracking  

---

## 4. Security & Compliance

- **Transport:** TLS everywhere  
- **Auth:** OAuth2 / OpenID Connect; support for SAML for enterprise  
- **Access Control:** RBAC, org-scoped multi-tenancy, least privilege  
- **Encryption:** AES-256 at rest for sensitive fields; TLS for data-in-transit  
- **Secrets:** Secrets manager (AWS Secrets Manager / HashiCorp Vault)  
- **Audit & Logging:** Immutable audit trails for OKRs, work items, integrations  
- **GDPR / Data Locality:** Data residency options by region; retention policies; export & delete support  
- **Pen Tests & Reviews:** Mandatory before enterprise release  

---

## 5. Non-Functional Requirements & Scalability

- **Availability:** 99.9% SLA target for core features  
- **Scaling Model:** Stateless services + horizontal autoscaling behind load balancers; DB read replicas; partitioning (row-level or schema-per-tenant)  
- **Performance:** Efficient queries, pagination, background processing; cache boards & timeline tiles  
- **Resilience:** Circuit breakers for external APIs; retry/backoff for integrations  

---

## 6. Observability & Operations

- **Metrics:** Prometheus + Grafana (latency, error rates, queue depth)  
- **Tracing:** OpenTelemetry for distributed tracing  
- **Error Tracking:** Sentry for exceptions  
- **Logs:** Centralized (ELK or hosted equivalent) with structured logs and retention policies  
- **Backups & DR:** Daily DB backups, point-in-time recovery, periodic restore tests  

---

## 7. CI/CD & Release Model

**GitHub Actions / GitLab CI Pipelines:**  
`Lint → Unit Tests → Integration Tests → Build → Canary Deploy → Smoke Tests → Promote to Prod`

- Feature flags for progressive rollout  
- Blue/green or canary deployments for minimal downtime  

---

## 8. Mobile & Offline Considerations

- **PWA / Mobile:** Read-only or light-edit workflows for roadmaps and approvals; local caching for recent boards  
- **Offline Mode:** Allow small offline actions queued locally and synced on reconnection (with conflict resolution UI)  

---

## 9. AI & Automation (V1 → V2 Possibilities)

- **AI Prioritization:** Background scoring service that computes priority using value, effort, and engagement signals  
- **AI Wizard:** Onboarding assistant for creating initial OKRs and templates  
- **Edge Caution:** Keep AI explainable and auditable; store inputs for reproducibility  

---

## 10. Delivery Checklist (Prioritized)

1. Design DB schema + APIs for MVP modules (Strategy, Roadmap, Jira sync)  
2. Implement Auth & multi-tenant Project scaffolding  
3. Build frontend Roadmap & Strategy UI with drag-and-drop + templates  
4. Integrations: basic Jira push (1-way), then 2-way reconciliation  
5. Add background jobs, caching, and basic dashboards  
6. Observability, logging, and secure deployment pipeline  
7. Expand features: Idea Portal, AI prioritization, SSO, enterprise controls  
