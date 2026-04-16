# Management Dashboard — Interface Design Specification
## Field Service Management System · Utility Field Response Optimization
**Document:** Interface 3 of 3
**Version:** 1.0 · April 2026

---

## 1. Purpose and Role

The Management Dashboard is the reporting and oversight layer of the FSM system. Where the Dispatcher Board is about what is happening right now and the Mobile App is about executing the work, the dashboard is about understanding how well the operation has been performing, where risks exist, and what decisions need to be made for planned work ahead.

Three users interact with the dashboard with different primary questions:

**Operations manager** — How did we perform today and this month? Are SLAs being met? Are any zones or priorities at risk? What does crew utilization look like? Are there any infrastructure vulnerabilities I need to escalate?

**Planning officer** — Which cluster proposals need my approval and by when? Which planned shutdowns are coming and are consumers notified? Are there N-1 vulnerabilities I need to prioritize for network reinforcement?

**Senior management / regulatory** — What are our SAIFI, SAIDI, CAIFI, and CAIDI indices for this reporting period? Are we improving versus last period? Are all regulatory submissions on track?

The dashboard must answer all three sets of questions without requiring navigation away from a single screen. The governing principle: all critical indicators visible on one screen without scrolling.

**Primary users:** Operations manager · Planning officer
**Secondary users:** Senior management · Network planning team
**Device:** Desktop web browser, 1920×1080 minimum
**Session type:** Review sessions — not continuous like the Dispatcher Board

---

## 2. Screen Layout — Six Regions

```
┌─────────────────────────────────────────────────────────────────┐
│ HEADER — title · period selector · zone filter · export         │
├─────┬──────┬──────┬──────┬───────┬────────┬─────────────────────┤
│     │      │      │      │       │        │                     │
│SAIFI│SAIDI │CAIFI │CAIDI │  SLA  │SAIFI   │  CREW UTIL %        │
│     │      │      │      │  %    │SAVED   │                     │
├─────┴──────┴──────┴──────┴───────┴────────┴─────────────────────┤
│ RELIABILITY TREND (6-month)          │ ZONE RELIABILITY STATUS  │
├──────────────────────────────────────┴──────────────────────────┤
│  SLA COMPLIANCE        │  CLUSTER PLANNING   │  ACTIVE OUTAGES  │
│  by priority + trend   │  upcoming shutdowns │  current events  │
├────────────────────────┴─────────────────────┴──────────────────┤
│  CREW UTILIZATION (top 10)      │  OVERRIDE ANALYSIS + REG STATUS│
└─────────────────────────────────┴─────────────────────────────────┘
```

---

## 3. Header Bar

### 3.1 Identity and Context

```
[System name] — [Division name]
Last updated: [time] · [date]
```

The "last updated" timestamp matters — a manager reviewing the dashboard at 16:00 should know whether the data includes events from the last 30 minutes or was last refreshed at 10:00.

### 3.2 Period Selector

Three options with a dropdown:

| Option | Data scope |
|---|---|
| Current month (default) | First day of current month to now |
| Previous quarter | Q1/Q2/Q3/Q4 of the relevant year |
| Year to date | 1 Jan to today |

When the period changes, all metrics, charts, and tables update simultaneously. The selected period is shown in the header for clarity.

### 3.3 Zone Filter

A dropdown allowing focus on one zone or all zones:
- All zones (default)
- Individual zone (by zone code)

Zone filtering affects all panels except Regulatory Submissions (which is division-wide) and Active Outage Events (which shows all regardless of filter).

### 3.4 Export

A single "Export PDF" button generates a formatted management report containing all six dashboard panels for the selected period and zone. This is the document used for management review meetings and regulatory submission cover documents.

---

## 4. Region 1 — KPI Metric Cards

Seven cards in a single row. These are the first thing every user reads and must be immediately interpretable without explanation.

### 4.1 The Seven Metrics

| Card | Value shown | Comparison shown | Source |
|---|---|---|---|
| SAIFI | Current period value (e.g. 0.42) | % change vs previous equivalent period | RELIABILITY_SNAPSHOT.saifi |
| SAIDI | Minutes (e.g. 38.7) | % change vs previous period | RELIABILITY_SNAPSHOT.saidi |
| CAIFI | Interruptions per interrupted consumer | % change | RELIABILITY_SNAPSHOT.caifi |
| CAIDI | Minutes per interruption | % change | RELIABILITY_SNAPSHOT.caidi |
| SLA compliance | Overall % across all priorities | % points change vs previous period | SLA_TRACKER computations |
| SAIFI saved by clustering | Count of outage events avoided | Absolute count this period | SUM(TASK_CLUSTER.saifi_events_saved) |
| Crew utilization | Average utilization % | Month-to-date | Computed from CREW_ASSIGNMENT and SHIFT_SCHEDULE |

### 4.2 Colour Rules for Metric Cards

For reliability indices (SAIFI, SAIDI, CAIFI, CAIDI) — lower is better:
- Comparison text green: metric improved (decreased) vs previous period
- Comparison text red: metric worsened (increased) vs previous period
- Comparison text neutral grey: no significant change (< 2% difference)

For SLA compliance — higher is better:
- Green if ≥ 90%
- Amber if 80–89%
- Red if < 80%

For crew utilization — target band 65–80%:
- Green if 65–80% (optimal)
- Amber if < 65% (underutilized) or 81–90% (high but manageable)
- Red if > 90% (overloaded — SLA risk)

---

## 5. Region 2 — Reliability Trend Chart

### 5.1 Chart Type and Data

A combination chart showing the six most recent periods (months or quarters depending on period selector):
- Bar series: clustering events count per period (shows when clustering was deployed and ramped up — the backdrop to SAIFI improvement)
- Line series 1: SAIFI — left axis, scale 0.0–1.5
- Line series 2: SAIDI in minutes — right axis, scale 0–120

The visual relationship between the clustering deployment bar and the declining SAIFI/SAIDI lines is the core story this chart tells. Managers should be able to see: "clustering was deployed in month X, and SAIFI started improving immediately."

### 5.2 Chart Behaviour

- Period selector changes the x-axis resolution: Monthly → 6 data points; Quarterly → 4 data points; YTD → one point per month elapsed
- Zone filter changes all three series to reflect only the selected zone's data
- Clicking any data point: shows a tooltip with the exact OUTAGE_EVENT count, planned vs unplanned split, and SAIFI saving from clustering for that period
- A horizontal reference line can be added at the regulatory target SAIFI value (configurable parameter)

### 5.3 Zone Reliability Status Panel

Adjacent to the trend chart, a compact zone-by-zone reliability summary:

```
[Zone name]     [SAIFI bar — visual fill]     [SAIFI value]     [N-1 security badge]
```

N-1 security badge states (from nightly SUPPLY_OPTIMIZATION_RUN contingency pre-computation):
- "N-1 secure" (green): viable alternative supply exists for all assets in zone
- "N-1 marginal" (amber): alternative exists but with low margin — needs planning attention
- "N-1 vulnerable" (red): one or more assets have no viable N-1 alternative — infrastructure risk

Clicking "N-1 vulnerable" badge opens a drill-down panel showing which specific assets have no alternative, the infeasibility reason from SUPPLY_OPTIMIZATION_RUN, and any existing planning alert tasks created for network reinforcement.

---

## 6. Region 3 — Three Side-by-Side Panels

### 6.1 SLA Compliance by Priority

Four rows — one per priority class (P1, P2, P3, P4) — each showing:
```
[P1/P2/P3/P4 label]  [Progress bar]  [%]  [Completed / Total tasks]
```

Bar colour:
- Green fill if ≥ target threshold for that priority
- Amber fill if within 5% below target
- Red fill if more than 5% below target

Target thresholds (configurable):
- P1: ≥ 95%
- P2: ≥ 90%
- P3: ≥ 95%
- P4: ≥ 100%

Below the four bars: a compact 6-period trend line chart (one line per priority class) showing SLA compliance movement over time. This answers the question "are we improving or declining?" without requiring a separate screen.

### 6.2 Cluster Planning Overview

Shows all TASK_CLUSTER records in the current and near-future planning horizon that require action:

```
[Cluster reference]                     [Status badge]
[Zone · Planned date · Consumer count]
[SAIDI saving vs individual shutdowns]
```

Status badges:
- Proposed (purple): needs planner approval — most urgent
- Approved (green): approved, notifications being sent
- Executing (blue): shutdown window currently active
- Completed (grey): historical record

Sort order: Proposed first (by approval deadline), then Approved by window date, then Executing.

The approval deadline is derived from planned_window_start − 72 hours (the regulatory notice cutoff). Any proposed cluster whose approval deadline has passed is highlighted in red — advance notices cannot be sent in time.

Clicking any cluster row opens the full cluster detail: all member tasks, consumer impact assessment, supply plan status, notification status for the affected consumers.

### 6.3 Active Outage Events

Shows all OUTAGE_EVENT records with outage_end IS NULL (currently active) plus events completed in the last 24 hours:

```
[Outage reference]              [Planned / Unplanned badge]
[Zone · Consumer count · Started time · Supply status]
```

Supply status indicators:
- "Alt supply: active": SUPPLY_SWITCHING_PLAN has been executed
- "Crew on site": field crew checked in, working
- "Awaiting crew": crew assigned but not yet arrived
- "Completed [time]": outage resolved

Cumulative summary below the list: total events this period, planned vs unplanned split.

---

## 7. Region 4 — Crew Utilization and Accountability

### 7.1 Crew Utilization Chart

Horizontal bar chart showing the top 10 workers by utilization percentage for the selected period. Utilization computed as:

```
utilization_pct =
  SUM(actual_duration_minutes for completed tasks)
  ÷ SUM(planned_shift_minutes for shifts worked)
  × 100
```

Bar colour by utilization range:
- Green (65–80%): optimal operating range
- Blue (81–90%): high but productive
- Amber (> 90%): risk of fatigue and SLA pressure
- Grey (< 65%): underutilized — investigation warranted

Below the chart: the override rate for the period (% of assignments where the top suggestion was not used), with a link to the detailed override analysis.

### 7.2 Override Analysis

A compact panel showing the distribution of override reason codes for the period:

```
[Reason code]  [Bar — relative count]  [Count]
```

This panel surfaces the most important signal for algorithm improvement. Dominant override reasons point to specific gaps:

| Dominant reason | What it signals |
|---|---|
| Local knowledge (> 50% of overrides) | Proximity model missing local access or road knowledge |
| Worker unavailable | Worker status updates are delayed — GPS or check-in lag |
| Special equipment | Van stock data is stale or equipment not catalogued correctly |
| Skill clarification | Skills registry is incomplete for affected task types |

The operations manager can click any reason code bar to see the individual override records — which workers were bypassed, for which task types, and what the dispatcher's notes said.

---

## 8. Region 5 — Regulatory Compliance

### 8.1 Submission Status Table

All regulatory submission obligations for the division, showing:

```
[Report type]     [Reporting period]     [Status badge]
```

Status badges:
- Submitted (green): successfully submitted to regulatory portal
- Pending (amber): within submission window, not yet submitted
- Due [date] (amber): upcoming deadline
- Overdue (red): deadline passed without submission

Report types tracked:
- Monthly reliability snapshot (SAIFI/SAIDI/CAIFI/CAIDI per zone)
- Annual test certificates (by asset type and count)
- PTW records (quarterly batch submission)
- Advance notice compliance evidence (monthly — CRM_NOTIFICATION regulatory records)

### 8.2 Advance Notice Compliance

Three summary cards showing the three most important advance notice metrics:

```
[% of planned shutdowns with ≥ 72-hr notice sent]
[Count of planned shutdowns / total this period]
[Count of compliance breaches — should always be 0]
```

A compliance breach is defined as a planned shutdown that proceeded without regulatory advance notice being sent. This should be zero at all times. A non-zero breach count is a regulatory reporting incident and triggers an automatic management alert.

---

## 9. Drill-Down Panels

Several dashboard elements open drill-down side panels without navigating away from the main dashboard view.

### 9.1 Zone N-1 Vulnerability Drill-Down

Triggered by clicking an N-1 vulnerable zone badge. Shows:
```
Zone: RURAL-MV-1 — N-1 Vulnerability Report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Vulnerable assets (3):
  33kV Line LN-141 (Murad Nagar to Modinagar)
    → No feasible N-1 alternative
    → Infeasibility reason: All candidates exceed thermal limit
    → Planning alert: PAL-2026-03-007 (Open — assigned to S.Gupta)

  33kV Transformer T5 at Modinagar Sub
    → No feasible N-1 alternative
    → Infeasibility reason: Source voltage outside limits
    → Planning alert: PAL-2026-03-012 (Open — review due 30 Apr)

N-2 Status: Not computed for this zone (only HV assets per config)

Last contingency run: 16 Apr 2026 01:30
Recommendation: Network reinforcement study required before monsoon season
[View planning alerts →]  [Request new study →]
```

### 9.2 Cluster Detail Drill-Down

Triggered by clicking any cluster row. Shows the complete cluster record as designed in the Dispatcher Board's Cluster Planning View:
- All member tasks with their individual SLA deadlines
- Consumer impact assessment
- Supply plan status
- Notification log (how many consumers notified, which channels, delivery status)
- Approval / rejection controls for proposed clusters

### 9.3 Outage Event Drill-Down

Triggered by clicking any outage event row. Shows:
- Full OUTAGE_EVENT record
- CONSUMER_OUTAGE_RECORD count and category breakdown
- Supply switching plan status and execution record
- CRM notifications sent (advance notice, restoration)
- SAIDI contribution calculation
- Link to the underlying TASK records

### 9.4 Override Record Drill-Down

Triggered by clicking an override reason code bar. Shows the individual CREW_ASSIGNMENT records:
```
Date      WO Reference    Bypassed worker   Assigned worker   Override notes
14 Apr    WO-2026-04-0089  V.Singh (#1)      R.Kumar           "Local road closure — R.Kumar knows bypass"
12 Apr    WO-2026-04-0081  A.Mehta (#1)      P.Sharma          "Sharma has prior history with this transformer"
```

This list is the primary input for the Phase 3 ML model training review and for the weekly algorithm performance discussion.

---

## 10. Period-over-Period Comparisons

For all key metrics, the dashboard shows the direction and magnitude of change versus the most relevant comparison period:

| Period selected | Comparison | Label shown |
|---|---|---|
| Current month | Previous month | "vs last month" |
| Previous quarter | Same quarter previous year | "vs Q[N] 2025" |
| Year to date | Same period previous year | "vs YTD 2025" |

For reliability indices, the direction arrows use the correct semantic:
- SAIFI ▼ 18% → green arrow + green text (decrease is improvement)
- SAIFI ▲ 8% → red arrow + red text (increase is deterioration)
- SLA ▲ 3pp → green arrow + green text (increase is improvement)
- Crew utilization ▲ 5pp → amber text (depends on whether it crossed the threshold bands)

---

## 11. Management Report (PDF Export)

The PDF export generates a formatted management report containing:

**Cover page:** Division name, reporting period, prepared date, prepared by (logged-in user name)

**Section 1 — Executive summary:** All seven KPI cards in tabular form with period-on-period comparison and brief automated commentary (e.g. "SAIFI improved 18% month-on-month, consistent with the clustering engine reducing planned outage events from 36 to 22").

**Section 2 — Reliability indices:** The 6-period trend chart, zone reliability table with N-1 status, and a plain-language paragraph summarizing the trend.

**Section 3 — SLA performance:** SLA compliance table by priority class, breach list (if any), and the 6-period trend chart.

**Section 4 — Planned operations:** Cluster planning table showing all completed clusters for the period, SAIFI events saved, and consumer notification compliance record.

**Section 5 — Operational performance:** Crew utilization chart, override analysis summary, and any infrastructure vulnerabilities flagged.

**Section 6 — Regulatory compliance:** Submission status table and advance notice compliance record.

**Appendix:** Active planning alerts (N-1 vulnerabilities), pending cluster approvals, pending test certificate submissions.

---

## 12. Data Sources for Each Dashboard Element

| Element | Primary tables | Computation |
|---|---|---|
| SAIFI, SAIDI, CAIFI, CAIDI | RELIABILITY_SNAPSHOT | Pre-computed by reliability engine; snapshot per zone per period |
| SAIFI saved by clustering | TASK_CLUSTER.saifi_events_saved | SUM for the period |
| SLA compliance | SLA_TRACKER | COUNT(is_breached=false) ÷ COUNT(all) per priority class |
| Crew utilization | CREW_ASSIGNMENT + SHIFT_SCHEDULE | actual_duration ÷ shift_duration per worker |
| Override rate | CREW_ASSIGNMENT | COUNT(suggestion_rank IS NULL) ÷ COUNT(all) |
| Reliability trend | RELIABILITY_SNAPSHOT (monthly records) | Direct read |
| Zone N-1 status | SUPPLY_OPTIMIZATION_RUN (CONTINGENCY_N1) | Most recent nightly run per zone |
| Cluster planning | TASK_CLUSTER | Current + future records by status |
| Active outages | OUTAGE_EVENT | WHERE outage_end IS NULL |
| Regulatory submissions | TEST_RECORD + RELIABILITY_SNAPSHOT + ISOLATION_RECORD | Submission_status fields |
| Advance notice compliance | CRM_NOTIFICATION | WHERE regulatory_record=true AND notification_type=PLANNED_OUTAGE_ADVANCE |

---

## 13. Non-Functional Requirements

| Requirement | Specification |
|---|---|
| Dashboard load time | < 5 seconds for default view (current month, all zones) |
| Chart rendering | Charts visible within 3 seconds of page load |
| Data freshness | Metrics updated every 5 minutes; charts every 15 minutes |
| Period filter response | Switching periods updates all panels within 2 seconds |
| PDF export | Generated within 30 seconds for any period |
| Browser support | Chrome 120+, Edge 120+, Firefox 118+ |
| Screen resolution | Minimum 1440×900; optimised for 1920×1080 |
| Accessibility | WCAG 2.1 AA; all chart data available as accessible table alternative |
| Concurrent users | Supports 20 simultaneous dashboard sessions |
| Print layout | All panels print correctly on A4 landscape |

---

## 14. Key Design Decisions

**Why is SAIFI saved by clustering shown as a separate metric card?**
The clustering engine's value is not visible in the SAIFI index alone — a SAIFI of 0.42 tells you the current performance but not how much worse it would have been without clustering. "14 outage events avoided" makes the engine's direct contribution tangible for management and justifies the planning investment in cluster approval.

**Why does crew utilization target 65–80% rather than maximizing to 100%?**
A workforce at 100% utilization has no capacity for P1 emergencies. A storm that generates 10 simultaneous fault tasks requires immediate surge capacity. A team running at 71% average has meaningful buffer. Managing to 65–80% is a deliberate operational decision, not an efficiency gap.

**Why does the dashboard show both SAIFI and CAIFI rather than just SAIFI?**
SAIFI captures how many interruptions occurred per consumer in total — including unplanned and planned. CAIFI captures how many interruptions affected consumers who experienced at least one interruption. A high CAIFI with a lower SAIFI means the same consumers are being repeatedly affected — which is both a customer satisfaction problem and a signal of infrastructure deterioration in specific areas. The two indices together tell a different story than either alone.

**Why is the N-1 vulnerability status shown in the operational dashboard rather than only in a separate planning tool?**
N-1 vulnerability is an operational risk that the operations manager needs to know about before an outage occurs. If RURAL-MV-1 has no viable N-1 alternative and a fault occurs there, the operations manager needs to know that supply restoration may not be possible, so they can prepare consumer communication and escalate to network planning. Confining this information to a planning tool means it is only seen when someone goes looking for it.

**Why is the override analysis on the management dashboard rather than just in the dispatch system?**
The operations manager is responsible for the overall quality of the system's recommendations. If 73% of overrides are for "local knowledge," that is a systematic gap the operations manager needs to address — either by improving the algorithm's data inputs or by conducting a training session to ensure dispatchers are logging the knowledge correctly. Without the override analysis at management level, this signal gets lost.

**Why include advance notice compliance on the main dashboard rather than only in a regulatory section?**
A compliance breach in advance notices (a planned outage proceeding without regulatory notice) is both a regulatory incident and an operational failure that the operations manager must know about immediately, not discover during a compliance audit. Placing it on the main dashboard alongside operational metrics ensures it is reviewed daily.

---

## 15. Dashboard Evolution — Phase 2 to Phase 3

| Feature | Phase 2 | Phase 3 |
|---|---|---|
| SAIFI/SAIDI data | Computed from completed tasks and outage events | Real-time continuous computation as events close |
| Zone N-1 status | Nightly pre-computation batch | Near-real-time update on every SCADA reading change |
| Crew utilization | Computed from shift records end-of-day | Rolling real-time as tasks complete |
| Reliability trend | Monthly RELIABILITY_SNAPSHOT records | Continuous — any period selectable by date range |
| ML prediction accuracy | Not shown | New card: predicted vs actual duration variance |
| Proactive maintenance rate | Not shown | New card: % of fault tasks that were preceded by scheduled inspection |
| Asset condition heat map | Not shown | Zone map showing average asset condition_score by zone |
| Consumer satisfaction | Not shown | Average CRM feedback score per zone per period |

---

*End of Management Dashboard Interface Design Specification*
*FSM System · Interface 3 of 3 · Version 1.0 · April 2026*
