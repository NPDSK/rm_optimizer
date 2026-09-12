# Application Core architecture

## Dependency direction

```text
Presentation / future Forge UI
          ->
Application Core workflows
          ->
Domain records + ports
          <-
Platform, persistence, solver and telemetry adapters
```

Application Core owns product workflows such as optimization, roadmap
publication, mapping activation and optimization-scope changes. It receives
ports through dependency injection and never imports Jira REST, Forge SQL,
Forge Container, or an analytics SDK. `OptimizationService.optimize(input)` is
the stable boundary around Solver MVP v1; the production
`SolverInputBuilder.build(state)` now compiles approved Planning Semantics into
that boundary. The frozen Python implementation remains outside this layer.

Canonical identifiers are platform-neutral: `platformType`,
`externalProjectId`, `externalTeamId`, `externalWorkItemId`,
`externalResourceId`, and `externalFieldId`. A future `JiraAdapter`,
`LinearAdapter`, `ClickUpAdapter`, or `MondayAdapter` may translate platform
payloads without changing workflows. This milestone implements the Jira read
adapter and Forge SQL repositories while retaining those interfaces.

## Implemented workflows

- `getEffectiveTeamMapping`: resolves only the team's authoritative mapping
  pointer and reports `NEEDS_SETUP` or `NEEDS_REVIEW` instead of guessing.
- `saveTeamMappingProfile`: materializes and validates a new version before the
  pointer switch; a BUILDING version is never effective.
- `updateOptimizationScope`: maps the TODO checkbox to persistent
  `FORCE_EXCLUDE`, protects DONE and reports IN_PROGRESS conflict.
- `optimizeRoadmap`: records the run lifecycle, compiles through a port, calls a
  replaceable optimizer, distinguishes infeasibility from failure, creates only
  fully materialized DRAFT roadmaps, and emits safe telemetry.
- `publishRoadmap`: publishes only a materialized DRAFT and changes the project
  pointer before reconciling version statuses.
- `getPlanningSetup`: persists Jira project identity and idempotently ensures one
  default RM planning Team while leaving room for later additional teams.
- `getAvailableFields`: returns stable Jira field identity and display/type
  metadata through the platform-neutral port.
- `syncPlanningData`: revalidates the ACTIVE profile, reads every enhanced-search
  page, applies transforms/overrides, persists current canonical state and Blocks
  dependencies, classifies out-of-Team prerequisites as external gates, appends
  meaningful history, and emits safe summary telemetry.
- planning settings operations persist SIMPLE/ADVANCED WIP policies and a
  separate stable-type structure policy.
- external dependency operations query gates and set/clear RM-only manual date
  assumptions while immediately refreshing planning readiness.
- `getCanonicalPlanningSummary`: returns the persisted current-state aggregate;
  it never invokes Solver MVP v1.

Correlation-ID generation and time are injectable. The correlation ID travels
through `OptimizationRun`, telemetry, diagnostics and future adapter logging.
Search and filters remain presentation state and never alter optimization scope.

## Domain boundary

The lightweight JavaScript entities describe application and persistence state;
they do not duplicate CP-SAT variables or Python solver validation. Person-day
effort, daily capacity and the v3 solver input/output contract remain the
canonical solver boundary.

Planning Semantics v1.1 adds an application compilation step before that
boundary. It resolves SIMPLE/ADVANCED WIP precedence into per-resource total and
type caps, applies the team's stable-type structure policy, compiles
value-bearing containers into DeliverableGroups, and resolves external gates to
earliest-start lower bounds or planning issues. Python deliberately receives no
role/person WIP precedence and no external platform records. Only executable
work reaches the solver, while explicit dependencies pass through unchanged.

## Jira and resolver boundary

Jira REST paths and response shapes stay inside `JiraAdapter`. Backend calls use
the invoking user through `@forge/api` and current REST v3 endpoints, including
`POST /rest/api/3/search/jql` token pagination. The resolver exposes planning
setup, available fields, mapping save, canonical sync/summary, planning settings
and external dependency assumption operations.
It derives the numeric project ID from the Forge backlog-action context and does
not accept a caller-selected cross-project context.

## Team foundation and capacity

Team membership is resolved before work scope and is never inferred from issue
assignees. `TeamMembershipSource` selects either an Atlassian Team adapter or
manual membership. Platform memberships combine with durable manual inclusions
and exclusions; exclusions win across later refreshes. Resource rows are
retained when membership becomes inactive.

`TeamWorkScopePolicy` is separate. Stable Jira Team-field identity selects
matching work, including unassigned issues; board scope requires explicit
confirmation. Scope is applied before Blocks links are classified as internal
Finish-to-Start dependencies or external dependency gates.

The production builder compiles capacity only for effective active members. The
pure compiler applies working week, planning allocation, then approved
availability. Employee role/skill proposals remain pending until manager review;
manager direct edits apply immediately.

Atlassian Team membership transport is behind a capability port. The production
adapter uses Forge `api.asApp().requestGraph` to read `team.teamV2` for only the
persisted, manager-confirmed Team ID and current Forge `cloudId`. Team IDs are
normalized to `ari:cloud:identity::team/<team-id>`. The request has no
speculative beta header; gateway scope, grant, lifecycle, schema and rate-limit
errors become controlled customer-scoped diagnostics. It never calls the Teams
REST API, and on-demand Jira user search/manual membership remains the fallback.

## Planning Team context: first multi-Team milestone

`teams.platform_type` and normalized `teams.external_team_id` identify an
Atlassian Planning Team across the installation (unique index v039). The existing
`project_id` remains origin metadata for Team Foundation; project-based canonical
planning workflows have not yet been migrated to cross-project planning.

`getPlanningTeamContext` resolves the requested external identity server-side.
Unambiguous legacy membership sources bind the existing internal Team ID without
copying child records. Multiple legacy claims or conflicting source/scope/platform
membership evidence return `LEGACY_TEAM_BINDING_REQUIRED`; there is no automatic
merge or reassignment of ambiguous manual data. Bare IDs and Team ARIs resolve to
the same identity. Initialization uses insert-if-absent source/scope writes so a
retry cannot overwrite manual membership mode or an explicitly saved board scope.

The active external Team ID lives in App session state, shared by TeamSetup and
the issue list. Normal switching loads another Team and does not mutate the old
source. Manual setup changes membership mode within the selected Team. Foundation
resolver commands whitelist editable fields and resolve internal IDs from the
server; cloud and actor identity still come only from Forge context. RM ACL is
not implemented by this milestone.

`getTeamScopedIssues` reads persisted scope without requiring a mapping profile
or optimizer readiness. Team-field scope currently reads only the invocation
project, with pagination and stable Team matching; explicit board scope reads the
configured board. Responses are read as the invoking Jira user. The frontend
clears old issues on selection and ignores responses from superseded requests.
No last-selected preference, global Team directory, cross-project aggregation,
work-item duplication, roadmap pointer migration, or Python integration is added.
