# Refactor of Calamari Conventions

## Status Quo
### Overview
The Octopus Server invokes `Calamari` with the command to be executed, e.g. `run-script` or `azure-web-app`. All commands invoked from a step use a pattern we'll call the `Convention Pattern` for the purposes of this document. 

### Convention Pattern
The way the pattern works can be seen in the [ConventionProcessor](https://github.com/OctopusDeploy/Calamari/blob/ccfb70ee8846df458a490179b5a8ecf16398415b/source/Calamari.Shared/Deployment/ConventionProcessor.cs#L22). The gist of it is that the command supplies a list of `IInstallConvention` and `IRollbackConvention`). 

The processor runs through all the `IInstallConvention` conventions in order, calling `Install`. It then calls the `Cleanup` methods of the `IRollbackConventions`

If an exception occurs it stops and and runs the `Rollback` method of the `IRollbackConvention` conventions and then the `Cleanup` methods of the `IRollbackConventions`, bailing out completely if another exception occurs.

### IRollbackConvention
There are only two `IRollbackConvention`s, `RollbackScriptConvention` and `FeatureRollbackConvention`. The latter has an empty `Cleanup` method.

Those rollback conventions are only used by three commands, `DeployPackage`, `DeployJavaArchive` and `JavaLibrary`. All other commands don't have any rollback.

## Problems
### Modularity
For modularity we want to seperate the command implementations from the shared conventions. An interface would be shared instead. Contructing the conventions inside the command is therefor not compatible with this change.

### Constructors
Since the `IInstallConvention` method is fixed, all conventions must accept command specific paramters via the constructor ([example in `TerraformCommand`](https://github.com/OctopusDeploy/Calamari/blob/84137f9d03c86b585ec286dc72ec933b2f490ac7/source/Calamari/Commands/DeployPackageCommand.cs#L81))

The conventions are also all constructed inside the command. This was managable before IoC was introduced as the constructor parameters were limited. However with more testing and adopting constructor injection of services it has lead to lengthy constructor calls, [example in `DeployPackageCommand`](https://github.com/OctopusDeploy/Calamari/blob/84137f9d03c86b585ec286dc72ec933b2f490ac7/source/Calamari/Commands/DeployPackageCommand.cs#L78-L101)

### Bailing out early
Steps can [set a variable](https://github.com/OctopusDeploy/Calamari/blob/026438033c79ea385db0a380fe621c3301db2b02/source/Calamari/Deployment/Conventions/AlreadyInstalledConvention.cs#L40) to make the convention processor stop further conventions from running. This is not obvious from outside the convention.

### Interacting Conventions
For one convention to pass values to another convention, it has to [pass strings via a variable](https://github.com/OctopusDeploy/Calamari/blob/388c014ea78372ae09da602c043ade446afc8aa4/source/Calamari.Shared/Deployment/Conventions/ConfigurationTransformsConvention.cs#L53) for it to be [read out and parsed](https://github.com/OctopusDeploy/Calamari/blob/388c014ea78372ae09da602c043ade446afc8aa4/source/Calamari.Shared/Deployment/Conventions/ConfigurationVariablesConvention.cs#L26) in the next convention. ([example](https://github.com/OctopusDeploy/Calamari/blob/84137f9d03c86b585ec286dc72ec933b2f490ac7/source/Calamari/Commands/DeployPackageCommand.cs#L85-L86) command that uses these two conventions).

Again this behaviour is hidden from the command code, making it harder to follow.

### Conditional Conventions
Sometimes a convention only needs to run in some configurations of a command. This is achieved in three different ways:

1. [Conditional](https://github.com/OctopusDeploy/Calamari/blob/d09e8799dec1b0e1555946bd043e09d2169865fd/source/Calamari/Commands/Java/DeployJavaArchiveCommand.cs#L80) when building the list
1. [Wrapping](https://github.com/OctopusDeploy/Calamari/blob/4fbf4dfa7c9c173fa28da2e774860c958806197e/source/Calamari.Aws/Commands/UploadAwsS3Command.cs#L56) using another convention built by a extension method
1. [Aggregate](https://github.com/OctopusDeploy/Calamari/blob/4fbf4dfa7c9c173fa28da2e774860c958806197e/source/Calamari.Aws/Commands/DeployAwsCloudFormationCommand.cs#L94) convention whith child conventions that runs conditionally using wrapping.
1. [Passed in predicate](https://github.com/OctopusDeploy/Calamari/blob/01ae85386f9249e9bde67daa3f70ead3e56d6d83/source/Calamari.Shared/Deployment/Conventions/SubstituteInFilesConvention.cs#L36)

In almost all these usages, whether the convention is needed is known at the start of the processing.

### Complicate Simple Commands
Some commands have a single convention specific to it (e.g. `TransferPackageCommand`). Another example is the Terraform step which has parallel hierachies for the command and associated convention. [PR #531](https://github.com/OctopusDeploy/Calamari/pull/531) shows it might be simpler to inline the convention in the step.

## Solution 1 - Convention Factory
[Spike](https://github.com/OctopusDeploy/Calamari/pull/509/files)

A `ConventionFactory` (implementing `IConventionFactory`) would take care of the construction. It would have one or more methods for each convention that passed the command specific options.

This would only address the `Modularity` concern and leave rest as it is.

Unit tests for the commands would need to stub out the `IConventionFactory` and still constuct all the dependencies of the command.

## Solution 2 - Procedural
Seeing the `IRollbackConvention` is only lightly used, the full power of the convention pattern has not been realised. Therefore we remove it altogether and replace it with standard procedural code.

Considering the following snippet which is a cut down version of the `DeployPackageCommand` with an added conditional command:
```csharp
var conventions = new List<IConvention>
{
    new AlreadyInstalledConvention(journal), // Bails out if installed
    new FeatureConvention(DeploymentStages.AfterPreDeploy, featureClasses, fileSystem, scriptEngine, commandLineRunner, embeddedResources),
    new SubstituteInFilesConvention(fileSystem, substituter).When(_ => shouldSubstituteInFiles),
    new ConfigurationTransformsConvention(fileSystem, configurationTransformer, transformFileLocator),
    // Output of ConfigurationTransformsConvention influences ConfigurationVariablesConvention
    new ConfigurationVariablesConvention(fileSystem, replacer),
    new RollbackScriptConvention(DeploymentStages.DeployFailed, fileSystem, scriptEngine, commandLineRunner),
    new FeatureRollbackConvention(DeploymentStages.DeployFailed, fileSystem, scriptEngine, commandLineRunner, embeddedResources)
};

var deployment = new RunningDeployment(packageFile, variables);
var conventionRunner = new ConventionProcessor(deployment, conventions);
conventionRunner.RunConventions();
return 0;
```

It would be implemented something like:

```csharp
var deployment = new RunningDeployment(packageFile, variables);
try
{
    if (alreadyInstalledChecker.ShouldSkip(journal, deployment))
        return 0;

    features.Install(DeploymentStages.AfterPreDeploy, deployment, featureClasses, embeddedResources);
    if (shouldSubstituteInFiles)
        substituteInFiles.Substitute();

    var appliedXmlTransforms = configurationTransforms.Transform();
    configurationVariable.Transform(appliedXmlTransforms);
    return 0;
}
catch
{
    packagedScriptRunner.Rollback();
    features.Rollback(embeddedResources);
    throw;
}
finally
{
    packagedScriptRunner.Cleanup();
}
```

Only 3 of the commands would need the try/catch/finally blocks. 

For something like the `TerraformCommand`, it'd be simpler:

```csharp
var deployment = new RunningDeployment(packageFile, variables);
if(packageFile != null)
    extractPackage.ToStaging();
substituteInFiles.Substitute(FileTargetFactory);
RunTerraformCommand()
return 0;
```

## Solution 3 - Builder
We could build a fluent interface that allows for flow control and passing values from one step to another.

Again, taking the same example:
```csharp
var conventions = new List<IConvention>
{
    new AlreadyInstalledConvention(journal), // Bails out if installed
    new FeatureConvention(DeploymentStages.AfterPreDeploy, featureClasses, fileSystem, scriptEngine, commandLineRunner, embeddedResources),
    new SubstituteInFilesConvention(fileSystem, substituter).When(_ => shouldSubstituteInFiles),
    new ConfigurationTransformsConvention(fileSystem, configurationTransformer, transformFileLocator),
    // Output of ConfigurationTransformsConvention influences ConfigurationVariablesConvention
    new ConfigurationVariablesConvention(fileSystem, replacer),
    new RollbackScriptConvention(DeploymentStages.DeployFailed, fileSystem, scriptEngine, commandLineRunner),
    new FeatureRollbackConvention(DeploymentStages.DeployFailed, fileSystem, scriptEngine, commandLineRunner, embeddedResources)
};

var deployment = new RunningDeployment(packageFile, variables);
var conventionRunner = new ConventionProcessor(deployment, conventions);
conventionRunner.RunConventions();
return 0;
```

The fluent interface:
```csharp
interface ICommandPipelineBuilder
{
    ICommandPipelineBuilder Install(Action action);
    ICommandPipelineBuilder<TReturn> Install<TReturn>(Func<TReturn> func);
    ICommandPipelineBuilder Rollback(Action action);
    ICommandPipelineBuilder Cleanup(Action action);
    ICommandPipelineBuilder<string[]> ConfigurationTransforms();
}

interface ICommandPipelineBuilder<TPrevious> : ICommandPipelineBuilder
{
    ICommandPipelineBuilder StopIf(Func<TPrevious, bool> predicate);
    ICommandPipelineBuilder Install(Action<TPrevious> action);
    ICommandPipelineBuilder<T> Install<T>(Func<TPrevious, T> func);
}
```

Which would result in a command implementation of:
```csharp
builder = builder.Install(() => alreadyInstalledChecker.ShouldSkip(journal))
    .StopIf(shouldSkip => shouldSkip);

if (shouldSubstituteInFiles)
    builder.Install(() => substituteInFiles.Substitute());

builder.Install(() => configurationTransforms.Transform())
    .Install(appliedXmlTransforms => configurationVariable.Transform(appliedXmlTransforms));

return builder;
```

or if there were overloads that took a predicate for conditional execution

```csharp
return builder.Install(() => alreadyInstalledChecker.ShouldSkip(journal))
    .StopIf(shouldSkip => shouldSkip);
    .Install(shouldSubstituteInFiles, () => substituteInFiles.Substitute())
    .Install(() => configurationTransforms.Transform())
    .Install(appliedXmlTransforms => configurationVariable.Transform(appliedXmlTransforms));
```

for a simple command:

```csharp
return builder.Install(packageFile != null, () => extractPackage.ToStaging())
    .Install(() => substituteInFiles.Substitute(FileTargetFactory))
    .Install(RunTerraformCommand)
```