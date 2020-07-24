# Overview

- **Subject**: How we remove functionality from Octopus Server
- **Decision date**: [when was the decision made]
- **Decision contributors**: [list who was involved in the decision]
- **Decision maker**: [who made the final decision]
- **Decision owner**: [who executes on the final decision]

# Executive Summary

In order to move forward and remove UX and technical debt we need to remove functionality from Octopus.

We do this by deprecating features and giving customers some time to transition over before eventually removing it. We develop features and procedures that assist customers in performing the transition.

# Overview

This document focuses on when we remove functionality from Octopus Server and the upgrade implications for customers

It does not cover
- How to decide whether to deprecate and when to remove
- Which functionality will help the user make the transition

# Background

**Deprecated** refers to functionality this has been replaced or has been planned for removal.

We have no set way to deprecate things in Octopus (and related products). We have strived for maximum compatibility and minimum pain for our customers. This has lead to:
- Little ever being truly deprecated and removed
- Features that don't quite fit together
- Deprecated features hanging around for years
- Old conversion and compatibility code hanging around
- An ever-increasing combination of Octopus and integration versions we need to support.
- Paralysis in the introduction of new features that break existing functionality

[Draft 1](https://docs.google.com/document/d/1JFgWg4T6yTjU1epz85bHHMaU73tQ43bggsq8tRFIaU8/edit#heading=h.b13zj4am1nqv) of this proposal with comments

# Breaking Changes Policy

## Breaking Releases

We designate certain releases to be breaking releases well in advance. All planned breaking changes are made in this release.

## Transition Period

We want to make the transition away from deprecated functionality smooth for customers. When we deprecate, we set the transition period by nominating the release it will be removed in.

The transition period should be minimum of 6 months, which also aligns with our support schedule.

When deprecating we add alerting and reporting to notify the user they are using deprecated functionality. We also add functionality and documentation to help the user to transition.

We also add telemetry to inform us whether customers have transitioned away. We may extend the transition period based on this data.

## Stepped Upgrade

There are times when we cannot automatically migrate data. For example, the goal of removing a feature (like `Management Certificates`) is to remove the code and data that goes along with it. We do not want the user to upgrade and find their configuration is gone, we want them to move away first.

Similarly when we transition from one feature to another (e.g. `Lifecycles/Channels` to `Pipelines`), we want to the user to transition to the new feature.

In this situation we need to force the user to stop at a particular version, perform the mitigation and then upgrade to the latest version. This stepping stone should be no shorter than the minimum transition period.

Even if we don't have these kinds of changes, we still recommend to customers that they take a stepped approach and run each step for some period of time (e.g. a week). This means they get all the benefits of the alerting we have built and can then move to the next stone with confidence. If customers always run a supported version, they will automatically fall into this pattern.

To put it another way, the stepping stone releases should never be so far apart that functionality is both deprecated and removed in that step.

## Schedule

### Option - As needed

We nominate `breaking releases` when required, which becomes the transition period. We may align things so that we batch several removals into one release.

The stepping stones would be any two releases that are no more than the minimum transition period apart.

This means we only need to keep the deprecated functionality for a small amount of time. It also avoids a "big bang" release with lots of changes.

Example, given a desired transition period of 6 months (3 releases):
| Deprecated In | Removed In | Transition Period |
| ------------- | ---------- | ----------------- |
| 2020.4.0 | 2021.1.0 | 6 months |
| 2020.4.1 | 2021.2.0 | 8 months |
| 2021.1.0 | 2021.4.0 | 6 months |

### Option - Once per year

We nominate the first release of the year (`.1.0`) as the `breaking release`. This regularity allow us and customers to make longer term. It also minimizes the number of times the customer needs to think about breaking changes. 

Functionality will be removed in the breaking release that comes after a breaking release that already had the functionality as deprecated. i.e a minimum of 12 months.

This means functionality would stay deprecated for 12 to 22 months (24 if it just missed).

The stepping stones would be major version to major version. (i.e max 12 months apart). The easiest rule would be "Upgrade to the last version of each year".

Examples:
| Deprecated In | Removed In | Transition Period |
| ------------- | ---------- | ----------------- |
| 2020.4.0 | 2022.1.0 | 18 months |
| 2021.1.0 | 2022.1.0 | 12 months |
| 2021.1.1 | 2023.1.0 | 24 months |


### Option - Once per year, shortened transition period

The breaking releases would be the same as the `Once per year` option. The difference is that functionality can be removed if at least 6 months (3 releases)has passed.

This means functionality would stay deprecated for 6 to 16 months (18 if it just missed).

The stepping stone advice is to upgrade to `202x.4.0` release or later of each major release. Again "last release of the year" would be the clearest.


Examples:
| Deprecated In | Removed In | Transition Period |
| ------------- | ---------- | ----------------- |
| 2020.4.0 | 2021.1.0 | 6 months |
| 2020.4.1 | 2022.1.0 | 18 months |
| 2021.1.0 | 2022.1.0 | 12 months |
| 2021.1.1 | 2023.1.0 | 12 months |

#### Option - Twice per year

We nominate `.1.0` and `.4.0` as the breaking releases. The 6 monthly interval gives us more opportunities and halves the breaking changes at one time. 

The same rule for transition period would apply as the once per year method.

Functionality would only need to stay deprecated for 6 to 10 months (12 if it just missed).

The stepping stones would be minor releases 3 or less releases apart. Customers who always stay within a supported version will never need to do a double step.

| Deprecated In | Removed In | Transition Period |
| ------------- | ---------- | ----------------- |
| 2020.4.0 | 2021.1.0 | 6 months |
| 2020.4.1 | 2022.1.0 | 12 months |
| 2021.1.0 | 2022.1.0 | 6 months |




