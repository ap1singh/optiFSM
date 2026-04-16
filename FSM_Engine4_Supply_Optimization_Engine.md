# Supply Optimization Engine — Design Specification
## Field Service Management System · Utility Field Response Optimization
**Document:** Engine 4 of 4
**Version:** 1.0 · April 2026

---

## 1. Purpose and Role

The Supply Optimization Engine selects the best alternative supply path to restore or maintain consumer supply during planned and unplanned outages. It operates at the boundary of power systems engineering and operational research — its decisions have direct safety consequences, and a wrong selection can overload a feeder, cause cascading faults, or leave critical consumers without supply.

The engine is conservative by design. It never selects an unapproved path. It never uses stale data without flagging it. It never silently fails — if no viable alternative exists, it says so explicitly and requires a documented human override.

**What it is:** An optimization engine that selects from pre-cataloged supply paths using real-time operating margin data.
**What it does:** Filters candidate paths against hard physical constraints, estimates losses, scores candidates on loss minimization and reliability, produces ranked supply switching plans.
**What it does not do:** Execute switching, contact consumers, manage protection settings, or replace a full power systems load flow (Phase 2 uses approximations; Phase 3 integrates an external solver).

---

## 2. Foundational Design Principle — A Priori Catalog

Every permissible alternative supply path must be pre-defined in ALT_SUPPLY_SOURCE by the network planning team before it can be used. This is not an operational restriction — it is a safety imperative.

Improvising supply paths under time pressure during an outage leads to:
- Selection of paths not studied for voltage or thermal limits
- Protection coordination failures (wrong relay settings for back-fed network)
- Unexpected overloads on the alternative source
- Cascading faults that compound the original outage

The pre-cataloged approach converts the question "can we use this path?" from a real-time judgment call into a pre-validated lookup. The optimization engine's job is selection from known-safe options, not discovery of new ones.

**Rule enforced by system:** OUTAGE_EVENT.status cannot advance to APPROVED unless SUPPLY_SWITCHING_PLAN records exist, or the dispatcher records a documented override with reason code.

---

## 3. Three Operating Modes

| Mode | Trigger | Scope | Target time | Phase |
|---|---|---|---|---|
| Mode 1 — Single outage | OUTAGE_EVENT created or cluster approved | One zone or asset | < 10 seconds | Phase 2 |
| Mode 2 — Multi-outage simultaneous | 2+ active OUTAGE_EVENT records | Multiple zones | < 60 seconds | Phase 2/3 |
| Mode 3 — Contingency pre-computation | Nightly at 01:30 · post network change | All HV/MV assets | Complete by 03:00 | Phase 2 |

All three modes execute the same four-step common pipeline. What differs is the trigger condition, the assignment problem scope, and the algorithm.

---

## 4. The Common Four-Step Pipeline

### Step 1 — Candidate Filter (Hard Constraints)

Six eliminations applied to every ALT_SUPPLY_SOURCE candidate:

**Elimination 1 — Expired or unapproved path:**
```
If ALT_SUPPLY_SOURCE.approved_date IS NULL → eliminate
If ALT_SUPPLY_SOURCE.valid_until < today → eliminate
Flag: "Path [ref] expired — network re-study required"
```

**Elimination 2 — Source unavailable:**
```
Get latest SOURCE_OPERATING_STATE for source_asset_id

Staleness check:
  age = now() − recorded_at
  If age > scada_stale_threshold_min (30 min):
    → Do NOT eliminate
    → Flag STALE — reduce reliability_score by 20%
    → Annotate plan: "Margin data [age] minutes old — verify before executing"
    → Require dispatcher confirmation before plan approved

  If age > manual_stale_threshold_min (240 min):
    → Block plan approval — require manual margin entry

If availability_flag = UNAVAILABLE → eliminate
If source.outage_planned = true AND overlaps with outage window → eliminate
```

**Elimination 3 — Insufficient thermal margin:**
```
If available_margin_mva < load_to_transfer_mva → eliminate

For Mode 2: check accumulated load across all already-assigned sources
  remaining_margin[j] = available_margin[j] − SUM(already assigned loads to j)
  If remaining_margin[j] < load_to_transfer[i] → eliminate for this assignment
```

**Elimination 4 — Voltage constraint:**
```
If SOURCE_OPERATING_STATE.voltage_within_limits = false → eliminate
If ALT_SUPPLY_SOURCE.voltage_level ≠ target_zone.voltage_level → eliminate
```

**Elimination 5 — Protection change required, no engineer available:**
```
If requires_protection_change = true:
  Check for available WORKER with PROTECTION_RELAY_ENGINEER skill
    in source_zone_id AND status = AVAILABLE

  If available: include candidate, annotate plan with protection change requirement
  If not available: flag CONDITIONAL_NO_PROTECTION_ENGINEER
    → Include at lower rank with warning to dispatcher
```

**Elimination 6 — Load dispatch center approval required:**
```
If requires_operator_approval = true:
  → Do NOT eliminate — include in plan
  → Set SUPPLY_SWITCHING_PLAN.status = AWAITING_LDC_APPROVAL
  → Plan cannot execute until LDC engineer confirms
```

Surviving candidates proceed to loss estimation.

### Step 2 — Loss Estimation

For each feasible candidate, estimate additional active power losses:

**Step 2a — Path current:**
```
For three-phase HV/MV path:
  I = (load_to_transfer_mva × 1,000,000) / (√3 × voltage_level_V)
  Result in amperes (A)

Example:
  load = 5 MVA, voltage = 33 kV
  I = 5,000,000 / (1.732 × 33,000) = 87.5 A
```

**Step 2b — I²R losses:**
```
R_total = resistance_ohm_per_km × path_length_km
losses_W = 3 × I² × R_total   (three-phase)
losses_MW = losses_W / 1,000,000

Example:
  I = 87.5 A, R = 0.185 Ω/km, path = 4.2 km
  R_total = 0.777 Ω
  losses = 3 × 87.5² × 0.777 = 17,885 W ≈ 0.018 MW
```

**Step 2c — Voltage drop estimate:**
```
ΔV_pu ≈ (load_MVA × (R_total × cosφ + X_total × sinφ))
         / (voltage_kV² × 1000)

Where:
  X_total = reactance_ohm_per_km × path_length_km
  cosφ = assumed power factor (default 0.85)
  sinφ = √(1 − cosφ²)

If ΔV_pu > max_voltage_drop_pu (default 0.05 = 5%):
  → Flag as MARGINAL_VOLTAGE — annotate plan with warning
  → Do not eliminate — dispatcher informed
```

**Phase note:** This is a simplified steady-state approximation suitable for Phase 2 planning. Phase 3 integration with an external load flow solver (PSS/E, PowerFactory, OpenDSS) will replace this with exact computed values written back into SUPPLY_OPTIMIZATION_RUN.actual_losses_mw.

### Step 3 — Composite Scoring

```
For each feasible candidate c:

Loss score (lower losses = higher score):
  loss_score(c) = 1 − (estimated_losses_mw(c) / MAX(estimated_losses across all candidates))
  Normalized: lowest-loss candidate scores 1.0

Reliability score (historical source availability):
  reliability_score(c) = ALT_SUPPLY_SOURCE.reliability_score(c) / 100
  This % is maintained from historical fault and availability records

Margin headroom score (safety buffer):
  margin_score(c) = available_margin_mva(c) / MAX(available_margin across all candidates)
  Rewards sources running well below capacity — better able to absorb unexpected demand

Composite score:
  composite_score(c) =
    (weight_loss_minimization × loss_score(c))
  + (weight_reliability × reliability_score(c))
  + (0.10 × margin_score(c))

Where:
  weight_loss_minimization + weight_reliability = 0.90 (configurable)
  margin component = 0.10 (fixed — represents a permanent physical safety buffer)

Default weights: loss = 0.45, reliability = 0.45, margin = 0.10 (fixed)
```

**Weight configuration guidance:**

| Operational condition | Recommended weights |
|---|---|
| Normal planned maintenance | loss 0.45 · reliability 0.45 · margin 0.10 |
| Emergency fault — fast restoration priority | loss 0.30 · reliability 0.60 · margin 0.10 |
| Light load period — efficiency focus | loss 0.60 · reliability 0.30 · margin 0.10 |
| System stressed — headroom critical | loss 0.25 · reliability 0.40 · margin 0.25* |

*Note: increasing margin weight requires reducing loss and/or reliability weights proportionally to maintain sum = 0.90 + 0.10. The margin component may be increased beyond 0.10 in exceptional circumstances by operator configuration.

### Step 4 — Assign and Produce Ranked Plan

```
Sort candidates by composite_score descending

Write SUPPLY_SWITCHING_PLAN records:
  Rank 1: highest score → primary selection
  Rank 2: second highest → first fallback (if primary unavailable at execution)
  Rank 3: third highest → second fallback

For each plan record set:
  load_to_transfer_mva = estimated load
  estimated_losses_mw = from Step 2
  margin_utilisation_pct = (load_to_transfer / available_margin) × 100
  status = PLANNED (or AWAITING_LDC_APPROVAL if approval required)

If fewer than 3 feasible candidates:
  Write available ranks only
  Flag: "Limited alternatives — only [N] options available for [zone]"

If zero feasible candidates:
  solution_found = false on SUPPLY_OPTIMIZATION_RUN
  infeasibility_reason = most common elimination reason from Step 1
  consumers_not_restorable = consumers_affected
  Alert management: "CRITICAL — no viable alternative supply for [zone/outage]"
  Dispatcher must record documented override with reason before outage approved
```

---

## 5. Mode 1 — Single Outage Optimization

### 5.1 Trigger and Scope

Triggered by:
- New OUTAGE_EVENT created (unplanned fault)
- TASK_CLUSTER.status → APPROVED (planned shutdown)
- Manual trigger by dispatcher or planner

Scope: one zone or one asset with its dependent network.

### 5.2 Load to Transfer Estimate

The engine needs to know how much load will move to the alternative supply. Three sources in order of preference:

```
Priority 1 — Smart meter interval data (if MDMS integrated, Phase 2+):
  SUM of last 15-min interval reads for all consumers in affected zone
  Most accurate — actual real-time demand

Priority 2 — SCADA feeder data:
  SOURCE_OPERATING_STATE.current_load_mva for the affected feeder
  Use directly where available — second most accurate

Priority 3 — Zone average estimate (Phase 1 fallback):
  NETWORK_ZONE.total_consumers × average_demand_kva_per_consumer
  (average_demand_kva_per_consumer configured per zone type)
  Least accurate — flag as ESTIMATED to dispatcher

Flag in SUPPLY_OPTIMIZATION_RUN.solution_notes:
  "Load estimate source: [SMART_METER / SCADA / ZONE_AVERAGE]"
```

### 5.3 Complete Mode 1 Run Example

```
Scenario: Emergency fault on Feeder F-14, Zone NORTH-HV-3
  Estimated load: 6.1 MVA
  Available candidates: ALT-NHV3-01, ALT-NHV3-02, ALT-NHV3-03

Step 1 — Filter:
  ALT-NHV3-01:
    approved_date = 2025-01-15 ✓
    valid_until = 2026-12-31 ✓
    age of STATE reading = 8 min (fresh) ✓
    availability_flag = AVAILABLE ✓
    available_margin = 8.2 MVA > 6.1 MVA ✓
    voltage OK ✓
    PASS → feasible

  ALT-NHV3-02:
    availability_flag = MARGINAL (loading 81%)
    available_margin = 3.1 MVA < 6.1 MVA
    ELIMINATE (thermal constraint)

  ALT-NHV3-03:
    requires_protection_change = true
    No protection engineer available in zone
    CONDITIONAL_NO_PROTECTION_ENGINEER
    INCLUDE with flag

Step 2 — Loss estimation:
  ALT-NHV3-01: path 4.2 km, R=0.0185 Ω/km, I=107A → losses = 0.018 MW
  ALT-NHV3-03: path 7.8 km, R=0.0210 Ω/km, I=107A → losses = 0.038 MW

Step 3 — Scoring (weights: loss 0.45, reliability 0.45, margin 0.10):
  ALT-NHV3-01:
    loss_score = 1.0 (lowest losses)
    reliability_score = 0.96 (96% historical availability)
    margin_score = 1.0 (8.2 MVA margin, highest)
    composite = 0.45 + 0.432 + 0.10 = 0.982

  ALT-NHV3-03:
    loss_score = 0.53
    reliability_score = 0.88
    margin_score = 0.41 (3.8 MVA remaining after load transfer)
    composite = 0.24 + 0.396 + 0.041 = 0.677

Step 4 — Ranked plans:
  Rank 1: ALT-NHV3-01 — 0.018 MW losses, 74% margin utilisation, status PLANNED
  Rank 2: ALT-NHV3-03 — CONDITIONAL: protection change required
           — annotated: "Contact Protection team before closing"
  Rank 3: None (only 2 feasible candidates)

SUPPLY_OPTIMIZATION_RUN record:
  scenario_type = SINGLE_OUTAGE
  alternatives_evaluated = 3
  alternatives_feasible = 2
  solution_found = true
  consumers_restorable = 847
  consumers_not_restorable = 0
  computation_seconds = 2.3
  algorithm_used = GREEDY_SCORED
```

---

## 6. Mode 2 — Multi-Outage Simultaneous Optimization

### 6.1 Why Mode 2 is Necessary

When two or more outages occur simultaneously in neighboring zones, their candidate supply sources may overlap. Running Mode 1 independently for each outage ignores this:

- Mode 1 for Zone A assigns Source S1 (carrying 5 MVA, margin = 6 MVA, assigns 5 MVA remaining = 1 MVA)
- Mode 1 for Zone B also assigns Source S1 (tries to assign 4 MVA → but only 1 MVA remains → OVERLOAD)

Mode 2 solves both assignments jointly, respecting the shared constraint.

### 6.2 Constrained Assignment Problem Formulation

```
Variables:
  x[i,j] ∈ {0,1}
  = 1 if outage i is assigned alternative source j as primary supply
  = 0 otherwise

Objective (maximize total composite score):
  Maximize: Σ_i Σ_j x[i,j] × composite_score[i,j]

Constraints:
  C1 — Each outage gets at most one primary source:
    Σ_j x[i,j] ≤ 1  for all outages i

  C2 — Each source's total assigned load ≤ available margin:
    Σ_i x[i,j] × load_to_transfer[i] ≤ available_margin[j]  for all sources j

  C3 — Infeasible pairs excluded:
    x[i,j] = 0 for all (i,j) where source j was eliminated for outage i in Step 1

  C4 — Voltage drop respected:
    x[i,j] = 0 where voltage_drop[i,j] > max_voltage_drop_pu
```

### 6.3 Phase 2 Algorithm — Greedy with Constraint Propagation

For up to 6 simultaneous outages and 15 candidate sources:

```
1. Build scoring matrix M[outage_i, source_j]
   = composite_score if feasible, 0 if infeasible

2. Remaining margin tracker:
   remaining_margin[j] = available_margin[j] for all sources j

3. Greedy assignment loop:
   Sort all (outage, source) pairs by M[i,j] descending
   For each pair (i,j) in sorted order:
     If outage i not yet assigned:
       If remaining_margin[j] >= load_to_transfer[i]:
         → Assign x[i,j] = 1
         → remaining_margin[j] -= load_to_transfer[i]
         → Mark outage i as assigned

4. Generate fallback assignments:
   For each assigned outage i:
     Repeat greedy pass on remaining candidates (excluding rank 1)
     Assign rank 2 and rank 3 if margin allows

5. Report unassigned outages:
   For any outage i where Σ_j x[i,j] = 0:
     consumers_not_restorable[i] = consumers_affected[i]
     Alert: "[Zone X] has no viable alternative within margin constraints"
```

### 6.4 Phase 3 Algorithm — Integer Linear Programming

When simultaneous outage count grows beyond 6 (major storm scenarios), or when a provably optimal solution is required:

```
Use ILP solver (CBC, GLPK, or Gurobi)
  Feed the formulation from 6.2 directly
  Solver returns globally optimal assignment
  Result written to SUPPLY_OPTIMIZATION_RUN and SUPPLY_SWITCHING_PLAN
  (no data model changes required — same tables, same fields)

SUPPLY_OPTIMIZATION_RUN.algorithm_used = 'ILP_CBC' or 'ILP_GUROBI'
```

---

## 7. Mode 3 — Contingency Pre-Computation (N-1 and N-2)

### 7.1 Purpose

Mode 3 answers the question: "If any single asset in our network fails tonight, does a viable alternative exist?" This is N-1 security analysis — the power systems standard that every asset's loss must be survivable.

The results drive two operational uses:
1. Dispatcher preparation — before any outage occurs, the plan is already computed
2. Infrastructure investment prioritization — zones with no N-1 alternative are flagged as network vulnerability risks

### 7.2 Nightly N-1 Scan

```
Run nightly at 01:30 (after Schedule Engine at 01:00):

For each ASSET where:
  status = OPERATIONAL
  voltage_level IN (HV, MV)
  asset_type.category IN (LINE, TRANSFORMER, SUBSTATION, SWITCHGEAR):

  Simulate loss of this asset:
    Find dependent zones = zones whose supply routes through this asset
    For each dependent zone:
      Run Steps 1–4 (common pipeline)
      Using current SOURCE_OPERATING_STATE readings

  Record to SUPPLY_OPTIMIZATION_RUN:
    scenario_type = CONTINGENCY_N1
    triggered_by = SCHEDULED_CONTINGENCY

  If solution_found = true:
    → Result available for instant use if this asset fails
    → No alert required

  If solution_found = false:
    → Create planning alert task:
         task_type = NETWORK_VULNERABILITY_FLAG
         description = "Zone [X] has NO viable N-1 alternative supply.
                        Asset [name] failure would result in unrestorable outage.
                        Network reinforcement review required."
         priority = P3 Planned
         assigned_to = network planning team
    → Flag in management dashboard vulnerability map
```

### 7.3 N-2 Scan for Critical Assets

```
For ASSET where asset_type.category IN (HV_LINE, HV_SUBSTATION)
  (i.e. high-impact assets per n2_critical_asset_categories config):

  Simulate loss of this asset AND its top-ranked N-1 alternative:
    For each dependent zone:
      Remove source_asset_id of the N-1 primary from candidate pool
      Re-run Steps 1–4 with remaining candidates

  Record to SUPPLY_OPTIMIZATION_RUN:
    scenario_type = CONTINGENCY_N2

  If solution_found = false for N-2:
    → Create HIGH PRIORITY planning alert:
         "CRITICAL N-2 vulnerability: Zone [X] has no supply after loss of
          [asset] AND its primary alternative [alt_asset].
          Immediate network planning review required."
    → Escalate to management dashboard as CRITICAL indicator
```

### 7.4 Pre-Computed Plan Retrieval

During a live outage, the dispatcher can retrieve the pre-computed N-1 plan:

```
When OUTAGE_EVENT is created for asset A in zone Z:
  Check: does a recent SUPPLY_OPTIMIZATION_RUN exist where:
    scenario_type = CONTINGENCY_N1
    asset in outage scope matches
    triggered_at > (now() − 24 hours)   ← recent enough to be current

  If yes:
    → Surface pre-computed plan to dispatcher immediately
    → Flag: "Pre-computed plan available — verify SOURCE_OPERATING_STATE
             before executing (data may be up to [age] old)"

  If no (or too old):
    → Trigger fresh Mode 1 run
    → Dispatcher waits up to 10 seconds for result
```

This means that for most common asset failures, the supply plan is available instantly rather than requiring the 10-second computation window.

---

## 8. Protection Coordination Handling

When `ALT_SUPPLY_SOURCE.requires_protection_change = true`, energizing the path with current protection settings risks incorrect fault clearance on the back-fed network.

```
Before including this candidate in any plan:

  Assess protection engineer availability:
    Query WORKER where:
      skill includes PROTECTION_RELAY_ENGINEER
      AND status = AVAILABLE
      AND base_location_id.zone_id = source_zone_id

  If protection engineer available:
    → Include candidate in plan at normal score
    → Add protection change steps as first entries in SUPPLY_PATH_SEGMENT:
         sequence_number = 1
         action_required = CHANGE_PROTECTION_SETTING
         action_description = "Set Zone 2 reach on relay [ID] to [value]
                               for back-fed configuration before closing [CB-ID]"
         requires_hv_authority = true
         verification_required = true
    → Annotate plan: "Protection setting change required.
                      Engineer [name] must execute steps 1–N before energizing.
                      Allow [X] additional minutes."

  If no protection engineer available:
    → Flag CONDITIONAL_NO_PROTECTION_ENGINEER
    → Include at lowest available rank
    → Annotate: "Protection change required but no engineer available.
                 Dispatcher must arrange engineer before execution.
                 Do NOT energize without protection change."
```

---

## 9. SCADA Staleness — Safety-Critical Data Quality

Optimization built on stale data can select an overloaded source. The staleness protocol is non-negotiable:

```
For each SOURCE_OPERATING_STATE reading used in optimization:

  staleness_minutes = now() − recorded_at

  < 30 minutes (SCADA_FRESH):
    → Use directly, no flag

  30–240 minutes (SCADA_STALE):
    → Use but:
       Reduce reliability_score by 20% for this source
       Annotate plan: "Margin data [X] min old. Verify loading before executing."
       Require dispatcher checkbox confirmation before plan approved

  > 240 minutes (MANUAL_REQUIRED):
    → Block plan approval
    → Alert dispatcher: "Source [name] margin data [X] hours old.
                         Enter current loading before optimization can proceed."
    → Display manual entry form in Dispatcher Board
    → Re-run optimization with manually entered value

During major weather events (when SCADA may be disrupted):
  → All stale readings treated as MANUAL_REQUIRED
  → Dispatcher must confirm loading before any plan is approved
  → Document in SUPPLY_OPTIMIZATION_RUN.solution_notes:
     "Executed with manually confirmed margin data due to SCADA disruption"
```

---

## 10. Operator Approval Path

For paths where `requires_operator_approval = true` (typically high-voltage tie switches or ring main connections involving the grid operator):

```
On SUPPLY_SWITCHING_PLAN creation for this candidate:
  status = AWAITING_LDC_APPROVAL (not PLANNED)

Notification to Load Dispatch Center engineer:
  {
    notification_type: SUPPLY_PLAN_APPROVAL_REQUIRED,
    plan_reference: plan_reference,
    outage_event: outage_reference,
    load_to_transfer_mva: value,
    estimated_losses_mw: value,
    margin_utilisation_pct: value,
    switching_sequence: [summary of SUPPLY_PATH_SEGMENT steps]
  }

On LDC approval:
  approved_by = LDC engineer worker_id
  approved_at = timestamp
  status = PLANNED → ready for execution

On LDC rejection:
  status = CANCELLED
  Annotation with rejection reason
  Engine re-runs with this source eliminated from candidate pool
  Next-ranked plan promoted to primary
```

---

## 11. Zero-Solution Handling — The Infeasibility Response

When no feasible alternative exists, the engine must respond explicitly rather than silently:

```
If all candidates eliminated in Step 1:
  solution_found = false
  infeasibility_reason = most common elimination reason:
    e.g. "All 3 candidates eliminated: 2 insufficient margin, 1 expired study"
  consumers_not_restorable = consumers_affected
  consumers_restorable = 0

  Immediate alerts:
    → Management alert: "CRITICAL — no viable alternative supply for
                         [zone/asset]. [N] consumers cannot be restored.
                         Manual review and emergency procurement required."
    → Network planning alert: "Zone [X] has no N-1 alternative —
                               infrastructure vulnerability confirmed."
    → Dispatcher notification with override prompt

  Dispatcher override procedure:
    Dispatcher must:
    1. Select override_reason from:
         EMERGENCY_PROCUREMENT (parts/mobile substation being arranged)
         MANUAL_ARRANGEMENT (direct coordination with grid operator)
         PARTIAL_RESTORATION (subset of consumers being restored manually)
         NO_ALTERNATIVE_ACCEPTED (outage proceeding without supply restoration)
    2. Enter free-text explanation
    3. Record emergency action being taken
    4. Override is logged permanently — cannot be deleted
    OUTAGE_EVENT.status then advances to APPROVED despite zero plan
```

---

## 12. Configuration Parameters

| Parameter | Default | Description |
|---|---|---|
| weight_loss_minimization | 0.45 | Default optimization weight for loss minimization |
| weight_reliability | 0.45 | Default weight for source reliability |
| max_voltage_drop_pu | 0.05 | Maximum permissible voltage drop on alternative path |
| marginal_threshold_pct | 75 | SOURCE loading % for MARGINAL flag |
| unavailable_threshold_pct | 90 | SOURCE loading % for UNAVAILABLE elimination |
| scada_stale_threshold_min | 30 | Minutes before SCADA reading flagged stale |
| manual_stale_threshold_min | 240 | Minutes before manual entry required |
| mode1_target_seconds | 10 | Maximum computation time — single outage |
| mode2_target_seconds | 60 | Maximum computation time — multi-outage |
| contingency_run_time | 01:30 | Daily N-1 pre-computation start |
| n2_critical_asset_categories | HV_LINE, SUBSTATION | Asset types receiving N-2 analysis |
| min_ranked_plans | 3 | Target number of ranked plans per outage |
| default_power_factor | 0.85 | Assumed pf for loss calculation where not available |
| max_voltage_drop_flag_pu | 0.04 | Voltage drop above this flagged as MARGINAL_VOLTAGE |

---

## 13. Phase Evolution

| Capability | Phase 2 implementation | Phase 3 upgrade |
|---|---|---|
| Single outage | Greedy scored, I²R loss estimate | Same + optional external load flow |
| Multi-outage | Greedy with constraint propagation | ILP exact solver (CBC / Gurobi) |
| N-1 contingency | Nightly greedy scan — all HV/MV assets | Real-time continuous monitoring |
| N-2 contingency | Critical HV assets only | All HV/MV assets |
| Loss calculation | Simplified I²R (resistance only) | Full load flow — exact P + Q losses |
| Voltage calculation | Estimated from R/X parameters | Power flow solution — exact ΔV at each bus |
| SCADA integration | Manual SOURCE_OPERATING_STATE entry | Real-time SCADA/EMS feed every 15 minutes |
| Load data | Zone averages or manual entry | Smart meter 15-min intervals for demand forecast |
| Load flow integration | Not connected | PSS/E / PowerFactory / OpenDSS via API |

---

## 14. Key Design Decisions

**Why is the catalog mandatory rather than allowing ad hoc path selection?**
Pre-validation by network planning ensures every cataloged path has been studied for thermal limits, protection coordination, and voltage. An ad hoc path selected under pressure during an outage may violate any of these. The cost of cataloging paths in advance is small; the cost of an incorrect selection during a live emergency is potentially a cascading fault affecting far more consumers than the original outage.

**Why is the margin component (10%) fixed and not configurable?**
Physical safety headroom should not be traded away for efficiency gains regardless of operational policy. A source already at 85% of capacity has very little buffer for unexpected demand, cold-load pickup on restoration, or a second contingency. The fixed 10% margin weighting ensures this physical reality always influences the recommendation.

**Why does the engine report consumers_not_restorable explicitly rather than simply failing silently?**
Operators need to know the full impact of a supply decision before they approve it. A zero-solution response forces acknowledgment of the unrestorable consumers and requires a documented override. Silent failure would allow an outage to be approved without anyone being aware that no restoration plan exists.

**Why does the N-1 pre-computation run nightly rather than on demand?**
During a live emergency fault, a dispatcher cannot wait 10 seconds for computation while consumers are off supply. Pre-computation means the plan is available in milliseconds. The nightly run also surfaces network vulnerabilities proactively — before an emergency happens — allowing infrastructure investment to address them.

**Why is Phase 3 ILP solver integration through the same tables as Phase 2?**
The data model was designed to be solver-agnostic. SUPPLY_OPTIMIZATION_RUN.algorithm_used records what was used; the SUPPLY_SWITCHING_PLAN records the result regardless of how it was computed. Upgrading from greedy to ILP requires no schema change and no interface change — only the engine computation module is replaced.

---

*End of Supply Optimization Engine Design Specification*
*FSM System · Engine 4 of 4 · Version 1.0 · April 2026*
