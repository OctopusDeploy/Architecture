# Step Packages

## Overview

**Step Packages** are Octopus Server's extensibility model for creating new Steps - the things that do the work of deployment!

A Step Package is a `zip` file. It contains all of the things Octopus Server needs to use the Step in a Deployment Process.

Step Packages are named in the format `Octopus.MyStepPackage.1.0.0.zip`, and are located under `%OctopusServerRoot%/bin/steps`

Step packages have been designed to allow using different technologies to develop the Step Executor they contain. Currently we support writing Step Packages in node, and will soon support writing them in dotnet.

Node will be our default choice for developing Step Packages.

Dotnet will only be used for migrating existing Sashimi-based steps into Step Packages.

![Step Packages](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/assets/building_blocks.png)

For further implementation-level detail on Step Packages, see the [Step Package Documentation](https://github.com/OctopusDeploy/step-api).

## Conventions

Step Packages use a convention-based structure. These conventions are in place for _simplicity_ - they make locating components within Step Packages simple for the [Step Package CLI](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/Components/StepPackageCLI.md), and for Octopus Server.

## Metadata

At the top level, each Step within a Step Package has metadata, captured within `metadata.json`. This metadata provides a language-agnostic way for Steps to supply Octopus Server with information it needs to use the Step - examples include it's name, and whether it can run on a deployment target.

## Step API

> The Step API is a WIP, and is likely to change as we find an optimal shape for its repository and packaging

The [Step API](https://github.com/OctopusDeploy/step-api) is a npm package that contains a set of types that Step Packages must implement in order to present a conforming step.

These types form the API surface between Steps and Octopus Server. If your Step implements these types, it will be guaranteed to work with Octopus Server, taking into account [versioning](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/Concepts/Versioning.md).

# Step Packages vs Community Step Templates

[Community Step Templates](https://github.com/OctopusDeploy/Library) are another mechanism that allows people to develop custom steps that can then be surfaced within Octopus.

They are expressed entirely within Json, are limited in what parameters they can express, and can only be written in a supported script syntax.

They are well-suited for creating small tasks that can then be shared between projects in Octopus.

They are not suited for more complex tasks - their UI cannot express logic, they can only be developed in a supported scripting language, they are not easily tested, and they cannot contribute some of the additional infrastructure often required by steps (validation, accounts, deployment targets, etc).

At this point, it is our opinion that Step Packages and Community Step Templates serve different purposes, and will likely _not converge_. Community Step Templates support simple reuse scenarios, where Step Packages provide a development model for more sophisticated functionality, composition, and integration within Octopus Server.

Both models allow third parties to develop custom steps for Octopus. Eventually, Step Packages will also be offered from an online marketplace, in a similar fashion to Community Step Templates.
