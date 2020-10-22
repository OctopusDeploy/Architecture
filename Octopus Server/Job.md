# Jobs

Jobs are background tasks that run periodically. There is a `JobRunner` that co-ordinates this. Jobs can be one of two types:
- `Per Node` - Runs on each node of a HA cluster
- `Per Cluster` - Runs only once per HA cluster, but can run on any node

Previously Octopus had the concept of Leader/Follower to make sure the `Per Cluster` tasks only run on one node and didn't run too often. Now a shared schedule is stored in the database. The nodes compete to be the next to run the jobs by attempting to acquire a `ClusterWideMutex`.

## Schedule

Some jobs run on fixed intervals, while others use a cron schedule. Some that use cron schedules add randomisation so that
external resources (e.g Hosted database pool, Octofront) do not get slammed.
