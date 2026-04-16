# Dispatcher Board — Interface Design Specification
## Field Service Management System · Utility Field Response Optimization
**Document:** Interface 1 of 3
**Version:** 1.0 · April 2026

---

## 1. Purpose and Role

The Dispatcher Board is the operational nerve centre of the FSM system. It is the primary interface through which all field task assignments are made — not by the system automatically, but by a trained dispatcher supported by the system's ranked suggestions, real-time location data, SLA timers, and safety status indicators.

The board is designed around one governing principle: a trained dispatcher must be able to assign a P1 emergency task within 60 seconds of it appearing in the queue. Every design decision — information hierarchy, panel layout, interaction pattern, colour usage — flows from this requirement.

**Primary user:** Dispatcher (operational role)
**Secondary user:** Shift supervisor (read-only monitoring)
**Device:** Desktop web browser, 1920×1080 minimum resolution
**Session type:** Continuous — dispatcher is on the board for an entire shift

---

## 2. Screen Layout — Five Panels

The board uses a fixed three-column layout with a top status bar and bottom status bar. The layout does not change based on what is selected — all five panels are always visible.

```
┌─────────────────────────────────────────────────────────────┐
│ TOP STATUS BAR — shift info · task counts · SLA % · weather │
├──────────────┬──────────────────────┬───────────────────────┤
│              │                      │                       │
│  TASK QUEUE  │       MAP VIEW       │  TASK DETAIL +        │
│  LEFT PANEL  │    CENTRE PANEL      │  SUGGESTIONS          │
│  260px       │    flexible          │  RIGHT PANEL 320px    │
│              │                      │                       │
│              │                      │                       │
├──────────────┴──────────────────────┴───────────────────────┤
│ BOTTOM STATUS BAR — crew counts · SLA risk · engine status  │
└─────────────────────────────────────────────────────────────┘
```

Total board height: 640px minimum (fits standard 1080p monitor with browser chrome).
Left panel: fixed 260px. Right panel: fixed 320px. Centre map: fills remaining width.

---

## 3. Panel 1 — Top Status Bar

### 3.1 Content

Always visible, never scrolled. Contains:

| Element | Content | Update frequency |
|---|---|---|
| System identity | Utility name · division · dispatcher name · shift hours | Static per session |
| Current time and date | Live clock updating every second | Real-time |
| P1 counter | Count of open P1 tasks | Real-time |
| P2 counter | Count of open P2 tasks | Real-time |
| P3 counter | Count of open P3 tasks (may be large) | Real-time |
| Available crew | Count of workers with status = AVAILABLE | Real-time |
| SLA compliance | % of open tasks within SLA for this shift | Per minute |

### 3.2 Weather Advisory Bar

Appears immediately below the top bar when any WEATHER_EVENT with severity HIGH or EXTREME is active in any zone served by this dispatcher:

```
[amber dot] Weather advisory — [event_type] ([severity]) in [zone_code]
· [N] outdoor tasks on hold · Indoor tasks unaffected
```

Rules:
- One bar per active weather event in scope
- If multiple weather events: bars stack vertically
- Disappears automatically when WEATHER_EVENT.ended_at is populated
- Clicking the bar filters the task queue to show only weather-held tasks

### 3.3 Colour Coding

| Element | Normal | At risk | Critical |
|---|---|---|---|
| P1 counter | Red dot always | — | — |
| P2 counter | Amber dot always | — | — |
| SLA compliance | Green text | Amber if < 90% | Red if < 85% |
| Available crew | Normal text | Amber if < 5 | Red if < 3 |

---

## 4. Panel 2 — Task Queue (Left Panel)

### 4.1 Sort Order

Tasks are always sorted in this priority order:
1. Priority class ascending (P1 first, P4 last)
2. Within same priority: SLA time remaining ascending (most urgent first)
3. NEW and UNASSIGNED tasks shown above ASSIGNED tasks within same priority

### 4.2 Task Row Structure

Each row has four lines:

```
Line 1: [Task name — truncated to 35 chars]          [P1/P2/P3/P4 badge]
Line 2: [WO reference] · [Zone code]
Line 3: [Status or consumer count]   [SLA bar — 100% to 0%]   [Time remaining]
```

SLA bar colour transitions:
- > 60% remaining → green fill
- 20–60% remaining → amber fill
- < 20% remaining → red fill
- Breached (negative) → red fill, row background turns pale red

### 4.3 Filter Bar

Single-click filters above the queue:

| Filter | Behaviour |
|---|---|
| All | Show all tasks in scope |
| P1 | Show only P1 tasks |
| P2 | Show only P2 tasks |
| Unassigned | Show only NEW status tasks |
| My zone | Show only tasks in dispatcher's assigned zone(s) |
| Weather hold | Show only ON_HOLD (weather) tasks |
| Clustering | Show only tasks with clubbing_opportunity_flagged = true |

Multiple filters can be combined (AND logic). Task count in panel header updates to reflect filtered view.

### 4.4 Row Selection Behaviour

Clicking a task row:
- Highlights the row with green left border
- Loads task detail in the right panel
- Highlights the task's location pin on the centre map
- Draws a dashed route line from the top-ranked suggested worker to the task site
- Does NOT auto-assign or trigger any workflow action

### 4.5 Real-Time Queue Updates

When a new task is created:
- Row is inserted at the correct sort position
- Row animates in (fade-in) to draw dispatcher attention
- P1 new tasks also trigger the Emergency Alert state (Section 9.1)

When a task is completed:
- Row fades out and disappears from queue
- Task count decrements

When SLA time changes category (e.g. crosses from green to amber):
- Row SLA bar colour transitions without interrupting the dispatcher

---

## 5. Panel 3 — Map (Centre Panel)

### 5.1 Content

The map shows the geographic distribution of field workers and open tasks. It is a situational awareness tool, not a navigation tool.

**Worker pins:** Circular pins with worker initials, coloured by current status:
- Green: AVAILABLE
- Blue: ON_TASK (actively working)
- Purple: STANDBY or ON_CALL
- Grey: location data stale (> staleness threshold)

**Task site pins:** Small filled circles, coloured by priority:
- Red: P1
- Amber: P2
- Blue: P3/P4

**Selected task highlighting:**
- When a task is selected in the left panel, that task pin pulses gently
- A dashed green line appears from the top-ranked suggested worker to the task site
- This line updates when suggestions are refreshed

**Cluster areas:** When an approved TASK_CLUSTER exists with an active shutdown window, a light shaded polygon covers the affected zone with the cluster reference label.

### 5.2 Pin Interaction

Clicking a worker pin:
```
Shows tooltip:
  [Worker name] · [Grade]
  Status: [AVAILABLE / ON_TASK]
  Current task: [WO reference or "None"]
  Queue depth: [N tasks]
  Location updated: [X min ago]
  [Assign to selected task →] button (only if a task is selected)
```

Clicking a task pin:
- Selects that task in the left queue panel
- Loads that task's detail in the right panel
- Equivalent to clicking the row in the queue

### 5.3 Map Controls

Minimal controls only:
- Zoom in/out buttons (keyboard: +/-)
- Fit all button: zooms to show all workers and tasks in viewport
- Layer toggle: show/hide worker pins, show/hide task pins independently
- No street-level navigation, no routing display (Phase 1)

### 5.4 Phase 2 Map Upgrades

- Actual road-network routing lines replacing straight dashed lines
- Zone boundary overlays coloured by SAIFI performance
- Alternative supply source locations shown as switchable layers
- Real-time worker GPS position updates (replacing shift-start check-in)

---

## 6. Panel 4 — Task Detail and Suggestions (Right Panel)

The right panel is the decision space. When no task is selected, it shows a shift summary. When a task is selected, it shows the full task detail in seven sections.

### 6.1 Task Header

```
[Full task name]
[WO reference] · [Zone code] · [Feeder/asset identifier]

Asset:           [Asset name and code]
Consumer impact: [N consumers affected]
Est. duration:   [X hrs (complexity_band)]
Supply plan:     [Plan reference and status, or "Not required"]
Last fault here: [Date and event_type from SERVICE_HISTORY, most recent]
Last test here:  [Date and test result if applicable]
```

The last service history entries are fetched on task selection. They require no separate navigation — the dispatcher sees the asset's recent history immediately without leaving the board.

### 6.2 SLA Banner

Full-width coloured banner showing:

```
[P1 red / P2 amber / P3 blue background]
SLA deadline: [date and time] · [X hrs Y min remaining]
```

For compliance-date SLAs (scheduled tasks):
```
Compliance deadline: [date] · [N days remaining]
```

The SLA banner is the highest-priority information element in the right panel. It is never obscured by scroll position — it is the first visible element when the right panel is loaded.

### 6.3 PTW Status Badge (Panel Header)

The PTW badge in the panel header changes state through the isolation workflow:

| State | Badge text | Colour |
|---|---|---|
| Task type does not require PTW | Not shown | — |
| PTW required, not yet issued | "PTW required" | Amber |
| PTW issued, pre-work gate open | "Pre-work pending" | Amber |
| Pre-work confirmed (isolation complete) | "Work authorised" | Green |
| Post-work pending (task complete, not yet re-energized) | "Re-energization pending" | Amber |
| Post-work confirmed | "PTW surrendered" | Green |

### 6.4 Clustering Opportunity Alert

When TASK.clubbing_opportunity_flagged = true, a dismissible banner appears:

```
[Purple background]
Clustering opportunity — [N] related tasks in [zone] within [N] days.
Saves [N] outage events if approved. [Review cluster →]
```

Clicking the banner opens the Cluster Planning View (Section 9.3) in a side panel over the right panel. The main task detail remains accessible via a back navigation.

This banner is informational — it does not block or interrupt the assignment workflow. The dispatcher may assign the current task first, then review clustering separately.

### 6.5 Supply Plan Readiness Indicator

When a supply plan exists for the task's outage event:

```
Supply plan: ALT-NHV3-01 ready (Rank 1) · 0.018 MW losses · 74% margin [green]
```

When no plan exists or plan is pending LDC approval:

```
Supply plan: Awaiting LDC approval — ALT-NHV3-03 [amber]
```

When no feasible plan was found:

```
Supply plan: NO VIABLE ALTERNATIVE — manual coordination required [red]
[View infeasibility reason →]
```

### 6.6 Suggested Crew — Three Ranked Cards

Each suggestion card contains:

**Card header row:**
- Rank badge (1 / 2 / 3) with colour coding (green / blue / grey)
- Worker name and whether they are a safety officer
- Worker grade
- ETA to site (travel time in minutes)

**Skill and parts tags row:**
- Each mandatory skill as a blue tag
- Parts status: "All parts on van" (green) / "Depot pickup: [part] +[X] min" (amber) / "Parts unavailable" (red)
- For standby workers: "Call to activate" (purple)

**Plain-language reason (one sentence):**
The reason is generated from the matching engine's output and explains the top factors in plain language — not score numbers. Example: "Nearest certified HT lineman with HV authority and cable fault kit. 1 task completing in ~40 min. Acts as safety officer — single crew sufficient."

**Action buttons:**
- "Assign [name]" (rank 1 card only, green) — one-click assignment for top suggestion
- "Assign crew pair" (for co-dispatch cards) — assigns both workers simultaneously
- "Activate + assign" (for standby workers) — creates phone contact task + assigns
- "Map" (all cards) — zooms map to show that worker's current location

**Score transparency bars:**
Compact bars showing relative proximity and workload scores without numbers. The dispatcher sees signal, not formula.

### 6.7 Co-dispatch Card

When the matching engine determines a safety officer pairing is required, the rank 1 card shows both workers:

```
[Rank 1 badge]
[Lead worker name] + [Safety Officer name]
[Lead grade] + [SO role]
ETAs shown separately: Lead: [X] min · SO: [Y] min

"Co-dispatch required — [lead name] is not a safety officer.
 [SO name] available [N km away]. Both must be on site before work starts."

[Assign crew pair] [Map both]
```

### 6.8 Override Section

Below the three suggestion cards, always visible:

```
Override — assign a different worker
[Worker unavailable] [Local knowledge] [Special equipment]
[Prior relationship] [Skill clarification] [Other reason]

[These are clickable reason code buttons — one must be selected]
[Once selected: worker search field appears + notes field]
[Assign (override) button activates only after reason code selected]
```

The reason code buttons use light grey background until one is selected. Selection highlights it. The assign button for override path is disabled until a reason code is selected — this is enforced in the UI and cannot be bypassed.

---

## 7. Panel 5 — Bottom Status Bar

One line of live operational metrics, always visible:

```
Available crew: 14 | On task: 9 | Standby: 3 | SLA at risk: 2 |
Weather holds: 6 | Active clusters: 1 | Supply plans ready: 3 |
Last engine run: 14:20
```

| Item | Colour condition |
|---|---|
| Available crew | Amber if < 5, red if < 3 |
| SLA at risk | Red if > 0 |
| Weather holds | Amber if > 0 |
| Last engine run | Amber if > 30 min ago, red if > 60 min ago |

---

## 8. Task Assignment Workflow — Step by Step

### 8.1 Standard Assignment (Top Suggestion)

```
Step 1: Dispatcher selects task in left panel
Step 2: Right panel loads task detail and three suggestions
Step 3: Dispatcher reviews SLA, supply plan, and suggestion card 1
Step 4: Clicks "Assign [name]"
Step 5: Confirmation modal appears:
        "Assign Ramesh Kumar to WO-2026-04-0091?
         [ETA: 11 min] [Parts: all on van]
         [PTW required — isolation record will be created]
         [Confirm assignment] [Cancel]"
Step 6: Dispatcher confirms
Step 7: System:
        → Creates CREW_ASSIGNMENT record (suggestion_rank = 1)
        → Updates TASK.status = ASSIGNED
        → Updates TASK.assigned_at = now()
        → Creates ISOLATION_RECORD stub (if PTW required)
        → Triggers CRM notification: "Crew assigned — ETA [time]"
        → Sends mobile app notification to worker
Step 8: Task row in left panel shows worker name and ASSIGNED status
Step 9: Map shows route line to task site
```

Target: Steps 1–7 completable in under 60 seconds for P1 tasks.

### 8.2 Override Assignment

```
Step 1–3: Same as standard
Step 4: Dispatcher clicks preferred override reason code
Step 5: Worker search field appears — dispatcher types name or employee ID
Step 6: Matching workers appear in dropdown (filtered by workforce pool and status)
        Workers who fail mandatory skill check shown with red warning
Step 7: Dispatcher selects worker (can select despite warning — override authority)
Step 8: Clicks "Assign (override)"
Step 9: If selected worker fails mandatory skill check:
        Secondary confirmation:
        "This worker lacks: [missing skills]. Assign anyway?
         This override will be logged. Confirm reason: [reason code]
         [Confirm — I accept responsibility] [Cancel]"
Step 10: Assignment proceeds with:
         CREW_ASSIGNMENT.suggestion_rank = NULL
         CREW_ASSIGNMENT.override_reason_code = selected code
         CREW_ASSIGNMENT.override_notes = text entered
```

### 8.3 Co-dispatch Assignment

```
Step 1–3: Same as standard
Step 4: Card shows two workers — lead and safety officer
Step 5: Dispatcher clicks "Assign crew pair"
Step 6: Confirmation shows both names, both ETAs
Step 7: Confirms
Step 8: System creates two CREW_ASSIGNMENT records:
        Record 1: lead worker, role = LEAD
        Record 2: safety officer, role = SAFETY_OFFICER
        Both assigned_at = same timestamp
        CRM notification includes both names and later ETA of the pair
```

---

## 9. Five Specialized Screen States

### 9.1 P1 Emergency Alert State

Trigger: New TASK with priority_class = P1 created.

```
Full-width red banner animates in at top of screen (below top bar):

[red background]
NEW P1 EMERGENCY — [task type] · [location] · [zone]
WO-[reference] · [N] consumers affected · SLA: [X hrs] remaining
[View and assign →]

The task is simultaneously inserted at the top of the task queue.
The banner cannot be dismissed without either:
  (a) Clicking "View and assign" (loads task in right panel)
  (b) Explicit acknowledgment: "Acknowledge — will handle shortly"
      (requires click of confirmation; logs acknowledgment with timestamp)

If the dispatcher is mid-assignment of another task:
  Banner appears but does not interrupt the current action
  Flashing indicator on the P1 counter in top bar
```

### 9.2 Isolation Workflow State

Trigger: Task status = IN_PROGRESS and ISOLATION_RECORD exists for this task.

The right panel replaces the suggestion cards with the isolation workflow:

```
ISOLATION RECORD — [PTW reference]
Issued by: [HV authority name] at [time]
Shutdown window: [start] to [end] · [X hrs remaining]

PRE-WORK CONFIRMATION
[Section header: Equipment status]
  Equipment (33kV CB-104): [DE_ENERGIZED — confirmed by R.Kumar at 14:35] ✓
  Source (Feeder F-14):   [LOCKED_OUT — confirmed by R.Kumar at 14:36] ✓

ISOLATION POINTS
  1. Open CB-104 at Ramdas Sub Bay 3 [CONFIRMED 14:34 — R.Kumar] ✓
  2. Open DS-104A (disconnect switch) [CONFIRMED 14:35 — R.Kumar] ✓
  3. Apply Earth at Bus Bar E-7 [CONFIRMED 14:36 — R.Kumar] ✓
  4. Lock out DS-104A [CONFIRMED 14:36 — R.Kumar] ✓

Pre-work gate: OPEN ✓ — Work may proceed

[Mark work complete →] [only active after post-work actions confirmed]
```

The "Mark work complete" button is greyed with tooltip: "Post-work confirmation required — field crew must confirm re-energization before task can be closed" until post_work_confirmed = true.

### 9.3 Cluster Planning State

Trigger: Dispatcher clicks "Review cluster" from the clustering opportunity banner, or navigates to the Clusters tab.

The right panel expands to show:

```
PROPOSED CLUSTER — CLU-2026-04-018
Lead task: [WO reference] — transformer test Substation 3 · due 25 Apr
Dependent tasks: 2 additional tasks

CONSUMER IMPACT ASSESSMENT
  Affected zone: NORTH-HV-3
  Total consumers affected: 847
  Alternative supply available: 847 (ALT-NHV3-01 ready)
  Consumers effectively interrupted: 0

RELIABILITY BENEFIT
  Outage events saved: 2 (3 tasks → 1 outage event instead of 3)
  SAIDI saving: 12,400 consumer-minutes vs individual shutdowns
  Planned window: 22 Apr 07:00 — 15:00 (8 hrs)

NOTIFICATION STATUS
  Advance notices (72 hr): Not yet sent — requires approval first
  Critical consumers: None in affected zone

SUPPLY PLAN
  Rank 1: ALT-NHV3-01 · 0.018 MW losses · 74% margin [green]
  Rank 2: ALT-NHV3-03 · conditional (protection change required) [amber]

[Approve cluster and begin notification →] [Reject — keep as individual tasks]
```

Approval triggers the 72-hour advance notice cascade immediately.

### 9.4 Supply Plan Review State

Trigger: Dispatcher clicks the supply plan indicator in the task detail header, or an outage event requires plan approval.

```
SUPPLY SWITCHING PLAN — [plan reference]
Outage: [outage reference] · [zone] · [affected consumers]

RANK 1 — PRIMARY (ALT-NHV3-01)
  Supply type: Feeder backfeed via Tie Switch TS-7
  Load to transfer: 5.8 MVA
  Estimated losses: 0.018 MW
  Margin utilisation: 74% (available margin: 8.2 MVA)
  Switching steps: 4 steps · est. 12 minutes
  Source loading: 62% (SCADA data 8 min old — fresh)
  [Approve this plan →]

RANK 2 — FALLBACK (ALT-NHV3-03)
  [CONDITIONAL — Protection change required]
  [Details collapsed — click to expand]

RANK 3 — TERTIARY
  [Not available — only 2 feasible candidates found]

[Approve Rank 1 plan →]  [View switching sequence]  [Request LDC approval]
```

### 9.5 Overrun Alert State

Trigger: Clustering engine detects cluster window will be exceeded by > 15 minutes.

```
[Amber banner over right panel]
OVERRUN ALERT — CLU-2026-04-018
Work not complete. Window ends in [X] min.
Revised estimated restoration: [time]
[N] consumers still without supply.

[Send overrun notification to consumers →]
  Message preview: "We apologise — maintenance is taking longer than expected.
                   Revised restoration time: [time]."

[Extend window — document reason] [Escalate to supervisor]
```

---

## 10. Keyboard Shortcuts

Dispatchers on continuous shifts benefit from keyboard navigation:

| Shortcut | Action |
|---|---|
| F1 | Select first P1 task in queue |
| F2 | Select first unassigned task in queue |
| Enter | Assign top-ranked suggestion for selected task |
| Ctrl + M | Focus map, enable pan with arrow keys |
| Esc | Dismiss any banner or confirmation modal |
| Ctrl + F | Focus task queue search |
| Ctrl + 1/2/3 | Assign suggestion rank 1/2/3 for selected task |
| Space | Acknowledge P1 emergency alert |

---

## 11. Non-Functional Requirements for the Dispatcher Board

| Requirement | Specification |
|---|---|
| Initial page load | < 3 seconds on standard utility WAN |
| Live data refresh | Every 30 seconds for task queue and worker locations |
| P1 alert latency | < 5 seconds from task creation to alert appearing |
| Matching engine suggestions | < 5 seconds for P1 tasks |
| Map tile load | < 2 seconds for zone-level map tiles |
| Accessibility | WCAG 2.1 AA — all status information conveyed via text as well as colour |
| Browser support | Chrome 120+, Edge 120+, Firefox 118+ |
| Screen resolution | Minimum 1366×768; optimised for 1920×1080 |
| Session persistence | No data loss if browser tab is refreshed mid-shift |
| Concurrent dispatchers | Supports 10 simultaneous dispatcher sessions per division |

---

## 12. Data Feeds Powering the Board

| Panel element | Data source | Update mechanism |
|---|---|---|
| Task queue | TASK + SLA_TRACKER | WebSocket push on status change |
| Worker pins | WORKER (lat/lng + status) | Poll every 30 seconds |
| Suggestions | CREW_ASSIGNMENT proposals from Matching Engine | Push on engine run |
| Isolation status | ISOLATION_RECORD + ISOLATION_POINT | WebSocket push from mobile app |
| Supply plan | SUPPLY_SWITCHING_PLAN + SOURCE_OPERATING_STATE | Push on plan creation |
| Cluster alerts | TASK_CLUSTER (PROPOSED status) | Push on engine detection |
| Weather bar | WEATHER_EVENT (active, HIGH+) | Push on event creation |
| Bottom bar counts | WORKER status counts + SLA_TRACKER | Poll every 60 seconds |
| Service history | SERVICE_HISTORY (most recent 3 events for selected asset) | Fetch on task selection |

---

## 13. Dispatcher Workflow — A Complete P1 Scenario

```
14:23:00 — P1 task created (smart meter last-gasp signal + field report)
14:23:02 — P1 emergency banner animates in on dispatcher board
14:23:04 — Matching engine completes — 3 suggestions ready
14:23:05 — Dispatcher sees alert, clicks "View and assign"
14:23:10 — Right panel shows: task detail, SLA 1h 24m, supply plan ready,
           Ramesh Kumar ranked #1, 11 min ETA, all parts on van
14:23:25 — Dispatcher reviews map — Ramesh Kumar pin is nearest to task site
14:23:30 — Dispatcher clicks "Assign Ramesh Kumar"
14:23:32 — Confirmation modal: ETA 11 min, PTW required, parts OK
14:23:35 — Dispatcher confirms
14:23:35 — CREW_ASSIGNMENT created, TASK.status = ASSIGNED
14:23:36 — CRM notification sent to 47 complainants: "Crew assigned, ETA 11 min"
14:23:36 — Mobile app push notification sent to Ramesh Kumar
14:23:37 — Task row in queue shows "ASSIGNED · R.Kumar"
14:23:40 — Supply plan ALT-NHV3-01 auto-linked to outage event
14:34:00 — Ramesh Kumar checks in on site (mobile app)
14:34:01 — TASK.started_at recorded — isolation workflow begins on mobile
14:36:00 — Isolation pre-work gate confirmed (mobile app)
14:36:01 — Dispatcher board: PTW badge changes to "Work authorised" (green)
14:36:02 — CRM: "Crew on site and working"
— Work in progress —
17:10:00 — Post-work gate confirmed, re-energization confirmed
17:10:01 — TASK.status = COMPLETED
17:10:02 — CRM: "Supply restored — issue resolved"
17:10:03 — OUTAGE_EVENT.outage_end = 17:10:00
17:10:04 — CONSUMER_OUTAGE_RECORD updated for all 847 consumers
17:10:05 — Feedback request sent: "How did we do? Rate your experience"
```

Total clock time from task creation to assignment confirmation: 35 seconds.

---

## 14. Key Design Decisions

**Why three suggestions rather than one?**
The algorithm has imperfect information. Three options preserve dispatcher judgment while making the default clear. In practice, rank 1 is used 70–80% of the time — rising as the system learns from overrides.

**Why is the map in the centre rather than the largest panel?**
Spatial awareness is important but not the primary decision-making surface. The task queue and suggestion cards carry more information per square centimetre for operational decisions. The map supports the queue; it does not replace it.

**Why must the override reason code be mandatory?**
An assignment without a reason is not a learning signal — it is noise. Mandatory reason codes create a structured dataset for the Phase 3 model to train on. Free text alone is not analyzable at scale.

**Why does the P1 alert not interrupt mid-assignment work?**
Interrupting a dispatcher mid-assignment of a P2 task could cause an error in that assignment. The P1 alert banner is prominent and persistent but does not block the current workflow. The dispatcher finishes their current action, then responds to the P1.

**Why is service history visible without navigation?**
A transformer that has faulted three times in six months needs different handling than a first-time fault. If the dispatcher must navigate away from the assignment screen to check history, they often won't. Surfacing the last three service events in the task detail panel ensures this context is used.

**Why does the PTW status badge live in the panel header?**
Isolation status is safety-critical. It must be visible at all times when the task detail panel is showing, regardless of scroll position. A badge in the panel header is always visible, never scrolled off.

---

*End of Dispatcher Board Interface Design Specification*
*FSM System · Interface 1 of 3 · Version 1.0 · April 2026*
