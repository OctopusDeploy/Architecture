# Step Packages

## Overview

**Step Packages** are Octopus Server's extensibility model for creating new Steps - the things that do the work of deployment!

A Step Package is a `zip` file. It contains all of the things Octopus Server needs to use the Step in a Deployment Process.

Step Packages are named in the format `Octopus.MyStepPackage.1.0.0.zip`, and are located under `%OctopusServerRoot%/bin/steps`

Step packages have been designed to allow using different technologies to develop the Step Executor they contain. Currently we support writing Step Packages in node, and will soon support writing them in dotnet.

Node will be our default choice for developing Step Packages.

Dotnet will only be used for migrating existing Sashimi-based steps into Step Packages.

![Step Packages](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/assets/building_blocks.png)

## Conventions

Step Packages use a convention-based structure. This structure, along with the structure of the `metadata.json` file, is [versioned](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/Concepts/Versioning.md) by the metadata's `version` property.

Example Step Package structure:

```
  |-- package.json
  |-- other node tooling configuration files
  |-- stepA/executor.ts
  |-- stepA/inputs.ts
  |-- stepA/logo.svg
  |-- stepA/metadata.json
  |-- stepA/validation.ts
  |-- stepA/ui.ts
  |-- stepA/step-package.json <== OPTIONAL
  |-- stepB/...
```

Step Packages can contain multiple steps. Each step's code should live in its own seperate folder. By convention, each logical step is identified by locating it's `metadata.json` file, and then finding the required sibling code files alongside it within the same folder.

Each Step within a package must declare the following code files: `executor.ts`, `inputs.ts`, `logo.svg`, `metadata.json`, `validation.ts`, and `ui.ts`.

Steps are free to then arbitrarily break down their code beyond those files however suits the step's developers, but the root files, and their expected `default exports` (more on this below), must be in place.

The [Step Package CLI](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/Components/StepPackageCLI.md) will look for these files and expect them to exist to build a conformant Step Package.

## Metadata

At the top level, the Step Package has Metadata, captured within `metadata.json`.

The metadata has the following contents:

```jsonc
{
  "version": "1.0",
  "type": "step",
  "id": null,
  "name": null,
  "description": null,
  "categories": [],
  "canRunOnDeploymentTarget": false,
  "stepBasedVariableNameForAccountIds": [],
  "launcher": "node"
}
```

### Properties

**Version:** the version of the step package metadata schema and convention structure this package conforms to.

**Type:** step packages contain steps, and deployment targets. The type field indicates which type of entity the metadata is describing.

**Id:** a unique identifier for this step. This must be unique across all steps that may be used by Octopus Server.

**Name:** the name of the step, which will be used in the Octopus Server UI when displaying it to a user.

**Description:** a description of what the step does, also presented to the user in the Octopus Server UI.

**Categories:** the categories a step may be presented within. Must be one of **TODO**

**CanRunOnDeploymentTarget:** a flag that controls whether this step can run on deployment targets via a role designation, or if it must run on Server or a Worker.

**StepBasedVariableNameForAccountIds:** **TODO:** validate whether this is required or not

**Launcher:** describes the Calamari LaunchTool that will coordinate the execution of this Step Package. This can currently be either _node_ for Node Step Packages, or _dotnet_ for dotnet based Step Packages.

## Step API

> Note: this section details our intentions for the Step API and monorepo / package structure. WIP.

The [Step API](https://github.com/OctopusDeploy/step-api) is a npm package that contains a set of types that Step Packages must implement in order to present a conforming step.

> npm -i --save-dev @octopus/step-api

These types cover the `executor`, `validation`, and `ui` components.

These types form the API surface between Steps and Octopus Server. If your Step implements these types, it will be guaranteed to work with Octopus Server, taking into account [versioning](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/Concepts/Versioning.md).

## Executor

The Executor is the part of the Step Package that does the work. It will consume the inputs configured by the Step UI, and do the work of the Step (i.e, deploy an application, upload a file, etc).

The step function is a function that accepts a defined input type and an OctopusContext instance.

```ts
const BlobStorageStepExecutor: Handler<BlobStorageStepInputs> = async (
  inputs: ExecutionInputs<BlobStorageStepInputs>,
  context: OctopusContext
) => {
  /*Do the work*/
};
```

By convention, the Executor must `export default` the `Handler` function. The bootstrapper relies on this convention to execute the function.

```ts
export default BlobStorageStepExecutor;
```

## UI

The [Step UI](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/StepUI.md) within a Step Package is a defined object that Octopus Server will consume to render our Step's UI via the Step UI Framework.

```ts
export const BlobStorageStepUI: StepUI<BlobStorageStepInputs> = {
  /*Define UI*/
};
```

By convention, the UI must `export default` the `StepUI` object.

```ts
export default BlobStorageStepUI;
```

## Inputs

Step Packages define [Structured Inputs](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/Concepts/InputsAndOutputs.md) that the Step's UI, Validator, and Executor will work with.

By convention, the Inputs must `export default` the root input type that we expect the UI, validator, and executor to consume.

```ts
export default interface BlobStorageStepInputs {
    ...
    containerName: string;
    package: PackageReference;
    accountName: string;
}
```

## Validation

TODO

# Step Packages vs Community Step Templates

[Community Step Templates](https://github.com/OctopusDeploy/Library) are another mechanism that allows people to develop custom steps that can then be surfaced within Octopus.

They are expressed entirely within Json, are limited in what parameters they can express, and can only be written in a supported script syntax.

They are well-suited for creating small tasks that can then be shared between projects in Octopus.

They are not suited for more complex tasks - their UI cannot express logic, they can only be developed in a supported scripting language, they are not easily tested, and they cannot contribute some of the additional infrastructure often required by steps (validation, accounts, deployment targets, etc).

At this point, it is our opinion that Step Packages and Community Step Templates serve different purposes, and will likely _not converge_. Community Step Templates support simple reuse scenarios, where Step Packages provide a development model for more sophisticated functionality, composition, and integration within Octopus Server.

Both models allow third parties to develop custom steps for Octopus. Eventually, Step Packages will also be offered from an online marketplace, in a similar fashion to Community Step Templates.
