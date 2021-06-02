# Introduction

One of the selling points of Octopus is its cloud neutrality. We aim to support all the most popular cloud native platforms, while retaining a consistent user experience and common best practices.

To achieve this, all cloud steps in Octopus should aim to implement a consistent set of base functionality. This document provides guidelines for pitching steps that are consistent, implement the lessons we have learned over the years, and express our opinions on best practice deployments.

# Ship incrementally

Team steps has devoted a great deal of effort to building a framework that allows features to be shipped independently of the main Octopus code base. This allows us to ship features more frequently, and respond to user feedback more quickly.

To take advantage of this, pitches should be broken down into milestones. Each milestone should:

* Deliver enough value to be useful for our customers.
* Lean towards delivering simple solutions that provide a easy migration path should more complexity be required in future milestones.

# Be opinionated

We are leaders in the deployment space. We have years of experience and our tools have performed millions of deployments, and customers look to us to provide clear opinions on what best practice deployments look like. These values have been distilled into [https://octopus.com/blog/ten-pillars-of-pragmatic-deployments](https://octopus.com/blog/ten-pillars-of-pragmatic-deployments).

The steps we deliver must have a clear vision for how they enable teams to deliver best practice deployments. Often this means combining, hiding, not supporting, or hard coding some features in the platforms we deploy to.

Where there is no clear decision on an opinion, refer to **Ship incrementally** for guidance.

# Define a path

Highly opinionated steps will fail some of our customers all of the time. To support those with advanced use cases, steps must provide a graduated path from opinionated (and often monolithic) steps, to granular and composable steps, to raw templates or scripts.

This path allows us to express our opinions regarding best practice deployments with opinionated steps that will often merge many underlying platforms and resources. The granular steps provide the ability to compose deployments in unique ways, while still retaining the benefits of a UI driven approach. Raw templates or scripts provide the ultimate level of customization, free of the opinions baked into the specialized steps.

