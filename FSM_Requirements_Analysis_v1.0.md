# Field Service Management System — Requirements Analysis
## Utility Field Response Optimization
**Document version:** 1.0
**Date:** April 2026
**Status:** Approved for development

---

## 1. Executive Summary

This document defines the complete requirements for a Field Service Management (FSM) system for a utility's Field Response team. The system addresses the full operational lifecycle of field work — from automatic task generation through skill-matched dispatch, route optimization, safety-governed execution, consumer communication, and reliability performance measurement.

The system is designed to deliver measurable improvement across six operational dimensions: SLA compliance, field workforce productivity, consumer satisfaction, network reliability indices (SAIFI/SAIDI/CAIFI/CAIDI), regulatory compliance, and operational cost efficiency.

---

## 2. Problem Statement

A utility's Field Response team operates a continuous, multi-priority task environment spanning emergency fault repairs, consumer complaint resolution, planned maintenance, scheduled inspections, routine testing, network upgrades, and vegetation management. This environment has the following characteristics that make manual management increasingly inadequate:

**Dynamic task inflow:** Tasks arrive from multiple sources — consumer complaints, field-detected faults, SCADA alerts, smart meter signals, and calendar-driven maintenance schedules — at unpredictable rates and with varying urgency. The task queue is never static.

**Heterogeneous workforce:** Field workers carry different certifications, tool authorizations, and equipment. Not every worker can perform every task. Skill mismatch at dispatch leads to re-dispatch, SLA breach, and safety risk.

**Multi-objective optimization:** No single metric governs good dispatch. The system must simultaneously minimize travel time, maximize tasks completed per shift, respect SLA deadlines, balance workload equitably, and minimize total network outage impact on consumers.

**Safety-critical operations:** Work on live electrical infrastructure requires formal isolation, permit-to-work, and re-energization confirmation. These are not administrative steps — they are mandatory safety protocols that must be enforced by the system, not left to individual judgment.

**Consumer impact visibility:** Every planned or unplanned shutdown affects consumers who deserve advance notice, real-time status updates, and confirmation of restoration. Consumer communication is both a regulatory obligation and a service quality imperative.

**Reliability measurement:** Regulatory bodies require periodic reporting of SAIFI, SAIDI, CAIFI, and CAIDI indices. These cannot be computed accurately without structured, per-consumer outage impact data that is not captured in current systems.

---

## 3. System Objectives

The primary stated objective:

> *"Constant maintenance, emergency repairs, and infrastructure upgrades are crucial for uninterrupted energy and utility service delivery. Field service dispatch software that allows for real-time updates and in-day scheduling that can adapt to the unpredictable nature of the industry. With the help of schedule optimization and job duration prediction, utility companies will be better equipped to respond to outages and service disruptions. Real-time data on service history will bolsters efficiency and increases technician productivity. Moreover, inventory management features will ensure that necessary spare parts are available to address service requests without delay."*

Derived from this, the system must achieve six measurable objectives:

| # | Objective | Primary metric |
|---|---|---|
| O1 | Reduce unmatched dispatch — right skill to right task | Skill match rate % |
| O2 | Reduce travel time and maximize tasks per shift | Tasks completed per shift · Average travel time per task |
| O3 | Minimize consumer interruption frequency and duration | SAIFI · CAIFI |
| O4 | Minimize consumer interruption duration | SAIDI · CAIDI |
| O5 | Achieve and maintain SLA compliance | SLA compliance % by priority class |
| O6 | Ensure regulatory compliance — safety and reporting | PTW compliance % · Regulatory submission rate |

---

## 4. Stakeholder Analysis

| Stakeholder | Role | Primary system interaction |
|---|---|---|
| Field Response dispatcher | Assigns tasks to crew in real time | Dispatcher board — receive suggestions, confirm assignments, monitor SLA |
| Field worker / lineman | Executes tasks on site | Mobile app — receive tasks, record isolation, update status, capture test data |
| Planning officer | Plans scheduled maintenance and shutdowns | Cluster planning — approve clusters, review advance notice, manage shutdown windows |
| Operations manager | Monitors field operations performance | Management dashboard — KPIs, SLA, reliability indices, crew utilization |
| Safety officer | Co-dispatched for HV and permit-required tasks | Mobile app — sign PTW, confirm isolation, sign completion |
| Load dispatch center | Manages network switching and alternative supply | Supply switching plan — approve and execute switching sequences |
| Consumer (external) | Affected by faults and planned outages | CRM — receives notifications, raises complaints, provides feedback |
| Regulatory body (external) | Receives compliance reports | Regulatory reporting — SAIFI/SAIDI reports, test certificates, PTW records |
| Network planning team | Defines and maintains alternative supply catalog | ALT_SUPPLY_SOURCE — defines and approves permissible supply paths |
| IT / system administrator | Maintains system configuration | Configuration parameters, integration management |

---

## 5. Scope

### 5.1 In Scope

**Task streams:**
- Reactive stream: Fault detection and emergency repair, consumer complaints and meter services, corrective maintenance (post-detection), outage restoration and storm response
- Scheduled stream: Asset inspection (lines, poles, battery backups, UPS, plants, substations), routine testing (transformers, protection relays), preventative maintenance, safety and compliance audits, vegetation management, DER and smart meter installation
- Project stream: Network upgrades and infrastructure expansion, grid integration and commissioning

**Core engines:**
- Schedule engine: Automatic task generation from MAINTENANCE_SCHEDULE
- Matching engine: Skill-matched, proximity-ranked dispatch suggestions
- Clustering engine: Task clubbing for SAIFI minimization and planned outage management
- Supply optimization engine: Alternative supply selection minimizing losses and maximizing reliability

**Interfaces:**
- Dispatcher board (web)
- Field worker mobile app (iOS and Android)
- Management dashboard (web)

**Integrations:**
- SCADA / EMS: Real-time asset loading, voltage, and energization status
- Smart Meter / MDMS: Last-gasp outage signals, restoration confirmation, consumer load data
- GIS: Asset geocoding, zone boundaries, network topology
- Billing system: Consumer registry, zone consumer counts
- CRM system: Consumer notifications, complaint management, feedback capture
- Regulatory portal: Compliance report submission

### 5.2 Out of Scope

- Full load flow simulation (integration hook provided for Phase 3 — PSS/E, PowerFactory, OpenDSS)
- Consumer billing and revenue management
- HR and payroll management
- Procurement and purchase order management (inventory alerts are in scope; procurement workflow is not)
- Asset capital planning and investment appraisal

---

## 6. Task Catalog — Complete Taxonomy

### 6.1 Task Classification

Every task belongs to one of three streams, carries one of four priority classes, and uses one of two SLA mechanisms:

| Stream | SLA mechanism | Auto-generated | Documentation output |
|---|---|---|---|
| Reactive | Response-time clock | No — event triggered | Job completion record |
| Scheduled | Compliance date | Yes — from MAINTENANCE_SCHEDULE | Test certificate / Inspection report / Audit report |
| Project | Milestone date | No — project plan driven | Commissioning certificate |

### 6.2 Reactive Stream Tasks

| Task type | Priority | SLA | Min crew | Hard co-dispatch rule | Workforce pool |
|---|---|---|---|---|---|
| Fault detection and emergency repair | P1 | 1–2 hrs | 2 | Safety officer + HV authority required | Electrical |
| Outage restoration and storm response | P1 | Immediate | 4–10 | Incident commander + HV authority | Electrical |
| Consumer complaint and meter services | P2 | Same day | 1 | None | Electrical |
| Corrective maintenance (post-detection) | P2 | 4–8 hrs | 1–3 | None | Electrical |

### 6.3 Scheduled Stream Tasks

| Task type | Priority | Frequency | Min crew | Documentation |
|---|---|---|---|---|
| Asset inspection — lines and poles | P3 | Per inspection cycle | 1–2 | Inspection report + photographs |
| Asset inspection — battery backups and UPS | P3 | Manufacturer schedule | 1–2 | Battery test report + capacity readings |
| Plant and substation inspection | P3 | Regulatory schedule | 2–3 | Inspection report + compliance certificate |
| Routine testing — transformers | P3 | Test interval schedule | 2 | Test certificate (dielectric, oil, turns ratio) |
| Routine testing — protection relays | P3 | Protection test schedule | 1–2 | Relay test record + coordination check |
| Preventative maintenance (scheduled) | P3 | Asset maintenance schedule | 1–3 | Maintenance record + parts log |
| Safety and compliance audit | P3 | Regulatory deadline | 2 | Audit report + corrective action log |
| Vegetation management | P4 | Seasonal / vegetation cycle | 2–4 | Clearance record + photographs |
| DER and smart meter installation | P3 | Customer appointment | 1–2 | Commissioning report + grid connection cert |

### 6.4 Project Stream Tasks

| Task type | Priority | Duration | Min crew | Documentation |
|---|---|---|---|---|
| Network upgrades and infrastructure expansion | P3 | 1–5 days | 4–10 | Project completion cert + as-built drawings |
| Grid integration and commissioning | P3 | 1–3 days | 2–4 | Commissioning cert + SCADA integration record |

### 6.5 Weather Sensitivity Matrix

| Task type | Rain | Lightning | High wind | Extreme heat | Flood | Duration impact |
|---|---|---|---|---|---|---|
| Fault repair (HV) | Conditional | BLOCKED | BLOCKED | Extended | BLOCKED | +30–50% |
| Line and pole inspection | Conditional | BLOCKED | Conditional | Extended | BLOCKED | +40–60% |
| Substation inspection | Conditional | BLOCKED | OK | Extended | BLOCKED | +20–30% |
| Transformer testing | Conditional | BLOCKED | OK | Extended | OK | +20–30% |
| Relay testing (indoor) | OK | OK | OK | OK | Conditional | Minimal |
| Battery inspection (indoor) | OK | OK | OK | OK | Conditional | Minimal |
| Meter services | OK | OK | OK | Extended | Conditional | +15–20% |
| Vegetation management | Conditional | BLOCKED | BLOCKED | Extended | BLOCKED | +30–40% |
| Network upgrades | Conditional | BLOCKED | BLOCKED | Extended | BLOCKED | +40–60% |

---

## 7. Functional Requirements

### 7.1 Schedule Engine

| ID | Requirement |
|---|---|
| SE-01 | The system shall automatically generate TASK records from active MAINTENANCE_SCHEDULE entries when today >= next_due_date minus advance_notice_days |
| SE-02 | The system shall generate one task per asset where asset_type_id matches; or one task for the specific asset where asset_id is set |
| SE-03 | The system shall adjust estimated task duration based on asset condition_score using the complexity band matrix |
| SE-04 | The system shall update MAINTENANCE_SCHEDULE.next_due_date and last_generated_date after each task generation |
| SE-05 | The system shall override next_due_date from TEST_RECORD.next_test_due_date when a conditional pass or shortened interval is specified |
| SE-06 | The system shall create a SLA_TRACKER record for every generated task with appropriate compliance-date deadline |
| SE-07 | The system shall fire 30-day, 7-day, and 1-day progressive SLA alerts for compliance-date tasks |
| SE-08 | The system shall fire response-time SLA alerts for reactive tasks in hours |
| SE-09 | The system shall set weather-sensitive outdoor tasks to ON_HOLD status when a HIGH or EXTREME weather event is active in the relevant zone |
| SE-10 | Weather holds shall not pause the compliance clock — the SLA deadline continues to run |
| SE-11 | The system shall scan for clustering opportunities immediately after each task generation and set clubbing_opportunity_flagged where applicable |
| SE-12 | The system shall run nightly at a configurable time and run a lighter alert sweep every 6 hours |
| SE-13 | The system shall update ASSET.last_inspection_date or last_test_date and condition_score on task completion |

### 7.2 Matching Engine

| ID | Requirement |
|---|---|
| ME-01 | The system shall trigger matching for every NEW task within the configured time limit for its priority class (P1: immediate, P2: 2 min, P3: 15 min, P4: daily) |
| ME-02 | Step 1 — The system shall exclude all workers whose workforce_pool does not match the task type's workforce_pool |
| ME-03 | Step 2 — The system shall exclude workers lacking any mandatory skill from TASK_SKILL_REQUIREMENT |
| ME-04 | Step 2 — The system shall exclude workers with expired certifications (WORKER_SKILL.is_active = false or expiry_date < today) |
| ME-05 | Step 2 — The system shall enforce is_safety_officer and is_hv_authority requirements where mandated by task type |
| ME-06 | Step 3 — The system shall exclude workers whose remaining shift time is less than estimated task duration plus travel buffer |
| ME-07 | Step 3 — P1 emergency tasks shall override shift end constraints and flag "overtime likely" |
| ME-08 | Step 4 — The system shall check mandatory parts against VAN_STOCK and flag depot pickup requirement where van stock is insufficient |
| ME-09 | The system shall score eligible workers by composite score: 50% proximity, 30% workload, 20% skill preference |
| ME-10 | The system shall surface the top 3 ranked candidates to the dispatcher with plain-language reasons |
| ME-11 | The system shall generate co-dispatch pair suggestions when safety officer co-dispatch is required |
| ME-12 | The system shall capture suggestion_rank, override_reason_code, and override_notes for every assignment |
| ME-13 | The system shall re-run matching after 30 minutes if no assignment has been made |
| ME-14 | The system shall re-run matching when worker status, location, or van stock changes affect the eligible pool |
| ME-15 | Scoring weights shall be configurable without code changes |

### 7.3 Clustering Engine

| ID | Requirement |
|---|---|
| CE-01 | The system shall identify same-isolation-scope clustering opportunities by querying tasks sharing an upstream switching point within a configurable date window |
| CE-02 | The system shall identify same-corridor clustering opportunities by querying tasks within a configurable geographic radius and date window |
| CE-03 | The system shall compute estimated consumer impact before presenting a cluster for approval |
| CE-04 | The system shall identify and flag CRITICAL_INFRASTRUCTURE consumers in the affected area |
| CE-05 | The system shall trigger SUPPLY_OPTIMIZATION_RUN for every proposed cluster |
| CE-06 | The system shall compute SAIDI saving versus the individual-shutdown alternative |
| CE-07 | On cluster approval, the system shall create a single OUTAGE_EVENT record for the entire cluster |
| CE-08 | On cluster approval, the system shall trigger regulatory advance consumer notifications at 72 hours and 24 hours before the planned window |
| CE-09 | The system shall create individual coordination tasks for CRITICAL_INFRASTRUCTURE consumers |
| CE-10 | The system shall trigger revised notifications if the cluster window changes after initial notification |
| CE-11 | The system shall monitor execution progress and trigger overrun notifications if the window is exceeded |
| CE-12 | On cluster completion, the system shall trigger supply-restored notifications to all affected consumers |
| CE-13 | On cluster completion, the system shall auto-close consumer complaint tasks linked to the outage and send resolution notifications |
| CE-14 | The system shall update OUTAGE_EVENT with actual duration and recompute consumer_minutes_interrupted |

### 7.4 Supply Optimization Engine

| ID | Requirement |
|---|---|
| SO-01 | The system shall maintain a pre-cataloged ALT_SUPPLY_SOURCE table defining every permissible supply path for every zone and asset |
| SO-02 | No alternative supply path shall be used operationally unless it exists in ALT_SUPPLY_SOURCE as an active record |
| SO-03 | The system shall read SOURCE_OPERATING_STATE to determine current loading and available margin before evaluating any supply path |
| SO-04 | The system shall exclude supply sources with availability_flag = UNAVAILABLE from optimization |
| SO-05 | The system shall flag supply sources with availability_flag = MARGINAL and deprioritize them in scoring |
| SO-06 | The system shall score feasible supply paths by composite score: configurable weights for loss minimization and reliability |
| SO-07 | The system shall produce ranked SUPPLY_SWITCHING_PLAN records (primary, fallback, tertiary) for each outage |
| SO-08 | For multi-outage scenarios, the system shall solve supply assignment as a constrained problem ensuring no source exceeds its available_margin_mva |
| SO-09 | The system shall record every optimization run in SUPPLY_OPTIMIZATION_RUN with full input parameters and outcome |
| SO-10 | The system shall run nightly contingency pre-computation (N-1) and flag zones with no viable alternative |
| SO-11 | The system shall check SOURCE_OPERATING_STATE staleness and warn if readings exceed the configured threshold |
| SO-12 | OUTAGE_EVENT.status shall not advance to APPROVED unless a SUPPLY_SWITCHING_PLAN exists or a documented override is recorded |

### 7.5 Isolation and Energization Safety

| ID | Requirement |
|---|---|
| IS-01 | The system shall enforce that TASK.status cannot transition to IN_PROGRESS unless ISOLATION_RECORD.pre_work_confirmed = true for tasks requiring a permit |
| IS-02 | The system shall enforce that TASK.status cannot transition to COMPLETED unless ISOLATION_RECORD.post_work_confirmed = true |
| IS-03 | The system shall require confirmation of both equipment energization status and source (feeding line) status at pre-work and post-work stages |
| IS-04 | Pre-work and post-work confirmations shall only be accepted from workers with is_hv_authority = true |
| IS-05 | The system shall maintain ISOLATION_POINT records for each individual switching step in sequence |
| IS-06 | All isolation points must show pre_action_confirmed = true before pre_work_confirmed can be set |
| IS-07 | All isolation points must show post_action_confirmed = true before post_work_confirmed can be set |
| IS-08 | The system shall record shutdown window start and end and flag overruns in ISOLATION_RECORD.shutdown_overrun |
| IS-09 | Overrun records shall require a mandatory overrun_reason text before the task can be marked complete |
| IS-10 | ASSET.energization_status shall be updated to reflect confirmed pre-work and post-work states |

### 7.6 Inventory Management

| ID | Requirement |
|---|---|
| INV-01 | The system shall check VAN_STOCK against TASK_PART_REQUIREMENT for mandatory parts at dispatch time |
| INV-02 | Insufficient van stock for mandatory parts shall prompt a depot check and flag "depot pickup required" |
| INV-03 | The system shall maintain DEPOT_STOCK levels and trigger reorder alerts when stock falls below reorder_level |
| INV-04 | Task completion shall automatically decrement VAN_STOCK or DEPOT_STOCK based on PARTS_CONSUMED |
| INV-05 | Van stock that falls below minimum_level shall trigger a restock alert |
| INV-06 | Critical spare parts (is_critical = true) shall be flagged if stock at any location falls to zero |

### 7.7 Outage Impact and Reliability

| ID | Requirement |
|---|---|
| RI-01 | The system shall create one OUTAGE_EVENT record per cluster (not per task) |
| RI-02 | The system shall create CONSUMER_OUTAGE_RECORD for every consumer in the affected zone |
| RI-03 | The system shall capture supply_lost_at from smart meter last-gasp signals where available |
| RI-04 | The system shall capture supply_fully_restored_at from smart meter restoration signals where available |
| RI-05 | The system shall compute effective_interruption_minutes per consumer accounting for alternative supply |
| RI-06 | The system shall compute consumer_minutes_interrupted at the outage event level |
| RI-07 | The system shall compute and store SAIFI, SAIDI, CAIFI, and CAIDI in RELIABILITY_SNAPSHOT per zone per period |
| RI-08 | RELIABILITY_SNAPSHOT shall distinguish planned from unplanned outage events |
| RI-09 | The system shall record saifi_events_saved_by_clustering per reporting period |
| RI-10 | CRITICAL_INFRASTRUCTURE consumers shall be identifiable in all outage reports |

### 7.8 Consumer Communication (CRM Integration)

| ID | Requirement |
|---|---|
| CC-01 | The system shall notify consumers at every significant task status change for complaint tasks |
| CC-02 | The system shall send regulatory advance notice to all affected consumers at 72 hours and 24 hours before a planned shutdown |
| CC-03 | Advance notices shall be sent per consumer communication preference (channel, language) |
| CC-04 | The system shall send updated notices if cluster window is revised after initial notification |
| CC-05 | The system shall send supply-restored notifications on outage event completion |
| CC-06 | All regulatory notifications shall be stored in CRM_NOTIFICATION with regulatory_record = true |
| CC-07 | The system shall create individual coordination tasks for CRITICAL_INFRASTRUCTURE consumers |
| CC-08 | Consumer opt-out and consent preferences shall be respected for all communication types |
| CC-09 | Alternative supply arrangement shall trigger an immediate notification to affected consumers |
| CC-10 | Overrun notifications shall be sent if restoration will exceed the planned window by more than 15 minutes |
| CC-11 | Complaint tasks shall be auto-closed and resolution notification sent on supply restoration |

### 7.9 Service History and Compliance

| ID | Requirement |
|---|---|
| SH-01 | SERVICE_HISTORY records shall be linked to assets, not tasks, and persist permanently |
| SH-02 | Field workers shall be able to view asset service history before starting any task on that asset |
| SH-03 | Task completion shall be blocked until all mandatory DOCUMENT_TEMPLATE fields are populated in TEST_RECORD |
| SH-04 | TEST_RECORD shall store test readings in flexible JSONB format validated against the document template schema |
| SH-05 | TEST_RECORD.next_test_due_date shall automatically update MAINTENANCE_SCHEDULE.next_due_date |
| SH-06 | Regulatory test certificates with submission_status = PENDING shall be submitted to the regulatory portal |
| SH-07 | Regulatory records shall not be deletable once created |

---

## 8. Non-Functional Requirements

### 8.1 Performance

| ID | Requirement |
|---|---|
| NF-P01 | P1 matching engine response time: suggestions surfaced within 5 seconds of task creation |
| NF-P02 | P2 matching engine response time: suggestions surfaced within 2 minutes |
| NF-P03 | Dispatcher board page load: initial load within 3 seconds, live updates within 1 second |
| NF-P04 | Mobile app task sync: new assignment notification within 30 seconds of dispatcher confirmation |
| NF-P05 | Schedule engine nightly run: complete processing of all MAINTENANCE_SCHEDULE records within 30 minutes |
| NF-P06 | Supply optimization (single outage): solution within 10 seconds |
| NF-P07 | Supply optimization (multi-outage, up to 10 simultaneous zones): solution within 60 seconds |
| NF-P08 | Reliability snapshot computation: monthly indices computed within 5 minutes of period end |
| NF-P09 | Consumer notification dispatch: CRM event published within 60 seconds of triggering state change |

### 8.2 Availability

| ID | Requirement |
|---|---|
| NF-A01 | Dispatcher board: 99.5% availability during operational hours (05:00–23:00) |
| NF-A02 | P1 emergency dispatch: 99.9% availability at all times including maintenance windows |
| NF-A03 | Mobile app: functional in degraded mode (offline task view) when connectivity is lost; sync on reconnection |
| NF-A04 | Schedule engine: failover mechanism — if nightly run fails, alerts sent to IT within 30 minutes |
| NF-A05 | Planned maintenance windows: maximum 4 hours per month, scheduled outside peak operational hours |

### 8.3 Security

| ID | Requirement |
|---|---|
| NF-S01 | Role-based access control: Dispatcher, Field Worker, Planner, Manager, Administrator, Read-Only roles |
| NF-S02 | PTW and isolation records: accessible only to workers with HV authority and designated safety roles |
| NF-S03 | Consumer personal data: access restricted to consumer-facing roles; field workers see only name and contact |
| NF-S04 | All data in transit encrypted (TLS 1.2 minimum) |
| NF-S05 | All data at rest encrypted |
| NF-S06 | Mobile app: biometric or PIN authentication; auto-lock after 5 minutes |
| NF-S07 | Audit log: all safety-critical actions (PTW issue, isolation confirmation, energization) logged with user, timestamp, and device |
| NF-S08 | CRM integration: consumer data exchanged over encrypted API with token-based authentication |

### 8.4 Scalability

| ID | Requirement |
|---|---|
| NF-SC01 | System shall support up to 500 concurrent field workers |
| NF-SC02 | System shall support up to 10,000 active tasks in the queue simultaneously |
| NF-SC03 | System shall support up to 1,000,000 CONSUMER records |
| NF-SC04 | System shall support up to 500,000 ASSET records |
| NF-SC05 | Historical data (SERVICE_HISTORY, CRM_NOTIFICATION, TEST_RECORD) shall be retained indefinitely with archiving after 7 years |
| NF-SC06 | Database shall support horizontal read scaling to accommodate dashboard and reporting query load |

### 8.5 Usability

| ID | Requirement |
|---|---|
| NF-U01 | Dispatcher board: a trained dispatcher shall be able to assign a P1 task within 60 seconds of it appearing |
| NF-U02 | Mobile app: field worker shall be able to update task status with no more than 3 taps |
| NF-U03 | Isolation workflow on mobile: each step clearly labelled; confirmation requires explicit acknowledgment |
| NF-U04 | Mobile app shall support operation in both English and Hindi (extensible to additional languages) |
| NF-U05 | Management dashboard: all KPI metrics visible on a single screen without scrolling |

---

## 9. Business Rules

### 9.1 Safety Rules (non-negotiable — system enforced)

| Rule | Description |
|---|---|
| BR-S01 | A task requiring a permit cannot start without ISOLATION_RECORD.pre_work_confirmed = true |
| BR-S02 | A task cannot be marked complete without ISOLATION_RECORD.post_work_confirmed = true |
| BR-S03 | Both equipment and source (feeding line) status must be confirmed at pre-work and post-work |
| BR-S04 | Only workers with is_hv_authority = true may confirm energization status |
| BR-S05 | A safety officer must be present (as lead or co-dispatch) for any task where requires_safety_officer = true |
| BR-S06 | Isolation points must be executed in sequence_number order |
| BR-S07 | All isolation points must be confirmed before pre_work_confirmed can be set |
| BR-S08 | Task completion is blocked if DOCUMENT_TEMPLATE mandatory fields are not populated |
| BR-S09 | Overrun of shutdown window requires mandatory overrun_reason before task can close |

### 9.2 Dispatch Rules

| Rule | Description |
|---|---|
| BR-D01 | A worker with an expired certification (WORKER_SKILL.is_active = false) shall never appear in suggestions for tasks requiring that skill |
| BR-D02 | Workforce pool mismatch eliminates a worker from all consideration — no exceptions |
| BR-D03 | A P1 emergency task immediately surfaces suggestions regardless of dispatcher's current activity |
| BR-D04 | Dispatcher override always requires selection of an override_reason_code |
| BR-D05 | Workers on ON_LEAVE or OFF_SHIFT status are excluded from all matching |
| BR-D06 | Parts unavailability does not prevent dispatch but must be flagged to the dispatcher |

### 9.3 Scheduling Rules

| Rule | Description |
|---|---|
| BR-SC01 | A task shall not be generated if one already exists for the same asset and schedule_id with status not in (COMPLETED, CANCELLED) |
| BR-SC02 | Weather holds do not pause compliance deadlines |
| BR-SC03 | A conditional test pass may specify a shorter re-test interval that overrides the schedule's frequency |
| BR-SC04 | CRITICAL_INFRASTRUCTURE consumers must receive individual (not bulk) advance notification for planned shutdowns |

### 9.4 Reliability Rules

| Rule | Description |
|---|---|
| BR-R01 | One cluster = one OUTAGE_EVENT — tasks within a cluster do not generate separate outage events |
| BR-R02 | SAIDI and CAIDI computations use effective interruption (after alternative supply deduction), not gross interruption |
| BR-R03 | No ALT_SUPPLY_SOURCE record may be used operationally unless approved_date is populated and valid_until > today |
| BR-R04 | A supply source with availability_flag = UNAVAILABLE must not appear in any SUPPLY_SWITCHING_PLAN |
| BR-R05 | Outage approval requires a SUPPLY_SWITCHING_PLAN or a documented override with reason |

### 9.5 Communication Rules

| Rule | Description |
|---|---|
| BR-C01 | Advance notice for planned shutdowns is mandatory at 72 hours and 24 hours — regulatory obligation |
| BR-C02 | Consumer opt-out preferences must be respected; opted-out consumers receive no non-regulatory communications |
| BR-C03 | Regulatory notices (CRM_NOTIFICATION.regulatory_record = true) cannot be deleted |
| BR-C04 | Overrun of planned window by more than 15 minutes triggers mandatory consumer notification |

---

## 10. Integration Requirements

### 10.1 SCADA / EMS (Load Dispatch Center)

| Attribute | Specification |
|---|---|
| Direction | Inbound to FSM |
| Data provided | Asset real-time loading (MVA), voltage (pu), CB/isolator status, available margin |
| Target table | SOURCE_OPERATING_STATE, ASSET.energization_status |
| Update frequency | Every 5–15 minutes (real-time feed during active outage management) |
| Protocol | REST API or OPC-UA depending on EMS platform |
| Staleness threshold | 30 minutes — readings older than this flagged as stale |
| Phase | Phase 2 |

### 10.2 Smart Meter / MDMS (Head-End System)

| Attribute | Specification |
|---|---|
| Direction | Inbound to FSM |
| Data provided | Last-gasp outage signals, power-restored signals, 15-min interval consumption, tamper/PQ alerts |
| Target tables | CONSUMER_OUTAGE_RECORD (timestamps), METER_ALERT (staging), TASK (auto-generated from alerts) |
| Update frequency | Event-driven for last-gasp and restore; 15-minute intervals for load data |
| Protocol | REST API or MQTT depending on MDMS platform |
| Consumer link | CONSUMER.meter_id matches MDMS meter identifier |
| Phase | Phase 2 (event signals), Phase 3 (load data for optimization) |

### 10.3 GIS

| Attribute | Specification |
|---|---|
| Direction | Inbound to FSM (initial import + periodic sync) |
| Data provided | Asset geocodes (lat/lng), zone polygon boundaries, feeder routing, network topology |
| Target tables | LOCATION, ASSET (geocodes), NETWORK_ZONE, ALT_SUPPLY_SOURCE (feeder routing) |
| Update frequency | Initial import at Phase 0; re-sync on network changes |
| Protocol | GIS file export (Shapefile / GeoJSON) or GIS API |
| Phase | Phase 0 |

### 10.4 Billing System

| Attribute | Specification |
|---|---|
| Direction | Bidirectional |
| Inbound data | Consumer registry (account number, name, premises, category), zone consumer counts |
| Outbound data | Outage incident records for billing adjustment |
| Target tables | CONSUMER, NETWORK_ZONE.total_consumers |
| Update frequency | Monthly for consumer counts; real-time for new consumer registration |
| Protocol | REST API |
| Phase | Phase 1 (consumer records), Phase 2 (outage data exchange) |

### 10.5 CRM System

| Attribute | Specification |
|---|---|
| Direction | Bidirectional |
| Outbound (FSM → CRM) | Notification events (complaint received, crew assigned, crew en route, resolved, planned outage advance, supply restored, etc.) |
| Inbound (CRM → FSM) | Delivery confirmation, read receipts, consumer satisfaction ratings |
| Target tables | CRM_NOTIFICATION (outbound), CONSUMER_COMMUNICATION_PREFERENCE (sync) |
| Integration mechanism | Asynchronous event bus (publish/subscribe) |
| Message format | JSON with standardized event schema |
| Phase | Phase 1 (reactive notifications), Phase 2 (proactive and regulatory) |

### 10.6 Regulatory Portal

| Attribute | Specification |
|---|---|
| Direction | Outbound from FSM |
| Data provided | Reliability snapshots (SAIFI/SAIDI/CAIFI/CAIDI), test certificates, PTW records |
| Source tables | RELIABILITY_SNAPSHOT, TEST_RECORD, ISOLATION_RECORD |
| Format | Regulatory authority prescribed format (CEA / state regulatory commission) |
| Frequency | Monthly for reliability indices; on-demand for certificates and PTW records |
| Phase | Phase 2 |

---

## 11. System Architecture — Engine and Interface Summary

### 11.1 Four Core Engines

| Engine | Primary function | Trigger | Phase |
|---|---|---|---|
| Schedule engine | Auto-generate tasks from MAINTENANCE_SCHEDULE | Nightly + 6-hourly sweep | Phase 1 |
| Matching engine | Surface ranked worker suggestions per task | New task created | Phase 1 |
| Clustering engine | Identify and manage task clubbing for SAIFI reduction | Task generation + planner review | Phase 2 |
| Supply optimizer | Select optimal alternative supply paths | Outage event approval + nightly contingency | Phase 2 |

### 11.2 Three Interfaces

| Interface | Primary users | Key functions |
|---|---|---|
| Dispatcher board (web) | Dispatcher | Task queue, map, ranked suggestions, SLA timers, isolation status, supply plan |
| Field worker mobile app | Field worker, Safety officer | Task receipt, service history, isolation workflow, test record capture, status update |
| Management dashboard (web) | Operations manager, Planning officer | SAIFI/SAIDI/CAIFI/CAIDI, SLA compliance, crew utilization, cluster planning, reliability trends |

---

## 12. Phased Implementation Plan

### Phase 0 — Foundation (Months 1–3)

**Objective:** Build the data infrastructure and establish baseline metrics.

**Deliverables:**
- All database schema created (all 40 tables, v1.4)
- GIS import: all assets geocoded and zoned
- Asset registry populated from existing records
- Skill catalog defined; workforce skill profiles loaded
- Task type catalog configured for all 15 task types
- Parts catalog created; van kit types defined
- Alternative supply catalog (ALT_SUPPLY_SOURCE) seeded by network planning team
- Maintenance schedules entered for all asset types and specific assets
- Baseline KPIs established from existing manual records
- Consumer registry imported from billing system

**Success criteria:**
- All assets have a geocoded LOCATION record
- All workers have a complete WORKER_SKILL profile
- All task types have TASK_SKILL_REQUIREMENT records
- Maintenance schedules cover all scheduled task types
- ALT_SUPPLY_SOURCE records cover all zones with at least one approved alternative

### Phase 1 — Assisted Dispatch (Months 3–6)

**Objective:** Activate schedule engine and matching engine; deliver dispatcher board and mobile app.

**Deliverables:**
- Schedule engine live — automated task generation from MAINTENANCE_SCHEDULE
- Matching engine live — skill-matched, proximity-ranked suggestions to dispatcher
- Dispatcher board deployed — task queue, map, suggestions, override capture
- Mobile app deployed — task receipt, status update, basic service history view
- Isolation workflow on mobile — PTW, pre-work and post-work confirmation
- Parts check at dispatch — van stock sufficiency flag
- CRM integration — reactive notifications (complaint received, crew assigned, resolved)
- Billing system integration — consumer records synchronized
- SLA_TRACKER alerts active

**Success criteria:**
- Skill match rate > 95%
- Suggestion acceptance rate > 70% (dispatchers use top suggestion)
- All PTW tasks have isolation records with both gate confirmations
- No task completed without post-work energization confirmation

### Phase 2 — Route Optimization and Planned Operations (Months 6–12)

**Objective:** Activate clustering engine and supply optimizer; deliver management dashboard; connect live data feeds.

**Deliverables:**
- Clustering engine live — opportunity identification, impact assessment, planned outage management
- Supply optimizer live — single-outage mode with margin check from manual SOURCE_OPERATING_STATE
- SCADA/EMS integration — real-time SOURCE_OPERATING_STATE updates
- Smart meter integration — last-gasp and restoration signals
- Management dashboard deployed — SAIFI/SAIDI/CAIFI/CAIDI, SLA, cluster planning
- Regulatory notifications live — advance notice for planned shutdowns
- Regulatory portal integration — monthly reliability index submission
- Van stock management — automatic decrement on task completion; reorder alerts

**Success criteria:**
- SAIFI reduction of ≥ 15% versus Phase 1 baseline (from clustering)
- Alternative supply plan exists for every planned outage before approval
- All planned shutdowns have advance consumer notifications sent ≥ 72 hours ahead
- Consumer_minutes_interrupted populated from smart meter timestamps (not manual estimate)

### Phase 3 — Intelligence and Prediction (Month 12 onwards)

**Objective:** Introduce ML-based prediction, multi-outage optimization, and proactive maintenance.

**Deliverables:**
- ML job duration prediction — trained from actual_duration_minutes and weather data
- ML weather-duration correlation — weather_duration_factors calibrated from historical records
- Multi-outage supply optimizer — constrained assignment for simultaneous outages
- N-1/N-2 contingency pre-computation — nightly flag of zones with no viable alternative
- Predictive maintenance — asset condition_score trending feeding proactive task generation
- Power systems tool integration hook — external load flow solver results accepted into SUPPLY_SWITCHING_PLAN
- Phase 3 supply optimizer — exact solver replacing greedy heuristic for complex scenarios

**Success criteria:**
- Duration prediction accuracy within ± 20% for 80% of tasks
- Zero approved outages without a viable supply plan (or documented override)
- Proactive maintenance tasks generated from condition_score trending reduce P1 emergency rate

---

## 13. KPIs and Success Metrics

### 13.1 Operational KPIs

| KPI | Formula | Target Phase 1 | Target Phase 2 | Target Phase 3 |
|---|---|---|---|---|
| Skill match rate | Tasks where worker held all mandatory skills ÷ total tasks | > 95% | > 99% | > 99.5% |
| SLA compliance — P1 | P1 tasks completed within SLA ÷ total P1 tasks | > 90% | > 95% | > 97% |
| SLA compliance — P2 | P2 tasks completed within SLA ÷ total P2 tasks | > 85% | > 92% | > 95% |
| SLA compliance — P3 scheduled | Scheduled tasks completed before compliance date | > 95% | > 98% | > 99% |
| Suggestion acceptance rate | Top suggestion used ÷ total assignments | > 65% | > 75% | > 80% |
| Average travel time per task | Total travel minutes ÷ total tasks | Baseline − 15% | Baseline − 30% | Baseline − 35% |
| Tasks completed per shift | Total tasks ÷ total shifts | Baseline + 15% | Baseline + 30% | Baseline + 40% |
| First-time fix rate | Tasks resolved in single visit ÷ total tasks | > 80% | > 87% | > 92% |

### 13.2 Reliability KPIs

| KPI | Formula | Measurement period |
|---|---|---|
| SAIFI | Sum(consumers_effectively_interrupted) ÷ total_consumers | Monthly, Quarterly, Annual per zone |
| SAIDI | Sum(consumer_minutes_interrupted) ÷ total_consumers | Monthly, Quarterly, Annual per zone |
| CAIFI | Sum(interruptions) ÷ consumers interrupted at least once | Monthly, Quarterly, Annual per zone |
| CAIDI | SAIDI ÷ SAIFI | Monthly, Quarterly, Annual per zone |
| SAIFI events saved by clustering | Sum(TASK_CLUSTER.saifi_events_saved) | Monthly per zone |
| Planned vs unplanned outage ratio | Planned outage events ÷ total outage events | Monthly per zone |

### 13.3 Safety and Compliance KPIs

| KPI | Formula | Target |
|---|---|---|
| PTW compliance rate | Tasks with isolation record and both gates confirmed ÷ tasks requiring PTW | 100% |
| Shutdown window compliance | Tasks completed within shutdown_window_end ÷ tasks with window | > 95% |
| Test certificate submission rate | TEST_RECORD with submission_status = ACCEPTED ÷ regulatory records | 100% |
| Certification expiry coverage | Workers with all active required certs ÷ total workers | 100% |

### 13.4 Consumer Experience KPIs

| KPI | Formula | Target |
|---|---|---|
| Advance notice compliance | Planned outages with ≥ 72 hr notice ÷ total planned outages | 100% |
| Proactive outage notification rate | Outage events with consumer notification before complaint ÷ total outages | > 80% |
| Complaint resolution notification rate | Resolved complaint tasks with completion notification sent ÷ resolved complaints | > 99% |
| Consumer satisfaction score | Average rating from CRM feedback | > 4.0 / 5.0 |

---

## 14. Assumptions and Constraints

### 14.1 Assumptions

- All field workers carry smartphones capable of running the mobile app
- SCADA/EMS system is available with a data API or compatible export interface
- Smart meter (AMI) infrastructure is deployed or being deployed in the service area
- GIS data for all network assets exists in a standard format (Shapefile or GeoJSON)
- Consumer billing system has an API or export capability for consumer records
- A CRM system exists or will be procured as a separate initiative — FSM publishes events to it
- Network planning team will validate and enter all ALT_SUPPLY_SOURCE records before Phase 2 go-live
- Regulatory advance notice requirement is 72 hours (confirm per applicable state regulation)

### 14.2 Constraints

- Alternative supply paths must be pre-approved by network planning — no ad hoc path selection
- Consumer personal data must be stored within India (data localization compliance)
- Regulatory records (PTW, test certificates, advance notices) must be retained for minimum 7 years
- The system shall not automate final dispatch — every assignment requires human dispatcher confirmation
- Budget and team availability constrain Phase 1 to 3 months — scope must not expand beyond defined deliverables

---

## 15. Glossary

| Term | Definition |
|---|---|
| SAIFI | System Average Interruption Frequency Index — average number of interruptions per consumer per period |
| SAIDI | System Average Interruption Duration Index — average minutes of interruption per consumer per period |
| CAIFI | Customer Average Interruption Frequency Index — average interruptions per interrupted customer |
| CAIDI | Customer Average Interruption Duration Index — average duration per interruption = SAIDI ÷ SAIFI |
| PTW | Permit to Work — formal written authorization to work on isolated electrical equipment |
| HV | High Voltage — typically 11kV and above in distribution context |
| MV | Medium Voltage — typically 440V to 11kV |
| LV | Low Voltage — typically 240V / 415V |
| AMI | Advanced Metering Infrastructure — smart meter system |
| MDMS | Meter Data Management System — system that collects and processes smart meter data |
| SCADA | Supervisory Control and Data Acquisition — real-time network monitoring and control |
| EMS | Energy Management System — load dispatch center software for network management |
| GIS | Geographic Information System — spatial data management for network assets |
| DER | Distributed Energy Resource — solar panels, battery storage, EV chargers at consumer premises |
| BESS | Battery Energy Storage System |
| SLA | Service Level Agreement — committed response or resolution time |
| FSM | Field Service Management |
| CRM | Customer Relationship Management |
| P1/P2/P3/P4 | Priority classes from Emergency (P1) to Routine (P4) |

---

*End of Requirements Analysis Document*
*Field Service Management System — Utility Field Response Optimization*
*Version 1.0 · April 2026*
