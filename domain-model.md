# RM Optimizer domain model and solver contracts

## Purpose and closed-loop product workflow

POC-4.1 extends the canonical contracts from a one-time roadmap generator into a
closed-loop planning model. Solver MVP Core consolidates those contracts with
daily scheduling, exact hard calendar deadlines, target-window Value, forecast
overflow, deterministic benchmarks, and human-verifiable scenarios. Persistence,
UI, Jira monitoring/write-back, and HTTP APIs remain outside this contract layer.

The target workflow is:

```text
Jira backlog -> canonical planning inputs -> initial optimization
        -> Published Roadmap
        -> Execution / Jira changes / employee changes
        -> Planning Inbox
        -> Manager accepts planning-impacting changes
        -> Execution Snapshot
         + Approved Planning Inputs
         + Existing Pins
         + Current Baseline Roadmap
        -> Reoptimization
        -> New DRAFT
        -> Change Preview
        -> Manager adjustments / pins
        -> Re-optimize if needed
        -> Publish
```

All schemas use JSON Schema Draft 2020-12. `schemaVersion` versions payload
shape, and `$id` identifies the schema. Optimization runs retain immutable input
and output snapshot references so results can later be audited or reproduced with
the same solver version.

Solver input/output contract `schemaVersion: 3.0.0` remains the frozen Solver
MVP v1 person-day contract. The canonical planning-model document is version
`5.0.0`: Planning Semantics v1.1 is a breaking application-contract extension;
it retains the prior requirement for
`projectConfig.effortUnit: PERSON_DAYS` and rejects the stale `HOURS` value.
Embedded solver snapshots inside that planning document remain version `3.0.0`.

## Plan vs Actual vs Forecast

The model deliberately separates three meanings.

### Business constraint

`deadline` is the date by which the business requires delivery. Every deadline is
a hard input constraint; RM Optimizer has no soft-deadline mode. The application
compiles the canonical date into solver form such as
`{"periodId": "2026-10-14"}`. Each solver period is exactly one calendar date,
so selected work must complete on or before that date. Deadline does not force
selection by itself, and it is not a prediction.

### Published plan

`RoadmapAssignment` stores the commitment published by RM Optimizer:

- `plannedStart`
- `plannedEnd`
- `plannedEffortDays`
- `plannedAssigneeResourceId`
- planned allocations by period

### Execution actuals and current forecast

`WorkItem` and `ExecutionSnapshot` describe reality at a point in time:

- `actualStart`
- `actualCompletedAt`
- `actualSpentEffortDays`, when the source provides it
- `progressFraction`
- `remainingEffortDays`
- `forecastEnd`
- `currentAssigneeResourceId`

The invariant is conceptual and explicit:

```text
deadline != plannedEnd != forecastEnd
```

For example, a task can have a 30 November deadline, a published planned start of
1 October and planned end of 31 October, and—after a 15 October re-estimate—a
current forecast end of 15 November. Updating the forecast does not silently
modify either the deadline or the published roadmap. A new plan is produced only
through review and reoptimization.

## Effort vs Duration

Effort is required work and is canonically measured in **person-days**. One
person-day is one full workday of reference effort for one eligible primary
assignee. It is independent of which eligible employee receives the task.
Duration is elapsed calendar time between start and finish: five person-days take
about five available dates at capacity `1.0`, or about ten available dates at
capacity `0.5`. The application may convert raw Jira estimates from another unit
before constructing the canonical input; the solver consumes person-days only.

The optimizer normally derives duration from effort, resource capacity,
dependencies, constraints, and scheduling. Users do not enter duration for every
task. Optional `minimumDurationPeriods` leaves room for work with intrinsic
calendar duration—external approval, an experiment, a waiting period, or a fixed
migration window—but is not mandatory in MVP.

CP-SAT keeps deterministic integers internally: one tick is `0.25` person-day.
Required effort and handover use `ceil(personDays * 4)`; available capacity uses
`floor(capacityDays * 4)`. This never underestimates work or overestimates
capacity, and ticks never cross the public contract boundary.

## Target planning horizon vs schedule/forecast horizon

The solver horizon separates two product questions:

- `startDate -> targetEndDate` is the manager-selected business-value window;
- `startDate -> scheduleEndDate` is the longer finite forecast window.

All supplied daily periods and compiled `capacityByPeriod` values cover the
schedule horizon. Every period ID, start date, and end date is the same calendar
date; dates are contiguous without gaps. The application layer generates them;
the solver never creates missing or arbitrary overflow periods. `targetEndDate`
must exist as an exact period. Only work completed on or before that target date
contributes Delivered Value.

DAY is the approved solver resolution because hard deadlines, single-day
absences, mid-week vacation boundaries, reduced availability, re-estimates, and
mid-quarter reoptimization all require exact dates. A weekly grid would need
irregular split weeks around those boundaries and approaches a daily grid as the
number of boundaries grows. Roadmap UI may still group daily results into weeks,
sprints, or months; presentation granularity does not alter solver resolution.

Work completing later is carryover and contributes zero Value to the target
objective. Crossing target end is not itself infeasible: it allows honest
completion forecasts for already-started, Must-Have, and `FORCE_INCLUDE` work.
Missing a deadline is never allowed, including when the deadline is in overflow.
Mandatory work that cannot fit by `scheduleEndDate` is infeasible. The
application should supply a schedule horizon long enough for committed work and
every applicable deadline.

## Jira raw data and approved canonical inputs

Jira configurations differ by project. Fields may be absent, customer-specific,
empty, or semantically different despite similar names. Raw Jira data therefore
does not form a stable solver API.

`WorkItem.source` retains Jira identity (`issueId`, `key`, `projectId`), while
canonical fields contain normalized planning values and provenance. Raw detected
changes are represented by `PlanningChangeRequest`. Accepted values are compiled
into an `ApprovedPlanningInputSet`; only that accepted/canonical boundary feeds
the solver.

A pending Jira change does not automatically alter production planning
assumptions. This also supports manual planning and future non-Jira inputs without
changing solver contracts.

## Field mappings, manual fallback, and RICE

A project-scoped `FieldMappingProfile` maps canonical Value, Effort, Deadline,
Required Role, and Must Have through these modes:

- `JIRA_FIELD`
- `MANUAL`
- `JIRA_FIELD_WITH_MANUAL_OVERRIDE`

`manualFallbackAllowed` is deliberately `true`. Missing or unusable Jira fields
must lead to manual input, not an unusable model. Teams may manually enter Value,
Effort, or both. Provenance retains effective source, Jira field ID, raw value,
transform, manual override, and optional capture time. A manual remaining-effort
re-estimate is a normal first-class input.

RICE is retained as an auxiliary signal with `semanticType: RICE` and
`semanticCategory: COMPOSITE_PRIORITIZATION_SCORE`. It is not silently converted
into pure Value and does not prohibit separate Effort. Story Points can be mapped
through a manager-confirmed `SCALE` or `LOOKUP_TABLE` transform. The application
also supports `IDENTITY` when a source field is already canonical. Unknown lookup
values are never interpolated.

## Hierarchy and dependencies

Hierarchy describes decomposition through `hierarchyParentWorkItemId`; it does
not imply scheduling order. Dependencies are explicit MVP
`FINISH_TO_START` relationships. “PEN-1 blocks PEN-2” becomes:

```text
prerequisiteId = wi-pen-1
dependentId    = wi-pen-2
```

No dependency is inferred from issue order, priority, type, hierarchy, or text.
Completed prerequisites satisfy dependencies and consume no future capacity.
For active work, selecting a dependent selects its unfinished prerequisite and
requires `completionDate(prerequisite) < startDate(dependent)`. A prerequisite
finishing Monday permits the dependent on Tuesday, never later on the same day.
The MVP intentionally has no intraday dependency ordering.

Planning Semantics v1.1 keeps three graphs separate. Structural hierarchy is
only decomposition, internal `FINISH_TO_START` is executable precedence, and an
`ExternalDependencyGate` is a lower-bound fact about work outside the team's
optimization backlog. Parent/child edges are never compiled into dependencies.

`planningKind` is determined only by a team structure policy keyed by stable
`workItemTypeId`. `EXECUTABLE` work may consume effort, ownership, allocation,
and WIP. `CONTAINER` work consumes none of them and is omitted from solver tasks;
it may carry Value. A value-bearing container compiles to a `DeliverableGroup`
whose executable descendants must all finish in the target horizon to earn the
group Value. DONE descendants already satisfy their member condition. Member
source Values remain canonical history but are suppressed from Objective 1, and
overlapping/nested Value ownership is a semantic error.

## WIP and concurrency

`TeamWipSettings` defaults to `SIMPLE`: one positive
`defaultMaxConcurrentTasks` is the total active-task cap for every team
resource. Persisted advanced policies are retained but ignored. `ADVANCED`
resolves total caps by resource, then primary planning role, then team default;
type caps resolve by resource+stable type, then primary role+stable type, then no
additional type cap. Total and type caps apply together. Additional eligibility
roles never participate in WIP inheritance.

For TODO, an active interval is inclusive from planned start through completion.
For IN_PROGRESS it starts on `optimizationDate`. Intermediate zero-allocation
dates remain active. DONE and CONTAINER records do not count. Resolved caps are
hard CP-SAT cumulative constraints; an existing committed overload is
`CONCURRENCY_LIMIT_CONFLICT`, never an implicit relaxation.

## External dependency gates

External work is customer-side planning context, not a solver candidate. DONE
uses factual `actualCompletedAt`; otherwise a manager date overrides a usable
future platform deadline. A stale deadline for unfinished work becomes
`OVERDUE_UNRESOLVED`. Missing or inaccessible dates create
`EXTERNAL_DEPENDENCY_DATE_REQUIRED` and block the dependent from solver input
without manufacturing `FORCE_EXCLUDE`.

For resolved gates, the latest completion date across all gates is used and the
next date becomes `earliestStartDate`. The solver enforces only
`plannedStart >= earliestStartDate`; it never requires equality. External work
therefore consumes no RM capacity, assignee, selection variable, or WIP slot.

## Work item execution state

`WorkItem.originalEffortDays` preserves the original planning estimate and its
provenance. `remainingEffortDays` is the primary future planning input.
`actualSpentEffortDays` is optional because not every Jira/team workflow records
it. `progressFraction` is nullable when unknown and otherwise bounded from 0 to 1.

State semantics are:

- `DONE`: remaining effort is zero, future capacity is zero, dependencies are
  satisfied, and execution data remains historical.
- `IN_PROGRESS`: only remaining effort can be scheduled in future periods;
  completed historical work is never rescheduled.
- `TODO`: the item is a normal future candidate.

Status category remains authoritative. The retained `started`/`completed` flags
serve application UX but must be semantically consistent with it.

## Execution Snapshot

An `ExecutionSnapshot` freezes actual/current execution state at a reoptimization
point. It records project, capture time, optimization date, source published
roadmap, and per-item status, assignee, actual dates/spend, remaining effort,
progress, and forecast end.

The POC-4.1 example captured on 15 October contains:

- PEN-1: DONE and historical;
- PEN-2: IN_PROGRESS, progress 0.50, 4.5 accepted remaining person-days;
- PEN-3 and PEN-4: TODO.

The solver input retains the snapshot for cutoff and handover semantics. Periods
before `optimizationDate` expose zero future capacity, while historical baseline
allocations remain available for audit. Application-level semantic validation
must ensure no output reschedules work before the cutoff.

## Remaining-effort re-estimation

An effort change is not an invisible overwrite. `EFFORT_REESTIMATED` stores the
previous and proposed remaining person-days, source/actor, timestamp, review
state, and review metadata. For example, PEN-2 changes from 2.5 to 4.5 remaining
person-days.

After manager acceptance, the canonical WorkItem records 4.5 person-days with
`basis: ACCEPTED_REESTIMATE`, provenance retains the manual override, and the
Approved Planning Input Set carries 4.5 into the next solver input. The prior
published assignment still preserves its original planned effort.

## Planning Change Requests

`PlanningChangeRequest` is the generic planning-impact event. Supported types are:

- `AVAILABILITY_CHANGED`
- `EFFORT_REESTIMATED`
- `DEADLINE_CHANGED`
- `FORECAST_CHANGED`
- `ASSIGNEE_CHANGED`
- `SCHEDULE_VARIANCE_DETECTED`
- `TASK_ADDED`
- `TASK_CANCELLED`
- `SCOPE_CHANGED`
- controlled future `CUSTOM_*` values

Each request records project, affected entity, previous/proposed values, source,
creator or detector, timestamp, review state, review metadata, and optional Jira
or availability-request origin references. Source types are
`EMPLOYEE_SUBMITTED`, `MANAGER_SUBMITTED`, `JIRA_DETECTED`, and
`SYSTEM_DETECTED`. Review states are `PENDING_REVIEW`, `ACCEPTED`, and `REJECTED`.

AvailabilityRequest remains the employee-facing source record; its accepted
planning impact is represented as an `AVAILABILITY_CHANGED` change request.

## Manager review semantics

Review approves a planning assumption; it does not necessarily mutate the source
system.

For an employee availability request, `ACCEPTED` means the adjustment becomes a
planning input and changes compiled capacity. `REJECTED` means it does not affect
capacity.

For a Jira-detected deadline edit, Jira has already changed. `ACCEPTED` means “use
the detected value in future planning assumptions.” `REJECTED` means “do not
incorporate it into the current assumption set yet.” It does not revert the Jira
field. POC-4.1 demonstrates a Jira deadline change that remains
`PENDING_REVIEW`, so the previous approved deadline is still sent to the solver.

## Planning Inbox

Planning Inbox is a future manager view, not UI implemented in POC-4.1. It groups
pending planning-impact changes such as:

```text
PEN-17  Remaining effort: 3d -> 6d
PEN-44  Deadline: 30 Nov -> 31 Oct
Ivan    Unavailable: 21-25 Oct
```

A manager can inspect a change, preview likely impact, accept or reject it for
planning, and trigger a reoptimized DRAFT. Alert thresholds and automatic inbox
generation policies remain configurable future behavior.

## Approved planning assumptions

Raw/current state and approved assumptions are separate contract layers.
`ApprovedPlanningInputSet` references one Execution Snapshot and only accepted
Planning Change Requests. It materializes accepted remaining effort, deadline,
current assignee/status, and compiled capacity.

Pending or rejected changes are excluded. `OptimizationRun` and solver `run`
reference the approved set and the accepted change IDs, making the decision
boundary auditable without asking the solver to interpret review workflows.

## Plan Variance

`PlanVariance` is a derived comparison between the published assignment and
actual/current state. Supported concepts include:

- `START_DELAY`
- `FINISH_DELAY`
- `EFFORT_OVERRUN`
- `ASSIGNEE_DEVIATION`
- `INCOMPLETE_AFTER_PLANNED_END`

It records the planned/current fields, both values, and a typed JSON delta. A
planned end of 31 October and forecast end of 15 November can be represented as a
two-period delay. A variance may generate `SCHEDULE_VARIANCE_DETECTED`, but not
every variance must alert the manager; thresholds are future policy.

## Completed-task presentation and performance

DONE work remains canonical history, remains searchable, and may satisfy
dependencies, but it is not a future optimization candidate and has no active
Include-in-optimization checkbox. The target task list places active planning
work first and presents history in a collapsed `Completed (N)` section. Expanding
it shows completed rows. If search matches a completed item, the future UI must
reveal that result even when the section was collapsed. Collapse state is
presentation state only and never alters solver input.

Completion performance is derived, not manually persisted when its source dates
exist. Deadline performance compares the calendar date of `actualCompletedAt`
with `deadline`:

- completion on or before deadline -> `ON_TIME`;
- completion after deadline -> `LATE`, with `lateByCalendarDays` equal to the
  positive calendar-day difference.

This does not weaken planning deadlines. A selected future plan may never finish
after its hard deadline, but observed execution history may contain
`actualCompletedAt > deadline`; reality missing a commitment is valid historical
data, not an invalid solver schedule.

Plan performance is a separate comparison with the **published** `plannedEnd`:

- earlier -> `EARLY_BY_N_DAYS`;
- equal -> `ON_PLAN`;
- later -> `LATE_VS_PLAN_BY_N_DAYS`.

Both metrics use calendar days. If a deadline exists, the compact primary badge
is `On time` or `Late by N days`, and plan variance may be secondary text. With
no deadline, show plan variance when published `plannedEnd` exists; with neither
reference, show only `Completed`. For example, deadline 30 November, published
end 15 November, and actual completion 20 November is on time but five days
later than plan.

An incomplete task observed after its deadline is a breached planning assumption.
The application must not silently move the deadline; a future planning/execution
workflow should surface it for manager attention. Solver MVP v1 does not redesign
Planning Inbox for this edge case.

## Resource identity, eligibility, and capacity

Before work is scoped, an RM Team selects a `TeamMembershipSource`:
`ATLASSIAN_TEAM` or `MANUAL`. Effective membership is platform membership union
manual inclusions minus manual exclusions. Assignees are execution metadata and
never membership evidence. `TeamWorkScopePolicy` independently selects work by
the stable Atlassian Team field or an explicitly confirmed Jira board; only then
are dependencies classified.

The default working week is Monday-Friday and each effective member has a
default `planningAllocationFraction` of `1.0`, constrained to `(0, 1]`. Daily
capacity is working-day base multiplied by allocation, then by an approved
bounded availability fraction. Approved vacation/day-off produces zero;
pending/rejected availability has no effect. Public holidays are not inferred.

Atlassian identity (`accountId`) is separate from RM Optimizer roles, level,
skills, nominal schedules, and allocation. Solver resources contain only internal
IDs and planning attributes—never display names, avatars, or emails. Roles and
skills answer only whether a resource is eligible; every eligible employee uses
the same canonical person-day effort.

The Atlassian Team adapter uses `accountId` as the stable external Resource
identity. Display name and avatar are optional presentation metadata. Atlassian
membership state and Team role are retained at the adapter boundary, but Team
roles such as `ADMIN` and `REGULAR` are never mapped to RM planning roles. If an
edge hides `accountId`, synchronization reports partial data and preserves prior
platform membership rather than inventing or deactivating that identity.

Person-specific productivity is **TARGET / NOT MVP**. Solver v1 has no
productivity field and does not accept or silently interpret role, level, skill,
or employee speed multipliers. A future milestone must first approve the semantic
meaning of reference effort before adding such a separate capability.

Approved availability is compiled before solving:

```text
daily period capacity
  = nominal workday fraction
  × planning allocation
  × approved availability adjustment
```

The solver consumes only `capacityByPeriod`; it does not need the sensitive reason
for an absence. Vacations, days off, holidays, reduced availability, and partial
current-day capacity remain application/compiler decisions. The solver consumes
only the resulting daily fraction, bounded from `0.0` to `1.0` person-day per
calendar date.

## Reassignment handover cost

`ReassignmentPolicy` configures real one-time future effort caused by transferring
already-started work. POC-4.1 supports:

- `progressThreshold`
- `fixedOverheadDays`
- `overheadPercentOfRemainingEffort`

Conceptual future solver semantics are:

```text
if progressFraction >= progressThreshold
and proposed assignee != currentAssigneeResourceId:
    handoverOverhead = fixedOverheadDays
                     + remainingEffortDays
                     * overheadPercentOfRemainingEffort
    effectiveRemainingEffort = remainingEffortDays + handoverOverhead
```

The Solver MVP Core implements this formula as real additional future effort on
the new assignee. Policy may later become team-, role-, skill-, or empirically
configured. It is configuration, not ML.

## Handover vs roadmap churn

Handover and churn are independent effects that may coexist.

- Roadmap churn is a soft preference against unnecessary differences from the
  published baseline: selection changes, assignee changes, and start/end shifts.
- Handover is actual additional capacity consumed when started work changes owner.

An in-progress reassignment can therefore incur a churn penalty and add handover
person-days. Handover must not be modeled merely as an artificially huge churn
weight.

Daily churn weights are applied only inside Objective 2: selection change 1000,
assignee change 100, TODO start-date shift 1 per calendar day, and completion-date
shift 1 per calendar day. IN_PROGRESS historical start is not penalized; its
future completion and assignee may be. DONE work has no future churn.

## Manual hard constraints

Supported constraints are `FORCE_INCLUDE`, `FORCE_EXCLUDE`, `PIN_ASSIGNEE`,
`PIN_START_PERIOD`, `PIN_END_PERIOD`, and `PIN_PERIOD_RANGE`. They preserve target,
creator, timestamp, persistence, and roadmap origin. Must Have compiles to
`FORCE_INCLUDE`; it is not an artificially large Value. Persistent pins are
passed unchanged into reoptimization.

A pin to an assignee does not itself force selection unless accompanied by
`FORCE_INCLUDE`. Hard constraints must be satisfied or produce a structured
`INFEASIBLE` result.

## Published roadmap, draft, and baseline

Roadmap versions are immutable after publication and linked by `parentVersionId`.
Statuses are `DRAFT`, `PUBLISHED`, and `ARCHIVED`. A solver result creates a DRAFT;
employees continue to see the PUBLISHED version until a manager publishes a
replacement.

Roadmap assignments preserve planned dates, effort, assignee, selection, and
per-day allocations. Tasks can span or skip dates. During reoptimization, the current
published version is a soft baseline—not an implicit hard lock. Completed history
is retained, while only remaining/future work is optimized.

## Deadline vs Planned End and Jira write-back

Input mappings and output/write-back mappings are separate. Input fields include
Deadline/Due Date, Value, Effort, and Required Role. Optional output mappings are
RM Planned Start, RM Planned End, RM Planned Effort, and Planned Assignee.

`ProjectConfig.writeBackConfig` defaults to disabled in the example and accepts
explicit Jira custom-field mappings. RM Planned End is never mapped to Jira Due
Date by default:

```text
deadline != plannedEnd
```

Otherwise the solver could write its prediction into the business constraint and
later read its own output as a deadline. POC-4.1 implements no write-back.

## Solver input/output and reoptimization boundary

Solver input remains Jira-independent and includes only compiled planning data:

- run trigger, optimization date, approved input set ID, and accepted change IDs;
- optional current Execution Snapshot;
- generated daily target and schedule/forecast horizon;
- normalized Value, remaining Effort, progress, role/skills, deadline, Must Have,
  and current assignee;
- compiled capacity and resource planning attributes;
- Finish-to-Start dependencies;
- persistent hard constraints;
- published roadmap baseline;
- Reassignment Policy;
- ordered objectives.

The snapshot carries historical context required to freeze completed work and
evaluate handover. Raw inbox/review records, sensitive absence semantics, mapping
provenance, and Jira UI identity remain outside solver data.

Successful solver output provides assignments, deferred reasons, Value/carryover/
churn summaries, resource metrics, and changes from baseline. Each selected
assignment exposes `completionWithinTargetHorizon`, and summary exposes
`carryoverItems`. `INFEASIBLE` returns
structured reasons rather than “no solution”; `ERROR` is reserved for execution
or contract failure. Optional `effortBreakdown` reports base remaining effort,
handover overhead, and effective effort separately. The result is converted into
a DRAFT, not published directly.

The CP-SAT allocation formulation is sparse. A task/resource/date variable is
created only when the resource satisfies role and all skills, the date is not
historical, that resource has positive compiled capacity, the task deadline has
not passed, and the date remains inside every safely derivable pin boundary.
`PIN_PERIOD_RANGE` removes outside dates, `PIN_START_PERIOD` removes earlier
dates, and `PIN_END_PERIOD` removes later dates. Zero-capacity dummy variables
are never created. Finish-to-Start dependencies are intentionally not used for
aggressive pre-pruning because predecessor dates are solver decisions; their
exact constraints remain in the mathematical model.

## Optimization objectives and resource utilization

Hard constraints—including every deadline—are satisfied first. Solver MVP Core uses
true sequential lexicographic optimization:

1. `MAXIMIZE_DELIVERED_VALUE_WITHIN_TARGET_HORIZON`
2. `MINIMIZE_ROADMAP_CHURN`
3. `MINIMIZE_SCHEDULE_COST`

Schedule cost is `sum(scaledValue[i] * completionDayIndex[i])` over selected
non-completed work. There is no lateness objective because deadline violations
are infeasible, not penalized outcomes. The objectives are not a casual weighted
sum.

`MAXIMIZE_RESOURCE_UTILIZATION` is no longer a required objective. Utilization is
a reported metric (`availableDays`, `plannedDays`, `utilization`) for bottleneck
and under/over-utilization visibility. It must never affect assignment or displace more
valuable feasible work.

## Employee and planner access/UX model

This is an application-level model, not a Forge permission change.

Employee capabilities:

- `VIEW_ROADMAP`
- `VIEW_MY_TASKS`
- `EDIT_MY_AVAILABILITY`

Employees see the published Roadmap with “All team” and “My tasks” filters, task
details, inclusion state, and My Availability. They may search/filter but cannot
change optimization inclusion. They do not run optimization or modify mappings,
resources or publication state.

Planner capabilities:

- `VIEW_ROADMAP`
- `VIEW_MY_TASKS`
- `EDIT_TEAM_CONFIG`
- `EDIT_MAPPING`
- `EDIT_RESOURCES`
- `APPROVE_AVAILABILITY`
- `RUN_OPTIMIZATION`
- `PIN_ROADMAP`
- `PUBLISH_ROADMAP`
- `WRITE_BACK_TO_JIRA`

Managers additionally see Planning Inbox, impact/change preview, pins,
draft/publish controls, mappings, and team capacity. `WRITE_BACK_TO_JIRA` remains
a future capability.

Every active task has one explicit `Include in optimization` state. Checked is
the default. Unchecking compiles to the existing persistent `FORCE_EXCLUDE` hard
constraint; checking again removes that constraint. It is not a second solver
selection concept. TODO may be toggled by a planner. IN_PROGRESS remains committed,
so attempting to exclude it is an explicit planning conflict rather than silent
abandonment. DONE is historical and its checkbox is disabled/not actionable.

Search covers at least Jira key and title. Filters may include issue type, status,
current/planned assignee, required role, hierarchy, Must Have, inclusion,
Selected/Deferred/Carryover, deadline range, execution state, and dependencies.
Search and filters are view state only: they never remove hidden tasks from solver
input. Only the explicit inclusion control changes optimization scope.

## MVP assumptions and explicit non-goals

MVP domain support includes effort re-estimation, availability and hard deadline
changes, plan variance, Execution Snapshot, remaining effort, manager review,
reoptimization, persistent pins, and roadmap versioning. Planning uses contiguous
daily buckets, person-days, one primary assignee, one primary required role, zero or
more skills, interruptible multi-day work, and Finish-to-Start dependencies.

Still outside the Solver MVP Core and contract scope:

- actual event monitoring or polling;
- automatic Jira change detection code;
- employee UI and manager Planning Inbox UI;
- notification delivery;
- Jira write-back;
- HTTP/persistence and Forge-to-solver integration layers;
- person-specific productivity or speed adjustment;
- ML estimation of performance or effort;
- HR/calendar integrations;
- multiple simultaneous assignees;
- complex Dev -> QA -> Dev internal phases;
- cross-team portfolio optimization;
- arbitrary dependency semantics beyond Finish-to-Start;
- advanced drag-and-drop roadmap behavior.

## Semantic validation beyond JSON Schema

JSON Schema validates structure and local bounds. Application-level validation
must additionally check cross-object ID integrity, uniqueness, chronological
ordering, dependency cycles, allocation totals, plan/current consistency,
accepted-only change references, overlap/conflict among hard constraints, and the
rule that no output reschedules history before `optimizationDate`. These rules are
kept out of brittle schema conditionals.

## Decisions remaining after Solver MVP Core

- whether a post-MVP person-specific productivity model is justified, and its
  explicit reference-effort semantics if introduced;
- application compilation of calendars, timezone boundaries, holidays, and
  partial current-day capacity;
- whether an in-progress current assignee is only a churn baseline or can become
  an implicit pin under product policy;
- variance alert thresholds and impact-preview policy;
- constraint conflict precedence and accepted-assumption versioning;
- canonical production URI/registry strategy for schema `$id` values.
