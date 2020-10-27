# Config as Code

## Delete a project

**When a project is deleted, we won't delete any files from git**

- [Decision in Slack](https://octopusdeploy.slack.com/archives/C01A1E8K9J5/p1599463673108000?thread_ts=1599458934.107900&cid=C01A1E8K9J5)

## Delete requests

**When deleting anything from git through our API, we supply the commit message in the delete body**

Normally, our delete requests have no body since the Id of the resource you are trying to delete is supplied in the route. 

In the case of Config as Code, we require extra information in the form of a commit message. For delete requests, we can provide the commit message through the body of the request.

An example of this is kind of operation is deleting a runbook.

- [Discussion in Slack](https://octopusdeploy.slack.com/archives/C01AJE4K3T2/p1600066810120000)

## Document Stores

**We should encapsulate access to our git-based documents within VCS "Document Stores", which are distinct from our database-based "Document Stores".**

We may want to consolidate our VCS and Database access behind a single "Document Store" in future, but this proves to be non-trivial due to the different signatures required for these two types of stores. For example, the VCS store requires commit messages and git references, while the database-backed store does not.

It is currently unclear the best path forward for consolidating these two types of stores, so keeping them separate for now gives us flexibility and allows us to iterate on this design more easily in future.

These VCS document stores are the place where we should apply permission checks.

- [Decision in Slack](https://octopusdeploy.slack.com/archives/C01AJE4K3T2/p1600058089113200?thread_ts=1600051433.112900&cid=C01AJE4K3T2)
- [Example of a "Database-based document store": ProjectDocumentStore](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Core/Features/Projects/ProjectDocumentStore.cs)
- [Example of a "VCS-based document store": VersionControlledDeploymentProcessDocumentStore](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Core/Features/DeploymentProcesses/VersionControlledDeploymentProcessDocumentStore.cs)

## Project Cloning

**Project cloning is not yet supported for VCS projects, but may be in future**

In part, cloning/copying/templating is what CaC is supposed to solve, but within Git rather than within Octopus. In future there is potential for new features here, such as cloning a project into a new repo, cloning into a new project from an existing repo, etc.

- [Decision in Slack](https://octopusdeploy.slack.com/archives/C01A1E8K9J5/p1599463673108000?thread_ts=1599458934.107900&cid=C01A1E8K9J5)