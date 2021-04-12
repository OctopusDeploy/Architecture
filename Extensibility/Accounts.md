# Account Extensibility

## Overview

Accounts are currently subtyped per row in the Accounts table in Octopus server. This document outlines some of the issues with this, and a proposed solution.

The following are some of the issues that have been found in previous attempts to make Account extensible

- Nevermore will fail hard when querying the table if it doesn’t understand the types being returned.
- Account subtypes all currently live in Octopus.Core and inherit from an Account base type. The model needs to change or this type has to move to ServerExtensibility. This move would drag a number of internal persistence concerns out into the extension/module world.

## Proposed solution

Change the model so that the Account table works more like the Machine table. Each row would be a sealed Account type, which stays in Octopus.Core, and it would have a property that is an interface from ServerExtensibility. 

The JsonConverter we then implement to handle this property could be more tolerant than we are today, and return null if it hits something it doesn’t understand. This means the row can still load, but would have a null property value and the server code can choose what to do about that.

Pros:

- We don’t bleed a lot of Octopus.Core persistence concerns into the extension/module world
- We get a better upgrade/downgrade story. If you rolled back to an earlier version of Octopus that didn’t have a module then queries wouldn’t explode
- We would expect modules to always be loaded, but if one wasn’t for some reason that isn’t going to cause the queries to explode, they’ll just filter things out
- Migrator might have some wins here too, with a move to being more tolerant around the shapes of data

Cons:

- We need to add mapping in at the API layer, because resource models would retain their current shape for backward compatibility 