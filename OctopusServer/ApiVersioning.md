# API Versioning

## Overview

This document describes how we version the Octopus Server API.

## Chosen Approach

TBD

## Scenarios

Assumption: For these scenarios the decision has been made that we want to make the change and maintain backwards compatibility for some period of time.

The scenarios are:
- **Rename** - The property `NuGetFeedId` should be renamed to `FeedId` wherever it exists (e.g on the `DeploymentAction` resource of the `DeploymentProcess` API). 
- **New Field** - An optional `Description` column is to be added
- **New Type** - The `Feed` API will now return an additional feed type `AwsFeed` that contains a mandatory field `Region`

### Out of scope

For the purposes of this proposal, the following is out of scope:

- Whether to make a change in the first place
- Versioning of config as code documents
- Versioning of step types
- Policy and mechanics of the removal of old versions, other than it will be done

## Background

Up until now we have not versioned the API since `Server 3.0`. We've chosen between hard breaking the API contract, keeping it fully compatible and somewhere in between.

One of the reasons we didn't adopt versioning was that there was no intention to remove old versions. This proposal implies we will remove them at some stage.

### Implementation

In the most common case we handle the requests and responses to be compatible with both the old (`v1`) and new (`v2`) schemas.

We use some marker in the request to determine whether it is `v1` and map to `v2` if required. e.g. If we receive a request with `NuGetFeedId` we will map that value to `FeedId`.

On response we write out the response in a way that is compatible with both schemas. e.g. the response will contain both `NuGetFeedId` and `FeedId`. For `list` we will return all records.

Clients implemented similar logic to determine if it received a `v1` or `v2` response or whether posting a `v2` formatted is possible.

### Applied to Scenarios

Applying the scenarios to the current approach:

- **Strict client** - The client deserializes the resource into a well known object structure, ignoring any fields it does not understand (e.g. C# client)
- **Loose client** - The client either manipulates the JSON directly or deserializes into a dynamic structure, including all fields in the resource (e.g. Javascript, PowerShell, JObject)


| Scenario | Strict Client | Loose Client |
| -------- | ------------- | ------------ |
| Rename | Handled transparently | On `update` the client will send both properties, making it hard to discern intent if the conflict |
| New Field | Field will be dropped on `update` | Handled transparently |
| New Type | On `list` it may work or fail depending on the operation, possibly silently | On `list` it may work or fail depending on the operation, possibly silently

## Outcome

By adoption API versioning we will have the freedom to radically change the schema and behaviour of our endpoints (within the bounds of the backend storage considerations). 

It would be as if we added a brand new endpoint. The `v1` API will retain it's behaviour, including limiting the records returned on a `list` to those previously understood (e.g. exclude `AwsFeed`). 

We will deprecate and remove previous versions of an API. The exact mechanism is the subject of a different proposal.

### Community Library

Community library steps would be treated the same as if they were posted via the API. We would need to update the library and sync process to indicate the `DeploymentProcess` API version targeted.


## SemVer

We would use SemVer where applicable to version the API. A major increment indicates a breaking change in the contract (e.g. field removal, new type, rename), while a minor increment indicates an addition (e.g. new field). 

Clients could choose how they handle a minor increment. e.g. The C# client can't roundtrip a minor change, so it would work for `get` but not for `update`. The javascript client can roundtrip, so would continue to work in a minor increment.

## Options - Surface Area

### Document

The document will contain the schema version. If it is missing, `v1` is assumed.

This client would still need another way to indicate the expected version on a `get`.

#### Conclusion
This option would likely just be a complication on some sort of API versioning.

### Whole of API

The whole API is versioned in tandem, including the root document. To a client, the API at a specific version never changes. 

This is easy to reason about and understand from a client perspective. If we made large but rare changes to the API, this would work well. 

The downside is that every route will  have a version bump regardless whether it changed, making it difficult to track which APIs have changed. Also if we retired an API version that an older client was using, it would break that API even though the client may have been fully compatible with the latest API.

#### Conclusion

This does not suit our cadence or the desire for not breaking clients without reason. We tend to make small and rare breaking changes.

### Per Endpoint

The API will be versioned per endpoint. An endpoint being a group of routes (and method) that handle the same shapped data. Example `/api/Spaces-1/feeds/all` and `/api/Spaces-1/feeds/{id}` would be grouped, but `/api/Spaces-1/feeds/summary` would not.

The client can choose to use new APIs bit by bit. It can also fall back to a previous API if it knows how. 

The `v1` and `v2` schemas will be handled by different responders and different resource types. This will make it easy to remove `v1` when the time comes. Removal all also only affect those clients actively using that endpoint.

#### Conclusion

**Recommend** This fits our use case the best

## Options - Mechanism

### Hypermedia Links

The API will be versioned via hypermedia.

In the case of per-endpoint versioning, the root document will contain additional links for each version. For example:
```json
"Links": {
    "DeploymentProcesses":  "`/api/Spaces-1/deploymentprocesses`",
    "DeploymentProcesses/v2":  "`/api/Spaces-1/deploymentprocesses-v2`",
}
```

The route itself will contain an additional version identifier after the resource name, e.g. `/api/Spaces-1/deploymentprocesses/v2` or `/api/Spaces-1/deploymentprocesses-v2`.

The client will pick the link knows about and wants to use. If the link is not available the client can fallback to a previous version, or present an informative error message. For clients not using the hypermedia, they will get a `404`.

This method also allows users to continue browsing the API via the browser.

#### Conclusion

**Recommend** This approach fits with out existing approach of API discovery. Clients can easily discover whether the version of the API they depend on still exists.

### Header

The API will be versioned per endpoint `ContentType` and `Accept` headers. e.g:
```
ContentType: application/vnd.octopus.deploymentprocess.v2
Accept: application/vnd.octopus.deploymentprocess.v3; q=1.0, application/vnd.octopus.deploymentprocess.v2; q=0.9
```

The correct handler will be picked based on the header value. If the server does not support the specified version, it should return `415 Unsupported Media Type` or `406 Not Acceptable`.

The client would tell the server all the versions it accepts.

#### Conclusion

This fits better with the spirit of a Resource based API. However it does not allow browsing and is more difficult to understand. 

### Query String

The API would be versioned by passing a query string e.g `&version=2.0.0`.

If the server does not support that version a `404` is returned.

#### Conclusion

This is convenient, but feels like the wrong use of query strings.

### Root Route

The root of the API would become `/api-v2` or `/api/v2`. This implies whole of API versioning.

#### Conclusion

This (or domain name based versioning) seems to be the most common approach used for whole of API versioning.

### All the options

See [Troy Hunt](https://www.troyhunt.com/your-api-versioning-is-wrong-which-is/).

The consumer would choose the method they prefer, header, URL or content type.

#### Conclusion

With some clever pipeline logic we could make this transparent to the developer. The appeal is in the flexibility. It's not clear how well it would work with per-api versioning.



## Options - version specifiers

### No version

If the client doesn't specify API version then we should fallback to the oldest supported version by the Server. This approach will keep existing clients as long operational as possible. There is a decent chance that the client won't be affected by the change. E.g. There is a new property in the property bag.

Example. If the server supports `/api/Spaces-1/projects/v1` and `/api/Spaces-1/projects/v2` we should fallback to `/api/Spaces-1/projects/v1`. When the server moves on and only supports `/api/Spaces-1/projects/v2` and `/api/Spaces-1/projects/v3` we should fallback to ``/api/Spaces-1/projects/v2``.


