# AI-Assisted Service Request Processing — Multi-Cloud Integration Pattern

**Role:** Program Solution Architect  
**Year:** 2025  
**Stack:** Salesforce GovCloud | MuleSoft CloudHub 1 GovCloud | GCP (Cloud Run, Pub/Sub, Firestore, Gemini Flash) | Splunk  
**Compliance:** FedRAMP Moderate | GovCloud | NIST 800-53  

---

## Problem Statement

Federal postal operations receive high volumes of service requests requiring document review and classification. Manual review processes created bottlenecks, inconsistent outcomes, and significant processing delays. This architecture automates request triage using GCP Gemini Flash AI — routing high-confidence results directly through an automated approval path while flagging low-confidence results for human review. The integration spans Salesforce GovCloud (request intake and approval workflow), MuleSoft (orchestration and file handling), and GCP (AI processing pipeline) — connected via event-driven patterns across Platform Events and Pub/Sub, with full FedRAMP compliance maintained throughout.

---

## Architecture Overview
![AI Service Request Processing Architecture](diagrams/Mule-Cloud%20AI%20Integration%20Arch.png)

---

## Key Design Decisions

### Why GCP Gemini Flash over alternatives
Gemini Flash provided the latency and cost profile required for high-volume document classification at scale. GCP's FedRAMP Moderate authorization covered the required data classification level, and existing GCP infrastructure (Cloud Run, Pub/Sub, Firestore) allowed the AI pipeline to be built within an already-authorized boundary — avoiding a new ATO process for a net-new cloud provider.

### Why Pub/Sub over Kafka for the GCP pipeline
Pub/Sub is native to GCP and already within the FedRAMP authorization boundary. For this pattern — one producer (MuleSoft), one primary consumer (AI pipeline) — Pub/Sub's managed infrastructure eliminated the operational overhead of running and maintaining a Kafka cluster inside the GCP VPC. Kafka was evaluated and rejected as over-engineered for this specific message volume and consumer topology.

### Why MuleSoft as the integration layer
MuleSoft CloudHub 1 GovCloud was the existing authorized integration platform. It provided the OAuth2.0 / JWT security controls, static IP (required for SF GovCloud allowlisting), and the operational pattern (Object Store, error handling, DLQ) needed to manage the async AI pipeline reliably. MuleSoft acts as the control plane between SF and GCP — handling file dispatch, AI response ingestion, and SF data writeback.

### Why event-driven over request-response
Service requests are created asynchronously by users and require variable AI processing time depending on file size and complexity. A synchronous request-response pattern would have required the user to wait on an indeterminate processing window. Platform Events decouple request creation from processing — the user submits and continues working; the system processes and routes in the background.

### Confidence-based routing
Rather than treating all AI results equally, the architecture routes based on Gemini's confidence score output. High-confidence results enter an automated approval queue. Low-confidence results are flagged for human review using the existing Salesforce approval process. This preserves human oversight where AI certainty is insufficient while eliminating manual touchpoints where it isn't.

---

## Architecture Patterns

| Pattern | Implementation |
|---------|----------------|
| Event-driven intake | SF Platform Event fires on service request creation → MuleSoft subscribes |
| File sharding | Large files sharded in SF, checked out and processed in parallel, rolled up on completion |
| Async AI processing | MuleSoft dispatches to GCP via async TLS/HTTPS + Workload Identity Federation |
| Pub/Sub orchestration | Cloud Run Orchestration Services coordinates AI pipeline via GCP Pub/Sub |
| Confidence-based routing | Gemini Flash returns confidence score → high/low path routing in SF approval process |
| DLQ / Error promotion | Failed processing routes to Promotion DLQ Process with Platform Event ledger logging |
| Writeback pattern | AI results written back to SF staging object via MuleSoft on completion |
| Object Store purge | MuleSoft Object Store purged on successful processing — no residual payload retention |

---

## Security & Compliance

| Control | Implementation |
|---------|----------------|
| Authentication | OAuth2.0 (SF ↔ MuleSoft), OAuth2.0/JWT (MuleSoft ↔ GCP), Workload Identity Federation (GCP internal) |
| Network | GCP VPC + SubNetwork isolation, Static IP on MuleSoft for SF allowlisting |
| Encryption | Default encryption at rest on Cloud Storage and Firestore |
| Transport | TLS/HTTPS enforced on all integration boundaries |
| Data handling | No payload or PII logging in Splunk or Anypoint Monitoring |
| Authorization boundary | FedRAMP Moderate — GovCloud (SF + MuleSoft), GCP FedRAMP authorized services only |

---

## Observability

- **Splunk** — Log aggregation across MuleSoft flows. Structured logging with no payload/PII.
- **Anypoint Monitoring** — MuleSoft flow performance, error rates, API metrics
- **Mule Error Alerts** — Threshold-based alerting on flow failures and DLQ promotion events
- **GCP Cloud Run** — Native GCP monitoring on orchestration and AI services

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| CRM & Workflow | Salesforce GovCloud Plus |
| Integration | MuleSoft CloudHub 1 GovCloud |
| File Storage | GCP Cloud Storage (encryption at rest) |
| Orchestration | GCP Cloud Run |
| Messaging | GCP Pub/Sub |
| AI/ML | GCP Gemini Flash via AI Pipelines |
| Document Store | GCP Firestore (encryption at rest) |
| Observability | Splunk, Anypoint Monitoring |
| Identity | OAuth2.0, JWT, Workload Identity Federation |

---

## What This Demonstrates

- **Multi-cloud architecture** — SF + MuleSoft + GCP designed as a cohesive system, not bolted together
- **FedRAMP-constrained design** — every technology decision made within an active authorization boundary
- **Event-driven patterns** — Platform Events, Pub/Sub, async processing across cloud boundaries
- **AI/ML integration** — practical production integration of LLM confidence scoring into an enterprise approval workflow
- **Failure design** — DLQ, error promotion, Object Store purge on success, Platform Event ledger
- **Security-first** — WIF, OAuth2.0/JWT, no PII logging, encrypted at rest and in transit throughout

---

*All diagrams redacted for public portfolio. Internal system identifiers, org IDs, and endpoint URLs removed. Architecture patterns and decisions are original work.*
