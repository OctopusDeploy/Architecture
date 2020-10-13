# Retention

Retention is an overloaded term in Octopus. There are several features called retention that do to quite different things and run at different times:

- [Target (Tentacle) Retention](TargetRetention.md). Runs at the end of a deployment.
- Release Retention.
- (Built-in) Package Retention
- Runbook Snapshot Retention
- Package Metadata Retention
- Event Retention
- Deployment Manifest Retention

All except `Target Retention` run as part of the `Apply retention policies` server task. `Target Retention` runs at the end of successful deployments.

To make it more confusing the first two, Target and Release retention, both use the settings on the Lifecycle. Package retention is affected by what releases and runbook snapshots are retained.