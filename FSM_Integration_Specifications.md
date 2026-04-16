# Integration Specifications
## Field Service Management System · Utility Field Response Optimization
**Document:** Integration Architecture
**Version:** 1.0 · April 2026

---

## 1. Purpose and Scope

This document defines the technical integration contracts between the FSM system and six external systems. For each integration it specifies the data exchanged, the protocol and format, the timing and frequency, the authentication and security model, the error handling and retry behaviour, and the operational monitoring approach.

These specifications are the engineering contract between the FSM team and the teams responsible for each external system. They are sufficient to allow both sides to build, test, and operate the integration independently.

---

## 2. Common Integration Principles

Before the individual specifications, several principles apply to every integration:

**Security baseline.** Every integration uses TLS 1.2 or higher. Every inbound connection authenticates with either mutual TLS certificates or OAuth 2.0 bearer tokens, with credentials rotated at least annually. No integration accepts unauthenticated requests. All authentication failures are logged with source IP and timestamp for security review.

**Idempotency.** Every write operation supports an idempotency key provided by the sender. Repeated delivery of the same message (due to network retry) produces the same outcome — not duplicate records. The FSM system deduplicates on idempotency key within a 24-hour window.

**Backpressure and rate limiting.** Each integration has a configured rate limit. When exceeded, the FSM returns HTTP 429 with a `Retry-After` header indicating seconds until retry is permitted. Senders must respect this header — repeated violation triggers a temporary block.

**Observability.** Every integration produces structured logs tagged with the integration name, message type, and correlation ID. Prometheus metrics expose message counts, latency percentiles, and error rates per integration. Critical integrations have dashboards visible to the operations team.

**Circuit breaker.** If an external system's error rate exceeds 50% over a 5-minute window, the FSM trips a circuit breaker on that integration. Subsequent messages are queued rather than attempted until the circuit half-opens (after 2 minutes) and a probe succeeds. This prevents a struggling external system from being overwhelmed by retry storms.

**Schema versioning.** Every payload includes a `schema_version` field. The FSM supports the current major version and one previous major version simultaneously, allowing external systems to upgrade on their schedule.

---

## 3. Integration 1 — SCADA / EMS (Load Dispatch Center)

### 3.1 Purpose

Feed real-time network state into the FSM's `SOURCE_OPERATING_STATE` table so the Supply Optimization Engine can select alternative supply paths based on current loading, voltage, and available margin. Also update `ASSET.energization_status` when switching operations are executed on the network.

### 3.2 Data Flow

**Direction:** Inbound to FSM (SCADA/EMS → FSM)

**Data provided:**
- Per source asset: current load in MVA, rated capacity, loading percentage, available margin, voltage in per unit, frequency, power factor, thermal status
- Per switching device: operating state (CB OPEN/CLOSED, isolator position, earth status)
- Per affected asset when state changes: derived energization status

**Target tables:**
- `SOURCE_OPERATING_STATE` — inserted as new record per reading (append-only for history)
- `ASSET.energization_status`, `energization_updated_at`, `energization_updated_by` — updated when switching changes confirmed

### 3.3 Protocol and Format

**Primary protocol:** REST API over HTTPS. Per-utility preference, OPC-UA or IEC 60870-5-104 may be supported via a protocol adapter.

**Endpoint:** `POST /api/v1/scada/source-state`

**Request payload (JSON):**

```json
{
  "schema_version": "1.0",
  "idempotency_key": "scada-reading-20260416T142300Z-SRC-A1B2",
  "source_system": "PVVNL_EMS_NORTH",
  "recorded_at": "2026-04-16T14:23:00Z",
  "readings": [
    {
      "asset_code": "XFMR-RAMDAS-T1",
      "current_load_mva": 15.2,
      "rated_capacity_mva": 25.0,
      "loading_percentage": 60.8,
      "available_margin_mva": 9.8,
      "voltage_pu": 1.012,
      "voltage_within_limits": true,
      "frequency_hz": 50.02,
      "power_factor": 0.89,
      "thermal_status": "NORMAL",
      "outage_planned": false
    },
    {
      "asset_code": "CB-104-BAY-3",
      "switching_state": "CLOSED",
      "energization_status_derived": "ENERGIZED"
    }
  ]
}
```

**Response (success):**
```json
{
  "status": "accepted",
  "records_processed": 2,
  "records_ingested": 2,
  "records_rejected": 0,
  "processing_latency_ms": 34
}
```

**Response (partial failure):**
```json
{
  "status": "partial",
  "records_processed": 2,
  "records_ingested": 1,
  "records_rejected": 1,
  "rejections": [
    {
      "asset_code": "CB-999-UNKNOWN",
      "reason": "ASSET_NOT_FOUND",
      "message": "Asset code not registered in FSM. Check asset master sync."
    }
  ]
}
```

### 3.4 Frequency and Volume

| Scenario | Frequency | Volume per message |
|---|---|---|
| Normal operation — polling | Every 15 minutes | All monitored source assets (typical: 200–500 records) |
| Planning period — before scheduled outage | Every 5 minutes | Affected zone source assets only |
| Active outage management | Every 1–2 minutes | Affected zone source assets only |
| Switching operation event | Real-time push | Single asset state change |

During a major incident, the integration can accept up to 10,000 readings per minute. Normal steady state is approximately 50 readings per minute.

### 3.5 Error Handling

**Staleness protection.** Each reading's `recorded_at` timestamp is compared with `now()`. If the gap exceeds 5 minutes, the reading is flagged as stale on ingestion. The Supply Optimization Engine respects the staleness thresholds defined in its specification (30-minute SCADA_STALE, 240-minute MANUAL_REQUIRED).

**Asset not found.** If an `asset_code` in the payload does not exist in the FSM `ASSET` table, the reading for that asset is rejected with `ASSET_NOT_FOUND`. Other readings in the batch are still processed. An alert is raised to IT operations to sync asset masters.

**Connection loss.** If SCADA/EMS cannot reach the FSM endpoint, readings must be buffered locally and retried. The FSM accepts readings with `recorded_at` up to 24 hours in the past — these are ingested but may be flagged stale for optimization use.

**Circuit breaker.** If SCADA/EMS sends malformed payloads or invalid data at a sustained rate, the FSM trips the circuit breaker and returns 503 Service Unavailable. SCADA/EMS must buffer and retry.

### 3.6 Security

- Mutual TLS authentication with certificates issued by utility PKI
- API accessible only from SCADA/EMS source IP range (firewall-enforced)
- Payload size limit: 5 MB (batches larger than this should be split)
- Rate limit: 1,000 requests per minute per source system

### 3.7 Monitoring

Key metrics exposed via Prometheus:
- `scada_readings_ingested_total` — counter by source_system and asset_code
- `scada_readings_rejected_total` — counter by rejection reason
- `scada_ingestion_latency_seconds` — histogram (target: p95 < 100ms)
- `scada_staleness_seconds` — gauge showing age of most recent reading per source
- `scada_circuit_breaker_state` — gauge (0 = closed, 1 = half-open, 2 = open)

Alerts configured for:
- No readings received from any zone for > 10 minutes during business hours
- Circuit breaker open for > 5 minutes
- Rejection rate > 10% over any 5-minute window

---

## 4. Integration 2 — Smart Meter / MDMS (Head-End System)

### 4.1 Purpose

Provide three distinct capabilities: automatic outage detection via last-gasp signals, automatic supply restoration confirmation via power-restored signals, and consumer-level load profile data for supply optimization. Also generate reactive tasks from tamper and power quality alerts.

### 4.2 Data Flows

This integration has four distinct message types flowing MDMS → FSM:

**Message type A — Last-gasp outage signal** (event-driven, high priority)
**Message type B — Power-restored signal** (event-driven)
**Message type C — Interval consumption data** (15-minute batch)
**Message type D — Meter alerts** (event-driven — tamper, voltage sag, frequency deviation)

### 4.3 Message Type A — Last-Gasp Signal

**Purpose:** Automatic outage detection before consumer calls.

**Protocol:** MQTT publish-subscribe (preferred) or REST webhook

**MQTT topic:** `fsm/ingestion/meter-alerts/last-gasp`

**Payload:**
```json
{
  "schema_version": "1.0",
  "alert_id": "MDMS-LG-2026-04-16-142301-00042",
  "meter_id": "MR-ACC-5521834",
  "received_at": "2026-04-16T14:23:01Z",
  "signal_timestamp": "2026-04-16T14:23:00Z",
  "signal_strength": 0.87,
  "last_voltage_reading_v": 234.1,
  "last_voltage_reading_at": "2026-04-16T14:22:59Z"
}
```

**FSM processing:**
```
1. Insert METER_ALERT record (alert_type = LAST_GASP, processed = false)
2. Resolve consumer_id from CONSUMER.meter_id = meter_id
3. Check for existing OUTAGE_EVENT covering this consumer:
   a. If exists → add CONSUMER_OUTAGE_RECORD with supply_lost_at = signal_timestamp
      Update CONSUMER_OUTAGE_RECORD.last_gasp_received_at
   b. If not exists → queue for outage clustering
      (wait 60 seconds for correlated signals, then create OUTAGE_EVENT if ≥ 5 consumers)
4. If this is the 5th+ last-gasp in the same feeder within 60 seconds:
   → Create TASK of type OUTAGE_DETECTED (P1)
   → Create OUTAGE_EVENT (outage_type = UNPLANNED)
   → Trigger matching engine
5. Set METER_ALERT.processed = true when task/outage created
```

### 4.4 Message Type B — Power-Restored Signal

**Purpose:** Confirm supply restoration from meter directly.

**Protocol:** Same as last-gasp (MQTT or REST webhook)

**MQTT topic:** `fsm/ingestion/meter-alerts/restored`

**Payload:**
```json
{
  "schema_version": "1.0",
  "alert_id": "MDMS-PR-2026-04-16-170501-00042",
  "meter_id": "MR-ACC-5521834",
  "received_at": "2026-04-16T17:05:01Z",
  "signal_timestamp": "2026-04-16T17:05:00Z",
  "voltage_reading_v": 236.8
}
```

**FSM processing:**
```
1. Insert METER_ALERT record (alert_type = POWER_RESTORED)
2. Find CONSUMER_OUTAGE_RECORD for this consumer with supply_fully_restored_at IS NULL
3. Set supply_fully_restored_at = signal_timestamp
   Set restoration_signal_received_at = received_at
4. Recompute effective_interruption_minutes
5. If all CONSUMER_OUTAGE_RECORD in parent OUTAGE_EVENT now have supply_fully_restored_at:
   → Consider OUTAGE_EVENT as confirmed restored
   → (The actual outage_end is set by operational flow — this is confirmation only)
```

### 4.5 Message Type C — Interval Consumption Data

**Purpose:** Consumer-level load data for supply optimization load estimation.

**Protocol:** REST batch upload (MDMS pushes every 15 minutes)

**Endpoint:** `POST /api/v1/mdms/interval-data`

**Payload:**
```json
{
  "schema_version": "1.0",
  "idempotency_key": "mdms-interval-20260416T1415-batch-0142",
  "interval_start": "2026-04-16T14:15:00Z",
  "interval_end": "2026-04-16T14:30:00Z",
  "readings": [
    {
      "meter_id": "MR-ACC-5521834",
      "consumption_kwh": 0.42,
      "avg_voltage_v": 235.1,
      "avg_power_factor": 0.91,
      "peak_load_kw": 1.8
    }
  ]
}
```

This data is stored in an aggregate analytics table (not in the core FSM schema) and queried by the Supply Optimization Engine during load-to-transfer estimation. Storage is 18 months rolling window per consumer.

### 4.6 Message Type D — Meter Alerts

**Alert types:** TAMPER, VOLTAGE_SAG, FREQUENCY_DEVIATION

**Protocol:** MQTT or REST webhook

**Payload:**
```json
{
  "schema_version": "1.0",
  "alert_id": "MDMS-AL-2026-04-16-163022-00089",
  "meter_id": "MR-ACC-5521834",
  "alert_type": "TAMPER",
  "severity": "HIGH",
  "signal_timestamp": "2026-04-16T16:30:22Z",
  "details": {
    "tamper_type": "COVER_OPEN",
    "duration_seconds": 120
  }
}
```

**FSM processing:**
```
1. Insert METER_ALERT record with alert_type
2. Determine appropriate task type:
   TAMPER → METER_SERVICES (P2)
   VOLTAGE_SAG → POWER_QUALITY_INVESTIGATION (P2)
   FREQUENCY_DEVIATION → POWER_QUALITY_INVESTIGATION (P2)
3. Create TASK with:
   task_type_id = corresponding type
   consumer_id = resolved from meter_id
   location_id = CONSUMER.location_id
   description = "Auto-created from MDMS alert: [alert_type] - [details]"
   triggered_by = EVENT
4. Set METER_ALERT.processed = true, METER_ALERT.task_id = new task ID
```

### 4.7 Error Handling

**Unknown meter_id.** If meter_id does not match any CONSUMER.meter_id, the message is stored in METER_ALERT with consumer_id NULL. A resolution job runs hourly attempting to match — if a new consumer is added to the system, previously unresolved alerts are linked. After 7 days of non-resolution, the alert is archived and an operations alert raised.

**Duplicate signals.** Smart meters sometimes send repeated signals due to network issues. Deduplication is performed on `alert_id` within a 24-hour window.

**Out-of-order signals.** Restoration signals arriving before their corresponding last-gasp signal (due to network reordering) are held in a reconciliation queue for up to 5 minutes before processing.

### 4.8 Security and Monitoring

- MQTT: mutual TLS on port 8883, client certificates per MDMS environment
- REST: OAuth 2.0 bearer token, token rotated every 24 hours
- Rate limit: 100,000 messages per hour per MDMS environment (sufficient for 500,000 meter fleet with 15-minute intervals)
- Key metrics: `mdms_last_gasp_received_total`, `mdms_restored_received_total`, `mdms_alerts_processed_total`, `mdms_meter_id_resolution_rate`

---

## 5. Integration 3 — GIS (Geographic Information System)

### 5.1 Purpose

Populate the FSM with geocoded asset locations, network zone polygon boundaries, feeder routing topology, and alternative supply path network data. This is primarily a one-time foundational import at Phase 0, with periodic re-syncs when the network changes.

### 5.2 Data Provided

- **Asset geocodes** — latitude and longitude for every substation, transformer, line pole, meter point
- **Zone boundaries** — polygon definitions for each NETWORK_ZONE
- **Feeder routing** — line connectivity showing which assets feed which zones
- **Substation topology** — bay layouts, switching diagrams

### 5.3 Protocol and Format

**Method:** Bulk file import (Phase 0 and major updates) + Web Feature Service (WFS) for incremental updates

**File formats supported:**
- ESRI Shapefile (.shp with companion .dbf, .shx, .prj)
- GeoJSON (.geojson)
- GML 3.2 (from WFS)

**File structure for asset import:**

Each asset record must contain:
```
asset_code         VARCHAR(30)   — matches FSM ASSET.asset_code
asset_type_code    VARCHAR(30)   — e.g. XFMR_33KV, HV_LINE
name               VARCHAR(150)
lat                DECIMAL(9,6)
lng                DECIMAL(9,6)
zone_code          VARCHAR(20)   — matches FSM NETWORK_ZONE.code
installation_date  DATE
manufacturer       VARCHAR(100)
model              VARCHAR(100)
serial_number      VARCHAR(100)
voltage_level      HV · MV · LV
parent_asset_code  VARCHAR(30)   — optional, for sub-components
```

**File structure for zone import:**

```
zone_code          VARCHAR(20)   — unique
name               VARCHAR(100)
voltage_level      HV · MV · LV · MIXED
responsible_team   VARCHAR(50)
boundary           POLYGON geometry (WGS84)
alt_supply_zone_code  VARCHAR(20) — optional FK to another zone
```

### 5.4 Import Process

```
Phase 0 initial import:
1. GIS team prepares Shapefile/GeoJSON batches per asset type
2. Import utility validates each file:
   - All required fields present
   - Coordinate system is WGS84 (EPSG:4326)
   - Asset codes are unique and follow naming convention
   - Zone codes referenced exist
3. Dry-run import shows counts and warnings
4. Commit import populates LOCATION, ASSET, NETWORK_ZONE tables
5. Validation report generated — counts, orphans, duplicates, warnings

Incremental updates via WFS:
1. GIS publishes WFS endpoint with update timestamps
2. FSM polls nightly at 02:00
3. Retrieves all features modified since last successful sync
4. Applies updates with full audit trail
5. Any delete requires manual review (assets cannot be auto-deleted)
```

### 5.5 Error Handling

**Duplicate asset codes.** Rejected. Import log flags the duplicate; GIS team must resolve before retry.

**Orphan references.** An asset with `parent_asset_code` pointing to a non-existent asset is imported with warning but parent_asset_id left NULL. Resolution job runs after all imports complete to link orphans.

**Invalid geometry.** Polygons with self-intersecting edges or invalid closure are rejected per feature. Other features in the same file still import.

**Zone boundary changes.** When a zone polygon is resized and this changes which consumers fall inside, the resolution requires manual confirmation — the system does not silently re-assign consumers to different zones (which would affect SAIFI/SAIDI historical baselines).

### 5.6 Security and Monitoring

- Bulk imports: SFTP with key-based authentication; files placed in timestamped folder
- WFS: HTTPS with basic auth; source IP whitelist
- All imports produce a signed audit record including importing user, file checksums, counts, and outcome

---

## 6. Integration 4 — Billing System

### 6.1 Purpose

Maintain the consumer registry (`CONSUMER` table) and zone consumer counts (`NETWORK_ZONE.total_consumers`), which form the denominator for SAIFI and SAIDI calculations. Also receive outage event records for billing adjustment where regulatory rebates apply.

### 6.2 Data Flows

**Inbound (Billing → FSM):** Consumer registry, zone consumer counts
**Outbound (FSM → Billing):** Outage incident records for rebate computation

### 6.3 Protocol and Format

**Protocol:** REST API over HTTPS in both directions

**Inbound — Consumer sync:**

`POST /api/v1/billing/consumer-sync`

**Full sync (monthly):**
```json
{
  "schema_version": "1.0",
  "sync_type": "FULL",
  "sync_date": "2026-04-01",
  "division_code": "PVVNL_GZB",
  "consumers": [
    {
      "account_number": "UP14-GZB-00421833",
      "name": "Aggarwal Enterprises",
      "address_line_1": "Plot 14, Sector 63",
      "address_line_2": "Industrial Area",
      "city": "Ghaziabad",
      "postal_code": "201301",
      "lat": 28.6139,
      "lng": 77.2090,
      "zone_code": "URBAN-MV-2",
      "consumer_category": "INDUSTRIAL",
      "contact_phone": "+919876543210",
      "contact_email": "ops@aggarwal-ent.in",
      "meter_id": "MR-ACC-5521834",
      "crm_account_id": "CRM-UP14-00421833",
      "billing_status": "ACTIVE",
      "connection_date": "2022-06-15"
    }
  ]
}
```

**Incremental sync (new connections, status changes):**
```json
{
  "schema_version": "1.0",
  "sync_type": "INCREMENTAL",
  "changes_since": "2026-04-15T00:00:00Z",
  "consumers": [
    {
      "change_type": "ADDED",
      "account_number": "UP14-GZB-00422017",
      ...
    },
    {
      "change_type": "STATUS_CHANGED",
      "account_number": "UP14-GZB-00421901",
      "billing_status": "SUSPENDED"
    },
    {
      "change_type": "MODIFIED",
      "account_number": "UP14-GZB-00421833",
      "field_changes": {
        "contact_phone": "+919876543211"
      }
    }
  ]
}
```

**Outbound — Outage record export:**

`POST /api/v1/billing/outage-export` (FSM → Billing)

Sent monthly covering all OUTAGE_EVENT records where restoration_method was completed during the period:

```json
{
  "schema_version": "1.0",
  "period_start": "2026-04-01",
  "period_end": "2026-04-30",
  "outages": [
    {
      "outage_reference": "OUT-2026-04-031",
      "outage_type": "UNPLANNED",
      "zone_code": "NORTH-HV-3",
      "outage_start": "2026-04-16T14:05:00Z",
      "outage_end": "2026-04-16T17:10:00Z",
      "duration_minutes": 185,
      "consumers_effectively_interrupted": 847,
      "consumer_outage_records": [
        {
          "account_number": "UP14-GZB-00421833",
          "supply_lost_at": "2026-04-16T14:05:32Z",
          "supply_fully_restored_at": "2026-04-16T17:08:15Z",
          "effective_interruption_minutes": 183
        }
      ]
    }
  ]
}
```

### 6.4 Frequency

| Operation | Frequency |
|---|---|
| Full consumer sync | Monthly (first Sunday of each month, 02:00) |
| Incremental consumer sync | Daily at 23:00 (covers that day's changes) |
| Real-time new connection | Per-event within 5 minutes |
| Outage export | Monthly (first working day of each month) |

### 6.5 Error Handling

**Consumer in FSM but not in billing.** This indicates a consumer was removed from billing but still appears in CONSUMER_OUTAGE_RECORD. After 90 days of inactivity, soft-delete the consumer (billing_status = ARCHIVED) but retain CONSUMER_OUTAGE_RECORD references for historical reliability computation.

**Billing account number reused.** Rare but possible. Detected by `connection_date` changing on existing account_number. Creates a new CONSUMER record, archives the old one.

**Zone_code mismatch.** If billing specifies a zone_code not in FSM NETWORK_ZONE, the consumer is imported with zone_id = NULL and an alert raised. Resolution job attempts to match nightly.

### 6.6 Security and Monitoring

- OAuth 2.0 client credentials flow between FSM and billing
- PII-containing payloads encrypted at the application layer (even over TLS)
- Consumer personal data access audited — every read logged with requesting service, consumer_id, and timestamp
- Compliance with data localisation (all consumer PII stored within India)

---

## 7. Integration 5 — CRM System

### 7.1 Purpose

Dispatch consumer notifications at every significant operational event (complaint received, crew en route, outage detected, planned shutdown advance notice, etc.). The FSM publishes notification events; the CRM owns channel delivery, message templates, consumer opt-out, and delivery tracking.

This is the most architecturally significant integration — it uses asynchronous event-driven messaging rather than request-response REST, because notification events are fire-and-forget from the FSM's perspective.

### 7.2 Architecture

**Pattern:** Publish/subscribe event bus. FSM publishes events; CRM subscribes and acts on them.

**Event bus technology:** Apache Kafka (preferred) or AMQP (RabbitMQ) — per utility technology stack

**Topic:** `fsm.notifications.consumer`

### 7.3 Event Payload Format

Each FSM-generated notification event has this envelope:

```json
{
  "schema_version": "1.0",
  "event_id": "FSM-NOTIF-2026-04-16-142305-00001",
  "event_type": "COMPLAINT_RECEIVED",
  "event_time": "2026-04-16T14:23:05Z",
  "consumer_id": "c8a9d7f2-4ed1-4a31-b6fe-9c2f8d4a7823",
  "crm_account_id": "CRM-UP14-00421833",
  "communication_type": "REACTIVE",
  "triggered_by_engine": "DISPATCHER",
  "regulatory_record": false,
  "related_entity": {
    "task_id": "WO-2026-04-0091",
    "outage_event_id": null,
    "cluster_id": null
  },
  "message_template_id": "COMPLAINT_RECEIVED_EN_V2",
  "preferred_language": "en",
  "message_variables": {
    "reference": "WO-2026-04-0091",
    "task_description": "HT cable fault · Sector 14",
    "sla_deadline": "2026-04-16T15:47:00Z",
    "sla_minutes_remaining": 84,
    "utility_helpline": "+919876543200"
  },
  "correlation_id": "abc-123-correlation-id"
}
```

### 7.4 Event Types Published by the FSM

| Event type | Trigger | Engine |
|---|---|---|
| COMPLAINT_RECEIVED | Complaint task created | Dispatcher / Matching |
| CREW_ASSIGNED | CREW_ASSIGNMENT created | Matching Engine |
| CREW_EN_ROUTE | CREW_ASSIGNMENT.dispatched_at set | Matching Engine |
| CREW_ARRIVED | CREW_ASSIGNMENT.arrived_at set | Mobile app |
| RESOLVED | TASK.status = COMPLETED | Mobile app |
| DEFERRED | TASK.status = DEFERRED | Dispatcher / Mobile app |
| OUTAGE_DETECTED | OUTAGE_EVENT created | Smart Meter / Clustering |
| ALT_SUPPLY | SUPPLY_SWITCHING_PLAN executed | Supply Optimizer |
| SUPPLY_RESTORED | OUTAGE_EVENT.outage_end set | Clustering Engine |
| PLANNED_OUTAGE_ADVANCE | TASK_CLUSTER.status = APPROVED | Clustering Engine |
| PLANNED_REMINDER | 24 hrs before planned window | Scheduler |
| PLAN_REVISED | TASK_CLUSTER window changed | Clustering Engine |
| PLAN_CANCELLED | TASK_CLUSTER.status = CANCELLED | Clustering Engine |
| EARLY_RESTORATION | Planned work complete before window end | Clustering Engine |
| SLA_AT_RISK | SLA_TRACKER.alert_1d_triggered | Schedule Engine |

### 7.5 CRM Delivery Confirmation Flow

After CRM processes and dispatches a notification, it publishes a confirmation event back to the FSM on topic `crm.delivery.confirmations`:

```json
{
  "schema_version": "1.0",
  "event_id": "CRM-DELIVER-2026-04-16-142310-00001",
  "fsm_event_id": "FSM-NOTIF-2026-04-16-142305-00001",
  "consumer_id": "c8a9d7f2-4ed1-4a31-b6fe-9c2f8d4a7823",
  "channel_used": "SMS",
  "delivery_status": "DELIVERED",
  "sent_at": "2026-04-16T14:23:07Z",
  "delivered_at": "2026-04-16T14:23:09Z",
  "read_at": null,
  "failure_reason": null,
  "crm_reference": "CRM-MSG-9283745"
}
```

FSM uses this confirmation to update `CRM_NOTIFICATION.delivery_status`, `delivered_at`, `read_at`, and `crm_reference`.

### 7.6 CRM Opt-Out and Consent Enforcement

The CRM is the single source of truth for consumer communication preferences. It enforces opt-out:

- Non-regulatory events: CRM checks `CONSUMER_COMMUNICATION_PREFERENCE` before sending. Opted-out consumers receive no message.
- Regulatory events (advance notices): CRM must send regardless of non-emergency opt-out, because advance notice is legally mandated. If a consumer has explicitly opted out of all communication, this is escalated to customer relations for individual handling.

When a consumer opts out via CRM, the CRM publishes a sync event to the FSM:

```json
{
  "event_type": "PREFERENCE_UPDATED",
  "consumer_id": "...",
  "crm_account_id": "...",
  "preferences": {
    "planned_outage_consent": false,
    "complaint_updates_consent": true,
    "marketing_consent": false,
    "opted_out_at": "2026-04-16T10:22:00Z"
  }
}
```

The FSM updates `CONSUMER_COMMUNICATION_PREFERENCE` accordingly.

### 7.7 Event Ordering and Reliability

**Ordering.** Events for the same consumer must be delivered in order. Kafka partition key = `consumer_id` ensures all events for one consumer go to the same partition, preserving order.

**At-least-once delivery.** FSM publishes with acks=all. CRM acknowledges after processing. Network failures result in redelivery — CRM handles idempotency via `event_id`.

**Dead letter queue.** Events that fail processing 5 times are sent to a dead letter topic `fsm.notifications.consumer.dlq` for manual review.

### 7.8 Regulatory Record Preservation

Events with `regulatory_record: true` (planned outage advance notices, regulatory notifications) are held permanently in the FSM `CRM_NOTIFICATION` table. The CRM is expected to also preserve these for its own audit purposes. A consistency check runs quarterly comparing FSM regulatory records against CRM delivery logs.

### 7.9 Security and Monitoring

- Kafka/AMQP: mutual TLS, SASL authentication per service account
- Consumer PII in message_variables encrypted at application layer
- Metrics: `fsm_notification_published_total`, `crm_delivery_confirmed_total`, `crm_delivery_failed_total`, `notification_end_to_end_latency_seconds`
- Alerts: delivery confirmation rate < 95% over any 1-hour window

---

## 8. Integration 6 — Regulatory Portal

### 8.1 Purpose

Submit mandatory regulatory reports to the Central Electricity Authority (CEA), state regulatory commissions (e.g. UPERC for Uttar Pradesh), and other bodies as required. Reports include monthly reliability indices, annual test certificates, permit-to-work records, and advance notice compliance evidence.

### 8.2 Report Types

| Report type | Frequency | Destination | Format |
|---|---|---|---|
| Reliability snapshot (SAIFI/SAIDI/CAIFI/CAIDI) | Monthly | CEA + State commission | XML (regulatory schema) |
| Test certificates | On-demand / annual batch | State commission | PDF + XML metadata |
| Permit-to-work records | Quarterly batch | State commission + inspector | XML |
| Advance notice compliance | Monthly | State commission | PDF report + CSV evidence |
| Outage incident reports (major) | Per incident | State commission + media | PDF |
| Annual performance summary | Annual | CEA + State commission | Structured document |

### 8.3 Protocol and Format

**Most regulatory portals use:** SFTP deposit with signed zip files, plus REST confirmation endpoint for status polling.

**A smaller number of modern portals use:** Direct REST API submission.

The FSM supports both via a pluggable submission adapter.

**SFTP submission pattern:**

```
1. FSM generates report file (XML or PDF)
2. Digitally signs with utility's registration certificate
3. Places in timestamped folder: /pvvnl-gzb/reliability/202604/
4. Filename convention: <DIVISION>_<REPORT_TYPE>_<PERIOD>_<SEQ>.xml.signed
   Example: PVVNL_GZB_SAIFI_202604_001.xml.signed
5. Places checksum file alongside: .sha256
6. Uploads confirmation manifest: manifest.json listing all files
```

**REST submission pattern:**

```
POST /api/v1/regulatory/submit
Authorization: Bearer <utility-issued-token>

Multipart payload:
  - report_metadata.json (type, period, division, prepared_by)
  - report.xml (the data)
  - supporting_evidence.zip (optional)
  - signature.p7s (detached signature)

Response:
  { "submission_id": "REG-2026-04-0042",
    "status": "ACCEPTED",
    "ack_at": "2026-04-16T10:30:00Z" }
```

### 8.4 Reliability Snapshot XML Schema

Conforms to CEA/state schema (example — actual schema per regulation):

```xml
<ReliabilityReport xmlns="http://cea.gov.in/schema/reliability/v2">
  <ReportHeader>
    <UtilityCode>PVVNL</UtilityCode>
    <DivisionCode>GZB</DivisionCode>
    <PeriodStart>2026-04-01</PeriodStart>
    <PeriodEnd>2026-04-30</PeriodEnd>
    <PreparedBy>operations.manager@pvvnl.in</PreparedBy>
    <PreparedAt>2026-05-02T10:00:00Z</PreparedAt>
    <Schema Version>2.0</SchemaVersion>
  </ReportHeader>
  <Zones>
    <Zone>
      <ZoneCode>NORTH-HV-3</ZoneCode>
      <VoltageLevel>HV</VoltageLevel>
      <TotalConsumers>847</TotalConsumers>
      <OutageEvents>
        <Total>3</Total>
        <Planned>1</Planned>
        <Unplanned>2</Unplanned>
      </OutageEvents>
      <ReliabilityIndices>
        <SAIFI>0.28</SAIFI>
        <SAIDI>12.4</SAIDI>
        <CAIFI>1.08</CAIFI>
        <CAIDI>44.3</CAIDI>
      </ReliabilityIndices>
      <ClusteringImpact>
        <TasksClubbed>5</TasksClubbed>
        <SaifiEventsSavedByClubbing>2</SaifiEventsSavedByClubbing>
      </ClusteringImpact>
    </Zone>
    ...
  </Zones>
  <DivisionSummary>
    <TotalConsumers>148523</TotalConsumers>
    <WeightedAverageSAIFI>0.42</WeightedAverageSAIFI>
    <WeightedAverageSAIDI>38.7</WeightedAverageSAIDI>
  </DivisionSummary>
  <Signature>...</Signature>
</ReliabilityReport>
```

### 8.5 Submission Lifecycle

```
Day 1 of month:
  01:00 — Reliability engine runs, computes RELIABILITY_SNAPSHOT for previous month
  02:00 — Regulatory report generator assembles XML from RELIABILITY_SNAPSHOT
  03:00 — Operations manager receives draft for review
  Working hours — Operations manager reviews, corrects errata if any, approves

Submission:
  On approval, submission service:
  1. Signs the XML with digital signature
  2. Computes SHA-256 checksum
  3. Uploads via SFTP or REST
  4. Records submission attempt in TEST_RECORD.submission_status = PENDING
  5. Polls regulatory portal for acknowledgment

  On acknowledgment:
  - submission_status = SUBMITTED
  - submitted_at populated
  - Acknowledgment receipt stored

  On rejection:
  - submission_status = REJECTED
  - Rejection reason recorded
  - Alert to operations manager
  - Retry after correction
```

### 8.6 Advance Notice Compliance Report

Monthly evidence that all planned outages had regulatory advance notices sent ≥ 72 hours ahead:

```
Report structure:
  For each OUTAGE_EVENT where outage_type = PLANNED in period:
    - Outage reference
    - Planned window start
    - First advance notice sent (earliest CRM_NOTIFICATION.sent_at
      where notification_type = PLANNED_OUTAGE_ADVANCE)
    - Hours ahead (computed)
    - Compliance status (COMPLIANT if ≥ 72 hrs, NON_COMPLIANT otherwise)
    - Count of consumers notified
    - Channel distribution (SMS: X, Email: Y, etc.)

  Summary:
    - Total planned outages
    - Compliance rate (%)
    - Any non-compliant events (should be zero)
```

This report is auto-generated from CRM_NOTIFICATION records where `regulatory_record = true`.

### 8.7 Error Handling

**Submission failure.** If the regulatory portal is unavailable, submission is retried with exponential backoff up to 5 times over 24 hours. After that, operational escalation to the operations manager.

**Rejection.** Rejection reasons vary — data quality issues, schema version mismatches, authentication failures. The submission service parses the rejection reason and categorizes:
- Data quality: operations manager alerted for correction
- Schema mismatch: IT alerted for integration update
- Authentication: IT Security alerted for certificate renewal

**Late submission.** If the monthly deadline is missed (typically 5th working day of following month), an incident is raised. Regulatory fines apply — the system cannot resolve this automatically.

### 8.8 Security

- Digital signatures use utility's PKI certificate issued by approved CA
- SFTP: key-based authentication, certificate rotated annually
- REST: OAuth 2.0 with regulatory portal's identity provider
- All submission artifacts retained for 7 years minimum

---

## 9. Cross-Integration Operational Runbook

### 9.1 Integration Health Dashboard

A single operational dashboard (separate from the management dashboard) shows real-time health of all six integrations:

| Integration | Status indicator | Key metric |
|---|---|---|
| SCADA/EMS | Last reading received | Age of most recent reading per source |
| Smart Meter MDMS | Message rate | Events per minute (normal vs current) |
| GIS | Last sync timestamp | Days since last sync |
| Billing | Last sync status | Current sync status (IDLE/SYNCING/FAILED) |
| CRM | Event publish rate + confirmation rate | % confirmed within 5 minutes |
| Regulatory | Pending submissions | Count + next deadline |

### 9.2 Incident Response

Each integration has a named escalation path:

**SCADA/EMS failure:** Impacts Supply Optimization Engine (must use stale data thresholds).
- Minor (single source stale): Operations manager notification
- Major (entire SCADA feed down): Operations manager + Network Operations Centre + IT Infrastructure

**Smart Meter MDMS failure:** Impacts automatic outage detection (consumers must call in).
- Detection revert to manual: Dispatcher team alerted
- Persistent failure: IT + MDMS vendor support

**CRM failure:** Impacts consumer notifications (complaints, advance notices).
- Non-regulatory messages queue and retry
- Regulatory message failure: Compliance team + IT (72-hour deadline at risk)

**Regulatory portal failure:** Does not block day-to-day operations but affects monthly submission.
- Submission failure: Retried automatically
- Persistent failure: Compliance team + regulatory body contact

### 9.3 Configuration Management

All integration endpoints, credentials, and rate limits are stored in a secure configuration service. Changes require dual approval (one engineer + one operations manager). Every change is logged with timestamp and approver.

---

## 10. Integration Build Sequence

Phase-by-phase integration delivery (aligned with the overall FSM implementation plan):

| Phase | Integration | Scope |
|---|---|---|
| Phase 0 | GIS (initial import) | Asset geocodes, zone boundaries, initial ALT_SUPPLY_SOURCE |
| Phase 1 | Billing (consumer sync) | Populate CONSUMER registry for matching engine + CRM |
| Phase 1 | CRM (reactive notifications) | Complaint lifecycle — received, assigned, en route, resolved |
| Phase 2 | SCADA/EMS | Real-time SOURCE_OPERATING_STATE for Supply Optimization Engine |
| Phase 2 | Smart Meter MDMS (event signals) | Last-gasp and restoration signals for automatic outage detection |
| Phase 2 | CRM (proactive + regulatory) | Outage events, planned shutdown advance notices, overrun alerts |
| Phase 2 | Regulatory portal | Monthly reliability snapshot submission |
| Phase 3 | Smart Meter MDMS (interval data) | Consumer-level load data for supply optimization |
| Phase 3 | GIS (WFS incremental) | Automated nightly sync of network changes |

---

## 11. Key Integration Design Decisions

**Why are SCADA/EMS and MDMS treated as separate integrations rather than unified "grid data"?**
They are operated by different teams, serve different purposes, use different protocols, and update at different frequencies. SCADA/EMS is the load dispatch centre's real-time grid monitoring; MDMS is the consumer-level metering data management. Conflating them would create artificial dependencies and confuse responsibility for uptime.

**Why does the CRM use asynchronous event bus rather than REST calls?**
Notification events are fire-and-forget — the FSM does not need to wait for CRM delivery confirmation before proceeding with operations. If the CRM is slow or temporarily down, FSM operations should not block. Event bus decouples the two systems. The confirmation comes back asynchronously when available.

**Why is the GIS integration primarily one-time import rather than real-time?**
Network topology changes are rare events — new substations, line extensions, zone redefinitions happen on project timescales (weeks to months), not real-time. Real-time sync would create unnecessary complexity for a change frequency of 1–10 events per month. Nightly WFS polling is sufficient.

**Why must regulatory records be preserved both in FSM and in CRM?**
Regulatory bodies may audit either system. Having the record only in CRM creates a single point of failure for compliance evidence. Having it in both creates defense-in-depth. The quarterly consistency check verifies they remain aligned.

**Why is billing integration bidirectional rather than just inbound?**
Outage records affect billing — consumers experiencing extended outages may be eligible for regulatory rebates. The FSM is the authoritative source of outage records, so it must provide them to billing for rebate computation. This is a formal data exchange, not a one-way import.

**Why are there staleness thresholds on SCADA data rather than simply rejecting stale readings?**
Operational continuity. During network incidents or maintenance, some SCADA feeds may lag — rejecting stale data outright would leave the Supply Optimization Engine with no data at all. Flagging staleness and reducing reliability score preserves operation while alerting that the data should be verified.

---

## 12. Summary — Integration Contract Artifacts

For every integration, three artifacts must be produced and maintained:

1. **API specification document** (this document — overall architecture)
2. **OpenAPI 3.0 schema files** for REST endpoints, one per integration
3. **Message schema files** (JSON Schema for event-driven, XML Schema for regulatory) per message type

These artifacts form the contract between the FSM team and the external system teams. Changes are versioned; breaking changes require coordination with dependent teams via a formal change management process.

---

*End of Integration Specifications Document*
*FSM System · Integration Architecture · Version 1.0 · April 2026*
