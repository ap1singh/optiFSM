# Deployment Architecture
## Field Service Management System · Utility Field Response Optimization
**Document:** Deployment Architecture
**Version:** 1.0 · April 2026

---

## 1. Purpose and Scope

This document defines how the FSM system is physically and logically deployed, including infrastructure topology, database design choices, security zones, disaster recovery strategy, environment separation, capacity planning, monitoring and alerting, and the phased go-live plan.

The architecture is designed for a 24×7 utility operation where P1 emergency dispatch must remain functional during planned maintenance, partial failures, and a regional data centre outage. Every design choice is traceable to one or more of these operational requirements.

**Primary design goals:**
- Zero unplanned downtime during P1 emergency response
- 99.5% availability for dispatcher board during operational hours
- 99.9% availability for the P1 emergency dispatch path
- Graceful degradation when external systems fail
- Recovery from regional disaster within 4 hours (RTO) with at most 15 minutes of data loss (RPO)
- Horizontal scalability to support growth from 500 to 5,000 concurrent field workers

---

## 2. Overall Architecture

The FSM system is structured into three security zones plus a disaster recovery site.

### 2.1 Zone Definitions

**DMZ (External-facing zone).** Hosts everything that receives connections from outside the utility network — load balancers, web application firewalls, API gateways for external system integrations, and the mobile backend-for-frontend (BFF). All external traffic terminates here; nothing from the DMZ reaches the data zone directly.

**Application zone.** Hosts all application logic — the web applications for dispatcher board and management dashboard, the worker app API backend, the four engines as separately deployable services, the notification publisher, SLA sweeper, and integration adapters. No direct external access — only the DMZ can reach it.

**Data zone.** Hosts all stateful services — primary PostgreSQL database, read replicas, Redis cluster, Kafka event bus, and object storage for photos and document attachments. Accessible only from the application zone. No direct external access under any circumstances.

**DR site.** A geographically separate data centre (different region, not just different availability zone) holding asynchronous replicas of the data zone. In steady state, the DR site is a warm standby — infrastructure running but not receiving traffic. During a primary region disaster, DNS failover switches traffic to the DR site after the recovery procedure completes.

### 2.2 Network Segmentation

```
Internet / Utility WAN
         |
    [Firewall]
         |
    DMZ Zone
      - No direct DB access
      - Public TLS endpoints
      - WAF inspects all traffic
         |
    [Firewall — only specific ports]
         |
    Application Zone
      - Stateless services
      - No external DNS resolution for non-whitelisted domains
         |
    [Firewall — database protocols only]
         |
    Data Zone
      - No internet access
      - Encrypted at rest
      - Backup traffic to DR site via dedicated link
```

Firewall rules are defined as infrastructure-as-code and reviewed quarterly. No ad-hoc firewall changes are permitted in production — all changes go through the change management process.

---

## 3. Infrastructure Components — Detailed

### 3.1 DMZ Zone Components

**Web Application Firewall (WAF).**
Deployment: ModSecurity with OWASP Core Rule Set, or a managed WAF service.
Purpose: Blocks common web attacks (SQL injection, XSS, DDoS) before traffic reaches application servers. Rate-limits per source IP.
Sizing: Active-active pair, each sized for 3× peak expected traffic.
Maintenance: Rule updates applied within 48 hours of OWASP advisory.

**Load Balancer.**
Deployment: HAProxy or NGINX in active-active configuration with VIP failover.
Purpose: Terminates TLS, distributes traffic across application zone instances, performs health checks.
Sizing: Each instance sized for 5,000 concurrent connections.
Health check: HTTP GET on `/healthz` endpoint — returns 200 only if service is truly operational (not just process running).

**API Gateway.**
Deployment: Kong or similar, HA cluster.
Purpose: Manages authentication, rate limiting, request routing, and API versioning for all external system integrations. Every API call from SCADA, MDMS, CRM, Billing, and the Regulatory Portal passes through this gateway.
Feature set: OAuth 2.0 introspection, mutual TLS termination, per-consumer rate limiting, circuit breaker integration, structured access logging.

**Mobile Backend-for-Frontend (Mobile BFF).**
Deployment: Node.js services behind the load balancer.
Purpose: Aggregates and optimises data for the field worker mobile app. Handles offline sync logic — field devices push a batch of offline actions on reconnection, and the BFF serialises them into the correct order for the backend.
Key responsibilities: Conflict detection (same task completed twice from two devices), priority sync ordering (safety-critical records first), compression for cellular networks.

### 3.2 Application Zone Components

**Web Applications (Dispatcher Board + Management Dashboard).**
Deployment: React SPA + Node.js API server containers.
Purpose: Serves the two web interfaces to internal users.
Sizing: 4 containers minimum in steady state, auto-scaling up to 16 during peak.
State: Stateless — session data in Redis, never in-memory.

**Worker App API Backend.**
Deployment: Node.js or Go services.
Purpose: Handles all mobile app requests — task lists, task detail, check-in, isolation workflow, status updates, test record capture.
Sizing: Scales based on concurrent worker count. 1 container per 100 concurrent workers as a baseline.
Critical path: All safety-critical writes (pre_work_confirmed, post_work_confirmed) go through dedicated priority queue with elevated SLA.

**Engine Workers.**
Each of the four engines runs as a separate deployable service with its own scaling characteristics:

| Engine | Deployment pattern | Scaling |
|---|---|---|
| Schedule Engine | Cron job (nightly) + lightweight API for ad-hoc triggers | Single instance with leader election |
| Matching Engine | Reactive service responding to task events | Horizontal — scales with task creation rate |
| Clustering Engine | Cron job (nightly) + API triggered on task creation | 2-3 instances for redundancy |
| Supply Optimization Engine | Synchronous API for single outage, async job for multi-outage | CPU-intensive — 2-4 dedicated containers |

**Integration Adapters.**
Deployment: One container per external system.
Purpose: Encapsulates protocol translation, retry logic, circuit breaker state, and schema validation for each integration. The adapter isolates the main application from the specific quirks of each external system — a change in SCADA's API does not require changes to the Supply Optimization Engine; only the SCADA adapter.

**SLA Sweeper.**
Deployment: Scheduled worker (every 6 hours).
Purpose: Scans SLA_TRACKER records, fires 30-day, 7-day, and 1-day alerts, marks breaches.
Sizing: Single instance — operation is idempotent and runs quickly.

**Notification Publisher.**
Deployment: Long-running worker consuming internal events and publishing to CRM Kafka topic.
Purpose: Bridges internal state changes to the external notification flow.
Critical: This service must not lose messages — internal events that trigger regulatory notifications (planned outage advance notice) have permanent compliance consequences if lost.

### 3.3 Container Orchestration

**Platform:** Kubernetes (self-hosted on utility infrastructure or managed — depending on utility IT policy).

**Cluster topology:**
- 3-node control plane (etcd, kube-apiserver, scheduler, controller-manager) for HA
- Minimum 6 worker nodes for workload distribution
- Separate node pools for:
  - Web/API services (small, many instances)
  - Engine workers (CPU-optimised)
  - Stateful services if not using managed services (but generally data zone components run as managed services or on dedicated VMs, not in Kubernetes)

**Deployment strategy:** Blue-green deployments for zero-downtime updates. New version deployed alongside old, traffic cut over after health validation, old version kept running for 10 minutes as rollback insurance.

**Resource governance:** Every workload has resource requests and limits defined. Namespace-level quotas prevent any single workload from starving others.

### 3.4 Data Zone Components

**Primary Database — PostgreSQL 15+ with PostGIS extension.**

Rationale for PostgreSQL:
- Mature, proven in utility and financial sectors
- PostGIS extension provides native spatial queries for proximity calculations in the matching engine
- JSONB column type supports the flexible TEST_RECORD.test_readings and DOCUMENT_TEMPLATE.fields_schema patterns without introducing a separate document store
- Strong consistency guarantees for safety-critical writes (isolation gates, energization status)
- Mature replication tooling

Deployment pattern:
- Primary + 2 synchronous replicas in primary data centre (one synchronous, one asynchronous for read scaling)
- 1 asynchronous replica in DR site
- Physical replication via streaming WAL
- Read replicas handle dispatcher board queries, management dashboard queries, and reporting
- Write traffic goes only to primary

Sizing at Phase 1 (500 workers, 100,000 consumers):
- Primary: 16 cores, 64 GB RAM, 2 TB SSD storage, 5,000 IOPS
- Each replica: equivalent to primary

Sizing at Phase 3 (5,000 workers, 1,000,000 consumers):
- Primary: 32 cores, 256 GB RAM, 8 TB SSD storage, 20,000 IOPS
- Each replica: equivalent

Partitioning strategy for high-volume tables:
- SOURCE_OPERATING_STATE partitioned by week (readings expire after 90 days of retention)
- SERVICE_HISTORY partitioned by year (permanent retention but aged partitions moved to cheaper storage)
- CRM_NOTIFICATION partitioned by month (regulatory records preserved 7 years, operational records 18 months)
- METER_ALERT partitioned by week (alerts archived after resolution + 30 days)

Connection pooling: PgBouncer in transaction-pooling mode between application zone and database, reducing connection overhead. Application services connect to PgBouncer, not directly to PostgreSQL.

**Cache — Redis Cluster.**

Purpose:
- Session storage for dispatcher and management web sessions
- Worker location cache (updated from mobile app, read by matching engine — millisecond reads critical for P1 dispatch speed)
- Rate limiter counters (API gateway enforcement)
- Distributed locks for engine coordination
- Pre-computed N-1 supply plan cache (stored for 24 hours after nightly contingency run)

Deployment: 6-node Redis cluster (3 masters, 3 replicas) with automatic failover via Redis Sentinel.

**Event Bus — Apache Kafka.**

Purpose:
- Internal event distribution (task status changes trigger engine reactions)
- CRM notification publishing
- Audit log streaming for compliance records

Deployment: 3-broker cluster with Zookeeper ensemble (or KRaft mode in newer Kafka versions).

Topic configuration:
| Topic | Partitions | Replication factor | Retention |
|---|---|---|---|
| fsm.tasks.events | 12 | 3 | 7 days |
| fsm.notifications.consumer | 6 | 3 | 30 days |
| fsm.isolation.events | 6 | 3 | 90 days (safety records) |
| fsm.audit.log | 12 | 3 | 2 years |
| crm.delivery.confirmations | 6 | 3 | 30 days |
| fsm.notifications.consumer.dlq | 3 | 3 | 90 days |

**Object Storage.**

Purpose: Photos attached to service records, generated PDF reports for regulatory submission, bulk GIS import files, log archives.

Deployment: S3-compatible object storage (MinIO for self-hosted or AWS S3/Azure Blob for cloud).

Lifecycle policies:
- Photos: standard storage for 90 days, then archive tier (lower cost, slower access) for 7 years
- Regulatory PDFs: permanent retention, Write Once Read Many (WORM) mode
- GIS import files: 1 year retention
- Log archives: 2 years retention

### 3.5 Observability Stack

**Metrics — Prometheus + Grafana.**

Every service exposes Prometheus-compatible metrics on a `/metrics` endpoint. Prometheus scrapes every 15 seconds. Grafana dashboards visualise per-service health, engine performance, integration health, and business KPIs.

Key dashboard groups:
- Infrastructure health (CPU, memory, disk, network per host)
- Application health (request rate, error rate, latency P50/P95/P99 per service)
- Engine performance (runs completed, duration, tasks processed, optimization solution times)
- Integration health (per-integration message rates, error rates, circuit breaker state)
- Business KPIs (real-time SLA compliance, task completion rates)

**Logs — Loki.**

Structured JSON logging from every service. Correlation IDs propagated through every request. Logs queryable by correlation ID, service, log level, and time range.

Retention: 30 days hot in Loki, 2 years cold in object storage.

**Distributed tracing — OpenTelemetry + Jaeger.**

End-to-end traces across services for debugging complex flows. Enabled for 10% of requests in normal operation, 100% during incident investigation.

**Alerting — Alertmanager.**

Tiered alerting:
- Critical (P1): pager duty, immediate response required (examples: P1 dispatch path down, database primary unavailable, regulatory deadline approaching with system unable to submit)
- High (P2): alert within business hours (examples: single integration failing, engine run missed)
- Warning: email only (examples: elevated error rate, approaching capacity thresholds)

Alert routing respects on-call rotation schedules.

---

## 4. Environment Strategy

Four environments support the development lifecycle:

| Environment | Purpose | Data | Availability |
|---|---|---|---|
| Development | Feature development, unit testing | Synthetic | Business hours |
| Integration Test | Cross-team integration testing | Synthetic + sanitised sample | Business hours |
| UAT/Pre-production | User acceptance testing, performance testing | Sanitised production snapshot | Business hours |
| Production | Live operations | Real | 24×7 |

### 4.1 Data Sanitisation for Non-Production

Production data cannot be used directly in non-production environments. A sanitisation pipeline runs weekly to:
- Replace CONSUMER.name, contact_phone, contact_email, address with synthetic values
- Mask account_number, crm_account_id, meter_id
- Keep structural data intact (zone codes, asset codes, geographic coordinates) for realism
- Sanitised snapshot refreshed weekly to UAT, monthly to Integration Test

### 4.2 Configuration Separation

All environment-specific configuration (endpoints, credentials, thresholds) held in a secrets management system (HashiCorp Vault or equivalent). Application containers receive only the configuration appropriate to their deployment environment. No configuration baked into container images.

---

## 5. Security Architecture

### 5.1 Authentication and Authorisation

**Internal users (dispatcher, manager, field worker):**
- Single sign-on via corporate identity provider (SAML 2.0 or OpenID Connect)
- MFA required for dispatcher and manager roles
- Role-based access control enforced at API gateway and within each service
- Session timeout: 12 hours for dispatcher/manager, 24 hours for field worker (allows full shift)

**External systems:**
- OAuth 2.0 client credentials grant for REST APIs
- Mutual TLS for SCADA/EMS and regulatory portal
- Token rotation every 24 hours
- Separate client credentials per external system (SCADA, MDMS, CRM, Billing, Regulatory all have independent credentials)

**Role definitions:**

| Role | Permissions |
|---|---|
| Dispatcher | Read all tasks, write assignments, trigger matching engine re-runs, initiate clusters |
| Field Worker | Read own assigned tasks, write task progress, write isolation records, write service history on own tasks |
| Safety Officer | Field worker permissions + sign-off on others' isolation records |
| Planning Officer | Read all, write cluster approvals, write maintenance schedule changes |
| Operations Manager | Full read, no direct write (approvals only), override authority |
| Network Planner | Write ALT_SUPPLY_SOURCE, trigger supply optimisation, approve supply plans |
| System Administrator | Infrastructure access, no business data write access |
| Read-only Auditor | Read all operational records, no write |

### 5.2 Data Encryption

**At rest:**
- PostgreSQL: Transparent Data Encryption via disk encryption (LUKS or cloud provider equivalent)
- Redis: encryption at rest on underlying volumes
- Kafka: disk encryption on brokers
- Object storage: server-side encryption
- Backups: encrypted before leaving primary data centre

**In transit:**
- TLS 1.2 minimum for all network traffic (including internal service-to-service)
- Mutual TLS for data zone component communication (PostgreSQL, Redis, Kafka)
- Certificate management via automated PKI (e.g. cert-manager for internal certificates)

**Application-level encryption for PII:**
- Consumer contact details (phone, email, address) encrypted with service-managed keys
- Encryption keys rotated annually
- Decryption audited — every read logged with requesting service and purpose

### 5.3 Compliance with Data Localisation

All consumer PII and operational data remains within India.
- Primary data centre: within India
- DR site: different region of India
- Cloud services (if used): Indian regions only
- Object storage replication: Indian regions only

External system integrations (SCADA, MDMS, CRM, Billing, Regulatory) are all operated within India by the utility or its regulated service providers.

### 5.4 Audit Logging

Immutable audit log captures:
- All authentication events (success and failure)
- All authorisation decisions (granted and denied)
- All safety-critical writes (isolation records, PTW issuance, energization status changes)
- All configuration changes
- All data export activities
- All role and permission changes

Audit log is streamed to a dedicated Kafka topic and archived to Write-Once-Read-Many object storage. Retention: 7 years minimum (regulatory requirement).

### 5.5 Vulnerability Management

- OS patching: monthly baseline, emergency patches within 48 hours of vendor advisory
- Container images: rebuilt nightly with latest security patches; critical CVEs patched within 24 hours
- Dependency scanning: automated on every code commit and nightly for deployed versions
- Penetration testing: annual external penetration test, remediation of critical findings within 30 days
- Security incident response plan reviewed annually

---

## 6. Disaster Recovery and Business Continuity

### 6.1 Recovery Objectives

| Disaster scenario | RTO | RPO |
|---|---|---|
| Single server failure | < 1 minute (automatic) | 0 |
| Availability zone failure | < 5 minutes | 0 |
| Primary database failure | < 15 minutes | 0 |
| Entire primary data centre failure | < 4 hours | < 15 minutes |
| Cyber incident requiring recovery from backup | < 12 hours | < 24 hours |

### 6.2 Backup Strategy

**Database backups:**
- Continuous WAL streaming to DR site
- Full database dump daily to object storage (retained 30 days)
- Weekly full backup retained 1 year
- Monthly backup retained 7 years (regulatory archive)
- Backup integrity verified by automated restore test monthly

**Configuration backups:**
- Infrastructure-as-code committed to version control
- Secrets backed up to dedicated secure storage with quarterly rotation
- Runbooks version-controlled alongside infrastructure code

**Object storage:**
- Cross-region replication for regulatory documents
- Versioning enabled on all buckets

### 6.3 DR Site Operations

The DR site is a warm standby:
- Application zone infrastructure provisioned but not serving traffic
- Database replica receiving asynchronous WAL stream (typical lag: 5-15 seconds)
- Redis not replicated (cached data regenerated after failover)
- Kafka not replicated (short-term events; sync log replayed from database if needed)

**Failover procedure:**
1. Operations team declares DR activation (requires executive approval)
2. DNS records updated to point to DR site
3. DR database promoted to primary
4. DR application zone brought online
5. Health checks verify readiness
6. Traffic shifts to DR site
7. Primary site assessed for recovery or continued isolation

**Failback procedure (when primary is restored):**
1. Primary database resynced from DR as source of truth
2. Applications verified against primary
3. Traffic migrated back during maintenance window
4. DR site reset to standby role

### 6.4 Quarterly DR Drill

Every quarter, a scheduled DR drill validates:
- Failover procedure completes within RTO
- Data loss within RPO
- All critical business functions available at DR site
- Failback completes cleanly

Failures found during drills become remediation tasks with tracked completion.

---

## 7. Capacity Planning

### 7.1 Initial Sizing (Phase 1)

Assumed scale:
- 500 field workers
- 100,000 consumers
- 50,000 assets
- 10 concurrent dispatchers
- 1,000 tasks per day average, 3,000 peak day (storm scenarios)

Infrastructure sizing:
- Application zone: 6 worker nodes (8 cores, 32 GB RAM each)
- Database: 16 cores, 64 GB RAM, 2 TB SSD
- Redis: 3-node cluster, 16 GB RAM each
- Kafka: 3 brokers, 8 cores, 32 GB RAM each
- Object storage: 10 TB initial

Cost baseline: approximately 25-35% of equivalent cloud-native deployment if self-hosted on utility infrastructure.

### 7.2 Growth Planning (Phase 2)

Assumed scale:
- 2,000 field workers
- 500,000 consumers
- 200,000 assets
- 25 concurrent dispatchers
- 5,000 tasks per day average, 15,000 peak

Scaling approach:
- Application zone auto-scales horizontally (add worker nodes)
- Database: scale up (more RAM, faster storage) and add read replicas
- Redis: add cluster nodes if memory pressure observed
- Kafka: increase partitions and add brokers if throughput requires

### 7.3 Target Scale (Phase 3)

Assumed scale:
- 5,000 field workers
- 1,000,000 consumers
- 500,000 assets
- 50 concurrent dispatchers
- 10,000 tasks per day average

At this scale, database may require read/write splitting with application logic directing reads to replicas, plus consideration of geographic partitioning if the utility's operational footprint expands.

### 7.4 Storage Growth Projections

| Table | Phase 1 (annual) | Phase 3 (annual) | Retention |
|---|---|---|---|
| TASK | 1 GB | 10 GB | 7 years (then archive) |
| SOURCE_OPERATING_STATE | 200 GB | 2 TB | 90 days (partition drop) |
| CONSUMER_OUTAGE_RECORD | 5 GB | 50 GB | 7 years |
| CRM_NOTIFICATION | 10 GB | 100 GB | 18 months operational, 7 years regulatory |
| SERVICE_HISTORY | 2 GB | 20 GB | Permanent |
| METER_ALERT | 15 GB | 150 GB | 30 days post-resolution |
| Object storage (photos, PDFs) | 500 GB | 5 TB | 7 years |

---

## 8. Phased Go-Live Plan

The deployment follows the same four-phase structure as the functional implementation, with infrastructure provisioning aligned to application delivery.

### 8.1 Phase 0 — Foundation (Months 1-3)

**Infrastructure provisioned:**
- All three security zones in primary data centre
- PostgreSQL primary + 1 replica (DR replica deferred to Phase 1)
- Redis cluster
- Kafka cluster
- Object storage
- Kubernetes cluster with initial node pools
- Observability stack (Prometheus, Grafana, Loki)
- Secrets management
- CI/CD pipeline with blue-green deployment

**Application components:**
- Data model schema deployed (all 40 tables)
- GIS import tooling operational
- Initial asset registry populated from GIS
- Workforce, skill, and task type catalogues configured
- Development and Integration Test environments fully operational

**Go-live criteria:**
- All Phase 0 migrations applied successfully
- Smoke tests pass for data integrity
- GIS import validated with full dataset
- Backup and restore procedure tested
- Development teams have working environments

**Not yet operational in production:**
- Dispatcher board (UI only available in integration test)
- Mobile app (testing only)
- No external integrations yet live

### 8.2 Phase 1 — Assisted Dispatch (Months 3-6)

**Infrastructure additions:**
- DR site provisioned with data zone replicas
- Full HA pair across primary and DR sites
- Mobile BFF deployed to DMZ
- API gateway configured for external integrations (although only Billing and CRM go live here)
- DR drill procedure documented and tested

**Application components go-live:**
- Schedule Engine active in production
- Matching Engine active in production
- Dispatcher Board rolled out to one pilot division
- Field worker mobile app deployed to pilot workforce (50-100 workers)
- Billing integration live (consumer sync)
- CRM integration live (reactive notifications only)

**Rollout strategy:**

Weeks 1-4: Shadow mode
- Production system runs alongside legacy dispatch
- Dispatchers continue using old process
- New system receives the same tasks and generates suggestions in background
- Comparison reports daily showing what the new system would have done

Weeks 5-8: Pilot mode
- One division fully cut over to new system
- Dedicated support team on-call for issues
- Daily standup on user feedback and defect triage
- Other divisions continue legacy process

Weeks 9-12: Phased rollout
- Division-by-division migration (one per week)
- Each division has 3 days of shadow, 4 days of full cutover before next division
- Pilot division teams support new divisions during cutover

**Go-live criteria per division:**
- Skill match rate > 95% in shadow mode
- Dispatcher training complete (minimum 8 hours per dispatcher)
- Field workers trained on mobile app (minimum 2 hours per worker)
- All assets in division have GIS geocodes
- Consumer sync from billing complete for division
- Division-specific escalation procedures documented

**Rollback plan:**
If critical issue discovered post-cutover:
- Immediate: Revert affected tasks to manual dispatch via dispatch team
- Within 1 hour: Decision on system-wide rollback
- Rollback mechanism: DNS revert to legacy system (maintained in parallel for 90 days)
- Data reconciliation procedure documented

### 8.3 Phase 2 — Optimisation and Integration (Months 6-12)

**Infrastructure additions:**
- Scale horizontal capacity for full worker population
- Add dedicated engine worker nodes for computational workloads (Clustering, Supply Optimizer)
- Additional Redis cluster nodes for caching pre-computed N-1 plans

**Application components go-live:**
- Clustering Engine active in production
- Supply Optimization Engine (Mode 1 — single outage)
- Management Dashboard rolled out
- SCADA/EMS integration live (real-time SOURCE_OPERATING_STATE)
- Smart Meter MDMS integration live (last-gasp and restoration signals)
- CRM integration extended (proactive and regulatory notifications)
- Regulatory portal integration live (monthly reliability submission)
- Van stock auto-decrement active
- Route-aware travel time replacing haversine approximation

**Rollout strategy:**
Each of the new capabilities rolls out independently:

| Capability | Rollout |
|---|---|
| Clustering Engine | Soft launch — system proposes clusters, planner reviews before approving (week 1-2). Then full mode with automated proposals. |
| Supply Optimizer | Parallel run — system produces ranked plan, load dispatch centre validates manually for first month. Then trusted execution. |
| SCADA integration | Read-only initially (data flows in, not used for optimization). Then enable optimization use after 2 weeks of data quality validation. |
| Smart Meter integration | Starts with alert ingestion (tamper, PQ alerts generate tasks). Then enables last-gasp outage detection after threshold tuning. |
| Regulatory submission | First month: generated but submitted manually. Then automated submission after verification. |

**Go-live criteria:**
- SCADA data flowing with < 2% rejection rate for 1 week before enabling for optimization
- Clustering engine produces valid proposals with < 5% false positive rate
- Supply optimizer matches manual recommendations in > 90% of cases before trusted execution
- Smart meter last-gasp correlation threshold tuned to avoid false outage events
- Regulatory submission format accepted by portal in test submissions

### 8.4 Phase 3 — Intelligence and Prediction (Month 12+)

**Infrastructure additions:**
- ML model training infrastructure (typically separate from production cluster)
- Model serving infrastructure with A/B testing capability
- Enhanced monitoring for ML model performance and drift detection
- Integration with external power systems analysis tools (PSS/E, PowerFactory, OpenDSS)

**Application components go-live:**
- ML-based duration prediction (replaces static complexity band multipliers)
- ML-trained override learning model (fed by 12+ months of CREW_ASSIGNMENT override data)
- ILP-based Multi-outage Supply Optimizer (Mode 2 replacement)
- Continuous N-1 monitoring (replaces nightly batch)
- Predictive maintenance triggers based on asset condition trending
- MDMS interval data feeding load estimation

**Rollout strategy:**
- ML models deployed in shadow mode initially (predictions logged but not used)
- A/B testing: 10% of dispatches use ML-assisted suggestions, 90% use rule-based
- Gradual traffic shift based on measured outcomes (SLA compliance, override rates)
- Fallback to rule-based engine always available

**Go-live criteria:**
- Duration prediction accuracy within ±20% for 80% of task types
- Override learning model reduces override rate by at least 20% in A/B test
- Supply Optimizer ILP solver produces provably optimal solutions matching external validation
- Continuous N-1 monitoring does not exceed compute budget

---

## 9. Operational Runbook

### 9.1 Routine Operations

**Daily:**
- Monitor observability dashboards — infrastructure, applications, integrations
- Review overnight batch job outcomes (Schedule Engine nightly run, Clustering Engine nightly run, Supply Optimizer N-1 pre-computation)
- Check integration health (stale SCADA zones, MDMS message rate, CRM confirmation rate)
- Review any critical or high-severity alerts from previous 24 hours
- Validate backup completion

**Weekly:**
- Review override analysis report (weekly output from Management Dashboard)
- Review capacity metrics trending
- Check certificate expiry (internal and external) — 30 days ahead
- Security patch review
- Integration error rate review

**Monthly:**
- Full capacity planning review
- Cost review
- DR drill if scheduled
- Data sanitisation refresh for non-production environments
- Regulatory submission verification

**Quarterly:**
- DR drill (mandatory)
- Penetration test findings remediation review
- Integration contract review with external systems
- Access control audit

### 9.2 Incident Response

**Severity definitions:**

| Severity | Definition | Response target |
|---|---|---|
| SEV-1 | P1 dispatch path down or data integrity at risk | 15 minutes to acknowledge, 1 hour to resolve or declare DR |
| SEV-2 | Single integration down, some feature degraded | 30 minutes to acknowledge, 4 hours to resolve |
| SEV-3 | Non-critical feature issue, workaround available | Next business day |
| SEV-4 | Cosmetic issue, no operational impact | Next planned release |

**SEV-1 response flow:**

```
0. Alert fires → on-call engineer paged
1. Acknowledge within 15 minutes
2. Initial triage:
   - Is P1 dispatch affected?
   - Is data integrity compromised?
   - Is this cyber-related?
3. If cyber-suspected → Security team engaged immediately
4. Operations manager notified within 30 minutes
5. Escalation to IT Director if resolution not in sight at 45 minutes
6. Customer communications prepared (if consumer-facing impact)
7. Dispatchers informed of any manual workarounds needed
8. Post-incident review within 5 working days
```

### 9.3 Deployment Procedures

**Standard deployment (weekly):**
- Release candidate built from version-controlled code
- Deployed to Development → Integration Test → UAT → Production
- UAT signoff required before production deployment
- Blue-green production deployment
- Health validation on green before traffic cutover
- Blue retained for 24 hours as rollback

**Emergency deployment (security patch):**
- Expedited testing in UAT (focused on affected components)
- Deployment outside standard window permitted with change manager approval
- Additional monitoring for 48 hours post-deployment

**Configuration changes:**
- Stored in version control
- Applied via CI/CD pipeline, not manual console access
- Peer review required on every change
- Automatic rollback on health check failure

---

## 10. Third-Party Dependencies

### 10.1 Infrastructure Dependencies

| Component | Vendor / Open Source | Criticality | Fallback |
|---|---|---|---|
| PostgreSQL + PostGIS | Open source | Critical | None — core system |
| Redis | Open source | High | In-memory cache fallback with degraded performance |
| Kafka | Open source | High | Internal queue fallback (limited retention) |
| Kubernetes | Open source | High | Core container orchestration |
| Object storage | MinIO / cloud vendor | Medium | Local filesystem fallback for short-term |
| Prometheus / Grafana | Open source | Medium | Cloud observability fallback |
| HashiCorp Vault | Commercial / open source | High | Alternative secrets store |
| Identity provider | Corporate (SAML / OIDC) | Critical | Emergency break-glass account |

### 10.2 External System Dependencies

Already covered in Integration Specifications — SCADA/EMS, Smart Meter/MDMS, GIS, Billing, CRM, Regulatory Portal. Each has its own operational agreement with response time commitments.

### 10.3 License and Support

- PostgreSQL: community support with optional commercial support (EnterpriseDB or similar)
- Redis: community support with optional commercial support (Redis Inc.)
- Kubernetes: community + commercial support (Red Hat OpenShift or similar if on-premise)
- WAF: commercial product with vendor SLA

Critical components have both community and commercial support paths. For P1 issues with open-source dependencies, commercial support contracts ensure vendor engagement within defined response times.

---

## 11. Cost Architecture

### 11.1 Cost Components

**Infrastructure costs:**
- Compute (Kubernetes nodes)
- Storage (PostgreSQL, object storage, backups)
- Network (data transfer, DR replication)
- Security (WAF, penetration testing)

**Operations costs:**
- DevOps engineers (typically 3-5 for Phase 1, scaling with system)
- Database administrator
- Security engineer (shared role with other systems)
- On-call compensation

**License and support:**
- Commercial support contracts
- Managed service components (if used)
- Commercial software components

### 11.2 Cost Comparison — Self-Hosted vs Cloud

Self-hosted on utility infrastructure generally provides better total cost of ownership for a utility's multi-year FSM deployment, but cloud provides faster initial provisioning and easier scaling. The decision depends on:
- Existing utility IT infrastructure capability
- Data residency constraints (India-only requirement addressable in both models)
- Team capability to operate infrastructure
- Capital vs operational expenditure preference

For a utility of 1-2 million consumers, self-hosted deployment typically requires 15-20 full-time operations staff. Cloud deployment typically requires 8-12 staff but has higher recurring cloud costs.

Recommendation: Start cloud for Phase 1 speed to market, evaluate for migration to self-hosted at Phase 2 once requirements are stable.

---

## 12. Key Deployment Design Decisions

**Why PostgreSQL rather than a NoSQL document store for the flexible JSONB fields?**
Safety-critical fields (isolation records, energization status) benefit from strong consistency and transactional integrity that document stores typically trade off. PostgreSQL provides both — relational integrity for the core schema and JSONB flexibility where needed (TEST_RECORD.test_readings, DOCUMENT_TEMPLATE.fields_schema). Using two data stores would double operational complexity for limited architectural benefit.

**Why synchronous replicas within primary site plus asynchronous to DR?**
Synchronous replication guarantees zero data loss for intra-site failures but adds latency for write operations. Within a single data centre the latency is negligible (sub-millisecond). Across data centres it would be prohibitive. Asynchronous DR replication accepts a small data loss window (15 minutes) in exchange for no write-path latency penalty — aligned with the 15-minute RPO target.

**Why Kafka rather than a simpler message queue?**
Three reasons. First, the CRM integration benefits from Kafka's partition-key ordering to preserve per-consumer event order. Second, the audit log use case requires long retention that is natural in Kafka but awkward in traditional queues. Third, Kafka's multi-subscriber pattern allows future additions (e.g. a data warehouse subscriber for analytics) without modifying the publisher.

**Why run the four engines as separate services rather than a monolith?**
Each engine has different scaling characteristics. Matching Engine must scale with task creation rate — it benefits from horizontal autoscaling. Supply Optimizer is CPU-intensive and benefits from dedicated compute nodes. Schedule Engine is a nightly batch — it benefits from being isolated from real-time paths. Separate services also allow independent deployment, so a change to clustering logic does not require redeploying matching.

**Why maintain the legacy dispatch system in parallel for 90 days post-cutover?**
The cost of maintaining it is small. The cost of having no rollback path if the new system fails in a way that requires reverting is enormous. 90 days gives confidence that seasonal patterns (monsoon, summer peak load, winter fog) have all been experienced under the new system before decommissioning the fallback.

**Why is the DR site a warm standby rather than active-active?**
Active-active across geographically separated data centres introduces complexity around write conflicts, split-brain scenarios, and latency on synchronous operations. For a utility with a single operational region, the complexity is not justified by the modest RTO improvement (minutes vs hours). Warm standby is simpler to operate, tested quarterly, and adequate for the stated recovery objectives.

**Why quarterly DR drills rather than monthly or annually?**
Monthly drills are operationally expensive and tend to create "drill fatigue" where teams treat them as routine rather than serious exercises. Annual drills allow too many configuration drifts to accumulate between tests. Quarterly strikes the balance — frequent enough to catch drift, infrequent enough to treat seriously.

**Why does the worker mobile app support 7 days of offline operation rather than shorter?**
Field workers in remote areas (rural zones, substation basements, underground cable routes) may have extended periods of poor connectivity. Designing for 7 days provides margin for multi-day maintenance visits in remote locations without forcing workarounds. The 7-day window is sufficient to complete any reasonable single task assignment including isolation, work, and documentation.

---

*End of Deployment Architecture Document*
*FSM System · Deployment Architecture · Version 1.0 · April 2026*
