# Index

- [Overview](#overview)
  - [Tl;Dr:](#tl-dr)
  - [Examples](#examples)
- [Versioning](#versioning)
- [Compatibility](#compatibility)
- [Developer Experience](#developer-experience)

# Overview

> This section is detailling our intentions for versioning and compatibility for Step Packages and Octopus Server, but may change as we implement them.

Versioning and Compatibility are orthogonal, but related concerns within the Step Package ecosystem.

_Versioning_ of a Step Package is about enforcing _internal consistency_ of Step Package components as they are used within Octopus Server - making sure we use the right Step Package components to edit, validate, and execute a step after it has been created, and defining useful tolerances within version ranges that allow us to more easily ship bug fixes and small changes to steps without requiring user intervention to upgrade steps in existing processes.

_Compatibility_ refers to the compatibility of various Step Package components with Server components. There are many compatibility surfaces, which we will explore below.

## Tl;Dr:

- We care about **versions** of Step Packages, and the **compatibility** of their components with Server components.
- We would like to pursue a monorepo for the API (agree strongly with this) - we are already part way there since we have merged them into a single repo.
- Compatibility surfaces include: manifest + conventions, UI API, input processing (both server and UI), validation (ditto), execution API.
- We would like to embed version information for all compatibility surfaces into metadata.json (probably a job for the Step Package CLI
- This information can be made available to the "other side" of the compatibility surfaces in a variety of ways.
- We are still debating how to best support versions of Step Packages changing over time
- We will use shims to support the versions of compatibility surfaces changing over time
- Server will be able to determine what steps it might support

## Examples

Different versions of Step Packages. For example, we ship `step-package-azurestorage.1.0.0`. We then make a change to the set of inputs such that it is no longer runtime compatible with existing steps that have been configured in deployment and runbook process. We then ship a new version, say `step-package-azurestorage.2.0.0`. We need to migrate/upgrade the set of inputs stored against those steps in our deployment and runbook processes, so that the they can be configured and executed using `step-package-azurestorage.2.0.0`.

We have multiple versions of the same "step" that we want to expose to users, so that users can pick which one they want to use. For example, we might have `terraform-1.13.0` which only works with the `0.13.x` versions of terraform. If a user wants to target terraform version `0.14.x`, they would need to upgrade their step to `terraform-1.14.0`. We could also implement this through configuration alone, so that there is a single step that supports multiple versions of terraform with a version selector at the top.

We have a step like `step-package-azurestorage.1.0.0` which is compiled against `step-ui-api.1.0.0`. We then make a breaking change to the `step-ui-api` (and release a new version like `step-ui-api.2.0.0`) and update our framework in portal as well, so that `step-package-azurestorage.1.0.0` is no longer directly compatible with portal. We can either upgrade the `step-ui-api` dependency and release a new version of our step (say `step-package-azurestorage.2.0.0`) to make it compatible and deprecate the old step somehow, or we could have a compatibility shim in portal so that portal can support steps compiled against both `step-ui-api.1.0.0` and `step-ui-api.2.0.0`.

# Versioning

A Step Package is a versioned component. It's version is denoted in its file name - `StepPackage.2.0.0.zip` would denote version `2.0.0` of the Step Package. The version will follow [SemVer 2.0.0](https://semver.org/) versioning semantics.

Over time, we may make changes to the contents of Step Packages, and increase their version number accordingly.

**Minor Version Increments:** Within Server, we will automatically retrieve the _latest version_ of the same major version a step was created with when working within any of the steps components - UI, validator, executor, etc. This means for all backwards-compatible feature additions and bug fixes, users will recieve these benefits immediately in their existing processes, without having to update anything.

**Major Version Increments:** For major version increments, a user will need to opt-in to them. There will be a version selector presented on the step UI if a newer version is available. If it is, and the user selects the next version, the step will be upgraded to the selected version. This will not be a reversable process.

For each major version upgrade, an _upgrade function_ will need to be provided by the Step Package in question to migrate the persisted step inputs from vCurrent to vSelected. This may require running a number of upgrade functions sequentially to upgrade the step to the required version.

We _may_ enforce this consistency within the `Step Package CLI` - we could refuse to build a step of version `v(n)` if the Step Package did not present `n-1` upgrade functions

# Compatibility

There are many compatibility surfaces within the Step Package ecosystem. Making sure these surfaces have explicit versioning in-place allows us to make changes of them over time, and make deliberate decisions about their compatibility as they evolve.

**Step Package Version**
Step packages have a version when they are built. When a step is added to a process, we care about this version, as it will dicatate how we edit and execute the step ongoing. We will eventually want to upgrade steps to newer version of the same step.

The version for the Step Package is included in its file name.

**Step Package Manifest Schema Version**
The component in Server that loads Step Packages cares about the the conventions adhered to within the Step Package, and the structure of the Step Package manifest file. This is an API surface between Server and Step Packages, and is also relied upon by the [Step Package CLI](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/Components/StepPackageCLI.md).

The JSON schema for the Step Package manifest will set a version that represents the current set of conventions and structure of the manifest.

**Step UI API Version**
The Step Package UI Framework cares about versions. When we load a `ui.ts` from a Step Package, it either needs to work with the current version of the UI Framework embedded in server, or be put through compatibility shims so that it will work with vLatest of the framework and Step UI API.

This version will be embedded within the Step Package manifest by the Step Package CLI at build time.

**Step Package Input Schema Version**
When a step is being edited via the UI framework, or is going through the execution pipeline, some components will inspect the step input Json Schema, looking for certain markers. These markers are encoded within the Step Package CLI.

The Step Package input schema will contain a version number that indicates what version of the Input Schema is currently in use. This version is owned by the Step Package CLI, and embedded in the Step Package input schema when it is generated at build time.

**Step Package Validation API Version**
When we want to persist a set of inputs for a step, Server needs to be able to run the validator defined by the step to ensure the inputs are correct and valid. If the Step Package Validation API surface changes, Server will need to interact with Step Packages that use older Validation API versions through compatibility shims.

This version will be embedded within the Step Package manifest by the Step Package CLI at build time.

**Step Package Execution API Version**
When we want to execute a step, Calamari needs to provide a set of arguments to our Step Package Bootstrapper, which then works with these inputs to decrypt files, establish the OctopusContext ingest inputs, and provide them all to the Executor.

We will always assume Calamari uses `vLatest` of the Step Bootstrapper. The Step Bootstrapper is the place where compatibility shims might be used should a step be built against Execution framework `vPrevious`.

This version will be embedded within the Step Package manifest by the Step Package CLI at build time.

**Overall Server support for a given Step API Version**
We need some mechanism so that Server can make decisions about what Step Packages to expose to users, based on what versions it knows it can support.

TBA: how we will accomplish this.

# Developer Experience

To ensure a simple experience for Step Package developers, we will produce a _meta-package_ that they can install via `npm i --save-dev @octopus/step-api`. We don't want them to have to worry about what version they should use.

This will include all of the Step Package APIs required to build a Step Package.

We do not need all of these APIs in Server though, so we will produce separate API packages that server can use: `npm i --save-dev @octopus/step-ui-api`

To make it simpler to build both the individual packages and the meta-package, we will use a _monorepo_ for these API codebases (TODO)
