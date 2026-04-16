# Field Service Management — Data Model v1.4
**Utility Field Response Optimization System**
Finalized: April 2026

---

## Document Summary

| Metric | v1.0 | v1.1 | v1.2 | v1.3 | v1.4 |
|---|---|---|---|---|---|
| Domains | 5 | 5 | 6 | 7 | 7 |
| Tables | 26 | 28 | 33 | 38 | 40 |
| Approximate fields | 205 | 255 | 310 | 390 | 420 |

---

## Version History

| Version | Summary |
|---|---|
| v1.0 | Weather integration. WEATHER_EVENT table. Weather constraint and capture fields on TASK_TYPE and TASK. |
| v1.1 | Energization and isolation workflow. ISOLATION_RECORD, ISOLATION_POINT. Hard system gates on task status. |
| v1.2 | Outage impact and reliability. OUTAGE_EVENT, CONSUMER_OUTAGE_RECORD, RELIABILITY_SNAPSHOT. Task clustering: TASK_CLUSTER, TASK_CLUSTER_MEMBER. |
| v1.3 | Alternative supply catalog. SOURCE_OPERATING_STATE, ALT_SUPPLY_SOURCE, SUPPLY_PATH_SEGMENT, SUPPLY_OPTIMIZATION_RUN, SUPPLY_SWITCHING_PLAN. |
| v1.4 | CRM integration and consumer communication. CONSUMER_COMMUNICATION_PREFERENCE, CRM_NOTIFICATION, METER_ALERT. CONSUMER.crm_account_id and meter_id added. |

> ★ v1.0 · ✦ v1.1 · ◆ v1.2 · ● v1.3 · ♦ v1.4

---

## System Enforcement Rules

**Isolation gate — pre-work:** TASK cannot transition to IN_PROGRESS without ISOLATION_RECORD.pre_work_confirmed = true and all ISOLATION_POINT.pre_action_confirmed = true (where requires_permit = true).

**Isolation gate — post-work:** TASK cannot transition to COMPLETED without ISOLATION_RECORD.post_work_confirmed = true, all ISOLATION_POINT.post_action_confirmed = true, and equipment_status_post and source_status_post confirmed.

**Documentation gate:** TASK cannot transition to COMPLETED unless all mandatory DOCUMENT_TEMPLATE.fields_schema fields are populated in TEST_RECORD.test_readings.

**Skill expiry gate:** WORKER_SKILL records with is_active = false are excluded from all matching engine queries.

**Supply planning gate:** OUTAGE_EVENT cannot advance to APPROVED unless SUPPLY_SWITCHING_PLAN exists or dispatcher records a documented override with reason.

**Communication gate:** Planned cluster cannot be marked IN_EXECUTION unless CRM_NOTIFICATION records confirm regulatory advance notices were sent ≥ 72 hours before planned_window_start.

---

## Reliability Index Computation

| Index | Formula |
|---|---|
| SAIFI | Sum(consumers_effectively_interrupted) ÷ total_consumers |
| SAIDI | Sum(consumer_minutes_interrupted) ÷ total_consumers |
| CAIFI | Sum(interruptions) ÷ consumers interrupted at least once |
| CAIDI | SAIDI ÷ SAIFI |

Reduced by: task clustering (fewer outage events → lower SAIFI), alternative supply (lower effective duration → lower SAIDI/CAIDI), proactive maintenance (fewer emergency events).

---

## Supply Optimization Objective Function

`composite_score = (weight_loss × loss_score) + (weight_reliability × reliability_score)`

Where loss_score = 1 − (estimated_losses ÷ max_possible_losses) and reliability_score = ALT_SUPPLY_SOURCE.reliability_score ÷ 100. Weights are configurable per SUPPLY_OPTIMIZATION_RUN and sum to 1.0.

---

## Matching Engine Composite Score

`composite_score = (0.50 × proximity_score) + (0.30 × workload_score) + (0.20 × skill_preference_score)`

Weights configurable. Proximity score based on travel time from worker location to task site. Workload score based on current queue depth vs remaining shift. Skill preference score based on preferred (non-mandatory) skills matched and worker grade.

---

## Domain 1 — Task Management · 9 tables

---

### TASK_TYPE [v1.0]

| Field | Ver | Type | Key | Description |
|---|---|---|---|---|
| task_type_id | | UUID | PK | Unique identifier |
| code | | VARCHAR(30) | UQ | e.g. XFMR_TEST, FAULT_REPAIR, BATT_INSPECT |
| name | | VARCHAR(100) | | Full task type name |
| stream_type | | ENUM | | REACTIVE · SCHEDULED · PROJECT |
| priority_class | | ENUM | | P1 · P2 · P3 · P4 |
| sla_type | | ENUM | | RESPONSE_TIME · COMPLIANCE_DATE |
| default_sla_minutes | | INTEGER | | Response-time SLAs only |
| default_duration_minutes | | INTEGER | | Baseline estimate |
| min_crew_size | | SMALLINT | | Minimum crew to start |
| requires_safety_officer | | BOOLEAN | | Hard co-dispatch constraint |
| requires_permit | | BOOLEAN | | PTW required before start |
| workforce_pool | | ENUM | | ELECTRICAL · VEGETATION · SPECIALIST |
| doc_template_id | | UUID | FK | → DOCUMENT_TEMPLATE (nullable) |
| is_active | | BOOLEAN | | Inactive types excluded from dispatch |
| weather_sensitive | ★ | BOOLEAN | | Does weather affect this task? |
| prohibited_in_lightning | ★ | BOOLEAN | | Hard block in lightning |
| prohibited_in_rain | ★ | BOOLEAN | | Hard block for outdoor HV in rain |
| max_wind_speed_kmh | ★ | SMALLINT | | Wind speed limit (nullable) |
| min_visibility_metres | ★ | SMALLINT | | Minimum visibility (nullable) |
| indoor_task | ★ | BOOLEAN | | Indoor = weather-insensitive |
| weather_duration_factors | ★ | JSONB | | Multipliers e.g. {"rain":1.4,"extreme_heat":1.2} |

---

### TASK [v1.0 · v1.1 · v1.2]

| Field | Ver | Type | Key | Description |
|---|---|---|---|---|
| task_id | | UUID | PK | Unique identifier |
| reference_number | | VARCHAR(20) | UQ | e.g. WO-2026-04-0042 |
| task_type_id | | UUID | FK | → TASK_TYPE |
| asset_id | | UUID | FK | → ASSET (nullable) |
| location_id | | UUID | FK | → LOCATION |
| consumer_id | | UUID | FK | → CONSUMER (nullable) |
| schedule_id | | UUID | FK | → MAINTENANCE_SCHEDULE (null if reactive) |
| stream_type | | ENUM | | REACTIVE · SCHEDULED · PROJECT |
| status | | ENUM | | NEW · ASSIGNED · IN_PROGRESS · ON_HOLD · COMPLETED · DEFERRED · CANCELLED |
| priority_class | | ENUM | | P1 · P2 · P3 · P4 |
| complexity_band | | ENUM | | LOW · MEDIUM · HIGH · VERY_HIGH |
| triggered_by | | ENUM | | EVENT · CALENDAR · PROJECT_PLAN · MANUAL · WEATHER_EVENT |
| sla_deadline | | TIMESTAMP | | Hard deadline |
| estimated_duration_minutes | | INTEGER | | Adjusted by complexity and weather |
| actual_duration_minutes | | INTEGER | | Filled on completion — ML training signal |
| description | | TEXT | | Task instructions or fault description |
| created_at | | TIMESTAMP | | Creation timestamp |
| assigned_at | | TIMESTAMP | | When crew assigned |
| started_at | | TIMESTAMP | | Crew checked in on site |
| completed_at | | TIMESTAMP | | Task marked complete |
| dispatcher_notes | | TEXT | | Context added at assignment |
| weather_event_id | ★ | UUID | FK | → WEATHER_EVENT (nullable) |
| weather_at_start | ★ | ENUM | | CLEAR · RAIN · HEAVY_RAIN · LIGHTNING · HIGH_WIND · FOG · EXTREME_HEAT · FLOOD |
| weather_caused_extension | ★ | BOOLEAN | | Weather extended this task? |
| weather_extension_minutes | ★ | INTEGER | | Extra minutes due to weather |
| isolation_record_id | ✦ | UUID | FK | → ISOLATION_RECORD (nullable) |
| shutdown_window_start | ✦ | TIMESTAMP | | Earliest permitted shutdown start |
| shutdown_window_end | ✦ | TIMESTAMP | | Mandatory restoration deadline |
| shutdown_overrun | ✦ | BOOLEAN | | True if re-energization exceeded window |
| cluster_id | ◆ | UUID | FK | → TASK_CLUSTER (nullable) |
| is_clubbed | ◆ | BOOLEAN | | True if cluster_id populated |
| clubbing_opportunity_flagged | ◆ | BOOLEAN | | System flagged opportunity not yet acted on |

---

### TASK_SKILL_REQUIREMENT

| Field | Type | Key | Description |
|---|---|---|---|
| req_id | UUID | PK | Unique identifier |
| task_type_id | UUID | FK | → TASK_TYPE |
| skill_id | UUID | FK | → SKILL |
| is_mandatory | BOOLEAN | | True = hard filter |
| min_count | SMALLINT | | Crew members required with this skill |

---

### TASK_PART_REQUIREMENT

| Field | Type | Key | Description |
|---|---|---|---|
| req_id | UUID | PK | Unique identifier |
| task_type_id | UUID | FK | → TASK_TYPE |
| part_id | UUID | FK | → PART |
| quantity_typical | DECIMAL(10,2) | | Expected quantity |
| is_mandatory | BOOLEAN | | True = dispatch blocked if stock insufficient |

---

### SLA_TRACKER

| Field | Type | Key | Description |
|---|---|---|---|
| sla_id | UUID | PK | Unique identifier |
| task_id | UUID | FK | → TASK (1:1) |
| sla_type | ENUM | | RESPONSE_TIME · COMPLIANCE_DATE |
| sla_deadline | TIMESTAMP | | Hard deadline (denormalised) |
| alert_30d_triggered | BOOLEAN | | 30-day advance warning sent |
| alert_7d_triggered | BOOLEAN | | 7-day escalation sent |
| alert_1d_triggered | BOOLEAN | | 1-day critical alert sent |
| is_breached | BOOLEAN | | True when deadline passed without completion |
| breached_at | TIMESTAMP | | Exact breach timestamp |
| breach_notified | BOOLEAN | | Management notification sent |

---

### ISOLATION_RECORD [v1.1]

| Field | Section | Type | Key | Description |
|---|---|---|---|---|
| isolation_record_id | | UUID | PK | Unique identifier |
| task_id | | UUID | FK | → TASK (1:1) |
| ptw_number | | VARCHAR(30) | UQ | Permit to Work reference |
| ptw_issued_by | | UUID | FK | → WORKER — HV authority holder |
| ptw_issued_at | | TIMESTAMP | | When PTW issued |
| shutdown_window_start | | TIMESTAMP | | Earliest work start |
| shutdown_window_end | | TIMESTAMP | | Latest restoration time |
| equipment_status_pre | Pre-work | ENUM | | ENERGIZED · DE_ENERGIZED · ISOLATED · EARTHED |
| equipment_status_pre_confirmed_by | Pre-work | UUID | FK | → WORKER — HV authority |
| equipment_status_pre_confirmed_at | Pre-work | TIMESTAMP | | Confirmation timestamp |
| source_asset_id | Pre-work | UUID | FK | → ASSET — feeding line isolated |
| source_status_pre | Pre-work | ENUM | | ENERGIZED · OPEN · LOCKED_OUT · TAGGED_OUT |
| source_status_pre_confirmed_by | Pre-work | UUID | FK | → WORKER |
| source_status_pre_confirmed_at | Pre-work | TIMESTAMP | | Source confirmation timestamp |
| pre_work_confirmed | HARD GATE | BOOLEAN | | Task cannot start until true |
| work_completion_declared_by | Post-work | UUID | FK | → WORKER — lead worker |
| work_completion_declared_at | Post-work | TIMESTAMP | | Declaration timestamp |
| area_clear_confirmed | Post-work | BOOLEAN | | All personnel clear, tools removed |
| safety_officer_signoff_by | Post-work | UUID | FK | → WORKER (where required) |
| safety_officer_signoff_at | Post-work | TIMESTAMP | | Sign-off timestamp |
| permit_surrendered_at | Post-work | TIMESTAMP | | PTW formally surrendered |
| post_work_confirmed | HARD GATE | BOOLEAN | | Re-energization cannot proceed until true |
| equipment_status_post | Re-energization | ENUM | | ENERGIZED |
| equipment_status_post_confirmed_by | Re-energization | UUID | FK | → WORKER — HV authority |
| equipment_status_post_confirmed_at | Re-energization | TIMESTAMP | | Re-energization timestamp |
| source_status_post | Re-energization | ENUM | | ENERGIZED · CLOSED |
| source_status_post_confirmed_by | Re-energization | UUID | FK | → WORKER |
| source_status_post_confirmed_at | Re-energization | TIMESTAMP | | Source restoration timestamp |
| shutdown_overrun | Outcome | BOOLEAN | | True if exceeded window |
| overrun_minutes | Outcome | INTEGER | | Minutes exceeded (nullable) |
| overrun_reason | Outcome | TEXT | | Mandatory explanation if overrun |
| notes | Outcome | TEXT | | Additional safety observations |

---

### ISOLATION_POINT [v1.1]

| Field | Type | Key | Description |
|---|---|---|---|
| point_id | UUID | PK | Unique identifier |
| isolation_record_id | UUID | FK | → ISOLATION_RECORD |
| sequence_number | SMALLINT | | Strict execution order |
| point_type | ENUM | | CIRCUIT_BREAKER · ISOLATOR · EARTH · LOCK · TAG · FUSE_REMOVAL |
| asset_id | UUID | FK | → ASSET — switching device or earthing point |
| point_description | VARCHAR(200) | | e.g. "33kV CB-104 at Ramdas Sub Bay 3 — OPEN" |
| pre_action_required | ENUM | | OPEN · CLOSE · APPLY_EARTH · LOCK_OUT · TAG_OUT · REMOVE_FUSE |
| pre_action_done_by | UUID | FK | → WORKER |
| pre_action_done_at | TIMESTAMP | | Action timestamp |
| pre_action_confirmed | BOOLEAN | | All points must be true for pre_work_confirmed |
| post_action_required | ENUM | | CLOSE · OPEN · REMOVE_EARTH · UNLOCK · REMOVE_TAG · REFIT_FUSE |
| post_action_done_by | UUID | FK | → WORKER |
| post_action_done_at | TIMESTAMP | | Reversal timestamp |
| post_action_confirmed | BOOLEAN | | All points must be true for post_work_confirmed |

---

### TASK_CLUSTER [v1.2]

| Field | Type | Key | Description |
|---|---|---|---|
| cluster_id | UUID | PK | Unique identifier |
| cluster_reference | VARCHAR(20) | UQ | e.g. CLU-2026-04-018 |
| lead_task_id | UUID | FK | → TASK — primary shutdown reason |
| isolation_record_id | UUID | FK | → ISOLATION_RECORD — shared across all clustered tasks |
| outage_event_id | UUID | FK | → OUTAGE_EVENT — single event for whole cluster |
| cluster_type | ENUM | | SAME_ISOLATION_SCOPE · SAME_CORRIDOR · SAME_CREW_VISIT |
| planned_window_start | TIMESTAMP | | Earliest planned start |
| planned_window_end | TIMESTAMP | | Latest planned end |
| status | ENUM | | PROPOSED · APPROVED · IN_EXECUTION · COMPLETED · CANCELLED |
| tasks_proposed | SMALLINT | | Tasks proposed to club |
| tasks_completed | SMALLINT | | Filled on completion |
| saifi_events_saved | SMALLINT | | tasks_proposed − 1 |
| approved_by | UUID | FK | → WORKER — planning authority |
| notes | TEXT | | Planning rationale |

---

### TASK_CLUSTER_MEMBER [v1.2]

| Field | Type | Key | Description |
|---|---|---|---|
| member_id | UUID | PK | Unique identifier |
| cluster_id | UUID | FK | → TASK_CLUSTER |
| task_id | UUID | FK | → TASK |
| member_role | ENUM | | LEAD · DEPENDENT · OPPORTUNISTIC |
| sequence_in_cluster | SMALLINT | | Execution order |
| added_by | ENUM | | SYSTEM · PLANNER · DISPATCHER |
| completed_in_window | BOOLEAN | | Completed within planned window? |
| deferred_reason | TEXT | | Reason if not completed |

---

## Domain 2 — Workforce · 5 tables

---

### WORKER

| Field | Type | Key | Description |
|---|---|---|---|
| worker_id | UUID | PK | Unique identifier |
| employee_id | VARCHAR(20) | UQ | HR system employee number |
| name | VARCHAR(100) | | Full name |
| workforce_pool | ENUM | | ELECTRICAL · VEGETATION · SPECIALIST |
| grade | VARCHAR(30) | | e.g. Junior Technician, AE, Senior Engineer, Supervisor |
| status | ENUM | | AVAILABLE · ON_TASK · ON_BREAK · OFF_SHIFT · ON_LEAVE |
| current_lat | DECIMAL(9,6) | | Last known latitude |
| current_lng | DECIMAL(9,6) | | Last known longitude |
| location_updated_at | TIMESTAMP | | Staleness check for dispatch |
| shift_start | TIME | | Default shift start |
| shift_end | TIME | | Default shift end |
| van_id | UUID | FK | → VAN (nullable) |
| base_location_id | UUID | FK | → LOCATION — home depot or office |
| is_safety_officer | BOOLEAN | | Qualifies for safety officer co-dispatch |
| is_hv_authority | BOOLEAN | | HV switching authority — live-line and isolation confirmation |
| mobile_number | VARCHAR(20) | | For dispatcher contact and mobile app |

### SKILL

| Field | Type | Key | Description |
|---|---|---|---|
| skill_id | UUID | PK | Unique identifier |
| code | VARCHAR(30) | UQ | e.g. HT_LINE, RELAY_TEST, BATT_INSPECT, ARBORIST |
| name | VARCHAR(100) | | Full skill or certification name |
| category | ENUM | | HV_OPS · LV_OPS · TESTING · INSPECTION · METERING · VEGETATION · SPECIALIST · MANAGEMENT |
| level | ENUM | | GENERAL · CERTIFIED · AUTHORITY_HOLDER · SPECIALIST |
| requires_renewal | BOOLEAN | | Whether certification expires |
| renewal_period_months | SMALLINT | | Null if no renewal |

### WORKER_SKILL

| Field | Type | Key | Description |
|---|---|---|---|
| worker_skill_id | UUID | PK | Unique identifier |
| worker_id | UUID | FK | → WORKER |
| skill_id | UUID | FK | → SKILL |
| certified_date | DATE | | Date granted or last renewed |
| expiry_date | DATE | | Null if no expiry. Matching engine excludes expired. |
| certification_reference | VARCHAR(50) | | Certificate or authority reference |
| is_active | BOOLEAN | | False = expired or suspended — matching engine safety gate |

### CREW_ASSIGNMENT

| Field | Type | Key | Description |
|---|---|---|---|
| assignment_id | UUID | PK | Unique identifier |
| task_id | UUID | FK | → TASK |
| worker_id | UUID | FK | → WORKER |
| role | ENUM | | LEAD · SUPPORT · SAFETY_OFFICER · OBSERVER |
| suggestion_rank | SMALLINT | | 1 = top, 2 = 2nd, NULL = dispatcher override |
| override_reason_code | VARCHAR(30) | | Reason category when overriding |
| override_notes | TEXT | | Free-text — Phase 3 learning input |
| assigned_at | TIMESTAMP | | When assignment made |
| dispatched_at | TIMESTAMP | | When worker started travel |
| arrived_at | TIMESTAMP | | When worker checked in on site |

### SHIFT_SCHEDULE

| Field | Type | Key | Description |
|---|---|---|---|
| shift_id | UUID | PK | Unique identifier |
| worker_id | UUID | FK | → WORKER |
| shift_date | DATE | | Calendar date |
| shift_type | ENUM | | DAY · NIGHT · STANDBY · ON_CALL |
| planned_start | TIME | | Planned shift start |
| planned_end | TIME | | Planned shift end |
| actual_start | TIME | | Actual start (retrospective) |
| actual_end | TIME | | Actual end |

---

## Domain 3 — Asset and Schedule · 6 tables

---

### ASSET [v1.1]

| Field | Ver | Type | Key | Description |
|---|---|---|---|---|
| asset_id | | UUID | PK | Unique identifier |
| asset_code | | VARCHAR(30) | UQ | Utility asset tag or GIS reference |
| asset_type_id | | UUID | FK | → ASSET_TYPE |
| name | | VARCHAR(150) | | e.g. Ramdas Substation 33kV Transformer T1 |
| location_id | | UUID | FK | → LOCATION |
| zone_id | | UUID | FK | → NETWORK_ZONE |
| voltage_level | | ENUM | | HV · MV · LV |
| manufacturer | | VARCHAR(100) | | Equipment manufacturer |
| model | | VARCHAR(100) | | Model designation |
| serial_number | | VARCHAR(100) | UQ | Manufacturer serial number |
| installation_date | | DATE | | Date commissioned |
| status | | ENUM | | OPERATIONAL · FAULTY · UNDER_MAINTENANCE · DECOMMISSIONED · STANDBY |
| condition_score | | SMALLINT | | 1–10 condition rating |
| last_inspection_date | | DATE | | Denormalised — synced from SERVICE_HISTORY |
| last_test_date | | DATE | | Denormalised — synced from TEST_RECORD |
| warranty_expiry | | DATE | | Manufacturer warranty expiry (nullable) |
| parent_asset_id | | UUID | FK | → ASSET self-reference for sub-components |
| energization_status | ✦ | ENUM | | ENERGIZED · DE_ENERGIZED · ISOLATED · EARTHED · UNKNOWN |
| energization_updated_at | ✦ | TIMESTAMP | | When status last confirmed |
| energization_updated_by | ✦ | UUID | FK | → WORKER — HV authority holder |

### ASSET_TYPE

| Field | Type | Key | Description |
|---|---|---|---|
| asset_type_id | UUID | PK | Unique identifier |
| code | VARCHAR(30) | UQ | e.g. XFMR_33KV, PROT_RELAY, BATT_BACKUP, HV_LINE |
| name | VARCHAR(100) | | Full asset type name |
| category | ENUM | | TRANSFORMER · RELAY · BATTERY · LINE · POLE · SUBSTATION · SWITCHGEAR · METER · DER · PLANT |
| typical_lifespan_years | SMALLINT | | Expected service life |
| description | TEXT | | Technical notes and applicable standards |

### MAINTENANCE_SCHEDULE

| Field | Type | Key | Description |
|---|---|---|---|
| schedule_id | UUID | PK | Unique identifier |
| asset_id | UUID | FK | → ASSET (null = type-level default) |
| asset_type_id | UUID | FK | → ASSET_TYPE (null = asset-specific) |
| task_type_id | UUID | FK | → TASK_TYPE |
| frequency_type | ENUM | | MONTHLY · QUARTERLY · SEMI_ANNUAL · ANNUAL · CUSTOM |
| frequency_days | INTEGER | | For CUSTOM frequency in days |
| last_generated_date | DATE | | Date last task auto-generated |
| next_due_date | DATE | | Next compliance deadline |
| advance_notice_days | SMALLINT | | Days before due to generate task (default 30) |
| is_regulatory | BOOLEAN | | Non-compliance carries regulatory penalty |
| regulatory_reference | VARCHAR(100) | | Regulation or standard number (nullable) |
| is_active | BOOLEAN | | Inactive schedules skipped by engine |
| notes | TEXT | | Special instructions |

### LOCATION

| Field | Type | Key | Description |
|---|---|---|---|
| location_id | UUID | PK | Unique identifier |
| address | TEXT | | Full postal address |
| lat | DECIMAL(9,6) | | Latitude — WGS84 |
| lng | DECIMAL(9,6) | | Longitude — WGS84 |
| zone_id | UUID | FK | → NETWORK_ZONE (nullable) |
| location_type | ENUM | | ASSET_SITE · DEPOT · CONSUMER_PREMISE · SUBSTATION · OFFICE |
| geocode_source | ENUM | | MANUAL · GIS_IMPORT · API |

### NETWORK_ZONE [v1.2]

| Field | Ver | Type | Key | Description |
|---|---|---|---|---|
| zone_id | | UUID | PK | Unique identifier |
| code | | VARCHAR(20) | UQ | e.g. NORTH-HV, URBAN-LV-1, RURAL-MV |
| name | | VARCHAR(100) | | Full zone name |
| voltage_level | | ENUM | | HV · MV · LV · MIXED |
| responsible_team | | VARCHAR(50) | | Primary team responsible |
| total_consumers | ◆ | INTEGER | | SAIFI/SAIDI denominator — synced from billing |
| alt_supply_zone_id | ◆ | UUID | FK | → NETWORK_ZONE self-ref — primary back-feed zone |
| alt_supply_capacity_pct | ◆ | SMALLINT | | % of this zone's load the alt zone can carry |

### WEATHER_EVENT [v1.0]

| Field | Type | Key | Description |
|---|---|---|---|
| event_id | UUID | PK | Unique identifier |
| event_type | ENUM | | STORM · CYCLONE · FLOOD · LIGHTNING · EXTREME_HEAT · FOG · HIGH_WIND |
| event_group_id | UUID | | Groups events across zones for one weather system |
| zone_id | UUID | FK | → NETWORK_ZONE |
| severity | ENUM | | LOW · MEDIUM · HIGH · EXTREME |
| started_at | TIMESTAMP | | Event start |
| ended_at | TIMESTAMP | | Event end (null if ongoing) |
| source | ENUM | | WEATHER_API · MANUAL · SCADA_ALERT |
| expected_task_surge | SMALLINT | | Estimated additional emergency tasks |
| affected_assets_count | INTEGER | | Assets in affected zone |
| notes | TEXT | | Operational notes |

---

## Domain 4 — Inventory · 5 tables

### PART

| Field | Type | Key | Description |
|---|---|---|---|
| part_id | UUID | PK | Unique identifier |
| part_code | VARCHAR(30) | UQ | Internal part number |
| name | VARCHAR(150) | | Part name and specification |
| category | VARCHAR(50) | | e.g. Cable Accessories, Relay Parts, Transformer Oil |
| unit_of_measure | VARCHAR(20) | | e.g. nos, metres, litres, kg |
| standard_unit_cost | DECIMAL(12,2) | | Standard cost for job reporting |
| lead_time_days | SMALLINT | | Procurement lead time |
| is_critical | BOOLEAN | | Critical spare — maintain minimum stock |

### VAN

| Field | Type | Key | Description |
|---|---|---|---|
| van_id | UUID | PK | Unique identifier |
| registration | VARCHAR(20) | UQ | Vehicle registration |
| van_kit_type | VARCHAR(50) | | e.g. CABLE_FAULT, GENERAL, METERING, RELAY_TEST |
| depot_id | UUID | FK | → DEPOT |
| last_restocked_date | DATE | | Last confirmed full restock |
| status | ENUM | | ACTIVE · IN_MAINTENANCE · DECOMMISSIONED |

### VAN_STOCK

| Field | Type | Key | Description |
|---|---|---|---|
| van_stock_id | UUID | PK | Unique identifier |
| van_id | UUID | FK | → VAN |
| part_id | UUID | FK | → PART |
| quantity_on_hand | DECIMAL(10,2) | | Current quantity |
| minimum_level | DECIMAL(10,2) | | Restock alert threshold |
| updated_at | TIMESTAMP | | Last confirmed stock count |

### DEPOT

| Field | Type | Key | Description |
|---|---|---|---|
| depot_id | UUID | PK | Unique identifier |
| name | VARCHAR(100) | | Depot name |
| location_id | UUID | FK | → LOCATION |
| is_main_depot | BOOLEAN | | Main vs satellite pickup |
| contact_number | VARCHAR(20) | | Parts availability contact |

### DEPOT_STOCK

| Field | Type | Key | Description |
|---|---|---|---|
| depot_stock_id | UUID | PK | Unique identifier |
| depot_id | UUID | FK | → DEPOT |
| part_id | UUID | FK | → PART |
| quantity_on_hand | DECIMAL(10,2) | | Current depot stock |
| reorder_level | DECIMAL(10,2) | | Triggers procurement |
| reorder_quantity | DECIMAL(10,2) | | Standard replenishment quantity |
| last_updated | TIMESTAMP | | Last inventory update |

---

## Domain 5 — History and Compliance · 5 tables

### SERVICE_HISTORY

| Field | Type | Key | Description |
|---|---|---|---|
| history_id | UUID | PK | Unique identifier |
| asset_id | UUID | FK | → ASSET — always required. Asset-centric. |
| task_id | UUID | FK | → TASK (nullable for pre-system records) |
| event_type | ENUM | | INSPECTION · TEST · MAINTENANCE · FAULT · REPAIR · COMMISSIONING |
| event_date | TIMESTAMP | | When event occurred on site |
| findings | TEXT | | What was observed or measured |
| action_taken | TEXT | | Work performed and outcome |
| condition_score_before | SMALLINT | | Asset condition 1–10 before |
| condition_score_after | SMALLINT | | Asset condition 1–10 after |
| next_action_recommended | TEXT | | Technician recommendation |
| lead_worker_id | UUID | FK | → WORKER — accountable for work |

### PARTS_CONSUMED

| Field | Type | Key | Description |
|---|---|---|---|
| consumed_id | UUID | PK | Unique identifier |
| task_id | UUID | FK | → TASK |
| part_id | UUID | FK | → PART |
| quantity_used | DECIMAL(10,2) | | Actual quantity consumed |
| source | ENUM | | VAN · DEPOT |
| van_id | UUID | FK | → VAN (nullable) |
| depot_id | UUID | FK | → DEPOT (nullable) |

### DOCUMENT_TEMPLATE

| Field | Type | Key | Description |
|---|---|---|---|
| template_id | UUID | PK | Unique identifier |
| task_type_id | UUID | FK | → TASK_TYPE |
| name | VARCHAR(100) | | e.g. Transformer Test Certificate |
| version | VARCHAR(10) | | Template version |
| fields_schema | JSONB | | JSON definition of required fields and validation |
| is_regulatory | BOOLEAN | | Must be submitted to regulatory body |
| regulatory_body | VARCHAR(100) | | Regulatory authority (nullable) |

### TEST_RECORD

| Field | Type | Key | Description |
|---|---|---|---|
| test_record_id | UUID | PK | Unique identifier |
| task_id | UUID | FK | → TASK |
| asset_id | UUID | FK | → ASSET |
| template_id | UUID | FK | → DOCUMENT_TEMPLATE |
| test_date | DATE | | Date performed on site |
| tester_worker_id | UUID | FK | → WORKER — accountable |
| test_readings | JSONB | | Flexible readings e.g. BDV value, relay operate time ms |
| pass_fail | ENUM | | PASS · FAIL · CONDITIONAL_PASS · PENDING_REVIEW |
| certificate_number | VARCHAR(50) | UQ | Issued certificate reference |
| next_test_due_date | DATE | | Updates MAINTENANCE_SCHEDULE.next_due_date |
| submission_status | ENUM | | NOT_REQUIRED · PENDING · SUBMITTED · ACCEPTED · REJECTED |
| submitted_at | TIMESTAMP | | When submitted to regulatory body |
| notes | TEXT | | Tester observations |

### CONSUMER [v1.4]

| Field | Ver | Type | Key | Description |
|---|---|---|---|---|
| consumer_id | | UUID | PK | Unique identifier |
| account_number | | VARCHAR(30) | UQ | Utility billing account number |
| name | | VARCHAR(150) | | Consumer name |
| location_id | | UUID | FK | → LOCATION |
| contact_phone | | VARCHAR(20) | | Primary contact |
| contact_email | | VARCHAR(100) | | Email for notifications |
| consumer_category | | ENUM | | RESIDENTIAL · COMMERCIAL · INDUSTRIAL · CRITICAL_INFRASTRUCTURE |
| meter_asset_id | | UUID | FK | → ASSET — linked meter or DER (nullable) |
| crm_account_id | ♦ | VARCHAR(50) | | Consumer's account ID in CRM system — join key |
| meter_id | ♦ | VARCHAR(50) | | Smart meter ID in MDMS — for last-gasp signal matching |

---

## Domain 6 — Outage Impact and Reliability · 3 tables [v1.2]

### OUTAGE_EVENT [v1.2 · v1.3]

| Field | Ver | Type | Key | Description |
|---|---|---|---|---|
| outage_event_id | | UUID | PK | Unique identifier |
| outage_reference | | VARCHAR(20) | UQ | e.g. OUT-2026-04-031 |
| cluster_id | | UUID | FK | → TASK_CLUSTER (nullable for standalone reactive) |
| primary_task_id | | UUID | FK | → TASK |
| zone_id | | UUID | FK | → NETWORK_ZONE |
| feeder_asset_id | | UUID | FK | → ASSET — upstream switching point |
| outage_type | | ENUM | | PLANNED · UNPLANNED · EXTENDED |
| switching_plan_id | ● | UUID | FK | → SUPPLY_SWITCHING_PLAN (nullable) |
| affected_area_description | | TEXT | | e.g. "Sector 14–18, Ghaziabad" |
| total_consumers_in_zone | | INTEGER | | Snapshot — SAIFI/SAIDI denominator |
| consumers_affected | | INTEGER | | Consumers who lost supply |
| consumers_on_alt_supply | | INTEGER | | Consumers restored via alternative |
| consumers_effectively_interrupted | | INTEGER | | consumers_affected − consumers_on_alt_supply |
| outage_start | | TIMESTAMP | | When supply interrupted |
| outage_end | | TIMESTAMP | | When fully restored |
| duration_minutes | | INTEGER | | Computed: outage_end − outage_start |
| consumer_minutes_interrupted | | BIGINT | | consumers_effectively_interrupted × duration_minutes |
| restoration_method | | ENUM | | NORMAL_SUPPLY_RESTORED · ALT_SUPPLY_PARTIAL · ALT_SUPPLY_FULL · NOT_RESTORED |

### CONSUMER_OUTAGE_RECORD [v1.2 · v1.4]

| Field | Ver | Type | Key | Description |
|---|---|---|---|---|
| record_id | | UUID | PK | Unique identifier |
| outage_event_id | | UUID | FK | → OUTAGE_EVENT |
| consumer_id | | UUID | FK | → CONSUMER |
| supply_lost_at | | TIMESTAMP | | When consumer lost supply |
| alt_supply_provided | | BOOLEAN | | Alternative supply arranged? |
| alt_supply_source_id | | UUID | FK | → ASSET — source of alternative supply |
| alt_supply_restored_at | | TIMESTAMP | | When alternative supply connected |
| supply_fully_restored_at | | TIMESTAMP | | When normal supply restored |
| effective_interruption_minutes | | INTEGER | | Actual minutes without any supply |
| consumer_category | | ENUM | | RESIDENTIAL · COMMERCIAL · INDUSTRIAL · CRITICAL_INFRASTRUCTURE |
| complaint_raised | | BOOLEAN | | Consumer raised complaint during outage |
| complaint_task_id | | UUID | FK | → TASK (nullable) |
| last_gasp_received_at | ♦ | TIMESTAMP | | Last-gasp signal from smart meter (authoritative supply_lost time) |
| restoration_signal_received_at | ♦ | TIMESTAMP | | Power-restored signal from smart meter |

### RELIABILITY_SNAPSHOT [v1.2]

| Field | Type | Key | Description |
|---|---|---|---|
| snapshot_id | UUID | PK | Unique identifier |
| zone_id | UUID | FK | → NETWORK_ZONE |
| period_type | ENUM | | MONTHLY · QUARTERLY · ANNUAL |
| period_start | DATE | | Start of reporting period |
| period_end | DATE | | End of reporting period |
| total_consumers | INTEGER | | Total consumers served |
| total_outage_events | INTEGER | | Total distinct interruption events |
| planned_outage_events | INTEGER | | Planned outages |
| saifi | DECIMAL(8,4) | | Sum(consumers_effectively_interrupted) ÷ total_consumers |
| saidi | DECIMAL(10,4) | | Sum(consumer_minutes_interrupted) ÷ total_consumers |
| caifi | DECIMAL(8,4) | | Sum(interruptions) ÷ consumers interrupted at least once |
| caidi | DECIMAL(8,4) | | SAIDI ÷ SAIFI |
| tasks_clubbed | INTEGER | | Tasks completed via clustering |
| saifi_events_saved_by_clustering | INTEGER | | Planning effectiveness metric |

---

## Domain 7 — Supply Topology and Optimization · 5 tables [v1.3]

### ALT_SUPPLY_SOURCE

| Field | Type | Key | Description |
|---|---|---|---|
| alt_source_id | UUID | PK | Unique identifier |
| source_reference | VARCHAR(30) | UQ | e.g. ALT-ZONE-NHV-01 |
| target_type | ENUM | | ZONE · ASSET · FEEDER |
| target_zone_id | UUID | FK | → NETWORK_ZONE (nullable) |
| target_asset_id | UUID | FK | → ASSET (nullable) |
| source_asset_id | UUID | FK | → ASSET — feeding asset |
| source_zone_id | UUID | FK | → NETWORK_ZONE — source origin |
| supply_type | ENUM | | FEEDER_BACKFEED · TIE_LINE · RING_MAIN · DG_SET · MOBILE_SUBSTATION · BESS |
| is_normally_open | BOOLEAN | | True if path normally open |
| switching_sequence_count | SMALLINT | | Number of switching operations |
| estimated_switching_minutes | SMALLINT | | Typical time to establish supply |
| path_length_km | DECIMAL(8,3) | | Electrical path length |
| resistance_ohm_per_km | DECIMAL(8,5) | | I²R loss calculation |
| reactance_ohm_per_km | DECIMAL(8,5) | | Voltage drop calculation |
| max_transferable_load_mva | DECIMAL(8,3) | | Thermal limit of path |
| voltage_level | ENUM | | HV · MV · LV |
| requires_protection_change | BOOLEAN | | Relay settings must change before energizing |
| requires_operator_approval | BOOLEAN | | Load dispatch center approval required |
| reliability_score | DECIMAL(4,2) | | Historical availability % — 0 to 100 |
| study_reference | VARCHAR(50) | | Network study that validated this path |
| approved_date | DATE | | Formally approved by network planning |
| valid_until | DATE | | Expiry — re-study required after network changes |
| is_active | BOOLEAN | | Inactive alternatives excluded from optimizer |
| notes | TEXT | | Operational notes e.g. seasonal restrictions |

### SUPPLY_PATH_SEGMENT

| Field | Type | Key | Description |
|---|---|---|---|
| segment_id | UUID | PK | Unique identifier |
| alt_source_id | UUID | FK | → ALT_SUPPLY_SOURCE |
| sequence_number | SMALLINT | | Strict execution order |
| switching_asset_id | UUID | FK | → ASSET — CB, isolator, or tie-switch |
| action_required | ENUM | | CLOSE · OPEN · CHECK_VOLTAGE · VERIFY_SYNCHRONISM · CHANGE_PROTECTION_SETTING |
| action_description | VARCHAR(200) | | e.g. "Close Tie Switch TS-14 at Sector 12" |
| requires_hv_authority | BOOLEAN | | Step requires HV authority holder |
| estimated_minutes | SMALLINT | | Time to execute this step |
| verification_required | BOOLEAN | | Second person verification before proceeding |

### SOURCE_OPERATING_STATE

| Field | Type | Key | Description |
|---|---|---|---|
| state_id | UUID | PK | Unique identifier |
| asset_id | UUID | FK | → ASSET — source asset |
| recorded_at | TIMESTAMP | | When reading taken — staleness check |
| source | ENUM | | SCADA · MANUAL · EMS_IMPORT |
| current_load_mva | DECIMAL(8,3) | | Current loading |
| rated_capacity_mva | DECIMAL(8,3) | | Rated capacity |
| loading_percentage | DECIMAL(5,2) | | current_load ÷ rated_capacity × 100 |
| available_margin_mva | DECIMAL(8,3) | | rated_capacity − current_load — primary optimizer input |
| voltage_pu | DECIMAL(6,4) | | Current voltage in per unit |
| voltage_within_limits | BOOLEAN | | Within permissible band (0.95–1.05 pu) |
| frequency_hz | DECIMAL(6,3) | | System frequency |
| power_factor | DECIMAL(4,3) | | Current power factor |
| thermal_status | ENUM | | NORMAL · WARM · HOT · OVERLOADED |
| availability_flag | ENUM | | AVAILABLE · MARGINAL · UNAVAILABLE |
| marginal_threshold_pct | SMALLINT | | Loading % for MARGINAL flag (default 75) |
| unavailable_threshold_pct | SMALLINT | | Loading % for UNAVAILABLE flag (default 90) |
| outage_planned | BOOLEAN | | Source has planned outage in relevant window |

### SUPPLY_OPTIMIZATION_RUN

| Field | Type | Key | Description |
|---|---|---|---|
| run_id | UUID | PK | Unique identifier |
| triggered_by | ENUM | | OUTAGE_EVENT · CLUSTER_PLANNING · MANUAL · SCHEDULED_CONTINGENCY |
| triggered_at | TIMESTAMP | | When optimization triggered |
| scenario_type | ENUM | | SINGLE_OUTAGE · MULTI_OUTAGE · CONTINGENCY_N1 · CONTINGENCY_N2 |
| outage_event_ids | UUID[] | | Array of OUTAGE_EVENT IDs in scope |
| zones_affected | UUID[] | | Array of NETWORK_ZONE IDs |
| total_load_to_transfer_mva | DECIMAL(10,3) | | Total load needing alternative supply |
| alternatives_evaluated | SMALLINT | | Candidates evaluated |
| alternatives_feasible | SMALLINT | | Passing capacity and voltage checks |
| weight_loss_minimization | DECIMAL(4,3) | | Optimizer weight for loss minimization |
| weight_reliability | DECIMAL(4,3) | | Optimizer weight for reliability (sums to 1.0) |
| solution_found | BOOLEAN | | Feasible solution found |
| infeasibility_reason | TEXT | | If solution_found = false |
| total_additional_losses_mw | DECIMAL(8,3) | | Estimated I²R losses with selected arrangement |
| combined_reliability_score | DECIMAL(5,2) | | Weighted avg reliability of selected sources |
| consumers_restorable | INTEGER | | Consumers coverable by selected alternatives |
| consumers_not_restorable | INTEGER | | Consumers with no viable alternative |
| computation_seconds | DECIMAL(6,2) | | Solution computation time |
| algorithm_used | VARCHAR(50) | | e.g. GREEDY_MARGIN, LINEAR_PROGRAM |
| solution_notes | TEXT | | Rationale or operator annotations |

### SUPPLY_SWITCHING_PLAN

| Field | Type | Key | Description |
|---|---|---|---|
| plan_id | UUID | PK | Unique identifier |
| plan_reference | VARCHAR(20) | UQ | e.g. SSP-2026-04-019 |
| optimization_run_id | UUID | FK | → SUPPLY_OPTIMIZATION_RUN |
| outage_event_id | UUID | FK | → OUTAGE_EVENT |
| alt_source_id | UUID | FK | → ALT_SUPPLY_SOURCE |
| rank | SMALLINT | | 1 = primary, 2 = fallback, 3 = tertiary |
| load_to_transfer_mva | DECIMAL(8,3) | | Load assigned to this path |
| estimated_losses_mw | DECIMAL(6,3) | | Additional losses at planned load |
| margin_utilisation_pct | DECIMAL(5,2) | | % of available margin consumed |
| status | ENUM | | PLANNED · APPROVED · EXECUTING · EXECUTED · SUPERSEDED · CANCELLED |
| approved_by | UUID | FK | → WORKER — load dispatch authority |
| approved_at | TIMESTAMP | | Approval timestamp |
| execution_started_at | TIMESTAMP | | When switching crew began |
| supply_established_at | TIMESTAMP | | When alternative supply confirmed energized |
| consumers_restored | INTEGER | | Actual consumers restored |
| actual_losses_mw | DECIMAL(6,3) | | Measured losses after energization (nullable) |

---

## Domain 8 — Consumer Communication · 3 tables [v1.4 NEW]

### CONSUMER_COMMUNICATION_PREFERENCE [v1.4]

| Field | Type | Key | Description |
|---|---|---|---|
| preference_id | UUID | PK | Unique identifier |
| consumer_id | UUID | FK | → CONSUMER |
| channel | ENUM | | SMS · EMAIL · WHATSAPP · APP_PUSH · IVR · CRM_PORTAL |
| is_primary | BOOLEAN | | Primary channel for single-channel delivery |
| language | VARCHAR(10) | | ISO code e.g. hi, en, ur — for localised messages |
| planned_outage_consent | BOOLEAN | | Consent for planned outage advance notices |
| complaint_updates_consent | BOOLEAN | | Consent for complaint status updates |
| marketing_consent | BOOLEAN | | Consent for non-operational communications |
| opted_out_at | TIMESTAMP | | If consumer opted out (nullable) |

### CRM_NOTIFICATION [v1.4]

| Field | Type | Key | Description |
|---|---|---|---|
| notification_id | UUID | PK | Unique identifier |
| consumer_id | UUID | FK | → CONSUMER |
| task_id | UUID | FK | → TASK (nullable for outage-level) |
| outage_event_id | UUID | FK | → OUTAGE_EVENT (nullable for task-level) |
| cluster_id | UUID | FK | → TASK_CLUSTER (nullable for planned shutdown notices) |
| notification_type | ENUM | | COMPLAINT_RECEIVED · CREW_ASSIGNED · CREW_EN_ROUTE · CREW_ARRIVED · RESOLVED · DEFERRED · OUTAGE_DETECTED · ALT_SUPPLY · SUPPLY_RESTORED · PLANNED_OUTAGE_ADVANCE · PLANNED_REMINDER · PLAN_REVISED · PLAN_CANCELLED · EARLY_RESTORATION · SLA_AT_RISK |
| communication_type | ENUM | | REACTIVE · PROACTIVE · REGULATORY |
| channel | ENUM | | SMS · EMAIL · WHATSAPP · APP_PUSH · IVR · CRM_PORTAL |
| triggered_by_engine | ENUM | | SCHEDULE_ENGINE · MATCHING_ENGINE · CLUSTERING_ENGINE · SUPPLY_OPTIMIZER · OUTAGE_EVENT · MANUAL |
| message_template_id | VARCHAR(50) | | Reference to template in CRM system |
| message_variables | JSONB | | Dynamic values e.g. {"worker_name":"Ramesh","eta":"11:35"} |
| sent_at | TIMESTAMP | | When notification dispatched |
| delivered_at | TIMESTAMP | | Delivery confirmation (nullable) |
| read_at | TIMESTAMP | | Read receipt where available |
| delivery_status | ENUM | | PENDING · SENT · DELIVERED · READ · FAILED · OPTED_OUT |
| failure_reason | TEXT | | If delivery failed (nullable) |
| crm_reference | VARCHAR(50) | | Reference number in CRM system |
| regulatory_record | BOOLEAN | | True for planned outage advance notices — cannot be deleted |

### METER_ALERT [v1.4]

| Field | Type | Key | Description |
|---|---|---|---|
| alert_id | UUID | PK | Unique identifier |
| meter_id | VARCHAR(30) | | Smart meter ID from MDMS |
| consumer_id | UUID | FK | → CONSUMER (resolved on receipt) |
| alert_type | ENUM | | LAST_GASP · POWER_RESTORED · TAMPER · VOLTAGE_SAG · FREQUENCY_DEVIATION |
| received_at | TIMESTAMP | | When signal arrived from MDMS |
| processed | BOOLEAN | | True once converted to TASK or CONSUMER_OUTAGE_RECORD |
| task_id | UUID | FK | → TASK (nullable until processed) |
| raw_payload | JSONB | | Original MDMS message — preserved for audit |

---

## Key Design Decisions

1. TASK is the hub — every domain connects through it
2. Two SLA mechanisms — response-time for reactive, compliance-date for scheduled
3. Automatic task generation from MAINTENANCE_SCHEDULE — no manual creation of scheduled tasks
4. Two hard safety gates — pre-work and post-work isolation confirmation enforced by system
5. One outage event per cluster — direct mechanism linking planning decisions to reliability indices
6. Effective interruption accounts for alternative supply — SAIDI computed on actual impact
7. Alternative supply defined a priori — no ad hoc path selection under pressure
8. Optimization weights configurable — loss vs reliability balance set by operations policy
9. Three ranked supply options always produced — primary, fallback, tertiary
10. WORKER_SKILL.is_active is the matching engine safety gate — expired = excluded automatically
11. CREW_ASSIGNMENT.suggestion_rank captures the learning loop — override patterns train Phase 3 model
12. TEST_RECORD.test_readings uses JSONB — one schema accommodates all 15 task-type test formats
13. ASSET.parent_asset_id is a self-reference — models asset hierarchy without a junction table
14. SOURCE_OPERATING_STATE staleness is a safety concern — stale readings flagged before use
15. CRM publishes, FSM records — FSM owns the event trigger and audit trail, CRM owns channel delivery
16. METER_ALERT is a staging table — no smart meter signal is lost or duplicated
17. regulatory_record = true notifications cannot be deleted — compliance evidence preserved permanently
18. indoor_task drives adverse-weather rebatching — relay tests and battery inspections fill weather-blocked days
19. RELIABILITY_SNAPSHOT.saifi_events_saved_by_clustering — planning discipline made measurable and reportable
20. CONSUMER.meter_id links FSM consumer records to MDMS — enables last-gasp to consumer matching at ingestion

---

## Recommended Database Indexes

| Table | Index fields | Purpose |
|---|---|---|
| TASK | status, priority_class | Live dispatch queue |
| TASK | sla_deadline | SLA breach monitoring |
| TASK | asset_id | Asset task history |
| TASK | cluster_id, is_clubbed | Cluster queries |
| TASK | clubbing_opportunity_flagged | Planner dashboard |
| WORKER | status, workforce_pool | Available worker filter |
| WORKER | current_lat, current_lng | Proximity (PostGIS) |
| WORKER_SKILL | worker_id, skill_id, is_active | Matching engine |
| WORKER_SKILL | expiry_date | Expiry monitoring |
| ASSET | zone_id, status | Zone-level queries |
| ASSET | energization_status | Isolation planning |
| MAINTENANCE_SCHEDULE | next_due_date, is_active | Schedule engine |
| VAN_STOCK | van_id, quantity_on_hand | Parts check |
| SERVICE_HISTORY | asset_id, event_date | Asset history |
| SLA_TRACKER | is_breached, sla_deadline | SLA monitoring |
| WEATHER_EVENT | zone_id, started_at | Active weather |
| ISOLATION_RECORD | task_id, pre_work_confirmed | Safety gate |
| ISOLATION_RECORD | post_work_confirmed | Completion gate |
| OUTAGE_EVENT | zone_id, outage_start | Reliability reporting |
| CONSUMER_OUTAGE_RECORD | consumer_id, outage_event_id | CAIFI computation |
| RELIABILITY_SNAPSHOT | zone_id, period_start | Trend reporting |
| ALT_SUPPLY_SOURCE | target_zone_id, is_active | Optimizer query |
| SOURCE_OPERATING_STATE | asset_id, recorded_at | Latest margin |
| SOURCE_OPERATING_STATE | availability_flag | Feasibility filter |
| SUPPLY_SWITCHING_PLAN | outage_event_id, rank | Plan retrieval |
| CRM_NOTIFICATION | consumer_id, notification_type | Communication audit |
| CRM_NOTIFICATION | regulatory_record, sent_at | Compliance evidence |
| METER_ALERT | consumer_id, processed | Staging queue |

---

## Phase Build Map

| Domain | Phase 0 | Phase 1 | Phase 2 | Phase 3 |
|---|---|---|---|---|
| Task Management | Schema + types | Dispatch + isolation gates | Cluster engine | ML duration prediction |
| Workforce | Schema + skills | Matching engine live | Route assignment | Performance learning |
| Asset and Schedule | Schema + geocode | Link tasks to assets + schedule engine | Condition trending | Predictive maintenance |
| Inventory | Schema + catalog | Van kit check | Depot routing + reorder | Auto-reorder ML |
| History and Compliance | Schema design | Capture on completion | Asset condition analytics | Failure prediction |
| Outage and Reliability | Schema design | Manual outage recording | Automated impact + SAIFI/SAIDI | Predictive modelling |
| Supply Topology | Schema + catalog | Manual selection from catalog | Optimizer Mode 1 + margin check | Multi-outage + load flow |
| Consumer Communication | Schema design | Reactive notifications (CRM) | Proactive + regulatory notices | Predictive communication |
| Weather | Schema + fields | Capture at completion | API integration + blocking | ML weather-duration |

---

*End of Document — Field Service Management Data Model v1.4*
*7 domains · 40 tables · ~420 fields · Version date: April 2026*
