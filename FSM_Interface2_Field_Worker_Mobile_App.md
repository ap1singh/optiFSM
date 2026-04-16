# Field Worker Mobile App — Interface Design Specification
## Field Service Management System · Utility Field Response Optimization
**Document:** Interface 2 of 3
**Version:** 1.0 · April 2026

---

## 1. Purpose and Role

The Field Worker Mobile App is the system's most safety-critical interface. It is the tool through which field workers receive tasks, access asset history, execute the isolation and re-energization safety workflow, capture test records and parts consumed, and update task status — all from the field, often in difficult physical conditions.

Three design imperatives govern every decision in this interface:

**Safety first.** The isolation workflow must be unmistakable. Steps must be completed in sequence. Confirmations must be deliberate and explicit. The system must never allow a worker to bypass a safety gate through an accidental tap.

**Field-ready.** The interface must be usable in direct sunlight, with work gloves, with dirty hands, under time pressure, and with intermittent connectivity. This means large touch targets (minimum 44px), high contrast, offline capability, and no critical information hidden behind small controls.

**Minimal cognitive load.** The worker's primary job is the electrical task. The app must require the minimum interaction to capture what the system needs. Pre-populated fields, smart defaults, and single-action confirmations reduce the burden on the worker.

**Primary user:** Field worker / lineman
**Secondary user:** Safety officer (co-dispatch)
**Device:** Android or iOS smartphone (5.5" minimum screen, IP54 rating or protective case)
**Connectivity:** 4G where available; offline mode when connectivity is lost

---

## 2. App Architecture — Six Sections

```
Task List          → All assigned tasks sorted by priority + SLA
Task Detail        → Full task information + asset history
Navigation         → Route to site
Site Check-in      → Arrival confirmation + asset history summary
Isolation Workflow → Pre-work PTW sequence + post-work confirmation
Work Execution     → Test record capture + parts consumed + completion
```

The bottom navigation bar provides access to four persistent sections:
- Tasks (primary)
- Map (navigation)
- History (asset lookup)
- Profile (status + settings)

---

## 3. Persistent Elements

### 3.1 Worker Status Bar

Always visible at the top of every screen (below phone OS status bar):

```
[green dot] [Worker name] · [Current status]           [Primary skill badges]
```

Status options shown to worker with single-tap change:
- AVAILABLE (green) — ready for new tasks
- ON_BREAK (amber) — on authorized break
- ON_TASK (blue) — actively working (set automatically on check-in)

The worker never manually sets ON_TASK — this is set automatically when they check in on site. They set AVAILABLE when they complete a task and before they accept the next. This distinction matters for the matching engine, which reads WORKER.status in real time.

### 3.2 Offline Indicator

When network connectivity is lost, a persistent bar appears below the status bar:

```
[grey dot] Offline · Changes saved locally · Syncing when connected
```

All field actions (status updates, isolation confirmations, test record entries, task completions) are stored locally and synced automatically when connectivity is restored. The worker is never blocked from completing an action because of connectivity — they continue working and the system catches up.

### 3.3 Unread Task Notification Badge

When a new task is assigned while the app is in the background, a push notification appears. Opening the app brings the worker directly to the new task detail. A red badge on the Tasks navigation item shows the count of unread new assignments.

---

## 4. Screen 1 — Task List

### 4.1 Layout

```
[Worker status bar]
[Offline indicator — if applicable]

Tasks (N)                                    [Filter]

[Task card 1 — highest priority]
[Task card 2]
[Task card 3]
...

[Bottom navigation]
```

### 4.2 Task Card Structure

Each card shows:

```
[Task name — max 40 chars, truncated]                    [P1/P2/P3/P4 badge]
[WO reference] · [Status: NEW · ASSIGNED · IN_PROGRESS]
[Zone · Location short name]
[SLA bar ────────── ] [Time remaining]
```

Card visual states:
- New unread task: bold border, "tap to accept" text below reference
- Accepted / assigned: normal border
- In progress: blue left border
- Weather held: amber background, "WEATHER HOLD" label
- Overdue SLA: red background, pulsing border

### 4.3 Sort Order

Tasks sorted by:
1. Status — NEW (needs acceptance) first
2. Priority class — P1 before P2 before P3 before P4
3. SLA time remaining — most urgent first within same priority

### 4.4 Task Acceptance

When a task arrives with status NEW:
```
Worker taps the task card
Task detail loads
Worker taps "Accept task + start navigation"
System: TASK.status changes to ASSIGNED
        CREW_ASSIGNMENT.dispatched_at recorded
        CRM notification sent: "Crew en route"
Navigation to site begins
```

If worker cannot accept (e.g. tool availability issue):
```
Worker taps "Cannot accept — report issue"
Reason code selector appears:
  Tool not available · Vehicle issue · Safety concern · Other
Reason sent to dispatcher with free-text note
Task returns to dispatcher queue for reassignment
```

---

## 5. Screen 2 — Task Detail

### 5.1 Layout

```
[Back] [WO reference] [Priority badge]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[PTW warning banner — if task requires permit]
[Fault/task description — full text]
[Asset details]
[Location details + map thumbnail]
[Parts on your van — check vs requirements]
[Asset service history — last 5 events]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Accept task + start navigation] (green)
[View supply plan] (secondary)
[View cluster info] (secondary — if task is in a cluster)
```

### 5.2 PTW Warning Banner

For task types where TASK_TYPE.requires_permit = true:

```
[amber background]
PTW required before work begins
HV authority confirmation needed for isolation steps
[View isolation checklist →]
```

This banner is the first thing the worker sees — before any task details. The safety requirement is communicated before operational details.

### 5.3 Asset Service History

The last 5 SERVICE_HISTORY events for this asset are shown inline — not behind a "view more" link. The worker sees the asset's recent history before they arrive on site:

```
Asset service history
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[red dot] 18 Feb 2026 — Cable joint failure · same section
          R. Kumar repaired · 3.5 hrs · condition after: 7/10

[blue dot] 12 Nov 2025 — Routine inspection
           Condition 6/10 · minor corrosion noted on joint box

[green dot] 3 Aug 2025 — Full cable replacement
            Condition restored 8/10
```

Colour coding of history dots:
- Red: FAULT or emergency repair event
- Blue: INSPECTION or TEST event
- Green: MAINTENANCE or COMMISSIONING event
- Amber: Deferred or incomplete event

This panel is read-only. It exists to give the worker context before they start work. A worker arriving at a transformer that has had three faults in six months should approach it differently from a first-time fault.

### 5.4 Parts Status Panel

Cross-checks TASK_PART_REQUIREMENT against VAN_STOCK and shows:

```
Parts check — WO-2026-04-0091
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Cable fault kit: 1 required · 1 on van ✓ [green]
Jointing kit: 1 required · 2 on van ✓ [green]
Heat shrink: 3m required · 4m on van ✓ [green]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
All parts confirmed on van
```

If parts are insufficient:
```
Jointing kit: 1 required · 0 on van ✗ [red]
→ Available at Depot A (3.2 km) · contact: [number]
Dispatcher has been notified
```

### 5.5 Supply Plan Access

For tasks linked to an OUTAGE_EVENT with a supply plan:

```
[View supply plan →]
```

Tapping opens the supply switching plan:
```
Supply plan — SSP-2026-04-019
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Plan: ALT-NHV3-01 · Feeder backfeed via TS-7
Load: 5.8 MVA · Losses: 0.018 MW
Approved by: Load Dispatch · 14:29

Switching sequence (4 steps):
1. Close Tie Switch TS-7 at Sector 12 junction
2. Verify voltage on restored section
3. Confirm consumer supply in Sectors 14–16
4. Report to dispatcher

[Mark switching complete]
```

The field worker executing the supply restoration follows this sequence step by step, confirming each action. The SUPPLY_PATH_SEGMENT records are updated as each step is confirmed.

---

## 6. Screen 3 — Navigation

Standard map navigation to task site. The app uses the device's native maps integration (Google Maps or Apple Maps) pre-loaded with the task site address and coordinates from LOCATION.lat/lng.

The app passes:
- Destination: task site name and coordinates
- Mode: Driving (default) or Walking (for dense urban areas)

On arrival (device GPS within 100m of task location):
- Navigation ends automatically
- App transitions to Site Check-in screen

If worker needs to detour to depot:
- "Add depot stop" option available from navigation screen
- Depot location pre-loaded from DEPOT records
- CREW_ASSIGNMENT note added: "Depot detour — [depot name]"

---

## 7. Screen 4 — Site Check-in

### 7.1 Arrival Confirmation

On arriving within 100m of the task site, the app prompts:

```
[Full screen confirmation]

You have arrived at:
[Asset name / site name]
[Address]

[Large green CHECK-IN button — 60px tall]

Asset service history summary:
[Last fault: 18 Feb 2026 — cable joint failure]
[Last test: 12 Nov 2025 — condition 6/10]
[Last maintenance: 3 Aug 2025]
```

The history summary is shown at check-in even if the worker already read it on the task detail screen. Context is repeated at the moment it is needed.

Tapping CHECK-IN:
```
TASK.started_at = now()
WORKER.status = ON_TASK
CREW_ASSIGNMENT.arrived_at = now()
CRM notification sent: "Crew arrived at site and working"
```

### 7.2 Manual Check-in

If the GPS-triggered prompt does not appear (GPS inaccuracy, indoor site):
```
"Not showing arrival prompt?"
[Manual check-in →]
Confirmation: "Confirm you are at [site name]? [Yes · No]"
```

Manual check-ins are logged with source = MANUAL in CREW_ASSIGNMENT for audit purposes.

---

## 8. Screen 5 — Isolation Workflow

This is the most safety-critical screen in the entire system. It is designed with three principles specific to isolation:

**Sequential enforcement:** Steps cannot be confirmed out of order. Step 4 confirm button is greyed until steps 1, 2, and 3 are confirmed. The sequence_number on ISOLATION_POINT records enforces this.

**Deliberate confirmation:** Each step requires a separate tap, not a swipe. The confirmation requires the worker to actively acknowledge — not accidentally confirm by scrolling past.

**No back-track after gate:** Once pre_work_confirmed is set to true (the isolation pre-work gate), the app does not allow un-confirming individual steps. To reverse, the worker must contact the dispatcher and an override process is initiated.

### 8.1 Pre-Work Isolation Screen

```
[Red header]
Isolation — pre-work confirmation
PTW-[reference] · Issued by: [name] · [time]
Shutdown window: [start] – [end] · [X hrs remaining]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Step 1: [Action description]
        [Sub-description — asset, location, type]
        [Confirm step 1 checkbox — unchecked initially]

Step 2: [Action description]
        [confirm step 2 — inactive until step 1 confirmed]

...

Step N: [Action description]
        [confirm step N]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Equipment status confirmation]
  Equipment (CB-104): [select: ENERGIZED | DE-ENERGIZED | ISOLATED | EARTHED]
  Source (Feeder F-14): [select: ENERGIZED | OPEN | LOCKED_OUT | TAGGED_OUT]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Confirm pre-work complete] — greyed until all steps + status confirmed
```

The "Confirm pre-work complete" button uses a deliberate two-tap pattern:
```
Tap 1: Button shows "Hold to confirm" with a 3-second press-and-hold requirement
Hold: Progress ring fills over 3 seconds (prevents accidental activation)
Release at completion: Confirmation dialog:
  "Confirm isolation complete?
   Equipment: DE-ENERGIZED
   Source: LOCKED_OUT
   All [N] steps confirmed.
   You are confirming it is safe to begin work.
   [CONFIRM — WORK MAY PROCEED]    [Cancel]"
```

The confirmation dialog repeats the status values so the worker reads them one more time before committing.

On confirmation:
```
ISOLATION_RECORD.pre_work_confirmed = true
ISOLATION_RECORD.equipment_status_pre = (selected value)
ISOLATION_RECORD.equipment_status_pre_confirmed_by = current worker_id
ISOLATION_RECORD.equipment_status_pre_confirmed_at = now()
ISOLATION_RECORD.source_status_pre = (selected value)
All ISOLATION_POINT.pre_action_confirmed = true
TASK.status = IN_PROGRESS
Dispatcher board: PTW badge changes to "Work authorised" (green)
```

### 8.2 Shutdown Window Timer

During work execution, a persistent timer is visible at the top of every screen:

```
[amber background]
Shutdown window: [start] – [end]
[X hrs Y min remaining] [progress bar depleting]
```

When 30 minutes remain:
```
[red background]
Shutdown window: 30 MIN REMAINING
[Complete work and confirm re-energization by end time]
```

If the window end time passes without post-work confirmation:
```
[red pulsing banner]
SHUTDOWN WINDOW EXCEEDED
Contact dispatcher immediately — isolation overrun
[Call dispatcher] [Record overrun reason]
```

The overrun reason is mandatory before the worker can proceed to post-work confirmation.

### 8.3 Post-Work Re-energization Screen

Accessed after the worker declares work complete. This is the post-work gate.

```
[Green header]
Post-work — re-energization confirmation
Work declared complete by [worker name] at [time]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Area clear checklist:
  [ ] All personnel clear of work zone
  [ ] All tools and equipment removed
  [ ] Temporary earths removed
  [ ] Area safe for re-energization

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Safety officer sign-off — if required by task type]
  Safety officer: [name]
  [Request sign-off] → Push notification sent to safety officer

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Reversal of isolation points (in reverse sequence):
Step 4 reversal: Remove earth — Bus Bar E-7
  [Confirm reversed]
Step 3 reversal: Close DS-104A disconnect
  [Confirm reversed — inactive until step 4 done]
...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Re-energization status confirmation:
  Equipment (CB-104):  [select: ENERGIZED · other options]
  Source (Feeder F-14): [select: ENERGIZED · CLOSED · other]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Confirm re-energization complete] — two-tap hold, same as pre-work
```

On confirmation:
```
ISOLATION_RECORD.post_work_confirmed = true
ISOLATION_RECORD.equipment_status_post = ENERGIZED
ISOLATION_RECORD.equipment_status_post_confirmed_by = worker_id
ISOLATION_RECORD.equipment_status_post_confirmed_at = now()
ISOLATION_RECORD.source_status_post = CLOSED (or selected)
All ISOLATION_POINT.post_action_confirmed = true
ASSET.energization_status = ENERGIZED
ASSET.energization_updated_at = now()
ASSET.energization_updated_by = worker_id
```

After post-work confirmation, the app transitions to the Work Execution / Completion screen.

---

## 9. Screen 6 — Work Execution and Completion

### 9.1 Test Record Capture

For scheduled testing tasks (transformer test, relay calibration, battery inspection), the app presents a structured form defined by the DOCUMENT_TEMPLATE.fields_schema for that task type.

**Transformer test record example:**

```
Test record — Transformer T1 33kV
Template: XFMR_DIELECTRIC_TEST v2.1
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Transformer oil BDV test
  HV-LV: [          ] kV    (min. 30 kV)
  LV-Earth: [        ] kV
  HV-Earth: [         ] kV

Turns ratio test
  Phase A: [          ]    (expected: 33.0)
  Phase B: [          ]
  Phase C: [          ]

Oil temperature at test: [    ] °C
Test instrument reference: [              ]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Overall result:
  [PASS] [CONDITIONAL PASS] [FAIL] [PENDING REVIEW]

Next test due:
  [date picker — default: today + standard interval]
  [Override if shorter interval needed]

Notes:
  [                                    ]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Submit test record] — disabled until all mandatory fields filled
```

Mandatory fields are marked with a red asterisk. The Submit button remains disabled until all mandatory fields contain valid values. This enforces the documentation gate — the task cannot be marked complete without the certificate data.

### 9.2 Parts Consumed Capture

After completing work, the worker records what was actually used:

```
Parts consumed — WO-2026-04-0091
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Cable fault kit: used [1] of [1] on van ✓
Jointing kit: used [1] of [2] on van
Heat shrink: used [2m] of [4m] on van

[Add part not on list +]
  → opens parts catalog search
  → worker selects from PART records
  → source: VAN or DEPOT (if depot detour was made)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Confirm parts]
```

On confirmation:
```
PARTS_CONSUMED records created for each part used
VAN_STOCK.quantity_on_hand decremented automatically
If stock falls below minimum_level → restock alert triggered
```

### 9.3 Service History Entry

The worker records what was found and what was done:

```
Service record — 33kV CB-104 Ramdas Sub Bay 3
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Findings:
  [Cable joint failure · water ingress · insulation breakdown]

Action taken:
  [XLPE cable joint replaced · section 14-B · new joint fitted and tested]

Asset condition before: [auto-filled from last history] 7/10
Asset condition after:
  [1] [2] [3] [4] [5] [6] [7*] [8] [9] [10]
  *7 selected

Recommended next action:
  [Inspect joint within 3 months — previous failure pattern]
```

This entry creates a SERVICE_HISTORY record linked to the asset — not the task — and updates ASSET.condition_score.

### 9.4 Task Completion and Submission

Final submission screen:

```
Ready to close task
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Checklist:
  ✓ Re-energization confirmed
  ✓ Test record completed (if applicable)
  ✓ Parts consumed recorded
  ✓ Service record completed
  ✓ Photos attached (optional but encouraged)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Submit and close task]
```

If any mandatory item is incomplete, the incomplete items are highlighted and the submit button is disabled with tooltip explaining what is missing.

On submission:
```
TASK.status = COMPLETED
TASK.completed_at = now()
TASK.actual_duration_minutes = (completed_at − started_at) in minutes
SERVICE_HISTORY record created
TEST_RECORD created (if applicable)
PARTS_CONSUMED records created
VAN_STOCK decremented
ASSET.condition_score updated
MAINTENANCE_SCHEDULE.next_due_date updated (schedule engine callback)
CRM notification: "Supply restored · issue resolved · [feedback link]"
WORKER.status → AVAILABLE (automatic)
App returns to task list
```

### 9.5 Photo Attachment

Available on any screen in the work execution flow:

```
[Camera icon — top right corner of any execution screen]
→ Opens device camera or photo library
→ Photo tagged with task_id, timestamp, and geolocation
→ Uploaded with task submission (or stored locally until connectivity restored)
→ Stored in SERVICE_HISTORY or TEST_RECORD (configurable per task type)
```

Photos are not mandatory by default. Task types that mandate photos (infrastructure inspection, storm damage) are configured with a required photo count in DOCUMENT_TEMPLATE.fields_schema.

---

## 10. Screen 7 — Asset History Lookup

Accessible from the History tab in the bottom navigation. Allows the worker to look up any asset's service history — not just the current task's asset.

Use cases:
- Worker notices a nearby asset with visible damage — looks it up before raising a reactive task
- Worker wants to check when the last test was done on an asset before starting
- Supervisor reviewing a recent fault pattern

```
[Search bar: asset code or name]

Recent assets (last 10 accessed):
[Asset 1 — last event date]
[Asset 2 — last event date]
...

On selecting an asset:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Asset name and code]
[Asset type · Voltage level · Zone]
Condition: [score]/10 · Status: [OPERATIONAL]
Energization: [ENERGIZED] as of [date time]

Service history (most recent first):
[same format as task detail history panel]

Open tasks on this asset:
[if any — shown with WO reference and status]

[Raise new reactive task on this asset →]
```

The "Raise new reactive task" button opens a pre-populated task creation form with the asset already linked, which is then submitted to the dispatcher board as a NEW task.

---

## 11. Offline Capability

### 11.1 What Works Offline

| Feature | Offline behaviour |
|---|---|
| View assigned tasks | Available — cached from last sync |
| View task detail | Available — cached from last sync |
| View asset service history | Available for recently accessed assets |
| Site check-in | Records locally, syncs on reconnection |
| Isolation step confirmation | Records locally, syncs on reconnection |
| Pre-work and post-work gate confirmation | Records locally, syncs — critical safety data gets priority sync |
| Test record data entry | Records locally, submits on reconnection |
| Parts consumed entry | Records locally, syncs |
| Task completion submission | Records locally, submits on reconnection |
| Worker status update | Records locally, syncs |

### 11.2 What Requires Connectivity

| Feature | Online only |
|---|---|
| Receiving new task assignments | Requires push notification or sync |
| Real-time dispatcher communication | Requires connectivity |
| Asset history lookup for non-cached assets | Requires connectivity |
| Supply plan access | Requires connectivity |
| Map/navigation | Requires connectivity for live tiles |

### 11.3 Sync Priority on Reconnection

When connectivity is restored, data syncs in this priority order:
1. Safety-critical records: pre_work_confirmed, post_work_confirmed, energization_status
2. Task status updates: started_at, completed_at
3. Test records and service history
4. Parts consumed and stock updates
5. Photos and attachments

---

## 12. Notifications and Alerts

### 12.1 Push Notifications (App in Background)

| Event | Notification text |
|---|---|
| New task assigned | "New P1 task — HT cable fault Sector 14. Tap to view." |
| Task reassigned away from worker | "WO-0091 has been reassigned. Check your task list." |
| Cluster window approaching | "Planned shutdown in 30 min — CLU-2026-04-018. Proceed to site." |
| Shutdown window near end | "30 min remaining on shutdown window WO-0091. Complete work." |
| Safety officer sign-off requested | "R.Kumar requests your safety officer sign-off on WO-0091." |

### 12.2 In-App Alerts

| Situation | In-app alert |
|---|---|
| SLA at risk | Persistent red banner at top of task detail |
| Shutdown window overrun | Pulsing red banner across full screen |
| Connectivity lost | Persistent offline bar |
| Sync conflict detected | Yellow banner: "Data conflict — tap to review" |
| Van stock below minimum | Badge on task if parts are now insufficient mid-task |

---

## 13. Safety Officer Co-Dispatch Flow

When two workers are assigned as a co-dispatch pair (lead + safety officer), the app handles both roles:

**Lead worker (main technician):**
- Receives task on their app as normal
- Isolation workflow shows: "Safety officer [name] required on site before pre-work confirmation"
- Pre-work confirmation button is inactive until safety officer sign-off is received

**Safety officer:**
- Receives separate push notification: "Safety officer requested for WO-0091 by R.Kumar"
- Opens app → sees a sign-off request card at top of task list
- Views the isolation steps that have been completed
- If satisfied: taps "Approve isolation — safe to proceed"
- Their worker_id is recorded in ISOLATION_RECORD.safety_officer_signoff_by

The pre-work gate on the lead worker's app activates only after the safety officer sign-off is received. Both workers must be on site and actively confirming before work can begin.

---

## 14. Non-Functional Requirements

| Requirement | Specification |
|---|---|
| App response time | All screens load within 2 seconds on 4G |
| Minimum screen resolution | 1080×1920 (standard smartphone) |
| Touch target size | Minimum 44×44px for all interactive elements |
| Font size | Minimum 14px body text; 18px for critical confirmations |
| Contrast ratio | Minimum 4.5:1 for all text (WCAG AA) |
| Outdoor visibility | High-contrast mode automatically activates in bright light (device ambient sensor) |
| Offline storage | Minimum 7 days of task data and 30 asset histories cached |
| Background sync | Priority sync of safety-critical data within 30 seconds of connectivity restoration |
| Battery impact | App designed for all-day use on a single charge (background processes minimized) |
| Platform support | Android 10+ · iOS 14+ |
| App update | Forced update mechanism for safety-critical patches |
| Session persistence | Worker remains logged in across shifts; re-authentication only on first daily use |

---

## 15. Data Flows to and from the Mobile App

| App action | Data written | Tables updated |
|---|---|---|
| Accept task | dispatched_at | CREW_ASSIGNMENT |
| Site check-in | arrived_at, TASK.started_at, WORKER.status = ON_TASK | CREW_ASSIGNMENT, TASK, WORKER |
| Isolation step confirmed | pre_action_confirmed | ISOLATION_POINT |
| Pre-work gate confirmed | pre_work_confirmed, equipment_status_pre, source_status_pre | ISOLATION_RECORD, ASSET.energization_status |
| Test reading entered | test_readings (JSONB) | TEST_RECORD |
| Parts consumed entered | quantity_used, source | PARTS_CONSUMED, VAN_STOCK |
| Service record entered | findings, action_taken, condition_score_after | SERVICE_HISTORY, ASSET.condition_score |
| Post-work gate confirmed | post_work_confirmed, equipment_status_post, source_status_post | ISOLATION_RECORD, ASSET.energization_status |
| Task completed | TASK.status = COMPLETED, actual_duration_minutes | TASK |
| Status changed | WORKER.status | WORKER |
| Location updated | current_lat, current_lng, location_updated_at | WORKER |

---

## 16. The Complete Field Worker Journey — P1 Fault Example

```
14:23 — Push notification: "New P1 task assigned — HT cable fault Sector 14"
14:23 — Worker opens app — task appears at top of list with pulsing border
14:24 — Worker opens task detail — reads PTW warning, fault description,
         sees last fault on this cable section was 18 Feb 2026
         confirms cable fault kit on van ✓
14:25 — Taps "Accept task + start navigation"
        CREW_ASSIGNMENT.dispatched_at recorded
        CRM: "Crew en route — ETA 11 min"
14:35 — Arrives within 100m of site — app shows check-in prompt
        Reads history summary: last fault same section 18 Feb 2026
        Taps CHECK-IN
        TASK.started_at = 14:35
        WORKER.status = ON_TASK
        CRM: "Crew on site and working"
14:35 — Isolation workflow screen opens automatically
        Worker works through 5 isolation steps with HV authority
        Confirms each step with individual taps
        Confirms equipment DE-ENERGIZED, source LOCKED_OUT
        Press-and-holds "Confirm pre-work complete" for 3 seconds
        Reads confirmation dialog: "Equipment: DE-ENERGIZED · Source: LOCKED_OUT"
        Taps CONFIRM — WORK MAY PROCEED
        ISOLATION_RECORD.pre_work_confirmed = true
        Dispatcher board: PTW badge turns green "Work authorised"
        CRM: consumer already received "Crew on site" — no additional notification
14:36 — Work begins — app shows shutdown window timer: 3h 54m remaining
        Worker carries out cable joint replacement
        Shutdown timer in amber background throughout
17:00 — Work complete — area cleared
        Worker opens post-work screen
        Confirms area clear checklist (4 items)
        Works through reversal steps in reverse sequence
        Selects equipment: ENERGIZED · source: CLOSED
        Press-and-holds "Confirm re-energization complete"
        Confirms in dialog
        ISOLATION_RECORD.post_work_confirmed = true
        ASSET.energization_status = ENERGIZED
        CRM notification triggered: supply restored to 847 consumers
17:02 — Work execution screen: enters service record
        Findings: "Cable joint failure · water ingress · insulation breakdown"
        Action: "XLPE cable joint replaced · section 14-B"
        Condition after: 7/10
        Recommendation: "Inspect joint within 3 months"
17:04 — Parts consumed: cable fault kit 1, jointing kit 1, heat shrink 2m
        VAN_STOCK auto-decremented
17:06 — All mandatory items complete — taps "Submit and close task"
        TASK.status = COMPLETED
        actual_duration_minutes = 151 (14:35 → 17:06)
        CRM feedback request sent to 47 complainants
        WORKER.status = AVAILABLE
        Maintenance schedule updated: next inspection in 91 days
17:06 — App returns to task list — next assigned task (relay test) visible
```

---

## 17. Key Design Decisions

**Why the 3-second press-and-hold for isolation confirmation?**
An accidental tap on the pre-work or post-work gate confirmation is a safety failure. A standard button tap can be triggered by a slip, a fall, or a pocket activation. The press-and-hold creates a deliberate, sustained action that cannot be triggered accidentally. The 3-second duration is long enough to prevent accidents but short enough not to impede a worker who knows what they are doing.

**Why is service history shown before arrival, not after?**
Context influences behaviour. A worker who knows this transformer has faulted three times in six months will inspect it more carefully, allocate more time, and approach differently than a worker seeing it as a first occurrence. The history must be visible before the worker has already committed to a diagnosis.

**Why does offline functionality include safety-critical records?**
Connectivity cannot be assumed in utility field work — substations, underground cable routes, and rural areas all have poor coverage. If the app required connectivity for isolation confirmation, a worker in a low-signal area would either be unable to proceed or would find workarounds. Offline capability for safety records means the system doesn't create pressure to bypass the workflow.

**Why is worker status automatically set to ON_TASK at check-in?**
Manual status updates create lag and are forgotten. The matching engine reads WORKER.status to build the eligible pool — a worker who has checked in at site but not updated their status manually may receive a second assignment. Automatic status management ensures the system's view matches reality.

**Why show the supply plan on the worker's app?**
The worker executing supply restoration switching needs to know the exact sequence. The SUPPLY_PATH_SEGMENT records are the operational instruction — the worker does not make decisions, they follow a pre-validated sequence that the Supply Optimization Engine produced. Showing the plan on the worker's app means the sequence is always with the person executing it.

---

*End of Field Worker Mobile App Interface Design Specification*
*FSM System · Interface 2 of 3 · Version 1.0 · April 2026*
