# Index

- [Index](#index)
- [Overview](#overview)
- [Key Constraints](#key-constraints)
- [Conceptual Model](#conceptual-model)
- [Components](#components)
- [Concepts](#concepts)

# Overview

At the start of 2021, the Steps Team was established within Octopus.

The initial responsibility of the team was to build _Concensus_ and _Clarity_ on how Steps should be developed going forward, to ensure we were confident the model and architecture would support Octopus over the next five years.

This document presents the key architectural design decisions for our modular step infrastructure.

# Key Constraints

**Independent**

Development of our steps should be a value stream that is independent of our core Octopus Server product. Our steps should live in Step Packages that can be included within Octopus Server. All the step-specific implementation details should live within these Step Packages.

**Simple**

We should build steps that are simple and easy to maintain. As development of steps scales up, we may end up with hundreds of steps. Every small unnecessary development cost also scales up with the number of steps, so minimising these costs is critical.

# Conceptual Model

Octopus’ users get a lot of value out of steps that take the heavy lifting out of deployment. When Octopus codifies the knowledge on how to deploy to various targets and environments, users no longer are burdened with needing to acquire that knowledge themselves to deploy their software effectively.

We have no plans to invest further in “traditional” steps that target physical machines or environments that lack comprehensive CLI or templating tooling.

All foreseeable future targets will have CLI or templating tooling available, and this tooling provides all of the flexibility a customer could possibly want in configuring the target environment.

Our ideal model will provide opinionated deployment tasks first and foremost. If users find their use case is not satisfied by the opinionated task, there will be a graduated path either to finer-grained deployment tasks they can compose their process of, or to script or template tasks that allow the customer to drive the CLI or templating tooling to accomplish their goals.

> **Example: AWS Lambda Deployments**
>
> With lambda deployments, we will offer three steps:
>
> The **opinionated step**, in which we will set up all of the things needed to establish a self-contained Lambda application - API Gateway, Lambdas, IAM roles, etc
>
> More **granular steps**, that allow the API Gateway and Lambdas to be established independently, which is important for some team scenarios
>
> Finally, all of these steps will be able to be decomposed to a **CloudFormation step**, should our users need ultimate control over the process

We will ensure the path between opinionated tasks to flexible tasks has a great user experience, allowing users to transition between seamlessly.

Composite tasks (tasks represented as a set of finer-grained functions or tasks that can be reordered / recomposed) have been considered, but ultimately could never provide the flexibility users would be seeking. The development cost and additional UX complexity they would introduce, when weighed up with the marginal benefits we would get, give a clear indication that they are not a viable model.

**Tasks should be singular for consumers (users), be composable for developers, and provide a graduated path to success when additional flexibility is required.**

[Further Discussion](https://docs.google.com/document/d/1fvB1FWEO9QBLqzAys6DmDryB4PlRYC5T0Pp2XbU8Vm8)

# Components

- [Step Packages](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/Components/StepPackages.md)
- [Step UI](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/Components/StepUI.md)

# Concepts

- [Inputs and Outputs](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/Concepts/InputsAndOutputs.md)
- [Execution](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/Concepts/Execution.md)
- [Building and Packaging](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/Concepts/BuildingAndPackaging.md)
- [Versioning](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/Concepts/Versioning.md)
