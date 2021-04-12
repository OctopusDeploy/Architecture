# Overview

- **Subject**: [explain what decision you are making]
- **Decision date**: [when was the decision made]
- **Decision contributors**: [list who was involved in the decision]
- **Decision maker**: [who made the final decision]
- **Decision owner**: [who executes on the final decision]

# Executive Summary

We will start officially deprecating things and removing them at a set time in the future. We will help customers understand they are using deprecated things and help them transition.

# Background

**Deprecated** refers to functions or elements that are in the process of being replaced by newer ones

We currently have no agreed way to deprecate things in Octopus (and related products). We have strived for maximum compatibility and minimum pain for our customers. 

This has lead to:
- Little ever being truly deprecated and removed
- Old conversion and compatibility code hanging around
- An ever-increasing combination of Octopus and integration versions we need to support.
- Paralysis in the introduction of breaking features
- Compromises to functionality and UX

This applies both to features (e.g. Azure Management Certificates, Deploy package to Cloud Target) and technical concerns (e.g. 3 different status endpoints, extra properties in the API).

# Reasons for Deprecation

Some of the reasons for breaking changes are:

- **Incremental Addition** to existing functionality.
- **Superseding Feature** that changes the core of or replaces existing functionality. Conversion from one to the other requires user input.
- **Removal** of a functionality

# Deciding on Approach

When we change a feature, we almost always want to keep backwards compatibility for some time. These are some approaches that can be used:

## Break Compatibility

The team and product decide that the downside of breaking the API is outweighed by the effort required to keep it compatible. The team makes no effort to preserve compatibility.

**Recommendation**: Use with caution and consultation with product

## Design for Compatibility

The feature may be able to be designed in such a way that it does not break compatibility, or at least not in most cases. There is a clear path between the new and the old.

This option would likely add some technical debt as the the schema no longer fits the usage. It's also likely that not all scenarios are compatible. Consider the scenario of round tripping of using a C# client that does not know the added field.

The conversion between old and new is deprecated.

**Recommendation**: Use sparingly

## API Versioning

It is possible to to convert from the old to the the new in both directions after upgrade. Post upgrade the old API may stop working in some circumstances that would make sense to the user.

The old API is deprecated.

**Recommendation**: Use when possible

## Side By Side

Likely the only option for superseding functionality.

Conversion from the old to the new functionality is difficult or not possible without user intervention.

The old functionality is deprecated.

**Recommendation**: Use when possible 

# Deprecating

When deprecating the team should:
- Identify the release the functionality is likely to be removed
- Add an open GitHub Issue to action that removal and provide details
- If practical attach a PR to remove the required code, which may become out of date, but still provides some assistance
- Hook in telemetry and other usage data to inform us to the extent the deprecated functionality is still in use to determine whether to extend the removal date
- Hook in notifications and alerting to the user so that they can identify they are using deprecated functionality
- User documentation to aid the user in migrating away
- Add some helper functionality to aid the user in their transition
- Ensure the old and new paths are covered by adequate test.

# Removal
