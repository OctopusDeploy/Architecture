# Overview

- **Subject**: Resource model for domain interactions
- **Decision date**: 2020-08-03

# Executive Summary

The Octopus API will leverage an _enveloped resource_ model for transferring resource and domain information to enable modelling of domain interactions.

# Detail

## Considerations

The Octopus API is strongly Resource-oriented, with a single model per Resource type used to both retrieve resources, and create or modify them.

This model does not support information for read-models or domain interactions well, as information outside of core resource state ends up being bundled together with the resource state fields, leading to single resource models that may be presented in different states in different cases.

The Config As Code API decision record presented a requirement to model a specific domain interaction - Version Control Commits as a way of persisting Version Controlled Resources.

There are other known features that would have the same requirement as well, such as capturing audit information any time a process is changed.

## Decision

An enveloped resource model would:

- allow us to continue to leverage our existing resource models
- sit alongside existing endpoints, not requiring any refactoring of the API for consistency
- allow us to provide domain-specific information to our API alongside existing resource models, better enabling modelling domain interactions
- cleanly separate domain-specific information from resource state information
- doesn't preclude a future state where we would move toward asymmetric read and write resource models for our API

An example of an enveloped resource (modelled on the Configuration As Code example above) might look like so:

```
POST/PUT

{
    “Resource”: {
        “Id”: “deploymentprocesses-1234”,
        “Name”: “My process”,
        ...
    },
    “CommitMessage”: “Added another step”
}
```

## Data Sources

Source listed below might provide additional context but keep in mind that they can disappear at any time.

[Config As Code API ADR](https://github.com/OctopusDeploy/Architecture/blob/master/Decisions/20200707-ConfigAsCodeAPI.md)

[Config As Code API Options & Considerations Document](https://docs.google.com/document/d/1d9XBGd9akeHWn0B-sTPFV6KvfsR17abpBrHtoBRQadE/edit#bookmark=id.avlcu8kxv8cq)
