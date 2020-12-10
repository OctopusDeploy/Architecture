# Overview

- **Subject**: Deciding What Version To Take Of An External Tool Dependency

# Executive Summary

In Octopus, we package external tools in various pieces of our software that allow Octopus to interact with and deploy to external services. Examples of these tools include the Terraform CLI and Azure CLI.

We will pin all dependencies on external tools to a single static version, as this provides the most stable and reliable way for us to manage these dependencies over time.

# Detail

You can see what tools we package and embed in Octopus Server [on GitHub](https://github.com/octopusdeploy/?q=Octopus.Dependencies.&type=&language=).

Another example of where we package and embed external tools is in our [Dynamic Worker images](https://github.com/OctopusDeploy/DynamicWorkerVmImages). We load both our packaged dependencies onto dynamic workers in their [installer scripts](https://github.com/OctopusDeploy/DynamicWorkerVmImages/blob/master/Win2019/scripts/install-tool-packages.ps1), and other external tools in other installer scripts, such as [kubectl](https://github.com/OctopusDeploy/DynamicWorkerVmImages/blob/master/Win2019/scripts/install-kubectl.ps1).

Bugs have arisen that are the direct result of us taking the 'latest' version of an external tool when building and packaging our software ([example 1](https://trello.com/c/uufTKTnY/3699-breaking-terraform-upgrade-has-been-deployed-across-octopus-cloud), [example 2](https://trello.com/c/NxqZvQ4U/3724-dynamic-worker-images-are-pre-loading-the-incorrect-version-of-some-tooling-packages)).

## Considerations

### Taking Latest At Build Time

Taking latest means to query some external feed as to what the latest available version of a tool is, and to use that regardless of what version number it is.

Taking the latest tool as a dependency ensures we have the latest patches and security updates for the tools we embed in our products.

If we take the latest at build time, it can be difficult to predict if this would have a negative impact on our software that depends on it. It can also be difficult to predict if it would have a negative impact on our customer's deployment processes, which also may utilize the packaged dependency.

Whilst we have a level of automated testing that exercises our software and its interactions with these tool dependencies, they could never be extensive enough to predict all behaviour changes that could be introduced in newer versions of these tools.

### Pinning Dependencies

Pinning dependencies means to hard-code the version of tool we embed in our product, and to request this version each time our tool dependency packages are re-packaged and build.

Pinning dependencies means we do not get the latest patches and security updates for the tools we embed in our products. We only get these updates when we deliberately increase the version of tool we depend on. This can be challenging, as our efforts are often concentrated elsewhere, and dependencies tend to be left locked to certain versions for long periods of time.

By pinning our tool dependencies, we ensure that our own code, and our customers deployment scripts, will always run reliably as our software evolves and is rebuilt over time. We won't "accidentally" break our own software, or our customer's scripts, by rebuilding a dependency package, or revving the version of a dependency package depended on in Server, Dynamic Workers or other locations.

### Tools Vs Libraries

Drawing a distinction between tool versions and library versions is important. People other than us depend on tool versions, so our test suites are never going to have enough coverage. We're the only ones who depend directly on libraries, so it's a not-unreasonable goal to have good enough test coverage for the latter. In this document we are addressing **tools**, not libraries.

### Version Ranges

Some software package managers allow version ranges to be specified, which allows you to 'safely' increase your version dependencies - for example you can pin your major and minor version, but allow for any new patch versions to be utilised.

This works if the world runs on SemVer. But it does not, as Terraform demonstrated in its `0.11` => `0.12` version bump that contains a large amount of breaking changes.

For us, being explicit about a single version for each tool we depend on makes our software easy to reason about and support.

### Handling Transient Dependencies

For us, often the packaged tool is a transient dependency. An example of this is:

`Octopus.Server` depends on `Sashimi.Terraform` which depends on `Octopus.Dependencies.TerraformCLI` which packages the Terraform CLI.

If we allow our build process to take the latest version of a tool as a dependency, we cannot appropriately apply SemVer to our own packages, and therefore cannot predict when we are updating dependencies in upstream packages what the impact of those updates would be.

By pinning our tool dependencies, we can rev our SemVer's appropriately when we deliberately increase our hard-coded version.

For example: when we update `Octopus.Dependencies.TerraformCLI` to Terraform `0.12`, we will rev the major version to `2.x`. `Sashimi.Terraform` would need to consider that breaking change and see if it could take the new dependency without introducing breaking changes itself, and so on up the dependency change. By pinning our dependencies, we can be explicit about SemVer.

## Data Sources

Source listed below might provide additional context but keep in mind that they can disappear at any time.

- [Issue](https://trello.com/c/uufTKTnY/3699-breaking-terraform-upgrade-has-been-deployed-across-octopus-cloud) with Terraform being accidentally updated and rolled out across Octopus Cloud
- [Issue](https://trello.com/c/NxqZvQ4U/3724-dynamic-worker-images-are-pre-loading-the-incorrect-version-of-some-tooling-packages)) with the version of the AzureCLI dependency package becoming out-of-sync between Server and Dynamic Worker Images
