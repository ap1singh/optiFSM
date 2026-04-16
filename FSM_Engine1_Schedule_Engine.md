# Schedule Engine — Design Specification
## Field Service Management System · Utility Field Response Optimization
**Document:** Engine 1 of 4
**Version:** 1.0 · April 2026

---

## 1. Purpose and Role

The Schedule Engine is the system's automated task generation backbone. It transforms static maintenance schedule definitions into live work orders, ensuring that every prescribed inspection, test, and planned maintenance activity appears in the dispatch queue at the right time — without any human creating it manually.

Without the Schedule Engine, the entire scheduled task stream (9 of 15 task types) requires manual work order creation. This creates gaps, missed compliance deadlines, and inconsistent scheduling. The engine eliminates this entirely.

**What it is:** A background process — not user-facing.
**What it does:** Reads MAINTENANCE_SCHEDULE, determines what is due, creates TASK records, fires alerts, detects clustering opportunities, and maintains the feedback loop.
**What it does not do:** Assign tasks, optimize routes, contact workers, or handle reactive tasks.

---

## 2. Trigger Schedule

| Run type | Time | Purpose |
|---|---|---|
| Primary nightly run | 01:00 daily | Full task generation sweep across all active schedules |
| Alert sweep | Every 6 hours | SLA alert monitoring for all open tasks |
| Post-completion callback | On TASK status → COMPLETED | Update next_due_date, condition_score, clustering scan |
| On-demand | Manual trigger by IT administrator | After bulk data import or schedule configuration change |

---

## 3. Core Task Generation Loop

The engine processes every active MAINTENANCE_SCHEDULE record on each nightly run.

### 3.1 Generation Condition

```
For each MAINTENANCE_SCHEDULE where is_active = true:

  check_date = next_due_date − advance_notice_days

  Condition to generate:
    today >= check_date
    AND last_generated_date < check_date
    AND no existing TASK with this schedule_id has status
        NOT IN (COMPLETED, CANCELLED)
```

The third condition prevents duplicate task creation if the engine runs multiple times or if a manually created task already covers the schedule.

### 3.2 Target Asset Resolution

```
If asset_id is populated:
  → Generate exactly one TASK for that specific asset

If only asset_type_id is populated (type-level schedule):
  → Query all ASSET records where:
       asset_type_id matches
       AND status = OPERATIONAL
       AND parent_asset_id IS NULL (top-level assets only,
           sub-components inherit through parent)
  → Generate one TASK per matching asset
```

### 3.3 TASK Record Creation

For each target asset, the engine creates a TASK record with the following fields populated:

| Field | Value set by engine |
|---|---|
| task_type_id | From MAINTENANCE_SCHEDULE.task_type_id |
| asset_id | From resolved asset |
| location_id | From ASSET.location_id |
| schedule_id | This MAINTENANCE_SCHEDULE.schedule_id |
| stream_type | SCHEDULED |
| triggered_by | CALENDAR |
| status | NEW (or ON_HOLD if weather condition active — see Section 7) |
| priority_class | From TASK_TYPE.priority_class default |
| sla_type | COMPLIANCE_DATE |
| sla_deadline | MAINTENANCE_SCHEDULE.next_due_date |
| estimated_duration_minutes | Calculated via complexity band (see Section 4) |
| complexity_band | Derived from ASSET.condition_score (see Section 4) |
| created_at | now() |

### 3.4 SLA_TRACKER Record Creation

Immediately after TASK creation, one SLA_TRACKER record is created:

```
sla_type = COMPLIANCE_DATE
sla_deadline = TASK.sla_deadline
alert_30d_triggered = false
alert_7d_triggered = false
alert_1d_triggered = false
is_breached = false
```

### 3.5 Schedule Update

After successful task generation:

```
MAINTENANCE_SCHEDULE.last_generated_date = today
MAINTENANCE_SCHEDULE.next_due_date = today + frequency_days
  (or computed from frequency_type if not CUSTOM)
```

Frequency computation by type:

| frequency_type | next_due_date formula |
|---|---|
| MONTHLY | today + 30 days |
| QUARTERLY | today + 91 days |
| SEMI_ANNUAL | today + 182 days |
| ANNUAL | today + 365 days |
| CUSTOM | today + frequency_days |

---

## 4. Duration Estimation with Complexity Adjustment

The engine does not use a flat default duration. It adjusts the estimate based on the asset's current condition — a deteriorating asset takes longer to test or inspect than a well-maintained one.

### 4.1 Complexity Band Assignment

| ASSET.condition_score | complexity_band assigned | Duration multiplier |
|---|---|---|
| 8 – 10 | LOW | 0.85 × default |
| 6 – 7 | MEDIUM | 1.00 × default |
| 4 – 5 | HIGH | 1.30 × default |
| 1 – 3 | VERY_HIGH | 1.60 × default |

### 4.2 Duration Calculation

```
estimated_duration_minutes =
  ROUND(TASK_TYPE.default_duration_minutes × multiplier)

Example:
  Transformer test default = 240 minutes
  Asset condition_score = 4 → complexity = HIGH → multiplier = 1.30
  estimated_duration_minutes = ROUND(240 × 1.30) = 312 minutes
```

This value is stored in TASK.estimated_duration_minutes and is what the dispatcher sees as the planned window for scheduling. The actual duration recorded at completion becomes training data for Phase 3 ML prediction.

---

## 5. SLA Alert Mechanism

The alert sweep runs every 6 hours and evaluates every SLA_TRACKER record where is_breached = false.

### 5.1 Compliance-Date SLA Alerts (Scheduled Tasks)

```
days_remaining = sla_deadline::date − today::date

If days_remaining <= 30 AND alert_30d_triggered = false:
  → Send notification to planning team:
     "Scheduled task [WO-REF] on asset [asset_name] due in 30 days.
      Not yet scheduled. Assign before [date]."
  → Set alert_30d_triggered = true

If days_remaining <= 7 AND alert_7d_triggered = false:
  → Send escalation to supervisor:
     "[URGENT] Task [WO-REF] due in 7 days — still unassigned."
  → Set alert_7d_triggered = true

If days_remaining <= 1 AND alert_1d_triggered = false:
  → Send critical alert to management:
     "[CRITICAL] Task [WO-REF] due TOMORROW — immediate action required."
  → Set alert_1d_triggered = true

If days_remaining < 0 AND TASK.status != COMPLETED:
  → Set SLA_TRACKER.is_breached = true
  → Set breached_at = now()
  → Set breach_notified = false (triggers management notification)
  → Escalate TASK.priority_class to P2 Urgent
  → Escalate to TASK.dispatcher_notes: "[COMPLIANCE BREACH — immediate scheduling required]"
```

### 5.2 Response-Time SLA Alerts (Reactive Tasks)

For reactive tasks (P1/P2) the same mechanism applies but in hours:

```
hours_remaining = (sla_deadline − now()) in hours

P1 tasks: alert if hours_remaining < 0.5 (30 minutes)
P2 tasks: alert if hours_remaining < 2 hours

Breach: same mechanism — is_breached = true, priority escalated
```

### 5.3 SLA Reporting

All breach events are recorded permanently in SLA_TRACKER and appear in the management dashboard's compliance report. The regulatory body may request evidence of SLA performance — these records constitute that evidence.

---

## 6. Clustering Opportunity Detection

Immediately after generating each new task (within the same nightly run), the engine scans for clustering opportunities before the task appears in the dispatch queue.

### 6.1 Same Isolation Scope Scan

```
For the newly generated task T on asset A in zone Z:

Query existing TASK records where:
  status IN (NEW, ASSIGNED)
  AND stream_type IN (SCHEDULED, PROJECT)
  AND cluster_id IS NULL
  AND asset_id IN (
    SELECT asset_id FROM ASSET
    WHERE zone_id = Z
    AND the asset shares an upstream switching point with A
    — determined via ASSET.parent_asset_id hierarchy
    and ALT_SUPPLY_SOURCE.feeder_asset_id lookup
  )
  AND sla_deadline BETWEEN
      (T.sla_deadline − clustering_date_window_days)
      AND (T.sla_deadline + clustering_date_window_days)
  AND estimated combined duration ≤ max_shutdown_window_hours × 60
```

### 6.2 Same Corridor Scan

```
Query existing TASK records where:
  status IN (NEW, ASSIGNED)
  AND stream_type IN (SCHEDULED, PROJECT)
  AND cluster_id IS NULL
  AND task_type.workforce_pool = T.task_type.workforce_pool
  AND ST_Distance(location.coords, T.location.coords) ≤ clustering_proximity_km
  AND sla_deadline BETWEEN
      (T.sla_deadline − corridor_date_window_days)
      AND (T.sla_deadline + corridor_date_window_days)
```

### 6.3 Opportunity Recording

```
If candidates found in either scan:
  Set T.clubbing_opportunity_flagged = true
  Set candidate tasks' clubbing_opportunity_flagged = true
  Create TASK_CLUSTER record:
    status = PROPOSED
    lead_task_id = task with nearest sla_deadline
    cluster_type = SAME_ISOLATION_SCOPE or SAME_CORRIDOR
    tasks_proposed = COUNT(T + candidates)
    saifi_events_saved = tasks_proposed − 1
  Create TASK_CLUSTER_MEMBER records for each task
  Notify planning team:
    "Clustering opportunity: [N] tasks in [zone/corridor],
     saves [saifi_events_saved] outage events. Review by [date]."
```

---

## 7. Weather-Hold Mechanism

Before setting a newly generated task to status = NEW, the engine checks for active adverse weather in the asset's zone.

```
If TASK_TYPE.weather_sensitive = true
AND TASK_TYPE.indoor_task = false:

  Check WEATHER_EVENT where:
    zone_id = ASSET.zone_id
    AND ended_at IS NULL (event ongoing)
    AND severity IN ('HIGH', 'EXTREME')
    AND event_type IN prohibited types for this task type
    (e.g. LIGHTNING for any HV outdoor task,
          HIGH_WIND for aerial lift tasks where max_wind_speed_kmh is set)

  If active adverse weather found:
    → Set TASK.status = ON_HOLD
    → Prefix TASK.description with "[WEATHER HOLD — pending clearance]"
    → SLA clock continues running — compliance deadline not paused
    → Notify planning team:
       "Task [WO-REF] generated but held due to [weather_type]
        in [zone]. SLA deadline: [date]. Activate when weather clears."

CRITICAL RULE: The compliance clock NEVER pauses for a weather hold.
The task is created, documented, and held — but the regulatory deadline
continues. This ensures regulatory records show the task was known and
managed, not forgotten.
```

When the weather event ends (`WEATHER_EVENT.ended_at` populated), a separate weather-clearance sweep changes status from ON_HOLD back to NEW for all held tasks in the affected zone.

---

## 8. Feedback Loop on Task Completion

When a field worker marks a task COMPLETED on the mobile app, a completion event triggers callbacks into the schedule engine.

### 8.1 Standard Completion

```
On TASK.status → COMPLETED (where schedule_id is not null):

  Update MAINTENANCE_SCHEDULE:
    next_due_date = TASK.completed_at + frequency_days
    last_generated_date = today

  Update ASSET:
    last_inspection_date = TASK.completed_at
      (if task_type.event_type = INSPECTION)
    last_test_date = TASK.completed_at
      (if task_type.event_type = TEST)
    condition_score = SERVICE_HISTORY.condition_score_after
      (if SERVICE_HISTORY record was created)
```

### 8.2 Test Record Override

When a TEST_RECORD is created with a specific next_test_due_date (particularly relevant for CONDITIONAL_PASS results where the tester prescribes a shorter re-test interval):

```
If TEST_RECORD.next_test_due_date IS NOT NULL:
  → Use TEST_RECORD.next_test_due_date instead of
    completed_at + frequency_days
  → Update MAINTENANCE_SCHEDULE.next_due_date accordingly

Example:
  Annual transformer test completed.
  Default next_due = completed_at + 365 days.
  But test result = CONDITIONAL_PASS with 6-month re-test prescription.
  → TEST_RECORD.next_test_due_date = completed_at + 182 days
  → MAINTENANCE_SCHEDULE.next_due_date updated to 182 days hence
  → Engine will generate the re-test task in 152 days (182 − 30 advance)
```

### 8.3 Duration Learning Record

```
On every TASK completion:
  Log: {
    task_type_id,
    asset_id,
    complexity_band,
    weather_at_start,
    weather_caused_extension,
    weather_extension_minutes,
    estimated_duration_minutes,
    actual_duration_minutes,
    variance = actual − estimated
  }

This dataset accumulates for Phase 3 ML duration prediction.
No action needed in Phase 1 or 2 — simply ensure the data is captured.
```

---

## 9. Configuration Parameters

All configurable without code changes — stored in a system configuration table.

| Parameter | Default | Description |
|---|---|---|
| engine_run_time | 01:00 | Primary nightly generation time |
| alert_sweep_frequency_hours | 6 | How often SLA alerts are checked |
| default_advance_notice_days | 30 | Days before compliance date to generate task |
| clustering_date_window_days | 14 | Days either side of deadline for same-isolation scan |
| corridor_date_window_days | 21 | Days either side of deadline for corridor scan |
| clustering_proximity_km | 5 | Geographic radius for corridor clustering |
| max_shutdown_window_hours | 8 | Maximum combined task duration for single shutdown |
| condition_score_low_threshold | 7 | Below this → MEDIUM complexity |
| condition_score_medium_threshold | 5 | Below this → HIGH complexity |
| condition_score_high_threshold | 3 | Below this → VERY_HIGH complexity |
| weather_hold_severities | HIGH,EXTREME | Severity levels triggering weather hold |
| breach_escalation_priority | P2 | Priority class to assign on SLA breach |
| nightly_run_timeout_minutes | 30 | Alert IT if engine does not complete within this |

---

## 10. What the Schedule Engine Does NOT Do

| Excluded responsibility | Who handles it |
|---|---|
| Assign tasks to workers | Matching Engine |
| Optimize routes or sequences | Route Optimizer (Phase 2) |
| Approve cluster proposals | Planning officer via Dispatcher Board |
| Generate reactive (P1/P2 event) tasks | Dispatcher or automated event feed |
| Contact field workers | Mobile app notifications post-assignment |
| Handle billing or regulatory submission | Separate integration processes |
| Compute reliability indices | RELIABILITY_SNAPSHOT computation job |

---

## 11. Key Design Decisions

**Why nightly generation and not on-demand?**
Scheduled tasks have weeks or months of lead time. Nightly generation is sufficient, predictable, and gives the planning team the full next day to review, cluster, and approve before the dispatch queue starts. Immediate generation would create noise in the live queue.

**Why does the compliance clock not pause for weather holds?**
Regulatory obligations do not pause for weather. If an annual transformer test is due on 15 May and cannot be performed due to monsoon, the regulatory record must show the task was planned, held for a legitimate reason, and completed as soon as conditions allowed. A paused clock would obscure the true compliance position.

**Why compute complexity from condition_score rather than using a flat default?**
A transformer with a condition score of 3 is likely to have additional findings, require more careful testing, and potentially reveal secondary issues during the visit. Flat duration estimates consistently underestimate these tasks, creating SLA pressure and unrealistic daily schedules.

**Why capture duration variance from day one even though ML is Phase 3?**
The ML model needs 12–18 months of diverse training data to be reliable. Starting capture at Phase 1 means the model can be trained by the time Phase 3 arrives. Starting at Phase 3 means waiting another 12–18 months after deployment.

---

*End of Schedule Engine Design Specification*
*FSM System · Engine 1 of 4 · Version 1.0 · April 2026*
