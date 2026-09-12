# Persistent team field mapping

A mapping profile belongs to exactly one Team and its Project/planning context.
Mappings are never globally shared across unrelated projects. A platform without
a native Team can use an RM Optimizer-owned Team with `externalTeamId: null`.

## Lifecycle and pointer strategy

1. Create a new profile version as `BUILDING`.
2. Copy and/or edit its `FieldMapping` entries.
3. Validate required mappings and available platform fields.
4. Mark the complete profile `READY`.
5. Switch `Team.currentMappingProfileId`.
6. Mark the new profile `ACTIVE`.
7. Archive the previous profile.

The pointer moves only after readiness. A failed or abandoned BUILDING profile
does not affect repeated visits, optimization, reoptimization, or roadmap
history; the old ACTIVE profile remains effective. Operations are designed to be
idempotently retryable rather than dependent on a cross-table transaction.

`getEffectiveTeamMapping` returns `READY` for a usable ACTIVE current profile,
`NEEDS_SETUP` when no pointer exists, and `NEEDS_REVIEW` for invalid, missing or
incompatible required mappings. Suggested matches from a future platform adapter
are not confirmed mappings.

## Stable identity and mapping state

`PLATFORM_FIELD`, `MANUAL`, and
`PLATFORM_FIELD_WITH_MANUAL_OVERRIDE` are platform-neutral source modes.
Platform mappings resolve exclusively by stable `externalFieldId`. Renaming a
field does not invalidate it; disappearance or type incompatibility produces
`MISSING`/`NEEDS_REVIEW`. Name similarity is never used to silently select a
replacement.

The extensible canonical registry currently represents Value, Effort, Deadline,
Must Have, Required Role and Required Skills. Adding a field preserves old
profiles; if the new field is required, those profiles naturally require review.

Mapping answers where a value normally comes from. Per-work-item manual values
and overrides belong to `work_item_planning_overrides`, not to a team mapping
profile. For example, `VALUE = MANUAL` is a mapping decision, while values 50 and
20 for two tasks are separate planning data.

## Persistent transforms and immediate activation

Each platform mapping persists one small transform configuration:
`IDENTITY`, positive finite `SCALE.factor`, or an explicit `LOOKUP_TABLE.values`
object. Unknown lookup keys produce `UNMAPPED_LOOKUP_VALUE`; no interpolation or
guessed person-day value is allowed. Canonical Effort is always person-days.

Saving performs BUILDING -> READY -> pointer switch -> ACTIVE against the
authoritative repository. The operation returns the new ACTIVE profile. No
mapping cache exists: an immediate `getEffectiveTeamMapping` and subsequent
`syncPlanningData` in the same open Custom UI session read the new SQL pointer.

`PLATFORM_FIELD_WITH_MANUAL_OVERRIDE` checks the active per-work-item override
first. `MANUAL` requires that override. Neither mode writes back to Jira.
