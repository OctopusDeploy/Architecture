# Deployment Target Extensibility

## Overview

Octopus server currently uses the terms Machine and Deployment Target somewhat interchangeably. These two concepts were essentially the same in the days of Tentacles and SSH targets but the cloud style targets have shifted the landscape.

Machines are still a core concept to Octopus server, but Deployment Target Types are more dynamic and we want to be able to contribute them from modules. There is a large amount of coupling around the `CommunicationStyle` enum especially and this needs to be remedied before Deployment Target Types can be contributed from outside of `Octopus.Core`.

## Proposed solution

### Deployment Target Type

Some steps have already been taken to make the DeploymentTargetType object extensible. It already lives in the Extensibility world and shouldn't require significant changes.

### CommunicationStyle

We should stop using CommunicationStyle where we mean DeploymentTargetType.

### Endpoint

This particular part of our model is hard to reason about. In all cases, it does represent the conceptual endpoint of a deployment but it doesn't always represent where the deployment actually executes.

We may be better with something like `IHasDirectConnection` for Tentacle and SSH targets. On that interface it would make sense to have a CommunicationStyle. For other targets though, their deployment would go through a Worker and its Endpoint, so they don't have a CommunicationStyle.

Similar to the discussion around [Accounts](Accounts.md), we need a way to allow modules to contribute implementations. One option could be to create an interface like `IDeploymentTarget`. The current Endpoint itself doesn't provide any real behaviour (the things it does provide are based around enum attributes and are counter to extensible design), so using an interface instead may be preferable.

The `IHasDirectConnection` interface mentioned above would be in internal concern and should stay in Core regardless of how Endpoint is changed.

### Portal UI

Work is underway exploring options for UI extensibility, for the moment the UI code would be assumed to stay the way it is.
