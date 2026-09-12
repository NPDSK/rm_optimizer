# Telemetry, diagnostics and analytics boundaries

## Safe product telemetry

`TelemetryPort` carries provider-neutral operational/product events. The current
adapters are no-op and in-memory only; there is no external egress. The envelope
supports version, time, app/environment/edition, hashed installation/project
identities, correlation/run IDs, actor category, outcome, error code, duration,
and allow-listed numeric/categorical metadata.

Safe telemetry rejects customer content and identifiers such as summary,
description, comments, email, display name, raw account ID, custom-field text,
and arbitrary platform payloads. This guard applies before an adapter stores or
sends an event.

Canonical synchronization uses `platform_sync_started`,
`platform_sync_completed`, and `platform_sync_failed`. Only counts, mapping
status, duration and categorical error codes are allowed. The configured runtime
adapter remains a validating no-op, so this milestone introduces no external
telemetry egress.

## Customer-scoped diagnostics

`DiagnosticRecord` is a separate troubleshooting channel. It has component,
severity, error code, message, structured details, stack trace, correlation/run
IDs and expiry. Diagnostics may later contain exact customer-specific technical
identifiers, but remain inside customer-scoped storage. No upload-to-support
workflow exists in this milestone.

Sync failures persist exact Jira operation/status/body details and stack traces
only through `DiagnosticRepository` in the installation-scoped database. Those
details never cross into `TelemetryPort`.

## Analytics readiness

Current state plus roadmap versions, snapshots and deduplicated execution events
retain enough facts to derive on-time completion, completion versus published
plan, initial effort versus re-estimates, carryover, roadmap stability,
reassignment frequency, role bottlenecks and trends. DONE work remains searchable
history for the future collapsed Completed section. No charts are implemented.

The same historical facts can later support estimate calibration, team-specific
forecasting and risk models, but this milestone implements no ML, personal speed,
productivity multiplier or bidding logic.

Customer tenant history is not Owner Analytics. Future owner-side analytics may
combine safe `TelemetryPort` events, Forge observability and Marketplace/license
reporting. No external analytics provider, warehouse or customer-content export
is introduced here.
