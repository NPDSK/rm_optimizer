# Forge SQL logical persistence model

`persistence/design/001_application_core.sql` is the MySQL-compatible logical
design. The previously approved ingestion tables are implemented as ordered,
idempotent `migrationRunner` operations in `src/adapters/sqlMigrations.js`;
`manifest.yml` enables one Forge SQL module and an hourly schema trigger.
Planning Semantics v1.1 additions are implemented by ordered migrations and
runtime repositories together with the canonical-ingestion subset.

Storage is already scoped to one app installation, so customer-domain rows do
not repeat `tenant_id`. Stable string application IDs are primary keys. Logical
references are validated by repositories/Application Core: the design avoids
foreign keys, cascading delete and correctness that depends on multi-table
transactions.

Normal columns contain important lookup keys, statuses, dates and pointers;
flexible technical details use JSON-compatible text. Unique dedupe keys and
version keys make retries idempotent where practical.

The logical groups are:

- planning context: `projects`, `teams`;
- Team foundation: `team_membership_sources`, `team_platform_memberships`,
  `team_membership_overrides`, `team_work_scope_policies`,
  `team_capacity_policies`, `resource_planning_settings`;
- persistent mapping: `field_mapping_profiles`, `field_mappings`;
- people and eligibility: `resources`, `roles`, `skills`, `resource_roles`,
  `resource_skills`;
- planning state: `availability_events`, `work_items`,
  `work_item_dependencies`, `planning_constraints`,
  `work_item_planning_overrides`, `planning_change_requests`;
- roadmap publication: `roadmap_versions`, `roadmap_items`,
  `roadmap_allocations`;
- history: `execution_snapshots`, `execution_snapshot_items`,
  `work_item_execution_events`;
- operations: `optimization_runs`, `diagnostic_records`.

Planning Semantics v1.1 extends the logical design with
`team_wip_settings`, durable `wip_limit_policies`, and
`external_dependency_gates`; team structure policy, resource primary planning
role, canonical hierarchy parent, and stable work-item type/kind fields are also
represented. Advanced WIP rows remain persisted while SIMPLE mode ignores them.

`Team.current_mapping_profile_id` and
`Project.current_published_roadmap_id` are authoritative pointers. Readers never
discover effective state by choosing the newest row. Reconciliation can therefore
be retried after a partial infrastructure failure without exposing a BUILDING
profile or roadmap.

Current-state tables remain authoritative. Execution events record only
meaningful changes and use `(project_id, dedupe_key)` for idempotency; this is not
full event sourcing.

## Activated ingestion repositories

The real Forge SQL adapters cover Project, Team, Field Mapping, Resource,
Work Item, Dependency, Planning Override, Execution History and customer-scoped
Diagnostic persistence, plus Team WIP settings/policies, structure policy and
external dependency gates. Prepared statements bind every value. Upserts use
stable application IDs and platform external identities. Because Forge SQL
provides no multi-statement transaction, synchronization writes current rows
first and only then removes stale dependency/gate rows.

The SQL database is scoped per app installation. Adding `modules.sql` is a Forge
major-version change; an existing installation must be upgraded before the
database is provisioned. The scheduled trigger can run within the following
hour, so resolver calls must not be treated as proof that migrations already ran.

Team foundation rows do not duplicate people, roles or skills. Membership and
override rows reference retained `resources`; eligibility continues to use
`roles`, `skills`, `resource_roles` and `resource_skills`. Synchronization
deactivates stale membership rows rather than deleting historical resources. No
workflow assumes writes across these tables share a transaction.
