# API Versioning

## Overview

This document describes how we version the Octopus Server API.

## Chosen Approach

TBD

## Scenarios

For these scenarios the decision has been made that we want to maintain backwards compatibility for some period of time.

The scenarios are:
- **Rename** - The property `NuGetFeedId` should be renamed to `FeedId` wherever it exists (e.g on the `DeploymentAction` resource of the `DeploymentProcess` API). 
- **New Field** - An optional `Description` column is to be added
- **New Type** - The `Feed` API will now return an additional feed type `AwsFeed` that contains a mandatory field `Region`

### Out of scope

For the purposes of this document, the following is out of scope:

- Whether to make a change in the first place
- Versioning of config as code documents
- Versioning of step types
- Policy and mechanics of the removal of old versions

## SemVer

We would use SemVer where applicable to version the API. A major increment indicates a breaking change in the contract (e.g. field removal, new type, rename), while a minor increment indicates an addition (e.g. new field). 

Clients could choose how they handle a minor increment. e.g. The C# client can't roundtrip a minor change, so it would work for `get` but not for `update`. The javascript client can roundtrip, so would continue to work in a minor increment.

## Community Library

Community library steps would be treated the same as if they were posted via the API. We would need to update the library and sync process to indicate the `DeploymentProcess` API version targeted.

## Options

### No Explicit Versioning

This is the approach we've used in the past, for example when adding support for feeds other than NuGet.

The API will not be explicitly versioned. We will handle the requests and responses to be compatible with both the old (`v1`) and new (`v2`) schemas.

We will use some marker in the request to determine whether it is `v1` and map to `v2` if required. e.g. If we receive a request with `NuGetFeedId` we will map that value to `FeedId`.

On response we will write out the response in a way that is compatible with both schemas. e.g. the response will contain both `NuGetFeedId` and `FeedId`. For `list` we will return all records.

Clients would need to implement similar logic to determine if it received a `v1` or `v2` response or whether posting a `v2` formatted is possible.

#### Scenarios

- **Strict client** - The client deserializes the resource into a well known object structure, ignoring any fields it does not understand (e.g. C# client)
- **Loose client** - The client either manipulates the JSON directly or deserializes into a dynamic structure, including all fields in the resource (e.g. Javascript, PowerShell, JObject)


| Scenario | Strict Client | Loose Client |
| -------- | ------------- | ------------ |
| Rename | Handled transparently | On `update` the client will send both properties, making it hard to discern intent if the conflict |
| New Field | Field will be dropped on `update` | Handled transparently |
| New Type | On `list` it may work or fail depending on the operation, possibly silently | On `list` it may work or fail depending on the operation, possibly silently

#### Conclusion
This approach is viable, but limited
 - The resources need to be kept closely aligned and largely compatible, limiting the scope of the change
 - Data may be inadvertently lost
 - The client may fail unexpectedly

### Document Versioning

The document will contain the schema version. If it is missing, `v1` is assumed.

This client would still need another way to indicate the expected version on a `get`.

#### Conclusion
This option would likely just be a complication on some sort of API versioning.

### Per Endpoint Hypermedia Versioning

The API will be versioned per endpoint via hypermedia. The root document will contain additional links for each version. For example in addition to `DeploymentProcesses`, it will contain `DeploymentProcesses/v2`. 

The route itself will contain an additional version identifier after the resource name, e.g. `/api/Spaces-1/deploymentprocesses/v2` or `/api/Spaces-1/deploymentprocesses-v2`. Other approaches (e.g. parameters or header) do not fit with the hypermedia link approach.

The client will pick the link knows about and wants to use. If the link is not available the client can fallback to a previous version, or present an informative error message. For clients not using the hypermedia, they will get a `404`.

This option is as if we added a brand new endpoint, allowing a total change of schema and behaviour. The `v1` API will retain it's behaviour, including limiting the records returned on a `list` to those previously understood (e.g. exclude `AwsFeed`). 

The `v1` and `v2` schemas will be handled by different responders and different resource types. This will make it easy to remove `v1` when the time comes. Removal all also only affect those clients actively using that endpoint.

#### Other versioning schemes
Passing the version via a query parameter (`&version=2.0.0`) or header (`X-Octopus-Api-Version: 2.0.0`) does not fit with the hypermedia link approach.

Adding the version before the resource in the route also does not make sense because `v2` of the `events` api did not appear at the same time as `v2` of `deploymentprocesses`.

#### Conclusion

**This is the recommended option**. The versioning is targeted to the API surface being changed and is the easiest to maintain and understand.

### Per Endpoint Header Versioning

The API will be versioned per endpoint `ContentType` and `Accept` headers. e.g:
```
ContentType: X-Octopus-DeploymentProcess-V2
Accept: X-Octopus-DeploymentProcessResponse-V2
```

The correct handler will be picked based on the header value. If the server does not support the specified version, it should return `415 Unsupported Media Type` or `406 Not Acceptable`.

The client would be able to determine the API version the server supports by [TBD].

This will also allow total change of schema and behaviour.


### Whole of Hypermedia Versioning

The root document itself is versioned. For example in `v1` of the root document the `DeploymentProcesses` link points to `/api/Spaces-1/deploymentprocesses` and in v2 it is `/api/Spaces-1/deploymentprocesses-v2`. Links and routes that have not changed between version will remain the same.

The client will pick the version of the API it wants in advance instead of per call. This simplifies the client a little, but hinders it from implementing fallback logic if desired.

This will also allow total change of schema and behaviour.

#### Conclusion

This approach doesn't have significant advantages over the `Per Route Hypermedia Versioning` option

### Whole of API versioning

A root document for each API version is available. It will contain all the routes available to that version. The link names stay the same, but the routes change to include the version either in the path (`/api/v2/Spaces-1/deploymentprocesses`) or via a query parameter (`&version=2.0.0`) or header (`X-Octopus-Api-Version: 2.0.0`).

This will also allow total change of schema and behaviour.

On the implementation side we will have the flexibility of just adding a new route, or transforming the whole API. 

This style of versioning is also the easiest for an end user to reason about and understand. This would lend itself well to whole of API changes. 

The downside is that every route will  have a version bump regardless whether it changed, making it difficult to track which APIs have changed. Also if we retired an API version that an older client was using, it would break that API even though the client may have been fully compatible with the latest API.

#### Conclusion

This option has some merit, but the removal scenario isn't that great.

