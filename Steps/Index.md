# Index

- [Overview](#overview)
- [Key Constraints](#key-constraints)
- [Conceptual Model](#conceptual-model)
- [Composition and Coordination](#composition-and-coordination)
- [Inputs and Outputs](#inputs-and-outputs)
- [Step Packages](#step-packages)
- [Step UI Framework](#step-ui-framework)
- [Validation](#tba)
- [Bound Variables](#tba)
- [Versioning and Upgrading](#tba)
- [Infrastructure](#tba)
- [Build](#tba)
- [Packaging](#tba)

# Overview

At the start of 2021, the Steps Team was established within Octopus.

The initial responsibility of the team was to build _Concensus_ and _Clarity_ on how Steps should be developed going forward, to ensure we were confident the model and architecture would support Octopus over the next five years.

This document summarises the initial discovery and decision making.

# Key Constraints

**Independent**

Development of our steps should be a value stream that is independent of our core Octopus Server product. Our steps should live in Step Packages that can be included within Octopus Server. All the step-specific implementation details should live within these Step Packages.

**Simple**

We should build steps that are simple and easy to maintain. As development of actions scales up, we may end up with hundreds of actions. Every small unnecessary development cost also scales up with the number of actions, so minimising these costs is critical.

# Conceptual Model

Octopus’ users get a lot of value out of steps that take the heavy lifting out of deployment. When Octopus codifies the knowledge on how to deploy to various targets and environments, users no longer are burdened with needing to acquire that knowledge themselves to deploy their software effectively.

We have no plans to invest further in “traditional” steps that target physical machines or environments that lack comprehensive CLI or templating tooling.

All foreseeable future targets will have CLI or templating tooling available, and this tooling provides all of the flexibility a customer could possibly want in configuring the target environment.

Our ideal model will provide opinionated deployment tasks first and foremost. If users find their use case is not satisfied by the opinionated task, there will be a graduated path either to finer-grained deployment tasks they can compose their process of, or to script or template tasks that allow the customer to drive the CLI or templating tooling to accomplish their goals.

We will ensure the path between opinionated tasks to flexible tasks has a great user experience, allowing users to transition between seamlessly.

Composite tasks (tasks represented as a set of finer-grained functions or tasks that can be reordered / recomposed) have been considered, but ultimately could never provide the flexibility users would be seeking. The development cost and additional UX complexity they would introduce, when weighed up with the marginal benefits we would get, give a clear indication that they are not a viable model.

**Tasks should be singular for consumers (users), be composable for developers, and provide a graduated path to success when additional flexibility is required.**

[Further Discussion](https://docs.google.com/document/d/1fvB1FWEO9QBLqzAys6DmDryB4PlRYC5T0Pp2XbU8Vm8)

# Composition and Coordination

In Octopus, a deployment process consists of a set of steps. The code that makes up steps expresses both behaviours and UI.

These behaviours consist of:

- Implicit behaviours that may be triggered, like package acquisition
- Shared behaviours that are configured via the UI, like package manipulation, powershell version setting, and deployment scripts (currently called "Features" in our UI)
- Handlers that do the work of the steps

Implicit and Shared behaviours largely fall under the banner of "Package acquisition and manipulation". Behaviours outside of Package acquisition and manipulation will not be required in new steps. There is no evidence to support the need for Custom Deployment Scripts within new steps. Our proposed Conceptual Model will provide users the flexibility and control they require of new steps.

## Problem

If we have a step that involves invoking multiple pieces of functionality that are implemented in different languages (like C# and typescript) and have a different release cadence (like being shipped with Octopus Server or as part of a Step Package), how do we compose them together to form a single cohesive step?

Do we need a step to declare what pieces of functionality should be invoked? Should the step coordinate this functionality itself, within its own handler code?

There are several options for composing arbitrary functionality together, but all have their own complexities and limitations, and enforcing contracts across boundaries becomes difficult.

## Solution: The Execution Manifest

The only "shared" functionality that needs to be composed with step functionality is package acquisition and manipulation. If we promote this to a first class feature set, we no longer need a generic way of plugging arbitrary units of functionality together.

We will develop UI elements that can be _Composed_ into steps for configuring package acquisition and manipulation.

Octopus Server will know how to interrogate that configuration, and will _Coordinate_ the appropriate functionality by building up an **Execution Manifest** for the step's execution, that it will send to Calamari to enact.

Problems we avoid by taking this approach:

- We remove the need for the declarative step manifest to attempt to describe what functions should be run prior to the step handler running.
- We remove the need for handlers to call other functions that may be written in another language (i.e. trying to call the variable substitution code, currently written in .NET, from a nodejs handler)
- We avoid having to develop a number of implicit behaviours that would be required to support composing arbitrary functionality and its supporting UI.

Some nice side-benefits we get by taking this approach:

- Given Octopus Server can now produce an **Execution Manifest** that describes what functions will be invoked, and the inputs supplied to those functions, we take a step towards being able to "preview" a deployment to an environment.
- Since behaviours become independent, and are coordinated by Octopus Server, shared behaviours can be updated independently of Step Packages. With this we avoid the current pain we experience with Sashimi where all Sashimi’s need to be updated when shared behaviours are updated.

[Execution Manifest Diagram](https://whimsical.com/steps-execution-manifest-N74pfHgDoLXNK8ck1UJmb9)

[Further Discussion](https://docs.google.com/document/d/1E5u3BnYlLzXQ4kwbQVb6XY9TmSFegxtAnwBi_A77WAE)

# Inputs And Outputs

Octopus currently provides no explicit schema definition for the inputs a given action needs defined to do its job.

The current model for inputs for Octopus Actions uses a Namespace-keyed state bag approach.

Keys for inputs are currently defined and redefined in several places.

## Problem

- It is difficult to find what inputs an action expects (currently convention based)
- It is difficult to tell what type of information each input should capture (i.e. number, string, complex type) - at the moment the only place we enforce the type of info is within validators (example)
- Complex types can only be expressed as flat sets of keys
- Input keys are redefined in up to three places for certain actions
- It is difficult to determine what keys within the state bag a given action might care about

## Solution: Input and Output Schemas

Steps will express a Schema that defines what inputs it expects to receive, and potentially one for what Outputs it may emit too.

The Schema for inputs will be described in a Language Agnostic way, so that the schema can be leveraged in Server, and from within Step components (UI, handlers, validators, etc).

Validation for inputs will be expressed in a Language Specific way, so that developers can develop validation procedures without the pain of learning a new language / specification.

We think a combination of https://json-schema.org/ for Schema definition, and TypeScript functions for validation that can be run via https://github.com/sebastienros/jint (or similar) in Server provides the best balance of tradeoffs for product requirements and developer requirements.

[Inputs Map](https://whimsical.com/steps-inputs-map-QyP5kQgsTtXSSStdDTAGVZ)

[Further Discussion](https://docs.google.com/document/d/19qz4U33sK_xwGJATBxJ52CdYNQM-hzSbULX2H8mxbBA)

# Step Packages

See [Step Packages](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/StepPackages.md)

# Step UI Framework

See [Step UI](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/StepUI.md)

# TBA

Many things are still evolving, and will appear in due course!
