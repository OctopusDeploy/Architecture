 - **Subject**: Design Principles For Octopus Server Clients
 - **Decision date**: 23/07/2020
 - **Decision contributors**: Pawel Pabich, Shannon Lewis, Rob Wagner, Mike Noonan
 - **Decision maker**: School Of Architecture
 - **Decision owner**: School Of Architecture

## Executive Summary

 Clients should be as light as possible and the heavy lifting should be done by the server. 

## Decision

Keep the domain logic in the server. 

The only logic clients should be responsible for is:
- marshalling user provided parameters
- allowing users to specify resource parameters by `Name` or `Id`
  - Referring to resources by name might lead long term to a less stable solution but it helps our customers to get a working solution quickly.  
- reporting errors
  - Any errors should be reported back with enough details so the customer knows which inputs need to be modified.

 

## Options & Considerations: 

### Keep the domain logic in the server


Pros:

* Clients become simple, technology specific wrappers around server API and they don't have to be modified every time the logic changes.
* Clients are nice-to-have because a plain HTTP client can be easily used to execute the logic on the server.

Cons:

* Fixes to the logic require a new version of the server.

### Keep the domain logic in the client

Pros:

* Fixes to the logic can be shipped without releasing a new version of the server.

Cons:

* This approach results in a significant code duplication because each client needs to re-implement the logic. If there is a bug in the current implementation then the corresponding bug fix needs to be applied multiple times. This process is error prone and consumes a significant amount of time.

* Customers have to either re-implement the logic themselves or use one of the clients provided by us. They can't make a simple call to the server using their favorite HTTP client.



## Examples

[Runbook run command in Octopus CLI](https://github.com/OctopusDeploy/OctopusCLI/blob/28e6e2014c708593609979b4ad8e8f1e51c21c7b/source/Octopus.Cli/Commands/Runbooks/RunRunbookCommand.cs), [runbook run API in Octopus Client](https://github.com/OctopusDeploy/OctopusClients/blob/ba2cb799eaf3d8f993b01e3d6af965fb64c586be/source/Octopus.Client/Repositories/RunbookRepository.cs#L77-L86) and [runbook run endpoint in Octopus Server](https://github.com/OctopusDeploy/OctopusDeploy/blob/3de0f22fd5bcf6066a6396e1bfaa5de30960233f/source/Octopus.Server/Web/Api/Actions/RunbookRunForPublishedRunbookCreateAction.cs) are good examples of how this principle can be applied to implement an end to end scenario.   

Both Octopus CLI and Octopus Client simply perform a few lookups and pass the data through to Octopus Server. If the runbook run endpoint allowed clients to refer to resources by either `Id` or `Name`  (as opposed to just by `Id`) then the lookups could be removed. `TODO`: Add link to ADR for `Resource References`.

If an input parameter required by a server endpoint is difficult for the client to calculate and the server can't calculate it automatically then it might make sense to expose a template endpoint for that endpoint. Template endpoint is an ancillary endpoint that provides default values for the difficult to calculate parameters.

[ReleaseTemplateAction](https://github.com/OctopusDeploy/OctopusDeploy/blob/c2ef1c646684e58e614922313e25434ca3022847/source/Octopus.Server/Web/Api/Actions/Releases/ReleaseTemplateAction.cs) endpoint is a template endpoint that can be used to retrieve some of the data required by [ReleaseCreateResponder](https://github.com/OctopusDeploy/OctopusDeploy/blob/2518dfd572f58209da1ec3fbaa4a3dfc85f4cbae/source/Octopus.Server/Web/Api/Actions/Releases/ReleaseCreateResponder.cs) endpoint. [NextVersionIncrement](https://github.com/OctopusDeploy/OctopusDeploy/blob/436c820fe691c3234f067672c4d7fc6184f7ca36/source/Octopus.Core/Resources/ReleaseTemplateResource.cs#L14) is a good example of an input parameter that might difficult to calculate on the client side.



## Data Sources

Sources listed below might provide additional context  but keep in mind that they can disappear at any time.

* [Should we keep Octopus Server clients simple?](https://octopusdeploy.slack.com/archives/C033W4273/p1594256099456300)
* [Should we implement a new complex octo command in the client or in the server?](https://octopusdeploy.slack.com/archives/CTZT49JFJ/p1591323248186100)
* [It would be great if setting a tenant variable was a one line operation.](https://octopusdeploy.slack.com/archives/C033W4273/p1554878139066700)