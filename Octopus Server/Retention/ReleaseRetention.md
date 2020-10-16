# Release Retention

Release retention is intended to remove any releases that are no longer relevant so that the list does not get too long. It also removes `Deployments` and `Artifacts` clearing up space. [Package Retention](PackageRetention.md) looks at releases to determine which packages are still in use by releases, and therefor can't clean up if no releases are removed.

The retention runs as part of the `Apply Retention Policies` server task.

The retention policy is defined on the lifecycle and is either a number of days, or a number of releases. 

A release is kept if any of these conditions is met:
- It is on the project's dashboard
- It is in use by another release (via a deploy release step)
- It falls within the policy days or count

For the count policy, releases are ordered by `Assembled` (created) date descending. For each environment the top N are kept. The top N releases that have never been deployed are also kept.