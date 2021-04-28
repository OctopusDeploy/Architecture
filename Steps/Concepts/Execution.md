# Index

- [Composition and Coordination](#composition-and-coordination)
  - [Problem](#problem)
  - [Solution: The Execution Manifest](#solution-the-execution-manifest)
- [Step Execution](#step-execution)
  - [Bootstrapper](#bootstrapper)

# Composition and Coordination

In Octopus, a deployment process consists of a set of steps. The code that makes up steps expresses both the execution-time functionality and the UI.

This functionality consist of:

- Implicit functions that may be triggered, like package acquisition
- Shared functions that are configured via the UI, like package manipulation, powershell version setting, and deployment scripts (currently grouped under "Features" in our UI)
- Handlers that do the work of the steps

The only shared functionality that is provided by Octopus Server is "Package acquisition and manipulation". No shared functions outside of Package acquisition and manipulation will be required in new steps. There is no evidence to support the need for Custom Deployment Scripts within new steps. Our proposed Conceptual Model will provide users the flexibility and control they require of new steps.

With that in mind, Step Packages will need to support the composition and coordination of:

- Implicit functions that may be triggered, like package acquisition
- Handlers that do the work of the steps

## Problem

If we have a step that involves invoking multiple pieces of functionality that are implemented in different languages (like C# and typescript) and have a different release cadence (like being shipped with Octopus Server or as part of a Step Package), how do we compose them together to form a single cohesive step?

Let's take the following contrived process example:

```
Acquire Package X
Substitute Variables in Files on Package X
Upload Package X contents to Azure Blob Storage
```

There are a few ways we might model this.

We could take a delcarative approach, where `Acquire Package` and `Substitute Variables` are declared in the Step Package `manifest.json`. It is left to Server and other framework components to show the right UI to enable these, and ensure they are enacted at execution time.

Another option would be to coordinate this functionality within the step handler code. The Step Executor would be supplied some sort of function it could call to acquire the package and substitute variables:

```
var files = stepPackages.acquirePackage(packageName, packageVersion, feedId);
stepPackages.substituteVariables(files, patterns, exclusions);
azure.uploadToBlobStorage(files);
```

There are several options, including the above two, for composing arbitrary functionality together. All have their own complexities and limitations, and enforcing contracts across boundaries becomes difficult in all of them.

## Solution: The Execution Manifest

The only functionality that needs to be composed with step functionality is package acquisition and manipulation. If we promote this to a first class feature set, we no longer need a generic way of plugging arbitrary units of functionality together.

We will develop UI elements that can be _Composed_ into steps for configuring package acquisition and manipulation.

Octopus Server will know how to interrogate that configuration, and will _Coordinate_ the appropriate functionality by building up an **Execution Manifest** for the step's execution, that it will send to Calamari to enact.

Problems we avoid by taking this approach:

- We remove the need for the declarative step manifest to attempt to describe what functions should be run prior to the step handler running.
- We remove the need for handlers to call other functions that may be written in another language (i.e. trying to call the variable substitution code, currently written in .NET, from a nodejs handler)
- We avoid needing potentially complex code within Server to deal with potential order-of-execution problems

Some nice side-benefits we get by taking this approach:

- Given Octopus Server can now produce an **Execution Manifest** that describes what functions will be invoked, and the inputs supplied to those functions, we take a step towards being able to "preview" a deployment to an environment.
- Since functions become independent, and are coordinated by Octopus Server, shared functions can be updated independently of Step Packages. With this we avoid the current pain we experience with Sashimi where all Sashimi’s need to be updated when shared functions are updated.

[Execution Manifest Diagram](https://whimsical.com/steps-execution-manifest-N74pfHgDoLXNK8ck1UJmb9)

[Further Discussion](https://docs.google.com/document/d/1E5u3BnYlLzXQ4kwbQVb6XY9TmSFegxtAnwBi_A77WAE)

# Step Execution

Steps from Step Packages are executed by [Calamari](https://github.com/octopusdeploy/Calamari).

Calamari, upon recieving an `execute-manifest` command with an accompanying Execution Manifest and variables, will do the work of marshalling the execution of the step.

It does this by retrieving the correct `LaunchTool` specified within the supplied Execution Manifest (which in turn was built from the `manifest.json` expressed by the Step Package), and running the correct `Bootstrapper` for the specified `LaunchTool` [example](https://github.com/OctopusDeploy/Calamari/blob/master/source/Calamari/Commands/ExecuteManifestCommand.cs).

## Bootstrapper

The Step Bootstrapper can be found in the [step-bootstrapper](https://github.com/OctopusDeploy/step-bootstrapper) repository.

It is responsible for:

- Launching the Step Package function and providing it with its inputs
- Providing an `OctopusContext` to the Step Package function as an additional input that provides ambient context about the deployment
- Providing common step functions to read and set variables, to send messages back to Octopus Server, etc.

The bootstrapper is coupled to the `LaunchTool` within Calamari - each tool knows the specific bootstrapper it requires to launch a step.

Each platform we plan to support for steps will require a bootstrapper. As of now that is `node`, and `dotnet`.

### Node.Js

The [step-bootstrapper](https://github.com/OctopusDeploy/step-bootstrapper) repository build produces a`node.bootstrapper` package, which is then embedded in server within the `/bin/tools` folder.

### DotNet

TODO
