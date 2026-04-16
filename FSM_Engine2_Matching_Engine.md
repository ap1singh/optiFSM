# Matching Engine — Design Specification
## Field Service Management System · Utility Field Response Optimization
**Document:** Engine 2 of 4
**Version:** 1.0 · April 2026

---

## 1. Purpose and Role

The Matching Engine surfaces the right field worker for every task. When a task enters the queue — whether a field crew reports a fault, the Schedule Engine generates a compliance inspection, or a consumer complaint arrives — the Matching Engine runs and presents ranked suggestions to the dispatcher within seconds.

It does not assign. Every assignment requires a human dispatcher confirmation. The engine's role is to eliminate the cognitive burden of searching through dozens of workers manually, while ensuring safety constraints are never bypassed.

**What it is:** A real-time query and scoring engine triggered by new tasks.
**What it does:** Filters ineligible workers through four hard constraint gates, scores eligible workers on three dimensions, surfaces top three candidates to the dispatcher with plain-language reasoning.
**What it does not do:** Assign workers, optimize routes, handle multi-task scheduling.

---

## 2. Trigger Conditions and Speed by Priority

| Priority | Trigger mechanism | Maximum time to suggestions |
|---|---|---|
| P1 Emergency | Immediate — synchronous blocking call on task creation | < 5 seconds |
| P2 Urgent | Near-real-time queue — runs within 2 minutes | < 2 minutes |
| P3 Scheduled | Batch sweep every 15 minutes | < 15 minutes |
| P4 Routine | Daily planning sweep | Next morning run |

For P1, the dispatcher sees suggestions appear on screen as the task is being entered — the engine runs synchronously. For P4, suggestions may be pre-computed overnight.

---

## 3. The Seven-Step Flow

Steps 1–4 are hard constraint gates. A worker eliminated at any gate cannot be reconsidered regardless of any other attribute.
Steps 5–7 are scoring steps for workers who pass all four gates.

---

### Step 1 — Workforce Pool Filter (Hard Gate)

```
TASK_TYPE.workforce_pool must match WORKER.workforce_pool

ELECTRICAL tasks → only ELECTRICAL workers considered
VEGETATION tasks → only VEGETATION workers considered
SPECIALIST tasks → only SPECIALIST workers considered

Query:
  SELECT worker_id FROM WORKER
  WHERE workforce_pool = :task_type.workforce_pool
  AND status NOT IN ('OFF_SHIFT', 'ON_LEAVE')
```

This single filter typically eliminates 60–80% of workers before any further checks run. A vegetation crew member is never surfaced for a transformer fault under any circumstances.

---

### Step 2 — Skill and Authority Filter (Hard Gate)

This is the most critical safety filter in the system.

#### 2a — Mandatory Skill Check

Every mandatory skill in TASK_SKILL_REQUIREMENT must be matched:

```
SELECT ws.worker_id
FROM WORKER_SKILL ws
JOIN TASK_SKILL_REQUIREMENT tsr
  ON ws.skill_id = tsr.skill_id
WHERE tsr.task_type_id = :task_type_id
  AND tsr.is_mandatory = true
  AND ws.is_active = true
  AND (ws.expiry_date IS NULL OR ws.expiry_date > CURRENT_DATE)
GROUP BY ws.worker_id
HAVING COUNT(DISTINCT ws.skill_id) = (
  SELECT COUNT(*) FROM TASK_SKILL_REQUIREMENT
  WHERE task_type_id = :task_type_id
  AND is_mandatory = true
)
```

The HAVING COUNT clause is critical — a worker with 4 of 5 required skills is as ineligible as one with none.

#### 2b — Authority Checks

```
If TASK_TYPE.requires_safety_officer = true:
  → Worker must have WORKER.is_safety_officer = true
  → OR a safety officer must be available for co-dispatch pairing
    (handled in co-dispatch logic — see Section 6)

If task requires live HV work (isolation_record will be needed):
  → At least one crew member must have WORKER.is_hv_authority = true
```

#### 2c — Preferred Skill Notation

Workers with non-mandatory preferred skills (TASK_SKILL_REQUIREMENT.is_mandatory = false) are not filtered out but are noted for Step 7 scoring bonus.

**Safety rule:** An expired certification (WORKER_SKILL.is_active = false OR expiry_date < today) results in permanent exclusion from that skill's tasks. This is automatic and cannot be overridden by a dispatcher.

---

### Step 3 — Shift Availability Filter (Hard Gate)

Three conditions checked in sequence:

#### 3a — Status Check

```
WORKER.status must be AVAILABLE or ON_TASK

Excluded statuses: ON_LEAVE, OFF_SHIFT, ON_BREAK (unless P1 override)

ON_TASK workers are included — they may complete their current
task before travel time elapses. Especially relevant for P3/P4 tasks.

STANDBY and ON_CALL workers included for P1 and P2 tasks:
  → Shown to dispatcher with flag: "Standby — call to activate"
```

#### 3b — Shift Window Check

```
From SHIFT_SCHEDULE for today:
  planned_end = shift_end for this worker today

  time_remaining = planned_end − now()

  required_time = TASK.estimated_duration_minutes
                + estimated_travel_minutes (from current location)
                + shift_buffer_min (default 30)

  If time_remaining < required_time → exclude

Exception:
  P1 Emergency tasks override this check
  → Worker included but flagged: "Shift ending in [X] min — overtime likely"
  → Dispatcher explicitly informed before assigning
```

#### 3c — ON_CALL Inclusion

Workers with shift_type = ON_CALL or STANDBY are always included in the eligible pool for P1 and P2 tasks, regardless of clock status.

---

### Step 4 — Parts Availability Check (Soft Gate)

Unlike Steps 1–3, parts insufficiency does not eliminate a worker but does change their display status to the dispatcher.

#### Three Outcomes

**Outcome A — Van stock sufficient (green):**
```
VAN_STOCK.quantity_on_hand >= TASK_PART_REQUIREMENT.quantity_typical
for ALL mandatory parts (is_mandatory = true)

Display: "All parts on van"
No penalty applied
```

**Outcome B — Depot pickup required (amber):**
```
Van stock insufficient for one or more mandatory parts
BUT DEPOT_STOCK at a reachable depot has sufficient stock

Display: "Depot pickup at [Depot Name] required — adds ~[X] min"
Detour time added to travel estimate shown to dispatcher
Workload score penalized slightly (depot trip consumes time)
```

**Outcome C — Parts unavailable (red):**
```
Neither van nor any depot has sufficient stock

Display: "Parts unavailable — check emergency procurement"
Worker shown in list but with prominent warning
Dispatcher must consciously choose to assign despite warning
```

Non-mandatory parts (is_mandatory = false): advisory only. Shown as informational note, no scoring impact.

---

### Step 5 — Proximity Score (50% weight)

```
Travel time calculation:

Phase 1 (Haversine approximation):
  distance_km = haversine(worker.current_lat, worker.current_lng,
                           task.location.lat, task.location.lng)
  speed_kmh = zone_speed[task.location.zone_type]
    (URBAN: 30, SUBURBAN: 45, RURAL: 60, HIGHWAY: 80)
  travel_minutes = (distance_km / speed_kmh) × 60

Phase 2 (Road routing API):
  Replace haversine with OSRM or Google Distance Matrix
  for actual road-network travel time

Proximity score formula:
  proximity_score = MAX(0,
    1 − (travel_minutes / max_travel_threshold_minutes))

  Where max_travel_threshold_minutes = 120 (configurable)
  Workers beyond threshold score 0 but are NOT excluded
  — shown to dispatcher as last resort if no closer worker available
```

---

### Step 6 — Workload Score (30% weight)

```
queue_hours = SUM of remaining estimated_duration_minutes
  for all TASK records where:
    worker_id appears in CREW_ASSIGNMENT for this worker
    AND status IN ('ASSIGNED', 'IN_PROGRESS')
    AND completed_at IS NULL
  Converted to hours

workload_score = MAX(0,
  1 − (queue_hours / remaining_shift_hours))

Where remaining_shift_hours = (shift_end − now()) in hours

Worker with empty queue → workload_score = 1.0
Worker whose entire remaining shift is filled → workload_score = 0.0

This prevents chronic overloading of high-performing workers
while allowing a capable nearby worker with a moderate queue
to score higher than a free but distant one.
```

---

### Step 7 — Skill Preference Score (20% weight)

```
preferred_skills_matched = COUNT of TASK_SKILL_REQUIREMENT records
  where is_mandatory = false
  AND worker holds the skill (is_active = true, not expired)

total_preferred_skills = COUNT of TASK_SKILL_REQUIREMENT records
  where is_mandatory = false

grade_level_map = {
  'Junior Technician': 1,
  'Technician': 2,
  'AE': 3,
  'Senior Technician': 3,
  'Senior Engineer': 4,
  'Supervisor': 5
}

grade_bonus = 0.05 × MAX(0,
  grade_level_map[worker.grade] − minimum_grade_for_task_type)
  Capped at 0.20

skill_preference_score = (preferred_skills_matched / MAX(1, total_preferred_skills))
  + grade_bonus
  Normalised to 0–1 (capped at 1.0)
```

This rewards depth of skill beyond the minimum requirement without ever overriding the hard safety constraints.

---

## 4. Composite Score Formula

```
composite_score =
    (0.50 × proximity_score)
  + (0.30 × workload_score)
  + (0.20 × skill_preference_score)
```

Workers ranked by composite_score descending. Top three become suggestions.

### Weight Configuration

The weights (0.50, 0.30, 0.20) are stored in a configuration table — not hardcoded. Operations may adjust them:

| Scenario | Suggested adjustment |
|---|---|
| Emergency storm response | Proximity: 0.70 · Workload: 0.20 · Skill: 0.10 — speed prioritized |
| Balanced normal operations | Default: 0.50 · 0.30 · 0.20 |
| Workload equity focus | Proximity: 0.40 · Workload: 0.45 · Skill: 0.15 — equity prioritized |
| Specialist-scarce tasks | Proximity: 0.35 · Workload: 0.25 · Skill: 0.40 — skill depth rewarded |

Every CREW_ASSIGNMENT record captures which weight configuration was active at the time of suggestion — enabling post-hoc analysis of how weighting choices affected outcomes.

---

## 5. Top Three Output Format

The dispatcher sees plain-language output, never raw scores:

**Suggestion card 1 (recommended):**
```
[Worker name] · [Grade] · [Key certifications]
ETA: [X] min to site
"[Plain reason]: Nearest certified HT lineman with HV authority
and cable fault kit. 1 task in queue completing in ~40 min.
Acts as safety officer — single crew sufficient."
[Assign button] [View on map button]
Score bars: Proximity [■■■■■■■■■□] 91%  Workload [■■■■■■■□□□] 80%
Parts status: All parts on van [green indicator]
```

**Suggestion card 2:**
```
[Worker name] + [Safety Officer name] (co-dispatch pair)
ETA: [X] min to site
"Co-dispatch required — [worker] is not a safety officer.
[SO name] available [Y km away]. Depot pickup needed for [part]."
[Assign crew pair button]
Parts status: Depot pickup at Depot A — adds ~15 min [amber]
```

**Suggestion card 3:**
```
[Worker name] · ON CALL — call to activate
ETA: [X] min to site
"On-call standby. Furthest option but fully equipped.
No current queue. Activate by phone before dispatching."
[Activate and assign button]
Parts status: All parts on van [green]
```

---

## 6. Co-Dispatch Logic

When TASK_TYPE.requires_safety_officer = true, the engine performs a two-pass selection:

```
Pass 1 — Lead worker selection:
  Run Steps 1–7 for LEAD role
  Identify top candidate

Pass 2 — Safety officer pairing:
  If top candidate has WORKER.is_safety_officer = true:
    → Single person dispatch — no co-dispatch needed
    → Card shows: "[Name] (Lead + Safety Officer)"

  If top candidate does NOT have is_safety_officer = true:
    → Run Steps 1–4 on remaining eligible pool
       filtered to WORKER.is_safety_officer = true
    → Score by proximity to task site only
    → Select nearest available safety officer
    → Card shows: "[Lead name] + [Safety Officer name]"
    → Two ETAs shown separately (both must be on site before work starts)
    → Both assignments recorded in CREW_ASSIGNMENT with respective roles

If NO safety officer is available:
  → All suggestion cards show: "NO SAFETY OFFICER AVAILABLE —
    task cannot start until one is assigned. Review standby roster."
  → Dispatcher must not assign lead worker without safety officer pairing
```

---

## 7. Override Capture — The Learning Loop

Every dispatcher override is a structured learning signal, not just a free-text note.

```
When dispatcher assigns someone NOT in the top suggestion:
  → Dispatcher must select override_reason_code from:
       WORKER_UNAVAILABLE (I know their actual status differs)
       LOCAL_KNOWLEDGE (better route or access knowledge)
       SPECIAL_EQUIPMENT (equipment on different van)
       PRIOR_RELATIONSHIP (worker has history with this asset/consumer)
       SKILL_CLARIFICATION (worker has unlisted skill/experience)
       WORKLOAD_PREFERENCE (different workload balance preferred)
       OTHER (free text required)

  → CREW_ASSIGNMENT.suggestion_rank = NULL (not 1, 2, or 3)
  → CREW_ASSIGNMENT.override_reason_code = selected code
  → CREW_ASSIGNMENT.override_notes = free text explanation

This data is reviewed weekly by operations manager:
  → Override distribution report surfaces systematic algorithm gaps
  → Phase 3 ML model trains on this dataset to learn local knowledge
```

**Override analysis queries:**

| Override reason cluster | What it reveals |
|---|---|
| LOCAL_KNOWLEDGE clustering | Proximity model missing local road conditions or access issues |
| SPECIAL_EQUIPMENT clustering | Van stock data is stale or inaccurate |
| WORKER_UNAVAILABLE clustering | Worker status updates are delayed or missing |
| SKILL_CLARIFICATION clustering | Skill registry is incomplete — worker has uncatalogued competencies |

---

## 8. Re-Trigger Conditions

The matching engine does not run once and stop. It re-runs when the operational situation changes:

| Trigger event | Re-run scope |
|---|---|
| Worker assigned to different task | Re-run for all tasks this worker was a top suggestion for |
| Worker checks in on site (location updated) | Re-run proximity scores for tasks in that zone |
| Worker shift ends | Remove from eligible pool; re-run affected tasks |
| New P1 task displaces P3 task's suggested worker | Re-run P3 tasks for new suggestions |
| Van stock changes after task completion | Re-run parts check for affected worker's pending suggestions |
| Weather event declared in zone | Re-run weather-sensitive tasks in that zone |
| 30 minutes elapsed without dispatcher action | Re-run with fresh worker locations |
| Worker status changes (ON_TASK → AVAILABLE) | Re-run as newly available worker may rank higher |

When a re-trigger produces updated suggestions, the dispatcher sees a subtle indicator on the task card: "Suggestions updated — worker locations refreshed."

---

## 9. Location Staleness Handling

Worker location accuracy is critical for proximity scoring. The engine monitors staleness:

```
location_age_minutes = now() − WORKER.location_updated_at

If location_age_minutes > location_staleness_threshold_min (default 60):
  → Worker shown in suggestions with amber flag:
     "Location data [X] min old — verify before dispatching"
  → proximity_score computed from last known location but flagged as uncertain
  → Dispatcher informed; travel time estimate may not be accurate

If location_age_minutes > 240 (4 hours):
  → Worker excluded from proximity scoring
  → Shown in list only as "Location unknown — contact directly"
```

Workers update their location in three ways: automatic GPS ping from mobile app (Phase 2), manual check-in at start of shift, and implicit update when they mark a task as arrived.

---

## 10. Configuration Parameters

| Parameter | Default | Description |
|---|---|---|
| proximity_weight | 0.50 | Weight for travel time in composite score |
| workload_weight | 0.30 | Weight for queue depth |
| skill_pref_weight | 0.20 | Weight for preferred skills and grade |
| max_travel_threshold_min | 120 | Beyond this, proximity_score = 0 |
| shift_buffer_min | 30 | Extra minutes required beyond task duration before shift end |
| p1_shift_override | true | P1 emergency ignores shift end constraint |
| suggestions_count | 3 | Number of ranked suggestions shown |
| re_trigger_interval_min | 30 | Re-run if dispatcher has not acted |
| location_staleness_threshold_min | 60 | Flag worker location as stale |
| location_exclude_threshold_min | 240 | Exclude worker from proximity scoring |
| urban_speed_kmh | 30 | Phase 1 proximity calculation urban speed |
| rural_speed_kmh | 60 | Phase 1 proximity calculation rural speed |

---

## 11. Key Design Decisions

**Why does parts unavailability not eliminate a worker?**
The dispatcher may know that parts are being sourced through emergency procurement, or that a part listed as required is already available at the site. Elimination would prevent a valid assignment. Warning with prominence achieves the safety intent while preserving dispatcher authority.

**Why is proximity weighted at 50%?**
Travel time is the most direct lever on SLA compliance, fuel cost, and task throughput. A worker 5 minutes away who is moderately loaded will almost always be a better choice than a free worker 45 minutes away for time-critical tasks.

**Why three suggestions rather than one?**
The algorithm has imperfect information. It does not know about informal arrangements, personal knowledge of site access, or recent equipment changes. Three options give the dispatcher choice while the ranked order guides their default. In practice, the top suggestion is accepted 70–80% of the time after initial deployment, rising to 80–90% over time as the algorithm learns from overrides.

**Why is the override reason code mandatory?**
An override without explanation is data loss. The reason codes are the training labels for the Phase 3 model. A required field ensures every override contributes to the learning loop. Free text alone is not sufficient — it cannot be systematically analyzed at scale.

---

*End of Matching Engine Design Specification*
*FSM System · Engine 2 of 4 · Version 1.0 · April 2026*
