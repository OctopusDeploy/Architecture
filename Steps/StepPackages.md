# Overview

** Step Packages ** are Octopus Server's extensibility model for creating new Steps - the things that do the work of deployment!

A Step Package is a zip file. It contains all of the things Octopus Server needs to use the Step in a Deployment Process.

Step Packages are named in the format `Octopus.MyStepPackage.1.0.0.zip`, and are located under `%OctopusServerRoot%\bin\steps`

A Step Package has the following content structure:

```
|-- manifest.json
  |-- src/index.ts
  |-- src/inputs.ts
  |-- src/outputs.ts
  |-- src/ui-definition.ts
```

Step packages have been designed to allow using different technologies to develop the Step Function they contain. Currently we support writing step packages in Node.js, and will soon support writing them in DotNet.

Node.js will be our default choice for developing Step Packages.

DotNet will only be used for migrating existing Sashimi-based steps into Step Packages.

TODO: Update when DotNet step packages are a thing.

## Manifest

At the top level, the Step Package has a Manifest, captured within `manifest.json`.

The manifest has the following contents:

```jsonc
{
  "categories": [],
  "description": null,
  "name": null,
  "id": null,
  "launcher": null,
  "entryPoint": null,
  "keywords": null,
  "canRunOnDeploymentTarget": false,
  "stepBasedVariableNameForAccountIds": [],
  "whenInAChildStepRunInTheContextOfTheTargetMachine": false,
  "inputs": {
    // Input JsonSchema
  },
  "outputs": {
    // Output JsonSchema
  }
}
```

TODO: Update with a better example once we develop some real step packages.

### Properties

**Launcher:** describes the Calamari LaunchTool that will coordinate the execution of this step package. This can currently be either _Node_ for Node step packages, or _DotNet_ for dotnet based step packages.

**EntryPoint:** the file within the package that will be used as the entrypoint, invoked by the Bootstrapper owned by the indicated LaunchTool.

**Inputs and Outputs:** JsonSchema objects that represent the structure the step package expects to work with for its inputs and outputs. See the [Index](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/Index.md) for further information on Step Package inputs and outputs.

TODO: Think about what the official doco looks like for this, consider linking to that from the architecture repo.

## Step Function

This is the part of the step package that does the work. It will consume the inputs configured by the Step UI, and do the work of the Step (i.e, deploy an application, upload a file, etc).

The step function is a simple function that accepts a defined input type and an OctopusContext instance.

By convention, the step function will be named **main**. The bootstrapper relies on this convention to execute the function.

A trivial Node.js step function may look like:

```ts
export function main(inputs: MyStepInput, context: OctopusContext) {
  console.log("Hello World!");
  console.log(`Your inputs are: ${inputs}`);
  console.log(`Your context is: ${JSON.stringify(context)}`);
  console.log(context.getOctopusVariable("Octopus.WorkerPool.Name"));
}
```

## Step UI

See [Step UI](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/StepUI.md)

## Execution

Step Packages are executed by [Calamari](https://github.com/octopusdeploy/Calamari).

Calamari, upon recieving an `execute-manifest` command with an accompanying Execution Manifest and variables, will do the work of marshalling the execution of Step Functions.

It does this by retrieving the correct LaunchTool specified within the Execution Manifest (which in turn was built from the manifest.json expressed by the Step Package), and running the correct Bootstrapper for the specified LaunchTool [example](https://github.com/OctopusDeploy/Calamari/blob/master/source/Calamari/Commands/ExecuteManifestCommand.cs).

### Node.js

The Node Step Package Bootstrapper can be found in the [Octopus.StepPackages.Node](https://github.com/OctopusDeploy/Octopus.StepPackages.Node/) repository.

It is responsible for:

- Launching the step package function and providing it with its inputs
- Providing an `OctopusContext` to the step package function as an additional input that provides ambient context about the deployment
- Providing common step functions to read and set variables, to send messages back to Octopus Server, etc

Each LaunchTool requires its own Bootstrapper.

### DotNet

The DotNet Step Package Bootstrapper is TBA.

# Step Packages vs Community Step Templates

[Community Step Templates](https://github.com/OctopusDeploy/Library) are another mechanism that allows people to develop custom steps that can then be surfaced within Octopus.

They are expressed entirely within Json, are limited in what parameters they can express, and can only be written in a supported script syntax.

They are well-suited for creating small tasks that can then be shared between projects in Octopus.

They are not suited for more complex tasks - their UI cannot express logic, they can only be developed in a supported scripting language, they are not easily tested, and they cannot contribute some of the additional infrastructure often required by steps (validation, accounts, deployment targets, etc).

At this point, it is our opinion that Step Packages and Community Step Templates serve different purposes, and will likely _not converge_. Community Step Templates support simple reuse scenarios, where Step Packages provide a development model for more sophisticated functionality, composition, and integration within Octopus Server.

Both models allow third parties to develop custom steps for Octopus. Eventually, Step Packages will also be offered from an online marketplace, in a similar fashion to Community Step Templates.
