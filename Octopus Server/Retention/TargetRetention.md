# Target Retention

aka `Tentacle Retention`

Target retention is designed to clean up old files from deployment targets (Tentacle, SSH) in order to preserve space.

## Timing

Retention is only run on successful deployments as we:
 - Want failed deployments to finish as soon possible. 
 - Don't want to cleanup the successful deployment that is currently being used. On successful deployment we know which is the currently in use directory.

We also want to keep some failed deployments as the user may:
- Want to inspect the files to figure out what went wrong
- Fix up the problem by hand and retry the deployment skipping the step

## Configuration

The `How long should we keep extracted packages and files on disk on Tentacles` on the lifecycle phase for the current deployment.

It is independent of the release retention policy. A release may be deleted from Octopus before it is cleaned from the target.

The retention can be configured as all, a number of days, or a number of "releases". The term **releases here is used incorrectly**, it should be the number of deployments. The number specified is the number to keep **in addition to** the current deployment.

## Package Cache

There is a package cache on the target so that the package does not need to be re-acquired on each deployment. This applies to both directly acquired from a feed or via the Octopus Server. This cache is also used to calculated delta compression diffs and reconstruct packages.

*Note*: The [server file (package) cache](..\FileCache.md) cleanup works differently.

## Deployment Directories

Some actions leave extracted packages behind as that is their purpose (e.g. IIS, NGIX, Deploy Package). Unless the `Custom Directory` option is configured they will use a new destination folder for every deployment. This is the folder that is cleaned up as part of this process.

Users may wish to keep these old deployments so that the files are in place if they need to manually roll back.

## Deployment Journal

The deployment journal (`DeploymentJournal.xml`) is used to track what has been deployed to a target and when.

Example:
```xml
<Deployments>
  <Deployment Id="8230be56-fcdf-4e9e-b9c1-f174bbfea928" EnvironmentId="Environments-1" TenantId="" ProjectId="Projects-1" PackageId="Test" PackageVersion="1.0.0" InstalledOn="2018-06-12 05:21:22" ExtractedFrom="C:\Octopus\Tentacle\Files\Test@S1.0.0@5C27F0F6BA16734C896550E72C34750A.zip" ExtractedTo="C:\Octopus\Applications\Tentacle\Local\Test\1.0.0" RetentionPolicySet="Environments-1/Projects-1/Step-Package/Machines-1/&lt;default&gt;" CustomInstallationDirectory="C:\Octopus\Applications\Tentacle\Local\Test\1.0.0" WasSuccessful="True" />
  <Deployment Id="6f7725da-54cf-447f-a3e3-4f4edd690d25" EnvironmentId="Environments-1" TenantId="" ProjectId="Projects-1" PackageId="Test" PackageVersion="1.0.0" InstalledOn="2018-06-12 05:29:55" ExtractedFrom="C:\Octopus\Tentacle\Files\Test@S1.0.0@5C27F0F6BA16734C896550E72C34750A.zip" ExtractedTo="C:\Octopus\Applications\Tentacle\Local\Test\1.0.0_1" RetentionPolicySet="Environments-1/Projects-1/Step-Package/Machines-1/&lt;default&gt;" CustomInstallationDirectory="C:\Octopus\Applications\Tentacle\Local\Test\1.0.0_1" WasSuccessful="True" />
 ```



 ## Process

Retention is run in a block at the end of the deployment. It is run for each action in the deployment that indicates it requires retention.

For each of these actions, a `RetentionPolicySet` key is derived. It contains the Environment, Project, Step Name, Machine Id and Tenant.

It then looks through the deployment journal for any entries with that `RetentionPolicySet` key and works out which entries should be removed according to the configured options. For those entries that are removed it will delete the `ExtractedTo` directory and `ExtractedFrom` file *unless* any kept entry still refers to that directory or file.

## Limitations

Since the `RetentionPolicySet` contains the Step Name, renaming a step will cause the previous deployments for that step to *never* be cleaned up.

The `MachineId` is used in case the same target is registered multiple times (e.g. was used for tenants before we had that feature). This means if the target is removed and re-registered no cleanup will occur of those older files.
