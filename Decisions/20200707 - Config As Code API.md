# Overview

- **Subject**: Public API Surface for Configuration As Code Feature
- **Decision date**: [when was the decision made]
- **Decision contributors**: Andrew Best, Paul Gradie, Adam McCoy, Matt Richardson, Andrew Moore, Rob Wagner
- **Decision owner**: School Of Architecture

# Executive Summary

For the _Configuration As Code - Multi Branch Round Trip_ pitch, one of the key outcomes was an agreed approach for the API surface that would drive CaC functionality.

Problems and constraints were identified in the _Considerations_ document attached below, and potential solutions to the problems that satisfied the constraints developed and discussed with the School of Architecture.

The final agreed decisions are documented below.

# Detail

## Considerations

Captured in this document: https://docs.google.com/document/d/1d9XBGd9akeHWn0B-sTPFV6KvfsR17abpBrHtoBRQadE

## Decisions

- Resource Identification:

VCS Resources will be identified by `{ProjectId}/{GitRef}` for resources that have a 1:1 relationship with a Project, and `{ProjectId}/{GitRef}/{ResourceSlug}` for resources that have a 1:\* relationship with Project.

- Resource Location:

VCS Resource location will be enabled by providing default links (links to the version of the resource on the default VCS branch) from the `Project` resource, and providing specific links to resources from other GitRef types from GitRef resources - for example, `VCSBranchResource` will have a `DeploymentProcess` Link that links to the DeploymentProcess for that branch.

- Resource Modification:

To modify a VCS-stored resource, commit details can optionally be supplied to the API.

To model this, we will create `Envelope` resources that embed both the `Resource` being manipulated, along with the metadata required for the domain operation being executed - in this case, a `Commit` of a resource.

- Retrieving VCS metadata:

VCS branches, tags, and commits will be retrievable from hypermedia links exposed on the `Project` resource. This provides access to the _Resource Location_ capability described above.

## Data Sources

Source listed below might provide additional context but keep in mind that they can disappear at any time.

- [API Design Document](https://docs.google.com/document/d/1d9XBGd9akeHWn0B-sTPFV6KvfsR17abpBrHtoBRQadE)
- [School Of Architecture Feedback](https://octopusdeploy.slack.com/archives/CTZT49JFJ/p1594593468361100)
