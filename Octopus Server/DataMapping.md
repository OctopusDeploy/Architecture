# Overview

- **Subject**: Implement explicit data mapping
- **Decision date**: 14/10/2020

# Executive Summary

Data mappings that required dependencies will be defined in classes that can be resolved and injected with those dependencies. Those that do not will be defined in private methods if testing and reuse are not required, or static methods if they are.

# Detail

## Considerations

Captured in this document: https://docs.google.com/document/d/1C-4DRUEaER_zWmecM5lECkZAHOie3mlaMWPZFJnfg-k/edit?usp=sharing

## Decision

If a data mapping requires any dependencies, use a separate class to resolve its dependencies and always use/inject a concrete data mapping type for better discoverability, e.g. [MapToVcsRunbookResources](https://github.com/OctopusDeploy/OctopusDeploy/blob/4702ef0677895d7d41f22af337e0d66f736f787c/source/Octopus.Server/Web/Mapping/MapToVcsRunbookResources.cs#L12). 

If dependencies are not required, use a private method (e.g. [MapToRunbookDomainModel](https://github.com/OctopusDeploy/OctopusDeploy/compare/spike-tenant-domain-model...spike-tenant-domain-model-runbooks#diff-a5c39abbfe2fc86ceb6451588fa356b4a2d73ed1504d87f1a230686ee6c5bd6eR89)) when testing of the mapping itself, and reuse of the mapping, is not required. Otherwise, use a static method.

## Data Sources

Source listed below might provide additional context but keep in mind that they can disappear at any time.

- [Different Explicit Data Mapping Solutions](https://docs.google.com/document/d/1C-4DRUEaER_zWmecM5lECkZAHOie3mlaMWPZFJnfg-k/edit?usp=sharing)
- [Project Config as Code Feedback](https://octopusdeploy.slack.com/archives/C01AJE4K3T2/p1602486226148900)
- [Initial Discussion PR](https://github.com/OctopusDeploy/Architecture/pull/21)