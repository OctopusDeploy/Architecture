# Machine Health


## Overview
The health information of a machine is currently changed via
- Health Check task
- Health Check step in a deployment/runbook 
- Being uncontactable during a deployment (and the option to exclude is enabled)

There are two types of health checks
1. Connectivity only, that runs an empty script
1. Full check, that runs a script that:
    - Collects the required information (versions, shell, os, etc) and returns it via output variables (notably service messages are not used)
    - Whether there is enough disk space
    - The user provided script configured on the `MachinePolicy`

If a health check task or step completes for a machine, it records a range of information (The connectivity only check only records the `HealthStatus`):
- On Machine.[MachineHealth](https://github.com/OctopusDeploy/OctopusDeploy/blob/80da6cae6ddeb139613a270b834e70f2e442013d/source/Octopus.Core/Model/Machines/MachineHealth.cs#L29)
    - `HealthStatus` (aka [`MachineStatus`](https://github.com/OctopusDeploy/OctopusDeploy/blob/a1252cd06d6ff22773b63f2b30a26749c3031afb/source/Octopus.Core/Model/Machines/MachineStatus.cs#L8))
    - Tentacle Version
    - Whether a Tentacle upgrade is required
    - Whether it has the latest Calamari
    - When it was last checked
    - The certificate signature algorithm
    - How long it has been unavailable
- On `Machine` itself
    - Operating System
    - Shell Name
    - Shell Version
- On the machine state change event
    - The [details](https://github.com/OctopusDeploy/OctopusDeploy/blob/e5f36b43537d4bce59ffc43285512d2a849b2102/source/Octopus.Server/Orchestration/ServerTasks/HealthCheck/HealthResultRecorder.cs#L100), generally warnings and errors that occured

## Health Status
### Problem
Customers with fleets of occasionally connected polling Tentacles do not want the overhead of a health check in order to trigger an auto-deploy. They want auto-deploy to trigger shortly after a Tentacle connects.

### Overview
The health status of a machine is one of `Healthy`, `Healthy with Warnings`, `Unhealthy`, `Unavailable`. The state can transition straight from one state to another. The state is determined by the outcome of the last health interaction as outlined above.

The `Healthy` state is set when a health check completes without any warnings, errors, exceptions and return code of 0. 

It may also be marked as `Healthy` if it is set to auto-upgrade Calamari and does so successfully. This is likely a bug as it ignores other warnings that may have occured.

The `Healthy with Warnings` state is set when a health check completes with warnings and return code of 0 and without exceptions. Warnings are raised for:
- The `Calamari` version being older than that shipped with server
- Low disk space
- A warning from a user defined script

The `Unavailable` state is set if the health check or deployment step fails with an exception that indicates a network error (e.g. failure on SSH connect, `HalibutException`). In the code this is described as a `Transient` exception. The code also uses the term `Offline` for this state.

The `Unhealthy` state is set if the health check or deployment step fails with an exception that is not classed as `Unavailable`.

### Proposed change
When a machine transitions from `Healthy` to `Unavailable`, the customer expects it to transition back to `Healthy` once any kind of communication was established. The assumption is that a network error does not change the healthyness of a machine.

The concepts of connectivity and healthyness should be seperated. A new property `Connectivity` with values `Available` and `Unavailable` should be added, and the `HealthStatus.Unavailable` state removed.

Anytime a connection is established (outgoing or incomming), the `Connectivity` flag should be updated. A callback on successful connection should be added to `Halibut` to make this happen. Performance should be considered as it is not unusual for 1000's of tentacles to open polling connections at once (e.g. server starts up). The solution should not make 1000's of calls to the database, particularly if nothing has changed.

A event will need to be recorded whenever the `Connectivity` flag changes so that customers can setup auto-deploy to that machine.

## Change health via API


## Remove Warning for Calamari
Not needed

## Separate Machine and Health
Both the table and events

## OS/Shell/ShellVersion

## Textual status messages for machines