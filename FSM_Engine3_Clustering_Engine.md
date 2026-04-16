# Clustering Engine — Design Specification
## Field Service Management System · Utility Field Response Optimization
**Document:** Engine 3 of 4
**Version:** 1.0 · April 2026

---

## 1. Purpose and Role

The Clustering Engine groups tasks that can be executed within a single planned shutdown window, replacing multiple separate outage events with one. Every task successfully clubbed saves at least one outage event from the SAIFI numerator — a direct, measurable improvement in the network reliability index.

Beyond the reliability benefit, clustering also creates a consumer communication obligation. Every planned shutdown must be communicated to affected consumers in advance. The Clustering Engine owns this obligation — it identifies affected consumers, triggers regulatory advance notices, monitors execution, and sends restoration confirmations.

**What it is:** A two-mode planning engine — runs proactively to identify opportunities, and reactively to validate and manage approved clusters.
**What it does:** Identifies clustering opportunities, computes consumer impact, triggers supply optimization, manages advance notification, monitors execution windows, and drives post-completion communication.
**What it does not do:** Approve clusters (that is the planner), assign crew (that is the Matching Engine), or execute switching (that is field crew).

---

## 2. Two Operating Modes

| Mode | Trigger | Purpose |
|---|---|---|
| Planning mode | Nightly after Schedule Engine · on new task generation | Proactively identify and propose cluster opportunities |
| Validation mode | Planner requests approval · dispatcher requests cluster | Validate feasibility of a manually proposed cluster |

---

## 3. Stage 1 — Opportunity Identification

Runs nightly (after the Schedule Engine nightly run at 01:00) and immediately after each new task is generated.

### 3.1 Same Isolation Scope Scan

Tasks on assets that share an upstream switching point can be done in a single shutdown:

```
For each target task T (newly generated or recently modified):

  Find upstream switching point for T's asset:
    Walk ASSET.parent_asset_id hierarchy upward
    Identify the feeder or switching point asset
    (also check ALT_SUPPLY_SOURCE.feeder_asset_id for the zone)

  Query candidate tasks:
    SELECT task_id FROM TASK
    WHERE status IN (NEW, ASSIGNED)
    AND stream_type IN (SCHEDULED, PROJECT)
    AND cluster_id IS NULL
    AND clubbing_opportunity_flagged = false
    AND asset_id IN (
      -- Assets downstream of same switching point
      SELECT asset_id FROM ASSET
      WHERE zone_id = T.asset.zone_id
      AND (shares upstream switching point with T.asset)
    )
    AND sla_deadline BETWEEN
        T.sla_deadline − clustering_date_window_days
        AND T.sla_deadline + clustering_date_window_days
    AND (T.estimated_duration + candidate.estimated_duration)
        ≤ max_shutdown_window_hours × 60

  cluster_type = SAME_ISOLATION_SCOPE
```

### 3.2 Same Corridor Scan

Tasks in geographic proximity with the same workforce pool can be batched into a single crew visit even without a shared switching point:

```
Query candidate tasks:
  SELECT task_id FROM TASK t
  JOIN LOCATION l ON t.location_id = l.location_id
  WHERE t.status IN (NEW, ASSIGNED)
  AND t.stream_type IN (SCHEDULED, PROJECT)
  AND t.cluster_id IS NULL
  AND t.task_type.workforce_pool = T.task_type.workforce_pool
  AND ST_Distance(l.coords, T_location.coords) ≤ clustering_proximity_km
  AND t.sla_deadline BETWEEN
      T.sla_deadline − corridor_date_window_days
      AND T.sla_deadline + corridor_date_window_days

  cluster_type = SAME_CORRIDOR
```

### 3.3 Proposed Cluster Creation

When candidates are found in either scan:

```
1. Set clubbing_opportunity_flagged = true on T and all candidates

2. Create TASK_CLUSTER:
   status = PROPOSED
   lead_task_id = task with nearest (most urgent) sla_deadline
   cluster_type = SAME_ISOLATION_SCOPE or SAME_CORRIDOR
   planned_window_start = lead_task sla_deadline − default_window_days
   planned_window_end = planned_window_start + estimated_combined_duration_hours
   tasks_proposed = COUNT(T + candidates)
   saifi_events_saved = tasks_proposed − 1

3. Create TASK_CLUSTER_MEMBER for each task:
   lead task → member_role = LEAD
   dependent tasks → member_role = DEPENDENT (require this shutdown)
   opportunistic tasks → member_role = OPPORTUNISTIC (can be done at this time)
   added_by = SYSTEM
   sequence_in_cluster = assigned by estimated_duration descending
                         (longest task first to ensure window is met)

4. Notify planning team via CRM_NOTIFICATION:
   notification_type = CLUSTER_OPPORTUNITY
   Message: "Clustering opportunity identified: [N] tasks in [zone/corridor],
    saves [saifi_events_saved] outage events.
    Earliest task due: [date]. Review and approve by [date − 14 days]."
```

---

## 4. Stage 2 — Consumer Impact Assessment

Before a planner approves a cluster, the engine computes the full consumer impact. This assessment is automatically generated and presented alongside the approval request.

### 4.1 Affected Consumer Count

```
1. Identify affected zones:
   affected_zones = DISTINCT zone_id for all assets in cluster

2. Count affected consumers:
   total_affected = COUNT(CONSUMER.consumer_id)
   WHERE CONSUMER.location_id IN (
     SELECT location_id FROM LOCATION
     WHERE zone_id IN (:affected_zones)
   )

3. Segment by category:
   residential = COUNT WHERE consumer_category = RESIDENTIAL
   commercial  = COUNT WHERE consumer_category = COMMERCIAL
   industrial  = COUNT WHERE consumer_category = INDUSTRIAL
   critical    = COUNT WHERE consumer_category = CRITICAL_INFRASTRUCTURE
```

### 4.2 Critical Infrastructure Identification

```
For each consumer where consumer_category = CRITICAL_INFRASTRUCTURE:
  Flag individually — these consumers receive individual advance contact
  (hospitals, water treatment plants, data centres, emergency services)
  NOT bulk SMS — personal telephone contact by customer relations team

  Create coordination task for each:
    task_type = CRITICAL_CONSUMER_COORDINATION (a new SCHEDULED task type)
    description = "Contact [consumer name] to confirm backup arrangements
                   before planned shutdown on [date]"
    sla_deadline = planned_window_start − 72 hours
    assigned_to = customer relations officer
```

### 4.3 Supply Optimization Trigger

```
Trigger SUPPLY_OPTIMIZATION_RUN for this cluster:
  scenario_type = SINGLE_OUTAGE (or MULTI_OUTAGE if adjacent clusters)
  outage_event_ids = (cluster's outage_event_id once created at approval)

From optimization result:
  consumers_on_alt_supply = SUPPLY_OPTIMIZATION_RUN.consumers_restorable
  consumers_effectively_interrupted = total_affected − consumers_on_alt_supply
```

### 4.4 SAIDI Impact Calculation

```
planned_duration_hours = planned_window_end − planned_window_start (in hours)

estimated_SAIDI_contribution =
  (consumers_effectively_interrupted × planned_duration_hours × 60)
  ÷ NETWORK_ZONE.total_consumers

comparison_SAIDI_without_clustering =
  SUM of individual task SAIDI contributions
  (each task run as separate outage at same duration)

SAIDI_saving = comparison_SAIDI_without_clustering − estimated_SAIDI_contribution

Present to planner:
  "This cluster affects [N] consumers for up to [hours].
   Alternative supply covers [M] consumers ([%]).
   Estimated SAIDI contribution: [X] consumer-minutes/consumer.
   Versus [Y] consumer-minutes if tasks run separately.
   SAIDI saving: [Z] consumer-minutes. Outage events saved: [saifi_events_saved]."
```

---

## 5. Stage 3 — Cluster Approval and Advance Consumer Notification

When the planner approves the cluster, the engine immediately begins the consumer notification cascade.

### 5.1 On Cluster Approval

```
On TASK_CLUSTER.status → APPROVED:

  1. Create OUTAGE_EVENT:
     outage_type = PLANNED
     zone_id = primary affected zone
     outage_start = planned_window_start
     outage_end = planned_window_end (estimated)
     total_consumers_in_zone = NETWORK_ZONE.total_consumers
     consumers_affected = total_affected (from Stage 2)
     consumers_on_alt_supply = from SUPPLY_OPTIMIZATION_RUN
     consumers_effectively_interrupted = computed

  2. Link TASK_CLUSTER.outage_event_id to new OUTAGE_EVENT

  3. Begin notification cascade (see 5.2)

  4. Schedule 24-hour reminder notification job

  5. For CRITICAL_INFRASTRUCTURE consumers:
     Escalate coordination task to URGENT
     Alert customer relations officer immediately
```

### 5.2 Regulatory Advance Notice — 72 Hours

```
For each CONSUMER in affected zones:
  Check CONSUMER_COMMUNICATION_PREFERENCE:
    planned_outage_consent = true (skip if opted out)
    primary channel = consumer's preference

  Publish to notification event bus:
  {
    notification_type: PLANNED_OUTAGE_ADVANCE,
    communication_type: REGULATORY,
    consumer_id: consumer_id,
    cluster_id: cluster_id,
    outage_event_id: outage_event_id,
    message_variables: {
      date: planned_window_start.date,
      start_time: planned_window_start.time,
      end_time: planned_window_end.time,
      duration_hours: estimated_duration,
      alt_supply_description: (from SUPPLY_SWITCHING_PLAN or "None arranged"),
      reference: cluster_reference,
      contact_number: utility_helpline
    }
  }

  Create CRM_NOTIFICATION record:
    regulatory_record = TRUE (cannot be deleted — compliance evidence)
    triggered_by_engine = CLUSTERING_ENGINE
```

### 5.3 24-Hour Reminder

```
24 hours before planned_window_start (run by scheduled notification job):

  For each consumer in affected zones:
    Publish: PLANNED_REMINDER notification
    Variables include updated supply plan if changed since advance notice

  Create CRM_NOTIFICATION records (regulatory_record = true)
```

### 5.4 Plan Revision Notifications

```
If planned_window_start or planned_window_end changes AFTER initial notice:
  → Re-send to all affected consumers: PLAN_REVISED notification
  → Create new CRM_NOTIFICATION records (regulatory_record = true)
  → Append note to original CRM_NOTIFICATION: "Superseded by [new notification_id]"
  → Re-trigger 24-hour reminder if revision was made more than 24 hours before new window
```

### 5.5 Cluster Cancellation

```
If TASK_CLUSTER.status → CANCELLED at any time after approval:
  → Notify all consumers: PLAN_CANCELLED notification
  → Create CRM_NOTIFICATION records
  → Remove OUTAGE_EVENT or mark as CANCELLED
  → Set all member tasks back to status = NEW for re-scheduling
  → Clear cluster_id from all member TASK records
```

---

## 6. Stage 4 — Execution Monitoring and Post-Completion Communication

### 6.1 Window Monitoring During Execution

Once TASK_CLUSTER.status = IN_EXECUTION, the engine monitors progress every 30 minutes:

```
Every 30 minutes during execution:

  current_time vs planned_window_end:

  If current_time > planned_window_end − 30 minutes
  AND all tasks NOT yet completed:
    → Compute likely overrun:
       remaining_work = SUM(estimated_duration of incomplete tasks)
       likely_completion = now() + remaining_work
       likely_overrun_minutes = MAX(0, likely_completion − planned_window_end)

  If likely_overrun_minutes > 15:
    → Update OUTAGE_EVENT.outage_end = likely_completion
    → Publish RESTORATION_OVERRUNNING notification to all affected consumers:
       "We apologise — the planned maintenance is taking longer than expected.
        Revised restoration time: [likely_completion time].
        We will update you as soon as supply is restored."
    → Create CRM_NOTIFICATION records
    → Alert dispatcher: "Cluster [REF] overrunning by [X] min. Review."
```

### 6.2 On Cluster Completion

```
On TASK_CLUSTER.status → COMPLETED:

  1. Determine actual restoration time:
     actual_restoration = MAX(completed_at) across all member tasks
     (or earlier if supply was restored before all tasks completed)

  2. Determine completion type:
     If actual_restoration < planned_window_end − 15 min:
       completion_type = EARLY
     Else:
       completion_type = ON_TIME (or OVERRUN if past window end)

  3. Consumer notifications:
     If completion_type = EARLY:
       Publish EARLY_RESTORATION to all consumers:
         "Good news — maintenance complete ahead of schedule.
          Supply restored at [time], earlier than planned [planned_window_end time]."
     Else:
       Publish SUPPLY_RESTORED to all consumers:
         "Power supply has been restored in your area as of [time].
          We apologise for the interruption of [actual_duration]."
     Create CRM_NOTIFICATION records

  4. Update OUTAGE_EVENT:
     outage_end = actual_restoration
     duration_minutes = (outage_end − outage_start) in minutes
     consumer_minutes_interrupted = consumers_effectively_interrupted × duration_minutes
     restoration_method = (from SUPPLY_SWITCHING_PLAN execution outcome)

  5. Update CONSUMER_OUTAGE_RECORD for each affected consumer:
     supply_fully_restored_at = actual_restoration
     effective_interruption_minutes = (actual_restoration − supply_lost_at) in minutes
       accounting for alt_supply_restored_at if alternative supply was provided

  6. Auto-close linked complaint tasks:
     For each CONSUMER_OUTAGE_RECORD where complaint_raised = true:
       Set complaint_task_id TASK.status → COMPLETED (auto-close)
       Publish RESOLVED notification:
         "Your complaint [WO-REF] has been resolved —
          planned maintenance is complete and supply is restored."

  7. Trigger RELIABILITY_SNAPSHOT recalculation for affected zones

  8. TASK_CLUSTER.tasks_completed = actual count of tasks completed in window
     (may differ from tasks_proposed if some were deferred)
     For deferred tasks: set TASK_CLUSTER_MEMBER.completed_in_window = false
                         set deferred_reason
                         set member task status = NEW for re-scheduling
```

---

## 7. Consumer Communication Matrix

| Trigger event | Notification type | Timing | Regulatory? |
|---|---|---|---|
| Cluster approved | PLANNED_OUTAGE_ADVANCE | Immediately (≥ 72 hrs ahead) | Yes |
| 24 hrs before window | PLANNED_REMINDER | Scheduled | Yes |
| Window revised | PLAN_REVISED | Immediately | Yes |
| Cluster cancelled | PLAN_CANCELLED | Immediately | No |
| Execution overrunning > 15 min | RESTORATION_OVERRUNNING | On detection | No |
| Cluster completed early | EARLY_RESTORATION | On completion | No |
| Cluster completed on time | SUPPLY_RESTORED | On completion | No |
| Complaint linked to outage | RESOLVED | On cluster completion | No |
| Critical infrastructure consumer | Individual call task | On approval | Mandatory |

---

## 8. SAIFI Accounting

The core reliability benefit of clustering is precisely tracked:

```
For each completed TASK_CLUSTER:
  saifi_events_saved = tasks_completed − 1
  (tasks that actually completed, not just proposed)

For RELIABILITY_SNAPSHOT computation:
  Total SAIFI events in period =
    COUNT(OUTAGE_EVENT) where period_start ≤ outage_start ≤ period_end

  saifi_events_saved_by_clustering =
    SUM(TASK_CLUSTER.saifi_events_saved) for clusters in period

  Report shows:
    "SAIFI this period: [X]
     Without clustering it would have been: [X + saifi_events_saved]
     Clustering reduced SAIFI by: [saifi_events_saved ÷ total_consumers] per consumer"
```

---

## 9. Configuration Parameters

| Parameter | Default | Description |
|---|---|---|
| clustering_date_window_days | 14 | Days either side of deadline for same-isolation scan |
| corridor_date_window_days | 21 | Days for corridor clustering scan |
| clustering_proximity_km | 5 | Geographic radius for corridor scan |
| max_shutdown_window_hours | 8 | Maximum combined task duration for single shutdown |
| advance_notice_hours_primary | 72 | Hours before window for regulatory advance notice |
| advance_notice_hours_reminder | 24 | Hours before window for reminder |
| overrun_alert_threshold_minutes | 15 | Minutes over window before overrun notification sent |
| overrun_monitoring_interval_minutes | 30 | How often progress is checked during execution |
| min_tasks_for_clustering | 2 | Minimum tasks to form a valid cluster |
| critical_infrastructure_contact_hours_before | 72 | When individual contact task is created |

---

## 10. Key Design Decisions

**Why is consumer notification the Clustering Engine's responsibility rather than a separate notification service?**
Because the Clustering Engine is the only component that knows the full context of a planned outage — which consumers are affected, what alternative supply is arranged, what the precise window is, and when it changes. A separate notification service would need all this information passed to it, creating coupling. Owning the notification trigger directly means the engine can send accurate, timely messages with the right variables at the right moments.

**Why does one cluster generate exactly one OUTAGE_EVENT?**
SAIFI is computed as total consumer interruption events ÷ total consumers. If five tasks each generated their own OUTAGE_EVENT, five interruption events would be counted. One cluster = one OUTAGE_EVENT = one interruption event counted — the direct reliability benefit of the clustering decision is captured in the index immediately.

**Why are regulatory notices marked as permanent records?**
Regulatory bodies may audit advance notice compliance years after the event. A notice that was sent and then deleted cannot be proven to have been sent. regulatory_record = true prevents deletion and ensures the audit trail exists for the lifetime of the system.

**Why are CRITICAL_INFRASTRUCTURE consumers handled differently?**
A hospital on a planned outage needs backup power arranged before supply is cut — not just an SMS. An SMS saying "supply interrupted 09:00 tomorrow" is insufficient for a critical facility. Individual telephone contact, confirmation of backup arrangements, and a coordination task tracked to completion are the appropriate response.

**What happens if a cluster window is missed entirely?**
If the planned_window_end passes and the cluster is not marked COMPLETED, the engine flags it as a status anomaly, alerts the dispatcher, and creates an incident review task. The member tasks remain in their current status — they are not auto-cancelled. The cluster record remains for the SAIDI impact calculation.

---

*End of Clustering Engine Design Specification*
*FSM System · Engine 3 of 4 · Version 1.0 · April 2026*
