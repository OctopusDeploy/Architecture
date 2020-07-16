 - **Subject**: Design Principles For Octopus Server Clients
 - **Decision date**: TBD
 - **Decision contributors**: Pawel Pabich, Shannon Lewis, Rob Wagner, Mike Noonan
 - **Decision maker**: TBD
 - **Decision owner**: School Of Architecture

## Executive Summary

 Clients should be as light as possible and the heavy lifting should be done by the server. 

## Decision

Keep the logic in the server. 

If references to existing resources need to be passed to a command as one of its inputs then there should be a way to pass them either by `Id` or `Name`. Referring to resources by name might lead long term to a less stable solution but it helps our customers to get a working solution quickly.  

Any errors should be reported back with enough details so the customer knows which inputs need to be modified. 

## Options & Considerations: 

### Keep the logic in the server


Pros:

* Clients become simple, technology specific wrappers around server API and they don't have to be modified every time the logic changes.
* Clients are nice-to-have because a plain HTTP client can be easily used to execute the logic on the server.

Cons:

* Fixes to the logic require a new version of the server.

### Keep the logic in the client

Pros:

* Fixes to the logic can be shipped without releasing a new version of the server.

Cons:

* This approach results in a significant code duplication because each client needs to re-implement the logic. If there is a bug in the current implementation then the corresponding bug fix needs to be applied multiple times. This process is error prone and consumes a significant amount of time.

* Customers have to either re-implement the logic themselves or use one of the clients provided by us. They can't make a simple call to the server using their favorite HTTP client.



## Data Sources

Source listed below might provide additional context  but keep in mind that they can disappear at any time.

* [Should we keep Octopus Server clients simple?](https://octopusdeploy.slack.com/archives/C033W4273/p1594256099456300)
* [Should we implement a new complex octo command in the client or in the server?](https://octopusdeploy.slack.com/archives/CTZT49JFJ/p1591323248186100)
* [It would be great if setting a tenant variable was a one line operation.](https://octopusdeploy.slack.com/archives/C033W4273/p1554878139066700)