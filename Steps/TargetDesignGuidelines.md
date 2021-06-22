# Introduction

Neutrality is one of the selling points of Octopus. We aim to support all the most popular cloud native platforms, while providing a consistent user experience and common best practices. To achieve this, all new targets in Octopus should aim to implement a consistent set of base functionality. 

This document provides guidelines for designing targets that are consistent and express our opinions on best practice deployments.

# Target design guidelines

Targets must be used to define where a deployment takes place. They help ensure deployment steps are decoupled from their final destination, which speaks to the [repeatable deployment](https://octopus.com/blog/ten-pillars-of-pragmatic-deployments#repeatable-deployments) pillar.

Targets should capture:

* The credentials required to perform a deployment.
* The default worker a deployment will be performed on.
* Cloud specific details that locate the target like regions, projects, and resource groups.
* The identifier of the service being deployed to e.g. the name of a ECS cluster, an Azure Web App, a Lambda function etc.
* Identifiers for any nested partitioning e.g. namespaces, slots etc.

## Overrideable and non-overridable fields

We aim to strike a balance between the flexibility of allowing target fields to be overridden on steps, and the pragmatism of encouraging target use as the foundation of repeatable deployments. This is often subjective, so there is no hard and fast rule here.

Targets provide the ability to lift deployment destination details out of the steps, and represent a security boundary within Octopus. For those teams that require a high level of control or a clear distinction between deployment processes and the infrastructure they are deployed to, the solution is to lift all details about the deployment destination into a target.

However, without the ability to override target details on a step, common deployment scenarios like feature branching and microservice deployments will result in an explosion of targets, where the individual targets provide little benefit. In these situations it is useful to consider which target fields must be overridable in steps to support these deployment patterns.

We do expect targets to become more flexible and expose more functionality in future. [Dynamic Environments](https://docs.google.com/document/d/1My5ZLdLeslUq29sQR6xSyEa0UfpwsQxPrXp2dVKZSi4/edit), [Project Groups](https://docs.google.com/document/d/1Yp4dBzGEMVcWWLytjGLMoawjOi0a6hV5yV62MmKWxh4/edit), and [Pipelines](https://docs.google.com/document/d/184_iYwPtctj5X3a44DdFC-T71dc2nO0vHp1GEtdpbxw/edit) provide some insight into the future direction.

Here are some guidelines:

* When in doubt, assume targets describe the **where** (e.g. service names, regions) and the **who** (e.g. credentials). Steps define the **what** (e.g. packages, app configuration) and the **how** (e.g. building templates, running scripts).
* When in doubt, refer to **Ship incrementally** from [Delivery Guidelines](DeliveryGuidelines.md) for guidance. It is easy to override a field in a step, but hard to remove a field once it has been exposed.
* Target accounts should not be overridable. If a target is used to deploy to a wide surface area, the account on the target must have enough permissions to do so.
* Cloud and platform partitions like regions, zones, projects, and namespaces, and individual service partitions like web app slots, should be considered in the context of a feature branch deployment or the deployment of many related microservices:
    * Are teams likely to deploy a feature branch in a new AWS region? Probably not.
    * Are teams likely to deploy a feature branch in a new Kubernetes namespace? This is likely.
    * Are teams likely to deploy related microservices in separate Kubernetes namespaces? This is likely.
    * Are teams likely to deploy a feature branch to a web app slot? This is likely.
    * Are microservices for a related service likely to be deployed in many regions or availability zones? This is likely for high availability. But is this better modeled as a single step with multiple targets, or a duplicated step with overridden AZ fields? Clearly duplicating steps to override one field will result in a mess, and multiple targets are a better choice here.
    * Are teams likely to deploy a feature branch to a new GCP project? This is likely given project scoped resources like GAE routing rules.
* The names of services, such as a Lambda function name, an Azure Container Instance name, or a Google Cloud Run instance, will typically be overridable. Feature branch deployments will almost certainly require renaming these values on a step.
* There is no way to get the union of two targets. For example, the deployment of a Lambda exposed by an API Gateway instance can not combine the details of a Lambda target and an API Gateway target. Where a deployment takes place to two services that could be targets in their own right, consider how a single target can lift those combined details from a step.

## Target identification

The fields on targets exist for us to identify a specific target we want to act upon during a deployment. There are two ways to classify fields that we might consider placing on targets.

### Knowable

The first classification is whether the resources are **knowable**. These are resource names that we know will be used by steps. This might sound obvious at first, but there are reasons why we would not know the resource names used by steps.

The Kubernetes target is a prime example. A single deployment may include one or more configmaps, secrets, persistent volume claims etc. It is impossible to know how many resources to provide default names for. 

Also, with Custom Resource Definitions (CRDs), it is impossible for a target to know the types of resources it is providing default names for.

Supplying default names in a Kubernetes target would involve supplying arrays of default values and custom key/value pairs naming the custom resource that the default value applies to. This is not useful, and so a Kubernetes target does not define such resource names.

### Instantiable

The second classification is whether the resources are **instantiable**. A resource is instantiable if the act of deploying an application usually also creates the resource that hosts it. Or, in other words, instantiable resources are self contained.

Many resources are instantiable. For example, providing a name for a Kubernetes resource, AWS Lambda, AWS App Runner service, Google App Engine service, or Google Cloud Runner service, along with the values usually supplied while deploying the application, is all that is required for that resource to be created during a deployment if it does not exist.

Services like Azure web or function apps do not create the underlying resources with a name. An app service needs a resource group, networking, service plan etc. So simply supplying a name, along with the details usually associated with an application deployment, can not create a web app instance if it doesn't exist.

### Target fields

The following table provides guidelines for including resource names on targets:

| Knowable    | Instantiable  | Target field type | Health Check |
|-------------|---------------|-------------------|--------------|
| Yes  | Yes  | Optional field that can select existing resource. | Verify credentials, don't assume resource exists. |
| Yes  | No   | Mandatory field that must select existing resource. | Verify credentials and verify resource exists. |
| No   | -    | Not shown on target. The steps will define these values. | Verify credentials. |

### Overridable and uninstantiable target values

Where a target field references a resource that is uninstantiable, it usually means it is a non-trivial task to create a new instance.

Where a target field can be overridden by a step, it usually means that deployment patterns like feature branches benefit from deploying to a new resource.

Where a field references a uninstantiable resource that can be overidden on a step, consider how such resources can be created as part of a deployment. It may be a case of documenting a Terraform, CloudFormation, or ARM Template that creates the necessary resources. Or it may be worth providing custom steps that create these resources.

## Credentialless authentication

All targets should support the ability to inherit credentials from the worker a deployment is executed from, and avoid the need for long lived credentials to be maintained by Octopus.

Examples include [AWS EC2 IAM roles](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html), [Azure Managed Service Identity](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview), [Google inherited service accounts](https://cloud.google.com/compute/docs/access/create-enable-service-accounts-for-instances), and [Kubernetes pod service accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/).