# Introduction

Neutrality is one of the selling points of Octopus. We aim to support all the most popular cloud native platforms, while retaining a consistent user experience and common best practices.

To achieve this, all new steps (which will typically be cloud steps) in Octopus should aim to implement a consistent set of base functionality. This document provides guidelines for pitching steps that are consistent, implement the lessons we have learned over the years, and express our opinions on best practice deployments.

# Pitch guidelines

The following guidelines capture the functionality and opinions that we should apply to all steps.

## Ship incrementally

Team steps has devoted a great deal of effort to building a framework that allows features to be shipped independently of the main Octopus code base. This allows us to ship features more frequently, and respond to user feedback more quickly.

To take advantage of this, pitches should be broken down into milestones. Each milestone should:

* Deliver enough value to be useful for our customers.
* Deliver or build on an opinionated deployment process.
* Lean towards delivering simple solutions that provide a easy migration path should more complexity be required in future milestones.

## Be opinionated

We are leaders in the deployment space. We have years of experience and our tools have performed millions of deployments, and customers look to us to provide clear opinions on what best practice deployments look like. These values have been distilled into [The ten pillars of pragmatic deployments](https://octopus.com/blog/ten-pillars-of-pragmatic-deployments).

The steps we deliver must have a clear vision for how they enable teams to deliver best practice deployments. Often this involves combining, hiding, not supporting, or hard coding some features in the platforms we deploy to.

Where there is no clear decision on an opinion, refer to **Ship incrementally** for guidance.

## Graduated path from opinionated to raw scripts

Highly opinionated steps will fail some of our customers all of the time. To support those with advanced use cases, steps should provide a graduated path from opinionated (and often monolithic) steps, to granular and composable steps, to raw templates or scripts.

This path allows us to express our opinions regarding best practice deployments with opinionated steps that will often merge many underlying platforms and resources. The granular steps provide the ability to compose deployments in unique ways, while still retaining the benefits of a UI driven approach. Raw templates or scripts provide the ultimate level of customization, free of the opinions baked into the specialized steps.

## Use targets

Targets must be used to define where a deployment takes place. Targets should capture:

* The credentials required to perform a deployment.
* The default worker a deployment will be performed on.
* Cloud specific details like regions, projects, and resource groups.
* The name of the service being deployed to for a third layer service.
* The default name of any deeper partitioning on a third layer service. This name must be able to be overridden on the steps.
* The default name of the deployable artifact for a second layer service. This name must be able to be overridden on the steps.

### Second layer service

A second layer service has only one instance in a cloud provider, and a one to many relationship with the deployments it holds. Examples include:

* AWS Lambdas
* AWS App Runner
* GCP App Engine
* GCP Cloud Run

![](assets/secondlayerservice.png)

### Third layer service

A third layer service has a many to one relationship with the cloud provider, and a one to many relationship with the deployments it holds. Examples include:

* Azure web apps
* Kubernetes clusters
* ECS clusters
* Service Fabric
* AWS API Gateway

![](assets/thirdlayerservice.png)

### Overriding target defaults

Second layer services are typically smaller targets i.e. there are likely to be many small deployments. Examples include individual lambda functions or single cloud run containers.

Third layer services may also have a way to partition deployments they hold. Examples include namespaces in Kubernetes clusters.

We have found that exposing second layer service names and third layer nested partitioning values as overridable default values on a target provides the most flexibility. Allowing target values to be overridden on the steps gives customers the flexibility to lift all information about the deployment destination from the steps, or to perform many smaller deployments without an explosion of targets.

## Credentialless authentication

All targets should support the ability to inherit credentials from the worker a deployment is executed from, and avoid the need for long lived credentials to be maintained by Octopus.

## Declarative over imperative

All cloud providers and modern orchestration platforms offer a template language for defining resources. They also provide a great deal of tooling and reporting for these "managed" resources. This speaks directly to the [auditable deployments](https://octopus.com/blog/ten-pillars-of-pragmatic-deployments#auditable-deployments) pillar.

Steps should prioritize the creation of resources through declarative templates rather than imperative CLI or SDK commands.

## Advanced deployment patterns

Many cloud native platforms have built in support for blue/green or canary deployments. If so, we want to expose these as part of an Octopus deployment where it makes sense.

We also want to provide support for other deployment scenarios liek hotfixes and feature branches. These are often not natively supported by the target platforms.

Where a target platform has functionality that prevent advanced deployment patterns, we'd likely offer an opinionated step that does not use the conflicting functionality so as to promote deployments that enable advanced deployment patterns. An example of this is [AWS API Gateway](https://octopus.com/blog/deploying-lambdas#why-limit-ourselves-to-one-stage-per-environment), whose stages prevent hotfixes and feature branches.

## Combine loosely coupled externalized configuration with the deployment

Deployments will often rely on externalized configuration. 

In some cases the configuration is tightly bound to the deployment. For example, environment variables are often defined as part of a deployment, and tightly bound to the application they are configured against.

In other cases two loosely coupled resources will combine to define a deployment. For example, a Kubernetes Pod may reference a ConfigMap for environment variables.

Where two loosely coupled resource must exist side by side for a deployment to operate in a predictable manner, consider an opinionated step that deploys both. This will help customers fall into the pit of success.

## Tags and labels

A failed deployment will often leave a number of old resources laying around. Eventually these need to be cleaned up. To identify old resources, anything that can be tagged or labeled should include the following values by default:

* `Octopus.Project.Id`: "projects-1"
* `Octopus.Action.Id`: "8427fd24-17c8-4c7d-b32f-d2b8d51f2121"
* `Octopus.Deployment.Id`: ""
* `Octopus.RunbookRun.Id`: "runbookruns-86"
* `Octopus.Step.Id`: "b6f8fd75-3acf-4186-a573-285a1aa11a9a"
* `Octopus.Environment.Id`: "environments-4"
* `Octopus.Deployment.Tenant.Id`: "untenanted"

These tags have proven useful for identifying resources related to previous deployments, regardless of environments, tenants etc.

## Output variables

Many deployments will create resources that need to be consumed by subsequent steps. The following are guidelines for output variables:

* Links to any resources accessible via a URL. This helps with testing.
* Names of any generated resource identifiers.
* Details of any generated revision numbers.

## Documentation

All new features must be documented. Ensure this is a required feature of any pitch being developed.

## Remember to update associated projects

New steps and targets will need to be reflected in other projects:

* Octopus client - this may need to be updated with new target types.
* Dynamic target scripts - the [docs](https://octopus.com/docs/infrastructure/deployment-targets/dynamic-infrastructure) will need to be updated with any new scripts.

# Pitch documents

The pitch process generates a number of documents, each with it's own use case and audience. These are detailed below.

## RFC blog post

The first document is a Request For Comment (RFC) style blog post. The purpose of this post is to:

* Inform customers, partners, and other internal Octopus departments of the proposed new functionality.
* Highlight the details of the new functionality, calling out how it expresses our vision of best practice deployments.
* Include enough UI mockups to provide a sense of what the feature looks like.
* Provide a link to a [GitHub issue](https://github.com/OctopusDeploy/StepsFeedback/issues) where people can provide feedback.
* But never promise to deliver any specific feature. Always call out that the feature has not been committed to, and that there is no release date.

## Pitch document

The pitch document is based on the [Shape Up](https://basecamp.com/shapeup/webbook) process.

This document is structured with the following sections:

* Executive Summary - two or three paragraphs outlining how this pitch aligns to the company's strategic goals, why this particular functionality is important, and a high level overview of the proposed solution.
* Problem - A description of the problems faced by customers that this pitch will solve.
* Constraints - Each pitch relates to a milestone, and milestones have been constrained to deliver the smallest unit of useful functionality. This section lists the constraints that exclude work to be done.
* Proposed Scope - This is the opposite of the constraints, and provides an overview of the features to be delivered. This section should link to the blog post and the UI table document.
* Future Considerations - Most pitches expect to be followed up with another related pitch with the next milestone of work. This section provides a very high level list of features that might be considered in the next milestone.

## UI table

This document details the proposed UI structure of the new steps. This is useful when collaborating with the design team to find any unique UX patterns that may need to incorporated into Octopus:

![](assets/uimockup.png)

An example of this document can be found [here](https://docs.google.com/document/d/13jZVn_L6U3_iFEy8W12NO2TochPEu_g_ZNEjo8R652o/edit).

# Post release tasks

There are a number of activities to perform once a new feature is made available:

* [Uservoice](https://octopusdeploy.uservoice.com/) - Close any suggestions that have now been satisfied.
* [Customer Solutions Product Feedback](https://trello.com/b/vZEB7drD/customer-solutions-product-feedback) - Add a note to any scenario that is now satisfied.
* [Issues Repo](https://github.com/OctopusDeploy/Issues/issues) - Close any issues that have been solved.
* [Feedback Repo](https://github.com/OctopusDeploy/StepsFeedback/issues) - Add a note to any feedback issue created for the milestone, and close the issue.
* [Terraform provider](https://github.com/OctopusDeployLabs/terraform-provider-octopusdeploy) - this will need to be updated with new targets. Ping #team-integrations when the target is available in master.

# References

* [Steps pitch process proposal](https://docs.google.com/document/d/1b94WXWKuGkocP8krgBhjv-mZEWOi2xTNJH_GnZbn360/edit) - A discussion around the limitations of a single pitch document.
* [Account selection for steps](https://docs.google.com/document/d/1MNgIGoE8Jponw9JknGlZylkNThMwd45IOsdkKf8pOok/edit#heading=h.bpy38qfq9mw9) - Why we use targets instead of accounts in steps.
* [Shape Up](https://basecamp.com/shapeup/webbook)
* [The ten pillars of pragmatic deployments](https://octopus.com/blog/ten-pillars-of-pragmatic-deployments)
* [Customer Solutions Product Feedback](https://trello.com/b/vZEB7drD/customer-solutions-product-feedback) - A trello board capturing customer feedback on Octopus.
* [Uservoice](https://octopusdeploy.uservoice.com/)
* [Issues Repo](https://github.com/OctopusDeploy/Issues/issues)
* [Feedback Repo](https://github.com/OctopusDeploy/StepsFeedback/issues)